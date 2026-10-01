# ตอนที่ 11: Preload Scripts

## ทำความเข้าใจ Preload Scripts ใน Electron

Preload Scripts คือไฟล์ JavaScript พิเศษที่ถูกรันก่อนที่ renderer process จะโหลดหน้าเว็บ โดยทำงานในบริบทพิเศษที่มีสิทธิ์เข้าถึงทั้ง Node.js APIs และ DOM APIs ซึ่งแตกต่างจาก renderer process ปกติที่ถูกจำกัดไม่ให้ใช้งาน Node.js โดยตรง

### ทำไมต้องใช้ Preload Scripts?

ก่อนที่ Electron จะมี `contextIsolation` นักพัฒนาสามารถใช้ Node.js ได้โดยตรงใน renderer process แต่นั่นสร้างความเสี่ยงด้านความปลอดภัยอย่างมาก เพราะหากมีโค้ดอันตรายถูกรันใน renderer (เช่น จาก XSS attack) มันจะสามารถเข้าถึง filesystem, exec commands, และทำสิ่งอันตรายอื่นๆ ได้

Preload Scripts แก้ปัญหานี้โดยทำหน้าที่เป็น "สะพาน" ที่ปลอดภัยระหว่าง main process และ renderer process

```
Main Process (Node.js)
       ↕ IPC
Preload Script (Node.js + DOM)  ← ทำงานตรงนี้
       ↕ contextBridge
Renderer Process (Browser only)
```

## การตั้งค่า Preload Script เบื้องต้น

### โครงสร้างโปรเจกต์

```
my-electron-app/
├── main.js
├── preload.js
├── index.html
├── renderer.js
└── package.json
```

### main.js - การลงทะเบียน Preload Script

```javascript
const { app, BrowserWindow } = require('electron')
const path = require('path')

function createWindow() {
  const win = new BrowserWindow({
    width: 900,
    height: 700,
    webPreferences: {
      // ระบุ path ของ preload script
      preload: path.join(__dirname, 'preload.js'),
      
      // เปิดใช้ context isolation (แนะนำเสมอ)
      contextIsolation: true,
      
      // ปิด nodeIntegration ใน renderer
      nodeIntegration: false,
      
      // ปิด remote module
      enableRemoteModule: false,
      
      // ปิด sandbox (เปิดได้ถ้าไม่ต้องการ Node.js ใน preload)
      sandbox: false
    }
  })
  
  win.loadFile('index.html')
}

app.whenReady().then(createWindow)

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})

app.on('activate', () => {
  if (BrowserWindow.getAllWindows().length === 0) createWindow()
})
```

### preload.js - โครงสร้างพื้นฐาน

```javascript
// preload.js
// ไฟล์นี้รันในบริบทพิเศษ: มีทั้ง Node.js และ browser APIs

const { contextBridge, ipcRenderer } = require('electron')

// ตรวจสอบว่า contextBridge พร้อมใช้งาน
console.log('Preload script loaded!')
console.log('Node version:', process.versions.node)
console.log('Chrome version:', process.versions.chrome)
console.log('Electron version:', process.versions.electron)

// expose APIs ไปยัง renderer
contextBridge.exposeInMainWorld('electronAPI', {
  // จะเพิ่ม methods ที่นี่
  getVersions: () => ({
    node: process.versions.node,
    chrome: process.versions.chrome,
    electron: process.versions.electron
  })
})
```

## contextBridge.exposeInMainWorld

`contextBridge.exposeInMainWorld` เป็น API หลักที่ใช้สำหรับส่งผ่านข้อมูลและ functions จาก preload context ไปยัง renderer context อย่างปลอดภัย

### รูปแบบพื้นฐาน

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('apiName', {
  // expose functions
  methodName: (arg) => {
    // ทำงานที่นี่
    return result
  },
  
  // expose properties (แบบ read-only)
  version: process.version,
  
  // expose nested objects
  utils: {
    log: (msg) => console.log('[Renderer]:', msg),
    warn: (msg) => console.warn('[Renderer]:', msg)
  }
})
```

### ตัวอย่างการ expose File System Operations

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')
const fs = require('fs')
const path = require('path')

contextBridge.exposeInMainWorld('fileAPI', {
  // อ่านไฟล์
  readFile: (filePath) => {
    try {
      return fs.readFileSync(filePath, 'utf8')
    } catch (error) {
      throw new Error(`Cannot read file: ${error.message}`)
    }
  },
  
  // เขียนไฟล์
  writeFile: (filePath, content) => {
    try {
      fs.writeFileSync(filePath, content, 'utf8')
      return true
    } catch (error) {
      throw new Error(`Cannot write file: ${error.message}`)
    }
  },
  
  // ตรวจสอบว่าไฟล์มีอยู่
  fileExists: (filePath) => {
    return fs.existsSync(filePath)
  },
  
  // join paths
  joinPath: (...args) => path.join(...args)
})
```

### การใช้งานใน renderer.js

```javascript
// renderer.js
// เรียกใช้ APIs ที่ expose ไว้
document.addEventListener('DOMContentLoaded', async () => {
  // อ่านไฟล์
  try {
    const content = window.fileAPI.readFile('/tmp/test.txt')
    console.log('File content:', content)
  } catch (error) {
    console.error('Error:', error.message)
  }
  
  // เขียนไฟล์
  try {
    window.fileAPI.writeFile('/tmp/output.txt', 'Hello from renderer!')
    console.log('File written successfully')
  } catch (error) {
    console.error('Error:', error.message)
  }
  
  // ตรวจสอบว่าไฟล์มีอยู่
  const exists = window.fileAPI.fileExists('/tmp/test.txt')
  console.log('File exists:', exists)
})
```

## ipcRenderer ใน Preload Scripts

`ipcRenderer` ใช้สำหรับการสื่อสารระหว่าง renderer และ main process ผ่าน preload script

### Pattern พื้นฐาน: ipcRenderer.send / ipcMain.on

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('ipcAPI', {
  // ส่งข้อความไป main process (ไม่รอผลลัพธ์)
  send: (channel, data) => {
    // whitelist ของ channels ที่อนุญาต
    const validChannels = ['app:minimize', 'app:maximize', 'app:close', 'log:write']
    if (validChannels.includes(channel)) {
      ipcRenderer.send(channel, data)
    }
  },
  
  // รับข้อความจาก main process
  on: (channel, callback) => {
    const validChannels = ['app:update-available', 'app:theme-changed', 'app:notification']
    if (validChannels.includes(channel)) {
      // ห่อ callback เพื่อความปลอดภัย
      const subscription = (_event, ...args) => callback(...args)
      ipcRenderer.on(channel, subscription)
      
      // return function สำหรับ cleanup
      return () => ipcRenderer.removeListener(channel, subscription)
    }
  }
})
```

### Pattern: invoke/handle (Async Request-Response)

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('electronAPI', {
  // invoke: ส่ง request และรอ response (async)
  invoke: async (channel, ...args) => {
    const validChannels = [
      'dialog:open-file',
      'dialog:save-file',
      'system:get-info',
      'db:query',
      'file:read',
      'file:write'
    ]
    
    if (validChannels.includes(channel)) {
      return await ipcRenderer.invoke(channel, ...args)
    }
    
    throw new Error(`Channel "${channel}" is not allowed`)
  }
})
```

```javascript
// main.js - handlers สำหรับ invoke
const { ipcMain, dialog } = require('electron')
const fs = require('fs')

// handler สำหรับเปิด dialog เลือกไฟล์
ipcMain.handle('dialog:open-file', async (event, options) => {
  const result = await dialog.showOpenDialog(options)
  return result
})

// handler สำหรับอ่านไฟล์
ipcMain.handle('file:read', async (event, filePath) => {
  try {
    const content = fs.readFileSync(filePath, 'utf8')
    return { success: true, content }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

// handler สำหรับเขียนไฟล์
ipcMain.handle('file:write', async (event, filePath, content) => {
  try {
    fs.writeFileSync(filePath, content, 'utf8')
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})
```

```javascript
// renderer.js - เรียกใช้งาน
async function openFileDialog() {
  const result = await window.electronAPI.invoke('dialog:open-file', {
    title: 'เลือกไฟล์',
    filters: [
      { name: 'Text Files', extensions: ['txt', 'md'] },
      { name: 'All Files', extensions: ['*'] }
    ],
    properties: ['openFile']
  })
  
  if (!result.canceled && result.filePaths.length > 0) {
    const filePath = result.filePaths[0]
    
    const fileResult = await window.electronAPI.invoke('file:read', filePath)
    
    if (fileResult.success) {
      document.getElementById('content').textContent = fileResult.content
    } else {
      console.error('Error reading file:', fileResult.error)
    }
  }
}
```

## Pattern สำหรับ Expose APIs ที่ดี

### Pattern 1: Typed API with Validation

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

// helper function สำหรับ validate channel
function validateChannel(channel, allowedChannels) {
  if (!allowedChannels.includes(channel)) {
    throw new Error(`Channel "${channel}" is not allowed`)
  }
}

// helper function สำหรับ validate data types
function validateString(value, name) {
  if (typeof value !== 'string') {
    throw new TypeError(`${name} must be a string`)
  }
}

function validateObject(value, name) {
  if (typeof value !== 'object' || value === null) {
    throw new TypeError(`${name} must be an object`)
  }
}

contextBridge.exposeInMainWorld('appAPI', {
  // Window controls
  window: {
    minimize: () => ipcRenderer.send('window:minimize'),
    maximize: () => ipcRenderer.send('window:maximize'),
    close: () => ipcRenderer.send('window:close'),
    
    onMaximized: (callback) => {
      const fn = () => callback(true)
      ipcRenderer.on('window:maximized', fn)
      return () => ipcRenderer.removeListener('window:maximized', fn)
    },
    
    onUnmaximized: (callback) => {
      const fn = () => callback(false)
      ipcRenderer.on('window:unmaximized', fn)
      return () => ipcRenderer.removeListener('window:unmaximized', fn)
    }
  },
  
  // File operations
  file: {
    open: async (options = {}) => {
      validateObject(options, 'options')
      return await ipcRenderer.invoke('file:open-dialog', options)
    },
    
    read: async (filePath) => {
      validateString(filePath, 'filePath')
      return await ipcRenderer.invoke('file:read', filePath)
    },
    
    write: async (filePath, content) => {
      validateString(filePath, 'filePath')
      validateString(content, 'content')
      return await ipcRenderer.invoke('file:write', filePath, content)
    },
    
    exists: async (filePath) => {
      validateString(filePath, 'filePath')
      return await ipcRenderer.invoke('file:exists', filePath)
    }
  },
  
  // App info
  app: {
    getVersion: () => ipcRenderer.invoke('app:get-version'),
    getName: () => ipcRenderer.invoke('app:get-name'),
    getPath: (name) => {
      const validPaths = ['home', 'appData', 'userData', 'temp', 'desktop', 'documents', 'downloads']
      if (!validPaths.includes(name)) {
        throw new Error(`Invalid path name: ${name}`)
      }
      return ipcRenderer.invoke('app:get-path', name)
    }
  },
  
  // Notifications
  notification: {
    show: async (title, body, options = {}) => {
      validateString(title, 'title')
      validateString(body, 'body')
      return await ipcRenderer.invoke('notification:show', { title, body, ...options })
    }
  }
})
```

### Pattern 2: Event Emitter Style

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

// สร้าง event registry สำหรับ cleanup
const listeners = new Map()

contextBridge.exposeInMainWorld('events', {
  // subscribe to events from main process
  on: (event, callback) => {
    const allowedEvents = [
      'app:theme-changed',
      'app:language-changed', 
      'app:update-available',
      'app:update-downloaded',
      'file:watcher-changed',
      'network:status-changed'
    ]
    
    if (!allowedEvents.includes(event)) {
      throw new Error(`Event "${event}" is not supported`)
    }
    
    const wrappedCallback = (_ipcEvent, data) => callback(data)
    ipcRenderer.on(event, wrappedCallback)
    
    // เก็บ reference สำหรับ cleanup
    if (!listeners.has(event)) {
      listeners.set(event, new Set())
    }
    listeners.get(event).add(wrappedCallback)
    
    // return unsubscribe function
    return () => {
      ipcRenderer.removeListener(event, wrappedCallback)
      listeners.get(event)?.delete(wrappedCallback)
    }
  },
  
  // emit event to main process
  emit: (event, data) => {
    const allowedEvents = [
      'renderer:ready',
      'renderer:error',
      'user:action'
    ]
    
    if (!allowedEvents.includes(event)) {
      throw new Error(`Event "${event}" cannot be emitted from renderer`)
    }
    
    ipcRenderer.send(event, data)
  },
  
  // once: listen for event only once
  once: (event, callback) => {
    const wrappedCallback = (_ipcEvent, data) => callback(data)
    ipcRenderer.once(event, wrappedCallback)
  }
})
```

### Pattern 3: Promise-based API

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('database', {
  // CRUD operations
  query: async (sql, params = []) => {
    if (typeof sql !== 'string') throw new TypeError('sql must be a string')
    if (!Array.isArray(params)) throw new TypeError('params must be an array')
    
    return await ipcRenderer.invoke('db:query', { sql, params })
  },
  
  insert: async (table, data) => {
    if (typeof table !== 'string') throw new TypeError('table must be a string')
    if (typeof data !== 'object' || data === null) throw new TypeError('data must be an object')
    
    return await ipcRenderer.invoke('db:insert', { table, data })
  },
  
  update: async (table, data, where) => {
    if (typeof table !== 'string') throw new TypeError('table must be a string')
    
    return await ipcRenderer.invoke('db:update', { table, data, where })
  },
  
  delete: async (table, where) => {
    if (typeof table !== 'string') throw new TypeError('table must be a string')
    
    return await ipcRenderer.invoke('db:delete', { table, where })
  },
  
  transaction: async (operations) => {
    if (!Array.isArray(operations)) throw new TypeError('operations must be an array')
    
    return await ipcRenderer.invoke('db:transaction', operations)
  }
})
```

## Security Considerations

### สิ่งที่ต้องระวังใน Preload Scripts

```javascript
// preload.js - ตัวอย่างที่ไม่ปลอดภัย (อย่าทำแบบนี้!)

// ❌ ห้ามทำ: expose ipcRenderer โดยตรง
contextBridge.exposeInMainWorld('ipc', ipcRenderer) // อันตราย!

// ❌ ห้ามทำ: expose require
contextBridge.exposeInMainWorld('require', require) // อันตราย!

// ❌ ห้ามทำ: expose process object
contextBridge.exposeInMainWorld('process', process) // อันตราย!

// ❌ ห้ามทำ: expose fs module
contextBridge.exposeInMainWorld('fs', require('fs')) // อันตราย!

// ❌ ห้ามทำ: ไม่มี channel validation
contextBridge.exposeInMainWorld('send', (channel, data) => {
  ipcRenderer.send(channel, data) // ไม่มีการกรอง channel!
})
```

```javascript
// preload.js - ตัวอย่างที่ปลอดภัย (ทำแบบนี้!)

const { contextBridge, ipcRenderer } = require('electron')

// ✅ กำหนด whitelist ของ channels อย่างชัดเจน
const ALLOWED_SEND_CHANNELS = Object.freeze([
  'window:minimize',
  'window:maximize', 
  'window:close',
  'app:quit'
])

const ALLOWED_INVOKE_CHANNELS = Object.freeze([
  'dialog:open',
  'dialog:save',
  'file:read',
  'file:write',
  'app:get-version'
])

const ALLOWED_RECEIVE_CHANNELS = Object.freeze([
  'app:update-available',
  'app:theme-changed',
  'window:state-changed'
])

// ✅ expose เฉพาะสิ่งที่จำเป็นพร้อม validation
contextBridge.exposeInMainWorld('electronAPI', {
  send: (channel, data) => {
    if (!ALLOWED_SEND_CHANNELS.includes(channel)) {
      console.error(`Blocked: sending to channel "${channel}" is not allowed`)
      return
    }
    ipcRenderer.send(channel, data)
  },
  
  invoke: async (channel, ...args) => {
    if (!ALLOWED_INVOKE_CHANNELS.includes(channel)) {
      throw new Error(`Channel "${channel}" is not allowed`)
    }
    return await ipcRenderer.invoke(channel, ...args)
  },
  
  on: (channel, callback) => {
    if (!ALLOWED_RECEIVE_CHANNELS.includes(channel)) {
      throw new Error(`Cannot listen to channel "${channel}"`)
    }
    
    if (typeof callback !== 'function') {
      throw new TypeError('callback must be a function')
    }
    
    const wrappedCallback = (_event, ...args) => callback(...args)
    ipcRenderer.on(channel, wrappedCallback)
    
    return () => ipcRenderer.removeListener(channel, wrappedCallback)
  },
  
  removeAllListeners: (channel) => {
    if (ALLOWED_RECEIVE_CHANNELS.includes(channel)) {
      ipcRenderer.removeAllListeners(channel)
    }
  }
})
```

## ตัวอย่างโปรเจกต์สมบูรณ์: Text Editor

### โครงสร้าง

```
text-editor/
├── main.js
├── preload.js
├── renderer/
│   ├── index.html
│   ├── renderer.js
│   └── styles.css
└── package.json
```

### main.js

```javascript
const { app, BrowserWindow, ipcMain, dialog, Menu } = require('electron')
const path = require('path')
const fs = require('fs')

let mainWindow
let currentFilePath = null

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    titleBarStyle: 'hidden',
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  mainWindow.loadFile('renderer/index.html')
  
  // ส่ง event เมื่อ window maximize/unmaximize
  mainWindow.on('maximize', () => {
    mainWindow.webContents.send('window:maximized')
  })
  
  mainWindow.on('unmaximize', () => {
    mainWindow.webContents.send('window:unmaximized')
  })
}

// Window controls
ipcMain.on('window:minimize', () => mainWindow.minimize())
ipcMain.on('window:maximize', () => {
  if (mainWindow.isMaximized()) {
    mainWindow.unmaximize()
  } else {
    mainWindow.maximize()
  }
})
ipcMain.on('window:close', () => mainWindow.close())

// File operations
ipcMain.handle('file:new', async () => {
  currentFilePath = null
  return { success: true }
})

ipcMain.handle('file:open', async () => {
  const result = await dialog.showOpenDialog(mainWindow, {
    title: 'เปิดไฟล์',
    filters: [
      { name: 'Text Files', extensions: ['txt', 'md', 'js', 'html', 'css'] },
      { name: 'All Files', extensions: ['*'] }
    ],
    properties: ['openFile']
  })
  
  if (result.canceled) return { canceled: true }
  
  try {
    const filePath = result.filePaths[0]
    const content = fs.readFileSync(filePath, 'utf8')
    currentFilePath = filePath
    return { success: true, filePath, content }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

ipcMain.handle('file:save', async (event, content) => {
  if (!currentFilePath) {
    return await saveFileAs(content)
  }
  
  try {
    fs.writeFileSync(currentFilePath, content, 'utf8')
    return { success: true, filePath: currentFilePath }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

ipcMain.handle('file:save-as', async (event, content) => {
  return await saveFileAs(content)
})

async function saveFileAs(content) {
  const result = await dialog.showSaveDialog(mainWindow, {
    title: 'บันทึกไฟล์',
    defaultPath: 'untitled.txt',
    filters: [
      { name: 'Text Files', extensions: ['txt'] },
      { name: 'Markdown', extensions: ['md'] },
      { name: 'All Files', extensions: ['*'] }
    ]
  })
  
  if (result.canceled) return { canceled: true }
  
  try {
    fs.writeFileSync(result.filePath, content, 'utf8')
    currentFilePath = result.filePath
    return { success: true, filePath: result.filePath }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

ipcMain.handle('app:get-version', () => app.getVersion())

app.whenReady().then(createWindow)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

### preload.js

```javascript
const { contextBridge, ipcRenderer } = require('electron')

// Window API
const windowAPI = {
  minimize: () => ipcRenderer.send('window:minimize'),
  maximize: () => ipcRenderer.send('window:maximize'),
  close: () => ipcRenderer.send('window:close'),
  
  onMaximized: (callback) => {
    const fn = () => callback()
    ipcRenderer.on('window:maximized', fn)
    return () => ipcRenderer.removeListener('window:maximized', fn)
  },
  
  onUnmaximized: (callback) => {
    const fn = () => callback()
    ipcRenderer.on('window:unmaximized', fn)
    return () => ipcRenderer.removeListener('window:unmaximized', fn)
  }
}

// File API
const fileAPI = {
  new: () => ipcRenderer.invoke('file:new'),
  open: () => ipcRenderer.invoke('file:open'),
  save: (content) => {
    if (typeof content !== 'string') throw new TypeError('content must be a string')
    return ipcRenderer.invoke('file:save', content)
  },
  saveAs: (content) => {
    if (typeof content !== 'string') throw new TypeError('content must be a string')
    return ipcRenderer.invoke('file:save-as', content)
  }
}

// App API
const appAPI = {
  getVersion: () => ipcRenderer.invoke('app:get-version')
}

// Expose APIs
contextBridge.exposeInMainWorld('windowAPI', windowAPI)
contextBridge.exposeInMainWorld('fileAPI', fileAPI)
contextBridge.exposeInMainWorld('appAPI', appAPI)

// แสดง log เมื่อ preload โหลดเสร็จ
console.log('Text Editor preload loaded, Electron:', process.versions.electron)
```

### renderer/renderer.js

```javascript
// renderer.js
const editor = document.getElementById('editor')
const titleBar = document.getElementById('title-bar')
const fileNameDisplay = document.getElementById('file-name')
const versionDisplay = document.getElementById('version')
const btnMinimize = document.getElementById('btn-minimize')
const btnMaximize = document.getElementById('btn-maximize')
const btnClose = document.getElementById('btn-close')

let hasUnsavedChanges = false
let currentFileName = 'ไม่มีชื่อ'

// ตั้งค่า window controls
btnMinimize.addEventListener('click', () => window.windowAPI.minimize())
btnMaximize.addEventListener('click', () => window.windowAPI.maximize())
btnClose.addEventListener('click', () => window.windowAPI.close())

// ติดตาม maximize state
const unsubMaximize = window.windowAPI.onMaximized(() => {
  btnMaximize.title = 'Restore'
  btnMaximize.innerHTML = '❐'
})

const unsubUnmaximize = window.windowAPI.onUnmaximized(() => {
  btnMaximize.title = 'Maximize'
  btnMaximize.innerHTML = '□'
})

// ติดตามการเปลี่ยนแปลงใน editor
editor.addEventListener('input', () => {
  hasUnsavedChanges = true
  updateTitle()
})

function updateTitle() {
  const indicator = hasUnsavedChanges ? '• ' : ''
  fileNameDisplay.textContent = `${indicator}${currentFileName}`
}

// Keyboard shortcuts
document.addEventListener('keydown', async (e) => {
  if (e.ctrlKey || e.metaKey) {
    switch (e.key) {
      case 'n':
        e.preventDefault()
        await newFile()
        break
      case 'o':
        e.preventDefault()
        await openFile()
        break
      case 's':
        e.preventDefault()
        if (e.shiftKey) {
          await saveFileAs()
        } else {
          await saveFile()
        }
        break
    }
  }
})

async function newFile() {
  if (hasUnsavedChanges) {
    const confirmed = confirm('มีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก คุณต้องการสร้างไฟล์ใหม่หรือไม่?')
    if (!confirmed) return
  }
  
  await window.fileAPI.new()
  editor.value = ''
  currentFileName = 'ไม่มีชื่อ'
  hasUnsavedChanges = false
  updateTitle()
}

async function openFile() {
  const result = await window.fileAPI.open()
  
  if (result.canceled) return
  
  if (result.success) {
    editor.value = result.content
    currentFileName = result.filePath.split('/').pop()
    hasUnsavedChanges = false
    updateTitle()
  } else {
    alert(`เกิดข้อผิดพลาด: ${result.error}`)
  }
}

async function saveFile() {
  const result = await window.fileAPI.save(editor.value)
  
  if (result && result.success) {
    currentFileName = result.filePath.split('/').pop()
    hasUnsavedChanges = false
    updateTitle()
  } else if (result && !result.canceled) {
    alert(`เกิดข้อผิดพลาด: ${result.error}`)
  }
}

async function saveFileAs() {
  const result = await window.fileAPI.saveAs(editor.value)
  
  if (result && result.success) {
    currentFileName = result.filePath.split('/').pop()
    hasUnsavedChanges = false
    updateTitle()
  }
}

// แสดง version
async function init() {
  const version = await window.appAPI.getVersion()
  versionDisplay.textContent = `v${version}`
}

init()
```

## สรุป

Preload Scripts เป็นส่วนสำคัญของ Electron architecture ที่ช่วยให้แอปพลิเคชันทำงานได้อย่างปลอดภัย โดยมีหลักการสำคัญดังนี้:

1. **ใช้ contextBridge.exposeInMainWorld** สำหรับส่ง APIs ไปยัง renderer
2. **ตรวจสอบ channels เสมอ** ด้วย whitelist ก่อน send/invoke
3. **Validate input** ทุกครั้งก่อนประมวลผล
4. **ไม่ expose APIs ที่กว้างเกินไป** เช่น ipcRenderer, fs, หรือ process โดยตรง
5. **Return cleanup functions** จาก event listeners เพื่อป้องกัน memory leaks
6. **ใช้ TypeScript** ถ้าเป็นไปได้เพื่อ type safety ที่ดีขึ้น

ในบทถัดไปเราจะเรียนรู้เพิ่มเติมเกี่ยวกับ Context Isolation และ Security ของ Electron
