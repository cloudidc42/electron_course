# ตอนที่ 15: Keyboard Shortcuts

## Global Shortcuts ใน Electron

Electron รองรับ keyboard shortcuts 2 ประเภทหลัก:
1. **Global Shortcuts** - ทำงานแม้แอปไม่ได้ focus
2. **Local Shortcuts** - ทำงานเฉพาะเมื่อ window มี focus

## globalShortcut.register

`globalShortcut` module ใช้สำหรับลงทะเบียน keyboard shortcuts ที่ทำงานในระดับ system

```javascript
// main.js
const { app, globalShortcut } = require('electron')

app.whenReady().then(() => {
  // ลงทะเบียน global shortcut
  const registered = globalShortcut.register('CommandOrControl+Shift+P', () => {
    console.log('Global shortcut triggered!')
  })
  
  if (!registered) {
    console.error('Shortcut registration failed - may be taken by another app')
  }
  
  // ตรวจสอบว่าลงทะเบียนสำเร็จหรือไม่
  console.log('Is registered:', globalShortcut.isRegistered('CommandOrControl+Shift+P'))
})

// ยกเลิกการลงทะเบียนก่อน quit
app.on('will-quit', () => {
  globalShortcut.unregisterAll()
})
```

### Accelerator Syntax

```javascript
// main.js - ตัวอย่าง accelerators
const { globalShortcut } = require('electron')

// CommandOrControl: Cmd บน macOS, Ctrl บน Windows/Linux
globalShortcut.register('CommandOrControl+C', handler)

// Alt/Option key
globalShortcut.register('Alt+F4', handler)

// Shift key
globalShortcut.register('Shift+F5', handler)

// Function keys
globalShortcut.register('F12', handler)

// ตัวอักษรและตัวเลข
globalShortcut.register('CommandOrControl+1', handler)

// Special keys
globalShortcut.register('CommandOrControl+Space', handler)
globalShortcut.register('CommandOrControl+Return', handler)
globalShortcut.register('CommandOrControl+BackSpace', handler)

// Arrow keys
globalShortcut.register('CommandOrControl+Up', handler)
globalShortcut.register('CommandOrControl+Down', handler)

// Media keys
globalShortcut.register('MediaPlayPause', handler)
globalShortcut.register('MediaNextTrack', handler)
globalShortcut.register('MediaPreviousTrack', handler)
globalShortcut.register('VolumeUp', handler)
globalShortcut.register('VolumeDown', handler)
globalShortcut.register('VolumeMute', handler)

function handler() {
  console.log('Shortcut pressed!')
}
```

## globalShortcut.unregister

```javascript
// main.js
const { globalShortcut } = require('electron')

// ยกเลิก shortcut เดี่ยว
globalShortcut.unregister('CommandOrControl+Shift+P')

// ยกเลิกทั้งหมด
globalShortcut.unregisterAll()

// ลงทะเบียน + ยกเลิก ตามสถานการณ์
function enableShortcuts() {
  globalShortcut.register('CommandOrControl+Shift+S', () => {
    // Screenshot or capture
    takeScreenshot()
  })
}

function disableShortcuts() {
  globalShortcut.unregister('CommandOrControl+Shift+S')
}
```

### Dynamic Shortcut Management

```javascript
// main.js - เพิ่ม/ลบ shortcuts แบบ dynamic
const { globalShortcut, BrowserWindow, ipcMain } = require('electron')

const registeredShortcuts = new Map()

function registerShortcut(accelerator, action, handler) {
  // ยกเลิก shortcut เดิมถ้ามี
  if (registeredShortcuts.has(accelerator)) {
    globalShortcut.unregister(accelerator)
  }
  
  const success = globalShortcut.register(accelerator, handler)
  
  if (success) {
    registeredShortcuts.set(accelerator, { action, handler })
    console.log(`Registered: ${accelerator} -> ${action}`)
  }
  
  return success
}

function unregisterShortcut(accelerator) {
  if (registeredShortcuts.has(accelerator)) {
    globalShortcut.unregister(accelerator)
    registeredShortcuts.delete(accelerator)
    return true
  }
  return false
}

// IPC handlers
ipcMain.handle('shortcuts:register', (event, { accelerator, action }) => {
  const mainWindow = BrowserWindow.fromWebContents(event.sender)
  
  const success = registerShortcut(accelerator, action, () => {
    mainWindow?.webContents.send('shortcut:triggered', { accelerator, action })
  })
  
  return { success, accelerator, action }
})

ipcMain.handle('shortcuts:unregister', (event, accelerator) => {
  const success = unregisterShortcut(accelerator)
  return { success }
})

ipcMain.handle('shortcuts:list', () => {
  return Array.from(registeredShortcuts.entries()).map(([accelerator, info]) => ({
    accelerator,
    action: info.action,
    isRegistered: globalShortcut.isRegistered(accelerator)
  }))
})

// ล้างก่อน quit
require('electron').app.on('will-quit', () => {
  globalShortcut.unregisterAll()
  registeredShortcuts.clear()
})
```

## Local Shortcuts (webContents)

Local shortcuts ทำงานเฉพาะเมื่อ window มี focus

### วิธีที่ 1: ผ่าน Menu

```javascript
// main.js
const { Menu, app, BrowserWindow } = require('electron')

app.whenReady().then(() => {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: __dirname + '/preload.js',
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  const menu = Menu.buildFromTemplate([
    {
      label: 'File',
      submenu: [
        {
          label: 'New',
          accelerator: 'CommandOrControl+N',
          click: () => {
            win.webContents.send('menu:new')
          }
        },
        {
          label: 'Open',
          accelerator: 'CommandOrControl+O',
          click: () => {
            win.webContents.send('menu:open')
          }
        },
        {
          label: 'Save',
          accelerator: 'CommandOrControl+S',
          click: () => {
            win.webContents.send('menu:save')
          }
        },
        {
          label: 'Save As',
          accelerator: 'CommandOrControl+Shift+S',
          click: () => {
            win.webContents.send('menu:save-as')
          }
        },
        { type: 'separator' },
        {
          label: 'Quit',
          accelerator: 'CommandOrControl+Q',
          click: () => app.quit()
        }
      ]
    },
    {
      label: 'Edit',
      submenu: [
        { role: 'undo', accelerator: 'CommandOrControl+Z' },
        { role: 'redo', accelerator: 'CommandOrControl+Shift+Z' },
        { type: 'separator' },
        { role: 'cut', accelerator: 'CommandOrControl+X' },
        { role: 'copy', accelerator: 'CommandOrControl+C' },
        { role: 'paste', accelerator: 'CommandOrControl+V' },
        { role: 'selectAll', accelerator: 'CommandOrControl+A' }
      ]
    },
    {
      label: 'View',
      submenu: [
        { role: 'reload', accelerator: 'CommandOrControl+R' },
        { role: 'forceReload', accelerator: 'CommandOrControl+Shift+R' },
        { role: 'toggleDevTools', accelerator: 'CommandOrControl+Shift+I' },
        { type: 'separator' },
        { role: 'zoomIn', accelerator: 'CommandOrControl+=' },
        { role: 'zoomOut', accelerator: 'CommandOrControl+-' },
        { role: 'resetZoom', accelerator: 'CommandOrControl+0' },
        { type: 'separator' },
        { role: 'togglefullscreen', accelerator: 'F11' }
      ]
    }
  ])
  
  Menu.setApplicationMenu(menu)
  win.loadFile('index.html')
})
```

### วิธีที่ 2: ใน Renderer process

```javascript
// renderer.js - keyboard shortcuts ใน renderer
document.addEventListener('keydown', handleKeyDown)

function handleKeyDown(e) {
  const isMac = navigator.platform.includes('Mac')
  const mod = isMac ? e.metaKey : e.ctrlKey
  
  if (mod && !e.altKey) {
    switch (e.key) {
      case 'n':
        e.preventDefault()
        newDocument()
        break
      case 'o':
        e.preventDefault()
        openDocument()
        break
      case 's':
        e.preventDefault()
        if (e.shiftKey) {
          saveDocumentAs()
        } else {
          saveDocument()
        }
        break
      case 'z':
        e.preventDefault()
        if (e.shiftKey) {
          redo()
        } else {
          undo()
        }
        break
      case 'f':
        e.preventDefault()
        toggleSearch()
        break
    }
  }
  
  // Function keys
  switch (e.key) {
    case 'F1':
      e.preventDefault()
      showHelp()
      break
    case 'F11':
      e.preventDefault()
      toggleFullscreen()
      break
    case 'Escape':
      closeModal()
      break
  }
}

function newDocument() { console.log('New document') }
function openDocument() { console.log('Open document') }
function saveDocument() { console.log('Save document') }
function saveDocumentAs() { console.log('Save document as') }
function undo() { console.log('Undo') }
function redo() { console.log('Redo') }
function toggleSearch() { console.log('Toggle search') }
function showHelp() { console.log('Show help') }
function toggleFullscreen() { console.log('Toggle fullscreen') }
function closeModal() {
  const modal = document.querySelector('.modal.open')
  if (modal) modal.classList.remove('open')
}
```

### วิธีที่ 3: before-input-event บน webContents

```javascript
// main.js - intercept keyboard input before renderer
const { BrowserWindow } = require('electron')

const win = new BrowserWindow(/* options */)

win.webContents.on('before-input-event', (event, input) => {
  // intercept Ctrl+Shift+D
  if (input.type === 'keyDown' && 
      input.control && 
      input.shift && 
      input.key === 'D') {
    event.preventDefault()
    console.log('Debug shortcut intercepted in main!')
    win.webContents.openDevTools()
  }
  
  // ป้องกัน F5 refresh ใน production
  if (input.type === 'keyDown' && input.key === 'F5') {
    if (!process.env.NODE_ENV === 'development') {
      event.preventDefault()
    }
  }
})
```

## Shortcut Conflicts

### ตรวจสอบและจัดการ Conflicts

```javascript
// main.js - Shortcut Conflict Manager
const { globalShortcut } = require('electron')

class ShortcutManager {
  constructor() {
    this.shortcuts = new Map()
    this.conflictLog = []
  }
  
  register(accelerator, handler, metadata = {}) {
    // ตรวจสอบว่า shortcut นี้ถูกใช้งานอยู่แล้วหรือไม่
    if (globalShortcut.isRegistered(accelerator)) {
      const conflict = {
        accelerator,
        existing: this.shortcuts.get(accelerator),
        attempting: metadata,
        timestamp: new Date().toISOString()
      }
      this.conflictLog.push(conflict)
      console.warn(`Shortcut conflict: ${accelerator}`, conflict)
      return { success: false, conflict: true, reason: 'Already registered' }
    }
    
    const success = globalShortcut.register(accelerator, handler)
    
    if (success) {
      this.shortcuts.set(accelerator, {
        handler,
        metadata,
        registeredAt: new Date().toISOString()
      })
      return { success: true }
    }
    
    return { 
      success: false, 
      conflict: true, 
      reason: 'System shortcut conflict' 
    }
  }
  
  unregister(accelerator) {
    if (!this.shortcuts.has(accelerator)) {
      return false
    }
    
    globalShortcut.unregister(accelerator)
    this.shortcuts.delete(accelerator)
    return true
  }
  
  getConflicts() {
    return this.conflictLog
  }
  
  getRegistered() {
    return Array.from(this.shortcuts.entries()).map(([acc, info]) => ({
      accelerator: acc,
      isActive: globalShortcut.isRegistered(acc),
      ...info.metadata
    }))
  }
  
  // ลองทางเลือกหาก shortcut ขัดแย้ง
  registerWithFallbacks(accelerators, handler, metadata = {}) {
    for (const accelerator of accelerators) {
      const result = this.register(accelerator, handler, {
        ...metadata,
        accelerator
      })
      
      if (result.success) {
        return { success: true, accelerator }
      }
    }
    
    return { 
      success: false, 
      error: `All shortcuts are taken: ${accelerators.join(', ')}` 
    }
  }
  
  cleanup() {
    globalShortcut.unregisterAll()
    this.shortcuts.clear()
  }
}

module.exports = ShortcutManager
```

### ใช้งาน ShortcutManager

```javascript
// main.js
const { app, BrowserWindow, ipcMain } = require('electron')
const ShortcutManager = require('./shortcut-manager')
const path = require('path')

let mainWindow
const shortcutManager = new ShortcutManager()

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 900,
    height: 700,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  mainWindow.loadFile('index.html')
  
  // ลงทะเบียน shortcuts หลัก
  setupShortcuts()
}

function setupShortcuts() {
  // Screenshot
  const screenshotResult = shortcutManager.registerWithFallbacks(
    ['CommandOrControl+Shift+4', 'CommandOrControl+Shift+5', 'CommandOrControl+PrintScreen'],
    () => {
      mainWindow?.webContents.send('shortcut:screenshot')
    },
    { name: 'Screenshot', category: 'capture' }
  )
  console.log('Screenshot shortcut:', screenshotResult)
  
  // Toggle window visibility
  shortcutManager.register('CommandOrControl+Shift+H', () => {
    if (mainWindow) {
      if (mainWindow.isVisible()) {
        mainWindow.hide()
      } else {
        mainWindow.show()
        mainWindow.focus()
      }
    }
  }, { name: 'Toggle Window', category: 'window' })
  
  // Quick search
  shortcutManager.register('CommandOrControl+Shift+F', () => {
    mainWindow?.webContents.send('shortcut:quick-search')
  }, { name: 'Quick Search', category: 'search' })
}

// IPC
ipcMain.handle('shortcuts:get-registered', () => {
  return shortcutManager.getRegistered()
})

ipcMain.handle('shortcuts:get-conflicts', () => {
  return shortcutManager.getConflicts()
})

ipcMain.handle('shortcuts:register', (event, { accelerator, name }) => {
  const result = shortcutManager.register(
    accelerator,
    () => mainWindow?.webContents.send('shortcut:custom', { accelerator, name }),
    { name, custom: true }
  )
  return result
})

ipcMain.handle('shortcuts:unregister', (event, accelerator) => {
  const success = shortcutManager.unregister(accelerator)
  return { success }
})

app.whenReady().then(createWindow)
app.on('will-quit', () => shortcutManager.cleanup())
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

## Complete Shortcut Manager App

### preload.js

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('shortcutsAPI', {
  getRegistered: () => ipcRenderer.invoke('shortcuts:get-registered'),
  getConflicts: () => ipcRenderer.invoke('shortcuts:get-conflicts'),
  register: (accelerator, name) => ipcRenderer.invoke('shortcuts:register', { accelerator, name }),
  unregister: (accelerator) => ipcRenderer.invoke('shortcuts:unregister', accelerator),
  
  onShortcut: (callback) => {
    const events = ['shortcut:screenshot', 'shortcut:quick-search', 'shortcut:custom']
    const fns = events.map(event => {
      const fn = (_e, data) => callback({ event, data })
      ipcRenderer.on(event, fn)
      return { event, fn }
    })
    
    return () => fns.forEach(({ event, fn }) => ipcRenderer.removeListener(event, fn))
  }
})
```

### index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="Content-Security-Policy" content="default-src 'self'; style-src 'self' 'unsafe-inline'; script-src 'self';">
  <title>Keyboard Shortcuts Manager</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui; background: #f0f2f5; padding: 20px; color: #333; }
    h1 { margin-bottom: 20px; color: #1a1a2e; }
    .card { background: white; border-radius: 10px; padding: 20px; margin-bottom: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.08); }
    h2 { font-size: 15px; color: #666; margin-bottom: 14px; text-transform: uppercase; letter-spacing: 0.5px; }
    .shortcut-item {
      display: flex; align-items: center; gap: 12px;
      padding: 10px 0; border-bottom: 1px solid #f0f0f0;
    }
    .shortcut-item:last-child { border-bottom: none; }
    .key-badge {
      background: #1a1a2e; color: #fff; padding: 4px 10px;
      border-radius: 4px; font-family: monospace; font-size: 13px;
      white-space: nowrap;
    }
    .shortcut-name { flex: 1; font-size: 14px; }
    .shortcut-status {
      width: 8px; height: 8px; border-radius: 50%;
      background: #34c759;
    }
    .shortcut-status.inactive { background: #ff3b30; }
    .form-row { display: flex; gap: 10px; margin-bottom: 10px; align-items: center; }
    input[type="text"] {
      flex: 1; padding: 8px 12px; border: 1px solid #ddd; border-radius: 6px; font-size: 14px;
    }
    input[type="text"]:focus { outline: none; border-color: #007aff; box-shadow: 0 0 0 2px rgba(0,122,255,0.2); }
    .btn { padding: 8px 18px; border: none; border-radius: 6px; cursor: pointer; font-size: 14px; }
    .btn-primary { background: #007aff; color: white; }
    .btn-danger { background: #ff3b30; color: white; }
    .btn-secondary { background: #e5e5ea; color: #333; }
    .btn:hover { opacity: 0.85; }
    kbd {
      background: #f0f0f0; border: 1px solid #ccc; border-radius: 3px;
      padding: 2px 6px; font-family: monospace; font-size: 12px;
    }
    #event-log {
      background: #1c1c1e; color: #30d158; padding: 12px; border-radius: 6px;
      font-family: monospace; font-size: 12px; min-height: 80px; max-height: 150px;
      overflow-y: auto;
    }
    .hint { font-size: 12px; color: #999; margin-top: 6px; }
    .conflict-item { 
      background: #fff3cd; border-radius: 4px; padding: 8px; 
      margin-bottom: 6px; font-size: 13px; 
    }
  </style>
</head>
<body>
  <h1>Keyboard Shortcuts Manager</h1>
  
  <!-- Active Shortcuts -->
  <div class="card">
    <h2>Shortcuts ที่ลงทะเบียนแล้ว</h2>
    <div id="shortcuts-list"></div>
    <button class="btn btn-secondary" id="btn-refresh" style="margin-top:12px;">รีเฟรช</button>
  </div>
  
  <!-- Register New Shortcut -->
  <div class="card">
    <h2>เพิ่ม Custom Shortcut</h2>
    <div class="form-row">
      <input type="text" id="shortcut-key" placeholder="เช่น CommandOrControl+Shift+X" />
      <input type="text" id="shortcut-name" placeholder="ชื่อ shortcut" />
      <button class="btn btn-primary" id="btn-register">เพิ่ม</button>
    </div>
    <p class="hint">
      Keys: <kbd>CommandOrControl</kbd> <kbd>Alt</kbd> <kbd>Shift</kbd> <kbd>F1-F12</kbd> <kbd>A-Z</kbd> <kbd>0-9</kbd> <kbd>Space</kbd>
    </p>
  </div>
  
  <!-- Local Shortcuts Demo -->
  <div class="card">
    <h2>Local Shortcuts (ใช้งานในหน้านี้)</h2>
    <table style="width:100%; font-size:14px; border-collapse:collapse;">
      <tr>
        <td style="padding:6px 0;"><kbd>Ctrl+K</kbd> / <kbd>Cmd+K</kbd></td>
        <td style="padding:6px 0; color:#666;">เปิด Quick Search</td>
      </tr>
      <tr>
        <td style="padding:6px 0;"><kbd>Ctrl+/</kbd></td>
        <td style="padding:6px 0; color:#666;">แสดง Shortcuts</td>
      </tr>
      <tr>
        <td style="padding:6px 0;"><kbd>Escape</kbd></td>
        <td style="padding:6px 0; color:#666;">ปิด Modal</td>
      </tr>
      <tr>
        <td style="padding:6px 0;"><kbd>Ctrl+Shift+D</kbd></td>
        <td style="padding:6px 0; color:#666;">เปิด DevTools</td>
      </tr>
    </table>
  </div>
  
  <!-- Conflicts -->
  <div class="card" id="conflicts-card" style="display:none;">
    <h2>Shortcut Conflicts</h2>
    <div id="conflicts-list"></div>
  </div>
  
  <!-- Event Log -->
  <div class="card">
    <h2>Event Log</h2>
    <div id="event-log">รอการกด shortcut...</div>
  </div>
  
  <!-- Quick Search Modal -->
  <div id="search-modal" style="display:none; position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.5); z-index:1000; display:none; justify-content:center; align-items:flex-start; padding-top:15vh;">
    <div style="background:white; border-radius:12px; padding:20px; width:500px; max-width:90%;">
      <h3 style="margin-bottom:10px;">Quick Search</h3>
      <input type="text" id="search-input" placeholder="ค้นหา..." style="width:100%; padding:10px; border:1px solid #ddd; border-radius:6px; font-size:16px;">
      <p style="margin-top:8px; font-size:12px; color:#999;">กด <kbd>Escape</kbd> เพื่อปิด</p>
    </div>
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

### renderer.js

```javascript
// renderer.js
'use strict'

const log = document.getElementById('event-log')
const searchModal = document.getElementById('search-modal')

function addLog(message) {
  const time = new Date().toLocaleTimeString('th-TH')
  log.textContent = `[${time}] ${message}\n` + log.textContent
  if (log.textContent.length > 2000) {
    log.textContent = log.textContent.slice(0, 2000)
  }
}

// ─── Render Shortcuts List ────────────────────────────────────────
async function loadShortcuts() {
  const shortcuts = await window.shortcutsAPI.getRegistered()
  const list = document.getElementById('shortcuts-list')
  
  if (shortcuts.length === 0) {
    list.innerHTML = '<p style="color:#999; font-size:14px;">ยังไม่มี shortcuts</p>'
    return
  }
  
  list.innerHTML = shortcuts.map(s => `
    <div class="shortcut-item">
      <div class="shortcut-status ${s.isActive ? '' : 'inactive'}"></div>
      <span class="key-badge">${s.accelerator}</span>
      <span class="shortcut-name">${s.name || 'Unknown'}</span>
      ${s.custom ? `<button class="btn btn-danger" style="padding:4px 10px; font-size:12px;" 
                    onclick="removeShortcut('${s.accelerator}')">ลบ</button>` : ''}
    </div>
  `).join('')
}

window.removeShortcut = async (accelerator) => {
  const result = await window.shortcutsAPI.unregister(accelerator)
  if (result.success) {
    addLog(`ลบ shortcut: ${accelerator}`)
    loadShortcuts()
  }
}

document.getElementById('btn-refresh').addEventListener('click', loadShortcuts)

// ─── Register Custom Shortcut ────────────────────────────────────
document.getElementById('btn-register').addEventListener('click', async () => {
  const accelerator = document.getElementById('shortcut-key').value.trim()
  const name = document.getElementById('shortcut-name').value.trim() || accelerator
  
  if (!accelerator) { addLog('กรุณาใส่ shortcut key'); return }
  
  const result = await window.shortcutsAPI.register(accelerator, name)
  
  if (result.success) {
    addLog(`ลงทะเบียน shortcut: ${accelerator} -> ${name}`)
    document.getElementById('shortcut-key').value = ''
    document.getElementById('shortcut-name').value = ''
    loadShortcuts()
  } else {
    addLog(`ไม่สามารถลงทะเบียน ${accelerator}: ${result.conflict ? 'ขัดแย้งกับ shortcut อื่น' : result.reason}`)
  }
})

// ─── Listen for Global Shortcut Events ──────────────────────────
const unsubShortcuts = window.shortcutsAPI.onShortcut(({ event, data }) => {
  if (event === 'shortcut:quick-search') {
    openSearch()
  } else if (event === 'shortcut:screenshot') {
    addLog('Screenshot shortcut triggered!')
  } else if (event === 'shortcut:custom') {
    addLog(`Custom shortcut triggered: ${data?.accelerator} (${data?.name})`)
  }
})

// ─── Local Keyboard Shortcuts ────────────────────────────────────
document.addEventListener('keydown', (e) => {
  const mod = e.ctrlKey || e.metaKey
  
  // Quick search: Ctrl/Cmd+K
  if (mod && e.key === 'k') {
    e.preventDefault()
    openSearch()
    addLog('Local shortcut: Quick Search')
    return
  }
  
  // Show shortcuts help: Ctrl+/
  if (mod && e.key === '/') {
    e.preventDefault()
    addLog('Local shortcut: แสดง shortcuts list')
    document.querySelector('.card').scrollIntoView({ behavior: 'smooth' })
    return
  }
  
  // DevTools: Ctrl+Shift+D
  if (mod && e.shiftKey && e.key === 'D') {
    addLog('Local shortcut: Toggle DevTools (จัดการโดย main process)')
  }
  
  // Escape
  if (e.key === 'Escape') {
    closeSearch()
  }
})

// ─── Quick Search Modal ──────────────────────────────────────────
function openSearch() {
  searchModal.style.display = 'flex'
  setTimeout(() => document.getElementById('search-input').focus(), 50)
}

function closeSearch() {
  searchModal.style.display = 'none'
  document.getElementById('search-input').value = ''
}

searchModal.addEventListener('click', (e) => {
  if (e.target === searchModal) closeSearch()
})

// ─── Load Conflicts ──────────────────────────────────────────────
async function loadConflicts() {
  const conflicts = await window.shortcutsAPI.getConflicts()
  
  if (conflicts.length > 0) {
    const card = document.getElementById('conflicts-card')
    const list = document.getElementById('conflicts-list')
    card.style.display = 'block'
    list.innerHTML = conflicts.map(c => `
      <div class="conflict-item">
        <strong>${c.accelerator}</strong>: ${c.attempting?.name || 'Unknown'} ขัดแย้งกับ ${c.existing?.metadata?.name || 'shortcut อื่น'}
      </div>
    `).join('')
  }
}

// Cleanup
window.addEventListener('beforeunload', () => unsubShortcuts())

// Init
loadShortcuts()
loadConflicts()
addLog('Keyboard Shortcuts Manager พร้อมใช้งาน')
```

## สรุป

Keyboard Shortcuts ใน Electron มีหลายระดับ:

1. **Global Shortcuts** (`globalShortcut`) - ทำงานทุกที่แม้ไม่ได้ focus app
   - ต้อง unregister เสมอใน `will-quit`
   - ตรวจสอบ conflicts ก่อนลงทะเบียน
   - ใช้ `CommandOrControl` แทน `Ctrl`/`Cmd` เพื่อ cross-platform

2. **Menu Accelerators** - ทำงานเมื่อ window มี focus
   - สะดวกที่สุด เพราะ Electron จัดการ conflict อัตโนมัติ
   - รองรับ `role` สำหรับ standard shortcuts

3. **Local (Renderer) Shortcuts** - จัดการใน JavaScript
   - ยืดหยุ่นมากที่สุด
   - ต้อง prevent default เพื่อหลีกเลี่ยง browser behavior

4. **before-input-event** - Intercept ที่ main process level
   - ใช้สำหรับ security หรือ override browser shortcuts

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Power Monitor & System Events
