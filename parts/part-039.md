# Part 039: Code Signing
## การ Sign Code สำหรับ Windows และ macOS

---

## 🎯 เป้าหมายของบทเรียนนี้

- Windows EV/OV Certificate
- Apple Developer ID
- Entitlements สำหรับ macOS
- Notarization (macOS)
- Code signing ใน CI/CD
- Environment variables สำหรับ signing
- ตรวจสอบ signature

---

## 1. ทำไมต้อง Code Sign?

Code signing ช่วย:
- แสดงให้ผู้ใช้รู้ว่าซอฟต์แวร์มาจากแหล่งที่เชื่อถือได้
- หลีกเลี่ยง Windows SmartScreen warning
- ผ่าน macOS Gatekeeper
- ป้องกัน tampering ของไฟล์

---

## 2. Windows Code Signing

### ประเภท Certificate

```
OV (Organization Validation)
├── ราคา: ~$200-400/ปี
├── ต้องผ่านการตรวจสอบองค์กร
├── ลด SmartScreen warnings (ต้องสะสม reputation)
└── เหมาะสำหรับ: บริษัทที่เพิ่งเริ่ม

EV (Extended Validation)
├── ราคา: ~$300-600/ปี
├── ต้องผ่านการตรวจสอบเข้มงวดกว่า
├── ผ่าน SmartScreen ทันที
├── ต้องใช้ Hardware token (USB)
└── เหมาะสำหรับ: บริษัทที่ต้องการความน่าเชื่อถือสูง
```

### Providers สำหรับซื้อ Certificate

- DigiCert
- Sectigo (เดิม Comodo)
- GlobalSign
- SSL.com

---

## 3. ตั้งค่า Windows Code Signing ใน electron-builder

### วิธีที่ 1: ใช้ CSC_LINK Environment Variable

```bash
# ตั้งค่า environment variables
export CSC_LINK=/path/to/certificate.pfx
export CSC_KEY_PASSWORD=your_certificate_password
```

### วิธีที่ 2: electron-builder.yml

```yaml
win:
  certificateFile: ${env.CSC_LINK}
  certificatePassword: ${env.CSC_KEY_PASSWORD}
  certificateSubjectName: "Your Company Name"
  
  # สำหรับ EV certificate (timestamp)
  timeStampServer: http://timestamp.digicert.com
  
  # SHA-256 signing (required for modern Windows)
  signingHashAlgorithms:
    - sha256
  
  # Sign อัตโนมัติ
  sign: true
  signDlls: false
```

### วิธีที่ 3: Custom signing function

```javascript
// electron-builder.js
module.exports = {
  win: {
    sign: async (configuration) => {
      // Custom sign function
      // ใช้ signtool หรือ third-party service
      
      const { path } = configuration;
      const { execSync } = require('child_process');
      
      // ใช้ signtool.exe (Windows SDK)
      const signtoolPath = 'C:\\Program Files (x86)\\Windows Kits\\10\\bin\\10.0.19041.0\\x64\\signtool.exe';
      
      execSync(`"${signtoolPath}" sign /f "${process.env.CSC_LINK}" /p "${process.env.CSC_KEY_PASSWORD}" /fd sha256 /tr http://timestamp.digicert.com /td sha256 "${path}"`);
    }
  }
};
```

---

## 4. macOS Code Signing

### ความต้องการ

1. Apple Developer Account ($99/ปี)
2. Developer ID Application certificate
3. Developer ID Installer certificate (สำหรับ .pkg)
4. macOS Sequoia หรือใหม่กว่าสำหรับ signing
5. Xcode Command Line Tools

### ติดตั้ง Certificate ใน Keychain

```bash
# ดาวน์โหลด certificate จาก developer.apple.com
# แล้ว double-click เพื่อติดตั้งใน Keychain

# ตรวจสอบ certificates ที่มี
security find-identity -v -p codesigning

# ผลลัพธ์ที่ควรได้:
# 1) ABC123... "Developer ID Application: Your Name (TEAMID)"
# 2) DEF456... "Apple Development: your@email.com (TEAMID)"
```

### electron-builder.yml สำหรับ macOS

```yaml
mac:
  # Identity สำหรับ signing
  identity: "Developer ID Application: Your Name (TEAMID)"
  
  # Hardened runtime (required for notarization)
  hardenedRuntime: true
  
  # Gatekeeper (skip ถ้าทำ notarization)
  gatekeeperAssess: false
  
  # Entitlements
  entitlements: build/entitlements.mac.plist
  entitlementsInherit: build/entitlements.mac.inherit.plist
  
  # Notarize หลัง sign
  notarize:
    teamId: YOUR_TEAM_ID

# Target
mac:
  target:
    - target: dmg
      arch: [x64, arm64]
    - target: zip
      arch: [x64, arm64]
```

---

## 5. macOS Notarization

Notarization คือกระบวนการส่งแอพให้ Apple ตรวจสอบ (automated) เพื่อให้ผ่าน Gatekeeper

### วิธีที่ 1: ใช้ electron-builder built-in notarize (v24+)

```yaml
# electron-builder.yml
mac:
  notarize:
    teamId: YOUR_TEAM_ID
```

```bash
# Environment variables ที่จำเป็น
export APPLE_ID=your@appleid.com
export APPLE_APP_SPECIFIC_PASSWORD=xxxx-xxxx-xxxx-xxxx
export APPLE_TEAM_ID=YOUR_TEAM_ID
```

### วิธีที่ 2: ใช้ @electron/notarize

```bash
npm install --save-dev @electron/notarize
```

```javascript
// build/notarize.js
const { notarize } = require('@electron/notarize');

async function notarizeApp(context) {
  const { electronPlatformName, appOutDir } = context;
  
  if (electronPlatformName !== 'darwin') {
    return;
  }
  
  if (!process.env.APPLE_ID || !process.env.APPLE_APP_SPECIFIC_PASSWORD) {
    console.log('Skipping notarization: Apple credentials not found');
    return;
  }
  
  const appName = context.packager.appInfo.productFilename;
  const appPath = `${appOutDir}/${appName}.app`;
  
  console.log(`Notarizing ${appPath}...`);
  
  await notarize({
    tool: 'notarytool',
    appPath,
    appleId: process.env.APPLE_ID,
    appleIdPassword: process.env.APPLE_APP_SPECIFIC_PASSWORD,
    teamId: process.env.APPLE_TEAM_ID,
  });
  
  console.log(`Notarization complete for ${appPath}`);
}

module.exports = notarizeApp;
```

```json
{
  "build": {
    "afterSign": "build/notarize.js"
  }
}
```

### สร้าง App-Specific Password

1. ไปที่ https://appleid.apple.com
2. Sign in → Security → App-Specific Passwords
3. Generate password สำหรับ "electron-notarize"
4. เก็บ password ไว้ใน environment variable

---

## 6. App-Specific Password ใน Keychain (safer)

```bash
# บันทึกใน Keychain แทน environment variable
xcrun notarytool store-credentials "AC_PASSWORD" \
  --apple-id your@email.com \
  --team-id YOUR_TEAM_ID \
  --password xxxx-xxxx-xxxx-xxxx

# ใช้ใน notarize
await notarize({
  tool: 'notarytool',
  appPath,
  keychainProfile: 'AC_PASSWORD',
});
```

---

## 7. Code Signing ใน CI/CD (GitHub Actions)

### .github/workflows/release.yml

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      # Decode certificate จาก base64
      - name: Decode certificate
        shell: powershell
        run: |
          $cert = [System.Convert]::FromBase64String("${{ secrets.CSC_LINK_BASE64 }}")
          [System.IO.File]::WriteAllBytes("$env:TEMP\cert.pfx", $cert)
          echo "CSC_LINK=$env:TEMP\cert.pfx" >> $env:GITHUB_ENV
      
      - name: Build Windows
        env:
          CSC_KEY_PASSWORD: ${{ secrets.CSC_KEY_PASSWORD }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npm run build:win
      
      - uses: actions/upload-artifact@v4
        with:
          name: windows-build
          path: dist/*.exe

  release-macos:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      # Import certificates
      - name: Import certificates
        uses: apple-actions/import-codesign-certs@v3
        with:
          p12-file-base64: ${{ secrets.MACOS_CERTIFICATE }}
          p12-password: ${{ secrets.MACOS_CERTIFICATE_PWD }}
      
      - name: Build macOS
        env:
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npm run build:mac
      
      - uses: actions/upload-artifact@v4
        with:
          name: macos-build
          path: dist/*.dmg

  release-linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Build Linux
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npm run build:linux
      
      - uses: actions/upload-artifact@v4
        with:
          name: linux-build
          path: |
            dist/*.AppImage
            dist/*.deb
```

---

## 8. ตั้งค่า GitHub Secrets

```bash
# แปลง certificate เป็น base64 สำหรับ GitHub Secrets

# Windows (.pfx)
base64 -i certificate.pfx | pbcopy  # macOS
# หรือ
[Convert]::ToBase64String([IO.File]::ReadAllBytes("certificate.pfx")) | Set-Clipboard  # PowerShell

# macOS (.p12)
base64 -i certificate.p12 | pbcopy
```

Secrets ที่ต้องตั้งใน GitHub:

```
CSC_LINK_BASE64          - Windows certificate (base64)
CSC_KEY_PASSWORD         - Windows certificate password
MACOS_CERTIFICATE        - macOS certificate (base64)
MACOS_CERTIFICATE_PWD    - macOS certificate password
APPLE_ID                 - Apple ID email
APPLE_APP_SPECIFIC_PASSWORD - App-specific password
APPLE_TEAM_ID            - Apple Team ID
```

---

## 9. ตรวจสอบ Signature

### Windows

```powershell
# ตรวจสอบ signature ด้วย signtool
signtool verify /pa /v "MyApp Setup.exe"

# หรือด้วย PowerShell
Get-AuthenticodeSignature "MyApp Setup.exe" | Format-List *
```

### macOS

```bash
# ตรวจสอบ code signature
codesign --verify --verbose=4 MyApp.app

# ตรวจสอบ notarization
spctl --assess --verbose=4 MyApp.app

# ดู entitlements
codesign -d --entitlements :- MyApp.app

# ตรวจสอบ DMG
spctl --assess --verbose=4 MyApp.dmg
```

---

## 10. Troubleshooting

### Windows SmartScreen ยัง warning

```
สาเหตุ: OV certificate ต้องสะสม reputation ก่อน
แก้ไข:
1. ใช้ EV certificate แทน (แพงกว่า แต่ผ่านทันที)
2. รอให้ผู้ใช้จำนวนมากพอ install ก่อน
3. ขอ report จาก Microsoft Security Intelligence Portal
```

### macOS Gatekeeper ปฏิเสธ

```bash
# ตรวจสอบ error
spctl --assess --verbose=4 MyApp.app
# ถ้าเห็น "rejected" หรือ "not notarized"

# ตรวจสอบ notarization log
xcrun notarytool log <submission-id> \
  --apple-id your@email.com \
  --team-id YOUR_TEAM_ID \
  --password xxxx-xxxx-xxxx-xxxx
```

### Certificate ไม่พบใน CI

```yaml
# ตรวจสอบว่า import ถูกต้อง
- name: Check certificate
  run: security find-identity -v -p codesigning
```

---

## 11. สรุป

| Platform | Tool | Requirements |
|----------|------|--------------|
| Windows OV | signtool | OV certificate (.pfx) |
| Windows EV | signtool | EV certificate + USB token |
| macOS | codesign | Apple Developer ID |
| macOS | notarytool | Apple ID + App-specific password |

### Checklist

- [ ] ซื้อ certificate จาก CA ที่น่าเชื่อถือ
- [ ] ตั้งค่า signing ใน electron-builder
- [ ] ตั้งค่า environment variables
- [ ] ทดสอบ signing ใน local ก่อน
- [ ] ตั้งค่า GitHub Secrets
- [ ] ทดสอบ CI build
- [ ] ตรวจสอบ signature ด้วย tools

---

*จบ Part 039 - ต่อไป Part 040: CI/CD Pipeline*
