# Part 31: Security Best Practices ใน Electron

## บทนำ

Security ใน Electron แตกต่างจาก web app ทั่วไปเพราะมีสิทธิ์เข้าถึง Node.js APIs โดยตรง การมี XSS vulnerability อาจทำให้ผู้โจมตีสามารถรัน arbitrary code ใน OS ได้ ในบทนี้จะครอบคลุม security checklist, CSP, permission handling, และ audit tools

## Electron Security Checklist

```javascript
// electron/security/securityCheck.js
/**
 * ตรวจสอบ security settings ทั้งหมด
 */
function runSecurityCheck(mainWindow) {
  const issues = []

  const wc = mainWindow.webContents
  
  // 1. ตรวจสอบ nodeIntegration
  if (wc.getWebPreferences().nodeIntegration) {
    issues.push({
      severity: 'CRITICAL',
      rule: 'NO_NODE_INTEGRATION',
      message: 'nodeIntegration ควรเป็น false'
    })
  }

  // 2. ตรวจสอบ contextIsolation
  if (!wc.getWebPreferences().contextIsolation) {
    issues.push({
      severity: 'CRITICAL',
      rule: 'CONTEXT_ISOLATION',
      message: 'contextIsolation ควรเป็น true'
    })
  }

  // 3. ตรวจสอบ sandbox
  if (!wc.getWebPreferences().sandbox) {
    issues.push({
      severity: 'HIGH',
      rule: 'ENABLE_SANDBOX',
      message: 'sandbox ควรเป็น true สำหรับ untrusted content'
    })
  }

  // 4. ตรวจสอบ webSecurity
  if (wc.getWebPreferences().webSecurity === false) {
    issues.push({
      severity: 'HIGH',
      rule: 'WEB_SECURITY',
      message: 'webSecurity ไม่ควรปิดใน production'
    })
  }

  // 5. ตรวจสอบ allowRunningInsecureContent
  if (wc.getWebPreferences().allowRunningInsecureContent) {
    issues.push({
      severity: 'HIGH',
      rule: 'NO_INSECURE_CONTENT',
      message: 'allowRunningInsecureContent ควรเป็น false'
    })
  }

  if (issues.length === 0) {
    console.log('✅ Security check passed!')
  } else {
    issues.forEach(issue => {
      console.warn(`[${issue.severity}] ${issue.rule}: ${issue.message}`)
    })
  }

  return issues
}

module.exports = { runSecurityCheck }
```

### Secure BrowserWindow Configuration

```javascript
// electron/main.js - Secure window creation
function createSecureWindow() {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      // === REQUIRED SECURITY SETTINGS ===
      nodeIntegration: false,          // ❌ อย่าเปิด
      contextIsolation: true,          // ✅ ต้องเปิด
      sandbox: true,                   // ✅ เปิดสำหรับ untrusted content
      preload: path.join(__dirname, 'preload.js'), // ✅ ใช้ preload แทน nodeIntegration
      
      // === RECOMMENDED SETTINGS ===
      webSecurity: true,               // ✅ เปิดเสมอ
      allowRunningInsecureContent: false, // ❌ อย่าอนุญาต mixed content
      experimentalFeatures: false,     // ❌ อย่าเปิด experimental features
      
      // === ADDITIONAL HARDENING ===
      enableBlinkFeatures: '',         // ❌ ไม่ enable features เพิ่มเติม
      disableBlinkFeatures: 'Auxclick', // ✅ Disable features ที่ไม่ต้องการ
      
      // === NAVIGATION ===
      navigateOnDragDrop: false,       // ❌ ป้องกัน navigation โดย drag
      
      // === PERMISSIONS ===
      // จัดการ permissions ผ่าน session handlers แทน
    }
  })

  return win
}
```

## Content Security Policy (CSP)

```javascript
// electron/security/csp.js
const { session } = require('electron')

/**
 * ตั้งค่า CSP Headers สำหรับ Electron app
 */
function setupCSP() {
  // ตั้งค่า CSP ผ่าน HTTP response headers
  session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
    const responseHeaders = {
      ...details.responseHeaders,
      'Content-Security-Policy': [buildCSP()]
    }
    
    callback({ responseHeaders })
  })
}

/**
 * สร้าง CSP string
 */
function buildCSP(options = {}) {
  const isDev = process.env.NODE_ENV === 'development'
  
  const policies = {
    'default-src': ["'self'"],
    'script-src': [
      "'self'",
      // ใน dev อนุญาต inline scripts สำหรับ hot reload
      ...(isDev ? ["'unsafe-inline'", "'unsafe-eval'"] : [])
    ],
    'style-src': [
      "'self'",
      "'unsafe-inline'",  // CSS-in-JS ต้องการสิ่งนี้
      'https://fonts.googleapis.com'
    ],
    'font-src': [
      "'self'",
      'https://fonts.gstatic.com',
      'data:'
    ],
    'img-src': [
      "'self'",
      'data:',
      'blob:',
      'https:'
    ],
    'connect-src': [
      "'self'",
      ...(isDev ? ['ws://localhost:5173', 'http://localhost:5173'] : []),
      // เพิ่ม APIs ที่จำเป็น
      'https://api.yourapp.com'
    ],
    'media-src': ["'self'", 'blob:'],
    'object-src': ["'none'"],         // ❌ Block Flash, etc.
    'base-uri': ["'self'"],
    'form-action': ["'self'"],
    'frame-ancestors': ["'none'"],    // ❌ Block iframe embedding
    'upgrade-insecure-requests': []
  }

  // Override ด้วย options ที่ส่งมา
  Object.assign(policies, options)

  return Object.entries(policies)
    .map(([key, values]) => {
      if (values.length === 0) return key
      return `${key} ${values.join(' ')}`
    })
    .join('; ')
}

module.exports = { setupCSP, buildCSP }
```

### HTML Meta CSP

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  
  <!-- CSP ใน HTML (สำรองจาก HTTP header) -->
  <meta http-equiv="Content-Security-Policy" 
        content="default-src 'self'; 
                 script-src 'self'; 
                 style-src 'self' 'unsafe-inline';
                 img-src 'self' data: blob:;
                 connect-src 'self';
                 object-src 'none';
                 base-uri 'self'">
  
  <title>Secure Electron App</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.jsx"></script>
</body>
</html>
```

## Permission Handler

```javascript
// electron/security/permissionHandler.js
const { session, dialog } = require('electron')

/**
 * จัดการ permissions ที่ web content ขอ
 */
function setupPermissionHandlers(mainWindow) {
  
  // Permission request handler
  session.defaultSession.setPermissionRequestHandler(
    async (webContents, permission, callback, details) => {
      // URL ของ page ที่ขอ permission
      const url = webContents.getURL()
      
      console.log(`[Permission] Request: ${permission} from ${url}`)
      
      // สร้าง whitelist ของ permissions ที่อนุญาต
      const allowedPermissions = getAllowedPermissions(url)
      
      if (!allowedPermissions.includes(permission)) {
        console.warn(`[Permission] Denied: ${permission}`)
        return callback(false)
      }
      
      // สำหรับ permissions ที่ sensitive ถามผู้ใช้
      if (isSensitivePermission(permission)) {
        const granted = await askUserPermission(
          mainWindow,
          permission,
          url,
          details
        )
        return callback(granted)
      }
      
      callback(true)
    }
  )

  // Permission check handler (ตรวจสอบก่อน request)
  session.defaultSession.setPermissionCheckHandler(
    (webContents, permission, requestingOrigin, details) => {
      const allowedOrigins = ['https://yourtrustedsite.com']
      
      if (permission === 'fullscreen') return true
      if (allowedOrigins.some(o => requestingOrigin.startsWith(o))) return true
      
      return false
    }
  )
}

function getAllowedPermissions(url) {
  // เฉพาะ local content อาจต้องการ permissions เหล่านี้
  if (url.startsWith('file://') || url.startsWith('http://localhost')) {
    return ['notifications', 'clipboard-read', 'clipboard-write', 'fullscreen']
  }
  
  // Remote content มี permissions น้อยกว่า
  return ['fullscreen']
}

function isSensitivePermission(permission) {
  return ['camera', 'microphone', 'geolocation', 'notifications'].includes(permission)
}

async function askUserPermission(mainWindow, permission, url, details) {
  const permissionNames = {
    camera: 'กล้อง',
    microphone: 'ไมโครโฟน',
    geolocation: 'ตำแหน่งที่ตั้ง',
    notifications: 'การแจ้งเตือน',
    'clipboard-read': 'อ่าน Clipboard'
  }

  const { response } = await dialog.showMessageBox(mainWindow, {
    type: 'question',
    title: 'คำขอ Permission',
    message: `เว็บไซต์ขอสิทธิ์ ${permissionNames[permission] || permission}`,
    detail: `จาก: ${new URL(url).hostname}`,
    buttons: ['อนุญาต', 'ปฏิเสธ'],
    defaultId: 1,
    cancelId: 1
  })

  return response === 0
}

module.exports = { setupPermissionHandlers }
```

## Safe shell.openExternal

```javascript
// electron/security/safeShell.js
const { shell } = require('electron')
const { URL } = require('url')

/**
 * รายการ protocols ที่อนุญาต
 */
const ALLOWED_PROTOCOLS = new Set([
  'https:',
  'http:',
  'mailto:',
  'tel:'
])

/**
 * รายการ domains ที่อนุญาต (ถ้าต้องการ whitelist)
 */
const TRUSTED_DOMAINS = new Set([
  'github.com',
  'stackoverflow.com',
  // เพิ่ม domains ที่เชื่อถือได้
])

/**
 * เปิด URL อย่างปลอดภัย
 */
async function safeOpenExternal(url) {
  // ตรวจสอบว่า url เป็น string
  if (typeof url !== 'string') {
    throw new Error('URL ต้องเป็น string')
  }

  // Trim whitespace
  url = url.trim()

  // Parse URL
  let parsed
  try {
    parsed = new URL(url)
  } catch {
    throw new Error(`Invalid URL: ${url}`)
  }

  // ตรวจสอบ protocol
  if (!ALLOWED_PROTOCOLS.has(parsed.protocol)) {
    throw new Error(`Protocol ไม่ได้รับอนุญาต: ${parsed.protocol}`)
  }

  // ป้องกัน localhost navigation (ถ้าต้องการ)
  if (parsed.hostname === 'localhost' || parsed.hostname === '127.0.0.1') {
    throw new Error('ไม่อนุญาตให้เปิด localhost')
  }

  // ป้องกัน javascript: protocol (แม้จะถูกกรองแล้ว)
  if (url.toLowerCase().startsWith('javascript:')) {
    throw new Error('JavaScript URLs ไม่ได้รับอนุญาต')
  }

  await shell.openExternal(url)
  return { success: true, url }
}

/**
 * IPC Handler สำหรับ safeOpenExternal
 */
function setupSafeShellHandlers() {
  const { ipcMain } = require('electron')
  
  ipcMain.handle('shell:openExternal', async (event, url) => {
    // Validate sender
    const senderURL = event.sender.getURL()
    if (!isTrustedSender(senderURL)) {
      throw new Error('Sender is not trusted')
    }
    
    return safeOpenExternal(url)
  })
}

function isTrustedSender(url) {
  return url.startsWith('file://') || url.startsWith('http://localhost')
}

module.exports = { safeOpenExternal, setupSafeShellHandlers }
```

## IPC Argument Validation

```javascript
// electron/security/ipcValidator.js

/**
 * Schema-based IPC validation
 */
const validators = {
  string: (val) => typeof val === 'string',
  number: (val) => typeof val === 'number' && !isNaN(val),
  integer: (val) => Number.isInteger(val),
  boolean: (val) => typeof val === 'boolean',
  array: (val) => Array.isArray(val),
  object: (val) => val !== null && typeof val === 'object' && !Array.isArray(val),
  
  // Custom validators
  nonEmptyString: (val) => typeof val === 'string' && val.trim().length > 0,
  positiveNumber: (val) => typeof val === 'number' && val > 0,
  filePath: (val) => typeof val === 'string' && val.length > 0 && !val.includes('\0'),
  email: (val) => typeof val === 'string' && /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(val),
  url: (val) => {
    try { new URL(val); return true } catch { return false }
  },
  safeId: (val) => typeof val === 'string' && /^[a-zA-Z0-9_-]+$/.test(val)
}

/**
 * Validate IPC arguments
 */
function validateArgs(args, schema) {
  const errors = []
  
  for (const [key, rule] of Object.entries(schema)) {
    const value = args[key]
    
    // ตรวจสอบ required
    if (rule.required && (value === undefined || value === null)) {
      errors.push(`${key}: required`)
      continue
    }
    
    // Skip validation ถ้า optional และ undefined
    if (!rule.required && (value === undefined || value === null)) {
      continue
    }
    
    // Type validation
    if (rule.type && !validators[rule.type]?.(value)) {
      errors.push(`${key}: must be ${rule.type}`)
      continue
    }
    
    // Min/Max สำหรับ numbers
    if (typeof value === 'number') {
      if (rule.min !== undefined && value < rule.min) {
        errors.push(`${key}: must be >= ${rule.min}`)
      }
      if (rule.max !== undefined && value > rule.max) {
        errors.push(`${key}: must be <= ${rule.max}`)
      }
    }
    
    // Length สำหรับ strings/arrays
    if (typeof value === 'string' || Array.isArray(value)) {
      if (rule.minLength !== undefined && value.length < rule.minLength) {
        errors.push(`${key}: length must be >= ${rule.minLength}`)
      }
      if (rule.maxLength !== undefined && value.length > rule.maxLength) {
        errors.push(`${key}: length must be <= ${rule.maxLength}`)
      }
    }
    
    // Enum validation
    if (rule.enum && !rule.enum.includes(value)) {
      errors.push(`${key}: must be one of ${rule.enum.join(', ')}`)
    }
    
    // Custom validator
    if (rule.validate && !rule.validate(value)) {
      errors.push(`${key}: ${rule.message || 'invalid value'}`)
    }
  }
  
  return errors
}

/**
 * IPC handler wrapper พร้อม validation
 */
function createValidatedHandler(schema, handler) {
  return async (event, ...args) => {
    // แปลง args เป็น object ถ้าจำเป็น
    const argsObject = Array.isArray(args[0]) ? args : args[0] || {}
    
    const errors = validateArgs(argsObject, schema)
    
    if (errors.length > 0) {
      throw new Error(`Validation errors: ${errors.join(', ')}`)
    }
    
    return handler(event, argsObject)
  }
}

// ตัวอย่างการใช้งาน
const noteSchema = {
  title: { type: 'string', maxLength: 500, required: true },
  content: { type: 'string', maxLength: 100000 },
  userId: { type: 'integer', required: true, min: 1 },
  color: { 
    type: 'string',
    validate: (v) => /^#[0-9a-fA-F]{6}$/.test(v),
    message: 'must be a valid hex color'
  }
}

module.exports = { validateArgs, createValidatedHandler, validators }
```

## ป้องกัน Dangerous Features

```javascript
// electron/security/disableFeatures.js
const { app } = require('electron')

/**
 * Disable features ที่ไม่จำเป็นและอาจเป็น security risk
 */
function disableDangerousFeatures() {
  // Disable GPU compositing (ถ้าไม่ต้องการ)
  // app.disableHardwareAcceleration()
  
  // Disable remote module (Electron 14+: removed by default)
  // ใน version เก่า: app.enableRemoteModule = false
  
  // ป้องกัน new-window events ที่อาจ bypass security
  app.on('web-contents-created', (event, contents) => {
    
    // Block การสร้าง windows ใหม่จาก renderer
    contents.setWindowOpenHandler(({ url, frameName, disposition }) => {
      // ตรวจสอบว่า URL ปลอดภัยหรือไม่
      const safeUrl = isSafeURL(url)
      
      if (!safeUrl) {
        console.warn('[Security] Blocked window.open:', url)
        return { action: 'deny' }
      }
      
      // เปิดใน external browser แทน
      const { shell } = require('electron')
      shell.openExternal(url)
      return { action: 'deny' }
    })

    // ป้องกัน navigation ไปยัง URLs ที่ไม่ปลอดภัย
    contents.on('will-navigate', (event, url) => {
      if (!isAllowedNavigation(url)) {
        console.warn('[Security] Blocked navigation:', url)
        event.preventDefault()
      }
    })

    // Block redirects ที่ไม่ปลอดภัย
    contents.on('will-redirect', (event, url) => {
      if (!isAllowedNavigation(url)) {
        console.warn('[Security] Blocked redirect:', url)
        event.preventDefault()
      }
    })
  })
}

function isSafeURL(url) {
  try {
    const parsed = new URL(url)
    return ['https:', 'http:'].includes(parsed.protocol)
  } catch {
    return false
  }
}

function isAllowedNavigation(url) {
  // อนุญาตเฉพาะ app pages
  const allowedURLs = [
    'http://localhost:5173',     // Dev server
    'file://'                    // Production build
  ]
  
  return allowedURLs.some(allowed => url.startsWith(allowed))
}

module.exports = { disableDangerousFeatures }
```

## Security Audit Tool

```javascript
// scripts/security-audit.js
const fs = require('fs')
const path = require('path')

class SecurityAudit {
  constructor(projectRoot) {
    this.projectRoot = projectRoot
    this.findings = []
  }

  /**
   * รัน audit ทั้งหมด
   */
  async run() {
    console.log('🔍 Starting Security Audit...\n')
    
    await this.checkPackageJson()
    await this.checkMainProcess()
    await this.checkPreloadScripts()
    await this.checkRendererFiles()
    
    this.printReport()
    return this.findings
  }

  /**
   * ตรวจสอบ package.json
   */
  async checkPackageJson() {
    const pkgPath = path.join(this.projectRoot, 'package.json')
    if (!fs.existsSync(pkgPath)) return
    
    const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf-8'))
    
    // ตรวจสอบ deprecated/vulnerable packages
    const dangerousPackages = ['electron-remote', 'remote']
    dangerousPackages.forEach(pkg_name => {
      if (pkg.dependencies?.[pkg_name] || pkg.devDependencies?.[pkg_name]) {
        this.addFinding('HIGH', 'DANGEROUS_PACKAGE', 
          `Package "${pkg_name}" มีความเสี่ยงด้าน security`
        )
      }
    })
  }

  /**
   * ตรวจสอบ main process files
   */
  async checkMainProcess() {
    const mainFiles = this.findFiles('electron', '*.js')
    
    for (const file of mainFiles) {
      const content = fs.readFileSync(file, 'utf-8')
      
      // ตรวจสอบ nodeIntegration: true
      if (/nodeIntegration\s*:\s*true/.test(content)) {
        this.addFinding('CRITICAL', 'NODE_INTEGRATION',
          `${file}: nodeIntegration: true - ควรเป็น false`
        )
      }
      
      // ตรวจสอบ contextIsolation: false
      if (/contextIsolation\s*:\s*false/.test(content)) {
        this.addFinding('CRITICAL', 'CONTEXT_ISOLATION',
          `${file}: contextIsolation: false - ควรเป็น true`
        )
      }
      
      // ตรวจสอบ webSecurity: false
      if (/webSecurity\s*:\s*false/.test(content)) {
        this.addFinding('HIGH', 'WEB_SECURITY',
          `${file}: webSecurity: false - อย่าใช้ใน production`
        )
      }
      
      // ตรวจสอบ shell.openExternal โดยไม่มี validation
      const openExternalMatches = content.match(/shell\.openExternal\([^)]+\)/g)
      if (openExternalMatches) {
        openExternalMatches.forEach(match => {
          if (!match.includes('safe') && !match.includes('validate')) {
            this.addFinding('MEDIUM', 'UNSAFE_OPEN_EXTERNAL',
              `${file}: พิจารณา validate URL ก่อน shell.openExternal`
            )
          }
        })
      }
    }
  }

  async checkPreloadScripts() {
    const preloadFiles = this.findFiles('electron', 'preload*.js')
    
    for (const file of preloadFiles) {
      const content = fs.readFileSync(file, 'utf-8')
      
      // ตรวจสอบว่าใช้ contextBridge
      if (!content.includes('contextBridge.exposeInMainWorld')) {
        this.addFinding('HIGH', 'MISSING_CONTEXT_BRIDGE',
          `${file}: ควรใช้ contextBridge.exposeInMainWorld`
        )
      }
      
      // ตรวจสอบ ipcRenderer.on โดยไม่มี channel validation
      const onPatterns = content.match(/ipcRenderer\.on\(['"`][^'"`]+['"`]/g)
      if (onPatterns) {
        this.addFinding('INFO', 'IPC_CHANNEL_CHECK',
          `${file}: ตรวจสอบว่า IPC channels ได้รับการ validate แล้ว`
        )
      }
    }
  }

  async checkRendererFiles() {
    const srcFiles = this.findFiles('src', '*.{js,jsx,ts,tsx}')
    
    for (const file of srcFiles) {
      const content = fs.readFileSync(file, 'utf-8')
      
      // ตรวจสอบ dangerouslySetInnerHTML
      if (content.includes('dangerouslySetInnerHTML')) {
        this.addFinding('HIGH', 'DANGEROUS_INNER_HTML',
          `${file}: dangerouslySetInnerHTML อาจเป็น XSS vulnerability`
        )
      }
      
      // ตรวจสอบ innerHTML
      if (/\.innerHTML\s*=/.test(content)) {
        this.addFinding('MEDIUM', 'INNER_HTML',
          `${file}: การใช้ innerHTML โดยตรงอาจเป็น XSS risk`
        )
      }
      
      // ตรวจสอบ eval
      if (/\beval\s*\(/.test(content)) {
        this.addFinding('HIGH', 'EVAL_USAGE',
          `${file}: การใช้ eval() เป็น security risk`
        )
      }
    }
  }

  addFinding(severity, rule, message) {
    this.findings.push({ severity, rule, message, timestamp: new Date() })
  }

  findFiles(dir, pattern) {
    const fullDir = path.join(this.projectRoot, dir)
    if (!fs.existsSync(fullDir)) return []
    
    // Simple file finder
    const files = []
    const walkDir = (currentDir) => {
      const entries = fs.readdirSync(currentDir, { withFileTypes: true })
      entries.forEach(entry => {
        const fullPath = path.join(currentDir, entry.name)
        if (entry.isDirectory() && !entry.name.includes('node_modules')) {
          walkDir(fullPath)
        } else if (entry.isFile() && entry.name.endsWith('.js')) {
          files.push(fullPath)
        }
      })
    }
    walkDir(fullDir)
    return files
  }

  printReport() {
    const counts = { CRITICAL: 0, HIGH: 0, MEDIUM: 0, LOW: 0, INFO: 0 }
    
    this.findings.forEach(f => {
      counts[f.severity] = (counts[f.severity] || 0) + 1
      
      const icons = { CRITICAL: '🚨', HIGH: '⚠️', MEDIUM: '⚡', LOW: 'ℹ️', INFO: '💡' }
      console.log(`${icons[f.severity]} [${f.severity}] ${f.rule}`)
      console.log(`   ${f.message}\n`)
    })
    
    console.log('=== Summary ===')
    Object.entries(counts).forEach(([level, count]) => {
      if (count > 0) console.log(`${level}: ${count}`)
    })
    
    if (counts.CRITICAL > 0) {
      console.log('\n🚨 พบ CRITICAL issues! ต้องแก้ไขก่อน deploy')
    } else if (counts.HIGH > 0) {
      console.log('\n⚠️ พบ HIGH severity issues - ควรแก้ไข')
    } else {
      console.log('\n✅ ไม่พบ critical security issues')
    }
  }
}

// รัน audit
const audit = new SecurityAudit(process.cwd())
audit.run().catch(console.error)
```

## สรุป Security Checklist

```javascript
// electron/security/securityChecklist.js
/**
 * Security checklist สำหรับ Electron apps
 */
const SECURITY_CHECKLIST = {
  CRITICAL: [
    '✅ nodeIntegration: false',
    '✅ contextIsolation: true',
    '✅ ใช้ contextBridge สำหรับ IPC',
    '✅ Validate ทุก IPC arguments',
    '✅ ไม่ expose Node.js APIs โดยตรง'
  ],
  HIGH: [
    '✅ sandbox: true (ถ้าเป็นไปได้)',
    '✅ webSecurity: true',
    '✅ Validate URLs ก่อน shell.openExternal',
    '✅ ตั้งค่า Content Security Policy',
    '✅ Block window.open ไปยัง untrusted URLs',
    '✅ ป้องกัน navigation ที่ไม่ได้รับอนุญาต'
  ],
  MEDIUM: [
    '✅ จัดการ permission requests อย่างรอบคอบ',
    '✅ ไม่ใช้ eval() หรือ new Function()',
    '✅ ไม่ใช้ innerHTML กับ untrusted data',
    '✅ Log security events'
  ],
  LOW: [
    '✅ ปิด DevTools ใน production',
    '✅ ไม่ส่ง sensitive data ผ่าน IPC ที่ไม่จำเป็น',
    '✅ ใช้ HTTPS สำหรับ external requests',
    '✅ รัน npm audit เป็นประจำ'
  ]
}

module.exports = SECURITY_CHECKLIST
```
