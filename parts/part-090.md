# Part 90: Electron Security Hardening

## เพิ่มความปลอดภัยให้ Electron App

ในบทนี้เราจะเรียน Content Security Policy, Subresource Integrity, certificate pinning, IPC security และ secure defaults

---

## 1. Security Checklist

```
✅ Security Configuration Checklist:

Main Process:
  ☑ contextIsolation: true
  ☑ nodeIntegration: false
  ☑ sandbox: true (ถ้าเป็นไปได้)
  ☑ webSecurity: true (ไม่เปิด allowRunningInsecureContent)
  ☑ allowRendererProcessReuse: true
  ☑ Check IPC handlers ว่ารับ request จาก trusted sources

Renderer:
  ☑ Content-Security-Policy header
  ☑ Subresource Integrity สำหรับ CDN resources
  ☑ No eval() / innerHTML ด้วย user input
  ☑ Sanitize HTML ก่อนแสดง

Network:
  ☑ Certificate pinning สำหรับ critical APIs
  ☑ ใช้ HTTPS เท่านั้น
  ☑ Validate certificates
  
Updates:
  ☑ Verify update signatures
  ☑ ใช้ HTTPS สำหรับ update server
```

---

## 2. Content Security Policy

```typescript
// src/main/security.ts
import { session, app } from 'electron'

export function setupSecurityHeaders(): void {
  session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
    callback({
      responseHeaders: {
        ...details.responseHeaders,
        'Content-Security-Policy': [buildCSP()],
        'X-Content-Type-Options': ['nosniff'],
        'X-Frame-Options': ['DENY'],
        'X-XSS-Protection': ['1; mode=block'],
        'Referrer-Policy': ['no-referrer'],
        'Permissions-Policy': [
          'camera=(), microphone=(), geolocation=(), payment=()'
        ]
      }
    })
  })
}

function buildCSP(): string {
  const isDev = process.env.NODE_ENV === 'development'

  const directives: Record<string, string[]> = {
    'default-src': ["'self'"],
    'script-src': [
      "'self'",
      ...(isDev ? ["'unsafe-eval'"] : []),  // สำหรับ dev HMR เท่านั้น
    ],
    'style-src': ["'self'", "'unsafe-inline'"],  // unsafe-inline จำเป็นสำหรับ CSS-in-JS
    'img-src': ["'self'", 'data:', 'blob:'],
    'font-src': ["'self'", 'data:'],
    'connect-src': [
      "'self'",
      ...(isDev ? ['ws://localhost:*', 'http://localhost:*'] : []),
      'https://api.example.com'
    ],
    'media-src': ["'self'", 'blob:'],
    'object-src': ["'none'"],
    'base-uri': ["'self'"],
    'form-action': ["'self'"],
    'frame-ancestors': ["'none'"],
    'worker-src': ["'self'", 'blob:']
  }

  return Object.entries(directives)
    .map(([key, values]) => `${key} ${values.join(' ')}`)
    .join('; ')
}
```

---

## 3. Certificate Pinning

```typescript
// src/main/certificatePinning.ts
import { app, net } from 'electron'
import * as crypto from 'crypto'

// Pinned certificate fingerprints (SHA-256)
const PINNED_CERTS: Record<string, string[]> = {
  'api.example.com': [
    'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=',  // Primary cert
    'sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB='   // Backup cert
  ]
}

export function setupCertificatePinning(): void {
  app.on('certificate-error', (event, webContents, url, error, certificate, callback) => {
    event.preventDefault()

    try {
      const urlObj = new URL(url)
      const pinnedFingerprints = PINNED_CERTS[urlObj.hostname]

      if (!pinnedFingerprints) {
        // ไม่มี pin สำหรับ host นี้ - ใช้ default behavior
        callback(false)
        return
      }

      // คำนวณ fingerprint ของ certificate ที่ได้รับ
      const certFingerprint = getCertFingerprint(certificate.data)
      const isPinned = pinnedFingerprints.includes(`sha256/${certFingerprint}`)

      if (!isPinned) {
        console.error(`Certificate pinning failed for ${urlObj.hostname}`)
        console.error(`Expected: ${pinnedFingerprints.join(', ')}`)
        console.error(`Got: sha256/${certFingerprint}`)
      }

      callback(isPinned)
    } catch {
      callback(false)
    }
  })
}

function getCertFingerprint(certData: string): string {
  // certData เป็น PEM format
  const der = Buffer.from(certData.replace(/-----[^-]+-----/g, '').replace(/\s/g, ''), 'base64')
  return crypto.createHash('sha256').update(der).digest('base64')
}
```

---

## 4. IPC Security

```typescript
// src/main/ipcSecurity.ts
import { ipcMain, BrowserWindow, webContents } from 'electron'

// Allowlist ของ IPC channels ที่ renderer อนุญาตให้เรียก
const ALLOWED_CHANNELS = new Set([
  'file:open', 'file:save', 'file:list',
  'settings:get', 'settings:set',
  'dialog:open', 'dialog:save'
])

// Middleware ตรวจสอบ IPC requests
export function setupIPCSecurity(): void {
  // Block IPC ที่ไม่อยู่ใน allowlist
  ipcMain.on('*' as string, (event, channel) => {
    if (!ALLOWED_CHANNELS.has(channel)) {
      console.warn(`Blocked unauthorized IPC channel: ${channel}`)
      event.returnValue = { error: 'Unauthorized channel' }
    }
  })

  // Validate sender origin
  function validateSender(event: Electron.IpcMainEvent | Electron.IpcMainInvokeEvent): boolean {
    const { url } = event.sender

    // อนุญาตเฉพาะ local files หรือ localhost (dev)
    const isLocal = url.startsWith('file://') ||
      url.startsWith('http://localhost:') ||
      url.startsWith('https://localhost:')

    if (!isLocal) {
      console.error(`IPC from untrusted origin: ${url}`)
      return false
    }

    return true
  }

  // Wrap ทุก handler ด้วย origin check
  const originalHandle = ipcMain.handle.bind(ipcMain)
  ipcMain.handle = function(channel, handler) {
    return originalHandle(channel, async (event, ...args) => {
      if (!validateSender(event)) {
        throw new Error('Unauthorized IPC origin')
      }
      return handler(event, ...args)
    }) as ReturnType<typeof originalHandle>
  }
}

// Rate limiting สำหรับ IPC
export function createIPCRateLimiter(maxRequests: number, windowMs: number) {
  const requests = new Map<string, number[]>()

  return function checkLimit(senderId: string): boolean {
    const now = Date.now()
    const windowStart = now - windowMs

    const timestamps = (requests.get(senderId) || []).filter(t => t > windowStart)
    timestamps.push(now)
    requests.set(senderId, timestamps)

    return timestamps.length <= maxRequests
  }
}
```

---

## 5. Secure Storage

```typescript
// src/main/secureStorage.ts
// ใช้ OS keychain สำหรับ sensitive data

import keytar from 'keytar'
import { app } from 'electron'

const SERVICE_NAME = app.getName()

export async function storeSecret(key: string, value: string): Promise<void> {
  await keytar.setPassword(SERVICE_NAME, key, value)
}

export async function getSecret(key: string): Promise<string | null> {
  return keytar.getPassword(SERVICE_NAME, key)
}

export async function deleteSecret(key: string): Promise<boolean> {
  return keytar.deletePassword(SERVICE_NAME, key)
}

// API token management
export async function storeAPIKey(serviceName: string, token: string): Promise<void> {
  await storeSecret(`api-key:${serviceName}`, token)
}

export async function getAPIKey(serviceName: string): Promise<string | null> {
  return getSecret(`api-key:${serviceName}`)
}
```

---

## 6. Permission System

```typescript
// src/main/permissionHandler.ts
import { session } from 'electron'

type Permission = 'media' | 'geolocation' | 'notifications' | 'clipboard-read'

const ALLOWED_PERMISSIONS: Set<Permission> = new Set(['media', 'notifications'])

export function setupPermissions(): void {
  session.defaultSession.setPermissionRequestHandler(
    (webContents, permission, callback, details) => {
      const perm = permission as Permission
      const url = webContents.getURL()

      // เฉพาะ local content เท่านั้น
      if (!url.startsWith('file://') && !url.startsWith('http://localhost')) {
        console.warn(`Permission ${permission} denied for external URL: ${url}`)
        callback(false)
        return
      }

      // Check allowlist
      if (!ALLOWED_PERMISSIONS.has(perm)) {
        console.log(`Permission ${permission} not in allowlist - denied`)
        callback(false)
        return
      }

      // Camera/microphone: ต้องมี user gesture
      if (perm === 'media' && !details.requestingUrl) {
        callback(false)
        return
      }

      callback(true)
    }
  )

  // ตรวจสอบ navigation
  session.defaultSession.setPermissionCheckHandler(
    (webContents, permission) => {
      const perm = permission as Permission
      return ALLOWED_PERMISSIONS.has(perm)
    }
  )
}
```

---

## 7. Audit Script

```typescript
// scripts/securityAudit.ts
// ตรวจสอบ security configuration อัตโนมัติ

import * as fs from 'fs'

interface AuditResult {
  check: string
  passed: boolean
  severity: 'critical' | 'high' | 'medium' | 'low'
  detail: string
}

function auditMainProcess(): AuditResult[] {
  const mainCode = fs.readFileSync('src/main/index.ts', 'utf-8')
  const results: AuditResult[] = []

  results.push({
    check: 'contextIsolation enabled',
    passed: mainCode.includes('contextIsolation: true'),
    severity: 'critical',
    detail: 'ต้องเปิด contextIsolation ทุกครั้ง'
  })

  results.push({
    check: 'nodeIntegration disabled',
    passed: mainCode.includes('nodeIntegration: false'),
    severity: 'critical',
    detail: 'ต้องปิด nodeIntegration ใน renderer'
  })

  results.push({
    check: 'No webSecurity:false',
    passed: !mainCode.includes('webSecurity: false'),
    severity: 'high',
    detail: 'อย่าปิด webSecurity'
  })

  return results
}

const results = auditMainProcess()
const failed = results.filter(r => !r.passed)

console.log('\n🔒 Security Audit Results\n')
for (const r of results) {
  const icon = r.passed ? '✅' : (r.severity === 'critical' ? '🚨' : '⚠️')
  console.log(`${icon} [${r.severity.toUpperCase()}] ${r.check}`)
  if (!r.passed) console.log(`   → ${r.detail}`)
}

console.log(`\n${failed.length === 0 ? '✅ All checks passed' : `❌ ${failed.length} checks failed`}`)
if (failed.some(r => r.severity === 'critical')) process.exit(1)
```

---

## สรุป

| มาตรการ | Implementation | ป้องกัน |
|--------|---------------|---------|
| CSP | webRequest headers | XSS, code injection |
| contextIsolation | webPreferences | preload script isolation |
| Certificate pinning | certificate-error event | MITM attacks |
| IPC origin check | sender.url validation | renderer hijacking |
| OS Keychain | keytar | credential storage |
| Permission handler | setPermissionRequestHandler | API abuse |
| Rate limiting | timestamp windows | IPC flooding |
