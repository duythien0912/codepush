# Self-Hosted Code Push with Shorebird

Minimal self-hosted code push system using Shorebird engine + **100% Cloudflare**.

## 🎯 Goal

Replace Shorebird's cloud service with your own infrastructure:

```bash
# Developer side
shorebird_custom_server patch android
→ Auto upload to Cloudflare

# End user side  
App opens → Check update → Download → Apply patch → Restart
```

## 🌟 Why Cloudflare?

- ✅ **100% FREE** (trong free tier)
- ✅ **Zero VPS** - Serverless
- ✅ **Auto SSL** - HTTPS tự động
- ✅ **Auto scaling** - Tự động scale
- ✅ **Global CDN** - Nhanh toàn cầu
- ✅ **Zero DevOps** - Chỉ cần `wrangler deploy`

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        DEVELOPER                            │
│                                                             │
│  shorebird_custom_server patch android                     │
│  → Run shorebird CLI                                        │
│  → Extract .vmcode file                                     │
│  → Upload to Cloudflare                                     │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              CLOUDFLARE (100% Serverless)                   │
│                                                             │
│  Workers: API endpoints (upload + check)                    │
│  D1: SQLite database (patch metadata)                       │
│  R2: Storage (patch files)                                  │
│                                                             │
│  ✅ 100% FREE (free tier)                                   │
│  ✅ Auto SSL/HTTPS                                          │
│  ✅ Auto scaling                                            │
│  ✅ Global CDN                                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│                    END USER (Flutter)                       │
│                                                             │
│  App starts                                                 │
│  → Check for update (GET /api/check)                        │
│  → Download patch from R2                                   │
│  → Apply with Shorebird engine                              │
│  → Restart app                                              │
└─────────────────────────────────────────────────────────────┘
```

## 📦 Components

### 1. CLI Tool (Dart)
**For:** Developers  
**Purpose:** Automate patch upload  

```bash
shorebird_custom_server patch android
```

### 2. Backend API (Cloudflare Workers)
**For:** Server infrastructure  
**Purpose:** Handle uploads, serve updates  
**Platform:** 100% Cloudflare (Workers + D1 + R2)

Endpoints:
- `POST /api/upload` - Receive patches from CLI
- `GET /api/check` - Serve updates to clients

### 3. Flutter Client (Package)
**For:** End users  
**Purpose:** Auto-update in production apps  

```dart
final updater = ShorebirdUpdater(
  apiUrl: 'https://your-worker.workers.dev',
  appId: 'com.yourapp.name',
);
await updater.checkAndUpdate();
```

## 🚀 Quick Start

### 1. Setup Backend (Cloudflare)

```bash
# Install Wrangler CLI
npm install -g wrangler
wrangler login

# Create project
mkdir shorebird-backend
cd shorebird-backend

# Follow plan/01-BACKEND.md for detailed steps
# Deploy
npm run deploy
```

### 2. Build CLI

```bash
cd cli
dart pub get
dart compile exe bin/shorebird_custom_server.dart -o shorebird_custom_server
sudo mv shorebird_custom_server /usr/local/bin/
```

### 3. Use in Flutter App

```bash
# In your Flutter project
flutter pub add shorebird_updater

# In main.dart
import 'package:shorebird_updater/shorebird_updater.dart';

void main() async {
  final updater = ShorebirdUpdater(
    apiUrl: 'https://your-worker.workers.dev',
    appId: 'com.yourapp.name',
  );
  
  await updater.checkAndUpdate();
  runApp(MyApp());
}
```

### 4. Deploy Patch

```bash
# Make code changes
# Then run:
shorebird_custom_server patch android
```

## 📁 Project Structure

```
codepush/
├── README.md                  # This file
├── PLANNING_COMPLETE.md      # Planning summary
└── plan/                      # Detailed planning docs
    ├── README.md             # Planning guide
    ├── 00-OVERVIEW.md        # System overview
    ├── 01-BACKEND.md         # Cloudflare Workers (30 min)
    ├── 02-CLI.md             # CLI tool (45 min)
    ├── 03-CLIENT.md          # Flutter client (20 min)
    ├── 04-DATABASE.md        # D1 database schema
    ├── 05-STORAGE.md         # R2 storage
    ├── 06-SECURITY.md        # Security measures
    ├── 07-DEPLOYMENT.md      # Deployment guide
    └── 08-CHECKLIST.md       # Step-by-step checklist
```

## ⏱️ Implementation Time

- Backend (Cloudflare): 30 min
- CLI: 45 min
- Client: 20 min
- Testing: 15 min
- **Total: ~110 min**

**Cost: $0/month (100% FREE!)** 🎉

## 🔧 Tech Stack

- **CLI:** Dart (compile to native binary)
- **Backend:** Cloudflare Workers (Serverless JavaScript)
- **Database:** Cloudflare D1 (SQLite)
- **Storage:** Cloudflare R2 (S3-compatible, free egress)
- **Client:** Flutter + Shorebird engine
- **Cost:** 100% FREE (in free tier)

## 📝 Next Steps

### 1. Read Planning Documents

**Start here:** [plan/README.md](plan/README.md)

Detailed planning documents:
- [00-OVERVIEW.md](plan/00-OVERVIEW.md) - System overview
- [01-BACKEND.md](plan/01-BACKEND.md) - Cloudflare Workers (30 min)
- [02-CLI.md](plan/02-CLI.md) - CLI implementation (45 min)
- [03-CLIENT.md](plan/03-CLIENT.md) - Client implementation (20 min)
- [04-DATABASE.md](plan/04-DATABASE.md) - D1 database schema
- [05-STORAGE.md](plan/05-STORAGE.md) - R2 storage setup
- [06-SECURITY.md](plan/06-SECURITY.md) - Security considerations
- [07-DEPLOYMENT.md](plan/07-DEPLOYMENT.md) - Deployment guide
- [08-CHECKLIST.md](plan/08-CHECKLIST.md) - Implementation checklist

### 2. Implementation Order

1. Setup Cloudflare backend (30 min)
2. Build CLI tool (45 min)
3. Setup Flutter client (20 min)
4. Test integration (15 min)
5. Done! (Already deployed)

**Total: ~110 min**

## 🔐 Security Notes

- Use API keys for authentication
- Verify SHA256 hash before applying patches
- Use HTTPS for all communications (auto with Cloudflare)
- Store credentials securely
- Cloudflare provides DDoS protection, WAF, bot protection

## 💰 Cost Breakdown

### Cloudflare Workers (Free Tier)
- Requests: 100K/day - **FREE**
- CPU time: 10ms per request - **FREE**
- Auto SSL - **FREE**

### Cloudflare R2 (Free Tier)
- Storage: 10 GB/month - **FREE**
- Egress: Unlimited - **FREE**
- Operations: 10M reads/month - **FREE**

### Cloudflare D1 (Free Tier)
- Rows read: 5M/day - **FREE**
- Rows written: 100K/day - **FREE**
- Storage: 5 GB - **FREE**

**Total: $0/month** 🎉

Đủ cho:
- ~1000 apps
- ~10K users
- ~100 patches/day

## 📚 Documentation

- [Setup Guide](plan/01-BACKEND.md)
- [CLI Usage](plan/02-CLI.md)
- [Client Integration](plan/03-CLIENT.md)
- [Deployment](plan/07-DEPLOYMENT.md)
- [Checklist](plan/08-CHECKLIST.md)

## 🤝 Credits

Based on [Shorebird](https://shorebird.dev/) technology.
Client forked from [shorebird/updater](https://github.com/shorebirdtech/updater).
Powered by [Cloudflare](https://cloudflare.com/).

## 🎓 Perfect for Freshers!

Planning documents được viết cực kỳ chi tiết với:
- ✅ Step-by-step instructions
- ✅ Code examples đầy đủ
- ✅ Giải thích từng dòng code
- ✅ Troubleshooting guide
- ✅ Screenshots (trong docs)
- ✅ Common errors & solutions

**Fresher có thể làm được trong 2 giờ!** 🚀
