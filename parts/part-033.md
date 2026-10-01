# Part 33: Memory Management ใน Electron

## บทนำ

Memory management เป็นสิ่งสำคัญใน Electron เนื่องจาก app รัน Node.js (main process) และ Chromium (renderer process) ไปพร้อมกัน ในบทนี้จะเรียนรู้วิธี profiling memory, ค้นหา leaks, และ patterns สำหรับจัดการ memory อย่างมีประสิทธิภาพ

## Memory Profiling เบื้องต้น

### process.memoryUsage()

```javascript
// electron/services/memoryProfiler.js
const { app, ipcMain } = require('electron')
const os = require('os')
const fs = require('fs')
const path = require('path')

class MemoryProfiler {
  constructor() {
    this.snapshots = []
    this.leakDetector = null
  }

  /**
   * เก็บ memory snapshot
   */
  takeSnapshot(label = '') {
    const mem = process.memoryUsage()
    const sysMem = {
      total: os.totalmem(),
      free: os.freemem()
    }

    const snapshot = {
      timestamp: Date.now(),
      label,
      process: {
        rss: mem.rss,                       // Resident Set Size - RAM ที่ process ใช้จริง
        heapTotal: mem.heapTotal,           // V8 heap ที่จอง
        heapUsed: mem.heapUsed,             // V8 heap ที่ใช้จริง
        external: mem.external,             // C++ objects ที่ V8 จัดการ
        arrayBuffers: mem.arrayBuffers      // ArrayBuffer + SharedArrayBuffer
      },
      system: {
        total: sysMem.total,
        free: sysMem.free,
        used: sysMem.total - sysMem.free,
        usedPercent: ((sysMem.total - sysMem.free) / sysMem.total * 100).toFixed(1)
      }
    }

    this.snapshots.push(snapshot)
    
    if (process.env.NODE_ENV === 'development') {
      console.log(`[Memory] ${label || 'Snapshot'}:`, {
        rss: `${(mem.rss / 1024 / 1024).toFixed(1)} MB`,
        heap: `${(mem.heapUsed / 1024 / 1024).toFixed(1)}/${(mem.heapTotal / 1024 / 1024).toFixed(1)} MB`
      })
    }

    return snapshot
  }

  /**
   * เปรียบเทียบ 2 snapshots
   */
  diff(snap1, snap2) {
    const toMB = (bytes) => (bytes / 1024 / 1024).toFixed(2)
    const delta = (a, b) => {
      const diff = b - a
      const pct = a > 0 ? ((diff / a) * 100).toFixed(1) : 'N/A'
      return { bytes: diff, mb: toMB(diff), percent: pct }
    }

    return {
      duration: snap2.timestamp - snap1.timestamp,
      rss: delta(snap1.process.rss, snap2.process.rss),
      heapUsed: delta(snap1.process.heapUsed, snap2.process.heapUsed),
      heapTotal: delta(snap1.process.heapTotal, snap2.process.heapTotal),
      external: delta(snap1.process.external, snap2.process.external)
    }
  }

  /**
   * ตรวจสอบการรั่วไหลของ memory
   */
  detectLeak(windowSize = 10, thresholdMB = 5) {
    if (this.snapshots.length < windowSize) {
      return { hasLeak: false, message: 'Not enough data' }
    }

    const recent = this.snapshots.slice(-windowSize)
    const first = recent[0]
    const last = recent[recent.length - 1]
    const growthMB = (last.process.heapUsed - first.process.heapUsed) / 1024 / 1024

    const isIncreasing = recent.every((snap, i) => {
      if (i === 0) return true
      return snap.process.heapUsed >= recent[i - 1].process.heapUsed * 0.95
    })

    return {
      hasLeak: isIncreasing && growthMB > thresholdMB,
      growthMB: growthMB.toFixed(2),
      message: isIncreasing && growthMB > thresholdMB
        ? `Potential leak: +${growthMB.toFixed(2)} MB over ${windowSize} samples`
        : 'No leak detected'
    }
  }

  /**
   * เริ่ม continuous monitoring
   */
  startMonitoring(intervalMs = 5000, label = '') {
    if (this.leakDetector) {
      clearInterval(this.leakDetector)
    }

    this.leakDetector = setInterval(() => {
      this.takeSnapshot(label)
      
      const leakStatus = this.detectLeak()
      if (leakStatus.hasLeak) {
        console.warn(`[MemoryProfiler] ${leakStatus.message}`)
      }
    }, intervalMs)

    return () => {
      clearInterval(this.leakDetector)
      this.leakDetector = null
    }
  }

  /**
   * บันทึก report ลงไฟล์
   */
  saveReport(outputPath) {
    const report = {
      generatedAt: new Date().toISOString(),
      snapshots: this.snapshots,
      leakAnalysis: this.detectLeak(),
      summary: this.getSummary()
    }

    fs.writeFileSync(outputPath, JSON.stringify(report, null, 2))
    return outputPath
  }

  getSummary() {
    if (this.snapshots.length === 0) return null

    const heapValues = this.snapshots.map(s => s.process.heapUsed)
    const rssValues = this.snapshots.map(s => s.process.rss)

    return {
      samples: this.snapshots.length,
      heap: {
        min: (Math.min(...heapValues) / 1024 / 1024).toFixed(1),
        max: (Math.max(...heapValues) / 1024 / 1024).toFixed(1),
        avg: (heapValues.reduce((a, b) => a + b, 0) / heapValues.length / 1024 / 1024).toFixed(1)
      },
      rss: {
        min: (Math.min(...rssValues) / 1024 / 1024).toFixed(1),
        max: (Math.max(...rssValues) / 1024 / 1024).toFixed(1)
      }
    }
  }
}

module.exports = new MemoryProfiler()
```

## Heap Snapshots

```javascript
// electron/services/heapSnapshot.js
const v8 = require('v8')
const fs = require('fs')
const path = require('path')
const { app } = require('electron')

class HeapSnapshotManager {
  constructor() {
    this.snapshotDir = path.join(app.getPath('userData'), 'heap-snapshots')
    this.snapshots = []
    
    if (!fs.existsSync(this.snapshotDir)) {
      fs.mkdirSync(this.snapshotDir, { recursive: true })
    }
  }

  /**
   * สร้าง heap snapshot
   * ใช้ Chrome DevTools เปิดไฟล์ .heapsnapshot เพื่อวิเคราะห์
   */
  async takeHeapSnapshot(label = '') {
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-')
    const filename = `heap-${timestamp}-${label || 'snapshot'}.heapsnapshot`
    const filepath = path.join(this.snapshotDir, filename)

    console.log(`[HeapSnapshot] Taking snapshot: ${filename}`)
    
    // สร้าง snapshot ผ่าน v8
    const snapshotStream = v8.writeHeapSnapshot(filepath)
    
    const stats = fs.statSync(filepath)
    const entry = {
      id: this.snapshots.length + 1,
      filename,
      filepath,
      label,
      timestamp: Date.now(),
      size: stats.size,
      sizeMB: (stats.size / 1024 / 1024).toFixed(2)
    }
    
    this.snapshots.push(entry)
    console.log(`[HeapSnapshot] Saved: ${filename} (${entry.sizeMB} MB)`)
    
    return entry
  }

  /**
   * เปิดไฟล์ใน Chrome DevTools
   */
  openInDevTools(filepath, mainWindow) {
    // ให้ user เปิด DevTools และ load snapshot เอง
    mainWindow.webContents.openDevTools()
    mainWindow.webContents.send('heap:snapshot-ready', { filepath })
  }

  listSnapshots() {
    return this.snapshots
  }

  deleteSnapshot(id) {
    const index = this.snapshots.findIndex(s => s.id === id)
    if (index === -1) return false
    
    const snapshot = this.snapshots[index]
    if (fs.existsSync(snapshot.filepath)) {
      fs.unlinkSync(snapshot.filepath)
    }
    
    this.snapshots.splice(index, 1)
    return true
  }

  setupIpcHandlers(mainWindow) {
    const { ipcMain } = require('electron')
    
    ipcMain.handle('heap:take-snapshot', async (_, label) => {
      return await this.takeHeapSnapshot(label)
    })

    ipcMain.handle('heap:list-snapshots', () => {
      return this.listSnapshots()
    })

    ipcMain.handle('heap:delete-snapshot', (_, id) => {
      return this.deleteSnapshot(id)
    })

    ipcMain.handle('heap:open-folder', () => {
      const { shell } = require('electron')
      shell.openPath(this.snapshotDir)
    })
  }
}

module.exports = new HeapSnapshotManager()
```

## ระบุ Memory Leaks

### Common Leak Patterns และวิธีแก้

```javascript
// electron/examples/memoryLeaks.js

// ❌ WRONG: Event listener ที่ไม่ถูก remove
class BadService {
  constructor(eventBus) {
    // ทุกครั้งที่สร้าง instance จะเพิ่ม listener
    eventBus.on('data', (data) => {
      this.processData(data)
    })
  }

  processData(data) { /* ... */ }
}

// ✅ CORRECT: เก็บ reference เพื่อ remove ภายหลัง
class GoodService {
  constructor(eventBus) {
    this.eventBus = eventBus
    this.dataHandler = (data) => this.processData(data)
    eventBus.on('data', this.dataHandler)
  }

  processData(data) { /* ... */ }

  destroy() {
    this.eventBus.off('data', this.dataHandler)
    this.eventBus = null
  }
}

// ❌ WRONG: Timer ที่ไม่ถูก clear
function startPolling() {
  setInterval(() => {
    fetch('/api/data')
  }, 1000)
}

// ✅ CORRECT: เก็บ timer id และ clear เมื่อไม่ใช้
class PollingService {
  constructor() {
    this.timerId = null
  }

  start() {
    if (this.timerId) return
    
    this.timerId = setInterval(() => {
      this.poll()
    }, 1000)
  }

  stop() {
    if (this.timerId) {
      clearInterval(this.timerId)
      this.timerId = null
    }
  }

  async poll() { /* ... */ }
}

// ❌ WRONG: Cache ไม่มี limit
const imageCache = {}

function loadImage(url) {
  if (!imageCache[url]) {
    imageCache[url] = fetch(url).then(r => r.blob())
  }
  return imageCache[url]
}

// ✅ CORRECT: LRU Cache ที่มี limit
class LRUCache {
  constructor(maxSize = 100) {
    this.maxSize = maxSize
    this.cache = new Map()
  }

  get(key) {
    if (!this.cache.has(key)) return undefined
    
    // Move to end (most recently used)
    const value = this.cache.get(key)
    this.cache.delete(key)
    this.cache.set(key, value)
    return value
  }

  set(key, value) {
    if (this.cache.has(key)) {
      this.cache.delete(key)
    } else if (this.cache.size >= this.maxSize) {
      // Delete least recently used (first item)
      const firstKey = this.cache.keys().next().value
      this.cache.delete(firstKey)
    }
    
    this.cache.set(key, value)
  }

  has(key) { return this.cache.has(key) }
  clear() { this.cache.clear() }
  get size() { return this.cache.size }
}

const imageCache2 = new LRUCache(50)

// ❌ WRONG: Closure ที่ capture ข้อมูลมากเกินไป
function processLargeData(bigArray) {
  const summary = bigArray.reduce((acc, item) => acc + item.value, 0)
  
  return {
    getResult: () => bigArray.length,  // closure keeps bigArray alive!
    summary
  }
}

// ✅ CORRECT: Copy เฉพาะ data ที่จำเป็น
function processLargeData2(bigArray) {
  const length = bigArray.length  // copy only what we need
  const summary = bigArray.reduce((acc, item) => acc + item.value, 0)
  
  return {
    getResult: () => length,  // closure only keeps length, not bigArray
    summary
  }
}
```

## WeakRef และ WeakMap

```javascript
// electron/utils/weakCache.js

/**
 * Cache ที่ใช้ WeakRef - items ถูก GC เมื่อไม่มี strong reference
 */
class WeakCache {
  constructor() {
    this.cache = new Map()
    this.registry = new FinalizationRegistry((key) => {
      // ล้าง dead entry เมื่อ GC เก็บ object แล้ว
      const ref = this.cache.get(key)
      if (ref && !ref.deref()) {
        this.cache.delete(key)
        console.log(`[WeakCache] Cleaned up: ${key}`)
      }
    })
  }

  set(key, value) {
    const ref = new WeakRef(value)
    this.cache.set(key, ref)
    this.registry.register(value, key)
  }

  get(key) {
    const ref = this.cache.get(key)
    if (!ref) return undefined
    
    const value = ref.deref()
    if (!value) {
      this.cache.delete(key)
      return undefined
    }
    
    return value
  }

  has(key) {
    return this.get(key) !== undefined
  }

  get size() {
    return this.cache.size
  }
}

/**
 * WeakMap สำหรับ associate data กับ object โดยไม่ prevent GC
 */
const objectMetadata = new WeakMap()

function attachMetadata(obj, meta) {
  objectMetadata.set(obj, meta)
}

function getMetadata(obj) {
  return objectMetadata.get(obj)
}

// ตัวอย่างการใช้
class WindowManager {
  constructor() {
    // WeakMap - เมื่อ window ถูก destroy, metadata ก็ถูก GC ด้วย
    this.windowData = new WeakMap()
  }

  registerWindow(win, data) {
    this.windowData.set(win, {
      ...data,
      createdAt: Date.now()
    })
  }

  getWindowData(win) {
    return this.windowData.get(win)
  }
}

/**
 * WeakSet สำหรับ tracking objects โดยไม่ prevent GC
 */
class ProcessedSet {
  constructor() {
    this.processed = new WeakSet()
  }

  markProcessed(obj) {
    this.processed.add(obj)
  }

  isProcessed(obj) {
    return this.processed.has(obj)
  }
}

module.exports = { WeakCache, WindowManager, ProcessedSet }
```

## IPC Message Size Limits

```javascript
// electron/ipc/messageGuard.js
const { ipcMain } = require('electron')

const MAX_MESSAGE_SIZE = 10 * 1024 * 1024  // 10 MB

/**
 * Guard IPC messages ที่ใหญ่เกินไป
 */
function createSizeGuardedHandler(handler, maxSizeBytes = MAX_MESSAGE_SIZE) {
  return async (event, ...args) => {
    // ประมาณขนาด message
    const size = estimateSize(args)
    
    if (size > maxSizeBytes) {
      const sizeMB = (size / 1024 / 1024).toFixed(2)
      const limitMB = (maxSizeBytes / 1024 / 1024).toFixed(2)
      throw new Error(`Message too large: ${sizeMB} MB > ${limitMB} MB limit`)
    }
    
    return handler(event, ...args)
  }
}

function estimateSize(value) {
  try {
    return JSON.stringify(value).length * 2  // approximate bytes
  } catch {
    return 0
  }
}

/**
 * สำหรับข้อมูลขนาดใหญ่ ใช้ streaming แทน
 */
class IPCStreamManager {
  constructor() {
    this.streams = new Map()
    this.chunkSize = 256 * 1024  // 256 KB per chunk
    
    this.setupHandlers()
  }

  setupHandlers() {
    // รับข้อมูลขนาดใหญ่แบบ chunks
    ipcMain.handle('stream:start', (_, streamId, totalSize) => {
      this.streams.set(streamId, {
        id: streamId,
        chunks: [],
        totalSize,
        received: 0,
        createdAt: Date.now()
      })
      return { ok: true }
    })

    ipcMain.handle('stream:chunk', (_, streamId, chunkData, chunkIndex) => {
      const stream = this.streams.get(streamId)
      if (!stream) throw new Error(`Unknown stream: ${streamId}`)
      
      stream.chunks[chunkIndex] = chunkData
      stream.received += chunkData.length
      
      return { 
        received: stream.received, 
        progress: stream.received / stream.totalSize 
      }
    })

    ipcMain.handle('stream:end', (_, streamId) => {
      const stream = this.streams.get(streamId)
      if (!stream) throw new Error(`Unknown stream: ${streamId}`)
      
      const combined = stream.chunks.join('')
      this.streams.delete(streamId)
      
      return { data: combined }
    })

    ipcMain.handle('stream:abort', (_, streamId) => {
      this.streams.delete(streamId)
      return { ok: true }
    })
  }

  // ส่งข้อมูลขนาดใหญ่ไปยัง renderer แบบ streaming
  async sendLargeData(webContents, data, channel) {
    const serialized = JSON.stringify(data)
    const streamId = `stream_${Date.now()}_${Math.random().toString(36).slice(2)}`
    const totalChunks = Math.ceil(serialized.length / this.chunkSize)
    
    webContents.send(`${channel}:start`, { streamId, totalChunks, size: serialized.length })
    
    for (let i = 0; i < totalChunks; i++) {
      const chunk = serialized.slice(i * this.chunkSize, (i + 1) * this.chunkSize)
      webContents.send(`${channel}:chunk`, { streamId, chunk, index: i })
      
      // Allow event loop to breathe
      await new Promise(r => setImmediate(r))
    }
    
    webContents.send(`${channel}:end`, { streamId })
  }
}

module.exports = { createSizeGuardedHandler, IPCStreamManager }
```

## Cleanup Patterns

```javascript
// electron/utils/disposable.js

/**
 * Disposable pattern - resource ที่ต้อง cleanup
 */
class Disposable {
  constructor() {
    this._disposed = false
    this._disposables = []
  }

  /**
   * Register resource ที่ต้อง cleanup
   */
  register(disposable) {
    if (this._disposed) {
      // ถ้า disposed แล้ว ให้ dispose ทันที
      if (typeof disposable.dispose === 'function') {
        disposable.dispose()
      } else if (typeof disposable === 'function') {
        disposable()
      }
      return
    }
    
    this._disposables.push(disposable)
  }

  /**
   * Register function ที่รันตอน dispose
   */
  onDispose(fn) {
    this.register({ dispose: fn })
  }

  dispose() {
    if (this._disposed) return
    this._disposed = true
    
    // Dispose ในลำดับย้อนกลับ
    for (let i = this._disposables.length - 1; i >= 0; i--) {
      const d = this._disposables[i]
      try {
        if (typeof d.dispose === 'function') {
          d.dispose()
        } else if (typeof d === 'function') {
          d()
        }
      } catch (err) {
        console.error('[Disposable] Error during dispose:', err)
      }
    }
    
    this._disposables = []
  }

  get isDisposed() {
    return this._disposed
  }
}

/**
 * Window lifecycle cleanup
 */
class WindowService extends Disposable {
  constructor(win) {
    super()
    
    this.win = win
    
    // Setup IPC handlers
    const { ipcMain } = require('electron')
    
    const handler = (_, data) => this.handleData(data)
    ipcMain.on('service:data', handler)
    this.onDispose(() => ipcMain.off('service:data', handler))
    
    // Setup timer
    const timer = setInterval(() => this.tick(), 1000)
    this.onDispose(() => clearInterval(timer))
    
    // Cleanup เมื่อ window ถูกปิด
    const onClosed = () => this.dispose()
    win.once('closed', onClosed)
    this.onDispose(() => {
      // win อาจ destroyed แล้ว
      if (!win.isDestroyed()) {
        win.off('closed', onClosed)
      }
      this.win = null
    })
  }

  handleData(data) {
    if (this.isDisposed) return
    // process data
  }

  tick() {
    if (this.isDisposed) return
    // periodic work
  }
}

/**
 * React hook สำหรับ automatic cleanup
 */
// src/hooks/useDisposable.js
import { useEffect, useRef } from 'react'

export function useDisposable(factory, deps = []) {
  const disposableRef = useRef(null)

  useEffect(() => {
    // สร้าง disposable ใหม่
    const disposable = factory()
    disposableRef.current = disposable
    
    return () => {
      // Cleanup เมื่อ unmount หรือ deps เปลี่ยน
      if (disposableRef.current) {
        if (typeof disposableRef.current.dispose === 'function') {
          disposableRef.current.dispose()
        } else if (typeof disposableRef.current === 'function') {
          disposableRef.current()
        }
        disposableRef.current = null
      }
    }
  }, deps)

  return disposableRef
}

// ตัวอย่างการใช้ใน React component
function MyComponent() {
  useDisposable(() => {
    const subscription = eventEmitter.on('event', handleEvent)
    return subscription  // ต้องมี .dispose() หรือเป็น function
  }, [])
  
  // ...
}
```

## Garbage Collection

```javascript
// electron/utils/gcHelper.js

/**
 * Force GC (ต้อง start Electron ด้วย --expose-gc)
 * เพิ่มใน package.json: "electron": "electron --expose-gc ."
 */
function forceGC() {
  if (typeof global.gc === 'function') {
    const before = process.memoryUsage().heapUsed
    global.gc()
    const after = process.memoryUsage().heapUsed
    const freed = (before - after) / 1024 / 1024
    
    console.log(`[GC] Freed ${freed.toFixed(2)} MB`)
    return { freed, before, after }
  } else {
    console.warn('[GC] global.gc not available. Start with --expose-gc')
    return null
  }
}

/**
 * Schedule GC หลังจาก operation ขนาดใหญ่
 */
function scheduleGC(delayMs = 100) {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve(forceGC())
    }, delayMs)
  })
}

/**
 * GC pressure test - ตรวจสอบว่า GC ทำงานถูกต้อง
 */
async function testGCPressure() {
  const results = []
  
  for (let i = 0; i < 5; i++) {
    // สร้าง garbage
    let largeArray = new Array(1000000).fill(0)
    const beforeRelease = process.memoryUsage().heapUsed
    
    // Release reference
    largeArray = null
    
    // Force GC
    if (global.gc) global.gc()
    
    await new Promise(r => setTimeout(r, 100))
    
    const afterGC = process.memoryUsage().heapUsed
    results.push({
      iteration: i + 1,
      freed: (beforeRelease - afterGC) / 1024 / 1024
    })
  }
  
  console.table(results)
  return results
}

// IPC handler
const { ipcMain } = require('electron')

ipcMain.handle('gc:force', () => {
  return forceGC()
})

ipcMain.handle('gc:memory-usage', () => {
  const mem = process.memoryUsage()
  return {
    rss: mem.rss,
    heapTotal: mem.heapTotal,
    heapUsed: mem.heapUsed,
    external: mem.external,
    arrayBuffers: mem.arrayBuffers
  }
})

module.exports = { forceGC, scheduleGC, testGCPressure }
```

## สรุป

### หลักการจัดการ Memory ใน Electron

1. **ใช้ WeakRef/WeakMap** สำหรับ caches และ metadata ที่ไม่ควรป้องกัน GC
2. **ตั้ง limits สำหรับ cache** ทุกอัน (LRU pattern)
3. **Remove event listeners** เสมอเมื่อ component/service ถูก destroy
4. **Clear timers** ทุกตัวที่สร้างขึ้น
5. **ใช้ Disposable pattern** เพื่อจัดการ cleanup อย่างเป็นระบบ
6. **Limit IPC message size** และใช้ streaming สำหรับข้อมูลขนาดใหญ่
7. **ตรวจสอบด้วย Heap Snapshots** เป็นระยะ

### เครื่องมือ

| เครื่องมือ | ใช้สำหรับ |
|-----------|-----------|
| Chrome DevTools Memory Tab | Heap snapshots, allocation profiler |
| `process.memoryUsage()` | Node.js memory stats |
| `v8.writeHeapSnapshot()` | Save heap ลงไฟล์ |
| `FinalizationRegistry` | Detect เมื่อ object ถูก GC |
| `--expose-gc` flag | Force GC ใน development |
