# Part 84: Data Sync & Conflict Resolution

## ซิงค์ข้อมูลและจัดการ Conflicts ด้วย CRDTs

ในบทนี้เราจะเรียน CRDTs (Conflict-free Replicated Data Types), PouchDB sync, optimistic updates และ conflict resolution strategies

---

## 1. CRDT Basics: LWW Register

```typescript
// src/shared/crdt/lwwRegister.ts
// Last-Write-Wins Register - ค่าล่าสุดชนะ

export interface LWWValue<T> {
  value: T
  timestamp: number
  nodeId: string
}

export class LWWRegister<T> {
  private state: LWWValue<T>

  constructor(initialValue: T, nodeId: string) {
    this.state = {
      value: initialValue,
      timestamp: Date.now(),
      nodeId
    }
  }

  get(): T {
    return this.state.value
  }

  set(value: T, nodeId: string): void {
    const timestamp = Date.now()
    this.state = { value, timestamp, nodeId }
  }

  // Merge state จาก remote
  merge(remote: LWWValue<T>): void {
    if (
      remote.timestamp > this.state.timestamp ||
      (remote.timestamp === this.state.timestamp && remote.nodeId > this.state.nodeId)
    ) {
      this.state = remote
    }
  }

  toJSON(): LWWValue<T> {
    return { ...this.state }
  }
}
```

---

## 2. Operational Transform สำหรับ Text

```typescript
// src/shared/crdt/otText.ts
// Simplified OT สำหรับ text editing

export type TextOp =
  | { type: 'insert'; pos: number; text: string }
  | { type: 'delete'; pos: number; len: number }
  | { type: 'retain'; len: number }

export function applyOp(text: string, op: TextOp): string {
  switch (op.type) {
    case 'insert':
      return text.slice(0, op.pos) + op.text + text.slice(op.pos)
    case 'delete':
      return text.slice(0, op.pos) + text.slice(op.pos + op.len)
    case 'retain':
      return text
    default:
      return text
  }
}

// Transform op1 against op2 เพื่อทำ concurrent edits ได้
export function transformOp(op1: TextOp, op2: TextOp): TextOp {
  if (op1.type === 'insert' && op2.type === 'insert') {
    if (op2.pos <= op1.pos) {
      return { ...op1, pos: op1.pos + op2.text.length }
    }
    return op1
  }

  if (op1.type === 'delete' && op2.type === 'insert') {
    if (op2.pos <= op1.pos) {
      return { ...op1, pos: op1.pos + op2.text.length }
    }
    return op1
  }

  if (op1.type === 'insert' && op2.type === 'delete') {
    if (op2.pos < op1.pos) {
      return { ...op1, pos: Math.max(op2.pos, op1.pos - op2.len) }
    }
    return op1
  }

  if (op1.type === 'delete' && op2.type === 'delete') {
    if (op2.pos < op1.pos) {
      return { ...op1, pos: op1.pos - Math.min(op2.len, op1.pos - op2.pos) }
    }
    if (op2.pos >= op1.pos + op1.len) {
      return op1
    }
    // Overlapping deletes
    const newPos = op1.pos
    const newLen = Math.max(0, op1.len - (Math.min(op2.pos + op2.len, op1.pos + op1.len) - Math.max(op2.pos, op1.pos)))
    return { type: 'delete', pos: newPos, len: newLen }
  }

  return op1
}
```

---

## 3. PouchDB Sync

```typescript
// src/renderer/sync/pouchSync.ts
import PouchDB from 'pouchdb'

PouchDB.plugin(require('pouchdb-adapter-idb'))
PouchDB.plugin(require('pouchdb-find'))

export interface Note {
  _id: string
  _rev?: string
  title: string
  content: string
  tags: string[]
  createdAt: number
  updatedAt: number
  _deleted?: boolean
}

export interface SyncState {
  status: 'idle' | 'syncing' | 'error' | 'paused'
  progress?: { docsWritten: number; docsPending: number }
  error?: string
  lastSync?: number
}

export class NoteSync {
  private local: PouchDB.Database<Note>
  private remote: PouchDB.Database<Note> | null = null
  private syncHandler: PouchDB.Replication.Sync<Note> | null = null
  private syncListeners: Set<(state: SyncState) => void> = new Set()

  constructor(localName = 'notes-local') {
    this.local = new PouchDB<Note>(localName)
    this.createIndex()
  }

  private async createIndex(): Promise<void> {
    await this.local.createIndex({
      index: { fields: ['updatedAt', 'tags'] }
    })
  }

  // เชื่อมต่อ remote (CouchDB/PouchDB server)
  connectRemote(remoteUrl: string, credentials?: { username: string; password: string }): void {
    this.remote = new PouchDB<Note>(remoteUrl, {
      auth: credentials
    })
    this.startSync()
  }

  private startSync(): void {
    if (!this.remote) return

    this.syncHandler = this.local.sync(this.remote, {
      live: true,        // sync แบบ live
      retry: true,       // retry เมื่อ connection ขาด
      batch_size: 50
    })
      .on('change', (info) => {
        this.notifyListeners({
          status: 'syncing',
          progress: {
            docsWritten: info.change.docs_written,
            docsPending: info.pending ?? 0
          }
        })
      })
      .on('paused', () => {
        this.notifyListeners({ status: 'paused', lastSync: Date.now() })
      })
      .on('active', () => {
        this.notifyListeners({ status: 'syncing' })
      })
      .on('error', (err) => {
        this.notifyListeners({ status: 'error', error: String(err) })
      })
  }

  stopSync(): void {
    this.syncHandler?.cancel()
    this.syncHandler = null
  }

  onSyncStateChange(cb: (state: SyncState) => void): () => void {
    this.syncListeners.add(cb)
    return () => this.syncListeners.delete(cb)
  }

  private notifyListeners(state: SyncState): void {
    this.syncListeners.forEach(cb => cb(state))
  }

  // CRUD operations
  async getNote(id: string): Promise<Note | null> {
    try {
      return await this.local.get(id)
    } catch {
      return null
    }
  }

  async getAllNotes(options?: { tags?: string[]; limit?: number }): Promise<Note[]> {
    const selector: PouchDB.Find.Selector = {
      _deleted: { $exists: false }
    }
    if (options?.tags?.length) {
      selector.tags = { $elemMatch: { $in: options.tags } }
    }

    const result = await this.local.find({
      selector,
      sort: [{ updatedAt: 'desc' }],
      limit: options?.limit ?? 100
    })

    return result.docs
  }

  async saveNote(note: Omit<Note, '_id' | 'createdAt' | 'updatedAt'> & { _id?: string }): Promise<Note> {
    const now = Date.now()

    if (note._id) {
      // Update
      const existing = await this.local.get(note._id).catch(() => null)
      const toSave: Note = {
        ...note as Note,
        _id: note._id,
        _rev: existing?._rev,
        createdAt: existing?.createdAt ?? now,
        updatedAt: now
      }
      const result = await this.local.put(toSave)
      return { ...toSave, _rev: result.rev }
    } else {
      // Create
      const newNote: Note = {
        _id: `note-${now}-${Math.random().toString(36).slice(2)}`,
        ...note as Note,
        createdAt: now,
        updatedAt: now
      }
      const result = await this.local.put(newNote)
      return { ...newNote, _rev: result.rev }
    }
  }

  async deleteNote(id: string): Promise<void> {
    const note = await this.local.get(id)
    await this.local.put({ ...note, _deleted: true })
  }
}
```

---

## 4. Conflict Resolution UI

```tsx
// src/renderer/components/ConflictResolver.tsx
import { useState } from 'react'

interface ConflictVersion {
  rev: string
  content: string
  updatedAt: number
  source: 'local' | 'remote'
}

interface ConflictResolverProps {
  versions: ConflictVersion[]
  onResolve: (chosenRev: string) => void
  onDismiss: () => void
}

export function ConflictResolver({ versions, onResolve, onDismiss }: ConflictResolverProps) {
  const [selected, setSelected] = useState<string | null>(null)
  const [merged, setMerged] = useState('')
  const [mode, setMode] = useState<'pick' | 'merge'>('pick')

  const handleResolve = () => {
    if (mode === 'pick' && selected) {
      onResolve(selected)
    }
    // merge mode: save merged content
  }

  return (
    <div className="conflict-resolver">
      <div className="conflict-header">
        <h3>⚠️ พบ Conflict</h3>
        <p>ข้อมูลนี้ถูกแก้ไขจากหลายอุปกรณ์พร้อมกัน กรุณาเลือกเวอร์ชั่นที่ถูกต้อง</p>
      </div>

      <div className="mode-tabs">
        <button
          className={mode === 'pick' ? 'active' : ''}
          onClick={() => setMode('pick')}
        >
          เลือกเวอร์ชั่น
        </button>
        <button
          className={mode === 'merge' ? 'active' : ''}
          onClick={() => setMode('merge')}
        >
          ผสานด้วยตัวเอง
        </button>
      </div>

      {mode === 'pick' ? (
        <div className="versions">
          {versions.map(v => (
            <div
              key={v.rev}
              className={`version-card ${selected === v.rev ? 'selected' : ''}`}
              onClick={() => setSelected(v.rev)}
            >
              <div className="version-meta">
                <span className={`badge ${v.source}`}>{v.source}</span>
                <span className="time">
                  {new Date(v.updatedAt).toLocaleString('th-TH')}
                </span>
              </div>
              <pre className="version-content">{v.content.substring(0, 200)}</pre>
            </div>
          ))}
        </div>
      ) : (
        <div className="merge-editor">
          <div className="diff-panels">
            {versions.map(v => (
              <div key={v.rev} className="diff-panel">
                <div className="panel-label">{v.source}</div>
                <pre>{v.content}</pre>
              </div>
            ))}
          </div>
          <div className="merged-editor">
            <label>เวอร์ชั่นที่ผสาน:</label>
            <textarea
              value={merged}
              onChange={e => setMerged(e.target.value)}
              rows={10}
            />
          </div>
        </div>
      )}

      <div className="actions">
        <button onClick={onDismiss} className="btn">ข้ามไปก่อน</button>
        <button
          onClick={handleResolve}
          disabled={mode === 'pick' ? !selected : !merged.trim()}
          className="btn-primary"
        >
          แก้ไข Conflict
        </button>
      </div>
    </div>
  )
}
```

---

## 5. Optimistic Update Pattern

```typescript
// src/renderer/hooks/useOptimisticNotes.ts
import { useState, useCallback } from 'react'
import type { Note } from '../sync/pouchSync'

export function useOptimisticNotes(sync: import('../sync/pouchSync').NoteSync) {
  const [notes, setNotes] = useState<Note[]>([])
  const [pendingOps, setPendingOps] = useState<Set<string>>(new Set())

  const updateNote = useCallback(async (id: string, updates: Partial<Note>) => {
    // Optimistic update: แสดงผลก่อน
    setNotes(prev => prev.map(n =>
      n._id === id ? { ...n, ...updates, updatedAt: Date.now() } : n
    ))
    setPendingOps(prev => new Set([...prev, id]))

    try {
      const updated = await sync.saveNote({ ...updates, _id: id } as Note)
      // Replace ด้วยข้อมูลจาก DB (มี _rev)
      setNotes(prev => prev.map(n => n._id === id ? updated : n))
    } catch (error) {
      // Revert optimistic update
      console.error('Failed to save:', error)
      const original = await sync.getNote(id)
      if (original) {
        setNotes(prev => prev.map(n => n._id === id ? original : n))
      }
    } finally {
      setPendingOps(prev => {
        const next = new Set(prev)
        next.delete(id)
        return next
      })
    }
  }, [sync])

  const isPending = (id: string) => pendingOps.has(id)

  return { notes, setNotes, updateNote, isPending }
}
```

---

## สรุป

| แนวคิด | Implementation | เหมาะกับ |
|-------|---------------|---------|
| LWW Register | timestamp comparison | simple values |
| OT Transform | operational transform | text editing |
| PouchDB sync | live replication | offline-first apps |
| Conflict detection | _conflicts field | versioned records |
| Conflict UI | side-by-side picker | user decision |
| Optimistic updates | local-first, sync later | responsive UX |
