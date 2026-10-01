# Part 75: Plugin System Architecture

## สร้าง Plugin System ใน Electron App

ในบทนี้เราจะเรียนการออกแบบ Plugin API ที่ extensible, sandbox isolation, lifecycle management และ event bus

---

## 1. Plugin API Design

```typescript
// src/shared/pluginTypes.ts

export interface PluginManifest {
  id: string
  name: string
  version: string
  description: string
  author: string
  main: string           // entry point ใน main process
  renderer?: string      // entry point ใน renderer
  permissions: PluginPermission[]
  contributes?: {
    commands?: CommandContribution[]
    menus?: MenuContribution[]
    settings?: SettingContribution[]
    themes?: ThemeContribution[]
  }
  engines: {
    app: string          // version range เช่น ">=2.0.0"
  }
}

export type PluginPermission =
  | 'filesystem:read'
  | 'filesystem:write'
  | 'network:http'
  | 'clipboard'
  | 'notifications'
  | 'shell:open'

export interface CommandContribution {
  id: string
  title: string
  shortcut?: string
  when?: string          // context expression เช่น "editorOpen"
}

export interface MenuContribution {
  location: 'editor/context' | 'file/menu' | 'toolbar'
  command: string
  group?: string
  order?: number
}

export interface SettingContribution {
  key: string
  type: 'string' | 'number' | 'boolean' | 'enum'
  default: unknown
  description: string
  enum?: string[]
}

export interface ThemeContribution {
  id: string
  label: string
  path: string
}

// Plugin API ที่ expose ให้ plugin ใช้
export interface PluginAPI {
  // File system (ถ้ามี permission)
  fs: {
    readFile(path: string): Promise<string>
    writeFile(path: string, content: string): Promise<void>
    exists(path: string): Promise<boolean>
  }
  // Events
  events: {
    on(event: string, callback: (...args: unknown[]) => void): () => void
    emit(event: string, ...args: unknown[]): void
  }
  // UI
  ui: {
    showNotification(message: string, type?: 'info' | 'warning' | 'error'): void
    showInputBox(options: { prompt: string; placeholder?: string }): Promise<string | null>
    showQuickPick(items: string[]): Promise<string | null>
  }
  // Commands
  commands: {
    register(id: string, handler: (...args: unknown[]) => void): () => void
    execute(id: string, ...args: unknown[]): Promise<unknown>
  }
  // Storage per plugin
  storage: {
    get<T>(key: string): T | undefined
    set(key: string, value: unknown): void
    delete(key: string): void
  }
}
```

---

## 2. Plugin Manager (Main Process)

```typescript
// src/main/pluginManager.ts
import { BrowserWindow } from 'electron'
import * as path from 'path'
import * as fs from 'fs/promises'
import { PluginManifest, PluginPermission } from '../shared/pluginTypes'
import { EventEmitter } from 'events'

interface LoadedPlugin {
  manifest: PluginManifest
  dir: string
  instance?: PluginInstance
  enabled: boolean
}

interface PluginInstance {
  activate: (api: MainPluginAPI) => void | Promise<void>
  deactivate?: () => void | Promise<void>
}

export class PluginManager extends EventEmitter {
  private plugins = new Map<string, LoadedPlugin>()
  private pluginsDir: string
  private win: BrowserWindow | null = null

  constructor(pluginsDir: string) {
    super()
    this.pluginsDir = pluginsDir
  }

  setWindow(win: BrowserWindow) {
    this.win = win
  }

  async loadAll(): Promise<void> {
    try {
      await fs.mkdir(this.pluginsDir, { recursive: true })
      const entries = await fs.readdir(this.pluginsDir, { withFileTypes: true })

      for (const entry of entries) {
        if (entry.isDirectory()) {
          await this.loadPlugin(path.join(this.pluginsDir, entry.name))
        }
      }
    } catch (error) {
      console.error('Failed to load plugins:', error)
    }
  }

  async loadPlugin(pluginDir: string): Promise<void> {
    try {
      const manifestPath = path.join(pluginDir, 'package.json')
      const manifestData = await fs.readFile(manifestPath, 'utf-8')
      const manifest: PluginManifest = JSON.parse(manifestData)

      if (!this.validateManifest(manifest)) {
        console.warn(`Invalid manifest for plugin at ${pluginDir}`)
        return
      }

      this.plugins.set(manifest.id, {
        manifest,
        dir: pluginDir,
        enabled: true
      })

      await this.activatePlugin(manifest.id)
      console.log(`Plugin loaded: ${manifest.name} v${manifest.version}`)
    } catch (error) {
      console.error(`Failed to load plugin at ${pluginDir}:`, error)
    }
  }

  private validateManifest(manifest: Partial<PluginManifest>): manifest is PluginManifest {
    return !!(manifest.id && manifest.name && manifest.version && manifest.main)
  }

  private async activatePlugin(pluginId: string): Promise<void> {
    const loaded = this.plugins.get(pluginId)
    if (!loaded || !loaded.enabled) return

    try {
      const mainFile = path.join(loaded.dir, loaded.manifest.main)
      // ใช้ vm sandbox แทน require ตรง
      const api = this.createPluginAPI(pluginId, loaded.manifest.permissions)
      // dynamic import
      const pluginModule = await import(mainFile)
      loaded.instance = pluginModule.default ?? pluginModule

      if (typeof loaded.instance?.activate === 'function') {
        await loaded.instance.activate(api)
      }

      this.emit('plugin:activated', pluginId)
    } catch (error) {
      console.error(`Failed to activate plugin ${pluginId}:`, error)
    }
  }

  async deactivatePlugin(pluginId: string): Promise<void> {
    const loaded = this.plugins.get(pluginId)
    if (!loaded?.instance) return

    try {
      if (typeof loaded.instance.deactivate === 'function') {
        await loaded.instance.deactivate()
      }
      loaded.enabled = false
      this.emit('plugin:deactivated', pluginId)
    } catch (error) {
      console.error(`Failed to deactivate plugin ${pluginId}:`, error)
    }
  }

  private createPluginAPI(pluginId: string, permissions: PluginPermission[]): MainPluginAPI {
    const hasPermission = (p: PluginPermission) => permissions.includes(p)
    const storage = new Map<string, unknown>()

    return {
      fs: {
        readFile: async (filePath: string) => {
          if (!hasPermission('filesystem:read')) throw new Error('No filesystem:read permission')
          return fs.readFile(filePath, 'utf-8')
        },
        writeFile: async (filePath: string, content: string) => {
          if (!hasPermission('filesystem:write')) throw new Error('No filesystem:write permission')
          return fs.writeFile(filePath, content, 'utf-8')
        },
        exists: async (filePath: string) => {
          if (!hasPermission('filesystem:read')) throw new Error('No filesystem:read permission')
          try { await fs.access(filePath); return true } catch { return false }
        }
      },
      events: {
        on: (event: string, cb: (...args: unknown[]) => void) => {
          this.on(`plugin:event:${event}`, cb)
          return () => this.off(`plugin:event:${event}`, cb)
        },
        emit: (event: string, ...args: unknown[]) => {
          this.emit(`plugin:event:${event}`, ...args)
          // Broadcast to renderer
          this.win?.webContents.send('plugin:event', { pluginId, event, args })
        }
      },
      ui: {
        showNotification: (message: string, type = 'info') => {
          this.win?.webContents.send('plugin:notification', { pluginId, message, type })
        },
        showInputBox: async (opts) => {
          return new Promise(resolve => {
            this.win?.webContents.send('plugin:show-input', { pluginId, ...opts })
            this.once(`plugin:input-result:${pluginId}`, resolve)
          })
        },
        showQuickPick: async (items) => {
          return new Promise(resolve => {
            this.win?.webContents.send('plugin:show-quickpick', { pluginId, items })
            this.once(`plugin:quickpick-result:${pluginId}`, resolve)
          })
        }
      },
      commands: {
        register: (id: string, handler: (...args: unknown[]) => void) => {
          const fullId = `${pluginId}.${id}`
          this.on(`command:${fullId}`, handler)
          return () => this.off(`command:${fullId}`, handler)
        },
        execute: async (id: string, ...args: unknown[]) => {
          this.emit(`command:${id}`, ...args)
        }
      },
      storage: {
        get: <T>(key: string) => storage.get(`${pluginId}:${key}`) as T | undefined,
        set: (key: string, value: unknown) => storage.set(`${pluginId}:${key}`, value),
        delete: (key: string) => storage.delete(`${pluginId}:${key}`)
      }
    }
  }

  getPlugins(): LoadedPlugin[] {
    return Array.from(this.plugins.values())
  }

  getPlugin(id: string): LoadedPlugin | undefined {
    return this.plugins.get(id)
  }
}

type MainPluginAPI = ReturnType<PluginManager['createPluginAPI']>
```

---

## 3. Sandboxed Plugin Execution

```typescript
// src/main/pluginSandbox.ts
import vm from 'vm'
import * as path from 'path'
import * as fs from 'fs'

export function runPluginInSandbox(
  pluginCode: string,
  pluginDir: string,
  api: Record<string, unknown>
): unknown {
  // สร้าง limited require
  const pluginRequire = (moduleName: string): unknown => {
    // อนุญาตเฉพาะ modules ที่ safe
    const allowedModules = ['path', 'url', 'querystring', 'events']
    if (allowedModules.includes(moduleName)) {
      return require(moduleName)
    }

    // Relative requires จาก plugin directory
    if (moduleName.startsWith('.')) {
      const resolvedPath = path.resolve(pluginDir, moduleName)
      // ตรวจสอบว่า path อยู่ใน plugin directory
      if (!resolvedPath.startsWith(pluginDir)) {
        throw new Error(`Plugin cannot require files outside its directory: ${moduleName}`)
      }
      const code = fs.readFileSync(resolvedPath + '.js', 'utf-8')
      return runPluginInSandbox(code, pluginDir, api)
    }

    throw new Error(`Module "${moduleName}" is not allowed in plugin context`)
  }

  // Sandbox context
  const sandbox = {
    require: pluginRequire,
    module: { exports: {} as Record<string, unknown> },
    exports: {} as Record<string, unknown>,
    console: {
      log: (...args: unknown[]) => console.log('[Plugin]', ...args),
      warn: (...args: unknown[]) => console.warn('[Plugin]', ...args),
      error: (...args: unknown[]) => console.error('[Plugin]', ...args)
    },
    setTimeout,
    clearTimeout,
    setInterval,
    clearInterval,
    Promise,
    // Plugin API
    pluginAPI: api,
    // Utilities
    JSON,
    Math,
    Date,
    Array,
    Object,
    String,
    Number,
    Boolean,
    RegExp,
    Error,
    Map,
    Set
  }

  vm.createContext(sandbox)
  vm.runInContext(pluginCode, sandbox, {
    timeout: 5000,         // 5s timeout
    filename: 'plugin.js'
  })

  return sandbox.module.exports
}
```

---

## 4. Plugin Marketplace UI

```tsx
// src/renderer/components/PluginMarketplace.tsx
import { useState, useEffect } from 'react'

interface PluginInfo {
  id: string
  name: string
  description: string
  version: string
  author: string
  downloads: number
  enabled: boolean
  installed: boolean
}

export function PluginMarketplace() {
  const [plugins, setPlugins] = useState<PluginInfo[]>([])
  const [search, setSearch] = useState('')
  const [tab, setTab] = useState<'installed' | 'marketplace'>('installed')
  const [loading, setLoading] = useState(false)

  useEffect(() => {
    loadInstalledPlugins()
  }, [])

  const loadInstalledPlugins = async () => {
    const result = await window.electronAPI.invoke('plugins:list')
    setPlugins(result as PluginInfo[])
  }

  const togglePlugin = async (id: string, enabled: boolean) => {
    await window.electronAPI.invoke('plugins:toggle', { id, enabled: !enabled })
    setPlugins(prev => prev.map(p => p.id === id ? { ...p, enabled: !enabled } : p))
  }

  const uninstallPlugin = async (id: string) => {
    if (!confirm('ถอนติดตั้ง plugin นี้?')) return
    await window.electronAPI.invoke('plugins:uninstall', { id })
    setPlugins(prev => prev.filter(p => p.id !== id))
  }

  const filtered = plugins.filter(p =>
    p.name.toLowerCase().includes(search.toLowerCase()) ||
    p.description.toLowerCase().includes(search.toLowerCase())
  )

  return (
    <div className="plugin-marketplace">
      <div className="marketplace-header">
        <h2>Plugin Manager</h2>
        <input
          type="search"
          placeholder="ค้นหา plugin..."
          value={search}
          onChange={e => setSearch(e.target.value)}
          className="search-box"
        />
      </div>

      <div className="tabs">
        <button
          className={tab === 'installed' ? 'active' : ''}
          onClick={() => setTab('installed')}
        >
          ติดตั้งแล้ว ({plugins.filter(p => p.installed).length})
        </button>
        <button
          className={tab === 'marketplace' ? 'active' : ''}
          onClick={() => setTab('marketplace')}
        >
          Marketplace
        </button>
      </div>

      <div className="plugin-list">
        {loading ? (
          <div className="loading">กำลังโหลด...</div>
        ) : filtered.length === 0 ? (
          <div className="empty">ไม่พบ plugin</div>
        ) : (
          filtered.map(plugin => (
            <div key={plugin.id} className="plugin-card">
              <div className="plugin-info">
                <h3>{plugin.name}</h3>
                <p>{plugin.description}</p>
                <div className="plugin-meta">
                  <span>v{plugin.version}</span>
                  <span>โดย {plugin.author}</span>
                  {plugin.downloads && (
                    <span>{plugin.downloads.toLocaleString()} downloads</span>
                  )}
                </div>
              </div>
              <div className="plugin-actions">
                {plugin.installed ? (
                  <>
                    <label className="toggle">
                      <input
                        type="checkbox"
                        checked={plugin.enabled}
                        onChange={() => togglePlugin(plugin.id, plugin.enabled)}
                      />
                      <span>{plugin.enabled ? 'เปิดใช้' : 'ปิด'}</span>
                    </label>
                    <button
                      className="btn-danger"
                      onClick={() => uninstallPlugin(plugin.id)}
                    >
                      ถอนติดตั้ง
                    </button>
                  </>
                ) : (
                  <button className="btn-primary">ติดตั้ง</button>
                )}
              </div>
            </div>
          ))
        )}
      </div>
    </div>
  )
}
```

---

## 5. ตัวอย่าง Plugin

```typescript
// plugins/word-counter/index.js
// Plugin ตัวอย่างที่นับคำ

module.exports = {
  activate(api) {
    console.log('Word Counter plugin activated!')

    // ลงทะเบียน command
    const dispose = api.commands.register('count-words', () => {
      api.events.emit('request-content')
    })

    // รับ content เพื่อนับ
    api.events.on('editor-content', (content) => {
      const words = String(content).split(/\s+/).filter(Boolean).length
      const chars = String(content).length
      api.ui.showNotification(`คำ: ${words}, ตัวอักษร: ${chars}`, 'info')
    })

    // เก็บ cleanup function
    api.storage.set('dispose', dispose)
  },

  deactivate(api) {
    const dispose = api.storage.get('dispose')
    if (typeof dispose === 'function') dispose()
    console.log('Word Counter plugin deactivated')
  }
}
```

```json
{
  "id": "word-counter",
  "name": "Word Counter",
  "version": "1.0.0",
  "description": "นับคำและตัวอักษรในเอกสาร",
  "author": "Example Author",
  "main": "index.js",
  "permissions": [],
  "contributes": {
    "commands": [
      {
        "id": "count-words",
        "title": "นับคำในเอกสาร",
        "shortcut": "Ctrl+Shift+W"
      }
    ]
  },
  "engines": {
    "app": ">=1.0.0"
  }
}
```

---

## สรุป

| ส่วนประกอบ | หน้าที่ |
|-----------|---------|
| PluginManifest | metadata และ permissions declaration |
| PluginManager | load/activate/deactivate lifecycle |
| Plugin Sandbox | vm.createContext isolation |
| Plugin API | abstracted access to app features |
| Permission system | runtime permission checks |
| Plugin Marketplace UI | browse/install/toggle plugins |
