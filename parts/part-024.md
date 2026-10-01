# Part 24: State Management ใน Electron

## บทนำ

การจัดการ State ใน Electron มีความซับซ้อนกว่า web app ทั่วไปเพราะมีหลาย processes (main และ renderer) และอาจมีหลาย windows state จำเป็นต้อง sync ระหว่าง processes และ persist ข้ามการเปิดปิดแอป ในบทนี้จะครอบคลุม electron-store, Redux, Zustand, state sync, undo/redo, และ persistence strategies

## electron-store: Persistent Settings

### การติดตั้งและใช้งานพื้นฐาน

```bash
npm install electron-store
```

```javascript
// electron/store/appStore.js
const Store = require('electron-store')

// Schema-based store ที่มี validation
const schema = {
  theme: {
    type: 'string',
    enum: ['light', 'dark', 'system'],
    default: 'system'
  },
  language: {
    type: 'string',
    default: 'th'
  },
  windowBounds: {
    type: 'object',
    properties: {
      width: { type: 'number', default: 1200 },
      height: { type: 'number', default: 800 },
      x: { type: 'number' },
      y: { type: 'number' }
    },
    default: { width: 1200, height: 800 }
  },
  recentFiles: {
    type: 'array',
    items: { type: 'string' },
    default: []
  },
  preferences: {
    type: 'object',
    properties: {
      autoSave: { type: 'boolean', default: true },
      autoSaveInterval: { type: 'number', default: 30 },
      fontSize: { type: 'number', minimum: 8, maximum: 72, default: 14 },
      notifications: { type: 'boolean', default: true }
    },
    default: {}
  },
  shortcuts: {
    type: 'object',
    default: {}
  }
}

// สร้าง store พร้อม migrations
const appStore = new Store({
  name: 'app-settings',
  schema,
  migrations: {
    '2.0.0': (store) => {
      // migrate จาก version 1.x ไป 2.x
      const oldTheme = store.get('darkMode')
      if (typeof oldTheme === 'boolean') {
        store.set('theme', oldTheme ? 'dark' : 'light')
        store.delete('darkMode')
      }
    },
    '3.0.0': (store) => {
      // รวม preferences เข้าด้วยกัน
      const fontSize = store.get('fontSize')
      if (fontSize) {
        store.set('preferences.fontSize', fontSize)
        store.delete('fontSize')
      }
    }
  },
  // เข้ารหัส store (ต้องการ encryptionKey ที่ random สำหรับ production)
  // encryptionKey: 'your-encryption-key',
  
  // Clear invalid keys
  clearInvalidConfig: true
})

module.exports = appStore
```

### Store Manager Class

```javascript
// electron/store/storeManager.js
const Store = require('electron-store')
const { BrowserWindow, ipcMain } = require('electron')

class StoreManager {
  constructor() {
    this.stores = new Map()
    this.watchers = new Map()
  }

  /**
   * สร้างหรือดึง store
   */
  getStore(name, options = {}) {
    if (!this.stores.has(name)) {
      const store = new Store({ name, ...options })
      this.stores.set(name, store)
    }
    return this.stores.get(name)
  }

  /**
   * Subscribe ให้ renderer process รับทราบการเปลี่ยนแปลง
   */
  watchStore(storeName, callback) {
    const store = this.getStore(storeName)
    const disposer = store.onDidAnyChange((newValue, oldValue) => {
      callback(newValue, oldValue)
    })
    
    this.watchers.set(storeName, disposer)
    return disposer
  }

  /**
   * Broadcast การเปลี่ยนแปลงไปยังทุก windows
   */
  broadcastChange(channel, data) {
    BrowserWindow.getAllWindows().forEach(win => {
      if (!win.isDestroyed()) {
        win.webContents.send(channel, data)
      }
    })
  }

  /**
   * ตั้งค่า IPC handlers
   */
  setupIpcHandlers() {
    const store = this.getStore('app-settings')

    ipcMain.handle('store:get', (event, key, defaultValue) => {
      return store.get(key, defaultValue)
    })

    ipcMain.handle('store:set', (event, key, value) => {
      store.set(key, value)
      
      // Broadcast ไปยัง windows อื่น
      this.broadcastChange('store:changed', { key, value })
      return true
    })

    ipcMain.handle('store:delete', (event, key) => {
      store.delete(key)
      this.broadcastChange('store:changed', { key, value: undefined })
      return true
    })

    ipcMain.handle('store:getAll', () => {
      return store.store
    })

    ipcMain.handle('store:reset', () => {
      store.clear()
      this.broadcastChange('store:reset', {})
      return true
    })

    // Watch สำหรับ specific keys
    ipcMain.handle('store:watch', (event, key) => {
      const disposer = store.onDidChange(key, (newValue, oldValue) => {
        event.sender.send(`store:changed:${key}`, { newValue, oldValue })
      })
      
      // Cleanup เมื่อ window ปิด
      event.sender.once('destroyed', disposer)
      return true
    })
  }
}

module.exports = new StoreManager()
```

## Redux ใน Electron

### ตั้งค่า Redux

```bash
npm install @reduxjs/toolkit react-redux
npm install redux-persist  # สำหรับ persistence
```

```javascript
// src/store/index.js
import { configureStore } from '@reduxjs/toolkit'
import { persistStore, persistReducer } from 'redux-persist'
import storage from 'redux-persist/lib/storage'
import { combineReducers } from 'redux'

import notesSlice from './slices/notesSlice'
import settingsSlice from './slices/settingsSlice'
import uiSlice from './slices/uiSlice'
import historySlice from './slices/historySlice'

// Electron storage adapter
const electronStorageAdapter = {
  getItem: async (key) => {
    if (!window.electronAPI?.store) return null
    const value = await window.electronAPI.store.get(key)
    return value ? JSON.stringify(value) : null
  },
  setItem: async (key, value) => {
    if (!window.electronAPI?.store) return
    await window.electronAPI.store.set(key, JSON.parse(value))
  },
  removeItem: async (key) => {
    if (!window.electronAPI?.store) return
    await window.electronAPI.store.delete(key)
  }
}

// Persist config
const persistConfig = {
  key: 'root',
  storage: window.electronAPI ? electronStorageAdapter : storage,
  whitelist: ['settings', 'notes'], // เฉพาะ slices เหล่านี้ที่ persist
  blacklist: ['ui', 'history']      // slices เหล่านี้ไม่ persist
}

const rootReducer = combineReducers({
  notes: notesSlice,
  settings: settingsSlice,
  ui: uiSlice,
  history: historySlice
})

const persistedReducer = persistReducer(persistConfig, rootReducer)

export const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['persist/PERSIST', 'persist/REHYDRATE']
      }
    }),
  devTools: process.env.NODE_ENV !== 'production'
})

export const persistor = persistStore(store)
```

### Notes Slice

```javascript
// src/store/slices/notesSlice.js
import { createSlice, createAsyncThunk, createEntityAdapter } from '@reduxjs/toolkit'

const notesAdapter = createEntityAdapter({
  sortComparer: (a, b) => {
    // Pin notes ไว้บนสุด จากนั้นเรียงตาม updatedAt
    if (a.isPinned !== b.isPinned) return b.isPinned - a.isPinned
    return new Date(b.updatedAt) - new Date(a.updatedAt)
  }
})

// Async thunks
export const fetchNotes = createAsyncThunk(
  'notes/fetchAll',
  async (userId, { rejectWithValue }) => {
    try {
      return await window.electronAPI.invoke('notes:getAll', userId)
    } catch (error) {
      return rejectWithValue(error.message)
    }
  }
)

export const createNote = createAsyncThunk(
  'notes/create',
  async (noteData, { rejectWithValue }) => {
    try {
      return await window.electronAPI.invoke('notes:create', noteData)
    } catch (error) {
      return rejectWithValue(error.message)
    }
  }
)

export const updateNote = createAsyncThunk(
  'notes/update',
  async ({ id, userId, updates }, { rejectWithValue }) => {
    try {
      return await window.electronAPI.invoke('notes:update', id, userId, updates)
    } catch (error) {
      return rejectWithValue(error.message)
    }
  }
)

export const deleteNote = createAsyncThunk(
  'notes/delete',
  async ({ id, userId }, { rejectWithValue }) => {
    try {
      await window.electronAPI.invoke('notes:delete', id, userId)
      return id
    } catch (error) {
      return rejectWithValue(error.message)
    }
  }
)

export const searchNotes = createAsyncThunk(
  'notes/search',
  async ({ userId, query }, { rejectWithValue }) => {
    try {
      return await window.electronAPI.invoke('notes:search', userId, query)
    } catch (error) {
      return rejectWithValue(error.message)
    }
  }
)

const notesSlice = createSlice({
  name: 'notes',
  initialState: notesAdapter.getInitialState({
    loading: 'idle',
    error: null,
    searchResults: [],
    searchQuery: '',
    activeNoteId: null,
    filter: 'all' // 'all' | 'pinned' | 'archived'
  }),
  reducers: {
    setActiveNote: (state, action) => {
      state.activeNoteId = action.payload
    },
    setFilter: (state, action) => {
      state.filter = action.payload
    },
    setSearchQuery: (state, action) => {
      state.searchQuery = action.payload
      if (!action.payload) {
        state.searchResults = []
      }
    },
    // Optimistic update
    updateNoteLocally: (state, action) => {
      const { id, changes } = action.payload
      notesAdapter.updateOne(state, { id, changes })
    }
  },
  extraReducers: (builder) => {
    builder
      // fetchNotes
      .addCase(fetchNotes.pending, (state) => {
        state.loading = 'pending'
      })
      .addCase(fetchNotes.fulfilled, (state, action) => {
        state.loading = 'idle'
        notesAdapter.setAll(state, action.payload)
      })
      .addCase(fetchNotes.rejected, (state, action) => {
        state.loading = 'idle'
        state.error = action.payload
      })
      
      // createNote
      .addCase(createNote.fulfilled, (state, action) => {
        notesAdapter.addOne(state, action.payload)
        state.activeNoteId = action.payload.id
      })
      
      // updateNote
      .addCase(updateNote.fulfilled, (state, action) => {
        notesAdapter.updateOne(state, {
          id: action.payload.id,
          changes: action.payload
        })
      })
      
      // deleteNote
      .addCase(deleteNote.fulfilled, (state, action) => {
        notesAdapter.removeOne(state, action.payload)
        if (state.activeNoteId === action.payload) {
          state.activeNoteId = null
        }
      })
      
      // searchNotes
      .addCase(searchNotes.fulfilled, (state, action) => {
        state.searchResults = action.payload
      })
  }
})

// Selectors
export const {
  selectAll: selectAllNotes,
  selectById: selectNoteById,
  selectIds: selectNoteIds
} = notesAdapter.getSelectors((state) => state.notes)

export const selectActiveNote = (state) => 
  state.notes.activeNoteId 
    ? selectNoteById(state, state.notes.activeNoteId) 
    : null

export const selectFilteredNotes = (state) => {
  const allNotes = selectAllNotes(state)
  const filter = state.notes.filter
  
  switch (filter) {
    case 'pinned':
      return allNotes.filter(n => n.isPinned && !n.isArchived && !n.deletedAt)
    case 'archived':
      return allNotes.filter(n => n.isArchived && !n.deletedAt)
    case 'all':
    default:
      return allNotes.filter(n => !n.isArchived && !n.deletedAt)
  }
}

export const { setActiveNote, setFilter, setSearchQuery, updateNoteLocally } = notesSlice.actions
export default notesSlice.reducer
```

## Zustand - Lightweight State Management

```javascript
// src/store/useAppStore.js
import { create } from 'zustand'
import { devtools, persist, subscribeWithSelector } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

// Electron persistence middleware
const electronPersist = (config) => (set, get, api) => {
  // โหลดสถานะที่บันทึกไว้
  const loadState = async () => {
    if (!window.electronAPI?.store) return
    const saved = await window.electronAPI.store.get('zustand-state')
    if (saved) {
      set(saved, true) // replace state
    }
  }

  // บันทึกสถานะ
  const saveState = async (state) => {
    if (!window.electronAPI?.store) return
    // ไม่บันทึกทุก fields
    const { ui, loading, error, ...persistableState } = state
    await window.electronAPI.store.set('zustand-state', persistableState)
  }

  const result = config(
    (...args) => {
      set(...args)
      saveState(get())
    },
    get,
    api
  )

  loadState()
  return result
}

// App Store ด้วย Zustand + Immer
const useAppStore = create(
  devtools(
    immer(
      electronPersist((set, get) => ({
        // Notes state
        notes: [],
        activeNoteId: null,
        searchResults: [],
        searchQuery: '',
        notesFilter: 'all',

        // UI state
        ui: {
          sidebarOpen: true,
          isLoading: false,
          notifications: []
        },

        // Settings
        settings: {
          theme: 'system',
          fontSize: 14,
          autoSave: true
        },

        // === Notes Actions ===
        setNotes: (notes) => set((state) => {
          state.notes = notes
        }),

        addNote: (note) => set((state) => {
          state.notes.unshift(note)
          state.activeNoteId = note.id
        }),

        updateNote: (id, updates) => set((state) => {
          const note = state.notes.find(n => n.id === id)
          if (note) Object.assign(note, updates)
        }),

        deleteNote: (id) => set((state) => {
          state.notes = state.notes.filter(n => n.id !== id)
          if (state.activeNoteId === id) {
            state.activeNoteId = state.notes[0]?.id || null
          }
        }),

        setActiveNote: (id) => set((state) => {
          state.activeNoteId = id
        }),

        setFilter: (filter) => set((state) => {
          state.notesFilter = filter
        }),

        setSearchQuery: (query) => set((state) => {
          state.searchQuery = query
        }),

        setSearchResults: (results) => set((state) => {
          state.searchResults = results
        }),

        // === UI Actions ===
        toggleSidebar: () => set((state) => {
          state.ui.sidebarOpen = !state.ui.sidebarOpen
        }),

        setLoading: (loading) => set((state) => {
          state.ui.isLoading = loading
        }),

        addNotification: (notification) => set((state) => {
          const id = Date.now().toString()
          state.ui.notifications.push({
            id,
            type: 'info',
            duration: 3000,
            ...notification
          })
        }),

        removeNotification: (id) => set((state) => {
          state.ui.notifications = state.ui.notifications.filter(n => n.id !== id)
        }),

        // === Settings Actions ===
        updateSettings: (updates) => set((state) => {
          Object.assign(state.settings, updates)
        }),

        // === Computed/Derived ===
        getFilteredNotes: () => {
          const { notes, notesFilter } = get()
          switch (notesFilter) {
            case 'pinned':
              return notes.filter(n => n.isPinned && !n.isArchived && !n.deletedAt)
            case 'archived':
              return notes.filter(n => n.isArchived && !n.deletedAt)
            default:
              return notes.filter(n => !n.isArchived && !n.deletedAt)
          }
        },

        getActiveNote: () => {
          const { notes, activeNoteId } = get()
          return notes.find(n => n.id === activeNoteId) || null
        }
      }))
    ),
    { name: 'AppStore' }
  )
)

export default useAppStore
```

## State Sync ระหว่าง Windows

```javascript
// electron/ipc/stateSyncHandlers.js
const { ipcMain, BrowserWindow } = require('electron')

class StateSyncManager {
  constructor() {
    this.stateStore = new Map()
    this.setupHandlers()
  }

  /**
   * Broadcast state change ไปยังทุก windows ยกเว้น sender
   */
  broadcast(channel, data, excludeWebContentsId = null) {
    BrowserWindow.getAllWindows().forEach(win => {
      if (!win.isDestroyed()) {
        if (win.webContents.id !== excludeWebContentsId) {
          win.webContents.send(channel, data)
        }
      }
    })
  }

  /**
   * ส่งไปยัง window เฉพาะ
   */
  sendToWindow(windowId, channel, data) {
    const win = BrowserWindow.fromId(windowId)
    if (win && !win.isDestroyed()) {
      win.webContents.send(channel, data)
    }
  }

  setupHandlers() {
    // Handler สำหรับ sync state
    ipcMain.handle('state:sync', (event, statePatch) => {
      const senderId = event.sender.id
      
      // อัปเดต central state store
      for (const [key, value] of Object.entries(statePatch)) {
        this.stateStore.set(key, value)
      }
      
      // Broadcast ไปยัง windows อื่น
      this.broadcast('state:update', statePatch, senderId)
      
      return { success: true }
    })

    ipcMain.handle('state:get', (event, key) => {
      if (key) return this.stateStore.get(key)
      return Object.fromEntries(this.stateStore)
    })

    ipcMain.handle('state:subscribe', (event, keys) => {
      // Window นี้ต้องการ receive updates สำหรับ keys เหล่านี้
      const webContentsId = event.sender.id
      
      // ส่งสถานะปัจจุบันให้
      const currentState = {}
      keys.forEach(key => {
        if (this.stateStore.has(key)) {
          currentState[key] = this.stateStore.get(key)
        }
      })
      
      return currentState
    })
  }
}

const stateSyncManager = new StateSyncManager()
module.exports = stateSyncManager
```

### Renderer Side State Sync

```javascript
// src/hooks/useStateSync.js
import { useEffect, useCallback, useRef } from 'react'
import { useDispatch } from 'react-redux'

/**
 * Hook สำหรับ sync state ระหว่าง windows
 */
export function useStateSync(storeKey, dispatcher) {
  const dispatch = useDispatch()
  const isSyncing = useRef(false)

  useEffect(() => {
    if (!window.electronAPI) return

    // Listen for state updates from other windows
    const cleanup = window.electronAPI.on('state:update', (statePatch) => {
      if (statePatch[storeKey] !== undefined && !isSyncing.current) {
        dispatch(dispatcher(statePatch[storeKey]))
      }
    })

    return cleanup
  }, [storeKey, dispatcher, dispatch])

  // Function สำหรับ sync ไปยัง windows อื่น
  const syncState = useCallback(async (value) => {
    if (!window.electronAPI) return
    
    isSyncing.current = true
    try {
      await window.electronAPI.invoke('state:sync', { [storeKey]: value })
    } finally {
      isSyncing.current = false
    }
  }, [storeKey])

  return syncState
}
```

## Undo/Redo Pattern

```javascript
// src/store/slices/historySlice.js
import { createSlice } from '@reduxjs/toolkit'

const MAX_HISTORY = 100

const historySlice = createSlice({
  name: 'history',
  initialState: {
    past: [],        // Array ของ state snapshots ที่ผ่านมา
    future: [],      // Array ของ state snapshots ที่ถูก undo
    canUndo: false,
    canRedo: false
  },
  reducers: {
    pushHistory: (state, action) => {
      state.past.push(action.payload)
      
      // จำกัด history size
      if (state.past.length > MAX_HISTORY) {
        state.past.shift()
      }
      
      // Clear future เมื่อมี action ใหม่
      state.future = []
      state.canUndo = state.past.length > 0
      state.canRedo = false
    },
    
    undo: (state) => {
      if (state.past.length === 0) return
      
      const previous = state.past.pop()
      state.future.unshift(previous)
      
      state.canUndo = state.past.length > 0
      state.canRedo = true
    },
    
    redo: (state) => {
      if (state.future.length === 0) return
      
      const next = state.future.shift()
      state.past.push(next)
      
      state.canUndo = true
      state.canRedo = state.future.length > 0
    },
    
    clearHistory: (state) => {
      state.past = []
      state.future = []
      state.canUndo = false
      state.canRedo = false
    }
  }
})

export const { pushHistory, undo, redo, clearHistory } = historySlice.actions
export default historySlice.reducer
```

```javascript
// src/hooks/useUndoRedo.js
import { useDispatch, useSelector } from 'react-redux'
import { useCallback, useEffect } from 'react'
import { undo, redo, pushHistory } from '../store/slices/historySlice'

/**
 * Hook สำหรับ Undo/Redo ใน text editor
 */
export function useUndoRedo(noteId) {
  const dispatch = useDispatch()
  const { canUndo, canRedo } = useSelector(state => state.history)

  // Handle keyboard shortcuts
  useEffect(() => {
    const handleKeyDown = (e) => {
      if ((e.ctrlKey || e.metaKey) && e.key === 'z') {
        e.preventDefault()
        if (e.shiftKey) {
          handleRedo()
        } else {
          handleUndo()
        }
      } else if ((e.ctrlKey || e.metaKey) && e.key === 'y') {
        e.preventDefault()
        handleRedo()
      }
    }

    window.addEventListener('keydown', handleKeyDown)
    return () => window.removeEventListener('keydown', handleKeyDown)
  }, [canUndo, canRedo])

  const handleUndo = useCallback(() => {
    if (!canUndo) return
    dispatch(undo())
    // Apply undo ไปยัง editor
  }, [dispatch, canUndo])

  const handleRedo = useCallback(() => {
    if (!canRedo) return
    dispatch(redo())
    // Apply redo ไปยัง editor
  }, [dispatch, canRedo])

  const recordChange = useCallback((snapshot) => {
    dispatch(pushHistory(snapshot))
  }, [dispatch])

  return {
    canUndo,
    canRedo,
    undo: handleUndo,
    redo: handleRedo,
    record: recordChange
  }
}
```

## State Persistence และ Migration

```javascript
// src/store/persistence.js

/**
 * State migration system
 * ช่วย migrate state เมื่อ structure เปลี่ยน
 */
const migrations = {
  1: (state) => {
    // Version 1: Initial state
    return state
  },
  2: (state) => {
    // Version 2: Rename 'darkMode' to 'theme'
    if (state.settings?.darkMode !== undefined) {
      return {
        ...state,
        settings: {
          ...state.settings,
          theme: state.settings.darkMode ? 'dark' : 'light',
          darkMode: undefined
        }
      }
    }
    return state
  },
  3: (state) => {
    // Version 3: Add new fields
    return {
      ...state,
      settings: {
        autoSave: true,
        autoSaveInterval: 30,
        ...state.settings
      }
    }
  }
}

const CURRENT_VERSION = 3

export function migrateState(savedState) {
  if (!savedState) return null

  let state = savedState
  let version = state._version || 1

  while (version < CURRENT_VERSION) {
    const nextVersion = version + 1
    const migration = migrations[nextVersion]
    
    if (migration) {
      console.log(`Migrating state from v${version} to v${nextVersion}`)
      state = migration(state)
      state._version = nextVersion
    }
    
    version = nextVersion
  }

  return state
}

/**
 * Persistence helper สำหรับ Electron
 */
export class ElectronStatePersistence {
  constructor(storeKey = 'app-state') {
    this.storeKey = storeKey
  }

  async save(state) {
    if (!window.electronAPI?.store) return
    
    const persistableState = {
      ...state,
      _version: CURRENT_VERSION,
      _savedAt: new Date().toISOString()
    }
    
    // ไม่บันทึก UI state ที่ temporary
    delete persistableState.ui
    delete persistableState.loading
    delete persistableState.errors
    
    await window.electronAPI.store.set(this.storeKey, persistableState)
  }

  async load() {
    if (!window.electronAPI?.store) return null
    
    const saved = await window.electronAPI.store.get(this.storeKey)
    if (!saved) return null
    
    return migrateState(saved)
  }

  async clear() {
    if (!window.electronAPI?.store) return
    await window.electronAPI.store.delete(this.storeKey)
  }
}
```

## BroadcastChannel สำหรับ Window Sync

```javascript
// src/hooks/useBroadcastChannel.js
import { useEffect, useRef, useCallback } from 'react'

/**
 * Hook สำหรับ BroadcastChannel API
 * ใช้ sync state ระหว่าง multiple windows โดยตรง
 */
export function useBroadcastChannel(channelName, onMessage) {
  const channelRef = useRef(null)
  const onMessageRef = useRef(onMessage)
  onMessageRef.current = onMessage

  useEffect(() => {
    // BroadcastChannel ทำงานได้ใน Electron renderer
    channelRef.current = new BroadcastChannel(channelName)
    
    channelRef.current.onmessage = (event) => {
      onMessageRef.current(event.data)
    }

    return () => {
      channelRef.current?.close()
      channelRef.current = null
    }
  }, [channelName])

  const postMessage = useCallback((data) => {
    channelRef.current?.postMessage(data)
  }, [])

  return { postMessage }
}

// ตัวอย่างการใช้งาน:
// const { postMessage } = useBroadcastChannel('app-state', (data) => {
//   if (data.type === 'NOTE_UPDATED') {
//     dispatch(updateNote(data.note))
//   }
// })
```

## สรุป Best Practices

### เลือก State Management ที่เหมาะสม

1. **electron-store** - สำหรับ persistent settings ง่ายๆ ที่ไม่ซับซ้อน
2. **Redux Toolkit** - สำหรับ app ขนาดใหญ่ที่ต้องการ predictability และ DevTools
3. **Zustand** - สำหรับ app ขนาดกลางที่ต้องการ simplicity

### Sync Strategy

```javascript
// Pattern ที่แนะนำสำหรับ state sync
// Main process เป็น source of truth
// Renderer sync ผ่าน IPC

// 1. Action เกิดขึ้นใน renderer
// 2. ส่ง IPC ไปยัง main process
// 3. Main process อัปเดต state
// 4. Main process broadcast ไปยังทุก windows
// 5. Windows อัปเดต local state
```

### ข้อควรระวัง

- อย่า serialize ข้อมูลขนาดใหญ่ผ่าน IPC บ่อยๆ
- Cache state ใน renderer แทนการ request จาก main ทุกครั้ง
- ใช้ debounce/throttle สำหรับ frequent state updates
- Cleanup event listeners เสมอ
