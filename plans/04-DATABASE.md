# 04 - DATABASE SCHEMA (D1)

**Platform**: Cloudflare D1 (SQLite)

---

## Overview

Database schema for storing patch metadata.

**CRITICAL UPDATES**:
- Added `arch` column (aarch64, x86_64, x86, arm)
- Updated unique constraint to include arch
- Updated indexes for performance

---

## Complete Schema

```sql
-- Patches table
CREATE TABLE patches (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  app_id TEXT NOT NULL,
  release_version TEXT NOT NULL,
  patch_number INTEGER NOT NULL,
  platform TEXT NOT NULL CHECK(platform IN ('android', 'ios')),
  arch TEXT NOT NULL CHECK(arch IN ('aarch64', 'x86_64', 'x86', 'arm')),
  download_url TEXT NOT NULL,
  hash TEXT NOT NULL,
  hash_signature TEXT,
  active INTEGER DEFAULT 1,
  created_at INTEGER NOT NULL,
  UNIQUE(app_id, release_version, platform, arch, patch_number)
);

-- Index for fast lookups
CREATE INDEX idx_patches_lookup 
ON patches(app_id, release_version, platform, arch, active, patch_number);

-- API keys table
CREATE TABLE api_keys (
  key TEXT PRIMARY KEY,
  app_id TEXT NOT NULL,
  created_at INTEGER NOT NULL
);
```

---

## Field Descriptions

### patches table

| Field | Type | Description |
|-------|------|-------------|
| id | INTEGER | Auto-increment primary key |
| app_id | TEXT | App bundle ID (e.g., com.example.app) |
| release_version | TEXT | Release version (e.g., 1.0.0+1) |
| patch_number | INTEGER | Patch number (starts at 1) |
| platform | TEXT | android or ios |
| arch | TEXT | aarch64, x86_64, x86, or arm |
| download_url | TEXT | R2 public URL for patch file |
| hash | TEXT | SHA256 hash (hex string) |
| hash_signature | TEXT | Optional signature for verification |
| active | INTEGER | 1 = active, 0 = rolled back |
| created_at | INTEGER | Unix timestamp |

### api_keys table

| Field | Type | Description |
|-------|------|-------------|
| key | TEXT | API key (primary key) |
| app_id | TEXT | Associated app ID |
| created_at | INTEGER | Unix timestamp |

---

## Unique Constraint

```sql
UNIQUE(app_id, release_version, platform, arch, patch_number)
```

**Why**: Prevents duplicate patches for same app/version/platform/arch combination.

**Example**: Can have:
- `com.example.app / 1.0.0+1 / android / aarch64 / patch 1` ✅
- `com.example.app / 1.0.0+1 / android / x86_64 / patch 1` ✅
- `com.example.app / 1.0.0+1 / ios / aarch64 / patch 1` ✅

But NOT:
- `com.example.app / 1.0.0+1 / android / aarch64 / patch 1` (duplicate) ❌

---

## Query Examples

### Check for Update

```sql
SELECT patch_number, download_url, hash, hash_signature
FROM patches
WHERE app_id = 'com.example.app'
  AND release_version = '1.0.0+1'
  AND platform = 'android'
  AND arch = 'aarch64'
  AND active = 1
ORDER BY patch_number DESC
LIMIT 1;
```

### Insert Patch

```sql
INSERT INTO patches (
  app_id, release_version, patch_number, platform, arch,
  download_url, hash, hash_signature, created_at
) VALUES (
  'com.example.app',
  '1.0.0+1',
  1,
  'android',
  'aarch64',
  'https://pub-xxx.r2.dev/...',
  'abc123def456...',
  NULL,
  1704067200
);
```

### Rollback Patch

```sql
UPDATE patches
SET active = 0
WHERE app_id = 'com.example.app'
  AND release_version = '1.0.0+1'
  AND platform = 'android'
  AND arch = 'aarch64'
  AND patch_number = 1;
```

### List All Patches for App

```sql
SELECT 
  release_version,
  platform,
  arch,
  patch_number,
  active,
  created_at
FROM patches
WHERE app_id = 'com.example.app'
ORDER BY created_at DESC;
```

---

## Architecture Support

### Supported Architectures

**Android**:
- `aarch64` - ARM 64-bit (most common)
- `x86_64` - Intel 64-bit (emulators)
- `x86` - Intel 32-bit (old emulators)
- `arm` - ARM 32-bit (old devices)

**iOS**:
- `aarch64` - ARM 64-bit (all modern devices)

### Why Architecture Matters

Different architectures need different patch files because:
1. Binary code is architecture-specific
2. `libapp.so` differs per architecture
3. Patch must match base architecture

**Example**: Android app on ARM64 device needs `aarch64` patch, not `x86_64` patch.

---

## Migration from Old Schema

If you have old schema without `arch` column:

```sql
-- Add arch column
ALTER TABLE patches ADD COLUMN arch TEXT;

-- Set default for existing rows (Android ARM64)
UPDATE patches SET arch = 'aarch64' WHERE platform = 'android';

-- Set default for existing rows (iOS ARM64)
UPDATE patches SET arch = 'aarch64' WHERE platform = 'ios';

-- Add constraint (SQLite doesn't support ALTER TABLE ADD CONSTRAINT)
-- Need to recreate table with new schema
```

---

## Summary

**Key Changes**:
- ✅ Added `arch` column
- ✅ Updated unique constraint
- ✅ Updated indexes
- ✅ Added CHECK constraints

**Why Important**:
- Supports multi-architecture apps
- Prevents wrong patch from being applied
- Matches Shorebird's architecture handling

---

## Next

See [05-STORAGE.md](05-STORAGE.md) for R2 storage structure.
