# Phase 1: Backend Implementation (30 minutes)

## 🎯 What You'll Build

A serverless backend on Cloudflare that:
- Receives patch uploads from CLI
- Stores patches in R2 (like AWS S3)
- Tracks metadata in D1 (SQLite database)
- Serves update checks to apps

**Result:** Live API at `https://your-worker.workers.dev`

## 📋 Prerequisites Check

Before starting, verify:
- [ ] Cloudflare account created (Step 0.4 from START-HERE.md)
- [ ] Node.js installed (`node --version` shows v18+)
- [ ] Terminal open
- [ ] 30 minutes available

**If any unchecked:** Go back to START-HERE.md

---

## STEP 1: Install Wrangler CLI

### Step 1.1: Install Wrangler

- **Goal:** Install Cloudflare's command-line tool
- **Input:** None
- **Tool/Command:** 
  ```bash
  npm install -g wrangler
  ```
- **Detailed Actions:**
  1. Open terminal
  2. Copy command above
  3. Paste into terminal
  4. Press Enter
  5. Wait for installation (30-60 seconds)
  
- **Output:**
  ```
  added 1 package in 15s
  ```
  
- **How to verify:**
  ```bash
  wrangler --version
  ```
  Should show: `⛅️ wrangler 3.x.x`
  
- **What can go wrong:**
  - "npm: command not found" → Node.js not installed
  - "Permission denied" → Use `sudo npm install -g wrangler`
  - Installation hangs → Check internet connection
  
- **If stuck, do this:**
  ```bash
  # Try with sudo
  sudo npm install -g wrangler
  
  # Or install locally
  npm install wrangler
  npx wrangler --version
  ```

### Step 1.2: Login to Cloudflare

- **Goal:** Connect Wrangler to your Cloudflare account
- **Input:** Cloudflare account credentials
- **Tool/Command:**
  ```bash
  wrangler login
  ```
- **Detailed Actions:**
  1. Run command above
  2. Browser opens automatically
  3. Click "Allow" button
  4. Return to terminal
  
- **Output:**
  ```
  Successfully logged in.
  ```
  
- **How to verify:**
  ```bash
  wrangler whoami
  ```
  Should show your email
  
- **What can go wrong:**
  - Browser doesn't open → Copy URL from terminal, paste in browser
  - "Not logged in" → Try `wrangler login` again
  - "Permission denied" → Check Cloudflare account is verified
  
- **If stuck, do this:**
  ```bash
  # Logout and try again
  wrangler logout
  wrangler login
  ```

---

## STEP 2: Create Project

### Step 2.1: Create Project Folder

- **Goal:** Create folder for backend code
- **Input:** None
- **Tool/Command:**
  ```bash
  mkdir -p ~/Desktop/codepush/backend
  cd ~/Desktop/codepush/backend
  ```
- **Detailed Actions:**
  1. Copy first command
  2. Paste in terminal
  3. Press Enter
  4. Copy second command
  5. Paste in terminal
  6. Press Enter
  
- **Output:** (none - silent success)
  
- **How to verify:**
  ```bash
  pwd
  ```
  Should show: `/Users/[your-name]/Desktop/codepush/backend`
  
- **What can go wrong:**
  - "No such file or directory" → Parent folder doesn't exist
  - Wrong location → Check `pwd` output
  
- **If stuck, do this:**
  ```bash
  # Create full path
  mkdir -p ~/Desktop/codepush/backend
  cd ~/Desktop/codepush/backend
  pwd
  ```

### Step 2.2: Initialize Node Project

- **Goal:** Create package.json file
- **Input:** None
- **Tool/Command:**
  ```bash
  npm init -y
  ```
- **Detailed Actions:**
  1. Make sure you're in backend folder (`pwd` to check)
  2. Run command above
  3. Wait 1 second
  
- **Output:**
  ```json
  {
    "name": "backend",
    "version": "1.0.0",
    ...
  }
  ```
  
- **How to verify:**
  ```bash
  ls -la
  ```
  Should see: `package.json`
  
- **What can go wrong:**
  - File not created → Check you're in correct folder
  - "npm: command not found" → Node.js not installed
  
- **If stuck, do this:**
  ```bash
  # Check location
  pwd
  
  # Should be in backend folder
  # If not:
  cd ~/Desktop/codepush/backend
  npm init -y
  ```

### Step 2.3: Create wrangler.toml

- **Goal:** Configure Cloudflare Workers project
- **Input:** Configuration file
- **Tool/Command:** Create file with text editor
- **Detailed Actions:**
  1. Create file: `touch wrangler.toml`
  2. Open in editor: `nano wrangler.toml` (or use VS Code)
  3. Copy content below
  4. Paste into file
  5. Save (Ctrl+O, Enter, Ctrl+X in nano)
  
- **File content:**
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
  API_KEY = "CHANGE-THIS-KEY"
  R2_PUBLIC_URL = "PLACEHOLDER"
  ```
  
- **Output:** File created
  
- **How to verify:**
  ```bash
  cat wrangler.toml
  ```
  Should show the content you pasted
  
- **What can go wrong:**
  - File empty → Paste didn't work, try again
  - Syntax error → Copy exactly as shown
  
- **If stuck, do this:**
  ```bash
  # Delete and recreate
  rm wrangler.toml
  nano wrangler.toml
  # Paste content again
  ```

---

## STEP 3: Create D1 Database

### Step 3.1: Create Database

- **Goal:** Create SQLite database on Cloudflare
- **Input:** None
- **Tool/Command:**
  ```bash
  wrangler d1 create shorebird-db
  ```
- **Detailed Actions:**
  1. Run command above
  2. Wait 5-10 seconds
  3. Read output carefully
  
- **Output:**
  ```
  ✅ Successfully created DB 'shorebird-db'

  [[d1_databases]]
  binding = "DB"
  database_name = "shorebird-db"
  database_id = "abc-123-def-456"
  ```
  
- **How to verify:**
  ```bash
  wrangler d1 list
  ```
  Should show: `shorebird-db`
  
- **What can go wrong:**
  - "Not logged in" → Run `wrangler login` again
  - "Database already exists" → That's OK, continue
  - Network error → Check internet connection
  
- **If stuck, do this:**
  ```bash
  # List existing databases
  wrangler d1 list
  
  # If shorebird-db exists, get its ID:
  wrangler d1 info shorebird-db
  ```

### Step 3.2: Update wrangler.toml with Database ID

- **Goal:** Add database ID to config
- **Input:** Database ID from Step 3.1 output
- **Tool/Command:** Edit wrangler.toml
- **Detailed Actions:**
  1. Copy the `database_id` from Step 3.1 output
     Example: `abc-123-def-456`
  2. Open wrangler.toml: `nano wrangler.toml`
  3. Find line: `database_id = "PLACEHOLDER"`
  4. Replace `PLACEHOLDER` with your actual ID
  5. Save file (Ctrl+O, Enter, Ctrl+X)
  
- **Before:**
  ```toml
  database_id = "PLACEHOLDER"
  ```
  
- **After:**
  ```toml
  database_id = "abc-123-def-456"
  ```
  
- **How to verify:**
  ```bash
  cat wrangler.toml | grep database_id
  ```
  Should show your actual ID (not "PLACEHOLDER")
  
- **What can go wrong:**
  - Forgot to save → Open and save again
  - Wrong ID → Copy from `wrangler d1 list` output
  - Typo → ID should be format: `xxx-xxx-xxx-xxx`
  
- **If stuck, do this:**
  ```bash
  # Get database ID again
  wrangler d1 list
  
  # Copy the ID
  # Edit file
  nano wrangler.toml
  # Replace PLACEHOLDER with ID
  ```

### Step 3.3: Create Database Schema

- **Goal:** Create tables in database
- **Input:** SQL schema file
- **Tool/Command:** Create and execute SQL file
- **Detailed Actions:**
  1. Create file: `touch schema.sql`
  2. Open: `nano schema.sql`
  3. Copy SQL below
  4. Paste into file
  5. Save (Ctrl+O, Enter, Ctrl+X)
  6. Execute: `wrangler d1 execute shorebird-db --file=schema.sql`
  
- **File content (schema.sql):**
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
  
- **Output:**
  ```
  🌀 Executing on shorebird-db:
  ✅ Successfully executed SQL
  ```
  
- **How to verify:**
  ```bash
  wrangler d1 execute shorebird-db --command="SELECT name FROM sqlite_master WHERE type='table'"
  ```
  Should show: `patches`
  
- **What can go wrong:**
  - "Syntax error" → Copy SQL exactly as shown
  - "Table already exists" → That's OK, continue
  - "Database not found" → Check database_id in wrangler.toml
  
- **If stuck, do this:**
  ```bash
  # Delete schema.sql and recreate
  rm schema.sql
  nano schema.sql
  # Paste SQL again
  # Execute again
  wrangler d1 execute shorebird-db --file=schema.sql
  ```

---

## STEP 4: Create R2 Bucket

### Step 4.1: Create Bucket

- **Goal:** Create storage for patch files
- **Input:** None
- **Tool/Command:**
  ```bash
  wrangler r2 bucket create shorebird-patches
  ```
- **Detailed Actions:**
  1. Run command above
  2. Wait 5 seconds
  
- **Output:**
  ```
  ✅ Created bucket 'shorebird-patches'
  ```
  
- **How to verify:**
  ```bash
  wrangler r2 bucket list
  ```
  Should show: `shorebird-patches`
  
- **What can go wrong:**
  - "Bucket already exists" → That's OK, continue
  - "Not logged in" → Run `wrangler login`
  - Network error → Check internet
  
- **If stuck, do this:**
  ```bash
  # List buckets
  wrangler r2 bucket list
  
  # If shorebird-patches exists, continue to next step
  ```

### Step 4.2: Enable Public Access

- **Goal:** Allow public downloads from bucket
- **Input:** None
- **Tool/Command:** Cloudflare Dashboard
- **Detailed Actions:**
  1. Go to https://dash.cloudflare.com
  2. Click "R2" in left sidebar
  3. Click "shorebird-patches" bucket
  4. Click "Settings" tab
  5. Scroll to "Public access"
  6. Click "Allow Access" button
  7. Copy the "Public R2.dev Bucket URL"
     Example: `https://pub-abc123xyz.r2.dev`
  
- **Output:** Public URL like `https://pub-abc123xyz.r2.dev`
  
- **How to verify:**
  Open the URL in browser - should show empty bucket (not error)
  
- **What can go wrong:**
  - Can't find R2 → Check left sidebar
  - No "Allow Access" button → Already enabled
  - 404 error → Wrong URL
  
- **If stuck, do this:**
  1. Go to https://dash.cloudflare.com/r2
  2. Find your bucket
  3. Look for public URL in settings

### Step 4.3: Update wrangler.toml with R2 URL

- **Goal:** Add R2 public URL to config
- **Input:** Public URL from Step 4.2
- **Tool/Command:** Edit wrangler.toml
- **Detailed Actions:**
  1. Copy public URL from Step 4.2
  2. Open: `nano wrangler.toml`
  3. Find: `R2_PUBLIC_URL = "PLACEHOLDER"`
  4. Replace `PLACEHOLDER` with your URL
  5. Save (Ctrl+O, Enter, Ctrl+X)
  
- **Before:**
  ```toml
  R2_PUBLIC_URL = "PLACEHOLDER"
  ```
  
- **After:**
  ```toml
  R2_PUBLIC_URL = "https://pub-abc123xyz.r2.dev"
  ```
  
- **How to verify:**
  ```bash
  cat wrangler.toml | grep R2_PUBLIC_URL
  ```
  Should show your actual URL (not "PLACEHOLDER")
  
- **What can go wrong:**
  - Forgot to save → Open and save again
  - Wrong URL → Should start with `https://pub-`
  - Trailing slash → Remove `/` at end
  
- **If stuck, do this:**
  ```bash
  # Edit again
  nano wrangler.toml
  # Make sure URL is correct
  # No trailing slash
  ```

---

## STEP 5: Implement Worker Code

### Step 5.1: Create src Folder

- **Goal:** Create folder for source code
- **Input:** None
- **Tool/Command:**
  ```bash
  mkdir src
  ```
- **Detailed Actions:**
  1. Make sure you're in backend folder
  2. Run command above
  
- **Output:** (none)
  
- **How to verify:**
  ```bash
  ls -la
  ```
  Should see: `src/` folder
  
- **What can go wrong:**
  - Folder not created → Check you're in backend folder
  
- **If stuck, do this:**
  ```bash
  pwd  # Should show .../backend
  mkdir -p src
  ls -la
  ```

### Step 5.2: Create index.js

- **Goal:** Main worker file
- **Input:** JavaScript code
- **Tool/Command:** Create file
- **Detailed Actions:**
  1. Create: `touch src/index.js`
  2. Open: `nano src/index.js`
  3. Copy code below (ALL of it)
  4. Paste into file
  5. Save (Ctrl+O, Enter, Ctrl+X)
  
- **File content (src/index.js):**
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

      if (url.pathname === '/api/upload' && request.method === 'POST') {
        return handleUpload(request, env, corsHeaders);
      }

      if (url.pathname === '/api/check' && request.method === 'GET') {
        return handleCheck(request, env, corsHeaders);
      }

      return new Response('Not Found', { status: 404, headers: corsHeaders });
    }
  };

  async function handleUpload(request, env, corsHeaders) {
    try {
      const apiKey = request.headers.get('X-API-Key');
      if (apiKey !== env.API_KEY) {
        return new Response(JSON.stringify({ error: 'Invalid API key' }), {
          status: 403,
          headers: { ...corsHeaders, 'Content-Type': 'application/json' }
        });
      }

      const formData = await request.formData();
      const file = formData.get('file');
      const app_id = formData.get('app_id');
      const platform = formData.get('platform');
      const version = formData.get('version');
      const patch_number = parseInt(formData.get('patch_number'));

      if (!file || !app_id || !platform || !version || !patch_number) {
        return new Response(JSON.stringify({ error: 'Missing fields' }), {
          status: 400,
          headers: { ...corsHeaders, 'Content-Type': 'application/json' }
        });
      }

      const arrayBuffer = await file.arrayBuffer();
      const hashBuffer = await crypto.subtle.digest('SHA-256', arrayBuffer);
      const hashArray = Array.from(new Uint8Array(hashBuffer));
      const sha256 = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');

      const r2Key = `${app_id}/${platform}/${patch_number}.bin`;
      await env.R2.put(r2Key, arrayBuffer);

      const download_url = `${env.R2_PUBLIC_URL}/${r2Key}`;

      await env.DB.prepare(`
        INSERT INTO patches (app_id, platform, patch_number, version, download_url, sha256, size_bytes)
        VALUES (?, ?, ?, ?, ?, ?, ?)
      `).bind(app_id, platform, patch_number, version, download_url, sha256, arrayBuffer.byteLength).run();

      return new Response(JSON.stringify({
        success: true,
        patch: { app_id, platform, patch_number, version, download_url, sha256, size_bytes: arrayBuffer.byteLength }
      }), {
        status: 200,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' }
      });
    } catch (error) {
      return new Response(JSON.stringify({ error: error.message }), {
        status: 500,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' }
      });
    }
  }

  async function handleCheck(request, env, corsHeaders) {
    try {
      const url = new URL(request.url);
      const app_id = url.searchParams.get('app_id');
      const platform = url.searchParams.get('platform');
      const version = url.searchParams.get('version');
      const current_patch = parseInt(url.searchParams.get('patch') || '0');

      if (!app_id || !platform || !version) {
        return new Response(JSON.stringify({ error: 'Missing parameters' }), {
          status: 400,
          headers: { ...corsHeaders, 'Content-Type': 'application/json' }
        });
      }

      const result = await env.DB.prepare(`
        SELECT patch_number, download_url, sha256, size_bytes
        FROM patches
        WHERE app_id = ? AND platform = ? AND version = ? AND patch_number > ? AND active = 1
        ORDER BY patch_number DESC LIMIT 1
      `).bind(app_id, platform, version, current_patch).first();

      if (!result) {
        return new Response(JSON.stringify({ has_update: false }), {
          status: 200,
          headers: { ...corsHeaders, 'Content-Type': 'application/json' }
        });
      }

      return new Response(JSON.stringify({
        has_update: true,
        patch: {
          patch_number: result.patch_number,
          download_url: result.download_url,
          sha256: result.sha256,
          size_bytes: result.size_bytes
        }
      }), {
        status: 200,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' }
      });
    } catch (error) {
      return new Response(JSON.stringify({ error: error.message }), {
        status: 500,
        headers: { ...corsHeaders, 'Content-Type': 'application/json' }
      });
    }
  }
  ```
  
- **Output:** File created with code
  
- **How to verify:**
  ```bash
  wc -l src/index.js
  ```
  Should show: around 100 lines
  
- **What can go wrong:**
  - File empty → Paste didn't work
  - Syntax error → Copy exactly as shown
  - Missing parts → Make sure you copied ALL code
  
- **If stuck, do this:**
  ```bash
  # Delete and recreate
  rm src/index.js
  nano src/index.js
  # Paste ALL code again
  # Save carefully
  ```

---

## STEP 6: Deploy to Cloudflare

### Step 6.1: Deploy Worker

- **Goal:** Upload code to Cloudflare
- **Input:** None
- **Tool/Command:**
  ```bash
  npx wrangler deploy
  ```
- **Detailed Actions:**
  1. Make sure you're in backend folder
  2. Run command above
  3. Wait 10-30 seconds
  4. Read output carefully
  
- **Output:**
  ```
  Total Upload: 5.23 KiB / gzip: 1.69 KiB
  Uploaded shorebird-backend (2.34 sec)
  Published shorebird-backend (0.89 sec)
    https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev
  Current Deployment ID: abc-123-def
  ```
  
- **How to verify:**
  Copy the URL from output, open in browser
  Should show: "Not Found" (that's correct!)
  
- **What can go wrong:**
  - "Syntax error" → Check src/index.js
  - "Database not found" → Check database_id in wrangler.toml
  - "Bucket not found" → Check bucket name
  - Deploy hangs → Check internet
  
- **If stuck, do this:**
  ```bash
  # Check wrangler.toml is correct
  cat wrangler.toml
  
  # Try deploy again
  npx wrangler deploy
  ```

### Step 6.2: Save Production URL

- **Goal:** Record your backend URL
- **Input:** URL from Step 6.1 output
- **Tool/Command:** Write it down
- **Detailed Actions:**
  1. Copy URL from deploy output
  2. Write it here: `_________________________________`
  3. You'll need this for CLI and Client setup
  
- **Example URL:**
  ```
  https://shorebird-backend.abc123xyz.workers.dev
  ```
  
- **How to verify:**
  Open URL in browser - should show "Not Found"
  
- **What can go wrong:**
  - Lost URL → Run `npx wrangler deployments list`
  - 404 error → That's OK, backend is working
  
- **If stuck, do this:**
  ```bash
  # Get URL again
  npx wrangler deployments list
  # Copy the URL shown
  ```

---

## STEP 7: Test Backend

### Step 7.1: Test Check Endpoint

- **Goal:** Verify API works
- **Input:** Your production URL
- **Tool/Command:**
  ```bash
  curl "https://YOUR-WORKER-URL/api/check?app_id=com.test.app&version=1.0.0&platform=android&patch=0"
  ```
- **Detailed Actions:**
  1. Replace `YOUR-WORKER-URL` with your actual URL
  2. Run command
  3. Check response
  
- **Output:**
  ```json
  {"has_update":false}
  ```
  
- **How to verify:**
  Response is valid JSON with `has_update` field
  
- **What can go wrong:**
  - "Not Found" → Check URL is correct
  - "Internal Server Error" → Check worker logs: `npx wrangler tail`
  - No response → Check internet
  
- **If stuck, do this:**
  ```bash
  # Check worker logs
  npx wrangler tail
  
  # In another terminal, run curl again
  # Check logs for errors
  ```

### Step 7.2: Test Upload Endpoint (Optional)

- **Goal:** Verify upload works
- **Input:** Test file
- **Tool/Command:**
  ```bash
  echo "test" > test.bin
  curl -X POST https://YOUR-WORKER-URL/api/upload \
    -H "X-API-Key: CHANGE-THIS-KEY" \
    -F "file=@test.bin" \
    -F "app_id=com.test.app" \
    -F "platform=android" \
    -F "version=1.0.0" \
    -F "patch_number=1"
  ```
- **Detailed Actions:**
  1. Create test file: `echo "test" > test.bin`
  2. Replace `YOUR-WORKER-URL` with your URL
  3. Replace `CHANGE-THIS-KEY` with API key from wrangler.toml
  4. Run curl command
  
- **Output:**
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
  
- **How to verify:**
  Response shows `"success": true`
  
- **What can go wrong:**
  - "Invalid API key" → Check API_KEY in wrangler.toml
  - "Missing fields" → Check all -F parameters
  - 500 error → Check logs: `npx wrangler tail`
  
- **If stuck, do this:**
  ```bash
  # Check API key
  cat wrangler.toml | grep API_KEY
  
  # Use that key in curl command
  ```

---

## ✅ Phase 1 Complete!

### What You Built

- ✅ Cloudflare Worker (serverless backend)
- ✅ D1 Database (for metadata)
- ✅ R2 Bucket (for patch files)
- ✅ API endpoints (upload + check)
- ✅ Live at: `https://your-worker.workers.dev`

### Verification Checklist

- [ ] `npx wrangler deployments list` shows deployment
- [ ] Check endpoint returns JSON
- [ ] Upload endpoint works (optional test)
- [ ] Production URL saved

### What's Next

**Next file:** `plan/02-CLI.md`

**Time needed:** 45 minutes

**What you'll build:** CLI tool to upload patches

**Take a break!** ☕ You've earned it!

---

## 🚨 Troubleshooting

### "wrangler: command not found"

```bash
# Install again
npm install -g wrangler

# Or use npx
npx wrangler --version
```

### "Not logged in"

```bash
wrangler logout
wrangler login
```

### "Database not found"

```bash
# List databases
wrangler d1 list

# Get correct ID
wrangler d1 info shorebird-db

# Update wrangler.toml with correct ID
```

### "Bucket not found"

```bash
# List buckets
wrangler r2 bucket list

# Create if missing
wrangler r2 bucket create shorebird-patches
```

### Deploy fails

```bash
# Check syntax
cat src/index.js

# Check config
cat wrangler.toml

# View logs
npx wrangler tail

# Try deploy again
npx wrangler deploy
```

### API returns errors

```bash
# View real-time logs
npx wrangler tail

# In another terminal, test API
curl "https://YOUR-URL/api/check?..."

# Check logs for error details
```

---

## 📊 Self-Assessment

Rate yourself (1-10):

- **Understanding:** Do you understand what you built? ___/10
- **Confidence:** Could you do this again? ___/10
- **Troubleshooting:** Can you fix errors? ___/10

**If any score < 7:** Re-read the steps you struggled with

**If all scores ≥ 7:** You're ready for Phase 2! 🎉

---

## 🎓 What You Learned

**Technical:**
- ✅ Cloudflare Workers (serverless functions)
- ✅ D1 Database (SQLite in the cloud)
- ✅ R2 Storage (object storage)
- ✅ REST API design
- ✅ Wrangler CLI

**Skills:**
- ✅ Following detailed instructions
- ✅ Using command-line tools
- ✅ Debugging errors
- ✅ Testing APIs with curl
- ✅ Reading logs

**Confidence:**
- ✅ You CAN build serverless backends
- ✅ You CAN deploy to production
- ✅ You CAN troubleshoot issues

---

## 🎉 Congratulations!

You've completed Phase 1!

**What you have:**
- ✅ Live backend on Cloudflare
- ✅ Working API endpoints
- ✅ Database and storage ready
- ✅ 100% FREE (no costs)

**Next:** Build CLI tool to upload patches

**File:** `plan/02-CLI.md`

**Ready?** Let's go! 🚀
