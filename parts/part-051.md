# Part 051: Advanced IPC Patterns
## Pattern IPC ขั้นสูงสำหรับ Electron

---

## 🎯 เป้าหมายของบทเรียนนี้

- Streaming ผ่าน IPC
- Binary data transfer
- IPC middleware
- Request deduplication
- IPC timeout
- Rate limiting
- IPC bridge pattern

---

## 1. Streaming ผ่าน IPC

### src/main/streaming-ipc.ts

```typescript
import { ipcMain, BrowserWindow } from 'electron';
import * as fs from 'fs';

// ==========================================
// File Streaming
// ==========================================

ipcMain.handle('stream:readFile', async (event, filePath: string) => {
  const win = BrowserWindow.fromWebContents(event.sender);
  const streamId = `stream-${Date.now()}-${Math.random().toString(36).slice(2)}`;
  
  // เริ่ม stream
  win!.webContents.send('stream:start', { streamId, filePath });
  
  return new Promise<void>((resolve, reject) => {
    const stream = fs.createReadStream(filePath, {
      highWaterMark: 64 * 1024, // 64KB chunks
    });
    
    let bytesRead = 0;
    const fileSize = fs.statSync(filePath).size;
    
    stream.on('data', (chunk: Buffer) => {
      bytesRead += chunk.length;
      const progress = (bytesRead / fileSize) * 100;
      
      // ส่ง chunk ไปยัง renderer
      win!.webContents.send('stream:chunk', {
        streamId,
        data: chunk.toString('base64'),
        bytesRead,
        total: fileSize,
        progress: Math.round(progress),
      });
    });
    
    stream.on('end', () => {
      win!.webContents.send('stream:end', { streamId });
      resolve();
    });
    
    stream.on('error', (error) => {
      win!.webContents.send('stream:error', { streamId, error: error.message });
      reject(error);
    });
  });
});

// ==========================================
// Async Generator Streaming
// ==========================================

async function* generateData(count: number): AsyncGenerator<{ index: number; value: number }> {
  for (let i = 0; i < count; i++) {
    yield { index: i, value: Math.random() };
    // Simulate async work
    await new Promise(resolve => setTimeout(resolve, 10));
  }
}

ipcMain.handle('stream:generate', async (event, count: number) => {
  const win = BrowserWindow.fromWebContents(event.sender);
  const streamId = `gen-${Date.now()}`;
  
  (async () => {
    try {
      for await (const item of generateData(count)) {
        win!.webContents.send('stream:item', { streamId, item });
      }
      win!.webContents.send('stream:done', { streamId });
    } catch (error: any) {
      win!.webContents.send('stream:error', { streamId, error: error.message });
    }
  })();
  
  return streamId;
});
```

### src/renderer/streaming-client.ts

```typescript
class StreamingClient {
  private streams: Map<string, {
    onChunk?: (data: any) => void;
    onEnd?: () => void;
    onError?: (error: string) => void;
    buffer: any[];
  }> = new Map();
  
  constructor() {
    this.setupListeners();
  }
  
  private setupListeners(): void {
    window.electronAPI.on('stream:start', ({ streamId }: any) => {
      this.streams.set(streamId, { buffer: [] });
    });
    
    window.electronAPI.on('stream:chunk', ({ streamId, data, progress }: any) => {
      const stream = this.streams.get(streamId);
      if (!stream) return;
      
      const chunk = atob(data); // decode base64
      stream.buffer.push(chunk);
      stream.onChunk?.({ chunk, progress });
    });
    
    window.electronAPI.on('stream:end', ({ streamId }: any) => {
      const stream = this.streams.get(streamId);
      if (!stream) return;
      
      stream.onEnd?.();
      this.streams.delete(streamId);
    });
    
    window.electronAPI.on('stream:error', ({ streamId, error }: any) => {
      const stream = this.streams.get(streamId);
      if (!stream) return;
      
      stream.onError?.(error);
      this.streams.delete(streamId);
    });
  }
  
  async readFile(filePath: string, callbacks: {
    onChunk?: (data: { chunk: string; progress: number }) => void;
    onEnd?: () => void;
    onError?: (error: string) => void;
  }): Promise<void> {
    // เริ่ม stream
    const streamIdPromise = window.electronAPI.invoke('stream:readFile', filePath);
    
    // ตั้งค่า callbacks ก่อน (อาจได้รับ event ก่อน invoke return)
    streamIdPromise.then(() => {
      // Callbacks จะถูกเซตโดย stream:start event
    });
    
    // Set up callbacks
    window.electronAPI.on('stream:start', ({ streamId }: any) => {
      const stream = this.streams.get(streamId);
      if (stream) {
        stream.onChunk = callbacks.onChunk;
        stream.onEnd = callbacks.onEnd;
        stream.onError = callbacks.onError;
      }
    });
    
    return streamIdPromise;
  }
}

export const streamingClient = new StreamingClient();
```

---

## 2. Binary Data Transfer

### src/main/binary-ipc.ts

```typescript
import { ipcMain } from 'electron';
import * as sharp from 'sharp';

// ==========================================
// Transfer image data as ArrayBuffer
// ==========================================

ipcMain.handle('binary:processImage', async (event, imageBuffer: ArrayBuffer) => {
  // Convert ArrayBuffer to Buffer
  const buffer = Buffer.from(imageBuffer);
  
  // Process image ด้วย sharp
  const processedBuffer = await sharp(buffer)
    .resize(800, 600, { fit: 'inside' })
    .webp({ quality: 80 })
    .toBuffer();
  
  // Return เป็น ArrayBuffer
  return processedBuffer.buffer.slice(
    processedBuffer.byteOffset,
    processedBuffer.byteOffset + processedBuffer.byteLength
  );
});

// ==========================================
// Batch transfer
// ==========================================

ipcMain.handle('binary:processImages', async (event, images: Array<{
  name: string;
  data: ArrayBuffer;
}>) => {
  const results = await Promise.all(images.map(async (img) => {
    const buffer = Buffer.from(img.data);
    const processed = await sharp(buffer)
      .resize(800, 600)
      .jpeg({ quality: 85 })
      .toBuffer();
    
    return {
      name: img.name,
      data: processed.buffer.slice(processed.byteOffset, processed.byteOffset + processed.byteLength),
      size: processed.length,
    };
  }));
  
  return results;
});
```

### ส่ง binary data จาก renderer

```typescript
// src/renderer/binary-client.ts
async function processImageFile(file: File): Promise<Blob> {
  const arrayBuffer = await file.arrayBuffer();
  
  // ส่ง ArrayBuffer ไปยัง main process
  const processedBuffer = await window.electronAPI.invoke(
    'binary:processImage',
    arrayBuffer
  );
  
  // แปลงกลับเป็น Blob
  return new Blob([processedBuffer], { type: 'image/webp' });
}

// ใช้งาน
const input = document.querySelector<HTMLInputElement>('#file-input');
input?.addEventListener('change', async (e) => {
  const file = (e.target as HTMLInputElement).files?.[0];
  if (!file) return;
  
  const processed = await processImageFile(file);
  const url = URL.createObjectURL(processed);
  document.querySelector<HTMLImageElement>('#preview')!.src = url;
});
```

---

## 3. IPC Middleware Pattern

### src/main/ipc-middleware.ts

```typescript
import { ipcMain, IpcMainInvokeEvent } from 'electron';

type Handler = (event: IpcMainInvokeEvent, ...args: any[]) => Promise<any>;
type Middleware = (
  event: IpcMainInvokeEvent,
  args: any[],
  next: () => Promise<any>
) => Promise<any>;

class IPCRouter {
  private middlewares: Middleware[] = [];
  
  // เพิ่ม middleware
  use(middleware: Middleware): this {
    this.middlewares.push(middleware);
    return this;
  }
  
  // Register handler พร้อม middleware
  handle(channel: string, handler: Handler): void {
    ipcMain.handle(channel, async (event, ...args) => {
      // สร้าง middleware chain
      let index = 0;
      
      const next = async (): Promise<any> => {
        if (index < this.middlewares.length) {
          const middleware = this.middlewares[index++];
          return middleware(event, args, next);
        }
        
        // Execute handler
        return handler(event, ...args);
      };
      
      return next();
    });
  }
}

// ==========================================
// Middlewares
// ==========================================

// Logging middleware
const loggingMiddleware: Middleware = async (event, args, next) => {
  const start = Date.now();
  const channel = 'unknown'; // ต้องส่งผ่าน context
  
  console.log(`IPC in: ${JSON.stringify(args)}`);
  
  try {
    const result = await next();
    console.log(`IPC out (${Date.now() - start}ms): ${JSON.stringify(result)}`);
    return result;
  } catch (error) {
    console.error(`IPC error (${Date.now() - start}ms):`, error);
    throw error;
  }
};

// Authentication middleware
const authMiddleware: Middleware = async (event, args, next) => {
  const token = args[0]?.token;
  
  if (!token || !validateToken(token)) {
    throw new Error('Unauthorized');
  }
  
  // ลบ token จาก args ก่อนส่งไป handler
  args[0] = { ...args[0] };
  delete args[0].token;
  
  return next();
};

// Rate limiting middleware
class RateLimiter {
  private requests: Map<string, number[]> = new Map();
  
  createMiddleware(maxRequests: number, windowMs: number): Middleware {
    return async (event, args, next) => {
      const senderId = event.sender.id.toString();
      const now = Date.now();
      
      const requests = this.requests.get(senderId) || [];
      const recentRequests = requests.filter(t => now - t < windowMs);
      
      if (recentRequests.length >= maxRequests) {
        throw new Error('Rate limit exceeded. Please wait.');
      }
      
      recentRequests.push(now);
      this.requests.set(senderId, recentRequests);
      
      return next();
    };
  }
}

function validateToken(token: string): boolean {
  // ตรวจสอบ token
  return token.startsWith('valid_');
}

// ==========================================
// Setup
// ==========================================

const rateLimiter = new RateLimiter();
const router = new IPCRouter();

router.use(loggingMiddleware);
router.use(rateLimiter.createMiddleware(100, 60000)); // 100 requests/minute

router.handle('api:getData', async (event, params) => {
  return { data: 'result', timestamp: Date.now() };
});

router.handle('api:secureData', async (event, params) => {
  // auth middleware ถูก apply แล้ว
  return { secret: 'data' };
});
```

---

## 4. Request Deduplication

### src/renderer/ipc-dedup.ts

```typescript
class DedupIPC {
  private pendingRequests: Map<string, Promise<any>> = new Map();
  
  async invoke<T>(channel: string, ...args: any[]): Promise<T> {
    // สร้าง key จาก channel + args
    const key = `${channel}:${JSON.stringify(args)}`;
    
    // ถ้ามี pending request อยู่แล้ว ให้รอผลเดิม
    if (this.pendingRequests.has(key)) {
      return this.pendingRequests.get(key) as Promise<T>;
    }
    
    // สร้าง request ใหม่
    const promise = window.electronAPI.invoke(channel, ...args)
      .finally(() => {
        this.pendingRequests.delete(key);
      });
    
    this.pendingRequests.set(key, promise);
    
    return promise as Promise<T>;
  }
}

export const dedupIPC = new DedupIPC();

// ตัวอย่างการใช้งาน
// ถ้า component หลายตัวเรียก getConfig พร้อมกัน จะส่ง IPC แค่ครั้งเดียว
const config1 = dedupIPC.invoke('app:getConfig');
const config2 = dedupIPC.invoke('app:getConfig'); // รอผลเดิม
```

---

## 5. IPC Timeout

### src/renderer/ipc-timeout.ts

```typescript
function invokeWithTimeout<T>(
  channel: string,
  timeoutMs: number,
  ...args: any[]
): Promise<T> {
  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(() => {
      reject(new Error(`IPC timeout: ${channel} (${timeoutMs}ms)`));
    }, timeoutMs);
    
    window.electronAPI.invoke(channel, ...args)
      .then((result) => {
        clearTimeout(timeoutId);
        resolve(result as T);
      })
      .catch((error) => {
        clearTimeout(timeoutId);
        reject(error);
      });
  });
}

// AbortController support
function invokeWithAbort<T>(
  channel: string,
  signal: AbortSignal,
  ...args: any[]
): Promise<T> {
  return new Promise((resolve, reject) => {
    if (signal.aborted) {
      reject(new DOMException('Aborted', 'AbortError'));
      return;
    }
    
    const onAbort = () => {
      reject(new DOMException('Aborted', 'AbortError'));
    };
    
    signal.addEventListener('abort', onAbort, { once: true });
    
    window.electronAPI.invoke(channel, ...args)
      .then(resolve)
      .catch(reject)
      .finally(() => {
        signal.removeEventListener('abort', onAbort);
      });
  });
}

// ตัวอย่าง
const controller = new AbortController();
setTimeout(() => controller.abort(), 5000); // Cancel หลัง 5 วินาที

try {
  const result = await invokeWithAbort('slow:operation', controller.signal);
} catch (error) {
  if (error instanceof DOMException && error.name === 'AbortError') {
    console.log('Request was cancelled');
  }
}
```

---

## 6. IPC Bridge Pattern

### src/renderer/ipc-bridge.ts

```typescript
// Bridge ที่ทำให้ IPC ดูเหมือน API ธรรมดา

class IPCBridge {
  // สร้าง proxy object ที่ forward method calls ไป IPC
  createProxy<T extends object>(namespace: string): T {
    return new Proxy({} as T, {
      get(_, prop: string) {
        return (...args: any[]) => {
          return window.electronAPI.invoke(`${namespace}:${prop}`, ...args);
        };
      },
    });
  }
  
  // สร้าง reactive store จาก IPC
  createStore<T>(channel: string, defaultValue: T): {
    get: () => T;
    subscribe: (callback: (value: T) => void) => () => void;
    fetch: () => Promise<T>;
  } {
    let value = defaultValue;
    const subscribers: Array<(val: T) => void> = [];
    
    // Initial fetch
    window.electronAPI.invoke(channel).then((data: T) => {
      value = data;
      subscribers.forEach(fn => fn(value));
    });
    
    // Listen for updates
    window.electronAPI.on(`${channel}:updated`, (newValue: T) => {
      value = newValue;
      subscribers.forEach(fn => fn(value));
    });
    
    return {
      get: () => value,
      subscribe: (callback) => {
        subscribers.push(callback);
        callback(value); // Immediate call with current value
        return () => {
          const idx = subscribers.indexOf(callback);
          if (idx !== -1) subscribers.splice(idx, 1);
        };
      },
      fetch: () => window.electronAPI.invoke(channel) as Promise<T>,
    };
  }
}

const bridge = new IPCBridge();

// ใช้งาน
interface FileAPI {
  read(path: string): Promise<string>;
  write(path: string, content: string): Promise<void>;
  list(dir: string): Promise<string[]>;
}

const fileAPI = bridge.createProxy<FileAPI>('file');
const content = await fileAPI.read('/path/to/file.txt');
await fileAPI.write('/path/to/output.txt', 'hello');

// Reactive store
const configStore = bridge.createStore('app:getConfig', {});
const unsubscribe = configStore.subscribe((config) => {
  console.log('Config updated:', config);
});
```

---

## 7. สรุป

| Pattern | ใช้เมื่อ |
|---------|---------|
| Streaming | ข้อมูลขนาดใหญ่ หรือ progress reporting |
| Binary transfer | Images, files, ArrayBuffers |
| Middleware | Cross-cutting concerns (auth, logging, rate limit) |
| Deduplication | Prevent duplicate concurrent requests |
| Timeout | Long-running operations ที่ต้องมี deadline |
| Bridge pattern | ทำ IPC ดูเป็น regular API |

---

*จบ Part 051 - ต่อไป Part 052: Electron DevTools Extensions*
