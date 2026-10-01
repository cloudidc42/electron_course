# Part 002: Setup & Installation
## การติดตั้งและเตรียมสภาพแวดล้อม

---

## 🎯 เป้าหมายของบทเรียนนี้

หลังจากเรียนจบบทนี้ คุณจะสามารถ:
- ติดตั้ง Node.js และ npm
- ติดตั้ง Electron.js
- ตั้งค่า development environment
- สร้างโปรเจกต์ Electron ใหม่
- รันแอพ Electron ตัวแรกของคุณ

---

## 1. ความต้องการของระบบ

### System Requirements

```
Operating System:
├── Windows: Windows 10 หรือใหม่กว่า (64-bit)
├── macOS: macOS 10.13 (High Sierra) หรือใหม่กว่า
└── Linux: Ubuntu 18.04 หรือ distro ที่เทียบเท่า

Hardware:
├── RAM: 4 GB ขั้นต่ำ (8 GB แนะนำ)
├── Storage: 10 GB ว่าง
└── Processor: 64-bit processor

Software:
├── Node.js: v16.x หรือใหม่กว่า (แนะนำ LTS)
├── npm: v8.x หรือใหม่กว่า (มาพร้อม Node.js)
└── Git: สำหรับ version control
```

---

## 2. ติดตั้ง Node.js

### 2.1 ดาวน์โหลดจากเว็บไซต์ทางการ

1. ไปที่ https://nodejs.org
2. เลือก **LTS (Long Term Support)** version
3. ดาวน์โหลดและติดตั้งตาม OS ของคุณ

### 2.2 ใช้ NVM (Node Version Manager) - แนะนำ

NVM ช่วยให้จัดการ Node.js หลายเวอร์ชันได้ง่าย:

**macOS / Linux:**
```bash
# ติดตั้ง nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# reload terminal หรือรัน
source ~/.bashrc  # หรือ ~/.zshrc

# ติดตั้ง Node.js LTS
nvm install --lts

# ใช้ Node.js LTS
nvm use --lts

# ตรวจสอบเวอร์ชัน
node --version
npm --version
```

**Windows (nvm-windows):**
```powershell
# ดาวน์โหลด nvm-windows จาก GitHub
# https://github.com/coreybutler/nvm-windows/releases

# ติดตั้ง Node.js LTS
nvm install lts

# ใช้งาน
nvm use lts

# ตรวจสอบ
node --version
npm --version
```

### 2.3 ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Node.js version
node --version
# ควรได้ประมาณ: v20.11.0

# ตรวจสอบ npm version
npm --version
# ควรได้ประมาณ: 10.2.4

# ทดสอบ Node.js
node -e "console.log('Node.js is working!')"
# Output: Node.js is working!
```

---

## 3. ติดตั้ง Code Editor

### VS Code (แนะนำ)

1. ดาวน์โหลดจาก https://code.visualstudio.com
2. ติดตั้ง Extensions ที่แนะนำ:

```
Extensions ที่จำเป็น:
├── ESLint - ตรวจสอบ code quality
├── Prettier - จัดรูปแบบโค้ด
├── GitLens - Git integration
└── Auto Rename Tag - rename HTML tags

Extensions ที่มีประโยชน์สำหรับ Electron:
├── Electron Snippets - code snippets
├── Path Intellisense - autocomplete file paths
├── npm Intellisense - autocomplete npm modules
└── DotENV - .env file syntax highlighting
```

### ตั้งค่า VS Code สำหรับ Electron

```json
// .vscode/settings.json
{
  "editor.tabSize": 2,
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "files.exclude": {
    "node_modules": true,
    "dist": true,
    ".git": true
  },
  "search.exclude": {
    "node_modules": true,
    "dist": true
  },
  "javascript.preferences.quoteStyle": "single",
  "typescript.preferences.quoteStyle": "single"
}
```

---

## 4. สร้างโปรเจกต์ Electron แรก

### 4.1 สร้าง Directory และ package.json

```bash
# สร้าง folder โปรเจกต์
mkdir my-first-electron-app
cd my-first-electron-app

# สร้าง package.json
npm init -y
```

### 4.2 package.json ที่ได้

```json
{
  "name": "my-first-electron-app",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

### 4.3 แก้ไข package.json

```json
{
  "name": "my-first-electron-app",
  "version": "1.0.0",
  "description": "My first Electron application",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "dev": "electron . --inspect"
  },
  "keywords": ["electron", "desktop"],
  "author": "Your Name",
  "license": "MIT",
  "devDependencies": {
    "electron": "^28.0.0"
  }
}
```

### 4.4 ติดตั้ง Electron

```bash
# ติดตั้ง Electron เป็น dev dependency
npm install --save-dev electron

# หรือติดตั้งเวอร์ชันเฉพาะ
npm install --save-dev electron@28.0.0

# ตรวจสอบ
npx electron --version
# ควรได้ประมาณ: v28.0.0
```

---

## 5. สร้างไฟล์หลัก

### 5.1 โครงสร้างโปรเจกต์

```
my-first-electron-app/
├── main.js
├── preload.js
├── index.html
├── renderer.js
├── styles.css
└── package.json
```

### 5.2 main.js

```javascript
// main.js
const { app, BrowserWindow } = require('electron')
const path = require('path')

function createWindow() {
  // สร้าง browser window
  const mainWindow = new BrowserWindow({
    width: 900,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,   // ปลอดภัยกว่า
      contextIsolation: true    // แนะนำให้เปิดเสมอ
    }
  })

  // โหลด index.html
  mainWindow.loadFile('index.html')

  // เปิด DevTools (สำหรับ development)
  if (process.env.NODE_ENV === 'development') {
    mainWindow.webContents.openDevTools()
  }
}

// เมื่อ Electron พร้อมทำงาน
app.whenReady().then(() => {
  createWindow()

  // macOS: สร้าง window ใหม่เมื่อ click dock icon
  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow()
    }
  })
})

// ออกจากแอพเมื่อปิด window ทั้งหมด (Windows & Linux)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})
```

### 5.3 preload.js

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

// เปิดเผย APIs ให้ Renderer Process ใช้งาน
contextBridge.exposeInMainWorld('electronAPI', {
  // ดึงข้อมูล versions
  getVersions: () => ({
    node: process.versions.node,
    chrome: process.versions.chrome,
    electron: process.versions.electron,
    platform: process.platform
  }),
  
  // ส่งข้อความไป Main Process
  sendMessage: (channel, data) => {
    ipcRenderer.send(channel, data)
  },
  
  // รับข้อความจาก Main Process
  onMessage: (channel, callback) => {
    ipcRenderer.on(channel, (event, ...args) => callback(...args))
  }
})
```

### 5.4 index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <!-- Security: Content Security Policy -->
  <meta http-equiv="Content-Security-Policy" 
        content="default-src 'self'; script-src 'self'">
  <title>My First Electron App</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div class="container">
    <header>
      <h1>🖥️ My First Electron App</h1>
      <p class="subtitle">Built with Electron.js</p>
    </header>
    
    <main>
      <section class="info-card">
        <h2>📊 System Information</h2>
        <div class="info-grid">
          <div class="info-item">
            <span class="label">Electron:</span>
            <span id="electron-version" class="value">Loading...</span>
          </div>
          <div class="info-item">
            <span class="label">Node.js:</span>
            <span id="node-version" class="value">Loading...</span>
          </div>
          <div class="info-item">
            <span class="label">Chromium:</span>
            <span id="chrome-version" class="value">Loading...</span>
          </div>
          <div class="info-item">
            <span class="label">Platform:</span>
            <span id="platform" class="value">Loading...</span>
          </div>
        </div>
      </section>
      
      <section class="action-card">
        <h2>🎮 Actions</h2>
        <button id="send-btn" class="btn btn-primary">
          Send Message to Main Process
        </button>
        <div id="response" class="response-box">
          Response will appear here...
        </div>
      </section>
    </main>
    
    <footer>
      <p>Electron App - Version 1.0.0</p>
    </footer>
  </div>
  
  <script src="renderer.js"></script>
</body>
</html>
```

### 5.5 styles.css

```css
/* styles.css */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 
               Oxygen, Ubuntu, sans-serif;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
  color: #e0e0e0;
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
}

.container {
  width: 100%;
  max-width: 800px;
  padding: 20px;
}

header {
  text-align: center;
  margin-bottom: 30px;
}

header h1 {
  font-size: 2.5rem;
  color: #e94560;
  margin-bottom: 10px;
}

.subtitle {
  color: #888;
  font-size: 1rem;
}

.info-card, .action-card {
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 12px;
  padding: 24px;
  margin-bottom: 20px;
  backdrop-filter: blur(10px);
}

.info-card h2, .action-card h2 {
  color: #e94560;
  margin-bottom: 16px;
  font-size: 1.3rem;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
}

.info-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 14px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 8px;
}

.label {
  color: #888;
  font-size: 0.9rem;
}

.value {
  color: #4ade80;
  font-weight: bold;
  font-family: 'Courier New', monospace;
  font-size: 0.9rem;
}

.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.3s ease;
  margin: 8px 4px;
}

.btn-primary {
  background: #e94560;
  color: white;
}

.btn-primary:hover {
  background: #c73652;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(233, 69, 96, 0.4);
}

.response-box {
  margin-top: 16px;
  padding: 14px;
  background: rgba(0, 0, 0, 0.3);
  border-radius: 8px;
  border-left: 4px solid #e94560;
  font-family: 'Courier New', monospace;
  font-size: 0.9rem;
  color: #4ade80;
  min-height: 50px;
}

footer {
  text-align: center;
  margin-top: 30px;
  color: #555;
  font-size: 0.85rem;
}
```

### 5.6 renderer.js

```javascript
// renderer.js
document.addEventListener('DOMContentLoaded', async () => {
  // แสดงข้อมูล versions
  const versions = window.electronAPI.getVersions()
  
  document.getElementById('electron-version').textContent = `v${versions.electron}`
  document.getElementById('node-version').textContent = `v${versions.node}`
  document.getElementById('chrome-version').textContent = `v${versions.chrome}`
  document.getElementById('platform').textContent = versions.platform
  
  // ปุ่มส่งข้อความ
  const sendBtn = document.getElementById('send-btn')
  const responseBox = document.getElementById('response')
  
  sendBtn.addEventListener('click', () => {
    window.electronAPI.sendMessage('renderer-message', {
      text: 'Hello from Renderer!',
      timestamp: new Date().toISOString()
    })
  })
  
  // รับข้อความจาก Main Process
  window.electronAPI.onMessage('main-reply', (data) => {
    responseBox.textContent = `Main replied: ${data.message} (${data.timestamp})`
    responseBox.style.color = '#4ade80'
  })
})
```

### 5.7 อัพเดท main.js เพื่อรับและตอบกลับข้อความ

```javascript
// main.js (เพิ่มเติม)
const { app, BrowserWindow, ipcMain } = require('electron')
const path = require('path')

function createWindow() {
  const mainWindow = new BrowserWindow({
    width: 900,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })

  mainWindow.loadFile('index.html')
  
  // เปิด DevTools ใน development
  // mainWindow.webContents.openDevTools()
  
  return mainWindow
}

// รับข้อความจาก Renderer
ipcMain.on('renderer-message', (event, data) => {
  console.log('Message from Renderer:', data)
  
  // ตอบกลับ
  event.reply('main-reply', {
    message: 'Hello from Main Process!',
    timestamp: new Date().toISOString()
  })
})

app.whenReady().then(() => {
  createWindow()

  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow()
    }
  })
})

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})
```

---

## 6. รันแอพ

```bash
# รันแอพ
npm start

# หรือ
npx electron .
```

### สิ่งที่จะเห็น

เมื่อรันสำเร็จ จะเห็นหน้าต่างแสดง:
1. ชื่อแอพ "My First Electron App"
2. ข้อมูล versions ของ Electron, Node.js, Chromium
3. ปุ่มสำหรับส่งข้อความ

---

## 7. การ Debug

### 7.1 เปิด DevTools

```javascript
// เปิด DevTools อัตโนมัติ
mainWindow.webContents.openDevTools()

// เปิดแบบ detached (แยกหน้าต่าง)
mainWindow.webContents.openDevTools({ mode: 'detach' })
```

### 7.2 Keyboard Shortcuts สำหรับ DevTools

```
Ctrl+Shift+I (Windows/Linux) / Cmd+Option+I (macOS) : เปิด DevTools
F12 : เปิด DevTools
Ctrl+R / Cmd+R : Reload page
Ctrl+Shift+R : Hard reload (clear cache)
```

### 7.3 Main Process Debug

```bash
# รันด้วย --inspect flag
electron . --inspect

# รันด้วย port เฉพาะ
electron . --inspect=5858

# รันแล้วรอ debugger
electron . --inspect-brk
```

จากนั้นเปิด Chrome และไปที่ `chrome://inspect` เพื่อ debug Main Process

### 7.4 console.log ใน Main vs Renderer

```javascript
// Main Process: console.log แสดงใน terminal
console.log('This shows in terminal')

// Renderer Process: console.log แสดงใน DevTools
console.log('This shows in DevTools Console')
```

---

## 8. Electron Fiddle - ทดลองโค้ดออนไลน์

[Electron Fiddle](https://www.electronjs.org/fiddle) เป็นเครื่องมือสำหรับทดลองโค้ด Electron โดยไม่ต้องตั้งค่าอะไร

```
ประโยชน์ของ Electron Fiddle:
├── ทดลองโค้ดได้ทันที
├── แชร์โค้ดกับผู้อื่น
├── เรียนรู้จาก examples
└── Prototype features ก่อน implement จริง
```

---

## 9. .gitignore สำหรับ Electron

```gitignore
# .gitignore
# Dependencies
node_modules/

# Build outputs
dist/
out/

# Electron packages
*.dmg
*.exe
*.AppImage

# OS generated files
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# IDE files
.vscode/
.idea/
*.swp
*.swo

# Environment files
.env
.env.local

# Log files
*.log
npm-debug.log*

# Electron builder cache
.cache/
```

---

## 10. Scripts ที่มีประโยชน์

### อัพเดท package.json

```json
{
  "name": "my-first-electron-app",
  "version": "1.0.0",
  "description": "My first Electron application",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "dev": "NODE_ENV=development electron .",
    "dev:win": "set NODE_ENV=development && electron .",
    "debug": "electron . --inspect=5858",
    "lint": "eslint .",
    "format": "prettier --write ."
  },
  "devDependencies": {
    "electron": "^28.0.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0"
  }
}
```

---

## 11. ตรวจสอบโปรเจกต์

### Checklist

```
✅ Node.js ติดตั้งแล้ว (node --version)
✅ npm ติดตั้งแล้ว (npm --version)
✅ Electron ติดตั้งแล้ว (npx electron --version)
✅ มีไฟล์ main.js
✅ มีไฟล์ preload.js
✅ มีไฟล์ index.html
✅ มีไฟล์ renderer.js
✅ มีไฟล์ styles.css
✅ package.json มี "main": "main.js"
✅ package.json มี script "start": "electron ."
✅ รัน npm start แล้วเห็นหน้าต่างแอพ
```

---

## 12. ปัญหาที่พบบ่อยและวิธีแก้

### ปัญหา 1: Cannot find module 'electron'

```bash
# วิธีแก้:
npm install --save-dev electron
```

### ปัญหา 2: App.asar ไม่ได้รับอนุญาต (macOS)

```bash
# วิธีแก้:
sudo npm install --save-dev electron
# หรือ
sudo chown -R $(whoami) ~/.npm
```

### ปัญหา 3: ELECTRON_RUN_AS_NODE

```bash
# ถ้าเห็น warning นี้ ไม่ต้องตกใจ
# ใช้ได้ตามปกติ
```

### ปัญหา 4: DevTools ไม่เปิด

```javascript
// ตรวจสอบว่าเรียก openDevTools ใน ready event
app.whenReady().then(() => {
  const win = createWindow()
  win.webContents.openDevTools()  // ต้องอยู่หลัง createWindow()
})
```

### ปัญหา 5: Cannot read property of undefined

```javascript
// ตรวจสอบ contextIsolation และ preload
webPreferences: {
  preload: path.join(__dirname, 'preload.js'),  // ต้องระบุ preload
  contextIsolation: true  // ต้อง true สำหรับ contextBridge
}
```

---

## 13. สรุป

ในบทนี้เราได้ตั้งค่า:
1. **Node.js**: เครื่องมือหลักสำหรับพัฒนา
2. **VS Code**: Editor พร้อม Extensions
3. **Electron Project**: โครงสร้างไฟล์พื้นฐาน
4. **IPC Communication**: การสื่อสารระหว่าง processes

---

## 🔗 บทถัดไป

➡️ **[Part 003: First Electron App](part-003.md)** - สร้างแอพ Electron ตัวแรกอย่างละเอียด

---

*📅 อัพเดทล่าสุด: 2024 | ระดับ: Beginner | เวลาเรียน: ~60 นาที*
