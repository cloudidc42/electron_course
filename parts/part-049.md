# Part 049: Monitoring & Logging
## การติดตามและบันทึก Log ใน Electron

---

## 🎯 เป้าหมายของบทเรียนนี้

- Winston logger
- Log rotation
- Structured logging
- Sentry error tracking
- Analytics ด้วย Mixpanel
- Performance metrics
- Custom telemetry

---

## 1. ติดตั้ง Dependencies

```bash
npm install winston winston-daily-rotate-file
npm install @sentry/electron
npm install electron-log
npm install mixpanel
```

---

## 2. Winston Logger Setup

### src/main/logger.ts

```typescript
import * as winston from 'winston';
import * as DailyRotateFile from 'winston-daily-rotate-file';
import { join } from 'path';
import { app } from 'electron';

const LOG_DIR = join(app.getPath('userData'), 'logs');

// Custom format สำหรับ structured logging
const structuredFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss.SSS' }),
  winston.format.errors({ stack: true }),
  winston.format.metadata({ fillExcept: ['message', 'level', 'timestamp', 'label'] }),
  winston.format.json()
);

// Console format (human readable)
const consoleFormat = winston.format.combine(
  winston.format.colorize(),
  winston.format.timestamp({ format: 'HH:mm:ss' }),
  winston.format.printf(({ timestamp, level, message, ...meta }) => {
    const metaStr = Object.keys(meta).length > 0 
      ? ` ${JSON.stringify(meta)}` 
      : '';
    return `[${timestamp}] ${level}: ${message}${metaStr}`;
  })
);

// สร้าง transports
const transports: winston.transport[] = [
  // Console (development only)
  ...(process.env.NODE_ENV !== 'production' ? [
    new winston.transports.Console({
      format: consoleFormat,
      level: 'debug',
    }),
  ] : []),
  
  // Error log (ทุก errors)
  new DailyRotateFile({
    filename: join(LOG_DIR, 'error-%DATE%.log'),
    datePattern: 'YYYY-MM-DD',
    level: 'error',
    format: structuredFormat,
    maxSize: '10m',
    maxFiles: '14d',  // เก็บ 14 วัน
    zippedArchive: true,
  }),
  
  // Combined log (ทุก levels)
  new DailyRotateFile({
    filename: join(LOG_DIR, 'app-%DATE%.log'),
    datePattern: 'YYYY-MM-DD',
    level: 'info',
    format: structuredFormat,
    maxSize: '20m',
    maxFiles: '7d',
    zippedArchive: true,
  }),
];

// สร้าง logger
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  defaultMeta: {
    app: app.getName(),
    version: app.getVersion(),
    platform: process.platform,
    arch: process.arch,
  },
  transports,
  
  // Handle exceptions
  exceptionHandlers: [
    new DailyRotateFile({
      filename: join(LOG_DIR, 'exceptions-%DATE%.log'),
      datePattern: 'YYYY-MM-DD',
      format: structuredFormat,
      maxFiles: '30d',
    }),
  ],
  
  // Handle unhandled promise rejections
  rejectionHandlers: [
    new DailyRotateFile({
      filename: join(LOG_DIR, 'rejections-%DATE%.log'),
      datePattern: 'YYYY-MM-DD',
      format: structuredFormat,
      maxFiles: '30d',
    }),
  ],
});

// ==========================================
// Typed Logger interface
// ==========================================

export interface LogContext {
  userId?: string;
  sessionId?: string;
  action?: string;
  component?: string;
  duration?: number;
  error?: Error;
  [key: string]: any;
}

export class AppLogger {
  private context: LogContext;
  
  constructor(context: LogContext = {}) {
    this.context = context;
  }
  
  // สร้าง child logger พร้อม context
  child(context: LogContext): AppLogger {
    return new AppLogger({ ...this.context, ...context });
  }
  
  debug(message: string, context?: LogContext): void {
    logger.debug(message, { ...this.context, ...context });
  }
  
  info(message: string, context?: LogContext): void {
    logger.info(message, { ...this.context, ...context });
  }
  
  warn(message: string, context?: LogContext): void {
    logger.warn(message, { ...this.context, ...context });
  }
  
  error(message: string, error?: Error | LogContext, context?: LogContext): void {
    if (error instanceof Error) {
      logger.error(message, {
        ...this.context,
        ...context,
        error: {
          message: error.message,
          stack: error.stack,
          name: error.name,
        },
      });
    } else {
      logger.error(message, { ...this.context, ...error });
    }
  }
  
  // Performance timing
  time(label: string): () => void {
    const start = Date.now();
    return () => {
      const duration = Date.now() - start;
      this.info(`${label} completed`, { duration, label });
    };
  }
  
  // Log IPC calls
  ipc(channel: string, direction: 'in' | 'out', data?: any): void {
    this.debug(`IPC ${direction}: ${channel}`, {
      channel,
      direction,
      hasData: !!data,
    });
  }
}

export const log = new AppLogger({ component: 'main' });

// Export raw winston logger สำหรับ special cases
export { logger as rawLogger };
```

---

## 3. Log Viewer ใน App

### src/main/log-viewer.ts

```typescript
import { ipcMain, shell } from 'electron';
import { join } from 'path';
import { app } from 'electron';
import * as fs from 'fs';

const LOG_DIR = join(app.getPath('userData'), 'logs');

ipcMain.handle('logs:getFiles', async () => {
  try {
    const files = fs.readdirSync(LOG_DIR)
      .filter(f => f.endsWith('.log'))
      .map(name => ({
        name,
        path: join(LOG_DIR, name),
        size: fs.statSync(join(LOG_DIR, name)).size,
        modified: fs.statSync(join(LOG_DIR, name)).mtime.toISOString(),
      }))
      .sort((a, b) => b.modified.localeCompare(a.modified));
    
    return { success: true, files };
  } catch (error: any) {
    return { success: false, error: error.message };
  }
});

ipcMain.handle('logs:read', async (_, filename: string) => {
  const filePath = join(LOG_DIR, filename);
  
  // Security check
  if (!filePath.startsWith(LOG_DIR)) {
    return { success: false, error: 'Access denied' };
  }
  
  try {
    const content = fs.readFileSync(filePath, 'utf-8');
    const lines = content.trim().split('\n');
    
    // Parse JSON logs
    const entries = lines.map(line => {
      try {
        return JSON.parse(line);
      } catch {
        return { raw: line };
      }
    });
    
    return { success: true, entries };
  } catch (error: any) {
    return { success: false, error: error.message };
  }
});

ipcMain.handle('logs:openFolder', async () => {
  await shell.openPath(LOG_DIR);
});

ipcMain.handle('logs:clear', async () => {
  try {
    const files = fs.readdirSync(LOG_DIR).filter(f => f.endsWith('.log'));
    files.forEach(f => fs.unlinkSync(join(LOG_DIR, f)));
    return { success: true, count: files.length };
  } catch (error: any) {
    return { success: false, error: error.message };
  }
});
```

---

## 4. Sentry Error Tracking

### src/main/sentry.ts

```typescript
import * as Sentry from '@sentry/electron/main';
import { app } from 'electron';

export function initSentry(): void {
  Sentry.init({
    dsn: process.env.SENTRY_DSN || 'https://your-sentry-dsn@sentry.io/project-id',
    
    environment: process.env.NODE_ENV || 'development',
    
    release: `my-app@${app.getVersion()}`,
    
    // Sample rate (0.0 - 1.0)
    tracesSampleRate: 0.1,
    
    beforeSend(event, hint) {
      // ไม่ส่ง errors ใน development
      if (process.env.NODE_ENV === 'development') {
        console.error('Sentry event (dev):', event);
        return null; // ไม่ส่ง
      }
      
      // ลบ sensitive data
      if (event.request) {
        delete event.request.cookies;
        delete event.request.headers?.authorization;
      }
      
      return event;
    },
    
    // Context
    initialScope: {
      tags: {
        platform: process.platform,
        arch: process.arch,
        electron: process.versions.electron,
        node: process.versions.node,
      },
      user: {
        // ตั้งค่า user ID ถ้ามี
      },
    },
  });
}

// Sentry ใน renderer
// src/renderer/sentry.ts
export function initRendererSentry(): void {
  // ใน renderer ใช้ @sentry/electron/renderer
  const { init } = require('@sentry/electron/renderer');
  
  init({
    dsn: process.env.VITE_SENTRY_DSN,
    environment: import.meta.env.MODE,
    integrations: [
      // Browser tracing
    ],
    tracesSampleRate: 0.1,
  });
}

// Custom error capturing
export function captureError(error: Error, context?: Record<string, any>): void {
  Sentry.withScope(scope => {
    if (context) {
      scope.setExtras(context);
    }
    Sentry.captureException(error);
  });
}

// Track performance
export function trackPerformance(
  name: string, 
  operation: string,
  callback: () => Promise<any>
): Promise<any> {
  return Sentry.startSpan({ name, op: operation }, callback);
}
```

---

## 5. Analytics ด้วย Mixpanel

### src/main/analytics.ts

```typescript
import Mixpanel from 'mixpanel';
import { app } from 'electron';
import { randomUUID } from 'crypto';

class Analytics {
  private client: ReturnType<typeof Mixpanel.init> | null = null;
  private distinctId: string;
  private enabled: boolean = true;
  
  constructor() {
    this.distinctId = this.getOrCreateUserId();
  }
  
  init(token: string): void {
    if (!token) {
      console.warn('Analytics token not provided');
      return;
    }
    
    this.client = Mixpanel.init(token, {
      protocol: 'https',
    });
    
    // Set super properties (ส่งทุก event)
    this.client.people.set(this.distinctId, {
      '$name': `User ${this.distinctId.slice(0, 8)}`,
      'platform': process.platform,
      'app_version': app.getVersion(),
      'electron_version': process.versions.electron,
    });
  }
  
  // Track event
  track(event: string, properties?: Record<string, any>): void {
    if (!this.client || !this.enabled) return;
    
    this.client.track(event, {
      distinct_id: this.distinctId,
      app_version: app.getVersion(),
      platform: process.platform,
      timestamp: new Date().toISOString(),
      ...properties,
    });
  }
  
  // Track screen view
  trackScreen(screenName: string): void {
    this.track('Screen Viewed', { screen: screenName });
  }
  
  // Track feature usage
  trackFeature(feature: string, action: string): void {
    this.track('Feature Used', { feature, action });
  }
  
  // Track error (non-critical)
  trackError(errorName: string, errorMessage: string): void {
    this.track('Error Occurred', {
      error_name: errorName,
      error_message: errorMessage.slice(0, 200), // จำกัดความยาว
    });
  }
  
  // Track performance
  trackPerformance(operation: string, durationMs: number): void {
    this.track('Performance', { operation, duration_ms: durationMs });
  }
  
  // Enable/disable tracking
  setEnabled(enabled: boolean): void {
    this.enabled = enabled;
  }
  
  // Opt out
  optOut(): void {
    this.enabled = false;
    // ลบ user data
    this.client?.people.deleteUser(this.distinctId);
  }
  
  private getOrCreateUserId(): string {
    const Store = require('electron-store');
    const store = new Store({ name: 'analytics' });
    
    let userId = store.get('userId') as string;
    if (!userId) {
      userId = randomUUID();
      store.set('userId', userId);
    }
    
    return userId;
  }
}

export const analytics = new Analytics();
```

---

## 6. Performance Monitoring

### src/main/performance.ts

```typescript
import { app, powerMonitor } from 'electron';
import { log } from './logger';

class PerformanceMonitor {
  private metrics: Map<string, number[]> = new Map();
  private intervalId: NodeJS.Timeout | null = null;
  
  start(): void {
    // ติดตาม memory ทุก 30 วินาที
    this.intervalId = setInterval(() => {
      this.collectMemoryMetrics();
    }, 30000);
    
    // ติดตาม system events
    this.setupSystemMonitoring();
    
    // ติดตาม app events
    this.setupAppMonitoring();
  }
  
  stop(): void {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
  
  private collectMemoryMetrics(): void {
    const memInfo = process.getProcessMemoryInfo();
    
    log.info('Memory metrics', {
      type: 'memory',
      rss: memInfo.residentSet,
      heapTotal: process.memoryUsage().heapTotal,
      heapUsed: process.memoryUsage().heapUsed,
      external: process.memoryUsage().external,
    });
    
    // เช็ค memory leak
    const heapUsed = process.memoryUsage().heapUsed;
    this.addMetric('heapUsed', heapUsed);
    
    const recentHeap = this.getRecentMetrics('heapUsed', 10);
    if (recentHeap.length >= 10) {
      const trend = recentHeap[recentHeap.length - 1] - recentHeap[0];
      if (trend > 100 * 1024 * 1024) { // 100MB ใน 5 นาที
        log.warn('Possible memory leak detected', { trend });
      }
    }
  }
  
  private setupSystemMonitoring(): void {
    powerMonitor.on('suspend', () => {
      log.info('System suspended');
    });
    
    powerMonitor.on('resume', () => {
      log.info('System resumed');
    });
    
    powerMonitor.on('lock-screen', () => {
      log.info('Screen locked');
    });
  }
  
  private setupAppMonitoring(): void {
    // Track render errors
    app.on('render-process-gone', (event, webContents, details) => {
      log.error('Renderer process crashed', {
        reason: details.reason,
        exitCode: details.exitCode,
        url: webContents.getURL(),
      });
    });
    
    // Track GPU crashes
    app.on('gpu-process-crashed', (event, killed) => {
      log.error('GPU process crashed', { killed });
    });
  }
  
  // Measure function execution time
  async measure<T>(name: string, fn: () => Promise<T>): Promise<T> {
    const start = performance.now();
    
    try {
      const result = await fn();
      const duration = performance.now() - start;
      
      this.addMetric(name, duration);
      
      log.info(`${name} completed`, {
        type: 'performance',
        name,
        duration: Math.round(duration),
      });
      
      return result;
    } catch (error) {
      const duration = performance.now() - start;
      log.error(`${name} failed`, error as Error, { duration: Math.round(duration) });
      throw error;
    }
  }
  
  private addMetric(name: string, value: number): void {
    const values = this.metrics.get(name) || [];
    values.push(value);
    
    // เก็บแค่ 100 ค่าล่าสุด
    if (values.length > 100) {
      values.shift();
    }
    
    this.metrics.set(name, values);
  }
  
  private getRecentMetrics(name: string, count: number): number[] {
    const values = this.metrics.get(name) || [];
    return values.slice(-count);
  }
  
  // ดึง summary
  getSummary(): Record<string, any> {
    const summary: Record<string, any> = {};
    
    this.metrics.forEach((values, name) => {
      if (values.length === 0) return;
      
      const sorted = [...values].sort((a, b) => a - b);
      summary[name] = {
        count: values.length,
        min: sorted[0],
        max: sorted[sorted.length - 1],
        avg: values.reduce((a, b) => a + b, 0) / values.length,
        p50: sorted[Math.floor(sorted.length * 0.5)],
        p90: sorted[Math.floor(sorted.length * 0.9)],
        p99: sorted[Math.floor(sorted.length * 0.99)],
      };
    });
    
    return summary;
  }
}

export const perfMonitor = new PerformanceMonitor();
```

---

## 7. electron-log (Simple Solution)

```typescript
// ใช้ electron-log สำหรับ simple logging
import log from 'electron-log';

// ตั้งค่า
log.transports.file.level = 'info';
log.transports.file.maxSize = 10 * 1024 * 1024; // 10MB
log.transports.file.fileName = 'main.log';

// Override console
log.transports.console.level = 'debug';

// ใช้งาน
log.info('App started', { version: app.getVersion() });
log.warn('Configuration missing');
log.error('Failed to load file', new Error('ENOENT'));

// Renderer ใช้ได้เหมือนกัน (ผ่าน IPC อัตโนมัติ)
```

---

## 8. สรุป

| Tool | ใช้สำหรับ | ข้อดี |
|------|-----------|-------|
| Winston | Structured logging | ยืดหยุ่น, หลาย transports |
| electron-log | Simple logging | ง่าย, built-in renderer support |
| Sentry | Error tracking | Dashboard, alert, replay |
| Mixpanel | Analytics | Funnel, cohort analysis |
| Custom metrics | Performance | Full control |

### Checklist

- [ ] ตั้งค่า log rotation
- [ ] ไม่ log sensitive data
- [ ] ตั้งค่า Sentry DSN
- [ ] ขอ user consent สำหรับ analytics
- [ ] Test log output ใน production build

---

*จบ Part 049 - ต่อไป Part 050: Offline-First Architecture*
