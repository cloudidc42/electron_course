# ตอนที่ 18: Protocol Handler

## Protocol Handler ใน Electron

Protocol Handler ให้ความสามารถในการสร้าง custom URL schemes สำหรับแอปพลิเคชัน Electron เช่น `app://`, `myapp://` หรือ intercept HTTP requests เพื่อ serve ไฟล์จาก filesystem หรือ process requests ในแบบพิเศษ

```javascript
const { protocol } = require('electron')
```

**สำคัญ:** ต้องลงทะเบียน protocol privileges ก่อน `app.whenReady()` ด้วย `protocol.registerSchemesAsPrivileged()`

## ทำไมต้องใช้ Protocol Handler?

1. **Serve local files** โดยไม่ต้องใช้ HTTP server
2. **สร้าง custom URL scheme** สำหรับ deep linking
3. **Intercept requests** เพื่อ modify หรือ block
4. **ป้องกัน CORS** สำหรับ local assets
5. **Implement virtual filesystem** หรือ CDN

## protocol.registerSchemesAsPrivileged

```javascript
// main.js
const { app, protocol } = require('electron')

// ต้องเรียกก่อน app.whenReady()!
protocol.registerSchemesAsPrivileged([
  {
    scheme: 'app',
    privileges: {
      secure: true,             // treat as HTTPS
      standard: true,           // use standard URL parsing
      corsEnabled: true,        // allow CORS
      supportFetchAPI: true,    // support fetch()
      allowServiceWorkers: true, // allow service workers
      stream: true              // support streaming responses
    }
  }
])

app.whenReady().then(() => {
  // ลงทะเบียน protocol handler ที่นี่
  protocol.handle('app', handleAppProtocol)
})
```

## protocol.registerFileProtocol

```javascript
// main.js
const { app, protocol } = require('electron')
const path = require('path')
const fs = require('fs')

protocol.registerSchemesAsPrivileged([
  { scheme: 'app', privileges: { secure: true, standard: true } }
])

app.whenReady().then(() => {
  // Serve files จาก app directory
  protocol.registerFileProtocol('app', (request, callback) => {
    const url = new URL(request.url)
    const filePath = path.join(__dirname, 'assets', url.pathname)
    
    // ป้องกัน path traversal
    const normalizedPath = path.normalize(filePath)
    const assetsDir = path.join(__dirname, 'assets')
    
    if (!normalizedPath.startsWith(assetsDir)) {
      callback({ error: -6 }) // PERMISSION_DENIED
      return
    }
    
    // ตรวจสอบว่าไฟล์มีอยู่
    if (!fs.existsSync(normalizedPath)) {
      callback({ error: -6 })
      return
    }
    
    callback({ path: normalizedPath })
  })
})
```

**หมายเหตุ:** `registerFileProtocol` เป็น legacy API ใช้ `protocol.handle` แทนใน Electron 25+

## protocol.handle (Modern API)

```javascript
// main.js - Modern API (Electron 25+)
const { app, protocol } = require('electron')
const path = require('path')
const fs = require('fs')

protocol.registerSchemesAsPrivileged([
  { scheme: 'app', privileges: { secure: true, standard: true, corsEnabled: true, supportFetchAPI: true } }
])

app.whenReady().then(() => {
  protocol.handle('app', async (request) => {
    const url = new URL(request.url)
    
    // Serve static files
    const filePath = path.join(__dirname, 'dist', url.pathname)
    const normalized = path.normalize(filePath)
    const distDir = path.join(__dirname, 'dist')
    
    if (!normalized.startsWith(distDir)) {
      return new Response('Forbidden', { status: 403 })
    }
    
    try {
      const content = fs.readFileSync(normalized)
      const mimeType = getMimeType(normalized)
      
      return new Response(content, {
        headers: { 'Content-Type': mimeType }
      })
    } catch (error) {
      // Fallback to index.html สำหรับ SPA
      if (url.pathname !== '/') {
        try {
          const indexPath = path.join(distDir, 'index.html')
          const indexContent = fs.readFileSync(indexPath)
          return new Response(indexContent, {
            headers: { 'Content-Type': 'text/html; charset=utf-8' }
          })
        } catch (e) {
          return new Response('Not Found', { status: 404 })
        }
      }
      return new Response('Not Found', { status: 404 })
    }
  })
  
  createWindow()
})

function getMimeType(filePath) {
  const ext = path.extname(filePath).toLowerCase()
  const mimeTypes = {
    '.html': 'text/html; charset=utf-8',
    '.js': 'application/javascript; charset=utf-8',
    '.mjs': 'application/javascript; charset=utf-8',
    '.css': 'text/css; charset=utf-8',
    '.json': 'application/json; charset=utf-8',
    '.png': 'image/png',
    '.jpg': 'image/jpeg',
    '.jpeg': 'image/jpeg',
    '.gif': 'image/gif',
    '.svg': 'image/svg+xml',
    '.webp': 'image/webp',
    '.ico': 'image/x-icon',
    '.woff': 'font/woff',
    '.woff2': 'font/woff2',
    '.ttf': 'font/ttf',
    '.otf': 'font/otf',
    '.eot': 'application/vnd.ms-fontobject',
    '.mp4': 'video/mp4',
    '.mp3': 'audio/mpeg',
    '.wav': 'audio/wav',
    '.pdf': 'application/pdf',
    '.xml': 'application/xml',
    '.zip': 'application/zip'
  }
  return mimeTypes[ext] || 'application/octet-stream'
}
```

## Custom app:// URL Scheme

### สร้าง Protocol สำหรับ SPA

```javascript
// main.js - Full SPA Protocol Setup
const { app, BrowserWindow, protocol } = require('electron')
const path = require('path')
const fs = require('fs')

const isDev = process.env.NODE_ENV === 'development'
const DIST_DIR = path.join(__dirname, 'dist')

// ลงทะเบียน scheme
protocol.registerSchemesAsPrivileged([
  {
    scheme: 'app',
    privileges: {
      secure: true,
      standard: true,
      corsEnabled: true,
      supportFetchAPI: true
    }
  }
])

function createWindow() {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  if (isDev) {
    // Development: โหลดจาก dev server
    win.loadURL('http://localhost:5173')
    win.webContents.openDevTools()
  } else {
    // Production: โหลดจาก custom protocol
    win.loadURL('app://./index.html')
  }
  
  return win
}

app.whenReady().then(() => {
  // ลงทะเบียน protocol handler
  protocol.handle('app', async (request) => {
    const url = new URL(request.url)
    let pathname = url.pathname
    
    // normalize pathname
    if (pathname === '/') pathname = '/index.html'
    
    const filePath = path.join(DIST_DIR, pathname)
    const normalizedPath = path.normalize(filePath)
    
    // ป้องกัน path traversal
    if (!normalizedPath.startsWith(DIST_DIR)) {
      return new Response('Forbidden', { status: 403 })
    }
    
    // ลอง serve ไฟล์จริง
    if (fs.existsSync(normalizedPath) && fs.statSync(normalizedPath).isFile()) {
      const content = fs.readFileSync(normalizedPath)
      return new Response(content, {
        headers: { 'Content-Type': getMimeType(normalizedPath) }
      })
    }
    
    // SPA fallback: return index.html
    const indexPath = path.join(DIST_DIR, 'index.html')
    if (fs.existsSync(indexPath)) {
      const content = fs.readFileSync(indexPath)
      return new Response(content, {
        headers: { 'Content-Type': 'text/html; charset=utf-8' }
      })
    }
    
    return new Response('Not Found', { status: 404 })
  })
  
  createWindow()
})

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})

function getMimeType(fp) {
  const ext = path.extname(fp).toLowerCase()
  const types = {
    '.html': 'text/html; charset=utf-8',
    '.js': 'application/javascript',
    '.css': 'text/css',
    '.json': 'application/json',
    '.png': 'image/png',
    '.jpg': 'image/jpeg',
    '.svg': 'image/svg+xml',
    '.woff2': 'font/woff2'
  }
  return types[ext] || 'application/octet-stream'
}
```

## Intercept HTTP Requests

### interceptHttpProtocol

```javascript
// main.js
const { app, protocol, session } = require('electron')
const path = require('path')

app.whenReady().then(() => {
  const defaultSession = session.defaultSession
  
  // Intercept เพื่อ redirect
  defaultSession.protocol.interceptHttpProtocol('https', (request, callback) => {
    const url = new URL(request.url)
    
    // Redirect requests ไปยัง local server
    if (url.hostname === 'api.example.com') {
      callback({
        url: `http://localhost:3000${url.pathname}${url.search}`
      })
      return
    }
    
    // ปล่อยให้ผ่านตามปกติ
    callback({ url: request.url })
  })
})
```

### protocol.interceptFileProtocol

```javascript
// main.js - Intercept file:// protocol
const { app, protocol } = require('electron')
const path = require('path')
const fs = require('fs')

app.whenReady().then(() => {
  // Intercept เพื่อ add custom headers หรือ modify responses
  protocol.interceptFileProtocol('file', (request, callback) => {
    const url = new URL(request.url)
    let filePath = decodeURIComponent(url.pathname)
    
    // แก้ไข path สำหรับ Windows
    if (process.platform === 'win32') {
      filePath = filePath.slice(1) // ลบ leading slash
    }
    
    // ตรวจสอบว่าไฟล์มีอยู่
    if (fs.existsSync(filePath)) {
      callback({ path: filePath })
    } else {
      // Fallback to assets
      const assetPath = path.join(__dirname, 'assets', path.basename(filePath))
      if (fs.existsSync(assetPath)) {
        callback({ path: assetPath })
      } else {
        callback({ error: -6 })
      }
    }
  })
})
```

## Protocol Security

### ป้องกัน Path Traversal

```javascript
// main.js
const { protocol, app } = require('electron')
const path = require('path')
const fs = require('fs')

const ALLOWED_BASES = [
  path.join(__dirname, 'assets'),
  path.join(__dirname, 'dist'),
  app.getPath('userData') // ใน app.whenReady()
]

function isPathAllowed(filePath) {
  const normalized = path.normalize(filePath)
  return ALLOWED_BASES.some(base => normalized.startsWith(base))
}

function sanitizePath(inputPath) {
  // ลบ null bytes
  const cleaned = inputPath.replace(/\0/g, '')
  // normalize
  return path.normalize(cleaned)
}

app.whenReady().then(() => {
  ALLOWED_BASES.push(app.getPath('userData'))
  
  protocol.handle('app', async (request) => {
    const url = new URL(request.url)
    const rawPath = decodeURIComponent(url.pathname)
    
    // Sanitize input
    const sanitized = sanitizePath(rawPath)
    const fullPath = path.join(__dirname, 'assets', sanitized)
    const normalized = path.normalize(fullPath)
    
    // ตรวจสอบ path safety
    if (!isPathAllowed(normalized)) {
      console.warn('Blocked path traversal attempt:', rawPath)
      return new Response('Forbidden', { status: 403 })
    }
    
    try {
      if (!fs.existsSync(normalized) || !fs.statSync(normalized).isFile()) {
        return new Response('Not Found', { status: 404 })
      }
      
      const content = fs.readFileSync(normalized)
      return new Response(content, {
        headers: { 'Content-Type': getMimeType(normalized) }
      })
    } catch (error) {
      return new Response('Internal Error', { status: 500 })
    }
  })
})
```

### ป้องกัน Protocol Privilege Escalation

```javascript
// main.js
const { protocol, app, BrowserWindow } = require('electron')

// เปิด secure privileges เฉพาะที่จำเป็น
protocol.registerSchemesAsPrivileged([
  {
    scheme: 'app',
    privileges: {
      secure: true,
      standard: true,
      corsEnabled: false,        // ปิด CORS ถ้าไม่จำเป็น
      supportFetchAPI: true,
      allowServiceWorkers: false  // ปิด service workers ถ้าไม่ต้องการ
    }
  }
])

app.whenReady().then(() => {
  const win = new BrowserWindow({
    webPreferences: {
      // บังคับ HTTPS-like security สำหรับ app:// scheme
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  // จำกัด navigation
  win.webContents.on('will-navigate', (event, url) => {
    const parsed = new URL(url)
    
    // อนุญาตเฉพาะ app:// หรือ http://localhost (dev)
    const allowedOrigins = ['app:', 'http:']
    const allowedHosts = ['localhost', '127.0.0.1']
    
    if (!allowedOrigins.includes(parsed.protocol) && 
        !allowedHosts.includes(parsed.hostname)) {
      event.preventDefault()
      console.warn('Blocked navigation to:', url)
    }
  })
  
  win.loadURL('app://./index.html')
})
```

## Complete Protocol Handler Example

### main.js - Multi-Protocol App

```javascript
// main.js
const { app, BrowserWindow, protocol, ipcMain } = require('electron')
const path = require('path')
const fs = require('fs')

// ลงทะเบียน schemes
protocol.registerSchemesAsPrivileged([
  { scheme: 'app', privileges: { secure: true, standard: true, corsEnabled: true, supportFetchAPI: true } },
  { scheme: 'asset', privileges: { secure: true, standard: false } }
])

const ASSETS_DIR = path.join(__dirname, 'assets')
const DIST_DIR = path.join(__dirname, 'dist')

function getMimeType(filePath) {
  const ext = path.extname(filePath).toLowerCase()
  const types = {
    '.html': 'text/html; charset=utf-8',
    '.js': 'application/javascript; charset=utf-8',
    '.css': 'text/css; charset=utf-8',
    '.json': 'application/json; charset=utf-8',
    '.png': 'image/png',
    '.jpg': 'image/jpeg',
    '.jpeg': 'image/jpeg',
    '.gif': 'image/gif',
    '.svg': 'image/svg+xml',
    '.webp': 'image/webp',
    '.ico': 'image/x-icon',
    '.woff2': 'font/woff2',
    '.woff': 'font/woff',
    '.ttf': 'font/ttf',
    '.mp4': 'video/mp4',
    '.mp3': 'audio/mpeg',
    '.pdf': 'application/pdf'
  }
  return types[ext] || 'application/octet-stream'
}

function safeReadFile(basePath, urlPath) {
  const decodedPath = decodeURIComponent(urlPath)
  const safePath = decodedPath.replace(/\0/g, '').replace(/\.\./g, '')
  const fullPath = path.normalize(path.join(basePath, safePath))
  
  if (!fullPath.startsWith(basePath)) {
    throw new Error('Path traversal detected')
  }
  
  return { fullPath, content: fs.readFileSync(fullPath) }
}

let mainWindow

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1000,
    height: 700,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  mainWindow.loadURL('app://./index.html')
  return mainWindow
}

app.whenReady().then(() => {
  // Protocol 1: app:// - serve app files
  protocol.handle('app', async (request) => {
    const url = new URL(request.url)
    let pathname = url.pathname === '/' ? '/index.html' : url.pathname
    
    // ลอง dist ก่อน
    try {
      const { fullPath, content } = safeReadFile(DIST_DIR, pathname)
      if (fs.statSync(fullPath).isFile()) {
        return new Response(content, {
          headers: { 'Content-Type': getMimeType(fullPath) }
        })
      }
    } catch (e) { /* file not found */ }
    
    // SPA fallback
    try {
      const indexPath = path.join(DIST_DIR, 'index.html')
      const content = fs.readFileSync(indexPath)
      return new Response(content, { headers: { 'Content-Type': 'text/html; charset=utf-8' } })
    } catch (e) {
      return new Response('Not Found', { status: 404 })
    }
  })
  
  // Protocol 2: asset:// - serve assets
  protocol.handle('asset', async (request) => {
    const url = new URL(request.url)
    const assetPath = url.hostname + url.pathname // asset://images/logo.png
    
    try {
      const { fullPath, content } = safeReadFile(ASSETS_DIR, assetPath)
      if (fs.statSync(fullPath).isFile()) {
        return new Response(content, {
          headers: {
            'Content-Type': getMimeType(fullPath),
            'Cache-Control': 'public, max-age=86400'
          }
        })
      }
    } catch (e) {
      return new Response('Not Found', { status: 404 })
    }
    
    return new Response('Not Found', { status: 404 })
  })
  
  createWindow()
})

// IPC handlers
ipcMain.handle('protocol:get-info', () => ({
  schemes: ['app', 'asset'],
  assetsDir: ASSETS_DIR,
  distDir: DIST_DIR
}))

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

### preload.js

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('protocolAPI', {
  getInfo: () => ipcRenderer.invoke('protocol:get-info')
})
```

### index.html (ใน dist/)

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="Content-Security-Policy" 
        content="default-src 'self' app: asset:; style-src 'self' 'unsafe-inline' app: asset:; script-src 'self' app:; img-src 'self' app: asset: data:; font-src 'self' app: asset:;">
  <title>Protocol Handler Demo</title>
  <style>
    body { font-family: system-ui; background: #1a1a2e; color: #e0e0ff; padding: 20px; }
    h1 { color: #7b8cde; }
    .card { background: #16213e; border-radius: 8px; padding: 16px; margin: 12px 0; border: 1px solid #0f3460; }
    .url-test { display: flex; gap: 10px; align-items: center; flex-wrap: wrap; margin-bottom: 8px; }
    input { flex: 1; padding: 8px; background: #0f3460; border: 1px solid #533483; color: #e0e0ff; border-radius: 4px; }
    button { padding: 8px 16px; background: #533483; color: white; border: none; border-radius: 4px; cursor: pointer; }
    button:hover { background: #7b5ea7; }
    #protocol-info { font-family: monospace; font-size: 13px; background: #0d0d1a; padding: 12px; border-radius: 4px; color: #6ee7b7; }
    .result { margin-top: 8px; padding: 8px; background: #0f3460; border-radius: 4px; font-size: 13px; }
    img#test-image { max-width: 200px; border: 1px solid #533483; border-radius: 4px; }
  </style>
</head>
<body>
  <h1>Protocol Handler Demo</h1>
  
  <div class="card">
    <h2>Protocol Info</h2>
    <div id="protocol-info">กำลังโหลด...</div>
  </div>
  
  <div class="card">
    <h2>ทดสอบ app:// Protocol</h2>
    <div class="url-test">
      <input type="text" id="app-url" value="app://./assets/test.json" placeholder="app:// URL">
      <button id="btn-fetch-app">Fetch</button>
    </div>
    <div class="result" id="app-result">ผลลัพธ์จะแสดงที่นี่</div>
  </div>
  
  <div class="card">
    <h2>ทดสอบ asset:// Protocol</h2>
    <div class="url-test">
      <input type="text" id="asset-url" value="asset://images/logo.png" placeholder="asset:// URL">
      <button id="btn-load-asset">Load</button>
    </div>
    <img id="test-image" style="display:none;" alt="Asset test">
    <div class="result" id="asset-result">ผลลัพธ์จะแสดงที่นี่</div>
  </div>
  
  <script src="app://./renderer.js"></script>
</body>
</html>
```

### renderer.js

```javascript
// renderer.js
async function init() {
  // แสดง protocol info
  const info = await window.protocolAPI.getInfo()
  document.getElementById('protocol-info').textContent = JSON.stringify(info, null, 2)
}

// ทดสอบ app:// fetch
document.getElementById('btn-fetch-app').addEventListener('click', async () => {
  const url = document.getElementById('app-url').value
  const resultEl = document.getElementById('app-result')
  
  try {
    const response = await fetch(url)
    
    if (response.ok) {
      const contentType = response.headers.get('Content-Type') || ''
      
      if (contentType.includes('json')) {
        const json = await response.json()
        resultEl.textContent = `✅ Status: ${response.status}\n${JSON.stringify(json, null, 2)}`
      } else {
        const text = await response.text()
        resultEl.textContent = `✅ Status: ${response.status}\n${text.slice(0, 200)}`
      }
    } else {
      resultEl.textContent = `❌ Status: ${response.status} ${response.statusText}`
    }
  } catch (error) {
    resultEl.textContent = `❌ Error: ${error.message}`
  }
})

// ทดสอบ asset:// loading
document.getElementById('btn-load-asset').addEventListener('click', () => {
  const url = document.getElementById('asset-url').value
  const img = document.getElementById('test-image')
  const resultEl = document.getElementById('asset-result')
  
  img.style.display = 'none'
  img.onload = () => {
    img.style.display = 'block'
    resultEl.textContent = `✅ โหลดรูปสำเร็จ: ${img.naturalWidth}×${img.naturalHeight}px`
  }
  img.onerror = () => {
    resultEl.textContent = `❌ ไม่สามารถโหลดจาก: ${url}`
  }
  img.src = url
})

init()
```

## สรุป

Protocol Handler เป็นเครื่องมือที่ทรงพลังใน Electron:

- **protocol.registerSchemesAsPrivileged** - กำหนด security privileges ก่อน app ready
- **protocol.handle** - Modern API สำหรับ handle requests (Electron 25+)
- **Custom schemes** (app://, asset://) - serve local files อย่างปลอดภัย
- **SPA support** - fallback ไปยัง index.html สำหรับ client-side routing
- **Security** - ป้องกัน path traversal และ protocol privilege escalation

กฎสำคัญด้านความปลอดภัย:
1. ตรวจสอบ path ทุกครั้งด้วย `path.normalize()` และ `startsWith(allowedDir)`
2. ไม่ expose `file://` หรือ Node.js APIs โดยตรงผ่าน custom protocols
3. ใช้ CSP headers เพื่อจำกัด origins ที่อนุญาต

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Session & Cookies
