# Part 94: Error Recovery & Resilience

## สร้าง App ที่ทนทานต่อข้อผิดพลาด

ในบทนี้เราจะเรียน circuit breaker pattern, process watchdog, crash reporting, graceful degradation และ automatic recovery

---

## 1. Circuit Breaker Pattern

```typescript
// src/main/circuitBreaker.ts
// ป้องกันการเรียก service ที่ล้มเหลวซ้ำๆ

type CircuitState = 'closed' | 'open' | 'half-open'

interface CircuitBreakerOptions {
  failureThreshold: number   // จำนวนครั้งที่ล้มเหลวก่อน open
  successThreshold: number   // จำนวนครั้งที่สำเร็จก่อน close จาก half-open
  timeout: number            // ms รอก่อน half-open
  onStateChange?: (from: CircuitState, to: CircuitState) => void
}

export class CircuitBreaker<T> {
  private state: CircuitState = 'closed'
  private failures = 0
  private successes = 0
  private lastFailureTime = 0
  private options: Required<CircuitBreakerOptions>

  constructor(
    private readonly fn: (...args: unknown[]) => Promise<T>,
    options: CircuitBreakerOptions
  ) {
    this.options = {
      onStateChange: () => {},
      ...options
    }
  }

  async execute(...args: unknown[]): Promise<T> {
    if (this.state === 'open') {
      const elapsed = Date.now() - this.lastFailureTime
      if (elapsed < this.options.timeout) {
        throw new Error(`Circuit breaker OPEN: waiting ${this.options.timeout - elapsed}ms`)
      }
      this.transition('half-open')
    }

    try {
      const result = await this.fn(...args)
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }

  private onSuccess(): void {
    this.failures = 0

    if (this.state === 'half-open') {
      this.successes++
      if (this.successes >= this.options.successThreshold) {
        this.successes = 0
        this.transition('closed')
      }
    }
  }

  private onFailure(): void {
    this.failures++
    this.lastFailureTime = Date.now()

    if (this.failures >= this.options.failureThreshold) {
      this.transition('open')
    }
  }

  private transition(newState: CircuitState): void {
    const oldState = this.state
    this.state = newState
    this.options.onStateChange(oldState, newState)
    console.log(`Circuit breaker: ${oldState} → ${newState}`)
  }

  getState(): CircuitState {
    return this.state
  }

  reset(): void {
    this.state = 'closed'
    this.failures = 0
    this.successes = 0
    this.lastFailureTime = 0
  }
}

// ตัวอย่างการใช้
const apiCircuit = new CircuitBreaker(
  async (url: string) => {
    const res = await fetch(url as string)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    return res.json()
  },
  {
    failureThreshold: 5,
    successThreshold: 2,
    timeout: 30_000,
    onStateChange: (from, to) => {
      if (to === 'open') {
        console.warn('API circuit breaker opened - switching to offline mode')
      }
    }
  }
)
```

---

## 2. Retry with Exponential Backoff

```typescript
// src/main/retry.ts

interface RetryOptions {
  maxAttempts: number
  initialDelay: number     // ms
  maxDelay: number         // ms
  factor: number           // exponential factor
  jitter?: boolean         // add randomness
  shouldRetry?: (error: Error, attempt: number) => boolean
}

export async function retry<T>(
  fn: () => Promise<T>,
  options: RetryOptions
): Promise<T> {
  const {
    maxAttempts = 3,
    initialDelay = 1000,
    maxDelay = 30000,
    factor = 2,
    jitter = true,
    shouldRetry = () => true
  } = options

  let lastError: Error = new Error('No attempts made')

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error as Error

      if (attempt === maxAttempts) break
      if (!shouldRetry(lastError, attempt)) break

      let delay = Math.min(initialDelay * Math.pow(factor, attempt - 1), maxDelay)
      if (jitter) {
        delay = delay * (0.5 + Math.random() * 0.5)
      }

      console.log(`Retry attempt ${attempt}/${maxAttempts} in ${Math.round(delay)}ms: ${lastError.message}`)
      await sleep(delay)
    }
  }

  throw lastError
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}
```

---

## 3. Process Watchdog

```typescript
// src/main/watchdog.ts
// ตรวจสอบและ restart renderer process ถ้า crash

import { BrowserWindow, app, ipcMain } from 'electron'
import * as path from 'path'

export class ProcessWatchdog {
  private heartbeatTimeout: NodeJS.Timeout | null = null
  private lastHeartbeat = Date.now()
  private crashCount = 0
  private readonly MAX_CRASHES = 3
  private readonly HEARTBEAT_INTERVAL = 5000
  private readonly HEARTBEAT_TIMEOUT = 15000

  constructor(private createWindow: () => BrowserWindow) {}

  start(win: BrowserWindow): void {
    this.setupHeartbeat(win)
    this.setupCrashHandler(win)
  }

  private setupHeartbeat(win: BrowserWindow): void {
    // Renderer ส่ง heartbeat ทุก 5 วินาที
    ipcMain.on('watchdog:heartbeat', () => {
      this.lastHeartbeat = Date.now()
    })

    // ตรวจสอบ heartbeat
    this.heartbeatTimeout = setInterval(() => {
      const elapsed = Date.now() - this.lastHeartbeat
      if (elapsed > this.HEARTBEAT_TIMEOUT && !win.isDestroyed()) {
        console.error(`Renderer not responding for ${elapsed}ms - reloading...`)
        win.reload()
      }
    }, this.HEARTBEAT_INTERVAL)

    win.on('closed', () => {
      if (this.heartbeatTimeout) clearInterval(this.heartbeatTimeout)
    })
  }

  private setupCrashHandler(win: BrowserWindow): void {
    win.webContents.on('render-process-gone', (_, details) => {
      console.error('Renderer crashed:', details)
      this.crashCount++

      if (this.crashCount > this.MAX_CRASHES) {
        console.error(`Too many crashes (${this.crashCount}). Giving up.`)
        app.quit()
        return
      }

      console.log(`Crash ${this.crashCount}/${this.MAX_CRASHES} - reloading in 1s...`)
      setTimeout(() => {
        if (!win.isDestroyed()) {
          win.reload()
        } else {
          this.createWindow()
        }
      }, 1000)
    })

    win.webContents.on('unresponsive', () => {
      console.warn('Renderer unresponsive - showing recovery dialog')
      win.webContents.send('app:unresponsive-warning')

      // รอ 10 วินาที แล้ว force reload
      setTimeout(() => {
        if (!win.isDestroyed()) {
          win.webContents.forcefullyCrashRenderer()
        }
      }, 10_000)
    })

    win.webContents.on('responsive', () => {
      console.log('Renderer responsive again')
      win.webContents.send('app:responsive-recovered')
    })
  }
}
```

---

## 4. Error Boundary ขั้นสูง

```tsx
// src/renderer/components/ErrorBoundary.tsx
import React from 'react'

interface ErrorState {
  hasError: boolean
  error: Error | null
  errorInfo: React.ErrorInfo | null
  retryCount: number
}

interface ErrorBoundaryProps {
  children: React.ReactNode
  fallback?: React.ComponentType<{ error: Error; retry: () => void; retryCount: number }>
  onError?: (error: Error, info: React.ErrorInfo) => void
  maxRetries?: number
}

export class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorState> {
  private resetTimer: NodeJS.Timeout | null = null

  constructor(props: ErrorBoundaryProps) {
    super(props)
    this.state = { hasError: false, error: null, errorInfo: null, retryCount: 0 }
  }

  static getDerivedStateFromError(error: Error): Partial<ErrorState> {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, info: React.ErrorInfo): void {
    this.setState({ errorInfo: info })
    this.props.onError?.(error, info)

    // Report to crash analytics
    window.electronAPI.invoke('crash:report', {
      message: error.message,
      stack: error.stack,
      componentStack: info.componentStack
    }).catch(console.error)
  }

  retry = (): void => {
    const { maxRetries = 3 } = this.props
    const newCount = this.state.retryCount + 1

    if (newCount > maxRetries) {
      // Force reload หลังจาก retry เกินกำหนด
      window.location.reload()
      return
    }

    this.setState({
      hasError: false,
      error: null,
      errorInfo: null,
      retryCount: newCount
    })
  }

  render(): React.ReactNode {
    if (this.state.hasError && this.state.error) {
      const FallbackComponent = this.props.fallback ?? DefaultErrorFallback

      return (
        <FallbackComponent
          error={this.state.error}
          retry={this.retry}
          retryCount={this.state.retryCount}
        />
      )
    }

    return this.props.children
  }
}

function DefaultErrorFallback({
  error, retry, retryCount
}: { error: Error; retry: () => void; retryCount: number }) {
  return (
    <div className="error-boundary">
      <div className="error-content">
        <h2>เกิดข้อผิดพลาด</h2>
        <p>{error.message}</p>
        <details>
          <summary>รายละเอียดทางเทคนิค</summary>
          <pre>{error.stack}</pre>
        </details>
        <div className="error-actions">
          <button onClick={retry} className="btn-primary">
            ลองใหม่ {retryCount > 0 ? `(${retryCount})` : ''}
          </button>
          <button onClick={() => window.location.reload()} className="btn">
            โหลดหน้าใหม่
          </button>
        </div>
      </div>
    </div>
  )
}
```

---

## 5. Crash Reporting

```typescript
// src/main/crashReporter.ts
import { crashReporter, ipcMain } from 'electron'
import * as fs from 'fs'
import * as path from 'path'

export function setupCrashReporter(): void {
  // Built-in Electron crash reporter
  crashReporter.start({
    submitURL: 'https://crashes.example.com/submit',
    uploadToServer: process.env.NODE_ENV === 'production',
    extra: {
      version: process.env.npm_package_version ?? '0.0.0',
      platform: process.platform
    }
  })

  // Custom crash log handler จาก renderer
  ipcMain.handle('crash:report', async (_, report: {
    message: string
    stack?: string
    componentStack?: string
  }) => {
    const logDir = path.join(require('electron').app.getPath('logs'), 'crashes')
    fs.mkdirSync(logDir, { recursive: true })

    const logFile = path.join(logDir, `crash-${Date.now()}.json`)
    const logEntry = {
      timestamp: new Date().toISOString(),
      ...report,
      platform: process.platform,
      electronVersion: process.versions.electron,
      nodeVersion: process.versions.node
    }

    fs.writeFileSync(logFile, JSON.stringify(logEntry, null, 2))
    console.error('[Crash Report]', report.message)
  })
}

// Uncaught exception handler
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error)
  // บันทึก log แล้ว restart app
})

process.on('unhandledRejection', (reason) => {
  console.error('Unhandled rejection:', reason)
})
```

---

## สรุป

| Pattern | Implementation | ป้องกัน |
|---------|---------------|---------|
| Circuit Breaker | State machine | cascading failures |
| Retry + Backoff | Exponential delay | transient errors |
| Process Watchdog | heartbeat + crash handler | renderer freeze/crash |
| Error Boundary | React error catching | UI crash |
| Crash Reporting | crashReporter + custom | visibility into failures |
| Graceful Degradation | offline fallback | network issues |
