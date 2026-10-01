# ตอนที่ 16: Power Monitor & System Events

## Power Monitor ใน Electron

`powerMonitor` module ให้ความสามารถในการ monitor สถานะการใช้พลังงานของระบบ เช่น เมื่อเครื่องเข้าสู่ sleep mode, lock screen หรือเมื่อ resume จาก sleep

```javascript
const { powerMonitor } = require('electron')
```

**หมายเหตุ:** `powerMonitor` ใช้ได้เฉพาะใน main process หลังจาก app `ready` event

## Events ที่สำคัญ

### suspend / resume

```javascript
// main.js
const { app, powerMonitor } = require('electron')

app.whenReady().then(() => {
  // เมื่อเครื่องเข้าสู่ sleep mode
  powerMonitor.on('suspend', () => {
    console.log('System is going to sleep')
    // บันทึกสถานะแอปพลิเคชัน
    saveApplicationState()
    // หยุด sync/polling ต่างๆ
    pauseBackgroundTasks()
  })
  
  // เมื่อเครื่อง resume จาก sleep
  powerMonitor.on('resume', () => {
    console.log('System resumed from sleep')
    // refresh data ที่อาจ stale
    refreshData()
    // resume background tasks
    resumeBackgroundTasks()
  })
})

function saveApplicationState() {
  // บันทึกสถานะก่อน sleep
  console.log('Saving state before sleep...')
}

function refreshData() {
  // รีเฟรชข้อมูลที่อาจ stale หลัง sleep
  console.log('Refreshing data after resume...')
}

let backgroundTasks = []

function pauseBackgroundTasks() {
  backgroundTasks.forEach(task => task.pause?.())
}

function resumeBackgroundTasks() {
  backgroundTasks.forEach(task => task.resume?.())
}
```

### shutdown

```javascript
// main.js
const { app, powerMonitor } = require('electron')

app.whenReady().then(() => {
  // เมื่อ Windows/Linux ปิดเครื่อง
  powerMonitor.on('shutdown', () => {
    console.log('System is shutting down')
    
    // บันทึก unsaved data
    emergencySave()
    
    // Electron จะ quit โดยอัตโนมัติหลัง event นี้
    // บน Windows: เรียก app.quit() อาจช่วยให้ quit ได้สะอาดขึ้น
    app.quit()
  })
})

function emergencySave() {
  const fs = require('fs')
  const path = require('path')
  const { app } = require('electron')
  
  try {
    const emergencyPath = path.join(app.getPath('userData'), 'emergency-save.json')
    fs.writeFileSync(emergencyPath, JSON.stringify({
      timestamp: new Date().toISOString(),
      state: 'saved-before-shutdown'
    }), 'utf8')
    console.log('Emergency save completed')
  } catch (error) {
    console.error('Emergency save failed:', error)
  }
}
```

### lock-screen / unlock-screen

```javascript
// main.js
const { app, powerMonitor, BrowserWindow } = require('electron')

let mainWindow

app.whenReady().then(() => {
  mainWindow = createWindow()
  
  // เมื่อ lock screen
  powerMonitor.on('lock-screen', () => {
    console.log('Screen locked')
    
    // ซ่อน sensitive content
    mainWindow?.webContents.send('app:screen-locked')
    
    // หยุด media playback
    mainWindow?.webContents.executeJavaScript(`
      document.querySelectorAll('video, audio').forEach(el => el.pause())
    `)
    
    // อาจต้องการ hide window
    if (mainWindow?.isVisible()) {
      mainWindow.minimize()
    }
  })
  
  // เมื่อ unlock screen
  powerMonitor.on('unlock-screen', () => {
    console.log('Screen unlocked')
    mainWindow?.webContents.send('app:screen-unlocked')
  })
})
```

### on-battery / on-ac

```javascript
// main.js
const { app, powerMonitor } = require('electron')

app.whenReady().then(() => {
  // เมื่อเปลี่ยนไปใช้แบตเตอรี่
  powerMonitor.on('on-battery', () => {
    console.log('Running on battery')
    // ลด background activity เพื่อประหยัดแบตเตอรี่
    enableBatterySaveMode()
  })
  
  // เมื่อเสียบชาร์จ
  powerMonitor.on('on-ac', () => {
    console.log('Running on AC power')
    // กลับสู่ performance mode
    disableBatterySaveMode()
  })
  
  // ตรวจสอบสถานะปัจจุบัน
  if (powerMonitor.isOnBatteryPower()) {
    console.log('Currently on battery')
    enableBatterySaveMode()
  }
})

let batterySaveMode = false

function enableBatterySaveMode() {
  if (batterySaveMode) return
  batterySaveMode = true
  console.log('Battery save mode: ON')
  // ลด polling frequency
  // ปิด animations
  // ลดความสว่างหน้าจอ (ถ้า API รองรับ)
}

function disableBatterySaveMode() {
  if (!batterySaveMode) return
  batterySaveMode = false
  console.log('Battery save mode: OFF')
}
```

## powerSaveBlocker

`powerSaveBlocker` ป้องกันไม่ให้ระบบเข้าสู่ sleep mode ขณะที่แอปกำลังทำงานสำคัญ

```javascript
// main.js
const { powerSaveBlocker, ipcMain } = require('electron')

let powerSaveBlockerId = null

// เริ่ม block sleep
function startPowerSaveBlocker(type = 'prevent-app-suspension') {
  if (powerSaveBlockerId !== null) {
    console.log('Power save blocker already active:', powerSaveBlockerId)
    return powerSaveBlockerId
  }
  
  // types:
  // 'prevent-app-suspension' - ป้องกัน app ถูก suspend (macOS)
  // 'prevent-display-sleep' - ป้องกัน display ดับ
  powerSaveBlockerId = powerSaveBlocker.start(type)
  console.log('Power save blocker started, id:', powerSaveBlockerId)
  return powerSaveBlockerId
}

// หยุด block sleep
function stopPowerSaveBlocker() {
  if (powerSaveBlockerId !== null && 
      powerSaveBlocker.isStarted(powerSaveBlockerId)) {
    powerSaveBlocker.stop(powerSaveBlockerId)
    console.log('Power save blocker stopped, id:', powerSaveBlockerId)
    powerSaveBlockerId = null
  }
}

// IPC handlers
ipcMain.handle('power:start-blocker', (event, type) => {
  const validTypes = ['prevent-app-suspension', 'prevent-display-sleep']
  if (!validTypes.includes(type)) type = 'prevent-app-suspension'
  
  const id = startPowerSaveBlocker(type)
  return { success: true, id, isActive: powerSaveBlocker.isStarted(id) }
})

ipcMain.handle('power:stop-blocker', () => {
  stopPowerSaveBlocker()
  return { success: true }
})

ipcMain.handle('power:blocker-status', () => {
  const isActive = powerSaveBlockerId !== null && 
                   powerSaveBlocker.isStarted(powerSaveBlockerId)
  return { isActive, id: powerSaveBlockerId }
})
```

### ตัวอย่าง: Video Player ป้องกัน sleep

```javascript
// main.js
const { powerSaveBlocker, ipcMain } = require('electron')

let displaySleepBlocker = null

// เมื่อเล่น video ป้องกันหน้าจอดับ
ipcMain.on('video:playing', () => {
  if (displaySleepBlocker === null || 
      !powerSaveBlocker.isStarted(displaySleepBlocker)) {
    displaySleepBlocker = powerSaveBlocker.start('prevent-display-sleep')
    console.log('Display sleep prevention started for video playback')
  }
})

// เมื่อ pause video
ipcMain.on('video:paused', () => {
  if (displaySleepBlocker !== null) {
    powerSaveBlocker.stop(displaySleepBlocker)
    displaySleepBlocker = null
    console.log('Display sleep prevention stopped')
  }
})

ipcMain.on('video:stopped', () => {
  if (displaySleepBlocker !== null) {
    powerSaveBlocker.stop(displaySleepBlocker)
    displaySleepBlocker = null
    console.log('Display sleep prevention stopped')
  }
})
```

## getSystemIdleState / getSystemIdleTime

```javascript
// main.js
const { powerMonitor } = require('electron')

// getSystemIdleState: ตรวจสอบ idle state ของระบบ
// Returns: 'active' | 'idle' | 'locked' | 'unknown'
function checkIdleState(thresholdSeconds = 60) {
  const state = powerMonitor.getSystemIdleState(thresholdSeconds)
  console.log('System idle state:', state)
  return state
}

// getSystemIdleTime: จำนวนวินาทีที่ผ่านไปนับจาก user interaction ล่าสุด
function getIdleTime() {
  const idleTime = powerMonitor.getSystemIdleTime()
  console.log('System idle time:', idleTime, 'seconds')
  return idleTime
}

// ตรวจสอบ idle state แบบ periodic
function startIdleMonitor(interval = 30000) {
  return setInterval(() => {
    const idleTime = getIdleTime()
    const state = checkIdleState(300) // 5 minutes threshold
    
    if (state === 'locked') {
      console.log('System is locked')
    } else if (state === 'idle' || idleTime > 300) {
      console.log(`User has been idle for ${idleTime} seconds`)
      handleUserIdle(idleTime)
    } else {
      // User is active
    }
  }, interval)
}

function handleUserIdle(idleSeconds) {
  // Auto-save, pause sync, etc.
  if (idleSeconds > 600) { // 10 minutes
    console.log('Auto-saving due to inactivity...')
  }
}
```

## Complete Power Monitor Application

### main.js

```javascript
// main.js
const { 
  app, BrowserWindow, ipcMain, 
  powerMonitor, powerSaveBlocker 
} = require('electron')
const path = require('path')

let mainWindow
let powerBlockerId = null
let idleMonitorTimer = null
const powerEvents = []

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 900,
    height: 700,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  mainWindow.loadFile('index.html')
  mainWindow.webContents.on('did-finish-load', () => {
    // ส่ง initial state
    sendPowerState()
  })
}

// ─── Power Event Tracking ────────────────────────────────────────
function logEvent(type, data = {}) {
  const event = {
    type,
    timestamp: new Date().toISOString(),
    ...data
  }
  powerEvents.push(event)
  if (powerEvents.length > 100) powerEvents.shift()
  
  mainWindow?.webContents.send('power:event', event)
  console.log('Power event:', event)
}

function sendPowerState() {
  const state = {
    isOnBattery: powerMonitor.isOnBatteryPower(),
    idleTime: powerMonitor.getSystemIdleTime(),
    idleState: powerMonitor.getSystemIdleState(60),
    powerBlockerActive: powerBlockerId !== null && powerSaveBlocker.isStarted(powerBlockerId)
  }
  mainWindow?.webContents.send('power:state', state)
}

// ─── Power Monitor Events ────────────────────────────────────────
app.whenReady().then(() => {
  createWindow()
  
  powerMonitor.on('suspend', () => {
    logEvent('suspend', { message: 'ระบบกำลังเข้าสู่ Sleep Mode' })
  })
  
  powerMonitor.on('resume', () => {
    logEvent('resume', { message: 'ระบบ Resume จาก Sleep Mode' })
    sendPowerState()
  })
  
  powerMonitor.on('shutdown', () => {
    logEvent('shutdown', { message: 'ระบบกำลังปิดเครื่อง' })
  })
  
  powerMonitor.on('lock-screen', () => {
    logEvent('lock-screen', { message: 'หน้าจอถูก Lock' })
    mainWindow?.webContents.send('security:screen-locked')
  })
  
  powerMonitor.on('unlock-screen', () => {
    logEvent('unlock-screen', { message: 'หน้าจอถูก Unlock' })
    sendPowerState()
  })
  
  powerMonitor.on('on-battery', () => {
    logEvent('on-battery', { message: 'เปลี่ยนไปใช้แบตเตอรี่' })
    sendPowerState()
  })
  
  powerMonitor.on('on-ac', () => {
    logEvent('on-ac', { message: 'เสียบชาร์จแล้ว' })
    sendPowerState()
  })
  
  // Idle monitoring
  idleMonitorTimer = setInterval(() => {
    const idleTime = powerMonitor.getSystemIdleTime()
    const idleState = powerMonitor.getSystemIdleState(120)
    mainWindow?.webContents.send('power:idle-update', { idleTime, idleState })
  }, 5000)
})

// ─── IPC Handlers ────────────────────────────────────────────────
ipcMain.handle('power:get-state', () => {
  return {
    isOnBattery: powerMonitor.isOnBatteryPower(),
    idleTime: powerMonitor.getSystemIdleTime(),
    idleState: powerMonitor.getSystemIdleState(60),
    powerBlockerActive: powerBlockerId !== null && 
                        powerSaveBlocker.isStarted(powerBlockerId),
    powerBlockerId
  }
})

ipcMain.handle('power:start-blocker', (event, type) => {
  const validTypes = ['prevent-app-suspension', 'prevent-display-sleep']
  const safeType = validTypes.includes(type) ? type : 'prevent-app-suspension'
  
  if (powerBlockerId !== null && powerSaveBlocker.isStarted(powerBlockerId)) {
    return { success: false, error: 'Blocker already active', id: powerBlockerId }
  }
  
  powerBlockerId = powerSaveBlocker.start(safeType)
  logEvent('blocker-started', { type: safeType, id: powerBlockerId })
  return { success: true, id: powerBlockerId, type: safeType }
})

ipcMain.handle('power:stop-blocker', () => {
  if (powerBlockerId === null || !powerSaveBlocker.isStarted(powerBlockerId)) {
    return { success: false, error: 'No active blocker' }
  }
  
  powerSaveBlocker.stop(powerBlockerId)
  logEvent('blocker-stopped', { id: powerBlockerId })
  powerBlockerId = null
  return { success: true }
})

ipcMain.handle('power:get-events', () => {
  return powerEvents.slice(-20)
})

app.on('window-all-closed', () => {
  if (idleMonitorTimer) clearInterval(idleMonitorTimer)
  if (powerBlockerId !== null && powerSaveBlocker.isStarted(powerBlockerId)) {
    powerSaveBlocker.stop(powerBlockerId)
  }
  if (process.platform !== 'darwin') app.quit()
})
```

### preload.js

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('powerAPI', {
  getState: () => ipcRenderer.invoke('power:get-state'),
  startBlocker: (type) => ipcRenderer.invoke('power:start-blocker', type),
  stopBlocker: () => ipcRenderer.invoke('power:stop-blocker'),
  getEvents: () => ipcRenderer.invoke('power:get-events'),
  
  onEvent: (callback) => {
    const fn = (_e, data) => callback(data)
    ipcRenderer.on('power:event', fn)
    return () => ipcRenderer.removeListener('power:event', fn)
  },
  
  onStateUpdate: (callback) => {
    const fn = (_e, data) => callback(data)
    ipcRenderer.on('power:state', fn)
    return () => ipcRenderer.removeListener('power:state', fn)
  },
  
  onIdleUpdate: (callback) => {
    const fn = (_e, data) => callback(data)
    ipcRenderer.on('power:idle-update', fn)
    return () => ipcRenderer.removeListener('power:idle-update', fn)
  },
  
  onScreenLocked: (callback) => {
    const fn = () => callback()
    ipcRenderer.on('security:screen-locked', fn)
    return () => ipcRenderer.removeListener('security:screen-locked', fn)
  }
})
```

### index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="Content-Security-Policy" content="default-src 'self'; style-src 'self' 'unsafe-inline'; script-src 'self';">
  <title>Power Monitor</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui; background: #0f0f1a; color: #e0e0ff; padding: 20px; }
    h1 { margin-bottom: 20px; color: #7b8cde; font-size: 22px; }
    .grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 16px; margin-bottom: 16px; }
    .card { background: #1a1a2e; border-radius: 10px; padding: 16px; border: 1px solid #2a2a4e; }
    h2 { font-size: 13px; color: #888; text-transform: uppercase; letter-spacing: 0.5px; margin-bottom: 12px; }
    .stat { display: flex; justify-content: space-between; padding: 6px 0; border-bottom: 1px solid #2a2a4e; font-size: 14px; }
    .stat:last-child { border-bottom: none; }
    .stat-value { color: #7b8cde; font-weight: 600; }
    .status-dot { width: 10px; height: 10px; border-radius: 50%; display: inline-block; margin-right: 6px; }
    .status-active { background: #34c759; box-shadow: 0 0 6px #34c759; }
    .status-idle { background: #ff9f0a; box-shadow: 0 0 6px #ff9f0a; }
    .status-locked { background: #ff3b30; box-shadow: 0 0 6px #ff3b30; }
    .btn-row { display: flex; gap: 8px; margin-top: 12px; flex-wrap: wrap; }
    button { padding: 8px 16px; border: none; border-radius: 6px; cursor: pointer; font-size: 13px; }
    .btn-green { background: #34c759; color: #000; }
    .btn-red { background: #ff3b30; color: #fff; }
    .btn-blue { background: #007aff; color: #fff; }
    button:hover { opacity: 0.8; }
    #event-log { height: 200px; overflow-y: auto; }
    .event-item { 
      padding: 6px 0; border-bottom: 1px solid #2a2a4e; font-size: 13px;
      display: flex; gap: 10px;
    }
    .event-time { color: #666; white-space: nowrap; font-family: monospace; }
    .event-type { 
      padding: 2px 6px; border-radius: 3px; font-size: 11px; white-space: nowrap;
      background: #2a2a4e; color: #7b8cde;
    }
    .event-suspend .event-type { background: #2a1a1a; color: #ff9f0a; }
    .event-resume .event-type { background: #1a2a1a; color: #34c759; }
    .event-shutdown .event-type { background: #2a0a0a; color: #ff3b30; }
    .event-lock-screen .event-type { background: #2a2a1a; color: #ffd60a; }
    #idle-bar { height: 6px; background: #2a2a4e; border-radius: 3px; margin-top: 8px; }
    #idle-fill { height: 100%; border-radius: 3px; background: #34c759; transition: width 0.5s, background 0.5s; }
    .screen-lock-overlay {
      display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%;
      background: rgba(0,0,0,0.9); z-index: 1000; justify-content: center; align-items: center;
      flex-direction: column; gap: 16px;
    }
    .screen-lock-overlay.visible { display: flex; }
    .lock-icon { font-size: 48px; }
    .lock-text { color: #e0e0ff; font-size: 18px; }
  </style>
</head>
<body>
  <h1>Power Monitor</h1>
  
  <div class="grid">
    <!-- Power State -->
    <div class="card">
      <h2>สถานะพลังงาน</h2>
      <div class="stat">
        <span>แหล่งพลังงาน</span>
        <span class="stat-value" id="power-source">กำลังโหลด...</span>
      </div>
      <div class="stat">
        <span>Power Blocker</span>
        <span class="stat-value" id="blocker-status">ไม่ใช้งาน</span>
      </div>
      <div class="btn-row">
        <button class="btn-green" id="btn-start-blocker">เปิด Sleep Prevention</button>
        <button class="btn-red" id="btn-stop-blocker">ปิด</button>
      </div>
    </div>
    
    <!-- Idle State -->
    <div class="card">
      <h2>สถานะ Idle</h2>
      <div class="stat">
        <span>สถานะ</span>
        <span class="stat-value" id="idle-state">
          <span class="status-dot status-active"></span>Active
        </span>
      </div>
      <div class="stat">
        <span>เวลา Idle</span>
        <span class="stat-value" id="idle-time">0 วินาที</span>
      </div>
      <div id="idle-bar"><div id="idle-fill" style="width:0%"></div></div>
    </div>
  </div>
  
  <!-- Event Log -->
  <div class="card">
    <h2>Power Events</h2>
    <div id="event-log">
      <p style="color:#666; font-size:13px;">รอ events...</p>
    </div>
    <div class="btn-row">
      <button class="btn-blue" id="btn-refresh">รีเฟรช State</button>
    </div>
  </div>
  
  <!-- Screen Lock Overlay -->
  <div class="screen-lock-overlay" id="lock-overlay">
    <div class="lock-icon">🔒</div>
    <div class="lock-text">หน้าจอถูก Lock</div>
    <div style="color:#666; font-size:14px;">ข้อมูล sensitive ถูกซ่อนไว้</div>
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

### renderer.js

```javascript
// renderer.js
const MAX_IDLE_DISPLAY = 300 // 5 minutes

function updatePowerState(state) {
  // Power source
  const sourceEl = document.getElementById('power-source')
  sourceEl.textContent = state.isOnBattery ? '🔋 แบตเตอรี่' : '🔌 AC Power'
  sourceEl.style.color = state.isOnBattery ? '#ff9f0a' : '#34c759'
  
  // Blocker status
  const blockerEl = document.getElementById('blocker-status')
  if (state.powerBlockerActive) {
    blockerEl.innerHTML = '<span style="color:#34c759">✓ ใช้งานอยู่</span>'
  } else {
    blockerEl.innerHTML = '<span style="color:#ff3b30">✗ ไม่ใช้งาน</span>'
  }
}

function updateIdleState({ idleTime, idleState }) {
  // Idle time display
  document.getElementById('idle-time').textContent = 
    idleTime >= 60 
      ? `${Math.floor(idleTime / 60)} นาที ${idleTime % 60} วินาที`
      : `${idleTime} วินาที`
  
  // Idle state display
  const stateEl = document.getElementById('idle-state')
  const stateColors = {
    active: { color: '#34c759', class: 'status-active', text: 'Active' },
    idle: { color: '#ff9f0a', class: 'status-idle', text: 'Idle' },
    locked: { color: '#ff3b30', class: 'status-locked', text: 'Locked' }
  }
  
  const stateInfo = stateColors[idleState] || { color: '#888', class: '', text: idleState }
  stateEl.innerHTML = `<span class="status-dot ${stateInfo.class}"></span>${stateInfo.text}`
  
  // Idle bar
  const percentage = Math.min((idleTime / MAX_IDLE_DISPLAY) * 100, 100)
  const fillEl = document.getElementById('idle-fill')
  fillEl.style.width = percentage + '%'
  fillEl.style.background = percentage > 75 ? '#ff3b30' : percentage > 40 ? '#ff9f0a' : '#34c759'
}

function addEventToLog(event) {
  const log = document.getElementById('event-log')
  
  // ลบ placeholder
  const placeholder = log.querySelector('p')
  if (placeholder) placeholder.remove()
  
  const time = new Date(event.timestamp).toLocaleTimeString('th-TH')
  const item = document.createElement('div')
  item.className = `event-item event-${event.type.replace(':', '-')}`
  item.innerHTML = `
    <span class="event-time">${time}</span>
    <span class="event-type">${event.type}</span>
    <span>${event.message || ''}</span>
  `
  
  log.insertBefore(item, log.firstChild)
  
  // จำกัดจำนวน items
  while (log.children.length > 30) {
    log.removeChild(log.lastChild)
  }
}

// ─── Power Save Blocker Controls ─────────────────────────────────
document.getElementById('btn-start-blocker').addEventListener('click', async () => {
  const result = await window.powerAPI.startBlocker('prevent-display-sleep')
  if (result.success) {
    addEventToLog({ type: 'blocker-started', timestamp: new Date().toISOString(), message: `ID: ${result.id}` })
  }
  const state = await window.powerAPI.getState()
  updatePowerState(state)
})

document.getElementById('btn-stop-blocker').addEventListener('click', async () => {
  const result = await window.powerAPI.stopBlocker()
  if (result.success) {
    addEventToLog({ type: 'blocker-stopped', timestamp: new Date().toISOString(), message: 'Power blocker stopped' })
  }
  const state = await window.powerAPI.getState()
  updatePowerState(state)
})

document.getElementById('btn-refresh').addEventListener('click', async () => {
  const state = await window.powerAPI.getState()
  updatePowerState(state)
  updateIdleState(state)
})

// ─── Subscribe to Events ─────────────────────────────────────────
const unsubs = [
  window.powerAPI.onEvent(addEventToLog),
  
  window.powerAPI.onStateUpdate(updatePowerState),
  
  window.powerAPI.onIdleUpdate(updateIdleState),
  
  window.powerAPI.onScreenLocked(() => {
    document.getElementById('lock-overlay').classList.add('visible')
    setTimeout(() => {
      document.getElementById('lock-overlay').classList.remove('visible')
    }, 3000)
  })
]

// ─── Load Historical Events ───────────────────────────────────────
async function init() {
  const [state, events] = await Promise.all([
    window.powerAPI.getState(),
    window.powerAPI.getEvents()
  ])
  
  updatePowerState(state)
  updateIdleState({ idleTime: state.idleTime, idleState: state.idleState })
  
  events.reverse().forEach(addEventToLog)
}

window.addEventListener('beforeunload', () => unsubs.forEach(fn => fn()))

init()
```

## สรุป

Power Monitor module ช่วยให้แอปตอบสนองต่อสถานการณ์พลังงานต่างๆ:

- **suspend/resume** - จัดการ sleep mode
- **shutdown** - บันทึกข้อมูลก่อนปิดเครื่อง
- **lock-screen/unlock-screen** - ซ่อน sensitive data เมื่อ lock
- **on-battery/on-ac** - ปรับ performance ตามแหล่งพลังงาน
- **powerSaveBlocker** - ป้องกัน sleep ขณะทำงานสำคัญ
- **getSystemIdleState/Time** - ตรวจสอบความ inactive ของผู้ใช้

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Screen API
