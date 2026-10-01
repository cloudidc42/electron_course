# Part 057: Windows-specific Features
## ฟีเจอร์เฉพาะ Windows ใน Electron

---

## เป้าหมายของบทเรียนนี้

- Jump Lists
- Thumbnail Toolbar
- Taskbar Progress
- Notification Badges
- Window Snap / DWM Composition
- Registry Integration

---

## 1. Jump Lists

```typescript
// src/main/windows-features/jump-list.ts
import { app } from 'electron';
import { join } from 'path';

export function setupJumpList(): void {
  if (process.platform !== 'win32') return;
  
  app.setJumpList([
    {
      type: 'custom',
      name: 'Recent Files',
      items: getRecentFiles().map(file => ({
        type: 'file' as const,
        path: file.path,
      })),
    },
    {
      type: 'custom',
      name: 'Tasks',
      items: [
        {
          type: 'task' as const,
          title: 'New Document',
          description: 'Create a new document',
          program: process.execPath,
          args: '--new-document',
          iconPath: process.execPath,
          iconIndex: 0,
        },
        {
          type: 'task' as const,
          title: 'Open File',
          description: 'Open an existing file',
          program: process.execPath,
          args: '--open-file',
          iconPath: process.execPath,
          iconIndex: 0,
        },
        {
          type: 'separator' as const,
        },
        {
          type: 'task' as const,
          title: 'Settings',
          program: process.execPath,
          args: '--settings',
          iconPath: join(app.getPath('userData'), 'settings-icon.ico'),
          iconIndex: 0,
        },
      ],
    },
    {
      type: 'recent',  // Windows จัดการ Recent automatically
    },
    {
      type: 'frequent', // Windows จัดการ Frequent automatically  
    },
  ]);
}

function getRecentFiles(): Array<{ path: string }> {
  // ดึงรายการไฟล์ล่าสุดจาก store
  return [];
}

// อัพเดท Jump List เมื่อเปิดไฟล์
export function addRecentFile(filePath: string): void {
  if (process.platform !== 'win32') return;
  
  app.addRecentDocument(filePath);
  setupJumpList(); // rebuild
}

export function clearRecentFiles(): void {
  if (process.platform !== 'win32') return;
  
  app.clearRecentDocuments();
  setupJumpList();
}
```

---

## 2. Thumbnail Toolbar

```typescript
// src/main/windows-features/thumbnail-toolbar.ts
import { BrowserWindow, nativeImage, app } from 'electron';
import { join } from 'path';

export function setupThumbnailToolbar(win: BrowserWindow): void {
  if (process.platform !== 'win32') return;
  
  const assetsPath = app.isPackaged
    ? join(process.resourcesPath, 'assets/toolbar')
    : join(__dirname, '../../../../assets/toolbar');
  
  win.setThumbarButtons([
    {
      tooltip: 'Previous',
      icon: nativeImage.createFromPath(join(assetsPath, 'prev.png')),
      click: () => {
        win.webContents.send('media:previous');
      },
    },
    {
      tooltip: 'Play/Pause',
      icon: nativeImage.createFromPath(join(assetsPath, 'play.png')),
      click: () => {
        win.webContents.send('media:playPause');
      },
      flags: ['enabled'],
    },
    {
      tooltip: 'Next',
      icon: nativeImage.createFromPath(join(assetsPath, 'next.png')),
      click: () => {
        win.webContents.send('media:next');
      },
    },
    {
      tooltip: 'Mute',
      icon: nativeImage.createFromPath(join(assetsPath, 'mute.png')),
      click: () => {
        win.webContents.send('media:mute');
      },
      flags: ['dismissonclick'],
    },
  ]);
}

// อัพเดท toolbar icons ตาม state
export function updateThumbnailToolbar(
  win: BrowserWindow,
  isPlaying: boolean,
  isMuted: boolean
): void {
  if (process.platform !== 'win32') return;
  
  const assetsPath = app.isPackaged
    ? join(process.resourcesPath, 'assets/toolbar')
    : join(__dirname, '../../../../assets/toolbar');
  
  win.setThumbarButtons([
    {
      tooltip: 'Previous',
      icon: nativeImage.createFromPath(join(assetsPath, 'prev.png')),
      click: () => win.webContents.send('media:previous'),
    },
    {
      tooltip: isPlaying ? 'Pause' : 'Play',
      icon: nativeImage.createFromPath(
        join(assetsPath, isPlaying ? 'pause.png' : 'play.png')
      ),
      click: () => win.webContents.send('media:playPause'),
    },
    {
      tooltip: 'Next',
      icon: nativeImage.createFromPath(join(assetsPath, 'next.png')),
      click: () => win.webContents.send('media:next'),
    },
    {
      tooltip: isMuted ? 'Unmute' : 'Mute',
      icon: nativeImage.createFromPath(
        join(assetsPath, isMuted ? 'unmute.png' : 'mute.png')
      ),
      click: () => win.webContents.send('media:mute'),
    },
  ]);
}
```

---

## 3. Taskbar Progress

```typescript
// src/main/windows-features/taskbar-progress.ts
import { BrowserWindow } from 'electron';

export type ProgressMode = 'none' | 'normal' | 'indeterminate' | 'error' | 'paused';

export class TaskbarProgress {
  private win: BrowserWindow;
  private currentProgress: number = 0;
  private currentMode: ProgressMode = 'none';
  
  constructor(win: BrowserWindow) {
    this.win = win;
  }
  
  setProgress(progress: number, mode: ProgressMode = 'normal'): void {
    this.currentProgress = progress;
    this.currentMode = mode;
    
    if (mode === 'none') {
      this.win.setProgressBar(-1); // ซ่อน progress
      return;
    }
    
    // progress: 0.0 - 1.0
    const normalizedProgress = Math.max(0, Math.min(1, progress / 100));
    
    const modeMap: Record<ProgressMode, number> = {
      none: -1,
      normal: 0, // ใช้ progress value
      indeterminate: 2,
      error: 1,
      paused: 3,
    };
    
    this.win.setProgressBar(normalizedProgress, {
      mode: mode === 'normal' ? undefined : mode,
    });
  }
  
  show(progress: number = 0): void {
    this.setProgress(progress, 'normal');
  }
  
  showIndeterminate(): void {
    this.setProgress(0, 'indeterminate');
  }
  
  showError(): void {
    this.setProgress(this.currentProgress, 'error');
  }
  
  pause(): void {
    this.setProgress(this.currentProgress, 'paused');
  }
  
  hide(): void {
    this.setProgress(0, 'none');
  }
  
  complete(): void {
    this.setProgress(100, 'normal');
    // ซ่อนหลัง 2 วินาที
    setTimeout(() => this.hide(), 2000);
  }
}

// ตัวอย่าง
export async function downloadWithProgress(
  win: BrowserWindow,
  url: string
): Promise<void> {
  const progress = new TaskbarProgress(win);
  
  progress.showIndeterminate();
  
  try {
    // simulate download
    for (let i = 0; i <= 100; i += 10) {
      await new Promise(r => setTimeout(r, 100));
      progress.show(i);
    }
    progress.complete();
  } catch (error) {
    progress.showError();
    throw error;
  }
}
```

---

## 4. Notification Badge (Overlay Icon)

```typescript
// src/main/windows-features/overlay-icon.ts
import { BrowserWindow, nativeImage, app } from 'electron';
import { join } from 'path';

// สร้าง badge overlay icon ด้วย Canvas
async function createBadgeIcon(count: number): Promise<Electron.NativeImage> {
  const { createCanvas } = await import('canvas');
  
  const size = 16;
  const canvas = createCanvas(size, size);
  const ctx = canvas.getContext('2d');
  
  // วาด circle background
  ctx.fillStyle = '#e74c3c';
  ctx.beginPath();
  ctx.arc(size / 2, size / 2, size / 2, 0, Math.PI * 2);
  ctx.fill();
  
  // วาด text
  ctx.fillStyle = '#ffffff';
  ctx.font = 'bold 9px Arial';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  
  const text = count > 99 ? '99+' : count.toString();
  ctx.fillText(text, size / 2, size / 2);
  
  // แปลง canvas เป็น NativeImage
  const buffer = canvas.toBuffer('image/png');
  return nativeImage.createFromBuffer(buffer);
}

export async function setBadgeCount(win: BrowserWindow, count: number): Promise<void> {
  if (process.platform === 'darwin') {
    // macOS ใช้ app.setBadgeCount()
    app.setBadgeCount(count);
    return;
  }
  
  if (process.platform !== 'win32') return;
  
  if (count === 0) {
    win.setOverlayIcon(null, '');
    return;
  }
  
  const icon = await createBadgeIcon(count);
  win.setOverlayIcon(icon, `${count} notifications`);
}

// ทางเลือก: ใช้ PNG file สำหรับ badge
export function setBadgeFromFile(win: BrowserWindow, badgeType: 'new' | 'alert' | null): void {
  if (process.platform !== 'win32') return;
  
  if (!badgeType) {
    win.setOverlayIcon(null, '');
    return;
  }
  
  const iconPath = app.isPackaged
    ? join(process.resourcesPath, `assets/badges/${badgeType}.png`)
    : join(__dirname, `../../../../assets/badges/${badgeType}.png`);
  
  const icon = nativeImage.createFromPath(iconPath);
  win.setOverlayIcon(icon, badgeType);
}
```

---

## 5. Windows App User Model ID

```typescript
// src/main/windows-features/app-id.ts
import { app } from 'electron';

export function setupWindowsAppId(): void {
  if (process.platform !== 'win32') return;
  
  // ต้องตั้งก่อน app.whenReady()
  app.setAppUserModelId('com.company.myapp');
}

// App User Model ID ใช้สำหรับ:
// - Toast notifications
// - Taskbar grouping
// - Jump Lists
// - Pinned taskbar items
```

---

## 6. DWM Composition / Mica Effect

```typescript
// src/main/windows-features/mica.ts
import { BrowserWindow } from 'electron';

export function enableMicaEffect(win: BrowserWindow): void {
  if (process.platform !== 'win32') return;
  
  // Mica effect (Windows 11)
  // ต้องใช้ electron >= 25 + Windows 11
  try {
    // @ts-ignore - experimental API
    win.setBackgroundMaterial('mica');
  } catch {
    // ไม่รองรับ ให้ใช้ acrylic แทน
    try {
      // @ts-ignore
      win.setBackgroundMaterial('acrylic');
    } catch {
      console.log('Material effects not supported');
    }
  }
}

export function setupWindowsWindow(win: BrowserWindow): void {
  if (process.platform !== 'win32') return;
  
  // Shadow effect
  win.setHasShadow(true);
  
  // Custom title bar (Windows 11)
  win.setTitleBarOverlay({
    color: '#1a1a2e',
    symbolColor: '#ffffff',
    height: 32,
  });
}
```

---

## 7. Registry Access

```typescript
// src/main/windows-features/registry.ts
// ใช้ winreg package สำหรับอ่าน/เขียน Windows Registry

import Winreg from 'winreg';

export async function readRegistryValue(
  hive: string,
  key: string,
  valueName: string
): Promise<string | null> {
  if (process.platform !== 'win32') return null;
  
  return new Promise((resolve) => {
    const regKey = new Winreg({
      hive: hive as any,
      key,
    });
    
    regKey.get(valueName, (err, item) => {
      if (err) {
        resolve(null);
      } else {
        resolve(item.value);
      }
    });
  });
}

export async function writeRegistryValue(
  hive: string,
  key: string,
  valueName: string,
  type: string,
  value: string
): Promise<void> {
  if (process.platform !== 'win32') return;
  
  return new Promise((resolve, reject) => {
    const regKey = new Winreg({
      hive: hive as any,
      key,
    });
    
    regKey.set(valueName, type as any, value, (err) => {
      if (err) reject(err);
      else resolve();
    });
  });
}

// ตัวอย่าง: อ่าน Windows theme
export async function getWindowsTheme(): Promise<'light' | 'dark'> {
  const value = await readRegistryValue(
    Winreg.HKCU,
    '\\Software\\Microsoft\\Windows\\CurrentVersion\\Themes\\Personalize',
    'AppsUseLightTheme'
  );
  
  return value === '0' ? 'dark' : 'light';
}
```

---

## 8. IPC Handlers

```typescript
// src/main/ipc-windows.ts
import { ipcMain, BrowserWindow } from 'electron';
import { setupJumpList, addRecentFile } from './windows-features/jump-list';
import { TaskbarProgress } from './windows-features/taskbar-progress';
import { setBadgeCount } from './windows-features/overlay-icon';

let taskbarProgress: TaskbarProgress | null = null;

export function setupWindowsIPC(win: BrowserWindow): void {
  taskbarProgress = new TaskbarProgress(win);
  
  ipcMain.on('jumplist:addRecent', (_, path: string) => {
    addRecentFile(path);
  });
  
  ipcMain.handle('taskbar:setProgress', (_, progress: number, mode: string) => {
    taskbarProgress?.setProgress(progress, mode as any);
  });
  
  ipcMain.handle('taskbar:setBadge', async (_, count: number) => {
    await setBadgeCount(win, count);
  });
}
```

---

## 9. สรุป

### Windows-specific API

| Feature | API | Windows Version |
|---------|-----|-----------------|
| Jump Lists | `app.setJumpList()` | 7+ |
| Thumbnail Toolbar | `win.setThumbarButtons()` | 7+ |
| Taskbar Progress | `win.setProgressBar()` | 7+ |
| Overlay Icon | `win.setOverlayIcon()` | 7+ |
| Mica Effect | `win.setBackgroundMaterial('mica')` | 11 |
| Acrylic | `win.setBackgroundMaterial('acrylic')` | 10 |

### เปรียบเทียบกับ macOS

| Windows | macOS |
|---------|-------|
| Jump List | Recent Documents |
| Thumbnail Toolbar | Touch Bar |
| Taskbar Progress | Dock Progress |
| Overlay Icon | Dock Badge |
| Snap | Spaces |

---

*จบ Part 057 - ต่อไป Part 058: macOS-specific Features*
