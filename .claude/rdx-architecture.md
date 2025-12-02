# Rancher Desktop Extensions (RDX) Architecture

## Overview

Rancher Desktop Extensions (RDX) are Docker Desktop-compatible extensions that run within Rancher Desktop. They are distributed as Docker images containing UI files, host binaries, and container services.

## Key Files

| File | Purpose |
|------|---------|
| `pkg/rancher-desktop/main/extensions/extensions.ts` | Core `ExtensionImpl` class - handles install/uninstall, metadata extraction |
| `pkg/rancher-desktop/main/extensions/manager.ts` | `ExtensionManagerImpl` - manages extension lifecycle, caches instances |
| `pkg/rancher-desktop/main/extensions/types.ts` | Type definitions for `ExtensionMetadata`, `Extension` interface |
| `pkg/rancher-desktop/main/extensions/index.ts` | Exports |
| `pkg/rancher-desktop/main/commandServer/httpCommandServer.ts` | HTTP API endpoints for extensions |
| `background.ts` | Command worker with `installExtension()`, `listExtensions()` |
| `pkg/rancher-desktop/store/extensions.ts` | Vuex store for UI state |

## Extension Metadata Structure

From Docker image's `/metadata.json`:

```typescript
interface ExtensionMetadata {
  icon: string;                    // Path to icon in image (e.g., "/icon.svg")
  ui?: {
    'dashboard-tab'?: {
      title: string;               // Sidebar title
      root: string;                // Directory with UI files
      src: string;                 // Initial HTML page
      backend?: { socket: string; }
    }
  };
  vm?: ({ image: string } | { composefile: string }) & {
    exposes?: { socket: string; }
  };
  host?: {
    binaries: PlatformSpecific<{ path: string }[]>[];
    'x-rd-install'?: PlatformSpecific<string | string[]>;    // Post-install script
    'x-rd-uninstall'?: PlatformSpecific<string | string[]>;  // Pre-uninstall script
    'x-rd-shutdown'?: PlatformSpecific<string | string[]>;   // Shutdown script
  };
}
```

## Installation Directory Structure

Extensions are installed to `{extensionRoot}/{base64url-encoded-id}/`:

```
{extensionRoot}/
  {base64url(image-id)}/
    version.txt          # Contains the installed tag/version
    metadata.json        # Cached metadata from Docker image
    labels.json          # Cached Docker image labels
    icon.{ext}           # Extracted icon file
    bin/                 # Host binaries
    ui/
      dashboard-tab/     # Extracted UI files
    compose/
      compose.yaml       # Container compose file
```

## Key Classes

### ExtensionImpl (`extensions.ts`)

Represents a single extension (image:tag combination).

**Important memoized properties:**
- `_metadata: Promise<ExtensionMetadata>` - Extracted from Docker image
- `_labels: Promise<Record<string, string>>` - Docker image labels
- `_iconName: Promise<string>` - Computed icon filename
- `_composeFile: Promise<any>` - Parsed compose file
- `_composeName: string` - Compose project name

**Key methods:**
- `install(allowedImages)` - Extract and install extension
- `uninstall()` - Remove extension and clear caches
- `isInstalled()` - Check if version.txt matches
- `readFile(path)` - Read file from Docker image
- `extractFile(src, dest)` - Copy file from Docker image

### ExtensionManagerImpl (`manager.ts`)

Manages all extensions.

**Caching:**
```typescript
protected extensions: Record<string, Record<string, ExtensionImpl>> = {};
// extensions[imageName][tag] = ExtensionImpl instance
```

**Key methods:**
- `getExtension(image, options)` - Get or create ExtensionImpl
- `getInstalledExtensions()` - List all installed extensions
- `init(config)` - Initialize and restore extensions from settings
- `shutdown()` - Clean up listeners and processes

## Installation Flow

1. `rdctl extension install <image>` calls HTTP API
2. `httpCommandServer.installExtension()` invokes `commandWorker.installExtension()`
3. `ExtensionManagerImpl.getExtension()` gets/creates `ExtensionImpl`
4. `ExtensionImpl.install()`:
   - Reads `metadata.json` from Docker image
   - Creates installation directory
   - Extracts icon, UI files, binaries, compose files
   - Starts containers via `compose up`
   - Writes `version.txt` to mark complete
   - Runs post-install script if defined
   - Updates settings

## Uninstallation Flow

1. `rdctl extension uninstall <image>` calls HTTP API
2. `ExtensionImpl.uninstall()`:
   - Runs pre-uninstall script if defined
   - Stops containers via `compose down`
   - Deletes installation directory
   - **Clears memoized caches** (fixed in this PR)
   - Updates settings

## Common Issues & Gotchas

### Metadata Caching Bug (Fixed)

**Problem:** `ExtensionImpl` instances are cached in `ExtensionManagerImpl.extensions`. When uninstalling, memoized properties (`_metadata`, etc.) were not cleared. Reinstalling reused the cached instance with stale metadata.

**Fix:** Clear all memoized caches in `uninstall()` before returning.

### Installed Tab Uninstall Button Not Working (Fixed)

**Problem:** In `installed.vue`, the SortableTable used `key-field="description"` but `ExtensionState` doesn't have a `description` field. This caused all rows to have `undefined` as their Vue key, breaking Vue's reactivity and event handling.

**Fix:** Change `key-field` to `"id"` which is the unique identifier in `ExtensionState`.

**Key UI Files:**
- `pkg/rancher-desktop/pages/extensions/installed.vue` - Installed tab
- `pkg/rancher-desktop/components/MarketplaceCard.vue` - Catalog cards

**ExtensionState fields:** `id`, `version`, `metadata`, `labels`, `availableVersion`, `canUpgrade`

### ddClient.extension Missing id and version Fields (Fixed)

**Problem:** The `ddClient.extension` object provided to extensions only had the `image` field set to the extension ID (without version). The `id` and `version` fields were missing entirely. According to Docker Desktop API:
- `image` should be `id:version` (full image reference)
- `id` should be the image name without tag
- `version` should be just the tag

**Fix:**
1. In `window/index.ts`, look up extension version from settings
2. Pass `extensionId` and `extensionVersion` to preload script via `additionalArguments`
3. Update `RDXClient` constructor to set all three fields properly

**Key Files:**
- `pkg/rancher-desktop/window/index.ts` - Creates WebContentsView and passes extension info
- `pkg/rancher-desktop/preload/extensions.ts` - Defines `RDXClient` and `ddClient.extension`

### Container Namespace

Extensions use a dedicated namespace: `rancher-desktop-extensions`

```typescript
static readonly extensionNamespace = 'rancher-desktop-extensions';
```

### Protocol Handler Registration

Extension UIs get their own Electron session with a custom protocol:
```typescript
const encodedId = Buffer.from(this.id).toString('hex');
await mainEvents.invoke('extensions/register-protocol', `persist:rdx-${encodedId}`);
```

### Platform-Specific Binaries

Metadata uses platform keys: `windows`, `linux`, `darwin`

```typescript
protected get platform() {
  switch (process.platform) {
    case 'win32': return 'windows';
    case 'linux':
    case 'darwin': return process.platform;
  }
}
```

## CLI Commands

```bash
rdctl extension install <image>    # Install extension
rdctl extension uninstall <image>  # Uninstall extension
rdctl extension ls                 # List installed extensions
```

## HTTP API Endpoints

- `GET /v1/extensions` - List extensions
- `POST /v1/extensions/install?id=<image>` - Install
- `POST /v1/extensions/uninstall?id=<image>` - Uninstall

## Testing

- Unit tests: `pkg/rancher-desktop/main/extensions/__tests__/extensions.spec.ts`
- E2E tests: `e2e/extensions.e2e.spec.ts`

## Debugging Tips

1. Check extension logs in the Extensions logging category
2. Look at `{extensionRoot}/{encoded-id}/` for installed files
3. Verify Docker image has valid `metadata.json` with required `icon` field
4. Check compose containers: `nerdctl --namespace rancher-desktop-extensions ps`
