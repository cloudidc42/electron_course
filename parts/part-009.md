# ตอนที่ 9: System Tray และ Notifications

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจากศึกษาตอนนี้ คุณจะสามารถ:
- สร้าง Tray icon และจัดการ tray menu
- ตั้งค่า tray tooltip และ balloon (Windows)
- ใช้ Notification API ของ Electron
- จัดการ notification events (close, click, action)
- ส่ง notification actions บน macOS
- ตั้งค่า badge count บน macOS
- สร้าง background app ที่ทำงานใน system tray

---

## บทนำ: Tray และ Notification

```
System Tray (Windows/Linux taskbar, macOS menu bar)
├── Tray Icon          → รูปไอคอนเล็กๆ ใน system tray
├── Tray Menu          → เมนูเมื่อคลิกที่ tray icon
├── Tray Tooltip       → ข้อความเมื่อ hover
└── Tray Balloon       → popup notification (Windows เท่านั้น)

Notifications
├── Notification API   → browser-like notification
├── Actions            → ปุ่มใน notification (macOS)
└── Badge Count        → ตัวเลขบน dock icon (macOS)
```

---

## 9.1 สร้าง Tray Icon พื้นฐาน

### โครงสร้างโปรเจ็กต์
```
tray-app/
├── main.js
├── preload.js
├── renderer.js
├── index.html
├── assets/
│   ├── tray-icon.png        # 16x16 หรือ 32x32 pixels
│   ├── tray-icon@2x.png     # สำหรับ HiDPI (retina)
│   └── notification.png     # รูปสำหรับ notification
└── package.json
```

### main.js - Tray พื้นฐาน
```javascript
const { app, BrowserWindow, Tray, Menu, nativeImage, ipcMain } = require('electron')
const path = require('path')

let mainWindow
let tray = null  // ต้องเก็บ reference เพื่อป้องกัน garbage collection!

function createTrayIcon() {
  // ==========================================
  // สร้าง icon จาก file
  // ==========================================
  
  // วิธีที่ 1: โหลดจากไฟล์
  const iconPath = path.join(__dirname, 'assets', 'tray-icon.png')
  const trayIcon = nativeImage.createFromPath(iconPath)
  
  // ปรับขนาด icon ให้เหมาะสม (optional)
  // const resizedIcon = trayIcon.resize({ width: 16, height: 16 })
  
  // วิธีที่ 2: สร้าง icon จาก base64 (สะดวกสำหรับ icon เล็กๆ)
  // const trayIcon = nativeImage.createFromDataURL('data:image/png;base64,...')
  
  // วิธีที่ 3: สร้าง nativeImage ว่าง (สำหรับ macOS template image)
  // trayIcon.setTemplateImage(true)  // macOS: ปรับสีอัตโนมัติตาม theme
  
  // ==========================================
  // สร้าง Tray instance
  // ==========================================
  tray = new Tray(trayIcon)
  
  // ตั้งค่า tooltip (ข้อความเมื่อ hover)
  tray.setToolTip('แอปพลิเคชันของฉัน\nเวอร์ชัน 1.0.0')
  
  // ตั้งค่า context menu
  updateTrayMenu()
  
  // ==========================================
  // Event Handlers
  // ==========================================
  
  // คลิกซ้าย (Windows/Linux): แสดง/ซ่อน window
  tray.on('click', () => {
    toggleWindow()
  })
  
  // คลิกขวา (macOS คลิกเดียว): แสดง context menu
  tray.on('right-click', () => {
    tray.popUpContextMenu()
  })
  
  // double-click (Windows): เปิด window
  tray.on('double-click', () => {
    showWindow()
  })
  
  // mouse enter (hover)
  tray.on('mouse-enter', () => {
    console.log('Mouse hover บน tray icon')
  })
  
  // mouse move (บน macOS เท่านั้น)
  tray.on('mouse-move', (event, position) => {
    // position = { x, y } ตำแหน่งของ mouse
  })
  
  console.log('Tray icon สร้างเสร็จแล้ว')
}

// ==========================================
// Toggle Window (แสดง/ซ่อน)
// ==========================================
function toggleWindow() {
  if (mainWindow.isVisible()) {
    if (mainWindow.isFocused()) {
      mainWindow.hide()
    } else {
      mainWindow.show()
      mainWindow.focus()
    }
  } else {
    showWindow()
  }
}

function showWindow() {
  if (mainWindow.isMinimized()) mainWindow.restore()
  mainWindow.show()
  mainWindow.focus()
}

// ==========================================
// อัพเดท Tray Menu
// ==========================================
function updateTrayMenu(options = {}) {
  const { isOnline = true, notificationsEnabled = true, userName = 'ผู้ใช้' } = options
  
  const contextMenu = Menu.buildFromTemplate([
    // ส่วนบน: ข้อมูลสถานะ
    {
      label: `👤 ${userName}`,
      enabled: false  // แสดงข้อมูล ไม่ให้คลิก
    },
    {
      label: isOnline ? '🟢 ออนไลน์' : '🔴 ออฟไลน์',
      enabled: false
    },
    { type: 'separator' },
    
    // เปิดแอป
    {
      label: 'เปิดแอปพลิเคชัน',
      click: () => showWindow(),
      accelerator: process.platform === 'darwin' ? 'Cmd+Shift+A' : undefined
    },
    { type: 'separator' },
    
    // การแจ้งเตือน
    {
      label: 'การแจ้งเตือน',
      type: 'checkbox',
      checked: notificationsEnabled,
      click: (item) => {
        console.log('Notifications:', item.checked)
        // บันทึก preference
        mainWindow.webContents.send('tray-notification-toggle', item.checked)
      }
    },
    { type: 'separator' },
    
    // Actions
    {
      label: 'ตรวจสอบอัพเดท',
      click: () => {
        mainWindow.webContents.send('tray-action', 'check-updates')
        showWindow()
      }
    },
    {
      label: 'การตั้งค่า',
      click: () => {
        mainWindow.webContents.send('tray-action', 'open-settings')
        showWindow()
      }
    },
    { type: 'separator' },
    
    // ออกจากแอป
    {
      label: 'ออกจากโปรแกรม',
      click: () => {
        app.quit()
      }
    }
  ])
  
  tray.setContextMenu(contextMenu)
}

// ==========================================
// สร้าง Window
// ==========================================
function createWindow() {
  mainWindow = new BrowserWindow({
    width: 900,
    height: 650,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    },
    // ซ่อน taskbar icon และ dock icon (สำหรับ background app)
    // skipTaskbar: true,  // Windows: ไม่แสดงใน taskbar
    show: true
  })
  
  mainWindow.loadFile('index.html')
  
  // ซ่อนแทนปิดเมื่อกด X
  mainWindow.on('close', (event) => {
    // ถ้าแอปไม่ได้กำลัง quit จริงๆ ให้ซ่อนแทน
    if (!app.isQuitting) {
      event.preventDefault()
      mainWindow.hide()
      
      // แสดง notification ว่าแอปยังทำงานอยู่
      if (Notification.isSupported()) {
        new Notification({
          title: 'แอปยังทำงานอยู่',
          body: 'แอปถูกย่อลงใน system tray คลิกที่ไอคอนเพื่อเปิดอีกครั้ง',
          icon: path.join(__dirname, 'assets', 'notification.png')
        }).show()
      }
    }
  })
}

app.whenReady().then(() => {
  createWindow()
  createTrayIcon()
})

// เมื่อปิดแอปจริงๆ
app.on('before-quit', () => {
  app.isQuitting = true
})

app.on('window-all-closed', () => {
  // อย่า quit เมื่อ window ทั้งหมดปิด (เพราะยังมี tray)
  // if (process.platform !== 'darwin') app.quit()
  // ไม่ทำอะไร เพราะเราต้องการให้แอปอยู่ใน tray
})
```

---

## 9.2 Tray Balloon (Windows)

Tray balloon เป็น popup notification ที่มาจาก tray icon บน Windows:

```javascript
// main.js - Tray Balloon (Windows only)

function showTrayBalloon(options) {
  // ตรวจสอบว่าเป็น Windows
  if (process.platform !== 'win32') {
    console.log('Tray balloon ใช้ได้บน Windows เท่านั้น')
    return
  }
  
  if (!tray) {
    console.error('Tray ยังไม่ได้สร้าง')
    return
  }
  
  tray.displayBalloon({
    title: options.title || 'แจ้งเตือน',
    content: options.content || '',
    
    // ไอคอนของ balloon
    // 'none', 'info', 'warning', 'error', หรือ NativeImage
    icon: options.icon || 'info',
    
    // ไม่เล่นเสียง (optional)
    noSound: options.noSound || false,
    
    // แสดง icon ขนาดใหญ่ (Windows Vista+)
    largeIcon: true
  })
}

// Event เมื่อ balloon ถูกคลิก
if (process.platform === 'win32' && tray) {
  tray.on('balloon-click', () => {
    console.log('ผู้ใช้คลิก balloon')
    showWindow()
  })
  
  tray.on('balloon-show', () => {
    console.log('Balloon แสดงแล้ว')
  })
  
  tray.on('balloon-closed', () => {
    console.log('Balloon ถูกปิด')
  })
}

// IPC: แสดง balloon
ipcMain.on('show-balloon', (event, options) => {
  showTrayBalloon(options)
})
```

---

## 9.3 Notification API

### การใช้งาน Notification พื้นฐาน

```javascript
// main.js - Notifications

const { Notification } = require('electron')

// ==========================================
// ตรวจสอบ support
// ==========================================
console.log('Notification supported:', Notification.isSupported())

// ==========================================
// สร้าง Notification พื้นฐาน
// ==========================================
function sendNotification(options) {
  if (!Notification.isSupported()) {
    console.warn('Notification ไม่ได้รับการสนับสนุนในระบบนี้')
    return null
  }
  
  const notification = new Notification({
    title: options.title,
    body: options.body || '',
    
    // ไอคอน
    icon: options.icon || path.join(__dirname, 'assets', 'notification.png'),
    
    // เสียง (macOS)
    silent: options.silent || false,
    
    // urgency (Linux): 'normal', 'critical', 'low'
    urgency: options.urgency || 'normal',
    
    // timeoutType: 'default', 'never'
    timeoutType: options.timeoutType || 'default',
    
    // subtitle (macOS)
    subtitle: options.subtitle || '',
    
    // hasReply (macOS): เพิ่มช่องพิมพ์ตอบกลับ
    hasReply: options.hasReply || false,
    
    // replyPlaceholder (macOS)
    replyPlaceholder: options.replyPlaceholder || 'พิมพ์คำตอบ...',
    
    // actions (macOS): ปุ่มใน notification
    actions: options.actions || [],
    
    // toastXml (Windows): XML สำหรับ Windows Toast Notification
    // toastXml: options.toastXml
  })
  
  return notification
}

// ==========================================
// IPC Handlers สำหรับ Notifications
// ==========================================

// Notification พื้นฐาน
ipcMain.handle('send-notification', async (event, options) => {
  return new Promise((resolve) => {
    const notification = sendNotification(options)
    
    if (!notification) {
      resolve({ shown: false, reason: 'not-supported' })
      return
    }
    
    // Event: แสดงแล้ว
    notification.on('show', () => {
      console.log('Notification แสดงแล้ว')
    })
    
    // Event: คลิก
    notification.on('click', () => {
      console.log('ผู้ใช้คลิก notification')
      showWindow()
      resolve({ action: 'click' })
    })
    
    // Event: ปิด
    notification.on('close', () => {
      console.log('Notification ถูกปิด')
      resolve({ action: 'close' })
    })
    
    // Event: ตอบกลับ (macOS hasReply)
    notification.on('reply', (event, reply) => {
      console.log('ผู้ใช้ตอบ:', reply)
      resolve({ action: 'reply', reply })
    })
    
    // Event: action button (macOS)
    notification.on('action', (event, index) => {
      console.log('Action button กด index:', index)
      resolve({ action: 'action', index })
    })
    
    // Event: failed
    notification.on('failed', (event, error) => {
      console.error('Notification ล้มเหลว:', error)
      resolve({ shown: false, error })
    })
    
    notification.show()
  })
})

// ==========================================
// Notification พร้อม Actions (macOS)
// ==========================================
ipcMain.handle('send-notification-with-actions', async (event, { title, body, actionLabels }) => {
  return new Promise((resolve) => {
    const actions = actionLabels.map(label => ({
      type: 'button',
      text: label
    }))
    
    const notification = new Notification({
      title,
      body,
      icon: path.join(__dirname, 'assets', 'notification.png'),
      actions,
      hasReply: false
    })
    
    notification.on('action', (e, index) => {
      resolve({
        userAction: 'button',
        buttonIndex: index,
        buttonLabel: actionLabels[index]
      })
      notification.close()
    })
    
    notification.on('click', () => {
      resolve({ userAction: 'click' })
      showWindow()
    })
    
    notification.on('close', () => {
      resolve({ userAction: 'dismissed' })
    })
    
    notification.show()
  })
})

// ==========================================
// Reply Notification (macOS)
// ==========================================
ipcMain.handle('send-reply-notification', async (event, { title, body }) => {
  return new Promise((resolve) => {
    const notification = new Notification({
      title,
      body,
      hasReply: true,
      replyPlaceholder: 'พิมพ์คำตอบที่นี่...'
    })
    
    notification.on('reply', (e, reply) => {
      resolve({ userAction: 'replied', reply })
    })
    
    notification.on('click', () => {
      resolve({ userAction: 'clicked' })
      showWindow()
    })
    
    notification.on('close', () => {
      resolve({ userAction: 'dismissed' })
    })
    
    notification.show()
  })
})
```

---

## 9.4 Badge Count (macOS)

```javascript
// main.js - Badge Count (macOS Dock icon)

// ตั้งค่า badge บน macOS dock icon
function setBadgeCount(count) {
  if (process.platform !== 'darwin') {
    // Windows: ใช้ overlay icon แทน
    setWindowsOverlayIcon(count)
    return
  }
  
  if (count > 0) {
    app.setBadgeCount(count)
  } else {
    app.setBadgeCount(0)  // หรือ app.setBadgeCount(-1) เพื่อลบ badge
  }
}

// Windows: ใช้ overlay icon บน taskbar
function setWindowsOverlayIcon(count) {
  if (process.platform !== 'win32' || !mainWindow) return
  
  if (count > 0) {
    // สร้าง overlay image แสดงตัวเลข
    const icon = createBadgeImage(count)
    mainWindow.setOverlayIcon(icon, `${count} รายการใหม่`)
  } else {
    // ลบ overlay
    mainWindow.setOverlayIcon(null, '')
  }
}

// สร้าง badge image สำหรับ Windows overlay
function createBadgeImage(count) {
  const { nativeImage } = require('electron')
  
  // สร้าง canvas เพื่อวาด badge
  // ต้องใช้ offscreen renderer หรือ library เช่น node-canvas
  // ตัวอย่างง่ายๆ: ใช้ pre-made icons
  const badgePath = path.join(__dirname, 'assets', `badge-${Math.min(count, 9)}.png`)
  
  try {
    return nativeImage.createFromPath(badgePath)
  } catch (e) {
    // ถ้าไม่มีไฟล์ ใช้ไฟล์เริ่มต้น
    return nativeImage.createFromPath(path.join(__dirname, 'assets', 'badge-new.png'))
  }
}

// IPC Handlers
ipcMain.on('set-badge-count', (event, count) => {
  setBadgeCount(count)
  
  // อัพเดท tray tooltip ด้วย
  if (tray) {
    const tooltipText = count > 0 
      ? `แอปพลิเคชัน (${count} รายการใหม่)`
      : 'แอปพลิเคชัน'
    tray.setToolTip(tooltipText)
  }
})

ipcMain.handle('get-badge-count', () => {
  if (process.platform === 'darwin') {
    return app.getBadgeCount()
  }
  return 0
})
```

---

## 9.5 ระบบแจ้งเตือนแบบสมบูรณ์

```javascript
// main.js - ระบบแจ้งเตือนครบวงจร

const { app, BrowserWindow, Tray, Menu, nativeImage, Notification, ipcMain } = require('electron')
const path = require('path')
const EventEmitter = require('events')

// ==========================================
// NotificationManager Class
// ==========================================
class NotificationManager extends EventEmitter {
  constructor() {
    super()
    this.queue = []
    this.history = []
    this.maxHistory = 50
    this.enabled = true
    this.unreadCount = 0
  }
  
  // เปิด/ปิดการแจ้งเตือน
  setEnabled(enabled) {
    this.enabled = enabled
    if (mainWindow) {
      mainWindow.webContents.send('notifications-state', enabled)
    }
  }
  
  // ส่ง notification
  async send(options) {
    if (!this.enabled) {
      console.log('Notifications ถูกปิดอยู่')
      return null
    }
    
    if (!Notification.isSupported()) {
      console.warn('Notification ไม่ได้รับการสนับสนุน')
      return null
    }
    
    const notifData = {
      id: Date.now().toString(),
      timestamp: new Date().toISOString(),
      ...options,
      read: false
    }
    
    // เพิ่มใน history
    this.history.unshift(notifData)
    if (this.history.length > this.maxHistory) {
      this.history.pop()
    }
    
    // เพิ่ม unread count
    this.unreadCount++
    setBadgeCount(this.unreadCount)
    
    // แจ้ง renderer
    if (mainWindow && !mainWindow.isDestroyed()) {
      mainWindow.webContents.send('notification-received', notifData)
    }
    
    // สร้างและแสดง notification
    const notification = this._createNotification(notifData)
    notification.show()
    
    return notifData.id
  }
  
  _createNotification(data) {
    const notification = new Notification({
      title: data.title,
      body: data.body,
      icon: data.icon || path.join(__dirname, 'assets', 'notification.png'),
      silent: data.silent || false,
      urgency: data.urgency || 'normal',
      subtitle: data.subtitle || '',
      actions: data.actions ? data.actions.map(a => ({ type: 'button', text: a })) : [],
      hasReply: data.hasReply || false,
      replyPlaceholder: data.replyPlaceholder || 'พิมพ์ที่นี่...'
    })
    
    notification.on('click', () => {
      this.markAsRead(data.id)
      showWindow()
      
      if (mainWindow) {
        mainWindow.webContents.send('notification-clicked', data.id)
      }
      
      this.emit('click', data.id)
    })
    
    notification.on('close', () => {
      this.emit('close', data.id)
    })
    
    notification.on('reply', (event, reply) => {
      if (mainWindow) {
        mainWindow.webContents.send('notification-reply', { id: data.id, reply })
      }
      this.emit('reply', data.id, reply)
    })
    
    notification.on('action', (event, index) => {
      const actionLabel = data.actions ? data.actions[index] : ''
      
      if (mainWindow) {
        mainWindow.webContents.send('notification-action', {
          id: data.id,
          actionIndex: index,
          actionLabel
        })
      }
      
      this.emit('action', data.id, index, actionLabel)
    })
    
    return notification
  }
  
  // อ่านแจ้งเตือน
  markAsRead(id) {
    const notif = this.history.find(n => n.id === id)
    if (notif && !notif.read) {
      notif.read = true
      this.unreadCount = Math.max(0, this.unreadCount - 1)
      setBadgeCount(this.unreadCount)
      
      if (mainWindow) {
        mainWindow.webContents.send('notification-read', id)
      }
    }
  }
  
  markAllAsRead() {
    this.history.forEach(n => n.read = true)
    this.unreadCount = 0
    setBadgeCount(0)
    
    if (mainWindow) {
      mainWindow.webContents.send('all-notifications-read')
    }
  }
  
  getHistory() {
    return this.history
  }
  
  getUnreadCount() {
    return this.unreadCount
  }
  
  clearHistory() {
    this.history = []
    this.unreadCount = 0
    setBadgeCount(0)
  }
}

const notificationManager = new NotificationManager()

// ==========================================
// IPC Handlers สำหรับ NotificationManager
// ==========================================

// ส่ง notification
ipcMain.handle('notify', async (event, options) => {
  const id = await notificationManager.send(options)
  return { id, sent: !!id }
})

// ดึง history
ipcMain.handle('get-notification-history', () => {
  return notificationManager.getHistory()
})

// อ่านแล้ว
ipcMain.on('mark-notification-read', (event, id) => {
  notificationManager.markAsRead(id)
})

ipcMain.on('mark-all-read', () => {
  notificationManager.markAllAsRead()
})

// เปิด/ปิดการแจ้งเตือน
ipcMain.on('toggle-notifications', (event, enabled) => {
  notificationManager.setEnabled(enabled)
  // อัพเดท tray menu ด้วย
  updateTrayMenu({ notificationsEnabled: enabled })
})

ipcMain.handle('get-unread-count', () => {
  return notificationManager.getUnreadCount()
})
```

---

## 9.6 Tray ขั้นสูง - Custom Tray Window

```javascript
// main.js - Custom Tray Popup Window (เหมือน calendar/clock ใน macOS menu bar)

let trayWindow = null

function createTrayWindow() {
  trayWindow = new BrowserWindow({
    width: 300,
    height: 400,
    show: false,
    frame: false,         // ไม่มี title bar
    transparent: true,    // ทำให้ background โปร่งใส
    resizable: false,
    alwaysOnTop: true,    // อยู่บนสุดเสมอ
    skipTaskbar: true,    // ไม่แสดงใน taskbar
    webPreferences: {
      preload: path.join(__dirname, 'tray-window-preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })
  
  trayWindow.loadFile('tray-window.html')
  
  // ซ่อนเมื่อ focus หาย
  trayWindow.on('blur', () => {
    if (!trayWindow.webContents.isDevToolsOpened()) {
      trayWindow.hide()
    }
  })
}

function toggleTrayWindow() {
  if (!trayWindow) {
    createTrayWindow()
    return
  }
  
  if (trayWindow.isVisible()) {
    trayWindow.hide()
  } else {
    showTrayWindow()
  }
}

function showTrayWindow() {
  // คำนวณตำแหน่งให้ popup อยู่ใกล้ tray icon
  const trayBounds = tray.getBounds()
  const windowBounds = trayWindow.getBounds()
  
  let x, y
  
  if (process.platform === 'darwin') {
    // macOS: แสดงด้านล่าง tray icon ใน menu bar
    x = Math.round(trayBounds.x + (trayBounds.width / 2) - (windowBounds.width / 2))
    y = Math.round(trayBounds.y + trayBounds.height)
  } else if (process.platform === 'win32') {
    // Windows: แสดงด้านบน tray icon ใน taskbar
    const { screen } = require('electron')
    const primaryDisplay = screen.getPrimaryDisplay()
    const { height: screenHeight } = primaryDisplay.workAreaSize
    
    x = Math.round(trayBounds.x + (trayBounds.width / 2) - (windowBounds.width / 2))
    y = Math.round(screenHeight - trayBounds.height - windowBounds.height)
  } else {
    // Linux
    const { screen } = require('electron')
    const cursorPos = screen.getCursorScreenPoint()
    x = cursorPos.x
    y = cursorPos.y
  }
  
  // ป้องกันออกนอกขอบจอ
  const { screen } = require('electron')
  const primaryDisplay = screen.getPrimaryDisplay()
  const { width: screenWidth } = primaryDisplay.workAreaSize
  
  x = Math.max(0, Math.min(x, screenWidth - windowBounds.width))
  
  trayWindow.setPosition(x, y, false)
  trayWindow.show()
  trayWindow.focus()
}
```

---

## 9.7 Preload Script สมบูรณ์

```javascript
// preload.js

const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('trayAPI', {
  // ==========================================
  // Notification APIs
  // ==========================================
  
  // ส่ง notification พื้นฐาน
  notify: (title, body, options = {}) =>
    ipcRenderer.invoke('notify', { title, body, ...options }),
  
  // ส่ง notification พร้อม action buttons (macOS)
  notifyWithActions: (title, body, actions) =>
    ipcRenderer.invoke('notify', {
      title, body,
      actions,
      silent: false
    }),
  
  // ส่ง notification พร้อม reply (macOS)
  notifyWithReply: (title, body, placeholder) =>
    ipcRenderer.invoke('notify', {
      title, body,
      hasReply: true,
      replyPlaceholder: placeholder
    }),
  
  // ==========================================
  // Notification History
  // ==========================================
  getHistory: () => ipcRenderer.invoke('get-notification-history'),
  getUnreadCount: () => ipcRenderer.invoke('get-unread-count'),
  markAsRead: (id) => ipcRenderer.send('mark-notification-read', id),
  markAllAsRead: () => ipcRenderer.send('mark-all-read'),
  
  // ==========================================
  // Badge / Unread Count
  // ==========================================
  setBadgeCount: (count) => ipcRenderer.send('set-badge-count', count),
  
  // ==========================================
  // Notifications Toggle
  // ==========================================
  toggleNotifications: (enabled) =>
    ipcRenderer.send('toggle-notifications', enabled),
  
  // ==========================================
  // Tray Balloon (Windows)
  // ==========================================
  showBalloon: (title, content) =>
    ipcRenderer.send('show-balloon', { title, content }),
  
  // ==========================================
  // Event Listeners
  // ==========================================
  
  // รับเมื่อมี notification ใหม่
  onNotificationReceived: (callback) => {
    const handler = (event, data) => callback(data)
    ipcRenderer.on('notification-received', handler)
    return () => ipcRenderer.removeListener('notification-received', handler)
  },
  
  // รับเมื่อ notification ถูกคลิก
  onNotificationClicked: (callback) => {
    const handler = (event, id) => callback(id)
    ipcRenderer.on('notification-clicked', handler)
    return () => ipcRenderer.removeListener('notification-clicked', handler)
  },
  
  // รับเมื่อมี notification reply
  onNotificationReply: (callback) => {
    const handler = (event, data) => callback(data)
    ipcRenderer.on('notification-reply', handler)
    return () => ipcRenderer.removeListener('notification-reply', handler)
  },
  
  // รับเมื่อ notification action ถูกกด
  onNotificationAction: (callback) => {
    const handler = (event, data) => callback(data)
    ipcRenderer.on('notification-action', handler)
    return () => ipcRenderer.removeListener('notification-action', handler)
  },
  
  // รับเมื่อ notification ถูกอ่าน
  onNotificationRead: (callback) => {
    const handler = (event, id) => callback(id)
    ipcRenderer.on('notification-read', handler)
    return () => ipcRenderer.removeListener('notification-read', handler)
  },
  
  // รับเมื่อ notifications state เปลี่ยน
  onNotificationsStateChanged: (callback) => {
    const handler = (event, enabled) => callback(enabled)
    ipcRenderer.on('notifications-state', handler)
    return () => ipcRenderer.removeListener('notifications-state', handler)
  },
  
  // รับ tray actions
  onTrayAction: (callback) => {
    const handler = (event, action) => callback(action)
    ipcRenderer.on('tray-action', handler)
    return () => ipcRenderer.removeListener('tray-action', handler)
  }
})
```

---

## 9.8 Renderer - Notification Center UI

```javascript
// renderer.js - Notification Center

class NotificationCenter {
  constructor() {
    this.notifications = []
    this.cleanups = []
    this.isEnabled = true
    this.init()
  }
  
  async init() {
    // โหลด history
    this.notifications = await window.trayAPI.getHistory()
    this.renderNotifications()
    
    // ลงทะเบียน event listeners
    this.cleanups.push(
      window.trayAPI.onNotificationReceived((data) => {
        this.notifications.unshift(data)
        this.renderNotifications()
        this.updateBadge()
        this.showToast(data)
      }),
      
      window.trayAPI.onNotificationClicked((id) => {
        this.markAsRead(id)
        this.scrollToNotification(id)
      }),
      
      window.trayAPI.onNotificationReply(({ id, reply }) => {
        console.log(`Reply สำหรับ ${id}:`, reply)
        this.addReplyToNotification(id, reply)
      }),
      
      window.trayAPI.onNotificationAction(({ id, actionIndex, actionLabel }) => {
        console.log(`Action "${actionLabel}" สำหรับ ${id}`)
        this.handleNotificationAction(id, actionIndex, actionLabel)
      }),
      
      window.trayAPI.onNotificationRead((id) => {
        const notif = this.notifications.find(n => n.id === id)
        if (notif) {
          notif.read = true
          this.renderNotifications()
          this.updateBadge()
        }
      }),
      
      window.trayAPI.onTrayAction((action) => {
        this.handleTrayAction(action)
      })
    )
    
    this.bindUIEvents()
    this.updateBadge()
  }
  
  bindUIEvents() {
    // ปุ่มส่ง notification ประเภทต่างๆ
    document.getElementById('btnBasicNotif').addEventListener('click', () => {
      this.sendBasicNotification()
    })
    
    document.getElementById('btnActionNotif').addEventListener('click', () => {
      this.sendActionNotification()
    })
    
    document.getElementById('btnReplyNotif').addEventListener('click', () => {
      this.sendReplyNotification()
    })
    
    document.getElementById('btnUrgentNotif').addEventListener('click', () => {
      this.sendUrgentNotification()
    })
    
    // ปุ่มจัดการ
    document.getElementById('btnMarkAllRead').addEventListener('click', () => {
      window.trayAPI.markAllAsRead()
      this.notifications.forEach(n => n.read = true)
      this.renderNotifications()
      this.updateBadge()
    })
    
    document.getElementById('btnClearAll').addEventListener('click', () => {
      this.notifications = []
      this.renderNotifications()
    })
    
    // Toggle notifications
    const toggle = document.getElementById('notifToggle')
    toggle.addEventListener('change', (e) => {
      this.isEnabled = e.target.checked
      window.trayAPI.toggleNotifications(this.isEnabled)
    })
  }
  
  async sendBasicNotification() {
    const result = await window.trayAPI.notify(
      'การแจ้งเตือนพื้นฐาน',
      'นี่คือการแจ้งเตือนธรรมดา คลิกเพื่อดูรายละเอียด'
    )
    console.log('ส่ง notification:', result)
  }
  
  async sendActionNotification() {
    const result = await window.trayAPI.notifyWithActions(
      'มีงานใหม่เข้ามา',
      'ต้องการรับงานหรือไม่?',
      ['รับงาน', 'ปฏิเสธ', 'ดูรายละเอียด']
    )
    console.log('Action notification sent:', result)
  }
  
  async sendReplyNotification() {
    const result = await window.trayAPI.notifyWithReply(
      'ข้อความจาก สมชาย',
      'สวัสดีครับ วันนี้ว่างไหม?',
      'พิมพ์ข้อความตอบกลับ...'
    )
    console.log('Reply notification sent:', result)
  }
  
  async sendUrgentNotification() {
    await window.trayAPI.notify(
      '⚠️ แจ้งเตือนด่วน',
      'พื้นที่เหลือน้อยมาก กรุณาล้างข้อมูลเก่า',
      {
        urgency: 'critical',
        silent: false
      }
    )
  }
  
  renderNotifications() {
    const container = document.getElementById('notificationList')
    const unread = this.notifications.filter(n => !n.read)
    
    if (this.notifications.length === 0) {
      container.innerHTML = `
        <div class="empty-state">
          <p>ไม่มีการแจ้งเตือน</p>
        </div>
      `
      return
    }
    
    container.innerHTML = this.notifications.map(notif => `
      <div class="notification-item ${notif.read ? 'read' : 'unread'}" 
           data-id="${notif.id}"
           onclick="notifCenter.markAsRead('${notif.id}')">
        <div class="notif-header">
          <strong>${this.escapeHtml(notif.title)}</strong>
          <span class="notif-time">${this.formatTime(notif.timestamp)}</span>
        </div>
        <p class="notif-body">${this.escapeHtml(notif.body)}</p>
        ${notif.read ? '' : '<span class="unread-dot"></span>'}
      </div>
    `).join('')
  }
  
  updateBadge() {
    const unreadCount = this.notifications.filter(n => !n.read).length
    window.trayAPI.setBadgeCount(unreadCount)
    
    const badge = document.getElementById('unreadBadge')
    if (unreadCount > 0) {
      badge.textContent = unreadCount > 99 ? '99+' : unreadCount
      badge.style.display = 'inline-block'
    } else {
      badge.style.display = 'none'
    }
  }
  
  markAsRead(id) {
    window.trayAPI.markAsRead(id)
    const notif = this.notifications.find(n => n.id === id)
    if (notif) {
      notif.read = true
      this.renderNotifications()
      this.updateBadge()
    }
  }
  
  showToast(data) {
    const toast = document.createElement('div')
    toast.className = 'toast'
    toast.innerHTML = `
      <strong>${this.escapeHtml(data.title)}</strong>
      <p>${this.escapeHtml(data.body)}</p>
    `
    
    document.body.appendChild(toast)
    
    // แสดง toast
    setTimeout(() => toast.classList.add('show'), 10)
    
    // ซ่อนหลัง 3 วินาที
    setTimeout(() => {
      toast.classList.remove('show')
      setTimeout(() => toast.remove(), 300)
    }, 3000)
  }
  
  handleTrayAction(action) {
    console.log('Tray action:', action)
    // จัดการตาม action
    switch (action) {
      case 'open-settings':
        document.getElementById('settingsTab').click()
        break
      case 'check-updates':
        this.checkForUpdates()
        break
    }
  }
  
  addReplyToNotification(id, reply) {
    const notif = this.notifications.find(n => n.id === id)
    if (notif) {
      notif.reply = reply
      this.renderNotifications()
    }
  }
  
  handleNotificationAction(id, actionIndex, actionLabel) {
    console.log(`Notification ${id}: Action "${actionLabel}" (index ${actionIndex})`)
  }
  
  scrollToNotification(id) {
    const element = document.querySelector(`[data-id="${id}"]`)
    if (element) {
      element.scrollIntoView({ behavior: 'smooth', block: 'center' })
    }
  }
  
  checkForUpdates() {
    this.sendBasicNotification()
  }
  
  formatTime(timestamp) {
    const date = new Date(timestamp)
    const now = new Date()
    const diff = now - date
    
    if (diff < 60000) return 'เมื่อกี้'
    if (diff < 3600000) return `${Math.floor(diff / 60000)} นาทีที่แล้ว`
    if (diff < 86400000) return `${Math.floor(diff / 3600000)} ชั่วโมงที่แล้ว`
    return date.toLocaleDateString('th-TH')
  }
  
  escapeHtml(text) {
    if (!text) return ''
    const div = document.createElement('div')
    div.appendChild(document.createTextNode(text))
    return div.innerHTML
  }
  
  destroy() {
    this.cleanups.forEach(fn => fn())
  }
}

const notifCenter = new NotificationCenter()
window.addEventListener('beforeunload', () => notifCenter.destroy())
```

---

## 9.9 index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Tray & Notifications Demo</title>
  <style>
    * { box-sizing: border-box; }
    
    body {
      font-family: 'Sarabun', sans-serif;
      margin: 0;
      background: #f5f5f5;
      display: flex;
      flex-direction: column;
      height: 100vh;
    }
    
    header {
      background: #1976d2;
      color: white;
      padding: 16px 20px;
      display: flex;
      align-items: center;
      justify-content: space-between;
    }
    
    header h1 { margin: 0; font-size: 20px; }
    
    #unreadBadge {
      background: #f44336;
      color: white;
      border-radius: 10px;
      padding: 2px 8px;
      font-size: 12px;
      display: none;
    }
    
    main {
      flex: 1;
      display: flex;
      gap: 16px;
      padding: 16px;
      overflow: hidden;
    }
    
    .panel {
      background: white;
      border-radius: 8px;
      padding: 16px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    
    .left-panel { width: 280px; flex-shrink: 0; overflow-y: auto; }
    .right-panel { flex: 1; overflow-y: auto; }
    
    h2 { font-size: 16px; color: #333; margin: 0 0 12px; border-bottom: 1px solid #eee; padding-bottom: 8px; }
    
    .btn-group { display: flex; flex-direction: column; gap: 8px; }
    
    button {
      padding: 10px 14px;
      border: none;
      border-radius: 6px;
      cursor: pointer;
      font-size: 14px;
      font-family: inherit;
      text-align: left;
    }
    
    .btn-blue { background: #e3f2fd; color: #1565c0; }
    .btn-green { background: #e8f5e9; color: #2e7d32; }
    .btn-orange { background: #fff3e0; color: #e65100; }
    .btn-red { background: #ffebee; color: #c62828; }
    
    button:hover { filter: brightness(0.95); }
    
    .toggle-row {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 8px 0;
      border-top: 1px solid #eee;
      margin-top: 8px;
    }
    
    .toggle {
      position: relative;
      width: 44px;
      height: 24px;
    }
    
    .toggle input { opacity: 0; width: 0; height: 0; }
    
    .toggle-slider {
      position: absolute;
      inset: 0;
      background: #ccc;
      border-radius: 24px;
      cursor: pointer;
      transition: 0.3s;
    }
    
    .toggle-slider::before {
      content: '';
      position: absolute;
      width: 18px;
      height: 18px;
      background: white;
      border-radius: 50%;
      top: 3px;
      left: 3px;
      transition: 0.3s;
    }
    
    input:checked + .toggle-slider { background: #1976d2; }
    input:checked + .toggle-slider::before { transform: translateX(20px); }
    
    .notification-item {
      padding: 12px;
      border-radius: 6px;
      margin-bottom: 8px;
      cursor: pointer;
      border: 1px solid transparent;
      position: relative;
      transition: background 0.2s;
    }
    
    .notification-item.unread {
      background: #e3f2fd;
      border-color: #90caf9;
    }
    
    .notification-item.read {
      background: #fafafa;
      border-color: #e0e0e0;
    }
    
    .notification-item:hover { filter: brightness(0.97); }
    
    .notif-header {
      display: flex;
      justify-content: space-between;
      margin-bottom: 4px;
    }
    
    .notif-time { font-size: 12px; color: #888; }
    
    .notif-body { font-size: 13px; color: #555; margin: 0; }
    
    .unread-dot {
      position: absolute;
      right: 12px;
      top: 50%;
      transform: translateY(-50%);
      width: 8px;
      height: 8px;
      background: #1976d2;
      border-radius: 50%;
    }
    
    .empty-state {
      text-align: center;
      color: #999;
      padding: 40px 20px;
    }
    
    .action-row {
      display: flex;
      gap: 8px;
      margin-bottom: 12px;
    }
    
    .action-row button {
      flex: 1;
      text-align: center;
      background: #f5f5f5;
      color: #555;
      font-size: 13px;
      padding: 8px;
    }
    
    /* Toast notification */
    .toast {
      position: fixed;
      bottom: 20px;
      right: 20px;
      background: #333;
      color: white;
      padding: 12px 16px;
      border-radius: 8px;
      max-width: 280px;
      opacity: 0;
      transform: translateY(20px);
      transition: all 0.3s;
      z-index: 1000;
    }
    
    .toast.show {
      opacity: 1;
      transform: translateY(0);
    }
    
    .toast strong { display: block; margin-bottom: 4px; }
    .toast p { margin: 0; font-size: 13px; opacity: 0.85; }
  </style>
</head>
<body>
  <header>
    <h1>Tray & Notifications</h1>
    <span id="unreadBadge">0</span>
  </header>
  
  <main>
    <div class="panel left-panel">
      <h2>ส่งการแจ้งเตือน</h2>
      <div class="btn-group">
        <button class="btn-blue" id="btnBasicNotif">📢 แจ้งเตือนพื้นฐาน</button>
        <button class="btn-green" id="btnActionNotif">🎯 แจ้งเตือน + Actions</button>
        <button class="btn-orange" id="btnReplyNotif">💬 แจ้งเตือน + Reply</button>
        <button class="btn-red" id="btnUrgentNotif">⚠️ แจ้งเตือนด่วน</button>
      </div>
      
      <div class="toggle-row">
        <span style="font-size:14px">เปิดการแจ้งเตือน</span>
        <label class="toggle">
          <input type="checkbox" id="notifToggle" checked>
          <span class="toggle-slider"></span>
        </label>
      </div>
    </div>
    
    <div class="panel right-panel">
      <h2>ศูนย์การแจ้งเตือน</h2>
      
      <div class="action-row">
        <button id="btnMarkAllRead">✓ อ่านทั้งหมด</button>
        <button id="btnClearAll">🗑 ล้างทั้งหมด</button>
      </div>
      
      <div id="notificationList">
        <div class="empty-state">ไม่มีการแจ้งเตือน</div>
      </div>
    </div>
  </main>
  
  <script src="renderer.js"></script>
</body>
</html>
```

---

## สรุปบทเรียน

### สิ่งที่ได้เรียนรู้

1. **Tray** - ไอคอนใน system tray พร้อม menu, tooltip, balloon
2. **Notification API** - ส่งการแจ้งเตือน native ของ OS
3. **Actions** - ปุ่มใน notification (macOS)
4. **Reply** - ช่องตอบกลับใน notification (macOS)
5. **Badge Count** - ตัวเลขบน dock icon (macOS) / overlay icon (Windows)
6. **Background App** - แอปที่ทำงานใน tray แม้ปิด window แล้ว

### Best Practices

- **เก็บ reference ของ `Tray`** ไว้เสมอ ไม่งั้น garbage collector จะลบทิ้ง
- **ใช้ template image** บน macOS เพื่อให้ icon ปรับสีอัตโนมัติ
- **ทดสอบบนทุก platform** เพราะ tray behavior ต่างกันมาก
- **ให้ผู้ใช้ควบคุม** การแจ้งเตือนได้ (เปิด/ปิด)
- **อย่าส่ง notification บ่อยเกินไป** จะรบกวนผู้ใช้
- **cleanup listeners** ทุกครั้งเพื่อป้องกัน memory leak
