# Flutter Client Implementation Plan

## 🎯 Mục tiêu

Sử dụng Shorebird Updater package và chỉ cần **đổi base URL** để trỏ về backend của bạn.

## 🌟 Tại sao đơn giản?

✅ **Không cần fork** - Dùng package có sẵn
✅ **Không cần sửa code** - Chỉ config
✅ **Đã test kỹ** - Official package từ Shorebird
✅ **Chỉ 1 thay đổi** - Đổi base_url

## 📦 Shorebird Updater Package

**Repository:** https://github.com/shorebirdtech/updater

**Đã có sẵn:**
- ✅ Check for updates
- ✅ Download patches
- ✅ Verify SHA256
- ✅ Install patches
- ✅ Restart app
- ✅ Error handling
- ✅ Rollback support

**Chúng ta chỉ cần:** Đổi base URL từ Shorebird API → Backend của bạn

## 🚀 BƯỚC 1: Add Package (2 phút)

### 1.1 Add dependency

**File:** `pubspec.yaml`

```yaml
dependencies:
  flutter:
    sdk: flutter
  shorebird_code_push: ^1.0.0  # Latest version
```

### 1.2 Install

```bash
flutter pub get
```

✅ **Checkpoint:** Package installed

## 🚀 BƯỚC 2: Configure Base URL (3 phút)

### 2.1 Tạo config file

**File:** `lib/shorebird_config.dart`

```dart
class ShorebirdConfig {
  // Đổi URL này thành backend của bạn
  static const String baseUrl = 'https://your-worker.workers.dev';
  
  // Hoặc custom domain
  // static const String baseUrl = 'https://api.yourapp.com';
  
  static const String appId = 'com.yourapp.name';
}
```

✅ **Checkpoint:** Config file created

### 2.2 Giải thích

**Base URL format:**
- Cloudflare Workers: `https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev`
- Custom domain: `https://api.yourapp.com`
- Local test: `http://localhost:8787`

**App ID:**
- Phải match với `app_id` trong backend
- Format: `com.company.appname`
- Ví dụ: `com.example.myapp`

## 🚀 BƯỚC 3: Initialize trong App (5 phút)

### 3.1 Update main.dart

**File:** `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:shorebird_code_push/shorebird_code_push.dart';
import 'shorebird_config.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize Shorebird với custom base URL
  final updater = ShorebirdCodePush(
    // Đổi base URL
    baseUrl: ShorebirdConfig.baseUrl,
  );
  
  // Check for update on app start
  try {
    final updateAvailable = await updater.isNewPatchAvailableForDownload();
    
    if (updateAvailable) {
      print('Update available! Downloading...');
      await updater.downloadUpdateIfAvailable();
      print('Update downloaded! Will apply on next restart.');
    } else {
      print('No update available.');
    }
  } catch (e) {
    print('Update check failed: $e');
  }
  
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'My App',
      home: HomePage(),
    );
  }
}
```

✅ **Checkpoint:** App initialized with custom base URL

### 3.2 Giải thích code

**Line by line:**

```dart
// 1. Import package
import 'package:shorebird_code_push/shorebird_code_push.dart';

// 2. Khởi tạo với custom base URL
final updater = ShorebirdCodePush(
  baseUrl: ShorebirdConfig.baseUrl,  // ← Đây là thay đổi duy nhất!
);

// 3. Check update
final updateAvailable = await updater.isNewPatchAvailableForDownload();

// 4. Download nếu có
if (updateAvailable) {
  await updater.downloadUpdateIfAvailable();
}
```

**Đơn giản vậy thôi!**

## 🚀 BƯỚC 4: Advanced Usage (Optional - 5 phút)

### 4.1 Show progress dialog

```dart
Future<void> checkAndUpdate(BuildContext context) async {
  final updater = ShorebirdCodePush(
    baseUrl: ShorebirdConfig.baseUrl,
  );
  
  // Show loading
  showDialog(
    context: context,
    barrierDismissible: false,
    builder: (context) => AlertDialog(
      content: Row(
        children: [
          CircularProgressIndicator(),
          SizedBox(width: 16),
          Text('Checking for updates...'),
        ],
      ),
    ),
  );
  
  try {
    final updateAvailable = await updater.isNewPatchAvailableForDownload();
    
    Navigator.pop(context); // Close loading
    
    if (updateAvailable) {
      // Show update dialog
      final shouldUpdate = await showDialog<bool>(
        context: context,
        builder: (context) => AlertDialog(
          title: Text('Update Available'),
          content: Text('A new version is available. Update now?'),
          actions: [
            TextButton(
              onPressed: () => Navigator.pop(context, false),
              child: Text('Later'),
            ),
            TextButton(
              onPressed: () => Navigator.pop(context, true),
              child: Text('Update'),
            ),
          ],
        ),
      );
      
      if (shouldUpdate == true) {
        // Download update
        await updater.downloadUpdateIfAvailable();
        
        // Show restart dialog
        showDialog(
          context: context,
          builder: (context) => AlertDialog(
            title: Text('Update Downloaded'),
            content: Text('Please restart the app to apply the update.'),
            actions: [
              TextButton(
                onPressed: () {
                  Navigator.pop(context);
                  // Restart app (platform specific)
                },
                child: Text('Restart'),
              ),
            ],
          ),
        );
      }
    } else {
      // No update
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('You are on the latest version')),
      );
    }
  } catch (e) {
    Navigator.pop(context); // Close loading if error
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Update check failed: $e')),
    );
  }
}
```

### 4.2 Check update on button press

```dart
class HomePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('My App')),
      body: Center(
        child: ElevatedButton(
          onPressed: () => checkAndUpdate(context),
          child: Text('Check for Updates'),
        ),
      ),
    );
  }
}
```

## 🧪 BƯỚC 5: Test (5 phút)

### 5.1 Test local

```bash
# Start backend local
cd backend
npm run dev

# Update config
# lib/shorebird_config.dart
static const String baseUrl = 'http://localhost:8787';

# Run app
flutter run
```

**Expected:**
- App starts
- Checks for update
- Shows "No update available" (nếu chưa có patch)

✅ **Checkpoint:** App connects to local backend

### 5.2 Test with real patch

**Bước 1: Deploy một patch**
```bash
# Make code change
# Change some text in app

# Deploy patch
shorebird_custom_server patch android
```

**Bước 2: Test update**
```bash
# Close app completely
# Reopen app
flutter run
```

**Expected:**
- App starts
- Detects update
- Downloads patch
- Shows "Update downloaded"
- Restart app
- New code running ✅

✅ **Checkpoint:** Update flow works!

## 📊 API Endpoints Used

Shorebird Updater package sẽ gọi các endpoints này:

### 1. Check for update
```
GET {baseUrl}/api/check?app_id=xxx&version=1.0.0&platform=android&patch=0
```

**Response:**
```json
{
  "has_update": true,
  "patch": {
    "patch_number": 5,
    "download_url": "https://cdn.../patch.bin",
    "sha256": "abc123...",
    "size_bytes": 1024000
  }
}
```

### 2. Download patch
```
GET {download_url}
```

**Response:** Binary patch file

## 🔧 Configuration Options

### Minimal (Chỉ cần base URL)

```dart
final updater = ShorebirdCodePush(
  baseUrl: 'https://your-worker.workers.dev',
);
```

### Full options (Optional)

```dart
final updater = ShorebirdCodePush(
  baseUrl: 'https://your-worker.workers.dev',
  // Các options khác (nếu cần)
);
```

**Lưu ý:** Hầu hết trường hợp chỉ cần `baseUrl`!

## 🚨 Troubleshooting

### Issue 1: "No update available" nhưng đã deploy patch

**Check:**
```bash
# Test API trực tiếp
curl "https://your-worker.workers.dev/api/check?app_id=com.yourapp.name&version=1.0.0&platform=android&patch=0"
```

**Solution:**
- Check `app_id` đúng chưa
- Check `version` đúng chưa
- Check backend có patch chưa

### Issue 2: "Download failed"

**Solution:**
- Check internet connection
- Check download URL accessible
- Check R2 public access enabled

### Issue 3: "Hash mismatch"

**Solution:**
- File bị corrupt khi upload
- Upload lại patch
- Check SHA256 trong database

### Issue 4: App không restart sau update

**Solution:**
- Shorebird engine sẽ apply patch on next cold start
- User cần close app hoàn toàn
- Hoặc implement force restart

## ✅ Success Criteria

- [ ] Package installed
- [ ] Config file created với base URL
- [ ] App initialized với custom base URL
- [ ] Test local successful
- [ ] Test with real patch successful
- [ ] Update flow works end-to-end
- [ ] Error handling works

## 📝 Summary

**Những gì cần làm:**
1. ✅ Add package: `shorebird_code_push`
2. ✅ Tạo config với base URL
3. ✅ Initialize trong main.dart
4. ✅ Test

**Những gì KHÔNG cần làm:**
- ❌ Fork repository
- ❌ Sửa code package
- ❌ Implement update logic
- ❌ Implement download logic
- ❌ Implement install logic

**Tất cả đã có sẵn trong package!**

## 🎉 Done!

Client của bạn giờ đã:
- ✅ Tự động check update
- ✅ Tự động download patch
- ✅ Tự động verify hash
- ✅ Tự động install patch
- ✅ Kết nối với backend của bạn

**Thời gian:** ~15 phút (thay vì 20 phút)

## 🔄 Next Steps

1. Test trong production app
2. Add UI cho update flow (optional)
3. Add analytics (optional)
4. Customize error messages (optional)

**Hoàn thành! 🚀**
