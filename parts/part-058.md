# Part 058: macOS-specific Features
## ฟีเจอร์เฉพาะ macOS ใน Electron

---

## เป้าหมายของบทเรียนนี้

- Dock Menu
- Full Screen เต็มรูปแบบ
- macOS Spaces
- AppleScript integration
- Spotlight support
- Handoff / Continuity

---

## 1. Dock Menu

```typescript
// src/main/macos-features/dock-menu.ts
import { app, Menu, BrowserWindow } from 'electron';

export function setupDockMenu(): void {
  if (process.platform !== 'darwin') return;
  
  const dockMenu = Menu.buildFromTemplate([
    {
      label: 'New Window',
      click: () => createMainWindow(),
    },
    {
      label: 'New Document',
      click: () => {
        const win = BrowserWindow.getAllWindows()[0];
        win?.webContents.send('document:new');
      },
    },
    { type: 'separator' },
    {
      label: 'Open Recent',
      submenu: getRecentDocumentsMenu(),
    },
  ]);
  
  app.dock.setMenu(dockMenu);
}

function getRecentDocumentsMenu(): Electron.MenuItemConstructorOptions[] {
  const recentFiles: string[] = []; // ดึงจาก store
  
  if (recentFiles.length === 0) {
    return [{ label: 'No Recent Files', enabled: false }];
  }
  
  return recentFiles.map(file => ({
    label: require('path').basename(file),
    click: () => openFile(file),
  }));
}

// Dock badge
export function setDockBadge(count: number): void {
  if (process.platform !== 'darwin') return;
  
  if (count <= 0) {
    app.dock.setBadge('');
  } else {
    app.dock.setBadge(count > 99 ? '99+' : count.toString());
  }
}

// Dock progress
export function setDockProgress(progress: number): void {
  if (process.platform !== 'darwin') return;
  
  // -1 = hide, 0-1 = progress
  app.dock.setBadge('');
}

// Dock bounce
export function bounceDock(type: 'critical' | 'informational' = 'informational'): void {
  if (process.platform !== 'darwin') return;
  
  const id = app.dock.bounce(type);
  
  // หยุด bounce หลัง 5 วินาที (สำหรับ informational)
  if (type === 'informational') {
    setTimeout(() => app.dock.cancelBounce(id), 5000);
  }
}
```

---

## 2. Full Screen / Spaces

```typescript
// src/main/macos-features/fullscreen.ts
import { BrowserWindow, app } from 'electron';

export function createFullScreenAwareWindow(): BrowserWindow {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    
    // macOS full screen behavior
    fullscreenable: true,
    
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
    },
    
    // macOS-specific
    titleBarStyle: 'hiddenInset', // หรือ 'hidden', 'customButtonsOnHover'
    trafficLightPosition: { x: 16, y: 16 },
    
    // ใช้ native macOS fullscreen (ไป Space ใหม่)
    // ถ้าต้องการ kiosk mode (ไม่ไป Space) ใช้ kiosk: true
  });
  
  // Listen for fullscreen events
  win.on('enter-full-screen', () => {
    win.webContents.send('window:fullscreen', true);
  });
  
  win.on('leave-full-screen', () => {
    win.webContents.send('window:fullscreen', false);
  });
  
  // macOS Spaces - window จะย้ายไปตาม Space
  win.on('moved', () => {
    // save position
  });
  
  return win;
}

// Custom traffic lights position
export function setupCustomTitleBar(win: BrowserWindow): void {
  if (process.platform !== 'darwin') return;
  
  // ตั้ง traffic lights position ให้ห่างจากขอบ
  win.setWindowButtonVisibility(true);
  
  // เมื่อ enter fullscreen ซ่อน custom title bar
  win.on('enter-full-screen', () => {
    win.webContents.send('titlebar:hidden', true);
  });
  
  win.on('leave-full-screen', () => {
    win.webContents.send('titlebar:hidden', false);
  });
}
```

---

## 3. Native macOS Menu

```typescript
// src/main/macos-features/native-menu.ts
import { app, Menu, shell, BrowserWindow, dialog } from 'electron';

export function createMacOSMenu(): void {
  if (process.platform !== 'darwin') return;
  
  const template: Electron.MenuItemConstructorOptions[] = [
    // App menu (ชื่อ app อัตโนมัติ)
    {
      label: app.getName(),
      submenu: [
        {
          label: `About ${app.getName()}`,
          role: 'about',
        },
        { type: 'separator' },
        {
          label: 'Preferences...',
          accelerator: 'Cmd+,',
          click: () => openPreferences(),
        },
        { type: 'separator' },
        {
          label: 'Services',
          role: 'services',
        },
        { type: 'separator' },
        {
          label: `Hide ${app.getName()}`,
          role: 'hide',
        },
        {
          label: 'Hide Others',
          role: 'hideOthers',
        },
        {
          label: 'Show All',
          role: 'unhide',
        },
        { type: 'separator' },
        {
          label: `Quit ${app.getName()}`,
          role: 'quit',
        },
      ],
    },
    
    // File
    {
      label: 'File',
      submenu: [
        {
          label: 'New',
          accelerator: 'Cmd+N',
          click: () => newDocument(),
        },
        {
          label: 'Open...',
          accelerator: 'Cmd+O',
          click: () => openDocument(),
        },
        {
          label: 'Open Recent',
          role: 'recentDocuments',
          submenu: [
            {
              label: 'Clear Recent',
              role: 'clearRecentDocuments',
            },
          ],
        },
        { type: 'separator' },
        {
          label: 'Save',
          accelerator: 'Cmd+S',
          click: () => saveDocument(),
        },
        {
          label: 'Save As...',
          accelerator: 'Cmd+Shift+S',
          click: () => saveDocumentAs(),
        },
        { type: 'separator' },
        {
          label: 'Close Window',
          role: 'close',
        },
      ],
    },
    
    // Edit
    {
      label: 'Edit',
      submenu: [
        { role: 'undo' },
        { role: 'redo' },
        { type: 'separator' },
        { role: 'cut' },
        { role: 'copy' },
        { role: 'paste' },
        { role: 'pasteAndMatchStyle' },
        { role: 'selectAll' },
        { type: 'separator' },
        {
          label: 'Find',
          submenu: [
            {
              label: 'Find...',
              accelerator: 'Cmd+F',
              click: () => openFind(),
            },
            {
              label: 'Find Next',
              accelerator: 'Cmd+G',
              click: () => findNext(),
            },
            {
              label: 'Find Previous',
              accelerator: 'Cmd+Shift+G',
              click: () => findPrevious(),
            },
          ],
        },
        // macOS-specific spell check
        {
          label: 'Spelling and Grammar',
          submenu: [
            { role: 'toggleDevTools' }, // ใช้ built-in role
          ],
        },
        { type: 'separator' },
        {
          label: 'Start Dictation...',
          role: 'startSpeaking',
        },
        {
          label: 'Emoji & Symbols',
          role: 'showSubstitutions',
        },
      ],
    },
    
    // View
    {
      label: 'View',
      submenu: [
        { role: 'reload' },
        { role: 'forceReload' },
        { role: 'toggleDevTools' },
        { type: 'separator' },
        { role: 'resetZoom' },
        { role: 'zoomIn' },
        { role: 'zoomOut' },
        { type: 'separator' },
        { role: 'togglefullscreen' },
      ],
    },
    
    // Window
    {
      label: 'Window',
      submenu: [
        { role: 'minimize' },
        { role: 'zoom' },
        { role: 'togglefullscreen' },
        { type: 'separator' },
        { role: 'front' }, // macOS: Bring All to Front
        { role: 'window' },
      ],
    },
    
    // Help
    {
      role: 'help',
      submenu: [
        {
          label: 'Documentation',
          click: () => shell.openExternal('https://docs.example.com'),
        },
        {
          label: 'Keyboard Shortcuts',
          click: () => openShortcutsWindow(),
        },
        { type: 'separator' },
        {
          label: 'Report Issue',
          click: () => shell.openExternal('https://github.com/org/app/issues'),
        },
      ],
    },
  ];
  
  Menu.setApplicationMenu(Menu.buildFromTemplate(template));
}
```

---

## 4. AppleScript Integration

```typescript
// src/main/macos-features/applescript.ts
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

export async function runAppleScript(script: string): Promise<string> {
  if (process.platform !== 'darwin') {
    throw new Error('AppleScript is only supported on macOS');
  }
  
  const { stdout } = await execAsync(`osascript -e '${script.replace(/'/g, "'\\''")}'`);
  return stdout.trim();
}

// ตัวอย่าง AppleScript operations

// แสดง dialog
export async function showAppleDialog(message: string): Promise<'OK' | 'Cancel'> {
  const result = await runAppleScript(
    `display dialog "${message}" buttons {"Cancel", "OK"} default button "OK"`
  );
  return result.includes('OK') ? 'OK' : 'Cancel';
}

// เปิด URL ใน Safari
export async function openInSafari(url: string): Promise<void> {
  await runAppleScript(`
    tell application "Safari"
      activate
      open location "${url}"
    end tell
  `);
}

// ดึงข้อมูลจาก Contacts (ต้องขอ permission)
export async function getContactInfo(name: string): Promise<string> {
  return runAppleScript(`
    tell application "Contacts"
      set thePerson to first person whose name is "${name}"
      set theEmail to value of first email of thePerson
      return theEmail
    end tell
  `);
}

// ส่ง notification ผ่าน AppleScript
export async function sendAppleNotification(
  title: string, 
  message: string,
  soundName?: string
): Promise<void> {
  const sound = soundName ? ` sound name "${soundName}"` : '';
  await runAppleScript(
    `display notification "${message}" with title "${title}"${sound}`
  );
}

// ดึง System Preferences
export async function getSystemPreference(domain: string, key: string): Promise<string> {
  const { stdout } = await execAsync(`defaults read ${domain} ${key}`);
  return stdout.trim();
}

// ตรวจสอบ Dark Mode
export async function isDarkMode(): Promise<boolean> {
  try {
    const result = await getSystemPreference(
      'NSGlobalDomain',
      'AppleInterfaceStyle'
    );
    return result === 'Dark';
  } catch {
    return false; // ถ้า error = Light mode
  }
}
```

---

## 5. Handoff / NSUserActivity

```typescript
// src/main/macos-features/handoff.ts
// Handoff ต้องใช้ native module หรือ node-mac-notifs

// ตัวอย่างการ implement ผ่าน protocol URL

import { app, protocol } from 'electron';

// เปิดรับ URL scheme สำหรับ Handoff
export function setupHandoff(): void {
  if (process.platform !== 'darwin') return;
  
  // Register URL scheme
  app.setAsDefaultProtocolClient('myapp');
  
  // Handle URL
  app.on('open-url', (event, url) => {
    event.preventDefault();
    handleHandoffURL(url);
  });
}

function handleHandoffURL(url: string): void {
  console.log('Handoff URL:', url);
  
  // Parse URL และ restore state
  const urlObj = new URL(url);
  const path = urlObj.pathname;
  const params = Object.fromEntries(urlObj.searchParams);
  
  // Focus window และ navigate
  const win = BrowserWindow.getAllWindows()[0];
  if (win) {
    if (win.isMinimized()) win.restore();
    win.focus();
    win.webContents.send('handoff:restore', { path, params });
  }
}
```

---

## 6. macOS Accessibility

```typescript
// src/main/macos-features/accessibility.ts
import { systemPreferences, app } from 'electron';

export function checkAccessibilityPermission(): boolean {
  if (process.platform !== 'darwin') return true;
  
  return systemPreferences.isTrustedAccessibilityClient(false);
}

export function requestAccessibilityPermission(): boolean {
  if (process.platform !== 'darwin') return true;
  
  // true = แสดง dialog ขอ permission
  return systemPreferences.isTrustedAccessibilityClient(true);
}

// ตรวจสอบ screen recording permission
export async function checkScreenRecordingPermission(): Promise<boolean> {
  if (process.platform !== 'darwin') return true;
  
  const status = systemPreferences.getMediaAccessStatus('screen');
  return status === 'granted';
}

// ตรวจสอบ microphone permission
export async function checkMicrophonePermission(): Promise<boolean> {
  if (process.platform !== 'darwin') return true;
  
  const status = systemPreferences.getMediaAccessStatus('microphone');
  if (status === 'not-determined') {
    return systemPreferences.askForMediaAccess('microphone');
  }
  return status === 'granted';
}
```

---

## 7. Spotlight / Quick Look

```typescript
// src/main/macos-features/spotlight.ts
import { shell } from 'electron';
import { exec } from 'child_process';

// เปิด Quick Look สำหรับไฟล์
export function quickLookFile(filePath: string): void {
  if (process.platform !== 'darwin') return;
  
  exec(`qlmanage -p "${filePath}"`);
}

// เพิ่ม file metadata สำหรับ Spotlight indexing
export async function addSpotlightMetadata(
  filePath: string,
  metadata: Record<string, string>
): Promise<void> {
  if (process.platform !== 'darwin') return;
  
  // ใช้ xattr เพื่อเพิ่ม metadata
  for (const [key, value] of Object.entries(metadata)) {
    await new Promise<void>((resolve, reject) => {
      exec(
        `xattr -w com.example.${key} "${value}" "${filePath}"`,
        (error) => {
          if (error) reject(error);
          else resolve();
        }
      );
    });
  }
}

// Force Spotlight reindex
export function reindexSpotlight(path: string): void {
  if (process.platform !== 'darwin') return;
  
  exec(`mdimport "${path}"`);
}
```

---

## 8. System Colors

```typescript
// src/main/macos-features/system-colors.ts
import { systemPreferences, nativeTheme } from 'electron';

export function getSystemColors(): Record<string, string> {
  if (process.platform !== 'darwin') return {};
  
  return {
    accentColor: systemPreferences.getAccentColor(),
    systemBlue: systemPreferences.getColor('blue'),
    controlBackground: systemPreferences.getColor('control-background'),
    label: systemPreferences.getColor('label'),
    secondaryLabel: systemPreferences.getColor('secondary-label'),
    windowBackground: systemPreferences.getColor('window-background'),
    selectedContent: systemPreferences.getColor('selected-content-background'),
  };
}

export function watchSystemTheme(
  callback: (isDark: boolean) => void
): () => void {
  const handler = () => {
    callback(nativeTheme.shouldUseDarkColors);
  };
  
  nativeTheme.on('updated', handler);
  
  return () => {
    nativeTheme.removeListener('updated', handler);
  };
}

export function watchAccentColor(
  callback: (color: string) => void
): () => void {
  if (process.platform !== 'darwin') return () => {};
  
  systemPreferences.on('accent-color-changed', (_, newColor) => {
    callback(newColor);
  });
  
  return () => {
    systemPreferences.removeAllListeners('accent-color-changed');
  };
}
```

---

## 9. สรุป

### macOS-specific APIs

| API | ใช้สำหรับ |
|-----|---------|
| `app.dock.setMenu()` | Dock context menu |
| `app.dock.setBadge()` | Dock badge count |
| `app.dock.bounce()` | Attention animation |
| `app.dock.hide/show()` | ซ่อน/แสดง Dock icon |
| `win.setVibrancy()` | Blur effects |
| `systemPreferences.getAccentColor()` | System accent color |
| `systemPreferences.isTrustedAccessibilityClient()` | Accessibility permission |

### Checklist

- [ ] ตั้ง Dock menu ด้วย `app.dock.setMenu()`
- [ ] ใช้ native menu roles (`about`, `hide`, `services`)
- [ ] รองรับ Dark Mode ด้วย `nativeTheme`
- [ ] ใช้ `titleBarStyle: 'hiddenInset'` สำหรับ modern look
- [ ] ตรวจสอบ permissions ก่อนใช้ camera/microphone/screen

---

*จบ Part 058 - ต่อไป Part 059: Linux-specific Features*
