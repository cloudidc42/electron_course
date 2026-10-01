# Part 72: Advanced TypeScript ใน Electron

## TypeScript แบบมืออาชีพสำหรับ Electron Apps

ในบทนี้เราจะเรียน advanced TypeScript patterns: declaration merging สำหรับ contextBridge, typed IPC channels ด้วย generics, zod validation, branded types และ strict mode

---

## 1. Typed IPC Channels ด้วย Generics

```typescript
// src/shared/ipcTypes.ts
// กำหนด type ของทุก IPC channel ที่ใช้ในแอป

// Request/Response pairs
export interface IPCChannels {
  // File operations
  'file:open': { request: { path?: string }; response: FileData | null }
  'file:save': { request: { path: string; content: string }; response: { success: boolean } }
  'file:delete': { request: { path: string }; response: { success: boolean; error?: string } }
  
  // System
  'system:get-info': { request: void; response: SystemInfo }
  'system:kill-process': { request: { pid: number }; response: { success: boolean } }
  
  // Settings
  'settings:get': { request: void; response: AppSettings }
  'settings:set': { request: Partial<AppSettings>; response: void }
  
  // Auth
  'auth:unlock': { request: { password: string }; response: { success: boolean; error?: string } }
}

// One-way events (main -> renderer)
export interface MainToRendererEvents {
  'app:before-close': void
  'theme:changed': { shouldUseDark: boolean }
  'update:available': { version: string; releaseNotes: string }
  'file:changed': { path: string; type: 'add' | 'change' | 'unlink' }
}

// One-way events (renderer -> main)
export interface RendererToMainEvents {
  'app:confirm-close': void
  'window:minimize': void
  'window:maximize': void
}

export interface FileData {
  path: string
  name: string
  content: string
  extension: string
  size: number
}

export interface SystemInfo {
  cpu: { usage: number; cores: number; model: string }
  memory: { total: number; used: number; free: number }
  platform: string
  uptime: number
}

export interface AppSettings {
  theme: 'light' | 'dark' | 'system'
  fontSize: number
  language: string
  autoUpdate: boolean
}
```

---

## 2. Type-Safe IPC Handler (Main Process)

```typescript
// src/main/typedIPC.ts
import { ipcMain, IpcMainInvokeEvent, IpcMainEvent, BrowserWindow } from 'electron'
import { IPCChannels, MainToRendererEvents, RendererToMainEvents } from '../shared/ipcTypes'

type Channel = keyof IPCChannels
type ExtractRequest<C extends Channel> = IPCChannels[C]['request']
type ExtractResponse<C extends Channel> = IPCChannels[C]['response']

// Type-safe ipcMain.handle
export function handleIPC<C extends Channel>(
  channel: C,
  handler: (
    event: IpcMainInvokeEvent,
    ...args: ExtractRequest<C> extends void ? [] : [ExtractRequest<C>]
  ) => Promise<ExtractResponse<C>> | ExtractResponse<C>
): void {
  ipcMain.handle(channel, handler as Parameters<typeof ipcMain.handle>[1])
}

// Type-safe ipcMain.on
export function onIPC<C extends keyof RendererToMainEvents>(
  channel: C,
  handler: (
    event: IpcMainEvent,
    ...args: RendererToMainEvents[C] extends void ? [] : [RendererToMainEvents[C]]
  ) => void
): void {
  ipcMain.on(channel, handler as Parameters<typeof ipcMain.on>[1])
}

// Type-safe send to renderer
export function sendToRenderer<C extends keyof MainToRendererEvents>(
  win: BrowserWindow,
  channel: C,
  ...args: MainToRendererEvents[C] extends void ? [] : [MainToRendererEvents[C]]
): void {
  win.webContents.send(channel, ...args)
}

// ตัวอย่างการใช้งาน
export function registerTypedHandlers(win: BrowserWindow): void {
  handleIPC('file:open', async (_, args) => {
    // args มี type: { path?: string }
    const { path } = args || {}
    // ... implementation
    return null
  })

  handleIPC('system:get-info', async () => {
    // ไม่มี args (void)
    return {
      cpu: { usage: 45.2, cores: 8, model: 'Intel Core i7' },
      memory: { total: 16 * 1024 ** 3, used: 8 * 1024 ** 3, free: 8 * 1024 ** 3 },
      platform: 'darwin',
      uptime: 3600
    }
  })

  onIPC('app:confirm-close', () => {
    win.destroy()
  })
}
```

---

## 3. Declaration Merging สำหรับ contextBridge

```typescript
// src/preload/index.ts
import { contextBridge, ipcRenderer } from 'electron'
import { IPCChannels, MainToRendererEvents, RendererToMainEvents } from '../shared/ipcTypes'

type Channel = keyof IPCChannels

// Type-safe electronAPI
const electronAPI = {
  // invoke - handles typed channels
  invoke: async <C extends Channel>(
    channel: C,
    ...args: IPCChannels[C]['request'] extends void ? [] : [IPCChannels[C]['request']]
  ): Promise<IPCChannels[C]['response']> => {
    return ipcRenderer.invoke(channel, ...args)
  },

  // send - one-way to main
  send: <C extends keyof RendererToMainEvents>(
    channel: C,
    ...args: RendererToMainEvents[C] extends void ? [] : [RendererToMainEvents[C]]
  ): void => {
    ipcRenderer.send(channel, ...args)
  },

  // on - listen to main events
  on: <C extends keyof MainToRendererEvents>(
    channel: C,
    callback: (data: MainToRendererEvents[C]) => void
  ): (() => void) => {
    const listener = (_: Electron.IpcRendererEvent, data: MainToRendererEvents[C]) => callback(data)
    ipcRenderer.on(channel, listener)
    return () => ipcRenderer.removeListener(channel, listener)
  }
}

contextBridge.exposeInMainWorld('electronAPI', electronAPI)

// Type declarations สำหรับ renderer
export type ElectronAPI = typeof electronAPI
```

```typescript
// src/renderer/types/electron.d.ts
import { ElectronAPI } from '../../preload/index'

// Declaration merging - เพิ่ม type เข้า Window
declare global {
  interface Window {
    electronAPI: ElectronAPI
  }
}
```

---

## 4. Zod Validation ใน IPC

```typescript
// src/shared/schemas.ts
import { z } from 'zod'

// Schema definitions
export const FileDataSchema = z.object({
  path: z.string().min(1),
  name: z.string().min(1),
  content: z.string(),
  extension: z.string(),
  size: z.number().nonnegative()
})

export const AppSettingsSchema = z.object({
  theme: z.enum(['light', 'dark', 'system']),
  fontSize: z.number().int().min(8).max(72),
  language: z.string().length(2),
  autoUpdate: z.boolean()
})

export const SystemInfoSchema = z.object({
  cpu: z.object({
    usage: z.number().min(0).max(100),
    cores: z.number().int().positive(),
    model: z.string()
  }),
  memory: z.object({
    total: z.number().positive(),
    used: z.number().nonnegative(),
    free: z.number().nonnegative()
  }),
  platform: z.string(),
  uptime: z.number().nonnegative()
})

// Inferred types จาก schemas
export type FileData = z.infer<typeof FileDataSchema>
export type AppSettings = z.infer<typeof AppSettingsSchema>
export type SystemInfo = z.infer<typeof SystemInfoSchema>

// Validation helper
export function validateIPC<T>(schema: z.ZodSchema<T>, data: unknown): { 
  success: true; data: T 
} | { 
  success: false; errors: z.ZodError 
} {
  const result = schema.safeParse(data)
  if (result.success) return { success: true, data: result.data }
  return { success: false, errors: result.error }
}
```

```typescript
// src/main/validatedHandlers.ts
import { ipcMain } from 'electron'
import { z } from 'zod'
import { AppSettingsSchema, validateIPC } from '../shared/schemas'

// Middleware pattern สำหรับ IPC validation
function createValidatedHandler<TRequest, TResponse>(
  schema: z.ZodSchema<TRequest>,
  handler: (data: TRequest) => Promise<TResponse>
) {
  return async (_: Electron.IpcMainInvokeEvent, rawData: unknown): Promise<TResponse | { error: string }> => {
    const validation = validateIPC(schema, rawData)
    if (!validation.success) {
      return { error: validation.errors.issues.map(i => i.message).join(', ') }
    }
    return handler(validation.data)
  }
}

// ตัวอย่างการใช้งาน
export function registerValidatedHandlers(): void {
  ipcMain.handle(
    'settings:set',
    createValidatedHandler(
      AppSettingsSchema.partial(),
      async (settings) => {
        // settings มี type AppSettings ที่ validated แล้ว
        console.log('Saving settings:', settings)
      }
    )
  )
}
```

---

## 5. Branded Types

```typescript
// src/shared/brandedTypes.ts

// Branded types ป้องกันการผสม path types
declare const PathBrand: unique symbol
export type AbsolutePath = string & { readonly [PathBrand]: 'absolute' }

declare const EncryptedBrand: unique symbol
export type EncryptedString = string & { readonly [EncryptedBrand]: 'encrypted' }

declare const SanitizedBrand: unique symbol
export type SanitizedHtml = string & { readonly [SanitizedBrand]: 'sanitized' }

declare const UserIdBrand: unique symbol
export type UserId = string & { readonly [UserIdBrand]: 'userId' }

// Type guards / constructors
export function toAbsolutePath(path: string): AbsolutePath {
  if (!path.startsWith('/') && !path.match(/^[A-Z]:\\/)) {
    throw new Error(`"${path}" is not an absolute path`)
  }
  return path as AbsolutePath
}

export function toUserId(id: string): UserId {
  if (!/^[a-f0-9]{16,}$/.test(id)) {
    throw new Error(`"${id}" is not a valid user ID`)
  }
  return id as UserId
}

// ตัวอย่างการใช้
function readFile(path: AbsolutePath): Promise<string> {
  // ไม่รับ relative path
  return Promise.resolve('')
}

function getUserData(userId: UserId): Promise<unknown> {
  return Promise.resolve({})
}

// Error cases ที่ TypeScript จะจับ
const relativePath = './some/file.txt'
// readFile(relativePath)  // ❌ TypeScript Error!

const absolutePath = toAbsolutePath('/home/user/file.txt')
readFile(absolutePath)  // ✅ OK
```

---

## 6. Strict Mode Configuration

```json
// tsconfig.json (Full strict mode)
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "lib": ["ES2022", "DOM"],
    "moduleResolution": "bundler",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "useUnknownInCatchVariables": true,
    "alwaysStrict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "allowUnusedLabels": false,
    "allowUnreachableCode": false
  }
}
```

---

## 7. Utility Types สำหรับ Electron Patterns

```typescript
// src/shared/utilityTypes.ts

// Deep readonly สำหรับ immutable state
export type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T

// Nullable utility
export type Nullable<T> = T | null
export type Optional<T> = T | undefined
export type Maybe<T> = T | null | undefined

// Promise-based IPC result
export type AsyncResult<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E }

export async function tryAsync<T>(
  fn: () => Promise<T>
): Promise<AsyncResult<T>> {
  try {
    const value = await fn()
    return { ok: true, value }
  } catch (error) {
    return { ok: false, error: error as Error }
  }
}

// Extract keys with specific value types
export type KeysWithValueType<T, V> = {
  [K in keyof T]: T[K] extends V ? K : never
}[keyof T]

// Partial with required fields
export type PartialWith<T, K extends keyof T> = Omit<T, K> & Pick<T, K>

// NonEmpty array
export type NonEmptyArray<T> = [T, ...T[]]
export function isNonEmpty<T>(arr: T[]): arr is NonEmptyArray<T> {
  return arr.length > 0
}

// Discriminated union helpers
export function match<T extends { type: string }, R>(
  value: T,
  handlers: { [K in T['type']]: (v: Extract<T, { type: K }>) => R }
): R {
  const handler = handlers[value.type as T['type']]
  return (handler as (v: T) => R)(value)
}

// ตัวอย่าง match
type AppEvent =
  | { type: 'file-opened'; path: string }
  | { type: 'file-saved'; path: string; size: number }
  | { type: 'error'; message: string }

function handleEvent(event: AppEvent): string {
  return match(event, {
    'file-opened': (e) => `เปิดไฟล์: ${e.path}`,
    'file-saved': (e) => `บันทึก ${e.path} (${e.size} bytes)`,
    'error': (e) => `ข้อผิดพลาด: ${e.message}`
  })
}
```

---

## สรุป

| Feature | ประโยชน์ |
|---------|---------|
| Typed IPC channels | ป้องกัน typos และ type mismatch |
| Declaration merging | Window.electronAPI มี type ครบ |
| Zod validation | Validate data จาก IPC ก่อนใช้ |
| Branded types | ป้องกันผสม path/ID types |
| Strict mode | จับ bugs ตั้งแต่ compile time |
| Utility types | Code ที่อ่านง่ายและ reusable |
| match function | Type-safe exhaustive checks |
