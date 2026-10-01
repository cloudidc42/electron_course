# ตอนที่ 7: เมนูและ Context Menus

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจากศึกษาตอนนี้ คุณจะสามารถ:
- สร้างเมนูแถบด้านบน (Application Menu) ด้วย `Menu.buildFromTemplate`
- ใช้ role shortcuts ที่มีอยู่ใน Electron
- สร้าง submenu และ separator ได้
- ใช้ checkbox และ radio items ในเมนู
- สร้าง context menu เมื่อคลิกขวา
- สร้างเมนูแบบ dynamic ที่เปลี่ยนแปลงได้
- จัดการ platform-specific menus (macOS, Windows, Linux)
- สร้าง macOS app menu ที่ถูกต้อง

---

## บทนำ: เมนูใน Electron

เมนูใน Electron มี 2 ประเภทหลัก:

```
Application Menu (เมนูบาร์)
├── File
│   ├── New
│   ├── Open
│   └── Exit
├── Edit
│   ├── Undo
│   └── Redo
└── Help
    └── About

Context Menu (คลิกขวา)
├── Copy
├── Paste
└── Properties
```

---

## 7.1 การสร้างเมนูพื้นฐาน

### main.js - เมนูพื้นฐาน
```javascript
const { app, BrowserWindow, Menu, MenuItem } = require('electron')
const path = require('path')

let mainWindow

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1000,
    height: 700,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  
  mainWindow.loadFile('index.html')
  createApplicationMenu()
}

function createApplicationMenu() {
  // Template สำหรับสร้างเมนู
  const template = [
    {
      label: 'ไฟล์',  // ชื่อเมนู
      submenu: [
        {
          label: 'ใหม่',
          accelerator: 'CmdOrCtrl+N',  // keyboard shortcut
          click: () => {
            console.log('คลิก: ไฟล์ใหม่')
            createNewDocument()
          }
        },
        {
          label: 'เปิด',
          accelerator: 'CmdOrCtrl+O',
          click: () => openFile()
        },
        {
          label: 'บันทึก',
          accelerator: 'CmdOrCtrl+S',
          click: () => saveFile()
        },
        {
          label: 'บันทึกเป็น',
          accelerator: 'CmdOrCtrl+Shift+S',
          click: () => saveFileAs()
        },
        { type: 'separator' },  // เส้นแบ่ง
        {
          label: 'ออกจากโปรแกรม',
          accelerator: process.platform === 'darwin' ? 'Cmd+Q' : 'Ctrl+Q',
          click: () => app.quit()
        }
      ]
    },
    {
      label: 'แก้ไข',
      submenu: [
        {
          label: 'เลิกทำ',
          role: 'undo'  // ใช้ role ที่มีใน Electron
        },
        {
          label: 'ทำซ้ำ',
          role: 'redo'
        },
        { type: 'separator' },
        {
          label: 'ตัด',
          role: 'cut'
        },
        {
          label: 'คัดลอก',
          role: 'copy'
        },
        {
          label: 'วาง',
          role: 'paste'
        },
        {
          label: 'เลือกทั้งหมด',
          role: 'selectAll'
        }
      ]
    },
    {
      label: 'มุมมอง',
      submenu: [
        {
          label: 'โหลดใหม่',
          role: 'reload'
        },
        {
          label: 'โหลดใหม่แบบ Force',
          role: 'forceReload'
        },
        {
          label: 'เปิด DevTools',
          role: 'toggleDevTools'
        },
        { type: 'separator' },
        {
          label: 'ขนาดจริง',
          role: 'resetZoom'
        },
        {
          label: 'ขยาย',
          role: 'zoomIn'
        },
        {
          label: 'ย่อ',
          role: 'zoomOut'
        },
        { type: 'separator' },
        {
          label: 'เต็มหน้าจอ',
          role: 'togglefullscreen'
        }
      ]
    },
    {
      label: 'ช่วยเหลือ',
      submenu: [
        {
          label: 'เกี่ยวกับ',
          click: () => showAboutDialog()
        },
        {
          label: 'ตรวจสอบอัพเดท',
          click: () => checkForUpdates()
        },
        { type: 'separator' },
        {
          label: 'รายงานปัญหา',
          click: () => {
            require('electron').shell.openExternal('https://github.com/example/issues')
          }
        }
      ]
    }
  ]
  
  // สำหรับ macOS ต้องเพิ่ม app menu ก่อน
  if (process.platform === 'darwin') {
    template.unshift(createMacAppMenu())
  }
  
  // สร้างและตั้งค่าเมนู
  const menu = Menu.buildFromTemplate(template)
  Menu.setApplicationMenu(menu)
}

// ฟังก์ชัน placeholder
function createNewDocument() { mainWindow.webContents.send('menu-new') }
function openFile() { mainWindow.webContents.send('menu-open') }
function saveFile() { mainWindow.webContents.send('menu-save') }
function saveFileAs() { mainWindow.webContents.send('menu-save-as') }
function showAboutDialog() { mainWindow.webContents.send('menu-about') }
function checkForUpdates() { mainWindow.webContents.send('menu-check-updates') }

app.whenReady().then(createWindow)

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

---

## 7.2 Roles ที่มีใน Electron

Roles คือ shortcut สำหรับ actions ที่ใช้บ่อย ไม่ต้องเขียน click handler เอง:

```javascript
// รายการ roles ทั้งหมด

// == Text Operations ==
{ role: 'undo' }           // Ctrl+Z / Cmd+Z
{ role: 'redo' }           // Ctrl+Shift+Z / Cmd+Shift+Z
{ role: 'cut' }            // Ctrl+X / Cmd+X
{ role: 'copy' }           // Ctrl+C / Cmd+C
{ role: 'paste' }          // Ctrl+V / Cmd+V
{ role: 'pasteAndMatchStyle' }  // Cmd+Option+Shift+V (macOS)
{ role: 'delete' }
{ role: 'selectAll' }      // Ctrl+A / Cmd+A

// == View Operations ==
{ role: 'reload' }         // Ctrl+R / Cmd+R
{ role: 'forceReload' }    // Ctrl+Shift+R / Cmd+Shift+R
{ role: 'toggleDevTools' } // F12 / Cmd+Option+I
{ role: 'resetZoom' }      // Ctrl+0 / Cmd+0
{ role: 'zoomIn' }         // Ctrl+Plus / Cmd+Plus
{ role: 'zoomOut' }        // Ctrl+Minus / Cmd+Minus
{ role: 'togglefullscreen' }  // F11

// == Window Operations ==
{ role: 'minimize' }
{ role: 'zoom' }           // macOS only
{ role: 'close' }
{ role: 'quit' }

// == macOS Specific ==
{ role: 'about' }          // macOS App > About
{ role: 'hide' }           // macOS: hide app
{ role: 'hideOthers' }
{ role: 'unhide' }
{ role: 'front' }          // ย้าย windows มาข้างหน้า
{ role: 'window' }         // macOS Window menu
{ role: 'help' }           // macOS Help menu
{ role: 'services' }       // macOS Services
{ role: 'recentDocuments' }  // macOS Open Recent
{ role: 'clearRecentDocuments' }
{ role: 'toggleTabBar' }   // macOS Tab Bar
{ role: 'selectNextTab' }
{ role: 'selectPreviousTab' }
{ role: 'mergeAllWindows' }
{ role: 'moveTabToNewWindow' }
{ role: 'showSubstitutions' }
{ role: 'toggleSmartQuotes' }
{ role: 'toggleSmartDashes' }
{ role: 'toggleTextReplacement' }

// ตัวอย่างการใช้งาน roles ร่วมกับ label
{
  label: 'คัดลอก (Copy)',  // กำหนด label เองได้แม้ใช้ role
  role: 'copy',
  accelerator: 'CmdOrCtrl+C'  // กำหนด shortcut เองก็ได้
}
```

---

## 7.3 Checkbox และ Radio Items

### Checkbox Menu Items
```javascript
// main.js - เมนูแบบ checkbox

// เก็บ state ของ settings
const appSettings = {
  darkMode: false,
  notifications: true,
  autoSave: false,
  spellCheck: true
}

function createSettingsMenu() {
  return {
    label: 'การตั้งค่า',
    submenu: [
      {
        label: 'โหมดมืด',
        type: 'checkbox',            // ชนิด checkbox
        checked: appSettings.darkMode,  // state เริ่มต้น
        click: (menuItem) => {
          // menuItem.checked จะอัพเดทอัตโนมัติ
          appSettings.darkMode = menuItem.checked
          console.log('Dark mode:', menuItem.checked)
          
          // แจ้ง renderer
          mainWindow.webContents.send('setting-changed', {
            key: 'darkMode',
            value: menuItem.checked
          })
        }
      },
      {
        label: 'แจ้งเตือน',
        type: 'checkbox',
        checked: appSettings.notifications,
        click: (menuItem) => {
          appSettings.notifications = menuItem.checked
          mainWindow.webContents.send('setting-changed', {
            key: 'notifications',
            value: menuItem.checked
          })
        }
      },
      {
        label: 'บันทึกอัตโนมัติ',
        type: 'checkbox',
        checked: appSettings.autoSave,
        click: (menuItem) => {
          appSettings.autoSave = menuItem.checked
        }
      },
      { type: 'separator' },
      {
        label: 'ตรวจสอบการสะกด',
        type: 'checkbox',
        checked: appSettings.spellCheck,
        click: (menuItem) => {
          appSettings.spellCheck = menuItem.checked
          // เปิด/ปิด spell check
          mainWindow.webContents.session.setSpellCheckerEnabled(menuItem.checked)
        }
      }
    ]
  }
}
```

### Radio Menu Items
```javascript
// main.js - เมนูแบบ radio (เลือกได้อย่างเดียว)

let currentTheme = 'system'  // 'light', 'dark', 'system'
let currentLanguage = 'th'   // 'th', 'en', 'ja'

function createThemeMenu() {
  return {
    label: 'ธีม',
    submenu: [
      {
        label: 'สว่าง',
        type: 'radio',
        checked: currentTheme === 'light',
        click: (menuItem) => {
          if (menuItem.checked) {
            currentTheme = 'light'
            applyTheme('light')
          }
        }
      },
      {
        label: 'มืด',
        type: 'radio',
        checked: currentTheme === 'dark',
        click: (menuItem) => {
          if (menuItem.checked) {
            currentTheme = 'dark'
            applyTheme('dark')
          }
        }
      },
      {
        label: 'ตามระบบ',
        type: 'radio',
        checked: currentTheme === 'system',
        click: (menuItem) => {
          if (menuItem.checked) {
            currentTheme = 'system'
            applyTheme('system')
          }
        }
      }
    ]
  }
}

function createLanguageMenu() {
  const languages = [
    { code: 'th', label: 'ภาษาไทย' },
    { code: 'en', label: 'English' },
    { code: 'ja', label: '日本語' },
    { code: 'zh', label: '中文' }
  ]
  
  return {
    label: 'ภาษา',
    submenu: languages.map(lang => ({
      label: lang.label,
      type: 'radio',
      checked: currentLanguage === lang.code,
      click: (menuItem) => {
        if (menuItem.checked) {
          currentLanguage = lang.code
          changeLanguage(lang.code)
        }
      }
    }))
  }
}

function applyTheme(theme) {
  mainWindow.webContents.send('theme-changed', theme)
}

function changeLanguage(code) {
  mainWindow.webContents.send('language-changed', code)
}
```

---

## 7.4 Dynamic Menus - เมนูที่เปลี่ยนแปลงได้

```javascript
// main.js - dynamic menus

const { ipcMain, Menu } = require('electron')

// เก็บรายการไฟล์ล่าสุด
let recentFiles = []

// อัพเดท recent files
function addRecentFile(filePath) {
  // ลบถ้ามีอยู่แล้ว
  recentFiles = recentFiles.filter(f => f !== filePath)
  // เพิ่มที่ต้น
  recentFiles.unshift(filePath)
  // จำกัดแค่ 10 ไฟล์
  recentFiles = recentFiles.slice(0, 10)
  // สร้างเมนูใหม่
  rebuildMenu()
}

function clearRecentFiles() {
  recentFiles = []
  rebuildMenu()
}

// สร้างส่วน Recent Files ของเมนู
function buildRecentFilesSubmenu() {
  if (recentFiles.length === 0) {
    return [{ label: 'ยังไม่มีไฟล์ล่าสุด', enabled: false }]
  }
  
  const fileItems = recentFiles.map(filePath => ({
    label: require('path').basename(filePath),  // แสดงแค่ชื่อไฟล์
    click: () => openRecentFile(filePath),
    toolTip: filePath  // tooltip แสดง path เต็ม
  }))
  
  return [
    ...fileItems,
    { type: 'separator' },
    {
      label: 'ล้างรายการล่าสุด',
      click: clearRecentFiles
    }
  ]
}

// สร้างเมนูใหม่ทั้งหมด (เรียกเมื่อ state เปลี่ยน)
function rebuildMenu() {
  const template = [
    {
      label: 'ไฟล์',
      submenu: [
        {
          label: 'ใหม่',
          accelerator: 'CmdOrCtrl+N',
          click: createNewDocument
        },
        {
          label: 'เปิด',
          accelerator: 'CmdOrCtrl+O',
          click: openFile
        },
        {
          label: 'เปิดล่าสุด',
          submenu: buildRecentFilesSubmenu()  // dynamic submenu
        },
        { type: 'separator' },
        {
          label: 'บันทึก',
          accelerator: 'CmdOrCtrl+S',
          click: saveFile,
          enabled: hasOpenDocument()  // เปิด/ปิดตาม state
        },
        { type: 'separator' },
        { label: 'ออก', role: 'quit' }
      ]
    },
    createEditMenu(),
    createThemeMenu(),
    createLanguageMenu(),
    createSettingsMenu(),
    buildWindowMenu(),
    buildHelpMenu()
  ]
  
  if (process.platform === 'darwin') {
    template.unshift(createMacAppMenu())
  }
  
  const menu = Menu.buildFromTemplate(template)
  Menu.setApplicationMenu(menu)
}

// ตัวแปร track state ของ document
let documentOpen = false

function hasOpenDocument() {
  return documentOpen
}

function openRecentFile(filePath) {
  console.log('เปิดไฟล์ล่าสุด:', filePath)
  documentOpen = true
  rebuildMenu()  // สร้างเมนูใหม่เพื่ออัพเดท enabled state
}

// IPC: รับคำสั่งให้ rebuild menu
ipcMain.on('rebuild-menu', () => {
  rebuildMenu()
})
```

---

## 7.5 Context Menu - เมนูคลิกขวา

### Preload Script
```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('menuAPI', {
  // แจ้ง main ให้แสดง context menu
  showContextMenu: (data) => ipcRenderer.send('show-context-menu', data),
  
  // รับคำสั่งจาก context menu
  onContextMenuAction: (callback) => {
    const handler = (event, action) => callback(action)
    ipcRenderer.on('context-menu-action', handler)
    return () => ipcRenderer.removeListener('context-menu-action', handler)
  },
  
  // รับ menu events
  onMenuAction: (callback) => {
    const channels = [
      'menu-new', 'menu-open', 'menu-save', 'menu-save-as',
      'menu-about', 'setting-changed', 'theme-changed'
    ]
    
    const handlers = channels.map(channel => {
      const handler = (event, data) => callback(channel, data)
      ipcRenderer.on(channel, handler)
      return { channel, handler }
    })
    
    return () => {
      handlers.forEach(({ channel, handler }) => {
        ipcRenderer.removeListener(channel, handler)
      })
    }
  }
})
```

### main.js - Context Menu Handler
```javascript
// main.js - ส่วน context menu

const { ipcMain, Menu, MenuItem } = require('electron')

// Context menu พื้นฐาน
ipcMain.on('show-context-menu', (event, data) => {
  const { type, selectedText, x, y } = data
  
  let template = []
  
  // เพิ่ม items ตาม type ของ element
  if (type === 'text-editor') {
    template = buildTextEditorContextMenu(event, selectedText)
  } else if (type === 'image') {
    template = buildImageContextMenu(event, data)
  } else if (type === 'link') {
    template = buildLinkContextMenu(event, data)
  } else {
    template = buildDefaultContextMenu(event)
  }
  
  const menu = Menu.buildFromTemplate(template)
  
  // แสดงเมนูที่ตำแหน่งที่คลิก
  const win = require('electron').BrowserWindow.fromWebContents(event.sender)
  menu.popup({
    window: win,
    x: x,
    y: y,
    callback: () => {
      console.log('Context menu ถูกปิดแล้ว')
    }
  })
})

// Context menu สำหรับ text editor
function buildTextEditorContextMenu(event, selectedText) {
  const hasSelection = selectedText && selectedText.length > 0
  
  return [
    {
      label: 'ตัด',
      role: 'cut',
      enabled: hasSelection
    },
    {
      label: 'คัดลอก',
      role: 'copy',
      enabled: hasSelection
    },
    {
      label: 'วาง',
      role: 'paste'
    },
    { type: 'separator' },
    {
      label: 'เลือกทั้งหมด',
      role: 'selectAll'
    },
    { type: 'separator' },
    {
      label: 'ค้นหาใน Google',
      enabled: hasSelection,
      click: () => {
        if (selectedText) {
          const query = encodeURIComponent(selectedText)
          require('electron').shell.openExternal(`https://www.google.com/search?q=${query}`)
        }
      }
    },
    {
      label: 'แปลภาษา',
      enabled: hasSelection,
      submenu: [
        {
          label: 'แปลเป็นภาษาไทย',
          click: () => {
            event.sender.send('context-menu-action', {
              action: 'translate',
              to: 'th',
              text: selectedText
            })
          }
        },
        {
          label: 'แปลเป็นภาษาอังกฤษ',
          click: () => {
            event.sender.send('context-menu-action', {
              action: 'translate',
              to: 'en',
              text: selectedText
            })
          }
        }
      ]
    }
  ]
}

// Context menu สำหรับรูปภาพ
function buildImageContextMenu(event, data) {
  return [
    {
      label: 'บันทึกรูปภาพ',
      click: async () => {
        const { dialog } = require('electron')
        const win = require('electron').BrowserWindow.fromWebContents(event.sender)
        
        const result = await dialog.showSaveDialog(win, {
          title: 'บันทึกรูปภาพ',
          defaultPath: 'image.png',
          filters: [
            { name: 'Images', extensions: ['png', 'jpg', 'gif', 'webp'] }
          ]
        })
        
        if (!result.canceled) {
          event.sender.send('context-menu-action', {
            action: 'save-image',
            src: data.src,
            savePath: result.filePath
          })
        }
      }
    },
    {
      label: 'คัดลอก URL รูปภาพ',
      click: () => {
        require('electron').clipboard.writeText(data.src)
      }
    },
    {
      label: 'เปิดรูปภาพในเบราว์เซอร์',
      click: () => {
        require('electron').shell.openExternal(data.src)
      }
    }
  ]
}

// Context menu สำหรับ link
function buildLinkContextMenu(event, data) {
  return [
    {
      label: 'เปิดลิ้งค์',
      click: () => {
        require('electron').shell.openExternal(data.href)
      }
    },
    {
      label: 'เปิดในหน้าต่างใหม่',
      click: () => {
        const newWin = new require('electron').BrowserWindow({ width: 800, height: 600 })
        newWin.loadURL(data.href)
      }
    },
    {
      label: 'คัดลอก URL',
      click: () => {
        require('electron').clipboard.writeText(data.href)
      }
    }
  ]
}

// Default context menu
function buildDefaultContextMenu(event) {
  return [
    {
      label: 'โหลดหน้าใหม่',
      click: () => event.sender.reload()
    },
    { type: 'separator' },
    {
      label: 'ตรวจสอบ Element',
      click: () => event.sender.inspectElement(0, 0)
    }
  ]
}
```

### renderer.js - ผูก Context Menu กับ Elements
```javascript
// renderer.js - แสดง context menu เมื่อคลิกขวา

// =============================================
// ผูก context menu กับ editor
// =============================================
const editor = document.getElementById('textEditor')

editor.addEventListener('contextmenu', (e) => {
  e.preventDefault()  // ป้องกัน browser context menu
  
  const selectedText = window.getSelection().toString()
  
  // ส่งข้อมูลไปให้ main แสดง context menu
  window.menuAPI.showContextMenu({
    type: 'text-editor',
    selectedText,
    x: e.x,
    y: e.y
  })
})

// =============================================
// ผูก context menu กับรูปภาพ
// =============================================
document.querySelectorAll('img').forEach(img => {
  img.addEventListener('contextmenu', (e) => {
    e.preventDefault()
    
    window.menuAPI.showContextMenu({
      type: 'image',
      src: e.target.src,
      x: e.x,
      y: e.y
    })
  })
})

// =============================================
// ผูก context menu กับ links
// =============================================
document.querySelectorAll('a').forEach(link => {
  link.addEventListener('contextmenu', (e) => {
    e.preventDefault()
    
    window.menuAPI.showContextMenu({
      type: 'link',
      href: e.target.href,
      text: e.target.textContent,
      x: e.x,
      y: e.y
    })
  })
})

// =============================================
// รับคำสั่งจาก context menu
// =============================================
const cleanup = window.menuAPI.onContextMenuAction(async (action) => {
  switch (action.action) {
    case 'translate':
      await translateText(action.text, action.to)
      break
      
    case 'save-image':
      await saveImage(action.src, action.savePath)
      break
      
    default:
      console.log('Unknown action:', action.action)
  }
})

async function translateText(text, targetLang) {
  // ใช้ API แปลภาษา (ตัวอย่าง)
  const resultDiv = document.getElementById('translateResult')
  resultDiv.textContent = `กำลังแปล: "${text}" เป็น ${targetLang}...`
  
  // จำลอง API call
  await new Promise(resolve => setTimeout(resolve, 500))
  resultDiv.textContent = `คำแปล: [${targetLang}] ${text}`
}

async function saveImage(src, savePath) {
  console.log(`บันทึกรูปจาก ${src} ไปที่ ${savePath}`)
  // ทำการบันทึกจริงๆ ใน main process
}

window.addEventListener('beforeunload', cleanup)
```

---

## 7.6 macOS App Menu

บน macOS แอปต้องมีเมนูพิเศษที่ชื่อ app name เป็นเมนูแรก:

```javascript
// main.js - macOS App Menu

function createMacAppMenu() {
  return {
    label: app.getName(),  // ชื่อแอป (ดึงจาก package.json)
    submenu: [
      {
        label: `เกี่ยวกับ ${app.getName()}`,
        role: 'about'
      },
      { type: 'separator' },
      {
        label: 'การตั้งค่า...',
        accelerator: 'Cmd+,',
        click: () => openPreferences()
      },
      { type: 'separator' },
      {
        label: 'บริการ',
        role: 'services',
        submenu: []  // macOS เติมให้อัตโนมัติ
      },
      { type: 'separator' },
      {
        label: `ซ่อน ${app.getName()}`,
        role: 'hide'
      },
      {
        label: 'ซ่อนหน้าต่างอื่น',
        role: 'hideOthers'
      },
      {
        label: 'แสดงทั้งหมด',
        role: 'unhide'
      },
      { type: 'separator' },
      {
        label: `ออกจาก ${app.getName()}`,
        role: 'quit'
      }
    ]
  }
}

// macOS Window Menu
function buildWindowMenu() {
  if (process.platform === 'darwin') {
    return {
      label: 'หน้าต่าง',
      submenu: [
        { role: 'minimize' },
        { role: 'zoom' },
        { type: 'separator' },
        { role: 'front' },
        { type: 'separator' },
        { role: 'window' }
      ]
    }
  }
  
  // Windows/Linux Window menu
  return {
    label: 'หน้าต่าง',
    submenu: [
      { role: 'minimize' },
      { role: 'close' }
    ]
  }
}
```

---

## 7.7 Platform-Specific Menus

```javascript
// main.js - เมนูที่ต่างกันตาม platform

function buildEditMenu() {
  const template = {
    label: 'แก้ไข',
    submenu: [
      { role: 'undo' },
      { role: 'redo' },
      { type: 'separator' },
      { role: 'cut' },
      { role: 'copy' },
      { role: 'paste' }
    ]
  }
  
  // เพิ่ม items เฉพาะ macOS
  if (process.platform === 'darwin') {
    template.submenu.push(
      { role: 'pasteAndMatchStyle' },
      { role: 'delete' },
      { role: 'selectAll' },
      { type: 'separator' },
      {
        label: 'Speech',
        submenu: [
          { role: 'startSpeaking' },
          { role: 'stopSpeaking' }
        ]
      }
    )
  } else {
    // Windows/Linux
    template.submenu.push(
      { role: 'delete' },
      { type: 'separator' },
      { role: 'selectAll' }
    )
  }
  
  return template
}

// เมนู Help ที่ต่างกันตาม platform
function buildHelpMenu() {
  const template = {
    role: 'help',
    submenu: [
      {
        label: 'เรียนรู้เพิ่มเติม',
        click: () => {
          require('electron').shell.openExternal('https://electronjs.org')
        }
      }
    ]
  }
  
  // เพิ่ม About ใน Windows/Linux (macOS มีใน App menu แล้ว)
  if (process.platform !== 'darwin') {
    template.submenu.push(
      { type: 'separator' },
      {
        label: `เกี่ยวกับ ${app.getName()}`,
        click: () => showAboutWindow()
      }
    )
  }
  
  return template
}

// แสดงหน้า About แบบ custom
function showAboutWindow() {
  const aboutWindow = new BrowserWindow({
    width: 400,
    height: 300,
    resizable: false,
    minimizable: false,
    maximizable: false,
    parent: mainWindow,
    modal: true,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  
  aboutWindow.loadFile('about.html')
  aboutWindow.setMenuBarVisibility(false)
}
```

---

## 7.8 เมนูขั้นสูง: การอัพเดทเมนูแบบ Dynamic

```javascript
// main.js - การจัดการเมนูขั้นสูง

const { Menu, MenuItem } = require('electron')

class MenuManager {
  constructor() {
    this.menuState = {
      hasUnsavedChanges: false,
      canUndo: false,
      canRedo: false,
      isLoggedIn: false,
      selectedItems: 0,
      recentFiles: []
    }
    
    this.menuRef = null  // reference ไปยังเมนูปัจจุบัน
  }
  
  // อัพเดท state และ rebuild menu
  updateState(newState) {
    this.menuState = { ...this.menuState, ...newState }
    this.rebuild()
  }
  
  rebuild() {
    const menu = Menu.buildFromTemplate(this.buildTemplate())
    Menu.setApplicationMenu(menu)
    this.menuRef = menu
  }
  
  buildTemplate() {
    const { hasUnsavedChanges, canUndo, canRedo, isLoggedIn, selectedItems } = this.menuState
    
    return [
      {
        label: 'ไฟล์',
        submenu: [
          {
            label: 'ใหม่',
            accelerator: 'CmdOrCtrl+N',
            click: () => this.handleMenuAction('new')
          },
          {
            label: 'เปิด',
            accelerator: 'CmdOrCtrl+O',
            click: () => this.handleMenuAction('open')
          },
          {
            label: 'เปิดล่าสุด',
            submenu: this.buildRecentFilesMenu()
          },
          { type: 'separator' },
          {
            label: 'บันทึก',
            accelerator: 'CmdOrCtrl+S',
            enabled: hasUnsavedChanges,
            click: () => this.handleMenuAction('save')
          },
          {
            label: 'บันทึกเป็น...',
            accelerator: 'CmdOrCtrl+Shift+S',
            click: () => this.handleMenuAction('save-as')
          },
          { type: 'separator' },
          { role: 'quit' }
        ]
      },
      {
        label: 'แก้ไข',
        submenu: [
          {
            label: 'เลิกทำ',
            accelerator: 'CmdOrCtrl+Z',
            enabled: canUndo,
            click: () => this.handleMenuAction('undo')
          },
          {
            label: 'ทำซ้ำ',
            accelerator: 'CmdOrCtrl+Shift+Z',
            enabled: canRedo,
            click: () => this.handleMenuAction('redo')
          },
          { type: 'separator' },
          {
            label: 'ลบที่เลือก',
            enabled: selectedItems > 0,
            click: () => this.handleMenuAction('delete-selected')
          }
        ]
      },
      {
        label: 'บัญชี',
        submenu: isLoggedIn ? [
          {
            label: 'โปรไฟล์',
            click: () => this.handleMenuAction('profile')
          },
          {
            label: 'การตั้งค่า',
            click: () => this.handleMenuAction('settings')
          },
          { type: 'separator' },
          {
            label: 'ออกจากระบบ',
            click: () => this.handleMenuAction('logout')
          }
        ] : [
          {
            label: 'เข้าสู่ระบบ',
            click: () => this.handleMenuAction('login')
          },
          {
            label: 'สมัครสมาชิก',
            click: () => this.handleMenuAction('register')
          }
        ]
      }
    ]
  }
  
  buildRecentFilesMenu() {
    if (this.menuState.recentFiles.length === 0) {
      return [{ label: 'ยังไม่มีไฟล์ล่าสุด', enabled: false }]
    }
    
    return [
      ...this.menuState.recentFiles.map(file => ({
        label: require('path').basename(file),
        toolTip: file,
        click: () => this.handleMenuAction('open-recent', file)
      })),
      { type: 'separator' },
      {
        label: 'ล้างรายการ',
        click: () => {
          this.updateState({ recentFiles: [] })
        }
      }
    ]
  }
  
  handleMenuAction(action, data = null) {
    console.log('Menu action:', action, data)
    
    if (mainWindow && !mainWindow.isDestroyed()) {
      mainWindow.webContents.send('menu-action', { action, data })
    }
  }
}

// สร้าง instance
const menuManager = new MenuManager()

// IPC: รับคำสั่งอัพเดท state จาก renderer
ipcMain.on('update-menu-state', (event, newState) => {
  menuManager.updateState(newState)
})

app.whenReady().then(() => {
  createWindow()
  menuManager.rebuild()
})
```

---

## 7.9 Tray Context Menu (เตรียมสำหรับบทถัดไป)

```javascript
// main.js - Context menu สำหรับ tray icon
// (จะอธิบายละเอียดในตอนที่ 9)

function createTrayMenu() {
  return Menu.buildFromTemplate([
    {
      label: 'เปิดแอป',
      click: () => {
        if (mainWindow) {
          mainWindow.show()
          mainWindow.focus()
        }
      }
    },
    { type: 'separator' },
    {
      label: 'การแจ้งเตือน',
      type: 'checkbox',
      checked: true,
      click: (menuItem) => {
        console.log('Notifications:', menuItem.checked)
      }
    },
    { type: 'separator' },
    {
      label: 'ออกจากโปรแกรม',
      click: () => app.quit()
    }
  ])
}
```

---

## 7.10 index.html และ renderer.js สมบูรณ์

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Menu Demo</title>
  <style>
    body {
      font-family: 'Sarabun', sans-serif;
      padding: 20px;
      background: #f0f0f0;
    }
    
    .demo-area {
      background: white;
      padding: 20px;
      border-radius: 8px;
      margin-bottom: 20px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    
    #textEditor {
      width: 100%;
      height: 200px;
      padding: 12px;
      border: 1px solid #ddd;
      border-radius: 4px;
      font-size: 14px;
      font-family: 'Sarabun', sans-serif;
      resize: vertical;
      box-sizing: border-box;
    }
    
    .status-bar {
      background: #333;
      color: white;
      padding: 4px 10px;
      font-size: 12px;
      position: fixed;
      bottom: 0;
      left: 0;
      right: 0;
    }
    
    .image-demo {
      display: flex;
      gap: 10px;
      flex-wrap: wrap;
    }
    
    .image-demo img {
      width: 150px;
      height: 100px;
      object-fit: cover;
      border-radius: 4px;
      cursor: pointer;
    }
    
    #translateResult {
      margin-top: 10px;
      padding: 8px;
      background: #e8f5e9;
      border-radius: 4px;
      font-size: 14px;
    }
    
    #menuActions {
      padding: 8px;
      background: #fff9c4;
      border-radius: 4px;
      min-height: 30px;
      font-size: 14px;
    }
  </style>
</head>
<body>
  <h1>Menu Demo</h1>
  
  <div class="demo-area">
    <h2>Menu Actions</h2>
    <div id="menuActions">รอรับคำสั่งจากเมนู...</div>
  </div>
  
  <div class="demo-area">
    <h2>Text Editor (คลิกขวาเพื่อดู context menu)</h2>
    <textarea id="textEditor">
ลองเลือกข้อความในนี้แล้วคลิกขวาเพื่อดู context menu
ที่มีตัวเลือก ตัด คัดลอก วาง และค้นหาใน Google
    </textarea>
    <div id="translateResult"></div>
  </div>
  
  <div class="demo-area">
    <h2>รูปภาพ (คลิกขวาเพื่อดู context menu)</h2>
    <div class="image-demo">
      <img src="https://picsum.photos/300/200?random=1" alt="รูปที่ 1">
      <img src="https://picsum.photos/300/200?random=2" alt="รูปที่ 2">
      <img src="https://picsum.photos/300/200?random=3" alt="รูปที่ 3">
    </div>
  </div>
  
  <div class="status-bar" id="statusBar">
    พร้อมใช้งาน
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

```javascript
// renderer.js - สมบูรณ์

// =============================================
// รับคำสั่งจากเมนู
// =============================================
const actionsDiv = document.getElementById('menuActions')
const statusBar = document.getElementById('statusBar')

const cleanupMenuListener = window.menuAPI.onMenuAction((channel, data) => {
  const timestamp = new Date().toLocaleTimeString('th-TH')
  actionsDiv.textContent = `[${timestamp}] รับคำสั่ง: ${channel}`
  if (data) {
    actionsDiv.textContent += ` - ข้อมูล: ${JSON.stringify(data)}`
  }
})

// =============================================
// Context Menu สำหรับ Text Editor
// =============================================
const editor = document.getElementById('textEditor')

editor.addEventListener('contextmenu', (e) => {
  e.preventDefault()
  const selectedText = window.getSelection().toString()
  
  window.menuAPI.showContextMenu({
    type: 'text-editor',
    selectedText,
    x: e.x,
    y: e.y
  })
})

// =============================================
// Context Menu สำหรับรูปภาพ
// =============================================
document.querySelectorAll('.image-demo img').forEach(img => {
  img.addEventListener('contextmenu', (e) => {
    e.preventDefault()
    
    window.menuAPI.showContextMenu({
      type: 'image',
      src: e.target.src,
      x: e.x,
      y: e.y
    })
  })
})

// =============================================
// รับผลลัพธ์จาก context menu
// =============================================
const cleanupContextMenuListener = window.menuAPI.onContextMenuAction(async (action) => {
  const translateResult = document.getElementById('translateResult')
  
  switch (action.action) {
    case 'translate':
      translateResult.textContent = `กำลังแปล: "${action.text}" เป็น ${action.to}...`
      
      // จำลองการแปล
      await new Promise(resolve => setTimeout(resolve, 800))
      translateResult.textContent = `คำแปล (${action.to}): ${action.text} [จำลอง]`
      break
      
    case 'save-image':
      statusBar.textContent = `กำลังบันทึกรูปภาพ...`
      await new Promise(resolve => setTimeout(resolve, 500))
      statusBar.textContent = `บันทึกรูปภาพเสร็จแล้ว: ${action.savePath}`
      break
  }
})

// =============================================
// แจ้ง main ว่า state เปลี่ยน (ตัวอย่าง)
// =============================================
editor.addEventListener('input', () => {
  // อัพเดท menu state ว่ามีการแก้ไขแล้ว
  // window.menuAPI.updateMenuState({ hasUnsavedChanges: true })
  statusBar.textContent = 'มีการแก้ไขที่ยังไม่ได้บันทึก'
})

// Cleanup
window.addEventListener('beforeunload', () => {
  cleanupMenuListener()
  cleanupContextMenuListener()
})
```

---

## สรุปบทเรียน

### สิ่งที่ได้เรียนรู้

1. **Menu.buildFromTemplate** - สร้างเมนูจาก template array
2. **Roles** - shortcuts ที่ Electron มีให้พร้อมใช้
3. **Checkbox/Radio** - items ที่มี state
4. **Context Menu** - menu เมื่อคลิกขวา
5. **Dynamic Menus** - เมนูที่เปลี่ยนแปลงตาม state
6. **Platform-specific** - เมนูที่ต่างกันตาม OS

### Best Practices

- ใช้ `role` เมื่อทำได้ เพื่อให้ keyboard shortcuts ทำงานถูกต้อง
- ใช้ `CmdOrCtrl` แทน `Ctrl` เพื่อรองรับ macOS
- แยกสร้างเมนู macOS ด้วย `process.platform === 'darwin'`
- Rebuild menu เมื่อ state เปลี่ยน เพื่ออัพเดท enabled/checked state
- แสดง tooltip บน menu items ที่ต้องการ path หรือ URL เต็ม
