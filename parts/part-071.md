# Part 71: Advanced React Patterns ใน Electron

## React Patterns สำหรับ Electron Apps แบบมืออาชีพ

ในบทนี้เราจะเรียน advanced React patterns ที่เหมาะสำหรับ Electron โดยเฉพาะ: lazy loading สำหรับ IPC modules, Suspense, Error Boundaries, custom hooks สำหรับ system APIs และ React 18 concurrent features

---

## 1. React.lazy สำหรับ IPC Modules

```tsx
// src/renderer/components/LazyIPCComponents.tsx
import { lazy, Suspense, ComponentType } from 'react'

// Lazy load หน้าที่ต้องการ IPC heavily
const SystemMonitor = lazy(() => import('./SystemMonitor'))
const FileManager = lazy(() => import('./FileManager'))
const Settings = lazy(() => import('./Settings'))

// Generic Loader component
function IPCLoader({ message = 'กำลังโหลด...' }: { message?: string }) {
  return (
    <div className="ipc-loader">
      <div className="loader-spinner" />
      <p>{message}</p>
    </div>
  )
}

// Route-based lazy loading
export function AppRouter() {
  const [route, setRoute] = useState('home')

  return (
    <div className="app">
      <nav>
        <button onClick={() => setRoute('monitor')}>Monitor</button>
        <button onClick={() => setRoute('files')}>Files</button>
        <button onClick={() => setRoute('settings')}>Settings</button>
      </nav>

      <Suspense fallback={<IPCLoader message="กำลังโหลดโมดูล..." />}>
        {route === 'monitor' && <SystemMonitor />}
        {route === 'files' && <FileManager />}
        {route === 'settings' && <Settings />}
      </Suspense>
    </div>
  )
}
```

---

## 2. Error Boundary สำหรับ Electron Errors

```tsx
// src/renderer/components/ElectronErrorBoundary.tsx
import React, { Component, ErrorInfo, ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback?: ReactNode
  onError?: (error: Error, info: ErrorInfo) => void
  context?: string
}

interface State {
  hasError: boolean
  error: Error | null
  errorInfo: ErrorInfo | null
}

export class ElectronErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props)
    this.state = { hasError: false, error: null, errorInfo: null }
  }

  static getDerivedStateFromError(error: Error): Partial<State> {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    this.setState({ errorInfo })

    // ส่ง error ไปยัง main process สำหรับ logging
    window.electronAPI?.reportError?.({
      message: error.message,
      stack: error.stack,
      componentStack: errorInfo.componentStack,
      context: this.props.context
    })

    this.props.onError?.(error, errorInfo)
  }

  private handleRetry = () => {
    this.setState({ hasError: false, error: null, errorInfo: null })
  }

  private handleReport = () => {
    const { error, errorInfo } = this.state
    const report = `Error: ${error?.message}\n\nStack: ${error?.stack}\n\nComponent Stack: ${errorInfo?.componentStack}`
    navigator.clipboard.writeText(report)
    alert('คัดลอก error report ไปยัง clipboard แล้ว')
  }

  render() {
    const { hasError, error } = this.state
    const { children, fallback } = this.props

    if (!hasError) return children

    if (fallback) return fallback

    return (
      <div className="error-boundary">
        <div className="error-icon">⚠️</div>
        <h3>เกิดข้อผิดพลาด</h3>
        <p className="error-message">{error?.message}</p>
        <div className="error-actions">
          <button onClick={this.handleRetry} className="btn-primary">ลองใหม่</button>
          <button onClick={this.handleReport}>📋 คัดลอก Report</button>
        </div>
        {process.env.NODE_ENV === 'development' && (
          <details className="error-details">
            <summary>รายละเอียด</summary>
            <pre>{error?.stack}</pre>
          </details>
        )}
      </div>
    )
  }
}

// Higher-order component wrapper
export function withErrorBoundary<P extends object>(
  Component: ComponentType<P>,
  options?: { context?: string; fallback?: ReactNode }
) {
  return function WrappedComponent(props: P) {
    return (
      <ElectronErrorBoundary context={options?.context} fallback={options?.fallback}>
        <Component {...props} />
      </ElectronErrorBoundary>
    )
  }
}
```

---

## 3. Custom Hooks สำหรับ System APIs

```typescript
// src/renderer/hooks/useSystemAPIs.ts
import { useState, useEffect, useCallback, useRef } from 'react'

// Hook สำหรับ file system watching
export function useFileWatcher(path: string | null) {
  const [changes, setChanges] = useState<{ type: string; file: string }[]>([])
  const [isWatching, setIsWatching] = useState(false)

  useEffect(() => {
    if (!path) return

    let cleanup: (() => void) | null = null

    const startWatching = async () => {
      const removeListener = await window.electronAPI.watchDirectory(path, (change) => {
        setChanges(prev => [change, ...prev].slice(0, 50))
      })
      cleanup = removeListener
      setIsWatching(true)
    }

    startWatching()
    return () => { cleanup?.(); setIsWatching(false) }
  }, [path])

  const clearChanges = () => setChanges([])
  return { changes, isWatching, clearChanges }
}

// Hook สำหรับ app theme
export function useNativeTheme() {
  const [theme, setTheme] = useState<'light' | 'dark' | 'system'>('system')
  const [shouldUseDark, setShouldUseDark] = useState(false)

  useEffect(() => {
    const updateTheme = async () => {
      const info = await window.electronAPI.getThemeInfo()
      setShouldUseDark(info.shouldUseDarkColors)
    }

    updateTheme()

    const removeListener = window.electronAPI.onThemeChange((shouldUseDark: boolean) => {
      setShouldUseDark(shouldUseDark)
    })

    return removeListener
  }, [])

  const setNativeTheme = async (newTheme: 'light' | 'dark' | 'system') => {
    await window.electronAPI.setTheme(newTheme)
    setTheme(newTheme)
  }

  return { theme, shouldUseDark, setNativeTheme }
}

// Hook สำหรับ window state
export function useWindowState() {
  const [isMaximized, setIsMaximized] = useState(false)
  const [isFocused, setIsFocused] = useState(true)
  const [bounds, setBounds] = useState({ x: 0, y: 0, width: 800, height: 600 })

  useEffect(() => {
    const cleanups = [
      window.electronAPI.onWindowMaximize(() => setIsMaximized(true)),
      window.electronAPI.onWindowUnmaximize(() => setIsMaximized(false)),
      window.electronAPI.onWindowFocus(() => setIsFocused(true)),
      window.electronAPI.onWindowBlur(() => setIsFocused(false)),
      window.electronAPI.onWindowResize((bounds: typeof bounds) => setBounds(bounds))
    ]

    return () => cleanups.forEach(c => c?.())
  }, [])

  return {
    isMaximized, isFocused, bounds,
    maximize: () => window.electronAPI.maximizeWindow(),
    minimize: () => window.electronAPI.minimizeWindow(),
    close: () => window.electronAPI.closeWindow()
  }
}

// Hook สำหรับ IPC calls พร้อม loading/error state
export function useIPC<T>(
  channel: string,
  options?: { immediate?: boolean; deps?: unknown[] }
) {
  const [data, setData] = useState<T | null>(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<Error | null>(null)
  const abortRef = useRef(false)

  const invoke = useCallback(async (...args: unknown[]): Promise<T | null> => {
    setLoading(true)
    setError(null)
    abortRef.current = false

    try {
      const result = await (window.electronAPI as Record<string, Function>)[channel]?.(...args)
      if (!abortRef.current) setData(result)
      return result
    } catch (err) {
      if (!abortRef.current) setError(err as Error)
      return null
    } finally {
      if (!abortRef.current) setLoading(false)
    }
  }, [channel])

  useEffect(() => {
    if (options?.immediate) invoke()
    return () => { abortRef.current = true }
  }, options?.deps || [])

  const reset = () => { setData(null); setError(null) }

  return { data, loading, error, invoke, reset }
}

// Hook สำหรับ keyboard shortcuts
export function useShortcuts(shortcuts: Record<string, (e: KeyboardEvent) => void>) {
  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      const key = [
        e.ctrlKey || e.metaKey ? 'mod' : '',
        e.shiftKey ? 'shift' : '',
        e.altKey ? 'alt' : '',
        e.key.toLowerCase()
      ].filter(Boolean).join('+')

      shortcuts[key]?.(e)
    }

    window.addEventListener('keydown', handler)
    return () => window.removeEventListener('keydown', handler)
  }, [shortcuts])
}
```

---

## 4. Zustand + IPC Sync Pattern

```typescript
// src/renderer/store/syncedStore.ts
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

// Generic store ที่ sync กับ main process
export function createSyncedStore<T extends Record<string, unknown>>(
  channelPrefix: string,
  initialState: T
) {
  return create<T & {
    _syncFromMain: (data: Partial<T>) => void
    _saveToMain: (key: keyof T, value: T[typeof key]) => void
  }>()(
    immer((set) => ({
      ...initialState,

      _syncFromMain: (data) => set(state => {
        Object.assign(state, data)
      }),

      _saveToMain: (key, value) => {
        window.electronAPI.storeSet(channelPrefix, String(key), value)
        set(state => { (state as Record<string, unknown>)[String(key)] = value })
      }
    }))
  )
}

// ตัวอย่าง: Settings Store ที่ sync กับ electron-store
export interface AppSettings {
  theme: 'light' | 'dark' | 'system'
  fontSize: number
  language: string
  notifications: boolean
  autoUpdate: boolean
}

const DEFAULT_SETTINGS: AppSettings = {
  theme: 'system',
  fontSize: 14,
  language: 'th',
  notifications: true,
  autoUpdate: true
}

export const useSettingsStore = createSyncedStore<AppSettings>('settings', DEFAULT_SETTINGS)

// Initialize - โหลดจาก main process
export async function initSettingsStore() {
  const settings = await window.electronAPI.getSettings()
  useSettingsStore.getState()._syncFromMain(settings)
}

// Listen for changes จาก main process
export function setupSettingsSync() {
  return window.electronAPI.onSettingsChange((settings: Partial<AppSettings>) => {
    useSettingsStore.getState()._syncFromMain(settings)
  })
}
```

---

## 5. React 18 Concurrent Features

```tsx
// src/renderer/components/ConcurrentFeatures.tsx
import {
  useState, useTransition, useDeferredValue,
  startTransition, Suspense, lazy, useId
} from 'react'

// useTransition - ทำให้ UI responsive ระหว่าง heavy update
export function SearchableList({ items }: { items: string[] }) {
  const [query, setQuery] = useState('')
  const [isPending, startTransition] = useTransition()
  const [filteredItems, setFilteredItems] = useState(items)

  const handleSearch = (value: string) => {
    setQuery(value)

    // mark as non-urgent - React สามารถ interrupt ได้
    startTransition(() => {
      const filtered = items.filter(item =>
        item.toLowerCase().includes(value.toLowerCase())
      )
      setFilteredItems(filtered)
    })
  }

  return (
    <div>
      <input
        value={query}
        onChange={e => handleSearch(e.target.value)}
        placeholder="ค้นหา..."
      />
      {isPending && <span>กำลังค้นหา...</span>}
      <ul style={{ opacity: isPending ? 0.6 : 1 }}>
        {filteredItems.map(item => <li key={item}>{item}</li>)}
      </ul>
    </div>
  )
}

// useDeferredValue - defer heavy render
export function FileList({ files }: { files: { name: string; size: number }[] }) {
  const [highlight, setHighlight] = useState('')
  const deferredHighlight = useDeferredValue(highlight)

  return (
    <div>
      <input
        value={highlight}
        onChange={e => setHighlight(e.target.value)}
        placeholder="highlight..."
      />
      <FileListContent files={files} highlight={deferredHighlight} />
    </div>
  )
}

function FileListContent({ files, highlight }: { files: { name: string; size: number }[]; highlight: string }) {
  return (
    <ul>
      {files.map(file => (
        <li key={file.name}>
          <HighlightText text={file.name} highlight={highlight} />
        </li>
      ))}
    </ul>
  )
}

function HighlightText({ text, highlight }: { text: string; highlight: string }) {
  if (!highlight) return <span>{text}</span>
  const parts = text.split(new RegExp(`(${highlight})`, 'gi'))
  return (
    <span>
      {parts.map((part, i) =>
        part.toLowerCase() === highlight.toLowerCase()
          ? <mark key={i}>{part}</mark>
          : part
      )}
    </span>
  )
}

// useId - สำหรับ accessibility ใน dynamic lists
export function FormField({ label, type = 'text' }: { label: string; type?: string }) {
  const id = useId()
  return (
    <div className="form-field">
      <label htmlFor={id}>{label}</label>
      <input id={id} type={type} />
    </div>
  )
}
```

---

## 6. Performance Optimization Patterns

```tsx
// src/renderer/components/OptimizedComponents.tsx
import { memo, useMemo, useCallback, useRef } from 'react'

// Virtualized list สำหรับ large datasets
import { FixedSizeList } from 'react-window'

interface VirtualFileListProps {
  files: { name: string; path: string; size: number }[]
  onSelect: (path: string) => void
  selectedPath: string | null
}

export const VirtualFileList = memo(function VirtualFileList({
  files,
  onSelect,
  selectedPath
}: VirtualFileListProps) {
  const Row = useCallback(({ index, style }: { index: number; style: React.CSSProperties }) => {
    const file = files[index]
    return (
      <div
        style={style}
        className={`file-row ${file.path === selectedPath ? 'selected' : ''}`}
        onClick={() => onSelect(file.path)}
      >
        <span>{file.name}</span>
        <span>{file.size}</span>
      </div>
    )
  }, [files, selectedPath, onSelect])

  return (
    <FixedSizeList
      height={600}
      width="100%"
      itemCount={files.length}
      itemSize={40}
    >
      {Row}
    </FixedSizeList>
  )
})

// Stable callback pattern
export function useStableCallback<T extends (...args: unknown[]) => unknown>(callback: T): T {
  const callbackRef = useRef(callback)
  callbackRef.current = callback

  return useCallback((...args: unknown[]) => {
    return callbackRef.current(...args)
  }, []) as T
}

// Memoized expensive computation
export function useExpensiveFilter<T>(
  items: T[],
  predicate: (item: T) => boolean,
  deps: unknown[]
) {
  return useMemo(() => {
    return items.filter(predicate)
  }, [items, ...deps])
}
```

---

## 7. Context + Reducer Pattern สำหรับ Complex State

```tsx
// src/renderer/context/AppContext.tsx
import { createContext, useContext, useReducer, ReactNode } from 'react'

type Action =
  | { type: 'SET_THEME'; theme: string }
  | { type: 'OPEN_FILE'; path: string }
  | { type: 'CLOSE_FILE'; path: string }
  | { type: 'SET_LOADING'; loading: boolean }

interface AppState {
  theme: string
  openFiles: string[]
  loading: boolean
  notifications: string[]
}

function appReducer(state: AppState, action: Action): AppState {
  switch (action.type) {
    case 'SET_THEME':
      return { ...state, theme: action.theme }
    case 'OPEN_FILE':
      return { ...state, openFiles: [...new Set([...state.openFiles, action.path])] }
    case 'CLOSE_FILE':
      return { ...state, openFiles: state.openFiles.filter(f => f !== action.path) }
    case 'SET_LOADING':
      return { ...state, loading: action.loading }
    default:
      return state
  }
}

const AppContext = createContext<{
  state: AppState
  dispatch: React.Dispatch<Action>
} | null>(null)

export function AppProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(appReducer, {
    theme: 'dark',
    openFiles: [],
    loading: false,
    notifications: []
  })

  return (
    <AppContext.Provider value={{ state, dispatch }}>
      {children}
    </AppContext.Provider>
  )
}

export function useApp() {
  const ctx = useContext(AppContext)
  if (!ctx) throw new Error('useApp must be used within AppProvider')
  return ctx
}
```

---

## สรุป Patterns

| Pattern | ใช้เมื่อไหร่ |
|---------|-------------|
| React.lazy + Suspense | โหลด component ขนาดใหญ่ช้าๆ |
| Error Boundary | จัดการ error ใน IPC calls |
| useTransition | Filter/Search บน large list |
| useDeferredValue | อัพเดท input แต่ delay render |
| Virtual List | แสดงไฟล์จำนวนมาก |
| useId | สร้าง unique ID สำหรับ accessibility |
| Synced Store | Zustand ที่ sync กับ main process |
| Custom IPC Hook | Wrapper สำหรับ IPC calls ที่สะอาด |
