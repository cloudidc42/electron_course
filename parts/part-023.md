# Part 23: Database with SQLite ใน Electron

## บทนำ

SQLite เป็น embedded database ที่เหมาะอย่างยิ่งสำหรับ Electron applications เพราะไม่ต้องการ server แยกต่างหาก ในบทนี้เราจะใช้ `better-sqlite3` ซึ่งเป็น library ที่ synchronous, เร็ว และใช้งานง่าย เราจะครอบคลุมตั้งแต่การออกแบบ schema, CRUD operations, transactions, migrations, full-text search และ best practices

## การติดตั้ง

```bash
# ติดตั้ง better-sqlite3
npm install better-sqlite3

# ติดตั้ง dev dependencies สำหรับ rebuilding
npm install --save-dev electron-rebuild

# เพิ่ม script ใน package.json
# "postinstall": "electron-rebuild"
```

## โครงสร้างโปรเจกต์

```
electron/
├── database/
│   ├── index.js          # Database connection manager
│   ├── schema.js         # Schema definitions
│   ├── migrations/
│   │   ├── index.js      # Migration runner
│   │   ├── 001_initial.js
│   │   ├── 002_add_tags.js
│   │   └── 003_fts.js
│   ├── models/
│   │   ├── Note.js
│   │   ├── Tag.js
│   │   └── User.js
│   └── repositories/
│       ├── BaseRepository.js
│       ├── NoteRepository.js
│       └── TagRepository.js
```

## Database Connection Manager

```javascript
// electron/database/index.js
const Database = require('better-sqlite3')
const path = require('path')
const fs = require('fs')
const { app } = require('electron')
const { runMigrations } = require('./migrations')

class DatabaseManager {
  constructor() {
    this.db = null
    this.dbPath = null
  }

  /**
   * เปิด database connection
   */
  connect(options = {}) {
    if (this.db) return this.db

    // กำหนด path สำหรับ database file
    const userDataPath = app ? app.getPath('userData') : process.cwd()
    const dbName = options.filename || 'app.db'
    this.dbPath = options.path || path.join(userDataPath, dbName)

    // สร้าง directory ถ้าไม่มี
    const dbDir = path.dirname(this.dbPath)
    if (!fs.existsSync(dbDir)) {
      fs.mkdirSync(dbDir, { recursive: true })
    }

    // เปิด database
    this.db = new Database(this.dbPath, {
      verbose: process.env.NODE_ENV === 'development' 
        ? console.log 
        : undefined
    })

    // ตั้งค่า pragmas สำหรับ performance
    this.db.pragma('journal_mode = WAL')      // Write-Ahead Logging ทำให้เร็วขึ้น
    this.db.pragma('foreign_keys = ON')        // เปิด foreign key constraints
    this.db.pragma('synchronous = NORMAL')     // Balance ระหว่าง safety กับ speed
    this.db.pragma('cache_size = -64000')      // 64MB cache
    this.db.pragma('temp_store = MEMORY')      // เก็บ temp data ใน memory
    this.db.pragma('mmap_size = 268435456')    // 256MB memory-mapped I/O

    // รัน migrations
    runMigrations(this.db)

    console.log(`[Database] Connected to: ${this.dbPath}`)
    return this.db
  }

  /**
   * ปิด database connection
   */
  disconnect() {
    if (this.db) {
      this.db.close()
      this.db = null
      console.log('[Database] Disconnected')
    }
  }

  /**
   * ดึง connection ปัจจุบัน
   */
  getConnection() {
    if (!this.db) {
      throw new Error('Database not connected. Call connect() first.')
    }
    return this.db
  }

  /**
   * Backup database
   */
  async backup(destination) {
    if (!this.db) throw new Error('Database not connected')
    await this.db.backup(destination)
    console.log(`[Database] Backed up to: ${destination}`)
  }

  /**
   * ดึงข้อมูล database statistics
   */
  getStats() {
    if (!this.db) return null
    
    const pageSize = this.db.pragma('page_size', { simple: true })
    const pageCount = this.db.pragma('page_count', { simple: true })
    const freelistCount = this.db.pragma('freelist_count', { simple: true })
    
    return {
      path: this.dbPath,
      size: pageSize * pageCount,
      freeSpace: pageSize * freelistCount,
      pageSize,
      pageCount,
      walMode: this.db.pragma('journal_mode', { simple: true }) === 'wal'
    }
  }
}

// Singleton instance
const dbManager = new DatabaseManager()

module.exports = dbManager
```

## Schema Definitions

```javascript
// electron/database/schema.js

/**
 * SQL สำหรับสร้าง tables ทั้งหมด
 * แบ่งออกตาม concern
 */

const USERS_TABLE = `
  CREATE TABLE IF NOT EXISTS users (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    username    TEXT    NOT NULL UNIQUE,
    email       TEXT    UNIQUE,
    displayName TEXT,
    avatar      BLOB,
    settings    TEXT    DEFAULT '{}',
    createdAt   TEXT    NOT NULL DEFAULT (datetime('now')),
    updatedAt   TEXT    NOT NULL DEFAULT (datetime('now')),
    deletedAt   TEXT
  )
`

const NOTES_TABLE = `
  CREATE TABLE IF NOT EXISTS notes (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    userId      INTEGER NOT NULL,
    title       TEXT    NOT NULL DEFAULT 'Untitled',
    content     TEXT    NOT NULL DEFAULT '',
    contentType TEXT    NOT NULL DEFAULT 'markdown',
    isPinned    INTEGER NOT NULL DEFAULT 0,
    isArchived  INTEGER NOT NULL DEFAULT 0,
    color       TEXT,
    metadata    TEXT    DEFAULT '{}',
    createdAt   TEXT    NOT NULL DEFAULT (datetime('now')),
    updatedAt   TEXT    NOT NULL DEFAULT (datetime('now')),
    deletedAt   TEXT,
    
    FOREIGN KEY (userId) REFERENCES users(id) ON DELETE CASCADE
  )
`

const TAGS_TABLE = `
  CREATE TABLE IF NOT EXISTS tags (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    userId    INTEGER NOT NULL,
    name      TEXT    NOT NULL,
    color     TEXT    DEFAULT '#6366f1',
    createdAt TEXT    NOT NULL DEFAULT (datetime('now')),
    
    UNIQUE(userId, name),
    FOREIGN KEY (userId) REFERENCES users(id) ON DELETE CASCADE
  )
`

const NOTE_TAGS_TABLE = `
  CREATE TABLE IF NOT EXISTS note_tags (
    noteId INTEGER NOT NULL,
    tagId  INTEGER NOT NULL,
    
    PRIMARY KEY (noteId, tagId),
    FOREIGN KEY (noteId) REFERENCES notes(id) ON DELETE CASCADE,
    FOREIGN KEY (tagId)  REFERENCES tags(id)  ON DELETE CASCADE
  )
`

const ATTACHMENTS_TABLE = `
  CREATE TABLE IF NOT EXISTS attachments (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    noteId      INTEGER NOT NULL,
    filename    TEXT    NOT NULL,
    mimeType    TEXT,
    size        INTEGER,
    data        BLOB,
    createdAt   TEXT    NOT NULL DEFAULT (datetime('now')),
    
    FOREIGN KEY (noteId) REFERENCES notes(id) ON DELETE CASCADE
  )
`

// FTS (Full-Text Search) virtual table
const NOTES_FTS_TABLE = `
  CREATE VIRTUAL TABLE IF NOT EXISTS notes_fts
  USING fts5(
    title,
    content,
    content=notes,
    content_rowid=id,
    tokenize='unicode61 tokenchars "-_."'
  )
`

// Triggers สำหรับ sync FTS
const FTS_TRIGGERS = `
  CREATE TRIGGER IF NOT EXISTS notes_ai AFTER INSERT ON notes BEGIN
    INSERT INTO notes_fts(rowid, title, content)
    VALUES (new.id, new.title, new.content);
  END;

  CREATE TRIGGER IF NOT EXISTS notes_ad AFTER DELETE ON notes BEGIN
    INSERT INTO notes_fts(notes_fts, rowid, title, content)
    VALUES ('delete', old.id, old.title, old.content);
  END;

  CREATE TRIGGER IF NOT EXISTS notes_au AFTER UPDATE ON notes BEGIN
    INSERT INTO notes_fts(notes_fts, rowid, title, content)
    VALUES ('delete', old.id, old.title, old.content);
    INSERT INTO notes_fts(rowid, title, content)
    VALUES (new.id, new.title, new.content);
  END;
`

// Indexes
const INDEXES = `
  CREATE INDEX IF NOT EXISTS idx_notes_userId ON notes(userId);
  CREATE INDEX IF NOT EXISTS idx_notes_createdAt ON notes(createdAt DESC);
  CREATE INDEX IF NOT EXISTS idx_notes_updatedAt ON notes(updatedAt DESC);
  CREATE INDEX IF NOT EXISTS idx_notes_isPinned ON notes(isPinned);
  CREATE INDEX IF NOT EXISTS idx_notes_isArchived ON notes(isArchived);
  CREATE INDEX IF NOT EXISTS idx_note_tags_noteId ON note_tags(noteId);
  CREATE INDEX IF NOT EXISTS idx_note_tags_tagId ON note_tags(tagId);
  CREATE INDEX IF NOT EXISTS idx_tags_userId ON tags(userId);
`

module.exports = {
  USERS_TABLE,
  NOTES_TABLE,
  TAGS_TABLE,
  NOTE_TAGS_TABLE,
  ATTACHMENTS_TABLE,
  NOTES_FTS_TABLE,
  FTS_TRIGGERS,
  INDEXES
}
```

## Migrations System

```javascript
// electron/database/migrations/index.js

const migrations = [
  require('./001_initial'),
  require('./002_add_tags'),
  require('./003_fts')
]

/**
 * รัน migrations ที่ยังไม่ได้รัน
 */
function runMigrations(db) {
  // สร้าง migrations table ถ้าไม่มี
  db.exec(`
    CREATE TABLE IF NOT EXISTS _migrations (
      id        INTEGER PRIMARY KEY AUTOINCREMENT,
      name      TEXT    NOT NULL UNIQUE,
      appliedAt TEXT    NOT NULL DEFAULT (datetime('now'))
    )
  `)

  // ดึงรายการ migrations ที่รันแล้ว
  const applied = new Set(
    db.prepare('SELECT name FROM _migrations').all().map(r => r.name)
  )

  // รัน migrations ที่ยังไม่ได้รัน
  let migrationCount = 0
  
  for (const migration of migrations) {
    if (!applied.has(migration.name)) {
      console.log(`[Migration] Running: ${migration.name}`)
      
      // รัน migration ใน transaction
      const runMigration = db.transaction(() => {
        migration.up(db)
        db.prepare('INSERT INTO _migrations (name) VALUES (?)').run(migration.name)
      })
      
      try {
        runMigration()
        migrationCount++
        console.log(`[Migration] Completed: ${migration.name}`)
      } catch (error) {
        console.error(`[Migration] Failed: ${migration.name}`, error)
        throw error
      }
    }
  }

  if (migrationCount > 0) {
    console.log(`[Migration] Applied ${migrationCount} migration(s)`)
  }
}

/**
 * Rollback migrations
 */
function rollbackMigrations(db, steps = 1) {
  const applied = db.prepare(
    'SELECT name FROM _migrations ORDER BY id DESC LIMIT ?'
  ).all(steps)

  for (const { name } of applied) {
    const migration = migrations.find(m => m.name === name)
    if (!migration?.down) {
      throw new Error(`Migration ${name} has no rollback`)
    }

    const rollback = db.transaction(() => {
      migration.down(db)
      db.prepare('DELETE FROM _migrations WHERE name = ?').run(name)
    })

    rollback()
    console.log(`[Migration] Rolled back: ${name}`)
  }
}

module.exports = { runMigrations, rollbackMigrations }
```

```javascript
// electron/database/migrations/001_initial.js
module.exports = {
  name: '001_initial',
  
  up(db) {
    db.exec(`
      CREATE TABLE IF NOT EXISTS users (
        id          INTEGER PRIMARY KEY AUTOINCREMENT,
        username    TEXT    NOT NULL UNIQUE,
        email       TEXT    UNIQUE,
        displayName TEXT,
        settings    TEXT    DEFAULT '{}',
        createdAt   TEXT    NOT NULL DEFAULT (datetime('now')),
        updatedAt   TEXT    NOT NULL DEFAULT (datetime('now'))
      );

      CREATE TABLE IF NOT EXISTS notes (
        id          INTEGER PRIMARY KEY AUTOINCREMENT,
        userId      INTEGER NOT NULL,
        title       TEXT    NOT NULL DEFAULT 'Untitled',
        content     TEXT    NOT NULL DEFAULT '',
        isPinned    INTEGER NOT NULL DEFAULT 0,
        isArchived  INTEGER NOT NULL DEFAULT 0,
        createdAt   TEXT    NOT NULL DEFAULT (datetime('now')),
        updatedAt   TEXT    NOT NULL DEFAULT (datetime('now')),
        
        FOREIGN KEY (userId) REFERENCES users(id) ON DELETE CASCADE
      );

      CREATE INDEX IF NOT EXISTS idx_notes_userId ON notes(userId);
      CREATE INDEX IF NOT EXISTS idx_notes_updatedAt ON notes(updatedAt DESC);
    `)

    // สร้าง default user
    db.prepare(`
      INSERT OR IGNORE INTO users (username, displayName) VALUES ('default', 'Default User')
    `).run()
  },

  down(db) {
    db.exec('DROP TABLE IF EXISTS notes; DROP TABLE IF EXISTS users;')
  }
}
```

```javascript
// electron/database/migrations/002_add_tags.js
module.exports = {
  name: '002_add_tags',
  
  up(db) {
    db.exec(`
      CREATE TABLE IF NOT EXISTS tags (
        id        INTEGER PRIMARY KEY AUTOINCREMENT,
        userId    INTEGER NOT NULL,
        name      TEXT    NOT NULL,
        color     TEXT    DEFAULT '#6366f1',
        createdAt TEXT    NOT NULL DEFAULT (datetime('now')),
        
        UNIQUE(userId, name),
        FOREIGN KEY (userId) REFERENCES users(id) ON DELETE CASCADE
      );

      CREATE TABLE IF NOT EXISTS note_tags (
        noteId INTEGER NOT NULL,
        tagId  INTEGER NOT NULL,
        PRIMARY KEY (noteId, tagId),
        FOREIGN KEY (noteId) REFERENCES notes(id) ON DELETE CASCADE,
        FOREIGN KEY (tagId)  REFERENCES tags(id)  ON DELETE CASCADE
      );

      -- เพิ่ม color column ใน notes
      ALTER TABLE notes ADD COLUMN color TEXT;
    `)
  },

  down(db) {
    db.exec(`
      DROP TABLE IF EXISTS note_tags;
      DROP TABLE IF EXISTS tags;
    `)
  }
}
```

```javascript
// electron/database/migrations/003_fts.js
module.exports = {
  name: '003_fts',
  
  up(db) {
    db.exec(`
      CREATE VIRTUAL TABLE IF NOT EXISTS notes_fts
      USING fts5(
        title,
        content,
        content=notes,
        content_rowid=id
      );

      -- Populate FTS with existing data
      INSERT INTO notes_fts(rowid, title, content)
      SELECT id, title, content FROM notes;

      -- Triggers สำหรับ auto-sync
      CREATE TRIGGER IF NOT EXISTS notes_ai AFTER INSERT ON notes BEGIN
        INSERT INTO notes_fts(rowid, title, content)
        VALUES (new.id, new.title, new.content);
      END;

      CREATE TRIGGER IF NOT EXISTS notes_ad AFTER DELETE ON notes BEGIN
        INSERT INTO notes_fts(notes_fts, rowid, title, content)
        VALUES ('delete', old.id, old.title, old.content);
      END;

      CREATE TRIGGER IF NOT EXISTS notes_au AFTER UPDATE ON notes BEGIN
        INSERT INTO notes_fts(notes_fts, rowid, title, content)
        VALUES ('delete', old.id, old.title, old.content);
        INSERT INTO notes_fts(rowid, title, content)
        VALUES (new.id, new.title, new.content);
      END;
    `)
  },

  down(db) {
    db.exec(`
      DROP TRIGGER IF EXISTS notes_au;
      DROP TRIGGER IF EXISTS notes_ad;
      DROP TRIGGER IF EXISTS notes_ai;
      DROP TABLE IF EXISTS notes_fts;
    `)
  }
}
```

## Base Repository Pattern

```javascript
// electron/database/repositories/BaseRepository.js

class BaseRepository {
  constructor(db, tableName) {
    this.db = db
    this.tableName = tableName
    this._preparedStatements = new Map()
  }

  /**
   * Cache prepared statements เพื่อ performance
   */
  prepare(sql) {
    if (!this._preparedStatements.has(sql)) {
      this._preparedStatements.set(sql, this.db.prepare(sql))
    }
    return this._preparedStatements.get(sql)
  }

  /**
   * ค้นหาตาม ID
   */
  findById(id) {
    return this.prepare(
      `SELECT * FROM ${this.tableName} WHERE id = ?`
    ).get(id)
  }

  /**
   * ค้นหาทั้งหมด
   */
  findAll(options = {}) {
    const { 
      limit = 100, 
      offset = 0, 
      orderBy = 'id', 
      order = 'ASC',
      where = null 
    } = options
    
    let sql = `SELECT * FROM ${this.tableName}`
    const params = []
    
    if (where) {
      const conditions = Object.entries(where)
        .map(([key, value]) => {
          params.push(value)
          return `${key} = ?`
        })
        .join(' AND ')
      sql += ` WHERE ${conditions}`
    }
    
    sql += ` ORDER BY ${orderBy} ${order} LIMIT ? OFFSET ?`
    params.push(limit, offset)
    
    return this.db.prepare(sql).all(...params)
  }

  /**
   * นับจำนวนทั้งหมด
   */
  count(where = null) {
    let sql = `SELECT COUNT(*) as count FROM ${this.tableName}`
    const params = []
    
    if (where) {
      const conditions = Object.entries(where)
        .map(([key, value]) => {
          params.push(value)
          return `${key} = ?`
        })
        .join(' AND ')
      sql += ` WHERE ${conditions}`
    }
    
    return this.db.prepare(sql).get(...params).count
  }

  /**
   * สร้างข้อมูลใหม่
   */
  create(data) {
    const keys = Object.keys(data)
    const values = Object.values(data)
    const placeholders = keys.map(() => '?').join(', ')
    
    const sql = `
      INSERT INTO ${this.tableName} (${keys.join(', ')})
      VALUES (${placeholders})
    `
    
    const result = this.db.prepare(sql).run(...values)
    return this.findById(result.lastInsertRowid)
  }

  /**
   * อัปเดตข้อมูล
   */
  update(id, data) {
    const keys = Object.keys(data)
    const values = Object.values(data)
    
    if (keys.length === 0) return this.findById(id)
    
    const setClause = keys.map(k => `${k} = ?`).join(', ')
    const sql = `
      UPDATE ${this.tableName}
      SET ${setClause}, updatedAt = datetime('now')
      WHERE id = ?
    `
    
    this.db.prepare(sql).run(...values, id)
    return this.findById(id)
  }

  /**
   * ลบข้อมูล (soft delete)
   */
  delete(id) {
    return this.prepare(
      `UPDATE ${this.tableName} SET deletedAt = datetime('now') WHERE id = ?`
    ).run(id)
  }

  /**
   * ลบข้อมูลถาวร
   */
  hardDelete(id) {
    return this.prepare(
      `DELETE FROM ${this.tableName} WHERE id = ?`
    ).run(id)
  }

  /**
   * Transaction helper
   */
  transaction(fn) {
    return this.db.transaction(fn)()
  }

  /**
   * Cleanup prepared statements
   */
  cleanup() {
    this._preparedStatements.clear()
  }
}

module.exports = BaseRepository
```

## Note Repository

```javascript
// electron/database/repositories/NoteRepository.js
const BaseRepository = require('./BaseRepository')

class NoteRepository extends BaseRepository {
  constructor(db) {
    super(db, 'notes')
    this.setupStatements()
  }

  setupStatements() {
    // Prepared statements ที่ใช้บ่อย
    this._stmts = {
      findByUserId: this.db.prepare(`
        SELECT n.*, 
               GROUP_CONCAT(t.name, ',') as tagNames,
               GROUP_CONCAT(t.id, ',') as tagIds,
               GROUP_CONCAT(t.color, ',') as tagColors
        FROM notes n
        LEFT JOIN note_tags nt ON n.id = nt.noteId
        LEFT JOIN tags t ON nt.tagId = t.id
        WHERE n.userId = ? 
          AND n.deletedAt IS NULL
          AND n.isArchived = 0
        GROUP BY n.id
        ORDER BY n.isPinned DESC, n.updatedAt DESC
        LIMIT ? OFFSET ?
      `),

      findArchivedByUserId: this.db.prepare(`
        SELECT * FROM notes
        WHERE userId = ? AND isArchived = 1 AND deletedAt IS NULL
        ORDER BY updatedAt DESC
        LIMIT ? OFFSET ?
      `),

      findByTag: this.db.prepare(`
        SELECT n.* FROM notes n
        JOIN note_tags nt ON n.id = nt.noteId
        JOIN tags t ON nt.tagId = t.id
        WHERE n.userId = ? AND t.name = ? AND n.deletedAt IS NULL
        ORDER BY n.updatedAt DESC
      `),

      fullTextSearch: this.db.prepare(`
        SELECT n.*, snippet(notes_fts, 0, '<mark>', '</mark>', '...', 10) as titleSnippet,
               snippet(notes_fts, 1, '<mark>', '</mark>', '...', 20) as contentSnippet,
               rank
        FROM notes_fts
        JOIN notes n ON notes_fts.rowid = n.id
        WHERE notes_fts MATCH ? AND n.userId = ? AND n.deletedAt IS NULL
        ORDER BY rank
        LIMIT ?
      `),

      togglePin: this.db.prepare(`
        UPDATE notes SET isPinned = NOT isPinned, updatedAt = datetime('now')
        WHERE id = ? AND userId = ?
      `),

      toggleArchive: this.db.prepare(`
        UPDATE notes SET isArchived = NOT isArchived, updatedAt = datetime('now')
        WHERE id = ? AND userId = ?
      `),

      addTag: this.db.prepare(`
        INSERT OR IGNORE INTO note_tags (noteId, tagId) VALUES (?, ?)
      `),

      removeTag: this.db.prepare(`
        DELETE FROM note_tags WHERE noteId = ? AND tagId = ?
      `),

      removeTags: this.db.prepare(`
        DELETE FROM note_tags WHERE noteId = ?
      `),

      getNoteTags: this.db.prepare(`
        SELECT t.* FROM tags t
        JOIN note_tags nt ON t.id = nt.tagId
        WHERE nt.noteId = ?
      `)
    }
  }

  /**
   * สร้าง note ใหม่
   */
  createNote({ userId, title = 'Untitled', content = '', color = null }) {
    const result = this.db.prepare(`
      INSERT INTO notes (userId, title, content, color)
      VALUES (?, ?, ?, ?)
    `).run(userId, title, content, color)
    
    return this.findById(result.lastInsertRowid)
  }

  /**
   * ดึง notes ของ user
   */
  getUserNotes(userId, { limit = 50, offset = 0 } = {}) {
    const rows = this._stmts.findByUserId.all(userId, limit, offset)
    
    // Parse tags จาก concatenated strings
    return rows.map(row => ({
      ...row,
      tags: row.tagIds 
        ? row.tagIds.split(',').map((id, i) => ({
            id: parseInt(id),
            name: row.tagNames.split(',')[i],
            color: row.tagColors.split(',')[i]
          }))
        : []
    }))
  }

  /**
   * อัปเดต note
   */
  updateNote(id, userId, updates) {
    const allowedFields = ['title', 'content', 'color', 'isPinned', 'isArchived']
    const safeUpdates = {}
    
    for (const [key, value] of Object.entries(updates)) {
      if (allowedFields.includes(key)) {
        safeUpdates[key] = value
      }
    }
    
    if (Object.keys(safeUpdates).length === 0) {
      return this.findById(id)
    }
    
    const setClauses = Object.keys(safeUpdates)
      .map(k => `${k} = @${k}`)
      .join(', ')
    
    this.db.prepare(`
      UPDATE notes 
      SET ${setClauses}, updatedAt = datetime('now')
      WHERE id = @id AND userId = @userId AND deletedAt IS NULL
    `).run({ ...safeUpdates, id, userId })
    
    return this.findById(id)
  }

  /**
   * ค้นหา full-text
   */
  searchNotes(userId, query, limit = 20) {
    if (!query || query.trim().length < 2) return []
    
    try {
      // FTS5 query syntax
      const ftsQuery = query.trim()
        .split(/\s+/)
        .map(term => `"${term.replace(/"/g, '')}"`)
        .join(' OR ')
      
      return this._stmts.fullTextSearch.all(ftsQuery, userId, limit)
    } catch (error) {
      // Fallback to LIKE search ถ้า FTS ล้มเหลว
      console.warn('FTS failed, using LIKE search:', error.message)
      return this.db.prepare(`
        SELECT * FROM notes
        WHERE userId = ? AND deletedAt IS NULL
        AND (title LIKE ? OR content LIKE ?)
        LIMIT ?
      `).all(userId, `%${query}%`, `%${query}%`, limit)
    }
  }

  /**
   * Set tags ของ note (replace all)
   */
  setNoteTags(noteId, tagIds) {
    return this.db.transaction(() => {
      this._stmts.removeTags.run(noteId)
      for (const tagId of tagIds) {
        this._stmts.addTag.run(noteId, tagId)
      }
      return this._stmts.getNoteTags.all(noteId)
    })()
  }

  /**
   * ดึง notes ที่ถูกลบในช่วง 30 วัน
   */
  getTrash(userId) {
    return this.db.prepare(`
      SELECT * FROM notes
      WHERE userId = ? AND deletedAt IS NOT NULL
      AND deletedAt > datetime('now', '-30 days')
      ORDER BY deletedAt DESC
    `).all(userId)
  }

  /**
   * กู้คืน note จาก trash
   */
  restoreNote(id, userId) {
    return this.db.prepare(`
      UPDATE notes SET deletedAt = NULL, updatedAt = datetime('now')
      WHERE id = ? AND userId = ?
    `).run(id, userId)
  }

  /**
   * ล้าง trash ที่เก่ากว่า 30 วัน
   */
  emptyOldTrash() {
    return this.db.prepare(`
      DELETE FROM notes
      WHERE deletedAt IS NOT NULL
      AND deletedAt < datetime('now', '-30 days')
    `).run()
  }

  /**
   * นับ notes ของ user
   */
  countUserNotes(userId) {
    return this.db.prepare(`
      SELECT 
        COUNT(*) FILTER (WHERE isArchived = 0 AND deletedAt IS NULL) as active,
        COUNT(*) FILTER (WHERE isArchived = 1 AND deletedAt IS NULL) as archived,
        COUNT(*) FILTER (WHERE deletedAt IS NOT NULL) as deleted
      FROM notes
      WHERE userId = ?
    `).get(userId)
  }

  /**
   * Export notes เป็น JSON
   */
  exportNotes(userId) {
    const notes = this.db.prepare(`
      SELECT n.*, 
             json_group_array(json_object('id', t.id, 'name', t.name, 'color', t.color)) 
               FILTER (WHERE t.id IS NOT NULL) as tags
      FROM notes n
      LEFT JOIN note_tags nt ON n.id = nt.noteId
      LEFT JOIN tags t ON nt.tagId = t.id
      WHERE n.userId = ? AND n.deletedAt IS NULL
      GROUP BY n.id
      ORDER BY n.createdAt
    `).all(userId)

    return notes.map(note => ({
      ...note,
      tags: note.tags ? JSON.parse(note.tags) : []
    }))
  }
}

module.exports = NoteRepository
```

## Tag Repository

```javascript
// electron/database/repositories/TagRepository.js
const BaseRepository = require('./BaseRepository')

class TagRepository extends BaseRepository {
  constructor(db) {
    super(db, 'tags')
  }

  /**
   * สร้าง tag ใหม่ หรือหา existing
   */
  findOrCreate(userId, name, color = '#6366f1') {
    const existing = this.db.prepare(
      'SELECT * FROM tags WHERE userId = ? AND name = ? COLLATE NOCASE'
    ).get(userId, name)
    
    if (existing) return existing
    
    const result = this.db.prepare(
      'INSERT INTO tags (userId, name, color) VALUES (?, ?, ?)'
    ).run(userId, name.trim(), color)
    
    return this.findById(result.lastInsertRowid)
  }

  /**
   * ดึง tags ของ user พร้อม note count
   */
  getUserTagsWithCount(userId) {
    return this.db.prepare(`
      SELECT t.*, COUNT(nt.noteId) as noteCount
      FROM tags t
      LEFT JOIN note_tags nt ON t.id = nt.tagId
      LEFT JOIN notes n ON nt.noteId = n.id AND n.deletedAt IS NULL
      WHERE t.userId = ?
      GROUP BY t.id
      ORDER BY t.name COLLATE NOCASE
    `).all(userId)
  }

  /**
   * เปลี่ยนชื่อ tag
   */
  renameTag(id, userId, newName) {
    return this.db.prepare(`
      UPDATE tags SET name = ? WHERE id = ? AND userId = ?
    `).run(newName.trim(), id, userId)
  }

  /**
   * ลบ tag (จะ cascade ลบ note_tags)
   */
  deleteTag(id, userId) {
    return this.db.prepare(
      'DELETE FROM tags WHERE id = ? AND userId = ?'
    ).run(id, userId)
  }
}

module.exports = TagRepository
```

## IPC Handlers สำหรับ Database

```javascript
// electron/ipc/databaseHandlers.js
const { ipcMain } = require('electron')
const dbManager = require('../database')
const NoteRepository = require('../database/repositories/NoteRepository')
const TagRepository = require('../database/repositories/TagRepository')

let noteRepo = null
let tagRepo = null

function setupDatabaseHandlers() {
  const db = dbManager.connect()
  noteRepo = new NoteRepository(db)
  tagRepo = new TagRepository(db)

  // Notes CRUD
  ipcMain.handle('notes:create', (event, data) => {
    return noteRepo.createNote(data)
  })

  ipcMain.handle('notes:getAll', (event, userId, options) => {
    return noteRepo.getUserNotes(userId, options)
  })

  ipcMain.handle('notes:getById', (event, id) => {
    return noteRepo.findById(id)
  })

  ipcMain.handle('notes:update', (event, id, userId, updates) => {
    return noteRepo.updateNote(id, userId, updates)
  })

  ipcMain.handle('notes:delete', (event, id, userId) => {
    // Soft delete
    const db = dbManager.getConnection()
    return db.prepare(`
      UPDATE notes SET deletedAt = datetime('now') WHERE id = ? AND userId = ?
    `).run(id, userId)
  })

  ipcMain.handle('notes:restore', (event, id, userId) => {
    return noteRepo.restoreNote(id, userId)
  })

  ipcMain.handle('notes:search', (event, userId, query, limit) => {
    return noteRepo.searchNotes(userId, query, limit)
  })

  ipcMain.handle('notes:setTags', (event, noteId, tagIds) => {
    return noteRepo.setNoteTags(noteId, tagIds)
  })

  ipcMain.handle('notes:getTrash', (event, userId) => {
    return noteRepo.getTrash(userId)
  })

  ipcMain.handle('notes:export', (event, userId) => {
    return noteRepo.exportNotes(userId)
  })

  ipcMain.handle('notes:count', (event, userId) => {
    return noteRepo.countUserNotes(userId)
  })

  // Tags CRUD
  ipcMain.handle('tags:findOrCreate', (event, userId, name, color) => {
    return tagRepo.findOrCreate(userId, name, color)
  })

  ipcMain.handle('tags:getAll', (event, userId) => {
    return tagRepo.getUserTagsWithCount(userId)
  })

  ipcMain.handle('tags:rename', (event, id, userId, newName) => {
    return tagRepo.renameTag(id, userId, newName)
  })

  ipcMain.handle('tags:delete', (event, id, userId) => {
    return tagRepo.deleteTag(id, userId)
  })

  // Database operations
  ipcMain.handle('db:stats', () => {
    return dbManager.getStats()
  })

  ipcMain.handle('db:backup', async (event, destination) => {
    await dbManager.backup(destination)
    return { success: true, destination }
  })

  // Transactions example
  ipcMain.handle('notes:batchCreate', (event, userId, notes) => {
    const db = dbManager.getConnection()
    
    const createBatch = db.transaction((notesList) => {
      const results = []
      for (const note of notesList) {
        const result = noteRepo.createNote({ userId, ...note })
        results.push(result)
      }
      return results
    })
    
    return createBatch(notes)
  })
}

module.exports = { setupDatabaseHandlers }
```

## Async Patterns สำหรับ Renderer

```javascript
// src/services/notesService.js
// Service layer ใน renderer process

class NotesService {
  async createNote(data) {
    return window.electronAPI.invoke('notes:create', data)
  }

  async getNotes(userId, options = {}) {
    return window.electronAPI.invoke('notes:getAll', userId, options)
  }

  async getNoteById(id) {
    return window.electronAPI.invoke('notes:getById', id)
  }

  async updateNote(id, userId, updates) {
    return window.electronAPI.invoke('notes:update', id, userId, updates)
  }

  async deleteNote(id, userId) {
    return window.electronAPI.invoke('notes:delete', id, userId)
  }

  async searchNotes(userId, query) {
    if (!query || query.length < 2) return []
    return window.electronAPI.invoke('notes:search', userId, query)
  }

  async exportNotes(userId) {
    const notes = await window.electronAPI.invoke('notes:export', userId)
    return JSON.stringify(notes, null, 2)
  }
}

export default new NotesService()
```

## Full-Text Search Example

```javascript
// ตัวอย่างการใช้ Full-Text Search

// Basic search
const results = noteRepo.searchNotes(userId, 'electron app')

// Advanced FTS5 query
const advancedResults = db.prepare(`
  SELECT n.*, 
         highlight(notes_fts, 0, '<b>', '</b>') as titleHighlight,
         highlight(notes_fts, 1, '<b>', '</b>') as contentHighlight,
         bm25(notes_fts) as relevance
  FROM notes_fts
  JOIN notes n ON notes_fts.rowid = n.id
  WHERE notes_fts MATCH 'title: electron AND content: ipc'
  AND n.userId = ?
  ORDER BY relevance
`).all(userId)

// Phrase search
const phraseSearch = db.prepare(`
  SELECT * FROM notes_fts
  WHERE notes_fts MATCH '"electron app"'
`).all()

// Proximity search (ค้นหาคำที่อยู่ใกล้กัน)
const proximitySearch = db.prepare(`
  SELECT * FROM notes_fts
  WHERE notes_fts MATCH 'NEAR(electron app, 10)'
`).all()
```

## Transaction Patterns

```javascript
// Complex transaction ที่มี rollback
function transferNote(db, noteId, fromUserId, toUserId) {
  const transferTx = db.transaction((noteId, fromId, toId) => {
    // ตรวจสอบว่า note มีอยู่จริง
    const note = db.prepare('SELECT * FROM notes WHERE id = ? AND userId = ?')
      .get(noteId, fromId)
    
    if (!note) throw new Error('Note not found')
    
    // อัปเดต userId
    db.prepare('UPDATE notes SET userId = ? WHERE id = ?')
      .run(toId, noteId)
    
    // Log การ transfer
    db.prepare(`
      INSERT INTO transfer_log (noteId, fromUserId, toUserId, transferredAt)
      VALUES (?, ?, ?, datetime('now'))
    `).run(noteId, fromId, toId)
    
    return db.prepare('SELECT * FROM notes WHERE id = ?').get(noteId)
  })
  
  try {
    return transferTx(noteId, fromUserId, toUserId)
  } catch (error) {
    console.error('Transfer failed, rolling back:', error)
    throw error
  }
}

// Savepoint example
function complexOperation(db) {
  db.exec('SAVEPOINT sp1')
  try {
    db.prepare('UPDATE notes SET title = ? WHERE id = ?').run('New Title', 1)
    
    db.exec('SAVEPOINT sp2')
    try {
      db.prepare('DELETE FROM note_tags WHERE noteId = ?').run(1)
      db.exec('RELEASE SAVEPOINT sp2')
    } catch {
      db.exec('ROLLBACK TO SAVEPOINT sp2')
    }
    
    db.exec('RELEASE SAVEPOINT sp1')
  } catch {
    db.exec('ROLLBACK TO SAVEPOINT sp1')
  }
}
```

## สรุป

### ข้อดีของ better-sqlite3
- **Synchronous API** - ง่ายกว่า async แต่ยังเร็วมาก
- **Type safety** - ค่าที่ return มีประเภทที่ถูกต้อง
- **Memory efficient** - ไม่มี overhead ของ async I/O
- **Prepared statements cache** - ลด overhead ของ query parsing

### Best Practices
1. ใช้ **Prepared statements** เสมอเพื่อป้องกัน SQL injection
2. ตั้งค่า **WAL mode** สำหรับ concurrent reads
3. ใช้ **Transactions** สำหรับ batch operations
4. เปิด **Foreign keys** ตั้งแต่เริ่มต้น
5. สร้าง **Indexes** ที่ columns ที่ query บ่อย
6. ใช้ **Migrations** สำหรับ schema changes
