# Self-Hosted Code Push with Shorebird - Agent Guide

## Project Overview

This is a **minimal self-hosted code push system** using Shorebird engine + **100% Cloudflare**. It allows Flutter developers to deploy patches to their own infrastructure instead of relying on Shorebird's cloud service.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        DEVELOPER                            │
│  shorebird_custom_server patch android                     │
│  → Run shorebird CLI → Extract .vmcode file → Upload       │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              CLOUDFLARE (100% Serverless)                   │
│  Workers: API endpoints (upload + check)                    │
│  D1: SQLite database (patch metadata)                       │
│  R2: Storage (patch files)                                  │
└─────────────────────┬───────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│                    END USER (Flutter)                       │
│  App starts → Check update → Download → Apply → Restart    │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

1. **CLI Tool** (Dart): Command-line tool for developers to upload patches
2. **Backend API** (Cloudflare Workers): Serverless API for uploads and update checks
3. **Flutter Client** (Dart package): In-app updater that checks and applies patches
4. **Updater Library** (Rust): Core updater logic forked from shorebirdtech/updater

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| CLI | Dart 3.0+ | Native binary for patch uploads |
| Backend | Cloudflare Workers (JavaScript) | Serverless API endpoints |
| Database | Cloudflare D1 (SQLite) | Patch metadata storage |
| Storage | Cloudflare R2 (S3-compatible) | Patch file storage |
| Client | Flutter + Shorebird engine | End-user app updates |
| Updater Core | Rust | Patch download and application |

## Project Structure

```
codepush/
├── README.md                  # Project overview (human-readable)
├── START-HERE.md             # Entry point for implementation
├── IMPLEMENTATION-GUIDE.md   # Complete 2.5-hour implementation guide
├── PROJECT-STATUS.md         # Current status and readiness
├── AGENTS.md                 # This file
├── plan/                     # Detailed planning documentation
│   ├── 00-OVERVIEW.md        # System architecture overview
│   ├── 01-BACKEND.md         # Cloudflare Workers backend (45 min)
│   ├── 02-CLI.md             # CLI tool implementation (60 min)
│   ├── 03-CLIENT.md          # Flutter client integration (10 min)
│   ├── 04-DATABASE.md        # D1 database schema
│   ├── 05-STORAGE.md         # R2 storage configuration
│   ├── 06-SECURITY.md        # Security considerations
│   ├── 07-DEPLOYMENT.md      # Deployment guide
│   └── 08-CHECKLIST.md       # Step-by-step checklist
└── updater/                  # Shorebird updater source (reference)
    ├── Cargo.toml            # Rust workspace definition
    ├── library/              # Core updater library (Rust)
    │   ├── Cargo.toml
    │   └── src/              # Rust source files
    ├── patch/                # Patch packaging tool (Rust)
    └── shorebird_code_push/  # Flutter package (Dart)
        ├── pubspec.yaml
        └── lib/              # Dart source files
```

## Build and Development Commands

### Backend (Cloudflare Workers)

```bash
# Install dependencies
npm install -g wrangler

# Login to Cloudflare
wrangler login

# Create D1 database
wrangler d1 create shorebird-db

# Create R2 bucket
wrangler r2 bucket create shorebird-patches

# Run locally
npm run dev

# Deploy to production
npm run deploy

# View logs
wrangler tail
```

### CLI Tool (Dart)

```bash
# Get dependencies
dart pub get

# Run from source
dart run bin/shorebird_custom_server.dart

# Compile to native binary
dart compile exe bin/shorebird_custom_server.dart -o shorebird_custom_server

# Install globally
sudo mv shorebird_custom_server /usr/local/bin/
```

### Updater Library (Rust)

```bash
# Build the library
cargo build --release

# Build for Android (requires cargo-ndk)
cargo ndk --target aarch64-linux-android build --release

# Run tests
cargo test

# Generate coverage report
cargo llvm-cov
```

### Flutter Client

```bash
# Get dependencies
flutter pub get

# Build with Shorebird (required for updater to work)
shorebird release android
shorebird release ios

# Create patch
shorebird patch android
shorebird patch ios
```

## API Endpoints

### Check for Updates (Shorebird-compatible)

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

**Response:**
```json
{
  "patch_available": true,
  "patch": {
    "number": 1,
    "hash": "abc123...",
    "download_url": "https://pub-xxx.r2.dev/...",
    "hash_signature": null
  },
  "rolled_back_patch_numbers": []
}
```

### Upload Patch (CLI only)

```
POST /api/upload
X-API-Key: your-secret-key
Content-Type: multipart/form-data

Fields:
- file: <patch binary>
- app_id: com.example.app
- release_version: 1.0.0+1
- platform: android
- arch: aarch64
- patch_number: 1
```

## Database Schema (D1)

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

## Code Style Guidelines

### JavaScript (Cloudflare Workers)

- Use ES modules (`export default`)
- Async/await for asynchronous operations
- CORS headers on all responses
- Prepared statements for database queries

### Dart (CLI and Flutter)

- Follow Effective Dart guidelines
- Use `package:args` for CLI argument parsing
- Use `package:http` for HTTP requests
- Calculate SHA256 hashes for patch verification

### Rust (Updater Library)

- Follow Rust naming conventions (snake_case)
- Use `anyhow` for error handling
- Use `serde` for JSON serialization
- Comprehensive unit tests with `mockall`

## Testing Instructions

### Backend Testing

```bash
# Test check endpoint
curl -X POST https://your-worker.workers.dev/api/v1/patches/check \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "com.test.app",
    "release_version": "1.0.0+1",
    "platform": "android",
    "arch": "aarch64",
    "channel": "stable",
    "client_id": "test"
  }'

# Test upload endpoint
curl -X POST https://your-worker.workers.dev/api/upload \
  -H "X-API-Key: your-key" \
  -F "file=@test.bin" \
  -F "app_id=com.test.app" \
  -F "release_version=1.0.0+1" \
  -F "platform=android" \
  -F "arch=aarch64" \
  -F "patch_number=1"
```

### CLI Testing

```bash
# Check command
shorebird_custom_server check \
  --api-url=https://your-worker.workers.dev \
  --app-id=com.example.app \
  --release-version=1.0.0+1 \
  --platform=android \
  --arch=aarch64

# Upload command
shorebird_custom_server upload patch.bin \
  --api-url=https://your-worker.workers.dev \
  --api-key=your-key \
  --app-id=com.example.app \
  --release-version=1.0.0+1 \
  --platform=android \
  --arch=aarch64 \
  --patch-number=1
```

### Rust Tests

```bash
cd updater/library
cargo test

# Run with coverage
cargo llvm-cov
```

## Security Considerations

### API Authentication

- Use strong API keys (32+ characters, hex-encoded)
- Store API key in `wrangler.toml` (backend) and `.env` (CLI)
- Never commit API keys to git
- Use constant-time comparison to prevent timing attacks

### Input Validation

- Validate `app_id` format (reverse domain notation)
- Validate `platform` is either `android` or `ios`
- Validate `arch` is one of: `aarch64`, `x86_64`, `x86`, `arm`
- Use prepared statements to prevent SQL injection

### File Validation

- Limit file size (50 MB max recommended)
- Calculate and verify SHA256 hash
- Store hash in database and verify on download

### Transport Security

- HTTPS enforced by Cloudflare (automatic)
- CORS headers configured for cross-origin requests
- Rate limiting via Cloudflare (100K requests/day on free tier)

## Deployment Process

### Prerequisites

1. Cloudflare account (free tier sufficient)
2. Node.js 18+ installed
3. Dart 3.0+ installed
4. Wrangler CLI authenticated

### Deployment Steps

1. **Backend**:
   ```bash
   cd backend
   npm run deploy
   ```

2. **CLI**:
   ```bash
   dart compile exe bin/shorebird_custom_server.dart -o shorebird_custom_server
   sudo mv shorebird_custom_server /usr/local/bin/
   ```

3. **Client**: No deployment needed - Flutter package is imported

### Post-Deployment Verification

- [ ] Backend responds at production URL
- [ ] D1 database has patches table
- [ ] R2 bucket has public access enabled
- [ ] CLI can upload patches
- [ ] Client can check for updates
- [ ] End-to-end patch flow works

## Cost Breakdown (Free Tier)

| Service | Limit | Cost |
|---------|-------|------|
| Cloudflare Workers | 100K requests/day | FREE |
| Cloudflare D1 | 5M reads/day, 100K writes/day | FREE |
| Cloudflare R2 | 10 GB storage, unlimited egress | FREE |

**Supports**: ~1000 apps, ~10K users, ~100 patches/day

## Known Limitations

1. **Manual download step**: Must manually download patch from Shorebird dashboard after `shorebird patch` (TODO: automate with Shorebird API)
2. **Architecture detection**: Must manually specify architecture when uploading
3. **Shorebird dependency**: Still depends on Shorebird cloud for initial patch creation

## Troubleshooting

### Backend Issues

- **"Database not found"**: Check `database_id` in `wrangler.toml`
- **"Bucket not found"**: Verify R2 bucket exists and is named correctly
- **"Invalid API key"**: Ensure API key matches between CLI and backend

### CLI Issues

- **"Patch file not found"**: Check file path and permissions
- **"Upload failed: 400"**: Missing required field in request
- **"SHA256 mismatch"**: Patch file corrupted, re-download from Shorebird

### Client Issues

- **"Updater not available"**: App must be built with `shorebird release`, not `flutter build`
- **"Connection refused"**: Check `base_url` in `shorebird.yaml`
- **"No update available"**: Verify app_id, version, platform, and arch match exactly

## Documentation References

- **Start here**: `START-HERE.md`
- **Complete guide**: `IMPLEMENTATION-GUIDE.md`
- **Backend details**: `plan/01-BACKEND.md`
- **CLI details**: `plan/02-CLI.md`
- **Client details**: `plan/03-CLIENT.md`
- **Security**: `plan/06-SECURITY.md`
- **Deployment**: `plan/07-DEPLOYMENT.md`

## License

The updater library is dual-licensed under MIT and Apache 2.0 licenses.
