# Part 21: React + Electron Integration

## บทนำ

การรวม React เข้ากับ Electron ทำให้เราสามารถสร้าง Desktop Application ที่มี UI ที่สวยงามและ Interactive ได้ง่ายขึ้น ในบทนี้เราจะเรียนรู้วิธีการตั้งค่าโปรเจกต์ React + Electron ด้วย Vite ซึ่งเป็น build tool ที่เร็วและทันสมัย รวมถึงการตั้งค่า hot reload ในระหว่าง development และการ packaging ด้วย electron-builder

## โครงสร้างโปรเจกต์

```
my-electron-react-app/
├── electron/
│   ├── main.js
│   ├── preload.js
│   └── utils/
├── src/
│   ├── App.jsx
│   ├── main.jsx
│   ├── hooks/
│   │   └── useElectron.js
│   ├── components/
│   └── pages/
├── public/
├── package.json
├── vite.config.js
└── electron-builder.json
```

## การติดตั้งและตั้งค่าโปรเจกต์

### ขั้นตอนที่ 1: สร้างโปรเจกต์ด้วย Vite

```bash
# สร้างโปรเจกต์ React ด้วย Vite
npm create vite@latest my-electron-react-app -- --template react
cd my-electron-react-app

# ติดตั้ง dependencies หลัก
npm install

# ติดตั้ง Electron และ tools
npm install --save-dev electron electron-builder vite-plugin-electron vite-plugin-electron-renderer
npm install --save-dev concurrently cross-env wait-on

# ติดตั้ง dependencies สำหรับ production
npm install electron-store
```

### ขั้นตอนที่ 2: ตั้งค่า Vite Configuration

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import electron from 'vite-plugin-electron'
import renderer from 'vite-plugin-electron-renderer'
import { resolve } from 'path'

export default defineConfig({
  plugins: [
    react(),
    electron([
      {
        // Main process entry
        entry: 'electron/main.js',
        vite: {
          build: {
            outDir: 'dist-electron',
            rollupOptions: {
              external: ['electron']
            }
          }
        }
      },
      {
        // Preload script
        entry: 'electron/preload.js',
        onstart(options) {
          // Notify renderer process to reload when preload changes
          options.reload()
        },
        vite: {
          build: {
            outDir: 'dist-electron',
            rollupOptions: {
              external: ['electron']
            }
          }
        }
      }
    ]),
    renderer()
  ],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '@electron': resolve(__dirname, 'electron')
    }
  },
  build: {
    rollupOptions: {
      input: {
        main: resolve(__dirname, 'index.html')
      }
    }
  },
  server: {
    port: 5173,
    strictPort: true
  }
})
```

### ขั้นตอนที่ 3: Main Process

```javascript
// electron/main.js
const { app, BrowserWindow, ipcMain, shell, dialog } = require('electron')
const path = require('path')
const { isDev } = require('./utils/env')

// ป้องกัน multiple instances
const gotTheLock = app.requestSingleInstanceLock()

if (!gotTheLock) {
  app.quit()
} else {
  app.on('second-instance', (event, commandLine, workingDirectory) => {
    // ถ้า user พยายามเปิด instance ที่สอง ให้ focus window ที่มีอยู่
    if (mainWindow) {
      if (mainWindow.isMinimized()) mainWindow.restore()
      mainWindow.focus()
    }
  })
}

let mainWindow = null

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    minWidth: 800,
    minHeight: 600,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, 'preload.js'),
      sandbox: false
    },
    titleBarStyle: process.platform === 'darwin' ? 'hiddenInset' : 'default',
    show: false, // ซ่อนก่อนจนกว่าจะ ready
    icon: path.join(__dirname, '../public/icon.png')
  })

  // โหลด URL ตาม environment
  if (isDev()) {
    mainWindow.loadURL('http://localhost:5173')
    mainWindow.webContents.openDevTools()
  } else {
    mainWindow.loadFile(path.join(__dirname, '../dist/index.html'))
  }

  // แสดง window เมื่อพร้อม
  mainWindow.once('ready-to-show', () => {
    mainWindow.show()
    
    // Flash taskbar ใน Windows เพื่อดึงความสนใจ
    if (process.platform === 'win32') {
      mainWindow.flashFrame(true)
      setTimeout(() => mainWindow.flashFrame(false), 1000)
    }
  })

  // Handle การปิด window
  mainWindow.on('closed', () => {
    mainWindow = null
  })

  // Handle navigation ออกนอก app
  mainWindow.webContents.on('will-navigate', (event, url) => {
    if (!url.startsWith('http://localhost') && !url.startsWith('file://')) {
      event.preventDefault()
      shell.openExternal(url)
    }
  })
}

// IPC Handlers
function setupIpcHandlers() {
  // Dialog handlers
  ipcMain.handle('dialog:openFile', async (event, options) => {
    const result = await dialog.showOpenDialog(mainWindow, options)
    return result
  })

  ipcMain.handle('dialog:saveFile', async (event, options) => {
    const result = await dialog.showSaveDialog(mainWindow, options)
    return result
  })

  ipcMain.handle('dialog:showMessage', async (event, options) => {
    const result = await dialog.showMessageBox(mainWindow, options)
    return result
  })

  // App info handlers
  ipcMain.handle('app:getVersion', () => {
    return app.getVersion()
  })

  ipcMain.handle('app:getPath', (event, name) => {
    return app.getPath(name)
  })

  // Window control handlers
  ipcMain.handle('window:minimize', () => {
    mainWindow?.minimize()
  })

  ipcMain.handle('window:maximize', () => {
    if (mainWindow?.isMaximized()) {
      mainWindow.unmaximize()
    } else {
      mainWindow?.maximize()
    }
    return mainWindow?.isMaximized()
  })

  ipcMain.handle('window:close', () => {
    mainWindow?.close()
  })

  ipcMain.handle('window:isMaximized', () => {
    return mainWindow?.isMaximized() ?? false
  })

  // Shell handlers
  ipcMain.handle('shell:openExternal', (event, url) => {
    // Validate URL ก่อนเปิด
    const allowedProtocols = ['https:', 'http:', 'mailto:']
    try {
      const parsed = new URL(url)
      if (allowedProtocols.includes(parsed.protocol)) {
        shell.openExternal(url)
        return { success: true }
      }
    } catch {
      return { success: false, error: 'Invalid URL' }
    }
  })
}

app.whenReady().then(() => {
  createWindow()
  setupIpcHandlers()

  app.on('activate', () => {
    // macOS: สร้าง window ใหม่ถ้าไม่มี
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow()
    }
  })
})

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})
```

### ขั้นตอนที่ 4: Preload Script

```javascript
// electron/preload.js
const { contextBridge, ipcRenderer } = require('electron')

// Helper function สำหรับ validate arguments
function validateString(value, name) {
  if (typeof value !== 'string') {
    throw new TypeError(`${name} must be a string`)
  }
  return value
}

// Expose API ที่ปลอดภัยให้กับ renderer process
contextBridge.exposeInMainWorld('electronAPI', {
  // App information
  app: {
    getVersion: () => ipcRenderer.invoke('app:getVersion'),
    getPath: (name) => ipcRenderer.invoke('app:getPath', validateString(name, 'name'))
  },

  // Dialog operations
  dialog: {
    openFile: (options = {}) => ipcRenderer.invoke('dialog:openFile', options),
    saveFile: (options = {}) => ipcRenderer.invoke('dialog:saveFile', options),
    showMessage: (options = {}) => ipcRenderer.invoke('dialog:showMessage', options)
  },

  // Window operations
  window: {
    minimize: () => ipcRenderer.invoke('window:minimize'),
    maximize: () => ipcRenderer.invoke('window:maximize'),
    close: () => ipcRenderer.invoke('window:close'),
    isMaximized: () => ipcRenderer.invoke('window:isMaximized'),
    
    // Event listeners
    onMaximizeChange: (callback) => {
      const listener = (event, isMaximized) => callback(isMaximized)
      ipcRenderer.on('window:maximizeChange', listener)
      return () => ipcRenderer.removeListener('window:maximizeChange', listener)
    }
  },

  // Shell operations
  shell: {
    openExternal: (url) => ipcRenderer.invoke('shell:openExternal', validateString(url, 'url'))
  },

  // Event system
  on: (channel, callback) => {
    const validChannels = ['app:update', 'notification', 'theme-change']
    if (validChannels.includes(channel)) {
      const listener = (event, ...args) => callback(...args)
      ipcRenderer.on(channel, listener)
      return () => ipcRenderer.removeListener(channel, listener)
    }
  }
})

// Type declarations (สำหรับ TypeScript)
// window.electronAPI: {
//   app: { getVersion: () => Promise<string>, getPath: (name: string) => Promise<string> },
//   dialog: { openFile: (options?: any) => Promise<any>, ... },
//   window: { minimize: () => Promise<void>, ... },
//   shell: { openExternal: (url: string) => Promise<any> },
//   on: (channel: string, callback: Function) => () => void
// }
```

## React Application

### App.jsx หลัก

```jsx
// src/App.jsx
import { useState, useEffect } from 'react'
import { Routes, Route, Navigate } from 'react-router-dom'
import { useElectron } from './hooks/useElectron'
import Layout from './components/Layout'
import Home from './pages/Home'
import Settings from './pages/Settings'
import FileManager from './pages/FileManager'
import './App.css'

function App() {
  const { isElectron, appVersion } = useElectron()
  const [theme, setTheme] = useState('light')

  useEffect(() => {
    // ตรวจสอบ system theme
    const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)')
    setTheme(mediaQuery.matches ? 'dark' : 'light')
    
    const handleChange = (e) => setTheme(e.matches ? 'dark' : 'light')
    mediaQuery.addEventListener('change', handleChange)
    
    return () => mediaQuery.removeEventListener('change', handleChange)
  }, [])

  return (
    <div className={`app ${theme}`} data-version={appVersion}>
      {isElectron && <TitleBar />}
      <Layout>
        <Routes>
          <Route path="/" element={<Navigate to="/home" replace />} />
          <Route path="/home" element={<Home />} />
          <Route path="/files" element={<FileManager />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Layout>
    </div>
  )
}

// Custom Title Bar สำหรับ Windows/Linux
function TitleBar() {
  const { window: win } = window.electronAPI || {}
  const [isMaximized, setIsMaximized] = useState(false)

  useEffect(() => {
    if (!win) return
    
    win.isMaximized().then(setIsMaximized)
    
    const cleanup = win.onMaximizeChange((maximized) => {
      setIsMaximized(maximized)
    })
    
    return cleanup
  }, [])

  if (process.platform === 'darwin') return null

  return (
    <div className="titlebar">
      <div className="titlebar-drag-region" />
      <div className="titlebar-controls">
        <button 
          className="titlebar-btn minimize"
          onClick={() => win?.minimize()}
          title="ย่อหน้าต่าง"
        >
          <span>─</span>
        </button>
        <button 
          className="titlebar-btn maximize"
          onClick={() => win?.maximize().then(setIsMaximized)}
          title={isMaximized ? "คืนขนาด" : "ขยายหน้าต่าง"}
        >
          <span>{isMaximized ? '❐' : '□'}</span>
        </button>
        <button 
          className="titlebar-btn close"
          onClick={() => win?.close()}
          title="ปิด"
        >
          <span>✕</span>
        </button>
      </div>
    </div>
  )
}

export default App
```

## Custom Hooks สำหรับ IPC

### useElectron Hook

```javascript
// src/hooks/useElectron.js
import { useState, useEffect, useCallback, useRef } from 'react'

/**
 * Main hook สำหรับ Electron integration
 * ตรวจสอบ environment และ expose Electron APIs
 */
export function useElectron() {
  const [appVersion, setAppVersion] = useState('')
  const [isElectron, setIsElectron] = useState(false)
  const [platform, setPlatform] = useState('')

  useEffect(() => {
    const electronAvailable = typeof window !== 'undefined' && 
                              !!window.electronAPI
    setIsElectron(electronAvailable)

    if (electronAvailable) {
      window.electronAPI.app.getVersion()
        .then(setAppVersion)
        .catch(console.error)
      
      // ตรวจสอบ platform จาก user agent หรือ API
      setPlatform(navigator.platform || 'unknown')
    }
  }, [])

  return {
    isElectron,
    appVersion,
    platform,
    api: window.electronAPI || null
  }
}

/**
 * Hook สำหรับ Dialog operations
 */
export function useDialog() {
  const [isLoading, setIsLoading] = useState(false)
  const [error, setError] = useState(null)

  const openFile = useCallback(async (options = {}) => {
    if (!window.electronAPI) return null
    
    setIsLoading(true)
    setError(null)
    
    try {
      const result = await window.electronAPI.dialog.openFile({
        properties: ['openFile'],
        ...options
      })
      
      if (result.canceled) return null
      return result.filePaths[0] || null
    } catch (err) {
      setError(err.message)
      return null
    } finally {
      setIsLoading(false)
    }
  }, [])

  const openFiles = useCallback(async (options = {}) => {
    if (!window.electronAPI) return []
    
    setIsLoading(true)
    setError(null)
    
    try {
      const result = await window.electronAPI.dialog.openFile({
        properties: ['openFile', 'multiSelections'],
        ...options
      })
      
      if (result.canceled) return []
      return result.filePaths || []
    } catch (err) {
      setError(err.message)
      return []
    } finally {
      setIsLoading(false)
    }
  }, [])

  const saveFile = useCallback(async (options = {}) => {
    if (!window.electronAPI) return null
    
    setIsLoading(true)
    setError(null)
    
    try {
      const result = await window.electronAPI.dialog.saveFile(options)
      if (result.canceled) return null
      return result.filePath || null
    } catch (err) {
      setError(err.message)
      return null
    } finally {
      setIsLoading(false)
    }
  }, [])

  const showMessage = useCallback(async (options = {}) => {
    if (!window.electronAPI) return null
    
    try {
      const result = await window.electronAPI.dialog.showMessage(options)
      return result
    } catch (err) {
      setError(err.message)
      return null
    }
  }, [])

  const confirm = useCallback(async (message, detail = '') => {
    return showMessage({
      type: 'question',
      buttons: ['ยืนยัน', 'ยกเลิก'],
      defaultId: 0,
      cancelId: 1,
      message,
      detail
    }).then(result => result?.response === 0)
  }, [showMessage])

  return {
    openFile,
    openFiles,
    saveFile,
    showMessage,
    confirm,
    isLoading,
    error
  }
}

/**
 * Hook สำหรับ Window operations
 */
export function useWindow() {
  const [isMaximized, setIsMaximized] = useState(false)
  const [isFocused, setIsFocused] = useState(true)

  useEffect(() => {
    if (!window.electronAPI?.window) return

    // ดึงสถานะเริ่มต้น
    window.electronAPI.window.isMaximized()
      .then(setIsMaximized)

    // Listen for maximize changes
    const cleanup = window.electronAPI.window.onMaximizeChange(setIsMaximized)

    // Focus events
    const handleFocus = () => setIsFocused(true)
    const handleBlur = () => setIsFocused(false)
    window.addEventListener('focus', handleFocus)
    window.addEventListener('blur', handleBlur)

    return () => {
      cleanup?.()
      window.removeEventListener('focus', handleFocus)
      window.removeEventListener('blur', handleBlur)
    }
  }, [])

  const minimize = useCallback(() => {
    window.electronAPI?.window.minimize()
  }, [])

  const maximize = useCallback(() => {
    window.electronAPI?.window.maximize()
      .then(setIsMaximized)
  }, [])

  const close = useCallback(() => {
    window.electronAPI?.window.close()
  }, [])

  return {
    isMaximized,
    isFocused,
    minimize,
    maximize,
    close
  }
}

/**
 * Hook สำหรับ IPC events
 */
export function useIpcEvent(channel, callback) {
  const callbackRef = useRef(callback)
  callbackRef.current = callback

  useEffect(() => {
    if (!window.electronAPI?.on) return

    const cleanup = window.electronAPI.on(channel, (...args) => {
      callbackRef.current(...args)
    })

    return () => cleanup?.()
  }, [channel])
}

/**
 * Hook สำหรับ persistent storage ผ่าน Electron
 */
export function useElectronStore(key, defaultValue) {
  const [value, setValue] = useState(defaultValue)
  const [isLoaded, setIsLoaded] = useState(false)

  useEffect(() => {
    if (!window.electronAPI?.store) {
      setIsLoaded(true)
      return
    }

    window.electronAPI.store.get(key, defaultValue)
      .then((stored) => {
        setValue(stored ?? defaultValue)
        setIsLoaded(true)
      })
  }, [key])

  const updateValue = useCallback(async (newValue) => {
    setValue(newValue)
    if (window.electronAPI?.store) {
      await window.electronAPI.store.set(key, newValue)
    }
  }, [key])

  const resetValue = useCallback(async () => {
    setValue(defaultValue)
    if (window.electronAPI?.store) {
      await window.electronAPI.store.delete(key)
    }
  }, [key, defaultValue])

  return [value, updateValue, resetValue, isLoaded]
}

/**
 * Hook สำหรับ external links
 */
export function useExternalLink() {
  const openLink = useCallback((url) => {
    if (window.electronAPI?.shell) {
      window.electronAPI.shell.openExternal(url)
    } else {
      window.open(url, '_blank', 'noopener,noreferrer')
    }
  }, [])

  return { openLink }
}
```

## React Components

### Layout Component

```jsx
// src/components/Layout.jsx
import { Link, useLocation } from 'react-router-dom'
import { useWindow } from '../hooks/useElectron'
import './Layout.css'

const navItems = [
  { path: '/home', label: 'หน้าหลัก', icon: '🏠' },
  { path: '/files', label: 'จัดการไฟล์', icon: '📁' },
  { path: '/settings', label: 'ตั้งค่า', icon: '⚙️' }
]

export default function Layout({ children }) {
  const location = useLocation()

  return (
    <div className="layout">
      <aside className="sidebar">
        <div className="sidebar-header">
          <img src="/logo.png" alt="Logo" className="sidebar-logo" />
          <h1 className="sidebar-title">My Electron App</h1>
        </div>
        
        <nav className="sidebar-nav">
          {navItems.map(item => (
            <Link
              key={item.path}
              to={item.path}
              className={`nav-item ${location.pathname === item.path ? 'active' : ''}`}
            >
              <span className="nav-icon">{item.icon}</span>
              <span className="nav-label">{item.label}</span>
            </Link>
          ))}
        </nav>
      </aside>
      
      <main className="main-content">
        {children}
      </main>
    </div>
  )
}
```

### FileManager Page

```jsx
// src/pages/FileManager.jsx
import { useState, useCallback } from 'react'
import { useDialog } from '../hooks/useElectron'

export default function FileManager() {
  const [selectedFiles, setSelectedFiles] = useState([])
  const [fileContent, setFileContent] = useState('')
  const { openFiles, saveFile, confirm, isLoading } = useDialog()

  const handleOpenFiles = useCallback(async () => {
    const files = await openFiles({
      filters: [
        { name: 'Text Files', extensions: ['txt', 'md', 'json'] },
        { name: 'All Files', extensions: ['*'] }
      ]
    })
    
    if (files.length > 0) {
      setSelectedFiles(files)
      // อ่านไฟล์แรก
      if (window.electronAPI?.fs) {
        const content = await window.electronAPI.fs.readFile(files[0])
        setFileContent(content)
      }
    }
  }, [openFiles])

  const handleSaveFile = useCallback(async () => {
    const savePath = await saveFile({
      defaultPath: 'output.txt',
      filters: [
        { name: 'Text Files', extensions: ['txt'] }
      ]
    })
    
    if (savePath && window.electronAPI?.fs) {
      await window.electronAPI.fs.writeFile(savePath, fileContent)
      alert(`บันทึกไฟล์แล้ว: ${savePath}`)
    }
  }, [saveFile, fileContent])

  const handleClearFiles = useCallback(async () => {
    if (selectedFiles.length === 0) return
    
    const confirmed = await confirm(
      'ล้างไฟล์ทั้งหมด?',
      'การดำเนินการนี้จะล้างรายการไฟล์ที่เลือกไว้'
    )
    
    if (confirmed) {
      setSelectedFiles([])
      setFileContent('')
    }
  }, [selectedFiles, confirm])

  return (
    <div className="file-manager">
      <div className="page-header">
        <h2>จัดการไฟล์</h2>
        <div className="toolbar">
          <button 
            onClick={handleOpenFiles}
            disabled={isLoading}
            className="btn btn-primary"
          >
            {isLoading ? 'กำลังโหลด...' : '📂 เปิดไฟล์'}
          </button>
          <button 
            onClick={handleSaveFile}
            disabled={!fileContent || isLoading}
            className="btn btn-secondary"
          >
            💾 บันทึกไฟล์
          </button>
          <button 
            onClick={handleClearFiles}
            disabled={selectedFiles.length === 0}
            className="btn btn-danger"
          >
            🗑️ ล้างทั้งหมด
          </button>
        </div>
      </div>

      <div className="file-manager-content">
        {/* รายการไฟล์ */}
        <div className="file-list">
          <h3>ไฟล์ที่เลือก ({selectedFiles.length})</h3>
          {selectedFiles.length === 0 ? (
            <div className="empty-state">
              <p>ยังไม่มีไฟล์ที่เลือก</p>
              <p>คลิก "เปิดไฟล์" เพื่อเลือกไฟล์</p>
            </div>
          ) : (
            <ul>
              {selectedFiles.map((file, index) => (
                <li key={index} className="file-item">
                  <span className="file-icon">📄</span>
                  <span className="file-path">{file}</span>
                </li>
              ))}
            </ul>
          )}
        </div>

        {/* ตัวแก้ไขไฟล์ */}
        <div className="file-editor">
          <h3>เนื้อหาไฟล์</h3>
          <textarea
            value={fileContent}
            onChange={(e) => setFileContent(e.target.value)}
            placeholder="เนื้อหาไฟล์จะแสดงที่นี่..."
            className="file-textarea"
            spellCheck={false}
          />
          <div className="file-stats">
            <span>{fileContent.length} ตัวอักษร</span>
            <span>{fileContent.split('\n').length} บรรทัด</span>
          </div>
        </div>
      </div>
    </div>
  )
}
```

## การตั้งค่า Hot Reload

### Electron Hot Reload Setup

```javascript
// electron/utils/env.js
const isDev = () => {
  return process.env.NODE_ENV === 'development' || 
         process.env.ELECTRON_IS_DEV === '1' ||
         !app?.isPackaged
}

module.exports = { isDev }
```

```javascript
// electron/utils/hotReload.js - ใช้เฉพาะใน development
const { app } = require('electron')
const path = require('path')
const fs = require('fs')

/**
 * ตั้งค่า hot reload สำหรับ main process
 * ใช้ fs.watch ติดตามการเปลี่ยนแปลงของไฟล์
 */
function setupHotReload(mainWindow) {
  if (process.env.NODE_ENV !== 'development') return

  const electronPath = path.join(__dirname, '../')
  
  // Watch electron directory
  const watcher = fs.watch(electronPath, { recursive: true }, (event, filename) => {
    if (filename && (filename.endsWith('.js') || filename.endsWith('.ts'))) {
      console.log(`[Hot Reload] File changed: ${filename}`)
      
      // Reload renderer
      if (mainWindow && !mainWindow.isDestroyed()) {
        mainWindow.webContents.reload()
      }
    }
  })

  app.on('quit', () => watcher.close())
}

module.exports = { setupHotReload }
```

### Package.json Scripts

```json
{
  "name": "my-electron-react-app",
  "version": "1.0.0",
  "description": "Electron + React App",
  "main": "dist-electron/main.js",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "build:electron": "electron-builder",
    "preview": "vite preview",
    "electron:dev": "cross-env NODE_ENV=development vite build --watch",
    "app:dev": "concurrently \"npm run dev\" \"wait-on http://localhost:5173 && electron .\"",
    "app:build": "npm run build && npm run build:electron",
    "postinstall": "electron-builder install-app-deps"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.0.0",
    "concurrently": "^8.0.0",
    "cross-env": "^7.0.3",
    "electron": "^27.0.0",
    "electron-builder": "^24.0.0",
    "vite": "^5.0.0",
    "vite-plugin-electron": "^0.15.0",
    "vite-plugin-electron-renderer": "^0.14.0",
    "wait-on": "^7.0.0"
  },
  "dependencies": {
    "electron-store": "^8.1.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0"
  }
}
```

## Electron Builder Configuration

```json
// electron-builder.json
{
  "appId": "com.yourcompany.myapp",
  "productName": "My Electron App",
  "copyright": "Copyright © 2024 Your Company",
  "directories": {
    "output": "release",
    "buildResources": "build"
  },
  "files": [
    "dist/**/*",
    "dist-electron/**/*",
    "package.json"
  ],
  "extraMetadata": {
    "main": "dist-electron/main.js"
  },
  "win": {
    "target": [
      {
        "target": "nsis",
        "arch": ["x64", "ia32"]
      },
      {
        "target": "portable",
        "arch": ["x64"]
      }
    ],
    "icon": "build/icon.ico",
    "publisherName": "Your Company"
  },
  "mac": {
    "target": [
      {
        "target": "dmg",
        "arch": ["x64", "arm64"]
      },
      {
        "target": "zip",
        "arch": ["x64", "arm64"]
      }
    ],
    "icon": "build/icon.icns",
    "category": "public.app-category.productivity",
    "hardenedRuntime": true,
    "gatekeeperAssess": false
  },
  "linux": {
    "target": [
      {
        "target": "AppImage",
        "arch": ["x64"]
      },
      {
        "target": "deb",
        "arch": ["x64"]
      }
    ],
    "icon": "build/icons/",
    "category": "Utility"
  },
  "nsis": {
    "oneClick": false,
    "allowToChangeInstallationDirectory": true,
    "createDesktopShortcut": true,
    "createStartMenuShortcut": true,
    "shortcutName": "My App"
  },
  "publish": {
    "provider": "github",
    "owner": "yourusername",
    "repo": "your-repo"
  },
  "compression": "maximum"
}
```

## TypeScript Support

### Type Declarations

```typescript
// src/types/electron.d.ts
export interface ElectronAPI {
  app: {
    getVersion(): Promise<string>
    getPath(name: string): Promise<string>
  }
  dialog: {
    openFile(options?: OpenDialogOptions): Promise<OpenDialogReturnValue>
    saveFile(options?: SaveDialogOptions): Promise<SaveDialogReturnValue>
    showMessage(options: MessageBoxOptions): Promise<MessageBoxReturnValue>
  }
  window: {
    minimize(): Promise<void>
    maximize(): Promise<boolean>
    close(): Promise<void>
    isMaximized(): Promise<boolean>
    onMaximizeChange(callback: (isMaximized: boolean) => void): () => void
  }
  shell: {
    openExternal(url: string): Promise<{ success: boolean; error?: string }>
  }
  on(channel: string, callback: (...args: any[]) => void): (() => void) | undefined
}

declare global {
  interface Window {
    electronAPI: ElectronAPI
  }
}
```

## CSS สำหรับ Application

```css
/* src/App.css */
:root {
  --primary-color: #6366f1;
  --primary-hover: #4f46e5;
  --bg-primary: #ffffff;
  --bg-secondary: #f8fafc;
  --text-primary: #1e293b;
  --text-secondary: #64748b;
  --border-color: #e2e8f0;
  --sidebar-width: 240px;
  --titlebar-height: 32px;
}

.app.dark {
  --bg-primary: #0f172a;
  --bg-secondary: #1e293b;
  --text-primary: #f1f5f9;
  --text-secondary: #94a3b8;
  --border-color: #334155;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.app {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background: var(--bg-primary);
  color: var(--text-primary);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

/* Titlebar */
.titlebar {
  display: flex;
  align-items: center;
  height: var(--titlebar-height);
  background: var(--bg-secondary);
  border-bottom: 1px solid var(--border-color);
  -webkit-app-region: drag;
  user-select: none;
}

.titlebar-drag-region {
  flex: 1;
}

.titlebar-controls {
  display: flex;
  -webkit-app-region: no-drag;
}

.titlebar-btn {
  width: 46px;
  height: 32px;
  border: none;
  background: transparent;
  cursor: pointer;
  font-size: 12px;
  color: var(--text-primary);
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.1s;
}

.titlebar-btn:hover {
  background: var(--border-color);
}

.titlebar-btn.close:hover {
  background: #ef4444;
  color: white;
}

/* Layout */
.layout {
  display: flex;
  flex: 1;
  overflow: hidden;
}

.sidebar {
  width: var(--sidebar-width);
  background: var(--bg-secondary);
  border-right: 1px solid var(--border-color);
  display: flex;
  flex-direction: column;
  overflow-y: auto;
}

.main-content {
  flex: 1;
  overflow-y: auto;
  padding: 24px;
}

/* Navigation */
.nav-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  text-decoration: none;
  color: var(--text-secondary);
  border-radius: 8px;
  margin: 2px 8px;
  transition: all 0.15s;
}

.nav-item:hover {
  background: var(--border-color);
  color: var(--text-primary);
}

.nav-item.active {
  background: var(--primary-color);
  color: white;
}

/* Buttons */
.btn {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.15s;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background: var(--primary-color);
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: var(--primary-hover);
}

.btn-secondary {
  background: var(--bg-secondary);
  color: var(--text-primary);
  border: 1px solid var(--border-color);
}

.btn-danger {
  background: #ef4444;
  color: white;
}
```

## สรุปและแนวทางปฏิบัติ

### Best Practices

1. **ใช้ contextBridge เสมอ** - อย่า expose API โดยตรงผ่าน nodeIntegration
2. **Validate inputs ทุก IPC call** - ตรวจสอบ type และค่าของ arguments
3. **แยก concerns ออกจากกัน** - Main process ทำงาน OS-level, Renderer ทำ UI
4. **Error handling ที่ครอบคลุม** - ใช้ try/catch ทุกที่และแสดง error ให้ user
5. **Lazy loading** - โหลดเฉพาะสิ่งที่จำเป็นเพื่อลด startup time

### ข้อควรระวัง

- อย่าใช้ `nodeIntegration: true` ใน production
- ระวัง security issues จาก remote content
- ทดสอบบนทุก platform ก่อน release
- จัดการ memory leaks ใน React components (cleanup effects)

### การ Debug

```javascript
// เปิด DevTools แบบโปรแกรม
mainWindow.webContents.openDevTools({ mode: 'detach' })

// Log ข้อมูล IPC
ipcMain.on('*', (event, ...args) => {
  console.log('[IPC]', event.type, args)
})
```
