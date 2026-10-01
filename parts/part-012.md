# ตอนที่ 12: Context Bridge & Security

## ทำความเข้าใจ Context Isolation

Context Isolation เป็นคุณสมบัติด้านความปลอดภัยที่สำคัญที่สุดใน Electron ซึ่งแยก JavaScript context ของ preload script ออกจาก renderer process อย่างสมบูรณ์

### ก่อนและหลัง Context Isolation

```
ก่อน Context Isolation (ไม่ปลอดภัย):
┌─────────────────────────────────────┐
│          Renderer Window            │
│  ┌────────────────────────────┐     │
│  │   Shared JavaScript Context│     │
│  │  • window object           │     │
│  │  • Node.js APIs (require)  │     │
│  │  • preload code            │     │
│  │  • page code               │     │
│  └────────────────────────────┘     │
└─────────────────────────────────────┘

หลัง Context Isolation (ปลอดภัย):
┌─────────────────────────────────────┐
│          Renderer Window            │
│  ┌────────────┐  ┌────────────┐     │
│  │  Isolated  │  │  Page      │     │
│  │  Context   │  │  Context   │     │
│  │ (preload)  │  │ (renderer) │     │
│  │            │  │            │     │
│  └────────────┘  └────────────┘     │
│       ↕ contextBridge only          │
└─────────────────────────────────────┘
```

### การเปิดใช้งาน Context Isolation

```javascript
// main.js
const { BrowserWindow } = require('electron')

const win = new BrowserWindow({
  webPreferences: {
    // เปิด context isolation (default: true ใน Electron 12+)
    contextIsolation: true,
    
    // ปิด node integration ใน renderer
    nodeIntegration: false,
    
    // ระบุ preload script
    preload: __dirname + '/preload.js'
  }
})
```

### ทดสอบ Context Isolation

```javascript
// preload.js
const { contextBridge } = require('electron')

// ตัวแปรนี้มีอยู่เฉพาะใน preload context
const secretKey = 'super-secret-key-12345'

contextBridge.exposeInMainWorld('test', {
  getKey: () => secretKey // expose ค่านี้ผ่าน contextBridge
})

// พยายาม set ค่าบน window object
window.directAccess = 'I should not be accessible from renderer'
```

```javascript
// renderer.js
// Context Isolation ทำงาน:
console.log(window.test.getKey())      // ✅ 'super-secret-key-12345'
console.log(window.directAccess)       // ❌ undefined (ไม่สามารถเข้าถึงได้!)
console.log(window.require)            // ❌ undefined
console.log(window.process)           // ❌ undefined (ถ้า contextIsolation: true)
```

## contextBridge API ในรายละเอียด

### ข้อจำกัดของ contextBridge

`contextBridge` สามารถส่งผ่านได้เฉพาะ:
- Primitive values (string, number, boolean, null, undefined)
- Arrays และ plain objects (ที่มีค่าเป็น primitive)
- Functions
- Error objects

**ไม่สามารถส่งผ่าน:**
- Class instances (ที่มี prototype methods)
- Symbols
- Functions ที่มี properties (functions อาจถูก clone)
- Getters/Setters บน objects
- WeakMaps, WeakSets, WeakRefs

```javascript
// preload.js
const { contextBridge } = require('electron')

// ✅ ทำได้: functions
contextBridge.exposeInMainWorld('api', {
  hello: () => 'world',
  add: (a, b) => a + b
})

// ✅ ทำได้: nested objects
contextBridge.exposeInMainWorld('nested', {
  level1: {
    level2: {
      value: 42,
      fn: () => 'nested function'
    }
  }
})

// ✅ ทำได้: arrays
contextBridge.exposeInMainWorld('arrays', {
  getNumbers: () => [1, 2, 3, 4, 5],
  getObjects: () => [{ id: 1 }, { id: 2 }]
})

// ❌ ไม่ได้: class instances
class MyClass {
  constructor() { this.value = 42 }
  getValue() { return this.value }
}

// จะ throw error หรือ strip ส่วนที่เป็น class methods
contextBridge.exposeInMainWorld('classExample', {
  getInstance: () => new MyClass() // ❌ อาจไม่ทำงานตามที่คาด
})

// ✅ แทนที่ด้วย plain object
contextBridge.exposeInMainWorld('objectExample', {
  createInstance: () => {
    const instance = new MyClass()
    return {
      value: instance.value,
      getValue: () => instance.getValue()
    }
  }
})
```

### การส่ง Callbacks ผ่าน contextBridge

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

// ✅ วิธีที่ถูกต้อง: รับ callback จาก renderer
contextBridge.exposeInMainWorld('events', {
  // renderer ส่ง callback มาเป็น argument
  onUpdate: (callback) => {
    if (typeof callback !== 'function') throw new TypeError('callback must be a function')
    
    // ห่อ callback เพื่อ filter event object
    const wrappedCallback = (_event, data) => callback(data)
    ipcRenderer.on('app:update', wrappedCallback)
    
    // return cleanup function
    return () => ipcRenderer.removeListener('app:update', wrappedCallback)
  }
})
```

```javascript
// renderer.js
// รับ cleanup function จาก onUpdate
const unsubscribe = window.events.onUpdate((data) => {
  console.log('Update received:', data)
})

// ทำ cleanup เมื่อไม่ต้องการแล้ว
window.addEventListener('beforeunload', () => {
  unsubscribe()
})
```

## ป้องกัน XSS (Cross-Site Scripting)

### ความเสี่ยง XSS ใน Electron

```javascript
// main.js
const win = new BrowserWindow({
  webPreferences: {
    nodeIntegration: true, // ❌ อันตราย!
    contextIsolation: false // ❌ อันตราย!
  }
})

// ถ้าหน้าเว็บโหลด content จากที่อื่น และมี XSS
// attacker สามารถรันโค้ดนี้ใน renderer:
// require('child_process').exec('rm -rf /')
```

### การป้องกัน XSS

```javascript
// main.js - การตั้งค่าที่ปลอดภัย
const { BrowserWindow, session } = require('electron')

const win = new BrowserWindow({
  webPreferences: {
    // ✅ เปิด context isolation
    contextIsolation: true,
    
    // ✅ ปิด node integration
    nodeIntegration: false,
    
    // ✅ เปิด sandbox (ถ้าทำได้)
    sandbox: true,
    
    // ✅ ปิด web security ไม่ควรทำ
    webSecurity: true, // default true
    
    // ✅ จำกัด navigation
    disableDialogs: false
  }
})

// ✅ ป้องกัน navigation ไปยัง URLs ที่ไม่ได้รับอนุญาต
win.webContents.on('will-navigate', (event, navigationUrl) => {
  const parsedUrl = new URL(navigationUrl)
  
  // อนุญาตเฉพาะ localhost หรือ file protocol
  if (parsedUrl.origin !== 'null' && // file:// มี origin เป็น null
      !parsedUrl.hostname.startsWith('localhost') &&
      parsedUrl.origin !== 'https://api.myapp.com') {
    event.preventDefault()
    console.log('Blocked navigation to:', navigationUrl)
  }
})

// ✅ ป้องกัน new window
win.webContents.setWindowOpenHandler(({ url }) => {
  // เปิดใน default browser แทน
  require('electron').shell.openExternal(url)
  return { action: 'deny' }
})
```

### Sanitize Content ก่อนแสดง

```javascript
// renderer.js - ใช้ DOMPurify สำหรับ sanitize HTML
// ติดตั้ง: npm install dompurify
import DOMPurify from 'dompurify'

function renderHTML(htmlContent) {
  const clean = DOMPurify.sanitize(htmlContent, {
    ALLOWED_TAGS: ['p', 'b', 'i', 'em', 'strong', 'a', 'ul', 'ol', 'li'],
    ALLOWED_ATTR: ['href', 'target']
  })
  
  document.getElementById('content').innerHTML = clean
}

// ❌ อย่าทำแบบนี้
function dangerousRender(htmlContent) {
  document.getElementById('content').innerHTML = htmlContent // อันตราย!
}

// ✅ หรือใช้ textContent สำหรับ plain text
function safeRender(textContent) {
  document.getElementById('content').textContent = textContent // ปลอดภัย
}
```

## Content Security Policy (CSP)

### ตั้งค่า CSP ผ่าน Meta Tag

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <!-- ตั้งค่า CSP ที่เข้มงวด -->
  <meta http-equiv="Content-Security-Policy" 
        content="default-src 'self'; 
                 script-src 'self'; 
                 style-src 'self' 'unsafe-inline';
                 img-src 'self' data:;
                 connect-src 'self' https://api.myapp.com;
                 font-src 'self';
                 object-src 'none';
                 base-uri 'self';">
  <title>Secure App</title>
</head>
<body>
  <div id="app"></div>
  <script src="renderer.js"></script>
</body>
</html>
```

### ตั้งค่า CSP ผ่าน HTTP Headers

```javascript
// main.js - ตั้งค่า CSP ผ่าน session headers
const { app, BrowserWindow, session } = require('electron')

app.whenReady().then(() => {
  // ตั้งค่า CSP headers
  session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
    callback({
      responseHeaders: {
        ...details.responseHeaders,
        'Content-Security-Policy': [
          "default-src 'self'; " +
          "script-src 'self'; " +
          "style-src 'self' 'unsafe-inline'; " +
          "img-src 'self' data: https:; " +
          "connect-src 'self' https://api.myapp.com; " +
          "font-src 'self'; " +
          "object-src 'none';"
        ]
      }
    })
  })
  
  const win = new BrowserWindow({
    webPreferences: {
      contextIsolation: true,
      nodeIntegration: false,
      preload: __dirname + '/preload.js'
    }
  })
  
  win.loadFile('index.html')
})
```

### CSP สำหรับ development vs production

```javascript
// main.js
const { app, BrowserWindow, session } = require('electron')
const isDev = process.env.NODE_ENV === 'development'

function getCSP() {
  if (isDev) {
    // Development: อนุญาต hot reload และ devtools
    return [
      "default-src 'self' 'unsafe-inline' 'unsafe-eval'; " +
      "script-src 'self' 'unsafe-inline' 'unsafe-eval'; " +
      "connect-src 'self' ws://localhost:* http://localhost:*; " +
      "img-src 'self' data: https:; " +
      "style-src 'self' 'unsafe-inline';"
    ]
  } else {
    // Production: เข้มงวดมากขึ้น
    return [
      "default-src 'self'; " +
      "script-src 'self'; " +
      "style-src 'self'; " +
      "img-src 'self' data:; " +
      "connect-src 'self' https://api.myapp.com; " +
      "font-src 'self'; " +
      "object-src 'none'; " +
      "base-uri 'self'; " +
      "form-action 'self';"
    ]
  }
}

app.whenReady().then(() => {
  session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
    callback({
      responseHeaders: {
        ...details.responseHeaders,
        'Content-Security-Policy': getCSP()
      }
    })
  })
  
  createWindow()
})
```

## Validation ใน IPC Handlers

### Schema Validation ด้วย zod

```javascript
// main.js
const { ipcMain } = require('electron')
const { z } = require('zod') // npm install zod

// Define schemas
const FileWriteSchema = z.object({
  filePath: z.string().min(1).max(1024),
  content: z.string().max(10 * 1024 * 1024), // max 10MB
  encoding: z.enum(['utf8', 'utf-8', 'ascii', 'base64']).default('utf8')
})

const QuerySchema = z.object({
  table: z.string().regex(/^[a-zA-Z_][a-zA-Z0-9_]*$/), // ป้องกัน SQL injection
  conditions: z.record(z.string(), z.unknown()).optional(),
  limit: z.number().int().min(1).max(1000).default(100),
  offset: z.number().int().min(0).default(0)
})

// IPC handler พร้อม validation
ipcMain.handle('file:write', async (event, data) => {
  // Validate input
  const parseResult = FileWriteSchema.safeParse(data)
  
  if (!parseResult.success) {
    return {
      success: false,
      error: 'Invalid input: ' + parseResult.error.message
    }
  }
  
  const { filePath, content, encoding } = parseResult.data
  
  // ตรวจสอบ path ว่าอยู่ใน allowed directory
  const allowedDir = app.getPath('userData')
  const resolvedPath = require('path').resolve(filePath)
  
  if (!resolvedPath.startsWith(allowedDir)) {
    return {
      success: false,
      error: 'Access denied: path is outside allowed directory'
    }
  }
  
  try {
    require('fs').writeFileSync(resolvedPath, content, encoding)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

ipcMain.handle('db:query', async (event, data) => {
  const parseResult = QuerySchema.safeParse(data)
  
  if (!parseResult.success) {
    return {
      success: false,
      error: 'Invalid query parameters'
    }
  }
  
  const { table, conditions, limit, offset } = parseResult.data
  
  // ดำเนินการ query ด้วย parameters ที่ validated แล้ว
  // ...
})
```

### Manual Validation

```javascript
// validators.js - helper functions สำหรับ validation
const path = require('path')
const { app } = require('electron')

/**
 * ตรวจสอบว่า path ปลอดภัย (อยู่ใน allowed directories)
 */
function isPathSafe(filePath, allowedDirectories = []) {
  if (!filePath || typeof filePath !== 'string') return false
  
  // ป้องกัน path traversal
  const normalized = path.normalize(filePath)
  
  // ตรวจสอบว่า path อยู่ใน allowed directories
  for (const dir of allowedDirectories) {
    if (normalized.startsWith(dir)) return true
  }
  
  return false
}

/**
 * ตรวจสอบ SQL table name (ป้องกัน injection)
 */
function isValidTableName(name) {
  return /^[a-zA-Z_][a-zA-Z0-9_]*$/.test(name)
}

/**
 * ตรวจสอบ email format
 */
function isValidEmail(email) {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  return emailRegex.test(email)
}

/**
 * Sanitize string input
 */
function sanitizeString(input, maxLength = 1000) {
  if (typeof input !== 'string') return ''
  return input.trim().slice(0, maxLength)
}

/**
 * ตรวจสอบ object ว่ามี keys ที่ต้องการ
 */
function hasRequiredKeys(obj, keys) {
  if (!obj || typeof obj !== 'object') return false
  return keys.every(key => key in obj)
}

module.exports = {
  isPathSafe,
  isValidTableName,
  isValidEmail,
  sanitizeString,
  hasRequiredKeys
}
```

```javascript
// main.js - ใช้งาน validators
const { ipcMain, app } = require('electron')
const path = require('path')
const fs = require('fs')
const { isPathSafe, sanitizeString } = require('./validators')

const ALLOWED_DIRS = [
  app.getPath('userData'),
  app.getPath('documents'),
  app.getPath('desktop')
]

ipcMain.handle('file:read', async (event, filePath) => {
  // ตรวจสอบ type
  if (typeof filePath !== 'string') {
    return { success: false, error: 'filePath must be a string' }
  }
  
  // ตรวจสอบ path safety
  if (!isPathSafe(filePath, ALLOWED_DIRS)) {
    return { success: false, error: 'Access denied' }
  }
  
  // ตรวจสอบ file size ก่อนอ่าน
  try {
    const stats = fs.statSync(filePath)
    if (stats.size > 50 * 1024 * 1024) { // 50MB limit
      return { success: false, error: 'File is too large' }
    }
    
    const content = fs.readFileSync(filePath, 'utf8')
    return { success: true, content }
  } catch (error) {
    return { success: false, error: 'Cannot read file' }
  }
})
```

## สิ่งที่ทำได้และทำไม่ได้ใน Preload

### ทำได้ใน Preload

```javascript
// preload.js - สิ่งที่ทำได้

// ✅ ใช้ Node.js modules
const fs = require('fs')
const path = require('path')
const os = require('os')

// ✅ ใช้ Electron modules สำหรับ renderer
const { contextBridge, ipcRenderer } = require('electron')

// ✅ ใช้ process object (preload context)
console.log(process.platform)
console.log(process.versions.electron)

// ✅ เข้าถึง DOM (แต่ระวัง: DOM ยังไม่โหลดตอน preload รัน)
// ต้องรอ DOMContentLoaded
window.addEventListener('DOMContentLoaded', () => {
  // ✅ ทำได้หลัง DOM โหลด
  document.title = 'My App'
})

// ✅ ตั้งค่า event listeners บน ipcRenderer
ipcRenderer.on('from-main', (event, data) => {
  console.log('Received from main:', data)
})

// ✅ ใช้ async/await
async function fetchData() {
  const result = await ipcRenderer.invoke('fetch-data')
  return result
}

// ✅ expose APIs ผ่าน contextBridge
contextBridge.exposeInMainWorld('myAPI', {
  // functions, values, objects
})
```

### ทำไม่ได้ใน Preload

```javascript
// preload.js - สิ่งที่ทำไม่ได้หรือไม่ควรทำ

// ❌ ไม่ควร expose ipcRenderer โดยตรง
// contextBridge.exposeInMainWorld('ipc', ipcRenderer)

// ❌ ไม่ควร expose require หรือ __dirname
// contextBridge.exposeInMainWorld('require', require)

// ❌ ไม่ควร expose process object โดยตรง
// contextBridge.exposeInMainWorld('process', process)

// ❌ ไม่ควรทำ operations ที่อาจ block
// const data = fs.readFileSync('/large-file.bin') // อาจ block UI

// ❌ ไม่ควร access DOM before DOMContentLoaded
// document.getElementById('app') // จะ return null

// ❌ ไม่ควร use modules ที่ไม่จำเป็น
// const exec = require('child_process').exec
// contextBridge.exposeInMainWorld('exec', exec) // อันตรายมาก!

// ❌ ไม่ควร expose functions ที่รับ arbitrary code
// contextBridge.exposeInMainWorld('eval', eval) // อันตรายมากที่สุด!
```

## Complete Secure Application Example

### โครงสร้าง

```
secure-app/
├── main.js
├── preload.js
├── validators.js
├── ipc-handlers.js
├── renderer/
│   ├── index.html
│   └── renderer.js
└── package.json
```

### ipc-handlers.js - แยก handlers ออกมา

```javascript
// ipc-handlers.js
const { ipcMain, app, dialog, shell } = require('electron')
const path = require('path')
const fs = require('fs')

function setupIpcHandlers(mainWindow) {
  const userData = app.getPath('userData')
  const documents = app.getPath('documents')
  
  // Helper สำหรับตรวจสอบ path
  function isAllowedPath(filePath) {
    const resolved = path.resolve(filePath)
    const allowed = [userData, documents]
    return allowed.some(dir => resolved.startsWith(dir))
  }
  
  // File operations
  ipcMain.handle('file:read', async (event, filePath) => {
    if (typeof filePath !== 'string' || !isAllowedPath(filePath)) {
      return { success: false, error: 'Access denied' }
    }
    
    try {
      const content = fs.readFileSync(filePath, 'utf8')
      return { success: true, content }
    } catch (e) {
      return { success: false, error: e.message }
    }
  })
  
  ipcMain.handle('file:write', async (event, filePath, content) => {
    if (typeof filePath !== 'string' || !isAllowedPath(filePath)) {
      return { success: false, error: 'Access denied' }
    }
    if (typeof content !== 'string') {
      return { success: false, error: 'content must be string' }
    }
    
    try {
      fs.mkdirSync(path.dirname(filePath), { recursive: true })
      fs.writeFileSync(filePath, content, 'utf8')
      return { success: true }
    } catch (e) {
      return { success: false, error: e.message }
    }
  })
  
  // Dialog operations
  ipcMain.handle('dialog:open', async (event, options) => {
    const safeOptions = {
      title: typeof options?.title === 'string' ? options.title : 'Open File',
      properties: Array.isArray(options?.properties) 
        ? options.properties.filter(p => ['openFile', 'openDirectory', 'multiSelections'].includes(p))
        : ['openFile'],
      filters: Array.isArray(options?.filters) ? options.filters.slice(0, 10) : []
    }
    
    return await dialog.showOpenDialog(mainWindow, safeOptions)
  })
  
  // App info (safe to expose)
  ipcMain.handle('app:info', () => ({
    version: app.getVersion(),
    name: app.getName(),
    platform: process.platform,
    arch: process.arch
  }))
  
  // Open URL in external browser
  ipcMain.handle('shell:open-url', async (event, url) => {
    if (typeof url !== 'string') return { success: false }
    
    // ตรวจสอบว่าเป็น https URL เท่านั้น
    try {
      const parsed = new URL(url)
      if (!['https:', 'http:'].includes(parsed.protocol)) {
        return { success: false, error: 'Only http/https URLs allowed' }
      }
      await shell.openExternal(url)
      return { success: true }
    } catch (e) {
      return { success: false, error: 'Invalid URL' }
    }
  })
  
  // Cleanup on window close
  mainWindow.on('closed', () => {
    ipcMain.removeHandler('file:read')
    ipcMain.removeHandler('file:write')
    ipcMain.removeHandler('dialog:open')
    ipcMain.removeHandler('app:info')
    ipcMain.removeHandler('shell:open-url')
  })
}

module.exports = { setupIpcHandlers }
```

### preload.js - สมบูรณ์

```javascript
// preload.js
'use strict'

const { contextBridge, ipcRenderer } = require('electron')

// สร้าง type-safe IPC wrapper
function createIPCWrapper() {
  const INVOKE_CHANNELS = Object.freeze([
    'file:read',
    'file:write',
    'dialog:open',
    'app:info',
    'shell:open-url'
  ])
  
  const SEND_CHANNELS = Object.freeze([
    'window:minimize',
    'window:maximize',
    'window:close',
    'app:quit'
  ])
  
  const RECEIVE_CHANNELS = Object.freeze([
    'app:update-available',
    'app:error',
    'window:focus'
  ])
  
  return {
    invoke: async (channel, ...args) => {
      if (!INVOKE_CHANNELS.includes(channel)) {
        throw new Error(`IPC channel "${channel}" is not allowed`)
      }
      return ipcRenderer.invoke(channel, ...args)
    },
    
    send: (channel, data) => {
      if (!SEND_CHANNELS.includes(channel)) {
        console.warn(`Blocked: IPC channel "${channel}" is not allowed`)
        return
      }
      ipcRenderer.send(channel, data)
    },
    
    on: (channel, callback) => {
      if (!RECEIVE_CHANNELS.includes(channel)) {
        throw new Error(`Cannot subscribe to channel "${channel}"`)
      }
      if (typeof callback !== 'function') {
        throw new TypeError('callback must be a function')
      }
      
      const fn = (_event, data) => callback(data)
      ipcRenderer.on(channel, fn)
      return () => ipcRenderer.removeListener(channel, fn)
    },
    
    once: (channel, callback) => {
      if (!RECEIVE_CHANNELS.includes(channel)) {
        throw new Error(`Cannot subscribe to channel "${channel}"`)
      }
      if (typeof callback !== 'function') {
        throw new TypeError('callback must be a function')
      }
      
      ipcRenderer.once(channel, (_event, data) => callback(data))
    }
  }
}

const ipc = createIPCWrapper()

// Expose specific, safe APIs
contextBridge.exposeInMainWorld('fileAPI', {
  read: (path) => ipc.invoke('file:read', path),
  write: (path, content) => ipc.invoke('file:write', path, content),
  open: (options) => ipc.invoke('dialog:open', options)
})

contextBridge.exposeInMainWorld('appAPI', {
  getInfo: () => ipc.invoke('app:info'),
  openURL: (url) => ipc.invoke('shell:open-url', url),
  quit: () => ipc.send('app:quit')
})

contextBridge.exposeInMainWorld('windowAPI', {
  minimize: () => ipc.send('window:minimize'),
  maximize: () => ipc.send('window:maximize'),
  close: () => ipc.send('window:close'),
  
  onFocus: (cb) => ipc.on('window:focus', cb),
  onUpdateAvailable: (cb) => ipc.on('app:update-available', cb)
})

// Expose only needed version info
contextBridge.exposeInMainWorld('versions', Object.freeze({
  node: process.versions.node,
  chrome: process.versions.chrome,
  electron: process.versions.electron
}))
```

### renderer/index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <!-- CSP เข้มงวด -->
  <meta http-equiv="Content-Security-Policy" 
        content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:;">
  <title>Secure App</title>
  <style>
    body { font-family: sans-serif; padding: 20px; }
    .info { background: #f0f0f0; padding: 10px; border-radius: 4px; }
    button { margin: 5px; padding: 8px 16px; cursor: pointer; }
    #output { background: #1e1e1e; color: #d4d4d4; padding: 15px; border-radius: 4px; min-height: 100px; }
  </style>
</head>
<body>
  <h1>Secure Electron App</h1>
  
  <div class="info" id="app-info">Loading app info...</div>
  
  <div>
    <button id="btn-open">เปิดไฟล์</button>
    <button id="btn-save">บันทึกไฟล์</button>
    <button id="btn-open-url">เปิด Website</button>
  </div>
  
  <pre id="output">ผลลัพธ์จะแสดงที่นี่...</pre>
  
  <script src="renderer.js"></script>
</body>
</html>
```

### renderer/renderer.js

```javascript
'use strict'

const output = document.getElementById('output')

function log(message) {
  output.textContent += message + '\n'
}

// แสดง app info
async function loadAppInfo() {
  try {
    const info = await window.appAPI.getInfo()
    document.getElementById('app-info').textContent = 
      `App: ${info.name} v${info.version} | Platform: ${info.platform} | Arch: ${info.arch}`
  } catch (e) {
    log('Error loading app info: ' + e.message)
  }
}

// เปิดไฟล์
document.getElementById('btn-open').addEventListener('click', async () => {
  try {
    const result = await window.fileAPI.open({
      title: 'เลือกไฟล์',
      properties: ['openFile'],
      filters: [{ name: 'Text', extensions: ['txt', 'md'] }]
    })
    
    if (!result.canceled && result.filePaths.length > 0) {
      const fileResult = await window.fileAPI.read(result.filePaths[0])
      if (fileResult.success) {
        log('File content:\n' + fileResult.content.slice(0, 200) + '...')
      } else {
        log('Error: ' + fileResult.error)
      }
    }
  } catch (e) {
    log('Error: ' + e.message)
  }
})

// เปิด URL ใน browser
document.getElementById('btn-open-url').addEventListener('click', async () => {
  try {
    const result = await window.appAPI.openURL('https://electronjs.org')
    if (result.success) {
      log('Opened URL in browser')
    } else {
      log('Error: ' + result.error)
    }
  } catch (e) {
    log('Error: ' + e.message)
  }
})

// Subscribe to events
const unsubUpdate = window.windowAPI.onUpdateAvailable((data) => {
  log('Update available: ' + JSON.stringify(data))
})

// แสดง versions
log(`Versions:\n  Node: ${window.versions.node}\n  Chrome: ${window.versions.chrome}\n  Electron: ${window.versions.electron}`)

// Cleanup
window.addEventListener('beforeunload', () => {
  unsubUpdate()
})

// Initialize
loadAppInfo()
```

## สรุป Security Best Practices

### รายการตรวจสอบความปลอดภัย

```
Security Checklist:
✅ contextIsolation: true
✅ nodeIntegration: false
✅ sandbox: true (ถ้าเป็นไปได้)
✅ webSecurity: true (default)
✅ ตั้งค่า Content Security Policy
✅ ใช้ contextBridge.exposeInMainWorld เท่านั้น
✅ ตรวจสอบ channel whitelist
✅ Validate input ทุกครั้ง
✅ ป้องกัน path traversal
✅ ไม่ expose Node.js modules โดยตรง
✅ จำกัด navigation ด้วย will-navigate
✅ ตั้งค่า setWindowOpenHandler
✅ ไม่ใช้ remote module
✅ อัปเดต Electron เป็นเวอร์ชันล่าสุดเสมอ
```

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Shell & System Integration ของ Electron
