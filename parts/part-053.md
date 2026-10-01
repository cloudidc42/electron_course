# Part 053: Custom Context Bridge Patterns
## Pattern การออกแบบ contextBridge ขั้นสูง

---

## 🎯 เป้าหมายของบทเรียนนี้

- Type-safe contextBridge
- Namespaced APIs
- Lazy initialization
- Event emitter pattern
- Observable/reactive patterns
- Cleanup

---

## 1. Basic Type-Safe contextBridge

### src/preload/preload.ts

```typescript
import { contextBridge, ipcRenderer } from 'electron';

// ==========================================
// Type Definitions
// ==========================================

export interface ElectronAPI {
  app: AppAPI;
  files: FileAPI;
  settings: SettingsAPI;
  events: EventAPI;
  shell: ShellAPI;
}

interface AppAPI {
  getVersion: () => Promise<string>;
  quit: () => void;
  minimize: () => void;
  maximize: () => void;
  isMaximized: () => Promise<boolean>;
}

interface FileAPI {
  read: (path: string) => Promise<{ content: string; encoding: string }>;
  write: (path: string, content: string) => Promise<void>;
  openDialog: (options?: Electron.OpenDialogOptions) => Promise<string[]>;
  saveDialog: (options?: Electron.SaveDialogOptions) => Promise<string | null>;
}

interface SettingsAPI {
  get: <T>(key: string) => Promise<T>;
  set: (key: string, value: any) => Promise<void>;
  delete: (key: string) => Promise<void>;
  getAll: () => Promise<Record<string, any>>;
}

type EventCallback<T = any> = (data: T) => void;
type RemoveListener = () => void;

interface EventAPI {
  on: (channel: string, callback: EventCallback) => RemoveListener;
  once: (channel: string, callback: EventCallback) => void;
}

interface ShellAPI {
  openExternal: (url: string) => Promise<void>;
  openPath: (path: string) => Promise<string>;
  showItemInFolder: (path: string) => void;
  beep: () => void;
}

// ==========================================
// Implementation
// ==========================================

const electronAPI: ElectronAPI = {
  app: {
    getVersion: () => ipcRenderer.invoke('app:getVersion'),
    quit: () => ipcRenderer.send('app:quit'),
    minimize: () => ipcRenderer.send('app:minimize'),
    maximize: () => ipcRenderer.send('app:maximize'),
    isMaximized: () => ipcRenderer.invoke('app:isMaximized'),
  },
  
  files: {
    read: (path) => ipcRenderer.invoke('file:read', path),
    write: (path, content) => ipcRenderer.invoke('file:write', path, content),
    openDialog: (options) => ipcRenderer.invoke('dialog:open', options),
    saveDialog: (options) => ipcRenderer.invoke('dialog:save', options),
  },
  
  settings: {
    get: (key) => ipcRenderer.invoke('settings:get', key),
    set: (key, value) => ipcRenderer.invoke('settings:set', key, value),
    delete: (key) => ipcRenderer.invoke('settings:delete', key),
    getAll: () => ipcRenderer.invoke('settings:getAll'),
  },
  
  events: {
    on: (channel, callback) => {
      const handler = (_: any, data: any) => callback(data);
      ipcRenderer.on(channel, handler);
      
      // Return cleanup function
      return () => ipcRenderer.removeListener(channel, handler);
    },
    once: (channel, callback) => {
      ipcRenderer.once(channel, (_, data) => callback(data));
    },
  },
  
  shell: {
    openExternal: (url) => ipcRenderer.invoke('shell:openExternal', url),
    openPath: (path) => ipcRenderer.invoke('shell:openPath', path),
    showItemInFolder: (path) => ipcRenderer.send('shell:showItemInFolder', path),
    beep: () => ipcRenderer.send('shell:beep'),
  },
};

contextBridge.exposeInMainWorld('electron', electronAPI);
```

---

## 2. Namespaced API

```typescript
// src/preload/namespaced.ts

// ==========================================
// Namespace pattern - แยก API ตาม domain
// ==========================================

const createNamespace = <T extends Record<string, (...args: any[]) => any>>(
  namespace: string,
  methods: { [K in keyof T]: T[K] }
): T => {
  const api: any = {};
  
  for (const [method, handler] of Object.entries(methods)) {
    api[method] = (...args: any[]) => {
      // Log ใน development
      if (process.env.NODE_ENV === 'development') {
        console.log(`[${namespace}:${method}]`, args);
      }
      
      return (handler as Function)(...args);
    };
  }
  
  return api;
};

// ==========================================
// Feature-based namespaces
// ==========================================

contextBridge.exposeInMainWorld('$notes', createNamespace('notes', {
  getAll: () => ipcRenderer.invoke('notes:getAll'),
  get: (id: string) => ipcRenderer.invoke('notes:get', id),
  create: (data: any) => ipcRenderer.invoke('notes:create', data),
  update: (id: string, data: any) => ipcRenderer.invoke('notes:update', id, data),
  delete: (id: string) => ipcRenderer.invoke('notes:delete', id),
  search: (query: string) => ipcRenderer.invoke('notes:search', query),
}));

contextBridge.exposeInMainWorld('$auth', createNamespace('auth', {
  login: (credentials: any) => ipcRenderer.invoke('auth:login', credentials),
  logout: () => ipcRenderer.invoke('auth:logout'),
  getUser: () => ipcRenderer.invoke('auth:getUser'),
  refreshToken: () => ipcRenderer.invoke('auth:refreshToken'),
}));
```

---

## 3. Lazy Initialization

```typescript
// src/preload/lazy-api.ts

type LazyModule<T> = {
  (): T;
};

function createLazy<T>(factory: () => T): LazyModule<T> {
  let instance: T | undefined;
  
  return () => {
    if (!instance) {
      instance = factory();
    }
    return instance;
  };
}

// APIs ที่ initialize ช้า
const lazyDatabaseAPI = createLazy(() => {
  console.log('Initializing Database API...');
  
  return {
    query: (sql: string, params?: any[]) => 
      ipcRenderer.invoke('db:query', sql, params),
    
    transaction: (operations: Array<{ sql: string; params: any[] }>) =>
      ipcRenderer.invoke('db:transaction', operations),
    
    backup: (path: string) =>
      ipcRenderer.invoke('db:backup', path),
  };
});

const lazyMLAPI = createLazy(() => {
  console.log('Initializing ML API...');
  
  return {
    predict: (input: any) => ipcRenderer.invoke('ml:predict', input),
    train: (data: any) => ipcRenderer.invoke('ml:train', data),
    getModel: () => ipcRenderer.invoke('ml:getModel'),
  };
});

// Expose ด้วย getter สำหรับ lazy initialization
contextBridge.exposeInMainWorld('electronAPI', {
  // Regular APIs
  app: { /* ... */ },
  
  // Lazy APIs
  get db() {
    return lazyDatabaseAPI();
  },
  
  get ml() {
    return lazyMLAPI();
  },
});
```

---

## 4. Event Emitter Pattern

### src/preload/event-emitter.ts

```typescript
type EventListener<T> = (data: T) => void;

// Type-safe event emitter ที่ใช้ contextBridge
class TypedEventBus<Events extends Record<string, any>> {
  private listeners: Map<keyof Events, Set<EventListener<any>>> = new Map();
  private ipcChannelPrefix: string;
  
  constructor(channelPrefix: string) {
    this.ipcChannelPrefix = channelPrefix;
    this.setupIPCForwarding();
  }
  
  // Forward IPC events ไปยัง listeners
  private setupIPCForwarding(): void {
    // รับ events จาก main process
    ipcRenderer.on('event-bus:*', (_, { type, data }) => {
      this.emit(type as keyof Events, data);
    });
  }
  
  // Subscribe to event
  on<K extends keyof Events>(event: K, listener: EventListener<Events[K]>): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    
    this.listeners.get(event)!.add(listener as EventListener<any>);
    
    // Subscribe ใน main process ด้วย
    ipcRenderer.send(`${this.ipcChannelPrefix}:subscribe`, String(event));
    
    return () => this.off(event, listener);
  }
  
  // Unsubscribe
  off<K extends keyof Events>(event: K, listener: EventListener<Events[K]>): void {
    this.listeners.get(event)?.delete(listener as EventListener<any>);
    
    if (this.listeners.get(event)?.size === 0) {
      ipcRenderer.send(`${this.ipcChannelPrefix}:unsubscribe`, String(event));
    }
  }
  
  // Emit event (renderer -> main)
  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    // Notify local listeners
    this.listeners.get(event)?.forEach(listener => listener(data));
  }
  
  // One-time listener
  once<K extends keyof Events>(event: K, listener: EventListener<Events[K]>): void {
    const remove = this.on(event, (data) => {
      remove();
      listener(data);
    });
  }
  
  // Remove all listeners
  removeAllListeners<K extends keyof Events>(event?: K): void {
    if (event) {
      this.listeners.get(event)?.clear();
    } else {
      this.listeners.clear();
    }
  }
}

// Define event types
interface AppEvents {
  'update:available': { version: string };
  'update:downloaded': { version: string };
  'file:changed': { path: string; type: 'created' | 'modified' | 'deleted' };
  'network:change': { isOnline: boolean };
  'theme:change': { theme: 'light' | 'dark' };
  'config:update': Record<string, any>;
}

const eventBus = new TypedEventBus<AppEvents>('events');

contextBridge.exposeInMainWorld('events', {
  on: <K extends keyof AppEvents>(
    event: K, 
    listener: EventListener<AppEvents[K]>
  ) => eventBus.on(event, listener),
  
  once: <K extends keyof AppEvents>(
    event: K, 
    listener: EventListener<AppEvents[K]>
  ) => eventBus.once(event, listener),
  
  off: <K extends keyof AppEvents>(
    event: K, 
    listener: EventListener<AppEvents[K]>
  ) => eventBus.off(event, listener),
});
```

---

## 5. Observable/Reactive Pattern

### src/preload/observable.ts

```typescript
// Simple Observable implementation
class Observable<T> {
  private subscribers: Set<(value: T) => void> = new Set();
  private _value: T;
  
  constructor(initialValue: T) {
    this._value = initialValue;
  }
  
  get value(): T {
    return this._value;
  }
  
  next(value: T): void {
    this._value = value;
    this.subscribers.forEach(fn => fn(value));
  }
  
  subscribe(callback: (value: T) => void): () => void {
    this.subscribers.add(callback);
    callback(this._value); // Immediate call
    
    return () => this.subscribers.delete(callback);
  }
  
  map<U>(transform: (value: T) => U): Observable<U> {
    const mapped = new Observable<U>(transform(this._value));
    
    this.subscribe(value => {
      mapped.next(transform(value));
    });
    
    return mapped;
  }
  
  filter(predicate: (value: T) => boolean): Observable<T> {
    const filtered = new Observable<T>(this._value);
    
    this.subscribe(value => {
      if (predicate(value)) {
        filtered.next(value);
      }
    });
    
    return filtered;
  }
}

// ==========================================
// Reactive State Store
// ==========================================

interface AppState {
  theme: 'light' | 'dark';
  isOnline: boolean;
  user: { id: string; name: string } | null;
}

const state$ = new Observable<AppState>({
  theme: 'light',
  isOnline: true,
  user: null,
});

// IPC -> Observable bridge
ipcRenderer.on('state:theme', (_, theme: 'light' | 'dark') => {
  state$.next({ ...state$.value, theme });
});

ipcRenderer.on('state:network', (_, isOnline: boolean) => {
  state$.next({ ...state$.value, isOnline });
});

ipcRenderer.on('state:user', (_, user: any) => {
  state$.next({ ...state$.value, user });
});

// Expose ไปยัง renderer
contextBridge.exposeInMainWorld('state$', {
  // Subscribe to full state
  subscribe: (callback: (state: AppState) => void) => state$.subscribe(callback),
  
  // Subscribe to specific field
  select: <K extends keyof AppState>(
    key: K,
    callback: (value: AppState[K]) => void
  ) => {
    return state$.map(state => state[key]).subscribe(callback);
  },
  
  // Get current value
  get: () => state$.value,
});
```

---

## 6. Cleanup Pattern

### src/preload/cleanup.ts

```typescript
class CleanupRegistry {
  private cleanups: Array<() => void> = [];
  
  register(cleanup: () => void): void {
    this.cleanups.push(cleanup);
  }
  
  cleanup(): void {
    this.cleanups.forEach(fn => {
      try {
        fn();
      } catch (err) {
        console.error('Cleanup error:', err);
      }
    });
    this.cleanups = [];
  }
}

const registry = new CleanupRegistry();

// IPC listeners ที่ต้อง cleanup
function setupIPCListeners(): void {
  const handleThemeChange = (_: any, theme: string) => {
    document.documentElement.setAttribute('data-theme', theme);
  };
  
  ipcRenderer.on('theme:change', handleThemeChange);
  
  registry.register(() => {
    ipcRenderer.removeListener('theme:change', handleThemeChange);
  });
}

// Cleanup เมื่อ page unload
window.addEventListener('beforeunload', () => {
  registry.cleanup();
  ipcRenderer.send('preload:cleanup');
});

// Expose cleanup API
contextBridge.exposeInMainWorld('cleanup', {
  register: (cleanup: () => void) => registry.register(cleanup),
  cleanup: () => registry.cleanup(),
});
```

---

## 7. Declaration File

### src/renderer/electron.d.ts

```typescript
import type { ElectronAPI } from '../preload/preload';
import type { AppEvents } from '../preload/event-emitter';
import type { AppState } from '../preload/observable';

declare global {
  interface Window {
    electron: ElectronAPI;
    
    $notes: {
      getAll(): Promise<any[]>;
      get(id: string): Promise<any>;
      create(data: any): Promise<any>;
      update(id: string, data: any): Promise<any>;
      delete(id: string): Promise<void>;
      search(query: string): Promise<any[]>;
    };
    
    events: {
      on<K extends keyof AppEvents>(event: K, listener: (data: AppEvents[K]) => void): () => void;
      once<K extends keyof AppEvents>(event: K, listener: (data: AppEvents[K]) => void): void;
      off<K extends keyof AppEvents>(event: K, listener: (data: AppEvents[K]) => void): void;
    };
    
    state$: {
      subscribe(callback: (state: AppState) => void): () => void;
      select<K extends keyof AppState>(key: K, callback: (value: AppState[K]) => void): () => void;
      get(): AppState;
    };
    
    cleanup: {
      register(cleanup: () => void): void;
      cleanup(): void;
    };
  }
}

export {};
```

---

## 8. สรุป

### Best Practices

1. **Type everything** - ใช้ TypeScript interfaces สำหรับทุก API
2. **Return cleanup functions** - event listeners ควร return cleanup function
3. **Namespace APIs** - แยก API ตาม domain (`$notes`, `$auth`, etc.)
4. **Lazy initialization** - โหลด APIs ที่ใช้น้อยแบบ lazy
5. **Declaration files** - สร้าง `electron.d.ts` สำหรับ type safety

### Pattern Summary

| Pattern | ใช้เมื่อ |
|---------|---------|
| Basic type-safe | APIs ทั่วไป |
| Namespaced | APIs มีหลาย domains |
| Lazy | APIs ที่ initialize ช้าหรือใช้น้อย |
| Event emitter | Reactive events |
| Observable | Reactive state |
| Cleanup | จัดการ lifecycle |

---

*จบ Part 053 - ต่อไป Part 054: Electron with SQLite + ORM*
