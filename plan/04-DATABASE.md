# Database Schema Plan

## 🎯 Goal

Design a minimal database schema to store patch metadata and support update queries.

## 📊 Database Choice

### Development: SQLite
- Zero configuration
- File-based
- Perfect for MVP
- Easy to backup

### Production: SQLite or PostgreSQL
- SQLite: Up to 10K users
- PostgreSQL: 10K+ users, multiple servers

## 📋 Schema Design

### patches Table

```sql
CREATE TABLE IF NOT EXISTS patches (
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
```

### Field Descriptions

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| id | INTEGER | Auto-increment primary key | 1, 2, 3... |
| app_id | TEXT | App identifier | com.yourapp.name |
| platform | TEXT | android or ios | android |
| patch_number | INTEGER | Sequential patch number | 1, 2, 3... |
| version | TEXT | App version (semver) | 1.0.0 |
| download_url | TEXT | CDN URL for patch file | https://cdn.../patch.bin |
| sha256 | TEXT | Hash for verification | abc123... |
| size_bytes | INTEGER | File size in bytes | 1024000 |
| active | INTEGER | Enable/disable (1/0) | 1 |
| created_at | TEXT | ISO 8601 timestamp | 2024-01-01T12:00:00Z |

### Indexes

```sql
-- For fast lookup by app_id + platform + active
CREATE INDEX IF NOT EXISTS idx_patches_lookup 
ON patches(app_id, platform, active, patch_number);

-- For version-specific queries
CREATE INDEX IF NOT EXISTS idx_patches_version 
ON patches(app_id, platform, version, active);

-- For time-based queries
CREATE INDEX IF NOT EXISTS idx_patches_created 
ON patches(created_at DESC);
```

## 🔍 Query Patterns

### 1. Check for Update

**Use Case:** Client checks if newer patch exists

```sql
SELECT 
  id,
  patch_number,
  download_url,
  sha256,
  size_bytes
FROM patches
WHERE app_id = ?
  AND platform = ?
  AND version = ?
  AND patch_number > ?
  AND active = 1
ORDER BY patch_number DESC
LIMIT 1;
```

**Parameters:**
- app_id: 'com.yourapp.name'
- platform: 'android'
- version: '1.0.0'
- current_patch: 3

**Result:**
- Returns latest patch if available
- Returns empty if no update

### 2. Insert New Patch

**Use Case:** CLI uploads new patch

```sql
INSERT INTO patches (
  app_id,
  platform,
  patch_number,
  version,
  download_url,
  sha256,
  size_bytes
) VALUES (?, ?, ?, ?, ?, ?, ?);
```

**Conflict Handling:**
```sql
-- If patch_number already exists, fail
-- UNIQUE constraint will throw error
```

### 3. Get Next Patch Number

**Use Case:** CLI needs to know next patch number

```sql
SELECT COALESCE(MAX(patch_number), 0) + 1 as next_patch
FROM patches
WHERE app_id = ?
  AND platform = ?;
```

### 4. List All Patches

**Use Case:** Admin dashboard

```sql
SELECT 
  id,
  app_id,
  platform,
  patch_number,
  version,
  size_bytes,
  active,
  created_at
FROM patches
WHERE app_id = ?
ORDER BY created_at DESC;
```

### 5. Disable Patch (Rollback)

**Use Case:** Rollback bad patch

```sql
UPDATE patches
SET active = 0
WHERE app_id = ?
  AND platform = ?
  AND patch_number = ?;
```

### 6. Get Patch Statistics

**Use Case:** Analytics

```sql
SELECT 
  app_id,
  platform,
  COUNT(*) as total_patches,
  MAX(patch_number) as latest_patch,
  SUM(size_bytes) as total_size
FROM patches
WHERE active = 1
GROUP BY app_id, platform;
```

## 💾 Database Operations (db.js)

### Initialize Database

```javascript
function initDatabase(dbPath) {
  const db = new Database(dbPath);
  
  // Create table
  db.exec(`
    CREATE TABLE IF NOT EXISTS patches (
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
  `);
  
  // Create indexes
  db.exec(`
    CREATE INDEX IF NOT EXISTS idx_patches_lookup 
    ON patches(app_id, platform, active, patch_number);
    
    CREATE INDEX IF NOT EXISTS idx_patches_version 
    ON patches(app_id, platform, version, active);
    
    CREATE INDEX IF NOT EXISTS idx_patches_created 
    ON patches(created_at DESC);
  `);
  
  return db;
}
```

### Insert Patch

```javascript
function insertPatch(db, patch) {
  const stmt = db.prepare(`
    INSERT INTO patches (
      app_id, platform, patch_number, version,
      download_url, sha256, size_bytes
    ) VALUES (?, ?, ?, ?, ?, ?, ?)
  `);
  
  const result = stmt.run(
    patch.app_id,
    patch.platform,
    patch.patch_number,
    patch.version,
    patch.download_url,
    patch.sha256,
    patch.size_bytes
  );
  
  return result.lastInsertRowid;
}
```

### Find Latest Patch

```javascript
function findLatestPatch(db, query) {
  const stmt = db.prepare(`
    SELECT 
      id, patch_number, download_url, sha256, size_bytes
    FROM patches
    WHERE app_id = ?
      AND platform = ?
      AND version = ?
      AND patch_number > ?
      AND active = 1
    ORDER BY patch_number DESC
    LIMIT 1
  `);
  
  return stmt.get(
    query.app_id,
    query.platform,
    query.version,
    query.current_patch || 0
  );
}
```

### Get Next Patch Number

```javascript
function getNextPatchNumber(db, app_id, platform) {
  const stmt = db.prepare(`
    SELECT COALESCE(MAX(patch_number), 0) + 1 as next_patch
    FROM patches
    WHERE app_id = ? AND platform = ?
  `);
  
  const result = stmt.get(app_id, platform);
  return result.next_patch;
}
```

## 🔄 Migration Strategy

### Version 1 (Initial)
- Create patches table
- Create indexes

### Future Versions

**Version 2: Add analytics table**
```sql
CREATE TABLE patch_installs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  patch_id INTEGER REFERENCES patches(id),
  device_id TEXT,
  status TEXT CHECK(status IN ('success', 'failed')),
  error_message TEXT,
  installed_at TEXT DEFAULT (datetime('now'))
);
```

**Version 3: Add release_id**
```sql
ALTER TABLE patches ADD COLUMN release_id TEXT;
```

## 🔐 Security Considerations

### SQL Injection Prevention
- Use prepared statements
- Never concatenate user input
- Validate all inputs

### Data Validation
- Check app_id format
- Validate platform enum
- Validate version format
- Validate patch_number > 0

## 📊 Sample Data

```sql
-- Insert sample patches
INSERT INTO patches (app_id, platform, patch_number, version, download_url, sha256, size_bytes)
VALUES 
  ('com.example.app', 'android', 1, '1.0.0', 'https://cdn.example.com/1.bin', 'abc123', 1024000),
  ('com.example.app', 'android', 2, '1.0.0', 'https://cdn.example.com/2.bin', 'def456', 1048576),
  ('com.example.app', 'ios', 1, '1.0.0', 'https://cdn.example.com/ios1.bin', 'ghi789', 2097152);
```

## ✅ Success Criteria

- [ ] Database initializes without errors
- [ ] Patches can be inserted
- [ ] Queries return correct results
- [ ] Indexes improve performance
- [ ] Unique constraint prevents duplicates
- [ ] Rollback works (active flag)
- [ ] Prepared statements prevent SQL injection
