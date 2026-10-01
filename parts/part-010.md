# ตอนที่ 10: File System Operations - การจัดการไฟล์

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจากศึกษาตอนนี้ คุณจะสามารถ:
- ใช้ `fs` module ใน Node.js ได้อย่างมีประสิทธิภาพ
- อ่านและเขียนไฟล์ประเภทต่างๆ
- ใช้ `fs.watch` เฝ้าดูการเปลี่ยนแปลงของไฟล์
- ใช้ `path` module จัดการ paths อย่างถูกต้อง
- ดึง paths พิเศษด้วย `app.getPath()`
- อ่านไฟล์ขนาดใหญ่ด้วย stream
- เชื่อมการทำงานกับ file dialog
- รองรับ drag & drop files
- จัดการชนิดไฟล์และ MIME types

---

## บทนำ: File System ใน Electron

```
Electron File System Stack
├── Main Process
│   ├── fs module (Node.js built-in)
│   ├── path module (Node.js built-in)
│   ├── app.getPath() (Electron API)
│   └── dialog (open/save file)
│
└── Renderer Process
    ├── Web File API (HTML5)
    ├── Drag & Drop API
    └── IPC → Main Process (สำหรับ Node.js fs)
```

---

## 10.1 fs Module - การอ่าน/เขียนไฟล์พื้นฐาน

### โครงสร้างโปรเจ็กต์
```
file-manager-app/
├── main.js
├── preload.js
├── renderer.js
├── index.html
├── fileUtils.js      # utility functions
└── package.json
```

### fileUtils.js - Utility Functions
```javascript
// fileUtils.js - ฟังก์ชันช่วยสำหรับ file operations

const fs = require('fs')
const fsp = require('fs').promises  // Promises version
const path = require('path')
const { app } = require('electron')

// ==========================================
// อ่านไฟล์ - หลายรูปแบบ
// ==========================================

// อ่านไฟล์ข้อความ (async/await)
async function readTextFile(filePath, encoding = 'utf-8') {
  try {
    const content = await fsp.readFile(filePath, encoding)
    return { success: true, content }
  } catch (error) {
    return { success: false, error: error.message, code: error.code }
  }
}

// อ่านไฟล์ JSON
async function readJsonFile(filePath) {
  try {
    const content = await fsp.readFile(filePath, 'utf-8')
    const data = JSON.parse(content)
    return { success: true, data }
  } catch (error) {
    if (error instanceof SyntaxError) {
      return { success: false, error: `JSON ไม่ถูกต้อง: ${error.message}` }
    }
    return { success: false, error: error.message }
  }
}

// อ่านไฟล์ binary (รูปภาพ, PDF เป็นต้น)
async function readBinaryFile(filePath) {
  try {
    const buffer = await fsp.readFile(filePath)
    return { success: true, buffer, size: buffer.length }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// อ่านไฟล์เป็น base64 (สำหรับส่งไป renderer)
async function readFileAsBase64(filePath) {
  try {
    const buffer = await fsp.readFile(filePath)
    const base64 = buffer.toString('base64')
    const ext = path.extname(filePath).slice(1).toLowerCase()
    const mimeType = getMimeType(ext)
    return {
      success: true,
      dataUrl: `data:${mimeType};base64,${base64}`,
      size: buffer.length
    }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// ==========================================
// เขียนไฟล์
// ==========================================

// เขียนไฟล์ข้อความ
async function writeTextFile(filePath, content, encoding = 'utf-8') {
  try {
    // สร้าง directory ถ้าไม่มี
    await fsp.mkdir(path.dirname(filePath), { recursive: true })
    await fsp.writeFile(filePath, content, encoding)
    
    const stats = await fsp.stat(filePath)
    return {
      success: true,
      filePath,
      size: stats.size,
      modified: stats.mtime
    }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// เขียนไฟล์ JSON
async function writeJsonFile(filePath, data, pretty = true) {
  const content = pretty 
    ? JSON.stringify(data, null, 2)  // format สวย
    : JSON.stringify(data)
  return writeTextFile(filePath, content)
}

// เขียนไฟล์ binary
async function writeBinaryFile(filePath, buffer) {
  try {
    await fsp.mkdir(path.dirname(filePath), { recursive: true })
    await fsp.writeFile(filePath, buffer)
    return { success: true, filePath, size: buffer.length }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// เพิ่มข้อความต่อท้ายไฟล์ (append)
async function appendToFile(filePath, content) {
  try {
    await fsp.mkdir(path.dirname(filePath), { recursive: true })
    await fsp.appendFile(filePath, content, 'utf-8')
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// ==========================================
// File/Directory Management
// ==========================================

// ตรวจสอบว่าไฟล์/โฟลเดอร์มีอยู่
async function exists(filePath) {
  try {
    await fsp.access(filePath, fs.constants.F_OK)
    return true
  } catch {
    return false
  }
}

// ดู metadata ของไฟล์
async function getFileStats(filePath) {
  try {
    const stats = await fsp.stat(filePath)
    return {
      success: true,
      size: stats.size,
      created: stats.birthtime,
      modified: stats.mtime,
      accessed: stats.atime,
      isFile: stats.isFile(),
      isDirectory: stats.isDirectory(),
      permissions: stats.mode.toString(8)
    }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// คัดลอกไฟล์
async function copyFile(src, dest) {
  try {
    await fsp.mkdir(path.dirname(dest), { recursive: true })
    await fsp.copyFile(src, dest)
    return { success: true, src, dest }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// ย้ายไฟล์/เปลี่ยนชื่อ
async function moveFile(src, dest) {
  try {
    await fsp.mkdir(path.dirname(dest), { recursive: true })
    await fsp.rename(src, dest)
    return { success: true, src, dest }
  } catch (error) {
    // ถ้า rename ข้ามไดรฟ์ไม่ได้ ให้ copy แล้วลบ
    if (error.code === 'EXDEV') {
      const copyResult = await copyFile(src, dest)
      if (copyResult.success) {
        await fsp.unlink(src)
        return { success: true, src, dest }
      }
      return copyResult
    }
    return { success: false, error: error.message }
  }
}

// ลบไฟล์
async function deleteFile(filePath) {
  try {
    await fsp.unlink(filePath)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// รายชื่อไฟล์ในโฟลเดอร์
async function listDirectory(dirPath, options = {}) {
  try {
    const { 
      includeHidden = false, 
      extension = null,
      recursive = false 
    } = options
    
    const entries = await fsp.readdir(dirPath, { withFileTypes: true })
    
    let files = []
    
    for (const entry of entries) {
      // ข้าม hidden files ถ้าไม่ต้องการ
      if (!includeHidden && entry.name.startsWith('.')) continue
      
      const fullPath = path.join(dirPath, entry.name)
      
      if (entry.isDirectory() && recursive) {
        const subFiles = await listDirectory(fullPath, options)
        files = files.concat(subFiles.files || [])
      } else if (entry.isFile()) {
        // กรองตาม extension ถ้าระบุ
        if (extension && path.extname(entry.name).slice(1) !== extension) continue
        
        const stats = await fsp.stat(fullPath)
        files.push({
          name: entry.name,
          path: fullPath,
          size: stats.size,
          modified: stats.mtime,
          extension: path.extname(entry.name).slice(1)
        })
      }
    }
    
    return { success: true, files, count: files.length }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// ==========================================
// MIME Types
// ==========================================
function getMimeType(extension) {
  const mimeTypes = {
    // Text
    'txt': 'text/plain',
    'html': 'text/html',
    'css': 'text/css',
    'csv': 'text/csv',
    'xml': 'text/xml',
    'md': 'text/markdown',
    
    // Application
    'json': 'application/json',
    'pdf': 'application/pdf',
    'zip': 'application/zip',
    'js': 'application/javascript',
    
    // Image
    'jpg': 'image/jpeg',
    'jpeg': 'image/jpeg',
    'png': 'image/png',
    'gif': 'image/gif',
    'webp': 'image/webp',
    'svg': 'image/svg+xml',
    'ico': 'image/x-icon',
    'bmp': 'image/bmp',
    
    // Audio
    'mp3': 'audio/mpeg',
    'wav': 'audio/wav',
    'ogg': 'audio/ogg',
    'flac': 'audio/flac',
    
    // Video
    'mp4': 'video/mp4',
    'webm': 'video/webm',
    'avi': 'video/x-msvideo',
    'mov': 'video/quicktime',
    
    // Document
    'docx': 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    'xlsx': 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    'pptx': 'application/vnd.openxmlformats-officedocument.presentationml.presentation'
  }
  
  return mimeTypes[extension.toLowerCase()] || 'application/octet-stream'
}

module.exports = {
  readTextFile,
  readJsonFile,
  readBinaryFile,
  readFileAsBase64,
  writeTextFile,
  writeJsonFile,
  writeBinaryFile,
  appendToFile,
  exists,
  getFileStats,
  copyFile,
  moveFile,
  deleteFile,
  listDirectory,
  getMimeType
}
```

---

## 10.2 app.getPath() - Paths พิเศษใน Electron

```javascript
// main.js - การใช้ app.getPath()

const { app } = require('electron')
const path = require('path')

function getAppPaths() {
  return {
    // ==========================================
    // Paths ที่ใช้บ่อย
    // ==========================================
    
    // โฟลเดอร์ Documents ของผู้ใช้
    documents: app.getPath('documents'),
    // Windows: C:\Users\<user>\Documents
    // macOS: /Users/<user>/Documents
    // Linux: /home/<user>/Documents
    
    // Desktop
    desktop: app.getPath('desktop'),
    
    // Downloads
    downloads: app.getPath('downloads'),
    
    // Pictures
    pictures: app.getPath('pictures'),
    
    // Music
    music: app.getPath('music'),
    
    // Videos
    videos: app.getPath('videos'),
    
    // ==========================================
    // App-specific Paths
    // ==========================================
    
    // ที่เก็บข้อมูลแอป (userData)
    // Windows: %APPDATA%\<AppName>
    // macOS: ~/Library/Application Support/<AppName>
    // Linux: ~/.config/<AppName>
    userData: app.getPath('userData'),
    
    // App executable
    exe: app.getPath('exe'),
    
    // App directory
    appDir: app.getAppPath(),
    
    // ==========================================
    // System Paths
    // ==========================================
    
    // Temp directory
    temp: app.getPath('temp'),
    
    // Home directory
    home: app.getPath('home'),
    
    // ==========================================
    // macOS specific
    // ==========================================
    
    // Application Support (macOS)
    // appData: app.getPath('appData'),
    
    // Logs directory
    logs: app.getPath('logs'),
    
    // Crash dumps
    crashDumps: app.getPath('crashDumps')
  }
}

// สร้าง path ภายใน userData
function getUserDataPath(...parts) {
  return path.join(app.getPath('userData'), ...parts)
}

// ตัวอย่าง: สร้าง config file path
const configPath = getUserDataPath('config', 'settings.json')
const cachePath = getUserDataPath('cache')
const logPath = getUserDataPath('logs', `app-${new Date().toISOString().split('T')[0]}.log`)

console.log('Config path:', configPath)
console.log('Cache path:', cachePath)
console.log('Log path:', logPath)
```

---

## 10.3 fs.watch - เฝ้าดูการเปลี่ยนแปลงไฟล์

```javascript
// main.js - File Watching

const fs = require('fs')
const { ipcMain, BrowserWindow } = require('electron')

// ==========================================
// File Watcher Class
// ==========================================
class FileWatcher {
  constructor() {
    this.watchers = new Map()  // path → watcher
    this.debounceTimers = new Map()  // path → timer
    this.debounceDelay = 100  // ms
  }
  
  // เริ่ม watch ไฟล์/โฟลเดอร์
  watch(watchPath, options = {}) {
    if (this.watchers.has(watchPath)) {
      console.log(`กำลัง watch อยู่แล้ว: ${watchPath}`)
      return false
    }
    
    const { recursive = false, callback } = options
    
    try {
      const watcher = fs.watch(
        watchPath,
        { recursive, persistent: false },
        (eventType, filename) => {
          // debounce เพื่อลดการ fire ซ้ำ
          this._debounce(watchPath, () => {
            const data = {
              type: eventType,  // 'rename' หรือ 'change'
              path: watchPath,
              filename: filename || watchPath,
              timestamp: new Date().toISOString()
            }
            
            if (callback) callback(data)
            
            // แจ้ง renderer ทุก window
            BrowserWindow.getAllWindows().forEach(win => {
              if (!win.isDestroyed()) {
                win.webContents.send('file-changed', data)
              }
            })
          })
        }
      )
      
      watcher.on('error', (error) => {
        console.error(`Watch error สำหรับ ${watchPath}:`, error)
        this.unwatch(watchPath)
      })
      
      this.watchers.set(watchPath, watcher)
      console.log(`เริ่ม watch: ${watchPath}`)
      return true
    } catch (error) {
      console.error(`ไม่สามารถ watch ได้:`, error)
      return false
    }
  }
  
  // หยุด watch
  unwatch(watchPath) {
    const watcher = this.watchers.get(watchPath)
    if (watcher) {
      watcher.close()
      this.watchers.delete(watchPath)
      console.log(`หยุด watch: ${watchPath}`)
      return true
    }
    return false
  }
  
  // หยุด watch ทั้งหมด
  unwatchAll() {
    this.watchers.forEach((watcher, path) => {
      watcher.close()
    })
    this.watchers.clear()
    this.debounceTimers.forEach(timer => clearTimeout(timer))
    this.debounceTimers.clear()
  }
  
  // รายชื่อที่กำลัง watch
  getWatchedPaths() {
    return Array.from(this.watchers.keys())
  }
  
  // Debounce ป้องกัน fire ซ้ำๆ
  _debounce(key, fn) {
    if (this.debounceTimers.has(key)) {
      clearTimeout(this.debounceTimers.get(key))
    }
    
    const timer = setTimeout(() => {
      this.debounceTimers.delete(key)
      fn()
    }, this.debounceDelay)
    
    this.debounceTimers.set(key, timer)
  }
}

const fileWatcher = new FileWatcher()

// IPC Handlers สำหรับ file watching
ipcMain.handle('watch-path', async (event, { watchPath, recursive }) => {
  const success = fileWatcher.watch(watchPath, { recursive })
  return { success, watchPath }
})

ipcMain.handle('unwatch-path', async (event, watchPath) => {
  const success = fileWatcher.unwatch(watchPath)
  return { success }
})

ipcMain.handle('get-watched-paths', () => {
  return fileWatcher.getWatchedPaths()
})

// Cleanup เมื่อแอปปิด
app.on('before-quit', () => {
  fileWatcher.unwatchAll()
})
```

---

## 10.4 Stream Reading - อ่านไฟล์ขนาดใหญ่

```javascript
// main.js - Stream Reading

const fs = require('fs')
const readline = require('readline')
const { ipcMain, BrowserWindow } = require('electron')

// ==========================================
// อ่านไฟล์ใหญ่โดยไม่ block process
// ==========================================
ipcMain.handle('read-large-file', async (event, filePath) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  return new Promise((resolve, reject) => {
    const stats = fs.statSync(filePath)
    const totalSize = stats.size
    let bytesRead = 0
    let content = ''
    
    const stream = fs.createReadStream(filePath, {
      encoding: 'utf8',
      highWaterMark: 64 * 1024  // อ่านครั้งละ 64KB
    })
    
    stream.on('data', (chunk) => {
      content += chunk
      bytesRead += Buffer.byteLength(chunk, 'utf8')
      
      // ส่ง progress ไปที่ renderer
      const percent = Math.round((bytesRead / totalSize) * 100)
      win.webContents.send('read-progress', {
        filePath,
        percent,
        bytesRead,
        totalSize
      })
    })
    
    stream.on('end', () => {
      win.webContents.send('read-complete', { filePath, size: bytesRead })
      resolve({ success: true, content, size: bytesRead })
    })
    
    stream.on('error', (error) => {
      win.webContents.send('read-error', { filePath, error: error.message })
      reject(error)
    })
  })
})

// ==========================================
// อ่านไฟล์ CSV ทีละบรรทัด (Line by Line)
// ==========================================
ipcMain.handle('read-csv-streaming', async (event, filePath) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  return new Promise((resolve, reject) => {
    const rows = []
    let lineCount = 0
    let headers = null
    
    const fileStream = fs.createReadStream(filePath, { encoding: 'utf8' })
    
    const rl = readline.createInterface({
      input: fileStream,
      crlfDelay: Infinity  // รองรับ Windows line endings
    })
    
    rl.on('line', (line) => {
      lineCount++
      
      if (!line.trim()) return  // ข้ามบรรทัดว่าง
      
      // แยก CSV (simple version - ไม่รองรับ quotes ซับซ้อน)
      const cells = line.split(',').map(cell => cell.trim())
      
      if (!headers) {
        // บรรทัดแรกเป็น headers
        headers = cells
      } else {
        // แปลงเป็น object
        const row = {}
        headers.forEach((header, index) => {
          row[header] = cells[index] || ''
        })
        rows.push(row)
        
        // ส่ง batch ทุก 100 rows
        if (rows.length % 100 === 0) {
          win.webContents.send('csv-batch', {
            rows: rows.slice(-100),
            totalRows: rows.length
          })
        }
      }
    })
    
    rl.on('close', () => {
      win.webContents.send('csv-complete', {
        totalRows: rows.length,
        headers
      })
      resolve({ success: true, data: rows, headers, count: rows.length })
    })
    
    rl.on('error', reject)
    fileStream.on('error', reject)
  })
})

// ==========================================
// เขียนไฟล์ขนาดใหญ่ด้วย Stream
// ==========================================
ipcMain.handle('write-large-file', async (event, { filePath, dataGenerator }) => {
  return new Promise((resolve, reject) => {
    const writeStream = fs.createWriteStream(filePath, { encoding: 'utf8' })
    
    writeStream.on('finish', () => {
      const stats = fs.statSync(filePath)
      resolve({ success: true, filePath, size: stats.size })
    })
    
    writeStream.on('error', reject)
    
    // เขียนข้อมูลเป็น chunk
    // ตัวอย่าง: เขียน CSV ขนาดใหญ่
    writeStream.write('id,name,value\n')
    
    for (let i = 0; i < 100000; i++) {
      writeStream.write(`${i},item_${i},${Math.random() * 1000}\n`)
      
      // ตรวจสอบ backpressure
      if (!writeStream.write('')) {
        // stream buffer เต็ม รอ drain
        writeStream.once('drain', () => {})
      }
    }
    
    writeStream.end()
  })
})
```

---

## 10.5 path Module - การจัดการ Paths

```javascript
// pathUtils.js - ทุกอย่างที่ควรรู้เกี่ยวกับ path module

const path = require('path')
const os = require('os')

// ==========================================
// path methods พื้นฐาน
// ==========================================

// ต่อ path เข้าด้วยกัน (cross-platform)
const fullPath = path.join('/home', 'user', 'documents', 'file.txt')
// Linux/macOS: /home/user/documents/file.txt
// Windows: \home\user\documents\file.txt

// resolve path ให้เป็น absolute
const absPath = path.resolve('documents', 'file.txt')
// คำนวณจาก process.cwd()

// แยกส่วนของ path
const filePath = '/home/user/docs/report.pdf'

console.log(path.dirname(filePath))   // /home/user/docs
console.log(path.basename(filePath))  // report.pdf
console.log(path.basename(filePath, '.pdf'))  // report (ไม่มี extension)
console.log(path.extname(filePath))   // .pdf

// แยกส่วนทั้งหมด
const parsed = path.parse(filePath)
// { root: '/', dir: '/home/user/docs', base: 'report.pdf', ext: '.pdf', name: 'report' }

// ประกอบกลับ
const reconstructed = path.format({
  dir: '/home/user/docs',
  name: 'report',
  ext: '.pdf'
})
// /home/user/docs/report.pdf

// ==========================================
// Cross-platform separator
// ==========================================
console.log(path.sep)      // '/' บน Unix, '\' บน Windows
console.log(path.delimiter)  // ':' บน Unix, ';' บน Windows

// ==========================================
// relative path
// ==========================================
const from = '/home/user/docs'
const to = '/home/user/pictures/photo.jpg'
const relativePath = path.relative(from, to)
// '../pictures/photo.jpg'

// ==========================================
// ตรวจสอบ absolute path
// ==========================================
console.log(path.isAbsolute('/home/user'))  // true
console.log(path.isAbsolute('relative'))    // false

// ==========================================
// ปรับ path ให้ถูกต้องสำหรับ OS
// ==========================================
function normalizePath(inputPath) {
  // แปลง forward slash เป็น OS path separator
  return path.normalize(inputPath.replace(/\//g, path.sep))
}

// ==========================================
// สร้าง safe filename
// ==========================================
function sanitizeFilename(filename) {
  // ลบตัวอักษรที่ไม่อนุญาตในชื่อไฟล์
  return filename
    .replace(/[<>:"/\\|?*\x00-\x1f]/g, '_')  // ตัวอักษรที่ไม่อนุญาต Windows
    .replace(/^\./, '_')  // ไม่ขึ้นต้นด้วย dot
    .replace(/[\s.]+$/, '')  // ไม่ลงท้ายด้วย space หรือ dot
    .slice(0, 255)  // จำกัดความยาว
}

// ==========================================
// สร้าง unique filename
// ==========================================
async function getUniqueFilePath(dir, filename) {
  const fsp = require('fs').promises
  const ext = path.extname(filename)
  const name = path.basename(filename, ext)
  
  let candidate = path.join(dir, filename)
  let counter = 1
  
  while (true) {
    try {
      await fsp.access(candidate)
      // ไฟล์มีอยู่แล้ว สร้างชื่อใหม่
      candidate = path.join(dir, `${name} (${counter})${ext}`)
      counter++
    } catch {
      // ไฟล์ไม่มี ใช้ชื่อนี้ได้
      return candidate
    }
  }
}

module.exports = {
  normalizePath,
  sanitizeFilename,
  getUniqueFilePath
}
```

---

## 10.6 Drag & Drop Files

### renderer.js - Drag & Drop Handler
```javascript
// renderer.js - Drag & Drop

class DragDropHandler {
  constructor(dropZoneElement) {
    this.dropZone = dropZoneElement
    this.onFilesDropped = null  // callback
    this.init()
  }
  
  init() {
    const zone = this.dropZone
    
    // ป้องกัน default browser behavior
    document.addEventListener('dragover', (e) => e.preventDefault())
    document.addEventListener('drop', (e) => e.preventDefault())
    
    // Drag enter: เปลี่ยน style
    zone.addEventListener('dragenter', (e) => {
      e.preventDefault()
      e.stopPropagation()
      zone.classList.add('drag-over')
      
      // ตรวจสอบว่าเป็นไฟล์
      if (e.dataTransfer.types.includes('Files')) {
        zone.querySelector('.drop-hint').textContent = 'วางไฟล์ได้เลย!'
      }
    })
    
    // Drag over: อัพเดทอยู่เรื่อยๆ
    zone.addEventListener('dragover', (e) => {
      e.preventDefault()
      e.stopPropagation()
      e.dataTransfer.dropEffect = 'copy'  // แสดง cursor แบบ copy
    })
    
    // Drag leave: คืน style
    zone.addEventListener('dragleave', (e) => {
      e.preventDefault()
      e.stopPropagation()
      
      // ตรวจสอบว่า mouse ออกจาก zone จริงๆ ไม่ใช่แค่เข้าไป child element
      if (!zone.contains(e.relatedTarget)) {
        zone.classList.remove('drag-over')
        zone.querySelector('.drop-hint').textContent = 'ลากไฟล์มาวางที่นี่'
      }
    })
    
    // Drop: รับไฟล์
    zone.addEventListener('drop', async (e) => {
      e.preventDefault()
      e.stopPropagation()
      zone.classList.remove('drag-over')
      zone.querySelector('.drop-hint').textContent = 'ลากไฟล์มาวางที่นี่'
      
      const files = this.extractFiles(e.dataTransfer)
      
      if (files.length > 0 && this.onFilesDropped) {
        await this.onFilesDropped(files)
      }
    })
  }
  
  extractFiles(dataTransfer) {
    const files = []
    
    if (dataTransfer.items) {
      // ใช้ DataTransferItemList API (รองรับโฟลเดอร์)
      for (const item of dataTransfer.items) {
        if (item.kind === 'file') {
          const file = item.getAsFile()
          if (file) files.push(file)
        }
      }
    } else {
      // fallback ใช้ FileList API
      for (const file of dataTransfer.files) {
        files.push(file)
      }
    }
    
    return files
  }
}

// ==========================================
// ใช้งาน DragDropHandler
// ==========================================
const dropZone = document.getElementById('dropZone')
const dragDrop = new DragDropHandler(dropZone)

dragDrop.onFilesDropped = async (files) => {
  const fileList = document.getElementById('fileList')
  fileList.innerHTML = ''
  
  for (const file of files) {
    // Web File API ให้เราดู metadata ได้
    const info = {
      name: file.name,
      size: file.size,
      type: file.type,
      lastModified: new Date(file.lastModified).toLocaleString('th-TH'),
      // path ใน Electron มี file.path (native path)
      path: file.path  // Electron-specific: full path ของไฟล์
    }
    
    // เพิ่มรายการ
    const item = document.createElement('div')
    item.className = 'file-item'
    item.innerHTML = `
      <div class="file-icon">${getFileIcon(file.type)}</div>
      <div class="file-info">
        <strong>${escapeHtml(info.name)}</strong>
        <small>${formatBytes(info.size)} · ${info.type || 'Unknown type'}</small>
        <small>${info.path || 'Path ไม่พร้อมใช้งาน'}</small>
      </div>
      <button class="btn-open" data-path="${escapeHtml(info.path)}">เปิด</button>
    `
    fileList.appendChild(item)
    
    // อ่านไฟล์ถ้าเป็นรูปภาพ
    if (file.type.startsWith('image/')) {
      await previewImage(file, item)
    }
    
    // ส่ง path ไปให้ main process ถ้าต้องการ
    if (info.path) {
      const stats = await window.fsAPI.getFileStats(info.path)
      if (stats.success) {
        console.log('File stats:', stats)
      }
    }
  }
  
  // ผูก event กับปุ่ม "เปิด"
  fileList.querySelectorAll('.btn-open').forEach(btn => {
    btn.addEventListener('click', async (e) => {
      const filePath = e.target.dataset.path
      if (filePath) {
        await window.fsAPI.openFileInSystem(filePath)
      }
    })
  })
}

// Preview รูปภาพ
async function previewImage(file, container) {
  return new Promise((resolve) => {
    const reader = new FileReader()
    reader.onload = (e) => {
      const img = document.createElement('img')
      img.src = e.target.result
      img.style.cssText = 'max-width:200px;max-height:150px;border-radius:4px;margin-top:8px;'
      container.appendChild(img)
      resolve()
    }
    reader.readAsDataURL(file)
  })
}

function getFileIcon(mimeType) {
  if (!mimeType) return '📄'
  if (mimeType.startsWith('image/')) return '🖼️'
  if (mimeType.startsWith('video/')) return '🎬'
  if (mimeType.startsWith('audio/')) return '🎵'
  if (mimeType === 'application/pdf') return '📕'
  if (mimeType === 'text/plain') return '📝'
  if (mimeType.includes('zip') || mimeType.includes('archive')) return '🗜️'
  return '📄'
}

function formatBytes(bytes) {
  if (bytes === 0) return '0 B'
  const sizes = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(1024))
  return `${(bytes / Math.pow(1024, i)).toFixed(1)} ${sizes[i]}`
}

function escapeHtml(text) {
  if (!text) return ''
  return text.replace(/[&<>"']/g, m => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;'
  }[m]))
}
```

---

## 10.7 main.js สมบูรณ์

```javascript
// main.js - สมบูรณ์

const { app, BrowserWindow, ipcMain, dialog, shell } = require('electron')
const path = require('path')
const fs = require('fs')
const fsp = require('fs').promises
const {
  readTextFile, readJsonFile, readFileAsBase64,
  writeTextFile, writeJsonFile, deleteFile,
  exists, getFileStats, copyFile, moveFile,
  listDirectory, getMimeType
} = require('./fileUtils')
const { sanitizeFilename, getUniqueFilePath } = require('./pathUtils')

let mainWindow

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1000,
    height: 750,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  mainWindow.loadFile('index.html')
}

// ==========================================
// IPC Handlers - File Operations
// ==========================================

// อ่านไฟล์ข้อความ
ipcMain.handle('fs:readText', async (event, filePath) => {
  return readTextFile(filePath)
})

// อ่านไฟล์ JSON
ipcMain.handle('fs:readJson', async (event, filePath) => {
  return readJsonFile(filePath)
})

// อ่านไฟล์เป็น base64 (รูปภาพ)
ipcMain.handle('fs:readBase64', async (event, filePath) => {
  return readFileAsBase64(filePath)
})

// เขียนไฟล์ข้อความ
ipcMain.handle('fs:writeText', async (event, { filePath, content }) => {
  return writeTextFile(filePath, content)
})

// เขียนไฟล์ JSON
ipcMain.handle('fs:writeJson', async (event, { filePath, data }) => {
  return writeJsonFile(filePath, data)
})

// ลบไฟล์
ipcMain.handle('fs:delete', async (event, filePath) => {
  // ยืนยันก่อนลบ
  const win = BrowserWindow.fromWebContents(event.sender)
  const { response } = await dialog.showMessageBox(win, {
    type: 'warning',
    message: `ลบไฟล์ "${path.basename(filePath)}"?`,
    buttons: ['ลบ', 'ยกเลิก'],
    defaultId: 1,
    cancelId: 1
  })
  
  if (response === 1) return { canceled: true }
  
  return deleteFile(filePath)
})

// เช็คว่าไฟล์มีอยู่
ipcMain.handle('fs:exists', async (event, filePath) => {
  const fileExists = await exists(filePath)
  return { exists: fileExists }
})

// ดู stats
ipcMain.handle('fs:stats', async (event, filePath) => {
  return getFileStats(filePath)
})

// คัดลอกไฟล์
ipcMain.handle('fs:copy', async (event, { src, dest }) => {
  // ตรวจสอบว่า dest มีอยู่แล้วหรือไม่
  const destExists = await exists(dest)
  
  if (destExists) {
    // สร้างชื่อใหม่
    const dir = path.dirname(dest)
    const filename = path.basename(dest)
    const uniqueDest = await getUniqueFilePath(dir, filename)
    return copyFile(src, uniqueDest)
  }
  
  return copyFile(src, dest)
})

// รายชื่อไฟล์
ipcMain.handle('fs:list', async (event, { dirPath, options }) => {
  return listDirectory(dirPath, options)
})

// เปิดไฟล์/โฟลเดอร์ใน system explorer
ipcMain.handle('fs:openInSystem', async (event, filePath) => {
  try {
    const stats = await fsp.stat(filePath)
    if (stats.isDirectory()) {
      await shell.openPath(filePath)
    } else {
      shell.showItemInFolder(filePath)
    }
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

// เปิดไฟล์ด้วยโปรแกรมเริ่มต้น
ipcMain.handle('fs:openFile', async (event, filePath) => {
  try {
    const error = await shell.openPath(filePath)
    if (error) return { success: false, error }
    return { success: true }
  } catch (err) {
    return { success: false, error: err.message }
  }
})

// ==========================================
// Dialog + File Operations
// ==========================================

// เปิดและอ่านไฟล์
ipcMain.handle('fs:openAndRead', async (event, filters) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showOpenDialog(win, {
    title: 'เปิดไฟล์',
    filters: filters || [
      { name: 'Text Files', extensions: ['txt', 'md', 'json', 'csv'] },
      { name: 'All Files', extensions: ['*'] }
    ],
    properties: ['openFile']
  })
  
  if (result.canceled) return { canceled: true }
  
  const filePath = result.filePaths[0]
  const ext = path.extname(filePath).slice(1).toLowerCase()
  
  // อ่านตาม extension
  if (ext === 'json') {
    const data = await readJsonFile(filePath)
    return { ...data, filePath }
  } else {
    const data = await readTextFile(filePath)
    return { ...data, filePath }
  }
})

// บันทึกไฟล์
ipcMain.handle('fs:saveAs', async (event, { content, defaultName, filters }) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showSaveDialog(win, {
    title: 'บันทึกไฟล์',
    defaultPath: path.join(app.getPath('documents'), defaultName || 'untitled.txt'),
    filters: filters || [
      { name: 'Text Files', extensions: ['txt'] },
      { name: 'JSON', extensions: ['json'] },
      { name: 'CSV', extensions: ['csv'] }
    ]
  })
  
  if (result.canceled) return { canceled: true }
  
  return writeTextFile(result.filePath, content)
})

// ==========================================
// App Paths
// ==========================================
ipcMain.handle('fs:getAppPaths', () => {
  return {
    userData: app.getPath('userData'),
    documents: app.getPath('documents'),
    downloads: app.getPath('downloads'),
    desktop: app.getPath('desktop'),
    temp: app.getPath('temp'),
    home: app.getPath('home')
  }
})

app.whenReady().then(createWindow)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

---

## 10.8 Preload Script สมบูรณ์

```javascript
// preload.js

const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('fsAPI', {
  // File Operations
  readText: (filePath) => ipcRenderer.invoke('fs:readText', filePath),
  readJson: (filePath) => ipcRenderer.invoke('fs:readJson', filePath),
  readBase64: (filePath) => ipcRenderer.invoke('fs:readBase64', filePath),
  writeText: (filePath, content) => ipcRenderer.invoke('fs:writeText', { filePath, content }),
  writeJson: (filePath, data) => ipcRenderer.invoke('fs:writeJson', { filePath, data }),
  deleteFile: (filePath) => ipcRenderer.invoke('fs:delete', filePath),
  fileExists: (filePath) => ipcRenderer.invoke('fs:exists', filePath),
  getFileStats: (filePath) => ipcRenderer.invoke('fs:stats', filePath),
  copyFile: (src, dest) => ipcRenderer.invoke('fs:copy', { src, dest }),
  listDirectory: (dirPath, options) => ipcRenderer.invoke('fs:list', { dirPath, options }),
  
  // Open/Save with Dialog
  openAndRead: (filters) => ipcRenderer.invoke('fs:openAndRead', filters),
  saveAs: (content, defaultName, filters) =>
    ipcRenderer.invoke('fs:saveAs', { content, defaultName, filters }),
  
  // System
  openInSystem: (filePath) => ipcRenderer.invoke('fs:openInSystem', filePath),
  openFile: (filePath) => ipcRenderer.invoke('fs:openFile', filePath),
  getAppPaths: () => ipcRenderer.invoke('fs:getAppPaths'),
  
  // File Watching
  watchPath: (watchPath, recursive) =>
    ipcRenderer.invoke('watch-path', { watchPath, recursive }),
  unwatchPath: (watchPath) => ipcRenderer.invoke('unwatch-path', watchPath),
  getWatchedPaths: () => ipcRenderer.invoke('get-watched-paths'),
  
  // Event Listeners
  onFileChanged: (callback) => {
    const handler = (event, data) => callback(data)
    ipcRenderer.on('file-changed', handler)
    return () => ipcRenderer.removeListener('file-changed', handler)
  },
  
  onReadProgress: (callback) => {
    const handler = (event, data) => callback(data)
    ipcRenderer.on('read-progress', handler)
    return () => ipcRenderer.removeListener('read-progress', handler)
  },
  
  onReadComplete: (callback) => {
    const handler = (event, data) => callback(data)
    ipcRenderer.on('read-complete', handler)
    return () => ipcRenderer.removeListener('read-complete', handler)
  }
})
```

---

## 10.9 index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>File System Demo</title>
  <style>
    * { box-sizing: border-box; }
    
    body {
      font-family: 'Sarabun', sans-serif;
      margin: 0;
      background: #f5f5f5;
      display: grid;
      grid-template-rows: auto 1fr;
      height: 100vh;
    }
    
    header {
      background: #2e7d32;
      color: white;
      padding: 12px 20px;
      font-size: 18px;
      font-weight: bold;
    }
    
    main {
      display: grid;
      grid-template-columns: 260px 1fr;
      gap: 0;
      overflow: hidden;
    }
    
    .sidebar {
      background: #fff;
      border-right: 1px solid #e0e0e0;
      overflow-y: auto;
      padding: 16px;
    }
    
    .content {
      overflow-y: auto;
      padding: 16px;
    }
    
    h3 { font-size: 14px; color: #555; margin: 0 0 8px; text-transform: uppercase; }
    
    .btn {
      width: 100%;
      padding: 10px 12px;
      margin-bottom: 6px;
      border: none;
      border-radius: 6px;
      cursor: pointer;
      font-size: 13px;
      font-family: inherit;
      text-align: left;
      background: #f5f5f5;
      color: #333;
      display: flex;
      align-items: center;
      gap: 8px;
    }
    
    .btn:hover { background: #e0e0e0; }
    .btn-primary { background: #e8f5e9; color: #1b5e20; }
    .btn-danger { background: #ffebee; color: #b71c1c; }
    
    .section-divider { border-top: 1px solid #eee; margin: 12px 0 8px; }
    
    .panel {
      background: white;
      border-radius: 8px;
      padding: 16px;
      margin-bottom: 16px;
      box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }
    
    .panel h2 {
      font-size: 15px;
      margin: 0 0 12px;
      padding-bottom: 8px;
      border-bottom: 2px solid #2e7d32;
      color: #1b5e20;
    }
    
    #dropZone {
      border: 2px dashed #ccc;
      border-radius: 8px;
      padding: 30px;
      text-align: center;
      cursor: pointer;
      transition: all 0.2s;
      background: #fafafa;
    }
    
    #dropZone.drag-over {
      border-color: #2e7d32;
      background: #e8f5e9;
    }
    
    .drop-hint {
      font-size: 14px;
      color: #888;
      margin-top: 8px;
    }
    
    #fileList .file-item {
      display: flex;
      align-items: flex-start;
      gap: 10px;
      padding: 12px;
      border: 1px solid #e0e0e0;
      border-radius: 6px;
      margin-bottom: 8px;
      background: #fafafa;
    }
    
    .file-icon { font-size: 28px; }
    
    .file-info { flex: 1; }
    .file-info strong { display: block; font-size: 14px; }
    .file-info small { display: block; color: #888; font-size: 12px; }
    
    .btn-open {
      padding: 6px 12px;
      background: #2e7d32;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 12px;
      white-space: nowrap;
    }
    
    .btn-open:hover { background: #1b5e20; }
    
    textarea {
      width: 100%;
      height: 150px;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 4px;
      font-size: 13px;
      font-family: monospace;
      resize: vertical;
    }
    
    .result {
      background: #f1f8e9;
      border: 1px solid #c8e6c9;
      border-radius: 4px;
      padding: 10px;
      font-size: 13px;
      font-family: monospace;
      white-space: pre-wrap;
      max-height: 200px;
      overflow-y: auto;
    }
    
    .progress-container {
      margin: 8px 0;
    }
    
    .progress-bar {
      height: 6px;
      background: #e0e0e0;
      border-radius: 3px;
      overflow: hidden;
    }
    
    .progress-fill {
      height: 100%;
      background: #2e7d32;
      width: 0%;
      transition: width 0.2s;
    }
    
    .app-paths {
      font-size: 12px;
      font-family: monospace;
    }
    
    .app-paths dt {
      color: #555;
      margin-top: 6px;
    }
    
    .app-paths dd {
      color: #1b5e20;
      word-break: break-all;
      margin-left: 0;
    }
  </style>
</head>
<body>
  <header>📁 File System Operations Demo</header>
  
  <main>
    <div class="sidebar">
      <h3>การดำเนินการ</h3>
      
      <button class="btn btn-primary" id="btnOpenRead">📂 เปิดและอ่านไฟล์</button>
      <button class="btn btn-primary" id="btnSaveText">💾 บันทึกข้อความ</button>
      <button class="btn" id="btnOpenFolder">📁 เปิดโฟลเดอร์</button>
      
      <div class="section-divider"></div>
      <h3>App Paths</h3>
      <button class="btn" id="btnShowPaths">🗂️ แสดง Paths</button>
      
      <div class="section-divider"></div>
      <h3>File Watching</h3>
      <button class="btn" id="btnWatchFile">👁️ เฝ้าดูไฟล์</button>
      <button class="btn btn-danger" id="btnUnwatchAll">🚫 หยุดดูทั้งหมด</button>
    </div>
    
    <div class="content">
      <!-- Drag & Drop -->
      <div class="panel">
        <h2>Drag & Drop Files</h2>
        <div id="dropZone">
          <div style="font-size:40px">📥</div>
          <div class="drop-hint">ลากไฟล์มาวางที่นี่</div>
        </div>
        <div id="fileList" style="margin-top:12px"></div>
      </div>
      
      <!-- Text Editor -->
      <div class="panel">
        <h2>แก้ไขไฟล์ข้อความ</h2>
        <textarea id="textContent" placeholder="เนื้อหาไฟล์..."></textarea>
        <div style="display:flex;gap:8px;margin-top:8px">
          <button class="btn btn-primary" id="btnOpenRead" style="width:auto">เปิดไฟล์</button>
          <button class="btn btn-primary" id="btnSaveText" style="width:auto">บันทึก</button>
        </div>
      </div>
      
      <!-- Progress -->
      <div class="panel">
        <h2>อ่านไฟล์ขนาดใหญ่</h2>
        <button class="btn btn-primary" id="btnReadLarge" style="width:auto">เลือกและอ่านไฟล์</button>
        <div class="progress-container" id="progressContainer" style="display:none">
          <div style="display:flex;justify-content:space-between;font-size:13px;margin-bottom:4px">
            <span id="progressLabel">กำลังอ่าน...</span>
            <span id="progressPercent">0%</span>
          </div>
          <div class="progress-bar">
            <div class="progress-fill" id="progressFill"></div>
          </div>
        </div>
      </div>
      
      <!-- App Paths -->
      <div class="panel" id="pathsPanel" style="display:none">
        <h2>App Paths</h2>
        <dl class="app-paths" id="pathsList"></dl>
      </div>
      
      <!-- File Watch Log -->
      <div class="panel">
        <h2>File Watch Log</h2>
        <div class="result" id="watchLog">ยังไม่มีการเปลี่ยนแปลง...</div>
      </div>
      
      <!-- Result -->
      <div class="panel">
        <h2>ผลลัพธ์</h2>
        <div class="result" id="result">รอการดำเนินการ...</div>
      </div>
    </div>
  </main>
  
  <script src="renderer.js"></script>
</body>
</html>
```

---

## สรุปบทเรียน

### สรุปหัวข้อที่เรียน

| หัวข้อ | API ที่ใช้ | ใช้เมื่อ |
|--------|----------|---------|
| อ่านไฟล์ | `fs.promises.readFile` | ไฟล์ขนาดเล็ก-กลาง |
| เขียนไฟล์ | `fs.promises.writeFile` | บันทึกข้อมูล |
| อ่านไฟล์ใหญ่ | `fs.createReadStream` | ไฟล์ > 10MB |
| เฝ้าดูไฟล์ | `fs.watch` | sync ไฟล์ |
| Path จัดการ | `path.join`, `path.resolve` | ทุกครั้งที่ต้องการ path |
| App paths | `app.getPath()` | ที่เก็บข้อมูลแอป |
| Drag & drop | Web API + `file.path` | รับไฟล์จากผู้ใช้ |

### Best Practices

1. **ใช้ `fs.promises` เสมอ** - ไม่ block main thread
2. **ใช้ `path.join` ไม่ใช่ string concatenation** - รองรับ cross-platform
3. **Validate file paths** ก่อน operation ทุกครั้ง
4. **ใช้ stream สำหรับไฟล์ใหญ่** - ประหยัด memory
5. **เก็บ watcher reference** และ close เมื่อไม่ใช้แล้ว
6. **Debounce fs.watch** เพราะมักจะ fire หลายครั้งสำหรับการเปลี่ยนแปลงเดียว
7. **สร้าง directory** ก่อนเขียนไฟล์เสมอ (`mkdir recursive`)
8. **Handle errors** ทุก operation อย่าปล่อยให้แอปพัง
