# Flutter Client Implementation Plan

## 🎯 Goal

Create a Flutter package that automatically checks for updates and applies patches using Shorebird engine.

## 📊 Base

Fork from: https://github.com/shorebirdtech/updater

**Why fork?**
- Already has Shorebird engine integration
- Handles platform-specific patch installation
- Proven architecture
- Just need to change API endpoints

## 📁 File Structure

```
client/
├── pubspec.yaml
├── README.md
├── CHANGELOG.md
├── LICENSE
├── lib/
│   ├── shorebird_updater.dart       # Main public API
│   ├── src/
│   │   ├── config.dart              # Configuration
│   │   ├── update_checker.dart      # Check for updates
│   │   ├── patch_downloader.dart    # Download patches
│   │   ├── patch_installer.dart     # Install patches
│   │   ├── storage.dart             # Local storage
│   │   └── models.dart              # Data models
│   └── shorebird_updater_platform_interface.dart
├── example/
│   ├── lib/
│   │   └── main.dart                # Example app
│   └── pubspec.yaml
└── test/
    └── shorebird_updater_test.dart
```

## 📦 Dependencies

### pubspec.yaml
```yaml
name: shorebird_updater
description: Auto-update Flutter apps using custom Shorebird server
version: 1.0.0

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.24.0'

dependencies:
  flutter:
    sdk: flutter
  http: ^1.1.0              # HTTP client
  crypto: ^3.0.3            # SHA256 verification
  path_provider: ^2.1.1     # Temp directory
  shared_preferences: ^2.2.2 # Store patch number
  package_info_plus: ^5.0.1  # Get app version

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
```

## 🎯 Public API

### Basic Usage

```dart
import 'package:shorebird_updater/shorebird_updater.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  final updater = ShorebirdUpdater(
    apiUrl: 'https://api.yourapp.com',
    appId: 'com.yourapp.name',
  );
  
  // Check and update on app start
  await updater.checkAndUpdate();
  
  runApp(MyApp());
}
```

### Advanced Usage

```dart
final updater = ShorebirdUpdater(
  apiUrl: 'https://api.yourapp.com',
  appId: 'com.yourapp.name',
  autoCheck: true,              // Auto check on init
  showProgress: true,           // Show progress dialog
  onUpdateAvailable: (info) {
    print('Update available: ${info.patchNumber}');
  },
  onUpdateDownloading: (progress) {
    print('Downloading: ${progress}%');
  },
  onUpdateInstalled: () {
    print('Update installed, restarting...');
  },
  onError: (error) {
    print('Update error: $error');
  },
);

// Manual check
final hasUpdate = await updater.checkForUpdate();
if (hasUpdate) {
  await updater.downloadAndInstall();
}

// Get current patch info
final currentPatch = await updater.getCurrentPatchNumber();
print('Current patch: $currentPatch');
```

## 🔄 Update Flow

### Complete Flow
```
1. App starts
   ↓
2. Initialize ShorebirdUpdater
   ↓
3. Get current version from package_info
   ↓
4. Get current patch number from local storage
   ↓
5. Call API: GET /api/check
   ↓
6. If no update → Continue app launch
   ↓
7. If update available → Download patch
   ↓
8. Verify SHA256 hash
   ↓
9. Save to temp directory
   ↓
10. Call Shorebird engine to install
   ↓
11. Update local patch number
   ↓
12. Restart app (or prompt user)
```

## 📝 Implementation Details

### 1. Main Class

**File:** `lib/shorebird_updater.dart`

```dart
class ShorebirdUpdater {
  final String apiUrl;
  final String appId;
  final bool autoCheck;
  final bool showProgress;
  final Function(UpdateInfo)? onUpdateAvailable;
  final Function(double)? onUpdateDownloading;
  final Function()? onUpdateInstalled;
  final Function(dynamic)? onError;
  
  ShorebirdUpdater({
    required this.apiUrl,
    required this.appId,
    this.autoCheck = false,
    this.showProgress = false,
    this.onUpdateAvailable,
    this.onUpdateDownloading,
    this.onUpdateInstalled,
    this.onError,
  }) {
    if (autoCheck) {
      checkAndUpdate();
    }
  }
  
  Future<bool> checkForUpdate() async {
    try {
      final checker = UpdateChecker(apiUrl: apiUrl, appId: appId);
      final updateInfo = await checker.check();
      
      if (updateInfo != null) {
        onUpdateAvailable?.call(updateInfo);
        return true;
      }
      
      return false;
    } catch (e) {
      onError?.call(e);
      return false;
    }
  }
  
  Future<void> downloadAndInstall() async {
    // Implementation
  }
  
  Future<void> checkAndUpdate() async {
    final hasUpdate = await checkForUpdate();
    if (hasUpdate) {
      await downloadAndInstall();
    }
  }
  
  Future<int> getCurrentPatchNumber() async {
    final storage = Storage();
    return await storage.getPatchNumber();
  }
}
```

### 2. Update Checker

**File:** `lib/src/update_checker.dart`

**Responsibilities:**
- Get current app version
- Get current patch number
- Call backend API
- Parse response

```dart
class UpdateChecker {
  final String apiUrl;
  final String appId;
  
  UpdateChecker({
    required this.apiUrl,
    required this.appId,
  });
  
  Future<UpdateInfo?> check() async {
    // Get current version
    final packageInfo = await PackageInfo.fromPlatform();
    final version = packageInfo.version;
    
    // Get current patch number
    final storage = Storage();
    final currentPatch = await storage.getPatchNumber();
    
    // Get platform
    final platform = Platform.isAndroid ? 'android' : 'ios';
    
    // Call API
    final url = Uri.parse('$apiUrl/api/check').replace(
      queryParameters: {
        'app_id': appId,
        'version': version,
        'platform': platform,
        'patch': currentPatch.toString(),
      },
    );
    
    final response = await http.get(url);
    
    if (response.statusCode != 200) {
      throw Exception('Check failed: ${response.statusCode}');
    }
    
    final data = jsonDecode(response.body);
    
    if (data['has_update'] == true) {
      return UpdateInfo.fromJson(data['patch']);
    }
    
    return null;
  }
}
```

### 3. Patch Downloader

**File:** `lib/src/patch_downloader.dart`

**Responsibilities:**
- Download patch from CDN
- Show progress
- Verify SHA256
- Save to temp directory

```dart
class PatchDownloader {
  final Function(double)? onProgress;
  
  PatchDownloader({this.onProgress});
  
  Future<File> download(UpdateInfo info) async {
    // Get temp directory
    final tempDir = await getTemporaryDirectory();
    final filePath = '${tempDir.path}/patch_${info.patchNumber}.bin';
    final file = File(filePath);
    
    // Download with progress
    final request = http.Request('GET', Uri.parse(info.downloadUrl));
    final response = await request.send();
    
    if (response.statusCode != 200) {
      throw Exception('Download failed: ${response.statusCode}');
    }
    
    final contentLength = response.contentLength ?? 0;
    var downloadedBytes = 0;
    
    final sink = file.openWrite();
    
    await for (final chunk in response.stream) {
      sink.add(chunk);
      downloadedBytes += chunk.length;
      
      if (contentLength > 0) {
        final progress = downloadedBytes / contentLength;
        onProgress?.call(progress);
      }
    }
    
    await sink.close();
    
    // Verify hash
    final bytes = await file.readAsBytes();
    final hash = sha256.convert(bytes).toString();
    
    if (hash != info.sha256) {
      await file.delete();
      throw Exception('Hash mismatch: expected ${info.sha256}, got $hash');
    }
    
    return file;
  }
}
```

### 4. Patch Installer

**File:** `lib/src/patch_installer.dart`

**Responsibilities:**
- Call Shorebird engine
- Install patch
- Update local patch number
- Handle errors

```dart
class PatchInstaller {
  Future<bool> install(File patchFile, int patchNumber) async {
    try {
      // Call Shorebird native method
      // This uses the existing shorebird/updater implementation
      final success = await _installPatch(patchFile.path);
      
      if (success) {
        // Update local patch number
        final storage = Storage();
        await storage.setPatchNumber(patchNumber);
        return true;
      }
      
      return false;
    } catch (e) {
      print('Install failed: $e');
      return false;
    }
  }
  
  Future<bool> _installPatch(String path) async {
    // Platform-specific implementation
    // Android: Use Shorebird AOT
    // iOS: Use Shorebird interpreter
    // This is already implemented in shorebird/updater
  }
}
```

### 5. Storage

**File:** `lib/src/storage.dart`

**Responsibilities:**
- Store current patch number
- Retrieve patch number

```dart
class Storage {
  static const _keyPatchNumber = 'shorebird_patch_number';
  
  Future<int> getPatchNumber() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getInt(_keyPatchNumber) ?? 0;
  }
  
  Future<void> setPatchNumber(int number) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setInt(_keyPatchNumber, number);
  }
  
  Future<void> clear() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove(_keyPatchNumber);
  }
}
```

### 6. Models

**File:** `lib/src/models.dart`

```dart
class UpdateInfo {
  final int patchNumber;
  final String downloadUrl;
  final String sha256;
  final int sizeBytes;
  
  UpdateInfo({
    required this.patchNumber,
    required this.downloadUrl,
    required this.sha256,
    required this.sizeBytes,
  });
  
  factory UpdateInfo.fromJson(Map<String, dynamic> json) {
    return UpdateInfo(
      patchNumber: json['patch_number'],
      downloadUrl: json['download_url'],
      sha256: json['sha256'],
      sizeBytes: json['size_bytes'],
    );
  }
}
```

## 🎨 UI Components (Optional)

### Progress Dialog

```dart
class UpdateProgressDialog extends StatelessWidget {
  final double progress;
  
  const UpdateProgressDialog({required this.progress});
  
  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: Text('Updating...'),
      content: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          LinearProgressIndicator(value: progress),
          SizedBox(height: 16),
          Text('${(progress * 100).toInt()}%'),
        ],
      ),
    );
  }
}
```

## 📱 Example App

**File:** `example/lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:shorebird_updater/shorebird_updater.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  final updater = ShorebirdUpdater(
    apiUrl: 'https://api.yourapp.com',
    appId: 'com.example.app',
    onUpdateAvailable: (info) {
      print('Update available: patch ${info.patchNumber}');
    },
    onUpdateDownloading: (progress) {
      print('Downloading: ${(progress * 100).toInt()}%');
    },
    onUpdateInstalled: () {
      print('Update installed!');
    },
    onError: (error) {
      print('Error: $error');
    },
  );
  
  // Check on start
  await updater.checkAndUpdate();
  
  runApp(MyApp(updater: updater));
}

class MyApp extends StatelessWidget {
  final ShorebirdUpdater updater;
  
  const MyApp({required this.updater});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: HomePage(updater: updater),
    );
  }
}

class HomePage extends StatefulWidget {
  final ShorebirdUpdater updater;
  
  const HomePage({required this.updater});
  
  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  int? currentPatch;
  
  @override
  void initState() {
    super.initState();
    _loadPatchNumber();
  }
  
  Future<void> _loadPatchNumber() async {
    final patch = await widget.updater.getCurrentPatchNumber();
    setState(() => currentPatch = patch);
  }
  
  Future<void> _checkForUpdate() async {
    final hasUpdate = await widget.updater.checkForUpdate();
    
    if (!hasUpdate) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('No updates available')),
      );
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Shorebird Updater Example')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('Current Patch: ${currentPatch ?? 0}'),
            SizedBox(height: 20),
            ElevatedButton(
              onPressed: _checkForUpdate,
              child: Text('Check for Update'),
            ),
          ],
        ),
      ),
    );
  }
}
```

## 🧪 Testing

### Unit Tests

```dart
void main() {
  group('UpdateChecker', () {
    test('returns null when no update', () async {
      // Mock API response
      // Test no update scenario
    });
    
    test('returns UpdateInfo when update available', () async {
      // Mock API response
      // Test update available scenario
    });
  });
  
  group('Storage', () {
    test('stores and retrieves patch number', () async {
      final storage = Storage();
      await storage.setPatchNumber(5);
      final number = await storage.getPatchNumber();
      expect(number, 5);
    });
  });
}
```

## 📋 Modifications from Original

### Changes needed from shorebird/updater:

1. **API Endpoint**
   - Change from Shorebird API to custom API
   - Update URL structure
   - Update query parameters

2. **Authentication**
   - Remove Shorebird auth
   - Add custom API key (optional)

3. **Response Format**
   - Parse custom API response
   - Map to UpdateInfo model

4. **Configuration**
   - Add apiUrl parameter
   - Add appId parameter
   - Remove Shorebird-specific config

## ✅ Success Criteria

- [ ] Package compiles without errors
- [ ] Checks for updates on app start
- [ ] Downloads patches from CDN
- [ ] Verifies SHA256 hash
- [ ] Installs patches successfully
- [ ] Updates local patch number
- [ ] Handles errors gracefully
- [ ] Works on Android and iOS
- [ ] Example app demonstrates usage

## 🔄 Next Steps

After client is complete:
1. Test in real Flutter app
2. Test end-to-end flow
3. Document usage
4. Publish package (optional)
