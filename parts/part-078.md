# Part 78: Electron Fiddle & Quick Prototyping

## ทดสอบ Ideas เร็วด้วย Electron Fiddle และ Prototyping Patterns

ในบทนี้เราจะเรียนการใช้ Electron Fiddle, gist sharing, prototyping patterns และ minimal Electron boilerplate

---

## 1. Electron Fiddle Overview

Electron Fiddle คือ IDE ขนาดเล็กสำหรับทดสอบ Electron code โดยไม่ต้อง setup project

```
┌─────────────────────────────────────────────┐
│              Electron Fiddle UI              │
├──────────┬──────────┬──────────┬────────────┤
│ main.js  │ preload  │ renderer │ package.json│
├──────────┴──────────┴──────────┴────────────┤
│                  Code Editor                │
│                                             │
├─────────────────────────────────────────────┤
│  [▶ Run]  [Save to Gist]  [Share]  Version  │
└─────────────────────────────────────────────┘
```

**Features:**
- Switch Electron versions ทันที
- Save/Load จาก GitHub Gist
- Share code snippets
- Built-in templates

---

## 2. Minimal Electron Boilerplate

```typescript
// minimal/main.ts - ไม่มี framework
import { app, BrowserWindow } from 'electron'
import * as path from 'path'

let mainWindow: BrowserWindow | null = null

function createWindow(): void {
  mainWindow = new BrowserWindow({
    width: 900,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })

  mainWindow.loadFile('index.html')

  if (process.env.NODE_ENV === 'development') {
    mainWindow.webContents.openDevTools()
  }

  mainWindow.on('closed', () => { mainWindow = null })
}

app.whenReady().then(createWindow)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
app.on('activate', () => {
  if (mainWindow === null) createWindow()
})
```

```typescript
// minimal/preload.ts
import { contextBridge, ipcRenderer } from 'electron'

contextBridge.exposeInMainWorld('electronAPI', {
  send: (channel: string, data?: unknown) => ipcRenderer.send(channel, data),
  invoke: (channel: string, data?: unknown) => ipcRenderer.invoke(channel, data),
  on: (channel: string, cb: (data: unknown) => void) => {
    const listener = (_: Electron.IpcRendererEvent, data: unknown) => cb(data)
    ipcRenderer.on(channel, listener)
    return () => ipcRenderer.off(channel, listener)
  }
})
```

```html
<!-- minimal/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="Content-Security-Policy"
    content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'">
  <title>Minimal Electron App</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: system-ui, sans-serif;
      background: #1e1e1e;
      color: #d4d4d4;
      height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .container { text-align: center; }
    button {
      background: #0078d4;
      color: white;
      border: none;
      padding: 10px 24px;
      border-radius: 4px;
      cursor: pointer;
      font-size: 14px;
      margin-top: 16px;
    }
    button:hover { background: #106ebe; }
    #output {
      margin-top: 20px;
      font-family: monospace;
      color: #4ec9b0;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>สวัสดี Electron!</h1>
    <p>กด button เพื่อทดสอบ IPC</p>
    <button id="btn">เรียก Main Process</button>
    <div id="output"></div>
  </div>
  <script src="renderer.js"></script>
</body>
</html>
```

---

## 3. Quick Prototype Patterns

```typescript
// patterns/quickDialog.ts
// Pattern: รวม IPC + dialog ในฟังก์ชันเดียว

import { ipcMain, dialog } from 'electron'

export function setupQuickDialogs(): void {
  // Quick file picker
  ipcMain.handle('pick-file', async (_, options: Electron.OpenDialogOptions) => {
    const result = await dialog.showOpenDialog({ ...options })
    return result.canceled ? null : result.filePaths[0]
  })

  // Quick save dialog
  ipcMain.handle('save-dialog', async (_, options: Electron.SaveDialogOptions) => {
    const result = await dialog.showSaveDialog({ ...options })
    return result.canceled ? null : result.filePath
  })

  // Quick message box
  ipcMain.handle('message-box', async (_, options: Electron.MessageBoxOptions) => {
    const result = await dialog.showMessageBox({ ...options })
    return result.response
  })
}
```

```typescript
// patterns/quickTray.ts
// Pattern: Tray app ใน < 30 บรรทัด

import { app, Tray, Menu, nativeImage } from 'electron'
import * as path from 'path'

export function createTrayApp(iconPath: string): Tray {
  const icon = nativeImage.createFromPath(iconPath).resize({ width: 16 })
  const tray = new Tray(icon)

  const updateMenu = (status: string) => {
    const menu = Menu.buildFromTemplate([
      { label: `Status: ${status}`, enabled: false },
      { type: 'separator' },
      { label: 'Settings', click: () => { /* open settings */ } },
      { label: 'Quit', click: () => app.quit() }
    ])
    tray.setContextMenu(menu)
  }

  updateMenu('Running')
  tray.setToolTip('My App')

  return tray
}
```

---

## 4. Hot Reload สำหรับ Development

```typescript
// dev/hotReload.ts
// Simple hot reload โดยไม่ใช้ framework

import * as chokidar from 'chokidar'
import { BrowserWindow } from 'electron'
import * as path from 'path'

export function setupHotReload(win: BrowserWindow, watchDir: string): void {
  if (process.env.NODE_ENV !== 'development') return

  const watcher = chokidar.watch(watchDir, {
    ignored: /node_modules/,
    persistent: true
  })

  let reloadTimeout: NodeJS.Timeout | null = null

  watcher.on('change', (filePath) => {
    // Debounce reloads
    if (reloadTimeout) clearTimeout(reloadTimeout)
    reloadTimeout = setTimeout(() => {
      if (filePath.endsWith('.html') || filePath.endsWith('.css') || filePath.endsWith('.js')) {
        console.log(`File changed: ${path.basename(filePath)} - reloading...`)
        win.webContents.reload()
      }
    }, 300)
  })

  win.on('closed', () => watcher.close())
}
```

---

## 5. Fiddle Gist Format

```javascript
// ตัวอย่าง Fiddle: System Info Display
// main.js (Fiddle format)

const { app, BrowserWindow, ipcMain } = require('electron')
const os = require('os')

let win

app.whenReady().then(() => {
  win = new BrowserWindow({
    width: 700,
    height: 500,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: `${__dirname}/preload.js`
    }
  })

  win.loadFile('index.html')
})

ipcMain.handle('get-system-info', () => ({
  platform: process.platform,
  arch: process.arch,
  cpus: os.cpus().length,
  totalMem: (os.totalmem() / 1024 / 1024 / 1024).toFixed(1) + ' GB',
  hostname: os.hostname(),
  electronVersion: process.versions.electron,
  nodeVersion: process.versions.node,
  chromeVersion: process.versions.chrome
}))
```

```javascript
// preload.js (Fiddle format)
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('api', {
  getSystemInfo: () => ipcRenderer.invoke('get-system-info')
})
```

```html
<!-- index.html (Fiddle format) -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: monospace; background: #0d1117; color: #58a6ff; padding: 24px; }
    .info-row { display: flex; justify-content: space-between; padding: 6px 0; border-bottom: 1px solid #21262d; }
    .label { color: #8b949e; }
    .value { color: #3fb950; }
  </style>
</head>
<body>
  <h2>⚡ System Info</h2>
  <div id="info"></div>
  <script>
    window.api.getSystemInfo().then(info => {
      document.getElementById('info').innerHTML = Object.entries(info)
        .map(([k, v]) => `<div class="info-row"><span class="label">${k}</span><span class="value">${v}</span></div>`)
        .join('')
    })
  </script>
</body>
</html>
```

---

## 6. Rapid Prototyping Checklist

```markdown
## Checklist สำหรับ Quick Prototype

### Setup (< 5 นาที)
- [ ] `npm create electron-vite@latest` หรือ copy minimal boilerplate
- [ ] กำหนด window size, title
- [ ] Setup preload + contextBridge

### Feature เพิ่มใน Main
- [ ] ipcMain.handle สำหรับแต่ละ operation
- [ ] import OS/FS modules ที่ต้องการ
- [ ] Error handling ง่ายๆ ด้วย try-catch

### Feature เพิ่มใน Renderer
- [ ] HTML UI โดยตรง (ไม่ต้องใช้ React ใน prototype)
- [ ] window.electronAPI.invoke() สำหรับ operations
- [ ] CSS inline หรือ <style> block

### ทดสอบ
- [ ] `npm run dev` รัน
- [ ] เปิด DevTools ตรวจ errors
- [ ] ทดสอบ happy path
```

---

## สรุป

| เครื่องมือ | เหมาะกับ |
|----------|---------|
| Electron Fiddle | Experiment API ใหม่ |
| Minimal boilerplate | เริ่มต้นไว โดยไม่มี overhead |
| Hot reload | Rapid iteration |
| Tray/Dialog patterns | Common UI patterns |
| Gist sharing | แชร์ snippets กับทีม |
