# Part 050: Offline-First Architecture
## การออกแบบแอพให้ทำงานได้แม้ไม่มีอินเทอร์เน็ต

---

## 🎯 เป้าหมายของบทเรียนนี้

- Offline-first design principles
- IndexedDB สำหรับ offline storage
- Sync เมื่อ online
- Conflict resolution
- Queue offline actions
- Background sync

---

## 1. Offline-First Architecture

```
┌──────────────────────────────────────┐
│           Electron App               │
│                                      │
│  ┌─────────────┐  ┌───────────────┐ │
│  │  UI Layer   │  │  Sync Engine  │ │
│  └──────┬──────┘  └───────┬───────┘ │
│         │                 │          │
│  ┌──────▼─────────────────▼───────┐ │
│  │         Local Database         │ │
│  │         (IndexedDB/SQLite)      │ │
│  └──────────────────────┬─────────┘ │
│                         │            │
└─────────────────────────┼────────────┘
                          │
             ┌────────────▼────────────┐
             │    Background Sync      │
             │  (when network avail.)  │
             └────────────┬────────────┘
                          │
             ┌────────────▼────────────┐
             │     Remote Server       │
             └─────────────────────────┘
```

---

## 2. Network Status Detection

### src/main/network-monitor.ts

```typescript
import { net, BrowserWindow, powerMonitor } from 'electron';

type NetworkStatus = 'online' | 'offline';

class NetworkMonitor {
  private status: NetworkStatus = 'online';
  private checkInterval: NodeJS.Timeout | null = null;
  private listeners: Array<(status: NetworkStatus) => void> = [];
  
  start(): void {
    // เช็คทันทีตอน start
    this.checkConnectivity();
    
    // เช็คทุก 5 วินาที
    this.checkInterval = setInterval(() => {
      this.checkConnectivity();
    }, 5000);
    
    // เช็คเมื่อ system resume
    powerMonitor.on('resume', () => {
      this.checkConnectivity();
    });
  }
  
  stop(): void {
    if (this.checkInterval) {
      clearInterval(this.checkInterval);
      this.checkInterval = null;
    }
  }
  
  private async checkConnectivity(): Promise<void> {
    const isOnline = net.isOnline();
    
    // ถ้า Electron บอกว่า online ให้ verify ด้วยการ ping จริงๆ
    if (isOnline) {
      try {
        await this.pingServer();
        this.updateStatus('online');
      } catch {
        this.updateStatus('offline');
      }
    } else {
      this.updateStatus('offline');
    }
  }
  
  private async pingServer(): Promise<void> {
    return new Promise((resolve, reject) => {
      const timeoutId = setTimeout(() => reject(new Error('Timeout')), 3000);
      
      net.fetch('https://api.yourapp.com/ping', { method: 'HEAD' })
        .then(() => {
          clearTimeout(timeoutId);
          resolve();
        })
        .catch(() => {
          clearTimeout(timeoutId);
          reject(new Error('Network unavailable'));
        });
    });
  }
  
  private updateStatus(newStatus: NetworkStatus): void {
    if (newStatus !== this.status) {
      const oldStatus = this.status;
      this.status = newStatus;
      
      console.log(`Network status changed: ${oldStatus} -> ${newStatus}`);
      
      // แจ้ง renderer
      BrowserWindow.getAllWindows().forEach(win => {
        win.webContents.send('network:status-changed', { 
          status: newStatus,
          isOnline: newStatus === 'online',
        });
      });
      
      // แจ้ง listeners
      this.listeners.forEach(fn => fn(newStatus));
    }
  }
  
  isOnline(): boolean {
    return this.status === 'online';
  }
  
  onStatusChange(callback: (status: NetworkStatus) => void): () => void {
    this.listeners.push(callback);
    return () => {
      this.listeners = this.listeners.filter(fn => fn !== callback);
    };
  }
}

export const networkMonitor = new NetworkMonitor();
```

---

## 3. IndexedDB Helper (Renderer)

### src/renderer/db/offline-db.ts

```typescript
const DB_NAME = 'app-offline-db';
const DB_VERSION = 1;

interface StoreSchema {
  notes: {
    key: string;
    value: {
      id: string;
      title: string;
      content: string;
      createdAt: string;
      updatedAt: string;
      syncStatus: 'synced' | 'pending' | 'conflict';
      localVersion: number;
      serverVersion?: number;
    };
    indexes: { 'by-sync-status': string; 'by-updated': string };
  };
  syncQueue: {
    key: number;
    value: {
      id?: number;
      action: 'create' | 'update' | 'delete';
      entityType: string;
      entityId: string;
      data: any;
      timestamp: string;
      retryCount: number;
      error?: string;
    };
    indexes: { 'by-entity': string; 'by-timestamp': string };
  };
  metadata: {
    key: string;
    value: any;
  };
}

class OfflineDatabase {
  private db: IDBDatabase | null = null;
  
  async open(): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(DB_NAME, DB_VERSION);
      
      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;
        this.setupSchema(db);
      };
      
      request.onsuccess = (event) => {
        this.db = (event.target as IDBOpenDBRequest).result;
        
        // Handle connection errors
        this.db.onerror = (error) => {
          console.error('Database error:', error);
        };
        
        resolve();
      };
      
      request.onerror = (event) => {
        reject((event.target as IDBOpenDBRequest).error);
      };
    });
  }
  
  private setupSchema(db: IDBDatabase): void {
    // Notes store
    if (!db.objectStoreNames.contains('notes')) {
      const notesStore = db.createObjectStore('notes', { keyPath: 'id' });
      notesStore.createIndex('by-sync-status', 'syncStatus', { unique: false });
      notesStore.createIndex('by-updated', 'updatedAt', { unique: false });
    }
    
    // Sync queue
    if (!db.objectStoreNames.contains('syncQueue')) {
      const syncStore = db.createObjectStore('syncQueue', { 
        keyPath: 'id', 
        autoIncrement: true 
      });
      syncStore.createIndex('by-entity', ['entityType', 'entityId'], { unique: false });
      syncStore.createIndex('by-timestamp', 'timestamp', { unique: false });
    }
    
    // Metadata
    if (!db.objectStoreNames.contains('metadata')) {
      db.createObjectStore('metadata', { keyPath: 'key' });
    }
  }
  
  // Generic get
  async get<T>(storeName: string, key: string | number): Promise<T | undefined> {
    return this.transaction(storeName, 'readonly', store => 
      new Promise((resolve, reject) => {
        const request = store.get(key);
        request.onsuccess = () => resolve(request.result);
        request.onerror = () => reject(request.error);
      })
    );
  }
  
  // Generic put
  async put<T>(storeName: string, value: T): Promise<void> {
    return this.transaction(storeName, 'readwrite', store =>
      new Promise((resolve, reject) => {
        const request = store.put(value);
        request.onsuccess = () => resolve();
        request.onerror = () => reject(request.error);
      })
    );
  }
  
  // Generic delete
  async delete(storeName: string, key: string | number): Promise<void> {
    return this.transaction(storeName, 'readwrite', store =>
      new Promise((resolve, reject) => {
        const request = store.delete(key);
        request.onsuccess = () => resolve();
        request.onerror = () => reject(request.error);
      })
    );
  }
  
  // Get all
  async getAll<T>(storeName: string, indexName?: string, query?: IDBKeyRange): Promise<T[]> {
    return this.transaction(storeName, 'readonly', store => {
      const target = indexName ? store.index(indexName) : store;
      
      return new Promise((resolve, reject) => {
        const request = target.getAll(query);
        request.onsuccess = () => resolve(request.result);
        request.onerror = () => reject(request.error);
      });
    });
  }
  
  // Count
  async count(storeName: string): Promise<number> {
    return this.transaction(storeName, 'readonly', store =>
      new Promise((resolve, reject) => {
        const request = store.count();
        request.onsuccess = () => resolve(request.result);
        request.onerror = () => reject(request.error);
      })
    );
  }
  
  // Transaction helper
  private async transaction<T>(
    storeName: string,
    mode: IDBTransactionMode,
    operation: (store: IDBObjectStore) => Promise<T>
  ): Promise<T> {
    if (!this.db) throw new Error('Database not open');
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction(storeName, mode);
      const store = transaction.objectStore(storeName);
      
      operation(store).then(resolve).catch(reject);
      
      transaction.onerror = () => reject(transaction.error);
    });
  }
}

export const offlineDB = new OfflineDatabase();
```

---

## 4. Sync Engine

### src/renderer/sync/SyncEngine.ts

```typescript
import { offlineDB } from '../db/offline-db';

interface SyncQueueItem {
  id?: number;
  action: 'create' | 'update' | 'delete';
  entityType: string;
  entityId: string;
  data: any;
  timestamp: string;
  retryCount: number;
  error?: string;
}

class SyncEngine {
  private isSyncing = false;
  private maxRetries = 3;
  
  // เพิ่ม action ลง queue
  async queueAction(
    action: 'create' | 'update' | 'delete',
    entityType: string,
    entityId: string,
    data: any
  ): Promise<void> {
    const item: SyncQueueItem = {
      action,
      entityType,
      entityId,
      data,
      timestamp: new Date().toISOString(),
      retryCount: 0,
    };
    
    await offlineDB.put('syncQueue', item);
  }
  
  // Sync ทุก pending items
  async sync(): Promise<{ synced: number; failed: number }> {
    if (this.isSyncing) {
      return { synced: 0, failed: 0 };
    }
    
    this.isSyncing = true;
    let synced = 0;
    let failed = 0;
    
    try {
      const queue = await offlineDB.getAll<SyncQueueItem>('syncQueue');
      
      for (const item of queue) {
        try {
          await this.processItem(item);
          await offlineDB.delete('syncQueue', item.id!);
          synced++;
        } catch (error: any) {
          failed++;
          
          if (item.retryCount >= this.maxRetries) {
            // Mark as failed permanently
            await offlineDB.put('syncQueue', {
              ...item,
              error: error.message,
            });
          } else {
            // Retry later
            await offlineDB.put('syncQueue', {
              ...item,
              retryCount: item.retryCount + 1,
              error: error.message,
            });
          }
        }
      }
    } finally {
      this.isSyncing = false;
    }
    
    return { synced, failed };
  }
  
  private async processItem(item: SyncQueueItem): Promise<void> {
    const { action, entityType, entityId, data } = item;
    
    let url = `https://api.yourapp.com/${entityType}`;
    let method = 'POST';
    
    if (action === 'update') {
      url += `/${entityId}`;
      method = 'PUT';
    } else if (action === 'delete') {
      url += `/${entityId}`;
      method = 'DELETE';
    }
    
    const response = await fetch(url, {
      method,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${await this.getToken()}`,
      },
      body: action !== 'delete' ? JSON.stringify(data) : undefined,
    });
    
    if (!response.ok) {
      throw new Error(`Server error: ${response.status} ${response.statusText}`);
    }
    
    const serverData = action !== 'delete' ? await response.json() : null;
    
    // อัพเดท local data ด้วย server response
    if (serverData) {
      await offlineDB.put(entityType, {
        ...serverData,
        syncStatus: 'synced',
        serverVersion: serverData.version,
      });
    }
  }
  
  private async getToken(): Promise<string> {
    const meta = await offlineDB.get<{ key: string; value: string }>('metadata', 'authToken');
    return meta?.value || '';
  }
  
  // ดึงจำนวน pending items
  async getPendingCount(): Promise<number> {
    return offlineDB.count('syncQueue');
  }
}

export const syncEngine = new SyncEngine();
```

---

## 5. Conflict Resolution

### src/renderer/sync/conflict-resolver.ts

```typescript
interface ConflictData {
  localData: any;
  serverData: any;
  entityType: string;
  entityId: string;
}

type ResolutionStrategy = 'server-wins' | 'local-wins' | 'merge' | 'manual';

class ConflictResolver {
  // ตรวจสอบว่ามี conflict หรือไม่
  hasConflict(local: any, server: any): boolean {
    // ถ้า server version สูงกว่า local version
    return server.version > (local.serverVersion || 0) && local.syncStatus === 'pending';
  }
  
  // แก้ conflict อัตโนมัติ
  async resolve(
    conflict: ConflictData,
    strategy: ResolutionStrategy = 'server-wins'
  ): Promise<any> {
    switch (strategy) {
      case 'server-wins':
        return this.serverWins(conflict);
        
      case 'local-wins':
        return this.localWins(conflict);
        
      case 'merge':
        return this.merge(conflict);
        
      case 'manual':
        return this.requestManualResolution(conflict);
    }
  }
  
  private serverWins(conflict: ConflictData): any {
    return {
      ...conflict.serverData,
      syncStatus: 'synced',
    };
  }
  
  private async localWins(conflict: ConflictData): Promise<any> {
    // บังคับ push local data ไป server
    await fetch(`https://api.yourapp.com/${conflict.entityType}/${conflict.entityId}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...conflict.localData,
        version: conflict.serverData.version, // ใช้ server version
      }),
    });
    
    return {
      ...conflict.localData,
      version: conflict.serverData.version + 1,
      syncStatus: 'synced',
    };
  }
  
  private merge(conflict: ConflictData): any {
    // Three-way merge: base + local changes + server changes
    const base = conflict.localData.baseVersion || {};
    const local = conflict.localData;
    const server = conflict.serverData;
    
    const merged: any = { ...base };
    
    // เอา field ที่ local เปลี่ยนแปลง
    Object.keys(local).forEach(key => {
      if (local[key] !== base[key]) {
        merged[key] = local[key];
      }
    });
    
    // เอา field ที่ server เปลี่ยนแปลง (ถ้า local ไม่ได้เปลี่ยน)
    Object.keys(server).forEach(key => {
      if (server[key] !== base[key] && local[key] === base[key]) {
        merged[key] = server[key];
      }
    });
    
    return { ...merged, syncStatus: 'synced' };
  }
  
  private requestManualResolution(conflict: ConflictData): Promise<any> {
    // แจ้ง user ให้เลือก
    return new Promise((resolve) => {
      window.dispatchEvent(new CustomEvent('sync:conflict', {
        detail: {
          conflict,
          resolve: (choice: 'local' | 'server') => {
            if (choice === 'local') {
              resolve(this.localWins(conflict));
            } else {
              resolve(this.serverWins(conflict));
            }
          },
        },
      }));
    });
  }
}

export const conflictResolver = new ConflictResolver();
```

---

## 6. React Hook สำหรับ Offline Status

### src/renderer/hooks/useOffline.ts

```typescript
import { useState, useEffect, useCallback } from 'react';
import { syncEngine } from '../sync/SyncEngine';

export function useOffline() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  const [pendingSync, setPendingSync] = useState(0);
  const [isSyncing, setIsSyncing] = useState(false);

  useEffect(() => {
    // Browser online/offline events
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    
    // Electron-specific events
    if (window.electronAPI) {
      window.electronAPI.on('network:status-changed', ({ isOnline: online }: any) => {
        setIsOnline(online);
      });
    }
    
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  // Auto-sync เมื่อ online
  useEffect(() => {
    if (isOnline && pendingSync > 0) {
      handleSync();
    }
  }, [isOnline]);

  // อัพเดท pending count
  useEffect(() => {
    const updatePending = async () => {
      const count = await syncEngine.getPendingCount();
      setPendingSync(count);
    };
    
    updatePending();
    const interval = setInterval(updatePending, 5000);
    return () => clearInterval(interval);
  }, []);

  const handleSync = useCallback(async () => {
    if (!isOnline || isSyncing) return;
    
    setIsSyncing(true);
    try {
      const result = await syncEngine.sync();
      console.log(`Sync complete: ${result.synced} synced, ${result.failed} failed`);
      
      const count = await syncEngine.getPendingCount();
      setPendingSync(count);
    } finally {
      setIsSyncing(false);
    }
  }, [isOnline, isSyncing]);

  return {
    isOnline,
    isOffline: !isOnline,
    pendingSync,
    isSyncing,
    sync: handleSync,
  };
}
```

---

## 7. สรุป

### Offline-First Checklist

- [ ] ใช้ IndexedDB หรือ SQLite สำหรับ local storage
- [ ] Sync queue สำหรับ pending operations
- [ ] Conflict resolution strategy
- [ ] Network status monitoring
- [ ] Auto-sync เมื่อกลับ online
- [ ] UI แสดงสถานะ offline/pending sync
- [ ] Test offline scenarios

### Strategies

| Conflict | Strategy |
|---------|---------|
| Simple data | Server wins (ง่ายที่สุด) |
| User-edited data | Manual resolution |
| Non-critical data | Local wins |
| Structured data | Three-way merge |

---

*จบ Part 050 - ต่อไป Part 051: Advanced IPC Patterns*
