# Part 005: BrowserWindow Deep Dive
## ทำความเข้าใจ BrowserWindow อย่างละเอียด

---

## 🎯 เป้าหมายของบทเรียนนี้

หลังจากเรียนจบบทนี้ คุณจะเข้าใจ:
- BrowserWindow options ทั้งหมด
- การจัดการ multiple windows
- Window states และ persistence
- Custom window controls
- Frameless windows
- Transparent windows
- Window animations

---

## 1. BrowserWindow Options ครบถ้วน

### 1.1 Size & Position Options

```javascript
const { BrowserWindow } = require('electron')

const win = new BrowserWindow({
  // ===== ขนาดเริ่มต้น =====
  width: 1200,
  height: 800,
  
  // ===== ขนาดขั้นต่ำ-สูงสุด =====
  minWidth: 400,
  minHeight: 300,
  maxWidth: 1920,
  maxHeight: 1080,
  
  // ===== ตำแหน่ง =====
  x: 100,        // ตำแหน่ง X จากขอบซ้าย
  y: 100,        // ตำแหน่ง Y จากขอบบน
  center: true,  // center ใน screen (override x, y)
  
  // ===== Behavior =====
  resizable: true,       // ปรับขนาดได้
  movable: true,         // เลื่อนได้
  minimizable: true,     // minimize ได้
  maximizable: true,     // maximize ได้
  closable: true,        // close ได้
  focusable: true,       // รับ focus ได้
  alwaysOnTop: false,    // อยู่บนสุดเสมอ
  fullscreen: false,     // เริ่มต้นแบบ fullscreen
  fullscreenable: true,  // เข้า fullscreen ได้
  simpleFullscreen: false, // macOS: simple fullscreen mode
  skipTaskbar: false,    // ซ่อนจาก taskbar/dock
  kiosk: false,          // kiosk mode (fullscreen, cannot exit)
  modal: false,          // modal window (requires parent)
})
```

### 1.2 Appearance Options

```javascript
const win = new BrowserWindow({
  // ===== Background =====
  backgroundColor: '#1a1a2e',  // ป้องกัน white flash
  backgroundThrottling: true,  // ลด CPU เมื่อ background
  
  // ===== Opacity =====
  opacity: 1.0,  // 0 (transparent) to 1 (opaque)
  
  // ===== Title Bar =====
  frame: true,   // แสดง native frame (title bar + border)
  
  // macOS specific
  titleBarStyle: 'default',
  // 'default': standard macOS title bar
  // 'hidden': hidden title bar, traffic lights visible
  // 'hiddenInset': inset traffic lights
  // 'customButtonsOnHover': custom buttons on hover only
  
  // Windows specific
  titleBarOverlay: {
    color: '#1a1a2e',
    symbolColor: '#ffffff',
    height: 40
  },
  
  // ===== Vibrancy (macOS only) =====
  vibrancy: 'under-window',
  // 'appearance-based' | 'light' | 'dark' | 'titlebar'
  // 'selection' | 'menu' | 'popover' | 'sidebar'
  // 'medium-light' | 'ultra-dark' | 'header'
  // 'sheet' | 'window' | 'hud' | 'fullscreen-ui'
  // 'tooltip' | 'content' | 'under-window' | 'under-page'
  
  // ===== Blur (Windows/macOS) =====
  backgroundMaterial: 'none',
  // 'none' | 'auto' | 'mica' | 'acrylic' | 'tabbed' (Windows 11)
  
  // ===== Shadow =====
  hasShadow: true,  // macOS only
  
  // ===== Round Corners (macOS) =====
  roundedCorners: true,
  
  // ===== Traffic Lights Position (macOS) =====
  trafficLightPosition: { x: 10, y: 15 },
})
```

### 1.3 Web Preferences

```javascript
const win = new BrowserWindow({
  webPreferences: {
    // ===== Security (สำคัญมาก) =====
    nodeIntegration: false,           // NEVER true
    contextIsolation: true,           // ALWAYS true
    webSecurity: true,                // NEVER false
    allowRunningInsecureContent: false,
    
    // ===== Preload =====
    preload: path.join(__dirname, 'preload.js'),
    
    // ===== Sandbox =====
    sandbox: true,        // แนะนำ
    
    // ===== DevTools =====
    devTools: true,       // false ใน production
    
    // ===== User Agent =====
    userAgent: 'MyApp/1.0.0',
    
    // ===== JavaScript =====
    javascript: true,
    
    // ===== Images =====
    images: true,
    
    // ===== Spellcheck =====
    spellcheck: true,
    
    // ===== Zoom =====
    zoomFactor: 1.0,      // default zoom
    
    // ===== Scroll Bounce =====
    scrollBounce: false,  // macOS: rubber band scrolling
    
    // ===== Plugins =====
    plugins: false,
    
    // ===== WebGL =====
    webgl: true,
    
    // ===== Local Storage =====
    disableHtmlFullscreenWindowResize: false,
    
    // ===== Partition (for separate sessions) =====
    partition: 'persist:myapp',  // หรือ 'myapp' (non-persistent)
    
    // ===== Background Color =====
    defaultBackgroundColor: '#1a1a2e',
    
    // ===== Additional Arguments =====
    additionalArguments: ['--my-arg=value'],
  }
})
```

---

## 2. Window Customization

### 2.1 Frameless Window

```javascript
// main.js - Frameless Window
const win = new BrowserWindow({
  width: 900,
  height: 600,
  frame: false,          // ลบ native frame ออก
  backgroundColor: '#1a1a2e',
  transparent: false,    // ถ้า true window จะโปร่งแสง
  webPreferences: {
    preload: path.join(__dirname, 'preload.js'),
    nodeIntegration: false,
    contextIsolation: true
  }
})
```

```css
/* styles.css - สำหรับ frameless window */

/* ทำให้ drag window ได้ด้วย CSS */
.titlebar {
  -webkit-app-region: drag;    /* drag window */
  height: 40px;
  background: #1a1a2e;
  display: flex;
  align-items: center;
  padding: 0 16px;
}

/* ส่วนที่คลิกได้ (buttons, inputs) ต้องใส่ no-drag */
.titlebar button,
.titlebar input,
.titlebar a {
  -webkit-app-region: no-drag;
}

/* Window Control Buttons */
.window-controls {
  display: flex;
  gap: 8px;
  -webkit-app-region: no-drag;
}

.window-btn {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: none;
  cursor: pointer;
  transition: all 0.2s;
}

.close-btn { background: #ff5f57; }
.minimize-btn { background: #febc2e; }
.maximize-btn { background: #28c840; }

.close-btn:hover { background: #ff3b30; }
.minimize-btn:hover { background: #ff9500; }
.maximize-btn:hover { background: #34c759; }
```

```html
<!-- index.html - Custom Title Bar -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Frameless App</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <!-- Custom Title Bar -->
  <div class="titlebar">
    <!-- macOS style controls (left side) -->
    <div class="window-controls">
      <button class="window-btn close-btn" id="closeBtn"></button>
      <button class="window-btn minimize-btn" id="minimizeBtn"></button>
      <button class="window-btn maximize-btn" id="maximizeBtn"></button>
    </div>
    
    <span class="title">My App</span>
    
    <!-- Windows style controls (right side) -->
    <!-- หรือสร้าง custom controls สำหรับ Windows -->
  </div>
  
  <!-- App Content -->
  <div class="content">
    <p>Hello from Frameless Window!</p>
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

```javascript
// preload.js - Window Controls
contextBridge.exposeInMainWorld('windowControls', {
  minimize: () => ipcRenderer.invoke('window:minimize'),
  maximize: () => ipcRenderer.invoke('window:maximize'),
  close: () => ipcRenderer.invoke('window:close'),
  isMaximized: () => ipcRenderer.invoke('window:isMaximized'),
  onMaximize: (cb) => ipcRenderer.on('window:maximized', cb),
  onUnmaximize: (cb) => ipcRenderer.on('window:unmaximized', cb)
})

// main.js - Window Control Handlers
ipcMain.handle('window:minimize', () => mainWindow.minimize())
ipcMain.handle('window:maximize', () => {
  if (mainWindow.isMaximized()) {
    mainWindow.unmaximize()
  } else {
    mainWindow.maximize()
  }
})
ipcMain.handle('window:close', () => mainWindow.close())
ipcMain.handle('window:isMaximized', () => mainWindow.isMaximized())

// ส่ง events ไปยัง renderer
mainWindow.on('maximize', () => {
  mainWindow.webContents.send('window:maximized')
})
mainWindow.on('unmaximize', () => {
  mainWindow.webContents.send('window:unmaximized')
})

// renderer.js - Window Controls
document.getElementById('closeBtn').addEventListener('click', () => {
  window.windowControls.close()
})

document.getElementById('minimizeBtn').addEventListener('click', () => {
  window.windowControls.minimize()
})

document.getElementById('maximizeBtn').addEventListener('click', () => {
  window.windowControls.maximize()
})

// อัพเดท maximize button state
window.windowControls.onMaximize(() => {
  document.getElementById('maximizeBtn').classList.add('restore')
})

window.windowControls.onUnmaximize(() => {
  document.getElementById('maximizeBtn').classList.remove('restore')
})
```

### 2.2 Transparent Window

```javascript
// main.js - Transparent Window
const win = new BrowserWindow({
  width: 400,
  height: 300,
  transparent: true,   // Window โปร่งแสง
  frame: false,
  alwaysOnTop: true,   // อยู่บนแอพอื่น
  webPreferences: {
    preload: path.join(__dirname, 'preload.js'),
    nodeIntegration: false,
    contextIsolation: true
  }
})
```

```css
/* styles.css - Transparent Window */
body {
  background: transparent !important;
  overflow: hidden;
}

.widget {
  background: rgba(26, 26, 46, 0.85);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 16px;
  padding: 20px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
  -webkit-app-region: drag;
}

.widget-content {
  -webkit-app-region: no-drag;
}
```

---

## 3. Multiple Windows

### 3.1 Window Manager Pattern

```javascript
// main.js - Window Manager
const { BrowserWindow, ipcMain } = require('electron')
const path = require('path')

class WindowManager {
  constructor() {
    this.windows = new Map()  // name -> BrowserWindow
  }
  
  // สร้าง window ใหม่
  create(name, options = {}) {
    if (this.windows.has(name)) {
      // ถ้ามีอยู่แล้ว ให้ focus
      const existing = this.windows.get(name)
      if (!existing.isDestroyed()) {
        existing.focus()
        return existing
      }
    }
    
    const defaultOptions = {
      width: 800,
      height: 600,
      show: false,
      backgroundColor: '#1a1a2e',
      webPreferences: {
        preload: path.join(__dirname, 'preload.js'),
        nodeIntegration: false,
        contextIsolation: true
      }
    }
    
    const win = new BrowserWindow({ ...defaultOptions, ...options })
    
    win.once('ready-to-show', () => win.show())
    win.on('closed', () => this.windows.delete(name))
    
    this.windows.set(name, win)
    return win
  }
  
  // ดึง window ตามชื่อ
  get(name) {
    return this.windows.get(name)
  }
  
  // ปิด window
  close(name) {
    const win = this.windows.get(name)
    if (win && !win.isDestroyed()) {
      win.close()
    }
  }
  
  // ปิดทุก window
  closeAll() {
    for (const win of this.windows.values()) {
      if (!win.isDestroyed()) {
        win.close()
      }
    }
  }
  
  // ส่งข้อความไปทุก window
  broadcast(channel, data) {
    for (const win of this.windows.values()) {
      if (!win.isDestroyed()) {
        win.webContents.send(channel, data)
      }
    }
  }
  
  // ส่งข้อความไปยัง window เฉพาะ
  sendTo(name, channel, data) {
    const win = this.windows.get(name)
    if (win && !win.isDestroyed()) {
      win.webContents.send(channel, data)
    }
  }
  
  // จำนวน windows ที่เปิดอยู่
  get count() {
    return this.windows.size
  }
}

// ใช้งาน
const windowManager = new WindowManager()

app.whenReady().then(() => {
  // สร้าง main window
  const mainWin = windowManager.create('main', {
    width: 1200,
    height: 800
  })
  mainWin.loadFile('src/index.html')
})

// IPC: เปิด window ใหม่
ipcMain.handle('window:openSettings', () => {
  const settingsWin = windowManager.create('settings', {
    width: 600,
    height: 500,
    parent: windowManager.get('main'),
    modal: false
  })
  settingsWin.loadFile('src/settings.html')
})

// IPC: เปิด About window
ipcMain.handle('window:openAbout', () => {
  const aboutWin = windowManager.create('about', {
    width: 400,
    height: 300,
    resizable: false,
    minimizable: false,
    maximizable: false
  })
  aboutWin.loadFile('src/about.html')
})
```

### 3.2 Splash Screen

```javascript
// main.js - Splash Screen Pattern
async function createSplashScreen() {
  const splash = new BrowserWindow({
    width: 500,
    height: 300,
    frame: false,
    transparent: true,
    alwaysOnTop: true,
    skipTaskbar: true,
    webPreferences: { nodeIntegration: false, contextIsolation: true }
  })
  
  splash.loadFile('src/splash.html')
  return splash
}

async function createMainWindow() {
  const main = new BrowserWindow({
    width: 1200,
    height: 800,
    show: false,  // ซ่อนก่อน
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  
  main.loadFile('src/index.html')
  return main
}

app.whenReady().then(async () => {
  const splash = await createSplashScreen()
  const main = await createMainWindow()
  
  // รอให้ main window โหลดเสร็จ
  main.webContents.once('did-finish-load', async () => {
    // จำลอง loading time
    await new Promise(resolve => setTimeout(resolve, 2000))
    
    // ปิด splash และแสดง main
    splash.close()
    main.show()
    main.focus()
  })
})
```

```html
<!-- src/splash.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    
    body {
      background: transparent;
      display: flex;
      align-items: center;
      justify-content: center;
      height: 100vh;
      overflow: hidden;
    }
    
    .splash {
      background: linear-gradient(135deg, #1a1a2e, #16213e);
      border-radius: 20px;
      padding: 40px;
      text-align: center;
      width: 460px;
      height: 260px;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5);
      border: 1px solid rgba(255, 255, 255, 0.1);
    }
    
    .logo {
      font-size: 4rem;
      margin-bottom: 16px;
    }
    
    h1 {
      color: #e94560;
      font-size: 1.8rem;
      margin-bottom: 8px;
    }
    
    p {
      color: #888;
      font-size: 0.9rem;
      margin-bottom: 24px;
    }
    
    .progress-bar {
      width: 100%;
      height: 4px;
      background: rgba(255,255,255,0.1);
      border-radius: 2px;
      overflow: hidden;
    }
    
    .progress {
      height: 100%;
      background: #e94560;
      width: 0%;
      border-radius: 2px;
      animation: loading 2s ease-in-out forwards;
    }
    
    @keyframes loading {
      0% { width: 0%; }
      50% { width: 70%; }
      100% { width: 100%; }
    }
  </style>
</head>
<body>
  <div class="splash">
    <div class="logo">🖥️</div>
    <h1>My App</h1>
    <p>กำลังโหลด...</p>
    <div class="progress-bar">
      <div class="progress"></div>
    </div>
  </div>
</body>
</html>
```

---

## 4. Window State Persistence

### 4.1 บันทึกและกู้คืน Window State

```javascript
// windowState.js - ไฟล์แยกสำหรับจัดการ window state
const { app, screen } = require('electron')
const path = require('path')
const fs = require('fs')

class WindowStateKeeper {
  constructor(windowName) {
    this.windowName = windowName
    this.stateFile = path.join(
      app.getPath('userData'), 
      `window-state-${windowName}.json`
    )
    this.state = this.loadState()
  }
  
  loadState() {
    try {
      if (fs.existsSync(this.stateFile)) {
        return JSON.parse(fs.readFileSync(this.stateFile, 'utf8'))
      }
    } catch (e) {
      console.error('Failed to load window state:', e)
    }
    
    // Default state
    return {
      width: 1200,
      height: 800,
      isMaximized: false,
      isFullScreen: false
    }
  }
  
  saveState() {
    try {
      fs.writeFileSync(
        this.stateFile, 
        JSON.stringify(this.state, null, 2), 
        'utf8'
      )
    } catch (e) {
      console.error('Failed to save window state:', e)
    }
  }
  
  // ตรวจสอบว่า position ยังอยู่ใน screen ที่ใช้งานได้
  isValidPosition(x, y) {
    const displays = screen.getAllDisplays()
    return displays.some(display => {
      const bounds = display.bounds
      return x >= bounds.x && 
             y >= bounds.y && 
             x < bounds.x + bounds.width && 
             y < bounds.y + bounds.height
    })
  }
  
  // สร้าง BrowserWindow options จาก state
  getBounds() {
    const state = this.state
    
    // ตรวจสอบ position ว่าอยู่ใน screen
    if (state.x !== undefined && state.y !== undefined) {
      if (!this.isValidPosition(state.x, state.y)) {
        // Reset position ถ้าอยู่นอก screen
        delete state.x
        delete state.y
      }
    }
    
    return {
      width: state.width || 1200,
      height: state.height || 800,
      x: state.x,
      y: state.y
    }
  }
  
  // Track window และบันทึก state อัตโนมัติ
  track(win) {
    const saveTimeout = null
    
    const updateState = () => {
      if (!win.isDestroyed()) {
        this.state.isMaximized = win.isMaximized()
        this.state.isFullScreen = win.isFullScreen()
        
        if (!this.state.isMaximized && !this.state.isFullScreen) {
          const bounds = win.getBounds()
          this.state.x = bounds.x
          this.state.y = bounds.y
          this.state.width = bounds.width
          this.state.height = bounds.height
        }
      }
    }
    
    const scheduleSave = () => {
      clearTimeout(this._saveTimeout)
      this._saveTimeout = setTimeout(() => {
        updateState()
        this.saveState()
      }, 500)  // debounce 500ms
    }
    
    win.on('resize', scheduleSave)
    win.on('move', scheduleSave)
    win.on('maximize', scheduleSave)
    win.on('unmaximize', scheduleSave)
    win.on('enter-full-screen', scheduleSave)
    win.on('leave-full-screen', scheduleSave)
    
    win.on('close', () => {
      clearTimeout(this._saveTimeout)
      updateState()
      this.saveState()
    })
    
    // Restore state
    if (this.state.isMaximized) {
      win.maximize()
    }
    if (this.state.isFullScreen) {
      win.setFullScreen(true)
    }
    
    return win
  }
}

module.exports = WindowStateKeeper

// ใช้งานใน main.js:
const WindowStateKeeper = require('./windowState')

app.whenReady().then(() => {
  const windowState = new WindowStateKeeper('main')
  
  const win = new BrowserWindow({
    ...windowState.getBounds(),
    backgroundColor: '#1a1a2e',
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  
  windowState.track(win)
  win.loadFile('src/index.html')
})
```

---

## 5. Window เพิ่มเติม

### 5.1 Child Windows

```javascript
// main.js
let mainWindow, childWindow

// สร้าง main window
mainWindow = new BrowserWindow({
  width: 1000,
  height: 700,
  webPreferences: {
    preload: path.join(__dirname, 'preload.js'),
    nodeIntegration: false,
    contextIsolation: true
  }
})

// สร้าง child window
ipcMain.handle('openChild', (event, config) => {
  if (childWindow && !childWindow.isDestroyed()) {
    childWindow.focus()
    return
  }
  
  childWindow = new BrowserWindow({
    width: 600,
    height: 400,
    parent: mainWindow,         // parent window
    modal: config.modal || false, // modal จะ block parent
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  
  childWindow.loadFile('src/child.html')
  
  childWindow.on('closed', () => {
    childWindow = null
  })
})
```

### 5.2 Always on Top

```javascript
// main.js
ipcMain.handle('window:setAlwaysOnTop', (event, value) => {
  mainWindow.setAlwaysOnTop(value, 'floating')
  // 'normal', 'floating', 'torn-off-menu', 'modal-panel',
  // 'main-menu', 'status', 'pop-up-menu', 'screen-saver'
})

ipcMain.handle('window:isAlwaysOnTop', () => {
  return mainWindow.isAlwaysOnTop()
})
```

### 5.3 Picture-in-Picture Style Window

```javascript
// main.js - PiP Window
function createPiPWindow() {
  const pip = new BrowserWindow({
    width: 300,
    height: 200,
    frame: false,
    transparent: true,
    alwaysOnTop: true,
    resizable: true,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  
  // จำกัดขนาดขั้นต่ำ
  pip.setMinimumSize(200, 150)
  
  pip.loadFile('src/pip.html')
  return pip
}
```

---

## 6. Window Events ทั้งหมด

```javascript
const win = new BrowserWindow({ ... })

// ===== Show/Hide =====
win.on('show', () => console.log('shown'))
win.on('hide', () => console.log('hidden'))

// ===== Focus =====
win.on('focus', () => console.log('focused'))
win.on('blur', () => console.log('blurred'))

// ===== State =====
win.on('maximize', () => console.log('maximized'))
win.on('unmaximize', () => console.log('unmaximized'))
win.on('minimize', () => console.log('minimized'))
win.on('restore', () => console.log('restored'))

// ===== Fullscreen =====
win.on('enter-full-screen', () => console.log('fullscreen'))
win.on('leave-full-screen', () => console.log('no fullscreen'))
win.on('enter-html-full-screen', () => {}) // HTML5 fullscreen
win.on('leave-html-full-screen', () => {})

// ===== Size/Position =====
win.on('resize', () => {
  const [w, h] = win.getSize()
  console.log(`${w}x${h}`)
})

win.on('will-resize', (event, newBounds, details) => {
  // details: { edge: 'top' | 'right' | 'bottom' | 'left' | ... }
  // event.preventDefault() ป้องกัน resize
})

win.on('move', () => {
  const [x, y] = win.getPosition()
  console.log(`${x},${y}`)
})

win.on('will-move', (event, newBounds) => {
  // event.preventDefault() ป้องกัน move
})

win.on('moved', () => {}) // macOS only

// ===== Close =====
win.on('close', (event) => {
  event.preventDefault()  // ป้องกันการปิด
  // ถามผู้ใช้ก่อนปิด
})

win.on('closed', () => {
  // window ถูก destroy แล้ว
  // ห้ามใช้ win อีกต่อไป
})

// ===== WebContents Events (ผ่าน win.webContents) =====
win.webContents.on('did-finish-load', () => {
  console.log('Page loaded')
})

win.webContents.on('did-fail-load', (event, errorCode, errorDescription) => {
  console.error('Load failed:', errorDescription)
})

win.webContents.on('dom-ready', () => {
  console.log('DOM ready')
})

win.webContents.on('new-window', (event, url) => {
  event.preventDefault()  // ป้องกัน popup
})

win.webContents.on('before-input-event', (event, input) => {
  // input.key, input.type, input.modifiers
})
```

---

## 7. โปรเจกต์: Multi-Window Note App

### 7.1 โครงสร้าง

```
multi-window-notes/
├── main.js
├── preload.js
├── windowState.js
├── src/
│   ├── main/
│   │   ├── index.html
│   │   ├── renderer.js
│   │   └── styles.css
│   ├── editor/
│   │   ├── index.html
│   │   ├── renderer.js
│   │   └── styles.css
│   └── settings/
│       ├── index.html
│       ├── renderer.js
│       └── styles.css
└── package.json
```

### 7.2 main.js

```javascript
const { app, BrowserWindow, ipcMain, Menu } = require('electron')
const path = require('path')
const fs = require('fs')

const DATA_DIR = path.join(app.getPath('userData'), 'notes')
let mainWindow

function createMainWindow() {
  mainWindow = new BrowserWindow({
    width: 1000,
    height: 700,
    minWidth: 600,
    minHeight: 400,
    backgroundColor: '#1a1a2e',
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  
  mainWindow.loadFile('src/main/index.html')
  mainWindow.on('closed', () => { mainWindow = null })
}

// เปิด editor window สำหรับ note
ipcMain.handle('note:openEditor', async (event, noteId) => {
  const editorWin = new BrowserWindow({
    width: 700,
    height: 500,
    backgroundColor: '#1a1a2e',
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true,
      additionalArguments: [`--note-id=${noteId}`]
    }
  })
  
  editorWin.loadFile('src/editor/index.html')
  
  // ส่ง noteId ผ่าน query string แทน
  // editorWin.loadFile('src/editor/index.html', {
  //   query: { noteId }
  // })
})

// CRUD operations
function ensureDataDir() {
  if (!fs.existsSync(DATA_DIR)) {
    fs.mkdirSync(DATA_DIR, { recursive: true })
  }
}

ipcMain.handle('notes:getAll', async () => {
  ensureDataDir()
  const files = fs.readdirSync(DATA_DIR)
    .filter(f => f.endsWith('.json'))
  
  return files.map(file => {
    const data = JSON.parse(fs.readFileSync(path.join(DATA_DIR, file), 'utf8'))
    return data
  }).sort((a, b) => new Date(b.updatedAt) - new Date(a.updatedAt))
})

ipcMain.handle('notes:save', async (event, note) => {
  ensureDataDir()
  if (!note.id) {
    note.id = Date.now().toString(36)
    note.createdAt = new Date().toISOString()
  }
  note.updatedAt = new Date().toISOString()
  
  fs.writeFileSync(
    path.join(DATA_DIR, `${note.id}.json`),
    JSON.stringify(note, null, 2),
    'utf8'
  )
  
  // แจ้ง main window ให้ refresh
  if (mainWindow && !mainWindow.isDestroyed()) {
    mainWindow.webContents.send('notes:updated')
  }
  
  return note
})

ipcMain.handle('notes:delete', async (event, noteId) => {
  const filePath = path.join(DATA_DIR, `${noteId}.json`)
  if (fs.existsSync(filePath)) {
    fs.unlinkSync(filePath)
  }
  
  if (mainWindow && !mainWindow.isDestroyed()) {
    mainWindow.webContents.send('notes:updated')
  }
  
  return { success: true }
})

app.whenReady().then(createMainWindow)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

---

## 8. สรุป

```
BrowserWindow ที่ควรรู้:
├── Frameless Windows: frame: false + CSS -webkit-app-region
├── Transparent Windows: transparent: true + body background: transparent
├── Child Windows: parent option + modal
├── State Persistence: บันทึก/กู้คืน size/position
├── Splash Screen: show: false pattern
└── Multiple Windows: WindowManager class pattern

Security Checklist:
✅ nodeIntegration: false (always)
✅ contextIsolation: true (always)
✅ webSecurity: true (always)
✅ sandbox: true (recommended)
✅ preload script (always)
✅ Content-Security-Policy header
```

---

## 🔗 บทถัดไป

➡️ **[Part 006: IPC Communication Basics](part-006.md)** - IPC อย่างละเอียด

---

*📅 อัพเดทล่าสุด: 2024 | ระดับ: Beginner | เวลาเรียน: ~90 นาที*
