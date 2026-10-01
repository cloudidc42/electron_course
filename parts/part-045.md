# Part 045: Custom Protocol Deep Dive
## การสร้าง Custom Protocol ใน Electron

---

## 🎯 เป้าหมายของบทเรียนนี้

- สร้าง custom app:// scheme
- Serve local files อย่างปลอดภัย
- protocol.registerFileProtocol
- interceptHTTPRequest
- Service Worker replacement
- Offline support

---

## 1. ทำไมต้องใช้ Custom Protocol?

```
ปัญหา: ถ้าใช้ file:// protocol
- ไม่มี origin → CORS ปัญหา
- Security restrictions
- Cannot use localStorage, IndexedDB properly
- Service Workers ไม่ทำงาน

แก้ไข: ใช้ custom protocol เช่น app://
- มี origin ชัดเจน (app://localhost)
- ผ่าน CORS ได้
- Security ดีกว่า
- Service Workers ทำงานได้
```

---

## 2. ลงทะเบียน Custom Protocol

### src/main/protocol.ts

```typescript
import { protocol, app, net } from 'electron';
import { join, normalize, extname } from 'path';
import { pathToFileURL } from 'url';
import { existsSync, statSync } from 'fs';

const APP_SCHEME = 'app';
const STATIC_SCHEME = 'static';

// Mime types
const MIME_TYPES: Record<string, string> = {
  '.html': 'text/html',
  '.js': 'application/javascript',
  '.mjs': 'application/javascript',
  '.css': 'text/css',
  '.json': 'application/json',
  '.png': 'image/png',
  '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg',
  '.gif': 'image/gif',
  '.svg': 'image/svg+xml',
  '.webp': 'image/webp',
  '.ico': 'image/x-icon',
  '.woff': 'font/woff',
  '.woff2': 'font/woff2',
  '.ttf': 'font/ttf',
  '.eot': 'application/vnd.ms-fontobject',
  '.mp4': 'video/mp4',
  '.webm': 'video/webm',
  '.mp3': 'audio/mpeg',
  '.wav': 'audio/wav',
  '.pdf': 'application/pdf',
  '.txt': 'text/plain',
  '.md': 'text/markdown',
};

function getMimeType(filePath: string): string {
  const ext = extname(filePath).toLowerCase();
  return MIME_TYPES[ext] || 'application/octet-stream';
}

// ==========================================
// Scheme 1: app:// - สำหรับ serve app files
// ==========================================

export function setupAppProtocol(): void {
  // ต้อง privilege ก่อน app.whenReady()
  protocol.registerSchemesAsPrivileged([
    {
      scheme: APP_SCHEME,
      privileges: {
        secure: true,           // HTTPS-like
        standard: true,         // Standard URL parsing
        supportFetchAPI: true,  // Fetch API support
        allowServiceWorkers: true,
        corsEnabled: true,
        stream: true,
      },
    },
    {
      scheme: STATIC_SCHEME,
      privileges: {
        secure: true,
        standard: true,
        supportFetchAPI: true,
        corsEnabled: true,
      },
    },
  ]);
}

// Register handlers หลัง app.whenReady()
export function registerProtocolHandlers(session: Electron.Session): void {
  // ==========================================
  // app:// protocol handler
  // ==========================================
  session.protocol.handle(APP_SCHEME, async (request) => {
    const url = new URL(request.url);
    const pathname = decodeURIComponent(url.pathname);
    
    // Root directory ของแอพ
    const appRoot = app.isPackaged
      ? join(process.resourcesPath, 'app')
      : join(__dirname, '../../');
    
    // Normalize path เพื่อป้องกัน path traversal
    let filePath = normalize(join(appRoot, 'dist/renderer', pathname));
    
    // ตรวจสอบว่า path อยู่ใน appRoot
    if (!filePath.startsWith(normalize(appRoot))) {
      return new Response('Forbidden', { status: 403 });
    }
    
    // ถ้า path เป็น directory → ลอง index.html
    if (existsSync(filePath) && statSync(filePath).isDirectory()) {
      filePath = join(filePath, 'index.html');
    }
    
    // SPA fallback → index.html
    if (!existsSync(filePath)) {
      filePath = join(appRoot, 'dist/renderer', 'index.html');
    }
    
    if (!existsSync(filePath)) {
      return new Response('Not Found', { status: 404 });
    }
    
    try {
      return net.fetch(pathToFileURL(filePath).toString());
    } catch (error) {
      console.error('Protocol handler error:', error);
      return new Response('Internal Server Error', { status: 500 });
    }
  });
  
  // ==========================================
  // static:// protocol - สำหรับ user files
  // ==========================================
  session.protocol.handle(STATIC_SCHEME, async (request) => {
    const url = new URL(request.url);
    const requestedPath = decodeURIComponent(url.pathname);
    
    // Allowed directories สำหรับ serve
    const allowedPaths = [
      app.getPath('userData'),
      app.getPath('documents'),
      app.getPath('temp'),
    ];
    
    // Resolve path
    const filePath = normalize(requestedPath);
    
    // Security check: อยู่ใน allowed paths หรือไม่
    const isAllowed = allowedPaths.some(allowed => 
      filePath.startsWith(normalize(allowed))
    );
    
    if (!isAllowed) {
      return new Response('Forbidden', {
        status: 403,
        headers: { 'Content-Type': 'text/plain' },
      });
    }
    
    if (!existsSync(filePath)) {
      return new Response('Not Found', { status: 404 });
    }
    
    const stat = statSync(filePath);
    if (!stat.isFile()) {
      return new Response('Not a file', { status: 400 });
    }
    
    const mimeType = getMimeType(filePath);
    
    // Range request support (สำหรับ video streaming)
    const rangeHeader = request.headers.get('Range');
    if (rangeHeader) {
      return handleRangeRequest(filePath, rangeHeader, mimeType, stat.size);
    }
    
    return net.fetch(pathToFileURL(filePath).toString(), {
      headers: {
        'Content-Type': mimeType,
        'Cache-Control': 'no-cache',
      },
    });
  });
}

// Range request handler สำหรับ video streaming
function handleRangeRequest(
  filePath: string,
  rangeHeader: string,
  mimeType: string,
  fileSize: number
): Response {
  const match = rangeHeader.match(/bytes=(\d+)-(\d*)/);
  
  if (!match) {
    return new Response('Invalid Range', { status: 416 });
  }
  
  const start = parseInt(match[1], 10);
  const end = match[2] ? parseInt(match[2], 10) : fileSize - 1;
  const chunkSize = end - start + 1;
  
  const { createReadStream } = require('fs');
  const stream = createReadStream(filePath, { start, end });
  
  return new Response(stream as any, {
    status: 206,
    headers: {
      'Content-Range': `bytes ${start}-${end}/${fileSize}`,
      'Accept-Ranges': 'bytes',
      'Content-Length': chunkSize.toString(),
      'Content-Type': mimeType,
    },
  });
}
```

---

## 3. ตั้งค่าใน Main Process

### src/main/index.ts

```typescript
import { app, BrowserWindow, session } from 'electron';
import { setupAppProtocol, registerProtocolHandlers } from './protocol';

// ต้อง setup BEFORE app.whenReady()
setupAppProtocol();

app.whenReady().then(() => {
  // Register protocol handlers
  registerProtocolHandlers(session.defaultSession);
  
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      contextIsolation: true,
      nodeIntegration: false,
    },
  });
  
  // โหลด app ผ่าน custom protocol
  win.loadURL('app://localhost/');
});
```

---

## 4. Intercept HTTP Requests

### src/main/request-interceptor.ts

```typescript
import { session, net } from 'electron';

export function setupRequestInterceptor(): void {
  const ses = session.defaultSession;
  
  // ==========================================
  // Intercept API requests - add auth headers
  // ==========================================
  ses.webRequest.onBeforeSendHeaders(
    { urls: ['https://api.yourapp.com/*'] },
    (details, callback) => {
      const requestHeaders = {
        ...details.requestHeaders,
        Authorization: `Bearer ${getAuthToken()}`,
        'X-App-Version': app.getVersion(),
        'X-Platform': process.platform,
      };
      
      callback({ requestHeaders });
    }
  );
  
  // ==========================================
  // Block unwanted requests
  // ==========================================
  ses.webRequest.onBeforeRequest(
    { urls: ['*://ads.example.com/*', '*://tracking.example.com/*'] },
    (details, callback) => {
      callback({ cancel: true });
    }
  );
  
  // ==========================================
  // Log response headers
  // ==========================================
  ses.webRequest.onHeadersReceived(
    { urls: ['*://*/*'] },
    (details, callback) => {
      const responseHeaders = {
        ...details.responseHeaders,
        // เพิ่ม security headers
        'X-Frame-Options': ['DENY'],
        'X-Content-Type-Options': ['nosniff'],
        'Referrer-Policy': ['strict-origin-when-cross-origin'],
      };
      
      callback({ responseHeaders });
    }
  );
  
  // ==========================================
  // Redirect requests
  // ==========================================
  ses.webRequest.onBeforeRequest(
    { urls: ['http://api.yourapp.com/*'] },
    (details, callback) => {
      // Redirect HTTP to HTTPS
      const redirectURL = details.url.replace('http://', 'https://');
      callback({ redirectURL });
    }
  );
}

function getAuthToken(): string {
  // ดึง token จาก storage
  return process.env.AUTH_TOKEN || '';
}
```

---

## 5. Content Security Policy (CSP)

```typescript
// src/main/security.ts
import { session } from 'electron';

export function setupCSP(): void {
  session.defaultSession.webRequest.onHeadersReceived(
    (details, callback) => {
      // กำหนด CSP สำหรับ app:// pages
      if (details.url.startsWith('app://')) {
        callback({
          responseHeaders: {
            ...details.responseHeaders,
            'Content-Security-Policy': [
              [
                "default-src 'self' app:",
                "script-src 'self' 'unsafe-inline'", // unsafe-inline สำหรับ dev
                "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
                "font-src 'self' https://fonts.gstatic.com",
                "img-src 'self' data: blob: app: static:",
                "media-src 'self' blob: static:",
                "connect-src 'self' https://api.yourapp.com wss://ws.yourapp.com",
                "worker-src 'self' blob:",
                "frame-src 'none'",
                "object-src 'none'",
                "base-uri 'self'",
                "form-action 'self'",
              ].join('; '),
            ],
          },
        });
      } else {
        callback({});
      }
    }
  );
}
```

---

## 6. Offline Support

### src/main/offline.ts

```typescript
import { session, net } from 'electron';
import { join } from 'path';
import { app } from 'electron';
import * as fs from 'fs';

const CACHE_DIR = join(app.getPath('userData'), 'cache');

// สร้าง cache directory
if (!fs.existsSync(CACHE_DIR)) {
  fs.mkdirSync(CACHE_DIR, { recursive: true });
}

// Simple cache implementation
class RequestCache {
  private cacheDir: string;
  
  constructor(cacheDir: string) {
    this.cacheDir = cacheDir;
  }
  
  getCachePath(url: string): string {
    const hash = require('crypto')
      .createHash('sha256')
      .update(url)
      .digest('hex');
    return join(this.cacheDir, hash);
  }
  
  async get(url: string): Promise<Buffer | null> {
    const cachePath = this.getCachePath(url);
    try {
      return fs.readFileSync(cachePath);
    } catch {
      return null;
    }
  }
  
  async set(url: string, data: Buffer): Promise<void> {
    const cachePath = this.getCachePath(url);
    fs.writeFileSync(cachePath, data);
  }
  
  has(url: string): boolean {
    const cachePath = this.getCachePath(url);
    return fs.existsSync(cachePath);
  }
}

const cache = new RequestCache(CACHE_DIR);

export function setupOfflineSupport(): void {
  const ses = session.defaultSession;
  
  // Intercept API requests และ cache responses
  ses.protocol.handle('https', async (request) => {
    const url = request.url;
    
    // ตรวจสอบ network
    if (!net.isOnline()) {
      // Try cache
      const cached = await cache.get(url);
      if (cached) {
        console.log(`Serving from cache: ${url}`);
        return new Response(cached, {
          headers: {
            'X-Cache': 'HIT',
            'X-Offline': 'true',
          },
        });
      }
      
      // ไม่มี cache
      return new Response('Offline', {
        status: 503,
        headers: { 'Content-Type': 'text/plain' },
      });
    }
    
    // Online: fetch normally
    try {
      const response = await net.fetch(url, {
        method: request.method,
        headers: Object.fromEntries(request.headers),
        body: request.body,
      });
      
      // Cache GET requests
      if (request.method === 'GET' && response.ok) {
        const data = Buffer.from(await response.clone().arrayBuffer());
        await cache.set(url, data);
      }
      
      return response;
    } catch (error) {
      // Network error → try cache
      const cached = await cache.get(url);
      if (cached) {
        return new Response(cached, {
          headers: { 'X-Cache': 'HIT', 'X-Offline': 'true' },
        });
      }
      
      throw error;
    }
  });
}
```

---

## 7. ใช้งานจาก Renderer

### src/renderer/api.ts

```typescript
// ใช้ static:// protocol เพื่อโหลดไฟล์ของ user
async function loadUserFile(filePath: string): Promise<string> {
  const response = await fetch(`static://${filePath}`);
  
  if (!response.ok) {
    throw new Error(`Failed to load file: ${response.status}`);
  }
  
  return response.text();
}

// โหลด video ผ่าน static:// (รองรับ range requests)
function createVideoUrl(filePath: string): string {
  return `static://${filePath}`;
}

// ตัวอย่างการใช้งาน
const content = await loadUserFile('/Users/john/Documents/note.txt');
const videoSrc = createVideoUrl('/Users/john/Videos/movie.mp4');

// ใน HTML
// <video src="static:///Users/john/Videos/movie.mp4" controls></video>
```

---

## 8. สรุป

| Protocol | การใช้งาน | ข้อดี |
|----------|-----------|-------|
| `app://` | Serve app files | เหมือน HTTPS origin |
| `static://` | Serve user files | รองรับ Range requests |
| `file://` | Built-in | ง่ายแต่ไม่แนะนำ |

### Best Practices

1. **Path validation** - ตรวจสอบ path traversal เสมอ
2. **Allowed paths** - จำกัด directories ที่ serve ได้
3. **CSP** - ตั้งค่า Content Security Policy
4. **Range requests** - รองรับ video streaming
5. **Caching** - Cache สำหรับ offline support

---

*จบ Part 045 - ต่อไป Part 046: WebRTC Integration*
