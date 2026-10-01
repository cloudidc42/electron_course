# ตอนที่ 19: Session & Cookies

## Session Module ใน Electron

`session` module จัดการ browser sessions ของ Electron รวมถึง cookies, cache, storage, proxies และ custom request headers โดยแต่ละ BrowserWindow สามารถใช้ session เดียวกันหรือ session ที่แยกออกมาได้

```javascript
const { session } = require('electron')
```

## session.defaultSession

```javascript
// main.js
const { app, session } = require('electron')

app.whenReady().then(() => {
  const ses = session.defaultSession
  
  // ดู user agent
  console.log('User Agent:', ses.getUserAgent())
  
  // ดู storage path
  console.log('Storage Path:', ses.getStoragePath())
  
  // ดู cache path
  console.log('Cache Path:', ses.getCachePath())
  
  // ดู partition
  console.log('Partition:', ses.partition || 'default')
})
```

### Partition Sessions

```javascript
// main.js
const { app, BrowserWindow, session } = require('electron')

app.whenReady().then(() => {
  // Persistent session (บันทึกลงดิสก์)
  const persistSession = session.fromPartition('persist:user-profile')
  
  // In-memory session (ล้างเมื่อปิดแอป)
  const tempSession = session.fromPartition('temp-browsing')
  
  // สร้าง window ที่ใช้ session ที่กำหนด
  const mainWindow = new BrowserWindow({
    webPreferences: {
      preload: __dirname + '/preload.js',
      contextIsolation: true,
      session: persistSession  // ใช้ persistent session
    }
  })
  
  const guestWindow = new BrowserWindow({
    webPreferences: {
      preload: __dirname + '/preload.js',
      contextIsolation: true,
      session: tempSession  // ใช้ temp session
    }
  })
  
  mainWindow.loadURL('https://myapp.com')
  guestWindow.loadURL('https://myapp.com/guest')
})
```

## Cookies CRUD

### อ่าน Cookies

```javascript
// main.js
const { session, ipcMain } = require('electron')

// อ่าน cookies ทั้งหมด
ipcMain.handle('cookies:get-all', async () => {
  const cookies = await session.defaultSession.cookies.get({})
  return cookies
})

// อ่าน cookies ตาม URL
ipcMain.handle('cookies:get-by-url', async (event, url) => {
  const cookies = await session.defaultSession.cookies.get({ url })
  return cookies
})

// อ่าน cookie เดี่ยวตามชื่อ
ipcMain.handle('cookies:get-by-name', async (event, { url, name }) => {
  const cookies = await session.defaultSession.cookies.get({ url, name })
  return cookies[0] || null
})

// อ่าน cookies ตาม domain
ipcMain.handle('cookies:get-by-domain', async (event, domain) => {
  const cookies = await session.defaultSession.cookies.get({ domain })
  return cookies
})

// ตัวอย่าง cookie object ที่ได้:
const exampleCookie = {
  name: 'session_token',
  value: 'abc123xyz',
  domain: '.example.com',
  path: '/',
  secure: true,
  httpOnly: true,
  expirationDate: 1735689600, // Unix timestamp
  session: false,
  sameSite: 'no_restriction' // 'strict', 'lax', 'no_restriction'
}
```

### สร้างและอัปเดต Cookies

```javascript
// main.js
const { session, ipcMain } = require('electron')

// สร้าง cookie ใหม่
ipcMain.handle('cookies:set', async (event, cookieData) => {
  try {
    await session.defaultSession.cookies.set(cookieData)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

// ตัวอย่างการใช้งาน
async function setAuthCookie(token) {
  await session.defaultSession.cookies.set({
    url: 'https://myapp.com',
    name: 'auth_token',
    value: token,
    domain: '.myapp.com',
    path: '/',
    secure: true,
    httpOnly: true,
    expirationDate: Math.floor(Date.now() / 1000) + (30 * 24 * 60 * 60) // 30 days
  })
}

// สร้าง session cookie (ไม่มี expirationDate)
async function setSessionCookie(data) {
  await session.defaultSession.cookies.set({
    url: 'https://myapp.com',
    name: 'session_data',
    value: JSON.stringify(data),
    secure: true,
    httpOnly: true
    // ไม่ใส่ expirationDate = session cookie
  })
}
```

### ลบ Cookies

```javascript
// main.js
const { session, ipcMain } = require('electron')

// ลบ cookie เดี่ยว
ipcMain.handle('cookies:remove', async (event, { url, name }) => {
  await session.defaultSession.cookies.remove(url, name)
  return { success: true }
})

// ลบ cookies ทั้งหมดของ URL
ipcMain.handle('cookies:remove-all-for-url', async (event, url) => {
  const cookies = await session.defaultSession.cookies.get({ url })
  
  const results = await Promise.allSettled(
    cookies.map(cookie => 
      session.defaultSession.cookies.remove(url, cookie.name)
    )
  )
  
  return {
    success: true,
    removed: results.filter(r => r.status === 'fulfilled').length,
    total: cookies.length
  }
})

// ลบ cookies ทั้งหมด
ipcMain.handle('cookies:clear-all', async () => {
  await session.defaultSession.clearStorageData({ storages: ['cookies'] })
  return { success: true }
})
```

### Cookie Change Events

```javascript
// main.js
const { session, BrowserWindow } = require('electron')

app.whenReady().then(() => {
  const win = createWindow()
  
  // ติดตาม cookie changes
  session.defaultSession.cookies.on('changed', (event, cookie, cause, removed) => {
    console.log('Cookie changed:')
    console.log('  Name:', cookie.name)
    console.log('  Value:', cookie.value)
    console.log('  Cause:', cause) // 'explicit', 'overwrite', 'expired', 'evicted', 'expired-overwrite'
    console.log('  Removed:', removed)
    
    win.webContents.send('cookie:changed', {
      cookie: {
        name: cookie.name,
        domain: cookie.domain,
        path: cookie.path
      },
      cause,
      removed
    })
  })
})
```

## session.clearCache

```javascript
// main.js
const { session, ipcMain } = require('electron')

// ล้าง cache ทั้งหมด
ipcMain.handle('session:clear-cache', async () => {
  try {
    await session.defaultSession.clearCache()
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

// ล้าง cache ของ specific session
ipcMain.handle('session:clear-partition-cache', async (event, partition) => {
  try {
    const ses = session.fromPartition(partition)
    await ses.clearCache()
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})
```

## session.clearStorageData

```javascript
// main.js
const { session, ipcMain } = require('electron')

// ล้าง storage data ต่างๆ
ipcMain.handle('session:clear-storage', async (event, options) => {
  try {
    const clearOptions = {}
    
    // origin ที่ต้องการล้าง (optional)
    if (options?.origin) {
      clearOptions.origin = options.origin
    }
    
    // ประเภท storage ที่ต้องการล้าง
    // ถ้าไม่ระบุ จะล้างทั้งหมด
    if (options?.storages) {
      clearOptions.storages = options.storages
    }
    
    // quota types ที่ต้องการล้าง
    if (options?.quotas) {
      clearOptions.quotas = options.quotas
    }
    
    await session.defaultSession.clearStorageData(clearOptions)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

// ตัวอย่างการใช้งาน
async function clearAllData() {
  // ล้างทุกอย่าง
  await session.defaultSession.clearStorageData({
    storages: [
      'cookies',        // cookies
      'filesystem',     // FileSystem API
      'indexdb',        // IndexedDB
      'localstorage',   // localStorage
      'shadercache',    // WebGL shader cache
      'websql',         // Web SQL
      'serviceworkers', // Service Workers
      'cachestorage'    // Cache Storage API
    ]
  })
}

async function clearOnlyLocal() {
  // ล้างเฉพาะ localStorage และ cookies
  await session.defaultSession.clearStorageData({
    storages: ['cookies', 'localstorage']
  })
}

async function clearForOrigin(origin) {
  // ล้างข้อมูลของ origin เดียว
  await session.defaultSession.clearStorageData({
    origin: origin, // 'https://example.com'
    storages: ['cookies', 'indexdb', 'localstorage']
  })
}
```

## Custom Headers

```javascript
// main.js
const { session, app } = require('electron')

app.whenReady().then(() => {
  // เพิ่ม custom headers สำหรับทุก request
  session.defaultSession.webRequest.onBeforeSendHeaders((details, callback) => {
    const headers = { ...details.requestHeaders }
    
    // เพิ่ม auth header
    headers['X-App-Token'] = getAppToken()
    headers['X-App-Version'] = app.getVersion()
    headers['X-Platform'] = process.platform
    
    // ลบ headers ที่ไม่ต้องการ
    delete headers['User-Agent'] // custom user agent
    headers['User-Agent'] = `MyApp/${app.getVersion()} Electron`
    
    callback({ requestHeaders: headers })
  })
  
  // แก้ไข response headers
  session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
    const headers = { ...details.responseHeaders }
    
    // เพิ่ม security headers
    headers['X-Frame-Options'] = ['SAMEORIGIN']
    headers['X-Content-Type-Options'] = ['nosniff']
    headers['Referrer-Policy'] = ['strict-origin-when-cross-origin']
    
    callback({ responseHeaders: headers })
  })
})

function getAppToken() {
  return 'my-app-token-123'
}
```

### Request Filtering

```javascript
// main.js
const { session, app } = require('electron')

app.whenReady().then(() => {
  // Block requests ไปยัง URLs ที่ไม่ได้รับอนุญาต
  session.defaultSession.webRequest.onBeforeRequest(
    { urls: ['https://tracking.example.com/*', 'https://ads.example.com/*'] },
    (details, callback) => {
      console.log('Blocking request:', details.url)
      callback({ cancel: true }) // block request
    }
  )
  
  // Log requests
  session.defaultSession.webRequest.onCompleted((details) => {
    if (details.statusCode >= 400) {
      console.warn(`HTTP ${details.statusCode}: ${details.url}`)
    }
  })
  
  // Monitor errors
  session.defaultSession.webRequest.onErrorOccurred((details) => {
    console.error(`Request error: ${details.error} for ${details.url}`)
  })
})
```

## Proxy Settings

```javascript
// main.js
const { session, app, ipcMain } = require('electron')

app.whenReady().then(() => {
  // ตั้งค่า proxy
  setupProxy()
})

async function setupProxy() {
  // ใช้ system proxy
  await session.defaultSession.setProxy({ mode: 'system' })
  
  // ใช้ direct (ไม่ผ่าน proxy)
  // await session.defaultSession.setProxy({ mode: 'direct' })
  
  // กำหนด proxy URL
  // await session.defaultSession.setProxy({
  //   mode: 'fixed_servers',
  //   proxyRules: 'socks5://myproxy.com:8080',
  //   proxyBypassRules: 'localhost,127.0.0.1'
  // })
  
  // PAC script
  // await session.defaultSession.setProxy({
  //   mode: 'pac_script',
  //   pacScript: 'http://myproxy.com/proxy.pac'
  // })
}

// IPC handlers
ipcMain.handle('proxy:set', async (event, config) => {
  try {
    await session.defaultSession.setProxy(config)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

ipcMain.handle('proxy:get', async () => {
  // ทดสอบ proxy สำหรับ URL เดียว
  const proxyURL = await session.defaultSession.resolveProxy('https://example.com')
  return { proxyURL }
})
```

## Complete Session Manager Application

### main.js

```javascript
// main.js
const { app, BrowserWindow, session, ipcMain } = require('electron')
const path = require('path')

let mainWindow

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1000,
    height: 750,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  mainWindow.loadFile('index.html')
}

// ─── Cookies ────────────────────────────────────────────────────
ipcMain.handle('cookies:get-all', async () => {
  return await session.defaultSession.cookies.get({})
})

ipcMain.handle('cookies:get-by-url', async (event, url) => {
  if (typeof url !== 'string') return []
  return await session.defaultSession.cookies.get({ url })
})

ipcMain.handle('cookies:set', async (event, cookieData) => {
  if (!cookieData || typeof cookieData !== 'object') {
    return { success: false, error: 'Invalid cookie data' }
  }
  
  // Validate required fields
  if (!cookieData.url || !cookieData.name) {
    return { success: false, error: 'url and name are required' }
  }
  
  try {
    await session.defaultSession.cookies.set(cookieData)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

ipcMain.handle('cookies:remove', async (event, url, name) => {
  try {
    await session.defaultSession.cookies.remove(url, name)
    return { success: true }
  } catch (error) {
    return { success: false, error: error.message }
  }
})

// ─── Cache & Storage ─────────────────────────────────────────────
ipcMain.handle('session:clear-cache', async () => {
  try {
    await session.defaultSession.clearCache()
    return { success: true }
  } catch (e) {
    return { success: false, error: e.message }
  }
})

ipcMain.handle('session:clear-storage', async (event, types) => {
  const validTypes = ['cookies', 'filesystem', 'indexdb', 'localstorage', 'shadercache', 'websql', 'serviceworkers', 'cachestorage']
  const storages = Array.isArray(types) 
    ? types.filter(t => validTypes.includes(t))
    : validTypes
  
  try {
    await session.defaultSession.clearStorageData({ storages })
    return { success: true, cleared: storages }
  } catch (e) {
    return { success: false, error: e.message }
  }
})

// ─── User Agent ────────────────────────────────────────────────
ipcMain.handle('session:get-user-agent', () => {
  return session.defaultSession.getUserAgent()
})

ipcMain.handle('session:set-user-agent', (event, ua) => {
  if (typeof ua !== 'string' || ua.length > 500) {
    return { success: false, error: 'Invalid user agent' }
  }
  session.defaultSession.setUserAgent(ua)
  return { success: true }
})

// ─── Cookie Events ────────────────────────────────────────────
session.defaultSession.cookies.on('changed', (event, cookie, cause, removed) => {
  mainWindow?.webContents.send('cookie:changed', {
    cookie: { name: cookie.name, domain: cookie.domain, value: cookie.value.slice(0, 50) },
    cause,
    removed
  })
})

// ─── Proxy ────────────────────────────────────────────────────
ipcMain.handle('proxy:set-system', async () => {
  try {
    await session.defaultSession.setProxy({ mode: 'system' })
    return { success: true }
  } catch (e) {
    return { success: false, error: e.message }
  }
})

ipcMain.handle('proxy:set-direct', async () => {
  try {
    await session.defaultSession.setProxy({ mode: 'direct' })
    return { success: true }
  } catch (e) {
    return { success: false, error: e.message }
  }
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

contextBridge.exposeInMainWorld('sessionAPI', {
  cookies: {
    getAll: () => ipcRenderer.invoke('cookies:get-all'),
    getByURL: (url) => ipcRenderer.invoke('cookies:get-by-url', url),
    set: (data) => ipcRenderer.invoke('cookies:set', data),
    remove: (url, name) => ipcRenderer.invoke('cookies:remove', url, name)
  },
  
  cache: {
    clear: () => ipcRenderer.invoke('session:clear-cache')
  },
  
  storage: {
    clear: (types) => ipcRenderer.invoke('session:clear-storage', types)
  },
  
  userAgent: {
    get: () => ipcRenderer.invoke('session:get-user-agent'),
    set: (ua) => ipcRenderer.invoke('session:set-user-agent', ua)
  },
  
  proxy: {
    setSystem: () => ipcRenderer.invoke('proxy:set-system'),
    setDirect: () => ipcRenderer.invoke('proxy:set-direct')
  },
  
  onCookieChanged: (callback) => {
    const fn = (_e, data) => callback(data)
    ipcRenderer.on('cookie:changed', fn)
    return () => ipcRenderer.removeListener('cookie:changed', fn)
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
  <title>Session Manager</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui; background: #f8fafc; color: #1e293b; padding: 20px; }
    h1 { color: #0f172a; margin-bottom: 20px; }
    .tabs { display: flex; gap: 0; margin-bottom: 0; border-bottom: 2px solid #e2e8f0; }
    .tab { padding: 10px 20px; border: none; background: none; cursor: pointer; font-size: 14px; color: #64748b; border-bottom: 2px solid transparent; margin-bottom: -2px; }
    .tab.active { color: #3b82f6; border-bottom-color: #3b82f6; }
    .panel { display: none; padding: 20px 0; }
    .panel.active { display: block; }
    .card { background: white; border-radius: 8px; padding: 16px; margin-bottom: 14px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
    h2 { font-size: 14px; color: #475569; margin-bottom: 12px; }
    table { width: 100%; border-collapse: collapse; font-size: 13px; }
    th { background: #f1f5f9; padding: 8px; text-align: left; border-bottom: 1px solid #e2e8f0; }
    td { padding: 8px; border-bottom: 1px solid #f1f5f9; vertical-align: top; }
    td code { background: #f1f5f9; padding: 1px 4px; border-radius: 3px; font-size: 12px; }
    .form-row { display: flex; gap: 8px; flex-wrap: wrap; margin-bottom: 10px; }
    input, select { padding: 8px 10px; border: 1px solid #cbd5e1; border-radius: 6px; font-size: 14px; }
    input { flex: 1; min-width: 120px; }
    button { padding: 8px 16px; border: none; border-radius: 6px; cursor: pointer; font-size: 14px; white-space: nowrap; }
    .btn-blue { background: #3b82f6; color: white; }
    .btn-red { background: #ef4444; color: white; }
    .btn-gray { background: #e2e8f0; color: #334155; }
    button:hover { opacity: 0.85; }
    .badge { padding: 2px 6px; border-radius: 10px; font-size: 11px; }
    .badge-secure { background: #dcfce7; color: #16a34a; }
    .badge-http { background: #fef3c7; color: #d97706; }
    #cookie-events { height: 120px; overflow-y: auto; background: #f8fafc; border: 1px solid #e2e8f0; border-radius: 6px; padding: 8px; font-family: monospace; font-size: 12px; }
    #ua-display { background: #f1f5f9; padding: 10px; border-radius: 6px; font-size: 13px; word-break: break-all; margin-bottom: 10px; }
    .storage-options { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 12px; }
    .storage-option { display: flex; align-items: center; gap: 6px; font-size: 14px; cursor: pointer; }
    .storage-option input[type=checkbox] { width: 16px; height: 16px; }
  </style>
</head>
<body>
  <h1>Session Manager</h1>
  
  <div class="tabs">
    <button class="tab active" data-panel="cookies">Cookies</button>
    <button class="tab" data-panel="storage">Storage</button>
    <button class="tab" data-panel="ua">User Agent</button>
    <button class="tab" data-panel="proxy">Proxy</button>
  </div>
  
  <!-- Cookies Panel -->
  <div class="panel active" id="panel-cookies">
    <div class="card">
      <h2>เพิ่ม/แก้ไข Cookie</h2>
      <div class="form-row">
        <input type="text" id="cookie-url" placeholder="URL (เช่น https://example.com)" value="https://example.com">
        <input type="text" id="cookie-name" placeholder="ชื่อ cookie" value="test_cookie">
        <input type="text" id="cookie-value" placeholder="ค่า" value="hello123">
      </div>
      <div class="form-row">
        <input type="text" id="cookie-domain" placeholder="Domain (เช่น .example.com)">
        <input type="text" id="cookie-path" placeholder="Path (เช่น /)" value="/">
        <button class="btn-blue" id="btn-set-cookie">บันทึก Cookie</button>
      </div>
    </div>
    
    <div class="card">
      <h2>รายการ Cookies</h2>
      <div class="form-row">
        <input type="text" id="filter-url" placeholder="Filter by URL">
        <button class="btn-gray" id="btn-load-cookies">โหลด Cookies</button>
        <button class="btn-red" id="btn-clear-cookies">ลบทั้งหมด</button>
      </div>
      <div style="overflow-x:auto;">
        <table id="cookies-table">
          <thead><tr><th>ชื่อ</th><th>ค่า</th><th>Domain</th><th>Path</th><th>Secure</th><th>การจัดการ</th></tr></thead>
          <tbody id="cookies-body"></tbody>
        </table>
      </div>
    </div>
    
    <div class="card">
      <h2>Cookie Events</h2>
      <div id="cookie-events">รอ cookie changes...</div>
    </div>
  </div>
  
  <!-- Storage Panel -->
  <div class="panel" id="panel-storage">
    <div class="card">
      <h2>ล้าง Storage Data</h2>
      <div class="storage-options">
        <label class="storage-option"><input type="checkbox" value="cookies" checked> Cookies</label>
        <label class="storage-option"><input type="checkbox" value="localstorage" checked> LocalStorage</label>
        <label class="storage-option"><input type="checkbox" value="indexdb" checked> IndexedDB</label>
        <label class="storage-option"><input type="checkbox" value="cachestorage"> Cache Storage</label>
        <label class="storage-option"><input type="checkbox" value="serviceworkers"> Service Workers</label>
        <label class="storage-option"><input type="checkbox" value="filesystem"> FileSystem</label>
      </div>
      <div class="form-row">
        <button class="btn-red" id="btn-clear-storage">ล้าง Storage ที่เลือก</button>
        <button class="btn-blue" id="btn-clear-cache">ล้าง HTTP Cache</button>
      </div>
      <div id="storage-result" style="margin-top:10px; color:#64748b; font-size:14px;"></div>
    </div>
  </div>
  
  <!-- User Agent Panel -->
  <div class="panel" id="panel-ua">
    <div class="card">
      <h2>User Agent ปัจจุบัน</h2>
      <div id="ua-display">กำลังโหลด...</div>
      <div class="form-row">
        <input type="text" id="new-ua" placeholder="User Agent ใหม่" style="flex:3;">
        <button class="btn-blue" id="btn-set-ua">ตั้งค่า</button>
        <button class="btn-gray" id="btn-reset-ua">รีเซต</button>
      </div>
      <div style="margin-top:10px;">
        <strong style="font-size:13px; color:#64748b;">Presets:</strong>
        <div class="form-row" style="margin-top:6px;">
          <button class="btn-gray btn-small" onclick="setPresetUA('chrome-win')">Chrome/Win</button>
          <button class="btn-gray btn-small" onclick="setPresetUA('chrome-mac')">Chrome/Mac</button>
          <button class="btn-gray btn-small" onclick="setPresetUA('mobile')">Mobile</button>
          <button class="btn-gray btn-small" onclick="setPresetUA('bot')">Bot</button>
        </div>
      </div>
    </div>
  </div>
  
  <!-- Proxy Panel -->
  <div class="panel" id="panel-proxy">
    <div class="card">
      <h2>Proxy Settings</h2>
      <div class="form-row">
        <button class="btn-blue" id="btn-proxy-system">ใช้ System Proxy</button>
        <button class="btn-gray" id="btn-proxy-direct">Direct (ไม่ใช้ Proxy)</button>
      </div>
      <div id="proxy-result" style="margin-top:10px; color:#64748b; font-size:14px;"></div>
    </div>
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

### renderer.js

```javascript
// renderer.js
'use strict'

// Tab switching
document.querySelectorAll('.tab').forEach(tab => {
  tab.addEventListener('click', () => {
    document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'))
    document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'))
    tab.classList.add('active')
    document.getElementById(`panel-${tab.dataset.panel}`).classList.add('active')
  })
})

// ─── Cookies ──────────────────────────────────────────────────────
async function loadCookies() {
  const url = document.getElementById('filter-url').value.trim()
  const cookies = url 
    ? await window.sessionAPI.cookies.getByURL(url)
    : await window.sessionAPI.cookies.getAll()
  
  const tbody = document.getElementById('cookies-body')
  tbody.innerHTML = cookies.length ? cookies.map(c => `
    <tr>
      <td><code>${c.name}</code></td>
      <td style="max-width:150px; overflow:hidden; text-overflow:ellipsis; white-space:nowrap;" title="${c.value}">${c.value.slice(0, 30)}${c.value.length > 30 ? '...' : ''}</td>
      <td>${c.domain || '-'}</td>
      <td>${c.path || '/'}</td>
      <td><span class="badge ${c.secure ? 'badge-secure' : 'badge-http'}">${c.secure ? 'HTTPS' : 'HTTP'}</span></td>
      <td><button class="btn-red" style="padding:3px 8px; font-size:12px;" onclick="removeCookie('${c.domain}${c.path}', '${c.name}')">ลบ</button></td>
    </tr>
  `).join('') : '<tr><td colspan="6" style="text-align:center; color:#94a3b8; padding:20px;">ไม่พบ cookies</td></tr>'
}

window.removeCookie = async (url, name) => {
  const fullURL = url.startsWith('http') ? url : `https://${url}`
  await window.sessionAPI.cookies.remove(fullURL, name)
  await loadCookies()
  addCookieEvent(`ลบ cookie: ${name} (${url})`)
}

document.getElementById('btn-load-cookies').addEventListener('click', loadCookies)

document.getElementById('btn-set-cookie').addEventListener('click', async () => {
  const data = {
    url: document.getElementById('cookie-url').value.trim(),
    name: document.getElementById('cookie-name').value.trim(),
    value: document.getElementById('cookie-value').value.trim(),
    domain: document.getElementById('cookie-domain').value.trim() || undefined,
    path: document.getElementById('cookie-path').value.trim() || '/'
  }
  
  if (!data.url || !data.name) { alert('URL และชื่อ cookie จำเป็นต้องกรอก'); return }
  
  const result = await window.sessionAPI.cookies.set(data)
  if (result.success) {
    addCookieEvent(`สร้าง cookie: ${data.name}`)
    await loadCookies()
  } else {
    alert('ไม่สามารถสร้าง cookie: ' + result.error)
  }
})

document.getElementById('btn-clear-cookies').addEventListener('click', async () => {
  if (!confirm('ต้องการลบ cookies ทั้งหมดหรือไม่?')) return
  await window.sessionAPI.storage.clear(['cookies'])
  await loadCookies()
  addCookieEvent('ล้าง cookies ทั้งหมด')
})

function addCookieEvent(message) {
  const el = document.getElementById('cookie-events')
  const time = new Date().toLocaleTimeString('th-TH')
  el.textContent = `[${time}] ${message}\n` + el.textContent
}

const unsubCookies = window.sessionAPI.onCookieChanged((data) => {
  const action = data.removed ? 'ลบ' : 'เพิ่ม/แก้ไข'
  addCookieEvent(`${action} cookie: ${data.cookie.name} (${data.cookie.domain}) - cause: ${data.cause}`)
})

// ─── Storage ──────────────────────────────────────────────────────
document.getElementById('btn-clear-storage').addEventListener('click', async () => {
  const checkboxes = document.querySelectorAll('.storage-options input[type=checkbox]:checked')
  const types = Array.from(checkboxes).map(cb => cb.value)
  
  if (!types.length) { alert('เลือก storage type อย่างน้อย 1 รายการ'); return }
  if (!confirm(`ต้องการล้าง ${types.join(', ')} หรือไม่?`)) return
  
  const result = await window.sessionAPI.storage.clear(types)
  document.getElementById('storage-result').textContent = 
    result.success ? `✅ ล้างสำเร็จ: ${result.cleared.join(', ')}` : `❌ Error: ${result.error}`
})

document.getElementById('btn-clear-cache').addEventListener('click', async () => {
  const result = await window.sessionAPI.cache.clear()
  document.getElementById('storage-result').textContent = 
    result.success ? '✅ ล้าง HTTP Cache สำเร็จ' : `❌ Error: ${result.error}`
})

// ─── User Agent ──────────────────────────────────────────────────
const UA_PRESETS = {
  'chrome-win': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
  'chrome-mac': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
  'mobile': 'Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1',
  'bot': 'MyBot/1.0 (+https://myapp.com/bot)'
}

window.setPresetUA = async (preset) => {
  const ua = UA_PRESETS[preset]
  if (!ua) return
  document.getElementById('new-ua').value = ua
}

async function loadUA() {
  const ua = await window.sessionAPI.userAgent.get()
  document.getElementById('ua-display').textContent = ua
}

document.getElementById('btn-set-ua').addEventListener('click', async () => {
  const ua = document.getElementById('new-ua').value.trim()
  if (!ua) return
  const result = await window.sessionAPI.userAgent.set(ua)
  if (result.success) await loadUA()
  else alert('Error: ' + result.error)
})

document.getElementById('btn-reset-ua').addEventListener('click', async () => {
  const result = await window.sessionAPI.userAgent.set('')
  if (result.success) await loadUA()
})

// ─── Proxy ────────────────────────────────────────────────────────
document.getElementById('btn-proxy-system').addEventListener('click', async () => {
  const result = await window.sessionAPI.proxy.setSystem()
  document.getElementById('proxy-result').textContent = 
    result.success ? '✅ ใช้ System Proxy แล้ว' : `❌ Error: ${result.error}`
})

document.getElementById('btn-proxy-direct').addEventListener('click', async () => {
  const result = await window.sessionAPI.proxy.setDirect()
  document.getElementById('proxy-result').textContent = 
    result.success ? '✅ ใช้ Direct Connection แล้ว' : `❌ Error: ${result.error}`
})

// Cleanup
window.addEventListener('beforeunload', () => unsubCookies())

// Init
loadCookies()
loadUA()
```

## สรุป

Session module เป็นเครื่องมือสำคัญสำหรับจัดการ browser state:

- **Cookies CRUD** - สร้าง อ่าน แก้ไข และลบ cookies
- **clearCache** - ล้าง HTTP cache
- **clearStorageData** - ล้าง localStorage, IndexedDB, cookies, และ Service Workers
- **Custom Headers** - เพิ่ม headers สำหรับทุก request
- **Proxy Settings** - กำหนด proxy ผ่าน system, direct, หรือ custom
- **Partition Sessions** - แยก session สำหรับ user profiles หรือ guest mode
- **Cookie Events** - ติดตามการเปลี่ยนแปลง cookies แบบ real-time

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Net & HTTP Requests
