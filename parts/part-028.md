# Part 28: Native Modules ใน Electron

## บทนำ

Native modules (หรือ Node.js addons) คือ C/C++ extensions ที่ compile เป็น `.node` files ซึ่งให้ performance สูงและเข้าถึง OS APIs ที่ JavaScript ทำไม่ได้ การใช้ native modules ใน Electron ต้องการ rebuild เพราะ Electron ใช้ V8 version ต่างจาก Node.js

## การติดตั้งและ Rebuild

### electron-rebuild

```bash
# ติดตั้ง electron-rebuild
npm install --save-dev electron-rebuild @electron/rebuild

# Rebuild modules ทั้งหมด
./node_modules/.bin/electron-rebuild

# หรือใช้ npx
npx electron-rebuild

# Rebuild สำหรับ platform เฉพาะ
npx electron-rebuild -f -w better-sqlite3
```

### Package.json Scripts

```json
{
  "scripts": {
    "postinstall": "electron-rebuild",
    "rebuild": "electron-rebuild -f",
    "rebuild:specific": "electron-rebuild -f -w"
  }
}
```

### electron-builder Configuration

```json
{
  "build": {
    "npmRebuild": true,
    "nativeRebuilder": "legacy"
  }
}
```

## better-sqlite3

```bash
npm install better-sqlite3
npm install --save-dev electron-rebuild
```

```javascript
// electron/database/sqlite.js
const Database = require('better-sqlite3')
const path = require('path')
const { app } = require('electron')

const dbPath = path.join(app.getPath('userData'), 'data.db')
const db = new Database(dbPath)

// ใช้งาน
db.pragma('journal_mode = WAL')
db.prepare('CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)').run()
db.prepare('INSERT INTO users (name) VALUES (?)').run('Alice')

const users = db.prepare('SELECT * FROM users').all()
console.log(users) // [{ id: 1, name: 'Alice' }]

db.close()
```

## Sharp - Image Processing

```bash
npm install sharp
```

```javascript
// electron/services/imageProcessor.js
const sharp = require('sharp')
const path = require('path')
const fs = require('fs')
const { ipcMain, dialog, app } = require('electron')

class ImageProcessor {
  /**
   * Resize รูปภาพ
   */
  async resize(inputPath, outputPath, options = {}) {
    const {
      width,
      height,
      fit = 'cover',        // cover | contain | fill | inside | outside
      position = 'centre',
      quality = 80,
      format = 'jpeg'
    } = options

    try {
      let pipeline = sharp(inputPath)
      
      if (width || height) {
        pipeline = pipeline.resize(width, height, { fit, position })
      }

      // เลือก format
      switch (format.toLowerCase()) {
        case 'jpeg':
        case 'jpg':
          pipeline = pipeline.jpeg({ quality, progressive: true })
          break
        case 'png':
          pipeline = pipeline.png({ compressionLevel: 6 })
          break
        case 'webp':
          pipeline = pipeline.webp({ quality })
          break
        case 'avif':
          pipeline = pipeline.avif({ quality })
          break
        default:
          pipeline = pipeline.jpeg({ quality })
      }

      await pipeline.toFile(outputPath)
      
      const stats = await fs.promises.stat(outputPath)
      const metadata = await sharp(outputPath).metadata()
      
      return {
        success: true,
        outputPath,
        size: stats.size,
        width: metadata.width,
        height: metadata.height,
        format: metadata.format
      }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  /**
   * สร้าง thumbnail
   */
  async createThumbnail(inputPath, size = 200) {
    const ext = path.extname(inputPath)
    const basename = path.basename(inputPath, ext)
    const dir = path.dirname(inputPath)
    const thumbPath = path.join(dir, `${basename}_thumb${ext}`)

    return this.resize(inputPath, thumbPath, {
      width: size,
      height: size,
      fit: 'cover',
      quality: 70
    })
  }

  /**
   * Compress รูปภาพ
   */
  async compress(inputPath, outputPath, targetSizeKB = 200) {
    try {
      const originalStats = await fs.promises.stat(inputPath)
      const originalSizeKB = originalStats.size / 1024

      if (originalSizeKB <= targetSizeKB) {
        // ไม่จำเป็นต้อง compress
        await fs.promises.copyFile(inputPath, outputPath)
        return { 
          success: true, 
          compressed: false,
          originalSize: originalStats.size,
          newSize: originalStats.size
        }
      }

      // Calculate quality ที่เหมาะสม
      const ratio = targetSizeKB / originalSizeKB
      const quality = Math.max(20, Math.min(90, Math.floor(ratio * 90)))

      const metadata = await sharp(inputPath).metadata()
      let pipeline = sharp(inputPath)

      if (metadata.format === 'jpeg' || metadata.format === 'jpg') {
        pipeline = pipeline.jpeg({ quality, progressive: true })
      } else if (metadata.format === 'png') {
        // Convert PNG ไปเป็น JPEG เพื่อ compression ที่ดีขึ้น
        pipeline = pipeline.jpeg({ quality })
        outputPath = outputPath.replace(/\.png$/, '.jpg')
      } else {
        pipeline = pipeline.webp({ quality })
      }

      await pipeline.toFile(outputPath)
      
      const newStats = await fs.promises.stat(outputPath)
      
      return {
        success: true,
        compressed: true,
        originalSize: originalStats.size,
        newSize: newStats.size,
        ratio: newStats.size / originalStats.size,
        outputPath
      }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  /**
   * Extract metadata
   */
  async getMetadata(imagePath) {
    try {
      const metadata = await sharp(imagePath).metadata()
      const stats = await sharp(imagePath).stats()
      
      return {
        success: true,
        metadata: {
          format: metadata.format,
          width: metadata.width,
          height: metadata.height,
          channels: metadata.channels,
          hasAlpha: metadata.hasAlpha,
          density: metadata.density,
          colorSpace: metadata.space,
          fileSize: (await fs.promises.stat(imagePath)).size,
          exif: metadata.exif ? parseExif(metadata.exif) : null
        },
        stats: {
          isOpaque: stats.isOpaque,
          entropy: stats.entropy,
          channels: stats.channels
        }
      }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  /**
   * Convert format
   */
  async convert(inputPath, outputFormat) {
    const ext = path.extname(inputPath)
    const outputPath = inputPath.replace(ext, `.${outputFormat}`)
    
    return this.resize(inputPath, outputPath, { format: outputFormat })
  }

  /**
   * Apply watermark
   */
  async applyWatermark(inputPath, watermarkPath, outputPath, options = {}) {
    const { gravity = 'southeast', opacity = 0.5 } = options
    
    try {
      const watermark = await sharp(watermarkPath)
        .composite([{
          input: Buffer.from([255, 255, 255, Math.floor(opacity * 255)]),
          raw: { width: 1, height: 1, channels: 4 },
          tile: true,
          blend: 'dest-in'
        }])
        .toBuffer()

      await sharp(inputPath)
        .composite([{
          input: watermark,
          gravity,
          blend: 'over'
        }])
        .toFile(outputPath)

      return { success: true, outputPath }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  /**
   * Batch processing
   */
  async batchProcess(files, operation, options = {}) {
    const results = []
    
    for (const file of files) {
      let result
      
      switch (operation) {
        case 'thumbnail':
          result = await this.createThumbnail(file, options.size)
          break
        case 'compress':
          const outputPath = file.replace(
            /(\.\w+)$/, 
            `_compressed$1`
          )
          result = await this.compress(file, outputPath, options.targetSizeKB)
          break
        case 'convert':
          result = await this.convert(file, options.format)
          break
        default:
          result = { success: false, error: 'Unknown operation' }
      }
      
      results.push({ file, ...result })
    }
    
    return results
  }

  setupIpcHandlers() {
    ipcMain.handle('image:resize', async (event, inputPath, outputPath, options) => {
      return this.resize(inputPath, outputPath, options)
    })

    ipcMain.handle('image:thumbnail', async (event, inputPath, size) => {
      return this.createThumbnail(inputPath, size)
    })

    ipcMain.handle('image:compress', async (event, inputPath, outputPath, targetSizeKB) => {
      return this.compress(inputPath, outputPath, targetSizeKB)
    })

    ipcMain.handle('image:metadata', async (event, imagePath) => {
      return this.getMetadata(imagePath)
    })

    ipcMain.handle('image:convert', async (event, inputPath, format) => {
      return this.convert(inputPath, format)
    })

    ipcMain.handle('image:batchProcess', async (event, files, operation, options) => {
      return this.batchProcess(files, operation, options)
    })
  }
}

function parseExif(buffer) {
  // Basic EXIF parsing
  try {
    return { raw: buffer.toString('base64').substring(0, 100) }
  } catch {
    return null
  }
}

module.exports = new ImageProcessor()
```

## bcrypt สำหรับ Password Hashing

```bash
npm install bcrypt
# หรือ pure JS implementation
npm install bcryptjs
```

```javascript
// electron/services/authService.js
const bcrypt = require('bcrypt')
const crypto = require('crypto')
const Database = require('better-sqlite3')

const SALT_ROUNDS = 12

class AuthService {
  constructor(db) {
    this.db = db
    this.setupTables()
    this.setupStatements()
  }

  setupTables() {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS users (
        id            INTEGER PRIMARY KEY AUTOINCREMENT,
        username      TEXT    NOT NULL UNIQUE,
        email         TEXT    UNIQUE,
        passwordHash  TEXT    NOT NULL,
        salt          TEXT,
        lastLogin     TEXT,
        loginAttempts INTEGER DEFAULT 0,
        lockedUntil   TEXT,
        createdAt     TEXT    DEFAULT (datetime('now'))
      );

      CREATE TABLE IF NOT EXISTS sessions (
        id        TEXT    PRIMARY KEY,
        userId    INTEGER NOT NULL,
        expiresAt TEXT    NOT NULL,
        createdAt TEXT    DEFAULT (datetime('now')),
        FOREIGN KEY (userId) REFERENCES users(id)
      );
    `)
  }

  setupStatements() {
    this.stmts = {
      findByUsername: this.db.prepare(
        'SELECT * FROM users WHERE username = ?'
      ),
      createUser: this.db.prepare(`
        INSERT INTO users (username, email, passwordHash) VALUES (?, ?, ?)
      `),
      updateLastLogin: this.db.prepare(`
        UPDATE users SET lastLogin = datetime('now'), loginAttempts = 0
        WHERE id = ?
      `),
      incrementAttempts: this.db.prepare(`
        UPDATE users SET loginAttempts = loginAttempts + 1,
        lockedUntil = CASE 
          WHEN loginAttempts >= 4 THEN datetime('now', '+15 minutes')
          ELSE lockedUntil 
        END
        WHERE id = ?
      `),
      createSession: this.db.prepare(`
        INSERT INTO sessions (id, userId, expiresAt) VALUES (?, ?, ?)
      `),
      findSession: this.db.prepare(`
        SELECT s.*, u.username, u.email 
        FROM sessions s
        JOIN users u ON s.userId = u.id
        WHERE s.id = ? AND s.expiresAt > datetime('now')
      `),
      deleteSession: this.db.prepare(
        'DELETE FROM sessions WHERE id = ?'
      )
    }
  }

  /**
   * Hash password
   */
  async hashPassword(password) {
    return bcrypt.hash(password, SALT_ROUNDS)
  }

  /**
   * Verify password
   */
  async verifyPassword(password, hash) {
    return bcrypt.compare(password, hash)
  }

  /**
   * สร้าง user ใหม่
   */
  async register(username, password, email = null) {
    // Validate
    if (!username || username.length < 3) {
      throw new Error('Username ต้องมีอย่างน้อย 3 ตัวอักษร')
    }
    if (!password || password.length < 8) {
      throw new Error('Password ต้องมีอย่างน้อย 8 ตัวอักษร')
    }

    // Check duplicate
    const existing = this.stmts.findByUsername.get(username)
    if (existing) {
      throw new Error('Username นี้ถูกใช้แล้ว')
    }

    const passwordHash = await this.hashPassword(password)
    const result = this.stmts.createUser.run(username, email, passwordHash)
    
    return { id: result.lastInsertRowid, username, email }
  }

  /**
   * Login
   */
  async login(username, password) {
    const user = this.stmts.findByUsername.get(username)
    
    if (!user) {
      // ใช้เวลา bcrypt.compare เพื่อป้องกัน timing attack
      await bcrypt.compare(password, '$2b$12$invalid_hash_for_timing')
      throw new Error('Username หรือ Password ไม่ถูกต้อง')
    }

    // ตรวจสอบว่า locked หรือไม่
    if (user.lockedUntil && new Date(user.lockedUntil) > new Date()) {
      const remaining = Math.ceil((new Date(user.lockedUntil) - new Date()) / 1000 / 60)
      throw new Error(`บัญชีถูกล็อค กรุณารอ ${remaining} นาที`)
    }

    const isValid = await this.verifyPassword(password, user.passwordHash)
    
    if (!isValid) {
      this.stmts.incrementAttempts.run(user.id)
      throw new Error('Username หรือ Password ไม่ถูกต้อง')
    }

    // อัปเดต last login
    this.stmts.updateLastLogin.run(user.id)

    // สร้าง session
    const sessionId = this.generateSessionId()
    const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 วัน
    
    this.stmts.createSession.run(sessionId, user.id, expiresAt.toISOString())

    return {
      sessionId,
      user: {
        id: user.id,
        username: user.username,
        email: user.email
      }
    }
  }

  /**
   * Verify session
   */
  verifySession(sessionId) {
    return this.stmts.findSession.get(sessionId)
  }

  /**
   * Logout
   */
  logout(sessionId) {
    this.stmts.deleteSession.run(sessionId)
  }

  generateSessionId() {
    return crypto.randomBytes(32).toString('hex')
  }
}

module.exports = AuthService
```

## node-gyp และสร้าง Native Module

```cpp
// addons/fast_hash/fast_hash.cpp
#include <node.h>
#include <v8.h>
#include <string>
#include <sstream>
#include <iomanip>

using namespace v8;

// Simple FNV-1a hash implementation
uint64_t fnv1a_hash(const std::string& data) {
    uint64_t hash = 14695981039346656037ULL;
    for (char c : data) {
        hash ^= static_cast<uint64_t>(c);
        hash *= 1099511628211ULL;
    }
    return hash;
}

// Function ที่ expose ไปยัง JavaScript
void Hash(const FunctionCallbackInfo<Value>& args) {
    Isolate* isolate = args.GetIsolate();
    
    if (args.Length() < 1 || !args[0]->IsString()) {
        isolate->ThrowException(
            Exception::TypeError(
                String::NewFromUtf8(isolate, "Expected string argument").ToLocalChecked()
            )
        );
        return;
    }
    
    String::Utf8Value str(isolate, args[0]);
    std::string input(*str);
    
    uint64_t hash = fnv1a_hash(input);
    
    // Convert to hex string
    std::ostringstream oss;
    oss << std::hex << std::setw(16) << std::setfill('0') << hash;
    
    args.GetReturnValue().Set(
        String::NewFromUtf8(isolate, oss.str().c_str()).ToLocalChecked()
    );
}

void Init(Local<Object> exports) {
    NODE_SET_METHOD(exports, "hash", Hash);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Init)
```

```python
# addons/fast_hash/binding.gyp
{
  "targets": [
    {
      "target_name": "fast_hash",
      "sources": ["fast_hash.cpp"],
      "include_dirs": [
        "<!@(node -p \"require('node-addon-api').include\")"
      ],
      "dependencies": [
        "<!(node -p \"require('node-addon-api').gyp\")"
      ],
      "cflags!": ["-fno-exceptions"],
      "cflags_cc!": ["-fno-exceptions"],
      "defines": ["NAPI_DISABLE_CPP_EXCEPTIONS"]
    }
  ]
}
```

## NAPI Module (Modern Approach)

```cpp
// addons/file_watcher/file_watcher.cpp
// NAPI-based file watcher
#include <napi.h>
#include <uv.h>
#include <string>
#include <map>
#include <functional>

class FileWatcher : public Napi::ObjectWrap<FileWatcher> {
public:
    static Napi::Object Init(Napi::Env env, Napi::Object exports) {
        Napi::Function func = DefineClass(env, "FileWatcher", {
            InstanceMethod("watch", &FileWatcher::Watch),
            InstanceMethod("unwatch", &FileWatcher::Unwatch),
            InstanceMethod("close", &FileWatcher::Close)
        });
        
        exports.Set("FileWatcher", func);
        return exports;
    }
    
    FileWatcher(const Napi::CallbackInfo& info)
        : Napi::ObjectWrap<FileWatcher>(info) {}

private:
    Napi::Value Watch(const Napi::CallbackInfo& info) {
        Napi::Env env = info.Env();
        
        if (info.Length() < 2 || !info[0].IsString() || !info[1].IsFunction()) {
            Napi::TypeError::New(env, "Expected (path: string, callback: function)")
                .ThrowAsJavaScriptException();
            return env.Null();
        }
        
        std::string path = info[0].As<Napi::String>().Utf8Value();
        // ... implementation
        
        return Napi::Boolean::New(env, true);
    }
    
    Napi::Value Unwatch(const Napi::CallbackInfo& info) {
        return Napi::Boolean::New(info.Env(), true);
    }
    
    Napi::Value Close(const Napi::CallbackInfo& info) {
        return info.Env().Undefined();
    }
};

Napi::Object Init(Napi::Env env, Napi::Object exports) {
    return FileWatcher::Init(env, exports);
}

NODE_API_MODULE(file_watcher, Init)
```

## Troubleshooting Native Modules

```javascript
// scripts/check-native-modules.js
const { execSync } = require('child_process')
const path = require('path')
const fs = require('fs')

const electronVersion = require('electron/package.json').version
const nodeVersion = process.versions.node

console.log('=== Native Module Check ===')
console.log('Node.js version:', nodeVersion)
console.log('Electron version:', electronVersion)
console.log('Platform:', process.platform, process.arch)

const nativeModules = [
  'better-sqlite3',
  'bcrypt',
  'sharp',
  'keytar'
]

nativeModules.forEach(moduleName => {
  const modulePath = path.join('node_modules', moduleName)
  
  if (!fs.existsSync(modulePath)) {
    console.log(`❌ ${moduleName}: NOT INSTALLED`)
    return
  }
  
  try {
    require(modulePath)
    console.log(`✅ ${moduleName}: OK`)
  } catch (error) {
    console.log(`❌ ${moduleName}: FAILED`)
    console.log(`   Error: ${error.message}`)
    console.log(`   Run: npx electron-rebuild -f -w ${moduleName}`)
  }
})
```

### Common Errors และวิธีแก้

```bash
# Error: The module was compiled against a different Node.js version
# Solution: Rebuild native modules
npx electron-rebuild

# Error: Cannot find module 'better-sqlite3'
# Solution: Install and rebuild
npm install better-sqlite3
npx electron-rebuild -f -w better-sqlite3

# Error: node-gyp fails on Windows
# Solution: Install build tools
npm install --global windows-build-tools
# หรือ
npm install --global --production windows-build-tools

# Error: node-pre-gyp fails
# Solution: Force rebuild from source
npm install --build-from-source
npx electron-rebuild --build-from-source

# Error: Python not found
# Solution: Specify Python path
npm config set python /usr/bin/python3
```

### electron-rebuild Configuration

```javascript
// .electronrc.js หรือใน package.json
{
  "electronRebuildConfig": {
    "onlyModules": ["better-sqlite3", "bcrypt", "sharp"],
    "arch": "x64",
    "headerDir": "./dist/headers",
    "buildFromSource": false
  }
}
```

## Prebuilt Binaries

```javascript
// package.json - ใช้ prebuilt binaries
{
  "dependencies": {
    "better-sqlite3": "^9.0.0"
  },
  "better-sqlite3": {
    "binary": {
      "module_name": "better_sqlite3",
      "module_path": "./build/Release",
      "host": "https://github.com/WiseLibs/better-sqlite3/releases/download/"
    }
  }
}
```

## Integration ใน Electron App

```javascript
// electron/main.js - การ load native modules

// ใน development: โหลดตรงๆ
// ใน production: ต้องใช้ path ที่ถูกต้อง

function loadNativeModule(moduleName) {
  try {
    // ลอง load ปกติก่อน
    return require(moduleName)
  } catch (firstError) {
    // ถ้าล้มเหลว ลอง path ใน app resources
    try {
      const appPath = app.isPackaged
        ? process.resourcesPath
        : path.join(__dirname, '../')
      
      return require(path.join(appPath, 'node_modules', moduleName))
    } catch (secondError) {
      console.error(`Failed to load native module: ${moduleName}`, secondError)
      throw secondError
    }
  }
}

// ใช้งาน
let sharp, Database, bcrypt

try {
  sharp = loadNativeModule('sharp')
  Database = loadNativeModule('better-sqlite3')
  bcrypt = loadNativeModule('bcrypt')
} catch (error) {
  console.error('Native modules unavailable:', error)
  // Fallback ไปยัง pure JS implementations ถ้ามี
}
```

## สรุป

### Native Modules ที่นิยมใช้ใน Electron

| Module | ใช้สำหรับ | Notes |
|--------|-----------|-------|
| better-sqlite3 | SQLite database | เร็ว, synchronous |
| bcrypt | Password hashing | ปลอดภัย |
| sharp | Image processing | เร็วมาก |
| keytar | Password storage | OS keychain |
| node-gyp | Build native addons | Dev tool |
| @serialport/bindings-cpp | Serial port | Hardware |

### Best Practices

1. ใช้ **prebuilt binaries** เมื่อทำได้เพื่อลดเวลา build
2. ตั้งค่า **postinstall** script สำหรับ electron-rebuild
3. **Test ทุก platform** (Windows, macOS, Linux)
4. ใช้ **NAPI** สำหรับ custom modules ใหม่ (compat ดีกว่า)
5. มี **fallback** สำหรับ platforms ที่ไม่รองรับ
