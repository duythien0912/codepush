# Backend Implementation Plan (Cloudflare Workers)

## 🎯 Mục tiêu

Tạo backend API hoàn toàn trên Cloudflare (Workers + D1 + R2) - KHÔNG CẦN VPS!

## 🌟 Tại sao Cloudflare?

✅ **100% FREE** (trong free tier)
✅ **Không cần VPS** - Serverless
✅ **Auto SSL** - HTTPS tự động
✅ **Auto scaling** - Tự động scale
✅ **Global CDN** - Nhanh toàn cầu
✅ **Zero DevOps** - Chỉ cần `wrangler deploy`

## 📊 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloudflare Workers                       │
│                   (Serverless Backend)                      │
│                                                             │
│  POST /api/upload  ──→  Upload to R2  ──→  Save to D1     │
│  GET  /api/check   ──→  Query D1      ──→  Return JSON    │
└─────────────────────────────────────────────────────────────┘
         │                    │                    │
         ↓                    ↓                    ↓
    ┌─────────┐         ┌─────────┐         ┌─────────┐
    │ Workers │         │   R2    │         │   D1    │
    │  Code   │         │ Storage │         │Database │
    └─────────┘         └─────────┘         └─────────┘
```

## 📁 File Structure

```
backend/
├── wrangler.toml           # Cloudflare config
├── src/
│   ├── index.js           # Main worker
│   ├── upload.js          # Upload handler
│   ├── check.js           # Check handler
│   └── utils.js           # Helper functions
├── schema.sql             # D1 database schema
├── package.json           # Dependencies
└── .dev.vars              # Local env variables
```

## 🚀 BƯỚC 1: Setup Cloudflare Account (5 phút)

### 1.1 Tạo tài khoản Cloudflare

1. Vào https://dash.cloudflare.com/sign-up
2. Nhập email và password
3. Verify email
4. Login vào dashboard

✅ **Checkpoint:** Bạn đã vào được Cloudflare Dashboard

### 1.2 Install Wrangler CLI

**macOS/Linux:**
```bash
npm install -g wrangler
```

**Windows:**
```bash
npm install -g wrangler
```

**Verify:**
```bash
wrangler --version
# Output: ⛅️ wrangler 3.x.x
```

✅ **Checkpoint:** Command `wrangler --version` chạy được

### 1.3 Login Wrangler

```bash
wrangler login
```

- Browser sẽ mở
- Click "Allow"
- Quay lại terminal

✅ **Checkpoint:** Terminal hiện "Successfully logged in"

## 🚀 BƯỚC 2: Create Workers Project (5 phút)

### 2.1 Tạo project

```bash
# Tạo thư mục
mkdir shorebird-backend
cd shorebird-backend

# Init project
npm init -y

# Install dependencies
npm install
```

### 2.2 Tạo wrangler.toml

**File:** `wrangler.toml`

```toml
name = "shorebird-backend"
main = "src/index.js"
compatibility_date = "2024-01-01"

# D1 Database binding
[[d1_databases]]
binding = "DB"
database_name = "shorebird-db"
database_id = "YOUR_DATABASE_ID"  # Sẽ update sau

# R2 Storage binding
[[r2_buckets]]
binding = "R2"
bucket_name = "shorebird-patches"

# Environment variables
[vars]
API_KEY = "your-secret-key-change-this"
```

✅ **Checkpoint:** File `wrangler.toml` đã tạo

### 2.3 Tạo package.json

**File:** `package.json`

```json
{
  "name": "shorebird-backend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler deploy"
  },
  "devDependencies": {
    "wrangler": "^3.0.0"
  }
}
```

✅ **Checkpoint:** File `package.json` đã tạo

## 🚀 BƯỚC 3: Setup D1 Database (5 phút)

### 3.1 Tạo D1 database

```bash
wrangler d1 create shorebird-db
```

**Output sẽ như này:**
```
✅ Successfully created DB 'shorebird-db'

[[d1_databases]]
binding = "DB"
database_name = "shorebird-db"
database_id = "abc-123-def-456"
```

### 3.2 Copy database_id

- Copy dòng `database_id = "abc-123-def-456"`
- Paste vào `wrangler.toml` (thay thế YOUR_DATABASE_ID)

✅ **Checkpoint:** `wrangler.toml` có database_id thật

### 3.3 Tạo database schema

**File:** `schema.sql`

```sql
-- Bảng lưu thông tin patches
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

-- Index để query nhanh
CREATE INDEX idx_patches_lookup 
ON patches(app_id, platform, active, patch_number);
```

### 3.4 Apply schema

```bash
wrangler d1 execute shorebird-db --file=schema.sql
```

**Output:**
```
🌀 Executing on shorebird-db:
✅ Successfully executed SQL
```

✅ **Checkpoint:** Database đã có bảng `patches`

### 3.5 Test database

```bash
wrangler d1 execute shorebird-db --command="SELECT * FROM patches"
```

**Output:**
```
┌────┐
│ id │
├────┤
└────┘
```

✅ **Checkpoint:** Query chạy được (kết quả rỗng là bình thường)

## 🚀 BƯỚC 4: Setup R2 Storage (3 phút)

### 4.1 Tạo R2 bucket

```bash
wrangler r2 bucket create shorebird-patches
```

**Output:**
```
✅ Created bucket 'shorebird-patches'
```

✅ **Checkpoint:** Bucket đã tạo

### 4.2 Enable public access

1. Vào Cloudflare Dashboard
2. Click "R2" ở sidebar
3. Click bucket "shorebird-patches"
4. Click "Settings"
5. Scroll xuống "Public access"
6. Click "Allow Access"
7. Copy "Public R2.dev Bucket URL"
   - Ví dụ: `https://pub-abc123.r2.dev`

✅ **Checkpoint:** Có public URL của bucket

### 4.3 Update wrangler.toml

Thêm vào `wrangler.toml`:

```toml
[vars]
API_KEY = "your-secret-key-change-this"
R2_PUBLIC_URL = "https://pub-abc123.r2.dev"  # URL vừa copy
```

✅ **Checkpoint:** `wrangler.toml` có R2_PUBLIC_URL

## 🚀 BƯỚC 5: Implement Worker Code (15 phút)

### 5.1 Tạo main worker

**File:** `src/index.js`

```javascript
import { handleUpload } from './upload.js';
import { handleCheck } from './check.js';

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // CORS headers
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, X-API-Key',
    };
    
    // Handle OPTIONS (CORS preflight)
    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }
    
    // Route: POST /api/upload
    if (url.pathname === '/api/upload' && request.method === 'POST') {
      const response = await handleUpload(request, env);
      return new Response(response.body, {
        status: response.status,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' }
      });
    }
    
    // Route: GET /api/check
    if (url.pathname === '/api/check' && request.method === 'GET') {
      const response = await handleCheck(request, env);
      return new Response(response.body, {
        status: response.status,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' }
      });
    }
    
    // 404
    return new Response('Not Found', { status: 404, headers: corsHeaders });
  }
};
```

**Giải thích cho Fresher:**
- `export default` = Export worker chính
- `fetch()` = Function xử lý mọi request
- `url.pathname` = Đường dẫn (vd: /api/upload)
- `request.method` = GET, POST, etc.
- `corsHeaders` = Cho phép browser gọi API

✅ **Checkpoint:** File `src/index.js` đã tạo

### 5.2 Implement upload handler

**File:** `src/upload.js`

```javascript
export async function handleUpload(request, env) {
  try {
    // 1. Check API key
    const apiKey = request.headers.get('X-API-Key');
    if (apiKey !== env.API_KEY) {
      return {
        status: 403,
        body: JSON.stringify({ error: 'Invalid API key' })
      };
    }
    
    // 2. Parse form data
    const formData = await request.formData();
    const file = formData.get('file');
    const app_id = formData.get('app_id');
    const platform = formData.get('platform');
    const version = formData.get('version');
    const patch_number = parseInt(formData.get('patch_number'));
    
    // 3. Validate
    if (!file || !app_id || !platform || !version || !patch_number) {
      return {
        status: 400,
        body: JSON.stringify({ error: 'Missing required fields' })
      };
    }
    
    // 4. Calculate SHA256
    const arrayBuffer = await file.arrayBuffer();
    const hashBuffer = await crypto.subtle.digest('SHA-256', arrayBuffer);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    const sha256 = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    
    // 5. Upload to R2
    const r2Key = `${app_id}/${platform}/${patch_number}.bin`;
    await env.R2.put(r2Key, arrayBuffer);
    
    // 6. Generate public URL
    const download_url = `${env.R2_PUBLIC_URL}/${r2Key}`;
    
    // 7. Save to D1
    await env.DB.prepare(`
      INSERT INTO patches (app_id, platform, patch_number, version, download_url, sha256, size_bytes)
      VALUES (?, ?, ?, ?, ?, ?, ?)
    `).bind(
      app_id,
      platform,
      patch_number,
      version,
      download_url,
      sha256,
      arrayBuffer.byteLength
    ).run();
    
    // 8. Return success
    return {
      status: 200,
      body: JSON.stringify({
        success: true,
        patch: {
          app_id,
          platform,
          patch_number,
          version,
          download_url,
          sha256,
          size_bytes: arrayBuffer.byteLength
        }
      })
    };
    
  } catch (error) {
    return {
      status: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
}
```

**Giải thích từng bước:**
1. Check API key có đúng không
2. Lấy data từ form (file + metadata)
3. Validate: Có đủ field không?
4. Tính SHA256 hash của file
5. Upload file lên R2
6. Tạo public URL
7. Lưu metadata vào D1
8. Trả về success

✅ **Checkpoint:** File `src/upload.js` đã tạo

### 5.3 Implement check handler

**File:** `src/check.js`

```javascript
export async function handleCheck(request, env) {
  try {
    // 1. Parse query parameters
    const url = new URL(request.url);
    const app_id = url.searchParams.get('app_id');
    const platform = url.searchParams.get('platform');
    const version = url.searchParams.get('version');
    const current_patch = parseInt(url.searchParams.get('patch') || '0');
    
    // 2. Validate
    if (!app_id || !platform || !version) {
      return {
        status: 400,
        body: JSON.stringify({ error: 'Missing required parameters' })
      };
    }
    
    // 3. Query D1 for latest patch
    const result = await env.DB.prepare(`
      SELECT patch_number, download_url, sha256, size_bytes
      FROM patches
      WHERE app_id = ?
        AND platform = ?
        AND version = ?
        AND patch_number > ?
        AND active = 1
      ORDER BY patch_number DESC
      LIMIT 1
    `).bind(app_id, platform, version, current_patch).first();
    
    // 4. Return result
    if (!result) {
      return {
        status: 200,
        body: JSON.stringify({ has_update: false })
      };
    }
    
    return {
      status: 200,
      body: JSON.stringify({
        has_update: true,
        patch: {
          patch_number: result.patch_number,
          download_url: result.download_url,
          sha256: result.sha256,
          size_bytes: result.size_bytes
        }
      })
    };
    
  } catch (error) {
    return {
      status: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
}
```

**Giải thích từng bước:**
1. Lấy parameters từ URL (?app_id=xxx&platform=yyy)
2. Validate: Có đủ parameters không?
3. Query database tìm patch mới nhất
4. Trả về kết quả (có update hoặc không)

✅ **Checkpoint:** File `src/check.js` đã tạo

## 🚀 BƯỚC 6: Test Local (5 phút)

### 6.1 Start dev server

```bash
npm run dev
```

**Output:**
```
⎔ Starting local server...
[wrangler:inf] Ready on http://localhost:8787
```

✅ **Checkpoint:** Server chạy ở http://localhost:8787

### 6.2 Test check endpoint

**Mở terminal mới, chạy:**

```bash
curl "http://localhost:8787/api/check?app_id=com.test.app&version=1.0.0&platform=android&patch=0"
```

**Expected output:**
```json
{"has_update":false}
```

✅ **Checkpoint:** API trả về JSON

### 6.3 Test upload endpoint

**Tạo file test:**
```bash
echo "test content" > test.bin
```

**Upload:**
```bash
curl -X POST http://localhost:8787/api/upload \
  -H "X-API-Key: your-secret-key-change-this" \
  -F "file=@test.bin" \
  -F "app_id=com.test.app" \
  -F "platform=android" \
  -F "version=1.0.0" \
  -F "patch_number=1"
```

**Expected output:**
```json
{
  "success": true,
  "patch": {
    "app_id": "com.test.app",
    "platform": "android",
    "patch_number": 1,
    ...
  }
}
```

✅ **Checkpoint:** Upload thành công

### 6.4 Test check again

```bash
curl "http://localhost:8787/api/check?app_id=com.test.app&version=1.0.0&platform=android&patch=0"
```

**Expected output:**
```json
{
  "has_update": true,
  "patch": {
    "patch_number": 1,
    ...
  }
}
```

✅ **Checkpoint:** Có update rồi!

## 🚀 BƯỚC 7: Deploy to Production (2 phút)

### 7.1 Deploy

```bash
npm run deploy
```

**Output:**
```
✨ Built successfully
✨ Uploaded successfully
✨ Deployed to https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev
```

✅ **Checkpoint:** Có production URL

### 7.2 Test production

**Replace localhost với production URL:**

```bash
curl "https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev/api/check?app_id=com.test.app&version=1.0.0&platform=android&patch=0"
```

✅ **Checkpoint:** Production API hoạt động!

## ✅ Success Criteria

- [ ] Cloudflare account created
- [ ] Wrangler CLI installed
- [ ] D1 database created and schema applied
- [ ] R2 bucket created with public access
- [ ] Worker code implemented
- [ ] Local testing passed
- [ ] Deployed to production
- [ ] Production testing passed

## 🎉 Hoàn thành!

Backend của bạn giờ đã:
- ✅ Chạy trên Cloudflare Workers (serverless)
- ✅ Có HTTPS tự động
- ✅ Scale tự động
- ✅ 100% FREE (trong free tier)
- ✅ Global CDN
- ✅ Zero maintenance

## 📝 Production URL

Lưu lại URL này để dùng cho CLI và Client:
```
https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev
```

## 🔄 Next Steps

Tiếp theo: Implement CLI tool (02-CLI.md)
