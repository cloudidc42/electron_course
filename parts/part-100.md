# Part 100: Final Project — NoteVault

## แอปบันทึกโน้ตครบวงจร (Capstone Project)

ในบทสุดท้ายนี้เราจะสร้าง **NoteVault** — แอปบันทึกโน้ตระดับมืออาชีพที่รวมทุกสิ่งที่เรียนมาทั้ง 100 บท

---

## สถาปัตยกรรมของ NoteVault

```
NoteVault
├── src/
│   ├── main/
│   │   ├── index.ts              ← app entry, window management
│   │   ├── database.ts           ← SQLite + better-sqlite3
│   │   ├── handlers/
│   │   │   ├── noteHandlers.ts   ← CRUD IPC handlers
│   │   │   └── tagHandlers.ts    ← Tag management
│   │   ├── updater.ts            ← electron-updater
│   │   └── menu.ts               ← native menu
│   ├── preload/
│   │   └── index.ts              ← contextBridge API
│   ├── renderer/
│   │   ├── App.tsx
│   │   ├── store/
│   │   │   └── notes.ts          ← Zustand store
│   │   ├── components/
│   │   │   ├── NoteList.tsx
│   │   │   ├── NoteEditor.tsx
│   │   │   └── TagBar.tsx
│   │   └── hooks/
│   │       ├── useNotes.ts
│   │       └── useSearch.ts
│   └── shared/
│       └── types.ts              ← shared TypeScript types
├── tests/
│   ├── unit/
│   └── e2e/
└── electron-builder.json5
```

---

## 1. Shared Types

```typescript
// src/shared/types.ts

export interface Note {
  id: number
  title: string
  content: string
  tags: string[]
  pinned: boolean
  createdAt: string   // ISO 8601
  updatedAt: string
}

export interface Tag {
  name: string
  color: string
  count: number
}

export interface NoteFilter {
  search?: string
  tags?: string[]
  pinned?: boolean
}

export interface NoteSort {
  field: 'updatedAt' | 'createdAt' | 'title'
  direction: 'asc' | 'desc'
}

export interface IPCChannels {
  // Notes
  'note:list':   { req: { filter?: NoteFilter; sort?: NoteSort }; res: Note[] }
  'note:get':    { req: { id: number };                           res: Note | null }
  'note:create': { req: Omit<Note, 'id' | 'createdAt' | 'updatedAt'>; res: Note }
  'note:update': { req: { id: number } & Partial<Note>;          res: Note }
  'note:delete': { req: { id: number };                          res: void }
  'note:search': { req: { query: string };                       res: Note[] }

  // Tags
  'tag:list':    { req: {};                                       res: Tag[] }
  'tag:rename':  { req: { from: string; to: string };            res: void }
  'tag:delete':  { req: { name: string };                        res: void }
}
```

---

## 2. Database Layer

```typescript
// src/main/database.ts
import Database from 'better-sqlite3'
import * as path from 'path'
import { app } from 'electron'
import type { Note, NoteFilter, NoteSort } from '../shared/types'

export class NoteDatabase {
  private db: Database.Database

  constructor() {
    const dbPath = path.join(app.getPath('userData'), 'notevault.db')
    this.db = new Database(dbPath)
    this.migrate()
  }

  private migrate(): void {
    this.db.exec(`
      PRAGMA journal_mode = WAL;
      PRAGMA foreign_keys = ON;

      CREATE TABLE IF NOT EXISTS notes (
        id         INTEGER PRIMARY KEY AUTOINCREMENT,
        title      TEXT    NOT NULL DEFAULT '',
        content    TEXT    NOT NULL DEFAULT '',
        pinned     INTEGER NOT NULL DEFAULT 0,
        created_at TEXT    NOT NULL DEFAULT (datetime('now')),
        updated_at TEXT    NOT NULL DEFAULT (datetime('now'))
      );

      CREATE TABLE IF NOT EXISTS tags (
        id   INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT    NOT NULL UNIQUE
      );

      CREATE TABLE IF NOT EXISTS note_tags (
        note_id INTEGER NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
        tag_id  INTEGER NOT NULL REFERENCES tags(id)  ON DELETE CASCADE,
        PRIMARY KEY (note_id, tag_id)
      );

      CREATE VIRTUAL TABLE IF NOT EXISTS notes_fts
        USING fts5(title, content, content=notes, content_rowid=id);

      CREATE TRIGGER IF NOT EXISTS notes_ai AFTER INSERT ON notes BEGIN
        INSERT INTO notes_fts(rowid, title, content)
          VALUES (new.id, new.title, new.content);
      END;

      CREATE TRIGGER IF NOT EXISTS notes_au AFTER UPDATE ON notes BEGIN
        INSERT INTO notes_fts(notes_fts, rowid, title, content)
          VALUES ('delete', old.id, old.title, old.content);
        INSERT INTO notes_fts(rowid, title, content)
          VALUES (new.id, new.title, new.content);
      END;

      CREATE TRIGGER IF NOT EXISTS notes_ad AFTER DELETE ON notes BEGIN
        INSERT INTO notes_fts(notes_fts, rowid, title, content)
          VALUES ('delete', old.id, old.title, old.content);
      END;
    `)
  }

  private rowToNote(row: Record<string, unknown>): Note {
    const tagNames = this.db.prepare(`
      SELECT t.name FROM tags t
      JOIN note_tags nt ON nt.tag_id = t.id
      WHERE nt.note_id = ?
    `).all(row['id'] as number) as Array<{ name: string }>

    return {
      id:        row['id'] as number,
      title:     row['title'] as string,
      content:   row['content'] as string,
      tags:      tagNames.map(t => t.name),
      pinned:    Boolean(row['pinned']),
      createdAt: row['created_at'] as string,
      updatedAt: row['updated_at'] as string
    }
  }

  listNotes(filter?: NoteFilter, sort?: NoteSort): Note[] {
    const conditions: string[] = ['1=1']
    const params: unknown[] = []

    if (filter?.pinned !== undefined) {
      conditions.push('n.pinned = ?')
      params.push(filter.pinned ? 1 : 0)
    }

    if (filter?.tags && filter.tags.length > 0) {
      const placeholders = filter.tags.map(() => '?').join(', ')
      conditions.push(`n.id IN (
        SELECT nt.note_id FROM note_tags nt
        JOIN tags t ON t.id = nt.tag_id
        WHERE t.name IN (${placeholders})
        GROUP BY nt.note_id
        HAVING COUNT(*) = ${filter.tags.length}
      )`)
      params.push(...filter.tags)
    }

    const sortField = sort?.field === 'title' ? 'n.title' : `n.${sort?.field ?? 'updated_at'}`
    const sortDir   = sort?.direction ?? 'desc'

    const rows = this.db.prepare(`
      SELECT n.* FROM notes n
      WHERE ${conditions.join(' AND ')}
      ORDER BY n.pinned DESC, ${sortField} ${sortDir}
    `).all(...params) as Array<Record<string, unknown>>

    return rows.map(r => this.rowToNote(r))
  }

  getNote(id: number): Note | null {
    const row = this.db.prepare('SELECT * FROM notes WHERE id = ?').get(id) as Record<string, unknown> | undefined
    return row ? this.rowToNote(row) : null
  }

  searchNotes(query: string): Note[] {
    const rows = this.db.prepare(`
      SELECT n.* FROM notes n
      JOIN notes_fts f ON f.rowid = n.id
      WHERE notes_fts MATCH ?
      ORDER BY rank
    `).all(query + '*') as Array<Record<string, unknown>>

    return rows.map(r => this.rowToNote(r))
  }

  createNote(data: Omit<Note, 'id' | 'createdAt' | 'updatedAt'>): Note {
    const result = this.db.prepare(`
      INSERT INTO notes (title, content, pinned)
      VALUES (?, ?, ?)
    `).run(data.title, data.content, data.pinned ? 1 : 0)

    const id = result.lastInsertRowid as number
    this.syncTags(id, data.tags)
    return this.getNote(id)!
  }

  updateNote(id: number, data: Partial<Note>): Note {
    const fields: string[] = []
    const params: unknown[] = []

    if (data.title   !== undefined) { fields.push('title = ?');   params.push(data.title) }
    if (data.content !== undefined) { fields.push('content = ?'); params.push(data.content) }
    if (data.pinned  !== undefined) { fields.push('pinned = ?');  params.push(data.pinned ? 1 : 0) }

    if (fields.length > 0) {
      fields.push("updated_at = datetime('now')")
      this.db.prepare(`UPDATE notes SET ${fields.join(', ')} WHERE id = ?`).run(...params, id)
    }

    if (data.tags !== undefined) {
      this.syncTags(id, data.tags)
    }

    return this.getNote(id)!
  }

  deleteNote(id: number): void {
    this.db.prepare('DELETE FROM notes WHERE id = ?').run(id)
  }

  private syncTags(noteId: number, tagNames: string[]): void {
    this.db.prepare('DELETE FROM note_tags WHERE note_id = ?').run(noteId)

    for (const name of tagNames) {
      this.db.prepare('INSERT OR IGNORE INTO tags (name) VALUES (?)').run(name)
      const tag = this.db.prepare('SELECT id FROM tags WHERE name = ?').get(name) as { id: number }
      this.db.prepare('INSERT OR IGNORE INTO note_tags (note_id, tag_id) VALUES (?, ?)').run(noteId, tag.id)
    }
  }

  listTags(): Array<{ name: string; color: string; count: number }> {
    return this.db.prepare(`
      SELECT t.name, COUNT(nt.note_id) as count
      FROM tags t
      LEFT JOIN note_tags nt ON nt.tag_id = t.id
      GROUP BY t.id
      ORDER BY count DESC, t.name
    `).all() as Array<{ name: string; color: string; count: number }>
  }

  close(): void {
    this.db.close()
  }
}

// Singleton
let _db: NoteDatabase | null = null
export function getDatabase(): NoteDatabase {
  _db ??= new NoteDatabase()
  return _db
}
```

---

## 3. IPC Handlers

```typescript
// src/main/handlers/noteHandlers.ts
import { ipcMain } from 'electron'
import { getDatabase } from '../database'
import type { Note, NoteFilter, NoteSort } from '../../shared/types'

export function registerNoteHandlers(): void {
  const db = getDatabase()

  ipcMain.handle('note:list', (_, { filter, sort }: { filter?: NoteFilter; sort?: NoteSort }) => {
    return db.listNotes(filter, sort)
  })

  ipcMain.handle('note:get', (_, { id }: { id: number }) => {
    return db.getNote(id)
  })

  ipcMain.handle('note:search', (_, { query }: { query: string }) => {
    if (!query.trim()) return db.listNotes()
    return db.searchNotes(query)
  })

  ipcMain.handle('note:create', (_, data: Omit<Note, 'id' | 'createdAt' | 'updatedAt'>) => {
    return db.createNote(data)
  })

  ipcMain.handle('note:update', (_, { id, ...data }: { id: number } & Partial<Note>) => {
    return db.updateNote(id, data)
  })

  ipcMain.handle('note:delete', (_, { id }: { id: number }) => {
    db.deleteNote(id)
  })

  ipcMain.handle('tag:list', () => {
    return db.listTags()
  })
}
```

---

## 4. Preload Bridge

```typescript
// src/preload/index.ts
import { contextBridge, ipcRenderer } from 'electron'

const api = {
  invoke: <T>(channel: string, args?: unknown): Promise<T> =>
    ipcRenderer.invoke(channel, args ?? {}),

  on: (channel: string, callback: (data: unknown) => void) => {
    const listener = (_: Electron.IpcRendererEvent, data: unknown) => callback(data)
    ipcRenderer.on(channel, listener)
    return () => ipcRenderer.removeListener(channel, listener)
  },

  send: (channel: string, data?: unknown) => {
    ipcRenderer.send(channel, data)
  }
}

contextBridge.exposeInMainWorld('electronAPI', api)

declare global {
  interface Window {
    electronAPI: typeof api
  }
}
```

---

## 5. Zustand Store

```typescript
// src/renderer/store/notes.ts
import { create } from 'zustand'
import type { Note, NoteFilter, NoteSort } from '../../shared/types'

interface NotesStore {
  notes:          Note[]
  selectedId:     number | null
  filter:         NoteFilter
  sort:           NoteSort
  searchQuery:    string
  isLoading:      boolean

  loadNotes:      () => Promise<void>
  selectNote:     (id: number | null) => void
  createNote:     () => Promise<Note>
  updateNote:     (id: number, data: Partial<Note>) => Promise<void>
  deleteNote:     (id: number) => Promise<void>
  setFilter:      (f: Partial<NoteFilter>) => void
  setSort:        (s: NoteSort) => void
  setSearch:      (q: string) => void
}

export const useNotesStore = create<NotesStore>((set, get) => ({
  notes:       [],
  selectedId:  null,
  filter:      {},
  sort:        { field: 'updatedAt', direction: 'desc' },
  searchQuery: '',
  isLoading:   false,

  loadNotes: async () => {
    set({ isLoading: true })
    const { filter, sort, searchQuery } = get()
    try {
      const notes: Note[] = searchQuery
        ? await window.electronAPI.invoke('note:search', { query: searchQuery })
        : await window.electronAPI.invoke('note:list', { filter, sort })
      set({ notes, isLoading: false })
    } catch (err) {
      console.error('Failed to load notes:', err)
      set({ isLoading: false })
    }
  },

  selectNote: (id) => set({ selectedId: id }),

  createNote: async () => {
    const note: Note = await window.electronAPI.invoke('note:create', {
      title:   'โน้ตใหม่',
      content: '',
      tags:    [],
      pinned:  false
    })
    await get().loadNotes()
    set({ selectedId: note.id })
    return note
  },

  updateNote: async (id, data) => {
    await window.electronAPI.invoke('note:update', { id, ...data })
    await get().loadNotes()
  },

  deleteNote: async (id) => {
    await window.electronAPI.invoke('note:delete', { id })
    const { selectedId } = get()
    if (selectedId === id) set({ selectedId: null })
    await get().loadNotes()
  },

  setFilter: (f) => {
    set(s => ({ filter: { ...s.filter, ...f } }))
    get().loadNotes()
  },

  setSort: (sort) => {
    set({ sort })
    get().loadNotes()
  },

  setSearch: (searchQuery) => {
    set({ searchQuery })
    get().loadNotes()
  }
}))
```

---

## 6. React Components

```tsx
// src/renderer/App.tsx
import { useEffect } from 'react'
import { NoteList }   from './components/NoteList'
import { NoteEditor } from './components/NoteEditor'
import { TagBar }     from './components/TagBar'
import { useNotesStore } from './store/notes'
import './App.css'

export function App() {
  const { loadNotes, selectedId } = useNotesStore()

  useEffect(() => {
    loadNotes()
  }, [loadNotes])

  return (
    <div className="app">
      <aside className="sidebar">
        <TagBar />
        <NoteList />
      </aside>
      <main className="content">
        {selectedId
          ? <NoteEditor noteId={selectedId} />
          : <EmptyState />
        }
      </main>
    </div>
  )
}

function EmptyState() {
  const { createNote } = useNotesStore()
  return (
    <div className="empty-state">
      <h2>ยังไม่ได้เลือกโน้ต</h2>
      <p>เลือกโน้ตจากรายการด้านซ้าย หรือสร้างโน้ตใหม่</p>
      <button className="btn-primary" onClick={createNote}>
        + สร้างโน้ตใหม่
      </button>
    </div>
  )
}
```

```tsx
// src/renderer/components/NoteList.tsx
import { useNotesStore } from '../store/notes'
import type { Note } from '../../../shared/types'

export function NoteList() {
  const { notes, selectedId, selectNote, createNote, setSearch, isLoading } = useNotesStore()

  return (
    <div className="note-list">
      <div className="note-list-header">
        <input
          type="search"
          placeholder="ค้นหา..."
          className="search-input"
          onChange={e => setSearch(e.target.value)}
        />
        <button className="btn-icon" title="โน้ตใหม่" onClick={createNote}>+</button>
      </div>

      {isLoading && <div className="loading-indicator">กำลังโหลด...</div>}

      <ul className="note-items">
        {notes.map(note => (
          <NoteItem
            key={note.id}
            note={note}
            isSelected={note.id === selectedId}
            onSelect={() => selectNote(note.id)}
          />
        ))}
        {!isLoading && notes.length === 0 && (
          <li className="empty-notes">ไม่มีโน้ต</li>
        )}
      </ul>
    </div>
  )
}

function NoteItem({ note, isSelected, onSelect }: {
  note: Note; isSelected: boolean; onSelect: () => void
}) {
  const preview = note.content.slice(0, 80).replace(/\n/g, ' ')
  const date    = new Date(note.updatedAt).toLocaleDateString('th-TH')

  return (
    <li
      className={`note-item${isSelected ? ' selected' : ''}${note.pinned ? ' pinned' : ''}`}
      onClick={onSelect}
    >
      {note.pinned && <span className="pin-icon">📌</span>}
      <div className="note-item-title">{note.title || 'ไม่มีชื่อ'}</div>
      <div className="note-item-preview">{preview || 'ว่างเปล่า'}</div>
      <div className="note-item-meta">
        <span className="note-date">{date}</span>
        {note.tags.slice(0, 2).map(tag => (
          <span key={tag} className="tag-chip">{tag}</span>
        ))}
      </div>
    </li>
  )
}
```

```tsx
// src/renderer/components/NoteEditor.tsx
import { useState, useEffect, useRef, useCallback } from 'react'
import { useNotesStore } from '../store/notes'
import type { Note } from '../../../shared/types'

interface NoteEditorProps {
  noteId: number
}

export function NoteEditor({ noteId }: NoteEditorProps) {
  const { notes, updateNote, deleteNote } = useNotesStore()
  const note = notes.find(n => n.id === noteId)

  const [title,   setTitle]   = useState(note?.title   ?? '')
  const [content, setContent] = useState(note?.content ?? '')
  const saveTimer = useRef<NodeJS.Timeout | null>(null)

  // Sync กับ store เมื่อเปลี่ยน note
  useEffect(() => {
    setTitle(note?.title ?? '')
    setContent(note?.content ?? '')
  }, [noteId, note?.title, note?.content])

  // Auto-save ทุก 1 วินาทีหลังพิมพ์หยุด
  const scheduleAutosave = useCallback((updates: Partial<Note>) => {
    if (saveTimer.current) clearTimeout(saveTimer.current)
    saveTimer.current = setTimeout(() => {
      updateNote(noteId, updates)
    }, 1000)
  }, [noteId, updateNote])

  const handleTitleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const val = e.target.value
    setTitle(val)
    scheduleAutosave({ title: val })
  }

  const handleContentChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    const val = e.target.value
    setContent(val)
    scheduleAutosave({ content: val })
  }

  const handleTagAdd = (tag: string) => {
    if (!note) return
    const tags = [...new Set([...note.tags, tag.trim()])]
    updateNote(noteId, { tags })
  }

  const handleTagRemove = (tag: string) => {
    if (!note) return
    const tags = note.tags.filter(t => t !== tag)
    updateNote(noteId, { tags })
  }

  const handleTogglePin = () => {
    if (!note) return
    updateNote(noteId, { pinned: !note.pinned })
  }

  const handleDelete = () => {
    if (confirm(`ลบโน้ต "${note?.title || 'ไม่มีชื่อ'}"?`)) {
      deleteNote(noteId)
    }
  }

  if (!note) return null

  return (
    <div className="note-editor">
      <div className="editor-toolbar">
        <button
          className={`btn-icon${note.pinned ? ' active' : ''}`}
          title={note.pinned ? 'เลิก pin' : 'Pin โน้ต'}
          onClick={handleTogglePin}
        >
          📌
        </button>
        <span className="editor-save-status">บันทึกอัตโนมัติ</span>
        <button className="btn-icon danger" title="ลบโน้ต" onClick={handleDelete}>
          🗑️
        </button>
      </div>

      <input
        className="editor-title"
        placeholder="ชื่อโน้ต"
        value={title}
        onChange={handleTitleChange}
      />

      <TagEditor
        tags={note.tags}
        onAdd={handleTagAdd}
        onRemove={handleTagRemove}
      />

      <textarea
        className="editor-content"
        placeholder="เริ่มพิมพ์โน้ตที่นี่..."
        value={content}
        onChange={handleContentChange}
      />
    </div>
  )
}

function TagEditor({
  tags, onAdd, onRemove
}: { tags: string[]; onAdd: (t: string) => void; onRemove: (t: string) => void }) {
  const [input, setInput] = useState('')

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && input.trim()) {
      onAdd(input.trim())
      setInput('')
    }
  }

  return (
    <div className="tag-editor">
      {tags.map(tag => (
        <span key={tag} className="tag-chip editable">
          {tag}
          <button onClick={() => onRemove(tag)}>×</button>
        </span>
      ))}
      <input
        className="tag-input"
        placeholder="+ แท็ก"
        value={input}
        onChange={e => setInput(e.target.value)}
        onKeyDown={handleKeyDown}
      />
    </div>
  )
}
```

---

## 7. Main Entry

```typescript
// src/main/index.ts
import { app, BrowserWindow } from 'electron'
import * as path from 'path'
import { registerNoteHandlers } from './handlers/noteHandlers'
import { buildMenuTemplate }    from './menu'

let mainWindow: BrowserWindow | null = null

function createWindow(): BrowserWindow {
  const win = new BrowserWindow({
    width: 1100,
    height: 700,
    minWidth: 700,
    minHeight: 500,
    titleBarStyle: process.platform === 'darwin' ? 'hiddenInset' : 'default',
    webPreferences: {
      preload: path.join(__dirname, '../preload/index.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true
    }
  })

  if (process.env.NODE_ENV === 'development') {
    win.loadURL('http://localhost:5173')
    win.webContents.openDevTools()
  } else {
    win.loadFile(path.join(__dirname, '../renderer/index.html'))
  }

  return win
}

app.whenReady().then(() => {
  registerNoteHandlers()

  mainWindow = createWindow()

  const menu = buildMenuTemplate(mainWindow)
  require('electron').Menu.setApplicationMenu(menu)

  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      mainWindow = createWindow()
    }
  })
})

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()
})
```

---

## 8. Tests

```typescript
// tests/unit/database.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest'
import { NoteDatabase } from '../../src/main/database'
import * as path from 'path'
import * as fs from 'fs'
import * as os from 'os'

describe('NoteDatabase', () => {
  let db: NoteDatabase
  let tmpDir: string

  beforeEach(() => {
    tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'notevault-test-'))
    process.env.ELECTRON_USER_DATA = tmpDir
    db = new NoteDatabase()
  })

  afterEach(() => {
    db.close()
    fs.rmSync(tmpDir, { recursive: true })
  })

  it('creates a note', () => {
    const note = db.createNote({
      title: 'Test Note', content: 'Hello', tags: ['work'], pinned: false
    })
    expect(note.id).toBeGreaterThan(0)
    expect(note.title).toBe('Test Note')
    expect(note.tags).toContain('work')
  })

  it('lists notes', () => {
    db.createNote({ title: 'A', content: '', tags: [], pinned: false })
    db.createNote({ title: 'B', content: '', tags: [], pinned: false })
    const notes = db.listNotes()
    expect(notes).toHaveLength(2)
  })

  it('searches notes with FTS', () => {
    db.createNote({ title: 'Meeting', content: 'Discuss budget', tags: [], pinned: false })
    db.createNote({ title: 'Todo',    content: 'Buy groceries',  tags: [], pinned: false })
    const results = db.searchNotes('budget')
    expect(results).toHaveLength(1)
    expect(results[0].title).toBe('Meeting')
  })

  it('updates a note', () => {
    const note = db.createNote({ title: 'Old', content: '', tags: [], pinned: false })
    const updated = db.updateNote(note.id, { title: 'New' })
    expect(updated.title).toBe('New')
  })

  it('deletes a note', () => {
    const note = db.createNote({ title: 'Delete me', content: '', tags: [], pinned: false })
    db.deleteNote(note.id)
    expect(db.getNote(note.id)).toBeNull()
  })

  it('filters by tag', () => {
    db.createNote({ title: 'A', content: '', tags: ['work'],     pinned: false })
    db.createNote({ title: 'B', content: '', tags: ['personal'], pinned: false })
    const results = db.listNotes({ tags: ['work'] })
    expect(results).toHaveLength(1)
    expect(results[0].title).toBe('A')
  })
})
```

```typescript
// tests/e2e/app.spec.ts
import { test, expect } from '@playwright/test'
import { _electron as electron } from 'playwright'
import * as path from 'path'

test.describe('NoteVault E2E', () => {
  test('launches and shows empty state', async () => {
    const app = await electron.launch({
      args: [path.join(__dirname, '../../dist/main/index.js')],
      env: { ...process.env, NODE_ENV: 'test' }
    })

    const win = await app.firstWindow()
    await win.waitForLoadState('domcontentloaded')

    await expect(win.locator('.empty-state h2')).toContainText('ยังไม่ได้เลือกโน้ต')

    await app.close()
  })

  test('creates and views a note', async () => {
    const app = await electron.launch({
      args: [path.join(__dirname, '../../dist/main/index.js')],
      env: { ...process.env, NODE_ENV: 'test' }
    })

    const win = await app.firstWindow()
    await win.waitForLoadState('domcontentloaded')

    await win.click('.btn-primary')
    await win.fill('.editor-title', 'My First Note')
    await win.fill('.editor-content', 'Hello NoteVault!')

    await win.waitForTimeout(1500)

    const titleInList = win.locator('.note-item-title').first()
    await expect(titleInList).toContainText('My First Note')

    await app.close()
  })
})
```

---

## สรุปสิ่งที่นำมาใช้ใน NoteVault

| บทที่เรียน | สิ่งที่ใช้ใน NoteVault |
|-----------|----------------------|
| Part 61-70 | โครงสร้างโปรเจกต์, TypeScript setup, Vite |
| Part 71    | SQLite + better-sqlite3 ใน database.ts |
| Part 74    | FTS5 full-text search |
| Part 79    | Vitest unit tests + Playwright E2E |
| Part 82    | Window management, titleBarStyle |
| Part 83    | CSS variables theme system |
| Part 88    | Atomic write (auto-save) |
| Part 89    | Native menu (menu.ts) |
| Part 90    | contextIsolation, sandbox |
| Part 93    | useNotesStore (Zustand), TagEditor |
| Part 94    | Error handling, process.on uncaughtException |
| Part 96    | electron-updater integration |
| Part 97    | Build optimization, lazy routes |
| Part 98    | JSDoc comments, CLAUDE.md |
| Part 99    | Release pipeline, GitHub Actions |

---

## เส้นทางต่อไปหลังจบคอร์ส

```
ระดับถัดไป:
  → ลองสร้างโปรเจกต์จริงของตัวเอง
  → ส่ง app ขึ้น App Store / GitHub Releases
  → เพิ่ม Collaboration (Part 87) ใน NoteVault
  → สร้าง Plugin system (Part 75) ให้ผู้ใช้ extend

แหล่งเรียนรู้เพิ่มเติม:
  → electron.build (electron-builder docs)
  → electronjs.org/docs (Electron official docs)
  → github.com/electron/electron (Electron source)
  → github.com/sindresorhus/awesome-electron
```

---

**จบคอร์ส Electron.js 100 บท** — ขอให้โชคดีในการสร้างแอปเดสก์ท็อปที่ยอดเยี่ยม!
