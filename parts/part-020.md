# ตอนที่ 20: Net & HTTP Requests

## Net Module ใน Electron

`net` module ให้ความสามารถในการสร้าง HTTP/HTTPS requests จาก main process โดยใช้ Chromium's native networking stack (แทนที่จะใช้ Node.js `http` module) ทำให้รองรับ proxy settings, certificate management, และ session cookies อัตโนมัติ

```javascript
const { net } = require('electron')
```

### ข้อดีของ net module เทียบกับ Node.js http

| Feature | electron.net | Node.js http |
|---------|-------------|--------------|
| Proxy support | ✅ Auto (system proxy) | ❌ Manual |
| Session cookies | ✅ Auto | ❌ Manual |
| Certificate handling | ✅ Chromium | ❌ Node.js |
| Custom protocols | ✅ Yes | ❌ No |
| HTTP/2 | ✅ Yes | ✅ Yes |
| NTLM/Kerberos auth | ✅ Yes | ❌ No |

## net.request

### รูปแบบพื้นฐาน

```javascript
// main.js
const { net } = require('electron')

function makeRequest(options) {
  return new Promise((resolve, reject) => {
    const request = net.request(options)
    
    let responseData = ''
    
    request.on('response', (response) => {
      console.log('Status:', response.statusCode)
      console.log('Headers:', response.headers)
      
      response.on('data', (chunk) => {
        responseData += chunk.toString()
      })
      
      response.on('end', () => {
        resolve({
          status: response.statusCode,
          headers: response.headers,
          data: responseData
        })
      })
      
      response.on('error', reject)
    })
    
    request.on('error', reject)
    request.end()
  })
}

// GET request
async function getRequest() {
  const result = await makeRequest({
    method: 'GET',
    url: 'https://api.github.com/users/electron',
    headers: {
      'Accept': 'application/vnd.github.v3+json',
      'User-Agent': 'MyElectronApp/1.0'
    }
  })
  
  console.log('Response:', result.status)
  const data = JSON.parse(result.data)
  console.log('Data:', data)
}

// POST request
async function postRequest() {
  return new Promise((resolve, reject) => {
    const request = net.request({
      method: 'POST',
      url: 'https://httpbin.org/post',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json'
      }
    })
    
    let responseData = ''
    
    request.on('response', (response) => {
      response.on('data', (chunk) => { responseData += chunk })
      response.on('end', () => {
        resolve({ status: response.statusCode, data: JSON.parse(responseData) })
      })
    })
    
    request.on('error', reject)
    
    // ส่ง request body
    const body = JSON.stringify({ name: 'test', value: 42 })
    request.write(body)
    request.end()
  })
}
```

## net.fetch

`net.fetch` เป็น API ที่ใหม่กว่าและใช้งานง่ายกว่า ทำงานเหมือน browser `fetch()` API แต่ใช้ Electron's networking stack

```javascript
// main.js (Electron 21+)
const { net } = require('electron')

// GET request
async function fetchGet(url) {
  const response = await net.fetch(url)
  const data = await response.json()
  return data
}

// POST request
async function fetchPost(url, body) {
  const response = await net.fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(body)
  })
  
  return await response.json()
}

// ตัวอย่างการใช้งาน
async function example() {
  // GET
  const user = await fetchGet('https://api.github.com/users/octocat')
  console.log('User:', user.name)
  
  // POST
  const result = await fetchPost('https://httpbin.org/post', {
    message: 'Hello from Electron!'
  })
  console.log('Posted:', result.json)
}
```

## Making API Calls from Main Process

### สร้าง HTTP Client สำหรับ Main Process

```javascript
// http-client.js
const { net, app } = require('electron')

class HttpClient {
  constructor(baseURL, defaultOptions = {}) {
    this.baseURL = baseURL
    this.defaultOptions = {
      headers: {
        'User-Agent': `${app.getName()}/${app.getVersion()}`,
        'Accept': 'application/json',
        ...defaultOptions.headers
      },
      timeout: defaultOptions.timeout || 30000,
      ...defaultOptions
    }
    this.interceptors = {
      request: [],
      response: []
    }
  }
  
  // เพิ่ม request interceptor
  addRequestInterceptor(fn) {
    this.interceptors.request.push(fn)
    return () => {
      this.interceptors.request = this.interceptors.request.filter(f => f !== fn)
    }
  }
  
  // เพิ่ม response interceptor
  addResponseInterceptor(fn) {
    this.interceptors.response.push(fn)
    return () => {
      this.interceptors.response = this.interceptors.response.filter(f => f !== fn)
    }
  }
  
  async request(path, options = {}) {
    const url = path.startsWith('http') ? path : `${this.baseURL}${path}`
    
    let requestOptions = {
      ...this.defaultOptions,
      ...options,
      headers: {
        ...this.defaultOptions.headers,
        ...options.headers
      }
    }
    
    // รัน request interceptors
    for (const interceptor of this.interceptors.request) {
      requestOptions = await interceptor(requestOptions, url) || requestOptions
    }
    
    try {
      const response = await net.fetch(url, {
        method: requestOptions.method || 'GET',
        headers: requestOptions.headers,
        body: requestOptions.body,
        signal: requestOptions.signal
      })
      
      let result = {
        status: response.status,
        statusText: response.statusText,
        headers: Object.fromEntries(response.headers.entries()),
        url: response.url,
        ok: response.ok
      }
      
      // Parse response body
      const contentType = response.headers.get('Content-Type') || ''
      if (contentType.includes('application/json')) {
        result.data = await response.json()
      } else if (contentType.includes('text/')) {
        result.data = await response.text()
      } else {
        result.data = await response.arrayBuffer()
      }
      
      // ตรวจสอบ HTTP errors
      if (!response.ok) {
        throw new HttpError(response.status, response.statusText, result.data)
      }
      
      // รัน response interceptors
      for (const interceptor of this.interceptors.response) {
        result = await interceptor(result) || result
      }
      
      return result
    } catch (error) {
      if (error instanceof HttpError) throw error
      throw new NetworkError(error.message)
    }
  }
  
  async get(path, options = {}) {
    return this.request(path, { ...options, method: 'GET' })
  }
  
  async post(path, body, options = {}) {
    return this.request(path, {
      ...options,
      method: 'POST',
      body: typeof body === 'string' ? body : JSON.stringify(body),
      headers: {
        'Content-Type': 'application/json',
        ...options.headers
      }
    })
  }
  
  async put(path, body, options = {}) {
    return this.request(path, {
      ...options,
      method: 'PUT',
      body: typeof body === 'string' ? body : JSON.stringify(body),
      headers: {
        'Content-Type': 'application/json',
        ...options.headers
      }
    })
  }
  
  async delete(path, options = {}) {
    return this.request(path, { ...options, method: 'DELETE' })
  }
}

class HttpError extends Error {
  constructor(status, statusText, data) {
    super(`HTTP ${status}: ${statusText}`)
    this.name = 'HttpError'
    this.status = status
    this.statusText = statusText
    this.data = data
  }
}

class NetworkError extends Error {
  constructor(message) {
    super(message)
    this.name = 'NetworkError'
  }
}

module.exports = { HttpClient, HttpError, NetworkError }
```

### ใช้งาน HttpClient

```javascript
// main.js
const { app, ipcMain } = require('electron')
const { HttpClient } = require('./http-client')

// สร้าง API client
const apiClient = new HttpClient('https://api.myapp.com', {
  headers: {
    'Authorization': `Bearer ${getStoredToken()}`
  }
})

// เพิ่ม auth interceptor
apiClient.addRequestInterceptor(async (options) => {
  const token = getStoredToken()
  if (token) {
    options.headers['Authorization'] = `Bearer ${token}`
  }
  return options
})

// เพิ่ม logging
apiClient.addResponseInterceptor(async (result) => {
  console.log(`[API] ${result.status} ${result.url}`)
  return result
})

// IPC handlers
ipcMain.handle('api:get-users', async () => {
  try {
    const result = await apiClient.get('/users')
    return { success: true, data: result.data }
  } catch (error) {
    return { success: false, error: error.message, status: error.status }
  }
})

ipcMain.handle('api:create-user', async (event, userData) => {
  if (!userData || typeof userData !== 'object') {
    return { success: false, error: 'Invalid user data' }
  }
  
  try {
    const result = await apiClient.post('/users', userData)
    return { success: true, data: result.data }
  } catch (error) {
    return { success: false, error: error.message, status: error.status }
  }
})

function getStoredToken() {
  // โหลด token จาก storage
  return process.env.API_TOKEN || ''
}

app.whenReady().then(() => {
  createWindow()
})
```

## Streaming Responses

```javascript
// main.js - Streaming สำหรับ large responses
const { net, ipcMain, BrowserWindow } = require('electron')

ipcMain.handle('net:stream-download', async (event, url) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  return new Promise((resolve, reject) => {
    const request = net.request({ method: 'GET', url })
    let totalBytes = 0
    let receivedBytes = 0
    const chunks = []
    
    request.on('response', (response) => {
      totalBytes = parseInt(response.headers['content-length'] || '0', 10)
      
      response.on('data', (chunk) => {
        chunks.push(chunk)
        receivedBytes += chunk.length
        
        // ส่ง progress update
        const progress = totalBytes > 0 
          ? Math.round((receivedBytes / totalBytes) * 100)
          : -1
        
        win?.webContents.send('download:progress', {
          url,
          received: receivedBytes,
          total: totalBytes,
          progress
        })
      })
      
      response.on('end', () => {
        const data = Buffer.concat(chunks)
        win?.webContents.send('download:complete', { url, size: data.length })
        resolve({ success: true, size: data.length, data: data.toString('base64') })
      })
      
      response.on('error', (error) => {
        reject(error)
      })
    })
    
    request.on('error', reject)
    request.end()
  })
})

// Streaming JSON API (Server-Sent Events)
ipcMain.handle('net:sse-connect', async (event, url) => {
  const win = BrowserWindow.fromWebContents(event.sender)
  
  return new Promise((resolve, reject) => {
    const request = net.request({
      method: 'GET',
      url,
      headers: {
        'Accept': 'text/event-stream',
        'Cache-Control': 'no-cache'
      }
    })
    
    let buffer = ''
    
    request.on('response', (response) => {
      if (response.statusCode !== 200) {
        reject(new Error(`SSE failed: ${response.statusCode}`))
        return
      }
      
      resolve({ success: true, connected: true })
      
      response.on('data', (chunk) => {
        buffer += chunk.toString()
        
        // parse SSE events
        const events = buffer.split('\n\n')
        buffer = events.pop() || ''
        
        events.forEach(eventBlock => {
          const lines = eventBlock.split('\n')
          let eventType = 'message'
          let data = ''
          
          lines.forEach(line => {
            if (line.startsWith('event:')) {
              eventType = line.slice(6).trim()
            } else if (line.startsWith('data:')) {
              data += line.slice(5).trim()
            }
          })
          
          if (data) {
            try {
              const parsedData = JSON.parse(data)
              win?.webContents.send('sse:event', { type: eventType, data: parsedData })
            } catch {
              win?.webContents.send('sse:event', { type: eventType, data })
            }
          }
        })
      })
      
      response.on('end', () => {
        win?.webContents.send('sse:closed', { url })
      })
    })
    
    request.on('error', (error) => {
      reject(error)
    })
    
    request.end()
  })
})
```

## Authentication

### Basic Authentication

```javascript
// main.js
const { net } = require('electron')

async function fetchWithBasicAuth(url, username, password) {
  const credentials = Buffer.from(`${username}:${password}`).toString('base64')
  
  const response = await net.fetch(url, {
    headers: {
      'Authorization': `Basic ${credentials}`
    }
  })
  
  return response.json()
}

// Bearer Token
async function fetchWithBearerToken(url, token) {
  const response = await net.fetch(url, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  })
  
  return response.json()
}

// API Key
async function fetchWithApiKey(url, apiKey) {
  const response = await net.fetch(url, {
    headers: {
      'X-API-Key': apiKey
    }
  })
  
  return response.json()
}
```

### Token Refresh

```javascript
// token-manager.js
const { net } = require('electron')

class TokenManager {
  constructor(refreshURL) {
    this.refreshURL = refreshURL
    this.accessToken = null
    this.refreshToken = null
    this.tokenExpiry = null
    this.refreshPromise = null
  }
  
  setTokens(accessToken, refreshToken, expiresIn) {
    this.accessToken = accessToken
    this.refreshToken = refreshToken
    this.tokenExpiry = Date.now() + (expiresIn * 1000) - 60000 // 1 min buffer
  }
  
  isTokenExpired() {
    return !this.tokenExpiry || Date.now() >= this.tokenExpiry
  }
  
  async getValidToken() {
    if (!this.isTokenExpired()) {
      return this.accessToken
    }
    
    // ป้องกัน race condition: refresh เพียงครั้งเดียว
    if (!this.refreshPromise) {
      this.refreshPromise = this.refreshAccessToken()
        .finally(() => { this.refreshPromise = null })
    }
    
    await this.refreshPromise
    return this.accessToken
  }
  
  async refreshAccessToken() {
    if (!this.refreshToken) {
      throw new Error('No refresh token available')
    }
    
    const response = await net.fetch(this.refreshURL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refresh_token: this.refreshToken })
    })
    
    if (!response.ok) {
      throw new Error(`Token refresh failed: ${response.status}`)
    }
    
    const data = await response.json()
    this.setTokens(data.access_token, data.refresh_token || this.refreshToken, data.expires_in)
    
    return this.accessToken
  }
}

module.exports = TokenManager
```

## Custom Headers

```javascript
// main.js
const { net, app, ipcMain } = require('electron')

// สร้าง request พร้อม custom headers
async function fetchWithCustomHeaders(url, customHeaders = {}) {
  const response = await net.fetch(url, {
    headers: {
      // Default headers
      'User-Agent': `${app.getName()}/${app.getVersion()} (Electron)`,
      'Accept': 'application/json',
      'Accept-Language': 'th,en;q=0.9',
      'X-App-Platform': process.platform,
      'X-App-Version': app.getVersion(),
      'X-Request-ID': generateRequestId(),
      'X-Timestamp': new Date().toISOString(),
      
      // Override หรือเพิ่ม headers จากผู้เรียก
      ...customHeaders
    }
  })
  
  return response
}

function generateRequestId() {
  return Math.random().toString(36).substring(2) + Date.now().toString(36)
}

// IPC handler
ipcMain.handle('net:fetch', async (event, url, options = {}) => {
  if (typeof url !== 'string') return { success: false, error: 'Invalid URL' }
  
  try {
    // ตรวจสอบ URL
    const parsed = new URL(url)
    if (!['https:', 'http:'].includes(parsed.protocol)) {
      return { success: false, error: 'Only http/https allowed' }
    }
    
    const response = await fetchWithCustomHeaders(url, options.headers)
    
    const contentType = response.headers.get('Content-Type') || ''
    let data
    
    if (contentType.includes('application/json')) {
      data = await response.json()
    } else {
      data = await response.text()
    }
    
    return {
      success: response.ok,
      status: response.status,
      statusText: response.statusText,
      data
    }
  } catch (error) {
    return { success: false, error: error.message }
  }
})
```

## Certificates

```javascript
// main.js
const { app, session } = require('electron')
const path = require('path')
const fs = require('fs')

app.whenReady().then(() => {
  // ตรวจสอบ certificate
  app.on('certificate-error', (event, webContents, url, error, certificate, callback) => {
    // ในการ development: ยอมรับ self-signed certs
    if (process.env.NODE_ENV === 'development' && 
        (url.startsWith('https://localhost') || url.startsWith('https://127.0.0.1'))) {
      event.preventDefault()
      callback(true) // ยอมรับ certificate
    } else {
      callback(false) // ปฏิเสธ certificate (default behavior)
    }
  })
  
  // ใช้ client certificate
  app.on('select-client-certificate', (event, webContents, url, list, callback) => {
    event.preventDefault()
    
    if (list.length > 0) {
      // เลือก certificate แรก หรือแสดง dialog ให้เลือก
      callback(list[0])
    } else {
      callback() // ไม่มี certificate
    }
  })
})

// เพิ่ม certificate authority custom
function addCustomCA(certPath) {
  const cert = fs.readFileSync(certPath)
  
  // ตั้งค่าใน session
  session.defaultSession.setCertificateVerifyProc((request, callback) => {
    // ตรวจสอบ certificate manually
    const { hostname } = new URL(request.hostname.startsWith('http') 
      ? request.hostname 
      : `https://${request.hostname}`)
    
    // ยอมรับ certificates จาก internal domain
    if (hostname.endsWith('.internal.company.com')) {
      callback(0) // ยอมรับ
    } else {
      callback(-3) // ใช้ default verification
    }
  })
}
```

## Complete Net API Application

### main.js

```javascript
// main.js
const { app, BrowserWindow, ipcMain, net } = require('electron')
const path = require('path')

let mainWindow
const requestHistory = []

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1100,
    height: 800,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  
  mainWindow.loadFile('index.html')
}

async function executeRequest(config) {
  const { url, method = 'GET', headers = {}, body, timeout = 30000 } = config
  
  // Validate URL
  try { new URL(url) } catch (e) {
    throw new Error('Invalid URL: ' + url)
  }
  
  const controller = new AbortController()
  const timeoutTimer = setTimeout(() => controller.abort(), timeout)
  
  const startTime = Date.now()
  
  try {
    const fetchOptions = {
      method,
      headers: {
        'User-Agent': `${app.getName()}/${app.getVersion()}`,
        'Accept': '*/*',
        ...headers
      },
      signal: controller.signal
    }
    
    if (body && ['POST', 'PUT', 'PATCH'].includes(method)) {
      fetchOptions.body = typeof body === 'string' ? body : JSON.stringify(body)
      if (!headers['Content-Type']) {
        fetchOptions.headers['Content-Type'] = 'application/json'
      }
    }
    
    const response = await net.fetch(url, fetchOptions)
    const elapsed = Date.now() - startTime
    const responseHeaders = Object.fromEntries(response.headers.entries())
    const contentType = response.headers.get('Content-Type') || ''
    
    let responseData
    if (contentType.includes('application/json')) {
      responseData = await response.json()
    } else {
      responseData = await response.text()
    }
    
    const result = {
      id: Date.now(),
      url,
      method,
      status: response.status,
      statusText: response.statusText,
      headers: responseHeaders,
      data: responseData,
      elapsed,
      timestamp: new Date().toISOString(),
      success: response.ok
    }
    
    requestHistory.unshift(result)
    if (requestHistory.length > 50) requestHistory.pop()
    
    return result
  } finally {
    clearTimeout(timeoutTimer)
  }
}

ipcMain.handle('net:request', async (event, config) => {
  if (!config || typeof config !== 'object') {
    return { success: false, error: 'Invalid config' }
  }
  if (typeof config.url !== 'string') {
    return { success: false, error: 'url is required' }
  }
  
  try {
    const result = await executeRequest(config)
    mainWindow?.webContents.send('net:request-completed', result)
    return { success: true, result }
  } catch (error) {
    const errResult = {
      id: Date.now(),
      url: config.url,
      method: config.method || 'GET',
      error: error.message,
      timestamp: new Date().toISOString(),
      success: false
    }
    requestHistory.unshift(errResult)
    return { success: false, error: error.message }
  }
})

ipcMain.handle('net:get-history', () => requestHistory.slice(0, 20))
ipcMain.handle('net:clear-history', () => { requestHistory.length = 0; return { success: true } })

app.whenReady().then(createWindow)
app.on('window-all-closed', () => { if (process.platform !== 'darwin') app.quit() })
```

### preload.js

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('netAPI', {
  request: (config) => ipcRenderer.invoke('net:request', config),
  getHistory: () => ipcRenderer.invoke('net:get-history'),
  clearHistory: () => ipcRenderer.invoke('net:clear-history'),
  
  onRequestComplete: (callback) => {
    const fn = (_e, data) => callback(data)
    ipcRenderer.on('net:request-completed', fn)
    return () => ipcRenderer.removeListener('net:request-completed', fn)
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
  <title>HTTP Client</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui; background: #0d1117; color: #c9d1d9; min-height: 100vh; }
    .layout { display: flex; height: 100vh; }
    .sidebar { width: 280px; background: #161b22; border-right: 1px solid #30363d; display: flex; flex-direction: column; }
    .main { flex: 1; display: flex; flex-direction: column; overflow: hidden; }
    .sidebar-header { padding: 14px 16px; border-bottom: 1px solid #30363d; }
    .sidebar-header h2 { font-size: 14px; color: #8b949e; text-transform: uppercase; letter-spacing: 0.5px; }
    .history-list { flex: 1; overflow-y: auto; }
    .history-item { padding: 10px 16px; border-bottom: 1px solid #21262d; cursor: pointer; transition: background 0.1s; }
    .history-item:hover { background: #1c2128; }
    .history-method { font-size: 11px; font-weight: 700; padding: 2px 5px; border-radius: 3px; margin-right: 6px; }
    .method-GET { background: #1f6feb; color: #cae8ff; }
    .method-POST { background: #2ea043; color: #aff5b4; }
    .method-PUT { background: #9e6a03; color: #f8e3a1; }
    .method-DELETE { background: #da3633; color: #ffd7d5; }
    .history-url { font-size: 12px; color: #8b949e; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
    .history-status { font-size: 11px; float: right; }
    .status-ok { color: #2ea043; }
    .status-err { color: #da3633; }
    .request-panel { padding: 16px; border-bottom: 1px solid #30363d; }
    .url-row { display: flex; gap: 8px; margin-bottom: 10px; }
    select { background: #21262d; border: 1px solid #30363d; color: #c9d1d9; padding: 8px 10px; border-radius: 6px; font-size: 14px; }
    input[type=text] { flex: 1; background: #21262d; border: 1px solid #30363d; color: #c9d1d9; padding: 8px 12px; border-radius: 6px; font-size: 14px; }
    input[type=text]:focus { outline: none; border-color: #58a6ff; }
    .btn-send { background: #238636; color: white; border: none; padding: 8px 20px; border-radius: 6px; cursor: pointer; font-size: 14px; }
    .btn-send:hover { background: #2ea043; }
    .tabs-mini { display: flex; gap: 0; border-bottom: 1px solid #30363d; padding: 0 16px; }
    .tab-mini { padding: 8px 14px; border: none; background: none; color: #8b949e; cursor: pointer; font-size: 13px; border-bottom: 2px solid transparent; margin-bottom: -1px; }
    .tab-mini.active { color: #58a6ff; border-bottom-color: #58a6ff; }
    textarea { width: 100%; height: 100px; background: #21262d; border: 1px solid #30363d; color: #c9d1d9; padding: 10px; border-radius: 6px; font-family: monospace; font-size: 13px; resize: vertical; }
    textarea:focus { outline: none; border-color: #58a6ff; }
    .response-area { flex: 1; overflow: hidden; display: flex; flex-direction: column; }
    .response-meta { padding: 10px 16px; background: #161b22; border-bottom: 1px solid #30363d; display: flex; gap: 20px; align-items: center; }
    .response-status { font-size: 14px; font-weight: 700; }
    .response-meta-item { font-size: 12px; color: #8b949e; }
    .response-body { flex: 1; overflow-y: auto; padding: 16px; }
    pre { font-family: 'Consolas', monospace; font-size: 13px; color: #c9d1d9; white-space: pre-wrap; word-break: break-all; }
    .loading { color: #58a6ff; padding: 20px; text-align: center; }
    .empty-state { color: #8b949e; padding: 40px; text-align: center; font-size: 14px; }
    .headers-area { padding: 10px 16px; }
    .header-row { display: flex; gap: 8px; margin-bottom: 6px; }
    .btn-add-header { background: none; border: 1px dashed #30363d; color: #8b949e; padding: 6px 12px; border-radius: 6px; cursor: pointer; font-size: 13px; }
    .btn-add-header:hover { border-color: #58a6ff; color: #58a6ff; }
    .btn-remove { background: none; border: none; color: #8b949e; cursor: pointer; font-size: 16px; padding: 0 4px; }
  </style>
</head>
<body>
  <div class="layout">
    <!-- Sidebar: History -->
    <div class="sidebar">
      <div class="sidebar-header">
        <h2>ประวัติ Requests</h2>
      </div>
      <div class="history-list" id="history-list">
        <div class="empty-state">ยังไม่มี requests</div>
      </div>
    </div>
    
    <!-- Main: Request + Response -->
    <div class="main">
      <!-- Request Panel -->
      <div class="request-panel">
        <div class="url-row">
          <select id="method">
            <option>GET</option>
            <option>POST</option>
            <option>PUT</option>
            <option>PATCH</option>
            <option>DELETE</option>
            <option>HEAD</option>
          </select>
          <input type="text" id="url" value="https://jsonplaceholder.typicode.com/todos/1" placeholder="URL">
          <button class="btn-send" id="btn-send">ส่ง</button>
        </div>
        
        <div class="tabs-mini">
          <button class="tab-mini active" data-reqtab="headers">Headers</button>
          <button class="tab-mini" data-reqtab="body">Body</button>
        </div>
        
        <div id="reqtab-headers" style="padding:10px 0;">
          <div id="headers-container"></div>
          <button class="btn-add-header" id="btn-add-header">+ เพิ่ม Header</button>
        </div>
        
        <div id="reqtab-body" style="display:none; padding:10px 0;">
          <textarea id="request-body" placeholder='{"key": "value"}'></textarea>
        </div>
      </div>
      
      <!-- Response Panel -->
      <div class="response-area">
        <div class="response-meta" id="response-meta" style="display:none;">
          <span class="response-status" id="response-status"></span>
          <span class="response-meta-item" id="response-time"></span>
          <span class="response-meta-item" id="response-size"></span>
        </div>
        
        <div id="response-container" class="response-body">
          <div class="empty-state">ส่ง request เพื่อดูผลลัพธ์</div>
        </div>
      </div>
    </div>
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

### renderer.js

```javascript
// renderer.js
let currentResponse = null
const customHeaders = []

// ─── Request Panel Tabs ───────────────────────────────────────────
document.querySelectorAll('.tab-mini').forEach(tab => {
  tab.addEventListener('click', () => {
    document.querySelectorAll('.tab-mini').forEach(t => t.classList.remove('active'))
    tab.classList.add('active')
    const panel = tab.dataset.reqtab
    document.getElementById('reqtab-headers').style.display = panel === 'headers' ? 'block' : 'none'
    document.getElementById('reqtab-body').style.display = panel === 'body' ? 'block' : 'none'
  })
})

// ─── Custom Headers ──────────────────────────────────────────────
document.getElementById('btn-add-header').addEventListener('click', addHeaderRow)

function addHeaderRow(key = '', value = '') {
  const container = document.getElementById('headers-container')
  const rowId = Date.now()
  
  const row = document.createElement('div')
  row.className = 'header-row'
  row.dataset.id = rowId
  row.innerHTML = `
    <input type="text" class="header-key" placeholder="Header Name" value="${key}" style="flex:1;">
    <input type="text" class="header-value" placeholder="Value" value="${value}" style="flex:2;">
    <button class="btn-remove" onclick="removeHeader(${rowId})">×</button>
  `
  
  container.appendChild(row)
}

window.removeHeader = (id) => {
  document.querySelector(`[data-id="${id}"]`)?.remove()
}

function getCustomHeaders() {
  const headers = {}
  document.querySelectorAll('.header-row').forEach(row => {
    const key = row.querySelector('.header-key').value.trim()
    const value = row.querySelector('.header-value').value.trim()
    if (key) headers[key] = value
  })
  return headers
}

// ─── Send Request ────────────────────────────────────────────────
document.getElementById('btn-send').addEventListener('click', sendRequest)

document.getElementById('url').addEventListener('keydown', (e) => {
  if (e.key === 'Enter') sendRequest()
})

async function sendRequest() {
  const method = document.getElementById('method').value
  const url = document.getElementById('url').value.trim()
  const bodyText = document.getElementById('request-body').value.trim()
  const headers = getCustomHeaders()
  
  if (!url) { return }
  
  const container = document.getElementById('response-container')
  container.innerHTML = '<div class="loading">กำลังส่ง request...</div>'
  document.getElementById('response-meta').style.display = 'none'
  document.getElementById('btn-send').disabled = true
  
  const config = { url, method, headers }
  if (bodyText && ['POST', 'PUT', 'PATCH'].includes(method)) {
    config.body = bodyText
  }
  
  const start = Date.now()
  const result = await window.netAPI.request(config)
  document.getElementById('btn-send').disabled = false
  
  if (result.success) {
    displayResponse(result.result)
    renderHistory()
  } else {
    container.innerHTML = `<pre style="color:#f85149;">Error: ${result.error}</pre>`
  }
}

function displayResponse(result) {
  currentResponse = result
  
  // Meta info
  const meta = document.getElementById('response-meta')
  meta.style.display = 'flex'
  
  const statusEl = document.getElementById('response-status')
  statusEl.textContent = `${result.status} ${result.statusText}`
  statusEl.style.color = result.status < 400 ? '#2ea043' : '#f85149'
  
  document.getElementById('response-time').textContent = `${result.elapsed} ms`
  
  const size = JSON.stringify(result.data).length
  document.getElementById('response-size').textContent = 
    size > 1024 ? `${(size / 1024).toFixed(1)} KB` : `${size} B`
  
  // Body
  const container = document.getElementById('response-container')
  let displayData
  
  if (typeof result.data === 'object') {
    displayData = JSON.stringify(result.data, null, 2)
  } else {
    displayData = result.data
  }
  
  container.innerHTML = `<pre>${escapeHTML(displayData)}</pre>`
}

function escapeHTML(str) {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .slice(0, 50000)
}

// ─── History ──────────────────────────────────────────────────────
async function renderHistory() {
  const history = await window.netAPI.getHistory()
  const list = document.getElementById('history-list')
  
  list.innerHTML = history.length ? history.map(item => `
    <div class="history-item" onclick='loadHistory(${JSON.stringify(item).replace(/'/g, "&#39;")})'>
      <div>
        <span class="history-method method-${item.method || 'GET'}">${item.method || 'GET'}</span>
        <span class="history-status ${item.success ? 'status-ok' : 'status-err'}">${item.status || '✗'}</span>
      </div>
      <div class="history-url">${item.url}</div>
    </div>
  `).join('') : '<div class="empty-state" style="padding:16px; font-size:13px;">ยังไม่มี requests</div>'
}

window.loadHistory = (item) => {
  if (item.url) document.getElementById('url').value = item.url
  if (item.method) document.getElementById('method').value = item.method
  if (item.success && item.data) displayResponse(item)
}

// ─── Events ───────────────────────────────────────────────────────
const unsubReq = window.netAPI.onRequestComplete((result) => {
  renderHistory()
})

window.addEventListener('beforeunload', () => unsubReq())

// Init
renderHistory()
```

## สรุป

Net module ของ Electron ให้ความสามารถ HTTP/HTTPS ที่ทรงพลัง:

- **net.request** - Traditional callback-based API
- **net.fetch** - Modern Promise-based API (Electron 21+)
- **Proxy support** - ใช้ system proxy อัตโนมัติ
- **Session integration** - ใช้ cookies จาก Electron session
- **Certificate handling** - จัดการ client/server certificates
- **Streaming** - รองรับ large files และ SSE
- **Authentication** - Basic, Bearer, API key และ auto token refresh

ข้อแตกต่างสำคัญจาก `fetch` ใน renderer:
- รันใน main process (ไม่มี CORS restrictions)
- รองรับ custom protocols
- ใช้ system proxy settings
- มี access ถึง Electron session cookies

จบหลักสูตร Electron.js ตอนที่ 11-20 แล้ว! เราได้เรียนรู้เกี่ยวกับ:
- Preload Scripts และ Context Bridge
- Security และ XSS Prevention
- Shell & System Integration
- Clipboard Operations
- Keyboard Shortcuts
- Power Monitor
- Screen API
- Protocol Handler
- Session & Cookies
- Net & HTTP Requests
