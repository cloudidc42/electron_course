# Part 77: Advanced electron-builder

## การสร้าง Installer และ Package สำหรับทุก Platform

ในบทนี้เราจะเรียน NSIS (Windows), DMG (macOS), AppImage/deb/rpm (Linux), code signing และ multi-arch builds

---

## 1. electron-builder Configuration

```json
// electron-builder.json5
{
  "appId": "com.example.myapp",
  "productName": "My Electron App",
  "copyright": "Copyright © 2025 Example Corp",
  "directories": {
    "output": "dist",
    "buildResources": "build"
  },
  "files": [
    "out/**/*",
    "!**/*.{ts,tsx,map}",
    "!**/node_modules/**",
    "node_modules/**"
  ],
  "extraResources": [
    {
      "from": "resources/",
      "to": ".",
      "filter": ["**/*"]
    }
  ],
  "asar": true,
  "asarUnpack": [
    "node_modules/sharp/**",
    "node_modules/better-sqlite3/**",
    "resources/bin/**"
  ],

  // Windows (NSIS)
  "win": {
    "target": [
      { "target": "nsis", "arch": ["x64", "arm64"] },
      { "target": "portable", "arch": ["x64"] },
      { "target": "zip", "arch": ["x64", "arm64"] }
    ],
    "icon": "build/icon.ico",
    "signingHashAlgorithms": ["sha256"],
    "certificateFile": "${env.WIN_CERT_FILE}",
    "certificatePassword": "${env.WIN_CERT_PASSWORD}",
    "verifyUpdateCodeSignature": true
  },
  "nsis": {
    "oneClick": false,
    "allowToChangeInstallationDirectory": true,
    "allowElevation": true,
    "installerIcon": "build/installer.ico",
    "uninstallerIcon": "build/uninstaller.ico",
    "installerHeader": "build/header.bmp",
    "installerSidebar": "build/sidebar.bmp",
    "license": "LICENSE.txt",
    "createDesktopShortcut": true,
    "createStartMenuShortcut": true,
    "shortcutName": "My Electron App",
    "runAfterFinish": true,
    "deleteAppDataOnUninstall": false,
    "include": "build/installer.nsh"
  },

  // macOS (DMG)
  "mac": {
    "target": [
      { "target": "dmg", "arch": ["x64", "arm64", "universal"] },
      { "target": "zip", "arch": ["x64", "arm64", "universal"] }
    ],
    "icon": "build/icon.icns",
    "category": "public.app-category.developer-tools",
    "darkModeSupport": true,
    "hardenedRuntime": true,
    "gatekeeperAssess": false,
    "entitlements": "build/entitlements.mac.plist",
    "entitlementsInherit": "build/entitlements.mac.inherit.plist",
    "notarize": {
      "teamId": "${env.APPLE_TEAM_ID}"
    }
  },
  "dmg": {
    "title": "${productName} ${version}",
    "icon": "build/icon.icns",
    "iconSize": 80,
    "window": { "width": 600, "height": 400 },
    "contents": [
      { "x": 130, "y": 200, "type": "file" },
      { "x": 410, "y": 200, "type": "link", "path": "/Applications" }
    ],
    "background": "build/dmg-background.png"
  },

  // Linux
  "linux": {
    "target": [
      { "target": "AppImage", "arch": ["x64", "arm64"] },
      { "target": "deb", "arch": ["x64", "arm64"] },
      { "target": "rpm", "arch": ["x64"] },
      { "target": "tar.gz", "arch": ["x64"] }
    ],
    "icon": "build/icons",
    "category": "Development",
    "desktop": {
      "Name": "My Electron App",
      "Comment": "My awesome app",
      "Keywords": "electron;app"
    },
    "maintainer": "dev@example.com"
  },
  "deb": {
    "depends": ["libgtk-3-0", "libnotify4", "libnss3", "libxss1", "libxtst6", "xdg-utils"],
    "afterInstall": "build/scripts/after-install.sh",
    "afterRemove": "build/scripts/after-remove.sh"
  },
  "rpm": {
    "depends": ["gtk3", "libnotify", "nss", "libXScrnSaver", "libXtst", "xdg-utils"]
  },

  // Auto-update
  "publish": [
    {
      "provider": "github",
      "owner": "myorg",
      "repo": "myapp",
      "releaseType": "release"
    }
  ]
}
```

---

## 2. Custom NSIS Script

```nsis
; build/installer.nsh
; ส่วนเพิ่มเติมสำหรับ Windows installer

!macro customInstall
  ; ลงทะเบียน file associations
  WriteRegStr HKCU "Software\Classes\.myext" "" "MyApp.Document"
  WriteRegStr HKCU "Software\Classes\MyApp.Document" "" "My App Document"
  WriteRegStr HKCU "Software\Classes\MyApp.Document\shell\open\command" "" '"$INSTDIR\${APP_EXECUTABLE_FILENAME}" "%1"'

  ; เพิ่ม PATH
  ${EnvVarUpdate} $0 "PATH" "A" "HKCU" "$INSTDIR"

  ; สร้าง registry key สำหรับ autostart (optional)
  ; WriteRegStr HKCU "Software\Microsoft\Windows\CurrentVersion\Run" "${PRODUCT_NAME}" '"$INSTDIR\${APP_EXECUTABLE_FILENAME}"'
!macroend

!macro customUnInstall
  ; ลบ file associations
  DeleteRegKey HKCU "Software\Classes\.myext"
  DeleteRegKey HKCU "Software\Classes\MyApp.Document"
  
  ; ลบออกจาก PATH
  ${un.EnvVarUpdate} $0 "PATH" "R" "HKCU" "$INSTDIR"
!macroend
```

---

## 3. macOS Entitlements

```xml
<!-- build/entitlements.mac.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.security.cs.allow-jit</key>
  <true/>
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
  <true/>
  <key>com.apple.security.cs.disable-library-validation</key>
  <true/>
  <key>com.apple.security.automation.apple-events</key>
  <true/>
  <!-- สำหรับ network access -->
  <key>com.apple.security.network.client</key>
  <true/>
  <!-- สำหรับ file access -->
  <key>com.apple.security.files.user-selected.read-write</key>
  <true/>
  <key>com.apple.security.files.downloads.read-write</key>
  <true/>
  <!-- สำหรับ camera/microphone (ถ้าจำเป็น) -->
  <!-- <key>com.apple.security.device.camera</key><true/> -->
  <!-- <key>com.apple.security.device.microphone</key><true/> -->
</dict>
</plist>
```

---

## 4. Build Scripts

```json
// package.json scripts
{
  "scripts": {
    "build": "electron-vite build",
    "build:win": "npm run build && electron-builder --win",
    "build:mac": "npm run build && electron-builder --mac",
    "build:linux": "npm run build && electron-builder --linux",
    "build:all": "npm run build && electron-builder -wml",
    "build:publish": "npm run build && electron-builder -wml --publish always",
    "build:win:arm64": "npm run build && electron-builder --win --arm64",
    "build:mac:universal": "npm run build && electron-builder --mac --universal"
  }
}
```

```typescript
// scripts/build.ts - Custom build script
import { execSync } from 'child_process'
import * as fs from 'fs'
import * as path from 'path'

async function build() {
  const platform = process.platform
  const version = JSON.parse(fs.readFileSync('package.json', 'utf-8')).version

  console.log(`Building v${version} for ${platform}...`)

  // Compile TypeScript
  execSync('npm run build', { stdio: 'inherit' })

  // Platform-specific build
  const targets: Record<string, string> = {
    win32: '--win',
    darwin: '--mac',
    linux: '--linux'
  }

  const target = targets[platform] || '--linux'
  execSync(`electron-builder ${target}`, { stdio: 'inherit', env: {
    ...process.env,
    CSC_IDENTITY_AUTO_DISCOVERY: 'false',  // ปิด code signing ใน dev
  }})

  console.log(`Build complete: dist/`)
  listOutput()
}

function listOutput() {
  const distDir = path.join(process.cwd(), 'dist')
  if (!fs.existsSync(distDir)) return

  const files = fs.readdirSync(distDir)
    .filter(f => !f.startsWith('win-') && !f.startsWith('mac-') && !f.startsWith('linux-'))

  console.log('\nOutput files:')
  for (const file of files) {
    const filePath = path.join(distDir, file)
    const stat = fs.statSync(filePath)
    if (stat.isFile()) {
      const sizeKB = (stat.size / 1024).toFixed(0)
      console.log(`  ${file} (${sizeKB} KB)`)
    }
  }
}

build().catch(console.error)
```

---

## 5. GitHub Actions CI/CD

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build-and-release:
    strategy:
      matrix:
        include:
          - os: windows-latest
            platform: win
          - os: macos-latest
            platform: mac
          - os: ubuntu-latest
            platform: linux

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Build & Release (Windows)
        if: matrix.platform == 'win'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          WIN_CERT_FILE: ${{ secrets.WIN_CERT_FILE }}
          WIN_CERT_PASSWORD: ${{ secrets.WIN_CERT_PASSWORD }}
        run: electron-builder --win --publish always

      - name: Build & Release (macOS)
        if: matrix.platform == 'mac'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CSC_LINK: ${{ secrets.MAC_CERTS }}
          CSC_KEY_PASSWORD: ${{ secrets.MAC_CERTS_PASSWORD }}
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_PASSWORD }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
        run: electron-builder --mac --publish always

      - name: Build & Release (Linux)
        if: matrix.platform == 'linux'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: electron-builder --linux --publish always
```

---

## 6. Icon Generation Script

```typescript
// scripts/generateIcons.ts
// ใช้ sharp สร้าง icons จาก SVG/PNG source
import sharp from 'sharp'
import * as path from 'path'
import * as fs from 'fs'

const SOURCE = 'assets/icon.png'
const BUILD_DIR = 'build'

async function generateIcons() {
  fs.mkdirSync(path.join(BUILD_DIR, 'icons'), { recursive: true })

  // macOS: 512x512 .icns (ใช้ iconutil ใน macOS)
  await sharp(SOURCE).resize(512, 512).toFile(path.join(BUILD_DIR, 'icon.png'))

  // Windows: .ico (หลายขนาด)
  const winSizes = [16, 24, 32, 48, 64, 128, 256]
  for (const size of winSizes) {
    await sharp(SOURCE)
      .resize(size, size)
      .toFile(path.join(BUILD_DIR, `icons/${size}x${size}.png`))
  }

  // Linux: ทุกขนาด
  const linuxSizes = [16, 32, 48, 64, 128, 256, 512, 1024]
  for (const size of linuxSizes) {
    await sharp(SOURCE)
      .resize(size, size)
      .toFile(path.join(BUILD_DIR, `icons/${size}x${size}.png`))
  }

  console.log('Icons generated!')
}

generateIcons().catch(console.error)
```

---

## สรุป

| Platform | Format | เหมาะสำหรับ |
|---------|--------|-----------|
| Windows | NSIS | Full installer พร้อม wizard |
| Windows | Portable | ไม่ต้องติดตั้ง |
| macOS | DMG | drag-and-drop install |
| macOS | Universal | Apple Silicon + Intel |
| Linux | AppImage | portable, ทุก distro |
| Linux | deb | Ubuntu/Debian |
| Linux | rpm | Fedora/RHEL |
| Code Signing | ทุก platform | ป้องกัน security warning |
