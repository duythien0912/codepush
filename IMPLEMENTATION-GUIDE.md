# Self-Hosted Code Push Implementation Guide

**Complete guide for building a self-hosted code push system using Shorebird + Cloudflare**

**Time**: ~2.5 hours  
**Cost**: $0 (100% FREE on Cloudflare free tier)  
**Difficulty**: Beginner-friendly

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Glossary](#glossary)
4. [Phase 1: Backend (45 min)](#phase-1-backend)
5. [Phase 2: CLI Tool (60 min)](#phase-2-cli-tool)
6. [Phase 3: Client Integration (10 min)](#phase-3-client-integration)
7. [Phase 4: Testing (20 min)](#phase-4-testing)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Accounts
- ✅ Cloudflare account (free): https://dash.cloudflare.com/sign-up
- ✅ Shorebird account (free): https://console.shorebird.dev

### Required Software
- ✅ Node.js 18+ (`node --version`)
- ✅ Dart SDK 3.0+ (`dart --version`)
- ✅ Flutter SDK 3.10+ (`flutter --version`)
- ✅ Shorebird CLI (`shorebird --version`)

### Required Knowledge
- Basic command-line usage
- Basic Flutter development
- Basic understanding of APIs

### Time Required
- Backend: 45 minutes
- CLI: 60 minutes
- Client: 10 minutes
- Testing: 20 minutes
- **Total**: ~2.5 hours

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│           DEVELOPER                     │
│                                         │
│  1. shorebird patch android             │
│     → Uploads to Shorebird cloud        │
│                                         │
│  2. Download patch from Shorebird       │
│     → TODO: Manual download step        │
│                                         │
│  3. shorebird_custom_server upload      │
│     → Uploads to Cloudflare R2          │
│     → Updates D1 metadata               │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         CLOUDFLARE (100% Serverless)    │
│                                         │
│  Workers: Serverless API                │
│    POST /api/v1/patches/check           │
│    POST /api/upload                     │
│                                         │
│  D1: SQLite database                    │
│    Stores patch metadata                │
│                                         │
│  R2: Object storage                     │
│    Stores patch files                   │
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

### Key Components

1. **Cloudflare Workers**: Serverless API endpoints
2. **D1 Database**: SQLite database for patch metadata
3. **R2 Storage**: Object storage for patch files (like AWS S3)
4. **CLI Tool**: Command-line tool to upload patches
5. **Flutter App**: Your app with Shorebird integration

---

## Glossary

**Essential Terms**:

- **API**: Application Programming Interface - how software talks to each other
- **Cloudflare Workers**: Serverless functions that run on Cloudflare's edge network
- **D1**: Cloudflare's SQLite database service
- **R2**: Cloudflare's object storage (like AWS S3, but free egress)
- **Patch**: A small update file that modifies your app without full reinstall
- **SHA256**: A cryptographic hash used to verify file integrity
- **Architecture (arch)**: CPU type (aarch64 = ARM 64-bit, x86_64 = Intel 64-bit)
- **Release version**: Your app version (e.g., 1.0.0+1)
- **Patch number**: Sequential number for patches (1, 2, 3...)
- **JSON**: JavaScript Object Notation - a data format
- **Binary**: Compiled machine code (not human-readable)
- **Endpoint**: A specific URL path in an API (e.g., /api/check)
- **CORS**: Cross-Origin Resource Sharing - allows web requests from different domains
- **Multipart form data**: A way to upload files via HTTP
- **pubspec.yaml**: Flutter's configuration file for dependencies
- **shorebird.yaml**: Shorebird's configuration file

---

## Phase 1: Backend

**Time**: 45 minutes  
**Goal**: Deploy serverless API on Cloudflare

### Step 1.1: Install Wrangler CLI

Wrangler is Cloudflare's command-line tool.

```bash
npm install -g wrangler
```

**Verify**:
```bash
wrangler --version
# Should show: ⛅️ wrangler 3.x.x
```

**If error "npm: command not found"**: Install Node.js from https://nodejs.org

### Step 1.2: Login to Cloudflare

```bash
wrangler login
```

- Browser opens automatically
- Click "Allow" button
- Return to terminal

**Verify**:
```bash
wrangler whoami
# Should show your email
```

### Step 1.3: Create Project

```bash
mkdir -p ~/Desktop/codepush/backend
cd ~/Desktop/codepush/backend
npm init -y
```

### Step 1.4: Create wrangler.toml

Create file `wrangler.toml`:

```toml
name = "shorebird-backend"
main = "src/index.js"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "shorebird-db"
database_id = "PLACEHOLDER"

[[r2_buckets]]
binding = "R2"
bucket_name = "shorebird-patches"

[vars]
API_KEY = "CHANGE-THIS-TO-RANDOM-STRING"
R2_PUBLIC_URL = "PLACEHOLDER"
```

### Step 1.5: Create D1 Database

```bash
wrangler d1 create shorebird-db
```

**Output**:
```
✅ Successfully created DB 'shorebird-db'

[[d1_databases]]
binding = "DB"
database_name = "shorebird-db"
database_id = "abc-123-def-456"
```

**Copy the `database_id`** and update `wrangler.toml`:

```toml
database_id = "abc-123-def-456"  # Replace PLACEHOLDER
```

### Step 1.6: Create Database Schema

Create file `schema.sql`:

```sql
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

CREATE INDEX idx_patches_lookup 
ON patches(app_id, release_version, platform, arch, active, patch_number);
```

Execute schema:

```bash
wrangler d1 execute shorebird-db --file=schema.sql
```

**Verify**:
```bash
wrangler d1 execute shorebird-db --command="SELECT name FROM sqlite_master WHERE type='table'"
# Should show: patches
```

### Step 1.7: Create R2 Bucket

```bash
wrangler r2 bucket create shorebird-patches
```

**Verify**:
```bash
wrangler r2 bucket list
# Should show: shorebird-patches
```

### Step 1.8: Enable R2 Public Access

1. Go to https://dash.cloudflare.com
2. Click "R2" in left sidebar
3. Click "shorebird-patches" bucket
4. Click "Settings" tab
5. Scroll to "Public access"
6. Click "Allow Access" button
7. **Copy the Public R2.dev Bucket URL**
   - Example: `https://pub-abc123xyz.r2.dev`

Update `wrangler.toml`:

```toml
R2_PUBLIC_URL = "https://pub-abc123xyz.r2.dev"  # Replace PLACEHOLDER
```

### Step 1.9: Create Worker Code

Create folder and file:

```bash
mkdir src
```

Create file `src/index.js`:

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, X-API-Key',
    };

    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }

    // Shorebird-compatible endpoint
    if (url.pathname === '/api/v1/patches/check' && request.method === 'POST') {
      return handlePatchCheck(request, env, corsHeaders);
    }

    // CLI upload endpoint
    if (url.pathname === '/api/upload' && request.method === 'POST') {
      return handleUpload(request, env, corsHeaders);
    }

    return new Response('Not Found', { status: 404, headers: corsHeaders });
  }
};

async function handlePatchCheck(request, env, corsHeaders) {
  try {
    const body = await request.json();
    const { app_id, release_version, platform, arch } = body;

    if (!app_id || !release_version || !platform || !arch) {
      return jsonResponse({ error: 'Missing required fields' }, 400, corsHeaders);
    }

    const result = await env.DB.prepare(`
      SELECT patch_number, download_url, hash, hash_signature
      FROM patches
      WHERE app_id = ? 
        AND release_version = ? 
        AND platform = ? 
        AND arch = ?
        AND active = 1
      ORDER BY patch_number DESC 
      LIMIT 1
    `).bind(app_id, release_version, platform, arch).first();

    if (!result) {
      return jsonResponse({
        patch_available: false,
        patch: null,
        rolled_back_patch_numbers: []
      }, 200, corsHeaders);
    }

    return jsonResponse({
      patch_available: true,
      patch: {
        number: result.patch_number,
        hash: result.hash,
        download_url: result.download_url,
        hash_signature: result.hash_signature
      },
      rolled_back_patch_numbers: []
    }, 200, corsHeaders);

  } catch (error) {
    return jsonResponse({ error: error.message }, 500, corsHeaders);
  }
}

async function handleUpload(request, env, corsHeaders) {
  try {
    const apiKey = request.headers.get('X-API-Key');
    if (apiKey !== env.API_KEY) {
      return jsonResponse({ error: 'Invalid API key' }, 403, corsHeaders);
    }

    const formData = await request.formData();
    const file = formData.get('file');
    const app_id = formData.get('app_id');
    const release_version = formData.get('release_version');
    const platform = formData.get('platform');
    const arch = formData.get('arch');
    const patch_number = parseInt(formData.get('patch_number'));

    if (!file || !app_id || !release_version || !platform || !arch || !patch_number) {
      return jsonResponse({ error: 'Missing required fields' }, 400, corsHeaders);
    }

    const arrayBuffer = await file.arrayBuffer();
    const hashBuffer = await crypto.subtle.digest('SHA-256', arrayBuffer);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    const hash = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');

    const r2Key = `${app_id}/${platform}/${arch}/${release_version}/${patch_number}`;
    await env.R2.put(r2Key, arrayBuffer);

    const download_url = `${env.R2_PUBLIC_URL}/${r2Key}`;

    await env.DB.prepare(`
      INSERT INTO patches (
        app_id, release_version, patch_number, platform, arch,
        download_url, hash, hash_signature, created_at
      ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
    `).bind(
      app_id, release_version, patch_number, platform, arch,
      download_url, hash, null, Date.now()
    ).run();

    return jsonResponse({
      success: true,
      patch: {
        app_id, release_version, patch_number, platform, arch,
        download_url, hash, size_bytes: arrayBuffer.byteLength
      }
    }, 200, corsHeaders);

  } catch (error) {
    return jsonResponse({ error: error.message }, 500, corsHeaders);
  }
}

function jsonResponse(data, status, corsHeaders) {
  return new Response(JSON.stringify(data), {
    status,
    headers: { ...corsHeaders, 'Content-Type': 'application/json' }
  });
}
```

### Step 1.10: Deploy to Cloudflare

```bash
npx wrangler deploy
```

**Output**:
```
Published shorebird-backend (0.89 sec)
  https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev
```

**Save this URL** - you'll need it for CLI and client setup.

### Step 1.11: Test Backend

```bash
curl -X POST https://YOUR-WORKER-URL/api/v1/patches/check \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "com.test.app",
    "release_version": "1.0.0+1",
    "platform": "android",
    "arch": "aarch64",
    "channel": "stable",
    "client_id": "test"
  }'
```

**Expected**:
```json
{"patch_available":false,"patch":null,"rolled_back_patch_numbers":[]}
```

✅ **Phase 1 Complete!** Backend is live.

---

## Phase 2: CLI Tool

**Time**: 60 minutes  
**Goal**: Build tool to upload patches to Cloudflare

### Step 2.1: Create CLI Project

```bash
cd ~/Desktop/codepush
mkdir -p cli/bin
cd cli
```

### Step 2.2: Create pubspec.yaml

Create file `pubspec.yaml`:

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

Install dependencies:

```bash
dart pub get
```

### Step 2.3: Create CLI Code

Create file `bin/shorebird_custom_server.dart`:

```dart
import 'dart:io';
import 'dart:convert';
import 'package:args/args.dart';
import 'package:http/http.dart' as http;
import 'package:crypto/crypto.dart';

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

Upload Options:
  --api-url=<url>           Cloudflare Worker URL (required)
  --api-key=<key>           API key (required)
  --app-id=<id>             App ID (required)
  --release-version=<ver>   Release version (required)
  --platform=<platform>     android or ios (required)
  --arch=<arch>             aarch64, x86_64, etc (required)
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
    --api-key=your-key \\
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
  print('   Size: ${(patchBytes.length / 1024).toStringAsFixed(2)} KB');

  print('🔐 Calculating SHA256...');
  final hash = sha256.convert(patchBytes);
  print('   Hash: $hash');

  print('☁️  Uploading to Cloudflare...');
  
  final request = http.MultipartRequest(
    'POST',
    Uri.parse('${results['api-url']}/api/upload'),
  );

  request.headers['X-API-Key'] = results['api-key']!;
  request.files.add(http.MultipartFile.fromBytes(
    'file',
    patchBytes,
    filename: 'patch.bin',
  ));
  request.fields['app_id'] = results['app-id']!;
  request.fields['release_version'] = results['release-version']!;
  request.fields['platform'] = results['platform']!;
  request.fields['arch'] = results['arch']!;
  request.fields['patch_number'] = results['patch-number']!;

  final response = await request.send();
  final responseBody = await response.stream.bytesToString();

  if (response.statusCode == 200) {
    final json = jsonDecode(responseBody);
    print('✅ Upload successful!');
    print('');
    print('Patch Details:');
    print('  App ID: ${results['app-id']}');
    print('  Version: ${results['release-version']}');
    print('  Platform: ${results['platform']}');
    print('  Arch: ${results['arch']}');
    print('  Patch #: ${results['patch-number']}');
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

  print('🔍 Checking for updates...');

  final response = await http.post(
    Uri.parse('${results['api-url']}/api/v1/patches/check'),
    headers: {'Content-Type': 'application/json'},
    body: jsonEncode({
      'app_id': results['app-id'],
      'release_version': results['release-version'],
      'platform': results['platform'],
      'arch': results['arch'],
      'channel': 'stable',
      'client_id': 'cli-test',
    }),
  );

  if (response.statusCode == 200) {
    final json = jsonDecode(response.body);
    
    if (json['patch_available'] == true) {
      final patch = json['patch'];
      print('✅ Update available!');
      print('  Patch #: ${patch['number']}');
      print('  SHA256: ${patch['hash']}');
      print('  Download: ${patch['download_url']}');
    } else {
      print('✅ No updates available');
    }
  } else {
    print('❌ Check failed: ${response.statusCode}');
    exit(1);
  }
}
```

### Step 2.4: Build CLI

```bash
dart compile exe bin/shorebird_custom_server.dart -o shorebird_custom_server
```

**Verify**:
```bash
./shorebird_custom_server
# Should show usage instructions
```

### Step 2.5: Install Globally (Optional)

```bash
sudo cp shorebird_custom_server /usr/local/bin/
sudo chmod +x /usr/local/bin/shorebird_custom_server
```

**Verify**:
```bash
shorebird_custom_server
# Should show usage from anywhere
```

✅ **Phase 2 Complete!** CLI tool is ready.

---

## Phase 3: Client Integration

**Time**: 10 minutes  
**Goal**: Configure Flutter app to use your backend

### Step 3.1: Add Package

In your Flutter project, open `pubspec.yaml` and add:

```yaml
dependencies:
  flutter:
    sdk: flutter
  shorebird_code_push: ^1.0.0  # Add this
```

Run:
```bash
flutter pub get
```

### Step 3.2: Configure shorebird.yaml

Open `shorebird.yaml` in your Flutter project root and add:

```yaml
app_id: your-app-id  # Already exists
base_url: https://your-worker.workers.dev  # Add this
auto_update: false  # Add this
```

**Replace** `your-worker.workers.dev` with your actual Worker URL from Phase 1.

### Step 3.3: Add Update Check Code

Open `lib/main.dart` and add:

```dart
import 'package:shorebird_code_push/shorebird_code_push.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  final updater = ShorebirdUpdater();
  
  if (updater.isAvailable) {
    print('[Shorebird] Checking for updates...');
    
    try {
      final status = await updater.checkForUpdate();
      
      if (status == UpdateStatus.outdated) {
        print('[Shorebird] Update available! Downloading...');
        await updater.update();
        print('[Shorebird] Update downloaded! Restart to apply.');
      } else if (status == UpdateStatus.upToDate) {
        print('[Shorebird] App is up to date!');
      }
    } catch (e) {
      print('[Shorebird] Update check failed: $e');
    }
  }
  
  runApp(MyApp());
}
```

### Step 3.4: Build with Shorebird

```bash
shorebird release android
```

**Important**: Must use `shorebird release`, not `flutter build`.

✅ **Phase 3 Complete!** App is configured.

---

## Phase 4: Testing

**Time**: 20 minutes  
**Goal**: Test complete workflow

### Step 4.1: Create Patch

Make a code change in your Flutter app, then:

```bash
shorebird patch android
```

**Output**:
```
✓ Building patch...
✓ Uploading to Shorebird...
✓ Patch 1 created successfully!
```

### Step 4.2: Download Patch from Shorebird

**TODO**: Manual download step

1. Go to https://console.shorebird.dev
2. Navigate to your app
3. Find the patch you just created
4. Click "Download" button
5. Save as `patch_1.bin`

**Note**: This step will be automated in future versions.

### Step 4.3: Upload to Cloudflare

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

**Expected**:
```
📦 Reading patch file...
   Size: 45.23 KB
🔐 Calculating SHA256...
☁️  Uploading to Cloudflare...
✅ Upload successful!
```

### Step 4.4: Verify Upload

```bash
shorebird_custom_server check \
  --api-url=https://your-worker.workers.dev \
  --app-id=com.example.app \
  --release-version=1.0.0+1 \
  --platform=android \
  --arch=aarch64
```

**Expected**:
```
🔍 Checking for updates...
✅ Update available!
  Patch #: 1
```

### Step 4.5: Test App Update

1. Restart your app
2. Check logs for "[Shorebird] Update available!"
3. Restart app again
4. Verify your code change is visible

✅ **Phase 4 Complete!** System is working end-to-end.

---

## Troubleshooting

### Backend Issues

**"wrangler: command not found"**
```bash
npm install -g wrangler
```

**"Database not found"**
```bash
wrangler d1 list
# Copy correct database_id to wrangler.toml
```

**"Bucket not found"**
```bash
wrangler r2 bucket list
# Create if missing: wrangler r2 bucket create shorebird-patches
```

### CLI Issues

**"dart: command not found"**
- Install Dart SDK: https://dart.dev/get-dart

**"Patch file not found"**
- Check file path is correct
- Use absolute path if needed

**"Invalid API key"**
- Check API_KEY in wrangler.toml matches command

### Client Issues

**"Updater not available"**
- Must build with `shorebird release`, not `flutter build`

**"Connection refused"**
- Check base_url in shorebird.yaml
- Test Worker URL in browser

**"No update available" but patch exists**
- Check app_id matches exactly
- Check release_version matches exactly
- Check platform and arch match

---

## Summary

**What you built**:
- ✅ Serverless backend on Cloudflare (Workers + D1 + R2)
- ✅ CLI tool to upload patches
- ✅ Flutter app configured for self-hosted updates
- ✅ Complete update workflow

**Cost**: $0/month (100% FREE on Cloudflare free tier)

**Time spent**: ~2.5 hours

**Next steps**:
- Deploy to production
- Monitor updates in Cloudflare dashboard
- Add more apps
- Automate patch download (TODO)

---

## Database Schema Reference

```sql
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
```

## R2 Storage Structure

```
Path format: {app_id}/{platform}/{arch}/{release_version}/{patch_number}

Example: com.example.app/android/aarch64/1.0.0+1/1
```

## API Endpoints

**Check for updates**:
```
POST /api/v1/patches/check
Content-Type: application/json

{
  "app_id": "com.example.app",
  "release_version": "1.0.0+1",
  "platform": "android",
  "arch": "aarch64",
  "channel": "stable",
  "client_id": "uuid"
}
```

**Upload patch** (CLI only):
```
POST /api/upload
X-API-Key: your-key
Content-Type: multipart/form-data

file: <binary>
app_id: com.example.app
release_version: 1.0.0+1
platform: android
arch: aarch64
patch_number: 1
```

---

**End of Implementation Guide**
