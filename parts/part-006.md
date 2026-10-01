# ตอนที่ 6: การสื่อสารระหว่าง Process ด้วย IPC

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจากศึกษาตอนนี้ คุณจะสามารถ:
- เข้าใจหลักการทำงานของ IPC (Inter-Process Communication) ใน Electron
- ใช้งาน `ipcMain` และ `ipcRenderer` ได้อย่างถูกต้อง
- เลือกใช้รูปแบบ `invoke/handle` หรือ `send/on` ให้เหมาะสมกับงาน
- ใช้ `contextBridge` เพื่อความปลอดภัยสูงสุด
- จัดการการสื่อสารแบบสองทิศทาง (two-way communication)
- จัดการ error ใน IPC ได้อย่างมืออาชีพ

---

## บทนำ: ทำไมต้องมี IPC?

ใน Electron แอปพลิเคชันทำงานใน 2 environment ที่แยกกันอย่างสิ้นเชิง:

```
┌─────────────────────────────────────────────────────┐
│                   Main Process                       │
│  - เข้าถึง Node.js APIs ได้เต็มรูปแบบ              │
│  - จัดการ windows, menus, tray                      │
│  - อ่าน/เขียนไฟล์, เชื่อมต่อ database              │
│  - ควบคุม OS level features                         │
└─────────────────────────┬───────────────────────────┘
                          │  IPC Channel
┌─────────────────────────┴───────────────────────────┐
│                  Renderer Process                    │
│  - แสดงผล HTML/CSS/JavaScript                       │
│  - จัดการ UI interactions                           │
│  - ไม่มีสิทธิ์เข้า Node.js โดยตรง (เมื่อ secure)   │
│  - ต้องส่งข้อความผ่าน IPC ไปหา Main                │
└─────────────────────────────────────────────────────┘
```

IPC (Inter-Process Communication) คือกลไกที่ทำให้ทั้งสอง process สามารถสื่อสารกันได้

---

## 6.1 โครงสร้างโปรเจ็กต์พื้นฐาน

```
my-ipc-app/
├── main.js           # Main Process
├── preload.js        # Preload Script (สะพานเชื่อมที่ปลอดภัย)
├── renderer.js       # Renderer Process Logic
├── index.html        # หน้าตา UI
└── package.json
```

### package.json
```json
{
  "name": "electron-ipc-demo",
  "version": "1.0.0",
  "description": "ตัวอย่างการใช้ IPC ใน Electron",
  "main": "main.js",
  "scripts": {
    "start": "electron ."
  },
  "devDependencies": {
    "electron": "^28.0.0"
  }
}
```

---

## 6.2 รูปแบบ invoke/handle (แนะนำสำหรับงานส่วนใหญ่)

รูปแบบนี้คือการส่งคำขอ (request) และรอรับผลลัพธ์ (response) คล้ายกับ async function call

### ทำไมควรใช้ invoke/handle?
- รองรับ async/await โดยธรรมชาติ
- จัดการ error ได้ง่าย
- โค้ดอ่านง่ายกว่า send/on
- ป้องกัน memory leak เนื่องจาก handler ถูก cleanup อัตโนมัติ

### main.js (Main Process)
```javascript
const { app, BrowserWindow, ipcMain } = require('electron')
const path = require('path')

// สร้าง window หลัก
function createWindow() {
  const mainWindow = new BrowserWindow({
    width: 900,
    height: 700,
    webPreferences: {
      // ระบุ preload script ที่จะโหลดก่อน renderer
      preload: path.join(__dirname, 'preload.js'),
      // ปิด nodeIntegration เพื่อความปลอดภัย
      nodeIntegration: false,
      // เปิด contextIsolation เสมอ
      contextIsolation: true
    }
  })

  mainWindow.loadFile('index.html')
}

// ==========================================
// ลงทะเบียน IPC Handlers ใน Main Process
// ==========================================

// Handler 1: รับข้อความและส่งกลับ (synchronous-like async)
ipcMain.handle('greet-user', async (event, name) => {
  // event คือ IpcMainInvokeEvent ใช้ดู sender ได้
  console.log(`[Main] ได้รับคำขอจาก renderer: greet-user, ชื่อ: ${name}`)
  
  // จำลองการทำงานที่ใช้เวลา เช่น อ่าน database
  await new Promise(resolve => setTimeout(resolve, 500))
  
  return `สวัสดี ${name}! ยินดีต้อนรับสู่ Electron IPC`
})

// Handler 2: คำนวณค่า
ipcMain.handle('calculate', async (event, { operation, a, b }) => {
  console.log(`[Main] คำนวณ: ${a} ${operation} ${b}`)
  
  switch (operation) {
    case 'add': return { result: a + b, message: 'บวกสำเร็จ' }
    case 'subtract': return { result: a - b, message: 'ลบสำเร็จ' }
    case 'multiply': return { result: a * b, message: 'คูณสำเร็จ' }
    case 'divide':
      if (b === 0) throw new Error('ไม่สามารถหารด้วยศูนย์ได้!')
      return { result: a / b, message: 'หารสำเร็จ' }
    default:
      throw new Error(`ไม่รู้จักการดำเนินการ: ${operation}`)
  }
})

// Handler 3: รับข้อมูลหลายชิ้น
ipcMain.handle('process-data', async (event, dataArray) => {
  console.log(`[Main] ประมวลผลข้อมูล ${dataArray.length} รายการ`)
  
  // จำลองการประมวลผล
  const processed = dataArray.map((item, index) => ({
    id: index + 1,
    original: item,
    processed: item.toString().toUpperCase(),
    timestamp: new Date().toISOString()
  }))
  
  return {
    success: true,
    count: processed.length,
    data: processed
  }
})

app.whenReady().then(() => {
  createWindow()
  
  app.on('activate', () => {
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

### preload.js (Preload Script - สะพานเชื่อมที่ปลอดภัย)
```javascript
const { contextBridge, ipcRenderer } = require('electron')

// ==========================================
// contextBridge.exposeInMainWorld
// เปิดเผย API ที่ปลอดภัยให้ renderer ใช้
// ==========================================
contextBridge.exposeInMainWorld('electronAPI', {
  
  // Wrapper สำหรับ invoke/handle
  greetUser: (name) => ipcRenderer.invoke('greet-user', name),
  
  calculate: (operation, a, b) => 
    ipcRenderer.invoke('calculate', { operation, a, b }),
  
  processData: (dataArray) => 
    ipcRenderer.invoke('process-data', dataArray),
})

// หมายเหตุ: อย่า expose ipcRenderer โดยตรง!
// ❌ ไม่ควรทำ:
// contextBridge.exposeInMainWorld('ipcRenderer', ipcRenderer)
// 
// ✅ ควรทำ: เปิดเผยเฉพาะ method ที่ต้องการ
```

### renderer.js (Renderer Process)
```javascript
// ตอนนี้ renderer เข้าถึง electronAPI ผ่าน window.electronAPI
// ซึ่ง contextBridge สร้างมาให้อย่างปลอดภัย

// =====================
// ตัวอย่างที่ 1: Greet
// =====================
async function greetUser() {
  const nameInput = document.getElementById('userName')
  const resultDiv = document.getElementById('greetResult')
  
  const name = nameInput.value.trim()
  if (!name) {
    resultDiv.textContent = 'กรุณากรอกชื่อก่อน'
    return
  }
  
  try {
    resultDiv.textContent = 'กำลังส่งคำขอ...'
    
    // เรียก IPC invoke ผ่าน electronAPI
    const message = await window.electronAPI.greetUser(name)
    
    resultDiv.textContent = message
    resultDiv.style.color = 'green'
  } catch (error) {
    resultDiv.textContent = `เกิดข้อผิดพลาด: ${error.message}`
    resultDiv.style.color = 'red'
  }
}

// ==========================
// ตัวอย่างที่ 2: Calculator
// ==========================
async function calculate() {
  const aInput = document.getElementById('numA')
  const bInput = document.getElementById('numB')
  const opSelect = document.getElementById('operation')
  const resultDiv = document.getElementById('calcResult')
  
  const a = parseFloat(aInput.value)
  const b = parseFloat(bInput.value)
  const operation = opSelect.value
  
  if (isNaN(a) || isNaN(b)) {
    resultDiv.textContent = 'กรุณากรอกตัวเลขให้ถูกต้อง'
    return
  }
  
  try {
    const result = await window.electronAPI.calculate(operation, a, b)
    resultDiv.textContent = `ผลลัพธ์: ${result.result} (${result.message})`
    resultDiv.style.color = 'green'
  } catch (error) {
    // จัดการ error ที่ throw มาจาก main process
    resultDiv.textContent = `ข้อผิดพลาด: ${error.message}`
    resultDiv.style.color = 'red'
  }
}

// ผูก event กับปุ่ม
document.getElementById('greetBtn').addEventListener('click', greetUser)
document.getElementById('calcBtn').addEventListener('click', calculate)
```

---

## 6.3 รูปแบบ send/on (สำหรับการแจ้งเตือนแบบ one-way)

รูปแบบนี้เหมาะสำหรับ:
- การแจ้งเตือนจาก main ไปหา renderer
- การส่ง event แบบ fire-and-forget
- การสตรีมข้อมูลอย่างต่อเนื่อง

### main.js - ส่วนที่เพิ่มเข้ามา
```javascript
const { app, BrowserWindow, ipcMain } = require('electron')
const path = require('path')

let mainWindow

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 900,
    height: 700,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  mainWindow.loadFile('index.html')
}

// ==========================================
// รูปแบบ send/on: รับข้อความจาก renderer
// ==========================================

// รับข้อความแบบ one-way (ไม่ต้องส่งค่ากลับ)
ipcMain.on('log-message', (event, { level, message }) => {
  const timestamp = new Date().toISOString()
  console.log(`[${timestamp}] [${level.toUpperCase()}] ${message}`)
  
  // บันทึก log ลงไฟล์ (ตัวอย่าง)
  // fs.appendFileSync('app.log', `[${timestamp}] [${level}] ${message}\n`)
})

// รับคำขอ แล้วตอบกลับด้วย send
ipcMain.on('get-app-info', (event) => {
  const info = {
    version: app.getVersion(),
    platform: process.platform,
    electronVersion: process.versions.electron,
    nodeVersion: process.versions.node
  }
  
  // ตอบกลับไปที่ window ที่ส่งมา
  event.reply('app-info-response', info)
  
  // หรือใช้ event.sender.send ก็ได้
  // event.sender.send('app-info-response', info)
})

// ==========================================
// ส่ง event จาก Main ไปหา Renderer
// ==========================================

// จำลองการส่ง progress update
function simulateProgress() {
  let progress = 0
  const interval = setInterval(() => {
    progress += 10
    
    // ส่งไปที่ทุก window
    if (mainWindow && !mainWindow.isDestroyed()) {
      mainWindow.webContents.send('download-progress', {
        percent: progress,
        loaded: progress * 1024,
        total: 100 * 1024
      })
    }
    
    if (progress >= 100) {
      clearInterval(interval)
      if (mainWindow && !mainWindow.isDestroyed()) {
        mainWindow.webContents.send('download-complete', {
          filename: 'example.zip',
          size: 100 * 1024
        })
      }
    }
  }, 300)
}

ipcMain.handle('start-download', async () => {
  simulateProgress()
  return { started: true, message: 'เริ่มดาวน์โหลดแล้ว' }
})

app.whenReady().then(createWindow)
```

### preload.js - อัพเดทสำหรับ send/on
```javascript
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('electronAPI', {
  
  // ==========================================
  // invoke/handle pattern
  // ==========================================
  greetUser: (name) => ipcRenderer.invoke('greet-user', name),
  calculate: (op, a, b) => ipcRenderer.invoke('calculate', { operation: op, a, b }),
  startDownload: () => ipcRenderer.invoke('start-download'),
  
  // ==========================================
  // send/on pattern - ส่งข้อความแบบ one-way
  // ==========================================
  logMessage: (level, message) => 
    ipcRenderer.send('log-message', { level, message }),
  
  getAppInfo: () => ipcRenderer.send('get-app-info'),
  
  // ==========================================
  // การรับ event จาก Main Process
  // ==========================================
  
  // รับ app info response
  onAppInfo: (callback) => {
    ipcRenderer.on('app-info-response', (event, info) => callback(info))
  },
  
  // รับ progress update
  onDownloadProgress: (callback) => {
    ipcRenderer.on('download-progress', (event, data) => callback(data))
  },
  
  // รับเมื่อดาวน์โหลดเสร็จ
  onDownloadComplete: (callback) => {
    ipcRenderer.on('download-complete', (event, data) => callback(data))
  },
  
  // ==========================================
  // สำคัญ: ต้องมีวิธี cleanup listeners
  // เพื่อป้องกัน memory leak
  // ==========================================
  removeAllListeners: (channel) => {
    ipcRenderer.removeAllListeners(channel)
  }
})
```

### renderer.js - ใช้ send/on pattern
```javascript
// =====================
// Send/On Pattern Usage
// =====================

// รับ app info
window.electronAPI.getAppInfo()

window.electronAPI.onAppInfo((info) => {
  const infoDiv = document.getElementById('appInfo')
  infoDiv.innerHTML = `
    <p>Version: ${info.version}</p>
    <p>Platform: ${info.platform}</p>
    <p>Electron: ${info.electronVersion}</p>
    <p>Node.js: ${info.nodeVersion}</p>
  `
})

// ===========================
// Download Progress ตัวอย่าง
// ===========================
const progressBar = document.getElementById('progressBar')
const progressText = document.getElementById('progressText')
const startBtn = document.getElementById('startDownloadBtn')

// ลงทะเบียนรับ progress updates
window.electronAPI.onDownloadProgress((data) => {
  const percent = data.percent
  progressBar.style.width = `${percent}%`
  progressText.textContent = `${percent}% (${formatBytes(data.loaded)} / ${formatBytes(data.total)})`
})

window.electronAPI.onDownloadComplete((data) => {
  progressText.textContent = `ดาวน์โหลดเสร็จสมบูรณ์: ${data.filename} (${formatBytes(data.size)})`
  progressBar.style.backgroundColor = 'green'
  startBtn.disabled = false
})

startBtn.addEventListener('click', async () => {
  startBtn.disabled = true
  progressBar.style.width = '0%'
  progressBar.style.backgroundColor = '#007bff'
  
  const result = await window.electronAPI.startDownload()
  console.log(result.message)
})

// ฟังก์ชันช่วยแปลงขนาดไฟล์
function formatBytes(bytes) {
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB'
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB'
}

// Cleanup เมื่อหน้าถูกปิด (ป้องกัน memory leak)
window.addEventListener('beforeunload', () => {
  window.electronAPI.removeAllListeners('download-progress')
  window.electronAPI.removeAllListeners('download-complete')
  window.electronAPI.removeAllListeners('app-info-response')
})
```

---

## 6.4 contextBridge Security - ความปลอดภัยระดับสูง

`contextBridge` เป็น API ที่สำคัญมากสำหรับความปลอดภัย มันสร้าง isolated bridge ระหว่าง preload และ renderer

### ทำความเข้าใจ contextBridge

```javascript
// preload.js

const { contextBridge, ipcRenderer } = require('electron')

// ==========================================
// ✅ วิธีที่ถูกต้อง: ห่อ API ไว้ใน wrapper
// ==========================================
contextBridge.exposeInMainWorld('safeAPI', {
  
  // เปิดเผยเฉพาะ method ที่ต้องการ
  readFile: (filePath) => ipcRenderer.invoke('read-file', filePath),
  
  // validate input ก่อนส่ง
  sendMessage: (message) => {
    // ตรวจสอบ type
    if (typeof message !== 'string') {
      throw new TypeError('message ต้องเป็น string')
    }
    // จำกัดความยาว
    if (message.length > 1000) {
      throw new RangeError('message ยาวเกินไป (สูงสุด 1000 ตัวอักษร)')
    }
    return ipcRenderer.invoke('send-message', message)
  },
  
  // รับ event แต่ห่อ callback เพื่อความปลอดภัย
  onUpdate: (callback) => {
    if (typeof callback !== 'function') {
      throw new TypeError('callback ต้องเป็น function')
    }
    
    // สร้าง wrapper เพื่อกรองข้อมูลที่รับมา
    const wrappedCallback = (event, data) => {
      // อย่า pass event object ไปให้ renderer
      // เพราะ event มี sender ที่อาจเป็นอันตราย
      callback(data) // ส่งแค่ data
    }
    
    ipcRenderer.on('app-update', wrappedCallback)
    
    // คืนค่า function สำหรับ cleanup
    return () => {
      ipcRenderer.removeListener('app-update', wrappedCallback)
    }
  }
})

// ==========================================
// ❌ วิธีที่ไม่ปลอดภัย: อย่าทำแบบนี้
// ==========================================

// 1. อย่า expose ipcRenderer โดยตรง
// contextBridge.exposeInMainWorld('ipc', ipcRenderer)

// 2. อย่า expose require โดยตรง
// contextBridge.exposeInMainWorld('require', require)

// 3. อย่า pass event object ไปให้ renderer
// ipcRenderer.on('channel', (event, data) => callback(event, data))

// 4. อย่า expose function ที่รับ channel name เป็น parameter โดยไม่ whitelist
// sendToChannel: (channel, data) => ipcRenderer.send(channel, data) // อันตราย!
```

### วิธีสร้าง Type-Safe API ที่ดี

```javascript
// preload.js - ตัวอย่างที่ดีมากสำหรับ production

const { contextBridge, ipcRenderer } = require('electron')

// รายชื่อ channels ที่อนุญาต (whitelist)
const ALLOWED_CHANNELS = {
  invoke: [
    'read-file',
    'write-file', 
    'get-system-info',
    'open-dialog',
    'calculate'
  ],
  send: [
    'log-message',
    'app-ready'
  ],
  receive: [
    'update-available',
    'download-progress',
    'download-complete'
  ]
}

// สร้าง safe IPC bridge
const ipcBridge = {
  // Invoke ที่ปลอดภัย - ตรวจสอบ channel ก่อน
  invoke: (channel, ...args) => {
    if (ALLOWED_CHANNELS.invoke.includes(channel)) {
      return ipcRenderer.invoke(channel, ...args)
    }
    return Promise.reject(new Error(`Channel "${channel}" ไม่ได้รับอนุญาต`))
  },
  
  // Send ที่ปลอดภัย
  send: (channel, data) => {
    if (ALLOWED_CHANNELS.send.includes(channel)) {
      ipcRenderer.send(channel, data)
    } else {
      console.warn(`Channel "${channel}" ไม่ได้รับอนุญาตสำหรับ send`)
    }
  },
  
  // รับ event ที่ปลอดภัย - คืนค่า cleanup function
  on: (channel, callback) => {
    if (!ALLOWED_CHANNELS.receive.includes(channel)) {
      console.warn(`Channel "${channel}" ไม่ได้รับอนุญาตสำหรับ receive`)
      return () => {}
    }
    
    const subscription = (event, ...args) => callback(...args)
    ipcRenderer.on(channel, subscription)
    
    // คืน cleanup function
    return () => {
      ipcRenderer.removeListener(channel, subscription)
    }
  },
  
  // ลบ listeners ทั้งหมดของ channel
  removeAllListeners: (channel) => {
    if (ALLOWED_CHANNELS.receive.includes(channel)) {
      ipcRenderer.removeAllListeners(channel)
    }
  }
}

contextBridge.exposeInMainWorld('electronAPI', ipcBridge)
```

---

## 6.5 การสื่อสารแบบสองทิศทาง (Two-Way Communication)

### ตัวอย่างระบบ Chat แบบ Real-Time

```javascript
// main.js - ระบบ messaging

const { ipcMain, BrowserWindow } = require('electron')

// เก็บ history ข้อความ
const messageHistory = []

// Handler: รับข้อความจาก renderer
ipcMain.handle('send-chat-message', async (event, message) => {
  const senderWindow = BrowserWindow.fromWebContents(event.sender)
  
  const newMessage = {
    id: Date.now(),
    text: message,
    sender: `Window ${senderWindow.id}`,
    timestamp: new Date().toLocaleTimeString('th-TH')
  }
  
  messageHistory.push(newMessage)
  
  // กระจายข้อความไปยังทุก window
  BrowserWindow.getAllWindows().forEach(win => {
    if (!win.isDestroyed()) {
      win.webContents.send('new-message', newMessage)
    }
  })
  
  return { success: true, messageId: newMessage.id }
})

// Handler: ดึง history
ipcMain.handle('get-message-history', async () => {
  return messageHistory
})

// Handler: ลบข้อความ
ipcMain.handle('delete-message', async (event, messageId) => {
  const index = messageHistory.findIndex(m => m.id === messageId)
  if (index === -1) {
    throw new Error(`ไม่พบข้อความ ID: ${messageId}`)
  }
  
  messageHistory.splice(index, 1)
  
  // แจ้งทุก window ว่าข้อความถูกลบ
  BrowserWindow.getAllWindows().forEach(win => {
    if (!win.isDestroyed()) {
      win.webContents.send('message-deleted', messageId)
    }
  })
  
  return { success: true }
})
```

```javascript
// preload.js - chat API
contextBridge.exposeInMainWorld('chatAPI', {
  sendMessage: (text) => ipcRenderer.invoke('send-chat-message', text),
  getHistory: () => ipcRenderer.invoke('get-message-history'),
  deleteMessage: (id) => ipcRenderer.invoke('delete-message', id),
  
  onNewMessage: (callback) => {
    const handler = (event, message) => callback(message)
    ipcRenderer.on('new-message', handler)
    return () => ipcRenderer.removeListener('new-message', handler)
  },
  
  onMessageDeleted: (callback) => {
    const handler = (event, id) => callback(id)
    ipcRenderer.on('message-deleted', handler)
    return () => ipcRenderer.removeListener('message-deleted', handler)
  }
})
```

```javascript
// renderer.js - Chat UI Logic
class ChatApp {
  constructor() {
    this.messages = []
    this.cleanupFunctions = []
    this.init()
  }
  
  async init() {
    // โหลด history เมื่อเริ่มต้น
    this.messages = await window.chatAPI.getHistory()
    this.renderMessages()
    
    // ลงทะเบียนรับข้อความใหม่
    const unsubscribeNew = window.chatAPI.onNewMessage((message) => {
      this.messages.push(message)
      this.renderMessages()
      this.scrollToBottom()
    })
    
    // ลงทะเบียนรับการแจ้งเตือนลบ
    const unsubscribeDelete = window.chatAPI.onMessageDeleted((id) => {
      this.messages = this.messages.filter(m => m.id !== id)
      this.renderMessages()
    })
    
    // เก็บ cleanup functions
    this.cleanupFunctions.push(unsubscribeNew, unsubscribeDelete)
    
    // ผูก event กับ UI
    this.bindEvents()
  }
  
  bindEvents() {
    const sendBtn = document.getElementById('sendBtn')
    const messageInput = document.getElementById('messageInput')
    
    sendBtn.addEventListener('click', () => this.sendMessage())
    
    messageInput.addEventListener('keypress', (e) => {
      if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault()
        this.sendMessage()
      }
    })
  }
  
  async sendMessage() {
    const input = document.getElementById('messageInput')
    const text = input.value.trim()
    
    if (!text) return
    
    input.value = ''
    input.disabled = true
    
    try {
      const result = await window.chatAPI.sendMessage(text)
      console.log(`ส่งข้อความสำเร็จ: ID ${result.messageId}`)
    } catch (error) {
      console.error('ส่งข้อความล้มเหลว:', error)
      alert(`ส่งข้อความล้มเหลว: ${error.message}`)
    } finally {
      input.disabled = false
      input.focus()
    }
  }
  
  renderMessages() {
    const container = document.getElementById('messages')
    container.innerHTML = this.messages.map(msg => `
      <div class="message" data-id="${msg.id}">
        <span class="sender">${msg.sender}</span>
        <span class="time">${msg.timestamp}</span>
        <p class="text">${this.escapeHtml(msg.text)}</p>
        <button class="delete-btn" onclick="chat.deleteMessage(${msg.id})">ลบ</button>
      </div>
    `).join('')
  }
  
  async deleteMessage(id) {
    try {
      await window.chatAPI.deleteMessage(id)
    } catch (error) {
      alert(`ลบข้อความล้มเหลว: ${error.message}`)
    }
  }
  
  scrollToBottom() {
    const container = document.getElementById('messages')
    container.scrollTop = container.scrollHeight
  }
  
  // ป้องกัน XSS
  escapeHtml(text) {
    const div = document.createElement('div')
    div.appendChild(document.createTextNode(text))
    return div.innerHTML
  }
  
  // Cleanup เมื่อ component ถูกทำลาย
  destroy() {
    this.cleanupFunctions.forEach(fn => fn())
  }
}

// สร้าง instance
const chat = new ChatApp()

// Cleanup เมื่อปิดหน้า
window.addEventListener('beforeunload', () => chat.destroy())
```

---

## 6.6 การจัดการ Error ใน IPC

### Error Handling ที่สมบูรณ์

```javascript
// main.js - error handling ที่ดี
const { ipcMain } = require('electron')

// ==========================================
// Custom Error Classes
// ==========================================
class DatabaseError extends Error {
  constructor(message, code) {
    super(message)
    this.name = 'DatabaseError'
    this.code = code
  }
}

class ValidationError extends Error {
  constructor(message, field) {
    super(message)
    this.name = 'ValidationError'
    this.field = field
  }
}

class NotFoundError extends Error {
  constructor(resource, id) {
    super(`ไม่พบ ${resource} ID: ${id}`)
    this.name = 'NotFoundError'
    this.resource = resource
    this.id = id
  }
}

// ==========================================
// Database Simulation
// ==========================================
const db = {
  users: [
    { id: 1, name: 'สมชาย', email: 'somchai@example.com', age: 30 },
    { id: 2, name: 'สมหญิง', email: 'somying@example.com', age: 25 },
  ]
}

// Handler พร้อม error handling เต็มรูปแบบ
ipcMain.handle('get-user', async (event, userId) => {
  // Validate input
  if (!userId || typeof userId !== 'number') {
    throw new ValidationError('userId ต้องเป็นตัวเลข', 'userId')
  }
  
  // จำลองการค้นหาใน database
  const user = db.users.find(u => u.id === userId)
  
  if (!user) {
    throw new NotFoundError('user', userId)
  }
  
  return user
})

ipcMain.handle('create-user', async (event, userData) => {
  // Validate ข้อมูล
  if (!userData.name || userData.name.length < 2) {
    throw new ValidationError('ชื่อต้องมีอย่างน้อย 2 ตัวอักษร', 'name')
  }
  
  if (!userData.email || !userData.email.includes('@')) {
    throw new ValidationError('รูปแบบ email ไม่ถูกต้อง', 'email')
  }
  
  if (!userData.age || userData.age < 0 || userData.age > 150) {
    throw new ValidationError('อายุต้องอยู่ระหว่าง 0-150', 'age')
  }
  
  // ตรวจสอบ email ซ้ำ
  const existingUser = db.users.find(u => u.email === userData.email)
  if (existingUser) {
    throw new DatabaseError(`Email ${userData.email} มีอยู่ในระบบแล้ว`, 'DUPLICATE_EMAIL')
  }
  
  // สร้าง user ใหม่
  const newUser = {
    id: Math.max(...db.users.map(u => u.id)) + 1,
    ...userData
  }
  
  db.users.push(newUser)
  
  return newUser
})

// Handler ที่ wraps error อย่างดี
ipcMain.handle('safe-operation', async (event, params) => {
  try {
    // ทำงานที่อาจ throw error
    const result = await riskyOperation(params)
    return { success: true, data: result }
  } catch (error) {
    // Log error ใน main process
    console.error('[IPC Error]', error)
    
    // Rethrow เพื่อให้ renderer จัดการ
    // Electron จะแปลง Error object เป็น serializable format อัตโนมัติ
    throw error
  }
})

async function riskyOperation(params) {
  if (!params) throw new Error('params ไม่สามารถเป็น null ได้')
  return { result: 'สำเร็จ' }
}
```

```javascript
// renderer.js - จัดการ error จาก IPC
class ErrorHandler {
  static async safeInvoke(fn, fallback = null) {
    try {
      return await fn()
    } catch (error) {
      console.error('IPC Error:', error)
      return fallback
    }
  }
  
  static displayError(error, container) {
    let message = error.message
    let type = 'error'
    
    // จัดการตาม error type
    if (error.name === 'ValidationError') {
      message = `ข้อมูลไม่ถูกต้อง (${error.field}): ${error.message}`
      type = 'warning'
    } else if (error.name === 'NotFoundError') {
      message = `ไม่พบข้อมูล: ${error.message}`
      type = 'info'
    } else if (error.name === 'DatabaseError') {
      message = `ข้อผิดพลาดฐานข้อมูล: ${error.message}`
      type = 'error'
    }
    
    container.innerHTML = `
      <div class="alert alert-${type}">
        <strong>${error.name || 'Error'}:</strong> ${message}
      </div>
    `
  }
}

// ตัวอย่างการใช้งาน
async function loadUser(userId) {
  const resultDiv = document.getElementById('userResult')
  
  try {
    const user = await window.electronAPI.invoke('get-user', userId)
    resultDiv.innerHTML = `
      <div class="user-card">
        <h3>${user.name}</h3>
        <p>Email: ${user.email}</p>
        <p>อายุ: ${user.age} ปี</p>
      </div>
    `
  } catch (error) {
    ErrorHandler.displayError(error, resultDiv)
  }
}

async function createUser(formData) {
  const resultDiv = document.getElementById('createResult')
  
  try {
    const newUser = await window.electronAPI.invoke('create-user', formData)
    resultDiv.innerHTML = `
      <div class="success">
        สร้างผู้ใช้ใหม่สำเร็จ! ID: ${newUser.id}, ชื่อ: ${newUser.name}
      </div>
    `
  } catch (error) {
    // จัดการ error ต่างๆ
    if (error.name === 'ValidationError') {
      // Highlight field ที่มี error
      const field = document.getElementById(error.field)
      if (field) {
        field.classList.add('is-invalid')
        field.focus()
      }
    }
    
    ErrorHandler.displayError(error, resultDiv)
  }
}
```

---

## 6.7 IPC Performance Best Practices

### หลีกเลี่ยงการส่งข้อมูลขนาดใหญ่

```javascript
// main.js - การจัดการข้อมูลขนาดใหญ่อย่างมีประสิทธิภาพ
const { ipcMain } = require('electron')
const fs = require('fs')

// ❌ ไม่ดี: ส่งข้อมูลทั้งหมดในครั้งเดียว
ipcMain.handle('load-large-data-bad', async () => {
  const hugeData = await readMillionRecords() // ข้อมูลหลาย MB
  return hugeData // ส่งทั้งก้อนผ่าน IPC - ช้ามาก!
})

// ✅ ดีกว่า: ใช้ pagination
ipcMain.handle('load-data-page', async (event, { page, pageSize = 50 }) => {
  const allData = await readAllRecords()
  const start = (page - 1) * pageSize
  const end = start + pageSize
  
  return {
    data: allData.slice(start, end),
    total: allData.length,
    page,
    pageSize,
    totalPages: Math.ceil(allData.length / pageSize)
  }
})

// ✅ ดีที่สุดสำหรับข้อมูลมาก: streaming ผ่าน events
ipcMain.handle('start-streaming', async (event) => {
  const senderWindow = require('electron').BrowserWindow.fromWebContents(event.sender)
  
  // อ่านข้อมูลทีละ chunk แล้วส่งผ่าน event
  const stream = fs.createReadStream('large-file.json', { encoding: 'utf8' })
  let buffer = ''
  
  stream.on('data', (chunk) => {
    buffer += chunk
    
    // ส่ง chunk ไปที่ renderer
    senderWindow.webContents.send('data-chunk', {
      chunk,
      bytesRead: Buffer.byteLength(buffer)
    })
  })
  
  stream.on('end', () => {
    senderWindow.webContents.send('data-complete', {
      totalBytes: Buffer.byteLength(buffer)
    })
  })
  
  stream.on('error', (error) => {
    senderWindow.webContents.send('data-error', error.message)
  })
  
  return { started: true }
})
```

### การใช้ SharedArrayBuffer สำหรับข้อมูลขนาดใหญ่

```javascript
// main.js - ใช้ SharedArrayBuffer
ipcMain.handle('share-buffer', async (event) => {
  // สร้าง shared memory buffer
  const sharedBuffer = new SharedArrayBuffer(1024 * 1024) // 1MB
  const data = new Uint8Array(sharedBuffer)
  
  // เติมข้อมูล
  for (let i = 0; i < data.length; i++) {
    data[i] = i % 256
  }
  
  // ส่ง buffer reference (ไม่ copy ข้อมูล)
  return sharedBuffer
})

// หมายเหตุ: ต้องเปิด SharedArrayBuffer ใน webPreferences
// webPreferences: {
//   enableSharedArrayBuffer: true
// }
```

---

## 6.8 IPC ระหว่าง Renderer Windows

บางครั้งต้องการให้ renderer windows สื่อสารกันโดยตรง:

```javascript
// main.js - Bridge สำหรับ renderer-to-renderer communication
const { ipcMain, BrowserWindow } = require('electron')

// แต่ละ window ลงทะเบียนชื่อตัวเอง
const windowRegistry = new Map()

ipcMain.handle('register-window', async (event, windowName) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  windowRegistry.set(windowName, win.id)
  console.log(`Window "${windowName}" ลงทะเบียนด้วย ID: ${win.id}`)
  return { success: true, windowId: win.id }
})

// ส่งข้อความไปยัง window อื่นผ่าน main
ipcMain.handle('message-to-window', async (event, { targetWindow, message }) => {
  const targetId = windowRegistry.get(targetWindow)
  
  if (!targetId) {
    throw new Error(`ไม่พบ window: ${targetWindow}`)
  }
  
  const targetWin = BrowserWindow.fromId(targetId)
  
  if (!targetWin || targetWin.isDestroyed()) {
    windowRegistry.delete(targetWindow)
    throw new Error(`Window "${targetWindow}" ไม่ได้เปิดอยู่`)
  }
  
  // ส่งข้อความไปยัง target window
  targetWin.webContents.send('window-message', {
    from: `Window ${BrowserWindow.fromWebContents(event.sender).id}`,
    message,
    timestamp: new Date().toISOString()
  })
  
  return { success: true }
})
```

---

## 6.9 Debugging IPC

### เพิ่ม IPC Logger สำหรับ Development

```javascript
// main.js - IPC Logger Middleware

function createIPCLogger() {
  const originalHandle = require('electron').ipcMain.handle.bind(require('electron').ipcMain)
  
  require('electron').ipcMain.handle = function(channel, listener) {
    return originalHandle(channel, async (event, ...args) => {
      const startTime = Date.now()
      console.log(`[IPC ▶] Channel: "${channel}", Args:`, args)
      
      try {
        const result = await listener(event, ...args)
        const duration = Date.now() - startTime
        console.log(`[IPC ✓] Channel: "${channel}", Duration: ${duration}ms, Result:`, result)
        return result
      } catch (error) {
        const duration = Date.now() - startTime
        console.error(`[IPC ✗] Channel: "${channel}", Duration: ${duration}ms, Error:`, error)
        throw error
      }
    })
  }
}

// เรียกใช้ในโหมด development เท่านั้น
if (process.env.NODE_ENV === 'development') {
  createIPCLogger()
}
```

---

## 6.10 index.html สมบูรณ์

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>IPC Communication Demo</title>
  <style>
    body {
      font-family: 'Sarabun', sans-serif;
      max-width: 800px;
      margin: 0 auto;
      padding: 20px;
      background: #f5f5f5;
    }
    
    .section {
      background: white;
      border-radius: 8px;
      padding: 20px;
      margin-bottom: 20px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    
    h2 { color: #333; border-bottom: 2px solid #007bff; padding-bottom: 10px; }
    
    input, select {
      padding: 8px 12px;
      border: 1px solid #ddd;
      border-radius: 4px;
      margin-right: 10px;
      font-size: 14px;
    }
    
    input.is-invalid { border-color: red; }
    
    button {
      padding: 8px 16px;
      background: #007bff;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 14px;
    }
    
    button:hover { background: #0056b3; }
    button:disabled { background: #6c757d; cursor: not-allowed; }
    
    .result {
      margin-top: 10px;
      padding: 10px;
      background: #f8f9fa;
      border-radius: 4px;
      min-height: 40px;
    }
    
    .alert { padding: 10px; border-radius: 4px; }
    .alert-error { background: #f8d7da; color: #721c24; }
    .alert-warning { background: #fff3cd; color: #856404; }
    .alert-info { background: #d1ecf1; color: #0c5460; }
    .alert-success { background: #d4edda; color: #155724; }
    
    .progress-bar-container {
      width: 100%;
      background: #e9ecef;
      border-radius: 4px;
      height: 20px;
      overflow: hidden;
    }
    
    #progressBar {
      height: 100%;
      background: #007bff;
      transition: width 0.3s;
      width: 0%;
    }
  </style>
</head>
<body>
  <h1>IPC Communication ใน Electron</h1>
  
  <!-- ส่วน Greet -->
  <div class="section">
    <h2>1. invoke/handle Pattern</h2>
    <input type="text" id="userName" placeholder="ชื่อของคุณ">
    <button id="greetBtn">ทักทาย</button>
    <div class="result" id="greetResult"></div>
  </div>
  
  <!-- ส่วน Calculator -->
  <div class="section">
    <h2>2. Calculator</h2>
    <input type="number" id="numA" value="10" style="width: 80px">
    <select id="operation">
      <option value="add">+</option>
      <option value="subtract">-</option>
      <option value="multiply">×</option>
      <option value="divide">÷</option>
    </select>
    <input type="number" id="numB" value="5" style="width: 80px">
    <button id="calcBtn">คำนวณ</button>
    <div class="result" id="calcResult"></div>
  </div>
  
  <!-- ส่วน Download Progress -->
  <div class="section">
    <h2>3. send/on Pattern - Download Progress</h2>
    <button id="startDownloadBtn">เริ่มดาวน์โหลด</button>
    <div class="progress-bar-container" style="margin-top: 10px;">
      <div id="progressBar"></div>
    </div>
    <div id="progressText" style="margin-top: 5px; font-size: 14px;"></div>
  </div>
  
  <!-- ส่วน App Info -->
  <div class="section">
    <h2>4. ข้อมูล Application</h2>
    <div id="appInfo">กำลังโหลด...</div>
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

---

## สรุปบทเรียน

### เมื่อไหร่ควรใช้อะไร?

| รูปแบบ | เมื่อไหร่ใช้ | ตัวอย่าง |
|--------|------------|---------|
| `invoke/handle` | ต้องการผลลัพธ์กลับ, async operations | อ่านไฟล์, query database |
| `send/on` | fire-and-forget, log, one-way events | บันทึก log, เปิด window |
| `webContents.send` | Main → Renderer notification | progress update, app events |

### สิ่งที่ต้องจำเสมอ

1. **ใช้ contextBridge เสมอ** - อย่า expose ipcRenderer โดยตรง
2. **Validate input** ทั้งฝั่ง renderer และ main
3. **Cleanup listeners** เพื่อป้องกัน memory leak
4. **จัดการ error** ให้ครบถ้วนทั้งสองฝั่ง
5. **อย่าส่งข้อมูลขนาดใหญ่** ใน IPC ครั้งเดียว ใช้ pagination หรือ streaming

### Checklist ก่อน Production

- [ ] `nodeIntegration: false`
- [ ] `contextIsolation: true`
- [ ] ใช้ `contextBridge.exposeInMainWorld`
- [ ] Whitelist channels ที่อนุญาต
- [ ] Validate input ทุก channel
- [ ] มี cleanup function สำหรับ listeners
- [ ] Error handling ครบทั้งสองฝั่ง
- [ ] ไม่ส่ง sensitive data ที่ไม่จำเป็นผ่าน IPC
