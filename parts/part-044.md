# Part 044: Vite + Electron (electron-vite)
## สร้างแอพ Electron ด้วย Vite สำหรับ Build ที่รวดเร็ว

---

## 🎯 เป้าหมายของบทเรียนนี้

- ติดตั้งและตั้งค่า electron-vite
- vite.config.ts สำหรับทุก process
- HMR ใน renderer
- Static assets
- Preload script bundling
- Build optimization

---

## 1. สร้างโปรเจกต์ใหม่

```bash
# ใช้ electron-vite template
npm create electron-vite@latest my-app

# เลือก template:
# ✔ Select a framework: › React
# ✔ Add TypeScript? Yes

cd my-app
npm install
npm run dev
```

---

## 2. โครงสร้างโปรเจกต์ electron-vite

```
my-app/
├── src/
│   ├── main/
│   │   └── index.ts          # Main process
│   ├── preload/
│   │   └── index.ts          # Preload script
│   └── renderer/
│       ├── index.html
│       ├── main.tsx
│       └── App.tsx
├── electron.vite.config.ts   # Vite config หลัก
├── tsconfig.json
├── tsconfig.node.json
└── package.json
```

---

## 3. electron.vite.config.ts

```typescript
import { resolve } from 'path';
import { defineConfig, externalizeDepsPlugin } from 'electron-vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  // ==========================================
  // Main Process Config
  // ==========================================
  main: {
    plugins: [
      // Externalize node_modules ไม่ให้ bundle
      externalizeDepsPlugin(),
    ],
    
    build: {
      outDir: 'out/main',
      
      rollupOptions: {
        input: {
          index: resolve(__dirname, 'src/main/index.ts'),
        },
        output: {
          format: 'cjs',
        },
        // External native modules
        external: ['better-sqlite3', 'sharp', 'canvas'],
      },
      
      // Sourcemaps สำหรับ debug
      sourcemap: true,
      
      // ไม่ minify ใน development
      minify: false,
    },
    
    resolve: {
      alias: {
        '@main': resolve('src/main'),
        '@shared': resolve('src/shared'),
      },
    },
    
    // Define constants
    define: {
      __MAIN_VERSION__: JSON.stringify(process.env.npm_package_version),
    },
  },
  
  // ==========================================
  // Preload Script Config
  // ==========================================
  preload: {
    plugins: [
      externalizeDepsPlugin(),
    ],
    
    build: {
      outDir: 'out/preload',
      
      rollupOptions: {
        input: {
          index: resolve(__dirname, 'src/preload/index.ts'),
        },
        output: {
          format: 'cjs',
        },
      },
      
      sourcemap: true,
      minify: false,
    },
    
    resolve: {
      alias: {
        '@shared': resolve('src/shared'),
      },
    },
  },
  
  // ==========================================
  // Renderer Process Config
  // ==========================================
  renderer: {
    root: 'src/renderer',
    
    plugins: [
      react({
        // Fast Refresh สำหรับ HMR
        fastRefresh: true,
      }),
    ],
    
    build: {
      outDir: 'out/renderer',
      
      rollupOptions: {
        input: {
          index: resolve(__dirname, 'src/renderer/index.html'),
        },
        
        output: {
          // Code splitting
          manualChunks(id) {
            if (id.includes('node_modules')) {
              if (id.includes('react') || id.includes('react-dom')) {
                return 'react';
              }
              if (id.includes('react-router')) {
                return 'react-router';
              }
              return 'vendor';
            }
          },
        },
      },
      
      // เพิ่ม CSS code splitting
      cssCodeSplit: true,
      
      // Sourcemaps
      sourcemap: true,
      
      // Warning threshold
      chunkSizeWarningLimit: 1000,
    },
    
    resolve: {
      alias: {
        '@renderer': resolve('src/renderer'),
        '@components': resolve('src/renderer/components'),
        '@hooks': resolve('src/renderer/hooks'),
        '@utils': resolve('src/renderer/utils'),
        '@assets': resolve('src/renderer/assets'),
        '@shared': resolve('src/shared'),
        '@styles': resolve('src/renderer/styles'),
      },
    },
    
    // CSS preprocessor
    css: {
      preprocessorOptions: {
        scss: {
          additionalData: `@use "@styles/variables" as *;`,
        },
      },
      
      modules: {
        localsConvention: 'camelCase',
        generateScopedName: '[name]__[local]__[hash:base64:5]',
      },
    },
    
    // Static assets
    assetsInclude: ['**/*.lottie', '**/*.glb', '**/*.gltf'],
    
    // Dev server
    server: {
      port: 5173,
      strictPort: true,
      hmr: {
        port: 5173,
      },
    },
    
    // Preview server
    preview: {
      port: 5174,
      strictPort: true,
    },
    
    // Optimize dependencies
    optimizeDeps: {
      include: ['react', 'react-dom', 'react-router-dom'],
      exclude: ['electron'],
    },
  },
});
```

---

## 4. Main Process

### src/main/index.ts

```typescript
import { app, BrowserWindow, shell } from 'electron';
import { join } from 'path';
import { electronApp, optimizer, is } from '@electron-toolkit/utils';

function createWindow(): void {
  const mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    show: false,
    autoHideMenuBar: true,
    
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      sandbox: false,
      contextIsolation: true,
      nodeIntegration: false,
    },
  });

  // Show window เมื่อ ready
  mainWindow.on('ready-to-show', () => {
    mainWindow.show();
  });

  // เปิด external links ใน browser
  mainWindow.webContents.setWindowOpenHandler(({ url }) => {
    shell.openExternal(url);
    return { action: 'deny' };
  });

  // HMR สำหรับ development
  if (is.dev && process.env['ELECTRON_RENDERER_URL']) {
    mainWindow.loadURL(process.env['ELECTRON_RENDERER_URL']);
    mainWindow.webContents.openDevTools();
  } else {
    mainWindow.loadFile(join(__dirname, '../renderer/index.html'));
  }
}

app.whenReady().then(() => {
  // Set app user model id for windows
  electronApp.setAppUserModelId('com.yourcompany.myapp');

  // Watch renderer file changes in dev
  app.on('browser-window-created', (_, window) => {
    optimizer.watchWindowShortcuts(window);
  });

  createWindow();

  app.on('activate', function () {
    if (BrowserWindow.getAllWindows().length === 0) createWindow();
  });
});

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});
```

---

## 5. Preload Script

### src/preload/index.ts

```typescript
import { contextBridge, ipcRenderer } from 'electron';
import { electronAPI } from '@electron-toolkit/preload';

// Expose electron API
contextBridge.exposeInMainWorld('electron', electronAPI);

// Custom API
contextBridge.exposeInMainWorld('api', {
  // ตัวอย่าง custom APIs
  readFile: (path: string) => ipcRenderer.invoke('file:read', path),
  writeFile: (path: string, content: string) => ipcRenderer.invoke('file:write', { path, content }),
  
  // Events
  onUpdate: (callback: (info: any) => void) => {
    const handler = (_: any, info: any) => callback(info);
    ipcRenderer.on('update:available', handler);
    return () => ipcRenderer.removeListener('update:available', handler);
  },
});
```

---

## 6. Renderer

### src/renderer/index.html

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My Electron App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/main.tsx"></script>
  </body>
</html>
```

### src/renderer/main.tsx

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './styles/global.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### src/renderer/App.tsx

```tsx
import React, { useState, useEffect } from 'react';
import styles from './App.module.scss';

// Import static assets
import logoUrl from './assets/logo.png';

// Import SVG as component
import { ReactComponent as IconMenu } from './assets/icons/menu.svg?react';

function App() {
  const [version, setVersion] = useState('');

  useEffect(() => {
    // ใช้ electron API จาก preload
    if (window.electron) {
      setVersion(window.electron.process.versions.electron);
    }
  }, []);

  return (
    <div className={styles.app}>
      <header className={styles.header}>
        <img src={logoUrl} alt="Logo" className={styles.logo} />
        <h1>My Electron App</h1>
        <IconMenu className={styles.menuIcon} />
      </header>
      <main>
        <p>Electron v{version}</p>
      </main>
    </div>
  );
}

export default App;
```

---

## 7. Environment Variables

### .env.development

```env
VITE_APP_TITLE=My Electron App (Dev)
VITE_API_URL=http://localhost:3001
VITE_ENABLE_DEVTOOLS=true
```

### .env.production

```env
VITE_APP_TITLE=My Electron App
VITE_API_URL=https://api.yourapp.com
VITE_ENABLE_DEVTOOLS=false
```

### การใช้งาน

```typescript
// ใน renderer
const title = import.meta.env.VITE_APP_TITLE;
const isDev = import.meta.env.DEV;
const isProd = import.meta.env.PROD;
const mode = import.meta.env.MODE; // 'development' | 'production'
```

---

## 8. Vite Plugins ที่มีประโยชน์

```typescript
// electron.vite.config.ts
import { defineConfig } from 'electron-vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';
import { VitePWA } from 'vite-plugin-pwa';
import checker from 'vite-plugin-checker';
import svgr from 'vite-plugin-svgr';

export default defineConfig({
  renderer: {
    plugins: [
      react(),
      
      // SVG as React components
      svgr(),
      
      // TypeScript type checking
      checker({
        typescript: true,
        eslint: {
          lintCommand: 'eslint "./src/**/*.{ts,tsx}"',
        },
      }),
      
      // Bundle analyzer
      visualizer({
        filename: 'dist/bundle-stats.html',
        open: true,
        gzipSize: true,
        brotliSize: true,
      }),
    ],
  },
});
```

---

## 9. package.json Scripts

```json
{
  "scripts": {
    "dev": "electron-vite dev",
    "build": "electron-vite build",
    "preview": "electron-vite preview",
    
    "typecheck": "tsc --noEmit",
    "typecheck:node": "tsc --noEmit -p tsconfig.node.json",
    
    "package": "npm run build && electron-builder",
    "package:win": "npm run build && electron-builder --win",
    "package:mac": "npm run build && electron-builder --mac",
    "package:linux": "npm run build && electron-builder --linux"
  }
}
```

---

## 10. สรุป

### Vite vs Webpack สำหรับ Electron

| Feature | Vite | Webpack |
|---------|------|---------|
| Dev start time | < 1s | 5-30s |
| HMR | เร็วมาก | ช้ากว่า |
| Config | ง่ายกว่า | ยืดหยุ่นกว่า |
| Plugin ecosystem | กำลังเติบโต | ใหญ่มาก |
| TypeScript | First-class | ต้องตั้งค่า |
| Production build | ดี | ดีมาก |

### ข้อดีของ electron-vite

- Dev mode เร็วมาก (Native ES modules)
- HMR สำหรับ main และ renderer
- TypeScript support ในตัว
- Config สำหรับทุก process ในไฟล์เดียว
- Community plugins มากมาย

---

*จบ Part 044 - ต่อไป Part 045: Custom Protocol Deep Dive*
