# 03 - CLIENT IMPLEMENTATION (Flutter App Integration)

**Time**: 10 minutes  
**Difficulty**: Very Easy  
**Platform**: Flutter app with Shorebird

---

## Overview

Integrate Shorebird updater into your Flutter app to check for and apply patches from your Cloudflare server.

**What we'll do**:
1. Add `shorebird_code_push` package to Flutter project
2. Configure `shorebird.yaml` to point to Cloudflare
3. Add update check code to app
4. Test update flow

**CRITICAL CORRECTION**: The updater is a **Flutter package**, NOT a Rust binary to build.

---

## Prerequisites

✅ Flutter project with Shorebird initialized (`shorebird init` already run)  
✅ Cloudflare backend deployed (from Step 1)  
✅ Worker URL ready (e.g., `https://your-worker.workers.dev`)  
✅ App built with `shorebird release` at least once

---

## Step 1: Add Package to pubspec.yaml

**Goal**: Add Shorebird updater package to your Flutter project

**Input**: Your Flutter project directory

**Tool/Command**: Text editor + `flutter pub get`

**Detailed Actions**:

1. Open your Flutter project in terminal:
```bash
cd /path/to/your/flutter/project
```

2. Open `pubspec.yaml` in text editor

3. Add under `dependencies:`:
```yaml
dependencies:
  flutter:
    sdk: flutter
  shorebird_code_push: ^1.0.0  # Add this line
```

4. Save the file

5. Run:
```bash
flutter pub get
```

**Output**:
```
Running "flutter pub get" in your_project...
Resolving dependencies...
+ shorebird_code_push 1.0.0
Changed 1 dependency!
```

**How to verify**:
```bash
flutter pub deps | grep shorebird_code_push
```

Should show: `shorebird_code_push 1.0.0`

**What can go wrong**:
- "Package doesn't exist" → Check spelling: `shorebird_code_push` (underscore, not hyphen)
- "Version solving failed" → Try without version: `shorebird_code_push: any`
- Network error → Check internet connection
- "SDK version conflict" → Update Flutter: `flutter upgrade`

**If stuck**:
1. Delete `pubspec.lock`
2. Run `flutter clean`
3. Run `flutter pub get` again
4. Check https://pub.dev/packages/shorebird_code_push for latest version

---

## Step 2: Configure shorebird.yaml

**Goal**: Point Shorebird updater to your Cloudflare Worker instead of Shorebird cloud

**Input**: Your Worker URL from Step 1

**Tool/Command**: Text editor

**Detailed Actions**:

1. Open `shorebird.yaml` in your Flutter project root (same level as `pubspec.yaml`)

2. Current content should look like:
```yaml
app_id: abc-123-def-456
```

3. Add `base_url` and `auto_update` lines:
```yaml
app_id: abc-123-def-456
base_url: https://your-worker.workers.dev  # Add this
auto_update: false  # Add this (we'll handle updates manually)
```

4. **Replace** `your-worker` with your actual Worker name from Cloudflare dashboard

5. Save the file

**Output**: Updated `shorebird.yaml` file

**How to verify**:
```bash
cat shorebird.yaml
```

Should show your Worker URL.

**What can go wrong**:
- File doesn't exist → Run `shorebird init` first
- Wrong URL format → Must start with `https://` and end with `.workers.dev`
- Typo in Worker name → Copy-paste from Cloudflare dashboard
- Trailing slash → Remove it: ❌ `https://worker.dev/` ✅ `https://worker.dev`

**If stuck**:
1. Get Worker URL from Cloudflare dashboard:
   - Go to Workers & Pages
   - Click your worker
   - Copy URL from "Preview" section
2. Test URL in browser (should return JSON)
3. Make sure YAML syntax is correct (spaces, not tabs)

---

## Step 3: Add Update Check Code

**Goal**: Add update checking logic to your Flutter app

**Input**: Your app's `main.dart` file

**Tool/Command**: Text editor

**Detailed Actions**:

1. Open `lib/main.dart`

2. Add import at the top:
```dart
import 'package:shorebird_code_push/shorebird_code_push.dart';
```

3. Modify your `main()` function to add update check:

**BEFORE**:
```dart
void main() {
  runApp(MyApp());
}
```

**AFTER**:
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Create updater instance
  final updater = ShorebirdUpdater();
  
  // Check if updater is available (only works in release mode)
  if (updater.isAvailable) {
    print('[Shorebird] Checking for updates...');
    
    try {
      // Check for updates
      final status = await updater.checkForUpdate();
      
      if (status == UpdateStatus.outdated) {
        print('[Shorebird] Update available! Downloading...');
        await updater.update();
        print('[Shorebird] Update downloaded! Restart app to apply.');
      } else if (status == UpdateStatus.upToDate) {
        print('[Shorebird] App is up to date!');
      } else if (status == UpdateStatus.restartRequired) {
        print('[Shorebird] Update ready! Please restart app.');
      }
    } catch (e) {
      print('[Shorebird] Update check failed: $e');
    }
  } else {
    print('[Shorebird] Updater not available (debug mode or not built with Shorebird)');
  }
  
  runApp(MyApp());
}
```

4. Save the file

**Output**: Code added to main.dart

**How to verify**:
```bash
flutter analyze
```

Should show no errors related to shorebird_code_push.

**What can go wrong**:
- Import error → Check package name: `shorebird_code_push` (not `shorebird_updater`)
- "UpdateStatus not found" → Make sure import is correct
- "ShorebirdUpdater not found" → Run `flutter pub get` again
- "WidgetsFlutterBinding not found" → Add: `import 'package:flutter/widgets.dart';`

**If stuck**:
1. Delete import line
2. Type `ShorebirdUpdater()` in code
3. Let IDE auto-import (should suggest correct package)
4. Check example: https://pub.dev/packages/shorebird_code_push/example

---

## Step 4: Build and Test

**Goal**: Build app with Shorebird and test update checking

**Input**: Flutter project with changes from Steps 1-3

**Tool/Command**: `shorebird release`

**Detailed Actions**:

1. Build release version with Shorebird:

**For Android**:
```bash
shorebird release android
```

**For iOS**:
```bash
shorebird release ios
```

2. Wait for build to complete (2-5 minutes)

**Output**:
```
Building with Shorebird...
✓ Building Android release...
✓ Uploading to Shorebird...
✓ Release 1.0.0+1 created successfully!

Release version: 1.0.0+1
Release ID: abc-123-def
```

3. Install app on device/emulator

4. Check logs to see update check:

**Android**:
```bash
adb logcat | grep Shorebird
```

**iOS**:
```bash
# In Xcode console
```

**Expected log output**:
```
[Shorebird] Checking for updates...
[Shorebird] App is up to date!
```

**How to verify**:
- App launches successfully
- Logs show "[Shorebird] Checking for updates..."
- No errors in logs
- App connects to your Cloudflare Worker (check Worker logs in Cloudflare dashboard)

**What can go wrong**:
- "Updater not available" in logs → App built with `flutter build` instead of `shorebird release`
- "Connection refused" → Worker URL wrong in shorebird.yaml
- "404 Not Found" → Worker not deployed or wrong endpoint
- No logs → App in debug mode (updater only works in release mode)

**If stuck**:
1. Verify shorebird.yaml has correct base_url
2. Test Worker URL in browser
3. Rebuild with `shorebird release`
4. Check Cloudflare Worker logs for incoming requests

---

## Step 5: Test Update Flow (End-to-End)

**Goal**: Verify complete update workflow

**Input**: App installed on device, patch uploaded to Cloudflare

**Tool/Command**: Device/emulator + CLI tool from Step 2

**Detailed Actions**:

1. Make a code change in your Flutter app (e.g., change a text string)

2. Create patch:
```bash
shorebird patch android
```

3. Download patch from Shorebird dashboard (manual step for now)

4. Upload to your Cloudflare using CLI tool:
```bash
shorebird_custom_server upload \
  --app-id com.yourapp.name \
  --version 1.0.0+1 \
  --platform android \
  --patch-file /path/to/downloaded/patch
```

5. Restart your app

6. Check logs:

**Expected**:
```
[Shorebird] Checking for updates...
[Shorebird] Update available! Downloading...
[Shorebird] Update downloaded! Restart app to apply.
```

7. Restart app again

8. Verify your code change is visible

**Output**: App shows updated code after restart

**How to verify**:
- First restart: Logs show "Update available"
- Patch downloads successfully
- Second restart: App shows new code
- Cloudflare R2 shows patch file
- Cloudflare D1 shows patch metadata

**What can go wrong**:
- "No update available" → Patch not uploaded to Cloudflare
- "Download failed" → R2 URL wrong or file missing
- "Hash mismatch" → Patch file corrupted during upload
- Code doesn't change → Patch not applied, check Shorebird logs

**If stuck**:
1. Check Cloudflare D1 database for patch entry
2. Check Cloudflare R2 for patch file
3. Verify app_id, version, platform match exactly
4. Check Worker logs for errors
5. Try `shorebird_custom_server check` to see what backend returns

---

## API Reference

### ShorebirdUpdater Methods

```dart
final updater = ShorebirdUpdater();

// Check if updater is available
bool isAvailable = updater.isAvailable;

// Get current patch number
Patch? currentPatch = await updater.readCurrentPatch();
print('Current patch: ${currentPatch?.number}');

// Get next patch number (if downloaded)
Patch? nextPatch = await updater.readNextPatch();

// Check for updates
UpdateStatus status = await updater.checkForUpdate();

// Download and install update
await updater.update();
```

### UpdateStatus Values

- `UpdateStatus.upToDate` - No update available
- `UpdateStatus.outdated` - Update available for download
- `UpdateStatus.restartRequired` - Update downloaded, restart needed
- `UpdateStatus.unavailable` - Updater not available (debug mode)

---

## Configuration Reference

### shorebird.yaml

```yaml
# Required: Your app's unique identifier
app_id: abc-123-def-456

# Optional: Custom update server URL
# Default: https://api.shorebird.dev
base_url: https://your-worker.workers.dev

# Optional: Update channel
# Default: stable
channel: stable

# Optional: Auto-update on app start
# Default: true
auto_update: false

# Optional: Patch verification mode
# Default: strict (verify at boot time)
# Options: strict, install_only
patch_verification: strict
```

---

## Troubleshooting

### Problem: "Updater not available" in logs

**Cause**: App not built with Shorebird

**Solution**:
```bash
# Don't use flutter build
flutter build apk  # ❌ Wrong

# Use shorebird release
shorebird release android  # ✅ Correct
```

---

### Problem: "Connection refused" or "404 Not Found"

**Cause**: Wrong base_url in shorebird.yaml

**Solution**:
1. Check Worker URL in Cloudflare dashboard
2. Test URL in browser: `https://your-worker.workers.dev/api/v1/patches/check`
3. Should return JSON (even if error)
4. Update shorebird.yaml
5. Rebuild with `shorebird release`

---

### Problem: "No update available" but patch exists

**Cause**: Mismatch in app_id, version, or platform

**Solution**:
1. Check app_id in shorebird.yaml matches D1 database
2. Check version matches exactly (including build number)
3. Check platform (android/ios)
4. Query D1 directly:
```sql
SELECT * FROM patches 
WHERE app_id = 'com.yourapp.name' 
AND release_version = '1.0.0+1' 
AND platform = 'android';
```

---

### Problem: "Hash mismatch" error

**Cause**: Patch file corrupted or wrong file uploaded

**Solution**:
1. Re-download patch from Shorebird
2. Verify SHA256 hash:
```bash
shasum -a 256 patch_file
```
3. Re-upload to Cloudflare
4. Make sure hash in D1 matches file hash

---

## Summary

**What you did**:
1. ✅ Added `shorebird_code_push` package
2. ✅ Configured `shorebird.yaml` with custom base_url
3. ✅ Added update check code to main.dart
4. ✅ Built with `shorebird release`
5. ✅ Tested update flow

**Time spent**: ~10 minutes

**Next steps**:
- Deploy app to production
- Create patches with `shorebird patch`
- Upload patches to Cloudflare
- Monitor updates in Cloudflare dashboard

---

## Key Takeaways

1. **Flutter package, not Rust binary** - No compilation needed
2. **shorebird.yaml controls server** - Just change base_url
3. **Only works in release mode** - Must use `shorebird release`
4. **Restart required** - Patches apply on next app start
5. **SHA256 verified** - Patches are cryptographically verified

---

## Next: Deploy Patches (Step 4)

Now that your app can check for updates, learn how to create and deploy patches in [04-DEPLOYMENT.md](04-DEPLOYMENT.md).
