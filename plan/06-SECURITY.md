# Security Plan

## 🎯 Goal

Ensure the code push system is secure against common attacks and vulnerabilities.

## 🔐 Security Layers

```
┌─────────────────────────────────────────┐
│         1. API Authentication           │
│         (API Key)                       │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         2. Input Validation             │
│         (Sanitize all inputs)           │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         3. File Validation              │
│         (Size, type, hash)              │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         4. Transport Security           │
│         (HTTPS only)                    │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         5. Patch Verification           │
│         (SHA256 hash check)             │
└─────────────────────────────────────────┘
```

## 🔑 API Authentication

### API Key Strategy

**Generate Strong Keys:**
```bash
# Generate 32-byte random key
openssl rand -hex 32
# Output: a1b2c3d4e5f6...
```

**Storage:**
- Backend: Environment variable
- CLI: .env file (gitignored)
- Never hardcode in source

**Validation:**
```javascript
function authenticateApiKey(req, res, next) {
  const apiKey = req.headers['x-api-key'];
  
  if (!apiKey) {
    return res.status(401).json({ 
      error: 'API key required' 
    });
  }
  
  // Constant-time comparison to prevent timing attacks
  const expectedKey = Buffer.from(process.env.API_KEY);
  const providedKey = Buffer.from(apiKey);
  
  if (expectedKey.length !== providedKey.length) {
    return res.status(403).json({ 
      error: 'Invalid API key' 
    });
  }
  
  if (!crypto.timingSafeEqual(expectedKey, providedKey)) {
    return res.status(403).json({ 
      error: 'Invalid API key' 
    });
  }
  
  next();
}
```

### Rate Limiting

**Prevent Abuse:**
```javascript
import rateLimit from 'express-rate-limit';

const uploadLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 10, // 10 uploads per window
  message: 'Too many uploads, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
});

app.post('/api/upload', uploadLimiter, authenticateApiKey, uploadHandler);
```

**Check Endpoint:**
```javascript
const checkLimiter = rateLimit({
  windowMs: 1 * 60 * 1000, // 1 minute
  max: 60, // 60 requests per minute
  message: 'Too many requests, please try again later',
});

app.get('/api/check', checkLimiter, checkHandler);
```

## 🛡️ Input Validation

### Request Validation

**Upload Endpoint:**
```javascript
function validateUploadRequest(req, res, next) {
  const { app_id, platform, version, patch_number } = req.body;
  
  // Validate app_id
  if (!app_id || !/^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)+$/.test(app_id)) {
    return res.status(400).json({ 
      error: 'Invalid app_id format' 
    });
  }
  
  // Validate platform
  if (!['android', 'ios'].includes(platform)) {
    return res.status(400).json({ 
      error: 'Platform must be android or ios' 
    });
  }
  
  // Validate version (semver)
  if (!version || !/^\d+\.\d+\.\d+$/.test(version)) {
    return res.status(400).json({ 
      error: 'Invalid version format (use semver: 1.0.0)' 
    });
  }
  
  // Validate patch_number
  const patchNum = parseInt(patch_number);
  if (isNaN(patchNum) || patchNum < 1) {
    return res.status(400).json({ 
      error: 'Patch number must be positive integer' 
    });
  }
  
  next();
}
```

**Check Endpoint:**
```javascript
function validateCheckRequest(req, res, next) {
  const { app_id, platform, version, patch } = req.query;
  
  // Same validation as upload
  // Plus validate patch parameter
  
  const patchNum = parseInt(patch || '0');
  if (isNaN(patchNum) || patchNum < 0) {
    return res.status(400).json({ 
      error: 'Invalid patch number' 
    });
  }
  
  next();
}
```

### SQL Injection Prevention

**Always use prepared statements:**
```javascript
// ❌ NEVER DO THIS
const query = `SELECT * FROM patches WHERE app_id = '${app_id}'`;

// ✅ ALWAYS DO THIS
const stmt = db.prepare('SELECT * FROM patches WHERE app_id = ?');
const result = stmt.get(app_id);
```

## 📁 File Validation

### File Upload Security

**Size Limit:**
```javascript
import multer from 'multer';

const upload = multer({
  dest: 'uploads/',
  limits: {
    fileSize: 50 * 1024 * 1024, // 50 MB
  },
  fileFilter: (req, file, cb) => {
    // Validate file extension
    const allowedExts = ['.vmcode', '.bin'];
    const ext = path.extname(file.originalname);
    
    if (!allowedExts.includes(ext)) {
      return cb(new Error('Invalid file type'));
    }
    
    cb(null, true);
  },
});
```

**File Content Validation:**
```javascript
function validatePatchFile(filePath) {
  // Check file exists
  if (!fs.existsSync(filePath)) {
    throw new Error('File not found');
  }
  
  // Check file size
  const stats = fs.statSync(filePath);
  if (stats.size === 0) {
    throw new Error('File is empty');
  }
  
  if (stats.size > 50 * 1024 * 1024) {
    throw new Error('File too large');
  }
  
  // Check magic bytes (optional)
  // Verify it's actually a valid patch file
  
  return true;
}
```

### Hash Verification

**Calculate and Store:**
```javascript
import crypto from 'crypto';

function calculateHash(filePath) {
  const fileBuffer = fs.readFileSync(filePath);
  const hash = crypto.createHash('sha256');
  hash.update(fileBuffer);
  return hash.digest('hex');
}
```

**Client-Side Verification:**
```dart
Future<bool> verifyHash(File file, String expectedHash) async {
  final bytes = await file.readAsBytes();
  final hash = sha256.convert(bytes).toString();
  
  if (hash != expectedHash) {
    throw Exception('Hash mismatch: file may be corrupted or tampered');
  }
  
  return true;
}
```

## 🔒 Transport Security

### HTTPS Only

**Development:**
```javascript
// Allow HTTP for local testing
if (process.env.NODE_ENV === 'production') {
  app.use((req, res, next) => {
    if (!req.secure) {
      return res.redirect('https://' + req.headers.host + req.url);
    }
    next();
  });
}
```

**Production:**
- Use reverse proxy (nginx, Caddy)
- Enable HTTPS with Let's Encrypt
- Force HTTPS redirect
- Enable HSTS header

**nginx config:**
```nginx
server {
  listen 443 ssl http2;
  server_name api.yourapp.com;
  
  ssl_certificate /etc/letsencrypt/live/api.yourapp.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/api.yourapp.com/privkey.pem;
  
  # Security headers
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-Frame-Options "DENY" always;
  
  location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

## 🚫 Attack Prevention

### 1. Path Traversal

**Prevent:**
```javascript
function sanitizePath(userInput) {
  // Remove any path traversal attempts
  const sanitized = userInput.replace(/\.\./g, '');
  
  // Validate against whitelist
  if (!/^[a-z0-9._-]+$/i.test(sanitized)) {
    throw new Error('Invalid path');
  }
  
  return sanitized;
}
```

### 2. Denial of Service

**Prevent:**
- Rate limiting (implemented above)
- File size limits
- Request timeout
- Connection limits

```javascript
import timeout from 'connect-timeout';

app.use(timeout('30s'));
app.use((req, res, next) => {
  if (!req.timedout) next();
});
```

### 3. Malicious Patches

**Prevent:**
- Hash verification (client-side)
- Code signing (future enhancement)
- Rollback mechanism
- Gradual rollout (future enhancement)

### 4. Man-in-the-Middle

**Prevent:**
- HTTPS only
- Certificate pinning (optional)
- Verify SSL certificates

## 🔍 Monitoring & Logging

### Security Logging

**Log Security Events:**
```javascript
function logSecurityEvent(event, details) {
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    type: 'security',
    event: event,
    details: details,
  }));
}

// Usage
logSecurityEvent('auth_failed', {
  ip: req.ip,
  endpoint: req.path,
});

logSecurityEvent('upload_success', {
  app_id: req.body.app_id,
  patch_number: req.body.patch_number,
});
```

### Alert on Suspicious Activity

**Monitor:**
- Multiple failed auth attempts
- Unusual upload patterns
- Large file uploads
- High request rates
- Failed hash verifications

## 📋 Security Checklist

### Backend
- [ ] API key authentication enabled
- [ ] Rate limiting configured
- [ ] Input validation on all endpoints
- [ ] SQL injection prevention (prepared statements)
- [ ] File size limits enforced
- [ ] File type validation
- [ ] HTTPS enforced in production
- [ ] Security headers configured
- [ ] Error messages don't leak info
- [ ] Logging enabled

### CLI
- [ ] API key stored in .env (gitignored)
- [ ] HTTPS used for uploads
- [ ] File hash calculated before upload
- [ ] Errors handled gracefully
- [ ] No sensitive data in logs

### Client
- [ ] Hash verification before install
- [ ] HTTPS used for downloads
- [ ] Rollback on failed patch
- [ ] No arbitrary code execution
- [ ] Version compatibility check

### Infrastructure
- [ ] SSL certificate valid
- [ ] Firewall configured
- [ ] Database backups enabled
- [ ] R2 bucket private (except public URL)
- [ ] API tokens rotated regularly
- [ ] Monitoring enabled
- [ ] Logs reviewed regularly

## 🔄 Incident Response

### If Malicious Patch Detected

1. **Immediate:**
   - Disable patch (set active = 0)
   - Remove from R2
   - Alert users

2. **Investigation:**
   - Review logs
   - Identify source
   - Assess impact

3. **Recovery:**
   - Deploy rollback patch
   - Update security measures
   - Document incident

### If API Key Compromised

1. **Immediate:**
   - Generate new API key
   - Update backend .env
   - Update CLI .env
   - Revoke old key

2. **Investigation:**
   - Review upload logs
   - Check for unauthorized patches
   - Assess damage

3. **Prevention:**
   - Rotate keys regularly
   - Use separate keys per environment
   - Monitor usage

## ✅ Success Criteria

- [ ] All endpoints require authentication
- [ ] All inputs validated
- [ ] All files validated
- [ ] HTTPS enforced
- [ ] Rate limiting works
- [ ] Hash verification works
- [ ] Security logging enabled
- [ ] Incident response plan documented
