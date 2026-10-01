# Part 80: Performance Profiling Advanced

## วิเคราะห์และปรับปรุง Performance ของ Electron App

ในบทนี้เราจะเรียน flame graphs, V8 CPU profiling, memory profiling, Electron tracing และ performance optimization techniques

---

## 1. V8 CPU Profiling จาก Main Process

```typescript
// src/main/profiler.ts
import { Session } from 'inspector'
import * as fs from 'fs'
import * as path from 'path'
import { app } from 'electron'

export class CPUProfiler {
  private session: Session
  private profiling = false

  constructor() {
    this.session = new Session()
    this.session.connect()
  }

  async start(): Promise<void> {
    if (this.profiling) return
    this.profiling = true

    return new Promise((resolve, reject) => {
      this.session.post('Profiler.enable', (err) => {
        if (err) return reject(err)
        this.session.post('Profiler.start', (err2) => {
          if (err2) return reject(err2)
          console.log('CPU profiling started')
          resolve()
        })
      })
    })
  }

  async stop(filename = 'cpu-profile'): Promise<string> {
    if (!this.profiling) throw new Error('Profiler not running')
    this.profiling = false

    return new Promise((resolve, reject) => {
      this.session.post('Profiler.stop', (err, params) => {
        if (err) return reject(err)

        const outputPath = path.join(
          app.getPath('downloads'),
          `${filename}-${Date.now()}.cpuprofile`
        )

        fs.writeFileSync(outputPath, JSON.stringify(params.profile))
        console.log(`Profile saved: ${outputPath}`)
        resolve(outputPath)
      })
    })
  }

  disconnect(): void {
    this.session.disconnect()
  }
}

// Heap snapshot
export function takeHeapSnapshot(filename = 'heap-snapshot'): Promise<string> {
  const session = new Session()
  session.connect()

  return new Promise((resolve, reject) => {
    const chunks: string[] = []

    session.on('HeapProfiler.addHeapSnapshotChunk', (m) => {
      chunks.push(m.params.chunk)
    })

    session.post('HeapProfiler.takeHeapSnapshot', { reportProgress: false }, (err) => {
      if (err) {
        session.disconnect()
        return reject(err)
      }

      const outputPath = path.join(
        app.getPath('downloads'),
        `${filename}-${Date.now()}.heapsnapshot`
      )
      fs.writeFileSync(outputPath, chunks.join(''))
      session.disconnect()
      console.log(`Heap snapshot saved: ${outputPath}`)
      resolve(outputPath)
    })
  })
}
```

---

## 2. Electron Tracing

```typescript
// src/main/tracing.ts
import { contentTracing, app } from 'electron'
import * as path from 'path'

export async function startTracing(categories?: string[]): Promise<void> {
  const defaultCategories = [
    'electron',
    'blink',
    'v8',
    'renderer.scheduler',
    'disabled-by-default-v8.cpu_profiler',
    'disabled-by-default-v8.gc',
    'cc',
    'gpu'
  ]

  await contentTracing.startRecording({
    included_categories: categories ?? defaultCategories
  })

  console.log('Tracing started')
}

export async function stopTracing(filename = 'trace'): Promise<string> {
  const outputPath = path.join(
    app.getPath('downloads'),
    `${filename}-${Date.now()}.json`
  )

  const resultPath = await contentTracing.stopRecording(outputPath)
  console.log(`Trace saved: ${resultPath}`)
  return resultPath
}

// ตัวอย่าง: trace operation และวิเคราะห์
export async function profileOperation<T>(
  name: string,
  operation: () => Promise<T>
): Promise<{ result: T; duration: number; traceFile: string }> {
  await startTracing()

  const start = performance.now()
  const result = await operation()
  const duration = performance.now() - start

  const traceFile = await stopTracing(name.replace(/\s/g, '-'))

  return { result, duration, traceFile }
}
```

---

## 3. Memory Leak Detection

```typescript
// src/main/memoryMonitor.ts
import { app, ipcMain } from 'electron'
import * as v8 from 'v8'

interface MemorySnapshot {
  timestamp: number
  heapUsed: number
  heapTotal: number
  external: number
  rss: number
}

export class MemoryMonitor {
  private snapshots: MemorySnapshot[] = []
  private interval: NodeJS.Timeout | null = null
  private threshold: number

  constructor(thresholdMB = 500) {
    this.threshold = thresholdMB * 1024 * 1024
  }

  start(intervalMs = 5000): void {
    this.interval = setInterval(() => {
      const mem = process.memoryUsage()
      const snapshot: MemorySnapshot = {
        timestamp: Date.now(),
        heapUsed: mem.heapUsed,
        heapTotal: mem.heapTotal,
        external: mem.external,
        rss: mem.rss
      }

      this.snapshots.push(snapshot)

      // เก็บ 1 ชั่วโมงล่าสุด (720 snapshots ที่ 5s interval)
      if (this.snapshots.length > 720) {
        this.snapshots.shift()
      }

      // แจ้งเตือนถ้า heap สูงเกิน threshold
      if (mem.heapUsed > this.threshold) {
        console.warn(`High memory usage: ${(mem.heapUsed / 1024 / 1024).toFixed(1)} MB`)
        this.detectLeak()
      }
    }, intervalMs)
  }

  stop(): void {
    if (this.interval) {
      clearInterval(this.interval)
      this.interval = null
    }
  }

  // ตรวจหา memory leak pattern
  private detectLeak(): void {
    if (this.snapshots.length < 12) return  // ต้องมีข้อมูลอย่างน้อย 1 นาที

    const recent = this.snapshots.slice(-12)
    const oldest = recent[0].heapUsed
    const newest = recent[recent.length - 1].heapUsed
    const growthRate = (newest - oldest) / oldest

    if (growthRate > 0.2) {  // เพิ่มขึ้น 20% ใน 1 นาที
      console.error(`Potential memory leak! Heap grew ${(growthRate * 100).toFixed(1)}% in last minute`)
      console.error(`Current heap: ${(newest / 1024 / 1024).toFixed(1)} MB`)
    }
  }

  getStats(): {
    current: MemorySnapshot
    trend: 'stable' | 'growing' | 'shrinking'
    history: MemorySnapshot[]
  } {
    const current = this.snapshots[this.snapshots.length - 1] ?? {
      timestamp: Date.now(),
      ...process.memoryUsage(),
      rss: process.memoryUsage().rss
    }

    let trend: 'stable' | 'growing' | 'shrinking' = 'stable'
    if (this.snapshots.length >= 5) {
      const recent = this.snapshots.slice(-5)
      const first = recent[0].heapUsed
      const last = recent[recent.length - 1].heapUsed
      const diff = (last - first) / first
      if (diff > 0.05) trend = 'growing'
      else if (diff < -0.05) trend = 'shrinking'
    }

    return { current, trend, history: this.snapshots }
  }

  // บังคับ GC (ต้องใช้ --expose-gc flag)
  forceGC(): void {
    if (typeof global.gc === 'function') {
      const before = process.memoryUsage().heapUsed
      global.gc()
      const after = process.memoryUsage().heapUsed
      console.log(`GC freed: ${((before - after) / 1024 / 1024).toFixed(1)} MB`)
    } else {
      console.warn('GC not exposed. Run with --expose-gc flag')
    }
  }
}
```

---

## 4. Renderer Performance Monitoring

```typescript
// src/renderer/performance/monitor.ts

export class RendererPerformanceMonitor {
  private observer: PerformanceObserver | null = null
  private marks: Map<string, number> = new Map()

  // Long task detection
  startLongTaskMonitoring(threshold = 50): void {
    this.observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration > threshold) {
          console.warn(`Long task detected: ${entry.duration.toFixed(0)}ms`, {
            name: entry.name,
            startTime: entry.startTime
          })
        }
      }
    })

    this.observer.observe({ entryTypes: ['longtask'] })
  }

  // Measure arbitrary code blocks
  mark(name: string): void {
    this.marks.set(name, performance.now())
    performance.mark(name)
  }

  measure(name: string, startMark: string): number {
    const start = this.marks.get(startMark)
    if (!start) throw new Error(`Mark "${startMark}" not found`)
    const duration = performance.now() - start
    performance.measure(name, startMark)
    return duration
  }

  // Render performance stats
  getNavigationTiming(): {
    dns: number
    connect: number
    ttfb: number
    load: number
    domInteractive: number
  } {
    const nav = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming
    return {
      dns: nav.domainLookupEnd - nav.domainLookupStart,
      connect: nav.connectEnd - nav.connectStart,
      ttfb: nav.responseStart - nav.requestStart,
      load: nav.loadEventEnd - nav.loadEventStart,
      domInteractive: nav.domInteractive - nav.fetchStart
    }
  }

  // Frame rate monitoring
  startFPSMonitoring(): () => void {
    let lastTime = performance.now()
    let frames = 0
    let rafId: number

    const measure = () => {
      frames++
      const now = performance.now()
      if (now - lastTime >= 1000) {
        const fps = frames
        frames = 0
        lastTime = now
        if (fps < 30) {
          console.warn(`Low FPS: ${fps}`)
        }
      }
      rafId = requestAnimationFrame(measure)
    }

    rafId = requestAnimationFrame(measure)
    return () => cancelAnimationFrame(rafId)
  }

  stop(): void {
    this.observer?.disconnect()
  }
}

// React component profiling wrapper
export function withPerformanceMeasure<P extends object>(
  Component: React.ComponentType<P>,
  name: string
): React.ComponentType<P> {
  return function WrappedComponent(props: P) {
    const monitor = new RendererPerformanceMonitor()
    monitor.mark(`${name}-start`)

    React.useEffect(() => {
      const duration = monitor.measure(`${name}-render`, `${name}-start`)
      if (duration > 16) {
        console.warn(`Slow render: ${name} took ${duration.toFixed(1)}ms`)
      }
    })

    return React.createElement(Component, props)
  }
}

import React from 'react'
```

---

## 5. Performance Dashboard

```tsx
// src/renderer/components/PerformanceDashboard.tsx
import { useState, useEffect } from 'react'

interface PerfMetrics {
  fps: number
  heapUsed: number
  heapTotal: number
  renderTime: number
  ipcLatency: number
}

export function PerformanceDashboard() {
  const [metrics, setMetrics] = useState<PerfMetrics>({
    fps: 60, heapUsed: 0, heapTotal: 0, renderTime: 0, ipcLatency: 0
  })

  useEffect(() => {
    const interval = setInterval(async () => {
      // วัด IPC latency
      const t1 = performance.now()
      await window.electronAPI.invoke('perf:ping')
      const latency = performance.now() - t1

      // ขอ memory info จาก main
      const mem = await window.electronAPI.invoke('perf:memory') as {
        heapUsed: number; heapTotal: number
      }

      setMetrics(prev => ({
        ...prev,
        heapUsed: mem.heapUsed / 1024 / 1024,
        heapTotal: mem.heapTotal / 1024 / 1024,
        ipcLatency: latency
      }))
    }, 2000)

    return () => clearInterval(interval)
  }, [])

  const getColor = (value: number, warn: number, critical: number) => {
    if (value > critical) return '#f85149'
    if (value > warn) return '#e3b341'
    return '#3fb950'
  }

  return (
    <div className="perf-dashboard">
      <h3>Performance Monitor</h3>
      <div className="metrics-grid">
        <MetricCard
          label="FPS"
          value={`${metrics.fps}`}
          color={getColor(60 - metrics.fps, 15, 30)}
        />
        <MetricCard
          label="Heap Used"
          value={`${metrics.heapUsed.toFixed(1)} MB`}
          color={getColor(metrics.heapUsed, 200, 400)}
        />
        <MetricCard
          label="Heap Total"
          value={`${metrics.heapTotal.toFixed(1)} MB`}
          color={getColor(metrics.heapTotal, 300, 500)}
        />
        <MetricCard
          label="IPC Latency"
          value={`${metrics.ipcLatency.toFixed(1)} ms`}
          color={getColor(metrics.ipcLatency, 5, 20)}
        />
      </div>
    </div>
  )
}

function MetricCard({ label, value, color }: { label: string; value: string; color: string }) {
  return (
    <div className="metric-card">
      <div className="metric-label">{label}</div>
      <div className="metric-value" style={{ color }}>{value}</div>
    </div>
  )
}
```

---

## สรุป

| เทคนิค | เครื่องมือ | วิเคราะห์ |
|-------|---------|---------|
| CPU Profiling | V8 Inspector | flame graphs, hot functions |
| Heap Snapshot | Inspector HeapProfiler | memory leaks, retention |
| Electron Tracing | contentTracing | GPU, blink, V8 events |
| Long Task Detection | PerformanceObserver | UI jank |
| FPS Monitoring | requestAnimationFrame | animation smoothness |
| IPC Latency | performance.now() | main↔renderer overhead |
| Memory Trend | process.memoryUsage() | leak detection |
