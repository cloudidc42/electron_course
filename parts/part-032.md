# Part 32: Performance Optimization ใน Electron

## บทนำ

Performance optimization ใน Electron ครอบคลุมหลายด้าน ตั้งแต่ startup time, runtime performance, และ memory usage ในบทนี้จะเรียนรู้วิธีวัด performance, ระบุ bottlenecks, และแก้ไขด้วยเทคนิคต่างๆ

## Profiling ด้วย Chrome DevTools

### เปิด DevTools

```javascript
// electron/main.js
function createWindow() {
  const win = new BrowserWindow({
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, 'preload.js')
    }
  })

  // เปิด DevTools ใน development
  if (process.env.NODE_ENV === 'development') {
    win.webContents.openDevTools({ mode: 'detach' })
  }

  // เปิด DevTools ด้วย keyboard shortcut
  win.webContents.on('before-input-event', (event, input) => {
    if (input.type === 'keyDown') {
      if (input.key === 'F12' || 
         (input.control && input.shift && input.key === 'I')) {
        win.webContents.toggleDevTools()
      }
    }
  })
}
```

### Performance Marks และ Measures

```javascript
// electron/main.js
const { app } = require('electron')

// Main process timing
const startupTimes = {
  appStart: process.hrtime.bigint(),
  appReady: null,
  windowCreated: null,
  domReady: null
}

app.whenReady().then(() => {
  startupTimes.appReady = process.hrtime.bigint()
  
  createWindow()
  startupTimes.windowCreated = process.hrtime.bigint()
  
  mainWindow.webContents.once('dom-ready', () => {
    startupTimes.domReady = process.hrtime.bigint()
    
    const toMs = (bigint) => Number(bigint) / 1_000_000
    
    console.log('=== Startup Performance ===')
    console.log(`App Ready: ${toMs(startupTimes.appReady - startupTimes.appStart).toFixed(2)}ms`)
    console.log(`Window Created: ${toMs(startupTimes.windowCreated - startupTimes.appReady).toFixed(2)}ms`)
    console.log(`DOM Ready: ${toMs(startupTimes.domReady - startupTimes.windowCreated).toFixed(2)}ms`)
    console.log(`Total Startup: ${toMs(startupTimes.domReady - startupTimes.appStart).toFixed(2)}ms`)
  })
})
```

```javascript
// src/utils/performance.js - Renderer process
/**
 * Performance measurement utilities
 */
export class PerformanceMonitor {
  constructor(name) {
    this.name = name
    this.marks = new Map()
  }

  mark(label) {
    const markName = `${this.name}:${label}`
    performance.mark(markName)
    this.marks.set(label, performance.now())
    return this
  }

  measure(label, startLabel, endLabel) {
    const measureName = `${this.name}:${label}`
    const startMark = `${this.name}:${startLabel}`
    const endMark = endLabel ? `${this.name}:${endLabel}` : undefined
    
    performance.measure(measureName, startMark, endMark)
    
    const entries = performance.getEntriesByName(measureName)
    const duration = entries[entries.length - 1]?.duration || 0
    
    console.log(`[Perf] ${measureName}: ${duration.toFixed(2)}ms`)
    return duration
  }

  getReport() {
    const entries = performance.getEntriesByType('measure')
    return entries.filter(e => e.name.startsWith(this.name))
  }

  clear() {
    performance.clearMarks()
    performance.clearMeasures()
    this.marks.clear()
  }
}

// ตัวอย่างการใช้
export function measureRender(componentName, fn) {
  const monitor = new PerformanceMonitor(componentName)
  monitor.mark('start')
  const result = fn()
  monitor.mark('end')
  monitor.measure('render', 'start', 'end')
  return result
}

// Web Vitals measurement
export function measureWebVitals(callback) {
  // LCP - Largest Contentful Paint
  const lcpObserver = new PerformanceObserver((list) => {
    const entries = list.getEntries()
    const lcp = entries[entries.length - 1]
    callback({ metric: 'LCP', value: lcp.startTime })
  })
  lcpObserver.observe({ entryTypes: ['largest-contentful-paint'] })

  // FID - First Input Delay
  const fidObserver = new PerformanceObserver((list) => {
    list.getEntries().forEach(entry => {
      callback({ 
        metric: 'FID', 
        value: entry.processingStart - entry.startTime 
      })
    })
  })
  fidObserver.observe({ entryTypes: ['first-input'] })

  // CLS - Cumulative Layout Shift
  let clsValue = 0
  const clsObserver = new PerformanceObserver((list) => {
    list.getEntries().forEach(entry => {
      if (!entry.hadRecentInput) {
        clsValue += entry.value
      }
    })
    callback({ metric: 'CLS', value: clsValue })
  })
  clsObserver.observe({ entryTypes: ['layout-shift'] })

  return () => {
    lcpObserver.disconnect()
    fidObserver.disconnect()
    clsObserver.disconnect()
  }
}
```

## ลด Startup Time

### Lazy Loading

```javascript
// electron/main.js
const { app, BrowserWindow } = require('electron')
const path = require('path')

// Lazy load modules ที่ไม่จำเป็นตอนเริ่มต้น
let autoUpdater = null
let crashReporter = null
let tray = null

function getAutoUpdater() {
  if (!autoUpdater) {
    autoUpdater = require('electron-updater').autoUpdater
  }
  return autoUpdater
}

// โหลด window ก่อน จากนั้น initialize features อื่นๆ
app.whenReady().then(async () => {
  const win = createWindow()
  
  // Initialize features แบบ async หลังจาก window พร้อม
  win.once('ready-to-show', async () => {
    win.show()
    
    // Initialize services แบบ non-blocking
    setImmediate(async () => {
      await initializeServices(win)
    })
  })
})

async function initializeServices(mainWindow) {
  // โหลด services แบบลำดับความสำคัญ
  const { setupIpcHandlers } = require('./ipc')
  setupIpcHandlers(mainWindow)
  
  // Delay heavy initialization
  setTimeout(async () => {
    // Check for updates หลังจาก 5 วินาที
    try {
      const updater = getAutoUpdater()
      await updater.checkForUpdates()
    } catch (err) {
      console.error('Update check failed:', err)
    }
  }, 5000)
}
```

### Code Splitting

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { resolve } from 'path'

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        // Manual chunks สำหรับ better caching
        manualChunks: {
          'vendor-react': ['react', 'react-dom'],
          'vendor-router': ['react-router-dom'],
          'vendor-state': ['@reduxjs/toolkit', 'react-redux'],
          'vendor-ui': ['lucide-react'],
        }
      }
    },
    // เพิ่ม chunk size limit
    chunkSizeWarningLimit: 1000
  }
})
```

```jsx
// src/router/index.jsx - Route-based code splitting
import { lazy, Suspense } from 'react'
import { createHashRouter, RouterProvider } from 'react-router-dom'
import LoadingSpinner from '../components/LoadingSpinner'

// Lazy load routes
const HomeView = lazy(() => import('../views/HomeView'))
const FilesView = lazy(() => import('../views/FilesView'))
const EditorView = lazy(() => import('../views/EditorView'))
const SettingsView = lazy(() => import('../views/SettingsView'))
const AnalyticsView = lazy(() => import('../views/AnalyticsView'))

// Preload ถ้าน่าจะถูก navigate ไปเร็วๆ
function preloadRoute(routeImport) {
  return routeImport()
}

const router = createHashRouter([
  {
    path: '/',
    element: <Layout />,
    children: [
      {
        index: true,
        element: (
          <Suspense fallback={<LoadingSpinner />}>
            <HomeView />
          </Suspense>
        )
      },
      {
        path: 'files',
        element: (
          <Suspense fallback={<LoadingSpinner />}>
            <FilesView />
          </Suspense>
        ),
        // Preload ก่อน navigate
        loader: () => preloadRoute(() => import('../views/FilesView'))
      },
      // ...
    ]
  }
])

export default router
```

## Background Tasks Optimization

```javascript
// electron/workers/backgroundScheduler.js
const { app } = require('electron')

class BackgroundScheduler {
  constructor() {
    this.tasks = []
    this.running = false
    this.idleTimeout = null
  }

  /**
   * Schedule task ให้รันตอน idle
   */
  scheduleIdleTask(fn, priority = 'normal') {
    return new Promise((resolve, reject) => {
      this.tasks.push({ fn, priority, resolve, reject })
      this.startIfIdle()
    })
  }

  startIfIdle() {
    if (this.running) return
    
    // ใช้ requestIdleCallback ถ้ามี (renderer) หรือ setTimeout (main)
    if (typeof requestIdleCallback !== 'undefined') {
      requestIdleCallback((deadline) => {
        this.runTasks(deadline)
      }, { timeout: 5000 })
    } else {
      setImmediate(() => this.runTasks({ timeRemaining: () => 50 }))
    }
  }

  runTasks(deadline) {
    this.running = true
    
    const sortedTasks = this.tasks.sort((a, b) => {
      const priorities = { high: 0, normal: 1, low: 2 }
      return (priorities[a.priority] || 1) - (priorities[b.priority] || 1)
    })

    while (sortedTasks.length > 0 && deadline.timeRemaining() > 0) {
      const task = sortedTasks.shift()
      const index = this.tasks.indexOf(task)
      if (index > -1) this.tasks.splice(index, 1)
      
      try {
        const result = task.fn()
        if (result instanceof Promise) {
          result.then(task.resolve).catch(task.reject)
        } else {
          task.resolve(result)
        }
      } catch (error) {
        task.reject(error)
      }
    }

    this.running = false
    
    if (this.tasks.length > 0) {
      this.startIfIdle()
    }
  }
}

module.exports = new BackgroundScheduler()
```

## Renderer Process Optimization

```jsx
// src/hooks/useVirtualList.js
import { useMemo, useRef, useState, useCallback } from 'react'

/**
 * Virtual list hook สำหรับ large datasets
 * Render เฉพาะ items ที่อยู่ใน viewport
 */
export function useVirtualList(items, options = {}) {
  const {
    itemHeight = 60,
    overscan = 3,     // render items เพิ่มทั้งด้านบนและล่าง
    containerHeight = 600
  } = options

  const [scrollTop, setScrollTop] = useState(0)
  const containerRef = useRef(null)

  const visibleItems = useMemo(() => {
    const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan)
    const endIndex = Math.min(
      items.length - 1,
      Math.ceil((scrollTop + containerHeight) / itemHeight) + overscan
    )

    return {
      startIndex,
      endIndex,
      items: items.slice(startIndex, endIndex + 1),
      totalHeight: items.length * itemHeight,
      offsetY: startIndex * itemHeight
    }
  }, [items, scrollTop, itemHeight, containerHeight, overscan])

  const handleScroll = useCallback((e) => {
    setScrollTop(e.target.scrollTop)
  }, [])

  return {
    containerRef,
    visibleItems,
    handleScroll,
    totalHeight: items.length * itemHeight
  }
}

// Component ที่ใช้ virtual list
export function VirtualList({ items, itemHeight = 60, renderItem }) {
  const containerHeight = 600
  const { containerRef, visibleItems, handleScroll, totalHeight } = useVirtualList(
    items,
    { itemHeight, containerHeight }
  )

  return (
    <div
      ref={containerRef}
      style={{ height: containerHeight, overflow: 'auto' }}
      onScroll={handleScroll}
    >
      {/* Spacer ด้านบน */}
      <div style={{ height: visibleItems.offsetY }} />
      
      {/* Items ที่ visible */}
      {visibleItems.items.map((item, i) => (
        <div
          key={item.id || visibleItems.startIndex + i}
          style={{ height: itemHeight }}
        >
          {renderItem(item, visibleItems.startIndex + i)}
        </div>
      ))}
      
      {/* Spacer ด้านล่าง */}
      <div style={{ 
        height: totalHeight - visibleItems.offsetY - 
                visibleItems.items.length * itemHeight 
      }} />
    </div>
  )
}
```

### React Performance Hooks

```jsx
// src/hooks/useOptimizedState.js
import { useState, useCallback, useTransition, useDeferredValue } from 'react'

/**
 * Hook สำหรับ search ที่ไม่ block UI
 */
export function useDeferredSearch(items, searchFn) {
  const [query, setQuery] = useState('')
  const deferredQuery = useDeferredValue(query)
  
  // เกิดขึ้นใน background ไม่ block UI
  const results = useMemo(() => {
    if (!deferredQuery) return items
    return searchFn(items, deferredQuery)
  }, [items, deferredQuery, searchFn])
  
  const isStale = query !== deferredQuery
  
  return { query, setQuery, results, isStale }
}

/**
 * Hook สำหรับ expensive computation
 */
export function useExpensiveComputation(data, computeFn, deps) {
  const [isPending, startTransition] = useTransition()
  const [result, setResult] = useState(null)
  
  const compute = useCallback(() => {
    startTransition(() => {
      const computed = computeFn(data)
      setResult(computed)
    })
  }, deps)
  
  return { result, isPending, compute }
}
```

## Memory Profiling

```javascript
// electron/services/memoryMonitor.js
const { app } = require('electron')
const os = require('os')

class MemoryMonitor {
  constructor(options = {}) {
    this.interval = options.interval || 30000  // 30 seconds
    this.threshold = options.threshold || 200  // MB
    this.history = []
    this.maxHistory = 100
    this.monitoring = false
    this.timerId = null
  }

  start() {
    if (this.monitoring) return
    this.monitoring = true
    
    this.timerId = setInterval(() => {
      this.collectMetrics()
    }, this.interval)
    
    // Collect initial metrics
    this.collectMetrics()
    console.log('[MemoryMonitor] Started')
  }

  stop() {
    if (this.timerId) {
      clearInterval(this.timerId)
      this.timerId = null
    }
    this.monitoring = false
  }

  collectMetrics() {
    const processMemory = process.memoryUsage()
    const systemMemory = {
      total: os.totalmem(),
      free: os.freemem(),
      used: os.totalmem() - os.freemem()
    }

    const metrics = {
      timestamp: Date.now(),
      process: {
        rss: processMemory.rss / 1024 / 1024,           // MB
        heapUsed: processMemory.heapUsed / 1024 / 1024,
        heapTotal: processMemory.heapTotal / 1024 / 1024,
        external: processMemory.external / 1024 / 1024,
        arrayBuffers: processMemory.arrayBuffers / 1024 / 1024
      },
      system: {
        total: systemMemory.total / 1024 / 1024 / 1024,  // GB
        free: systemMemory.free / 1024 / 1024 / 1024,
        usedPercent: (systemMemory.used / systemMemory.total * 100).toFixed(1)
      }
    }

    this.history.push(metrics)
    if (this.history.length > this.maxHistory) {
      this.history.shift()
    }

    // แจ้งเตือนถ้าใช้ memory มากเกินไป
    if (metrics.process.rss > this.threshold) {
      console.warn(`[MemoryMonitor] High memory usage: ${metrics.process.rss.toFixed(1)} MB`)
      this.onHighMemory(metrics)
    }

    return metrics
  }

  onHighMemory(metrics) {
    // Force GC ถ้า expose ไว้
    if (global.gc) {
      global.gc()
      console.log('[MemoryMonitor] GC triggered')
    }
  }

  getReport() {
    if (this.history.length === 0) return null
    
    const latest = this.history[this.history.length - 1]
    const oldest = this.history[0]
    
    const maxRss = Math.max(...this.history.map(m => m.process.rss))
    const avgHeap = this.history.reduce((sum, m) => sum + m.process.heapUsed, 0) / 
                    this.history.length

    return {
      current: latest,
      statistics: {
        maxRss: maxRss.toFixed(1),
        avgHeap: avgHeap.toFixed(1),
        duration: ((latest.timestamp - oldest.timestamp) / 1000).toFixed(0),
        samples: this.history.length
      },
      trend: this.calculateTrend()
    }
  }

  calculateTrend() {
    if (this.history.length < 5) return 'insufficient data'
    
    const recent = this.history.slice(-5)
    const first = recent[0].process.rss
    const last = recent[recent.length - 1].process.rss
    const change = ((last - first) / first * 100)
    
    if (change > 10) return 'increasing'
    if (change < -10) return 'decreasing'
    return 'stable'
  }
}

module.exports = new MemoryMonitor()
```

## Performance Dashboard

```jsx
// src/components/PerformanceDashboard.jsx
import { useState, useEffect, useRef } from 'react'

export default function PerformanceDashboard() {
  const [metrics, setMetrics] = useState(null)
  const [history, setHistory] = useState([])
  const rafRef = useRef(null)
  const frameCount = useRef(0)
  const lastTime = useRef(performance.now())
  const [fps, setFps] = useState(60)

  // Measure FPS
  useEffect(() => {
    const measureFPS = () => {
      frameCount.current++
      const now = performance.now()
      
      if (now - lastTime.current >= 1000) {
        setFps(frameCount.current)
        frameCount.current = 0
        lastTime.current = now
      }
      
      rafRef.current = requestAnimationFrame(measureFPS)
    }
    
    rafRef.current = requestAnimationFrame(measureFPS)
    return () => cancelAnimationFrame(rafRef.current)
  }, [])

  // Collect metrics
  useEffect(() => {
    const collect = async () => {
      if (!window.electronAPI) return
      
      const data = await window.electronAPI.invoke('perf:getMetrics')
      setMetrics(data)
      setHistory(prev => [...prev.slice(-30), data])
    }
    
    collect()
    const interval = setInterval(collect, 2000)
    return () => clearInterval(interval)
  }, [])

  if (!metrics) return <div>Loading metrics...</div>

  return (
    <div className="perf-dashboard">
      <h2>Performance Monitor</h2>
      
      <div className="metrics-grid">
        <MetricCard
          title="FPS"
          value={fps}
          unit="fps"
          status={fps >= 55 ? 'good' : fps >= 30 ? 'warn' : 'bad'}
        />
        <MetricCard
          title="Heap Used"
          value={(metrics.heapUsed / 1024 / 1024).toFixed(1)}
          unit="MB"
          status={metrics.heapUsed < 100 * 1024 * 1024 ? 'good' : 'warn'}
        />
        <MetricCard
          title="RSS"
          value={(metrics.rss / 1024 / 1024).toFixed(1)}
          unit="MB"
          status={metrics.rss < 200 * 1024 * 1024 ? 'good' : 'bad'}
        />
        <MetricCard
          title="External"
          value={(metrics.external / 1024 / 1024).toFixed(1)}
          unit="MB"
          status="info"
        />
      </div>

      {/* Memory History Chart */}
      <div className="memory-chart">
        <h3>Memory Usage History</h3>
        <svg width="100%" height="80" viewBox="0 0 300 80">
          {history.length > 1 && (
            <polyline
              points={history.map((m, i) => {
                const x = (i / (history.length - 1)) * 300
                const y = 80 - (m.heapUsed / (256 * 1024 * 1024)) * 80
                return `${x},${y}`
              }).join(' ')}
              fill="none"
              stroke="#6366f1"
              strokeWidth="2"
            />
          )}
        </svg>
      </div>

      <div className="perf-actions">
        <button onClick={() => window.electronAPI?.invoke('perf:forceGC')}>
          🗑️ Force GC
        </button>
        <button onClick={() => window.electronAPI?.invoke('perf:clearCache')}>
          🧹 Clear Cache
        </button>
      </div>
    </div>
  )
}

function MetricCard({ title, value, unit, status }) {
  const colors = { good: '#10b981', warn: '#f59e0b', bad: '#ef4444', info: '#6366f1' }
  
  return (
    <div className="metric-card" style={{ borderColor: colors[status] }}>
      <div className="metric-value" style={{ color: colors[status] }}>
        {value} <span className="metric-unit">{unit}</span>
      </div>
      <div className="metric-title">{title}</div>
    </div>
  )
}
```

## สรุป

### เครื่องมือ Profiling

1. **Chrome DevTools Performance Tab** - JS execution profiling
2. **Chrome DevTools Memory Tab** - Heap snapshots, memory leaks
3. **Timeline** - Frame-by-frame analysis
4. **process.memoryUsage()** - Node.js memory stats

### Optimization Strategies

| ปัญหา | วิธีแก้ |
|-------|---------|
| Slow startup | Lazy loading, code splitting |
| Janky UI | Virtual lists, React.memo, useMemo |
| High memory | WeakMap/WeakRef, cleanup, limit cache |
| Slow IPC | Batch requests, avoid large payloads |
| Heavy compute | Worker threads, background scheduler |
