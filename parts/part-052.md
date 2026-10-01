# Part 052: Electron DevTools Extensions
## การติดตั้งและใช้ DevTools Extensions ใน Electron

---

## 🎯 เป้าหมายของบทเรียนนี้

- React DevTools
- Redux DevTools
- Vue DevTools
- ติดตั้ง extensions
- session.loadExtension API
- Custom DevTools panels

---

## 1. ทำไมต้องใช้ DevTools Extensions?

DevTools Extensions ช่วยให้ debug ง่ายขึ้น:
- **React DevTools**: ดู component tree, props, state, performance
- **Redux DevTools**: ดู actions, state changes, time-travel debugging
- **Vue DevTools**: เหมือน React DevTools แต่สำหรับ Vue

---

## 2. ติดตั้ง React DevTools

### วิธีที่ 1: ใช้ electron-devtools-installer

```bash
npm install --save-dev electron-devtools-installer
```

```typescript
// src/main/devtools.ts
import { app, BrowserWindow } from 'electron';

export async function installDevTools(): Promise<void> {
  if (!app.isPackaged && process.env.NODE_ENV === 'development') {
    try {
      const {
        default: installExtension,
        REACT_DEVELOPER_TOOLS,
        REDUX_DEVTOOLS,
        VUEJS_DEVTOOLS,
      } = await import('electron-devtools-installer');

      const extensions = [
        REACT_DEVELOPER_TOOLS,
        REDUX_DEVTOOLS,
      ];

      for (const ext of extensions) {
        try {
          const name = await installExtension(ext, {
            loadExtensionOptions: {
              allowFileAccess: true,
            },
            forceDownload: false,
          });
          console.log(`Installed DevTools: ${name}`);
        } catch (err) {
          console.error(`Failed to install ${ext}:`, err);
        }
      }
    } catch (err) {
      console.error('DevTools installation failed:', err);
    }
  }
}
```

```typescript
// src/main/index.ts
import { app } from 'electron';
import { installDevTools } from './devtools';

app.whenReady().then(async () => {
  // ติดตั้ง DevTools ก่อนสร้าง window
  await installDevTools();
  
  createWindow();
});
```

### วิธีที่ 2: ใช้ session.loadExtension โดยตรง

```typescript
// src/main/devtools-manual.ts
import { session, app } from 'electron';
import { join } from 'path';
import { existsSync } from 'fs';

// Path ของ Chrome extensions ที่ติดตั้งแล้ว
function getChromeExtensionsPath(): string {
  const homeDir = process.env.HOME || process.env.USERPROFILE || '';
  
  switch (process.platform) {
    case 'darwin':
      return join(
        homeDir,
        'Library/Application Support/Google/Chrome/Default/Extensions'
      );
    case 'win32':
      return join(
        process.env.LOCALAPPDATA || '',
        'Google/Chrome/User Data/Default/Extensions'
      );
    default: // linux
      return join(homeDir, '.config/google-chrome/Default/Extensions');
  }
}

async function loadChromeExtension(extensionId: string): Promise<void> {
  const extensionsPath = getChromeExtensionsPath();
  const extensionDir = join(extensionsPath, extensionId);
  
  if (!existsSync(extensionDir)) {
    console.warn(`Extension not found: ${extensionId}`);
    return;
  }
  
  // ดึง version directory
  const { readdirSync } = await import('fs');
  const versions = readdirSync(extensionDir);
  
  if (versions.length === 0) {
    console.warn(`No versions found for extension: ${extensionId}`);
    return;
  }
  
  const latestVersion = versions[versions.length - 1];
  const extensionPath = join(extensionDir, latestVersion);
  
  try {
    const ext = await session.defaultSession.loadExtension(extensionPath, {
      allowFileAccess: true,
    });
    console.log(`Loaded extension: ${ext.name} (${ext.id})`);
  } catch (err) {
    console.error(`Failed to load extension ${extensionId}:`, err);
  }
}

// React DevTools extension ID (จาก Chrome Web Store)
const REACT_DEVTOOLS_ID = 'fmkadmapgofadopljbjfkapdkoienihi';
// Redux DevTools
const REDUX_DEVTOOLS_ID = 'lmhkpmbekcpmknklioeibfkpmmfibljd';
// Vue DevTools
const VUE_DEVTOOLS_ID = 'nhdogjmejiglipccpnnnanhbledajbpd';

export async function setupManualDevTools(): Promise<void> {
  if (process.env.NODE_ENV !== 'development') return;
  
  await loadChromeExtension(REACT_DEVTOOLS_ID);
  await loadChromeExtension(REDUX_DEVTOOLS_ID);
}
```

---

## 3. Redux DevTools Integration

### src/renderer/store/store.ts

```typescript
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counterSlice';

// ตั้งค่า Redux DevTools
const store = configureStore({
  reducer: {
    counter: counterReducer,
  },
  
  // Redux DevTools Extension ทำงานอัตโนมัติใน development
  devTools: process.env.NODE_ENV !== 'production',
  
  // หรือตั้งค่า custom
  devTools: process.env.NODE_ENV !== 'production' ? {
    name: 'My Electron App',
    trace: true,
    traceLimit: 25,
    actionsBlacklist: ['HEAVY_ACTION'], // ไม่ log action นี้
    stateSanitizer: (state: any) => ({
      ...state,
      // ซ่อน sensitive data
      auth: state.auth ? { ...state.auth, token: '***' } : state.auth,
    }),
    actionSanitizer: (action: any) => ({
      ...action,
      // ซ่อน sensitive action data
      payload: action.type === 'auth/login' 
        ? { ...action.payload, password: '***' } 
        : action.payload,
    }),
  } : false,
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

export default store;
```

---

## 4. Custom DevTools Panel

### สร้าง Custom Panel ใน DevTools

```typescript
// src/main/custom-devtools.ts
import { BrowserWindow, session } from 'electron';

// Inject custom DevTools panel ผ่าน JavaScript
async function addCustomDevPanel(win: BrowserWindow): Promise<void> {
  // รอให้ DevTools เปิด
  win.webContents.on('devtools-opened', () => {
    win.webContents.executeJavaScript(`
      (function() {
        // สร้าง custom panel ใน DevTools
        if (window.__electron_custom_panel__) return;
        window.__electron_custom_panel__ = true;
        
        // ถ้า devtools ยังไม่พร้อม ให้รอ
        const addPanel = () => {
          if (typeof DevToolsAPI === 'undefined') {
            setTimeout(addPanel, 500);
            return;
          }
          
          // DevTools API (experimental)
          // ส่วนใหญ่ custom panels ต้องใช้ extension mechanism
        };
        
        addPanel();
      })();
    `);
  });
}

// ใช้ session.loadExtension สำหรับ custom panel ที่สมบูรณ์
async function loadCustomDevPanel(): Promise<void> {
  // Extension จะต้องอยู่ใน local directory
  const panelPath = join(__dirname, '../../devtools-extension');
  
  try {
    const ext = await session.defaultSession.loadExtension(panelPath);
    console.log('Custom DevTools panel loaded:', ext.name);
  } catch (err) {
    console.error('Failed to load custom panel:', err);
  }
}
```

### สร้าง DevTools Extension Directory

```
devtools-extension/
├── manifest.json
├── panel.html
├── panel.js
└── background.js
```

### devtools-extension/manifest.json

```json
{
  "name": "My App DevTools",
  "version": "1.0.0",
  "manifest_version": 3,
  "description": "DevTools for My Electron App",
  
  "devtools_page": "devtools.html",
  
  "permissions": ["storage", "debugger"],
  
  "background": {
    "service_worker": "background.js"
  },
  
  "content_security_policy": {
    "extension_pages": "script-src 'self'; object-src 'self'"
  }
}
```

### devtools-extension/devtools.html

```html
<!DOCTYPE html>
<html>
<head>
  <script src="devtools.js"></script>
</head>
</html>
```

### devtools-extension/devtools.js

```javascript
// สร้าง panel ใน DevTools
chrome.devtools.panels.create(
  'My App',          // Panel title
  'icon.png',         // Icon
  'panel.html',       // Panel HTML
  function(panel) {
    console.log('Panel created!', panel);
  }
);
```

### devtools-extension/panel.html

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: sans-serif; padding: 10px; }
    .section { margin-bottom: 16px; }
    pre { background: #f5f5f5; padding: 8px; border-radius: 4px; }
  </style>
</head>
<body>
  <h2>My App DevTools</h2>
  
  <div class="section">
    <h3>IPC Events</h3>
    <div id="ipc-events"></div>
  </div>
  
  <div class="section">
    <h3>App State</h3>
    <pre id="app-state">Loading...</pre>
  </div>
  
  <script src="panel.js"></script>
</body>
</html>
```

### devtools-extension/panel.js

```javascript
// ส่วนสื่อสารกับ inspected page
const port = chrome.runtime.connect({ name: 'panel' });

// ดึง state จาก page
function getAppState() {
  chrome.devtools.inspectedWindow.eval(
    'window.__APP_STATE__ ? JSON.stringify(window.__APP_STATE__) : null',
    function(result) {
      if (result) {
        document.getElementById('app-state').textContent = 
          JSON.stringify(JSON.parse(result), null, 2);
      }
    }
  );
}

// อัพเดททุก 1 วินาที
setInterval(getAppState, 1000);
getAppState();

// รับ messages จาก background
port.onMessage.addListener(function(msg) {
  if (msg.type === 'ipc-event') {
    const div = document.getElementById('ipc-events');
    const item = document.createElement('div');
    item.textContent = `${new Date().toLocaleTimeString()} - ${msg.channel}: ${JSON.stringify(msg.data)}`;
    div.prepend(item);
    
    // เก็บแค่ 50 items ล่าสุด
    while (div.children.length > 50) {
      div.removeChild(div.lastChild!);
    }
  }
});
```

---

## 5. DevTools Shortcuts

```typescript
// src/main/index.ts
import { app, BrowserWindow, globalShortcut } from 'electron';

function setupDevToolsShortcuts(win: BrowserWindow): void {
  if (process.env.NODE_ENV !== 'development') return;
  
  // F12 - Toggle DevTools
  win.webContents.on('before-input-event', (event, input) => {
    if (input.key === 'F12') {
      win.webContents.toggleDevTools();
      event.preventDefault();
    }
    
    // Ctrl+Shift+I
    if (input.control && input.shift && input.key === 'I') {
      win.webContents.toggleDevTools();
      event.preventDefault();
    }
    
    // Ctrl+R - Reload
    if (input.control && input.key === 'R') {
      win.webContents.reload();
      event.preventDefault();
    }
    
    // Ctrl+Shift+R - Hard reload
    if (input.control && input.shift && input.key === 'R') {
      win.webContents.reloadIgnoringCache();
      event.preventDefault();
    }
  });
  
  // เปิด DevTools อัตโนมัติใน development
  if (process.env.OPEN_DEVTOOLS === 'true') {
    win.webContents.openDevTools({ mode: 'detach' });
  }
}
```

---

## 6. Expose App State for DevTools

```typescript
// src/renderer/devtools-bridge.ts
// Expose state to custom DevTools panel

function exposeDevToolsAPI(): void {
  if (process.env.NODE_ENV !== 'development') return;
  
  // Expose app state
  (window as any).__APP_STATE__ = {
    get: () => window.__store?.getState(),
    dispatch: (action: any) => window.__store?.dispatch(action),
  };
  
  // IPC logger
  const originalInvoke = window.electronAPI?.invoke;
  if (window.electronAPI && originalInvoke) {
    window.electronAPI.invoke = async (channel: string, ...args: any[]) => {
      console.groupCollapsed(`IPC → ${channel}`);
      console.log('Args:', args);
      
      const result = await originalInvoke(channel, ...args);
      
      console.log('Result:', result);
      console.groupEnd();
      
      return result;
    };
  }
}

exposeDevToolsAPI();
```

---

## 7. สรุป

| Extension | ID | ใช้กับ |
|-----------|-----|-------|
| React DevTools | fmkadmapgofadopljbjfkapdkoienihi | React |
| Redux DevTools | lmhkpmbekcpmknklioeibfkpmmfibljd | Redux/Zustand |
| Vue DevTools | nhdogjmejiglipccpnnnanhbledajbpd | Vue |
| Angular DevTools | ienfalfjdbdpebioblfackkekamfmbnh | Angular |

### Best Practices

- ติดตั้ง DevTools เฉพาะ `process.env.NODE_ENV === 'development'`
- ใช้ `electron-devtools-installer` สำหรับความสะดวก
- สร้าง custom DevTools panel สำหรับ app-specific debugging
- ใช้ `session.loadExtension` แทน deprecated `BrowserWindow.addDevToolsExtension`

---

*จบ Part 052 - ต่อไป Part 053: Custom Context Bridge Patterns*
