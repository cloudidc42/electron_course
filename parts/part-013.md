# ตอนที่ 13: Shell & System Integration

## Shell Module ใน Electron

`shell` module ของ Electron ให้ความสามารถในการโต้ตอบกับระบบปฏิบัติการ เช่น การเปิดไฟล์ โฟลเดอร์ URLs และการทำงานกับ filesystem ผ่าน default applications ของระบบ

```javascript
const { shell } = require('electron')
```

## shell.openExternal

ใช้สำหรับเปิด URL ใน default web browser หรือเปิดไฟล์ด้วย default application

### รูปแบบพื้นฐาน

```javascript
// main.js
const { shell } = require('electron')

// เปิด URL ใน default browser
async function openInBrowser(url) {
  try {
    await shell.openExternal(url)
    console.log('Opened:', url)
  } catch (error) {
    console.error('Failed to open URL:', error)
  }
}

// ตัวอย่างการใช้งาน
openInBrowser('https://www.google.com')
openInBrowser('mailto:user@example.com')        // เปิด email client
openInBrowser('tel:+66812345678')               // เปิด phone dialer
```

### ตัวอย่างการใช้งานใน preload + renderer

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('shellAPI', {
  openExternal: (url) => ipcRenderer.invoke('shell:open-external', url)
})
```

```javascript
// main.js
const { ipcMain, shell } = require('electron')

ipcMain.handle('shell:open-external', async (event, url) => {
  // ตรวจสอบ URL ก่อนเปิด
  if (typeof url !== 'string') return { success: false, error: 'Invalid URL' }
  
  try {
    const parsed = new URL(url)
    // อนุญาตเฉพาะ protocol ที่ปลอดภัย
    const allowedProtocols = ['https:', 'http:', 'mailto:']
    if (!allowedProtocols.includes(parsed.protocol)) {
      return { success: false, error: `Protocol ${parsed.protocol} not allowed` }
    }
    
    await shell.openExternal(url)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})
```

```javascript
// renderer.js
// เปิด link ใน browser เมื่อ click
document.addEventListener('click', async (e) => {
  if (e.target.tagName === 'A' && e.target.href) {
    const href = e.target.href
    
    // เปิด external links ใน browser
    if (href.startsWith('http://') || href.startsWith('https://')) {
      e.preventDefault()
      const result = await window.shellAPI.openExternal(href)
      if (!result.success) {
        console.error('Failed to open link:', result.error)
      }
    }
  }
})
```

## shell.openPath

ใช้สำหรับเปิดไฟล์หรือโฟลเดอร์ด้วย default application ของระบบ

```javascript
// main.js
const { shell } = require('electron')
const path = require('path')

async function openFile(filePath) {
  try {
    const errorMessage = await shell.openPath(filePath)
    
    if (errorMessage === '') {
      console.log('File opened successfully')
      return { success: true }
    } else {
      console.error('Failed to open file:', errorMessage)
      return { success: false, error: errorMessage }
    }
  } catch (error) {
    return { success: false, error: error.message }
  }
}

// ตัวอย่าง
openFile('/Users/user/Documents/report.pdf')    // เปิด PDF ด้วย Preview
openFile('/Users/user/Music/song.mp3')          // เปิดด้วย Music app
openFile('/Users/user/Documents/')              // เปิด Finder/Explorer
```

### สร้าง File Opener Application

```javascript
// main.js - complete file opener
const { app, BrowserWindow, ipcMain, shell, dialog } = require('electron')
const path = require('path')
const fs = require('fs')

function createWindow() {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  win.loadFile('index.html')
  return win
}

// Handler สำหรับเลือกและเปิดไฟล์
ipcMain.handle('file:choose-and-open', async (event) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showOpenDialog(win, {
    title: 'เลือกไฟล์ที่ต้องการเปิด',
    properties: ['openFile'],
    filters: [
      { name: 'Documents', extensions: ['pdf', 'doc', 'docx', 'txt'] },
      { name: 'Images', extensions: ['jpg', 'jpeg', 'png', 'gif'] },
      { name: 'All Files', extensions: ['*'] }
    ]
  })
  
  if (result.canceled) return { canceled: true }
  
  const filePath = result.filePaths[0]
  const errorMessage = await shell.openPath(filePath)
  
  if (errorMessage) {
    return { success: false, error: errorMessage, filePath }
  }
  
  return { success: true, filePath }
})

// Handler สำหรับเปิดไฟล์โดยตรง
ipcMain.handle('file:open-with-default', async (event, filePath) => {
  if (!fs.existsSync(filePath)) {
    return { success: false, error: 'File not found' }
  }
  
  const errorMessage = await shell.openPath(filePath)
  return errorMessage ? { success: false, error: errorMessage } : { success: true }
})

app.whenReady().then(createWindow)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

## shell.showItemInFolder

เปิด File Manager (Finder/Explorer) และ highlight ไฟล์ที่ระบุ

```javascript
// main.js
const { shell, ipcMain } = require('electron')
const fs = require('fs')

ipcMain.handle('shell:show-in-folder', async (event, filePath) => {
  if (!fs.existsSync(filePath)) {
    return { success: false, error: 'File not found' }
  }
  
  shell.showItemInFolder(filePath)
  return { success: true }
})

// ใช้งาน: เปิด Finder แล้ว highlight ไฟล์
// shell.showItemInFolder('/Users/user/Documents/report.pdf')
```

### ตัวอย่าง: File Manager Component

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('shellAPI', {
  openFile: (path) => ipcRenderer.invoke('file:open-with-default', path),
  showInFolder: (path) => ipcRenderer.invoke('shell:show-in-folder', path),
  openURL: (url) => ipcRenderer.invoke('shell:open-external', url),
  trashFile: (path) => ipcRenderer.invoke('shell:trash-item', path)
})
```

```html
<!-- index.html -->
<div id="file-list">
  <div class="file-item" data-path="/Users/user/Downloads/file.pdf">
    <span class="file-name">file.pdf</span>
    <div class="file-actions">
      <button class="btn-open">เปิด</button>
      <button class="btn-show">แสดงใน Finder</button>
      <button class="btn-trash">ลบ</button>
    </div>
  </div>
</div>
```

```javascript
// renderer.js
document.querySelectorAll('.file-item').forEach(item => {
  const filePath = item.dataset.path
  
  item.querySelector('.btn-open').addEventListener('click', async () => {
    const result = await window.shellAPI.openFile(filePath)
    if (!result.success) {
      alert('ไม่สามารถเปิดไฟล์ได้: ' + result.error)
    }
  })
  
  item.querySelector('.btn-show').addEventListener('click', async () => {
    await window.shellAPI.showInFolder(filePath)
  })
  
  item.querySelector('.btn-trash').addEventListener('click', async () => {
    if (confirm(`ต้องการลบ ${filePath.split('/').pop()} หรือไม่?`)) {
      const result = await window.shellAPI.trashFile(filePath)
      if (result.success) {
        item.remove()
      } else {
        alert('ไม่สามารถลบได้: ' + result.error)
      }
    }
  })
})
```

## shell.trashItem

ย้ายไฟล์ไปยัง Trash/Recycle Bin แทนการลบถาวร

```javascript
// main.js
const { shell, ipcMain } = require('electron')
const fs = require('fs')

ipcMain.handle('shell:trash-item', async (event, filePath) => {
  if (!fs.existsSync(filePath)) {
    return { success: false, error: 'File not found' }
  }
  
  try {
    await shell.trashItem(filePath)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

// ตัวอย่าง
async function deleteFile(filePath) {
  try {
    await shell.trashItem(filePath)
    console.log('File moved to trash:', filePath)
  } catch (error) {
    console.error('Cannot trash file:', error)
    // Fallback: ลบถาวร
    fs.unlinkSync(filePath)
  }
}
```

### Batch Trash

```javascript
// main.js
ipcMain.handle('shell:trash-multiple', async (event, filePaths) => {
  if (!Array.isArray(filePaths)) {
    return { success: false, error: 'filePaths must be an array' }
  }
  
  const results = []
  
  for (const filePath of filePaths) {
    try {
      if (!fs.existsSync(filePath)) {
        results.push({ filePath, success: false, error: 'Not found' })
        continue
      }
      
      await shell.trashItem(filePath)
      results.push({ filePath, success: true })
    } catch (error) {
      results.push({ filePath, success: false, error: error.message })
    }
  }
  
  const successCount = results.filter(r => r.success).length
  return { 
    success: true, 
    results,
    summary: `${successCount}/${filePaths.length} ไฟล์ถูกย้ายไปยัง Trash`
  }
})
```

## child_process.exec และ spawn

Node.js `child_process` ใช้สำหรับรัน external commands และ programs จาก Electron application

### child_process.exec

```javascript
// main.js
const { exec } = require('child_process')
const { promisify } = require('util')

const execAsync = promisify(exec)

// รัน command แบบง่าย
async function runCommand(command) {
  try {
    const { stdout, stderr } = await execAsync(command)
    return { success: true, stdout, stderr }
  } catch (error) {
    return { 
      success: false, 
      error: error.message,
      stdout: error.stdout,
      stderr: error.stderr
    }
  }
}

// ตัวอย่างการใช้งาน
async function getSystemInfo() {
  const platform = process.platform
  
  if (platform === 'darwin') {
    return await runCommand('system_profiler SPHardwareDataType')
  } else if (platform === 'win32') {
    return await runCommand('systeminfo')
  } else {
    return await runCommand('uname -a && cat /etc/os-release')
  }
}

// ตรวจสอบว่า Git ติดตั้งอยู่
async function checkGit() {
  const result = await runCommand('git --version')
  if (result.success) {
    return { installed: true, version: result.stdout.trim() }
  }
  return { installed: false }
}
```

### child_process.spawn

ใช้สำหรับ commands ที่ต้องการ streaming output หรือ long-running processes

```javascript
// main.js
const { spawn } = require('child_process')
const { ipcMain } = require('electron')
const path = require('path')

// รัน Python script พร้อม real-time output
ipcMain.handle('process:run-python', async (event, scriptPath, args = []) => {
  return new Promise((resolve) => {
    const win = BrowserWindow.fromWebContents(event.sender)
    const process = spawn('python3', [scriptPath, ...args])
    
    let stdout = ''
    let stderr = ''
    
    process.stdout.on('data', (data) => {
      const text = data.toString()
      stdout += text
      // ส่ง output แบบ real-time ไปยัง renderer
      win.webContents.send('process:output', { type: 'stdout', text })
    })
    
    process.stderr.on('data', (data) => {
      const text = data.toString()
      stderr += text
      win.webContents.send('process:output', { type: 'stderr', text })
    })
    
    process.on('close', (code) => {
      win.webContents.send('process:done', { exitCode: code })
      resolve({
        exitCode: code,
        stdout,
        stderr,
        success: code === 0
      })
    })
    
    process.on('error', (error) => {
      win.webContents.send('process:error', { error: error.message })
      resolve({ success: false, error: error.message })
    })
  })
})
```

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('processAPI', {
  runPython: (scriptPath, args) => ipcRenderer.invoke('process:run-python', scriptPath, args),
  
  onOutput: (callback) => {
    const fn = (_e, data) => callback(data)
    ipcRenderer.on('process:output', fn)
    return () => ipcRenderer.removeListener('process:output', fn)
  },
  
  onDone: (callback) => {
    const fn = (_e, data) => callback(data)
    ipcRenderer.on('process:done', fn)
    return () => ipcRenderer.removeListener('process:done', fn)
  },
  
  onError: (callback) => {
    const fn = (_e, data) => callback(data)
    ipcRenderer.on('process:error', fn)
    return () => ipcRenderer.removeListener('process:error', fn)
  }
})
```

```javascript
// renderer.js
const outputEl = document.getElementById('output')
const btnRun = document.getElementById('btn-run')

let unsubOutput, unsubDone, unsubError

btnRun.addEventListener('click', async () => {
  outputEl.textContent = ''
  btnRun.disabled = true
  
  // Subscribe to events
  unsubOutput = window.processAPI.onOutput((data) => {
    const className = data.type === 'stderr' ? 'error' : 'info'
    outputEl.innerHTML += `<span class="${className}">${data.text}</span>`
  })
  
  unsubDone = window.processAPI.onDone((data) => {
    outputEl.innerHTML += `\n\nProcess exited with code: ${data.exitCode}`
    cleanup()
  })
  
  unsubError = window.processAPI.onError((data) => {
    outputEl.innerHTML += `\n\nError: ${data.error}`
    cleanup()
  })
  
  const result = await window.processAPI.runPython('/path/to/script.py', ['--verbose'])
  console.log('Result:', result)
})

function cleanup() {
  btnRun.disabled = false
  unsubOutput?.()
  unsubDone?.()
  unsubError?.()
}
```

### Process Manager

```javascript
// main.js - Process Manager พร้อม kill support
const { ipcMain, BrowserWindow } = require('electron')
const { spawn } = require('child_process')

const runningProcesses = new Map()

ipcMain.handle('process:start', async (event, { id, command, args = [], cwd }) => {
  if (runningProcesses.has(id)) {
    return { success: false, error: `Process ${id} already running` }
  }
  
  const win = BrowserWindow.fromWebContents(event.sender)
  
  try {
    const child = spawn(command, args, {
      cwd: cwd || process.cwd(),
      shell: true
    })
    
    runningProcesses.set(id, child)
    
    child.stdout.on('data', (data) => {
      win?.webContents.send('process:stdout', { id, data: data.toString() })
    })
    
    child.stderr.on('data', (data) => {
      win?.webContents.send('process:stderr', { id, data: data.toString() })
    })
    
    child.on('close', (code, signal) => {
      runningProcesses.delete(id)
      win?.webContents.send('process:exit', { id, code, signal })
    })
    
    child.on('error', (error) => {
      runningProcesses.delete(id)
      win?.webContents.send('process:error', { id, error: error.message })
    })
    
    return { success: true, pid: child.pid }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

ipcMain.handle('process:stop', async (event, id) => {
  const child = runningProcesses.get(id)
  if (!child) {
    return { success: false, error: `Process ${id} not found` }
  }
  
  child.kill()
  runningProcesses.delete(id)
  return { success: true }
})

ipcMain.handle('process:list', () => {
  return Array.from(runningProcesses.keys()).map(id => ({
    id,
    pid: runningProcesses.get(id).pid
  }))
})
```

## เปิดไฟล์ด้วย Default Application

### ตรวจสอบ File Type และเปิดตามที่เหมาะสม

```javascript
// main.js
const { shell, ipcMain, dialog } = require('electron')
const path = require('path')
const fs = require('fs')

// แสดง context menu สำหรับไฟล์
ipcMain.handle('file:open-with-options', async (event, filePath) => {
  const ext = path.extname(filePath).toLowerCase()
  const fileName = path.basename(filePath)
  
  // ตัดสินใจว่าจะเปิดอย่างไรตาม file type
  const textExtensions = ['.txt', '.md', '.json', '.js', '.css', '.html', '.csv']
  const imageExtensions = ['.jpg', '.jpeg', '.png', '.gif', '.svg', '.webp']
  const documentExtensions = ['.pdf', '.doc', '.docx', '.xls', '.xlsx']
  
  if (textExtensions.includes(ext)) {
    // อ่านและแสดงใน app
    try {
      const content = fs.readFileSync(filePath, 'utf8')
      return { action: 'display', content, fileName }
    } catch (e) {
      return { action: 'error', error: e.message }
    }
  } else if (imageExtensions.includes(ext) || documentExtensions.includes(ext)) {
    // เปิดด้วย default app
    const errorMessage = await shell.openPath(filePath)
    if (errorMessage) {
      return { action: 'error', error: errorMessage }
    }
    return { action: 'opened-external', fileName }
  } else {
    // ถามว่าจะทำอะไร
    return { action: 'unknown', ext, fileName }
  }
})
```

### Complete Shell Integration Example

```javascript
// main.js - Complete Example
const { app, BrowserWindow, ipcMain, shell, dialog } = require('electron')
const { exec } = require('child_process')
const { promisify } = require('util')
const path = require('path')
const fs = require('fs')

const execAsync = promisify(exec)
let mainWindow

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1000,
    height: 700,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  mainWindow.loadFile('index.html')
}

// ─── Shell Operations ───────────────────────────────────────────

ipcMain.handle('shell:open-url', async (event, url) => {
  try {
    await shell.openExternal(url)
    return { success: true }
  } catch (e) {
    return { success: false, error: e.message }
  }
})

ipcMain.handle('shell:open-path', async (event, filePath) => {
  const error = await shell.openPath(filePath)
  return error ? { success: false, error } : { success: true }
})

ipcMain.handle('shell:show-in-folder', async (event, filePath) => {
  shell.showItemInFolder(filePath)
  return { success: true }
})

ipcMain.handle('shell:trash', async (event, filePath) => {
  try {
    await shell.trashItem(filePath)
    return { success: true }
  } catch (e) {
    return { success: false, error: e.message }
  }
})

// ─── System Commands ────────────────────────────────────────────

ipcMain.handle('system:get-platform-info', async () => {
  const platform = process.platform
  let info = {
    platform,
    arch: process.arch,
    nodeVersion: process.version
  }
  
  try {
    if (platform === 'darwin') {
      const { stdout } = await execAsync('sw_vers')
      info.osInfo = stdout.trim()
    } else if (platform === 'linux') {
      const { stdout } = await execAsync('cat /etc/os-release | head -3')
      info.osInfo = stdout.trim()
    } else if (platform === 'win32') {
      const { stdout } = await execAsync('ver')
      info.osInfo = stdout.trim()
    }
  } catch (e) {
    info.osInfo = 'Unable to get OS info'
  }
  
  return info
})

ipcMain.handle('system:open-terminal', async (event, directory) => {
  const platform = process.platform
  
  try {
    if (platform === 'darwin') {
      await execAsync(`open -a Terminal "${directory}"`)
    } else if (platform === 'linux') {
      // ลอง terminals ต่างๆ
      const terminals = ['gnome-terminal', 'xterm', 'konsole', 'xfce4-terminal']
      let opened = false
      
      for (const terminal of terminals) {
        try {
          await execAsync(`${terminal} --working-directory="${directory}"`)
          opened = true
          break
        } catch (e) {
          continue
        }
      }
      
      if (!opened) throw new Error('No terminal emulator found')
    } else if (platform === 'win32') {
      await execAsync(`start cmd /K "cd /d ${directory}"`)
    }
    
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

// ─── File Dialog Operations ─────────────────────────────────────

ipcMain.handle('dialog:open-file', async () => {
  const result = await dialog.showOpenDialog(mainWindow, {
    properties: ['openFile'],
    title: 'เลือกไฟล์'
  })
  
  if (result.canceled) return { canceled: true }
  return { filePath: result.filePaths[0] }
})

ipcMain.handle('dialog:open-directory', async () => {
  const result = await dialog.showOpenDialog(mainWindow, {
    properties: ['openDirectory'],
    title: 'เลือกโฟลเดอร์'
  })
  
  if (result.canceled) return { canceled: true }
  return { dirPath: result.filePaths[0] }
})

app.whenReady().then(createWindow)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

### preload.js

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('shellAPI', {
  openURL: (url) => ipcRenderer.invoke('shell:open-url', url),
  openPath: (path) => ipcRenderer.invoke('shell:open-path', path),
  showInFolder: (path) => ipcRenderer.invoke('shell:show-in-folder', path),
  trash: (path) => ipcRenderer.invoke('shell:trash', path)
})

contextBridge.exposeInMainWorld('systemAPI', {
  getPlatformInfo: () => ipcRenderer.invoke('system:get-platform-info'),
  openTerminal: (dir) => ipcRenderer.invoke('system:open-terminal', dir)
})

contextBridge.exposeInMainWorld('dialogAPI', {
  openFile: () => ipcRenderer.invoke('dialog:open-file'),
  openDirectory: () => ipcRenderer.invoke('dialog:open-directory')
})
```

### index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="Content-Security-Policy" 
        content="default-src 'self'; style-src 'self' 'unsafe-inline'; script-src 'self';">
  <title>Shell Integration Demo</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { 
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      background: #1e1e2e; color: #cdd6f4; padding: 20px;
    }
    h1 { margin-bottom: 20px; color: #89b4fa; }
    .section { 
      background: #313244; border-radius: 8px; 
      padding: 15px; margin-bottom: 15px;
    }
    h2 { font-size: 16px; margin-bottom: 12px; color: #cba6f7; }
    .btn-row { display: flex; flex-wrap: wrap; gap: 8px; }
    button {
      background: #89b4fa; color: #1e1e2e; border: none;
      padding: 8px 16px; border-radius: 4px; cursor: pointer;
      font-size: 14px; transition: opacity 0.2s;
    }
    button:hover { opacity: 0.8; }
    button.danger { background: #f38ba8; }
    #output {
      background: #11111b; border-radius: 4px;
      padding: 12px; font-family: monospace; font-size: 13px;
      min-height: 60px; margin-top: 15px;
      color: #a6e3a1; white-space: pre-wrap; max-height: 200px; overflow-y: auto;
    }
    .info-grid { 
      display: grid; grid-template-columns: repeat(2, 1fr); gap: 8px;
    }
    .info-item { background: #1e1e2e; padding: 8px; border-radius: 4px; }
  </style>
</head>
<body>
  <h1>Shell & System Integration</h1>
  
  <div class="section">
    <h2>เปิด URLs และไฟล์</h2>
    <div class="btn-row">
      <button id="btn-open-url">เปิด Google</button>
      <button id="btn-open-file">เลือกและเปิดไฟล์</button>
      <button id="btn-open-folder">เลือกและเปิดโฟลเดอร์</button>
    </div>
  </div>
  
  <div class="section">
    <h2>File Manager</h2>
    <div class="btn-row">
      <button id="btn-show-file">แสดงไฟล์ใน Finder</button>
      <button id="btn-trash-file" class="danger">ย้ายไป Trash</button>
    </div>
  </div>
  
  <div class="section">
    <h2>ระบบ</h2>
    <div class="btn-row">
      <button id="btn-sys-info">ข้อมูลระบบ</button>
      <button id="btn-open-terminal">เปิด Terminal</button>
    </div>
    <div class="info-grid" id="sys-info"></div>
  </div>
  
  <div id="output">ผลลัพธ์จะแสดงที่นี่...</div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

### renderer.js

```javascript
// renderer.js
const output = document.getElementById('output')

function log(msg) {
  const time = new Date().toLocaleTimeString('th-TH')
  output.textContent += `[${time}] ${msg}\n`
  output.scrollTop = output.scrollHeight
}

// เปิด URL
document.getElementById('btn-open-url').addEventListener('click', async () => {
  const result = await window.shellAPI.openURL('https://electronjs.org')
  log(result.success ? 'เปิด electronjs.org ใน browser แล้ว' : `ผิดพลาด: ${result.error}`)
})

// เลือกและเปิดไฟล์
document.getElementById('btn-open-file').addEventListener('click', async () => {
  const dialResult = await window.dialogAPI.openFile()
  if (dialResult.canceled) { log('ยกเลิก'); return }
  
  const result = await window.shellAPI.openPath(dialResult.filePath)
  log(result.success 
    ? `เปิดไฟล์: ${dialResult.filePath}` 
    : `ผิดพลาด: ${result.error}`)
})

// เลือกและเปิดโฟลเดอร์
document.getElementById('btn-open-folder').addEventListener('click', async () => {
  const dialResult = await window.dialogAPI.openDirectory()
  if (dialResult.canceled) { log('ยกเลิก'); return }
  
  const result = await window.shellAPI.openPath(dialResult.dirPath)
  log(result.success 
    ? `เปิดโฟลเดอร์: ${dialResult.dirPath}` 
    : `ผิดพลาด: ${result.error}`)
})

// แสดงไฟล์ใน Finder
document.getElementById('btn-show-file').addEventListener('click', async () => {
  const dialResult = await window.dialogAPI.openFile()
  if (dialResult.canceled) { log('ยกเลิก'); return }
  
  await window.shellAPI.showInFolder(dialResult.filePath)
  log(`แสดงไฟล์ใน Finder: ${dialResult.filePath}`)
})

// ย้ายไป Trash
document.getElementById('btn-trash-file').addEventListener('click', async () => {
  const dialResult = await window.dialogAPI.openFile()
  if (dialResult.canceled) { log('ยกเลิก'); return }
  
  if (!confirm(`ต้องการย้าย ${dialResult.filePath} ไปยัง Trash หรือไม่?`)) return
  
  const result = await window.shellAPI.trash(dialResult.filePath)
  log(result.success 
    ? `ย้ายไป Trash: ${dialResult.filePath}` 
    : `ผิดพลาด: ${result.error}`)
})

// ข้อมูลระบบ
document.getElementById('btn-sys-info').addEventListener('click', async () => {
  const info = await window.systemAPI.getPlatformInfo()
  const sysInfoEl = document.getElementById('sys-info')
  
  sysInfoEl.innerHTML = Object.entries(info).map(([key, val]) => `
    <div class="info-item">
      <strong>${key}:</strong> ${val}
    </div>
  `).join('')
  
  log('โหลดข้อมูลระบบแล้ว')
})

// เปิด Terminal
document.getElementById('btn-open-terminal').addEventListener('click', async () => {
  const dialResult = await window.dialogAPI.openDirectory()
  if (dialResult.canceled) { log('ยกเลิก'); return }
  
  const result = await window.systemAPI.openTerminal(dialResult.dirPath)
  log(result.success 
    ? `เปิด Terminal ที่: ${dialResult.dirPath}` 
    : `ผิดพลาด: ${result.error}`)
})
```

## สรุป

Shell & System Integration ช่วยให้ Electron app ทำงานร่วมกับระบบปฏิบัติการได้อย่างลึกซึ้ง:

- **shell.openExternal** - เปิด URLs และ mailto ใน default browser/client
- **shell.openPath** - เปิดไฟล์/โฟลเดอร์ด้วย default app
- **shell.showItemInFolder** - highlight ไฟล์ใน File Manager
- **shell.trashItem** - ย้ายไฟล์ไป Trash อย่างปลอดภัย
- **child_process.exec** - รัน system commands
- **child_process.spawn** - รัน long-running processes พร้อม streaming output

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Clipboard Operations
