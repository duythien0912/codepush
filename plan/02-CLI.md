# CLI Implementation Plan

## 🎯 Goal

Create a Dart CLI tool that wraps `shorebird patch` and automatically uploads patches to your backend.

## 📊 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  shorebird_custom_server                    │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Config  │→ │ Shorebird│→ │  Patch   │→ │ Uploader │  │
│  │  Loader  │  │  Runner  │  │  Finder  │  │          │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 📁 File Structure

```
cli/
├── pubspec.yaml              # Dart project config
├── .env.example             # Environment template
├── .gitignore              # Git ignore rules
├── bin/
│   └── shorebird_custom_server.dart  # Main entry point
├── lib/
│   ├── config.dart          # Load .env configuration
│   ├── shorebird_runner.dart # Execute shorebird commands
│   ├── patch_finder.dart    # Find .vmcode files
│   ├── uploader.dart        # Upload to backend API
│   ├── models.dart          # Data models
│   └── utils.dart           # Utility functions
└── test/
    └── cli_test.dart        # Unit tests
```

## 📦 Dependencies

### pubspec.yaml
```yaml
name: shorebird_custom_server
description: CLI tool to upload Shorebird patches to custom server
version: 1.0.0

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  args: ^2.4.2          # Command-line argument parsing
  http: ^1.1.0          # HTTP client for uploads
  crypto: ^3.0.3        # SHA256 hashing
  path: ^1.8.3          # Path manipulation
  yaml: ^3.1.2          # Parse pubspec.yaml
  dotenv: ^4.2.0        # Load .env files

dev_dependencies:
  lints: ^3.0.0
  test: ^1.24.0

executables:
  shorebird_custom_server:
```

## 🔧 Configuration

### .env.example
```env
# Backend API
API_URL=http://localhost:3000
API_KEY=your-secret-api-key

# App Configuration
APP_ID=com.yourapp.name

# Optional: Shorebird CLI path
SHOREBIRD_PATH=shorebird
```

## 🎯 Command Structure

### Basic Usage
```bash
shorebird_custom_server patch <platform> [options]
```

### Examples
```bash
# Patch Android
shorebird_custom_server patch android

# Patch iOS
shorebird_custom_server patch ios

# With custom config
shorebird_custom_server patch android --config=.env.production

# Dry run (don't upload)
shorebird_custom_server patch android --dry-run

# Verbose output
shorebird_custom_server patch android --verbose
```

### Arguments
- `patch`: Command (required)
- `<platform>`: android or ios (required)

### Options
- `--config=<path>`: Custom .env file path
- `--dry-run`: Run without uploading
- `--verbose`: Show detailed output
- `--help`: Show help message

## 🔄 Execution Flow

### Main Flow
```
1. Parse command-line arguments
   ↓
2. Load configuration from .env
   ↓
3. Validate configuration
   ↓
4. Run shorebird patch <platform>
   ↓
5. Wait for shorebird to complete
   ↓
6. Find generated .vmcode file
   ↓
7. Read app version from pubspec.yaml
   ↓
8. Calculate next patch number
   ↓
9. Calculate SHA256 hash
   ↓
10. Upload to backend API
   ↓
11. Display success message
```

### Error Handling
```
At each step:
- Catch errors
- Display user-friendly message
- Exit with appropriate code
- Clean up temp files
```

## 📝 Implementation Details

### 1. Main Entry Point

**File:** `bin/shorebird_custom_server.dart`

**Responsibilities:**
- Parse command-line arguments
- Load configuration
- Orchestrate the flow
- Handle errors
- Display output

**Code chi tiết (Fresher-friendly):**
```dart
void main(List<String> args) async {
  try {
    // Parse args
    final argParser = ArgParser()
      ..addOption('config', defaultsTo: '.env')
      ..addFlag('dry-run', defaultsTo: false)
      ..addFlag('verbose', defaultsTo: false)
      ..addFlag('help', abbr: 'h', negatable: false);
    
    final results = argParser.parse(args);
    
    if (results['help']) {
      printHelp();
      exit(0);
    }
    
    // Validate command
    if (results.rest.length < 2) {
      print('Error: Missing command or platform');
      exit(1);
    }
    
    final command = results.rest[0]; // "patch"
    final platform = results.rest[1]; // "android" or "ios"
    
    // Load config
    final config = Config.load(results['config']);
    
    // Run shorebird
    await ShorebirdRunner.run(command, platform);
    
    // Find patch file
    final patchFile = await PatchFinder.find(platform);
    
    // Get metadata
    final version = await getAppVersion();
    final patchNumber = await getNextPatchNumber(config, platform);
    
    // Upload
    if (!results['dry-run']) {
      await Uploader.upload(
        config: config,
        file: patchFile,
        platform: platform,
        version: version,
        patchNumber: patchNumber,
      );
    }
    
    print('✓ Success!');
    
  } catch (e) {
    print('Error: $e');
    exit(1);
  }
}
```

### 2. Config Loader

**File:** `lib/config.dart`

**Responsibilities:**
- Load .env file
- Validate required fields
- Provide config object

**Required Fields:**
- API_URL
- API_KEY
- APP_ID

**Validation:**
- API_URL must be valid URL
- API_KEY must not be empty
- APP_ID must match pattern

### 3. Shorebird Runner

**File:** `lib/shorebird_runner.dart`

**Responsibilities:**
- Execute shorebird CLI
- Stream output to console
- Wait for completion
- Handle errors

**Process:**
```dart
static Future<void> run(String command, String platform) async {
  print('Running: shorebird $command $platform');
  
  final process = await Process.start(
    'shorebird',
    [command, platform],
  );
  
  // Stream stdout
  process.stdout.transform(utf8.decoder).listen((data) {
    stdout.write(data);
  });
  
  // Stream stderr
  process.stderr.transform(utf8.decoder).listen((data) {
    stderr.write(data);
  });
  
  final exitCode = await process.exitCode;
  
  if (exitCode != 0) {
    throw Exception('Shorebird failed with exit code $exitCode');
  }
}
```

### 4. Patch Finder

**File:** `lib/patch_finder.dart`

**Responsibilities:**
- Search for .vmcode files
- Find the latest one
- Return file path

**Search Locations:**
```
Android:
- build/app/intermediates/shorebird/
- build/shorebird/

iOS:
- build/ios/shorebird/
- build/shorebird/
```

**Algorithm:**
```dart
static Future<File> find(String platform) async {
  final searchPaths = _getSearchPaths(platform);
  
  for (final searchPath in searchPaths) {
    final dir = Directory(searchPath);
    if (!dir.existsSync()) continue;
    
    final files = dir
      .listSync(recursive: true)
      .whereType<File>()
      .where((f) => f.path.endsWith('.vmcode'))
      .toList();
    
    if (files.isNotEmpty) {
      // Return the most recent file
      files.sort((a, b) => 
        b.statSync().modified.compareTo(a.statSync().modified)
      );
      return files.first;
    }
  }
  
  throw Exception('Patch file not found');
}
```

### 5. Uploader

**File:** `lib/uploader.dart`

**Responsibilities:**
- Calculate SHA256 hash
- Prepare multipart request
- Upload to backend
- Handle response

**Upload Process:**
```dart
static Future<UploadResponse> upload({
  required Config config,
  required File file,
  required String platform,
  required String version,
  required int patchNumber,
}) async {
  // Calculate hash
  final bytes = await file.readAsBytes();
  final hash = sha256.convert(bytes).toString();
  
  print('File size: ${bytes.length} bytes');
  print('SHA256: $hash');
  
  // Prepare request
  final uri = Uri.parse('${config.apiUrl}/api/upload');
  final request = http.MultipartRequest('POST', uri);
  
  // Add headers
  request.headers['X-API-Key'] = config.apiKey;
  
  // Add file
  request.files.add(
    http.MultipartFile.fromBytes(
      'file',
      bytes,
      filename: 'patch.vmcode',
    ),
  );
  
  // Add fields
  request.fields['app_id'] = config.appId;
  request.fields['platform'] = platform;
  request.fields['version'] = version;
  request.fields['patch_number'] = patchNumber.toString();
  
  // Send request
  print('Uploading to ${config.apiUrl}...');
  final response = await request.send();
  
  // Handle response
  if (response.statusCode == 200) {
    final body = await response.stream.bytesToString();
    return UploadResponse.fromJson(jsonDecode(body));
  } else {
    final body = await response.stream.bytesToString();
    throw Exception('Upload failed: $body');
  }
}
```

### 6. Models

**File:** `lib/models.dart`

**Config Model:**
```dart
class Config {
  final String apiUrl;
  final String apiKey;
  final String appId;
  
  Config({
    required this.apiUrl,
    required this.apiKey,
    required this.appId,
  });
  
  static Config load(String path) {
    // Load from .env file
    // Validate fields
    // Return Config instance
  }
}
```

**UploadResponse Model:**
```dart
class UploadResponse {
  final bool success;
  final PatchInfo? patch;
  final String? error;
  
  UploadResponse({
    required this.success,
    this.patch,
    this.error,
  });
  
  factory UploadResponse.fromJson(Map<String, dynamic> json) {
    return UploadResponse(
      success: json['success'],
      patch: json['patch'] != null 
        ? PatchInfo.fromJson(json['patch']) 
        : null,
      error: json['error'],
    );
  }
}
```

**PatchInfo Model:**
```dart
class PatchInfo {
  final int id;
  final String appId;
  final String platform;
  final int patchNumber;
  final String version;
  final String downloadUrl;
  final String sha256;
  final int sizeBytes;
  
  PatchInfo({
    required this.id,
    required this.appId,
    required this.platform,
    required this.patchNumber,
    required this.version,
    required this.downloadUrl,
    required this.sha256,
    required this.sizeBytes,
  });
  
  factory PatchInfo.fromJson(Map<String, dynamic> json) {
    // Parse JSON
  }
}
```

## 🔨 Build & Install

### Development
```bash
cd cli
dart pub get
dart run bin/shorebird_custom_server.dart patch android
```

### Compile to Binary
```bash
dart compile exe bin/shorebird_custom_server.dart -o shorebird_custom_server
```

### Install Globally (macOS/Linux)
```bash
sudo mv shorebird_custom_server /usr/local/bin/
sudo chmod +x /usr/local/bin/shorebird_custom_server
```

### Install Globally (Windows)
```bash
move shorebird_custom_server.exe C:\Windows\System32\
```

## 🧪 Testing

### Manual Testing
```bash
# Test in Flutter project
cd your-flutter-project
shorebird_custom_server patch android --dry-run --verbose
```

### Unit Tests
```dart
// test/cli_test.dart

void main() {
  group('Config', () {
    test('loads valid config', () {
      // Test config loading
    });
    
    test('throws on missing fields', () {
      // Test validation
    });
  });
  
  group('PatchFinder', () {
    test('finds vmcode file', () {
      // Test file finding
    });
    
    test('throws when not found', () {
      // Test error handling
    });
  });
}
```

## 📊 Output Examples

### Success Output
```
Running: shorebird patch android
[shorebird output...]
✓ Patch created successfully

Finding patch file...
✓ Found: build/shorebird/patch.vmcode

Reading app version...
✓ Version: 1.0.0

Calculating hash...
✓ SHA256: abc123...

Uploading to https://api.yourapp.com...
✓ Upload complete

Patch Details:
  App ID: com.yourapp.name
  Platform: android
  Version: 1.0.0
  Patch Number: 5
  Download URL: https://cdn.yourapp.com/patches/...
  Size: 1.2 MB

✓ Success! Patch deployed.
```

### Error Output
```
Running: shorebird patch android
Error: Shorebird command failed with exit code 1

Please check:
- Shorebird CLI is installed
- You're in a Flutter project
- Project is initialized with Shorebird
```

## ✅ Success Criteria

- [ ] CLI compiles to binary
- [ ] Runs shorebird commands
- [ ] Finds patch files
- [ ] Calculates SHA256
- [ ] Uploads to backend
- [ ] Handles errors gracefully
- [ ] Shows progress messages
- [ ] Works on macOS/Linux/Windows

## 🔄 Next Steps

After CLI is complete:
1. Test with backend
2. Proceed to `03-CLIENT.md`
