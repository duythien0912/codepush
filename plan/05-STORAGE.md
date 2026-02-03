# 05 - STORAGE STRUCTURE (R2)

**Platform**: Cloudflare R2 (S3-compatible)

---

## Overview

R2 bucket structure for storing patch files.

**CRITICAL UPDATE**: Path now includes architecture.

---

## Bucket Structure

```
Bucket: shorebird-patches

Path format:
{app_id}/{platform}/{arch}/{release_version}/{patch_number}

Examples:
com.example.app/android/aarch64/1.0.0+1/1
com.example.app/android/x86_64/1.0.0+1/1
com.example.app/ios/aarch64/1.0.0+1/1
com.example.app/ios/aarch64/1.0.0+1/2
```

---

## Path Components

| Component | Description | Example |
|-----------|-------------|---------|
| app_id | App bundle ID | com.example.app |
| platform | android or ios | android |
| arch | aarch64, x86_64, x86, arm | aarch64 |
| release_version | Release version | 1.0.0+1 |
| patch_number | Patch number | 1 |

---

## Why This Structure?

### 1. Organized by App

```
com.example.app/
├── android/
│   ├── aarch64/
│   └── x86_64/
└── ios/
    └── aarch64/
```

Easy to find all patches for one app.

### 2. Separated by Platform

```
android/
├── aarch64/
└── x86_64/
```

Android and iOS patches are separate.

### 3. Separated by Architecture

```
aarch64/
├── 1.0.0+1/
│   ├── 1
│   └── 2
└── 1.0.1+2/
    └── 1
```

Different architectures have different patches.

### 4. Versioned

```
1.0.0+1/
├── 1  (first patch)
├── 2  (second patch)
└── 3  (third patch)
```

Multiple patches per version.

---

## File Format

**Extension**: None (binary blob)

**Content**: Zstd-compressed bipatch binary

**Size**: Typically 10-100 KB

**MIME Type**: `application/octet-stream`

---

## Public Access

R2 bucket must have public access enabled for downloads.

**Public URL Format**:
```
https://pub-{random}.r2.dev/{path}
```

**Example**:
```
https://pub-abc123xyz.r2.dev/com.example.app/android/aarch64/1.0.0+1/1
```

---

## Upload Example

### Using Wrangler

```bash
wrangler r2 object put shorebird-patches/com.example.app/android/aarch64/1.0.0+1/1 \
  --file=patch.bin
```

### Using Worker (Recommended)

```javascript
await env.R2.put(
  'com.example.app/android/aarch64/1.0.0+1/1',
  patchBytes
);
```

---

## Download Example

### Direct URL

```bash
curl https://pub-abc123xyz.r2.dev/com.example.app/android/aarch64/1.0.0+1/1 \
  -o patch.bin
```

### From Worker

```javascript
const object = await env.R2.get('com.example.app/android/aarch64/1.0.0+1/1');
const bytes = await object.arrayBuffer();
```

---

## Listing Objects

### List all patches for app

```bash
wrangler r2 object list shorebird-patches \
  --prefix=com.example.app/
```

### List patches for specific version

```bash
wrangler r2 object list shorebird-patches \
  --prefix=com.example.app/android/aarch64/1.0.0+1/
```

---

## Deletion

### Delete specific patch

```bash
wrangler r2 object delete shorebird-patches/com.example.app/android/aarch64/1.0.0+1/1
```

### Delete all patches for version

```bash
# List first
wrangler r2 object list shorebird-patches \
  --prefix=com.example.app/android/aarch64/1.0.0+1/

# Delete each one
wrangler r2 object delete shorebird-patches/com.example.app/android/aarch64/1.0.0+1/1
wrangler r2 object delete shorebird-patches/com.example.app/android/aarch64/1.0.0+1/2
```

---

## Storage Limits (Free Tier)

- **Storage**: 10 GB/month
- **Class A Operations** (writes): 1M/month
- **Class B Operations** (reads): 10M/month
- **Egress**: Unlimited (FREE!)

**Typical Usage**:
- Patch size: 50 KB
- 10 GB = ~200,000 patches
- 10M reads = ~10M downloads/month

**Enough for**:
- 100 apps
- 10,000 users
- 100 patches/day

---

## Best Practices

### 1. Use Consistent Naming

Always use same format:
```
{app_id}/{platform}/{arch}/{version}/{number}
```

### 2. No File Extensions

Don't add `.bin` or `.patch`:
```
✅ com.example.app/android/aarch64/1.0.0+1/1
❌ com.example.app/android/aarch64/1.0.0+1/1.bin
```

### 3. URL Encode Special Characters

If app_id has special chars:
```
com.example.app → com.example.app (OK)
com.example/app → com.example%2Fapp (encode /)
```

### 4. Keep Metadata in D1

Don't rely on R2 for metadata:
- ✅ Store hash in D1
- ✅ Store size in D1
- ✅ Store created_at in D1
- ❌ Don't query R2 for metadata

---

## Migration from Old Structure

If you have old structure without arch:

**Old**:
```
com.example.app/android/1.0.0+1/1
```

**New**:
```
com.example.app/android/aarch64/1.0.0+1/1
```

**Migration Script**:
```bash
# List old objects
wrangler r2 object list shorebird-patches --prefix=com.example.app/android/1.0.0+1/

# Copy to new location (assume aarch64)
wrangler r2 object get shorebird-patches/com.example.app/android/1.0.0+1/1 --file=temp.bin
wrangler r2 object put shorebird-patches/com.example.app/android/aarch64/1.0.0+1/1 --file=temp.bin

# Delete old
wrangler r2 object delete shorebird-patches/com.example.app/android/1.0.0+1/1
```

---

## Summary

**Key Changes**:
- ✅ Added `arch` to path
- ✅ Path format: `{app_id}/{platform}/{arch}/{version}/{number}`
- ✅ Supports multi-architecture apps

**Why Important**:
- Different architectures need different patches
- Prevents wrong patch from being downloaded
- Matches Shorebird's architecture handling

---

## Next

See [06-SECURITY.md](06-SECURITY.md) for security considerations.
