# Overview: Self-Hosted Code Push System

## 🎯 Project Goal

Build a complete self-hosted code push system that replaces Shorebird's cloud service with **100% Cloudflare infrastructure** (Workers + D1 + R2).

**🌟 Tại sao Cloudflare?**
- ✅ 100% FREE (trong free tier)
- ✅ Zero VPS - Serverless
- ✅ Auto SSL - HTTPS tự động
- ✅ Auto scaling - Tự động scale
- ✅ Global CDN - Nhanh toàn cầu
- ✅ Zero DevOps - Chỉ cần `wrangler deploy`

## 📊 System Components

```
┌─────────────────────────────────────────────────────────────┐
│                     COMPONENT 1: CLI                        │
│                                                             │
│  Language: Dart                                             │
│  Purpose: Developer tool to upload patches                  │
│  Input: shorebird_custom_server patch android              │
│  Output: Patch uploaded to backend                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   COMPONENT 2: BACKEND                      │
│                                                             │
│  Platform: Cloudflare Workers (Serverless)                  │
│  Purpose: API server + Storage management                   │
│  Endpoints:                                                 │
│    - POST /api/upload (from CLI)                            │
│    - GET /api/check (from Flutter client)                   │
│  Storage: Cloudflare R2                                     │
│  Database: Cloudflare D1 (SQLite)                           │
│  Cost: 100% FREE (free tier)                                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   COMPONENT 3: CLIENT                       │
│                                                             │
│  Language: Dart/Flutter                                     │
│  Purpose: Auto-update in production apps                    │
│  Base: Fork from github.com/shorebirdtech/updater           │
│  Features:                                                  │
│    - Check for updates on app start                         │
│    - Download patches from CDN                              │
│    - Apply patches using Shorebird engine                   │
│    - Restart app with new code                              │
└─────────────────────────────────────────────────────────────┘
```

## 🔄 Complete Flow

```
┌──────────────┐
│  Developer   │
└──────┬───────┘
       │
       │ 1. Make code changes
       │
       ▼
┌──────────────────────────────────────┐
│ shorebird_custom_server patch android│
└──────┬───────────────────────────────┘
       │
       │ 2. CLI runs shorebird patch
       │ 3. CLI finds .vmcode file
       │ 4. CLI calculates SHA256
       │ 5. CLI uploads to backend
       │
       ▼
┌──────────────────────────────────────┐
│  Backend API (POST /api/upload)      │
└──────┬───────────────────────────────┘
       │
       │ 6. Receive patch file
       │ 7. Upload to R2/S3
       │ 8. Save metadata to DB
       │ 9. Return success + CDN URL
       │
       ▼
┌──────────────────────────────────────┐
│  Storage (R2) + Database (SQLite)    │
└──────────────────────────────────────┘
       │
       │ Patch is now available
       │
       ▼
┌──────────────────────────────────────┐
│  End User Opens App                  │
└──────┬───────────────────────────────┘
       │
       │ 10. App starts
       │ 11. Client checks for update
       │
       ▼
┌──────────────────────────────────────┐
│  Backend API (GET /api/check)        │
└──────┬───────────────────────────────┘
       │
       │ 12. Query DB for latest patch
       │ 13. Return patch info if available
       │
       ▼
┌──────────────────────────────────────┐
│  Flutter Client                      │
└──────┬───────────────────────────────┘
       │
       │ 14. Download patch from CDN
       │ 15. Verify SHA256 hash
       │ 16. Apply patch (Shorebird engine)
       │ 17. Restart app
       │
       ▼
┌──────────────────────────────────────┐
│  App Running with New Code ✓         │
└──────────────────────────────────────┘
```

## 📋 Implementation Order

### Phase 1: Backend (Foundation)
**Time: ~30 minutes**
- Setup Node.js project
- Create database schema
- Implement upload endpoint
- Implement check endpoint
- Setup R2 storage
- Test with curl/Postman

### Phase 2: CLI (Developer Tool)
**Time: ~45 minutes**
- Setup Dart project
- Implement config loader
- Implement shorebird runner
- Implement patch finder
- Implement uploader
- Compile to binary
- Test end-to-end

### Phase 3: Client (End User)
**Time: ~20 minutes**
- Fork shorebird/updater
- Modify API endpoints
- Update configuration
- Test in sample app
- Document usage

### Phase 4: Testing & Documentation
**Time: ~15 minutes**
- End-to-end testing
- Write usage guides
- Create deployment guide
- Security checklist

## 📁 File Structure

```
codepush/
├── plan/                          # ← Planning documents (current)
│   ├── 00-OVERVIEW.md            # This file
│   ├── 01-BACKEND.md             # Backend detailed plan
│   ├── 02-CLI.md                 # CLI detailed plan
│   ├── 03-CLIENT.md              # Client detailed plan
│   ├── 04-DATABASE.md            # Database schema plan
│   ├── 05-STORAGE.md             # R2/S3 storage plan
│   ├── 06-SECURITY.md            # Security considerations
│   └── 07-DEPLOYMENT.md          # Deployment strategy
│
├── backend/                       # Backend implementation
│   ├── package.json
│   ├── .env.example
│   ├── index.js
│   ├── db.js
│   ├── storage.js
│   └── routes/
│       ├── upload.js
│       └── check.js
│
├── cli/                          # CLI implementation
│   ├── pubspec.yaml
│   ├── .env.example
│   ├── bin/
│   │   └── shorebird_custom_server.dart
│   └── lib/
│       ├── config.dart
│       ├── shorebird_runner.dart
│       ├── patch_finder.dart
│       ├── uploader.dart
│       └── models.dart
│
├── client/                       # Flutter client package
│   ├── pubspec.yaml
│   ├── lib/
│   │   ├── shorebird_updater.dart
│   │   ├── update_checker.dart
│   │   ├── patch_downloader.dart
│   │   └── patch_installer.dart
│   └── example/
│       └── main.dart
│
└── docs/                         # User documentation
    ├── 01-SETUP.md
    ├── 02-CLI-USAGE.md
    ├── 03-CLIENT-USAGE.md
    └── 04-DEPLOY.md
```

## 🔧 Technology Stack

### Backend
- **Platform:** Cloudflare Workers
- **Runtime:** V8 JavaScript
- **Database:** Cloudflare D1 (SQLite)
- **Storage:** Cloudflare R2
- **Auth:** API Key based
- **Deployment:** Wrangler CLI
- **Cost:** FREE

### CLI
- **Language:** Dart 3.0+
- **HTTP Client:** http package
- **Config:** dotenv
- **File Operations:** dart:io
- **Crypto:** crypto package
- **Process:** dart:io Process

### Client
- **Framework:** Flutter 3.24+
- **Base:** shorebird/updater fork
- **HTTP:** http package
- **Storage:** shared_preferences
- **Crypto:** crypto package
- **Engine:** Shorebird patched engine

### Storage
- **Primary:** Cloudflare R2
- **CDN:** R2 public URL (built-in)
- **Cost:** FREE (10GB storage, unlimited egress)

### Database
- **Platform:** Cloudflare D1
- **Type:** SQLite (serverless)
- **ORM:** None (raw SQL for simplicity)
- **Cost:** FREE (5M reads/day, 100K writes/day)

## 📊 Data Flow

### Upload Flow (CLI → Backend)
```
1. CLI reads .env config
2. CLI runs: shorebird patch android
3. CLI finds: build/**/*.vmcode
4. CLI calculates: SHA256 hash
5. CLI reads: pubspec.yaml for version
6. CLI prepares: multipart/form-data
7. CLI sends: POST /api/upload
8. Backend receives: file + metadata
9. Backend uploads: file to R2
10. Backend inserts: metadata to DB
11. Backend returns: success + CDN URL
12. CLI displays: success message
```

### Check Flow (Client → Backend)
```
1. App starts
2. Client reads: current version + patch number
3. Client sends: GET /api/check?app_id=xxx&version=1.0.0&platform=android&patch=0
4. Backend queries: DB for latest patch
5. Backend checks: version compatibility
6. Backend returns: patch info or no update
7. Client decides: download or skip
```

### Download & Apply Flow (Client)
```
1. Client downloads: patch from CDN URL
2. Client verifies: SHA256 hash
3. Client saves: patch to temp directory
4. Client calls: Shorebird engine native method
5. Engine applies: patch to app
6. Client updates: local patch number
7. Client restarts: app (or prompts user)
```

## 🔐 Security Considerations

### API Security
- API key authentication for all endpoints
- Rate limiting on upload endpoint
- File size limits (max 50MB)
- File type validation (.vmcode only)
- HTTPS only in production

### Storage Security
- Private R2 bucket
- Signed URLs for downloads (optional)
- SHA256 verification before apply
- Secure credential storage

### Client Security
- Hash verification mandatory
- Rollback on failed patch
- Version compatibility check
- No arbitrary code execution

## 📈 Scalability

### Current Design (MVP)
- Single server
- SQLite database
- R2 storage
- Handles: ~1000 apps, ~10K users

### Future Scaling
- Multiple backend instances
- PostgreSQL with replication
- Redis for caching
- CDN for global distribution
- Load balancer

## 🎯 Success Criteria

### Backend
- [ ] Upload endpoint working
- [ ] Check endpoint working
- [ ] R2 storage integrated
- [ ] Database persisting data
- [ ] API key auth working

### CLI
- [ ] Runs shorebird commands
- [ ] Finds patch files
- [ ] Uploads successfully
- [ ] Handles errors gracefully
- [ ] Compiles to binary

### Client
- [ ] Checks for updates
- [ ] Downloads patches
- [ ] Verifies hashes
- [ ] Applies patches
- [ ] Restarts app

### Integration
- [ ] End-to-end flow works
- [ ] Multiple platforms (Android/iOS)
- [ ] Error handling complete
- [ ] Documentation clear

## 📝 Next Steps

1. Read `01-BACKEND.md` for backend implementation details
2. Read `02-CLI.md` for CLI implementation details
3. Read `03-CLIENT.md` for client implementation details
4. Review `04-DATABASE.md` for schema design
5. Review `05-STORAGE.md` for R2 setup
6. Review `06-SECURITY.md` for security checklist
7. Review `07-DEPLOYMENT.md` for deployment guide

## ⏱️ Estimated Timeline

| Phase | Component | Time | Status |
|-------|-----------|------|--------|
| 1 | Backend Setup | 30 min | ⏳ Pending |
| 2 | CLI Development | 45 min | ⏳ Pending |
| 3 | Client Integration | 20 min | ⏳ Pending |
| 4 | Testing | 15 min | ⏳ Pending |
| **Total** | | **110 min** | |

## 🚀 Ready to Start?

Once you approve this plan, we'll proceed with detailed implementation plans for each component.
