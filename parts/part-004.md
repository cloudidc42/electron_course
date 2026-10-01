# Part 004: Main Process & Renderer Process
## ทำความเข้าใจสถาปัตยกรรม Multi-Process ของ Electron

---

## 🎯 เป้าหมายของบทเรียนนี้

หลังจากเรียนจบบทนี้ คุณจะเข้าใจ:
- ทำไม Electron ถึงใช้ Multi-Process Architecture
- ความแตกต่างระหว่าง Main Process และ Renderer Process
- Process Communication ผ่าน IPC
- การจัดการ Memory และ Performance
- Best Practices ในการแบ่งงานระหว่าง Processes

---

## 1. ทำไมต้องใช้ Multi-Process Architecture?

### 1.1 ปัญหาของ Single-Process

ลองนึกภาพว่าแอพคุณเป็น Single Process:

```
Single Process App:
┌─────────────────────────────────────────┐
│          Process 1 (ทั้งหมด)             │
│                                         │
│  UI Code + Business Logic + File I/O    │
│  + Network + OS APIs + ...              │
│                                         │
│  ปัญหา:                                  │
│  - ถ้า UI code crash → ทั้งแอพ crash     │
│  - ถ้า JS ทำงานหนัก → UI หยุดตอบสนอง   │
│  - Security risk: ทุกอย่างเข้าถึง OS ได้ │
└─────────────────────────────────────────┘
```

### 1.2 วิธีแก้: Multi-Process

```
Multi-Process App (Electron):
┌─────────────────┐    IPC    ┌─────────────────┐
│   Main Process   │◄────────►│ Renderer Process │
│                 │          │                  │
│  ควบคุมแอพ       │          │  แสดง UI          │
│  จัดการ Windows  │          │  รับ User Input   │
│  OS Integration │          │  JavaScript/HTML  │
│                 │          │                  │
│  ถ้า crash:      │          │  ถ้า crash:        │
│  ทั้งแอพ close   │          │  แค่ tab นั้น      │
└─────────────────┘          └─────────────────┘
```

### 1.3 ข้อดีของ Multi-Process

```
Security:
- Renderer ไม่สามารถเข้าถึง OS โดยตรง
- Each renderer is sandboxed
- IPC เป็น controlled communication channel

Stability:
- Renderer crash ไม่ crash ทั้งแอพ
- Main process remains stable
- User experience ดีขึ้น

Performance:
- แต่ละ process ใช้ core แยกกัน
- Main process ไม่ถูก block โดย renderer work
- UI stays responsive
```

---

## 2. Main Process เชิงลึก

### 2.1 Main Process คืออะไร

Main Process คือ entry point ของแอพ Electron:

```javascript
// main.js - นี่คือ Main Process
const { app, BrowserWindow, ipcMain, Menu } = require('electron')
const path = require('path')

// Main Process มีสิทธิ์เต็ม:
// - สร้าง/ทำลาย Windows
// - เข้าถึง Node.js APIs ทั้งหมด
// - เรียก OS native functions
// - จัดการ System tray, Menu bar
// - ควบคุม app lifecycle
```

### 2.2 สิ่งที่ Main Process ทำได้

```javascript
const { 
  app,           // App lifecycle
  BrowserWindow, // Window management
  ipcMain,       // Receive from renderer
  Menu,          // Application menus
  Tray,          // System tray
  dialog,        // File/alert dialogs
  shell,         // Shell operations
  globalShortcut,// Global keyboard shortcuts
  powerMonitor,  // Battery/sleep events
  screen,        // Display information
  session,       // Browser sessions
  protocol,      // Custom URL protocols
  net,           // Network requests
  nativeTheme,   // Dark/light mode
  clipboard,     // Clipboard access
  crashReporter  // Crash reporting
} = require('electron')

const path = require('path')
const fs = require('fs')      // File system
const os = require('os')      // OS information
const child_process = require('child_process')  // Spawn processes
```

### 2.3 App Lifecycle Methods

```javascript
// main.js

// ===== App Events =====

// แอพพร้อมทำงาน (ready แล้ว)
app.on('ready', callback)
app.whenReady().then(callback)  // Promise version

// Window ทั้งหมดถูกปิด
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})

// macOS: คลิก Dock icon
app.on('activate', (event, hasVisibleWindows) => {
  if (!hasVisibleWindows) {
    createWindow()
  }
})

// กำลังจะปิดแอพ
app.on('before-quit', (event) => {
  console.log('Before quit')
  // event.preventDefault() // ยกเลิกการปิด
})

// ทุก windows ถูก close และแอพจะ quit
app.on('will-quit', (event) => {
  // cleanup before exit
  globalShortcut.unregisterAll()
})

// แอพ quit แล้ว
app.on('quit', (event, exitCode) => {
  console.log('Quit with code:', exitCode)
})

// ===== App Methods =====

// ออกจากแอพ
app.quit()

// Force quit (ไม่รอ will-quit)
app.exit(0)

// รีสตาร์ทแอพ
app.relaunch({ args: process.argv.slice(1) })
app.quit()

// ดึง path ของระบบ
app.getPath('userData')     // ~/Library/Application Support/MyApp
app.getPath('temp')         // /tmp หรือ %TEMP%
app.getPath('home')         // ~/
app.getPath('desktop')      // ~/Desktop
app.getPath('downloads')    // ~/Downloads
app.getPath('documents')    // ~/Documents
app.getPath('pictures')     // ~/Pictures
app.getPath('videos')       // ~/Videos
app.getPath('music')        // ~/Music
app.getPath('exe')          // path to electron executable
app.getPath('logs')         // ~/Library/Logs/MyApp

// ตรวจสอบ single instance
const gotTheLock = app.requestSingleInstanceLock()
if (!gotTheLock) {
  app.quit()  // มีแอพรันอยู่แล้ว
}
```

### 2.4 BrowserWindow Management

```javascript
const { BrowserWindow } = require('electron')

// สร้าง window
function createMainWindow() {
  const win = new BrowserWindow({
    // ===== ขนาด =====
    width: 1200,
    height: 800,
    minWidth: 600,
    minHeight: 400,
    maxWidth: 1920,
    maxHeight: 1080,
    
    // ===== ตำแหน่ง =====
    x: 100,
    y: 100,
    center: true,   // center บนหน้าจอ
    
    // ===== Window Behavior =====
    resizable: true,
    movable: true,
    minimizable: true,
    maximizable: true,
    closable: true,
    focusable: true,
    alwaysOnTop: false,
    fullscreen: false,
    fullscreenable: true,
    skipTaskbar: false,  // ซ่อนจาก taskbar
    
    // ===== Appearance =====
    show: false,         // ซ่อนก่อน (ป้องกัน flash)
    opacity: 1.0,        // 0-1
    backgroundColor: '#1a1a2e',
    hasShadow: true,
    
    // ===== Title Bar =====
    frame: true,
    titleBarStyle: 'default',  // 'default' | 'hidden' | 'hiddenInset' | 'customButtonsOnHover'
    title: 'My App',
    
    // ===== Web Preferences =====
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,      // NEVER true in production
      contextIsolation: true,      // ALWAYS true
      sandbox: true,               // แนะนำ
      webSecurity: true,           // NEVER false in production
      allowRunningInsecureContent: false,
      enableRemoteModule: false,   // Deprecated - don't use
      spellcheck: true
    }
  })
  
  // โหลด URL หรือ File
  win.loadFile('src/index.html')
  // หรือ
  // win.loadURL('http://localhost:3000')
  
  // แสดงเมื่อพร้อม
  win.once('ready-to-show', () => {
    win.show()
    win.focus()
  })
  
  return win
}

// ===== Window Events =====
const win = createMainWindow()

win.on('close', (event) => {
  // event.preventDefault()  // ป้องกันการปิด
})

win.on('closed', () => {
  // Window ถูก destroy แล้ว
})

win.on('resize', () => {
  const [width, height] = win.getSize()
  console.log(`Resized to ${width}x${height}`)
})

win.on('move', () => {
  const [x, y] = win.getPosition()
  console.log(`Moved to ${x},${y}`)
})

win.on('focus', () => {
  console.log('Window focused')
})

win.on('blur', () => {
  console.log('Window blurred')
})

win.on('maximize', () => {
  console.log('Window maximized')
})

win.on('minimize', () => {
  console.log('Window minimized')
})

win.on('restore', () => {
  console.log('Window restored')
})

// ===== Window Methods =====

// ขนาดและตำแหน่ง
win.setSize(800, 600)
win.getSize()          // [800, 600]
win.setPosition(100, 100)
win.getPosition()      // [100, 100]
win.center()

// State
win.show()
win.hide()
win.focus()
win.blur()
win.minimize()
win.maximize()
win.unmaximize()
win.restore()
win.close()
win.destroy()

// Check state
win.isVisible()
win.isFocused()
win.isMinimized()
win.isMaximized()
win.isFullScreen()
win.isDestroyed()

// อื่นๆ
win.setTitle('New Title')
win.getTitle()
win.setAlwaysOnTop(true)
win.setFullScreen(true)
win.loadURL('https://example.com')
win.reload()

// ดึง window ทั้งหมด
const windows = BrowserWindow.getAllWindows()
const focused = BrowserWindow.getFocusedWindow()
const fromId = BrowserWindow.fromId(winId)
```

---

## 3. Renderer Process เชิงลึก

### 3.1 Renderer Process คืออะไร

```
Renderer Process = Chromium Tab + Optional Node.js access

แต่ละ BrowserWindow มี 1 Renderer Process
แต่ละ iframe อาจมี Renderer Process แยก
```

### 3.2 สิ่งที่ Renderer ทำได้

```javascript
// renderer.js หรือใน <script> tags

// ===== Web APIs (เหมือน browser ปกติ) =====
document.getElementById('...')
window.location.href
fetch('https://api.example.com')
localStorage.setItem('key', 'value')
sessionStorage.getItem('key')
navigator.geolocation.getCurrentPosition()
window.matchMedia('(prefers-color-scheme: dark)')
new WebSocket('wss://...')
new Worker('worker.js')

// ===== Electron APIs (ผ่าน contextBridge เท่านั้น) =====
window.electronAPI.someFunction()

// ===== ที่ไม่ทำได้โดยตรง (เพื่อ Security) =====
// require('fs')          // ❌ ถ้า contextIsolation: true
// require('electron')    // ❌ ถ้า contextIsolation: true
// process.env.SECRET     // ❌ ถ้า sandbox: true
```

### 3.3 Renderer Process Lifecycle

```javascript
// renderer.js

// ===== DOM Events =====

// DOM โหลดเสร็จ (HTML parsed)
document.addEventListener('DOMContentLoaded', () => {
  console.log('DOM ready')
  initApp()
})

// ทุกอย่างโหลดเสร็จ (images, scripts)
window.addEventListener('load', () => {
  console.log('Page fully loaded')
})

// กำลังออกจากหน้า
window.addEventListener('beforeunload', (event) => {
  // event.preventDefault()  // อาจแสดง "Are you sure?" dialog
  event.returnValue = ''
})

// ออกจากหน้าแล้ว
window.addEventListener('unload', () => {
  cleanup()
})

// ===== Visibility =====

document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    console.log('Window is hidden')
    pauseAnimations()
  } else {
    console.log('Window is visible')
    resumeAnimations()
  }
})
```

---

## 4. IPC Communication เชิงลึก

### 4.1 ประเภทของ IPC

```
IPC Patterns:
├── Fire-and-forget (ipcRenderer.send / ipcMain.on)
├── Request-Response (ipcRenderer.invoke / ipcMain.handle) ← แนะนำ
├── Main to Renderer (webContents.send / ipcRenderer.on)
└── Between Windows (ผ่าน Main process)
```

### 4.2 Pattern 1: Fire-and-Forget

```javascript
// preload.js
contextBridge.exposeInMainWorld('api', {
  sendLog: (message) => ipcRenderer.send('log', message)
})

// renderer.js
window.api.sendLog('User clicked button')  // ไม่รอผลลัพธ์

// main.js
ipcMain.on('log', (event, message) => {
  console.log('[LOG]', message)
  // ไม่ต้อง reply
})
```

### 4.3 Pattern 2: Request-Response (Invoke/Handle) ✅ แนะนำ

```javascript
// preload.js
contextBridge.exposeInMainWorld('api', {
  // Invoke: ส่งและรอผลลัพธ์
  readFile: (filePath) => ipcRenderer.invoke('readFile', filePath),
  writeFile: (filePath, content) => ipcRenderer.invoke('writeFile', filePath, content),
  showDialog: (options) => ipcRenderer.invoke('showDialog', options)
})

// renderer.js
async function loadData() {
  try {
    const content = await window.api.readFile('/path/to/file.txt')
    displayContent(content)
  } catch (error) {
    showError(error.message)
  }
}

// main.js
ipcMain.handle('readFile', async (event, filePath) => {
  // ถ้า throw error จะถูกส่งกลับไปยัง renderer
  const content = await fs.promises.readFile(filePath, 'utf8')
  return content  // ค่านี้จะถูกส่งกลับไปยัง renderer
})

ipcMain.handle('writeFile', async (event, filePath, content) => {
  await fs.promises.writeFile(filePath, content, 'utf8')
  return { success: true }
})

ipcMain.handle('showDialog', async (event, options) => {
  const result = await dialog.showOpenDialog(mainWindow, options)
  return result
})
```

### 4.4 Pattern 3: Main to Renderer (Push)

```javascript
// main.js - ส่งข้อมูลไปยัง renderer
function sendToRenderer(channel, data) {
  const windows = BrowserWindow.getAllWindows()
  windows.forEach(win => {
    if (!win.isDestroyed()) {
      win.webContents.send(channel, data)
    }
  })
}

// ตัวอย่าง: ส่ง progress updates
function processLargeFile(filePath) {
  const stream = fs.createReadStream(filePath)
  let processed = 0
  const total = fs.statSync(filePath).size
  
  stream.on('data', (chunk) => {
    processed += chunk.length
    const progress = Math.round((processed / total) * 100)
    sendToRenderer('download-progress', { progress, processed, total })
  })
  
  stream.on('end', () => {
    sendToRenderer('download-complete', { filePath })
  })
}

// preload.js
contextBridge.exposeInMainWorld('api', {
  onProgress: (callback) => {
    ipcRenderer.on('download-progress', (event, data) => callback(data))
  },
  onComplete: (callback) => {
    ipcRenderer.on('download-complete', (event, data) => callback(data))
  },
  // cleanup
  removeAllListeners: (channel) => {
    ipcRenderer.removeAllListeners(channel)
  }
})

// renderer.js
window.api.onProgress((data) => {
  progressBar.style.width = `${data.progress}%`
  progressText.textContent = `${data.progress}%`
})

window.api.onComplete((data) => {
  showSuccess(`ดาวน์โหลดเสร็จแล้ว: ${data.filePath}`)
})

// Cleanup เมื่อ component ถูก destroy
window.addEventListener('beforeunload', () => {
  window.api.removeAllListeners('download-progress')
  window.api.removeAllListeners('download-complete')
})
```

### 4.5 Pattern 4: Two-Way between Windows

```javascript
// main.js
let mainWin, settingsWin

ipcMain.handle('openSettings', async () => {
  settingsWin = new BrowserWindow({
    parent: mainWin,
    modal: true,
    width: 500,
    height: 400,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  settingsWin.loadFile('settings.html')
})

// รับการตั้งค่าจาก settings window
ipcMain.handle('saveSettings', async (event, settings) => {
  // บันทึก settings
  saveSettings(settings)
  // ส่ง settings ใหม่ไปยัง main window
  mainWin.webContents.send('settingsUpdated', settings)
  settingsWin.close()
  return { success: true }
})
```

---

## 5. Process Communication - ตัวอย่างโปรเจกต์

### 5.1 สร้าง File Explorer อย่างง่าย

```javascript
// main.js (excerpt)
const { ipcMain } = require('electron')
const fs = require('fs').promises
const path = require('path')

// อ่าน directory
ipcMain.handle('fs:readDir', async (event, dirPath) => {
  const entries = await fs.readdir(dirPath, { withFileTypes: true })
  
  return entries.map(entry => ({
    name: entry.name,
    path: path.join(dirPath, entry.name),
    isDirectory: entry.isDirectory(),
    isFile: entry.isFile()
  }))
})

// อ่านไฟล์
ipcMain.handle('fs:readFile', async (event, filePath) => {
  const stat = await fs.stat(filePath)
  
  // ตรวจขนาดไฟล์ (จำกัด 10MB)
  if (stat.size > 10 * 1024 * 1024) {
    throw new Error('File too large (max 10MB)')
  }
  
  const content = await fs.readFile(filePath, 'utf8')
  return { content, size: stat.size, modified: stat.mtime }
})

// ดู file stats
ipcMain.handle('fs:stat', async (event, filePath) => {
  const stat = await fs.stat(filePath)
  return {
    size: stat.size,
    created: stat.birthtime,
    modified: stat.mtime,
    isDirectory: stat.isDirectory(),
    isFile: stat.isFile()
  }
})
```

```javascript
// preload.js (excerpt)
contextBridge.exposeInMainWorld('fsAPI', {
  readDir: (dirPath) => ipcRenderer.invoke('fs:readDir', dirPath),
  readFile: (filePath) => ipcRenderer.invoke('fs:readFile', filePath),
  stat: (filePath) => ipcRenderer.invoke('fs:stat', filePath),
  
  // Helper: get home directory
  getHomePath: () => ipcRenderer.invoke('fs:homePath')
})
```

```javascript
// renderer.js (excerpt)
class FileExplorer {
  constructor() {
    this.currentPath = null
    this.history = []
    this.historyIndex = -1
  }
  
  async navigate(dirPath) {
    try {
      const entries = await window.fsAPI.readDir(dirPath)
      
      // Sort: directories first, then files
      entries.sort((a, b) => {
        if (a.isDirectory && !b.isDirectory) return -1
        if (!a.isDirectory && b.isDirectory) return 1
        return a.name.localeCompare(b.name)
      })
      
      this.currentPath = dirPath
      this.renderEntries(entries)
      this.updateBreadcrumb(dirPath)
      
      // Add to history
      this.history = this.history.slice(0, this.historyIndex + 1)
      this.history.push(dirPath)
      this.historyIndex++
      
    } catch (error) {
      console.error('Navigation error:', error)
    }
  }
  
  renderEntries(entries) {
    const container = document.getElementById('fileList')
    container.innerHTML = ''
    
    entries.forEach(entry => {
      const item = document.createElement('div')
      item.className = 'file-item'
      item.innerHTML = `
        <span class="icon">${entry.isDirectory ? '📁' : '📄'}</span>
        <span class="name">${entry.name}</span>
      `
      
      item.addEventListener('dblclick', async () => {
        if (entry.isDirectory) {
          await this.navigate(entry.path)
        } else {
          await this.openFile(entry.path)
        }
      })
      
      container.appendChild(item)
    })
  }
  
  async openFile(filePath) {
    try {
      const { content, size } = await window.fsAPI.readFile(filePath)
      const editor = document.getElementById('editor')
      editor.textContent = content
    } catch (error) {
      alert(`Error: ${error.message}`)
    }
  }
}

const explorer = new FileExplorer()
document.addEventListener('DOMContentLoaded', async () => {
  await explorer.navigate('/Users')  // หรือ path.join(__dirname, '..')
})
```

---

## 6. Memory Management

### 6.1 Renderer Process Memory

```javascript
// renderer.js

// ===== ป้องกัน Memory Leaks =====

// ❌ BAD: Event listener ที่ไม่ถูกลบ
function setupBadListeners() {
  for (let i = 0; i < 1000; i++) {
    document.addEventListener('click', handleClick)  // ไม่ถูกลบ!
  }
}

// ✅ GOOD: ลบ listener เมื่อไม่ต้องการ
class Component {
  constructor(element) {
    this.element = element
    this.handlers = []
  }
  
  on(event, handler) {
    this.element.addEventListener(event, handler)
    this.handlers.push({ event, handler })
  }
  
  destroy() {
    // ลบ listeners ทั้งหมด
    this.handlers.forEach(({ event, handler }) => {
      this.element.removeEventListener(event, handler)
    })
    this.handlers = []
  }
}

// ===== ป้องกัน Circular References =====

// ❌ BAD: Circular reference
function createNode(data) {
  const node = { data }
  node.self = node  // circular!
  return node
}

// ✅ GOOD: ใช้ WeakRef สำหรับ back-references
function createNode(parent, data) {
  return {
    data,
    parent: new WeakRef(parent)  // ไม่ prevent garbage collection
  }
}
```

### 6.2 Main Process Memory

```javascript
// main.js

// ===== Monitor Memory Usage =====
setInterval(() => {
  const usage = process.memoryUsage()
  console.log({
    rss: `${Math.round(usage.rss / 1024 / 1024)} MB`,       // Resident Set Size
    heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)} MB`,
    heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)} MB`,
    external: `${Math.round(usage.external / 1024 / 1024)} MB`
  })
}, 30000)  // ทุก 30 วินาที

// ===== Clear unused resources =====
let cache = new Map()

function clearOldCache() {
  const maxAge = 5 * 60 * 1000  // 5 minutes
  const now = Date.now()
  
  for (const [key, entry] of cache.entries()) {
    if (now - entry.timestamp > maxAge) {
      cache.delete(key)
    }
  }
}

setInterval(clearOldCache, 60000)  // ทุก 1 นาที
```

---

## 7. Process Security

### 7.1 Security Configuration

```javascript
// main.js - Security Settings

function createSecureWindow() {
  const win = new BrowserWindow({
    webPreferences: {
      // ===== Security: NEVER change these in production =====
      nodeIntegration: false,          // ❌ Never true
      nodeIntegrationInWorker: false,  // ❌ Never true
      nodeIntegrationInSubFrames: false,// ❌ Never true
      contextIsolation: true,          // ✅ Always true
      webSecurity: true,               // ✅ Always true
      allowRunningInsecureContent: false, // ✅ Always false
      experimentalFeatures: false,     // ✅ Usually false
      
      // ===== Security: Depends on your needs =====
      sandbox: true,     // แนะนำ - restrict renderer capabilities
      preload: path.join(__dirname, 'preload.js'),  // ใช้ preload เสมอ
      
      // ===== Permissions =====
      // จัดการ permissions อย่างระมัดระวัง
    }
  })
  
  // ===== Content Security Policy =====
  win.webContents.session.webRequest.onHeadersReceived((details, callback) => {
    callback({
      responseHeaders: {
        ...details.responseHeaders,
        'Content-Security-Policy': [
          "default-src 'self'; " +
          "script-src 'self'; " +
          "style-src 'self' 'unsafe-inline'; " +
          "img-src 'self' data: https:; " +
          "connect-src 'self' https://api.example.com"
        ]
      }
    })
  })
  
  return win
}
```

### 7.2 ตรวจสอบ Process Context

```javascript
// ตรวจสอบว่ารันใน Main หรือ Renderer
function isMainProcess() {
  return process.type === 'browser'
}

function isRendererProcess() {
  return process.type === 'renderer'
}

// ใน Main Process:
console.log(process.type)  // 'browser'

// ใน Renderer Process:
console.log(process.type)  // 'renderer'

// ใน Preload Script:
console.log(process.type)  // 'renderer' (รันใน renderer context)
```

---

## 8. Debugging Multi-Process App

### 8.1 Debug Main Process

```bash
# ใช้ --inspect flag
electron . --inspect=9229

# หรือ
electron . --inspect-brk=9229  # รอ debugger ก่อนรัน
```

จากนั้นเปิด Chrome → `chrome://inspect` → Configure... → เพิ่ม `localhost:9229`

### 8.2 Debug Renderer Process

```javascript
// main.js - เปิด DevTools อัตโนมัติ
mainWindow.webContents.openDevTools()

// หรือ keyboard shortcut ใน renderer
document.addEventListener('keydown', (e) => {
  if (e.key === 'F12') {
    // ส่งไปยัง main process เพื่อเปิด DevTools
    window.electronAPI.openDevTools()
  }
})

// main.js
ipcMain.handle('openDevTools', () => {
  mainWindow.webContents.openDevTools()
})
```

### 8.3 Process Information

```javascript
// Main Process
console.log({
  type: process.type,       // 'browser'
  pid: process.pid,
  platform: process.platform,
  arch: process.arch,
  versions: process.versions,
  env: process.env.NODE_ENV
})

// Renderer Process (จาก preload)
contextBridge.exposeInMainWorld('processInfo', {
  type: process.type,       // 'renderer'
  versions: process.versions,
  platform: process.platform
})
```

---

## 9. Best Practices

### 9.1 Main Process

```
✅ DO:
- ใส่ business logic ที่ต้องการ Node.js ใน Main Process
- ใช้ ipcMain.handle (async) แทน ipcMain.on สำหรับ two-way communication
- Handle errors ใน IPC handlers
- ใช้ app.getPath() สำหรับ file paths
- จัดการ Window lifecycle อย่างระมัดระวัง

❌ DON'T:
- อย่าใส่ UI logic ใน Main Process
- อย่าทำงานหนักใน Main Process (ใช้ Worker threads แทน)
- อย่าเก็บข้อมูล sensitive ใน IPC messages โดยไม่เข้ารหัส
```

### 9.2 Renderer Process

```
✅ DO:
- ใส่ UI logic ทั้งหมดใน Renderer
- ใช้ contextBridge เพื่อเข้าถึง Electron APIs
- Handle loading states และ errors
- ลบ event listeners เมื่อไม่ต้องการ

❌ DON'T:
- อย่าใช้ nodeIntegration: true
- อย่าทำ CPU-intensive tasks ใน main thread
- อย่า expose Node.js APIs โดยตรงผ่าน contextBridge
```

---

## 10. สรุป

```
Main Process:
├── รัน Node.js environment เต็มรูปแบบ
├── ควบคุม App lifecycle
├── สร้าง/จัดการ BrowserWindows
├── เข้าถึง OS APIs
└── สื่อสารกับ Renderer ผ่าน IPC

Renderer Process:
├── รัน Chromium (web browser)
├── แสดง HTML/CSS/JavaScript UI
├── รับ User interactions
├── สื่อสารกับ Main ผ่าน IPC
└── (Optional) เข้าถึง Node.js ผ่าน Preload + contextBridge

IPC (Inter-Process Communication):
├── ipcRenderer.invoke / ipcMain.handle → Request-Response ✅
├── ipcRenderer.send / ipcMain.on → Fire-and-Forget
└── webContents.send / ipcRenderer.on → Push from Main
```

---

## 🔗 บทถัดไป

➡️ **[Part 005: BrowserWindow Deep Dive](part-005.md)** - ทำความเข้าใจ BrowserWindow อย่างละเอียด

---

*📅 อัพเดทล่าสุด: 2024 | ระดับ: Beginner | เวลาเรียน: ~75 นาที*
