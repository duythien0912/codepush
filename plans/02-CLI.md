# 02 - CLI TOOL IMPLEMENTATION

**Time**: 60 minutes  
**Difficulty**: Medium  
**Platform**: Dart CLI (macOS)

---

## Overview

Build a CLI tool that:
1. Downloads patches from Shorebird cloud (after `shorebird patch` command)
2. Uploads patches to your Cloudflare R2
3. Updates D1 database with metadata

**Result**: Command `shorebird_custom_server upload <patch_file>`

**IMPORTANT**: This is Option C workflow - we don't extract patches directly from Shorebird CLI. Instead, we download from Shorebird cloud and re-upload to Cloudflare.

---

## Prerequisites

✅ Cloudflare backend deployed (Step 1)  
✅ Dart SDK installed (`dart --version`)  
✅ Shorebird CLI installed (`shorebird --version`)  
✅ Flutter project with Shorebird initialized

---

## Workflow Understanding

```
Developer runs:
  shorebird patch android
  → Uploads to Shorebird cloud ✅

Developer manually:
  1. Go to Shorebird dashboard
  2. Download patch file
  3. Run: shorebird_custom_server upload patch_file
  → Uploads to Cloudflare R2 ✅
  → Updates D1 metadata ✅

End user:
  App checks Cloudflare for updates ✅
```

---

## Step 1: Create CLI Project

### 1.1 Create Project Structure

```bash
cd ~/Desktop/codepush
mkdir -p cli/bin
cd cli
```

### 1.2 Create pubspec.yaml

```yaml
name: shorebird_custom_server
description: CLI tool to upload Shorebird patches to Cloudflare
version: 1.0.0

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  args: ^2.4.0
  http: ^1.1.0
  crypto: ^3.0.3
  path: ^1.8.3

executables:
  shorebird_custom_server:
```

### 1.3 Install Dependencies

```bash
dart pub get
```

---

## Step 2: Implement CLI Tool

### 2.1 Create bin/shorebird_custom_server.dart

```dart
import 'dart:io';
import 'dart:convert';
import 'package:args/args.dart';
import 'package:http/http.dart' as http;
import 'package:crypto/crypto.dart';
import 'package:path/path.dart' as path;

void main(List<String> arguments) async {
  final parser = ArgParser()
    ..addCommand('upload')
    ..addCommand('check');

  try {
    final results = parser.parse(arguments);
    
    if (results.command == null) {
      printUsage(parser);
      exit(1);
    }

    final command = results.command!;
    
    if (command.name == 'upload') {
      await handleUpload(command.arguments);
    } else if (command.name == 'check') {
      await handleCheck(command.arguments);
    }
  } catch (e) {
    print('Error: $e');
    exit(1);
  }
}

void printUsage(ArgParser parser) {
  print('''
Shorebird Custom Server CLI

Usage:
  shorebird_custom_server upload <patch_file> [options]
  shorebird_custom_server check [options]

Commands:
  upload    Upload a patch file to Cloudflare
  check     Check for available updates

Upload Options:
  --api-url=<url>           Cloudflare Worker URL (required)
  --api-key=<key>           API key for authentication (required)
  --app-id=<id>             App ID from shorebird.yaml (required)
  --release-version=<ver>   Release version (e.g., 1.0.0+1) (required)
  --platform=<platform>     Platform: android or ios (required)
  --arch=<arch>             Architecture: aarch64, x86_64, etc (required)
  --patch-number=<num>      Patch number (required)

Check Options:
  --api-url=<url>           Cloudflare Worker URL (required)
  --app-id=<id>             App ID (required)
  --release-version=<ver>   Release version (required)
  --platform=<platform>     Platform (required)
  --arch=<arch>             Architecture (required)

Example:
  shorebird_custom_server upload patch.bin \\
    --api-url=https://your-worker.workers.dev \\
    --api-key=your-api-key \\
    --app-id=com.example.app \\
    --release-version=1.0.0+1 \\
    --platform=android \\
    --arch=aarch64 \\
    --patch-number=1
''');
}

Future<void> handleUpload(List<String> args) async {
  final parser = ArgParser()
    ..addOption('api-url', mandatory: true)
    ..addOption('api-key', mandatory: true)
    ..addOption('app-id', mandatory: true)
    ..addOption('release-version', mandatory: true)
    ..addOption('platform', mandatory: true)
    ..addOption('arch', mandatory: true)
    ..addOption('patch-number', mandatory: true);

  final results = parser.parse(args);
  
  if (results.rest.isEmpty) {
    print('Error: Patch file path required');
    exit(1);
  }

  final patchFilePath = results.rest[0];
  final patchFile = File(patchFilePath);

  if (!patchFile.existsSync()) {
    print('Error: Patch file not found: $patchFilePath');
    exit(1);
  }

  print('📦 Reading patch file...');
  final patchBytes = await patchFile.readAsBytes();
  final fileSize = patchBytes.length;
  print('   Size: ${(fileSize / 1024).toStringAsFixed(2)} KB');

  print('🔐 Calculating SHA256...');
  final hash = sha256.convert(patchBytes);
  final hashHex = hash.toString();
  print('   Hash: $hashHex');

  print('☁️  Uploading to Cloudflare...');
  
  final apiUrl = results['api-url'];
  final apiKey = results['api-key'];
  final appId = results['app-id'];
  final releaseVersion = results['release-version'];
  final platform = results['platform'];
  final arch = results['arch'];
  final patchNumber = results['patch-number'];

  final request = http.MultipartRequest(
    'POST',
    Uri.parse('$apiUrl/api/upload'),
  );

  request.headers['X-API-Key'] = apiKey;
  request.files.add(http.MultipartFile.fromBytes(
    'file',
    patchBytes,
    filename: 'patch.bin',
  ));
  request.fields['app_id'] = appId;
  request.fields['release_version'] = releaseVersion;
  request.fields['platform'] = platform;
  request.fields['arch'] = arch;
  request.fields['patch_number'] = patchNumber;

  final response = await request.send();
  final responseBody = await response.stream.bytesToString();

  if (response.statusCode == 200) {
    final json = jsonDecode(responseBody);
    print('✅ Upload successful!');
    print('');
    print('Patch Details:');
    print('  App ID: $appId');
    print('  Version: $releaseVersion');
    print('  Platform: $platform');
    print('  Arch: $arch');
    print('  Patch #: $patchNumber');
    print('  SHA256: $hashHex');
    print('  Download URL: ${json['patch']['download_url']}');
  } else {
    print('❌ Upload failed: ${response.statusCode}');
    print('Response: $responseBody');
    exit(1);
  }
}

Future<void> handleCheck(List<String> args) async {
  final parser = ArgParser()
    ..addOption('api-url', mandatory: true)
    ..addOption('app-id', mandatory: true)
    ..addOption('release-version', mandatory: true)
    ..addOption('platform', mandatory: true)
    ..addOption('arch', mandatory: true);

  final results = parser.parse(args);

  final apiUrl = results['api-url'];
  final appId = results['app-id'];
  final releaseVersion = results['release-version'];
  final platform = results['platform'];
  final arch = results['arch'];

  print('🔍 Checking for updates...');
  print('   App: $appId');
  print('   Version: $releaseVersion');
  print('   Platform: $platform');
  print('   Arch: $arch');

  final response = await http.post(
    Uri.parse('$apiUrl/api/v1/patches/check'),
    headers: {'Content-Type': 'application/json'},
    body: jsonEncode({
      'app_id': appId,
      'release_version': releaseVersion,
      'platform': platform,
      'arch': arch,
      'channel': 'stable',
      'client_id': 'cli-test',
    }),
  );

  if (response.statusCode == 200) {
    final json = jsonDecode(response.body);
    
    if (json['patch_available'] == true) {
      final patch = json['patch'];
      print('✅ Update available!');
      print('');
      print('Patch Details:');
      print('  Patch #: ${patch['number']}');
      print('  SHA256: ${patch['hash']}');
      print('  Download: ${patch['download_url']}');
    } else {
      print('✅ No updates available');
    }
  } else {
    print('❌ Check failed: ${response.statusCode}');
    print('Response: ${response.body}');
    exit(1);
  }
}
```

---

## Step 3: Build CLI Tool

### 3.1 Compile to Native Binary

```bash
dart compile exe bin/shorebird_custom_server.dart -o shorebird_custom_server
```

**Output**:
```
Generated: /Users/neun/Desktop/codepush/cli/shorebird_custom_server
```

### 3.2 Test Binary

```bash
./shorebird_custom_server
```

Should show usage instructions.

### 3.3 Install Globally (Optional)

```bash
sudo cp shorebird_custom_server /usr/local/bin/
sudo chmod +x /usr/local/bin/shorebird_custom_server
```

**Verify**:
```bash
shorebird_custom_server
```

---

## Step 4: Usage Workflow

### 4.1 Create Patch with Shorebird

```bash
cd /path/to/your/flutter/project
shorebird patch android
```

**Output**:
```
✓ Building patch...
✓ Uploading to Shorebird...
✓ Patch 1 created successfully!
```

### 4.2 Download Patch from Shorebird

**Manual Method** (for now):
1. Go to https://console.shorebird.dev
2. Navigate to your app
3. Find the patch you just created
4. Click "Download" button
5. Save as `patch_1.bin`

**TODO**: Automate this with Shorebird API

### 4.3 Upload to Cloudflare

```bash
shorebird_custom_server upload patch_1.bin \
  --api-url=https://your-worker.workers.dev \
  --api-key=your-api-key \
  --app-id=com.example.app \
  --release-version=1.0.0+1 \
  --platform=android \
  --arch=aarch64 \
  --patch-number=1
```

**Output**:
```
📦 Reading patch file...
   Size: 45.23 KB
🔐 Calculating SHA256...
   Hash: abc123def456...
☁️  Uploading to Cloudflare...
✅ Upload successful!

Patch Details:
  App ID: com.example.app
  Version: 1.0.0+1
  Platform: android
  Arch: aarch64
  Patch #: 1
  SHA256: abc123def456...
  Download URL: https://pub-xxx.r2.dev/...
```

### 4.4 Verify Upload

```bash
shorebird_custom_server check \
  --api-url=https://your-worker.workers.dev \
  --app-id=com.example.app \
  --release-version=1.0.0+1 \
  --platform=android \
  --arch=aarch64
```

**Output**:
```
🔍 Checking for updates...
   App: com.example.app
   Version: 1.0.0+1
   Platform: android
   Arch: aarch64
✅ Update available!

Patch Details:
  Patch #: 1
  SHA256: abc123def456...
  Download: https://pub-xxx.r2.dev/...
```

---

## Step 5: Create Config File (Optional)

To avoid typing all parameters every time, create a config file:

### 5.1 Create .shorebird_server.yaml

```yaml
api_url: https://your-worker.workers.dev
api_key: your-api-key
app_id: com.example.app
```

### 5.2 Update CLI to Read Config

Add to CLI code:
```dart
// Read from ~/.shorebird_server.yaml or ./.shorebird_server.yaml
// Merge with command-line arguments
// Command-line args override config file
```

---

## Architecture Notes

### Why This Approach?

1. **Shorebird CLI is closed-source** - Can't extract patches directly
2. **Shorebird uploads to their cloud** - We download from there
3. **Re-upload to Cloudflare** - Gives us control over distribution

### Limitations

1. **Manual download step** - Need to automate with Shorebird API
2. **Requires Shorebird account** - Still depends on their service
3. **Double upload** - Patch goes to Shorebird, then Cloudflare

### Future Improvements

1. **Shorebird API integration** - Auto-download patches
2. **Batch upload** - Upload multiple patches at once
3. **Diff detection** - Only upload if changed
4. **Rollback support** - Mark patches as inactive

---

## Troubleshooting

### "Patch file not found"

Check file path is correct:
```bash
ls -la patch_1.bin
```

### "Invalid API key"

Check API key in Cloudflare Worker:
```bash
# In backend folder
cat wrangler.toml | grep API_KEY
```

### "Upload failed: 400"

Missing required field. Check all parameters:
- app_id
- release_version
- platform
- arch
- patch_number

### "SHA256 mismatch"

Patch file corrupted during download. Re-download from Shorebird.

---

## Summary

**What you built**:
- ✅ CLI tool to upload patches
- ✅ SHA256 hash calculation
- ✅ Cloudflare R2 upload
- ✅ D1 metadata update
- ✅ Check command for verification

**Time spent**: ~60 minutes

**Next**: Integrate client (03-CLIENT-NEW.md)

---

## Key Takeaways

1. **Option C workflow** - Download from Shorebird, upload to Cloudflare
2. **Manual step required** - Download patch from Shorebird dashboard
3. **Architecture support** - Must specify arch (aarch64, x86_64, etc)
4. **SHA256 verification** - Ensures patch integrity
5. **Future automation** - Can integrate Shorebird API to auto-download

---

For complete implementation details and error handling, refer to the source code above.
