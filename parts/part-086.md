# Part 86: Electron App Store Distribution

## เผยแพร่แอปผ่าน Mac App Store และ Microsoft Store

ในบทนี้เราจะเรียนการ submit ไปยัง Mac App Store (MAS), Microsoft Store (MSIX), sandbox requirements และ review guidelines

---

## 1. Mac App Store (MAS) Setup

```json
// electron-builder.json5 - MAS configuration
{
  "mac": {
    "target": [
      { "target": "mas", "arch": ["x64", "arm64", "universal"] },
      { "target": "mas-dev", "arch": ["universal"] }  // สำหรับ testing
    ],
    "hardenedRuntime": false,   // MAS ไม่ใช้ hardened runtime
    "gatekeeperAssess": false,
    "entitlements": "build/entitlements.mas.plist",
    "entitlementsInherit": "build/entitlements.mas.inherit.plist",
    "provisioningProfile": "build/MyApp_AppStore.provisionprofile"
  },
  "mas": {
    "entitlements": "build/entitlements.mas.plist",
    "entitlementsInherit": "build/entitlements.mas.inherit.plist",
    "provisioningProfile": "build/MyApp_AppStore.provisionprofile"
  }
}
```

```xml
<!-- build/entitlements.mas.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- App Sandbox (required for MAS) -->
  <key>com.apple.security.app-sandbox</key>
  <true/>

  <!-- Network access -->
  <key>com.apple.security.network.client</key>
  <true/>
  <key>com.apple.security.network.server</key>
  <true/>

  <!-- File access -->
  <key>com.apple.security.files.user-selected.read-write</key>
  <true/>
  <key>com.apple.security.files.downloads.read-write</key>
  <true/>

  <!-- JIT (required for V8) -->
  <key>com.apple.security.cs.allow-jit</key>
  <true/>
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
  <true/>

  <!-- Bookmarks สำหรับ persistent file access -->
  <key>com.apple.security.files.bookmarks.app-scope</key>
  <true/>
  <key>com.apple.security.files.bookmarks.document-scope</key>
  <true/>
</dict>
</plist>
```

---

## 2. Security-Scoped Bookmarks (MAS)

```typescript
// src/main/masBookmarks.ts
// MAS sandbox จำกัด file access ต้องใช้ security-scoped bookmarks

import { app, ipcMain, dialog } from 'electron'
import ElectronStore from 'electron-store'
import * as path from 'path'

const store = new ElectronStore<{ bookmarks: Record<string, string> }>()

export function registerBookmarkHandlers(): void {
  // เปิดไฟล์พร้อม bookmark
  ipcMain.handle('mas:open-file-with-bookmark', async () => {
    const result = await dialog.showOpenDialog({
      properties: ['openFile']
    })
    if (result.canceled || !result.filePaths[0]) return null

    const filePath = result.filePaths[0]

    // บันทึก security-scoped bookmark (macOS เท่านั้น)
    if (process.platform === 'darwin' && app.startAccessingSecurityScopedResource) {
      const bookmark = await createBookmark(filePath)
      if (bookmark) {
        const bookmarks = store.get('bookmarks', {})
        bookmarks[filePath] = bookmark
        store.set('bookmarks', bookmarks)
      }
    }

    return filePath
  })

  // เข้าถึงไฟล์ที่ bookmark ไว้
  ipcMain.handle('mas:access-bookmarked-file', async (_, { filePath }: { filePath: string }) => {
    const bookmarks = store.get('bookmarks', {})
    const bookmark = bookmarks[filePath]
    if (!bookmark) return false

    const success = resolveBookmark(bookmark)
    return success
  })
}

function createBookmark(filePath: string): Promise<string | null> {
  return new Promise((resolve) => {
    // ใน real MAS app ใช้ native addon หรือ Electron API
    // สำหรับ example นี้ return placeholder
    resolve(Buffer.from(filePath).toString('base64'))
  })
}

function resolveBookmark(bookmark: string): boolean {
  try {
    const filePath = Buffer.from(bookmark, 'base64').toString('utf-8')
    // เริ่ม accessing security-scoped resource
    app.startAccessingSecurityScopedResource?.(filePath)
    return true
  } catch {
    return false
  }
}
```

---

## 3. Microsoft Store (MSIX)

```json
// electron-builder.json5 - MSIX
{
  "win": {
    "target": [
      { "target": "nsis", "arch": ["x64"] },
      { "target": "appx", "arch": ["x64", "arm64"] }
    ]
  },
  "appx": {
    "applicationId": "MyCompany.MyApp",
    "backgroundColor": "#1e1e1e",
    "showNameOnTiles": true,
    "identityName": "MyCompany.MyApp",
    "publisher": "CN=MyCompany, O=MyCompany, L=Bangkok, S=Bangkok, C=TH",
    "publisherDisplayName": "My Company",
    "languages": ["th-TH", "en-US"],
    "addAutoLaunchExtension": false,
    "setBuildNumber": true
  }
}
```

```powershell
# สร้าง MSIX certificate สำหรับ testing (PowerShell)
# ต้องใช้ certificate จาก publisher ที่ trust แล้วสำหรับ production

$cert = New-SelfSignedCertificate `
  -Type CodeSigningCert `
  -Subject "CN=MyCompany, O=MyCompany, L=Bangkok, S=Bangkok, C=TH" `
  -KeyUsage DigitalSignature `
  -FriendlyName "MyApp Code Signing" `
  -CertStoreLocation "Cert:\CurrentUser\My" `
  -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.3", "2.5.29.19={text}")

# Export PFX
$password = ConvertTo-SecureString -String "password" -Force -AsPlainText
Export-PfxCertificate -Cert $cert -FilePath "build\cert.pfx" -Password $password
```

---

## 4. App Capabilities Declarations

```typescript
// src/main/capabilities.ts
// ตรวจสอบว่า app ทำงานได้ใน sandbox

export function checkSandboxCompatibility(): {
  issues: string[]
  warnings: string[]
} {
  const issues: string[] = []
  const warnings: string[] = []

  // ตรวจสอบ operations ที่ไม่รองรับใน sandbox
  const sandboxUnsafeOps = [
    { name: 'shell.openExternal', uses: checkShellOpenExternal },
    { name: 'child_process.exec', uses: checkChildProcess },
    { name: 'require native modules', uses: checkNativeModules }
  ]

  for (const op of sandboxUnsafeOps) {
    if (op.uses()) {
      warnings.push(`${op.name} requires sandbox exception`)
    }
  }

  return { issues, warnings }
}

function checkShellOpenExternal(): boolean {
  // ตรวจว่ามีการใช้ shell.openExternal หรือไม่
  return true  // placeholder
}

function checkChildProcess(): boolean {
  return false  // placeholder
}

function checkNativeModules(): boolean {
  // ตรวจ native modules ที่ต้องการ
  const natives = ['better-sqlite3', 'sharp', 'argon2']
  return natives.some(m => {
    try { require.resolve(m); return true } catch { return false }
  })
}
```

---

## 5. Store Submission Checklist

```typescript
// scripts/preSubmitCheck.ts
// ตรวจสอบก่อน submit ไปยัง Store

import * as fs from 'fs'
import * as path from 'path'

interface CheckResult {
  name: string
  passed: boolean
  message: string
}

async function runChecks(): Promise<void> {
  const checks: CheckResult[] = []

  // Package.json checks
  const pkg = JSON.parse(fs.readFileSync('package.json', 'utf-8'))

  checks.push({
    name: 'Version format',
    passed: /^\d+\.\d+\.\d+$/.test(pkg.version),
    message: `Version: ${pkg.version} (must be x.y.z)`
  })

  checks.push({
    name: 'App ID set',
    passed: !!pkg.build?.appId,
    message: `App ID: ${pkg.build?.appId || 'NOT SET'}`
  })

  checks.push({
    name: 'Category set (macOS)',
    passed: !!pkg.build?.mac?.category,
    message: `Category: ${pkg.build?.mac?.category || 'NOT SET'}`
  })

  // Icon checks
  const iconFiles = [
    { path: 'build/icon.icns', platform: 'macOS' },
    { path: 'build/icon.ico', platform: 'Windows' },
    { path: 'build/icons/512x512.png', platform: 'Linux' }
  ]

  for (const icon of iconFiles) {
    checks.push({
      name: `Icon exists (${icon.platform})`,
      passed: fs.existsSync(icon.path),
      message: icon.path
    })
  }

  // Privacy policy check
  checks.push({
    name: 'Privacy policy URL',
    passed: !!pkg.build?.extraMetadata?.privacyPolicyUrl,
    message: 'Required for App Store submission'
  })

  // Print results
  console.log('\n📋 Pre-submission Checklist\n')
  let allPassed = true

  for (const check of checks) {
    const icon = check.passed ? '✅' : '❌'
    console.log(`${icon} ${check.name}: ${check.message}`)
    if (!check.passed) allPassed = false
  }

  console.log(`\n${allPassed ? '✅ All checks passed!' : '❌ Some checks failed'}`)

  if (!allPassed) process.exit(1)
}

runChecks().catch(console.error)
```

---

## 6. macOS App Store Connect API

```typescript
// scripts/uploadToMAS.ts
// อัปโหลดไปยัง App Store Connect ด้วย altool หรือ notarytool

import { execSync } from 'child_process'

const BUNDLE_ID = process.env.BUNDLE_ID!
const APPLE_ID = process.env.APPLE_ID!
const APP_PASSWORD = process.env.APP_PASSWORD!

async function uploadToMAS(pkgPath: string): Promise<void> {
  console.log(`Uploading ${pkgPath} to App Store Connect...`)

  execSync(
    `xcrun altool --upload-app \
      --type macos \
      --file "${pkgPath}" \
      --bundle-id "${BUNDLE_ID}" \
      --username "${APPLE_ID}" \
      --password "${APP_PASSWORD}" \
      --verbose`,
    { stdio: 'inherit' }
  )

  console.log('Upload complete! Check App Store Connect for status.')
}

const pkg = process.argv[2]
if (!pkg) { console.error('Usage: ts-node uploadToMAS.ts <path-to.pkg>'); process.exit(1) }
uploadToMAS(pkg).catch(console.error)
```

---

## สรุป

| Platform | Format | Process | Requirements |
|---------|--------|---------|-------------|
| Mac App Store | .pkg (mas) | Xcode + altool | App Sandbox, signing |
| Microsoft Store | .appx/.msix | Partner Center | Publisher cert |
| Direct (macOS) | .dmg | Notarization | Developer ID |
| Direct (Win) | .exe (NSIS) | Code signing | EV cert (no SmartScreen) |
| Sandbox | N/A | Review | Limit filesystem/network |
| Bookmarks | NSData | Persist access | MAS requirement |
