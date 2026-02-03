# Plan: Self-Hosted Code Push with Shorebird Engine

## Overview

Use Shorebird's patched engine, self-host API and CDN for check update and download.

---

## Architecture

```
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   Client    │────→│  Your API   │────→│  Your DB    │
│ (Shorebird  │     │ (check/     │     │ (versions,  │
│   engine)   │←────│  download)  │     │   patches)  │
└──────┬──────┘ └─────────────┘ └─────────────┘
       │
       ↓
┌─────────────┐
│  Your CDN   │
│(patch files)│
└─────────────┘
```

---

## 1. Build System (Shorebird CLI)

### 1.1 Build Release

```bash
# Input: Flutter source code
# Output: Release artifact with patched engine

shorebird release android --flutter-version 3.24.0
shorebird release ios --flutter-version 3.24.0

# Artifacts located at:
# build/app/outputs/flutter-apk/app-release.apk
# build/ios/ipa/*.ipa
```

### 1.2 Create Patch

```bash
# Input: Modified source code
# Output: Patch file (.vmcode/.bin)

shorebird patch android
shorebird patch ios

# Patch file extracted from:
# build/shorebird/.../patch.vmcode
```

---

## 2. Upload System

### 2.1 Extract Patch

```bash
# After shorebird patch, find and copy patch file
find build -name "*.vmcode" -exec cp {} patches/patch_{version}_{platform}.bin \;
```

### 2.2 Upload to CDN

```bash
# Upload patch file to S3/MinIO/R2
aws s3 cp patch_5_android.bin s3://your-bucket/patches/
  → https://cdn.yourapp.com/patches/patch_5_android.bin
```

### 2.3 Register to Database

```sql
INSERT INTO patches (
  app_id,
  platform,
  patch_number,
  min_app_version,
  download_url,
  sha256,
  active,
  created_at
) VALUES (
  'com.yourapp.name',
  'android',
  5,
  '1.0.0',
  'https://cdn.yourapp.com/patches/patch_5_android.bin',
  'sha256_of_file',
  true,
  NOW()
);
```

---

## 3. Client System (Flutter)

### 3.1 Initialize

```dart
// main.dart
void main() {
  final updater = YourUpdater(
    apiUrl: 'https://api.yourapp.com',
    appId: 'com.yourapp.name',
    apiKey: 'your-key',
  );

  runApp(MyApp(updater: updater));
}
```

### 3.2 Check Update

```dart
// your_updater.dart
Future<UpdateInfo?> checkUpdate() async {
  // Get current version from package_info
  currentVersion = await PackageInfo.fromPlatform().version;
  currentPatch = await getCurrentPatchNumber(); // from local storage

  // Call YOUR API
  response = await http.get(
    '$apiUrl/v1/update',
    params: {
      'app_id': appId,
      'version': currentVersion,
      'platform': Platform.operatingSystem,
      'patch': currentPatch.toString(),
    },
    headers: {'X-Key': apiKey},
  );

  if (response.has_update == false) return null;

  return UpdateInfo(
    patchNumber: response.patch_number,
    downloadUrl: response.url,
    hash: response.hash,
  );
}
```

### 3.3 Download Patch

```dart
Future<File> downloadPatch(UpdateInfo info) async {
  // Download from YOUR CDN
  response = await http.get(info.downloadUrl);

  // Verify hash
  actualHash = sha256(response.body);
  if (actualHash != info.hash) throw ChecksumError();

  // Save to temp
  file = File(tempDir + '/patch_${info.patchNumber}.bin');
  await file.writeAsBytes(response.bodyBytes);

  return file;
}
```

### 3.4 Apply Patch (Shorebird Engine)

```dart
Future<bool> applyPatch(File patchFile) async {
  // Call Shorebird native function
  // Engine handles iOS interpreter / Android AOT

  result = nativeApplyPatch(patchFile.path);

  if (result.success) {
    await saveCurrentPatchNumber(newPatchNumber);
    return true;
  }

  return false;
}
```

### 3.5 Full Update Flow

```dart
Future<void> performUpdate() async {
  // 1. Check
  update = await checkUpdate();
  if (update == null) return; // No update

  // 2. Download
  file = await downloadPatch(update);

  // 3. Apply (Shorebird engine handles platform specifics)
  success = await applyPatch(file);

  // 4. Report (optional)
  if (success) {
    await reportStatus('success', update.patchNumber);
    await restartApp(); // or prompt user
  } else {
    await reportStatus('failed', update.patchNumber);
  }
}
```

---

## 4. Backend API (Node.js/Express)

### 4.1 Check Update Endpoint

```javascript
// GET /v1/update?app_id=xxx&version=1.0.0&platform=android&patch=3

function handleCheckUpdate(req, res) {
  const { app_id, version, platform, patch } = req.query;

  // Query database
  const latestPatch = db.patches.findOne({
    appId: app_id,
    platform: platform,
    active: true,
    patchNumber: { $gt: parseInt(patch) },
    minAppVersion: { $lte: version },
  });

  if (!latestPatch) {
    return res.json({ has_update: false });
  }

  return res.json({
    has_update: true,
    patch_number: latestPatch.patchNumber,
    url: latestPatch.downloadUrl,
    hash: latestPatch.sha256,
    size: latestPatch.size,
  });
}
```

### 4.2 Report Status Endpoint (Optional)

```javascript
// POST /v1/report

function handleReport(req, res) {
  const { device_id, patch_number, status, error } = req.body;

  db.analytics.insert({
    deviceId: device_id,
    patchNumber: patch_number,
    status: status, // 'success', 'failed', 'rolled_back'
    error: error,
    timestamp: new Date(),
  });

  res.json({ ok: true });
}
```

---

## 5. Database Schema

### 5.1 Patches Table

```sql
CREATE TABLE patches (
  id UUID PRIMARY KEY,
  app_id VARCHAR(255) NOT NULL,
  platform VARCHAR(10) CHECK (platform IN ('android', 'ios')),
  patch_number INTEGER NOT NULL,
  min_app_version VARCHAR(20) NOT NULL,
  download_url TEXT NOT NULL,
  sha256 VARCHAR(64) NOT NULL,
  size_bytes INTEGER,
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),

  UNIQUE(app_id, platform, patch_number)
);
```

### 5.2 Analytics Table (Optional)

```sql
CREATE TABLE patch_installs (
  id UUID PRIMARY KEY,
  device_id VARCHAR(255) NOT NULL,
  patch_id UUID REFERENCES patches(id),
  status VARCHAR(20) CHECK (status IN ('success', 'failed', 'rolled_back')),
  error_message TEXT,
  installed_at TIMESTAMP DEFAULT NOW()
);
```

---

## 6. Rollback Flow

```javascript
// To rollback patch 5 to patch 4:

// Option 1: Disable patch 5
db.patches.update(
  { appId: 'xxx', patchNumber: 5 },
  { $set: { active: false } }
);

// Option 2: Create patch 6 = patch 4 (revert)
// Upload patch 4 content as patch 6
```

---

## 7. Deployment Flow

```
Developer                    Your System                  Client
─────────                    ───────────                  ──────
   │                              │                          │
   │── shorebird release ────────→│                          │
   │   (build with engine)        │                          │
   │                              │                          │
   │── shorebird patch ──────────→│                          │
   │   (generate patch file)      │                          │
   │                              │                          │
   │── upload to S3 ─────────────→│                          │
   │                              │── save to CDN ─────────→│
   │                              │                          │
   │── insert to DB ─────────────→│                          │
   │   (patch metadata)           │                          │
   │                              │                          │
   │                                                         │
   │                              │←── check update ─────────│
   │                              │    (call your API)       │
   │                              │── return update info ───→│
   │                                                         │
   │                              │←── download patch ───────│
   │                              │    (from your CDN)       │
   │                              │── patch file ───────────→│
   │                                                         │
   │                                                         │── apply
   │                                                         │   (Shorebird
   │                                                         │    engine)
   │                                                         │
   │                              │←── report status ────────│
```
