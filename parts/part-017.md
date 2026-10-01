# ตอนที่ 17: Screen API

## Screen Module ใน Electron

`screen` module ให้ข้อมูลเกี่ยวกับ displays ที่เชื่อมต่ออยู่ รวมถึง resolution, scale factor, และตำแหน่งของ cursor

```javascript
const { screen } = require('electron')
```

**หมายเหตุ:** `screen` module ใช้ได้หลังจาก app `ready` event เท่านั้น

## screen.getPrimaryDisplay

ดึงข้อมูลของ primary display (จอหลัก)

```javascript
// main.js
const { app, screen } = require('electron')

app.whenReady().then(() => {
  const primaryDisplay = screen.getPrimaryDisplay()
  
  console.log('Primary Display:')
  console.log('  ID:', primaryDisplay.id)
  console.log('  Bounds:', primaryDisplay.bounds)           // { x, y, width, height }
  console.log('  Work Area:', primaryDisplay.workArea)      // ส่วนที่ไม่มี taskbar
  console.log('  Scale Factor:', primaryDisplay.scaleFactor)
  console.log('  Rotation:', primaryDisplay.rotation)       // 0, 90, 180, 270
  console.log('  Size:', primaryDisplay.size)               // { width, height }
  console.log('  Work Area Size:', primaryDisplay.workAreaSize)
  console.log('  Color Depth:', primaryDisplay.colorDepth)
  console.log('  Color Space:', primaryDisplay.colorSpace)
  console.log('  Depth Per Component:', primaryDisplay.depthPerComponent)
  console.log('  Display Frequency:', primaryDisplay.displayFrequency)
  console.log('  Is Internal:', primaryDisplay.internal)    // true ถ้าเป็น built-in display
  console.log('  Touch Support:', primaryDisplay.touchSupport) // 'available', 'unavailable', 'unknown'
  console.log('  Accelerometer Support:', primaryDisplay.accelerometerSupport)
  console.log('  Monochrome:', primaryDisplay.monochrome)
})
```

### Display Object Properties

```javascript
// Display object ตัวอย่าง:
const displayExample = {
  id: 2779098949,              // unique identifier
  label: 'Built-in Display',
  bounds: {
    x: 0,          // ตำแหน่ง x บน virtual desktop
    y: 0,          // ตำแหน่ง y บน virtual desktop
    width: 2560,   // ความกว้างรวม HiDPI pixels
    height: 1440
  },
  workArea: {
    x: 0,
    y: 23,         // เว้นพื้นที่ menu bar บน macOS
    width: 2560,
    height: 1417
  },
  scaleFactor: 2,   // HiDPI: 2x = Retina
  rotation: 0,      // องศา
  size: {
    width: 1280,    // ความกว้างในหน่วย logical pixels
    height: 720
  },
  workAreaSize: {
    width: 1280,
    height: 708
  },
  displayFrequency: 60,
  internal: true,
  touchSupport: 'unknown',
  colorDepth: 24,
  colorSpace: 'sRGB',
  monochrome: false
}
```

## screen.getAllDisplays

```javascript
// main.js
const { app, screen } = require('electron')

app.whenReady().then(() => {
  const displays = screen.getAllDisplays()
  
  console.log(`พบ ${displays.length} displays:`)
  
  displays.forEach((display, index) => {
    console.log(`\nDisplay ${index + 1}:`)
    console.log('  ID:', display.id)
    console.log('  Label:', display.label || 'Unknown')
    console.log('  Bounds:', display.bounds)
    console.log('  Scale Factor:', display.scaleFactor)
    console.log('  Primary:', display === screen.getPrimaryDisplay())
    console.log('  Internal:', display.internal)
    
    // คำนวณ actual pixels
    const actualWidth = display.bounds.width * display.scaleFactor
    const actualHeight = display.bounds.height * display.scaleFactor
    console.log(`  Actual Resolution: ${actualWidth} × ${actualHeight}`)
  })
})
```

### Display ที่ดีที่สุดสำหรับ Window

```javascript
// main.js
const { screen, BrowserWindow } = require('electron')

function getDisplayForWindow(win) {
  const bounds = win.getBounds()
  const centerX = bounds.x + bounds.width / 2
  const centerY = bounds.y + bounds.height / 2
  
  return screen.getDisplayNearestPoint({ x: centerX, y: centerY })
}

function getDisplayForPoint(x, y) {
  return screen.getDisplayNearestPoint({ x, y })
}

// สร้าง window ที่ center ของ display ที่ cursor อยู่
function createWindowAtCursor() {
  const cursorPos = screen.getCursorScreenPoint()
  const display = screen.getDisplayNearestPoint(cursorPos)
  const { workArea } = display
  
  const width = 800
  const height = 600
  const x = workArea.x + Math.floor((workArea.width - width) / 2)
  const y = workArea.y + Math.floor((workArea.height - height) / 2)
  
  const win = new BrowserWindow({
    x, y, width, height,
    webPreferences: {
      preload: __dirname + '/preload.js',
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  return win
}
```

## Display Bounds

### การทำงานกับ Multi-Monitor Setups

```javascript
// main.js
const { screen, BrowserWindow } = require('electron')

// ดู virtual desktop layout
function getVirtualDesktopBounds() {
  const displays = screen.getAllDisplays()
  
  let minX = Infinity, minY = Infinity
  let maxX = -Infinity, maxY = -Infinity
  
  displays.forEach(display => {
    const { x, y, width, height } = display.bounds
    minX = Math.min(minX, x)
    minY = Math.min(minY, y)
    maxX = Math.max(maxX, x + width)
    maxY = Math.max(maxY, y + height)
  })
  
  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY
  }
}

// ตรวจสอบว่า window อยู่ใน visible area
function isWindowVisible(win) {
  const bounds = win.getBounds()
  const displays = screen.getAllDisplays()
  
  return displays.some(display => {
    const d = display.workArea
    // ตรวจสอบว่า window มี overlap กับ display อย่างน้อย 50%
    const overlapX = Math.max(0, Math.min(bounds.x + bounds.width, d.x + d.width) - Math.max(bounds.x, d.x))
    const overlapY = Math.max(0, Math.min(bounds.y + bounds.height, d.y + d.height) - Math.max(bounds.y, d.y))
    const overlapArea = overlapX * overlapY
    const windowArea = bounds.width * bounds.height
    
    return overlapArea > windowArea * 0.5
  })
}

// ย้าย window กลับมาใน screen ถ้าอยู่นอก
function bringWindowIntoView(win) {
  if (isWindowVisible(win)) return
  
  const display = screen.getPrimaryDisplay()
  const { workArea } = display
  const bounds = win.getBounds()
  
  const newX = Math.max(workArea.x, Math.min(bounds.x, workArea.x + workArea.width - bounds.width))
  const newY = Math.max(workArea.y, Math.min(bounds.y, workArea.y + workArea.height - bounds.height))
  
  win.setPosition(newX, newY)
}

// วาง windows บน displays ต่างๆ
function arrangeWindowsAcrossDisplays(windows) {
  const displays = screen.getAllDisplays()
  
  windows.forEach((win, i) => {
    const display = displays[i % displays.length]
    const { workArea } = display
    
    const width = Math.floor(workArea.width * 0.8)
    const height = Math.floor(workArea.height * 0.8)
    const x = workArea.x + Math.floor((workArea.width - width) / 2)
    const y = workArea.y + Math.floor((workArea.height - height) / 2)
    
    win.setBounds({ x, y, width, height })
  })
}
```

## Cursor Position

```javascript
// main.js
const { screen, ipcMain } = require('electron')

// ดูตำแหน่ง cursor ปัจจุบัน
ipcMain.handle('screen:get-cursor-position', () => {
  return screen.getCursorScreenPoint()
})

// ดู display ที่ cursor อยู่
ipcMain.handle('screen:get-display-at-cursor', () => {
  const cursor = screen.getCursorScreenPoint()
  const display = screen.getDisplayNearestPoint(cursor)
  return { cursor, display }
})

// ตรวจสอบ cursor position แบบ periodic
let cursorMonitorTimer = null

ipcMain.handle('screen:start-cursor-monitor', (event, interval = 100) => {
  const win = require('electron').BrowserWindow.fromWebContents(event.sender)
  
  if (cursorMonitorTimer) clearInterval(cursorMonitorTimer)
  
  cursorMonitorTimer = setInterval(() => {
    const pos = screen.getCursorScreenPoint()
    win?.webContents.send('screen:cursor-moved', pos)
  }, interval)
  
  return { success: true }
})

ipcMain.handle('screen:stop-cursor-monitor', () => {
  if (cursorMonitorTimer) {
    clearInterval(cursorMonitorTimer)
    cursorMonitorTimer = null
  }
  return { success: true }
})
```

## Multi-Monitor Support

### Window Management สำหรับ Multi-Monitor

```javascript
// main.js
const { app, BrowserWindow, screen, ipcMain } = require('electron')
const path = require('path')

let windows = new Map()

function createWindowOnDisplay(displayId, options = {}) {
  const displays = screen.getAllDisplays()
  const display = displays.find(d => d.id === displayId) || screen.getPrimaryDisplay()
  const { workArea } = display
  
  const defaultWidth = 800
  const defaultHeight = 600
  const width = options.width || defaultWidth
  const height = options.height || defaultHeight
  const x = options.x !== undefined ? options.x + workArea.x : workArea.x + Math.floor((workArea.width - width) / 2)
  const y = options.y !== undefined ? options.y + workArea.y : workArea.y + Math.floor((workArea.height - height) / 2)
  
  const win = new BrowserWindow({
    x, y, width, height,
    title: options.title || `Window on Display ${display.label || displayId}`,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  win.loadFile('index.html')
  windows.set(win.id, { win, displayId })
  
  win.on('closed', () => windows.delete(win.id))
  
  return win
}

// IPC handlers
ipcMain.handle('screen:get-all-displays', () => {
  const displays = screen.getAllDisplays()
  const primary = screen.getPrimaryDisplay()
  
  return displays.map(d => ({
    id: d.id,
    label: d.label || `Display ${d.id}`,
    bounds: d.bounds,
    workArea: d.workArea,
    scaleFactor: d.scaleFactor,
    rotation: d.rotation,
    size: d.size,
    workAreaSize: d.workAreaSize,
    displayFrequency: d.displayFrequency,
    internal: d.internal,
    isPrimary: d.id === primary.id
  }))
})

ipcMain.handle('screen:get-primary-display', () => {
  const d = screen.getPrimaryDisplay()
  const primary = screen.getPrimaryDisplay()
  return {
    id: d.id,
    label: d.label || `Display ${d.id}`,
    bounds: d.bounds,
    workArea: d.workArea,
    scaleFactor: d.scaleFactor,
    size: d.size,
    workAreaSize: d.workAreaSize,
    isPrimary: true
  }
})

ipcMain.handle('screen:create-window-on-display', (event, displayId) => {
  const win = createWindowOnDisplay(displayId)
  return { success: true, windowId: win.id }
})

ipcMain.handle('screen:move-window-to-display', (event, windowId, displayId) => {
  const winInfo = windows.get(windowId)
  if (!winInfo) return { success: false, error: 'Window not found' }
  
  const displays = screen.getAllDisplays()
  const display = displays.find(d => d.id === displayId)
  if (!display) return { success: false, error: 'Display not found' }
  
  const { workArea } = display
  const bounds = winInfo.win.getBounds()
  
  winInfo.win.setPosition(
    workArea.x + Math.floor((workArea.width - bounds.width) / 2),
    workArea.y + Math.floor((workArea.height - bounds.height) / 2)
  )
  
  windows.set(windowId, { ...winInfo, displayId })
  return { success: true }
})

ipcMain.handle('screen:get-cursor', () => screen.getCursorScreenPoint())

ipcMain.handle('screen:get-display-near-cursor', () => {
  const cursor = screen.getCursorScreenPoint()
  const d = screen.getDisplayNearestPoint(cursor)
  return { cursor, display: { id: d.id, bounds: d.bounds, workArea: d.workArea } }
})
```

## Display Events

```javascript
// main.js
const { app, screen, BrowserWindow } = require('electron')

app.whenReady().then(() => {
  const win = createWindow()
  
  // เมื่อเพิ่ม display ใหม่
  screen.on('display-added', (event, newDisplay) => {
    console.log('New display connected:', newDisplay.id)
    console.log('  Bounds:', newDisplay.bounds)
    win.webContents.send('screen:display-added', {
      id: newDisplay.id,
      bounds: newDisplay.bounds,
      workArea: newDisplay.workArea,
      scaleFactor: newDisplay.scaleFactor
    })
  })
  
  // เมื่อถอด display
  screen.on('display-removed', (event, removedDisplay) => {
    console.log('Display disconnected:', removedDisplay.id)
    win.webContents.send('screen:display-removed', {
      id: removedDisplay.id
    })
    
    // ย้าย windows ที่อยู่บน display ที่ถอดออก
    const allWindows = BrowserWindow.getAllWindows()
    allWindows.forEach(w => {
      const bounds = w.getBounds()
      const centerX = bounds.x + bounds.width / 2
      const centerY = bounds.y + bounds.height / 2
      
      const nearestDisplay = screen.getDisplayNearestPoint({ x: centerX, y: centerY })
      
      // ตรวจสอบว่า window อยู่ใน removed display area หรือไม่
      const rd = removedDisplay.bounds
      if (centerX >= rd.x && centerX <= rd.x + rd.width &&
          centerY >= rd.y && centerY <= rd.y + rd.height) {
        // ย้ายไปยัง primary display
        const primary = screen.getPrimaryDisplay()
        const { workArea } = primary
        w.setPosition(
          workArea.x + Math.floor((workArea.width - bounds.width) / 2),
          workArea.y + Math.floor((workArea.height - bounds.height) / 2)
        )
        console.log(`Moved window ${w.id} to primary display`)
      }
    })
  })
  
  // เมื่อ display เปลี่ยนแปลง (resolution, scale factor, etc.)
  screen.on('display-metrics-changed', (event, changedDisplay, changedMetrics) => {
    console.log('Display metrics changed:', changedDisplay.id)
    console.log('  Changed metrics:', changedMetrics) // array of changed properties
    
    win.webContents.send('screen:display-changed', {
      id: changedDisplay.id,
      bounds: changedDisplay.bounds,
      scaleFactor: changedDisplay.scaleFactor,
      changedMetrics
    })
  })
})

function createWindow() {
  const primary = screen.getPrimaryDisplay()
  const { workArea } = primary
  
  return new BrowserWindow({
    x: workArea.x + 100,
    y: workArea.y + 100,
    width: 900,
    height: 700,
    webPreferences: {
      preload: __dirname + '/preload.js',
      contextIsolation: true,
      nodeIntegration: false
    }
  })
}
```

## Complete Screen Manager Application

### preload.js

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('screenAPI', {
  getAllDisplays: () => ipcRenderer.invoke('screen:get-all-displays'),
  getPrimaryDisplay: () => ipcRenderer.invoke('screen:get-primary-display'),
  getCursor: () => ipcRenderer.invoke('screen:get-cursor'),
  getDisplayNearCursor: () => ipcRenderer.invoke('screen:get-display-near-cursor'),
  createWindowOnDisplay: (id) => ipcRenderer.invoke('screen:create-window-on-display', id),
  
  onDisplayAdded: (cb) => {
    const fn = (_e, d) => cb(d)
    ipcRenderer.on('screen:display-added', fn)
    return () => ipcRenderer.removeListener('screen:display-added', fn)
  },
  onDisplayRemoved: (cb) => {
    const fn = (_e, d) => cb(d)
    ipcRenderer.on('screen:display-removed', fn)
    return () => ipcRenderer.removeListener('screen:display-removed', fn)
  },
  onDisplayChanged: (cb) => {
    const fn = (_e, d) => cb(d)
    ipcRenderer.on('screen:display-changed', fn)
    return () => ipcRenderer.removeListener('screen:display-changed', fn)
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
  <title>Screen Manager</title>
  <style>
    * { box-sizing: border-box; }
    body { font-family: system-ui; background: #111827; color: #f3f4f6; padding: 20px; margin: 0; }
    h1 { color: #60a5fa; margin-bottom: 20px; }
    h2 { font-size: 13px; color: #9ca3af; text-transform: uppercase; letter-spacing: 0.5px; margin-bottom: 12px; }
    .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin-bottom: 16px; }
    .card { background: #1f2937; border-radius: 10px; padding: 16px; border: 1px solid #374151; }
    .display-card { 
      background: #1f2937; border-radius: 8px; padding: 12px; 
      margin-bottom: 10px; border: 2px solid #374151; transition: border-color 0.2s;
    }
    .display-card.primary { border-color: #3b82f6; }
    .display-card:hover { border-color: #6b7280; }
    .display-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px; }
    .display-name { font-weight: 600; font-size: 14px; color: #f3f4f6; }
    .badge { padding: 2px 8px; border-radius: 12px; font-size: 11px; }
    .badge-primary { background: #1d4ed8; color: #bfdbfe; }
    .badge-external { background: #065f46; color: #a7f3d0; }
    .display-stats { display: grid; grid-template-columns: repeat(3, 1fr); gap: 6px; margin-top: 8px; }
    .stat-box { background: #111827; padding: 6px 8px; border-radius: 4px; text-align: center; }
    .stat-box .label { font-size: 10px; color: #6b7280; }
    .stat-box .value { font-size: 13px; font-weight: 600; color: #60a5fa; }
    button { 
      padding: 7px 14px; border: none; border-radius: 6px; cursor: pointer; 
      font-size: 13px; background: #3b82f6; color: white;
    }
    button:hover { background: #2563eb; }
    .btn-small { padding: 4px 10px; font-size: 12px; }
    .monitor-canvas { 
      width: 100%; height: 200px; background: #111827; border-radius: 6px; 
      position: relative; overflow: hidden; border: 1px solid #374151; 
    }
    .cursor-dot { 
      width: 12px; height: 12px; background: #f59e0b; border-radius: 50%;
      position: absolute; transform: translate(-50%, -50%);
      box-shadow: 0 0 8px #f59e0b; pointer-events: none;
      transition: left 0.1s, top 0.1s;
    }
    .display-rect {
      position: absolute; border: 2px solid;
      display: flex; align-items: center; justify-content: center;
      font-size: 11px; color: white; font-weight: 600;
    }
    #event-log {
      height: 120px; overflow-y: auto; background: #111827; border-radius: 6px;
      padding: 8px; font-family: monospace; font-size: 12px; color: #6ee7b7;
    }
    .cursor-info { background: #111827; padding: 8px; border-radius: 6px; font-size: 13px; }
    .cursor-pos { color: #f59e0b; font-family: monospace; font-size: 14px; font-weight: 600; }
  </style>
</head>
<body>
  <h1>Screen Manager</h1>
  
  <div class="grid">
    <!-- Display List -->
    <div class="card">
      <h2>Displays ที่เชื่อมต่อ</h2>
      <div id="display-list">กำลังโหลด...</div>
      <button id="btn-refresh" style="margin-top:10px;" class="btn-small">รีเฟรช</button>
    </div>
    
    <!-- Virtual Desktop Map -->
    <div class="card">
      <h2>Virtual Desktop Map</h2>
      <div class="monitor-canvas" id="monitor-canvas">
        <div class="cursor-dot" id="cursor-dot"></div>
      </div>
      <div class="cursor-info" style="margin-top:8px;">
        Cursor: <span class="cursor-pos" id="cursor-pos">-, -</span>
      </div>
    </div>
  </div>
  
  <!-- Cursor Info -->
  <div class="card" style="margin-bottom:16px;">
    <h2>ข้อมูล Cursor</h2>
    <div style="display:flex; gap:16px; align-items:center; flex-wrap:wrap;">
      <div>
        <div style="font-size:12px;color:#9ca3af;">ตำแหน่ง</div>
        <div class="cursor-pos" id="cursor-full-pos">กำลังโหลด...</div>
      </div>
      <div>
        <div style="font-size:12px;color:#9ca3af;">Display ที่ cursor อยู่</div>
        <div id="cursor-display" style="font-size:14px;">กำลังโหลด...</div>
      </div>
      <button id="btn-check-cursor">ตรวจสอบตำแหน่ง</button>
    </div>
  </div>
  
  <!-- Events -->
  <div class="card">
    <h2>Display Events</h2>
    <div id="event-log">รอ events...</div>
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

### renderer.js

```javascript
// renderer.js
let allDisplays = []
let virtualBounds = { x: 0, y: 0, width: 0, height: 0 }

function addLog(message) {
  const log = document.getElementById('event-log')
  const time = new Date().toLocaleTimeString('th-TH')
  log.textContent = `[${time}] ${message}\n` + log.textContent
  if (log.textContent.length > 3000) log.textContent = log.textContent.slice(0, 3000)
}

// ─── Display List ─────────────────────────────────────────────────
function renderDisplayList(displays) {
  const container = document.getElementById('display-list')
  container.innerHTML = displays.map(d => `
    <div class="display-card ${d.isPrimary ? 'primary' : ''}">
      <div class="display-header">
        <span class="display-name">${d.label}</span>
        <span class="badge ${d.isPrimary ? 'badge-primary' : 'badge-external'}">
          ${d.isPrimary ? 'Primary' : d.internal ? 'Internal' : 'External'}
        </span>
      </div>
      <div class="display-stats">
        <div class="stat-box">
          <div class="label">ความละเอียด</div>
          <div class="value">${d.bounds.width}×${d.bounds.height}</div>
        </div>
        <div class="stat-box">
          <div class="label">Scale</div>
          <div class="value">${d.scaleFactor}×</div>
        </div>
        <div class="stat-box">
          <div class="label">Hz</div>
          <div class="value">${d.displayFrequency || '?'}</div>
        </div>
      </div>
      <div style="margin-top:6px; font-size:12px; color:#6b7280;">
        Position: (${d.bounds.x}, ${d.bounds.y}) | 
        Work Area: ${d.workArea.width}×${d.workArea.height}
      </div>
    </div>
  `).join('')
}

// ─── Virtual Desktop Map ─────────────────────────────────────────
function calculateVirtualBounds(displays) {
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity
  displays.forEach(d => {
    minX = Math.min(minX, d.bounds.x)
    minY = Math.min(minY, d.bounds.y)
    maxX = Math.max(maxX, d.bounds.x + d.bounds.width)
    maxY = Math.max(maxY, d.bounds.y + d.bounds.height)
  })
  return { x: minX, y: minY, width: maxX - minX, height: maxY - minY }
}

function renderVirtualDesktop(displays) {
  const canvas = document.getElementById('monitor-canvas')
  const canvasRect = canvas.getBoundingClientRect()
  const padding = 20
  
  virtualBounds = calculateVirtualBounds(displays)
  const scaleX = (canvasRect.width - padding * 2) / virtualBounds.width
  const scaleY = (canvasRect.height - padding * 2) / virtualBounds.height
  const scale = Math.min(scaleX, scaleY) * 0.9
  
  // ลบ display rects เก่า
  canvas.querySelectorAll('.display-rect').forEach(el => el.remove())
  
  const colors = ['#3b82f6', '#10b981', '#f59e0b', '#ef4444', '#8b5cf6']
  
  displays.forEach((d, i) => {
    const rect = document.createElement('div')
    rect.className = 'display-rect'
    
    const screenX = (d.bounds.x - virtualBounds.x) * scale + padding
    const screenY = (d.bounds.y - virtualBounds.y) * scale + padding
    const screenW = d.bounds.width * scale
    const screenH = d.bounds.height * scale
    
    rect.style.cssText = `
      left: ${screenX}px; top: ${screenY}px;
      width: ${screenW}px; height: ${screenH}px;
      border-color: ${colors[i % colors.length]};
      background: ${colors[i % colors.length]}22;
      font-size: ${Math.max(10, screenH * 0.12)}px;
    `
    rect.textContent = d.label.split(' ')[0] + (d.isPrimary ? ' ★' : '')
    canvas.appendChild(rect)
  })
  
  return { scale, padding }
}

function updateCursorOnCanvas(cursor, scale, padding) {
  const dot = document.getElementById('cursor-dot')
  if (!dot || !virtualBounds.width) return
  
  const x = (cursor.x - virtualBounds.x) * scale + padding
  const y = (cursor.y - virtualBounds.y) * scale + padding
  
  dot.style.left = x + 'px'
  dot.style.top = y + 'px'
}

let mapScaleInfo = { scale: 1, padding: 20 }

async function loadDisplays() {
  allDisplays = await window.screenAPI.getAllDisplays()
  renderDisplayList(allDisplays)
  mapScaleInfo = renderVirtualDesktop(allDisplays)
  addLog(`โหลด ${allDisplays.length} displays`)
}

// ─── Cursor Tracking ─────────────────────────────────────────────
async function checkCursor() {
  const { cursor, display } = await window.screenAPI.getDisplayNearCursor()
  
  document.getElementById('cursor-pos').textContent = `(${cursor.x}, ${cursor.y})`
  document.getElementById('cursor-full-pos').textContent = `X: ${cursor.x}, Y: ${cursor.y}`
  
  const displayInfo = allDisplays.find(d => d.id === display.id)
  document.getElementById('cursor-display').textContent = 
    displayInfo ? `${displayInfo.label} (ID: ${display.id})` : `Display ID: ${display.id}`
  
  updateCursorOnCanvas(cursor, mapScaleInfo.scale, mapScaleInfo.padding)
}

// อัปเดต cursor ทุก 200ms
setInterval(checkCursor, 200)

document.getElementById('btn-check-cursor').addEventListener('click', checkCursor)
document.getElementById('btn-refresh').addEventListener('click', loadDisplays)

// ─── Display Events ───────────────────────────────────────────────
const unsubs = [
  window.screenAPI.onDisplayAdded((d) => {
    addLog(`Display เพิ่มเข้ามา: ID ${d.id} (${d.bounds.width}×${d.bounds.height})`)
    loadDisplays()
  }),
  
  window.screenAPI.onDisplayRemoved((d) => {
    addLog(`Display ถูกถอดออก: ID ${d.id}`)
    loadDisplays()
  }),
  
  window.screenAPI.onDisplayChanged((d) => {
    addLog(`Display เปลี่ยนแปลง: ID ${d.id} (${d.changedMetrics?.join(', ')})`)
    loadDisplays()
  })
]

window.addEventListener('beforeunload', () => unsubs.forEach(fn => fn()))

// Init
loadDisplays()
checkCursor()
```

## สรุป

Screen API ของ Electron ให้ข้อมูลครบครันเกี่ยวกับ displays:

- **getPrimaryDisplay** - ดึงข้อมูล primary display
- **getAllDisplays** - รายการ displays ทั้งหมด
- **getDisplayNearestPoint** - หา display ที่ใกล้จุดที่กำหนด
- **getCursorScreenPoint** - ตำแหน่ง cursor ปัจจุบัน
- **display-added/removed/metrics-changed events** - ติดตามการเปลี่ยนแปลง

ใช้ประโยชน์สำหรับ:
- วาง windows ให้อยู่กึ่งกลางของ display ที่ถูกต้อง
- รองรับ multi-monitor setups
- ปรับ UI ตาม scale factor (HiDPI/Retina)
- ย้าย windows กลับมาใน visible area เมื่อ display ถูกถอด

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Protocol Handler
