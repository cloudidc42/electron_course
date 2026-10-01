# Part 038: electron-builder Packaging
## การ Package แอพ Electron พร้อม Distribute

---

## 🎯 เป้าหมายของบทเรียนนี้

- ตั้งค่า electron-builder
- Build targets ต่างๆ (NSIS, DMG, AppImage, deb, rpm)
- Code signing
- จัดการ icons และ assets
- extraResources และ extraFiles
- Portable build
- App metadata

---

## 1. ติดตั้ง electron-builder

```bash
npm install --save-dev electron-builder
```

---

## 2. ตั้งค่าใน package.json

```json
{
  "name": "my-electron-app",
  "version": "1.0.0",
  "description": "My cross-platform desktop application",
  "main": "src/main/index.js",
  "author": {
    "name": "Your Name",
    "email": "your@email.com",
    "url": "https://yourwebsite.com"
  },
  "license": "MIT",
  
  "scripts": {
    "build": "electron-builder",
    "build:win": "electron-builder --win",
    "build:mac": "electron-builder --mac",
    "build:linux": "electron-builder --linux",
    "build:all": "electron-builder -mwl",
    "dist": "npm run build",
    "pack": "electron-builder --dir"
  },
  
  "build": {
    "appId": "com.yourcompany.myapp",
    "productName": "My Electron App",
    "copyright": "Copyright © 2024 Your Name",
    
    "directories": {
      "output": "dist",
      "buildResources": "build"
    },
    
    "files": [
      "src/**/*",
      "node_modules/**/*",
      "!node_modules/*/{CHANGELOG.md,README.md,README,readme.md,readme}",
      "!node_modules/*/{test,__tests__,tests,powered-test,example,examples}",
      "!node_modules/.bin",
      "!**/*.{iml,o,hprof,orig,pyc,pyo,rbc,swp,csproj,sln,xproj}",
      "!.editorconfig",
      "!**/._*",
      "!**/{.DS_Store,.git,.hg,.svn,CVS,RCS,SCCS,.gitignore,.gitattributes}",
      "!**/{__pycache__,thumbs.db,.flowconfig,.idea,.vs,.nyc_output}",
      "!**/{appveyor.yml,.travis.yml,circle.yml}",
      "!**/{npm-debug.log,yarn.lock,.yarn-integrity,.yarn-metadata.json}"
    ],
    
    "extraResources": [
      {
        "from": "resources/",
        "to": "resources/",
        "filter": ["**/*"]
      }
    ],
    
    "extraFiles": [
      {
        "from": "config/",
        "to": "config/",
        "filter": ["*.json"]
      }
    ]
  }
}
```

---

## 3. ไฟล์ electron-builder.yml (แบบ YAML)

```yaml
appId: com.yourcompany.myapp
productName: "My Electron App"
copyright: "Copyright © 2024 Your Name"

directories:
  output: dist
  buildResources: build

# ไฟล์ที่จะ include ใน build
files:
  - src/**/*
  - "!src/**/*.test.js"
  - "!src/**/*.spec.js"

# ไฟล์ extra ที่จะ copy ไปใน resources
extraResources:
  - from: locales/
    to: locales/
    filter:
      - "**/*.json"
  - from: assets/sounds/
    to: sounds/

# Windows Build
win:
  target:
    - target: nsis
      arch: [x64, ia32]
    - target: portable
      arch: [x64]
    - target: zip
      arch: [x64]
  
  icon: build/icons/icon.ico
  
  # Code signing
  certificateFile: ${env.CSC_LINK}
  certificatePassword: ${env.CSC_KEY_PASSWORD}
  
  # Request UAC elevation
  requestedExecutionLevel: asInvoker
  
  verifyUpdateCodeSignature: true

# NSIS Installer
nsis:
  oneClick: false
  allowToChangeInstallationDirectory: true
  createDesktopShortcut: always
  createStartMenuShortcut: true
  shortcutName: "My Electron App"
  
  # Custom pages
  include: build/installer.nsh
  
  # ข้อมูล installer
  installerIcon: build/icons/installer.ico
  uninstallerIcon: build/icons/uninstaller.ico
  installerHeaderIcon: build/icons/installer-header.ico
  
  license: LICENSE.txt
  
  # Language
  language: 1033  # English
  
  deleteAppDataOnUninstall: false

# macOS Build
mac:
  target:
    - target: dmg
      arch: [x64, arm64]
    - target: zip
      arch: [x64, arm64]
  
  icon: build/icons/icon.icns
  category: public.app-category.productivity
  
  # Code signing
  identity: "Developer ID Application: Your Name (TEAMID)"
  
  # Notarization
  notarize:
    teamId: YOUR_TEAM_ID
  
  # Entitlements
  entitlements: build/entitlements.mac.plist
  entitlementsInherit: build/entitlements.mac.inherit.plist
  
  hardenedRuntime: true
  gatekeeperAssess: false

# DMG Settings
dmg:
  contents:
    - x: 130
      y: 220
    - x: 410
      y: 220
      type: link
      path: /Applications
  
  window:
    x: 130
    y: 150
    width: 540
    height: 380
  
  background: build/dmg-background.png
  icon: build/icons/icon.icns
  iconSize: 128
  iconTextSize: 12

# Linux Build
linux:
  target:
    - target: AppImage
      arch: [x64]
    - target: deb
      arch: [x64]
    - target: rpm
      arch: [x64]
  
  icon: build/icons/
  category: Utility
  
  # Desktop integration
  desktop:
    Name: My Electron App
    GenericName: Desktop Application
    Comment: A powerful desktop application
    Keywords: electron;desktop;

# Debian Package
deb:
  depends:
    - libgtk-3-0
    - libnotify4
    - libnss3
    - libxss1
    - libxtst6
    - xdg-utils
    - libatspi2.0-0
    - libsecret-1-0

# RPM Package
rpm:
  depends:
    - gtk3
    - libnotify
    - nss
    - libXScrnSaver
    - libXtst
    - xdg-utils
    - at-spi2-core
    - libsecret

# AppImage
appImage:
  artifactName: "${productName}-${version}.AppImage"

# Auto update
publish:
  - provider: github
    owner: yourusername
    repo: my-electron-app
    private: false
  - provider: s3
    bucket: my-app-releases
    region: us-east-1
```

---

## 4. Icons และ Assets

### โครงสร้าง build/ directory

```
build/
├── icons/
│   ├── icon.ico          # Windows (256x256, multi-size)
│   ├── icon.icns         # macOS
│   ├── icon.png          # Linux (512x512)
│   ├── 16x16.png
│   ├── 32x32.png
│   ├── 48x48.png
│   ├── 64x64.png
│   ├── 128x128.png
│   ├── 256x256.png
│   └── 512x512.png
├── entitlements.mac.plist
├── entitlements.mac.inherit.plist
├── installer.nsh         # Custom NSIS script
├── dmg-background.png    # DMG background (540x380)
└── LICENSE.txt
```

### สร้าง Icons ด้วย electron-icon-maker

```bash
npm install --save-dev electron-icon-maker

# สร้างจาก PNG ขนาด 1024x1024
npx electron-icon-maker --input=icon-source.png --output=build/icons
```

### Script สร้าง icons อัตโนมัติ

```javascript
// scripts/generate-icons.js
const sharp = require('sharp');
const path = require('path');
const fs = require('fs');

const SIZES = [16, 24, 32, 48, 64, 128, 256, 512, 1024];
const INPUT = path.join(__dirname, '../assets/icon-source.png');
const OUTPUT_DIR = path.join(__dirname, '../build/icons');

async function generateIcons() {
  // สร้าง directory
  fs.mkdirSync(OUTPUT_DIR, { recursive: true });
  
  // สร้าง PNG หลายขนาด
  for (const size of SIZES) {
    await sharp(INPUT)
      .resize(size, size)
      .png()
      .toFile(path.join(OUTPUT_DIR, `${size}x${size}.png`));
    
    console.log(`Generated ${size}x${size}.png`);
  }
  
  console.log('Icons generated successfully!');
}

generateIcons().catch(console.error);
```

---

## 5. Entitlements (macOS)

### build/entitlements.mac.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- Hardened Runtime -->
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
  <false/>
  
  <!-- JIT - needed for some JavaScript engines -->
  <key>com.apple.security.cs.allow-jit</key>
  <true/>
  
  <!-- App Sandbox (disable for most Electron apps) -->
  <key>com.apple.security.app-sandbox</key>
  <false/>
  
  <!-- Network access -->
  <key>com.apple.security.network.client</key>
  <true/>
  <key>com.apple.security.network.server</key>
  <false/>
  
  <!-- File access -->
  <key>com.apple.security.files.user-selected.read-write</key>
  <true/>
  <key>com.apple.security.files.downloads.read-write</key>
  <true/>
  
  <!-- Device access -->
  <key>com.apple.security.device.camera</key>
  <false/>
  <key>com.apple.security.device.microphone</key>
  <false/>
</dict>
</plist>
```

### build/entitlements.mac.inherit.plist

```xml
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
</dict>
</plist>
```

---

## 6. Custom NSIS Script

### build/installer.nsh

```nsis
; Custom NSIS Script
; เพิ่ม custom pages หรือ actions

!macro customInstall
  ; ทำ custom action ตอน install
  WriteRegStr HKCU "Software\MyApp" "InstallDate" "$YEAR-$MONTH-$DAY"
  WriteRegStr HKCU "Software\MyApp" "Version" "${VERSION}"
!macroend

!macro customUninstall
  ; ทำ custom action ตอน uninstall
  DeleteRegKey HKCU "Software\MyApp"
  
  ; ลบ user data ถ้าต้องการ
  ; RMDir /r "$APPDATA\MyApp"
!macroend

!macro customHeader
  ; Custom header
  !define MUI_HEADERIMAGE
  !define MUI_HEADERIMAGE_BITMAP "${NSISDIR}\Contrib\Graphics\Header\win.bmp"
!macroend
```

---

## 7. การจัดการ Native Modules

```javascript
// package.json - rebuild native modules
{
  "build": {
    "npmRebuild": true,
    "buildDependenciesFromSource": false,
    
    "nativeRebuilder": "legacy",
    
    "extraMetadata": {
      "main": "src/main/index.js"
    }
  }
}
```

### ตั้งค่าสำหรับ native modules

```bash
# ติดตั้ง electron-rebuild
npm install --save-dev @electron/rebuild

# Rebuild สำหรับ Electron version ปัจจุบัน
npx electron-rebuild

# หรือ rebuild specific module
npx electron-rebuild -m better-sqlite3
```

---

## 8. Portable Build

```json
{
  "build": {
    "portable": {
      "artifactName": "${productName} Portable ${version}.exe",
      "requestExecutionLevel": "user",
      "unpackDirName": "my-app-portable"
    }
  }
}
```

---

## 9. App Metadata

```javascript
// src/main/index.js
const { app } = require('electron');

// ตั้งค่า app metadata
app.setAppUserModelId('com.yourcompany.myapp');

// สำหรับ Windows
if (process.platform === 'win32') {
  app.setAppUserModelId(process.execPath);
}

// ดึง app info
console.log('App name:', app.getName());
console.log('App version:', app.getVersion());
console.log('Electron version:', process.versions.electron);
console.log('Node version:', process.versions.node);
console.log('Chrome version:', process.versions.chrome);
```

---

## 10. Build Scripts

### scripts/build.js

```javascript
const builder = require('electron-builder');
const path = require('path');

async function build(platform) {
  const config = {
    config: {
      appId: 'com.yourcompany.myapp',
      productName: 'My Electron App',
      directories: {
        output: 'dist',
      },
    },
  };

  switch (platform) {
    case 'win':
      config.win = ['nsis', 'portable'];
      break;
    case 'mac':
      config.mac = ['dmg', 'zip'];
      break;
    case 'linux':
      config.linux = ['AppImage', 'deb'];
      break;
    default:
      // Build ทุก platform
      config.win = ['nsis'];
      config.mac = ['dmg'];
      config.linux = ['AppImage'];
  }

  try {
    const result = await builder.build(config);
    console.log('Build successful!');
    console.log('Output files:', result);
  } catch (error) {
    console.error('Build failed:', error);
    process.exit(1);
  }
}

const platform = process.argv[2];
build(platform);
```

---

## 11. สรุป

ใน electron-builder:

| Target | Platform | Format |
|--------|----------|--------|
| NSIS | Windows | .exe installer |
| portable | Windows | portable .exe |
| DMG | macOS | .dmg disk image |
| AppImage | Linux | .AppImage |
| deb | Linux | .deb package |
| rpm | Linux | .rpm package |

### Best Practices

1. **Icons**: ใช้ PNG ขนาด 1024x1024 เป็น source
2. **Files filter**: exclude test files, docs ออกจาก build
3. **extraResources**: ใส่ไฟล์ที่ต้องการใน resources folder
4. **Native modules**: ใช้ `@electron/rebuild` สำหรับ native modules
5. **Code signing**: ทำ code signing ก่อน distribute เสมอ

---

*จบ Part 038 - ต่อไป Part 039: Code Signing*
