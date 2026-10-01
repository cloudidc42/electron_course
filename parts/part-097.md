# Part 97: Build Optimization

## ปรับปรุง Build Size และ Performance

ในบทนี้เราจะเรียน tree shaking, ASAR optimization, bundle analysis, code splitting และ startup performance

---

## 1. Bundle Analysis

```bash
# ติดตั้ง rollup-plugin-visualizer
npm install -D rollup-plugin-visualizer

# วิเคราะห์ bundle
npm run build -- --analyze
```

```typescript
// electron.vite.config.ts
import { defineConfig } from 'electron-vite'
import react from '@vitejs/plugin-react'
import { visualizer } from 'rollup-plugin-visualizer'

export default defineConfig({
  renderer: {
    plugins: [
      react(),
      // เฉพาะเมื่อ analyze
      ...(process.env.ANALYZE ? [
        visualizer({
          filename: 'dist/stats.html',
          open: true,
          gzipSize: true,
          brotliSize: true
        })
      ] : [])
    ],
    build: {
      rollupOptions: {
        output: {
          // Manual chunk splitting
          manualChunks: {
            // Vendor chunks
            'vendor-react': ['react', 'react-dom'],
            'vendor-monaco': ['monaco-editor'],
            'vendor-charts': ['chart.js'],
            'vendor-utils': ['lodash-es', 'date-fns']
          }
        }
      },
      // Minification
      minify: 'terser',
      terserOptions: {
        compress: {
          drop_console: process.env.NODE_ENV === 'production',
          drop_debugger: true,
          pure_funcs: ['console.log', 'console.info']
        }
      },
      // Source maps สำหรับ production debugging
      sourcemap: process.env.NODE_ENV === 'production' ? 'hidden' : true,
      // Chunk size warning
      chunkSizeWarningLimit: 1000  // 1MB
    }
  },
  main: {
    build: {
      rollupOptions: {
        external: [
          // Node.js built-ins ไม่ต้อง bundle
          'path', 'fs', 'os', 'crypto', 'http', 'https', 'stream', 'buffer',
          'events', 'url', 'util', 'child_process', 'worker_threads',
          // Electron
          'electron',
          // Native modules
          'better-sqlite3', 'sharp', 'argon2', 'keytar'
        ]
      }
    }
  }
})
```

---

## 2. Lazy Loading Routes

```tsx
// src/renderer/App.tsx
import { Suspense, lazy } from 'react'
import { createHashRouter, RouterProvider } from 'react-router-dom'

// Route-based code splitting
const Dashboard = lazy(() => import('./pages/Dashboard'))
const Editor = lazy(() => import('./pages/Editor'))
const FileManager = lazy(() => import('./pages/FileManager'))
const Settings = lazy(() => import('./pages/Settings'))
const Analytics = lazy(() => import('./pages/Analytics'))

const LoadingFallback = () => (
  <div className="page-loading">
    <div className="spinner" />
  </div>
)

const router = createHashRouter([
  {
    path: '/',
    lazy: async () => {
      const { Layout } = await import('./components/Layout')
      return { Component: Layout }
    },
    children: [
      { index: true, element: <Suspense fallback={<LoadingFallback />}><Dashboard /></Suspense> },
      { path: 'editor', element: <Suspense fallback={<LoadingFallback />}><Editor /></Suspense> },
      { path: 'files', element: <Suspense fallback={<LoadingFallback />}><FileManager /></Suspense> },
      { path: 'settings', element: <Suspense fallback={<LoadingFallback />}><Settings /></Suspense> },
      { path: 'analytics', element: <Suspense fallback={<LoadingFallback />}><Analytics /></Suspense> }
    ]
  }
])

export function App() {
  return <RouterProvider router={router} />
}
```

---

## 3. ASAR Optimization

```javascript
// electron-builder.json5
{
  "asar": true,
  "asarUnpack": [
    // Native modules ต้องอยู่นอก ASAR
    "node_modules/better-sqlite3/**",
    "node_modules/sharp/**",
    "node_modules/argon2/**",
    "node_modules/keytar/**",
    // WASM files
    "**/*.wasm",
    // Large binary files
    "resources/models/**"
  ],
  "compression": "maximum",
  "files": [
    "out/**/*",
    "node_modules/**",
    "!node_modules/**/{CHANGELOG.md,README.md,readme.md,CHANGES.md}",
    "!node_modules/**/{test,tests,__tests__,spec,specs}/**",
    "!node_modules/**/*.{ts,map}",
    "!node_modules/**/.*",
    "!node_modules/**/Makefile"
  ]
}
```

---

## 4. Startup Performance

```typescript
// src/main/startup.ts
// ปรับ startup time ให้เร็วขึ้น

import { app, BrowserWindow } from 'electron'

export async function optimizedStartup(): Promise<BrowserWindow> {
  // 1. สร้าง window เร็วที่สุด
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    show: false,  // ซ่อนก่อน เพื่อป้องกัน flash
    // preload ที่เบาที่สุด
    webPreferences: {
      preload: require('path').join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
      backgroundThrottling: false  // ไม่ throttle เมื่อ background
    }
  })

  // 2. โหลด URL ทันที (ไม่รอ initialize handlers)
  const loadPromise = process.env.NODE_ENV === 'development'
    ? win.loadURL('http://localhost:5173')
    : win.loadFile(require('path').join(__dirname, '../renderer/index.html'))

  // 3. Initialize handlers พร้อมกัน
  const initPromise = initializeHandlers(win)

  // 4. รอทั้งคู่
  await Promise.all([loadPromise, initPromise])

  // 5. แสดง window
  win.show()
  win.focus()

  return win
}

async function initializeHandlers(win: BrowserWindow): Promise<void> {
  // Import เฉพาะที่ต้องการ (lazy imports)
  const [
    { registerFileHandlers },
    { registerDialogHandlers },
    { registerSettingsHandlers }
  ] = await Promise.all([
    import('./handlers/fileHandlers'),
    import('./handlers/dialogHandlers'),
    import('./handlers/settingsHandlers')
  ])

  registerFileHandlers()
  registerDialogHandlers()
  registerSettingsHandlers(win)
}
```

---

## 5. V8 Snapshot (Startup Speed)

```typescript
// scripts/generateSnapshot.ts
// V8 startup snapshot สำหรับ startup ที่เร็วขึ้น

import { mksnapshot } from '@electron/mksnapshot'
import * as path from 'path'

async function generateSnapshot(): Promise<void> {
  const snapshotScript = `
    // Pre-initialize modules ที่ใช้บ่อย
    global.cachedModules = {
      path: require('path'),
      os: require('os')
    }
    
    // Pre-parse heavy data
    global.preloadedData = JSON.parse('${JSON.stringify({ version: '1.0.0' })}')
  `

  await mksnapshot(snapshotScript, {
    outputDir: path.join(__dirname, '../resources'),
    snapshotFilename: 'v8_context_snapshot.bin'
  })

  console.log('V8 snapshot generated!')
}

generateSnapshot().catch(console.error)
```

---

## 6. Dependency Audit

```typescript
// scripts/auditDependencies.ts
// ตรวจสอบ dependencies ที่ไม่จำเป็น

import * as fs from 'fs'
import * as path from 'path'

function analyzeDependencies(): void {
  const pkg = JSON.parse(fs.readFileSync('package.json', 'utf-8'))
  const allDeps = {
    ...pkg.dependencies,
    ...pkg.devDependencies
  }

  console.log('\n📦 Dependency Analysis\n')
  console.log(`Total dependencies: ${Object.keys(allDeps).length}`)

  // ตรวจหา large packages
  const largePackages = [
    'lodash', 'moment', 'rxjs', 'core-js',
    'babel-polyfill', 'jquery'
  ]

  for (const pkg of largePackages) {
    if (allDeps[pkg]) {
      console.warn(`⚠️  ${pkg} found - consider using lighter alternative`)
    }
  }

  // ตรวจ duplicates
  const nodeModulesDir = path.join(process.cwd(), 'node_modules')
  if (fs.existsSync(nodeModulesDir)) {
    // check for duplicate heavy packages
    const duplicates = findDuplicates(nodeModulesDir)
    if (duplicates.length > 0) {
      console.warn('\nDuplicate packages found:')
      duplicates.forEach(d => console.warn(`  ${d}`))
    }
  }
}

function findDuplicates(dir: string): string[] {
  // simplified - check ที่ top level เท่านั้น
  return []
}

analyzeDependencies()
```

---

## 7. Build Size Report

```json
// package.json scripts
{
  "scripts": {
    "analyze": "ANALYZE=true npm run build && open dist/stats.html",
    "size": "ls -lh dist/*.{exe,dmg,AppImage,deb} 2>/dev/null || echo 'Run build first'",
    "size-check": "node scripts/checkBuildSize.js"
  }
}
```

```javascript
// scripts/checkBuildSize.js
const fs = require('fs')
const path = require('path')

const LIMITS = {
  '.exe': 120 * 1024 * 1024,  // 120 MB
  '.dmg': 130 * 1024 * 1024,  // 130 MB
  '.AppImage': 110 * 1024 * 1024,  // 110 MB
  '.deb': 90 * 1024 * 1024   // 90 MB
}

const distDir = path.join(__dirname, '../dist')
if (!fs.existsSync(distDir)) { console.log('No dist directory'); process.exit(0) }

const files = fs.readdirSync(distDir)
let passed = true

for (const file of files) {
  const ext = path.extname(file)
  const limit = LIMITS[ext]
  if (!limit) continue

  const size = fs.statSync(path.join(distDir, file)).size
  const mb = (size / 1024 / 1024).toFixed(1)
  const limitMb = (limit / 1024 / 1024).toFixed(0)
  const ok = size <= limit

  console.log(`${ok ? '✅' : '❌'} ${file}: ${mb} MB (limit: ${limitMb} MB)`)
  if (!ok) passed = false
}

process.exit(passed ? 0 : 1)
```

---

## สรุป

| เทคนิค | ผลลัพธ์ | หมายเหตุ |
|-------|---------|---------|
| Manual chunks | ลด initial bundle | load on demand |
| Tree shaking | ลบ dead code | ต้องใช้ ESM |
| ASAR exclude | native modules ทำงานได้ | ต้องใส่ asarUnpack |
| Lazy routes | เร็วขึ้น initial load | React.lazy |
| Parallel init | startup เร็วขึ้น | Promise.all |
| Terser minify | ลดขนาด JS | ลบ console ใน prod |
| Bundle analysis | เห็น bottlenecks | visualizer |
