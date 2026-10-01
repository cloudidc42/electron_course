# ตอนที่ 8: Dialog Boxes - หน้าต่างโต้ตอบ

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจากศึกษาตอนนี้ คุณจะสามารถ:
- ใช้งาน `dialog.showMessageBox` เพื่อแสดงข้อความ
- ใช้ `dialog.showErrorBox` เพื่อแสดงข้อผิดพลาด
- ใช้ `dialog.showOpenDialog` เพื่อเปิดไฟล์
- ใช้ `dialog.showSaveDialog` เพื่อบันทึกไฟล์
- กำหนด options ต่างๆ ของ dialog
- ใช้ async/await กับ dialog
- สร้าง custom dialog ด้วย HTML
- ใช้ confirm dialog pattern ที่ถูกต้อง

---

## บทนำ: Dialog ใน Electron

Electron มี native dialogs ที่ดูเข้ากันกับ OS ของแต่ละแพลตฟอร์ม:

```
dialog module
├── showMessageBox    → แสดงข้อความ / ยืนยัน
├── showMessageBoxSync → (synchronous version)
├── showErrorBox      → แสดง error message
├── showOpenDialog    → เลือกไฟล์/โฟลเดอร์ที่จะเปิด
├── showOpenDialogSync
├── showSaveDialog    → เลือกที่บันทึกไฟล์
└── showSaveDialogSync
```

---

## 8.1 showMessageBox - หน้าต่างข้อความ

### การใช้งานพื้นฐาน

```javascript
// main.js
const { app, BrowserWindow, dialog, ipcMain } = require('electron')
const path = require('path')

let mainWindow

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 900,
    height: 650,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  mainWindow.loadFile('index.html')
}

// ==========================================
// showMessageBox แบบ async (แนะนำ)
// ==========================================
ipcMain.handle('show-message', async (event, options) => {
  // ดึง window ที่เรียกมา (เพื่อให้ dialog เป็น modal)
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showMessageBox(win, {
    type: options.type || 'info',   // 'none', 'info', 'error', 'question', 'warning'
    title: options.title || 'ข้อความ',
    message: options.message,
    detail: options.detail || '',   // รายละเอียดเพิ่มเติม
    buttons: options.buttons || ['ตกลง'],
    defaultId: options.defaultId || 0,     // ปุ่ม default (กด Enter)
    cancelId: options.cancelId,            // ปุ่ม cancel (กด Escape)
    checkboxLabel: options.checkboxLabel,  // label สำหรับ checkbox (optional)
    checkboxChecked: options.checkboxChecked || false,
    noLink: false  // Windows: ปุ่มแสดงเป็น link หรือปุ่ม
  })
  
  // result.response = index ของปุ่มที่กด
  // result.checkboxChecked = state ของ checkbox
  return result
})

// ==========================================
// ตัวอย่างต่างๆ ของ showMessageBox
// ==========================================

// 1. Simple Info Message
ipcMain.handle('show-info', async (event, { title, message }) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  await dialog.showMessageBox(win, {
    type: 'info',
    title: title || 'ข้อมูล',
    message: message,
    buttons: ['ตกลง']
  })
  
  return { dismissed: true }
})

// 2. Warning Message
ipcMain.handle('show-warning', async (event, { message, detail }) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const { response } = await dialog.showMessageBox(win, {
    type: 'warning',
    title: 'คำเตือน',
    message: message,
    detail: detail,
    buttons: ['ดำเนินการต่อ', 'ยกเลิก'],
    defaultId: 1,   // ปุ่ม 'ยกเลิก' เป็น default
    cancelId: 1
  })
  
  // response = 0 ถ้ากด 'ดำเนินการต่อ', 1 ถ้ากด 'ยกเลิก'
  return { confirmed: response === 0 }
})

// 3. Confirm Dialog
ipcMain.handle('show-confirm', async (event, { title, message, detail }) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const { response } = await dialog.showMessageBox(win, {
    type: 'question',
    title: title || 'ยืนยัน',
    message: message,
    detail: detail || '',
    buttons: ['ใช่', 'ไม่'],
    defaultId: 0,
    cancelId: 1
  })
  
  return { confirmed: response === 0 }
})

// 4. Delete Confirmation with "Don't ask again" option
ipcMain.handle('confirm-delete', async (event, { itemName }) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const { response, checkboxChecked } = await dialog.showMessageBox(win, {
    type: 'warning',
    title: 'ยืนยันการลบ',
    message: `ต้องการลบ "${itemName}" ใช่หรือไม่?`,
    detail: 'การกระทำนี้ไม่สามารถยกเลิกได้',
    buttons: ['ลบ', 'ยกเลิก'],
    defaultId: 1,  // ยกเลิกเป็น default เพื่อความปลอดภัย
    cancelId: 1,
    checkboxLabel: 'อย่าถามอีก',
    checkboxChecked: false
  })
  
  return {
    confirmed: response === 0,
    dontAskAgain: checkboxChecked
  }
})

// 5. Choice Dialog (หลายตัวเลือก)
ipcMain.handle('show-choice', async (event, { title, message, choices }) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const { response } = await dialog.showMessageBox(win, {
    type: 'question',
    title: title,
    message: message,
    buttons: choices,
    defaultId: 0
  })
  
  return {
    selectedIndex: response,
    selectedValue: choices[response]
  }
})

app.whenReady().then(createWindow)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

---

## 8.2 showErrorBox - หน้าต่าง Error

```javascript
// main.js - ต่อเนื่อง

// showErrorBox เป็น synchronous และง่ายมาก
ipcMain.handle('show-error', (event, { title, content }) => {
  // showErrorBox ไม่รับ parent window เพราะ synchronous
  dialog.showErrorBox(title || 'เกิดข้อผิดพลาด', content)
  return { shown: true }
})

// ตัวอย่างการใช้งาน error dialog ใน main process
function handleCriticalError(error) {
  console.error('Critical Error:', error)
  
  dialog.showErrorBox(
    'เกิดข้อผิดพลาดร้ายแรง',
    `โปรแกรมพบปัญหาที่ไม่สามารถแก้ไขได้:\n\n${error.message}\n\nโปรแกรมจะปิดตัว`
  )
  
  app.exit(1)  // ออกด้วย error code 1
}

// Handle uncaught exceptions
process.on('uncaughtException', (error) => {
  handleCriticalError(error)
})

// Handle unhandled promise rejections
process.on('unhandledRejection', (reason) => {
  console.error('Unhandled Rejection:', reason)
  // แสดง error ที่ไม่ critical
  if (mainWindow && !mainWindow.isDestroyed()) {
    mainWindow.webContents.send('unhandled-error', reason.toString())
  }
})
```

---

## 8.3 showOpenDialog - เปิดไฟล์

```javascript
// main.js - Open Dialog

// ==========================================
// เปิดไฟล์เดียว
// ==========================================
ipcMain.handle('open-file', async (event, options = {}) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showOpenDialog(win, {
    title: options.title || 'เปิดไฟล์',
    defaultPath: options.defaultPath || app.getPath('documents'),
    buttonLabel: options.buttonLabel || 'เปิด',
    
    // กรองชนิดไฟล์
    filters: options.filters || [
      { name: 'ไฟล์ทั้งหมด', extensions: ['*'] }
    ],
    
    // properties
    properties: [
      'openFile',        // เปิดไฟล์ (ค่า default)
      // 'openDirectory' // เปิดโฟลเดอร์
      // 'multiSelections' // เลือกได้หลายไฟล์
      // 'showHiddenFiles' // แสดงไฟล์ซ่อน
      // 'createDirectory'  // macOS: สร้างโฟลเดอร์ใหม่
      // 'promptToCreate'   // Windows: สร้างไฟล์ถ้าไม่มี
      // 'noResolveAliases' // macOS: ไม่ resolve symlinks
      // 'treatPackageAsDirectory' // macOS: เปิด .app เป็น directory
      // 'dontAddToRecent'  // Windows: ไม่เพิ่มใน recent files
    ]
  })
  
  // result.canceled = true ถ้าผู้ใช้กด cancel
  // result.filePaths = array ของ paths ที่เลือก
  if (result.canceled) {
    return { canceled: true, filePath: null }
  }
  
  return {
    canceled: false,
    filePath: result.filePaths[0]  // ไฟล์แรก (กรณีเปิดไฟล์เดียว)
  }
})

// ==========================================
// เปิดหลายไฟล์
// ==========================================
ipcMain.handle('open-multiple-files', async (event, filters) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showOpenDialog(win, {
    title: 'เลือกไฟล์ภาพ',
    defaultPath: app.getPath('pictures'),
    filters: filters || [
      { name: 'ภาพ', extensions: ['jpg', 'jpeg', 'png', 'gif', 'webp', 'svg'] },
      { name: 'ไฟล์ทั้งหมด', extensions: ['*'] }
    ],
    properties: ['openFile', 'multiSelections']
  })
  
  if (result.canceled) {
    return { canceled: true, filePaths: [] }
  }
  
  // ส่ง metadata ของไฟล์กลับ
  const fileInfos = await Promise.all(
    result.filePaths.map(async (filePath) => {
      const fs = require('fs')
      const stats = fs.statSync(filePath)
      return {
        path: filePath,
        name: path.basename(filePath),
        size: stats.size,
        modified: stats.mtime
      }
    })
  )
  
  return {
    canceled: false,
    files: fileInfos
  }
})

// ==========================================
// เปิดโฟลเดอร์
// ==========================================
ipcMain.handle('open-folder', async (event) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showOpenDialog(win, {
    title: 'เลือกโฟลเดอร์',
    defaultPath: app.getPath('home'),
    properties: ['openDirectory', 'createDirectory']
  })
  
  if (result.canceled) {
    return { canceled: true, folderPath: null }
  }
  
  return {
    canceled: false,
    folderPath: result.filePaths[0]
  }
})

// ==========================================
// Open Dialog สำหรับไฟล์ประเภทต่างๆ
// ==========================================

// เปิดไฟล์ Text
ipcMain.handle('open-text-file', async (event) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showOpenDialog(win, {
    title: 'เปิดไฟล์ข้อความ',
    filters: [
      { name: 'ไฟล์ข้อความ', extensions: ['txt', 'md', 'csv', 'log'] },
      { name: 'JSON', extensions: ['json', 'jsonl'] },
      { name: 'ไฟล์ทั้งหมด', extensions: ['*'] }
    ],
    properties: ['openFile']
  })
  
  if (result.canceled) return { canceled: true }
  
  // อ่านเนื้อหาไฟล์
  const fs = require('fs')
  try {
    const content = fs.readFileSync(result.filePaths[0], 'utf-8')
    return {
      canceled: false,
      filePath: result.filePaths[0],
      content,
      size: Buffer.byteLength(content, 'utf8')
    }
  } catch (error) {
    throw new Error(`อ่านไฟล์ล้มเหลว: ${error.message}`)
  }
})
```

---

## 8.4 showSaveDialog - บันทึกไฟล์

```javascript
// main.js - Save Dialog

// ==========================================
// บันทึกไฟล์พื้นฐาน
// ==========================================
ipcMain.handle('save-file', async (event, { defaultName, content, filters }) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showSaveDialog(win, {
    title: 'บันทึกไฟล์',
    defaultPath: path.join(app.getPath('documents'), defaultName || 'untitled.txt'),
    buttonLabel: 'บันทึก',
    filters: filters || [
      { name: 'ไฟล์ข้อความ', extensions: ['txt'] },
      { name: 'ไฟล์ทั้งหมด', extensions: ['*'] }
    ],
    properties: [
      // 'showHiddenFiles',
      // 'createDirectory',  // macOS
      // 'treatPackageAsDirectory',  // macOS
      // 'showOverwriteConfirmation', // Linux
      // 'dontAddToRecent'  // Windows
    ]
  })
  
  if (result.canceled) {
    return { canceled: true, filePath: null }
  }
  
  // บันทึกไฟล์จริง
  const fs = require('fs')
  try {
    fs.writeFileSync(result.filePath, content, 'utf-8')
    return {
      canceled: false,
      filePath: result.filePath,
      success: true
    }
  } catch (error) {
    throw new Error(`บันทึกไฟล์ล้มเหลว: ${error.message}`)
  }
})

// ==========================================
// บันทึกรูปภาพ
// ==========================================
ipcMain.handle('save-image', async (event, { defaultName, imageDataUrl }) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showSaveDialog(win, {
    title: 'บันทึกรูปภาพ',
    defaultPath: path.join(app.getPath('pictures'), defaultName || 'image.png'),
    filters: [
      { name: 'PNG', extensions: ['png'] },
      { name: 'JPEG', extensions: ['jpg', 'jpeg'] },
      { name: 'WebP', extensions: ['webp'] }
    ]
  })
  
  if (result.canceled) return { canceled: true }
  
  // แปลง data URL เป็น binary
  const fs = require('fs')
  const base64Data = imageDataUrl.replace(/^data:image\/\w+;base64,/, '')
  const buffer = Buffer.from(base64Data, 'base64')
  
  try {
    fs.writeFileSync(result.filePath, buffer)
    return {
      canceled: false,
      filePath: result.filePath,
      success: true
    }
  } catch (error) {
    throw new Error(`บันทึกรูปภาพล้มเหลว: ${error.message}`)
  }
})

// ==========================================
// Export เป็น PDF
// ==========================================
ipcMain.handle('export-pdf', async (event, { defaultName }) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  const result = await dialog.showSaveDialog(win, {
    title: 'ส่งออกเป็น PDF',
    defaultPath: path.join(app.getPath('documents'), defaultName || 'document.pdf'),
    filters: [
      { name: 'PDF', extensions: ['pdf'] }
    ]
  })
  
  if (result.canceled) return { canceled: true }
  
  // Print หน้าเป็น PDF
  try {
    const pdfData = await win.webContents.printToPDF({
      marginsType: 0,        // 0 = default, 1 = none, 2 = minimum
      pageSize: 'A4',        // 'A3', 'A4', 'A5', 'Legal', 'Letter', 'Tabloid'
      printBackground: true,
      landscape: false
    })
    
    require('fs').writeFileSync(result.filePath, pdfData)
    
    return {
      canceled: false,
      filePath: result.filePath,
      success: true
    }
  } catch (error) {
    throw new Error(`ส่งออก PDF ล้มเหลว: ${error.message}`)
  }
})
```

---

## 8.5 Async Dialogs - Pattern ที่ถูกต้อง

```javascript
// main.js - Async Dialog Patterns

// ==========================================
// Pattern 1: Simple async/await
// ==========================================
async function askUserToSave() {
  const { response } = await dialog.showMessageBox(mainWindow, {
    type: 'question',
    title: 'บันทึกการเปลี่ยนแปลง?',
    message: 'มีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก',
    detail: 'ต้องการบันทึกก่อนปิดไหม?',
    buttons: ['บันทึก', 'ไม่บันทึก', 'ยกเลิก'],
    defaultId: 0,
    cancelId: 2
  })
  
  // 0 = บันทึก, 1 = ไม่บันทึก, 2 = ยกเลิก
  return ['save', 'discard', 'cancel'][response]
}

// ใช้ในกระบวนการปิดหน้าต่าง
function setupWindowCloseHandler() {
  mainWindow.on('close', async (event) => {
    if (!hasUnsavedChanges) return  // ปิดได้เลย
    
    // ป้องกันการปิด
    event.preventDefault()
    
    const action = await askUserToSave()
    
    switch (action) {
      case 'save':
        await saveCurrentFile()
        mainWindow.destroy()
        break
        
      case 'discard':
        mainWindow.destroy()
        break
        
      case 'cancel':
        // ไม่ทำอะไร หน้าต่างยังเปิดอยู่
        break
    }
  })
}

// ==========================================
// Pattern 2: Dialog Queue (ป้องกัน dialog ซ้อนกัน)
// ==========================================
class DialogQueue {
  constructor() {
    this.queue = []
    this.isShowing = false
  }
  
  async show(options) {
    return new Promise((resolve) => {
      this.queue.push({ options, resolve })
      this.processNext()
    })
  }
  
  async processNext() {
    if (this.isShowing || this.queue.length === 0) return
    
    this.isShowing = true
    const { options, resolve } = this.queue.shift()
    
    const result = await dialog.showMessageBox(mainWindow, options)
    this.isShowing = false
    
    resolve(result)
    this.processNext()  // ประมวลผล dialog ถัดไป
  }
}

const dialogQueue = new DialogQueue()

// ใช้งาน
ipcMain.handle('show-queued-dialog', async (event, options) => {
  return dialogQueue.show(options)
})

// ==========================================
// Pattern 3: Dialog Result Handler
// ==========================================
const dialogHandlers = {
  'confirm-delete': async (event, data) => {
    const win = BrowserWindow.fromWebContents(event.sender)
    const { response } = await dialog.showMessageBox(win, {
      type: 'warning',
      title: 'ยืนยันการลบ',
      message: `ลบ "${data.name}" ใช่หรือไม่?`,
      buttons: ['ลบ', 'ยกเลิก'],
      defaultId: 1,
      cancelId: 1
    })
    return { confirmed: response === 0 }
  },
  
  'confirm-overwrite': async (event, data) => {
    const win = BrowserWindow.fromWebContents(event.sender)
    const { response } = await dialog.showMessageBox(win, {
      type: 'question',
      title: 'ไฟล์มีอยู่แล้ว',
      message: `"${data.fileName}" มีอยู่แล้ว`,
      detail: 'ต้องการเขียนทับไหม?',
      buttons: ['เขียนทับ', 'เปลี่ยนชื่อ', 'ยกเลิก'],
      defaultId: 2,
      cancelId: 2
    })
    return { action: ['overwrite', 'rename', 'cancel'][response] }
  }
}

ipcMain.handle('show-dialog', async (event, { type, data }) => {
  const handler = dialogHandlers[type]
  if (!handler) throw new Error(`Unknown dialog type: ${type}`)
  return handler(event, data)
})
```

---

## 8.6 Custom Dialog ด้วย HTML

บางครั้ง native dialogs ไม่เพียงพอ เราสามารถสร้าง custom dialog ด้วย BrowserWindow:

```javascript
// main.js - Custom Dialog

class CustomDialog {
  constructor() {
    this.windows = new Map()
  }
  
  // สร้าง custom dialog window
  async show(type, data) {
    return new Promise((resolve) => {
      const dialogId = `dialog_${Date.now()}`
      
      const dialogWindow = new BrowserWindow({
        width: 400,
        height: 300,
        resizable: false,
        minimizable: false,
        maximizable: false,
        parent: mainWindow,
        modal: true,         // ล็อค parent window
        show: false,         // ไม่แสดงจนกว่าจะพร้อม
        frame: false,        // ซ่อน title bar (optional)
        webPreferences: {
          preload: path.join(__dirname, 'dialog-preload.js'),
          nodeIntegration: false,
          contextIsolation: true
        }
      })
      
      // เก็บ resolve function
      this.windows.set(dialogId, { window: dialogWindow, resolve })
      
      // ส่งข้อมูลไป dialog ผ่าน query string
      const url = new URL(`file://${path.join(__dirname, 'dialogs', `${type}.html`)}`)
      url.searchParams.set('dialogId', dialogId)
      url.searchParams.set('data', JSON.stringify(data))
      
      dialogWindow.loadURL(url.toString())
      
      // แสดงเมื่อพร้อม
      dialogWindow.once('ready-to-show', () => {
        dialogWindow.show()
      })
      
      // ถ้าปิด window โดยไม่ตอบ
      dialogWindow.on('closed', () => {
        const entry = this.windows.get(dialogId)
        if (entry) {
          this.windows.delete(dialogId)
          resolve({ canceled: true })
        }
      })
    })
  }
  
  // รับผลลัพธ์จาก dialog
  handleResult(dialogId, result) {
    const entry = this.windows.get(dialogId)
    if (!entry) return
    
    const { window, resolve } = entry
    this.windows.delete(dialogId)
    
    window.close()
    resolve(result)
  }
}

const customDialog = new CustomDialog()

// IPC: แสดง custom dialog
ipcMain.handle('show-custom-dialog', async (event, { type, data }) => {
  return customDialog.show(type, data)
})

// IPC: รับผลลัพธ์จาก dialog
ipcMain.on('dialog-result', (event, { dialogId, result }) => {
  customDialog.handleResult(dialogId, result)
})
```

```javascript
// dialog-preload.js - Preload สำหรับ custom dialog
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('dialogAPI', {
  // ส่งผลลัพธ์กลับไปหา main
  submitResult: (dialogId, result) => {
    ipcRenderer.send('dialog-result', { dialogId, result })
  },
  
  // ปิด dialog
  close: (dialogId) => {
    ipcRenderer.send('dialog-result', { dialogId, result: { canceled: true } })
  }
})
```

```html
<!-- dialogs/confirm-delete.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>ยืนยันการลบ</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    
    body {
      font-family: 'Sarabun', -apple-system, sans-serif;
      background: #fff;
      display: flex;
      flex-direction: column;
      height: 100vh;
      overflow: hidden;
    }
    
    /* Draggable title bar */
    .title-bar {
      background: #f5f5f5;
      padding: 12px 16px;
      border-bottom: 1px solid #e0e0e0;
      -webkit-app-region: drag;  /* ให้ลากหน้าต่างได้ */
      display: flex;
      align-items: center;
      gap: 8px;
    }
    
    .title-bar .icon {
      color: #f44336;
      font-size: 20px;
    }
    
    .title-bar h1 {
      font-size: 15px;
      font-weight: 600;
      color: #333;
    }
    
    .content {
      flex: 1;
      padding: 20px;
      display: flex;
      flex-direction: column;
      justify-content: center;
    }
    
    .message {
      font-size: 16px;
      font-weight: 600;
      color: #333;
      margin-bottom: 8px;
    }
    
    .detail {
      font-size: 14px;
      color: #666;
      margin-bottom: 16px;
    }
    
    .item-name {
      background: #fff3e0;
      border: 1px solid #ffcc02;
      border-radius: 4px;
      padding: 8px 12px;
      font-size: 14px;
      color: #e65100;
      word-break: break-all;
    }
    
    .checkbox-row {
      display: flex;
      align-items: center;
      gap: 8px;
      margin-top: 12px;
      font-size: 13px;
      color: #555;
    }
    
    .buttons {
      display: flex;
      justify-content: flex-end;
      gap: 8px;
      padding: 12px 16px;
      border-top: 1px solid #e0e0e0;
      background: #f9f9f9;
    }
    
    button {
      padding: 8px 20px;
      border-radius: 4px;
      border: none;
      cursor: pointer;
      font-size: 14px;
      font-family: inherit;
    }
    
    .btn-cancel {
      background: #f0f0f0;
      color: #333;
    }
    
    .btn-cancel:hover { background: #e0e0e0; }
    
    .btn-delete {
      background: #f44336;
      color: white;
    }
    
    .btn-delete:hover { background: #d32f2f; }
  </style>
</head>
<body>
  <div class="title-bar">
    <span class="icon">⚠️</span>
    <h1>ยืนยันการลบ</h1>
  </div>
  
  <div class="content">
    <p class="message">ต้องการลบรายการนี้ใช่หรือไม่?</p>
    <div class="item-name" id="itemName">กำลังโหลด...</div>
    <p class="detail" style="margin-top: 8px;">การกระทำนี้ไม่สามารถยกเลิกได้</p>
    
    <label class="checkbox-row">
      <input type="checkbox" id="dontAsk">
      อย่าถามอีกสำหรับรายการประเภทนี้
    </label>
  </div>
  
  <div class="buttons">
    <button class="btn-cancel" id="cancelBtn">ยกเลิก</button>
    <button class="btn-delete" id="deleteBtn">ลบ</button>
  </div>
  
  <script>
    // รับข้อมูลจาก URL params
    const urlParams = new URLSearchParams(window.location.search)
    const dialogId = urlParams.get('dialogId')
    const data = JSON.parse(urlParams.get('data') || '{}')
    
    // แสดงชื่อ item
    document.getElementById('itemName').textContent = data.name || 'ไม่ระบุชื่อ'
    
    // ปุ่มยกเลิก
    document.getElementById('cancelBtn').addEventListener('click', () => {
      window.dialogAPI.submitResult(dialogId, { confirmed: false })
    })
    
    // ปุ่มลบ
    document.getElementById('deleteBtn').addEventListener('click', () => {
      const dontAsk = document.getElementById('dontAsk').checked
      window.dialogAPI.submitResult(dialogId, {
        confirmed: true,
        dontAskAgain: dontAsk
      })
    })
    
    // กด Escape = ยกเลิก
    document.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') {
        window.dialogAPI.submitResult(dialogId, { confirmed: false })
      }
      if (e.key === 'Enter') {
        document.getElementById('deleteBtn').click()
      }
    })
  </script>
</body>
</html>
```

---

## 8.7 Preload และ Renderer สำหรับ Dialogs

```javascript
// preload.js - Dialog API

const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('dialogAPI', {
  // Message boxes
  showInfo: (title, message) =>
    ipcRenderer.invoke('show-info', { title, message }),
  
  showWarning: (message, detail) =>
    ipcRenderer.invoke('show-warning', { message, detail }),
  
  showError: (title, content) =>
    ipcRenderer.invoke('show-error', { title, content }),
  
  showConfirm: (title, message, detail) =>
    ipcRenderer.invoke('show-confirm', { title, message, detail }),
  
  confirmDelete: (itemName) =>
    ipcRenderer.invoke('confirm-delete', { itemName }),
  
  showChoice: (title, message, choices) =>
    ipcRenderer.invoke('show-choice', { title, message, choices }),
  
  // File dialogs
  openFile: (options) =>
    ipcRenderer.invoke('open-file', options),
  
  openMultipleFiles: (filters) =>
    ipcRenderer.invoke('open-multiple-files', filters),
  
  openFolder: () =>
    ipcRenderer.invoke('open-folder'),
  
  openTextFile: () =>
    ipcRenderer.invoke('open-text-file'),
  
  saveFile: (defaultName, content, filters) =>
    ipcRenderer.invoke('save-file', { defaultName, content, filters }),
  
  saveImage: (defaultName, imageDataUrl) =>
    ipcRenderer.invoke('save-image', { defaultName, imageDataUrl }),
  
  exportPDF: (defaultName) =>
    ipcRenderer.invoke('export-pdf', { defaultName }),
  
  // Custom dialogs
  showCustomDialog: (type, data) =>
    ipcRenderer.invoke('show-custom-dialog', { type, data })
})
```

```javascript
// renderer.js - Demo การใช้ Dialog APIs

// =============================================
// Message Box Demos
// =============================================
document.getElementById('btnInfo').addEventListener('click', async () => {
  await window.dialogAPI.showInfo(
    'ข้อมูล',
    'นี่คือข้อความแจ้งเตือนธรรมดา'
  )
  logResult('แสดง info dialog เสร็จ')
})

document.getElementById('btnWarning').addEventListener('click', async () => {
  const result = await window.dialogAPI.showWarning(
    'ข้อมูลจะสูญหาย',
    'กรุณาบันทึกงานก่อนดำเนินการต่อ'
  )
  logResult(`Warning result: ${JSON.stringify(result)}`)
})

document.getElementById('btnConfirm').addEventListener('click', async () => {
  const result = await window.dialogAPI.showConfirm(
    'ยืนยัน',
    'ต้องการดำเนินการต่อใช่หรือไม่?',
    'การกระทำนี้จะเปลี่ยนแปลงข้อมูลของคุณ'
  )
  
  if (result.confirmed) {
    logResult('ผู้ใช้ยืนยัน - ดำเนินการต่อ')
  } else {
    logResult('ผู้ใช้ยกเลิก - หยุดการดำเนินการ')
  }
})

document.getElementById('btnDelete').addEventListener('click', async () => {
  const result = await window.dialogAPI.confirmDelete('เอกสาร_สำคัญ.docx')
  
  if (result.confirmed) {
    logResult(`ลบรายการ${result.dontAskAgain ? ' (ไม่ถามอีก)' : ''}`)
  } else {
    logResult('ยกเลิกการลบ')
  }
})

document.getElementById('btnChoice').addEventListener('click', async () => {
  const result = await window.dialogAPI.showChoice(
    'เลือกรูปแบบการส่งออก',
    'ต้องการส่งออกไฟล์ในรูปแบบใด?',
    ['PDF', 'Word Document', 'CSV', 'ยกเลิก']
  )
  
  if (result.selectedValue !== 'ยกเลิก') {
    logResult(`เลือก: ${result.selectedValue}`)
  }
})

// =============================================
// File Dialog Demos
// =============================================
document.getElementById('btnOpenFile').addEventListener('click', async () => {
  const result = await window.dialogAPI.openFile({
    title: 'เลือกไฟล์',
    filters: [
      { name: 'Text Files', extensions: ['txt', 'md'] },
      { name: 'All Files', extensions: ['*'] }
    ]
  })
  
  if (!result.canceled) {
    logResult(`เปิดไฟล์: ${result.filePath}`)
  }
})

document.getElementById('btnOpenImages').addEventListener('click', async () => {
  const result = await window.dialogAPI.openMultipleFiles([
    { name: 'Images', extensions: ['jpg', 'png', 'gif', 'webp'] }
  ])
  
  if (!result.canceled) {
    logResult(`เลือกรูป ${result.files.length} ไฟล์`)
    
    // แสดงรูปภาพ
    const gallery = document.getElementById('imageGallery')
    gallery.innerHTML = ''
    result.files.forEach(file => {
      const img = document.createElement('img')
      img.src = `file://${file.path}`
      img.style.cssText = 'width:100px;height:100px;object-fit:cover;margin:5px;border-radius:4px;'
      gallery.appendChild(img)
    })
  }
})

document.getElementById('btnOpenTextFile').addEventListener('click', async () => {
  const result = await window.dialogAPI.openTextFile()
  
  if (!result.canceled) {
    document.getElementById('textContent').value = result.content
    logResult(`อ่านไฟล์: ${result.filePath} (${result.size} bytes)`)
  }
})

document.getElementById('btnSaveFile').addEventListener('click', async () => {
  const content = document.getElementById('textContent').value
  
  if (!content) {
    await window.dialogAPI.showWarning('ไม่มีเนื้อหา', 'กรุณากรอกเนื้อหาก่อนบันทึก')
    return
  }
  
  const result = await window.dialogAPI.saveFile(
    'document.txt',
    content,
    [{ name: 'Text Files', extensions: ['txt'] }]
  )
  
  if (!result.canceled) {
    logResult(`บันทึกไฟล์: ${result.filePath}`)
  }
})

document.getElementById('btnExportPDF').addEventListener('click', async () => {
  const result = await window.dialogAPI.exportPDF('รายงาน.pdf')
  
  if (!result.canceled) {
    logResult(`ส่งออก PDF: ${result.filePath}`)
    
    // เปิดไฟล์หลังบันทึก
    const openResult = await window.dialogAPI.showChoice(
      'บันทึกสำเร็จ',
      'ต้องการเปิดไฟล์ PDF ที่บันทึกไว้หรือไม่?',
      ['เปิด', 'ปิด']
    )
    
    if (openResult.selectedValue === 'เปิด') {
      // เปิดไฟล์ด้วยโปรแกรมเริ่มต้น
      require('electron').shell?.openPath?.(result.filePath)
    }
  }
})

// =============================================
// Custom Dialog Demo
// =============================================
document.getElementById('btnCustomDialog').addEventListener('click', async () => {
  const result = await window.dialogAPI.showCustomDialog('confirm-delete', {
    name: 'ไฟล์สำคัญ.pdf'
  })
  
  logResult(`Custom dialog result: ${JSON.stringify(result)}`)
})

// =============================================
// Helper function
// =============================================
function logResult(message) {
  const log = document.getElementById('resultLog')
  const time = new Date().toLocaleTimeString('th-TH')
  const entry = document.createElement('div')
  entry.textContent = `[${time}] ${message}`
  entry.style.marginBottom = '4px'
  log.insertBefore(entry, log.firstChild)
  
  // จำกัด 20 รายการ
  while (log.children.length > 20) {
    log.removeChild(log.lastChild)
  }
}
```

---

## 8.8 index.html - UI ครบถ้วน

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Dialog Demo</title>
  <style>
    body {
      font-family: 'Sarabun', sans-serif;
      max-width: 900px;
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
    
    h2 {
      color: #333;
      border-bottom: 2px solid #2196f3;
      padding-bottom: 8px;
      margin-bottom: 16px;
    }
    
    .btn-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
      gap: 8px;
      margin-bottom: 16px;
    }
    
    button {
      padding: 10px 16px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 14px;
      font-family: inherit;
      transition: opacity 0.2s;
    }
    
    button:hover { opacity: 0.85; }
    
    .btn-info { background: #2196f3; color: white; }
    .btn-warning { background: #ff9800; color: white; }
    .btn-error { background: #f44336; color: white; }
    .btn-success { background: #4caf50; color: white; }
    .btn-primary { background: #9c27b0; color: white; }
    .btn-secondary { background: #607d8b; color: white; }
    
    #imageGallery {
      display: flex;
      flex-wrap: wrap;
      gap: 8px;
      min-height: 50px;
      padding: 8px;
      background: #f9f9f9;
      border-radius: 4px;
      border: 1px dashed #ddd;
      margin: 10px 0;
    }
    
    #textContent {
      width: 100%;
      height: 120px;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 4px;
      font-size: 14px;
      font-family: 'Sarabun', sans-serif;
      resize: vertical;
      box-sizing: border-box;
    }
    
    #resultLog {
      background: #1e1e1e;
      color: #d4d4d4;
      padding: 12px;
      border-radius: 4px;
      font-family: monospace;
      font-size: 13px;
      min-height: 100px;
      max-height: 200px;
      overflow-y: auto;
    }
  </style>
</head>
<body>
  <h1>Dialog Boxes Demo</h1>
  
  <!-- Message Boxes -->
  <div class="section">
    <h2>Message Boxes</h2>
    <div class="btn-grid">
      <button class="btn-info" id="btnInfo">Info Dialog</button>
      <button class="btn-warning" id="btnWarning">Warning Dialog</button>
      <button class="btn-primary" id="btnConfirm">Confirm Dialog</button>
      <button class="btn-error" id="btnDelete">Delete Confirm</button>
      <button class="btn-secondary" id="btnChoice">Choice Dialog</button>
    </div>
  </div>
  
  <!-- File Dialogs -->
  <div class="section">
    <h2>File Dialogs</h2>
    <div class="btn-grid">
      <button class="btn-info" id="btnOpenFile">เปิดไฟล์</button>
      <button class="btn-success" id="btnOpenImages">เปิดรูปภาพ</button>
      <button class="btn-info" id="btnOpenTextFile">เปิดไฟล์ข้อความ</button>
    </div>
    
    <div id="imageGallery">
      <span style="color:#999;align-self:center">รูปภาพจะแสดงที่นี่</span>
    </div>
    
    <textarea id="textContent" placeholder="เนื้อหาไฟล์ข้อความจะแสดงที่นี่..."></textarea>
    
    <div class="btn-grid" style="margin-top:8px">
      <button class="btn-success" id="btnSaveFile">บันทึกไฟล์</button>
      <button class="btn-primary" id="btnExportPDF">ส่งออก PDF</button>
    </div>
  </div>
  
  <!-- Custom Dialog -->
  <div class="section">
    <h2>Custom HTML Dialog</h2>
    <button class="btn-error" id="btnCustomDialog">แสดง Custom Dialog</button>
  </div>
  
  <!-- Result Log -->
  <div class="section">
    <h2>ผลลัพธ์</h2>
    <div id="resultLog">รอการดำเนินการ...</div>
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

---

## สรุปบทเรียน

### ตารางเปรียบเทียบ Dialog Types

| Dialog | ใช้เมื่อ | คืนค่า |
|--------|---------|--------|
| `showMessageBox` | แสดงข้อความ / ยืนยัน | `{ response, checkboxChecked }` |
| `showErrorBox` | แสดง error (blocking) | `void` |
| `showOpenDialog` | เลือกไฟล์/โฟลเดอร์ | `{ canceled, filePaths }` |
| `showSaveDialog` | เลือกที่บันทึก | `{ canceled, filePath }` |
| Custom Dialog | UI ที่ซับซ้อน | ตามที่กำหนด |

### Best Practices

1. **ใช้ async version เสมอ** - Sync version จะ block UI thread
2. **ระบุ parent window** - ทำให้ dialog เป็น modal ของ window ที่ถูกต้อง
3. **ตั้ง `cancelId`** - กำหนดปุ่มที่ทำงานเมื่อกด Escape
4. **ตั้ง `defaultId`** - กำหนดปุ่ม default ที่ทำงานเมื่อกด Enter
5. **ใช้ `detail`** - ให้ข้อมูลเพิ่มเติมนอกจาก message หลัก
6. **Custom dialog** สำหรับ UI ที่ซับซ้อนกว่า native dialog รองรับ
