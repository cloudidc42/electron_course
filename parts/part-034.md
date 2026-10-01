# Part 34: Multi-Window Applications ใน Electron

## บทนำ

แอปพลิเคชัน Electron ที่มีหลาย window ต้องการ patterns สำหรับการสื่อสาร, การแชร์ state, และการจัดการ lifecycle ที่ดี ในบทนี้จะเรียนรู้วิธีสร้างระบบ multi-window ที่มีประสิทธิภาพ

## Window Communication Patterns

### Pattern 1: ipcMain เป็น Message Bus

```javascript
// electron/windowManager.js
const { BrowserWindow, ipcMain, screen } = require('electron')
const path = require('path')
const EventEmitter = require('events')

class WindowManager extends EventEmitter {
  constructor() {
    super()
    this.windows = new Map()     // id -> BrowserWindow
    this.windowTypes = new Map() // id -> type
    this.nextId = 1
    
    this.setupMessageBus()
  }

  /**
   * สร้าง window และลงทะเบียน
   */
  createWindow(type, options = {}) {
    const id = this.nextId++
    
    const defaults = {
      width: 800,
      height: 600,
      show: false,
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
        preload: path.join(__dirname, 'preload.js'),
        additionalArguments: [`--window-id=${id}`, `--window-type=${type}`]
      }
    }

    const win = new BrowserWindow({ ...defaults, ...options })
    
    this.windows.set(id, win)
    this.windowTypes.set(id, type)
    
    win.once('closed', () => {
      this.windows.delete(id)
      this.windowTypes.delete(id)
      this.emit('window-closed', { id, type })
    })
    
    win.once('ready-to-show', () => {
      win.show()
      this.emit('window-ready', { id, type, win })
    })
    
    return { id, win }
  }

  /**
   * ipcMain เป็น message bus ระหว่าง windows
   */
  setupMessageBus() {
    // ส่ง message ไปยัง window เฉพาะ
    ipcMain.on('bus:send', (event, { targetId, channel, data }) => {
      const targetWin = this.windows.get(targetId)
      if (targetWin && !targetWin.isDestroyed()) {
        targetWin.webContents.send(channel, data)
      }
    })

    // Broadcast ไปทุก window
    ipcMain.on('bus:broadcast', (event, { channel, data, excludeSelf = true }) => {
      const senderId = this.getWindowId(event.sender)
      
      for (const [id, win] of this.windows) {
        if (excludeSelf && id === senderId) continue
        if (!win.isDestroyed()) {
          win.webContents.send(channel, data)
        }
      }
    })

    // Broadcast ไปยัง windows ประเภทเดียวกัน
    ipcMain.on('bus:broadcast-type', (event, { type, channel, data }) => {
      for (const [id, win] of this.windows) {
        if (this.windowTypes.get(id) === type && !win.isDestroyed()) {
          win.webContents.send(channel, data)
        }
      }
    })

    // Request-response pattern
    ipcMain.handle('bus:request', async (event, { targetId, channel, data, timeout = 5000 }) => {
      const targetWin = this.windows.get(targetId)
      if (!targetWin || targetWin.isDestroyed()) {
        throw new Error(`Window ${targetId} not found`)
      }

      return new Promise((resolve, reject) => {
        const responseChannel = `${channel}:response:${Date.now()}`
        
        const timer = setTimeout(() => {
          ipcMain.removeAllListeners(responseChannel)
          reject(new Error(`Request timeout after ${timeout}ms`))
        }, timeout)
        
        ipcMain.once(responseChannel, (_, response) => {
          clearTimeout(timer)
          resolve(response)
        })
        
        targetWin.webContents.send(channel, { data, responseChannel })
      })
    })

    // List windows
    ipcMain.handle('bus:list-windows', () => {
      return Array.from(this.windows.entries()).map(([id, win]) => ({
        id,
        type: this.windowTypes.get(id),
        title: win.isDestroyed() ? null : win.getTitle(),
        focused: win.isDestroyed() ? false : win.isFocused()
      }))
    })

    // Focus window
    ipcMain.handle('bus:focus-window', (_, id) => {
      const win = this.windows.get(id)
      if (win && !win.isDestroyed()) {
        if (win.isMinimized()) win.restore()
        win.focus()
        return true
      }
      return false
    })
  }

  getWindowId(webContents) {
    for (const [id, win] of this.windows) {
      if (!win.isDestroyed() && win.webContents === webContents) {
        return id
      }
    }
    return null
  }

  getWindow(id) {
    return this.windows.get(id)
  }

  getAllWindows() {
    return Array.from(this.windows.entries())
  }

  closeAll() {
    for (const [, win] of this.windows) {
      if (!win.isDestroyed()) {
        win.close()
      }
    }
  }
}

module.exports = new WindowManager()
```

### Pattern 2: Shared State ด้วย electron-store

```javascript
// electron/sharedState.js
const Store = require('electron-store')
const { ipcMain } = require('electron')
const windowManager = require('./windowManager')

class SharedStateManager {
  constructor() {
    this.store = new Store({
      name: 'shared-state',
      defaults: {
        theme: 'system',
        activeProject: null,
        recentFiles: [],
        windowStates: {}
      }
    })
    
    // Watch for changes และ broadcast
    this.store.onDidAnyChange((newValue, oldValue) => {
      this.broadcastStateChange(newValue, oldValue)
    })
    
    this.setupIpcHandlers()
  }

  get(key) {
    return this.store.get(key)
  }

  set(key, value) {
    this.store.set(key, value)
    // onChange handler จะ broadcast โดยอัตโนมัติ
  }

  broadcastStateChange(newState, oldState) {
    const changes = this.diffState(newState, oldState)
    if (Object.keys(changes).length === 0) return
    
    for (const [, win] of windowManager.getAllWindows()) {
      if (!win.isDestroyed()) {
        win.webContents.send('state:changed', changes)
      }
    }
  }

  diffState(newState, oldState) {
    const changes = {}
    for (const key in newState) {
      if (JSON.stringify(newState[key]) !== JSON.stringify(oldState[key])) {
        changes[key] = { new: newState[key], old: oldState[key] }
      }
    }
    return changes
  }

  setupIpcHandlers() {
    ipcMain.handle('state:get', (_, key) => {
      return key ? this.store.get(key) : this.store.store
    })
    
    ipcMain.handle('state:set', (_, key, value) => {
      this.store.set(key, value)
      return true
    })
    
    ipcMain.handle('state:delete', (_, key) => {
      this.store.delete(key)
      return true
    })
    
    ipcMain.handle('state:reset', () => {
      this.store.clear()
      return true
    })
  }
}

module.exports = new SharedStateManager()
```

## BroadcastChannel API

```javascript
// src/hooks/useWindowBus.js
import { useEffect, useCallback, useRef } from 'react'

/**
 * Hook สำหรับสื่อสารระหว่าง windows ผ่าน BroadcastChannel
 * ทำงานได้เฉพาะ windows ที่มี same origin (ซึ่งทุก window ใน Electron มี)
 */
export function useWindowBus(channelName) {
  const channelRef = useRef(null)
  const handlersRef = useRef(new Map())

  useEffect(() => {
    channelRef.current = new BroadcastChannel(channelName)
    
    channelRef.current.onmessage = (event) => {
      const { type, data } = event.data
      const handler = handlersRef.current.get(type)
      if (handler) {
        handler(data, event)
      }
    }
    
    return () => {
      channelRef.current.close()
      channelRef.current = null
    }
  }, [channelName])

  const send = useCallback((type, data) => {
    if (channelRef.current) {
      channelRef.current.postMessage({ type, data })
    }
  }, [])

  const on = useCallback((type, handler) => {
    handlersRef.current.set(type, handler)
    
    return () => {
      handlersRef.current.delete(type)
    }
  }, [])

  return { send, on }
}

// ตัวอย่างการใช้
function EditorWindow() {
  const { send, on } = useWindowBus('app-channel')
  
  useEffect(() => {
    const off = on('file:opened', (file) => {
      console.log('Another window opened:', file)
    })
    return off
  }, [on])
  
  const handleSave = () => {
    send('file:saved', { name: 'document.txt', timestamp: Date.now() })
  }
  
  return <button onClick={handleSave}>Save</button>
}
```

## Window Lifecycle Management

```javascript
// electron/main.js
const { app, BrowserWindow, ipcMain } = require('electron')
const path = require('path')
const windowManager = require('./windowManager')
const sharedState = require('./sharedState')

// Main window
let mainWindow = null

function createMainWindow() {
  const { id, win } = windowManager.createWindow('main', {
    width: 1200,
    height: 800,
    minWidth: 800,
    minHeight: 600
  })
  
  mainWindow = win
  
  // โหลด URL
  const url = process.env.VITE_DEV_SERVER_URL || `file://${path.join(__dirname, '../dist/index.html')}`
  win.loadURL(url)
  
  // จัดการ window state
  restoreWindowState(win, 'main')
  win.on('resize', () => saveWindowState(win, 'main'))
  win.on('move', () => saveWindowState(win, 'main'))
  
  win.on('close', (e) => {
    if (shouldConfirmClose()) {
      e.preventDefault()
      confirmClose(win)
    }
  })
  
  return { id, win }
}

// Secondary/Panel window
function createPanelWindow(panelType, parentWin) {
  const bounds = calculatePanelBounds(panelType, parentWin)
  
  const { id, win } = windowManager.createWindow('panel', {
    ...bounds,
    parent: parentWin,   // child window
    resizable: true,
    minimizable: false,
    maximizable: false,
    frame: false,
    transparent: true
  })
  
  const url = `${getBaseUrl()}#/panel/${panelType}`
  win.loadURL(url)
  
  // ติดตาม parent window
  if (parentWin) {
    const onParentMove = () => {
      if (!win.isDestroyed()) {
        const newBounds = calculatePanelBounds(panelType, parentWin)
        win.setBounds(newBounds)
      }
    }
    
    parentWin.on('move', onParentMove)
    win.on('closed', () => parentWin.off('move', onParentMove))
  }
  
  return { id, win }
}

function calculatePanelBounds(panelType, parentWin) {
  if (!parentWin || parentWin.isDestroyed()) {
    return { width: 300, height: 600, x: 0, y: 0 }
  }
  
  const parentBounds = parentWin.getBounds()
  
  switch (panelType) {
    case 'sidebar-left':
      return {
        width: 250,
        height: parentBounds.height,
        x: parentBounds.x - 250,
        y: parentBounds.y
      }
    case 'sidebar-right':
      return {
        width: 250,
        height: parentBounds.height,
        x: parentBounds.x + parentBounds.width,
        y: parentBounds.y
      }
    default:
      return { width: 300, height: 400, x: parentBounds.x + 50, y: parentBounds.y + 50 }
  }
}

// Saving/Restoring window layout
function saveWindowState(win, key) {
  if (win.isDestroyed() || win.isMinimized()) return
  
  const bounds = win.getBounds()
  const isMaximized = win.isMaximized()
  const isFullScreen = win.isFullScreen()
  
  const windowStates = sharedState.get('windowStates') || {}
  windowStates[key] = { bounds, isMaximized, isFullScreen, savedAt: Date.now() }
  sharedState.set('windowStates', windowStates)
}

function restoreWindowState(win, key) {
  const windowStates = sharedState.get('windowStates') || {}
  const state = windowStates[key]
  
  if (!state) return
  
  // ตรวจสอบว่า bounds ยังอยู่ใน screen
  const { screen } = require('electron')
  const displays = screen.getAllDisplays()
  const bounds = state.bounds
  
  const isVisible = displays.some(display => {
    const db = display.bounds
    return bounds.x >= db.x && bounds.y >= db.y &&
           bounds.x + bounds.width <= db.x + db.width &&
           bounds.y + bounds.height <= db.y + db.height
  })
  
  if (isVisible) {
    win.setBounds(bounds)
  }
  
  if (state.isMaximized) win.maximize()
  if (state.isFullScreen) win.setFullScreen(true)
}

// Window Groups
class WindowGroup {
  constructor(name) {
    this.name = name
    this.windowIds = new Set()
  }

  add(windowId) {
    this.windowIds.add(windowId)
  }

  remove(windowId) {
    this.windowIds.delete(windowId)
  }

  broadcast(channel, data) {
    for (const id of this.windowIds) {
      const win = windowManager.getWindow(id)
      if (win && !win.isDestroyed()) {
        win.webContents.send(channel, data)
      }
    }
  }

  closeAll() {
    for (const id of this.windowIds) {
      const win = windowManager.getWindow(id)
      if (win && !win.isDestroyed()) {
        win.close()
      }
    }
  }

  get size() {
    return this.windowIds.size
  }
}

const windowGroups = new Map()

ipcMain.handle('group:create', (_, groupName) => {
  if (!windowGroups.has(groupName)) {
    windowGroups.set(groupName, new WindowGroup(groupName))
  }
  return { ok: true }
})

ipcMain.handle('group:add-window', (event, { groupName, windowId }) => {
  if (!windowGroups.has(groupName)) {
    windowGroups.set(groupName, new WindowGroup(groupName))
  }
  windowGroups.get(groupName).add(windowId)
  return { ok: true }
})

ipcMain.handle('group:broadcast', (event, { groupName, channel, data }) => {
  const group = windowGroups.get(groupName)
  if (group) {
    group.broadcast(channel, data)
    return { sent: group.size }
  }
  return { sent: 0 }
})

// Utility functions
function shouldConfirmClose() {
  // ตรวจสอบว่ามี unsaved changes ไหม
  return false // implement as needed
}

async function confirmClose(win) {
  const { dialog } = require('electron')
  const result = await dialog.showMessageBox(win, {
    type: 'question',
    buttons: ['Cancel', 'Close without saving', 'Save and close'],
    defaultId: 0,
    title: 'Unsaved Changes',
    message: 'You have unsaved changes. What would you like to do?'
  })
  
  if (result.response === 1) {
    win.destroy()
  } else if (result.response === 2) {
    // save then close
    win.webContents.send('app:save-and-close')
  }
}

function getBaseUrl() {
  return process.env.VITE_DEV_SERVER_URL || `file://${path.join(__dirname, '../dist/index.html')}`
}

module.exports = { createMainWindow, createPanelWindow, saveWindowState, restoreWindowState }
```

## Preload สำหรับ Multi-Window

```javascript
// electron/preload.js
const { contextBridge, ipcRenderer } = require('electron')

// อ่าน window info จาก additionalArguments
const args = process.argv
const windowIdArg = args.find(a => a.startsWith('--window-id='))
const windowTypeArg = args.find(a => a.startsWith('--window-type='))

const WINDOW_ID = windowIdArg ? parseInt(windowIdArg.split('=')[1]) : null
const WINDOW_TYPE = windowTypeArg ? windowTypeArg.split('=')[1] : 'unknown'

contextBridge.exposeInMainWorld('electronAPI', {
  // Window info
  windowId: WINDOW_ID,
  windowType: WINDOW_TYPE,

  // Message bus
  bus: {
    send: (targetId, channel, data) =>
      ipcRenderer.send('bus:send', { targetId, channel, data }),
    
    broadcast: (channel, data, excludeSelf = true) =>
      ipcRenderer.send('bus:broadcast', { channel, data, excludeSelf }),
    
    broadcastToType: (type, channel, data) =>
      ipcRenderer.send('bus:broadcast-type', { type, channel, data }),
    
    request: (targetId, channel, data, timeout) =>
      ipcRenderer.invoke('bus:request', { targetId, channel, data, timeout }),
    
    listWindows: () =>
      ipcRenderer.invoke('bus:list-windows'),
    
    focusWindow: (id) =>
      ipcRenderer.invoke('bus:focus-window', id),
    
    on: (channel, handler) => {
      const wrappedHandler = (_, ...args) => handler(...args)
      ipcRenderer.on(channel, wrappedHandler)
      return () => ipcRenderer.off(channel, wrappedHandler)
    }
  },

  // Shared state
  state: {
    get: (key) => ipcRenderer.invoke('state:get', key),
    set: (key, value) => ipcRenderer.invoke('state:set', key, value),
    delete: (key) => ipcRenderer.invoke('state:delete', key),
    onChange: (handler) => {
      const wrappedHandler = (_, changes) => handler(changes)
      ipcRenderer.on('state:changed', wrappedHandler)
      return () => ipcRenderer.off('state:changed', wrappedHandler)
    }
  },

  // Window groups
  groups: {
    create: (name) => ipcRenderer.invoke('group:create', name),
    addWindow: (groupName, windowId) =>
      ipcRenderer.invoke('group:add-window', { groupName, windowId }),
    broadcast: (groupName, channel, data) =>
      ipcRenderer.invoke('group:broadcast', { groupName, channel, data })
  }
})
```

## React Hooks สำหรับ Multi-Window

```jsx
// src/hooks/useSharedState.js
import { useState, useEffect, useCallback } from 'react'

/**
 * Hook ที่ sync state กับ electron-store และ broadcast ไปทุก window
 */
export function useSharedState(key, initialValue) {
  const [value, setValue] = useState(initialValue)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // โหลด initial value
    window.electronAPI.state.get(key).then(stored => {
      if (stored !== undefined) setValue(stored)
      setLoading(false)
    })
    
    // Listen for changes จาก windows อื่น
    const cleanup = window.electronAPI.state.onChange((changes) => {
      if (changes[key]) {
        setValue(changes[key].new)
      }
    })
    
    return cleanup
  }, [key])

  const setState = useCallback(async (newValue) => {
    const computed = typeof newValue === 'function' ? newValue(value) : newValue
    setValue(computed)  // optimistic update
    await window.electronAPI.state.set(key, computed)
  }, [key, value])

  return [value, setState, loading]
}

// src/hooks/useWindowBroadcast.js
import { useEffect, useCallback } from 'react'

export function useWindowBroadcast() {
  const broadcast = useCallback((channel, data) => {
    window.electronAPI.bus.broadcast(channel, data)
  }, [])

  const onBroadcast = useCallback((channel, handler) => {
    return window.electronAPI.bus.on(channel, handler)
  }, [])

  return { broadcast, onBroadcast }
}

// src/hooks/useWindowList.js
import { useState, useEffect } from 'react'

export function useWindowList() {
  const [windows, setWindows] = useState([])

  useEffect(() => {
    const loadWindows = async () => {
      const list = await window.electronAPI.bus.listWindows()
      setWindows(list)
    }
    
    loadWindows()
    
    // Refresh เมื่อมี window เปิด/ปิด
    const cleanup = window.electronAPI.bus.on('window:changed', loadWindows)
    return cleanup
  }, [])

  const focusWindow = (id) => {
    window.electronAPI.bus.focusWindow(id)
  }

  return { windows, focusWindow, currentId: window.electronAPI.windowId }
}
```

## Window Switcher Component

```jsx
// src/components/WindowSwitcher.jsx
import { useState } from 'react'
import { useWindowList } from '../hooks/useWindowList'

export default function WindowSwitcher() {
  const { windows, focusWindow, currentId } = useWindowList()
  const [isOpen, setIsOpen] = useState(false)

  const otherWindows = windows.filter(w => w.id !== currentId)

  if (otherWindows.length === 0) return null

  return (
    <div className="window-switcher">
      <button
        className="switcher-toggle"
        onClick={() => setIsOpen(!isOpen)}
        title="Switch Windows"
      >
        <span>Windows ({otherWindows.length})</span>
        <span className={`arrow ${isOpen ? 'up' : 'down'}`}>▾</span>
      </button>

      {isOpen && (
        <div className="switcher-dropdown">
          {otherWindows.map(win => (
            <button
              key={win.id}
              className={`window-item ${win.focused ? 'focused' : ''}`}
              onClick={() => {
                focusWindow(win.id)
                setIsOpen(false)
              }}
            >
              <span className="window-type">{win.type}</span>
              <span className="window-title">{win.title || `Window ${win.id}`}</span>
              {win.focused && <span className="active-badge">Active</span>}
            </button>
          ))}
        </div>
      )}
    </div>
  )
}
```

## สรุป

### Patterns สำหรับ Multi-Window

| Pattern | เหมาะกับ |
|---------|----------|
| ipcMain Message Bus | ข้อความที่ต้องผ่าน main process |
| BroadcastChannel | ข้อมูล real-time ระหว่าง windows |
| Shared electron-store | State ที่ต้องการ persist |
| Window Groups | จัดการ windows หลายอันพร้อมกัน |

### Best Practices

1. **เก็บ window ID** ใน additionalArguments เพื่อระบุ window
2. **Save window state** ด้วย resize/move events
3. **ตรวจสอบ screen bounds** ก่อน restore window position
4. **Use child windows** สำหรับ panels ที่ follow parent
5. **BroadcastChannel** เร็วกว่า IPC สำหรับ frequent updates
