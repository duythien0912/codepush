# Phase 2 Research Findings

## Critical Discoveries

### 1. Shorebird Updater Architecture

**FINDING**: The updater is a **Rust library** with **Flutter Dart bindings**, NOT a standalone Rust binary.

**Components**:
- `library/` - Rust core (network, config, patch application)
- `shorebird_code_push/` - Flutter package (Dart API)
- Integration: Flutter → Dart FFI → Rust library

### 2. Base URL Configuration ✅

**File**: `library/src/config.rs`
```rust
const DEFAULT_BASE_URL: &str = "https://api.shorebird.dev";
```

**Override Method**: In `shorebird.yaml`:
```yaml
app_id: your-app-id
base_url: https://your-worker.workers.dev  # Custom server
channel: stable
auto_update: false
```

### 3. API Endpoints ✅

**Check for Updates**:
```
POST {base_url}/api/v1/patches/check
```

**Request Body**:
```json
{
  "app_id": "com.example.app",
  "channel": "stable",
  "release_version": "1.0.0+1",
  "platform": "android",
  "arch": "aarch64",
  "client_id": "random-uuid"
}
```

**Response Body**:
```json
{
  "patch_available": true,
  "patch": {
    "number": 1,
    "hash": "sha256_hex_string",
    "download_url": "https://storage.../patch.bin",
    "hash_signature": null
  },
  "rolled_back_patch_numbers": []
}
```

**Report Events**:
```
POST {base_url}/api/v1/patches/events
```

### 4. Patch File Format ✅

**Format**: Binary compressed patch
- **Compression**: Zstd
- **Patch Algorithm**: bipatch (binary diff)
- **Base**: `libapp.so` from APK/IPA
- **Output**: Full patched `libapp.so`
- **Verification**: SHA256 hash check

**File Extension**: No specific extension (binary blob)

### 5. Patch Workflow ✅

**Developer Side**:
1. Run `shorebird patch android`
2. Shorebird CLI uploads to Shorebird cloud
3. Developer downloads patch manually from Shorebird dashboard
4. Developer uploads to Cloudflare using CLI tool

**End User Side**:
1. App calls `ShorebirdUpdater().checkForUpdate()`
2. Rust library sends POST to `{base_url}/api/v1/patches/check`
3. If update available, downloads from `download_url`
4. Verifies SHA256 hash
5. Applies patch using bipatch
6. Saves to disk
7. Next app restart loads patched version

### 6. Client Integration ✅

**CORRECTION**: Plan was WRONG about "Rust binary"

**Actual Integration**:
```yaml
# pubspec.yaml
dependencies:
  shorebird_code_push: ^1.0.0
```

```yaml
# shorebird.yaml (in Flutter project root)
app_id: your-app-id
base_url: https://your-worker.workers.dev
auto_update: false
```

```dart
// main.dart
import 'package:shorebird_code_push/shorebird_code_push.dart';

void main() async {
  final updater = ShorebirdUpdater();
  
  if (updater.isAvailable) {
    final status = await updater.checkForUpdate();
    if (status == UpdateStatus.outdated) {
      await updater.update();
      // Restart required
    }
  }
  
  runApp(MyApp());
}
```

### 7. Shorebird Patch Command Behavior ✅

**Command**: `shorebird patch android`

**What it does**:
1. Builds release APK/AAB
2. Extracts `libapp.so`
3. Creates binary diff from base version
4. Compresses with zstd
5. **Uploads directly to Shorebird cloud**
6. Updates metadata on Shorebird servers

**Problem**: No direct file output to extract

**Solution (Option C)**:
1. Let Shorebird upload normally
2. Download patch from Shorebird dashboard/API
3. Re-upload to Cloudflare R2

### 8. Database Schema (Finalized)

```sql
-- D1 Database
CREATE TABLE patches (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  app_id TEXT NOT NULL,
  release_version TEXT NOT NULL,
  patch_number INTEGER NOT NULL,
  platform TEXT NOT NULL,
  arch TEXT NOT NULL,
  download_url TEXT NOT NULL,
  hash TEXT NOT NULL,
  hash_signature TEXT,
  created_at INTEGER NOT NULL,
  UNIQUE(app_id, release_version, platform, arch, patch_number)
);

CREATE INDEX idx_patches_lookup 
ON patches(app_id, release_version, platform, arch);

CREATE TABLE api_keys (
  key TEXT PRIMARY KEY,
  app_id TEXT NOT NULL,
  created_at INTEGER NOT NULL
);
```

### 9. R2 Storage Structure

```
Bucket: shorebird-patches

Path format:
{app_id}/{platform}/{arch}/{release_version}/{patch_number}

Example:
com.example.app/android/aarch64/1.0.0+1/1
com.example.app/android/x86_64/1.0.0+1/1
com.example.app/ios/aarch64/1.0.0+1/1
```

### 10. Architecture (Revised)

```
┌─────────────────────────────────────────┐
│           DEVELOPER                     │
│                                         │
│  1. shorebird patch android             │
│     → Uploads to Shorebird cloud        │
│                                         │
│  2. Download patch from Shorebird       │
│     → Manual or via API                 │
│                                         │
│  3. shorebird_custom_server upload      │
│     → Uploads to Cloudflare R2          │
│     → Updates D1 metadata               │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         CLOUDFLARE                      │
│                                         │
│  Workers:                               │
│    POST /api/v1/patches/check           │
│    POST /api/upload (CLI only)          │
│                                         │
│  D1: Patch metadata                     │
│  R2: Patch files                        │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         END USER (Flutter App)          │
│                                         │
│  shorebird.yaml:                        │
│    base_url: https://worker.dev         │
│                                         │
│  App checks Cloudflare for updates      │
│  Downloads and applies patches          │
└─────────────────────────────────────────┘
```

## Implementation Changes Required

### Plan Corrections

1. **03-CLIENT.md** - MAJOR ERROR
   - ❌ OLD: "Clone Rust updater, change config.rs"
   - ✅ NEW: "Add Flutter package, configure shorebird.yaml"

2. **02-CLI.md** - Workflow Update
   - ❌ OLD: "Extract .vmcode file from shorebird output"
   - ✅ NEW: "Download patch from Shorebird, upload to Cloudflare"

3. **01-BACKEND.md** - API Endpoint
   - ❌ OLD: `GET /api/check`
   - ✅ NEW: `POST /api/v1/patches/check`

4. **All Docs** - Add Missing Info
   - ✅ Request/response JSON schemas
   - ✅ Patch file format details
   - ✅ SHA256 verification
   - ✅ Platform/arch handling

## Blocking Issues Resolved

✅ **B1. Patch Extraction** - Use Option C (download from Shorebird)
✅ **B2. Updater Integration** - Use Flutter package + shorebird.yaml
✅ **B3. API Format** - Documented exact JSON schemas
✅ **B4. Base URL Override** - Confirmed via shorebird.yaml

## Ready for Documentation Rewrite

All critical information gathered. Proceeding to rewrite all planning documents.
