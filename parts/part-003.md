# Part 003: First Electron App - Todo List
## สร้างแอพ Todo List ด้วย Electron.js

---

## 🎯 เป้าหมายของบทเรียนนี้

สร้าง Todo List Application ที่มีฟีเจอร์:
- เพิ่ม/ลบ/แก้ไข tasks
- Mark tasks เป็น complete
- บันทึกข้อมูลลง local file
- UI สวยงามด้วย CSS
- IPC Communication ระหว่าง Main และ Renderer

---

## 1. โครงสร้างโปรเจกต์

```
electron-todo/
├── package.json
├── main.js
├── preload.js
├── src/
│   ├── index.html
│   ├── renderer.js
│   └── styles.css
├── data/
│   └── todos.json      (สร้างอัตโนมัติ)
└── assets/
    └── icon.png
```

---

## 2. package.json

```json
{
  "name": "electron-todo",
  "version": "1.0.0",
  "description": "Todo List App built with Electron",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "dev": "NODE_ENV=development electron ."
  },
  "author": "Your Name",
  "license": "MIT",
  "devDependencies": {
    "electron": "^28.0.0"
  }
}
```

---

## 3. main.js - Main Process

```javascript
// main.js
const { app, BrowserWindow, ipcMain, dialog } = require('electron')
const path = require('path')
const fs = require('fs')

// ===== Configuration =====
const DATA_DIR = path.join(app.getPath('userData'), 'data')
const TODOS_FILE = path.join(DATA_DIR, 'todos.json')

// ===== Helper Functions =====

// ตรวจสอบและสร้าง data directory
function ensureDataDir() {
  if (!fs.existsSync(DATA_DIR)) {
    fs.mkdirSync(DATA_DIR, { recursive: true })
  }
}

// อ่านข้อมูล todos จากไฟล์
function readTodos() {
  try {
    ensureDataDir()
    if (fs.existsSync(TODOS_FILE)) {
      const data = fs.readFileSync(TODOS_FILE, 'utf8')
      return JSON.parse(data)
    }
    return []
  } catch (error) {
    console.error('Error reading todos:', error)
    return []
  }
}

// บันทึก todos ลงไฟล์
function saveTodos(todos) {
  try {
    ensureDataDir()
    fs.writeFileSync(TODOS_FILE, JSON.stringify(todos, null, 2), 'utf8')
    return true
  } catch (error) {
    console.error('Error saving todos:', error)
    return false
  }
}

// สร้าง ID ใหม่
function generateId() {
  return Date.now().toString(36) + Math.random().toString(36).substr(2)
}

// ===== Window Management =====

let mainWindow

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 900,
    height: 700,
    minWidth: 600,
    minHeight: 500,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: false,
      contextIsolation: true
    },
    // Window Appearance
    frame: true,
    titleBarStyle: process.platform === 'darwin' ? 'hiddenInset' : 'default',
    backgroundColor: '#1a1a2e',
    show: false  // ซ่อนก่อน จนกว่าจะโหลดเสร็จ
  })

  mainWindow.loadFile(path.join(__dirname, 'src', 'index.html'))

  // แสดง window หลังจากโหลดเสร็จ (ป้องกัน flash)
  mainWindow.once('ready-to-show', () => {
    mainWindow.show()
  })

  // เปิด DevTools ใน development
  if (process.env.NODE_ENV === 'development') {
    mainWindow.webContents.openDevTools()
  }

  mainWindow.on('closed', () => {
    mainWindow = null
  })
}

// ===== IPC Handlers =====

// ดึง todos ทั้งหมด
ipcMain.handle('todos:getAll', async () => {
  return readTodos()
})

// เพิ่ม todo ใหม่
ipcMain.handle('todos:add', async (event, text) => {
  const todos = readTodos()
  const newTodo = {
    id: generateId(),
    text: text.trim(),
    completed: false,
    createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString()
  }
  todos.push(newTodo)
  saveTodos(todos)
  return newTodo
})

// อัพเดท todo
ipcMain.handle('todos:update', async (event, id, updates) => {
  const todos = readTodos()
  const index = todos.findIndex(t => t.id === id)
  
  if (index === -1) {
    throw new Error(`Todo with id ${id} not found`)
  }
  
  todos[index] = {
    ...todos[index],
    ...updates,
    updatedAt: new Date().toISOString()
  }
  
  saveTodos(todos)
  return todos[index]
})

// Toggle completed status
ipcMain.handle('todos:toggle', async (event, id) => {
  const todos = readTodos()
  const index = todos.findIndex(t => t.id === id)
  
  if (index === -1) {
    throw new Error(`Todo with id ${id} not found`)
  }
  
  todos[index].completed = !todos[index].completed
  todos[index].updatedAt = new Date().toISOString()
  
  saveTodos(todos)
  return todos[index]
})

// ลบ todo
ipcMain.handle('todos:delete', async (event, id) => {
  const todos = readTodos()
  const filtered = todos.filter(t => t.id !== id)
  saveTodos(filtered)
  return { success: true, deletedId: id }
})

// ลบ todos ที่ completed ทั้งหมด
ipcMain.handle('todos:clearCompleted', async () => {
  const todos = readTodos()
  const active = todos.filter(t => !t.completed)
  const cleared = todos.length - active.length
  saveTodos(active)
  return { success: true, cleared }
})

// Export todos เป็น JSON
ipcMain.handle('todos:export', async () => {
  const todos = readTodos()
  
  const result = await dialog.showSaveDialog(mainWindow, {
    title: 'Export Todos',
    defaultPath: 'todos-export.json',
    filters: [
      { name: 'JSON Files', extensions: ['json'] },
      { name: 'All Files', extensions: ['*'] }
    ]
  })
  
  if (!result.canceled && result.filePath) {
    fs.writeFileSync(result.filePath, JSON.stringify(todos, null, 2), 'utf8')
    return { success: true, filePath: result.filePath }
  }
  
  return { success: false, canceled: true }
})

// Import todos จาก JSON
ipcMain.handle('todos:import', async () => {
  const result = await dialog.showOpenDialog(mainWindow, {
    title: 'Import Todos',
    filters: [
      { name: 'JSON Files', extensions: ['json'] },
      { name: 'All Files', extensions: ['*'] }
    ],
    properties: ['openFile']
  })
  
  if (!result.canceled && result.filePaths.length > 0) {
    try {
      const data = fs.readFileSync(result.filePaths[0], 'utf8')
      const importedTodos = JSON.parse(data)
      
      if (!Array.isArray(importedTodos)) {
        throw new Error('Invalid file format')
      }
      
      saveTodos(importedTodos)
      return { success: true, count: importedTodos.length }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }
  
  return { success: false, canceled: true }
})

// ===== App Events =====

app.whenReady().then(() => {
  createWindow()
  
  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow()
    }
  })
})

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})
```

---

## 4. preload.js

```javascript
// preload.js
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('todoAPI', {
  // ดึง todos ทั้งหมด
  getAll: () => ipcRenderer.invoke('todos:getAll'),
  
  // เพิ่ม todo ใหม่
  add: (text) => ipcRenderer.invoke('todos:add', text),
  
  // อัพเดท todo
  update: (id, updates) => ipcRenderer.invoke('todos:update', id, updates),
  
  // Toggle completed
  toggle: (id) => ipcRenderer.invoke('todos:toggle', id),
  
  // ลบ todo
  delete: (id) => ipcRenderer.invoke('todos:delete', id),
  
  // ลบ completed todos
  clearCompleted: () => ipcRenderer.invoke('todos:clearCompleted'),
  
  // Export
  export: () => ipcRenderer.invoke('todos:export'),
  
  // Import
  import: () => ipcRenderer.invoke('todos:import'),
  
  // App info
  getVersions: () => ({
    electron: process.versions.electron,
    node: process.versions.node,
    platform: process.platform
  })
})
```

---

## 5. src/index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="Content-Security-Policy" 
        content="default-src 'self'; style-src 'self' 'unsafe-inline'; font-src 'self' data:">
  <title>Electron Todo App</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div id="app">
    <!-- Header -->
    <header class="app-header">
      <div class="header-left">
        <h1>✅ Todo App</h1>
        <span class="version-badge" id="versionBadge">Electron</span>
      </div>
      <div class="header-actions">
        <button class="btn btn-icon" id="exportBtn" title="Export todos">
          📤
        </button>
        <button class="btn btn-icon" id="importBtn" title="Import todos">
          📥
        </button>
      </div>
    </header>

    <!-- Input Section -->
    <section class="input-section">
      <div class="input-wrapper">
        <input 
          type="text" 
          id="todoInput" 
          placeholder="เพิ่ม task ใหม่... (กด Enter เพื่อบันทึก)"
          maxlength="200"
          autocomplete="off"
        />
        <button class="btn btn-add" id="addBtn">
          <span>+</span>
        </button>
      </div>
      <div class="input-hint">
        <span id="charCount">0</span>/200 ตัวอักษร
      </div>
    </section>

    <!-- Filter Section -->
    <section class="filter-section">
      <div class="filter-tabs">
        <button class="filter-tab active" data-filter="all">
          ทั้งหมด (<span id="totalCount">0</span>)
        </button>
        <button class="filter-tab" data-filter="active">
          กำลังทำ (<span id="activeCount">0</span>)
        </button>
        <button class="filter-tab" data-filter="completed">
          เสร็จแล้ว (<span id="completedCount">0</span>)
        </button>
      </div>
      <button class="btn btn-clear" id="clearCompletedBtn" style="display:none">
        🗑️ ลบที่เสร็จแล้ว
      </button>
    </section>

    <!-- Todo List -->
    <section class="todo-section">
      <div id="todoList" class="todo-list">
        <!-- Todos จะถูกสร้างโดย JavaScript -->
      </div>
      
      <!-- Empty State -->
      <div id="emptyState" class="empty-state" style="display:none">
        <div class="empty-icon">📝</div>
        <h3>ยังไม่มี tasks</h3>
        <p>เพิ่ม task แรกของคุณด้านบน!</p>
      </div>
    </section>

    <!-- Stats Footer -->
    <footer class="app-footer">
      <div class="stats">
        <span id="statsText">ไม่มี task</span>
      </div>
      <div class="footer-info">
        <span id="footerVersion"></span>
      </div>
    </footer>
  </div>

  <!-- Edit Modal -->
  <div id="editModal" class="modal" style="display:none">
    <div class="modal-overlay" id="modalOverlay"></div>
    <div class="modal-content">
      <h3>แก้ไข Task</h3>
      <textarea 
        id="editInput" 
        rows="3" 
        maxlength="200"
        placeholder="แก้ไขข้อความ..."
      ></textarea>
      <div class="modal-actions">
        <button class="btn btn-cancel" id="cancelEdit">ยกเลิก</button>
        <button class="btn btn-save" id="saveEdit">บันทึก</button>
      </div>
    </div>
  </div>

  <!-- Notification Toast -->
  <div id="toast" class="toast" style="display:none"></div>

  <script src="renderer.js"></script>
</body>
</html>
```

---

## 6. src/styles.css

```css
/* styles.css */

/* ===== CSS Variables ===== */
:root {
  --bg-primary: #1a1a2e;
  --bg-secondary: #16213e;
  --bg-card: rgba(255, 255, 255, 0.05);
  --bg-card-hover: rgba(255, 255, 255, 0.08);
  --border: rgba(255, 255, 255, 0.1);
  --accent: #e94560;
  --accent-hover: #c73652;
  --success: #4ade80;
  --warning: #fbbf24;
  --text-primary: #e0e0e0;
  --text-secondary: #888;
  --text-muted: #555;
  --radius: 12px;
  --shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  --transition: 0.2s ease;
}

/* ===== Reset & Base ===== */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 
               sans-serif;
  background: var(--bg-primary);
  color: var(--text-primary);
  min-height: 100vh;
  overflow-y: auto;
  -webkit-app-region: no-drag;
}

#app {
  max-width: 700px;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* ===== Header ===== */
.app-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 0;
  margin-bottom: 20px;
  border-bottom: 1px solid var(--border);
  -webkit-app-region: drag;  /* ลาก window ด้วย header */
}

.app-header * {
  -webkit-app-region: no-drag;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.app-header h1 {
  font-size: 1.8rem;
  color: var(--accent);
  font-weight: 700;
}

.version-badge {
  background: var(--bg-card);
  border: 1px solid var(--border);
  padding: 4px 10px;
  border-radius: 20px;
  font-size: 0.75rem;
  color: var(--text-secondary);
}

.header-actions {
  display: flex;
  gap: 8px;
}

/* ===== Buttons ===== */
.btn {
  border: none;
  cursor: pointer;
  transition: all var(--transition);
  font-size: 0.9rem;
}

.btn-icon {
  background: var(--bg-card);
  color: var(--text-primary);
  width: 40px;
  height: 40px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.1rem;
  border: 1px solid var(--border);
}

.btn-icon:hover {
  background: var(--bg-card-hover);
  transform: translateY(-1px);
}

.btn-add {
  background: var(--accent);
  color: white;
  width: 48px;
  height: 48px;
  border-radius: 10px;
  font-size: 1.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.btn-add:hover {
  background: var(--accent-hover);
  transform: scale(1.05);
}

.btn-clear {
  background: transparent;
  color: var(--text-secondary);
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 0.85rem;
  border: 1px solid var(--border);
}

.btn-clear:hover {
  background: rgba(233, 69, 96, 0.1);
  color: var(--accent);
  border-color: var(--accent);
}

/* ===== Input Section ===== */
.input-section {
  margin-bottom: 16px;
}

.input-wrapper {
  display: flex;
  gap: 8px;
  align-items: center;
}

#todoInput {
  flex: 1;
  padding: 14px 18px;
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: 10px;
  color: var(--text-primary);
  font-size: 1rem;
  outline: none;
  transition: border-color var(--transition);
}

#todoInput:focus {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(233, 69, 96, 0.1);
}

#todoInput::placeholder {
  color: var(--text-muted);
}

.input-hint {
  margin-top: 6px;
  font-size: 0.8rem;
  color: var(--text-muted);
  text-align: right;
  padding-right: 56px;
}

/* ===== Filter Section ===== */
.filter-section {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.filter-tabs {
  display: flex;
  gap: 4px;
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 4px;
}

.filter-tab {
  background: transparent;
  border: none;
  color: var(--text-secondary);
  padding: 8px 14px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all var(--transition);
}

.filter-tab.active {
  background: var(--accent);
  color: white;
}

.filter-tab:hover:not(.active) {
  background: var(--bg-card-hover);
  color: var(--text-primary);
}

/* ===== Todo List ===== */
.todo-section {
  flex: 1;
}

.todo-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.todo-item {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding: 16px;
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  transition: all var(--transition);
  animation: slideIn 0.2s ease;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.todo-item:hover {
  background: var(--bg-card-hover);
  border-color: rgba(255, 255, 255, 0.15);
  transform: translateX(2px);
}

.todo-item.completed {
  opacity: 0.6;
}

.todo-checkbox {
  flex-shrink: 0;
  width: 22px;
  height: 22px;
  border: 2px solid var(--border);
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all var(--transition);
  margin-top: 2px;
}

.todo-checkbox:hover {
  border-color: var(--success);
}

.todo-item.completed .todo-checkbox {
  background: var(--success);
  border-color: var(--success);
}

.todo-checkbox::after {
  content: '✓';
  color: white;
  font-size: 0.8rem;
  font-weight: bold;
  opacity: 0;
  transition: opacity var(--transition);
}

.todo-item.completed .todo-checkbox::after {
  opacity: 1;
}

.todo-content {
  flex: 1;
  min-width: 0;
}

.todo-text {
  font-size: 1rem;
  line-height: 1.5;
  word-break: break-word;
  transition: all var(--transition);
}

.todo-item.completed .todo-text {
  text-decoration: line-through;
  color: var(--text-muted);
}

.todo-meta {
  font-size: 0.75rem;
  color: var(--text-muted);
  margin-top: 4px;
}

.todo-actions {
  display: flex;
  gap: 4px;
  opacity: 0;
  transition: opacity var(--transition);
  flex-shrink: 0;
}

.todo-item:hover .todo-actions {
  opacity: 1;
}

.todo-btn {
  background: transparent;
  border: none;
  width: 30px;
  height: 30px;
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.9rem;
  transition: all var(--transition);
  color: var(--text-secondary);
}

.todo-btn:hover {
  background: var(--bg-card-hover);
}

.todo-btn.delete:hover {
  background: rgba(233, 69, 96, 0.2);
  color: var(--accent);
}

.todo-btn.edit:hover {
  background: rgba(251, 191, 36, 0.2);
  color: var(--warning);
}

/* ===== Empty State ===== */
.empty-state {
  text-align: center;
  padding: 60px 20px;
  color: var(--text-muted);
}

.empty-icon {
  font-size: 4rem;
  margin-bottom: 16px;
}

.empty-state h3 {
  font-size: 1.2rem;
  margin-bottom: 8px;
  color: var(--text-secondary);
}

/* ===== Footer ===== */
.app-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 0;
  margin-top: 20px;
  border-top: 1px solid var(--border);
  font-size: 0.8rem;
  color: var(--text-muted);
}

/* ===== Modal ===== */
.modal {
  position: fixed;
  inset: 0;
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

.modal-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(4px);
}

.modal-content {
  position: relative;
  background: #1e2a3a;
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 24px;
  width: 100%;
  max-width: 480px;
  box-shadow: var(--shadow);
  animation: modalIn 0.2s ease;
}

@keyframes modalIn {
  from {
    opacity: 0;
    transform: scale(0.9);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.modal-content h3 {
  margin-bottom: 16px;
  color: var(--accent);
}

#editInput {
  width: 100%;
  padding: 12px;
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: 8px;
  color: var(--text-primary);
  font-size: 1rem;
  resize: vertical;
  outline: none;
  font-family: inherit;
}

#editInput:focus {
  border-color: var(--accent);
}

.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 16px;
}

.btn-cancel {
  background: var(--bg-card);
  color: var(--text-secondary);
  padding: 10px 20px;
  border-radius: 8px;
  border: 1px solid var(--border);
}

.btn-cancel:hover {
  background: var(--bg-card-hover);
}

.btn-save {
  background: var(--accent);
  color: white;
  padding: 10px 20px;
  border-radius: 8px;
}

.btn-save:hover {
  background: var(--accent-hover);
}

/* ===== Toast Notification ===== */
.toast {
  position: fixed;
  bottom: 20px;
  right: 20px;
  padding: 12px 20px;
  border-radius: 8px;
  font-size: 0.9rem;
  z-index: 2000;
  animation: toastIn 0.3s ease;
  max-width: 300px;
}

@keyframes toastIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.toast.success {
  background: #1a4731;
  border: 1px solid var(--success);
  color: var(--success);
}

.toast.error {
  background: #4a1a1a;
  border: 1px solid var(--accent);
  color: var(--accent);
}

.toast.info {
  background: #1a2a4a;
  border: 1px solid #4a8af4;
  color: #4a8af4;
}

/* ===== Scrollbar ===== */
::-webkit-scrollbar {
  width: 6px;
}

::-webkit-scrollbar-track {
  background: transparent;
}

::-webkit-scrollbar-thumb {
  background: var(--border);
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: var(--text-muted);
}
```

---

## 7. src/renderer.js

```javascript
// renderer.js

// ===== State =====
let todos = []
let currentFilter = 'all'
let editingId = null

// ===== DOM References =====
const todoInput = document.getElementById('todoInput')
const addBtn = document.getElementById('addBtn')
const todoList = document.getElementById('todoList')
const emptyState = document.getElementById('emptyState')
const filterTabs = document.querySelectorAll('.filter-tab')
const clearCompletedBtn = document.getElementById('clearCompletedBtn')
const totalCount = document.getElementById('totalCount')
const activeCount = document.getElementById('activeCount')
const completedCount = document.getElementById('completedCount')
const statsText = document.getElementById('statsText')
const charCount = document.getElementById('charCount')
const versionBadge = document.getElementById('versionBadge')
const footerVersion = document.getElementById('footerVersion')
const exportBtn = document.getElementById('exportBtn')
const importBtn = document.getElementById('importBtn')
const editModal = document.getElementById('editModal')
const editInput = document.getElementById('editInput')
const cancelEdit = document.getElementById('cancelEdit')
const saveEdit = document.getElementById('saveEdit')
const modalOverlay = document.getElementById('modalOverlay')
const toast = document.getElementById('toast')

// ===== Initialization =====
async function init() {
  // แสดงข้อมูล version
  const versions = window.todoAPI.getVersions()
  versionBadge.textContent = `Electron v${versions.electron}`
  footerVersion.textContent = `Node ${versions.node} | ${versions.platform}`
  
  // โหลด todos
  await loadTodos()
  
  // Setup event listeners
  setupEventListeners()
}

// ===== Load Todos =====
async function loadTodos() {
  try {
    todos = await window.todoAPI.getAll()
    renderTodos()
    updateStats()
  } catch (error) {
    showToast('เกิดข้อผิดพลาดในการโหลดข้อมูล', 'error')
    console.error('Load error:', error)
  }
}

// ===== Render Todos =====
function renderTodos() {
  const filtered = getFilteredTodos()
  
  todoList.innerHTML = ''
  
  if (filtered.length === 0) {
    emptyState.style.display = 'block'
    todoList.style.display = 'none'
  } else {
    emptyState.style.display = 'none'
    todoList.style.display = 'flex'
    
    filtered.forEach(todo => {
      const item = createTodoElement(todo)
      todoList.appendChild(item)
    })
  }
}

// ===== Create Todo Element =====
function createTodoElement(todo) {
  const div = document.createElement('div')
  div.className = `todo-item ${todo.completed ? 'completed' : ''}`
  div.dataset.id = todo.id
  
  const date = new Date(todo.createdAt)
  const dateStr = date.toLocaleDateString('th-TH', {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  })
  
  div.innerHTML = `
    <div class="todo-checkbox" data-id="${todo.id}"></div>
    <div class="todo-content">
      <div class="todo-text">${escapeHtml(todo.text)}</div>
      <div class="todo-meta">${dateStr}</div>
    </div>
    <div class="todo-actions">
      <button class="todo-btn edit" data-id="${todo.id}" title="แก้ไข">✏️</button>
      <button class="todo-btn delete" data-id="${todo.id}" title="ลบ">🗑️</button>
    </div>
  `
  
  // Checkbox click
  div.querySelector('.todo-checkbox').addEventListener('click', () => {
    toggleTodo(todo.id)
  })
  
  // Edit button
  div.querySelector('.todo-btn.edit').addEventListener('click', () => {
    openEditModal(todo)
  })
  
  // Delete button
  div.querySelector('.todo-btn.delete').addEventListener('click', () => {
    deleteTodo(todo.id)
  })
  
  return div
}

// ===== Get Filtered Todos =====
function getFilteredTodos() {
  switch (currentFilter) {
    case 'active':
      return todos.filter(t => !t.completed)
    case 'completed':
      return todos.filter(t => t.completed)
    default:
      return todos
  }
}

// ===== Update Stats =====
function updateStats() {
  const total = todos.length
  const completed = todos.filter(t => t.completed).length
  const active = total - completed
  
  totalCount.textContent = total
  activeCount.textContent = active
  completedCount.textContent = completed
  
  if (total === 0) {
    statsText.textContent = 'ยังไม่มี task'
  } else {
    statsText.textContent = `${active} task กำลังทำ | ${completed} เสร็จแล้ว`
  }
  
  clearCompletedBtn.style.display = completed > 0 ? 'block' : 'none'
}

// ===== Add Todo =====
async function addTodo() {
  const text = todoInput.value.trim()
  
  if (!text) {
    showToast('กรุณาใส่ข้อความ', 'error')
    todoInput.focus()
    return
  }
  
  if (text.length > 200) {
    showToast('ข้อความยาวเกินไป (สูงสุด 200 ตัวอักษร)', 'error')
    return
  }
  
  try {
    addBtn.disabled = true
    const newTodo = await window.todoAPI.add(text)
    todos.push(newTodo)
    todoInput.value = ''
    charCount.textContent = '0'
    renderTodos()
    updateStats()
    showToast('เพิ่ม task สำเร็จ ✅', 'success')
  } catch (error) {
    showToast('เกิดข้อผิดพลาด', 'error')
    console.error('Add error:', error)
  } finally {
    addBtn.disabled = false
    todoInput.focus()
  }
}

// ===== Toggle Todo =====
async function toggleTodo(id) {
  try {
    const updated = await window.todoAPI.toggle(id)
    const index = todos.findIndex(t => t.id === id)
    if (index !== -1) {
      todos[index] = updated
    }
    renderTodos()
    updateStats()
  } catch (error) {
    showToast('เกิดข้อผิดพลาด', 'error')
    console.error('Toggle error:', error)
  }
}

// ===== Delete Todo =====
async function deleteTodo(id) {
  try {
    await window.todoAPI.delete(id)
    todos = todos.filter(t => t.id !== id)
    renderTodos()
    updateStats()
    showToast('ลบ task สำเร็จ', 'success')
  } catch (error) {
    showToast('เกิดข้อผิดพลาด', 'error')
    console.error('Delete error:', error)
  }
}

// ===== Edit Modal =====
function openEditModal(todo) {
  editingId = todo.id
  editInput.value = todo.text
  editModal.style.display = 'flex'
  editInput.focus()
  editInput.select()
}

function closeEditModal() {
  editingId = null
  editModal.style.display = 'none'
  editInput.value = ''
}

async function saveEditedTodo() {
  const text = editInput.value.trim()
  
  if (!text || !editingId) return
  
  try {
    const updated = await window.todoAPI.update(editingId, { text })
    const index = todos.findIndex(t => t.id === editingId)
    if (index !== -1) {
      todos[index] = updated
    }
    closeEditModal()
    renderTodos()
    showToast('แก้ไขสำเร็จ ✅', 'success')
  } catch (error) {
    showToast('เกิดข้อผิดพลาด', 'error')
    console.error('Update error:', error)
  }
}

// ===== Clear Completed =====
async function clearCompleted() {
  try {
    const result = await window.todoAPI.clearCompleted()
    todos = todos.filter(t => !t.completed)
    renderTodos()
    updateStats()
    showToast(`ลบ ${result.cleared} tasks ที่เสร็จแล้ว`, 'success')
  } catch (error) {
    showToast('เกิดข้อผิดพลาด', 'error')
    console.error('Clear error:', error)
  }
}

// ===== Export / Import =====
async function exportTodos() {
  try {
    const result = await window.todoAPI.export()
    if (result.success) {
      showToast('Export สำเร็จ ✅', 'success')
    } else if (!result.canceled) {
      showToast('Export ไม่สำเร็จ', 'error')
    }
  } catch (error) {
    showToast('เกิดข้อผิดพลาด', 'error')
    console.error('Export error:', error)
  }
}

async function importTodos() {
  try {
    const result = await window.todoAPI.import()
    if (result.success) {
      await loadTodos()
      showToast(`Import ${result.count} tasks สำเร็จ ✅`, 'success')
    } else if (!result.canceled) {
      showToast(`Import ไม่สำเร็จ: ${result.error}`, 'error')
    }
  } catch (error) {
    showToast('เกิดข้อผิดพลาด', 'error')
    console.error('Import error:', error)
  }
}

// ===== Utility Functions =====

function escapeHtml(text) {
  const div = document.createElement('div')
  div.appendChild(document.createTextNode(text))
  return div.innerHTML
}

let toastTimeout = null
function showToast(message, type = 'info') {
  if (toastTimeout) clearTimeout(toastTimeout)
  
  toast.textContent = message
  toast.className = `toast ${type}`
  toast.style.display = 'block'
  
  toastTimeout = setTimeout(() => {
    toast.style.display = 'none'
  }, 3000)
}

// ===== Event Listeners =====
function setupEventListeners() {
  // Add todo
  addBtn.addEventListener('click', addTodo)
  todoInput.addEventListener('keydown', (e) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault()
      addTodo()
    }
  })
  
  // Character counter
  todoInput.addEventListener('input', () => {
    charCount.textContent = todoInput.value.length
  })
  
  // Filter tabs
  filterTabs.forEach(tab => {
    tab.addEventListener('click', () => {
      filterTabs.forEach(t => t.classList.remove('active'))
      tab.classList.add('active')
      currentFilter = tab.dataset.filter
      renderTodos()
    })
  })
  
  // Clear completed
  clearCompletedBtn.addEventListener('click', clearCompleted)
  
  // Export / Import
  exportBtn.addEventListener('click', exportTodos)
  importBtn.addEventListener('click', importTodos)
  
  // Modal
  cancelEdit.addEventListener('click', closeEditModal)
  modalOverlay.addEventListener('click', closeEditModal)
  saveEdit.addEventListener('click', saveEditedTodo)
  editInput.addEventListener('keydown', (e) => {
    if (e.key === 'Enter' && e.ctrlKey) {
      saveEditedTodo()
    }
    if (e.key === 'Escape') {
      closeEditModal()
    }
  })
}

// ===== Start =====
document.addEventListener('DOMContentLoaded', init)
```

---

## 8. รันแอพ

```bash
# รันแอพ
npm start
```

### ฟีเจอร์ที่ทำงานได้:
- ✅ เพิ่ม task ด้วยการพิมพ์แล้วกด Enter
- ✅ Mark task เป็น complete ด้วยการคลิก checkbox
- ✅ แก้ไข task ด้วยปุ่ม ✏️
- ✅ ลบ task ด้วยปุ่ม 🗑️
- ✅ Filter: ทั้งหมด / กำลังทำ / เสร็จแล้ว
- ✅ ลบ tasks ที่เสร็จแล้วทั้งหมด
- ✅ Export/Import เป็น JSON
- ✅ บันทึกข้อมูลอัตโนมัติ (ไม่หายเมื่อปิดแอพ)

---

## 9. สรุปสิ่งที่เรียนรู้

1. **IPC invoke/handle**: การสื่อสารแบบ async ระหว่าง processes
2. **contextBridge**: การ expose APIs อย่างปลอดภัย
3. **File System**: อ่าน/เขียนไฟล์ผ่าน Node.js fs module
4. **app.getPath('userData')**: เก็บข้อมูลใน user folder
5. **Dialog**: เปิด file dialog ด้วย dialog.showSaveDialog/showOpenDialog
6. **show: false + ready-to-show**: ป้องกัน window flash

---

## 🔗 บทถัดไป

➡️ **[Part 004: Main Process & Renderer Process](part-004.md)** - เข้าใจสถาปัตยกรรมเชิงลึก

---

*📅 อัพเดทล่าสุด: 2024 | ระดับ: Beginner | เวลาเรียน: ~90 นาที*
