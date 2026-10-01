# Part 001: Introduction to Electron.js
## ทำความรู้จักกับ Electron.js

---

## 🎯 เป้าหมายของบทเรียนนี้

หลังจากเรียนจบบทนี้ คุณจะเข้าใจ:
- Electron.js คืออะไร และทำงานอย่างไร
- ทำไมถึงต้องใช้ Electron.js
- สถาปัตยกรรมของ Electron.js
- แอพพลิเคชันที่สร้างด้วย Electron.js ที่มีชื่อเสียง
- ข้อดีและข้อเสียของ Electron.js

---

## 1. Electron.js คืออะไร?

**Electron.js** เป็น framework แบบ open-source ที่พัฒนาโดย GitHub (ปัจจุบันอยู่ภายใต้ Microsoft) ที่ช่วยให้คุณสร้างแอพพลิเคชัน Desktop แบบ cross-platform ได้โดยใช้เทคโนโลยีเว็บ ได้แก่ **JavaScript**, **HTML**, และ **CSS**

```
Electron = Chromium + Node.js
```

### ความหมายของสมการนี้:
- **Chromium**: เป็น browser engine ที่ใช้แสดง UI ของแอพ (เหมือนกับที่ Chrome ใช้)
- **Node.js**: ทำให้แอพสามารถเข้าถึง file system, network, และ OS APIs ได้

---

## 2. ทำไมถึงใช้ Electron.js?

### ปัญหาก่อนมี Electron
ก่อนมี Electron นักพัฒนาเว็บที่ต้องการสร้างแอพ Desktop ต้องเรียนรู้ภาษาใหม่:
- **Windows**: C#, C++, VB.NET
- **macOS**: Swift, Objective-C
- **Linux**: C, C++, Python + GTK

ซึ่งหมายความว่าต้องเขียนโค้ดแยกกัน 3 ชุด สำหรับแต่ละ platform!

### วิธีแก้ปัญหาของ Electron

```
เขียนโค้ดครั้งเดียว → รันได้บน Windows, macOS, Linux
```

### ข้อได้เปรียบหลัก

1. **Cross-Platform**: โค้ดชุดเดียวทำงานได้บนทุก OS
2. **ใช้ความรู้เว็บที่มีอยู่**: HTML, CSS, JavaScript
3. **Ecosystem ของ npm**: ใช้ package ได้กว่า 1 ล้านตัว
4. **Community ใหญ่**: มีเอกสารและตัวอย่างมากมาย
5. **Native Integration**: เข้าถึง system APIs ได้
6. **Hot Reload**: พัฒนาได้รวดเร็ว
7. **DevTools**: ใช้ Chrome DevTools ได้ทันที

---

## 3. สถาปัตยกรรมของ Electron.js

### 3.1 ภาพรวม

```
┌─────────────────────────────────────────────────────────┐
│                    Electron Application                  │
│                                                         │
│  ┌─────────────────┐    IPC    ┌─────────────────────┐  │
│  │   Main Process   │◄────────►│  Renderer Process   │  │
│  │   (Node.js)      │          │    (Chromium)        │  │
│  │                 │          │                     │  │
│  │  - app          │          │  - window.document  │  │
│  │  - BrowserWindow│          │  - window.location  │  │
│  │  - Menu         │          │  - DOM APIs         │  │
│  │  - Tray         │          │  - Web APIs         │  │
│  │  - Dialog       │          │  - JavaScript       │  │
│  │  - ipcMain      │          │  - ipcRenderer      │  │
│  └─────────────────┘          └─────────────────────┘  │
│           │                                             │
│           ▼                                             │
│  ┌─────────────────┐                                    │
│  │    Node.js APIs  │                                    │
│  │  - fs           │                                    │
│  │  - path         │                                    │
│  │  - os           │                                    │
│  │  - net          │                                    │
│  └─────────────────┘                                    │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Main Process (กระบวนการหลัก)

Main Process คือ "หัวใจ" ของแอพ Electron:

```javascript
// main.js - ตัวอย่าง Main Process
const { app, BrowserWindow } = require('electron')

// สร้าง window เมื่อแอพพร้อม
app.whenReady().then(() => {
  const win = new BrowserWindow({
    width: 800,
    height: 600
  })
  
  win.loadFile('index.html')
})

// ออกจากแอพเมื่อปิด windows ทั้งหมด (Windows & Linux)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})
```

**หน้าที่ของ Main Process:**
- สร้างและจัดการ BrowserWindow
- ควบคุม lifecycle ของแอพ
- เข้าถึง OS และ native APIs
- จัดการ system tray, menus, notifications
- สื่อสารกับ Renderer Process ผ่าน IPC

### 3.3 Renderer Process (กระบวนการแสดงผล)

Renderer Process ทำงานใน Chromium window:

```javascript
// renderer.js - ตัวอย่าง Renderer Process
document.addEventListener('DOMContentLoaded', () => {
  const btn = document.getElementById('myBtn')
  
  btn.addEventListener('click', () => {
    // ส่งข้อความไปยัง Main Process
    window.electronAPI.sendMessage('hello from renderer')
  })
})
```

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <title>My Electron App</title>
  <meta charset="UTF-8">
</head>
<body>
  <h1>Hello from Electron!</h1>
  <button id="myBtn">Click Me</button>
  <script src="renderer.js"></script>
</body>
</html>
```

**หน้าที่ของ Renderer Process:**
- แสดง UI ผ่าน HTML/CSS
- รัน JavaScript ใน browser context
- สื่อสารกับ Main Process ผ่าน IPC
- จัดการ user interactions

### 3.4 Preload Script

Preload Script เป็นสะพานเชื่อมระหว่าง Main และ Renderer:

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

// เปิดเผย API ที่ปลอดภัยให้ Renderer ใช้งาน
contextBridge.exposeInMainWorld('electronAPI', {
  sendMessage: (msg) => ipcRenderer.send('message', msg),
  onReply: (callback) => ipcRenderer.on('reply', callback)
})
```

---

## 4. แอพที่สร้างด้วย Electron.js

### 4.1 แอพระดับโลก

| แอพ | บริษัท | ประเภท |
|-----|--------|--------|
| **VS Code** | Microsoft | Code Editor |
| **Slack** | Salesforce | Communication |
| **Discord** | Discord Inc. | Gaming/Chat |
| **Figma** (Desktop) | Adobe | Design |
| **Notion** | Notion Labs | Productivity |
| **WhatsApp** (Desktop) | Meta | Messaging |
| **GitHub Desktop** | GitHub | Developer Tool |
| **Atom** | GitHub | Code Editor |
| **Postman** | Postman Inc. | API Testing |
| **Twitch** (Desktop) | Amazon | Streaming |

### 4.2 ตัวเลขที่น่าสนใจ

```
VS Code:        ~36 ล้าน users/เดือน
Slack:          ~20 ล้าน users/วัน  
Discord:        ~19 ล้าน users/วัน
WhatsApp:       ~2 พันล้าน users
```

---

## 5. ข้อดีและข้อเสียของ Electron.js

### ✅ ข้อดี

```
1. Cross-Platform Development
   - เขียนครั้งเดียว รันได้ทุก OS
   - ลดต้นทุนการพัฒนา
   
2. Web Technologies
   - ใช้ HTML, CSS, JavaScript ที่รู้จักอยู่แล้ว
   - ใช้ npm packages ได้
   - มี CSS frameworks มากมาย
   
3. Rapid Development
   - Hot reload
   - Browser DevTools
   - เอกสารดี
   
4. Native Features
   - File system access
   - System notifications
   - Tray icons
   - Custom protocols
   
5. Large Community
   - GitHub Stars: ~115,000+
   - npm downloads: หลายล้านครั้ง/สัปดาห์
```

### ❌ ข้อเสีย

```
1. ขนาดไฟล์ใหญ่
   - แอพเริ่มต้นมักมีขนาด 50-150 MB
   - เพราะรวม Chromium และ Node.js ไว้ด้วย
   
2. การใช้ RAM สูง
   - ใช้ RAM มากกว่าแอพ native ปกติ
   - VS Code ใช้ RAM ~300-500 MB
   
3. Performance
   - ช้ากว่า native apps
   - Not suitable for CPU-intensive tasks
   
4. Security
   - ต้องระวัง XSS และ Remote Code Execution
   - ต้องตั้งค่า Security ให้ถูกต้อง
```

---

## 6. Electron เทียบกับทางเลือกอื่น

### เปรียบเทียบ Frameworks

| Feature | Electron | Tauri | NW.js | Flutter |
|---------|---------|-------|-------|---------|
| Language | JS/TS | JS/Rust | JS | Dart |
| Size | ~150MB | ~5MB | ~100MB | ~30MB |
| RAM Usage | สูง | ต่ำ | สูง | ปานกลาง |
| Native Feel | ปานกลาง | ดี | ปานกลาง | ดี |
| Maturity | สูง | กลาง | กลาง | สูง |
| Community | ใหญ่มาก | กำลังโต | กลาง | ใหญ่ |

### เมื่อไหร่ควรใช้ Electron?

```
✅ เหมาะกับ:
- Internal business tools
- Developer tools
- Creative/productivity apps
- Apps ที่ต้องการ web technologies
- Teams ที่มีความรู้ web development

❌ ไม่เหมาะกับ:
- Mobile apps (ใช้ React Native หรือ Flutter)
- Games ที่ต้องการ high performance
- Simple utilities ที่ต้องการ small footprint
- Apps ที่ต้องการ pixel-perfect native look
```

---

## 7. เวอร์ชันและ Versioning

### Electron Version Policy

```
Electron ใช้ Semantic Versioning (SemVer):
v{MAJOR}.{MINOR}.{PATCH}

ตัวอย่าง: v28.0.0
- MAJOR: เปลี่ยนแปลงใหญ่ (breaking changes)
- MINOR: feature ใหม่ (backward compatible)
- PATCH: bug fixes
```

### Chromium และ Node.js Versions

```
Electron 28 (2023):
├── Chromium: 120
├── Node.js: 18.18.2
└── V8: 12.0

Electron 29 (2024):
├── Chromium: 122
├── Node.js: 20.9.0
└── V8: 12.2

Electron 30 (2024):
├── Chromium: 124
├── Node.js: 20.11.1
└── V8: 12.4
```

### ตรวจสอบเวอร์ชัน

```javascript
// ตรวจสอบเวอร์ชันใน Main Process
console.log('Electron:', process.versions.electron)
console.log('Node.js:', process.versions.node)
console.log('Chromium:', process.versions.chrome)
console.log('V8:', process.versions.v8)
console.log('OS:', process.platform)
```

---

## 8. โครงสร้างพื้นฐานของโปรเจกต์ Electron

```
my-electron-app/
├── package.json          # โปรเจกต์ config
├── main.js               # Main Process entry point
├── preload.js            # Preload script
├── renderer/
│   ├── index.html        # Main window
│   ├── styles.css        # CSS styles
│   └── renderer.js       # Renderer scripts
├── assets/
│   ├── icons/            # App icons
│   └── images/           # Images
└── node_modules/         # Dependencies
```

### package.json พื้นฐาน

```json
{
  "name": "my-electron-app",
  "version": "1.0.0",
  "description": "My first Electron app",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "build": "electron-builder"
  },
  "keywords": ["electron"],
  "author": "Your Name",
  "license": "MIT",
  "devDependencies": {
    "electron": "^28.0.0"
  }
}
```

---

## 9. Electron Lifecycle

### App Lifecycle Events

```
แอพเริ่มต้น
     │
     ▼
app.on('will-finish-launching')
     │
     ▼
app.on('ready') / app.whenReady()
     │
     ▼
สร้าง BrowserWindow
     │
     ▼
โหลด HTML
     │
     ▼
แสดง Window
     │
     ▼
app.on('activate')          [macOS only]
     │
     ▼
app.on('before-quit')
     │
     ▼
app.on('will-quit')
     │
     ▼
app.on('quit')
```

### ตัวอย่าง Lifecycle ครบถ้วน

```javascript
const { app, BrowserWindow } = require('electron')

// เมื่อแอพพร้อมทำงาน
app.on('ready', () => {
  console.log('App is ready!')
  createWindow()
})

// macOS: เมื่อ click dock icon และไม่มี window
app.on('activate', () => {
  if (BrowserWindow.getAllWindows().length === 0) {
    createWindow()
  }
})

// ก่อนที่แอพจะปิด
app.on('before-quit', (event) => {
  console.log('App is about to quit')
  // event.preventDefault() // ยกเลิกการปิด
})

// ปิดทุก window แล้ว (ยกเว้น macOS)
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})

// แอพปิดแล้ว
app.on('quit', (event, exitCode) => {
  console.log('App quit with code:', exitCode)
})

function createWindow() {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js')
    }
  })
  win.loadFile('index.html')
}
```

---

## 10. IPC Communication พื้นฐาน

### IPC = Inter-Process Communication

```
Main Process ◄──────────────► Renderer Process
              ipcMain / ipcRenderer
```

### ตัวอย่าง IPC แบบง่าย

```javascript
// main.js (Main Process)
const { ipcMain } = require('electron')

ipcMain.on('ping', (event, message) => {
  console.log('Received:', message)  // "hello from renderer"
  event.reply('pong', 'hello from main')
})
```

```javascript
// renderer.js (Renderer Process)
const { ipcRenderer } = require('electron')

// ส่งข้อความ
ipcRenderer.send('ping', 'hello from renderer')

// รับข้อความตอบกลับ
ipcRenderer.on('pong', (event, message) => {
  console.log('Received:', message)  // "hello from main"
})
```

---

## 11. แหล่งข้อมูลที่แนะนำ

### เอกสารทางการ
- 🌐 [Electron.js Official Docs](https://www.electronjs.org/docs)
- 📦 [Electron NPM Package](https://www.npmjs.com/package/electron)
- 💻 [Electron GitHub Repository](https://github.com/electron/electron)

### ชุมชนและการเรียนรู้
- 💬 [Electron Discord Server](https://discord.gg/APGC3k5yaH)
- 📚 [Awesome Electron](https://github.com/sindresorhus/awesome-electron)
- 🎯 [Electron Fiddle](https://www.electronjs.org/fiddle) - ทดลองโค้ดออนไลน์

---

## 12. สรุป

ในบทนี้เราได้เรียนรู้:

1. **Electron คืออะไร**: Framework สร้าง Desktop app ด้วย Web Technologies
2. **สถาปัตยกรรม**: Main Process + Renderer Process + IPC
3. **ข้อดีข้อเสีย**: Cross-platform แต่ใช้ RAM สูง
4. **แอพที่ใช้**: VS Code, Slack, Discord, Notion
5. **Lifecycle**: วงจรชีวิตของแอพ Electron

---

## 🔗 บทถัดไป

➡️ **[Part 002: Setup & Installation](part-002.md)** - ติดตั้ง Node.js, npm, และ Electron เพื่อเริ่มพัฒนา

---

*📅 อัพเดทล่าสุด: 2024 | ระดับ: Beginner | เวลาเรียน: ~45 นาที*
