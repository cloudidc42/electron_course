# ตอนที่ 14: Clipboard Operations

## Clipboard Module ใน Electron

`clipboard` module ให้ความสามารถในการอ่านและเขียนข้อมูลลง system clipboard ของ macOS, Windows และ Linux รองรับหลายรูปแบบข้อมูล ได้แก่ text, HTML, images, และ RTF

```javascript
const { clipboard } = require('electron')
```

## clipboard.readText / writeText

การอ่านและเขียนข้อความลง clipboard เป็นการใช้งานพื้นฐานที่สุด

### รูปแบบพื้นฐาน

```javascript
// main.js
const { clipboard } = require('electron')

// เขียน text ลง clipboard
clipboard.writeText('Hello, World!')

// อ่าน text จาก clipboard
const text = clipboard.readText()
console.log(text) // 'Hello, World!'

// เขียน text สำหรับ selection clipboard (Linux only)
if (process.platform === 'linux') {
  clipboard.writeText('Selected text', 'selection')
  const selectedText = clipboard.readText('selection')
}
```

### ตัวอย่างการ expose ผ่าน preload

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('clipboardAPI', {
  readText: () => ipcRenderer.invoke('clipboard:read-text'),
  writeText: (text) => ipcRenderer.invoke('clipboard:write-text', text),
  readHTML: () => ipcRenderer.invoke('clipboard:read-html'),
  writeHTML: (html) => ipcRenderer.invoke('clipboard:write-html', html),
  readImage: () => ipcRenderer.invoke('clipboard:read-image'),
  writeImage: (dataURL) => ipcRenderer.invoke('clipboard:write-image', dataURL),
  clear: () => ipcRenderer.invoke('clipboard:clear'),
  hasContent: (format) => ipcRenderer.invoke('clipboard:has-content', format),
  getFormats: () => ipcRenderer.invoke('clipboard:get-formats'),
  onChanged: (callback) => {
    const fn = (_e, data) => callback(data)
    ipcRenderer.on('clipboard:changed', fn)
    return () => ipcRenderer.removeListener('clipboard:changed', fn)
  }
})
```

```javascript
// main.js - handlers
const { ipcMain, clipboard, nativeImage } = require('electron')

ipcMain.handle('clipboard:read-text', () => {
  return clipboard.readText()
})

ipcMain.handle('clipboard:write-text', (event, text) => {
  if (typeof text !== 'string') return { success: false, error: 'text must be string' }
  clipboard.writeText(text)
  return { success: true }
})
```

## clipboard.readHTML / writeHTML

สำหรับทำงานกับ HTML content ใน clipboard

```javascript
// main.js
const { clipboard, ipcMain } = require('electron')

ipcMain.handle('clipboard:read-html', () => {
  return clipboard.readHTML()
})

ipcMain.handle('clipboard:write-html', (event, html) => {
  if (typeof html !== 'string') {
    return { success: false, error: 'html must be string' }
  }
  
  // เมื่อเขียน HTML ควรเขียน plain text ด้วยเสมอ
  const textContent = html.replace(/<[^>]*>/g, '') // strip HTML tags
  
  clipboard.write({
    html: html,
    text: textContent
  })
  
  return { success: true }
})

// อ่าน HTML
const htmlContent = clipboard.readHTML()
console.log('HTML:', htmlContent)

// เขียน HTML พร้อม plain text fallback
clipboard.write({
  html: '<h1>Hello <strong>World</strong></h1>',
  text: 'Hello World'
})
```

### ตัวอย่าง Rich Text Editor Copy

```javascript
// renderer.js
async function copySelectedText() {
  const selection = window.getSelection()
  if (!selection.rangeCount) return
  
  const range = selection.getRangeAt(0)
  const fragment = range.cloneContents()
  const div = document.createElement('div')
  div.appendChild(fragment)
  
  const htmlContent = div.innerHTML
  const textContent = div.textContent
  
  // เขียน HTML ลง clipboard
  const result = await window.clipboardAPI.writeHTML(htmlContent)
  
  if (result.success) {
    showToast('คัดลอกแล้ว!')
  }
}

function showToast(message) {
  const toast = document.createElement('div')
  toast.className = 'toast'
  toast.textContent = message
  document.body.appendChild(toast)
  setTimeout(() => toast.remove(), 2000)
}
```

## clipboard.readImage / writeImage

```javascript
// main.js
const { clipboard, nativeImage, ipcMain } = require('electron')
const path = require('path')
const fs = require('fs')

// อ่านรูปภาพจาก clipboard
ipcMain.handle('clipboard:read-image', () => {
  const image = clipboard.readImage()
  
  if (image.isEmpty()) {
    return { success: false, error: 'No image in clipboard' }
  }
  
  // แปลงเป็น data URL เพื่อส่งไปยัง renderer
  const dataURL = image.toDataURL()
  const size = image.getSize()
  
  return {
    success: true,
    dataURL,
    width: size.width,
    height: size.height
  }
})

// เขียนรูปภาพลง clipboard จาก data URL
ipcMain.handle('clipboard:write-image', (event, dataURL) => {
  try {
    if (!dataURL || typeof dataURL !== 'string') {
      return { success: false, error: 'Invalid dataURL' }
    }
    
    const image = nativeImage.createFromDataURL(dataURL)
    
    if (image.isEmpty()) {
      return { success: false, error: 'Could not create image' }
    }
    
    clipboard.writeImage(image)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

// บันทึกรูปภาพจาก clipboard ลงไฟล์
ipcMain.handle('clipboard:save-image', async (event, filePath) => {
  const image = clipboard.readImage()
  
  if (image.isEmpty()) {
    return { success: false, error: 'No image in clipboard' }
  }
  
  try {
    const ext = path.extname(filePath).toLowerCase()
    let imageBuffer
    
    switch (ext) {
      case '.png':
        imageBuffer = image.toPNG()
        break
      case '.jpg':
      case '.jpeg':
        imageBuffer = image.toJPEG(90)
        break
      default:
        imageBuffer = image.toPNG()
    }
    
    fs.writeFileSync(filePath, imageBuffer)
    return { success: true, filePath }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

// เขียนรูปจากไฟล์ลง clipboard
ipcMain.handle('clipboard:write-image-from-file', (event, filePath) => {
  try {
    if (!fs.existsSync(filePath)) {
      return { success: false, error: 'File not found' }
    }
    
    const image = nativeImage.createFromPath(filePath)
    
    if (image.isEmpty()) {
      return { success: false, error: 'Could not load image' }
    }
    
    clipboard.writeImage(image)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})
```

### Image Clipboard Component (renderer)

```javascript
// renderer.js - Image Clipboard Manager
const pasteArea = document.getElementById('paste-area')
const imagePreview = document.getElementById('image-preview')

// Paste event สำหรับรับรูปจาก clipboard
document.addEventListener('paste', async (e) => {
  const clipboardItems = e.clipboardData.items
  
  for (const item of clipboardItems) {
    if (item.type.startsWith('image/')) {
      const blob = item.getAsFile()
      const dataURL = await blobToDataURL(blob)
      
      // แสดง preview
      imagePreview.src = dataURL
      imagePreview.style.display = 'block'
      
      break
    }
  }
})

// อ่านรูปจาก clipboard ผ่าน Electron API
document.getElementById('btn-paste-image').addEventListener('click', async () => {
  const result = await window.clipboardAPI.readImage()
  
  if (result.success) {
    imagePreview.src = result.dataURL
    imagePreview.style.display = 'block'
    document.getElementById('img-info').textContent = 
      `${result.width} × ${result.height} pixels`
  } else {
    alert('ไม่มีรูปภาพใน clipboard: ' + result.error)
  }
})

// คัดลอกรูปจาก canvas ลง clipboard
async function copyCanvasToClipboard(canvas) {
  const dataURL = canvas.toDataURL('image/png')
  const result = await window.clipboardAPI.writeImage(dataURL)
  
  if (result.success) {
    showToast('คัดลอกรูปแล้ว!')
  }
}

function blobToDataURL(blob) {
  return new Promise((resolve) => {
    const reader = new FileReader()
    reader.onload = (e) => resolve(e.target.result)
    reader.readAsDataURL(blob)
  })
}
```

## clipboard.readRTF

RTF (Rich Text Format) สำหรับ word processors

```javascript
// main.js
const { clipboard, ipcMain } = require('electron')

// อ่าน RTF จาก clipboard
ipcMain.handle('clipboard:read-rtf', () => {
  const rtf = clipboard.readRTF()
  return { success: true, rtf }
})

// เขียน RTF ลง clipboard
ipcMain.handle('clipboard:write-rtf', (event, rtf) => {
  if (typeof rtf !== 'string') {
    return { success: false, error: 'rtf must be string' }
  }
  
  clipboard.writeRTF(rtf)
  return { success: true }
})

// ตัวอย่าง RTF format
const sampleRTF = `{\\rtf1\\ansi\\deff0
{\\fonttbl{\\f0 Arial;}}
\\f0\\fs24 Hello, {\\b World}!
}`

clipboard.writeRTF(sampleRTF)
const readBack = clipboard.readRTF()
console.log('RTF content:', readBack)
```

## Clipboard Formats (ทุก format พร้อมกัน)

```javascript
// main.js
const { clipboard, nativeImage, ipcMain } = require('electron')

// อ่านทุก format ที่มีใน clipboard
ipcMain.handle('clipboard:read-all', () => {
  const result = {}
  
  // อ่าน plain text
  const text = clipboard.readText()
  if (text) result.text = text
  
  // อ่าน HTML
  const html = clipboard.readHTML()
  if (html) result.html = html
  
  // อ่าน RTF
  const rtf = clipboard.readRTF()
  if (rtf) result.rtf = rtf
  
  // อ่านรูปภาพ
  const image = clipboard.readImage()
  if (!image.isEmpty()) {
    result.image = {
      dataURL: image.toDataURL(),
      size: image.getSize()
    }
  }
  
  // ดู available formats
  result.formats = clipboard.availableFormats()
  
  return result
})

// เขียนหลาย format พร้อมกัน
ipcMain.handle('clipboard:write-all', (event, { text, html, rtf }) => {
  const data = {}
  
  if (typeof text === 'string') data.text = text
  if (typeof html === 'string') data.html = html
  if (typeof rtf === 'string') data.rtf = rtf
  
  clipboard.write(data)
  return { success: true }
})

// ตรวจสอบ formats ที่มี
ipcMain.handle('clipboard:get-formats', () => {
  return clipboard.availableFormats()
})

// เช็คว่ามี format ที่ต้องการ
ipcMain.handle('clipboard:has-format', (event, format) => {
  const formats = clipboard.availableFormats()
  return formats.includes(format)
})

// ล้าง clipboard
ipcMain.handle('clipboard:clear', () => {
  clipboard.clear()
  return { success: true }
})
```

## Clipboard Monitoring

Electron ไม่มี built-in clipboard change event แต่เราสามารถ implement ได้ด้วย polling

```javascript
// main.js - Clipboard Monitor
const { clipboard, nativeImage } = require('electron')
const { EventEmitter } = require('events')

class ClipboardMonitor extends EventEmitter {
  constructor(interval = 500) {
    super()
    this.interval = interval
    this.lastText = ''
    this.lastImage = null
    this.timer = null
    this.isRunning = false
  }
  
  start() {
    if (this.isRunning) return
    this.isRunning = true
    
    // บันทึกค่าเริ่มต้น
    this.lastText = clipboard.readText()
    const img = clipboard.readImage()
    this.lastImageHash = img.isEmpty() ? '' : img.toDataURL().slice(0, 100)
    
    this.timer = setInterval(() => this._check(), this.interval)
    console.log('Clipboard monitoring started')
  }
  
  stop() {
    if (this.timer) {
      clearInterval(this.timer)
      this.timer = null
    }
    this.isRunning = false
    console.log('Clipboard monitoring stopped')
  }
  
  _check() {
    try {
      // ตรวจสอบ text
      const currentText = clipboard.readText()
      if (currentText !== this.lastText) {
        this.lastText = currentText
        this.emit('text-changed', currentText)
        this.emit('changed', { type: 'text', content: currentText })
      }
      
      // ตรวจสอบรูปภาพ
      const currentImage = clipboard.readImage()
      const currentImageHash = currentImage.isEmpty() ? '' : currentImage.toDataURL().slice(0, 100)
      
      if (currentImageHash !== this.lastImageHash) {
        this.lastImageHash = currentImageHash
        
        if (!currentImage.isEmpty()) {
          this.emit('image-changed', currentImage)
          this.emit('changed', { 
            type: 'image', 
            image: {
              dataURL: currentImage.toDataURL(),
              size: currentImage.getSize()
            }
          })
        }
      }
    } catch (error) {
      console.error('Clipboard check error:', error)
    }
  }
  
  getCurrentContent() {
    const text = clipboard.readText()
    const html = clipboard.readHTML()
    const image = clipboard.readImage()
    const formats = clipboard.availableFormats()
    
    return {
      text: text || null,
      html: html || null,
      image: image.isEmpty() ? null : {
        dataURL: image.toDataURL(),
        size: image.getSize()
      },
      formats
    }
  }
}

// ใช้งาน ClipboardMonitor
const monitor = new ClipboardMonitor(300)

module.exports = { ClipboardMonitor, monitor }
```

### ใช้งานกับ BrowserWindow

```javascript
// main.js
const { app, BrowserWindow, ipcMain } = require('electron')
const { ClipboardMonitor } = require('./clipboard-monitor')
const path = require('path')

let mainWindow
let clipboardMonitor

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 900,
    height: 650,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  mainWindow.loadFile('index.html')
  
  // เริ่ม monitor
  clipboardMonitor = new ClipboardMonitor(500)
  
  clipboardMonitor.on('changed', (data) => {
    if (!mainWindow.isDestroyed()) {
      mainWindow.webContents.send('clipboard:changed', data)
    }
  })
  
  clipboardMonitor.start()
  
  mainWindow.on('closed', () => {
    clipboardMonitor.stop()
    clipboardMonitor = null
  })
}

ipcMain.handle('clipboard:start-monitoring', () => {
  if (clipboardMonitor && !clipboardMonitor.isRunning) {
    clipboardMonitor.start()
  }
  return { success: true }
})

ipcMain.handle('clipboard:stop-monitoring', () => {
  if (clipboardMonitor?.isRunning) {
    clipboardMonitor.stop()
  }
  return { success: true }
})

ipcMain.handle('clipboard:get-current', () => {
  return clipboardMonitor?.getCurrentContent() || {}
})

app.whenReady().then(createWindow)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

## Clipboard History

```javascript
// main.js - Clipboard History Manager
const { clipboard, nativeImage, ipcMain, BrowserWindow } = require('electron')

class ClipboardHistory {
  constructor(maxItems = 50) {
    this.maxItems = maxItems
    this.history = []
    this.lastText = ''
    this.timer = null
  }
  
  start(win, interval = 500) {
    this.win = win
    this.timer = setInterval(() => this._poll(), interval)
  }
  
  stop() {
    if (this.timer) {
      clearInterval(this.timer)
      this.timer = null
    }
  }
  
  _poll() {
    const currentText = clipboard.readText()
    
    if (currentText && currentText !== this.lastText) {
      this.lastText = currentText
      
      const item = {
        id: Date.now(),
        type: 'text',
        content: currentText,
        timestamp: new Date().toISOString(),
        preview: currentText.slice(0, 100)
      }
      
      // ลบรายการที่ซ้ำ
      this.history = this.history.filter(h => 
        h.type !== 'text' || h.content !== currentText
      )
      
      // เพิ่มที่หัว
      this.history.unshift(item)
      
      // จำกัดจำนวน
      if (this.history.length > this.maxItems) {
        this.history.pop()
      }
      
      // แจ้ง renderer
      this.win?.webContents.send('clipboard:history-updated', {
        item,
        total: this.history.length
      })
    }
  }
  
  getHistory(limit = 20, offset = 0) {
    return this.history.slice(offset, offset + limit)
  }
  
  deleteItem(id) {
    this.history = this.history.filter(item => item.id !== id)
    return true
  }
  
  clear() {
    this.history = []
    this.lastText = ''
  }
  
  copyItem(id) {
    const item = this.history.find(h => h.id === id)
    if (!item) return false
    
    if (item.type === 'text') {
      clipboard.writeText(item.content)
      this.lastText = item.content
      return true
    }
    
    return false
  }
}

const clipboardHistory = new ClipboardHistory(100)

// IPC Handlers
ipcMain.handle('clipboard-history:get', (event, { limit, offset }) => {
  return clipboardHistory.getHistory(limit, offset)
})

ipcMain.handle('clipboard-history:delete', (event, id) => {
  return clipboardHistory.deleteItem(id)
})

ipcMain.handle('clipboard-history:clear', () => {
  clipboardHistory.clear()
  return { success: true }
})

ipcMain.handle('clipboard-history:copy', (event, id) => {
  const success = clipboardHistory.copyItem(id)
  return { success }
})

module.exports = { clipboardHistory }
```

## Complete Clipboard Manager App

### main.js

```javascript
// main.js
const { app, BrowserWindow, ipcMain, clipboard, nativeImage } = require('electron')
const path = require('path')

let mainWindow

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  mainWindow.loadFile('index.html')
}

// Text operations
ipcMain.handle('clipboard:read-text', () => clipboard.readText())
ipcMain.handle('clipboard:write-text', (e, text) => {
  if (typeof text === 'string') { clipboard.writeText(text); return { success: true } }
  return { success: false }
})

// HTML operations
ipcMain.handle('clipboard:read-html', () => clipboard.readHTML())
ipcMain.handle('clipboard:write-html', (e, html) => {
  if (typeof html === 'string') {
    clipboard.write({ html, text: html.replace(/<[^>]*>/g, '') })
    return { success: true }
  }
  return { success: false }
})

// Image operations
ipcMain.handle('clipboard:read-image', () => {
  const img = clipboard.readImage()
  if (img.isEmpty()) return { success: false, error: 'No image' }
  return { success: true, dataURL: img.toDataURL(), size: img.getSize() }
})

ipcMain.handle('clipboard:write-image', (e, dataURL) => {
  try {
    clipboard.writeImage(nativeImage.createFromDataURL(dataURL))
    return { success: true }
  } catch (err) {
    return { success: false, error: err.message }
  }
})

// Utility
ipcMain.handle('clipboard:clear', () => { clipboard.clear(); return { success: true } })
ipcMain.handle('clipboard:formats', () => clipboard.availableFormats())

app.whenReady().then(createWindow)
app.on('window-all-closed', () => { if (process.platform !== 'darwin') app.quit() })
```

### index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="Content-Security-Policy" 
        content="default-src 'self'; style-src 'self' 'unsafe-inline'; script-src 'self'; img-src 'self' data:;">
  <title>Clipboard Manager</title>
  <style>
    * { box-sizing: border-box; }
    body { font-family: system-ui; background: #f5f5f5; margin: 0; padding: 20px; }
    .container { max-width: 700px; margin: 0 auto; }
    h1 { color: #333; border-bottom: 2px solid #007aff; padding-bottom: 10px; }
    .tabs { display: flex; gap: 8px; margin-bottom: 20px; }
    .tab { padding: 8px 20px; border: none; border-radius: 20px; cursor: pointer;
           background: #e0e0e0; font-size: 14px; }
    .tab.active { background: #007aff; color: white; }
    .panel { display: none; }
    .panel.active { display: block; }
    .section { background: white; border-radius: 8px; padding: 15px; margin-bottom: 15px; 
                box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
    h2 { margin: 0 0 12px; font-size: 14px; color: #666; text-transform: uppercase; letter-spacing: 0.5px; }
    textarea { width: 100%; height: 100px; border: 1px solid #ddd; border-radius: 4px;
               padding: 8px; font-size: 14px; resize: vertical; }
    .btn-row { display: flex; gap: 8px; margin-top: 8px; flex-wrap: wrap; }
    button { padding: 8px 16px; border: none; border-radius: 4px; cursor: pointer;
             background: #007aff; color: white; font-size: 14px; }
    button:hover { opacity: 0.85; }
    button.secondary { background: #e0e0e0; color: #333; }
    button.danger { background: #ff3b30; }
    #image-preview { max-width: 100%; max-height: 300px; border: 1px solid #ddd;
                     border-radius: 4px; display: none; margin-top: 10px; }
    .format-list { display: flex; flex-wrap: wrap; gap: 6px; }
    .format-tag { background: #e8f4ff; color: #007aff; padding: 3px 8px; border-radius: 12px; font-size: 12px; }
    #history-list { max-height: 300px; overflow-y: auto; }
    .history-item { 
      display: flex; justify-content: space-between; align-items: center;
      padding: 8px; border-bottom: 1px solid #f0f0f0; gap: 10px;
    }
    .history-text { flex: 1; font-size: 13px; color: #333; overflow: hidden;
                    text-overflow: ellipsis; white-space: nowrap; }
    .history-time { font-size: 11px; color: #999; white-space: nowrap; }
    .history-btn { padding: 4px 8px; font-size: 12px; }
    #status { background: #1c1c1e; color: #30d158; padding: 10px; border-radius: 4px;
              font-family: monospace; font-size: 13px; min-height: 40px; }
  </style>
</head>
<body>
  <div class="container">
    <h1>Clipboard Manager</h1>
    
    <div class="tabs">
      <button class="tab active" data-panel="text">ข้อความ</button>
      <button class="tab" data-panel="html">HTML</button>
      <button class="tab" data-panel="image">รูปภาพ</button>
      <button class="tab" data-panel="history">ประวัติ</button>
    </div>
    
    <!-- Text Panel -->
    <div class="panel active" id="panel-text">
      <div class="section">
        <h2>อ่าน/เขียน ข้อความ</h2>
        <textarea id="text-input" placeholder="พิมพ์ข้อความที่นี่..."></textarea>
        <div class="btn-row">
          <button id="btn-write-text">คัดลอกไปยัง Clipboard</button>
          <button id="btn-read-text" class="secondary">อ่านจาก Clipboard</button>
          <button id="btn-clear" class="danger">ล้าง Clipboard</button>
        </div>
      </div>
      <div class="section">
        <h2>Formats ที่มีใน Clipboard</h2>
        <div class="format-list" id="format-list"></div>
        <div class="btn-row">
          <button id="btn-get-formats" class="secondary">ตรวจสอบ Formats</button>
        </div>
      </div>
    </div>
    
    <!-- HTML Panel -->
    <div class="panel" id="panel-html">
      <div class="section">
        <h2>HTML Content</h2>
        <textarea id="html-input" placeholder="<h1>Hello <strong>World</strong></h1>"></textarea>
        <div class="btn-row">
          <button id="btn-write-html">เขียน HTML</button>
          <button id="btn-read-html" class="secondary">อ่าน HTML</button>
        </div>
        <div id="html-preview" style="margin-top:10px; border:1px solid #ddd; padding:10px; border-radius:4px; min-height:50px;"></div>
      </div>
    </div>
    
    <!-- Image Panel -->
    <div class="panel" id="panel-image">
      <div class="section">
        <h2>รูปภาพ</h2>
        <div class="btn-row">
          <button id="btn-read-image">อ่านรูปจาก Clipboard</button>
          <button id="btn-paste-image" class="secondary">Paste รูปภาพ (Ctrl+V)</button>
        </div>
        <img id="image-preview" alt="Preview">
        <div id="image-info" style="color:#666; font-size:13px; margin-top:5px;"></div>
        <div class="btn-row" id="image-actions" style="display:none;">
          <button id="btn-copy-image">คัดลอกรูปกลับ</button>
        </div>
      </div>
    </div>
    
    <!-- History Panel -->
    <div class="panel" id="panel-history">
      <div class="section">
        <h2>ประวัติ Clipboard</h2>
        <div id="history-list"></div>
        <div class="btn-row" style="margin-top:10px;">
          <button id="btn-clear-history" class="danger">ล้างประวัติ</button>
        </div>
      </div>
    </div>
    
    <div id="status">พร้อมใช้งาน...</div>
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

### renderer.js

```javascript
// renderer.js
const status = document.getElementById('status')
const history = []

function log(msg) {
  status.textContent = `${new Date().toLocaleTimeString('th-TH')}: ${msg}`
}

// Tab switching
document.querySelectorAll('.tab').forEach(tab => {
  tab.addEventListener('click', () => {
    document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'))
    document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'))
    
    tab.classList.add('active')
    document.getElementById(`panel-${tab.dataset.panel}`).classList.add('active')
  })
})

// ─── Text Operations ──────────────────────────────────────────────
document.getElementById('btn-write-text').addEventListener('click', async () => {
  const text = document.getElementById('text-input').value
  if (!text) { log('กรุณาพิมพ์ข้อความก่อน'); return }
  
  const result = await window.clipboardAPI.writeText(text)
  log(result.success ? 'คัดลอกข้อความแล้ว' : `ผิดพลาด: ${result.error}`)
})

document.getElementById('btn-read-text').addEventListener('click', async () => {
  const text = await window.clipboardAPI.readText()
  document.getElementById('text-input').value = text
  log(`อ่านข้อความสำเร็จ: "${text.slice(0, 50)}${text.length > 50 ? '...' : ''}"`)
  addToHistory(text)
})

document.getElementById('btn-clear').addEventListener('click', async () => {
  const result = await window.clipboardAPI.clear()
  log(result.success ? 'ล้าง Clipboard แล้ว' : 'เกิดข้อผิดพลาด')
})

document.getElementById('btn-get-formats').addEventListener('click', async () => {
  const formats = await window.clipboardAPI.getFormats()
  const list = document.getElementById('format-list')
  list.innerHTML = formats.map(f => `<span class="format-tag">${f}</span>`).join('')
  log(`พบ ${formats.length} formats: ${formats.join(', ')}`)
})

// ─── HTML Operations ──────────────────────────────────────────────
document.getElementById('btn-write-html').addEventListener('click', async () => {
  const html = document.getElementById('html-input').value
  const result = await window.clipboardAPI.writeHTML(html)
  log(result.success ? 'เขียน HTML แล้ว' : `ผิดพลาด: ${result.error}`)
})

document.getElementById('btn-read-html').addEventListener('click', async () => {
  const html = await window.clipboardAPI.readHTML()
  document.getElementById('html-input').value = html
  // แสดง preview (sanitized)
  const preview = document.getElementById('html-preview')
  preview.textContent = html // ใช้ textContent เพื่อความปลอดภัย
  log('อ่าน HTML แล้ว')
})

// ─── Image Operations ─────────────────────────────────────────────
document.getElementById('btn-read-image').addEventListener('click', async () => {
  const result = await window.clipboardAPI.readImage()
  
  if (result.success) {
    const preview = document.getElementById('image-preview')
    preview.src = result.dataURL
    preview.style.display = 'block'
    document.getElementById('image-info').textContent = 
      `${result.width} × ${result.height} pixels`
    document.getElementById('image-actions').style.display = 'flex'
    log('อ่านรูปจาก Clipboard แล้ว')
  } else {
    log('ไม่มีรูปภาพใน Clipboard')
  }
})

document.getElementById('btn-copy-image').addEventListener('click', async () => {
  const preview = document.getElementById('image-preview')
  if (!preview.src || preview.src === window.location.href) { log('ไม่มีรูปให้คัดลอก'); return }
  
  const result = await window.clipboardAPI.writeImage(preview.src)
  log(result.success ? 'คัดลอกรูปภาพแล้ว' : `ผิดพลาด: ${result.error}`)
})

// Paste event
document.addEventListener('paste', (e) => {
  const items = e.clipboardData?.items || []
  for (const item of items) {
    if (item.type.startsWith('image/')) {
      const blob = item.getAsFile()
      const reader = new FileReader()
      reader.onload = (ev) => {
        const preview = document.getElementById('image-preview')
        preview.src = ev.target.result
        preview.style.display = 'block'
        document.getElementById('image-actions').style.display = 'flex'
        log('Paste รูปภาพแล้ว')
      }
      reader.readAsDataURL(blob)
      break
    }
  }
})

// ─── History ──────────────────────────────────────────────────────
function addToHistory(text) {
  if (!text || history.includes(text)) return
  history.unshift(text)
  if (history.length > 20) history.pop()
  renderHistory()
}

function renderHistory() {
  const list = document.getElementById('history-list')
  list.innerHTML = history.map((text, i) => `
    <div class="history-item">
      <span class="history-text">${text.replace(/</g, '&lt;')}</span>
      <button class="history-btn" onclick="pasteHistoryItem(${i})">ใช้</button>
    </div>
  `).join('') || '<p style="color:#999; padding:8px;">ยังไม่มีประวัติ</p>'
}

window.pasteHistoryItem = async (index) => {
  const text = history[index]
  const result = await window.clipboardAPI.writeText(text)
  if (result.success) {
    document.getElementById('text-input').value = text
    log(`คัดลอก "${text.slice(0, 30)}..." ไปยัง Clipboard`)
  }
}

document.getElementById('btn-clear-history').addEventListener('click', () => {
  history.length = 0
  renderHistory()
  log('ล้างประวัติแล้ว')
})

// Listen for clipboard changes
const unsubClipboard = window.clipboardAPI.onChanged((data) => {
  if (data.type === 'text') {
    addToHistory(data.content)
    log(`Clipboard เปลี่ยนแปลง: "${data.content.slice(0, 50)}"`)
  }
})

window.addEventListener('beforeunload', () => unsubClipboard())

log('Clipboard Manager พร้อมใช้งาน')
```

## สรุป

Clipboard module ของ Electron ให้ความสามารถครบครัน:

- **readText/writeText** - สำหรับ plain text
- **readHTML/writeHTML** - สำหรับ HTML content
- **readImage/writeImage** - สำหรับรูปภาพ (ผ่าน nativeImage)
- **readRTF/writeRTF** - สำหรับ Rich Text Format
- **availableFormats** - ตรวจสอบว่ามี format อะไรบ้าง
- **write** - เขียนหลาย format พร้อมกัน
- **clear** - ล้าง clipboard

การ monitor clipboard ทำได้ด้วยการ polling เพราะ Electron ไม่มี native clipboard change event

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Keyboard Shortcuts
