# Part 26: Crash Reporter & Error Handling ใน Electron

## บทนำ

การจัดการ errors อย่างถูกต้องใน Electron มีความสำคัญมาก ทั้งสำหรับ debugging และ user experience ในบทนี้จะครอบคลุม crashReporter, Sentry integration, error boundaries, logging ด้วย Winston, และ crash dumps

## crashReporter

### ตั้งค่า crashReporter

```javascript
// electron/main.js
const { crashReporter, app } = require('electron')
const path = require('path')

// เริ่ม crashReporter ก่อน app ready
crashReporter.start({
  productName: 'My Electron App',
  companyName: 'Your Company',
  submitURL: 'https://your-crash-server.com/crashes',
  uploadToServer: true,
  ignoreSystemCrashHandler: false,
  rateLimit: true,
  compress: true,
  extra: {
    appVersion: app.getVersion(),
    platform: process.platform,
    arch: process.arch
  }
})

// ดึง crash dump path
console.log('Crash dumps location:', crashReporter.getCrashesDirectory())

// ดึงรายการ crash ล่าสุด
const lastCrash = crashReporter.getLastCrash()
if (lastCrash) {
  console.log('Last crash:', lastCrash)
}
```

### Local Crash Reporting

```javascript
// electron/services/crashHandler.js
const { crashReporter, app, dialog } = require('electron')
const path = require('path')
const fs = require('fs')

class CrashHandler {
  constructor() {
    this.crashLogPath = path.join(app.getPath('userData'), 'crashes')
    this.ensureCrashDir()
  }

  ensureCrashDir() {
    if (!fs.existsSync(this.crashLogPath)) {
      fs.mkdirSync(this.crashLogPath, { recursive: true })
    }
  }

  /**
   * ตั้งค่า crash reporter สำหรับ local storage
   */
  setupLocalCrashReporter() {
    crashReporter.start({
      productName: app.getName(),
      companyName: 'Your Company',
      submitURL: '',           // ว่างหมายความว่า local only
      uploadToServer: false,
      ignoreSystemCrashHandler: false
    })
  }

  /**
   * ตรวจสอบว่า crash ก่อนหน้า
   */
  checkPreviousCrash() {
    const lastCrash = crashReporter.getLastCrash()
    
    if (lastCrash && lastCrash.date) {
      const crashTime = new Date(lastCrash.date)
      const now = new Date()
      const hoursSinceCrash = (now - crashTime) / (1000 * 60 * 60)
      
      // ถ้า crash เกิดภายใน 1 ชั่วโมง แสดง dialog
      if (hoursSinceCrash < 1) {
        return lastCrash
      }
    }
    
    return null
  }

  /**
   * แสดง dialog เมื่อพบ crash ก่อนหน้า
   */
  async showCrashRecoveryDialog(crashInfo) {
    const response = await dialog.showMessageBox({
      type: 'warning',
      title: 'แอปหยุดทำงานก่อนหน้า',
      message: 'แอปหยุดทำงานโดยไม่คาดคิดเมื่อครั้งที่แล้ว',
      detail: `เวลา: ${new Date(crashInfo.date).toLocaleString('th-TH')}\n\nต้องการส่งรายงานข้อผิดพลาดหรือไม่?`,
      buttons: ['ส่งรายงาน', 'ไม่ส่ง', 'ดูรายละเอียด'],
      defaultId: 0
    })

    if (response.response === 0) {
      await this.submitCrashReport(crashInfo)
    } else if (response.response === 2) {
      this.showCrashDetails(crashInfo)
    }
  }

  async submitCrashReport(crashInfo) {
    // ส่งไปยัง Sentry หรือ custom endpoint
    console.log('Submitting crash report:', crashInfo)
  }

  showCrashDetails(crashInfo) {
    dialog.showMessageBox({
      type: 'info',
      title: 'รายละเอียด Crash',
      message: JSON.stringify(crashInfo, null, 2)
    })
  }
}

module.exports = new CrashHandler()
```

## Sentry Integration

```bash
npm install @sentry/electron
```

```javascript
// electron/services/sentry.js
const Sentry = require('@sentry/electron/main')
const { app } = require('electron')

/**
 * ตั้งค่า Sentry สำหรับ Main Process
 */
function initSentryMain() {
  Sentry.init({
    dsn: process.env.SENTRY_DSN || 'https://your-key@sentry.io/your-project',
    
    environment: process.env.NODE_ENV || 'production',
    release: `my-app@${app.getVersion()}`,
    
    // ตั้งค่า sampling
    tracesSampleRate: 0.1,        // Sample 10% ของ transactions
    profilesSampleRate: 0.1,
    
    // Integrations
    integrations: [
      Sentry.electronMinidumpIntegration(),  // Capture minidumps
    ],
    
    // ปรับแต่ง events ก่อนส่ง
    beforeSend(event, hint) {
      // กรอง events ที่ไม่ต้องการ
      if (event.exception?.values?.[0]?.type === 'NetworkError') {
        return null // ไม่ส่ง network errors
      }
      
      // เพิ่ม context
      event.contexts = {
        ...event.contexts,
        app: {
          version: app.getVersion(),
          name: app.getName(),
          platform: process.platform
        }
      }
      
      return event
    },
    
    // Filter ข้อมูล sensitive
    beforeSendTransaction(event) {
      // ลบ sensitive data
      return event
    }
  })
}

/**
 * เพิ่ม user context
 */
function setUserContext(user) {
  Sentry.setUser({
    id: user.id,
    email: user.email,
    username: user.username
  })
}

/**
 * Capture exception manually
 */
function captureException(error, context = {}) {
  Sentry.withScope((scope) => {
    // เพิ่ม extra context
    Object.entries(context).forEach(([key, value]) => {
      scope.setExtra(key, value)
    })
    
    Sentry.captureException(error)
  })
}

/**
 * Capture message
 */
function captureMessage(message, level = 'info') {
  Sentry.captureMessage(message, level)
}

/**
 * เริ่ม transaction (performance monitoring)
 */
function startTransaction(name, op = 'task') {
  return Sentry.startNewTrace(() => {
    return {
      name,
      op,
      finish: () => {}
    }
  })
}

module.exports = {
  initSentryMain,
  setUserContext,
  captureException,
  captureMessage,
  startTransaction
}
```

### Sentry สำหรับ Renderer Process

```javascript
// src/sentry.js
import * as Sentry from '@sentry/electron/renderer'

export function initSentryRenderer() {
  Sentry.init({
    // DSN จะ inherit จาก main process อัตโนมัติ
    tracesSampleRate: 0.1,
    
    beforeSend(event) {
      // Filter ข้อมูล sensitive
      if (event.request?.cookies) {
        delete event.request.cookies
      }
      return event
    },
    
    integrations: [
      Sentry.browserTracingIntegration()
    ]
  })
}

// Error boundary สำหรับ React
export function reportError(error, errorInfo) {
  Sentry.withScope(scope => {
    scope.setExtra('componentStack', errorInfo?.componentStack)
    Sentry.captureException(error)
  })
}
```

## Winston Logger

```bash
npm install winston winston-daily-rotate-file
```

```javascript
// electron/services/logger.js
const winston = require('winston')
const DailyRotateFile = require('winston-daily-rotate-file')
const path = require('path')
const { app } = require('electron')

// Custom log format
const logFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss.SSS' }),
  winston.format.errors({ stack: true }),
  winston.format.printf(({ timestamp, level, message, stack, ...meta }) => {
    let log = `[${timestamp}] [${level.toUpperCase().padEnd(5)}] ${message}`
    
    if (Object.keys(meta).length > 0) {
      log += ` ${JSON.stringify(meta)}`
    }
    
    if (stack) {
      log += `\n${stack}`
    }
    
    return log
  })
)

// JSON format สำหรับ file logging
const jsonFormat = winston.format.combine(
  winston.format.timestamp(),
  winston.format.errors({ stack: true }),
  winston.format.json()
)

function createLogger(options = {}) {
  const logDir = options.logDir || path.join(
    app ? app.getPath('userData') : process.cwd(),
    'logs'
  )

  const transports = [
    // Console transport สำหรับ development
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        logFormat
      ),
      level: process.env.NODE_ENV === 'development' ? 'debug' : 'warn',
      silent: process.env.NODE_ENV === 'test'
    }),

    // File transport ด้วย daily rotation
    new DailyRotateFile({
      filename: path.join(logDir, 'app-%DATE%.log'),
      datePattern: 'YYYY-MM-DD',
      zippedArchive: true,
      maxSize: '20m',
      maxFiles: '14d',    // เก็บ 14 วัน
      level: 'info',
      format: jsonFormat
    }),

    // Error file แยกต่างหาก
    new DailyRotateFile({
      filename: path.join(logDir, 'error-%DATE%.log'),
      datePattern: 'YYYY-MM-DD',
      zippedArchive: true,
      maxSize: '10m',
      maxFiles: '30d',
      level: 'error',
      format: jsonFormat
    })
  ]

  const logger = winston.createLogger({
    level: 'debug',
    format: logFormat,
    transports,
    exitOnError: false,
    
    // Handle exceptions
    exceptionHandlers: [
      new winston.transports.File({
        filename: path.join(logDir, 'exceptions.log'),
        format: jsonFormat
      })
    ],
    
    // Handle unhandled rejections
    rejectionHandlers: [
      new winston.transports.File({
        filename: path.join(logDir, 'rejections.log'),
        format: jsonFormat
      })
    ]
  })

  // เพิ่ม methods ที่สะดวก
  logger.logError = (error, context = {}) => {
    logger.error(error.message, {
      stack: error.stack,
      name: error.name,
      ...context
    })
  }

  logger.logAction = (action, data = {}) => {
    logger.info(`Action: ${action}`, data)
  }

  return logger
}

const logger = createLogger()
module.exports = logger
```

## Error Handling ใน Main Process

```javascript
// electron/services/errorHandler.js
const logger = require('./logger')
const { captureException } = require('./sentry')
const { app, dialog } = require('electron')

/**
 * ตั้งค่า global error handlers
 */
function setupGlobalErrorHandlers() {
  
  // Uncaught exceptions ใน main process
  process.on('uncaughtException', (error) => {
    logger.error('Uncaught Exception:', {
      message: error.message,
      stack: error.stack,
      name: error.name
    })
    
    captureException(error, { process: 'main', type: 'uncaughtException' })
    
    // แสดง dialog ให้ user
    dialog.showErrorBox(
      'เกิดข้อผิดพลาดที่ไม่คาดคิด',
      `${error.message}\n\nแอปอาจทำงานไม่ปกติ กรุณา restart`
    )
    
    // ไม่ exit เพื่อให้ app ยังทำงานได้
    // ยกเว้น fatal errors
    if (isFatalError(error)) {
      app.exit(1)
    }
  })

  // Unhandled promise rejections
  process.on('unhandledRejection', (reason, promise) => {
    const error = reason instanceof Error ? reason : new Error(String(reason))
    
    logger.error('Unhandled Rejection:', {
      message: error.message,
      stack: error.stack,
      promise: promise.toString()
    })
    
    captureException(error, { process: 'main', type: 'unhandledRejection' })
  })

  // Warning handler
  process.on('warning', (warning) => {
    logger.warn('Process Warning:', {
      name: warning.name,
      message: warning.message,
      stack: warning.stack
    })
  })
}

/**
 * ตรวจสอบว่าเป็น fatal error หรือไม่
 */
function isFatalError(error) {
  const fatalErrors = [
    'ENOENT',    // File not found
    'EACCES',    // Permission denied
    'ENOMEM',    // Out of memory
  ]
  
  return fatalErrors.some(code => error.message.includes(code))
}

/**
 * Error wrapper สำหรับ IPC handlers
 */
function withErrorHandling(handler) {
  return async (event, ...args) => {
    try {
      return await handler(event, ...args)
    } catch (error) {
      logger.error('IPC Handler Error:', {
        handler: handler.name,
        args,
        error: error.message,
        stack: error.stack
      })
      
      captureException(error, {
        ipcChannel: handler.name,
        args: JSON.stringify(args).substring(0, 1000)
      })
      
      throw error // Re-throw ให้ renderer จัดการ
    }
  }
}

module.exports = { setupGlobalErrorHandlers, withErrorHandling }
```

## Error Boundaries ใน Renderer (React)

```jsx
// src/components/ErrorBoundary.jsx
import { Component } from 'react'
import { reportError } from '../sentry'

class ErrorBoundary extends Component {
  constructor(props) {
    super(props)
    this.state = {
      hasError: false,
      error: null,
      errorInfo: null,
      errorId: null
    }
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error }
  }

  componentDidCatch(error, errorInfo) {
    // Log error
    console.error('React Error Boundary caught:', error, errorInfo)
    
    // ส่งไปยัง Sentry
    const errorId = this.generateErrorId()
    reportError(error, errorInfo)
    
    this.setState({
      errorInfo,
      errorId
    })
    
    // Log ไปยัง main process
    if (window.electronAPI?.logError) {
      window.electronAPI.logError({
        message: error.message,
        stack: error.stack,
        componentStack: errorInfo.componentStack,
        errorId
      })
    }
  }

  generateErrorId() {
    return `err-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }

  handleRetry = () => {
    this.setState({
      hasError: false,
      error: null,
      errorInfo: null
    })
  }

  handleReload = () => {
    window.location.reload()
  }

  handleReport = async () => {
    const { error, errorInfo, errorId } = this.state
    
    await window.electronAPI?.invoke('error:report', {
      errorId,
      message: error.message,
      stack: error.stack,
      componentStack: errorInfo?.componentStack
    })
    
    alert('ส่งรายงานข้อผิดพลาดแล้ว ขอบคุณ!')
  }

  render() {
    if (!this.state.hasError) {
      return this.props.children
    }

    const { error, errorId } = this.state
    const isDev = process.env.NODE_ENV === 'development'

    return (
      <div className="error-boundary">
        <div className="error-container">
          <div className="error-icon">💥</div>
          <h2>เกิดข้อผิดพลาด</h2>
          <p className="error-message">{error?.message}</p>
          
          {isDev && (
            <details className="error-details">
              <summary>รายละเอียดข้อผิดพลาด (Dev Mode)</summary>
              <pre>{error?.stack}</pre>
              {this.state.errorInfo && (
                <pre>{this.state.errorInfo.componentStack}</pre>
              )}
            </details>
          )}
          
          {errorId && (
            <p className="error-id">Error ID: {errorId}</p>
          )}
          
          <div className="error-actions">
            <button 
              onClick={this.handleRetry}
              className="btn-primary"
            >
              🔄 ลองใหม่
            </button>
            <button 
              onClick={this.handleReload}
              className="btn-secondary"
            >
              🔃 Reload
            </button>
            <button 
              onClick={this.handleReport}
              className="btn-secondary"
            >
              📤 ส่งรายงาน
            </button>
          </div>
        </div>
      </div>
    )
  }
}

// HOC สำหรับ wrap component ด้วย ErrorBoundary
export function withErrorBoundary(Component, fallback = null) {
  return function WrappedComponent(props) {
    return (
      <ErrorBoundary fallback={fallback}>
        <Component {...props} />
      </ErrorBoundary>
    )
  }
}

export default ErrorBoundary
```

## Renderer Process Error Handler

```javascript
// src/utils/errorHandler.js
import { captureException, captureMessage } from '../sentry'

/**
 * ตั้งค่า global error handlers ใน renderer
 */
export function setupRendererErrorHandlers() {
  // Window error handler
  window.addEventListener('error', (event) => {
    console.error('Window Error:', event.error)
    
    captureException(event.error, {
      type: 'window-error',
      source: event.filename,
      line: event.lineno,
      col: event.colno
    })
    
    // Log ไปยัง main process
    logToMain('error', {
      type: 'window-error',
      message: event.error?.message || event.message,
      stack: event.error?.stack,
      source: event.filename,
      line: event.lineno
    })
  })

  // Unhandled promise rejection
  window.addEventListener('unhandledrejection', (event) => {
    const error = event.reason instanceof Error 
      ? event.reason 
      : new Error(String(event.reason))
    
    console.error('Unhandled Rejection:', error)
    
    captureException(error, { type: 'unhandled-rejection' })
    
    logToMain('error', {
      type: 'unhandled-rejection',
      message: error.message,
      stack: error.stack
    })
  })
}

/**
 * ส่ง log ไปยัง main process
 */
function logToMain(level, data) {
  window.electronAPI?.invoke('logger:log', { level, ...data })
    .catch(console.error)
}

/**
 * Error class ที่ custom สำหรับ app errors
 */
export class AppError extends Error {
  constructor(message, code, context = {}) {
    super(message)
    this.name = 'AppError'
    this.code = code
    this.context = context
    this.timestamp = new Date().toISOString()
  }
}

export class NetworkError extends AppError {
  constructor(message, statusCode, url) {
    super(message, 'NETWORK_ERROR', { statusCode, url })
    this.name = 'NetworkError'
    this.statusCode = statusCode
    this.url = url
  }
}

export class ValidationError extends AppError {
  constructor(message, fields) {
    super(message, 'VALIDATION_ERROR', { fields })
    this.name = 'ValidationError'
    this.fields = fields
  }
}

/**
 * Async error handler wrapper
 */
export function handleAsync(fn) {
  return async (...args) => {
    try {
      return await fn(...args)
    } catch (error) {
      if (error instanceof AppError) {
        // Handle known errors
        console.error(`[${error.code}]`, error.message, error.context)
      } else {
        // Handle unexpected errors
        captureException(error)
        console.error('Unexpected error:', error)
      }
      throw error
    }
  }
}
```

## CSS สำหรับ Error UI

```css
/* src/components/ErrorBoundary.css */
.error-boundary {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  min-height: 300px;
  padding: 32px;
  background: var(--bg-primary);
}

.error-container {
  text-align: center;
  max-width: 500px;
}

.error-icon {
  font-size: 64px;
  margin-bottom: 16px;
  animation: shake 0.5s ease-in-out;
}

@keyframes shake {
  0%, 100% { transform: rotate(0deg); }
  25% { transform: rotate(-5deg); }
  75% { transform: rotate(5deg); }
}

.error-container h2 {
  font-size: 24px;
  color: var(--text-primary);
  margin-bottom: 8px;
}

.error-message {
  color: #ef4444;
  font-family: monospace;
  font-size: 14px;
  background: #fef2f2;
  padding: 12px;
  border-radius: 8px;
  margin: 12px 0;
  word-break: break-all;
}

.error-details {
  text-align: left;
  background: #1e1e1e;
  color: #d4d4d4;
  padding: 12px;
  border-radius: 8px;
  margin: 12px 0;
  font-family: monospace;
  font-size: 12px;
  overflow-x: auto;
}

.error-details pre {
  white-space: pre-wrap;
  word-break: break-all;
}

.error-id {
  font-size: 11px;
  color: var(--text-muted);
  font-family: monospace;
}

.error-actions {
  display: flex;
  gap: 8px;
  justify-content: center;
  margin-top: 20px;
  flex-wrap: wrap;
}
```

## Log Viewer Component

```jsx
// src/components/LogViewer.jsx
import { useState, useEffect, useRef } from 'react'

export default function LogViewer() {
  const [logs, setLogs] = useState([])
  const [filter, setFilter] = useState('all')
  const [search, setSearch] = useState('')
  const [autoScroll, setAutoScroll] = useState(true)
  const listRef = useRef(null)

  useEffect(() => {
    // โหลด logs จาก file
    loadLogs()
    
    // Listen for real-time logs
    if (window.electronAPI?.on) {
      const cleanup = window.electronAPI.on('logger:new-log', (log) => {
        setLogs(prev => [...prev.slice(-999), log]) // เก็บ 1000 บรรทัดล่าสุด
      })
      return cleanup
    }
  }, [])

  useEffect(() => {
    // Auto scroll ไปท้ายสุด
    if (autoScroll && listRef.current) {
      listRef.current.scrollTop = listRef.current.scrollHeight
    }
  }, [logs, autoScroll])

  const loadLogs = async () => {
    if (!window.electronAPI) return
    const logContent = await window.electronAPI.invoke('logger:getLogs')
    if (logContent) {
      const parsed = logContent.split('\n')
        .filter(Boolean)
        .map(line => {
          try {
            return JSON.parse(line)
          } catch {
            return { message: line, level: 'info', timestamp: '' }
          }
        })
      setLogs(parsed)
    }
  }

  const clearLogs = async () => {
    if (window.electronAPI) {
      await window.electronAPI.invoke('logger:clear')
      setLogs([])
    }
  }

  const filteredLogs = logs.filter(log => {
    if (filter !== 'all' && log.level !== filter) return false
    if (search && !log.message?.includes(search)) return false
    return true
  })

  const getLevelColor = (level) => {
    const colors = {
      error: '#ef4444',
      warn: '#f59e0b',
      info: '#3b82f6',
      debug: '#10b981',
      verbose: '#8b5cf6'
    }
    return colors[level] || '#6b7280'
  }

  return (
    <div className="log-viewer">
      <div className="log-toolbar">
        <h3>Application Logs</h3>
        <div className="log-controls">
          <select value={filter} onChange={e => setFilter(e.target.value)}>
            <option value="all">ทั้งหมด</option>
            <option value="error">Error</option>
            <option value="warn">Warning</option>
            <option value="info">Info</option>
            <option value="debug">Debug</option>
          </select>
          <input
            type="search"
            placeholder="ค้นหา..."
            value={search}
            onChange={e => setSearch(e.target.value)}
          />
          <label>
            <input
              type="checkbox"
              checked={autoScroll}
              onChange={e => setAutoScroll(e.target.checked)}
            />
            Auto Scroll
          </label>
          <button onClick={clearLogs} className="btn-danger-sm">
            ล้าง
          </button>
        </div>
      </div>
      
      <div className="log-list" ref={listRef}>
        {filteredLogs.length === 0 ? (
          <div className="log-empty">ไม่มี logs</div>
        ) : (
          filteredLogs.map((log, i) => (
            <div key={i} className="log-entry" style={{ borderLeftColor: getLevelColor(log.level) }}>
              <span className="log-time">{log.timestamp}</span>
              <span className="log-level" style={{ color: getLevelColor(log.level) }}>
                {log.level?.toUpperCase()}
              </span>
              <span className="log-message">{log.message}</span>
            </div>
          ))
        )}
      </div>
      
      <div className="log-stats">
        แสดง {filteredLogs.length} จาก {logs.length} รายการ
      </div>
    </div>
  )
}
```

## สรุป

### Error Handling Layers

1. **crashReporter** - OS-level crashes (C++ crashes)
2. **process.on('uncaughtException')** - Unhandled JS errors in main
3. **process.on('unhandledRejection')** - Unhandled Promise rejections
4. **window.addEventListener('error')** - JS errors in renderer
5. **React Error Boundary** - React component errors
6. **try/catch** - Expected errors ที่จัดการเอง

### Best Practices

- Log ทุก error ด้วย context ที่เพียงพอสำหรับ debugging
- ส่ง errors ไปยัง Sentry ใน production
- แสดง error ที่เข้าใจได้สำหรับ user
- อย่า show sensitive data ใน error messages
- มี fallback UI สำหรับทุก component
