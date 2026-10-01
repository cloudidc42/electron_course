# Part 61: Project - Text Editor App

## สร้าง Full-Featured Text Editor คล้าย Notepad++

ในบทนี้เราจะสร้าง Text Editor แบบมืออาชีพที่มีฟีเจอร์ครบครัน รองรับ syntax highlighting, file tabs, find & replace, line numbers และอื่นๆ

---

## โครงสร้างโปรเจค

```
text-editor/
├── src/
│   ├── main/
│   │   ├── index.ts
│   │   ├── menu.ts
│   │   ├── fileManager.ts
│   │   └── ipcHandlers.ts
│   ├── renderer/
│   │   ├── index.html
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── TabBar.tsx
│   │   │   ├── Editor.tsx
│   │   │   ├── StatusBar.tsx
│   │   │   ├── FindReplace.tsx
│   │   │   └── Sidebar.tsx
│   │   ├── hooks/
│   │   │   ├── useEditor.ts
│   │   │   └── useFileSystem.ts
│   │   └── store/
│   │       └── editorStore.ts
│   └── preload/
│       └── index.ts
├── package.json
└── electron.vite.config.ts
```

---

## 1. Main Process - index.ts

```typescript
// src/main/index.ts
import { app, BrowserWindow, ipcMain, dialog, shell } from 'electron'
import { join } from 'path'
import { setupMenu } from './menu'
import { registerFileHandlers } from './fileManager'
import { registerEditorHandlers } from './ipcHandlers'

let mainWindow: BrowserWindow | null = null

function createWindow(): void {
  mainWindow = new BrowserWindow({
    width: 1400,
    height: 900,
    minWidth: 800,
    minHeight: 600,
    titleBarStyle: 'hidden',
    titleBarOverlay: {
      color: '#1e1e1e',
      symbolColor: '#cccccc',
      height: 32
    },
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      nodeIntegration: false,
      contextIsolation: true,
      sandbox: false
    },
    backgroundColor: '#1e1e1e',
    show: false
  })

  mainWindow.once('ready-to-show', () => {
    mainWindow?.show()
  })

  // Handle close with unsaved changes
  mainWindow.on('close', async (e) => {
    e.preventDefault()
    mainWindow?.webContents.send('app:before-close')
  })

  setupMenu(mainWindow)
  registerFileHandlers()
  registerEditorHandlers()

  if (process.env.ELECTRON_RENDERER_URL) {
    mainWindow.loadURL(process.env.ELECTRON_RENDERER_URL)
  } else {
    mainWindow.loadFile(join(__dirname, '../renderer/index.html'))
  }
}

app.whenReady().then(createWindow)

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})

ipcMain.on('app:confirm-close', () => {
  mainWindow?.destroy()
})
```

---

## 2. File Manager

```typescript
// src/main/fileManager.ts
import { ipcMain, dialog, app } from 'electron'
import { readFile, writeFile, stat } from 'fs/promises'
import { extname, basename, dirname } from 'path'
import Store from 'electron-store'

interface RecentFile {
  path: string
  name: string
  lastOpened: number
}

const store = new Store<{ recentFiles: RecentFile[] }>({
  defaults: { recentFiles: [] }
})

export function registerFileHandlers(): void {
  // เปิดไฟล์
  ipcMain.handle('file:open', async (_, filePath?: string) => {
    let targetPath = filePath

    if (!targetPath) {
      const result = await dialog.showOpenDialog({
        properties: ['openFile', 'multiSelections'],
        filters: [
          { name: 'Text Files', extensions: ['txt', 'md', 'js', 'ts', 'jsx', 'tsx', 'py', 'css', 'html', 'json', 'yaml', 'yml', 'xml'] },
          { name: 'All Files', extensions: ['*'] }
        ]
      })
      if (result.canceled || result.filePaths.length === 0) return null
      targetPath = result.filePaths[0]
    }

    try {
      const content = await readFile(targetPath, 'utf-8')
      const stats = await stat(targetPath)
      const ext = extname(targetPath).slice(1)

      // เพิ่มใน recent files
      addRecentFile(targetPath)

      return {
        path: targetPath,
        name: basename(targetPath),
        content,
        extension: ext,
        size: stats.size,
        modified: stats.mtimeMs
      }
    } catch (error) {
      return { error: `ไม่สามารถเปิดไฟล์ได้: ${(error as Error).message}` }
    }
  })

  // บันทึกไฟล์
  ipcMain.handle('file:save', async (_, { path: filePath, content }: { path: string; content: string }) => {
    try {
      await writeFile(filePath, content, 'utf-8')
      return { success: true, path: filePath }
    } catch (error) {
      return { success: false, error: (error as Error).message }
    }
  })

  // บันทึกเป็น
  ipcMain.handle('file:save-as', async (_, { content, defaultName }: { content: string; defaultName?: string }) => {
    const result = await dialog.showSaveDialog({
      defaultPath: defaultName,
      filters: [
        { name: 'Text Files', extensions: ['txt', 'md', 'js', 'ts'] },
        { name: 'All Files', extensions: ['*'] }
      ]
    })

    if (result.canceled || !result.filePath) return null

    try {
      await writeFile(result.filePath, content, 'utf-8')
      addRecentFile(result.filePath)
      return {
        success: true,
        path: result.filePath,
        name: basename(result.filePath)
      }
    } catch (error) {
      return { success: false, error: (error as Error).message }
    }
  })

  // ดึง recent files
  ipcMain.handle('file:get-recent', () => {
    return store.get('recentFiles', [])
  })

  // ล้าง recent files
  ipcMain.handle('file:clear-recent', () => {
    store.set('recentFiles', [])
  })
}

function addRecentFile(filePath: string): void {
  const recent = store.get('recentFiles', [])
  const filtered = recent.filter(f => f.path !== filePath)
  filtered.unshift({
    path: filePath,
    name: basename(filePath),
    lastOpened: Date.now()
  })
  store.set('recentFiles', filtered.slice(0, 20))
}
```

---

## 3. Editor Store (Zustand)

```typescript
// src/renderer/store/editorStore.ts
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

export interface EditorTab {
  id: string
  path: string | null
  name: string
  content: string
  originalContent: string
  isDirty: boolean
  language: string
  cursorPosition: { line: number; col: number }
  scrollPosition: number
  encoding: string
}

interface EditorStore {
  tabs: EditorTab[]
  activeTabId: string | null
  showFindReplace: boolean
  findQuery: string
  replaceQuery: string
  wrapLines: boolean
  fontSize: number
  
  // Actions
  createTab: (data?: Partial<EditorTab>) => string
  openFile: (fileData: { path: string; name: string; content: string; extension: string }) => void
  closeTab: (id: string) => void
  setActiveTab: (id: string) => void
  updateContent: (id: string, content: string) => void
  markSaved: (id: string, path: string, name: string) => void
  updateCursor: (id: string, line: number, col: number) => void
  toggleFindReplace: () => void
  setFindQuery: (q: string) => void
  setReplaceQuery: (q: string) => void
  toggleWordWrap: () => void
  setFontSize: (size: number) => void
}

function detectLanguage(ext: string): string {
  const map: Record<string, string> = {
    js: 'javascript', jsx: 'javascript',
    ts: 'typescript', tsx: 'typescript',
    py: 'python', rb: 'ruby',
    css: 'css', scss: 'scss', less: 'less',
    html: 'html', xml: 'xml',
    json: 'json', yaml: 'yaml', yml: 'yaml',
    md: 'markdown', sh: 'shell',
    sql: 'sql', rs: 'rust', go: 'go',
    java: 'java', cpp: 'cpp', c: 'c'
  }
  return map[ext] || 'plaintext'
}

let tabCounter = 1

export const useEditorStore = create<EditorStore>()(
  immer((set, get) => ({
    tabs: [],
    activeTabId: null,
    showFindReplace: false,
    findQuery: '',
    replaceQuery: '',
    wrapLines: false,
    fontSize: 14,

    createTab: (data) => {
      const id = `tab-${Date.now()}-${tabCounter++}`
      const tab: EditorTab = {
        id,
        path: null,
        name: `Untitled ${tabCounter}`,
        content: '',
        originalContent: '',
        isDirty: false,
        language: 'plaintext',
        cursorPosition: { line: 1, col: 1 },
        scrollPosition: 0,
        encoding: 'UTF-8',
        ...data
      }
      set(state => {
        state.tabs.push(tab)
        state.activeTabId = id
      })
      return id
    },

    openFile: (fileData) => {
      const { tabs } = get()
      // ตรวจว่าไฟล์เปิดอยู่แล้ว
      const existing = tabs.find(t => t.path === fileData.path)
      if (existing) {
        set(state => { state.activeTabId = existing.id })
        return
      }

      const id = `tab-${Date.now()}-${tabCounter++}`
      const tab: EditorTab = {
        id,
        path: fileData.path,
        name: fileData.name,
        content: fileData.content,
        originalContent: fileData.content,
        isDirty: false,
        language: detectLanguage(fileData.extension),
        cursorPosition: { line: 1, col: 1 },
        scrollPosition: 0,
        encoding: 'UTF-8'
      }

      set(state => {
        state.tabs.push(tab)
        state.activeTabId = id
      })
    },

    closeTab: (id) => {
      set(state => {
        const idx = state.tabs.findIndex(t => t.id === id)
        state.tabs.splice(idx, 1)
        if (state.activeTabId === id) {
          state.activeTabId = state.tabs[Math.max(0, idx - 1)]?.id || null
        }
      })
    },

    setActiveTab: (id) => set(state => { state.activeTabId = id }),

    updateContent: (id, content) => {
      set(state => {
        const tab = state.tabs.find(t => t.id === id)
        if (tab) {
          tab.content = content
          tab.isDirty = content !== tab.originalContent
        }
      })
    },

    markSaved: (id, path, name) => {
      set(state => {
        const tab = state.tabs.find(t => t.id === id)
        if (tab) {
          tab.path = path
          tab.name = name
          tab.originalContent = tab.content
          tab.isDirty = false
        }
      })
    },

    updateCursor: (id, line, col) => {
      set(state => {
        const tab = state.tabs.find(t => t.id === id)
        if (tab) tab.cursorPosition = { line, col }
      })
    },

    toggleFindReplace: () => set(state => { state.showFindReplace = !state.showFindReplace }),
    setFindQuery: (q) => set(state => { state.findQuery = q }),
    setReplaceQuery: (q) => set(state => { state.replaceQuery = q }),
    toggleWordWrap: () => set(state => { state.wrapLines = !state.wrapLines }),
    setFontSize: (size) => set(state => { state.fontSize = size })
  }))
)
```

---

## 4. Monaco Editor Component

```tsx
// src/renderer/components/Editor.tsx
import { useRef, useEffect, useCallback } from 'react'
import * as monaco from 'monaco-editor'
import { useEditorStore } from '../store/editorStore'

// ติดตั้ง Monaco workers
self.MonacoEnvironment = {
  getWorkerUrl: (_, label) => {
    if (label === 'json') return './json.worker.js'
    if (label === 'css' || label === 'scss' || label === 'less') return './css.worker.js'
    if (label === 'html' || label === 'handlebars' || label === 'razor') return './html.worker.js'
    if (label === 'typescript' || label === 'javascript') return './ts.worker.js'
    return './editor.worker.js'
  }
}

interface EditorProps {
  tabId: string
}

export function Editor({ tabId }: EditorProps) {
  const containerRef = useRef<HTMLDivElement>(null)
  const editorRef = useRef<monaco.editor.IStandaloneCodeEditor | null>(null)
  
  const { tabs, wrapLines, fontSize, updateContent, updateCursor, findQuery, showFindReplace } = useEditorStore()
  const tab = tabs.find(t => t.id === tabId)

  const initEditor = useCallback(() => {
    if (!containerRef.current || !tab) return

    editorRef.current = monaco.editor.create(containerRef.current, {
      value: tab.content,
      language: tab.language,
      theme: 'vs-dark',
      fontSize,
      wordWrap: wrapLines ? 'on' : 'off',
      automaticLayout: true,
      minimap: { enabled: true },
      lineNumbers: 'on',
      renderLineHighlight: 'all',
      cursorBlinking: 'smooth',
      smoothScrolling: true,
      fontFamily: '"JetBrains Mono", "Fira Code", "Cascadia Code", Consolas, monospace',
      fontLigatures: true,
      tabSize: 2,
      insertSpaces: true,
      scrollBeyondLastLine: false,
      bracketPairColorization: { enabled: true },
      guides: {
        bracketPairs: true,
        indentation: true
      },
      suggest: {
        showKeywords: true,
        showSnippets: true
      }
    })

    // Content change handler
    editorRef.current.onDidChangeModelContent(() => {
      const value = editorRef.current?.getValue() || ''
      updateContent(tabId, value)
    })

    // Cursor position handler
    editorRef.current.onDidChangeCursorPosition((e) => {
      updateCursor(tabId, e.position.lineNumber, e.position.column)
    })

    return () => {
      editorRef.current?.dispose()
    }
  }, [tabId])

  useEffect(() => {
    const cleanup = initEditor()
    return cleanup
  }, [tabId])

  // อัพเดท options เมื่อ settings เปลี่ยน
  useEffect(() => {
    editorRef.current?.updateOptions({
      fontSize,
      wordWrap: wrapLines ? 'on' : 'off'
    })
  }, [fontSize, wrapLines])

  // Find in file
  useEffect(() => {
    if (showFindReplace && findQuery) {
      editorRef.current?.trigger('', 'actions.find', null)
    }
  }, [showFindReplace, findQuery])

  return (
    <div
      ref={containerRef}
      className="editor-container"
      style={{ width: '100%', height: '100%' }}
    />
  )
}
```

---

## 5. Tab Bar Component

```tsx
// src/renderer/components/TabBar.tsx
import { useRef } from 'react'
import { useEditorStore } from '../store/editorStore'

export function TabBar() {
  const { tabs, activeTabId, setActiveTab, closeTab, createTab } = useEditorStore()
  const tabBarRef = useRef<HTMLDivElement>(null)

  const handleWheel = (e: React.WheelEvent) => {
    if (tabBarRef.current) {
      tabBarRef.current.scrollLeft += e.deltaY
    }
  }

  const handleClose = (e: React.MouseEvent, tabId: string) => {
    e.stopPropagation()
    const tab = tabs.find(t => t.id === tabId)
    if (tab?.isDirty) {
      if (!confirm(`"${tab.name}" มีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก\nต้องการปิดโดยไม่บันทึกหรือไม่?`)) return
    }
    closeTab(tabId)
  }

  const getFileIcon = (language: string) => {
    const icons: Record<string, string> = {
      javascript: '🟨', typescript: '🔷', python: '🐍',
      markdown: '📝', json: '📋', html: '🌐',
      css: '🎨', rust: '🦀', go: '🔵'
    }
    return icons[language] || '📄'
  }

  return (
    <div className="tab-bar-wrapper">
      <div
        ref={tabBarRef}
        className="tab-bar"
        onWheel={handleWheel}
      >
        {tabs.map(tab => (
          <div
            key={tab.id}
            className={`tab ${tab.id === activeTabId ? 'active' : ''} ${tab.isDirty ? 'dirty' : ''}`}
            onClick={() => setActiveTab(tab.id)}
            title={tab.path || tab.name}
          >
            <span className="tab-icon">{getFileIcon(tab.language)}</span>
            <span className="tab-name">{tab.name}</span>
            {tab.isDirty && <span className="dirty-indicator">●</span>}
            <button
              className="tab-close"
              onClick={(e) => handleClose(e, tab.id)}
              title="ปิด"
            >
              ×
            </button>
          </div>
        ))}
      </div>
      <button
        className="new-tab-btn"
        onClick={() => createTab()}
        title="เปิด Tab ใหม่ (Ctrl+N)"
      >
        +
      </button>
    </div>
  )
}
```

---

## 6. Find & Replace Component

```tsx
// src/renderer/components/FindReplace.tsx
import { useState, useRef, useEffect } from 'react'
import { useEditorStore } from '../store/editorStore'

export function FindReplace() {
  const { showFindReplace, findQuery, replaceQuery, setFindQuery, setReplaceQuery, toggleFindReplace } = useEditorStore()
  const [caseSensitive, setCaseSensitive] = useState(false)
  const [useRegex, setUseRegex] = useState(false)
  const [wholeWord, setWholeWord] = useState(false)
  const [matchCount, setMatchCount] = useState({ current: 0, total: 0 })
  const [showReplace, setShowReplace] = useState(false)
  const findRef = useRef<HTMLInputElement>(null)

  useEffect(() => {
    if (showFindReplace) {
      setTimeout(() => findRef.current?.focus(), 50)
    }
  }, [showFindReplace])

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Escape') toggleFindReplace()
    if (e.key === 'Enter') {
      e.shiftKey ? findPrevious() : findNext()
    }
  }

  const findNext = () => {
    window.dispatchEvent(new CustomEvent('editor:find-next', { detail: { query: findQuery, caseSensitive, useRegex, wholeWord } }))
  }

  const findPrevious = () => {
    window.dispatchEvent(new CustomEvent('editor:find-prev', { detail: { query: findQuery, caseSensitive, useRegex, wholeWord } }))
  }

  const replaceOne = () => {
    window.dispatchEvent(new CustomEvent('editor:replace-one', { detail: { find: findQuery, replace: replaceQuery } }))
  }

  const replaceAll = () => {
    window.dispatchEvent(new CustomEvent('editor:replace-all', { detail: { find: findQuery, replace: replaceQuery } }))
  }

  if (!showFindReplace) return null

  return (
    <div className="find-replace-panel">
      <div className="find-row">
        <input
          ref={findRef}
          type="text"
          placeholder="ค้นหา..."
          value={findQuery}
          onChange={e => setFindQuery(e.target.value)}
          onKeyDown={handleKeyDown}
          className="find-input"
        />
        <div className="find-options">
          <button
            className={`option-btn ${caseSensitive ? 'active' : ''}`}
            onClick={() => setCaseSensitive(!caseSensitive)}
            title="พิมพ์ใหญ่-เล็ก"
          >Aa</button>
          <button
            className={`option-btn ${wholeWord ? 'active' : ''}`}
            onClick={() => setWholeWord(!wholeWord)}
            title="ทั้งคำ"
          >ab</button>
          <button
            className={`option-btn ${useRegex ? 'active' : ''}`}
            onClick={() => setUseRegex(!useRegex)}
            title="Regular Expression"
          >.*</button>
        </div>
        <span className="match-count">{matchCount.total > 0 ? `${matchCount.current}/${matchCount.total}` : 'ไม่พบ'}</span>
        <button onClick={findPrevious} title="ก่อนหน้า (Shift+Enter)">↑</button>
        <button onClick={findNext} title="ถัดไป (Enter)">↓</button>
        <button className="toggle-replace" onClick={() => setShowReplace(!showReplace)}>
          {showReplace ? '▲' : '▼'}
        </button>
        <button className="close-find" onClick={toggleFindReplace}>×</button>
      </div>
      {showReplace && (
        <div className="replace-row">
          <input
            type="text"
            placeholder="แทนที่ด้วย..."
            value={replaceQuery}
            onChange={e => setReplaceQuery(e.target.value)}
            onKeyDown={handleKeyDown}
            className="replace-input"
          />
          <button onClick={replaceOne}>แทนที่</button>
          <button onClick={replaceAll}>แทนที่ทั้งหมด</button>
        </div>
      )}
    </div>
  )
}
```

---

## 7. Status Bar Component

```tsx
// src/renderer/components/StatusBar.tsx
import { useEditorStore } from '../store/editorStore'

export function StatusBar() {
  const { tabs, activeTabId, fontSize, setFontSize, toggleWordWrap, wrapLines } = useEditorStore()
  const activeTab = tabs.find(t => t.id === activeTabId)

  if (!activeTab) {
    return <div className="status-bar"><span>ยินดีต้อนรับสู่ Text Editor</span></div>
  }

  const { cursorPosition, language, encoding, isDirty } = activeTab

  return (
    <div className="status-bar">
      <div className="status-left">
        <span className="status-item cursor-pos" title="ตำแหน่ง cursor">
          Ln {cursorPosition.line}, Col {cursorPosition.col}
        </span>
        <span className="status-item encoding">{encoding}</span>
        {isDirty && <span className="status-item unsaved">● ยังไม่ได้บันทึก</span>}
      </div>
      <div className="status-right">
        <button
          className={`status-btn ${wrapLines ? 'active' : ''}`}
          onClick={toggleWordWrap}
          title="Word Wrap (Alt+Z)"
        >
          Word Wrap: {wrapLines ? 'เปิด' : 'ปิด'}
        </button>
        <span className="status-item language">{language}</span>
        <div className="font-size-control">
          <button onClick={() => setFontSize(Math.max(10, fontSize - 1))}>A-</button>
          <span>{fontSize}px</span>
          <button onClick={() => setFontSize(Math.min(32, fontSize + 1))}>A+</button>
        </div>
      </div>
    </div>
  )
}
```

---

## 8. App Component รวมทุกอย่าง

```tsx
// src/renderer/App.tsx
import { useEffect, useCallback } from 'react'
import { TabBar } from './components/TabBar'
import { Editor } from './components/Editor'
import { StatusBar } from './components/StatusBar'
import { FindReplace } from './components/FindReplace'
import { Sidebar } from './components/Sidebar'
import { useEditorStore } from './store/editorStore'
import './styles/app.css'

export default function App() {
  const { tabs, activeTabId, createTab, openFile, markSaved, toggleFindReplace } = useEditorStore()

  // Keyboard shortcuts
  useEffect(() => {
    const handleKeyDown = async (e: KeyboardEvent) => {
      const mod = e.ctrlKey || e.metaKey

      if (mod && e.key === 'n') {
        e.preventDefault()
        createTab()
      }
      if (mod && e.key === 'o') {
        e.preventDefault()
        const result = await window.electronAPI.openFile()
        if (result && !result.error) openFile(result)
      }
      if (mod && e.key === 's') {
        e.preventDefault()
        if (e.shiftKey) {
          await saveAs()
        } else {
          await save()
        }
      }
      if (mod && e.key === 'f') {
        e.preventDefault()
        toggleFindReplace()
      }
      if (mod && e.key === 'w') {
        e.preventDefault()
        if (activeTabId) closeActiveTab()
      }
    }

    window.addEventListener('keydown', handleKeyDown)
    return () => window.removeEventListener('keydown', handleKeyDown)
  }, [activeTabId, tabs])

  const save = async () => {
    const activeTab = tabs.find(t => t.id === activeTabId)
    if (!activeTab) return

    if (activeTab.path) {
      const result = await window.electronAPI.saveFile({ path: activeTab.path, content: activeTab.content })
      if (result?.success) markSaved(activeTabId!, activeTab.path, activeTab.name)
    } else {
      await saveAs()
    }
  }

  const saveAs = async () => {
    const activeTab = tabs.find(t => t.id === activeTabId)
    if (!activeTab) return

    const result = await window.electronAPI.saveFileAs({ content: activeTab.content, defaultName: activeTab.name })
    if (result?.success) markSaved(activeTabId!, result.path, result.name)
  }

  const closeActiveTab = () => {
    // handled in TabBar
  }

  // Listen for app close
  useEffect(() => {
    window.electronAPI.onBeforeClose(() => {
      const dirtyTabs = tabs.filter(t => t.isDirty)
      if (dirtyTabs.length === 0) {
        window.electronAPI.confirmClose()
        return
      }

      const names = dirtyTabs.map(t => `• ${t.name}`).join('\n')
      const ok = confirm(`ไฟล์ต่อไปนี้มีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก:\n${names}\n\nต้องการออกโดยไม่บันทึกหรือไม่?`)
      if (ok) window.electronAPI.confirmClose()
    })
  }, [tabs])

  return (
    <div className="app">
      <TabBar />
      <div className="main-area">
        <Sidebar />
        <div className="editor-area">
          <FindReplace />
          {tabs.length === 0 ? (
            <div className="welcome-screen">
              <h1>📝 Text Editor</h1>
              <p>เริ่มต้นด้วยการเปิดไฟล์หรือสร้างไฟล์ใหม่</p>
              <div className="quick-actions">
                <button onClick={() => createTab()}>✨ ไฟล์ใหม่ (Ctrl+N)</button>
                <button onClick={async () => {
                  const r = await window.electronAPI.openFile()
                  if (r && !r.error) openFile(r)
                }}>📁 เปิดไฟล์ (Ctrl+O)</button>
              </div>
            </div>
          ) : (
            tabs.map(tab => (
              <div
                key={tab.id}
                className={`editor-wrapper ${tab.id === activeTabId ? 'visible' : 'hidden'}`}
              >
                <Editor tabId={tab.id} />
              </div>
            ))
          )}
        </div>
      </div>
      <StatusBar />
    </div>
  )
}
```

---

## 9. CSS Styles

```css
/* src/renderer/styles/app.css */
:root {
  --bg-primary: #1e1e1e;
  --bg-secondary: #252526;
  --bg-tertiary: #2d2d30;
  --border: #3e3e42;
  --text-primary: #cccccc;
  --text-secondary: #969696;
  --accent: #007acc;
  --accent-hover: #1a8ad4;
  --tab-height: 35px;
  --status-height: 24px;
}

* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  background: var(--bg-primary);
  color: var(--text-primary);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  font-size: 13px;
  height: 100vh;
  overflow: hidden;
}

.app {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

/* Tab Bar */
.tab-bar-wrapper {
  display: flex;
  background: var(--bg-secondary);
  border-bottom: 1px solid var(--border);
  height: var(--tab-height);
  overflow: hidden;
}

.tab-bar {
  display: flex;
  flex: 1;
  overflow-x: auto;
  scrollbar-width: none;
}

.tab-bar::-webkit-scrollbar { display: none; }

.tab {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 0 12px;
  min-width: 120px;
  max-width: 200px;
  cursor: pointer;
  border-right: 1px solid var(--border);
  color: var(--text-secondary);
  white-space: nowrap;
  font-size: 12px;
  transition: background 0.1s;
}

.tab:hover { background: var(--bg-tertiary); }
.tab.active { background: var(--bg-primary); color: var(--text-primary); }
.tab.dirty .tab-name { font-style: italic; }
.dirty-indicator { color: #f5a623; font-size: 16px; line-height: 1; }

.tab-close {
  margin-left: auto;
  background: none;
  border: none;
  color: var(--text-secondary);
  cursor: pointer;
  font-size: 16px;
  padding: 0 2px;
  opacity: 0;
  transition: opacity 0.1s;
}

.tab:hover .tab-close { opacity: 1; }
.tab-close:hover { color: var(--text-primary); }

/* Main Area */
.main-area {
  display: flex;
  flex: 1;
  overflow: hidden;
}

.editor-area {
  display: flex;
  flex-direction: column;
  flex: 1;
  overflow: hidden;
}

.editor-wrapper.hidden { display: none; }
.editor-wrapper.visible { display: flex; flex: 1; }
.editor-container { flex: 1; }

/* Find & Replace */
.find-replace-panel {
  background: var(--bg-secondary);
  border-bottom: 1px solid var(--border);
  padding: 6px 12px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.find-row, .replace-row {
  display: flex;
  align-items: center;
  gap: 6px;
}

.find-input, .replace-input {
  background: var(--bg-tertiary);
  border: 1px solid var(--border);
  color: var(--text-primary);
  padding: 4px 8px;
  border-radius: 3px;
  font-size: 12px;
  width: 250px;
}

.find-input:focus, .replace-input:focus {
  outline: none;
  border-color: var(--accent);
}

/* Status Bar */
.status-bar {
  background: var(--accent);
  color: white;
  height: var(--status-height);
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 12px;
  font-size: 12px;
}

.status-left, .status-right {
  display: flex;
  align-items: center;
  gap: 12px;
}

.status-btn {
  background: none;
  border: none;
  color: white;
  cursor: pointer;
  font-size: 12px;
  opacity: 0.8;
}

.status-btn:hover { opacity: 1; }

/* Welcome Screen */
.welcome-screen {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 16px;
  color: var(--text-secondary);
}

.welcome-screen h1 { font-size: 2em; color: var(--text-primary); }

.quick-actions {
  display: flex;
  gap: 12px;
  margin-top: 16px;
}

.quick-actions button {
  background: var(--bg-secondary);
  border: 1px solid var(--border);
  color: var(--text-primary);
  padding: 10px 20px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
}

.quick-actions button:hover {
  background: var(--bg-tertiary);
  border-color: var(--accent);
}
```

---

## 10. Preload Script

```typescript
// src/preload/index.ts
import { contextBridge, ipcRenderer } from 'electron'

contextBridge.exposeInMainWorld('electronAPI', {
  openFile: (path?: string) => ipcRenderer.invoke('file:open', path),
  saveFile: (data: { path: string; content: string }) => ipcRenderer.invoke('file:save', data),
  saveFileAs: (data: { content: string; defaultName?: string }) => ipcRenderer.invoke('file:save-as', data),
  getRecentFiles: () => ipcRenderer.invoke('file:get-recent'),
  clearRecentFiles: () => ipcRenderer.invoke('file:clear-recent'),
  
  onBeforeClose: (cb: () => void) => {
    ipcRenderer.on('app:before-close', cb)
    return () => ipcRenderer.removeAllListeners('app:before-close')
  },
  confirmClose: () => ipcRenderer.send('app:confirm-close'),
  
  onMenuAction: (cb: (action: string) => void) => {
    ipcRenderer.on('menu:action', (_, action) => cb(action))
  }
})
```

---

## สรุปฟีเจอร์ที่สร้าง

| ฟีเจอร์ | สถานะ |
|---------|-------|
| Monaco Editor integration | ✅ |
| Syntax Highlighting (20+ ภาษา) | ✅ |
| File Tabs พร้อม dirty indicator | ✅ |
| Find & Replace | ✅ |
| Line Numbers | ✅ (built-in Monaco) |
| Word Wrap toggle | ✅ |
| Recent Files | ✅ |
| Unsaved changes warning | ✅ |
| Keyboard shortcuts | ✅ |
| Dark Theme | ✅ |

## ขั้นตอนต่อไป

- เพิ่ม Multi-cursor editing
- เพิ่ม Code snippets
- เพิ่ม Git integration (แสดง diff inline)
- เพิ่ม Extension system
- เพิ่ม Terminal panel ในตัว
