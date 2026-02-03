# Implementation Checklist (Cloudflare)

## 📋 Complete Step-by-Step Guide

Hướng dẫn chi tiết từng bước để implement toàn bộ hệ thống với **100% Cloudflare**.

## 🎯 Prerequisites

- [ ] Cloudflare account (FREE - tạo tại dash.cloudflare.com)
- [ ] Node.js 18+ installed
- [ ] Dart 3.0+ installed  
- [ ] Shorebird CLI installed
- [ ] Flutter project with Shorebird initialized
- [ ] Code editor (VS Code recommended)

## 📚 Planning Phase (Đọc trước - 15 phút)

- [ ] Read `00-OVERVIEW.md` - Hiểu tổng quan hệ thống
- [ ] Read `01-BACKEND.md` - Backend Cloudflare Workers chi tiết
- [ ] Read `02-CLI.md` - CLI tool chi tiết
- [ ] Read `03-CLIENT.md` - Flutter client chi tiết
- [ ] Read `04-DATABASE.md` - D1 database schema
- [ ] Read `05-STORAGE.md` - R2 storage setup
- [ ] Read `06-SECURITY.md` - Security measures
- [ ] Read `07-DEPLOYMENT.md` - Deployment guide

## 🏗️ Phase 1: Backend - Cloudflare Workers (30 phút)

### Bước 1.1: Tạo Cloudflare Account (2 phút)

1. Vào https://dash.cloudflare.com/sign-up
2. Nhập email và password
3. Verify email
4. Login vào dashboard

- [ ] Account created
- [ ] Email verified
- [ ] Logged in

### Bước 1.2: Install Wrangler CLI (1 phút)

```bash
npm install -g wrangler
wrangler --version
wrangler login
```

- [ ] Wrangler installed
- [ ] Version hiển thị (3.x.x)
- [ ] Login successful

### Bước 1.3: Create Project (2 phút)

```bash
mkdir shorebird-backend
cd shorebird-backend
npm init -y
```

- [ ] Folder created
- [ ] package.json created

### Bước 1.4: Create wrangler.toml (2 phút)

**Tạo file:** `wrangler.toml`

```toml
name = "shorebird-backend"
main = "src/index.js"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "shorebird-db"
database_id = "YOUR_DATABASE_ID"

[[r2_buckets]]
binding = "R2"
bucket_name = "shorebird-patches"

[vars]
API_KEY = "CHANGE-THIS-TO-STRONG-KEY"
R2_PUBLIC_URL = "YOUR_R2_PUBLIC_URL"
```

- [ ] File created

### Bước 1.5: Setup D1 Database (3 phút)

```bash
wrangler d1 create shorebird-db
```

**Copy database_id từ output và paste vào wrangler.toml**

**Tạo file:** `schema.sql`

```sql
CREATE TABLE patches (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  app_id TEXT NOT NULL,
  platform TEXT NOT NULL CHECK(platform IN ('android', 'ios')),
  patch_number INTEGER NOT NULL,
  version TEXT NOT NULL,
  download_url TEXT NOT NULL,
  sha256 TEXT NOT NULL,
  size_bytes INTEGER NOT NULL,
  active INTEGER DEFAULT 1,
  created_at TEXT DEFAULT (datetime('now')),
  UNIQUE(app_id, platform, patch_number)
);

CREATE INDEX idx_patches_lookup 
ON patches(app_id, platform, active, patch_number);
```

```bash
wrangler d1 execute shorebird-db --file=schema.sql
```

- [ ] D1 database created
- [ ] database_id updated in wrangler.toml
- [ ] schema.sql created
- [ ] Schema applied

### Bước 1.6: Setup R2 Storage (2 phút)

```bash
wrangler r2 bucket create shorebird-patches
```

**Enable public access:**
1. Vào Cloudflare Dashboard
2. R2 → shorebird-patches
3. Settings → Public access → Allow
4. Copy public URL (vd: https://pub-abc123.r2.dev)
5. Paste vào wrangler.toml (R2_PUBLIC_URL)

- [ ] R2 bucket created
- [ ] Public access enabled
- [ ] Public URL copied to wrangler.toml

### Bước 1.7: Implement Code (15 phút)

**Tạo file:** `src/index.js`

```javascript
import { handleUpload } from './upload.js';
import { handleCheck } from './check.js';

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, X-API-Key',
    };
    
    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }
    
    if (url.pathname === '/api/upload' && request.method === 'POST') {
      const response = await handleUpload(request, env);
      return new Response(response.body, {
        status: response.status,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' }
      });
    }
    
    if (url.pathname === '/api/check' && request.method === 'GET') {
      const response = await handleCheck(request, env);
      return new Response(response.body, {
        status: response.status,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' }
      });
    }
    
    return new Response('Not Found', { status: 404, headers: corsHeaders });
  }
};
```

**Tạo file:** `src/upload.js` - Copy code từ `plan/01-BACKEND.md` section 5.2

**Tạo file:** `src/check.js` - Copy code từ `plan/01-BACKEND.md` section 5.3

- [ ] src/index.js created
- [ ] src/upload.js created  
- [ ] src/check.js created

### Bước 1.8: Test Local (3 phút)

```bash
npm run dev
```

**Terminal mới:**
```bash
curl "http://localhost:8787/api/check?app_id=com.test.app&version=1.0.0&platform=android&patch=0"
```

- [ ] Dev server running
- [ ] Check endpoint returns JSON

### Bước 1.9: Deploy (1 phút)

```bash
npm run deploy
```

**Lưu production URL:**
```
https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev
```

- [ ] Deploy successful
- [ ] Production URL saved

### Bước 1.10: Test Production (1 phút)

```bash
curl "https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev/api/check?app_id=com.test.app&version=1.0.0&platform=android&patch=0"
```

- [ ] Production API working

## 🔧 Phase 2: CLI Development (45 phút)

### Bước 2.1: Create Project (2 phút)

```bash
mkdir cli
cd cli
dart create -t console .
```

- [ ] Project created

### Bước 2.2: Update pubspec.yaml (2 phút)

```yaml
name: shorebird_custom_server
description: CLI tool to upload Shorebird patches
version: 1.0.0

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  args: ^2.4.2
  http: ^1.1.0
  crypto: ^3.0.3
  path: ^1.8.3
  yaml: ^3.1.2
  dotenv: ^4.2.0

executables:
  shorebird_custom_server:
```

```bash
dart pub get
```

- [ ] pubspec.yaml updated
- [ ] Dependencies installed

### Bước 2.3: Create .env (1 phút)

**File:** `.env`

```env
API_URL=https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev
API_KEY=SAME-KEY-AS-WRANGLER-TOML
APP_ID=com.yourapp.name
```

- [ ] .env created

### Bước 2.4-2.9: Implement Code (35 phút)

**Tham khảo:** `plan/02-CLI.md` sections 2.4 đến 2.9

**Files cần tạo:**
- [ ] lib/config.dart
- [ ] lib/shorebird_runner.dart
- [ ] lib/patch_finder.dart
- [ ] lib/uploader.dart
- [ ] lib/models.dart
- [ ] bin/shorebird_custom_server.dart

### Bước 2.10: Test CLI (3 phút)

```bash
dart run bin/shorebird_custom_server.dart patch android --dry-run
```

- [ ] CLI runs
- [ ] Finds shorebird
- [ ] Finds patch file
- [ ] Would upload (dry-run)

### Bước 2.11: Compile & Install (2 phút)

```bash
dart compile exe bin/shorebird_custom_server.dart -o shorebird_custom_server
sudo mv shorebird_custom_server /usr/local/bin/
shorebird_custom_server --help
```

- [ ] Compiled to binary
- [ ] Installed globally
- [ ] Help shows

## 📱 Phase 3: Flutter Client (20 phút)

### Bước 3.1: Fork Repository (2 phút)

```bash
git clone https://github.com/shorebirdtech/updater.git client
cd client
flutter pub get
```

- [ ] Repository cloned
- [ ] Dependencies installed

### Bước 3.2-3.3: Modify Code (15 phút)

**Tham khảo:** `plan/03-CLIENT.md` sections 3.2 và 3.3

**Files cần sửa:**
- [ ] lib/src/update_checker.dart - Change API URL
- [ ] lib/shorebird_updater.dart - Add apiUrl parameter

### Bước 3.4: Test (3 phút)

**Trong Flutter app:**

```dart
final updater = ShorebirdUpdater(
  apiUrl: 'https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev',
  appId: 'com.test.app',
);
await updater.checkAndUpdate();
```

- [ ] Client compiles
- [ ] Checks for updates

## 🧪 Phase 4: End-to-End Test (15 phút)

### Bước 4.1: Create Test App (3 phút)

```bash
flutter create test_app
cd test_app
shorebird init
shorebird release android
```

- [ ] App created
- [ ] Release built

### Bước 4.2: Make Change (1 phút)

```dart
// Change text
Text('Version 2')  // was 'Version 1'
```

- [ ] Code changed

### Bước 4.3: Deploy Patch (2 phút)

```bash
shorebird_custom_server patch android
```

- [ ] Patch uploaded
- [ ] Success message

### Bước 4.4: Verify Backend (1 phút)

```bash
curl "https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev/api/check?app_id=com.test.app&version=1.0.0&platform=android&patch=0"
```

- [ ] Update available
- [ ] Correct patch info

### Bước 4.5: Test Update (8 phút)

```bash
flutter run
```

- [ ] App starts
- [ ] Detects update
- [ ] Downloads patch
- [ ] Installs patch
- [ ] Shows "Version 2"

## 🚀 Phase 5: Production Ready (5 phút)

### Bước 5.1: Security Check (2 phút)

- [ ] API key changed from default
- [ ] API key strong (32+ chars)
- [ ] API key not in git
- [ ] HTTPS working
- [ ] CORS configured

### Bước 5.2: Documentation (3 phút)

- [ ] Production URL documented
- [ ] API key stored securely
- [ ] Team informed
- [ ] Usage guide written

## ✅ Final Checklist

### Backend (Cloudflare Workers)
- [ ] Deployed to production
- [ ] HTTPS auto-enabled
- [ ] D1 database working
- [ ] R2 storage working
- [ ] API endpoints tested
- [ ] Logs accessible (`wrangler tail`)

### CLI
- [ ] Compiled to binary
- [ ] Installed globally
- [ ] .env configured
- [ ] Tested with real project
- [ ] Works from any directory

### Client
- [ ] Modified from shorebird/updater
- [ ] API URL updated
- [ ] Tested in app
- [ ] Update flow works
- [ ] Error handling works

### Integration
- [ ] End-to-end flow tested
- [ ] Upload works
- [ ] Check works
- [ ] Download works
- [ ] Install works
- [ ] App restarts with new code

### Documentation
- [ ] README updated
- [ ] Production URL saved
- [ ] API key documented
- [ ] Usage guide complete
- [ ] Troubleshooting guide ready

## 🎉 Success!

Hệ thống của bạn giờ đã:
- ✅ 100% Cloudflare (Workers + D1 + R2)
- ✅ 100% FREE (trong free tier)
- ✅ Auto SSL/HTTPS
- ✅ Auto scaling
- ✅ Global CDN
- ✅ Zero maintenance

## 📊 Monitoring

### View Logs
```bash
wrangler tail
```

### View Analytics
1. Cloudflare Dashboard
2. Workers & Pages
3. shorebird-backend
4. Metrics tab

## 🔄 Update Backend

```bash
# Edit code
nano src/index.js

# Test local
npm run dev

# Deploy
npm run deploy
```

## 💾 Backup

### D1 Database
```bash
wrangler d1 export shorebird-db --output=backup.sql
```

### R2 Storage
- Auto-replicated by Cloudflare
- 99.999999999% durability

## 📞 Troubleshooting

### Issue: Deploy failed
```bash
wrangler --version
npm install -g wrangler@latest
npm run deploy
```

### Issue: API returns error
```bash
wrangler tail  # View logs
# Fix code
npm run deploy
```

### Issue: Can't access URL
- Wait 1-2 minutes
- Check URL spelling
- Try incognito mode

## 🎓 Tips cho Fresher

1. **Test local trước:** `npm run dev`
2. **Đọc logs:** `wrangler tail`
3. **Backup trước khi thay đổi lớn**
4. **Document mọi thứ**
5. **Đừng sợ deploy** - Cloudflare có rollback

## 🔄 Next Steps

- Monitor performance
- Add more apps
- Scale if needed (upgrade to paid tier)
- Add analytics
- Enjoy! 🎉

**Chúc mừng! Bạn đã hoàn thành! 🚀**
