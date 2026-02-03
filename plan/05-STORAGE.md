# Storage & CDN Plan

## 🎯 Goal

Setup cloud storage for patch files with CDN delivery.

## 📊 Storage Options

### Option 1: Cloudflare R2 (Recommended)
**Pros:**
- S3-compatible API
- Zero egress fees
- Fast global CDN
- Free tier: 10GB storage, unlimited egress
- Easy setup

**Cons:**
- Requires Cloudflare account

### Option 2: AWS S3 + CloudFront
**Pros:**
- Industry standard
- Highly reliable
- Global infrastructure

**Cons:**
- Egress fees can be expensive
- More complex setup

### Option 3: Local Filesystem (Development Only)
**Pros:**
- Zero cost
- Simple setup
- Good for testing

**Cons:**
- Not scalable
- No CDN
- Single point of failure

## 🔧 Cloudflare R2 Setup

### Step 1: Create R2 Bucket

1. Go to Cloudflare Dashboard
2. Navigate to R2 Object Storage
3. Click "Create bucket"
4. Bucket name: `patches` (or your choice)
5. Location: Automatic
6. Click "Create bucket"

### Step 2: Generate API Credentials

1. Go to R2 → Manage R2 API Tokens
2. Click "Create API token"
3. Token name: `shorebird-backend`
4. Permissions: Object Read & Write
5. Bucket: `patches`
6. Click "Create API token"
7. Save credentials:
   - Access Key ID
   - Secret Access Key
   - Account ID

### Step 3: Enable Public Access

1. Go to bucket settings
2. Enable "Public URL access"
3. Copy public URL: `https://pub-xxxxx.r2.dev`
4. Or setup custom domain

### Step 4: Configure CORS (Optional)

```json
[
  {
    "AllowedOrigins": ["*"],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedHeaders": ["*"],
    "MaxAgeSeconds": 3600
  }
]
```

## 📁 File Organization

### Directory Structure

```
patches/
├── com.yourapp.name/
│   ├── android/
│   │   ├── 1.bin
│   │   ├── 2.bin
│   │   └── 3.bin
│   └── ios/
│       ├── 1.bin
│       └── 2.bin
├── com.otherapp.name/
│   └── android/
│       └── 1.bin
```

### File Naming Convention

```
{app_id}/{platform}/{patch_number}.bin

Examples:
- com.yourapp.name/android/1.bin
- com.yourapp.name/android/2.bin
- com.yourapp.name/ios/1.bin
```

### Why This Structure?
- Easy to organize by app
- Easy to separate platforms
- Sequential patch numbers
- Simple to query
- Easy to delete old patches

## 💻 Storage Implementation (storage.js)

### Initialize S3 Client

```javascript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import fs from 'fs';
import path from 'path';

class Storage {
  constructor(config) {
    this.client = new S3Client({
      region: 'auto',
      endpoint: `https://${config.accountId}.r2.cloudflarestorage.com`,
      credentials: {
        accessKeyId: config.accessKeyId,
        secretAccessKey: config.secretAccessKey,
      },
    });
    
    this.bucket = config.bucketName;
    this.publicUrl = config.publicUrl;
  }
}
```

### Upload File

```javascript
async upload(filePath, metadata) {
  // Read file
  const fileContent = fs.readFileSync(filePath);
  
  // Generate key
  const key = `${metadata.app_id}/${metadata.platform}/${metadata.patch_number}.bin`;
  
  // Upload to R2
  const command = new PutObjectCommand({
    Bucket: this.bucket,
    Key: key,
    Body: fileContent,
    ContentType: 'application/octet-stream',
    Metadata: {
      'app-id': metadata.app_id,
      'platform': metadata.platform,
      'patch-number': metadata.patch_number.toString(),
      'version': metadata.version,
      'sha256': metadata.sha256,
    },
  });
  
  await this.client.send(command);
  
  // Return public URL
  return `${this.publicUrl}/${key}`;
}
```

### Delete File (Optional)

```javascript
import { DeleteObjectCommand } from '@aws-sdk/client-s3';

async delete(app_id, platform, patch_number) {
  const key = `${app_id}/${platform}/${patch_number}.bin`;
  
  const command = new DeleteObjectCommand({
    Bucket: this.bucket,
    Key: key,
  });
  
  await this.client.send(command);
}
```

### List Files (Optional)

```javascript
import { ListObjectsV2Command } from '@aws-sdk/client-s3';

async list(app_id, platform) {
  const prefix = `${app_id}/${platform}/`;
  
  const command = new ListObjectsV2Command({
    Bucket: this.bucket,
    Prefix: prefix,
  });
  
  const response = await this.client.send(command);
  return response.Contents || [];
}
```

## 🔐 Security Best Practices

### Access Control

**R2 Bucket:**
- Private by default
- Public read access for patch files
- No public write access
- Use API tokens with minimal permissions

**API Tokens:**
- Separate tokens for different environments
- Rotate tokens regularly
- Never commit tokens to git
- Use environment variables

### File Validation

**Before Upload:**
- Verify file size < 50MB
- Verify file extension (.vmcode, .bin)
- Calculate SHA256 hash
- Validate metadata

**After Upload:**
- Verify file exists in R2
- Verify public URL is accessible
- Store hash in database

## 📊 CDN Configuration

### Cloudflare R2 (Built-in CDN)

**Automatic:**
- Global edge network
- Automatic caching
- DDoS protection
- No extra configuration needed

**Custom Domain (Optional):**
1. Add CNAME record: `cdn.yourapp.com` → `pub-xxxxx.r2.dev`
2. Enable SSL
3. Update `R2_PUBLIC_URL` in .env

### Cache Headers

```javascript
const command = new PutObjectCommand({
  Bucket: this.bucket,
  Key: key,
  Body: fileContent,
  ContentType: 'application/octet-stream',
  CacheControl: 'public, max-age=31536000, immutable',
});
```

**Why immutable?**
- Patch files never change
- Aggressive caching is safe
- Reduces bandwidth
- Faster downloads

## 🧪 Testing

### Test Upload

```javascript
// test-upload.js
import Storage from './storage.js';
import fs from 'fs';
import crypto from 'crypto';

async function test() {
  const storage = new Storage({
    accountId: process.env.R2_ACCOUNT_ID,
    accessKeyId: process.env.R2_ACCESS_KEY_ID,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY,
    bucketName: process.env.R2_BUCKET_NAME,
    publicUrl: process.env.R2_PUBLIC_URL,
  });
  
  // Create test file
  const testFile = '/tmp/test-patch.bin';
  fs.writeFileSync(testFile, 'test content');
  
  // Calculate hash
  const content = fs.readFileSync(testFile);
  const hash = crypto.createHash('sha256').update(content).digest('hex');
  
  // Upload
  const url = await storage.upload(testFile, {
    app_id: 'com.test.app',
    platform: 'android',
    patch_number: 1,
    version: '1.0.0',
    sha256: hash,
  });
  
  console.log('Uploaded to:', url);
  
  // Verify URL is accessible
  const response = await fetch(url);
  console.log('Status:', response.status);
  console.log('Content:', await response.text());
}

test();
```

### Test Download

```bash
# Test public URL
curl https://pub-xxxxx.r2.dev/com.test.app/android/1.bin

# Should return file content
```

## 💰 Cost Estimation

### Cloudflare R2 Free Tier
- Storage: 10 GB/month
- Class A operations: 1M/month (writes)
- Class B operations: 10M/month (reads)
- Egress: Unlimited (FREE!)

### Example Usage
- 100 apps
- 10 patches per app
- 2 MB per patch
- 10K users
- 1 update per user per month

**Storage:**
- 100 apps × 10 patches × 2 MB = 2 GB
- Cost: FREE (under 10 GB)

**Operations:**
- Uploads: 1000/month (Class A)
- Downloads: 10K/month (Class B)
- Cost: FREE (under limits)

**Egress:**
- 10K users × 2 MB = 20 GB
- Cost: FREE (R2 has no egress fees!)

**Total: $0/month**

### Scaling Beyond Free Tier

**Storage:** $0.015/GB/month
- 100 GB = $1.50/month

**Class A Operations:** $4.50/million
- 10M writes = $45/month

**Class B Operations:** $0.36/million
- 100M reads = $36/month

## 🔄 Backup Strategy

### Automatic Backups

**Option 1: R2 Replication**
- Enable bucket replication
- Replicate to second bucket
- Automatic and real-time

**Option 2: Periodic Sync**
```bash
# Sync to local backup
rclone sync r2:patches /backup/patches

# Run daily via cron
0 2 * * * rclone sync r2:patches /backup/patches
```

**Option 3: Database Backup**
- SQLite database contains all metadata
- Backup database daily
- Can rebuild from database + R2

## ✅ Success Criteria

- [ ] R2 bucket created
- [ ] API credentials generated
- [ ] Public access enabled
- [ ] Upload function works
- [ ] Files accessible via public URL
- [ ] CDN delivers files globally
- [ ] Security configured properly
- [ ] Backup strategy in place

## 🔄 Next Steps

After storage is setup:
1. Test upload from backend
2. Test download from client
3. Verify CDN performance
4. Setup monitoring
