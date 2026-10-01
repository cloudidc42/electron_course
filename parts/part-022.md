# Part 22: Vue.js + Electron Integration

## บทนำ

Vue.js 3 ร่วมกับ Electron เป็น combination ที่ทรงพลังสำหรับการสร้าง Desktop Application ด้วย Composition API และ Reactivity System ของ Vue 3 บวกกับ Electron's native capabilities ทำให้เราสร้าง app ที่ responsive และ feature-rich ได้อย่างรวดเร็ว ในบทนี้จะครอบคลุมการตั้งค่าตั้งแต่ต้น, Vue Router, Pinia state management, และ hot reload

## โครงสร้างโปรเจกต์

```
my-vue-electron-app/
├── electron/
│   ├── main.js
│   ├── preload.js
│   └── ipc/
│       ├── fileHandlers.js
│       └── systemHandlers.js
├── src/
│   ├── main.js
│   ├── App.vue
│   ├── router/
│   │   └── index.js
│   ├── stores/
│   │   ├── app.js
│   │   ├── files.js
│   │   └── settings.js
│   ├── composables/
│   │   ├── useElectron.js
│   │   ├── useIPC.js
│   │   └── useDialog.js
│   ├── components/
│   │   ├── TitleBar.vue
│   │   ├── SideNav.vue
│   │   └── common/
│   └── views/
│       ├── HomeView.vue
│       ├── FilesView.vue
│       └── SettingsView.vue
├── package.json
├── vite.config.js
└── electron-builder.json
```

## การติดตั้งและตั้งค่า

### ขั้นตอนที่ 1: สร้างโปรเจกต์

```bash
# สร้างโปรเจกต์ Vue 3 ด้วย Vite
npm create vite@latest my-vue-electron-app -- --template vue
cd my-vue-electron-app
npm install

# ติดตั้ง Electron dependencies
npm install --save-dev electron electron-builder
npm install --save-dev vite-plugin-electron vite-plugin-electron-renderer
npm install --save-dev concurrently cross-env wait-on

# ติดตั้ง Vue ecosystem
npm install vue-router@4 pinia
npm install @vueuse/core

# ติดตั้ง Electron utilities
npm install electron-store
```

### ขั้นตอนที่ 2: Vite Configuration

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import electron from 'vite-plugin-electron'
import renderer from 'vite-plugin-electron-renderer'
import { resolve } from 'path'

export default defineConfig(({ command }) => ({
  plugins: [
    vue(),
    electron([
      {
        entry: 'electron/main.js',
        vite: {
          build: {
            sourcemap: command === 'serve',
            minify: command !== 'serve',
            outDir: 'dist-electron',
            rollupOptions: {
              external: ['electron', 'electron-store', 'fs', 'path', 'os']
            }
          }
        }
      },
      {
        entry: 'electron/preload.js',
        onstart(options) {
          options.reload()
        },
        vite: {
          build: {
            sourcemap: command === 'serve' ? 'inline' : undefined,
            outDir: 'dist-electron'
          }
        }
      }
    ]),
    renderer()
  ],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '@electron': resolve(__dirname, 'electron')
    }
  },
  server: {
    port: 5174,
    strictPort: true
  },
  build: {
    target: 'esnext',
    rollupOptions: {
      input: {
        main: resolve(__dirname, 'index.html')
      }
    }
  }
}))
```

## Main Process

```javascript
// electron/main.js
const { app, BrowserWindow, ipcMain, dialog, shell, Menu, Tray, nativeImage } = require('electron')
const path = require('path')
const Store = require('electron-store')

const store = new Store()
const isDev = process.env.NODE_ENV === 'development'

let mainWindow = null
let tray = null

function createWindow() {
  // ดึงขนาด window ที่บันทึกไว้
  const windowBounds = store.get('windowBounds', {
    width: 1280,
    height: 800,
    x: undefined,
    y: undefined
  })

  mainWindow = new BrowserWindow({
    ...windowBounds,
    minWidth: 900,
    minHeight: 600,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, 'preload.js'),
      sandbox: false,
      webSecurity: !isDev
    },
    titleBarStyle: process.platform === 'darwin' ? 'hidden' : 'default',
    trafficLightPosition: { x: 12, y: 12 },
    vibrancy: process.platform === 'darwin' ? 'window' : undefined,
    backgroundMaterial: process.platform === 'win32' ? 'mica' : undefined,
    frame: process.platform !== 'win32',
    show: false,
    icon: getAppIcon()
  })

  // โหลด URL
  if (isDev) {
    mainWindow.loadURL('http://localhost:5174')
    mainWindow.webContents.openDevTools()
  } else {
    mainWindow.loadFile(path.join(__dirname, '../dist/index.html'))
  }

  // จัดการ window events
  mainWindow.once('ready-to-show', () => {
    mainWindow.show()
  })

  // บันทึกขนาด window เมื่อปิด
  mainWindow.on('close', () => {
    store.set('windowBounds', mainWindow.getBounds())
  })

  mainWindow.on('closed', () => {
    mainWindow = null
  })

  // สร้าง application menu
  createAppMenu()
}

function getAppIcon() {
  if (process.platform === 'darwin') {
    return path.join(__dirname, '../build/icon.icns')
  } else if (process.platform === 'win32') {
    return path.join(__dirname, '../build/icon.ico')
  }
  return path.join(__dirname, '../build/icon.png')
}

function createAppMenu() {
  const template = [
    {
      label: 'ไฟล์',
      submenu: [
        {
          label: 'เปิดไฟล์',
          accelerator: 'CmdOrCtrl+O',
          click: () => mainWindow?.webContents.send('menu:openFile')
        },
        {
          label: 'บันทึก',
          accelerator: 'CmdOrCtrl+S',
          click: () => mainWindow?.webContents.send('menu:save')
        },
        { type: 'separator' },
        { role: 'quit', label: 'ออก' }
      ]
    },
    {
      label: 'แก้ไข',
      submenu: [
        { role: 'undo', label: 'เลิกทำ' },
        { role: 'redo', label: 'ทำซ้ำ' },
        { type: 'separator' },
        { role: 'cut', label: 'ตัด' },
        { role: 'copy', label: 'คัดลอก' },
        { role: 'paste', label: 'วาง' }
      ]
    },
    {
      label: 'มุมมอง',
      submenu: [
        { role: 'reload', label: 'โหลดใหม่' },
        { role: 'toggleDevTools', label: 'Developer Tools' },
        { type: 'separator' },
        { role: 'togglefullscreen', label: 'เต็มหน้าจอ' }
      ]
    }
  ]

  if (process.platform === 'darwin') {
    template.unshift({
      label: app.name,
      submenu: [
        { role: 'about', label: 'เกี่ยวกับ' },
        { type: 'separator' },
        { role: 'hide', label: 'ซ่อน' },
        { role: 'hideOthers' },
        { role: 'unhide' },
        { type: 'separator' },
        { role: 'quit', label: 'ออก' }
      ]
    })
  }

  Menu.setApplicationMenu(Menu.buildFromTemplate(template))
}

// IPC Handlers
function setupIpcHandlers() {
  // File operations
  ipcMain.handle('dialog:openFile', async (event, options) => {
    return await dialog.showOpenDialog(mainWindow, {
      properties: ['openFile'],
      ...options
    })
  })

  ipcMain.handle('dialog:openDirectory', async (event, options) => {
    return await dialog.showOpenDialog(mainWindow, {
      properties: ['openDirectory'],
      ...options
    })
  })

  ipcMain.handle('dialog:saveFile', async (event, options) => {
    return await dialog.showSaveDialog(mainWindow, options)
  })

  ipcMain.handle('dialog:message', async (event, options) => {
    return await dialog.showMessageBox(mainWindow, options)
  })

  // Store operations
  ipcMain.handle('store:get', (event, key, defaultValue) => {
    return store.get(key, defaultValue)
  })

  ipcMain.handle('store:set', (event, key, value) => {
    store.set(key, value)
    return true
  })

  ipcMain.handle('store:delete', (event, key) => {
    store.delete(key)
    return true
  })

  ipcMain.handle('store:clear', () => {
    store.clear()
    return true
  })

  ipcMain.handle('store:getAll', () => {
    return store.store
  })

  // App operations
  ipcMain.handle('app:getVersion', () => app.getVersion())
  ipcMain.handle('app:getPath', (event, name) => app.getPath(name))
  ipcMain.handle('app:getName', () => app.getName())
  
  ipcMain.handle('shell:openExternal', (event, url) => {
    const allowed = url.startsWith('https://') || url.startsWith('http://')
    if (allowed) {
      return shell.openExternal(url)
    }
    return Promise.reject(new Error('URL ไม่ได้รับอนุญาต'))
  })

  // Window operations
  ipcMain.handle('window:minimize', () => mainWindow?.minimize())
  ipcMain.handle('window:maximize', () => {
    if (mainWindow?.isMaximized()) {
      mainWindow.unmaximize()
    } else {
      mainWindow?.maximize()
    }
  })
  ipcMain.handle('window:close', () => mainWindow?.close())
  ipcMain.handle('window:isMaximized', () => mainWindow?.isMaximized() ?? false)
  ipcMain.handle('window:setTitle', (event, title) => mainWindow?.setTitle(title))
  
  // Theme
  ipcMain.handle('theme:getSystem', () => {
    const { nativeTheme } = require('electron')
    return nativeTheme.shouldUseDarkColors ? 'dark' : 'light'
  })
}

app.whenReady().then(() => {
  createWindow()
  setupIpcHandlers()

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

## Preload Script

```javascript
// electron/preload.js
const { contextBridge, ipcRenderer } = require('electron')

const electronAPI = {
  // Dialog
  dialog: {
    openFile: (options) => ipcRenderer.invoke('dialog:openFile', options),
    openDirectory: (options) => ipcRenderer.invoke('dialog:openDirectory', options),
    saveFile: (options) => ipcRenderer.invoke('dialog:saveFile', options),
    message: (options) => ipcRenderer.invoke('dialog:message', options)
  },
  
  // Store (persistent settings)
  store: {
    get: (key, defaultValue) => ipcRenderer.invoke('store:get', key, defaultValue),
    set: (key, value) => ipcRenderer.invoke('store:set', key, value),
    delete: (key) => ipcRenderer.invoke('store:delete', key),
    clear: () => ipcRenderer.invoke('store:clear'),
    getAll: () => ipcRenderer.invoke('store:getAll')
  },
  
  // App info
  app: {
    getVersion: () => ipcRenderer.invoke('app:getVersion'),
    getPath: (name) => ipcRenderer.invoke('app:getPath', name),
    getName: () => ipcRenderer.invoke('app:getName')
  },
  
  // Shell
  shell: {
    openExternal: (url) => ipcRenderer.invoke('shell:openExternal', url)
  },
  
  // Window
  window: {
    minimize: () => ipcRenderer.invoke('window:minimize'),
    maximize: () => ipcRenderer.invoke('window:maximize'),
    close: () => ipcRenderer.invoke('window:close'),
    isMaximized: () => ipcRenderer.invoke('window:isMaximized'),
    setTitle: (title) => ipcRenderer.invoke('window:setTitle', title),
    
    onMaximized: (callback) => {
      ipcRenderer.on('window:maximized', (e, state) => callback(state))
      return () => ipcRenderer.removeAllListeners('window:maximized')
    }
  },
  
  // Theme
  theme: {
    getSystem: () => ipcRenderer.invoke('theme:getSystem'),
    onChange: (callback) => {
      ipcRenderer.on('theme:changed', (e, theme) => callback(theme))
      return () => ipcRenderer.removeAllListeners('theme:changed')
    }
  },
  
  // Menu events
  menu: {
    onOpenFile: (callback) => {
      ipcRenderer.on('menu:openFile', callback)
      return () => ipcRenderer.removeListener('menu:openFile', callback)
    },
    onSave: (callback) => {
      ipcRenderer.on('menu:save', callback)
      return () => ipcRenderer.removeListener('menu:save', callback)
    }
  }
}

contextBridge.exposeInMainWorld('electronAPI', electronAPI)
```

## Vue Application

### Main Entry Point

```javascript
// src/main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './router'
import './assets/main.css'

const app = createApp(App)
const pinia = createPinia()

app.use(pinia)
app.use(router)

// Global error handler
app.config.errorHandler = (err, vm, info) => {
  console.error('Vue Error:', err, info)
  // ส่ง error ไปยัง main process ถ้าต้องการ log
}

// Global properties
app.config.globalProperties.$isElectron = !!window.electronAPI

app.mount('#app')
```

### App.vue Component

```vue
<!-- src/App.vue -->
<template>
  <div :class="['app', theme]" :data-platform="platform">
    <!-- Title Bar สำหรับ Windows/Linux -->
    <TitleBar v-if="showTitleBar" />
    
    <div class="app-layout">
      <!-- Side Navigation -->
      <SideNav />
      
      <!-- Main Content -->
      <main class="app-content">
        <RouterView v-slot="{ Component, route }">
          <Transition :name="route.meta.transition || 'fade'" mode="out-in">
            <component :is="Component" :key="route.path" />
          </Transition>
        </RouterView>
      </main>
    </div>
    
    <!-- Global Notifications -->
    <NotificationContainer />
  </div>
</template>

<script setup>
import { computed, onMounted, onUnmounted } from 'vue'
import { RouterView } from 'vue-router'
import { useAppStore } from '@/stores/app'
import TitleBar from '@/components/TitleBar.vue'
import SideNav from '@/components/SideNav.vue'
import NotificationContainer from '@/components/NotificationContainer.vue'

const appStore = useAppStore()

const theme = computed(() => appStore.theme)
const platform = computed(() => appStore.platform)
const showTitleBar = computed(() => {
  return appStore.isElectron && appStore.platform !== 'MacIntel'
})

onMounted(async () => {
  await appStore.initialize()
  
  // Listen สำหรับ theme changes
  if (window.electronAPI?.theme) {
    appStore.cleanupFunctions.push(
      window.electronAPI.theme.onChange((newTheme) => {
        appStore.setTheme(newTheme)
      })
    )
  }
})

onUnmounted(() => {
  appStore.cleanup()
})
</script>

<style>
/* Transition animations */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.slide-enter-active,
.slide-leave-active {
  transition: transform 0.3s ease;
}

.slide-enter-from {
  transform: translateX(20px);
}

.slide-leave-to {
  transform: translateX(-20px);
}
</style>
```

## Vue Router

```javascript
// src/router/index.js
import { createRouter, createWebHashHistory } from 'vue-router'

const routes = [
  {
    path: '/',
    redirect: '/home'
  },
  {
    path: '/home',
    name: 'Home',
    component: () => import('@/views/HomeView.vue'),
    meta: {
      title: 'หน้าหลัก',
      icon: '🏠',
      transition: 'fade'
    }
  },
  {
    path: '/files',
    name: 'Files',
    component: () => import('@/views/FilesView.vue'),
    meta: {
      title: 'ไฟล์',
      icon: '📁',
      transition: 'slide'
    }
  },
  {
    path: '/editor/:id?',
    name: 'Editor',
    component: () => import('@/views/EditorView.vue'),
    meta: {
      title: 'แก้ไข',
      icon: '✏️',
      transition: 'slide'
    }
  },
  {
    path: '/settings',
    name: 'Settings',
    component: () => import('@/views/SettingsView.vue'),
    meta: {
      title: 'ตั้งค่า',
      icon: '⚙️',
      transition: 'fade'
    }
  },
  {
    path: '/:pathMatch(.*)*',
    name: 'NotFound',
    component: () => import('@/views/NotFoundView.vue')
  }
]

const router = createRouter({
  // ใช้ Hash history สำหรับ Electron (ไม่ต้องการ server)
  history: createWebHashHistory(),
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      return savedPosition
    }
    return { top: 0 }
  }
})

// Navigation guards
router.beforeEach((to, from, next) => {
  // อัปเดต window title
  if (to.meta.title && window.electronAPI?.window) {
    window.electronAPI.window.setTitle(`My App - ${to.meta.title}`)
  }
  next()
})

export default router
```

## Pinia Stores

### App Store

```javascript
// src/stores/app.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useAppStore = defineStore('app', () => {
  // State
  const theme = ref('light')
  const platform = ref('')
  const appVersion = ref('')
  const isElectron = ref(false)
  const isInitialized = ref(false)
  const cleanupFunctions = ref([])
  
  // Computed
  const isDarkMode = computed(() => theme.value === 'dark')
  
  // Actions
  async function initialize() {
    if (isInitialized.value) return
    
    // ตรวจสอบ Electron environment
    isElectron.value = !!window.electronAPI
    platform.value = navigator.platform
    
    if (isElectron.value) {
      try {
        // ดึงข้อมูล app
        appVersion.value = await window.electronAPI.app.getVersion()
        
        // ดึง theme จาก system
        const systemTheme = await window.electronAPI.theme.getSystem()
        const savedTheme = await window.electronAPI.store.get('theme', systemTheme)
        theme.value = savedTheme
      } catch (error) {
        console.error('Failed to initialize app store:', error)
      }
    } else {
      // Web environment
      const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)')
      theme.value = mediaQuery.matches ? 'dark' : 'light'
    }
    
    // Apply theme to document
    applyTheme(theme.value)
    
    isInitialized.value = true
  }
  
  async function setTheme(newTheme) {
    theme.value = newTheme
    applyTheme(newTheme)
    
    if (isElectron.value) {
      await window.electronAPI.store.set('theme', newTheme)
    }
  }
  
  function applyTheme(themeName) {
    document.documentElement.setAttribute('data-theme', themeName)
  }
  
  function cleanup() {
    cleanupFunctions.value.forEach(fn => {
      if (typeof fn === 'function') fn()
    })
    cleanupFunctions.value = []
  }
  
  return {
    theme,
    platform,
    appVersion,
    isElectron,
    isInitialized,
    cleanupFunctions,
    isDarkMode,
    initialize,
    setTheme,
    cleanup
  }
})
```

### Files Store

```javascript
// src/stores/files.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useFilesStore = defineStore('files', () => {
  // State
  const recentFiles = ref([])
  const openFiles = ref([])
  const activeFileId = ref(null)
  const isLoading = ref(false)
  const error = ref(null)
  
  // Computed
  const activeFile = computed(() => 
    openFiles.value.find(f => f.id === activeFileId.value)
  )
  
  const hasUnsavedChanges = computed(() => 
    openFiles.value.some(f => f.isDirty)
  )
  
  // Actions
  async function openFile(filePath) {
    // ตรวจสอบว่าไฟล์เปิดอยู่แล้ว
    const existing = openFiles.value.find(f => f.path === filePath)
    if (existing) {
      activeFileId.value = existing.id
      return existing
    }
    
    isLoading.value = true
    error.value = null
    
    try {
      // อ่านไฟล์ (ต้องมี fs API ใน preload)
      const content = await window.electronAPI?.fs?.readFile(filePath) || ''
      
      const file = {
        id: Date.now().toString(),
        path: filePath,
        name: filePath.split(/[\\/]/).pop(),
        content,
        originalContent: content,
        isDirty: false,
        createdAt: new Date()
      }
      
      openFiles.value.push(file)
      activeFileId.value = file.id
      
      // เพิ่มลงรายการล่าสุด
      addToRecent(filePath)
      
      return file
    } catch (err) {
      error.value = err.message
      throw err
    } finally {
      isLoading.value = false
    }
  }
  
  function closeFile(fileId) {
    const index = openFiles.value.findIndex(f => f.id === fileId)
    if (index > -1) {
      openFiles.value.splice(index, 1)
      
      // อัปเดต active file
      if (activeFileId.value === fileId) {
        activeFileId.value = openFiles.value[index - 1]?.id || 
                             openFiles.value[0]?.id || 
                             null
      }
    }
  }
  
  function updateFileContent(fileId, content) {
    const file = openFiles.value.find(f => f.id === fileId)
    if (file) {
      file.content = content
      file.isDirty = content !== file.originalContent
    }
  }
  
  async function saveFile(fileId) {
    const file = openFiles.value.find(f => f.id === fileId)
    if (!file || !file.isDirty) return
    
    isLoading.value = true
    
    try {
      await window.electronAPI?.fs?.writeFile(file.path, file.content)
      file.originalContent = file.content
      file.isDirty = false
    } catch (err) {
      error.value = err.message
      throw err
    } finally {
      isLoading.value = false
    }
  }
  
  function addToRecent(filePath) {
    const maxRecent = 10
    const existing = recentFiles.value.indexOf(filePath)
    
    if (existing > -1) {
      recentFiles.value.splice(existing, 1)
    }
    
    recentFiles.value.unshift(filePath)
    
    if (recentFiles.value.length > maxRecent) {
      recentFiles.value = recentFiles.value.slice(0, maxRecent)
    }
    
    // บันทึกลง store
    window.electronAPI?.store?.set('recentFiles', recentFiles.value)
  }
  
  async function loadRecentFiles() {
    if (window.electronAPI?.store) {
      const saved = await window.electronAPI.store.get('recentFiles', [])
      recentFiles.value = saved
    }
  }
  
  return {
    recentFiles,
    openFiles,
    activeFileId,
    activeFile,
    isLoading,
    error,
    hasUnsavedChanges,
    openFile,
    closeFile,
    updateFileContent,
    saveFile,
    loadRecentFiles
  }
})
```

### Settings Store

```javascript
// src/stores/settings.js
import { defineStore } from 'pinia'
import { ref, watch } from 'vue'

const defaultSettings = {
  theme: 'system',
  language: 'th',
  fontSize: 14,
  fontFamily: 'Consolas, monospace',
  autoSave: true,
  autoSaveInterval: 30000,
  showLineNumbers: true,
  wordWrap: false,
  tabSize: 2,
  notifications: true
}

export const useSettingsStore = defineStore('settings', () => {
  const settings = ref({ ...defaultSettings })
  const isLoaded = ref(false)
  
  async function load() {
    if (!window.electronAPI?.store) {
      isLoaded.value = true
      return
    }
    
    const saved = await window.electronAPI.store.get('settings', defaultSettings)
    settings.value = { ...defaultSettings, ...saved }
    isLoaded.value = true
  }
  
  async function save() {
    if (!window.electronAPI?.store) return
    await window.electronAPI.store.set('settings', settings.value)
  }
  
  async function update(key, value) {
    settings.value[key] = value
    await save()
  }
  
  async function reset() {
    settings.value = { ...defaultSettings }
    await save()
  }
  
  function get(key) {
    return settings.value[key] ?? defaultSettings[key]
  }
  
  // Auto-save เมื่อ settings เปลี่ยน
  watch(settings, () => {
    if (isLoaded.value) {
      save()
    }
  }, { deep: true })
  
  return {
    settings,
    isLoaded,
    load,
    save,
    update,
    reset,
    get
  }
})
```

## Vue Composables

### useElectron Composable

```javascript
// src/composables/useElectron.js
import { ref, onMounted, onUnmounted } from 'vue'

/**
 * Composable สำหรับ Electron integration
 */
export function useElectron() {
  const isElectron = ref(!!window.electronAPI)
  const appVersion = ref('')
  const platform = ref(navigator.platform)
  
  onMounted(async () => {
    if (isElectron.value) {
      appVersion.value = await window.electronAPI.app.getVersion()
    }
  })
  
  return {
    isElectron,
    appVersion,
    platform,
    api: window.electronAPI
  }
}

/**
 * Composable สำหรับ IPC events
 */
export function useIpcEvent(channel, handler) {
  onMounted(() => {
    if (!window.electronAPI?.on) return
    window.electronAPI.on(channel, handler)
  })
  
  onUnmounted(() => {
    // cleanup handled by preload
  })
}

/**
 * Composable สำหรับ dialog operations
 */
export function useDialog() {
  const isLoading = ref(false)
  const lastError = ref(null)
  
  async function withLoading(fn) {
    isLoading.value = true
    lastError.value = null
    try {
      return await fn()
    } catch (err) {
      lastError.value = err.message
      throw err
    } finally {
      isLoading.value = false
    }
  }
  
  const openFile = (options = {}) => withLoading(async () => {
    if (!window.electronAPI) return null
    const result = await window.electronAPI.dialog.openFile(options)
    return result.canceled ? null : result.filePaths[0]
  })
  
  const openDirectory = (options = {}) => withLoading(async () => {
    if (!window.electronAPI) return null
    const result = await window.electronAPI.dialog.openDirectory(options)
    return result.canceled ? null : result.filePaths[0]
  })
  
  const saveFile = (options = {}) => withLoading(async () => {
    if (!window.electronAPI) return null
    const result = await window.electronAPI.dialog.saveFile(options)
    return result.canceled ? null : result.filePath
  })
  
  const confirm = async (message, detail = '') => {
    if (!window.electronAPI) return window.confirm(message)
    const result = await window.electronAPI.dialog.message({
      type: 'question',
      buttons: ['ยืนยัน', 'ยกเลิก'],
      defaultId: 0,
      cancelId: 1,
      message,
      detail
    })
    return result.response === 0
  }
  
  return { openFile, openDirectory, saveFile, confirm, isLoading, lastError }
}

/**
 * Composable สำหรับ window operations
 */
export function useWindowControls() {
  const isMaximized = ref(false)
  
  onMounted(async () => {
    if (window.electronAPI?.window) {
      isMaximized.value = await window.electronAPI.window.isMaximized()
      
      window.electronAPI.window.onMaximized((state) => {
        isMaximized.value = state
      })
    }
  })
  
  return {
    isMaximized,
    minimize: () => window.electronAPI?.window.minimize(),
    maximize: () => window.electronAPI?.window.maximize(),
    close: () => window.electronAPI?.window.close()
  }
}
```

## Vue Components

### TitleBar.vue

```vue
<!-- src/components/TitleBar.vue -->
<template>
  <div class="titlebar" :class="{ maximized: isMaximized }">
    <!-- Drag region -->
    <div class="titlebar-drag" />
    
    <!-- App title -->
    <div class="titlebar-title">{{ title }}</div>
    
    <!-- Window controls -->
    <div class="titlebar-controls">
      <button
        class="titlebar-btn minimize-btn"
        @click="minimize"
        title="ย่อ"
      >
        <svg width="10" height="1" viewBox="0 0 10 1">
          <path d="M0 0h10v1H0z" fill="currentColor" />
        </svg>
      </button>
      
      <button
        class="titlebar-btn maximize-btn"
        @click="maximize"
        :title="isMaximized ? 'คืนขนาด' : 'ขยาย'"
      >
        <svg v-if="!isMaximized" width="10" height="10" viewBox="0 0 10 10">
          <path d="M0 0v10h10V0H0zm1 1h8v8H1V1z" fill="currentColor" />
        </svg>
        <svg v-else width="10" height="10" viewBox="0 0 10 10">
          <path d="M2 0v2H0v8h8V8h2V0H2zm1 1h6v6H8V2H3V1zm-2 2h6v6H1V3z" fill="currentColor" />
        </svg>
      </button>
      
      <button
        class="titlebar-btn close-btn"
        @click="close"
        title="ปิด"
      >
        <svg width="10" height="10" viewBox="0 0 10 10">
          <path d="M1 0L0 1l4 4-4 4 1 1 4-4 4 4 1-1-4-4 4-4-1-1-4 4-4-4z" fill="currentColor" />
        </svg>
      </button>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'
import { useWindowControls } from '@/composables/useElectron'
import { useAppStore } from '@/stores/app'

const route = useRoute()
const appStore = useAppStore()
const { isMaximized, minimize, maximize, close } = useWindowControls()

const title = computed(() => {
  return route.meta?.title ? `My App - ${route.meta.title}` : 'My App'
})
</script>

<style scoped>
.titlebar {
  display: flex;
  align-items: center;
  height: 32px;
  background: var(--titlebar-bg, #1e1e1e);
  color: var(--titlebar-color, #cccccc);
  -webkit-app-region: drag;
  user-select: none;
  position: relative;
  z-index: 1000;
}

.titlebar-drag {
  flex: 1;
}

.titlebar-title {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  font-size: 12px;
  font-weight: 400;
  pointer-events: none;
}

.titlebar-controls {
  display: flex;
  -webkit-app-region: no-drag;
}

.titlebar-btn {
  width: 46px;
  height: 32px;
  border: none;
  background: transparent;
  color: inherit;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.1s;
}

.titlebar-btn:hover {
  background: rgba(255, 255, 255, 0.1);
}

.close-btn:hover {
  background: #e81123 !important;
  color: white;
}
</style>
```

### SideNav.vue

```vue
<!-- src/components/SideNav.vue -->
<template>
  <nav class="sidenav">
    <div class="sidenav-header">
      <img src="/logo.png" alt="Logo" class="app-logo" />
      <span class="app-name">My Vue App</span>
    </div>
    
    <ul class="nav-list">
      <li v-for="item in navItems" :key="item.path">
        <RouterLink
          :to="item.path"
          class="nav-link"
          :class="{ active: isActive(item.path) }"
          :title="item.label"
        >
          <span class="nav-icon">{{ item.icon }}</span>
          <span class="nav-label">{{ item.label }}</span>
          <span v-if="item.badge" class="nav-badge">{{ item.badge }}</span>
        </RouterLink>
      </li>
    </ul>
    
    <div class="sidenav-footer">
      <div class="version-info">v{{ appVersion }}</div>
    </div>
  </nav>
</template>

<script setup>
import { computed } from 'vue'
import { RouterLink, useRoute } from 'vue-router'
import { useAppStore } from '@/stores/app'
import { useFilesStore } from '@/stores/files'

const route = useRoute()
const appStore = useAppStore()
const filesStore = useFilesStore()

const appVersion = computed(() => appStore.appVersion)

const navItems = computed(() => [
  { path: '/home', label: 'หน้าหลัก', icon: '🏠' },
  {
    path: '/files',
    label: 'ไฟล์',
    icon: '📁',
    badge: filesStore.openFiles.length || null
  },
  { path: '/settings', label: 'ตั้งค่า', icon: '⚙️' }
])

function isActive(path) {
  return route.path === path || route.path.startsWith(path + '/')
}
</script>

<style scoped>
.sidenav {
  width: 220px;
  min-height: 0;
  flex-shrink: 0;
  background: var(--sidebar-bg);
  border-right: 1px solid var(--border-color);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.sidenav-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 16px;
  border-bottom: 1px solid var(--border-color);
}

.app-logo {
  width: 28px;
  height: 28px;
  border-radius: 6px;
}

.nav-list {
  list-style: none;
  padding: 8px;
  flex: 1;
  overflow-y: auto;
}

.nav-link {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 8px;
  text-decoration: none;
  color: var(--text-secondary);
  font-size: 14px;
  transition: all 0.15s ease;
}

.nav-link:hover {
  background: var(--hover-bg);
  color: var(--text-primary);
}

.nav-link.active {
  background: var(--accent-color);
  color: white;
}

.nav-badge {
  margin-left: auto;
  background: var(--accent-color);
  color: white;
  font-size: 11px;
  padding: 1px 6px;
  border-radius: 10px;
  font-weight: 600;
}

.nav-link.active .nav-badge {
  background: rgba(255, 255, 255, 0.3);
}

.sidenav-footer {
  padding: 12px 16px;
  border-top: 1px solid var(--border-color);
  font-size: 11px;
  color: var(--text-muted);
}
</style>
```

## CSS Variables และ Theming

```css
/* src/assets/main.css */
:root {
  /* Colors */
  --accent-color: #6366f1;
  --accent-hover: #4f46e5;
  
  /* Background */
  --bg-primary: #ffffff;
  --bg-secondary: #f8fafc;
  --sidebar-bg: #f1f5f9;
  --titlebar-bg: #f8fafc;
  
  /* Text */
  --text-primary: #0f172a;
  --text-secondary: #475569;
  --text-muted: #94a3b8;
  --titlebar-color: #334155;
  
  /* Borders */
  --border-color: #e2e8f0;
  
  /* Hover */
  --hover-bg: rgba(0, 0, 0, 0.05);
}

[data-theme="dark"] {
  --bg-primary: #0f172a;
  --bg-secondary: #1e293b;
  --sidebar-bg: #1a2436;
  --titlebar-bg: #1e293b;
  
  --text-primary: #f1f5f9;
  --text-secondary: #94a3b8;
  --text-muted: #64748b;
  --titlebar-color: #94a3b8;
  
  --border-color: #334155;
  --hover-bg: rgba(255, 255, 255, 0.05);
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html, body, #app {
  height: 100%;
  overflow: hidden;
}

.app {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background: var(--bg-primary);
  color: var(--text-primary);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Noto Sans Thai, sans-serif;
}

.app-layout {
  display: flex;
  flex: 1;
  overflow: hidden;
}

.app-content {
  flex: 1;
  overflow-y: auto;
  background: var(--bg-primary);
}
```

## Packaging Configuration

```json
// electron-builder.json
{
  "appId": "com.example.vue-electron-app",
  "productName": "Vue Electron App",
  "directories": {
    "output": "release/${version}"
  },
  "files": [
    "dist/**",
    "dist-electron/**"
  ],
  "win": {
    "target": ["nsis", "portable"],
    "icon": "build/icon.ico"
  },
  "mac": {
    "target": {
      "target": "default",
      "arch": ["x64", "arm64"]
    },
    "icon": "build/icon.icns",
    "extendInfo": {
      "NSMicrophoneUsageDescription": false,
      "NSCameraUsageDescription": false
    }
  },
  "linux": {
    "target": ["AppImage", "deb", "rpm"],
    "icon": "build/icons/"
  },
  "nsis": {
    "oneClick": true,
    "perMachine": false,
    "deleteAppDataOnUninstall": false
  }
}
```

## Package.json Scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "electron:dev": "concurrently \"vite\" \"wait-on http://localhost:5174 && cross-env NODE_ENV=development electron .\"",
    "electron:build": "vite build && electron-builder",
    "electron:build:win": "vite build && electron-builder --win",
    "electron:build:mac": "vite build && electron-builder --mac",
    "electron:build:linux": "vite build && electron-builder --linux"
  }
}
```

## สรุป

การใช้ Vue 3 กับ Electron มีข้อดีหลายประการ:
- **Composition API** ทำให้ logic ที่เกี่ยวกับ Electron แยกออกเป็น composables ที่ reusable ได้
- **Pinia** ช่วยจัดการ state ที่ซับซ้อนระหว่าง components
- **Vue Router** กับ hash history ทำงานได้ดีใน Electron environment
- **Vite** ให้ hot reload ที่รวดเร็วมากระหว่าง development
