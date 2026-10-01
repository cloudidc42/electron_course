# Part 042: TypeScript Integration
## การใช้ TypeScript กับ Electron อย่างครบถ้วน

---

## 🎯 เป้าหมายของบทเรียนนี้

- tsconfig สำหรับ Electron
- Type-safe IPC ด้วย typed channels
- electron-ts-ipc pattern
- Declaration files
- Paths mapping

---

## 1. โครงสร้างโปรเจกต์ TypeScript

```
src/
├── main/
│   ├── index.ts
│   ├── ipc-handlers.ts
│   └── tsconfig.json
├── renderer/
│   ├── index.tsx
│   ├── App.tsx
│   └── tsconfig.json
├── preload/
│   ├── preload.ts
│   └── tsconfig.json
└── shared/
    ├── types.ts
    ├── ipc-channels.ts
    └── constants.ts
```

---

## 2. tsconfig สำหรับแต่ละ Process

### tsconfig.base.json (root)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "baseUrl": ".",
    "paths": {
      "@shared/*": ["src/shared/*"],
      "@main/*": ["src/main/*"],
      "@renderer/*": ["src/renderer/*"],
      "@preload/*": ["src/preload/*"]
    }
  }
}
```

### src/main/tsconfig.json

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "outDir": "../../dist/main",
    "rootDir": "../",
    "lib": ["ES2022"],
    "types": ["node"]
  },
  "include": [
    "./**/*",
    "../shared/**/*"
  ],
  "exclude": [
    "node_modules",
    "**/*.spec.ts",
    "**/*.test.ts"
  ]
}
```

### src/renderer/tsconfig.json

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "outDir": "../../dist/renderer",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "types": ["react", "react-dom"],
    "noEmit": true
  },
  "include": [
    "./**/*",
    "../shared/**/*"
  ]
}
```

### src/preload/tsconfig.json

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "outDir": "../../dist/preload",
    "lib": ["ES2022", "DOM"],
    "types": ["node"]
  },
  "include": [
    "./**/*",
    "../shared/**/*"
  ]
}
```

---

## 3. Shared Types

### src/shared/types.ts

```typescript
// ==========================================
// App State Types
// ==========================================

export interface AppConfig {
  theme: 'light' | 'dark' | 'system';
  language: string;
  autoUpdate: boolean;
  notifications: boolean;
  fontSize: number;
  windowBounds?: WindowBounds;
}

export interface WindowBounds {
  x: number;
  y: number;
  width: number;
  height: number;
  maximized: boolean;
}

// ==========================================
// File Types
// ==========================================

export interface FileInfo {
  path: string;
  name: string;
  size: number;
  extension: string;
  modifiedAt: Date;
  isDirectory: boolean;
}

export interface FileReadResult {
  success: boolean;
  content?: string;
  encoding?: string;
  error?: string;
}

export interface FileWriteResult {
  success: boolean;
  path?: string;
  error?: string;
}

// ==========================================
// System Types
// ==========================================

export interface SystemInfo {
  platform: NodeJS.Platform;
  arch: string;
  version: string;
  electronVersion: string;
  nodeVersion: string;
  chromeVersion: string;
  memory: {
    total: number;
    free: number;
    used: number;
  };
  cpu: {
    model: string;
    speed: number;
    cores: number;
  };
}

// ==========================================
// Update Types
// ==========================================

export interface UpdateInfo {
  version: string;
  releaseDate: string;
  releaseNotes?: string;
  size?: number;
}

export interface UpdateProgress {
  percent: number;
  transferred: number;
  total: number;
  bytesPerSecond: number;
}

export type UpdateStatus =
  | 'checking'
  | 'available'
  | 'not-available'
  | 'downloading'
  | 'downloaded'
  | 'error';

// ==========================================
// Notification Types
// ==========================================

export interface NotificationOptions {
  title: string;
  body: string;
  icon?: string;
  urgency?: 'low' | 'normal' | 'critical';
  timeoutType?: 'default' | 'never';
  actions?: NotificationAction[];
}

export interface NotificationAction {
  type: 'button';
  text: string;
}

// ==========================================
// API Response Types
// ==========================================

export type ApiResult<T> = 
  | { success: true; data: T }
  | { success: false; error: string; code?: string };

export function isSuccess<T>(result: ApiResult<T>): result is { success: true; data: T } {
  return result.success === true;
}

export function isError<T>(result: ApiResult<T>): result is { success: false; error: string } {
  return result.success === false;
}
```

---

## 4. Type-safe IPC Channels

### src/shared/ipc-channels.ts

```typescript
import type { 
  AppConfig, 
  FileInfo, 
  FileReadResult, 
  FileWriteResult,
  SystemInfo,
  UpdateInfo,
  UpdateProgress,
  UpdateStatus,
  NotificationOptions,
  ApiResult 
} from './types';

// ==========================================
// IPC Channel Definitions
// ==========================================

// Channel definition: { request params } -> { response }
export interface IpcChannels {
  // App
  'app:getConfig': { request: void; response: AppConfig };
  'app:setConfig': { request: Partial<AppConfig>; response: AppConfig };
  'app:getSystemInfo': { request: void; response: SystemInfo };
  'app:quit': { request: void; response: void };
  'app:minimize': { request: void; response: void };
  'app:maximize': { request: void; response: void };
  
  // Files
  'file:read': { request: string; response: FileReadResult };
  'file:write': { request: { path: string; content: string }; response: FileWriteResult };
  'file:delete': { request: string; response: ApiResult<void> };
  'file:list': { request: string; response: ApiResult<FileInfo[]> };
  'file:openDialog': { request: { filters?: Electron.FileFilter[]; properties?: string[] }; response: string[] };
  'file:saveDialog': { request: { filters?: Electron.FileFilter[]; defaultPath?: string }; response: string | null };
  
  // Updates
  'update:check': { request: void; response: UpdateInfo | null };
  'update:download': { request: void; response: void };
  'update:install': { request: void; response: void };
  
  // Notifications
  'notification:show': { request: NotificationOptions; response: void };
}

// ==========================================
// IPC Event Channels (main -> renderer)
// ==========================================

export interface IpcEvents {
  'update:status': UpdateStatus;
  'update:progress': UpdateProgress;
  'update:available': UpdateInfo;
  'update:downloaded': UpdateInfo;
  'app:config-changed': AppConfig;
  'app:theme-changed': 'light' | 'dark';
  'file:external-change': { path: string; eventType: 'change' | 'rename' };
}

// ==========================================
// Type Helpers
// ==========================================

export type IpcChannelName = keyof IpcChannels;
export type IpcEventName = keyof IpcEvents;

export type IpcRequest<K extends IpcChannelName> = IpcChannels[K]['request'];
export type IpcResponse<K extends IpcChannelName> = IpcChannels[K]['response'];
export type IpcEventData<K extends IpcEventName> = IpcEvents[K];
```

---

## 5. Type-safe Main Process IPC

### src/main/ipc-handlers.ts

```typescript
import { ipcMain, app, dialog, BrowserWindow, Notification } from 'electron';
import type { IpcChannels, IpcChannelName, IpcRequest, IpcResponse } from '@shared/ipc-channels';

// Type-safe handler registration
type IpcHandler<K extends IpcChannelName> = (
  event: Electron.IpcMainInvokeEvent,
  request: IpcRequest<K>
) => Promise<IpcResponse<K>> | IpcResponse<K>;

function handle<K extends IpcChannelName>(
  channel: K,
  handler: IpcHandler<K>
): void {
  ipcMain.handle(channel, handler);
}

// ==========================================
// Register all handlers
// ==========================================

export function registerIpcHandlers(): void {
  // App handlers
  handle('app:getConfig', async () => {
    const config = await loadConfig();
    return config;
  });

  handle('app:setConfig', async (_, partialConfig) => {
    const config = await saveConfig(partialConfig);
    
    // Notify all windows
    BrowserWindow.getAllWindows().forEach(win => {
      win.webContents.send('app:config-changed', config);
    });
    
    return config;
  });

  handle('app:getSystemInfo', () => {
    return {
      platform: process.platform,
      arch: process.arch,
      version: app.getVersion(),
      electronVersion: process.versions.electron,
      nodeVersion: process.versions.node,
      chromeVersion: process.versions.chrome,
      memory: {
        total: process.getSystemMemoryInfo().total,
        free: process.getSystemMemoryInfo().free,
        used: process.getSystemMemoryInfo().total - process.getSystemMemoryInfo().free,
      },
      cpu: {
        model: require('os').cpus()[0]?.model || 'Unknown',
        speed: require('os').cpus()[0]?.speed || 0,
        cores: require('os').cpus().length,
      },
    };
  });

  handle('app:quit', () => {
    app.quit();
  });

  // File handlers
  handle('file:read', async (_, filePath) => {
    const fs = require('fs').promises;
    try {
      const content = await fs.readFile(filePath, 'utf-8');
      return { success: true, content, encoding: 'utf-8' };
    } catch (error: any) {
      return { success: false, error: error.message };
    }
  });

  handle('file:write', async (_, { path: filePath, content }) => {
    const fs = require('fs').promises;
    try {
      await fs.writeFile(filePath, content, 'utf-8');
      return { success: true, path: filePath };
    } catch (error: any) {
      return { success: false, error: error.message };
    }
  });

  handle('file:openDialog', async (event, options) => {
    const win = BrowserWindow.fromWebContents(event.sender);
    const result = await dialog.showOpenDialog(win!, {
      filters: options.filters,
      properties: (options.properties || ['openFile']) as Electron.OpenDialogOptions['properties'],
    });
    return result.filePaths;
  });

  handle('file:saveDialog', async (event, options) => {
    const win = BrowserWindow.fromWebContents(event.sender);
    const result = await dialog.showSaveDialog(win!, {
      filters: options.filters,
      defaultPath: options.defaultPath,
    });
    return result.canceled ? null : result.filePath || null;
  });

  // Notification handlers
  handle('notification:show', (_, options) => {
    const notification = new Notification({
      title: options.title,
      body: options.body,
      icon: options.icon,
    });
    notification.show();
  });
}

// Helper functions
async function loadConfig(): Promise<any> {
  // Load from electron-store or similar
  return {
    theme: 'system',
    language: 'en',
    autoUpdate: true,
    notifications: true,
    fontSize: 14,
  };
}

async function saveConfig(partial: any): Promise<any> {
  const current = await loadConfig();
  return { ...current, ...partial };
}
```

---

## 6. Type-safe Preload

### src/preload/preload.ts

```typescript
import { contextBridge, ipcRenderer } from 'electron';
import type { 
  IpcChannels, 
  IpcChannelName, 
  IpcRequest, 
  IpcResponse,
  IpcEvents,
  IpcEventName,
  IpcEventData
} from '@shared/ipc-channels';

// Type-safe invoke
async function invoke<K extends IpcChannelName>(
  channel: K,
  ...args: IpcRequest<K> extends void ? [] : [IpcRequest<K>]
): Promise<IpcResponse<K>> {
  return ipcRenderer.invoke(channel, ...args);
}

// Type-safe event listener
function on<K extends IpcEventName>(
  channel: K,
  callback: (data: IpcEventData<K>) => void
): () => void {
  const handler = (_event: Electron.IpcRendererEvent, data: IpcEventData<K>) => {
    callback(data);
  };
  
  ipcRenderer.on(channel, handler);
  
  // Return cleanup function
  return () => {
    ipcRenderer.removeListener(channel, handler);
  };
}

// ==========================================
// Expose APIs
// ==========================================

export type ElectronAPI = typeof electronAPI;

const electronAPI = {
  // App
  app: {
    getConfig: () => invoke('app:getConfig'),
    setConfig: (config: IpcRequest<'app:setConfig'>) => invoke('app:setConfig', config),
    getSystemInfo: () => invoke('app:getSystemInfo'),
    quit: () => invoke('app:quit'),
    minimize: () => invoke('app:minimize'),
    maximize: () => invoke('app:maximize'),
  },
  
  // Files
  files: {
    read: (path: string) => invoke('file:read', path),
    write: (path: string, content: string) => invoke('file:write', { path, content }),
    delete: (path: string) => invoke('file:delete', path),
    list: (dir: string) => invoke('file:list', dir),
    openDialog: (options?: IpcRequest<'file:openDialog'>) => 
      invoke('file:openDialog', options || {}),
    saveDialog: (options?: IpcRequest<'file:saveDialog'>) => 
      invoke('file:saveDialog', options || {}),
  },
  
  // Updates
  updates: {
    check: () => invoke('update:check'),
    download: () => invoke('update:download'),
    install: () => invoke('update:install'),
  },
  
  // Notifications
  notifications: {
    show: (options: IpcRequest<'notification:show'>) => 
      invoke('notification:show', options),
  },
  
  // Events
  events: {
    onUpdateStatus: (cb: (status: IpcEventData<'update:status'>) => void) => 
      on('update:status', cb),
    onUpdateProgress: (cb: (progress: IpcEventData<'update:progress'>) => void) => 
      on('update:progress', cb),
    onConfigChanged: (cb: (config: IpcEventData<'app:config-changed'>) => void) => 
      on('app:config-changed', cb),
    onThemeChanged: (cb: (theme: IpcEventData<'app:theme-changed'>) => void) => 
      on('app:theme-changed', cb),
  },
};

contextBridge.exposeInMainWorld('electron', electronAPI);
```

---

## 7. Type Declaration สำหรับ Renderer

### src/renderer/electron.d.ts

```typescript
import type { ElectronAPI } from '../preload/preload';

declare global {
  interface Window {
    electron: ElectronAPI;
  }
}

export {};
```

---

## 8. ใช้งานใน React Component

### src/renderer/App.tsx

```tsx
import React, { useState, useEffect } from 'react';
import type { AppConfig, SystemInfo } from '@shared/types';

function App() {
  const [config, setConfig] = useState<AppConfig | null>(null);
  const [systemInfo, setSystemInfo] = useState<SystemInfo | null>(null);

  useEffect(() => {
    // โหลด config
    window.electron.app.getConfig().then(setConfig);
    window.electron.app.getSystemInfo().then(setSystemInfo);
    
    // Listen for config changes
    const cleanup = window.electron.events.onConfigChanged((newConfig) => {
      setConfig(newConfig);
    });
    
    // Listen for theme changes
    const themeCleanup = window.electron.events.onThemeChanged((theme) => {
      document.documentElement.setAttribute('data-theme', theme);
    });
    
    return () => {
      cleanup();
      themeCleanup();
    };
  }, []);

  const handleThemeChange = async (theme: AppConfig['theme']) => {
    const newConfig = await window.electron.app.setConfig({ theme });
    setConfig(newConfig);
  };

  const handleOpenFile = async () => {
    const paths = await window.electron.files.openDialog({
      filters: [
        { name: 'Text Files', extensions: ['txt', 'md'] },
        { name: 'All Files', extensions: ['*'] },
      ],
    });
    
    if (paths.length > 0) {
      const result = await window.electron.files.read(paths[0]);
      if (result.success) {
        console.log('File content:', result.content);
      }
    }
  };

  if (!config) return <div>Loading...</div>;

  return (
    <div>
      <h1>My Electron App</h1>
      
      <div>
        <h2>Theme</h2>
        <select 
          value={config.theme}
          onChange={(e) => handleThemeChange(e.target.value as AppConfig['theme'])}
        >
          <option value="light">Light</option>
          <option value="dark">Dark</option>
          <option value="system">System</option>
        </select>
      </div>
      
      {systemInfo && (
        <div>
          <h2>System Info</h2>
          <p>Platform: {systemInfo.platform}</p>
          <p>Electron: {systemInfo.electronVersion}</p>
          <p>Node: {systemInfo.nodeVersion}</p>
        </div>
      )}
      
      <button onClick={handleOpenFile}>Open File</button>
    </div>
  );
}

export default App;
```

---

## 9. สรุป

### Pattern สำคัญ

1. **Shared Types** - กำหนด types ใน `shared/` ให้ทุก process ใช้ร่วมกัน
2. **IPC Channel Map** - ใช้ TypeScript interface เพื่อ type-safe IPC
3. **Generic Helpers** - `invoke<K>()` และ `on<K>()` สำหรับ type inference
4. **Declaration Files** - `electron.d.ts` ใน renderer เพื่อรู้จัก `window.electron`
5. **Path Aliases** - `@shared/*`, `@main/*`, `@renderer/*` เพื่อ import ง่ายขึ้น

---

*จบ Part 042 - ต่อไป Part 043: Webpack Configuration*
