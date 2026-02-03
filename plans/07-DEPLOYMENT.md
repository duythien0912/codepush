# Deployment Plan (Cloudflare Only)

## 🎯 Mục tiêu

Deploy toàn bộ hệ thống lên Cloudflare - KHÔNG CẦN VPS!

## 🌟 Tại sao đơn giản?

✅ **Không cần VPS** - Serverless
✅ **Không cần nginx** - Cloudflare lo
✅ **Không cần SSL setup** - Auto HTTPS
✅ **Không cần PM2** - Workers tự chạy
✅ **Không cần firewall** - Cloudflare bảo vệ
✅ **Chỉ cần 1 command:** `wrangler deploy`

## 🏗️ Infrastructure

```
┌─────────────────────────────────────────────────────────────┐
│                      CLOUDFLARE                             │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Workers    │  │      D1      │  │      R2      │    │
│  │   (Backend)  │  │  (Database)  │  │  (Storage)   │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│                                                             │
│  URL: https://your-worker.workers.dev                      │
│  SSL: ✅ Auto                                              │
│  CDN: ✅ Global                                            │
│  Scale: ✅ Auto                                            │
└─────────────────────────────────────────────────────────────┘
```

## 📋 BƯỚC 1: Chuẩn bị (5 phút)

### 1.1 Checklist trước khi deploy

- [ ] Backend code đã test local (`npm run dev`)
- [ ] Upload endpoint hoạt động
- [ ] Check endpoint hoạt động
- [ ] D1 database có data test
- [ ] R2 bucket có file test
- [ ] API key đã đổi (không dùng default)

### 1.2 Update API key

**File:** `wrangler.toml`

```toml
[vars]
# ❌ ĐỪNG DÙNG KEY NÀY
API_KEY = "your-secret-key-change-this"

# ✅ TẠO KEY MỚI
API_KEY = "abc123xyz789-your-real-secret-key"
```

**Cách tạo API key mạnh:**

```bash
# macOS/Linux
openssl rand -hex 32

# Output: a1b2c3d4e5f6...
# Copy và paste vào wrangler.toml
```

✅ **Checkpoint:** API key đã đổi

## 📋 BƯỚC 2: Deploy Backend (2 phút)

### 2.1 Deploy lên Cloudflare

```bash
cd backend
npm run deploy
```

**Output sẽ như này:**
```
⛅️ wrangler 3.x.x
------------------
Total Upload: xx.xx KiB / gzip: xx.xx KiB
Uploaded shorebird-backend (x.xx sec)
Published shorebird-backend (x.xx sec)
  https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev
Current Deployment ID: abc-123-def-456
```

✅ **Checkpoint:** Có production URL

### 2.2 Copy production URL

**Ví dụ URL:**
```
https://shorebird-backend.abc123.workers.dev
```

**Lưu URL này vào:**
1. File text để dùng sau
2. CLI config (.env)
3. Flutter client config

✅ **Checkpoint:** URL đã lưu

### 2.3 Test production API

**Test check endpoint:**
```bash
curl "https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev/api/check?app_id=com.test.app&version=1.0.0&platform=android&patch=0"
```

**Expected:**
```json
{"has_update":false}
```

✅ **Checkpoint:** Production API hoạt động

## 📋 BƯỚC 3: Setup Custom Domain (Optional - 10 phút)

### 3.1 Tại sao cần custom domain?

**workers.dev domain:**
- ✅ FREE
- ✅ HTTPS auto
- ❌ URL dài: `shorebird-backend.abc123.workers.dev`
- ❌ Không professional

**Custom domain:**
- ✅ URL ngắn: `api.yourapp.com`
- ✅ Professional
- ❌ Cần mua domain ($10-15/year)

### 3.2 Add custom domain

**Nếu bạn có domain trên Cloudflare:**

1. Vào Cloudflare Dashboard
2. Click "Workers & Pages"
3. Click worker "shorebird-backend"
4. Click tab "Settings"
5. Scroll xuống "Domains & Routes"
6. Click "Add Custom Domain"
7. Nhập: `api.yourapp.com`
8. Click "Add Custom Domain"
9. Đợi 1-2 phút

✅ **Checkpoint:** Custom domain active

**Test:**
```bash
curl "https://api.yourapp.com/api/check?app_id=com.test.app&version=1.0.0&platform=android&patch=0"
```

## 📋 BƯỚC 4: Update CLI Config (2 phút)

### 4.1 Update .env

**File:** `cli/.env`

```env
# Production URL
API_URL=https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev
# Or custom domain
# API_URL=https://api.yourapp.com

# Same API key as wrangler.toml
API_KEY=abc123xyz789-your-real-secret-key

# Your app ID
APP_ID=com.yourapp.name
```

✅ **Checkpoint:** CLI config updated

### 4.2 Test CLI với production

```bash
cd your-flutter-project
shorebird_custom_server patch android --dry-run
```

**Output:**
```
Running: shorebird patch android
✓ Patch created successfully
Finding patch file...
✓ Found: build/shorebird/patch.vmcode
Uploading to https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev...
[DRY RUN] Would upload patch
```

✅ **Checkpoint:** CLI kết nối được production

## 📋 BƯỚC 5: Update Flutter Client (2 phút)

### 5.1 Update client config

**File:** `lib/main.dart`

```dart
final updater = ShorebirdUpdater(
  // Production URL
  apiUrl: 'https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev',
  // Or custom domain
  // apiUrl: 'https://api.yourapp.com',
  
  appId: 'com.yourapp.name',
);
```

✅ **Checkpoint:** Client config updated

## 📋 BƯỚC 6: End-to-End Test (10 phút)

### 6.1 Deploy một patch thật

**Trong Flutter project:**

1. **Make a code change:**
```dart
// Before
Text('Version 1')

// After
Text('Version 2')
```

2. **Deploy patch:**
```bash
shorebird_custom_server patch android
```

**Expected output:**
```
Running: shorebird patch android
[shorebird output...]
✓ Patch created successfully

Finding patch file...
✓ Found: build/shorebird/patch.vmcode

Uploading to https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev...
✓ Upload complete

Patch Details:
  App ID: com.yourapp.name
  Platform: android
  Version: 1.0.0
  Patch Number: 1
  Download URL: https://pub-xxxxx.r2.dev/...
  Size: 1.2 MB

✓ Success! Patch deployed.
```

✅ **Checkpoint:** Patch uploaded thành công

### 6.2 Test client update

**Run app:**
```bash
flutter run
```

**App sẽ:**
1. Start
2. Check for update (call API)
3. Download patch
4. Install patch
5. Restart
6. Show "Version 2" ✅

✅ **Checkpoint:** Update hoạt động!

## 📊 Monitoring & Logs

### View Worker Logs

**Real-time logs:**
```bash
wrangler tail
```

**Output:**
```
[2024-01-01 12:00:00] POST /api/upload - 200 OK
[2024-01-01 12:00:05] GET /api/check - 200 OK
```

### View Analytics

1. Vào Cloudflare Dashboard
2. Click "Workers & Pages"
3. Click "shorebird-backend"
4. Click tab "Metrics"

**Xem được:**
- Requests per second
- Success rate
- Error rate
- CPU time
- Bandwidth

## 🔄 Update Backend Code

### Khi cần update backend:

```bash
# 1. Make changes to code
nano src/index.js

# 2. Test local
npm run dev

# 3. Deploy
npm run deploy
```

**Đơn giản vậy thôi!** Không cần:
- ❌ SSH vào server
- ❌ Restart service
- ❌ Update nginx
- ❌ Worry about downtime

Cloudflare tự động:
- ✅ Deploy code mới
- ✅ Zero downtime
- ✅ Rollback nếu lỗi

## 💾 Backup Strategy

### D1 Database Backup

**Export database:**
```bash
wrangler d1 export shorebird-db --output=backup.sql
```

**Restore database:**
```bash
wrangler d1 execute shorebird-db --file=backup.sql
```

**Automated backup (optional):**
- Setup GitHub Actions
- Export DB daily
- Commit to repo

### R2 Storage Backup

**R2 đã có:**
- ✅ Automatic replication
- ✅ 99.999999999% durability
- ✅ Không cần backup thêm

**Nếu muốn backup local:**
```bash
# Install rclone
brew install rclone  # macOS
# or
apt install rclone   # Linux

# Configure R2
rclone config

# Sync to local
rclone sync r2:shorebird-patches ./backup/
```

## 🔐 Security Checklist

### Production Security

- [ ] API key đã đổi (không dùng default)
- [ ] API key đủ mạnh (32+ characters)
- [ ] API key không commit vào git
- [ ] HTTPS enabled (auto với Cloudflare)
- [ ] CORS configured properly
- [ ] Rate limiting enabled (Workers có built-in)
- [ ] Input validation implemented
- [ ] File size limits set
- [ ] SHA256 verification working

### Cloudflare Security Features

**Tự động có:**
- ✅ DDoS protection
- ✅ WAF (Web Application Firewall)
- ✅ Bot protection
- ✅ SSL/TLS encryption
- ✅ Rate limiting

## 📈 Scaling

### Current Capacity (Free Tier)

**Workers:**
- 100,000 requests/day
- = ~1,157 requests/hour
- = ~19 requests/minute

**D1:**
- 5M reads/day
- 100K writes/day

**R2:**
- 10 GB storage
- Unlimited egress

**Đủ cho:**
- ~1000 apps
- ~10K users
- ~100 patches/day

### Nếu vượt Free Tier

**Workers Paid ($5/month):**
- 10M requests/month
- No daily limit

**D1 Paid ($5/month):**
- 25M reads/day
- 50M writes/month

**R2 Paid:**
- $0.015/GB storage
- Still FREE egress!

## ✅ Deployment Checklist

### Pre-Deployment
- [ ] Code tested locally
- [ ] API key changed
- [ ] D1 database ready
- [ ] R2 bucket ready

### Deployment
- [ ] `wrangler deploy` successful
- [ ] Production URL working
- [ ] Custom domain added (optional)

### Post-Deployment
- [ ] CLI config updated
- [ ] Client config updated
- [ ] End-to-end test passed
- [ ] Monitoring setup
- [ ] Backup strategy in place

### Documentation
- [ ] Production URL documented
- [ ] API key stored securely
- [ ] Team members informed
- [ ] Troubleshooting guide ready

## 🎉 Hoàn thành!

Backend của bạn giờ đã:
- ✅ Live trên production
- ✅ HTTPS tự động
- ✅ Scale tự động
- ✅ Global CDN
- ✅ Zero maintenance
- ✅ 100% FREE (trong free tier)

## 🔄 Next Steps

1. Monitor logs: `wrangler tail`
2. Check analytics trong dashboard
3. Test với real users
4. Document cho team
5. Enjoy! 🎉

## 📞 Troubleshooting

### Issue: Deploy failed

**Solution:**
```bash
# Check wrangler version
wrangler --version

# Update wrangler
npm install -g wrangler@latest

# Try again
npm run deploy
```

### Issue: API returns 500

**Solution:**
```bash
# View logs
wrangler tail

# Check for errors
# Fix code
# Deploy again
```

### Issue: Can't access production URL

**Solution:**
- Wait 1-2 minutes after deploy
- Check URL spelling
- Try incognito mode
- Check Cloudflare status page

## 🎓 Fresher Tips

1. **Đừng sợ deploy** - Cloudflare có rollback tự động
2. **Test local trước** - `npm run dev`
3. **Đọc logs** - `wrangler tail`
4. **Backup trước khi thay đổi lớn**
5. **Document mọi thứ**

**Chúc mừng! Bạn đã deploy thành công! 🚀**
