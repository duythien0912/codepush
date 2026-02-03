# 01 - BACKEND IMPLEMENTATION (Cloudflare Workers)

**Time**: 45 minutes  
**Difficulty**: Medium  
**Platform**: Cloudflare (Workers + D1 + R2)

---

## Overview

Build a serverless backend on Cloudflare that:
- Receives patch uploads from CLI
- Stores patches in R2 (object storage)
- Tracks metadata in D1 (SQLite database)
- Serves update checks to Flutter apps

**Result**: Live API at `https://your-worker.workers.dev`

**CRITICAL**: API must match Shorebird's format exactly:
- Endpoint: `POST /api/v1/patches/check` (not GET /api/check)
- Request/response JSON must match Shorebird schema

---

## Prerequisites

✅ Cloudflare account created  
✅ Node.js 18+ installed (`node --version`)  
✅ Terminal open  
✅ 45 minutes available

---

## Architecture

```
POST /api/v1/patches/check  ← Flutter app checks for updates
POST /api/upload             ← CLI uploads patches

D1 Database: patches table (metadata)
R2 Bucket: patch files (binary)
```

---

## Implementation Steps

Due to length constraints, I'll provide the corrected Worker code with proper API endpoints. The setup steps (Wrangler, D1, R2) remain the same as the original doc.

### Corrected Worker Code (src/index.js)

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

// Shorebird-compatible patch check
async function handlePatchCheck(request, env, corsHeaders) {
  try {
    const body = await request.json();
    const { app_id, release_version, platform, arch, channel } = body;

    if (!app_id || !release_version || !platform || !arch) {
      return jsonResponse({ error: 'Missing required fields' }, 400, corsHeaders);
    }

    // Query latest patch
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

// CLI upload endpoint
async function handleUpload(request, env, corsHeaders) {
  try {
    // Verify API key
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

    // Calculate SHA256
    const arrayBuffer = await file.arrayBuffer();
    const hashBuffer = await crypto.subtle.digest('SHA-256', arrayBuffer);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    const hash = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');

    // Upload to R2
    const r2Key = `${app_id}/${platform}/${arch}/${release_version}/${patch_number}`;
    await env.R2.put(r2Key, arrayBuffer);

    const download_url = `${env.R2_PUBLIC_URL}/${r2Key}`;

    // Save to D1
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

### Corrected Database Schema (schema.sql)

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

CREATE TABLE api_keys (
  key TEXT PRIMARY KEY,
  app_id TEXT NOT NULL,
  created_at INTEGER NOT NULL
);
```

---

## Testing

### Test Patch Check (Shorebird-compatible)

```bash
curl -X POST https://your-worker.workers.dev/api/v1/patches/check \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "com.example.app",
    "release_version": "1.0.0+1",
    "platform": "android",
    "arch": "aarch64",
    "channel": "stable",
    "client_id": "test-uuid"
  }'
```

**Expected Response (no patches)**:
```json
{
  "patch_available": false,
  "patch": null,
  "rolled_back_patch_numbers": []
}
```

**Expected Response (patch exists)**:
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

### Test Upload

```bash
echo "test" > test.bin

curl -X POST https://your-worker.workers.dev/api/upload \
  -H "X-API-Key: YOUR-API-KEY" \
  -F "file=@test.bin" \
  -F "app_id=com.example.app" \
  -F "release_version=1.0.0+1" \
  -F "platform=android" \
  -F "arch=aarch64" \
  -F "patch_number=1"
```

---

## Key Changes from Original Plan

1. **API Endpoint**: `POST /api/v1/patches/check` (was `GET /api/check`)
2. **Request Format**: JSON body (was query parameters)
3. **Response Format**: Shorebird-compatible JSON schema
4. **Database Schema**: Added `arch` column, updated constraints
5. **R2 Path**: Includes arch: `{app_id}/{platform}/{arch}/{version}/{number}`

---

## Summary

**What you built**:
- ✅ Shorebird-compatible API endpoint
- ✅ D1 database with proper schema
- ✅ R2 storage with correct path structure
- ✅ CLI upload endpoint with authentication

**Time spent**: ~45 minutes

**Next**: Build CLI tool (02-CLI.md)

---

## Troubleshooting

**"Invalid JSON" error**: Make sure Content-Type is `application/json`

**"Missing required fields"**: Check all fields in request body

**"Database constraint failed"**: Duplicate patch number, increment it

**"R2 upload failed"**: Check bucket name in wrangler.toml

---

For complete step-by-step setup instructions (Wrangler installation, D1 creation, R2 setup), refer to the original 01-BACKEND.md file. This document focuses on the corrected API implementation.
