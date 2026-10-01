# Part 056: Menu Bar App (macOS)
## สร้าง Menu Bar Application บน macOS

---

## เป้าหมายของบทเรียนนี้

- สร้าง Tray-only app ที่ซ่อน Dock icon
- LSUIElement สำหรับ background app
- Menu bar window positioning
- Login item / Launch at startup
- Status bar menus

---

## 1. LSUIElement - ซ่อน Dock Icon

```typescript
// src/main/index.ts
import { app, Tray, Menu, BrowserWindow, nativeImage, screen } from 'electron';
import { join } from 'path';

// ซ่อน Dock icon บน macOS
// ต้องตั้งค่าใน Info.plist หรือเรียก setActivationPolicy
function setupMenuBarApp(): void {
  if (process.platform === 'darwin') {
    // ซ่อน Dock icon
    app.dock.hide();
    
    // หรือใช้ LSUIElement ใน Info.plist:
    // <key>LSUIElement</key>
    // <string>1</string>
  }
}

// เรียกก่อน app.whenReady()
setupMenuBarApp();

app.whenReady().then(() => {
  createTray();
  createMenuBarWindow();
});
```

### electron-builder.yml สำหรับ LSUIElement

```yaml
# electron-builder.yml
mac:
  extendInfo:
    LSUIElement: true   # ซ่อน Dock icon
    NSHighResolutionCapable: true
```

---

## 2. Tray Icon Setup

```typescript
// src/main/tray.ts
import { Tray, Menu, nativeImage, app } from 'electron';
import { join } from 'path';

let tray: Tray | null = null;

export function createTray(): Tray {
  // สร้าง tray icon
  const iconPath = getTrayIconPath();
  const icon = nativeImage.createFromPath(iconPath);
  
  // macOS ใช้ template image (สีขาว/ดำตาม system)
  if (process.platform === 'darwin') {
    icon.setTemplateImage(true);
  }
  
  tray = new Tray(icon);
  tray.setToolTip('My Menu Bar App');
  tray.setTitle(''); // macOS แสดง text ข้าง icon
  
  // สร้าง context menu
  const contextMenu = Menu.buildFromTemplate([
    {
      label: 'Open',
      click: () => showWindow(),
    },
    { type: 'separator' },
    {
      label: 'Preferences',
      accelerator: 'Cmd+,',
      click: () => openPreferences(),
    },
    { type: 'separator' },
    {
      label: 'Launch at Login',
      type: 'checkbox',
      checked: app.getLoginItemSettings().openAtLogin,
      click: (menuItem) => toggleLoginItem(menuItem.checked),
    },
    { type: 'separator' },
    {
      label: 'Quit',
      accelerator: 'Cmd+Q',
      click: () => app.quit(),
    },
  ]);
  
  tray.setContextMenu(contextMenu);
  
  // คลิกซ้ายเพื่อเปิด window (macOS)
  tray.on('click', () => toggleWindow());
  
  return tray;
}

function getTrayIconPath(): string {
  const basePath = app.isPackaged
    ? join(process.resourcesPath, 'assets/tray')
    : join(__dirname, '../../../assets/tray');
  
  if (process.platform === 'darwin') {
    return join(basePath, 'iconTemplate.png'); // @2x support อัตโนมัติ
  } else if (process.platform === 'win32') {
    return join(basePath, 'icon.ico');
  } else {
    return join(basePath, 'icon.png');
  }
}

function toggleLoginItem(enable: boolean): void {
  app.setLoginItemSettings({
    openAtLogin: enable,
    openAsHidden: true, // เริ่ม hidden (macOS)
  });
}
```

---

## 3. Menu Bar Window

```typescript
// src/main/menu-bar-window.ts
import { BrowserWindow, Tray, screen, app } from 'electron';
import { join } from 'path';

let menuBarWindow: BrowserWindow | null = null;
let trayRef: Tray | null = null;

export function createMenuBarWindow(): BrowserWindow {
  menuBarWindow = new BrowserWindow({
    width: 360,
    height: 480,
    show: false,
    frame: false,           // ไม่มี title bar
    resizable: false,
    movable: false,
    alwaysOnTop: true,
    skipTaskbar: true,       // ไม่ปรากฏใน taskbar
    
    // macOS specific
    transparent: true,       // ทำให้ window โปร่งใส
    vibrancy: 'menu',        // blur effect (macOS)
    
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: join(__dirname, '../preload/index.js'),
    },
  });
  
  if (app.isPackaged) {
    menuBarWindow.loadFile(join(__dirname, '../renderer/index.html'));
  } else {
    menuBarWindow.loadURL('http://localhost:5173');
  }
  
  // ซ่อน window เมื่อ focus หาย
  menuBarWindow.on('blur', () => {
    if (!menuBarWindow?.webContents.isDevToolsOpened()) {
      menuBarWindow?.hide();
    }
  });
  
  return menuBarWindow;
}

export function setTrayRef(tray: Tray): void {
  trayRef = tray;
}

export function showWindow(): void {
  if (!menuBarWindow || !trayRef) return;
  
  positionWindow();
  menuBarWindow.show();
  menuBarWindow.focus();
}

export function hideWindow(): void {
  menuBarWindow?.hide();
}

export function toggleWindow(): void {
  if (menuBarWindow?.isVisible()) {
    hideWindow();
  } else {
    showWindow();
  }
}

function positionWindow(): void {
  if (!menuBarWindow || !trayRef) return;
  
  const trayBounds = trayRef.getBounds();
  const windowBounds = menuBarWindow.getBounds();
  const screenBounds = screen.getPrimaryDisplay().workArea;
  
  // คำนวณ position
  let x = Math.round(trayBounds.x + (trayBounds.width / 2) - (windowBounds.width / 2));
  let y = Math.round(trayBounds.y + trayBounds.height + 4);
  
  // macOS - menu bar อยู่บน ให้ window อยู่ใต้ tray icon
  if (process.platform === 'darwin') {
    y = Math.round(trayBounds.y + trayBounds.height + 4);
  }
  
  // Windows - taskbar อาจอยู่ด้านล่าง
  if (process.platform === 'win32') {
    if (trayBounds.y > screenBounds.height / 2) {
      // Taskbar อยู่ด้านล่าง
      y = trayBounds.y - windowBounds.height - 4;
    }
  }
  
  // ตรวจสอบว่าไม่ออกนอกหน้าจอ
  if (x + windowBounds.width > screenBounds.x + screenBounds.width) {
    x = screenBounds.x + screenBounds.width - windowBounds.width;
  }
  if (x < screenBounds.x) {
    x = screenBounds.x;
  }
  
  menuBarWindow.setPosition(x, y, false);
}
```

---

## 4. Login Item (Launch at Startup)

```typescript
// src/main/login-item.ts
import { app } from 'electron';

export function getLoginItemStatus(): boolean {
  return app.getLoginItemSettings().openAtLogin;
}

export function setLoginItem(enable: boolean): void {
  if (process.platform === 'darwin') {
    app.setLoginItemSettings({
      openAtLogin: enable,
      openAsHidden: enable, // เริ่มแบบ hidden
    });
  } else if (process.platform === 'win32') {
    app.setLoginItemSettings({
      openAtLogin: enable,
      // Windows ต้องระบุ path
      path: process.execPath,
      args: ['--startup'],
    });
  } else {
    // Linux: ต้องสร้าง .desktop file ใน ~/.config/autostart/
    setLinuxAutostart(enable);
  }
}

function setLinuxAutostart(enable: boolean): void {
  const { writeFileSync, mkdirSync, rmSync, existsSync } = require('fs');
  const { join } = require('path');
  const os = require('os');
  
  const autostartDir = join(os.homedir(), '.config/autostart');
  const desktopFilePath = join(autostartDir, 'myapp.desktop');
  
  if (enable) {
    mkdirSync(autostartDir, { recursive: true });
    writeFileSync(desktopFilePath, `[Desktop Entry]
Type=Application
Name=My App
Exec=${process.execPath}
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
`);
  } else {
    if (existsSync(desktopFilePath)) {
      rmSync(desktopFilePath);
    }
  }
}

// ตรวจสอบว่า app เริ่มจาก startup
export function isStartingFromLogin(): boolean {
  const settings = app.getLoginItemSettings();
  return settings.wasOpenedAtLogin || process.argv.includes('--startup');
}
```

---

## 5. Dynamic Tray Menu

```typescript
// src/main/tray-menu.ts
import { Menu, Tray, app } from 'electron';
import { getLoginItemStatus, setLoginItem } from './login-item';

interface AppStatus {
  isConnected: boolean;
  syncCount: number;
  lastSync?: Date;
}

export function buildTrayMenu(tray: Tray, status: AppStatus): void {
  const menu = Menu.buildFromTemplate([
    // Status item
    {
      label: status.isConnected ? 'Connected' : 'Disconnected',
      enabled: false,
      icon: createStatusIcon(status.isConnected),
    },
    
    ...(status.syncCount > 0 ? [{
      label: `${status.syncCount} items to sync`,
      enabled: false,
    }] : []),
    
    ...(status.lastSync ? [{
      label: `Last sync: ${formatTime(status.lastSync)}`,
      enabled: false,
    }] : []),
    
    { type: 'separator' as const },
    
    {
      label: 'Open App',
      click: () => showWindow(),
    },
    {
      label: 'Sync Now',
      click: () => triggerSync(),
      enabled: status.isConnected,
    },
    
    { type: 'separator' as const },
    
    {
      label: 'Preferences...',
      click: () => openPreferences(),
    },
    {
      label: 'Launch at Login',
      type: 'checkbox' as const,
      checked: getLoginItemStatus(),
      click: (item) => setLoginItem(item.checked),
    },
    
    { type: 'separator' as const },
    
    {
      label: 'About',
      click: () => app.setAboutPanelOptions({
        applicationName: 'My App',
        applicationVersion: app.getVersion(),
      }) && app.showAboutPanel(),
    },
    
    { type: 'separator' as const },
    
    {
      label: 'Quit My App',
      accelerator: 'Cmd+Q',
      click: () => app.quit(),
    },
  ]);
  
  tray.setContextMenu(menu);
}

function createStatusIcon(connected: boolean): Electron.NativeImage {
  const { nativeImage } = require('electron');
  // สร้าง simple colored circle
  // ใน production ควรใช้ PNG files
  return nativeImage.createEmpty();
}

function formatTime(date: Date): string {
  const now = new Date();
  const diffMs = now.getTime() - date.getTime();
  const diffMins = Math.floor(diffMs / 60000);
  
  if (diffMins < 1) return 'just now';
  if (diffMins < 60) return `${diffMins}m ago`;
  if (diffMins < 1440) return `${Math.floor(diffMins / 60)}h ago`;
  return date.toLocaleDateString();
}

// อัพเดท tray menu เมื่อ status เปลี่ยน
let updateTimer: NodeJS.Timeout | null = null;

export function scheduleMenuUpdate(tray: Tray, getStatus: () => AppStatus): void {
  if (updateTimer) clearInterval(updateTimer);
  
  // อัพเดทเมื่อ status เปลี่ยน
  updateTimer = setInterval(() => {
    buildTrayMenu(tray, getStatus());
  }, 30000); // อัพเดททุก 30 วินาที
}
```

---

## 6. macOS Vibrancy Window

```typescript
// src/main/vibrancy-window.ts
import { BrowserWindow } from 'electron';

export function createVibrancyWindow(): BrowserWindow {
  const win = new BrowserWindow({
    width: 360,
    height: 480,
    frame: false,
    transparent: true,
    
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
    },
  });
  
  // macOS vibrancy materials
  if (process.platform === 'darwin') {
    win.setVibrancy('fullscreen-ui'); // หรือ 'menu', 'popover', 'sidebar', etc.
    win.setWindowButtonVisibility(false);
    
    // Rounded corners
    win.setBackgroundColor('#00000000'); // transparent
    
    // Traffic lights position (macOS)
    win.setWindowButtonVisibility(false);
  }
  
  return win;
}
```

---

## 7. IPC Handlers สำหรับ Menu Bar

```typescript
// src/main/ipc-tray.ts
import { ipcMain, app } from 'electron';
import { getLoginItemStatus, setLoginItem } from './login-item';
import { hideWindow } from './menu-bar-window';

export function setupTrayIPC(): void {
  ipcMain.handle('tray:getLoginItemStatus', () => {
    return getLoginItemStatus();
  });
  
  ipcMain.handle('tray:setLoginItem', (_, enable: boolean) => {
    setLoginItem(enable);
  });
  
  ipcMain.handle('tray:getVersion', () => {
    return app.getVersion();
  });
  
  ipcMain.on('window:close', () => {
    hideWindow();
  });
  
  ipcMain.on('app:quit', () => {
    app.quit();
  });
}
```

---

## 8. Renderer - Menu Bar UI

```tsx
// src/renderer/MenuBarApp.tsx
import React, { useEffect, useState } from 'react';

interface AppData {
  loginItemEnabled: boolean;
  version: string;
}

export function MenuBarApp() {
  const [data, setData] = useState<AppData>({
    loginItemEnabled: false,
    version: '',
  });
  
  useEffect(() => {
    Promise.all([
      window.electron.tray.getLoginItemStatus(),
      window.electron.tray.getVersion(),
    ]).then(([loginItem, version]) => {
      setData({ loginItemEnabled: loginItem, version });
    });
  }, []);
  
  const handleClose = () => {
    window.electron.window.close();
  };
  
  const handleLoginItemToggle = async () => {
    const newValue = !data.loginItemEnabled;
    await window.electron.tray.setLoginItem(newValue);
    setData(prev => ({ ...prev, loginItemEnabled: newValue }));
  };
  
  return (
    <div className="menu-bar-app">
      <div className="header">
        <h1>My App</h1>
        <span className="version">v{data.version}</span>
        <button onClick={handleClose} className="close-btn">×</button>
      </div>
      
      <div className="content">
        {/* App content */}
      </div>
      
      <div className="footer">
        <label>
          <input
            type="checkbox"
            checked={data.loginItemEnabled}
            onChange={handleLoginItemToggle}
          />
          Launch at login
        </label>
        <button onClick={() => window.electron.app.quit()}>
          Quit
        </button>
      </div>
    </div>
  );
}
```

---

## 9. สรุป

### Checklist สำหรับ Menu Bar App

- [ ] ซ่อน Dock icon ด้วย `app.dock.hide()` หรือ `LSUIElement`
- [ ] สร้าง Tray icon ด้วย template image (macOS)
- [ ] คำนวณ window position จาก tray bounds
- [ ] ซ่อน window เมื่อ blur
- [ ] ตั้ง `skipTaskbar: true`
- [ ] เพิ่ม Launch at Login option
- [ ] ใช้ `vibrancy` สำหรับ macOS visual effects

### API สำคัญ

| API | ใช้สำหรับ |
|-----|---------|
| `app.dock.hide()` | ซ่อน Dock icon |
| `app.setLoginItemSettings()` | Login at startup |
| `tray.getBounds()` | หาตำแหน่ง tray icon |
| `win.setVibrancy()` | Blur/vibrancy effect |
| `win.setWindowButtonVisibility()` | ซ่อน traffic lights |

---

*จบ Part 056 - ต่อไป Part 057: Windows-specific Features*
