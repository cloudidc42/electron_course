# Part 88: Advanced File Operations

## การจัดการไฟล์ขั้นสูงใน Electron

ในบทนี้เราจะเรียน file watchers, virtual filesystem, file locking, streaming large files และ atomic writes

---

## 1. File Watcher ด้วย Chokidar

```typescript
// src/main/fileWatcher.ts
import chokidar from 'chokidar'
import { BrowserWindow, ipcMain } from 'electron'
import * as path from 'path'

export interface FileEvent {
  type: 'add' | 'addDir' | 'change' | 'unlink' | 'unlinkDir'
  path: string
  relativePath?: string
}

export class FileWatcher {
  private watchers = new Map<string, chokidar.FSWatcher>()
  private win: BrowserWindow

  constructor(win: BrowserWindow) {
    this.win = win
  }

  watch(dirPath: string, options?: {
    ignored?: string | string[] | RegExp
    recursive?: boolean
  }): string {
    const watchId = `watch-${Date.now()}`

    const watcher = chokidar.watch(dirPath, {
      ignored: options?.ignored ?? /(^|[/\\])\../,  // ข้าม hidden files
      persistent: true,
      ignoreInitial: false,
      followSymlinks: true,
      depth: options?.recursive ? undefined : 0,
      awaitWriteFinish: {
        stabilityThreshold: 100,
        pollInterval: 100
      }
    })

    const emit = (type: FileEvent['type']) => (filePath: string) => {
      this.win.webContents.send('file:event', {
        watchId,
        event: {
          type,
          path: filePath,
          relativePath: path.relative(dirPath, filePath)
        } as FileEvent
      })
    }

    watcher
      .on('add', emit('add'))
      .on('addDir', emit('addDir'))
      .on('change', emit('change'))
      .on('unlink', emit('unlink'))
      .on('unlinkDir', emit('unlinkDir'))
      .on('error', (err) => console.error('Watcher error:', err))

    this.watchers.set(watchId, watcher)
    return watchId
  }

  unwatch(watchId: string): void {
    const watcher = this.watchers.get(watchId)
    if (watcher) {
      watcher.close()
      this.watchers.delete(watchId)
    }
  }

  unwatchAll(): void {
    for (const [id] of this.watchers) {
      this.unwatch(id)
    }
  }
}
```

---

## 2. File Locking

```typescript
// src/main/fileLock.ts
// ป้องกันไฟล์ถูก modify พร้อมกัน

import * as fs from 'fs/promises'
import * as path from 'path'

interface LockInfo {
  pid: number
  timestamp: number
  owner: string
}

const LOCK_TIMEOUT_MS = 10_000  // 10 seconds

export class FileLockManager {
  private locks = new Map<string, LockInfo>()

  // Acquire lock
  async lock(filePath: string, owner = 'main'): Promise<boolean> {
    const lockPath = filePath + '.lock'

    // ตรวจสอบ lock ที่มีอยู่
    try {
      const existing = await fs.readFile(lockPath, 'utf-8')
      const info: LockInfo = JSON.parse(existing)

      // ตรวจ timeout
      if (Date.now() - info.timestamp < LOCK_TIMEOUT_MS) {
        // Lock ยังใช้งานอยู่
        if (info.pid !== process.pid) {
          return false  // ไม่สามารถ lock ได้
        }
        // เรา lock อยู่แล้ว, renew
      }
      // Lock หมดอายุแล้ว
    } catch {
      // ไม่มี lock file
    }

    const lockInfo: LockInfo = {
      pid: process.pid,
      timestamp: Date.now(),
      owner
    }

    // สร้าง lock file แบบ atomic
    try {
      await fs.writeFile(lockPath, JSON.stringify(lockInfo), { flag: 'wx' })
      this.locks.set(filePath, lockInfo)
      return true
    } catch {
      return false
    }
  }

  // Release lock
  async unlock(filePath: string): Promise<void> {
    const lockPath = filePath + '.lock'
    try {
      await fs.unlink(lockPath)
    } catch {
      // ignore if already gone
    }
    this.locks.delete(filePath)
  }

  // Execute with lock
  async withLock<T>(filePath: string, fn: () => Promise<T>): Promise<T> {
    const acquired = await this.lock(filePath)
    if (!acquired) {
      throw new Error(`Cannot acquire lock on ${filePath}`)
    }

    try {
      return await fn()
    } finally {
      await this.unlock(filePath)
    }
  }

  // Renew lock ก่อน timeout
  startHeartbeat(filePath: string): () => void {
    const interval = setInterval(() => {
      const info = this.locks.get(filePath)
      if (info) {
        info.timestamp = Date.now()
        const lockPath = filePath + '.lock'
        fs.writeFile(lockPath, JSON.stringify(info)).catch(console.error)
      }
    }, LOCK_TIMEOUT_MS / 2)

    return () => clearInterval(interval)
  }
}
```

---

## 3. Atomic File Write

```typescript
// src/main/atomicWrite.ts
// เขียนไฟล์แบบ atomic เพื่อป้องกัน data corruption

import * as fs from 'fs/promises'
import * as path from 'path'
import * as crypto from 'crypto'
import * as os from 'os'

export async function atomicWrite(
  filePath: string,
  content: string | Buffer,
  options?: { encoding?: BufferEncoding; mode?: number }
): Promise<void> {
  const dir = path.dirname(filePath)
  const tmpName = path.join(
    os.tmpdir(),  // ใช้ tmpdir เพื่อ cross-filesystem safety
    `tmp-${crypto.randomBytes(8).toString('hex')}`
  )

  // เขียนไปยัง temp file ก่อน
  await fs.writeFile(tmpName, content, {
    encoding: options?.encoding ?? 'utf-8',
    mode: options?.mode ?? 0o644
  })

  // ย้ายแบบ atomic (rename)
  try {
    await fs.rename(tmpName, filePath)
  } catch (err: unknown) {
    // ถ้า cross-device (EXDEV) ต้อง copy แล้วลบ
    if ((err as NodeJS.ErrnoException).code === 'EXDEV') {
      await fs.copyFile(tmpName, filePath)
      await fs.unlink(tmpName)
    } else {
      await fs.unlink(tmpName).catch(() => {})
      throw err
    }
  }
}

// Atomic JSON write
export async function atomicWriteJSON(filePath: string, data: unknown): Promise<void> {
  const content = JSON.stringify(data, null, 2)
  await atomicWrite(filePath, content, { encoding: 'utf-8' })
}
```

---

## 4. Streaming Large Files

```typescript
// src/main/streamHandler.ts
import * as fs from 'fs'
import * as path from 'path'
import { ipcMain, BrowserWindow } from 'electron'
import * as crypto from 'crypto'
import * as zlib from 'zlib'
import { pipeline } from 'stream/promises'

const CHUNK_SIZE = 64 * 1024  // 64KB chunks

export function registerStreamHandlers(win: BrowserWindow): void {
  // Stream ไฟล์ขนาดใหญ่ไปยัง renderer
  ipcMain.handle('file:stream-read', async (_, { filePath, requestId }: {
    filePath: string
    requestId: string
  }) => {
    const stat = await fs.promises.stat(filePath)
    const totalSize = stat.size
    let bytesRead = 0

    return new Promise<void>((resolve, reject) => {
      const stream = fs.createReadStream(filePath, { highWaterMark: CHUNK_SIZE })

      stream.on('data', (chunk: Buffer) => {
        bytesRead += chunk.length
        win.webContents.send('file:stream-chunk', {
          requestId,
          chunk: chunk.toString('base64'),
          progress: bytesRead / totalSize
        })
      })

      stream.on('end', () => {
        win.webContents.send('file:stream-done', { requestId })
        resolve()
      })

      stream.on('error', (err) => {
        win.webContents.send('file:stream-error', { requestId, error: err.message })
        reject(err)
      })
    })
  })

  // Hash ไฟล์ขนาดใหญ่โดยไม่ต้อง load ทั้งหมดในหน่วยความจำ
  ipcMain.handle('file:hash', async (_, { filePath, algorithm = 'sha256' }: {
    filePath: string
    algorithm?: string
  }) => {
    return new Promise<string>((resolve, reject) => {
      const hash = crypto.createHash(algorithm)
      const stream = fs.createReadStream(filePath)

      stream.on('data', (chunk: Buffer) => hash.update(chunk))
      stream.on('end', () => resolve(hash.digest('hex')))
      stream.on('error', reject)
    })
  })

  // Compress ไฟล์
  ipcMain.handle('file:compress', async (_, { inputPath, outputPath }: {
    inputPath: string
    outputPath: string
  }) => {
    await pipeline(
      fs.createReadStream(inputPath),
      zlib.createGzip({ level: 6 }),
      fs.createWriteStream(outputPath)
    )
    return { outputPath }
  })
}
```

---

## 5. Virtual Filesystem

```typescript
// src/renderer/vfs/virtualFS.ts
// Virtual Filesystem สำหรับ in-memory file tree

interface VFSNode {
  name: string
  type: 'file' | 'directory'
  content?: string
  children?: Map<string, VFSNode>
  size?: number
  modified?: Date
}

export class VirtualFileSystem {
  private root: VFSNode = {
    name: '/',
    type: 'directory',
    children: new Map()
  }

  private resolvePath(filePath: string): VFSNode | null {
    const parts = filePath.split('/').filter(Boolean)
    let current = this.root

    for (const part of parts) {
      if (!current.children?.has(part)) return null
      current = current.children.get(part)!
    }

    return current
  }

  mkdir(dirPath: string): void {
    const parts = dirPath.split('/').filter(Boolean)
    let current = this.root

    for (const part of parts) {
      if (!current.children) current.children = new Map()
      if (!current.children.has(part)) {
        current.children.set(part, {
          name: part,
          type: 'directory',
          children: new Map()
        })
      }
      current = current.children.get(part)!
    }
  }

  writeFile(filePath: string, content: string): void {
    const dir = filePath.split('/').slice(0, -1).join('/')
    const name = filePath.split('/').pop()!

    if (dir) this.mkdir(dir)

    const parent = dir ? this.resolvePath(dir) : this.root
    if (!parent || parent.type !== 'directory') throw new Error(`Not a directory: ${dir}`)

    if (!parent.children) parent.children = new Map()
    parent.children.set(name, {
      name,
      type: 'file',
      content,
      size: new TextEncoder().encode(content).length,
      modified: new Date()
    })
  }

  readFile(filePath: string): string {
    const node = this.resolvePath(filePath)
    if (!node) throw new Error(`File not found: ${filePath}`)
    if (node.type !== 'file') throw new Error(`Not a file: ${filePath}`)
    return node.content ?? ''
  }

  listDir(dirPath: string): Array<{ name: string; type: 'file' | 'directory'; size?: number }> {
    const node = dirPath === '/' ? this.root : this.resolvePath(dirPath)
    if (!node || node.type !== 'directory') throw new Error(`Not a directory: ${dirPath}`)

    return Array.from(node.children?.values() ?? []).map(n => ({
      name: n.name,
      type: n.type,
      size: n.size
    }))
  }

  exists(filePath: string): boolean {
    return this.resolvePath(filePath) !== null
  }

  delete(filePath: string): void {
    const parts = filePath.split('/').filter(Boolean)
    const name = parts.pop()!
    const parent = parts.length ? this.resolvePath('/' + parts.join('/')) : this.root
    parent?.children?.delete(name)
  }
}
```

---

## สรุป

| ฟีเจอร์ | เครื่องมือ | ข้อดี |
|--------|---------|-------|
| File watching | chokidar | cross-platform, debounced |
| File locking | lock files | prevent concurrent edits |
| Atomic writes | rename trick | no data corruption |
| Stream reading | fs.createReadStream | ไม่กิน RAM |
| File hashing | crypto.createHash + stream | hash ไฟล์ใหญ่ |
| Virtual FS | in-memory Map | testing, temp files |
