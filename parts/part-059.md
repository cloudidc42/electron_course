# Part 059: Linux-specific Features
## ฟีเจอร์เฉพาะ Linux ใน Electron

---

## เป้าหมายของบทเรียนนี้

- AppImage packaging
- Flatpak / Snap
- D-Bus integration
- GTK Theme support
- XDG Desktop integration
- Desktop file / autostart

---

## 1. Linux Packaging Formats

### AppImage

```yaml
# electron-builder.yml
linux:
  target:
    - target: AppImage
      arch: [x64, arm64]
    - target: deb
      arch: [x64, arm64]
    - target: rpm
      arch: [x64]
    - target: snap
    - target: flatpak
  
  icon: build/icons/
  category: Office
  
  desktop:
    Name: My App
    Comment: A great application
    Keywords: editor;notes;productivity
    StartupNotify: true
    StartupWMClass: my-app
  
appImage:
  artifactName: "${productName}-${version}-${arch}.AppImage"
  license: LICENSE
  
deb:
  depends:
    - libgtk-3-0
    - libnotify4
    - libnss3
    - libxss1
    - libxtst6
    - xdg-utils
    - libatspi2.0-0
    - libuuid1
    - libsecret-1-0
  recommends:
    - libappindicator3-1
  
rpm:
  depends:
    - gtk3
    - libnotify
    - nss
    - libXScrnSaver
    - libXtst
    - xdg-utils
    - at-spi2-atk
    - libuuid
```

---

## 2. D-Bus Integration

```typescript
// src/main/linux-features/dbus.ts
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

// ส่ง D-Bus notification
export async function sendDBusNotification(
  summary: string,
  body: string,
  icon?: string,
  timeout?: number
): Promise<number> {
  if (process.platform !== 'linux') {
    throw new Error('D-Bus is only available on Linux');
  }
  
  const iconArg = icon ? `"${icon}"` : '""';
  const timeoutMs = timeout || 5000;
  
  const script = `
    dbus-send --session --dest=org.freedesktop.Notifications \
      --type=method_call --print-reply \
      /org/freedesktop/Notifications \
      org.freedesktop.Notifications.Notify \
      string:"my-app" \
      uint32:0 \
      string:${iconArg} \
      string:"${summary}" \
      string:"${body}" \
      array:string: \
      dict:string:variant: \
      int32:${timeoutMs}
  `;
  
  const { stdout } = await execAsync(script);
  // parse notification ID from response
  const match = stdout.match(/uint32 (\d+)/);
  return match ? parseInt(match[1]) : 0;
}

// ใช้ libnotify ผ่าน command
export async function libnotifyNotification(
  title: string,
  message: string,
  urgency: 'low' | 'normal' | 'critical' = 'normal',
  icon?: string
): Promise<void> {
  const iconFlag = icon ? `--icon="${icon}"` : '';
  await execAsync(
    `notify-send --urgency=${urgency} ${iconFlag} "${title}" "${message}"`
  );
}

// ดึง MPRIS media player ที่กำลังเล่น
export async function getActiveMPRISPlayer(): Promise<string | null> {
  try {
    const { stdout } = await execAsync(
      `dbus-send --session --dest=org.freedesktop.DBus \
       --type=method_call --print-reply \
       / org.freedesktop.DBus.ListNames`
    );
    
    const match = stdout.match(/org\.mpris\.MediaPlayer2\.\w+/);
    return match ? match[0] : null;
  } catch {
    return null;
  }
}
```

---

## 3. GTK Theme Detection

```typescript
// src/main/linux-features/gtk-theme.ts
import { exec } from 'child_process';
import { promisify } from 'util';
import { nativeTheme } from 'electron';

const execAsync = promisify(exec);

interface GTKThemeInfo {
  name: string;
  isDark: boolean;
  iconTheme: string;
  cursorTheme: string;
}

export async function getGTKTheme(): Promise<GTKThemeInfo> {
  if (process.platform !== 'linux') {
    return { name: '', isDark: false, iconTheme: '', cursorTheme: '' };
  }
  
  try {
    const { stdout } = await execAsync(
      'gsettings get org.gnome.desktop.interface gtk-theme'
    );
    
    const themeName = stdout.trim().replace(/'/g, '');
    const isDark = themeName.toLowerCase().includes('dark');
    
    const { stdout: iconOut } = await execAsync(
      'gsettings get org.gnome.desktop.interface icon-theme'
    );
    
    const { stdout: cursorOut } = await execAsync(
      'gsettings get org.gnome.desktop.interface cursor-theme'
    );
    
    return {
      name: themeName,
      isDark,
      iconTheme: iconOut.trim().replace(/'/g, ''),
      cursorTheme: cursorOut.trim().replace(/'/g, ''),
    };
  } catch {
    // KDE Plasma
    return await getKDETheme();
  }
}

async function getKDETheme(): Promise<GTKThemeInfo> {
  try {
    const { stdout } = await execAsync(
      'kreadconfig5 --group General --key ColorScheme'
    );
    
    const themeName = stdout.trim();
    const isDark = themeName.toLowerCase().includes('dark');
    
    return { name: themeName, isDark, iconTheme: '', cursorTheme: '' };
  } catch {
    return { name: '', isDark: nativeTheme.shouldUseDarkColors, iconTheme: '', cursorTheme: '' };
  }
}

// Watch for theme changes ผ่าน gsettings
export function watchGTKTheme(callback: (isDark: boolean) => void): () => void {
  if (process.platform !== 'linux') return () => {};
  
  const child = require('child_process').spawn('gsettings', [
    'monitor',
    'org.gnome.desktop.interface',
    'gtk-theme',
  ]);
  
  child.stdout.on('data', (data: Buffer) => {
    const theme = data.toString().trim();
    callback(theme.toLowerCase().includes('dark'));
  });
  
  return () => child.kill();
}
```

---

## 4. XDG Desktop Entry

```typescript
// src/main/linux-features/desktop-entry.ts
import { writeFileSync, mkdirSync, existsSync, rmSync } from 'fs';
import { join } from 'path';
import os from 'os';

interface DesktopEntry {
  name: string;
  genericName?: string;
  comment?: string;
  exec: string;
  icon?: string;
  categories?: string[];
  mimeTypes?: string[];
  keywords?: string[];
  startupNotify?: boolean;
  terminal?: boolean;
}

export function createDesktopEntry(entry: DesktopEntry): string {
  const content = `[Desktop Entry]
Type=Application
Version=1.1
Name=${entry.name}
${entry.genericName ? `GenericName=${entry.genericName}` : ''}
${entry.comment ? `Comment=${entry.comment}` : ''}
Exec=${entry.exec}
${entry.icon ? `Icon=${entry.icon}` : ''}
${entry.categories ? `Categories=${entry.categories.join(';')};` : ''}
${entry.mimeTypes ? `MimeType=${entry.mimeTypes.join(';')};` : ''}
${entry.keywords ? `Keywords=${entry.keywords.join(';')};` : ''}
StartupNotify=${entry.startupNotify !== false}
Terminal=${entry.terminal === true}
`;
  
  return content.replace(/\n\n/g, '\n'); // ลบ empty lines
}

// ติดตั้ง desktop entry
export function installDesktopEntry(appName: string, iconPath: string): void {
  if (process.platform !== 'linux') return;
  
  const execPath = process.execPath;
  const appId = appName.toLowerCase().replace(/\s+/g, '-');
  
  const entry = createDesktopEntry({
    name: appName,
    comment: `${appName} - Desktop Application`,
    exec: `${execPath} %U`,
    icon: iconPath || appId,
    categories: ['Utility', 'Application'],
    startupNotify: true,
  });
  
  // ติดตั้งใน user applications directory
  const appsDir = join(os.homedir(), '.local/share/applications');
  mkdirSync(appsDir, { recursive: true });
  
  const desktopFile = join(appsDir, `${appId}.desktop`);
  writeFileSync(desktopFile, entry, { mode: 0o644 });
  
  // อัพเดท desktop database
  require('child_process').exec(
    `update-desktop-database ${appsDir}`,
    (err: any) => {
      if (err) console.warn('Could not update desktop database:', err.message);
    }
  );
  
  console.log(`Desktop entry installed: ${desktopFile}`);
}

export function removeDesktopEntry(appName: string): void {
  if (process.platform !== 'linux') return;
  
  const appId = appName.toLowerCase().replace(/\s+/g, '-');
  const desktopFile = join(
    os.homedir(),
    `.local/share/applications/${appId}.desktop`
  );
  
  if (existsSync(desktopFile)) {
    rmSync(desktopFile);
  }
}
```

---

## 5. Autostart Entry

```typescript
// src/main/linux-features/autostart.ts
import { writeFileSync, mkdirSync, existsSync, rmSync } from 'fs';
import { join } from 'path';
import os from 'os';

export function setLinuxAutostart(appName: string, enable: boolean): void {
  if (process.platform !== 'linux') return;
  
  const autostartDir = join(os.homedir(), '.config/autostart');
  const appId = appName.toLowerCase().replace(/\s+/g, '-');
  const desktopFile = join(autostartDir, `${appId}.desktop`);
  
  if (enable) {
    mkdirSync(autostartDir, { recursive: true });
    
    const content = `[Desktop Entry]
Type=Application
Name=${appName}
Exec=${process.execPath} --startup
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
X-GNOME-Autostart-Delay=5
Comment=Start ${appName} on login
`;
    
    writeFileSync(desktopFile, content, { mode: 0o644 });
    console.log(`Autostart enabled: ${desktopFile}`);
  } else {
    if (existsSync(desktopFile)) {
      rmSync(desktopFile);
      console.log(`Autostart disabled`);
    }
  }
}

export function isLinuxAutostartEnabled(appName: string): boolean {
  if (process.platform !== 'linux') return false;
  
  const appId = appName.toLowerCase().replace(/\s+/g, '-');
  return existsSync(
    join(os.homedir(), `.config/autostart/${appId}.desktop`)
  );
}
```

---

## 6. MIME Type Registration

```typescript
// src/main/linux-features/mime-types.ts
import { exec } from 'child_process';
import { writeFileSync, mkdirSync } from 'fs';
import { join } from 'path';
import os from 'os';

export function registerMimeType(
  mimeType: string,
  extensions: string[],
  appDesktopFile: string
): void {
  if (process.platform !== 'linux') return;
  
  // สร้าง MIME type definition file
  const mimeContent = `<?xml version="1.0" encoding="UTF-8"?>
<mime-info xmlns="http://www.freedesktop.org/standards/shared-mime-info">
  <mime-type type="${mimeType}">
    <glob pattern="*.${extensions[0]}"/>
    ${extensions.slice(1).map(ext => `<glob pattern="*.${ext}"/>`).join('\n    ')}
    <comment>My App Document</comment>
  </mime-type>
</mime-info>`;
  
  const mimeDir = join(os.homedir(), '.local/share/mime/packages');
  mkdirSync(mimeDir, { recursive: true });
  
  const mimeFile = join(mimeDir, 'myapp.xml');
  writeFileSync(mimeFile, mimeContent);
  
  // อัพเดท MIME database
  exec(`update-mime-database ${join(os.homedir(), '.local/share/mime')}`);
  
  // ตั้ง default application
  exec(`xdg-mime default ${appDesktopFile} ${mimeType}`);
}

export async function getMimeDefaultApp(mimeType: string): Promise<string> {
  return new Promise((resolve) => {
    exec(`xdg-mime query default ${mimeType}`, (err, stdout) => {
      resolve(err ? '' : stdout.trim());
    });
  });
}
```

---

## 7. Flatpak / Snap Considerations

```typescript
// src/main/linux-features/sandbox.ts

// ตรวจสอบว่า app รันใน Flatpak/Snap sandbox
export function isInSandbox(): boolean {
  const { existsSync } = require('fs');
  
  // Flatpak
  if (existsSync('/.flatpak-info')) return true;
  
  // Snap
  if (process.env.SNAP) return true;
  
  return false;
}

export function getSandboxType(): 'flatpak' | 'snap' | 'none' {
  const { existsSync } = require('fs');
  
  if (existsSync('/.flatpak-info')) return 'flatpak';
  if (process.env.SNAP) return 'snap';
  return 'none';
}

// Flatpak portal สำหรับ file access
export async function flatpakOpenFileDialog(): Promise<string[]> {
  if (getSandboxType() !== 'flatpak') {
    // ใช้ Electron dialog ปกติ
    const { dialog } = require('electron');
    const result = await dialog.showOpenDialog({});
    return result.filePaths;
  }
  
  // ใช้ xdg-open portal สำหรับ Flatpak
  return new Promise((resolve) => {
    const { spawn } = require('child_process');
    const proc = spawn('flatpak-spawn', ['--host', 'zenity', '--file-selection']);
    
    let output = '';
    proc.stdout.on('data', (data: Buffer) => {
      output += data.toString();
    });
    
    proc.on('close', () => {
      const files = output.trim().split('\n').filter(Boolean);
      resolve(files);
    });
  });
}
```

---

## 8. System Tray บน Linux

```typescript
// src/main/linux-features/tray.ts
import { Tray, Menu, nativeImage } from 'electron';
import { join } from 'path';

export function createLinuxTray(): Tray {
  // Linux อาจต้องการ libappindicator
  // ถ้าไม่มี จะ fallback ไป XEmbed
  
  const iconPath = getLinuxTrayIcon();
  const icon = nativeImage.createFromPath(iconPath);
  
  const tray = new Tray(icon);
  tray.setToolTip('My App');
  
  // Linux tray ต้องมี context menu เสมอ
  // (คลิกซ้ายไม่ emit 'click' บน GNOME)
  const menu = Menu.buildFromTemplate([
    { label: 'Open', click: () => showWindow() },
    { type: 'separator' },
    { label: 'Quit', click: () => app.quit() },
  ]);
  
  tray.setContextMenu(menu);
  
  return tray;
}

function getLinuxTrayIcon(): string {
  const basePath = process.env.NODE_ENV === 'development'
    ? join(__dirname, '../../../assets/icons')
    : join(process.resourcesPath, 'assets/icons');
  
  // ลองใช้ SVG ก่อน (GNOME รองรับ)
  return join(basePath, 'tray.png'); // หรือ .svg
}

// ตรวจสอบว่า tray รองรับหรือไม่
export function isTraySupported(): boolean {
  if (process.platform !== 'linux') return true;
  
  // ตรวจสอบ environment
  const desktop = process.env.XDG_CURRENT_DESKTOP || '';
  const supportedDesktops = ['GNOME', 'KDE', 'XFCE', 'LXDE', 'Unity', 'Cinnamon'];
  
  return supportedDesktops.some(d => desktop.toUpperCase().includes(d));
}
```

---

## 9. IPC Handlers

```typescript
// src/main/ipc-linux.ts
import { ipcMain, app } from 'electron';
import { installDesktopEntry } from './linux-features/desktop-entry';
import { setLinuxAutostart, isLinuxAutostartEnabled } from './linux-features/autostart';
import { getGTKTheme } from './linux-features/gtk-theme';

const APP_NAME = app.getName();

export function setupLinuxIPC(): void {
  if (process.platform !== 'linux') return;
  
  ipcMain.handle('linux:getTheme', async () => {
    return getGTKTheme();
  });
  
  ipcMain.handle('linux:getAutostartEnabled', () => {
    return isLinuxAutostartEnabled(APP_NAME);
  });
  
  ipcMain.handle('linux:setAutostart', (_, enable: boolean) => {
    setLinuxAutostart(APP_NAME, enable);
  });
  
  ipcMain.handle('linux:installDesktopEntry', (_, iconPath: string) => {
    installDesktopEntry(APP_NAME, iconPath);
  });
}
```

---

## 10. สรุป

### Linux Package Formats

| Format | ใช้กับ | ข้อดี | ข้อเสีย |
|--------|--------|-------|---------|
| AppImage | ทุก distro | portable | ไม่มี auto-update |
| deb | Ubuntu/Debian | apt integration | เฉพาะ Debian-based |
| rpm | Fedora/RHEL | dnf integration | เฉพาะ RPM-based |
| Snap | Ubuntu | sandboxed | ช้า startup |
| Flatpak | ทุก distro | sandboxed | file access จำกัด |

### Checklist

- [ ] สร้าง desktop entry file
- [ ] Register MIME types ถ้าต้องการ
- [ ] รองรับ D-Bus notifications
- [ ] ตรวจสอบ GTK theme สำหรับ Dark Mode
- [ ] Handle Flatpak/Snap sandbox
- [ ] Tray icon ด้วย context menu เสมอ

---

*จบ Part 059 - ต่อไป Part 060: Electron Security Audit*
