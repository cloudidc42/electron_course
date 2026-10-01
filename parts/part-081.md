# Part 81: Electron & Chromium Extensions

## ใช้งาน Chrome Extensions ใน Electron App

ในบทนี้เราจะเรียนการ load Chrome extensions, สร้าง custom DevTools extensions และ manage extension lifecycle

---

## 1. Loading Chrome Extensions

```typescript
// src/main/extensions.ts
import { app, session, BrowserWindow } from 'electron'
import * as path from 'path'
import * as fs from 'fs'

// โหลด Chrome extension จาก directory
export async function loadExtension(extensionPath: string): Promise<Electron.Extension> {
  const ext = await session.defaultSession.loadExtension(extensionPath, {
    allowFileAccess: true  // สำหรับ file:// URLs
  })
  console.log(`Extension loaded: ${ext.name} v${ext.version}`)
  return ext
}

// โหลด React DevTools
export async function loadReactDevTools(): Promise<void> {
  if (process.env.NODE_ENV !== 'development') return

  // หา React DevTools ใน Chrome profile
  const chromeExtPath = getChromeExtensionPath('fmkadmapgofadopljbjfkapdkoienihi')

  if (chromeExtPath && fs.existsSync(chromeExtPath)) {
    await loadExtension(chromeExtPath)
    console.log('React DevTools loaded from Chrome profile')
    return
  }

  // หรือโหลดจาก electron-devtools-installer
  try {
    const { default: installExtension, REACT_DEVELOPER_TOOLS } = await import('electron-devtools-installer')
    await installExtension(REACT_DEVELOPER_TOOLS, {
      loadExtensionOptions: { allowFileAccess: true }
    })
    console.log('React DevTools installed')
  } catch (error) {
    console.error('Failed to install React DevTools:', error)
  }
}

function getChromeExtensionPath(extensionId: string): string | null {
  const platform = process.platform
  const homeDir = app.getPath('home')

  const chromePaths: Record<string, string[]> = {
    darwin: [
      `${homeDir}/Library/Application Support/Google/Chrome/Default/Extensions/${extensionId}`,
      `${homeDir}/Library/Application Support/Microsoft Edge/Default/Extensions/${extensionId}`
    ],
    win32: [
      `${homeDir}\\AppData\\Local\\Google\\Chrome\\User Data\\Default\\Extensions\\${extensionId}`,
    ],
    linux: [
      `${homeDir}/.config/google-chrome/Default/Extensions/${extensionId}`,
      `${homeDir}/.config/chromium/Default/Extensions/${extensionId}`
    ]
  }

  const paths = chromePaths[platform] || []
  for (const basePath of paths) {
    if (fs.existsSync(basePath)) {
      // หาเวอร์ชั่นล่าสุด
      const versions = fs.readdirSync(basePath).sort().reverse()
      if (versions.length > 0) {
        return path.join(basePath, versions[0])
      }
    }
  }
  return null
}

// ลบ extension
export async function removeExtension(extensionId: string): Promise<void> {
  await session.defaultSession.removeExtension(extensionId)
  console.log(`Extension removed: ${extensionId}`)
}

// รายการ extensions ที่ติดตั้ง
export function getInstalledExtensions(): Electron.Extension[] {
  return Object.values(session.defaultSession.getAllExtensions())
}
```

---

## 2. Custom DevTools Extension

```json
// devtools-extension/manifest.json
{
  "manifest_version": 3,
  "name": "My App DevTools",
  "version": "1.0.0",
  "description": "Custom DevTools for My Electron App",
  "devtools_page": "devtools.html",
  "icons": { "16": "icon16.png", "48": "icon48.png" }
}
```

```html
<!-- devtools-extension/devtools.html -->
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"></head>
<body>
<script>
  // สร้าง DevTools panel
  chrome.devtools.panels.create(
    'My App',
    'icon16.png',
    'panel.html',
    function(panel) {
      panel.onShown.addListener(function(window) {
        console.log('Panel shown')
      })
    }
  )
</script>
</body>
</html>
```

```html
<!-- devtools-extension/panel.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: monospace; font-size: 12px; background: #1e1e1e; color: #d4d4d4; padding: 8px; }
    .section { margin-bottom: 16px; }
    h3 { color: #4ec9b0; margin-bottom: 8px; font-size: 13px; }
    .item { padding: 4px 0; border-bottom: 1px solid #333; display: flex; justify-content: space-between; }
    .value { color: #ce9178; }
    button { background: #0078d4; color: white; border: none; padding: 4px 12px; cursor: pointer; border-radius: 2px; }
    #ipc-log { height: 200px; overflow-y: auto; background: #252526; padding: 8px; margin-top: 8px; }
    .ipc-entry { padding: 2px 0; }
    .ipc-entry.invoke { color: #569cd6; }
    .ipc-entry.send { color: #4ec9b0; }
    .ipc-entry.event { color: #c586c0; }
  </style>
</head>
<body>
  <div class="section">
    <h3>App Info</h3>
    <div id="app-info"></div>
  </div>
  <div class="section">
    <h3>IPC Monitor</h3>
    <button onclick="clearLog()">Clear</button>
    <div id="ipc-log"></div>
  </div>

  <script src="panel.js"></script>
</body>
</html>
```

```javascript
// devtools-extension/panel.js
const appInfoEl = document.getElementById('app-info')
const ipcLogEl = document.getElementById('ipc-log')

// ดึงข้อมูลจาก inspected window
function getAppInfo() {
  chrome.devtools.inspectedWindow.eval(
    `({
      electronVersion: process.versions.electron,
      nodeVersion: process.versions.node,
      platform: process.platform,
      pid: process.pid
    })`,
    (result, isException) => {
      if (isException || !result) return
      appInfoEl.innerHTML = Object.entries(result)
        .map(([k, v]) => `<div class="item"><span>${k}</span><span class="value">${v}</span></div>`)
        .join('')
    }
  )
}

function addIPCEntry(type, channel, data) {
  const entry = document.createElement('div')
  entry.className = `ipc-entry ${type}`
  entry.textContent = `[${type.toUpperCase()}] ${channel}: ${JSON.stringify(data).substring(0, 100)}`
  ipcLogEl.insertBefore(entry, ipcLogEl.firstChild)

  // เก็บ 100 entries ล่าสุด
  while (ipcLogEl.children.length > 100) {
    ipcLogEl.removeChild(ipcLogEl.lastChild)
  }
}

function clearLog() {
  ipcLogEl.innerHTML = ''
}

// Background script connection สำหรับ IPC monitoring
const connection = chrome.runtime.connect({ name: 'devtools' })
connection.onMessage.addListener((message) => {
  if (message.type === 'ipc-log') {
    addIPCEntry(message.kind, message.channel, message.data)
  }
})

getAppInfo()
setInterval(getAppInfo, 5000)
```

---

## 3. Extension Management UI

```tsx
// src/renderer/components/ExtensionManager.tsx
import { useState, useEffect } from 'react'

interface ExtensionInfo {
  id: string
  name: string
  version: string
  description: string
  enabled: boolean
  path: string
}

export function ExtensionManager() {
  const [extensions, setExtensions] = useState<ExtensionInfo[]>([])

  useEffect(() => {
    loadExtensions()
  }, [])

  const loadExtensions = async () => {
    const list = await window.electronAPI.invoke('extensions:list') as ExtensionInfo[]
    setExtensions(list)
  }

  const removeExtension = async (id: string) => {
    if (!confirm('ลบ extension นี้?')) return
    await window.electronAPI.invoke('extensions:remove', { id })
    setExtensions(prev => prev.filter(e => e.id !== id))
  }

  const loadFromPath = async () => {
    const dir = await window.electronAPI.invoke('pick-directory') as string | null
    if (!dir) return

    const ext = await window.electronAPI.invoke('extensions:load', { path: dir }) as ExtensionInfo | null
    if (ext) {
      setExtensions(prev => [...prev, ext])
    }
  }

  return (
    <div className="extension-manager">
      <div className="header">
        <h2>Extensions</h2>
        <button onClick={loadFromPath} className="btn-primary">
          โหลดจากโฟลเดอร์...
        </button>
      </div>

      <div className="extension-list">
        {extensions.length === 0 ? (
          <div className="empty">ยังไม่มี extensions</div>
        ) : (
          extensions.map(ext => (
            <div key={ext.id} className="extension-card">
              <div className="ext-info">
                <strong>{ext.name}</strong>
                <span className="version">v{ext.version}</span>
                <p className="description">{ext.description}</p>
                <code className="ext-id">{ext.id}</code>
              </div>
              <div className="ext-actions">
                <button
                  className="btn-danger"
                  onClick={() => removeExtension(ext.id)}
                >
                  ลบ
                </button>
              </div>
            </div>
          ))
        )}
      </div>
    </div>
  )
}
```

---

## 4. IPC Interceptor สำหรับ DevTools

```typescript
// src/main/ipcInterceptor.ts
// Intercept IPC เพื่อ logging และ debugging

import { ipcMain, BrowserWindow } from 'electron'

interface IPCLogEntry {
  timestamp: number
  type: 'invoke' | 'send' | 'event'
  channel: string
  args: unknown[]
  duration?: number
}

const log: IPCLogEntry[] = []
const MAX_LOG = 1000

export function enableIPCLogging(devToolsWin?: BrowserWindow): void {
  // Wrap ipcMain.handle เพื่อ log
  const originalHandle = ipcMain.handle.bind(ipcMain)
  ipcMain.handle = function(channel: string, handler: (...args: unknown[]) => unknown) {
    return originalHandle(channel, async (event, ...args) => {
      const start = performance.now()
      const entry: IPCLogEntry = {
        timestamp: Date.now(),
        type: 'invoke',
        channel,
        args
      }

      try {
        const result = await handler(event, ...args)
        entry.duration = performance.now() - start
        addLog(entry, devToolsWin)
        return result
      } catch (error) {
        entry.duration = performance.now() - start
        addLog({ ...entry, args: [...args, { error: (error as Error).message }] }, devToolsWin)
        throw error
      }
    }) as ReturnType<typeof originalHandle>
  }
}

function addLog(entry: IPCLogEntry, devToolsWin?: BrowserWindow): void {
  log.unshift(entry)
  if (log.length > MAX_LOG) log.pop()

  // ส่งไปยัง DevTools window
  devToolsWin?.webContents.send('ipc-log', entry)
}

export function getIPCLog(): IPCLogEntry[] {
  return [...log]
}
```

---

## สรุป

| ฟีเจอร์ | วิธีการ | ข้อควรระวัง |
|--------|---------|------------|
| Load extension | `session.loadExtension()` | ต้องระบุ path สมบูรณ์ |
| React DevTools | `electron-devtools-installer` | dev-only |
| Custom DevTools | manifest.json + devtools_page | MV3 format |
| DevTools Panel | `chrome.devtools.panels.create()` | iframe ใน DevTools |
| IPC Monitor | intercept + log | performance overhead |
| Extension removal | `session.removeExtension()` | ต้อง restart |
