# Part 30: Drag & Drop ใน Electron

## บทนำ

Drag and Drop เป็น feature ที่ทำให้ Desktop Application รู้สึก native มากขึ้น Electron รองรับทั้ง HTML5 drag events และการ drag ไฟล์จาก OS เข้ามา หรือ drag ไฟล์ออกจากแอปไปยัง OS ในบทนี้จะครอบคลุมทุกรูปแบบของ drag & drop

## HTML5 Drag Events พื้นฐาน

### Draggable Elements

```jsx
// src/components/DraggableItem.jsx
import { useRef, useState } from 'react'

export default function DraggableItem({ item, onDrop }) {
  const [isDragging, setIsDragging] = useState(false)
  const dragRef = useRef(null)

  const handleDragStart = (e) => {
    setIsDragging(true)
    
    // ตั้งค่าข้อมูลที่จะส่งไปกับ drag
    e.dataTransfer.effectAllowed = 'move'
    e.dataTransfer.setData('application/json', JSON.stringify(item))
    e.dataTransfer.setData('text/plain', item.name)
    
    // Custom drag image
    if (dragRef.current) {
      const clone = dragRef.current.cloneNode(true)
      clone.style.position = 'absolute'
      clone.style.top = '-1000px'
      clone.style.opacity = '0.8'
      document.body.appendChild(clone)
      e.dataTransfer.setDragImage(clone, 20, 20)
      
      // Cleanup clone หลังจาก drag เริ่ม
      setTimeout(() => document.body.removeChild(clone), 0)
    }
  }

  const handleDragEnd = (e) => {
    setIsDragging(false)
    
    // ตรวจสอบว่า drop สำเร็จหรือไม่
    if (e.dataTransfer.dropEffect === 'move') {
      console.log('Item was moved successfully')
    }
  }

  return (
    <div
      ref={dragRef}
      className={`draggable-item ${isDragging ? 'dragging' : ''}`}
      draggable="true"
      onDragStart={handleDragStart}
      onDragEnd={handleDragEnd}
    >
      <span className="item-icon">📄</span>
      <span className="item-name">{item.name}</span>
      <span className="item-size">{formatSize(item.size)}</span>
    </div>
  )
}

function formatSize(bytes) {
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
}
```

### Drop Zone Component

```jsx
// src/components/DropZone.jsx
import { useState, useCallback, useRef } from 'react'

export default function DropZone({ onDrop, accept = [], children, className = '' }) {
  const [isDragOver, setIsDragOver] = useState(false)
  const [isDragValid, setIsDragValid] = useState(true)
  const dragCountRef = useRef(0) // ป้องกัน flickering
  
  const isValidDrop = useCallback((e) => {
    if (accept.length === 0) return true
    
    const items = Array.from(e.dataTransfer.items)
    return items.some(item => {
      if (item.kind === 'file') {
        return accept.some(type => {
          if (type.startsWith('.')) {
            // Check extension
            const file = item.getAsFile()
            return file?.name.endsWith(type)
          }
          return item.type.startsWith(type.replace('*', ''))
        })
      }
      return false
    })
  }, [accept])

  const handleDragEnter = useCallback((e) => {
    e.preventDefault()
    e.stopPropagation()
    
    dragCountRef.current++
    
    if (dragCountRef.current === 1) {
      const valid = isValidDrop(e)
      setIsDragOver(true)
      setIsDragValid(valid)
      e.dataTransfer.dropEffect = valid ? 'copy' : 'none'
    }
  }, [isValidDrop])

  const handleDragOver = useCallback((e) => {
    e.preventDefault()
    e.stopPropagation()
    
    const valid = isValidDrop(e)
    e.dataTransfer.dropEffect = valid ? 'copy' : 'none'
  }, [isValidDrop])

  const handleDragLeave = useCallback((e) => {
    e.preventDefault()
    e.stopPropagation()
    
    dragCountRef.current--
    
    if (dragCountRef.current === 0) {
      setIsDragOver(false)
      setIsDragValid(true)
    }
  }, [])

  const handleDrop = useCallback(async (e) => {
    e.preventDefault()
    e.stopPropagation()
    
    dragCountRef.current = 0
    setIsDragOver(false)
    setIsDragValid(true)

    if (!isDragValid) return

    const files = Array.from(e.dataTransfer.files)
    const items = Array.from(e.dataTransfer.items)
    
    // Handle file drops
    if (files.length > 0) {
      const filePaths = files.map(f => f.path) // path available ใน Electron
      const fileInfos = files.map(f => ({
        name: f.name,
        path: f.path,
        size: f.size,
        type: f.type,
        lastModified: f.lastModified
      }))
      
      onDrop?.({ type: 'files', files: fileInfos, paths: filePaths })
      return
    }

    // Handle data transfers
    const jsonData = e.dataTransfer.getData('application/json')
    if (jsonData) {
      try {
        const data = JSON.parse(jsonData)
        onDrop?.({ type: 'data', data })
      } catch {
        onDrop?.({ type: 'text', text: e.dataTransfer.getData('text/plain') })
      }
      return
    }
    
    // Handle URL drops
    const url = e.dataTransfer.getData('text/uri-list')
    if (url) {
      onDrop?.({ type: 'url', url })
    }
  }, [isDragValid, onDrop])

  return (
    <div
      className={`drop-zone ${className} ${isDragOver ? 'drag-over' : ''} ${!isDragValid ? 'drag-invalid' : ''}`}
      onDragEnter={handleDragEnter}
      onDragOver={handleDragOver}
      onDragLeave={handleDragLeave}
      onDrop={handleDrop}
    >
      {isDragOver && (
        <div className="drop-overlay">
          {isDragValid ? (
            <div className="drop-indicator valid">
              <span>📂</span>
              <p>วางไฟล์ที่นี่</p>
            </div>
          ) : (
            <div className="drop-indicator invalid">
              <span>🚫</span>
              <p>ไฟล์ประเภทนี้ไม่รองรับ</p>
            </div>
          )}
        </div>
      )}
      {children}
    </div>
  )
}
```

## Drag Files จาก OS (OS → App)

```jsx
// src/components/FileDropArea.jsx
import { useState, useCallback } from 'react'
import DropZone from './DropZone'

export default function FileDropArea({ onFilesDropped }) {
  const [droppedFiles, setDroppedFiles] = useState([])
  const [isProcessing, setIsProcessing] = useState(false)

  const handleDrop = useCallback(async ({ type, files, paths }) => {
    if (type !== 'files' || !files?.length) return
    
    setIsProcessing(true)
    
    try {
      // อ่านข้อมูลไฟล์
      const processedFiles = await Promise.all(
        files.map(async file => {
          const stats = await window.electronAPI?.invoke('fs:stat', file.path)
          return {
            ...file,
            isDirectory: stats?.isDirectory || false,
            extension: file.name.split('.').pop().toLowerCase()
          }
        })
      )
      
      setDroppedFiles(prev => [...prev, ...processedFiles])
      onFilesDropped?.(processedFiles)
    } catch (error) {
      console.error('Error processing dropped files:', error)
    } finally {
      setIsProcessing(false)
    }
  }, [onFilesDropped])

  const removeFile = (index) => {
    setDroppedFiles(prev => prev.filter((_, i) => i !== index))
  }

  const clearAll = () => setDroppedFiles([])

  return (
    <div className="file-drop-area">
      <DropZone
        onDrop={handleDrop}
        accept={['image/*', 'text/*', '.pdf', '.docx', '.xlsx']}
        className="file-drop-zone"
      >
        {droppedFiles.length === 0 ? (
          <div className="drop-empty">
            <div className="drop-icon">📂</div>
            <h3>ลากไฟล์มาวางที่นี่</h3>
            <p>รองรับ รูปภาพ, PDF, Word, Excel, Text</p>
            <p className="hint">หรือ</p>
            <button 
              onClick={() => window.electronAPI?.dialog.openFile({
                properties: ['openFile', 'multiSelections']
              }).then(result => {
                if (!result.canceled) {
                  handleDrop({ type: 'files', files: result.filePaths.map(p => ({
                    path: p, name: p.split('/').pop(), size: 0
                  }))})
                }
              })}
              className="btn-outline"
            >
              เลือกไฟล์
            </button>
          </div>
        ) : (
          <div className="dropped-files">
            <div className="files-header">
              <h3>{droppedFiles.length} ไฟล์</h3>
              <button onClick={clearAll} className="btn-sm">ล้างทั้งหมด</button>
            </div>
            <div className="files-list">
              {droppedFiles.map((file, i) => (
                <FileItem 
                  key={i} 
                  file={file} 
                  onRemove={() => removeFile(i)} 
                />
              ))}
            </div>
          </div>
        )}
      </DropZone>
      
      {isProcessing && (
        <div className="processing-overlay">
          <div className="spinner" />
          <p>กำลังประมวลผล...</p>
        </div>
      )}
    </div>
  )
}

function FileItem({ file, onRemove }) {
  const getIcon = (ext) => {
    const icons = {
      jpg: '🖼️', jpeg: '🖼️', png: '🖼️', gif: '🖼️', svg: '🖼️',
      pdf: '📑', doc: '📝', docx: '📝', txt: '📄',
      xls: '📊', xlsx: '📊', csv: '📊',
      zip: '🗜️', rar: '🗜️',
      mp4: '🎬', mp3: '🎵', wav: '🎵'
    }
    return icons[ext] || '📄'
  }

  return (
    <div className="file-item">
      <span className="file-icon">{getIcon(file.extension)}</span>
      <div className="file-details">
        <span className="file-name" title={file.path}>{file.name}</span>
        <span className="file-meta">
          {file.isDirectory ? 'โฟลเดอร์' : formatSize(file.size)}
        </span>
      </div>
      <button 
        onClick={onRemove}
        className="file-remove"
        title="ลบออก"
      >×</button>
    </div>
  )
}

function formatSize(bytes) {
  if (!bytes) return 'Unknown'
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
}
```

## Drag Files ออกจาก App (App → OS)

```jsx
// src/components/DraggableFile.jsx
import { useCallback } from 'react'

export default function DraggableFile({ file }) {
  const handleDragStart = useCallback((e) => {
    // สำคัญ: ต้องใช้ startDrag ของ Electron สำหรับการ drag ออกไปยัง OS
    if (!window.electronAPI) return
    
    e.preventDefault()
    
    // เรียก Electron's startDrag
    window.electronAPI.invoke('startDrag', {
      filePath: file.path,
      iconPath: getFileIcon(file)
    })
  }, [file])

  return (
    <div
      className="draggable-file"
      draggable="true"
      onDragStart={handleDragStart}
      title={`ลากไปยัง Desktop หรือโฟลเดอร์อื่น: ${file.name}`}
    >
      <span className="drag-handle">⠿</span>
      <span className="file-icon">📄</span>
      <span className="file-name">{file.name}</span>
    </div>
  )
}

function getFileIcon(file) {
  // Return path ของ icon ที่เหมาะสม
  return '/assets/icons/file.png'
}
```

### Main Process Handler สำหรับ Drag ออก

```javascript
// electron/ipc/dragHandlers.js
const { ipcMain, nativeImage } = require('electron')
const path = require('path')

function setupDragHandlers(mainWindow) {
  ipcMain.handle('startDrag', (event, { filePath, iconPath }) => {
    // สร้าง drag icon
    let icon
    try {
      if (iconPath && iconPath !== '') {
        icon = nativeImage.createFromPath(iconPath)
        if (icon.isEmpty()) {
          icon = createDefaultIcon()
        }
      } else {
        icon = createDefaultIcon()
      }
    } catch {
      icon = createDefaultIcon()
    }

    // เริ่ม drag operation
    event.sender.startDrag({
      file: filePath,
      icon: icon.resize({ width: 32, height: 32 })
    })
    
    return { success: true }
  })

  // Drag หลายไฟล์
  ipcMain.handle('startDragMultiple', (event, { filePaths, iconPath }) => {
    let icon = createDefaultIcon()
    
    event.sender.startDrag({
      files: filePaths,
      icon: icon.resize({ width: 32, height: 32 })
    })
    
    return { success: true }
  })
}

function createDefaultIcon() {
  // สร้าง simple drag icon
  return nativeImage.createFromDataURL(`data:image/png;base64,${getDefaultIconBase64()}`)
}

function getDefaultIconBase64() {
  // 32x32 semi-transparent document icon (Base64)
  // ในความเป็นจริงควรใช้ไฟล์ icon จริงๆ
  return 'iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAYAAABzenr0AAAABmJLR0QA/wD/AP+gvaeTAAABiUlEQVRYhe2XzUrDQBCGv6QWvHgUvHgUPHgUPHgUPHgUPHhQDzn1nkOOOdQYMExmdna3yZ4FD57dHTjv7owzxhiTJEmSJEmSJEl6iogI2A9V9QzYAkfAbX/bAFjVxqS2JdFz'
}

module.exports = { setupDragHandlers }
```

## Drag Between Windows

```javascript
// electron/ipc/windowDragHandlers.js
const { ipcMain, BrowserWindow } = require('electron')

// Shared drag state ระหว่าง windows
let currentDragData = null

function setupWindowDragHandlers() {
  
  // Window A เริ่ม drag
  ipcMain.on('drag:start', (event, data) => {
    currentDragData = {
      ...data,
      sourceWindowId: BrowserWindow.fromWebContents(event.sender).id
    }
    
    // Broadcast ไปยัง windows อื่น
    BrowserWindow.getAllWindows().forEach(win => {
      if (win.webContents.id !== event.sender.id) {
        win.webContents.send('drag:external:start', {
          data: currentDragData,
          sourceWindowId: currentDragData.sourceWindowId
        })
      }
    })
  })

  // Window B รับ drag
  ipcMain.on('drag:drop', (event, position) => {
    if (!currentDragData) return
    
    const targetWindowId = BrowserWindow.fromWebContents(event.sender).id
    
    // แจ้ง source window ว่า drop เสร็จแล้ว
    const sourceWindow = BrowserWindow.fromId(currentDragData.sourceWindowId)
    sourceWindow?.webContents.send('drag:dropped', {
      data: currentDragData,
      targetWindowId,
      position
    })
    
    currentDragData = null
  })

  ipcMain.on('drag:cancel', () => {
    currentDragData = null
    
    BrowserWindow.getAllWindows().forEach(win => {
      win.webContents.send('drag:external:end')
    })
  })
}

module.exports = { setupWindowDragHandlers }
```

## Custom Drag Feedback

```jsx
// src/components/DragGhost.jsx
import { useState, useEffect } from 'react'
import ReactDOM from 'react-dom'

/**
 * Custom drag ghost ที่ follow cursor
 */
export default function DragGhost({ item, position }) {
  if (!item) return null
  
  return ReactDOM.createPortal(
    <div 
      className="drag-ghost"
      style={{
        position: 'fixed',
        left: position.x + 10,
        top: position.y + 10,
        pointerEvents: 'none',
        zIndex: 9999,
        opacity: 0.8,
        transform: 'rotate(2deg)'
      }}
    >
      <div className="ghost-card">
        <span className="ghost-icon">📄</span>
        <span className="ghost-name">{item.name}</span>
      </div>
    </div>,
    document.body
  )
}

// Hook สำหรับ custom drag behavior
export function useCustomDrag(item) {
  const [isDragging, setIsDragging] = useState(false)
  const [ghostPosition, setGhostPosition] = useState({ x: 0, y: 0 })

  useEffect(() => {
    if (!isDragging) return

    const handleMouseMove = (e) => {
      setGhostPosition({ x: e.clientX, y: e.clientY })
    }

    const handleMouseUp = () => {
      setIsDragging(false)
    }

    document.addEventListener('mousemove', handleMouseMove)
    document.addEventListener('mouseup', handleMouseUp)

    return () => {
      document.removeEventListener('mousemove', handleMouseMove)
      document.removeEventListener('mouseup', handleMouseUp)
    }
  }, [isDragging])

  const startDrag = (e) => {
    setIsDragging(true)
    setGhostPosition({ x: e.clientX, y: e.clientY })
  }

  return {
    isDragging,
    ghostPosition,
    startDrag,
    DragGhost: isDragging ? <DragGhost item={item} position={ghostPosition} /> : null
  }
}
```

## Sortable List ด้วย Drag & Drop

```jsx
// src/components/SortableList.jsx
import { useState, useRef, useCallback } from 'react'

export default function SortableList({ items: initialItems, onReorder, renderItem }) {
  const [items, setItems] = useState(initialItems)
  const [draggedIndex, setDraggedIndex] = useState(null)
  const [dragOverIndex, setDragOverIndex] = useState(null)
  const dragNode = useRef(null)

  const handleDragStart = useCallback((e, index) => {
    setDraggedIndex(index)
    dragNode.current = e.target
    
    e.dataTransfer.effectAllowed = 'move'
    e.dataTransfer.setData('text/plain', index.toString())
    
    // เพิ่ม class หลังจาก drag เริ่มต้น (ป้องกัน flickering)
    setTimeout(() => {
      dragNode.current?.classList.add('dragging')
    }, 0)
  }, [])

  const handleDragOver = useCallback((e, index) => {
    e.preventDefault()
    e.dataTransfer.dropEffect = 'move'
    
    if (draggedIndex !== index) {
      setDragOverIndex(index)
    }
  }, [draggedIndex])

  const handleDrop = useCallback((e, index) => {
    e.preventDefault()
    
    if (draggedIndex === null || draggedIndex === index) return

    const newItems = [...items]
    const [removed] = newItems.splice(draggedIndex, 1)
    newItems.splice(index, 0, removed)
    
    setItems(newItems)
    onReorder?.(newItems)
    
    setDraggedIndex(null)
    setDragOverIndex(null)
  }, [items, draggedIndex, onReorder])

  const handleDragEnd = useCallback(() => {
    dragNode.current?.classList.remove('dragging')
    dragNode.current = null
    setDraggedIndex(null)
    setDragOverIndex(null)
  }, [])

  return (
    <ul className="sortable-list">
      {items.map((item, index) => (
        <li
          key={item.id || index}
          className={[
            'sortable-item',
            draggedIndex === index ? 'is-dragging' : '',
            dragOverIndex === index ? 'drag-over' : ''
          ].join(' ')}
          draggable="true"
          onDragStart={e => handleDragStart(e, index)}
          onDragOver={e => handleDragOver(e, index)}
          onDrop={e => handleDrop(e, index)}
          onDragEnd={handleDragEnd}
        >
          <span className="drag-handle" title="ลากเพื่อจัดเรียง">⠿</span>
          {renderItem(item, index)}
        </li>
      ))}
    </ul>
  )
}
```

## CSS สำหรับ Drag & Drop

```css
/* src/styles/dragdrop.css */

/* Drop Zone */
.drop-zone {
  position: relative;
  border: 2px dashed var(--border-color, #e2e8f0);
  border-radius: 12px;
  transition: all 0.2s ease;
  min-height: 200px;
}

.drop-zone.drag-over {
  border-color: #6366f1;
  background: rgba(99, 102, 241, 0.05);
  transform: scale(1.01);
}

.drop-zone.drag-invalid {
  border-color: #ef4444;
  background: rgba(239, 68, 68, 0.05);
}

.drop-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 10px;
  pointer-events: none;
  z-index: 10;
}

.drop-indicator {
  text-align: center;
  padding: 24px;
}

.drop-indicator.valid span {
  font-size: 48px;
  animation: bounce 0.5s infinite alternate;
}

.drop-indicator.invalid span {
  font-size: 48px;
}

@keyframes bounce {
  from { transform: translateY(-4px); }
  to { transform: translateY(4px); }
}

/* Draggable Items */
.draggable-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 16px;
  border-radius: 8px;
  background: var(--bg-secondary);
  cursor: grab;
  user-select: none;
  transition: all 0.15s;
}

.draggable-item:hover {
  background: var(--hover-bg);
}

.draggable-item.dragging {
  opacity: 0.5;
  cursor: grabbing;
}

/* Sortable List */
.sortable-item {
  list-style: none;
  transition: transform 0.15s ease;
}

.sortable-item.drag-over {
  border-top: 2px solid #6366f1;
  transform: translateY(2px);
}

.sortable-item.is-dragging {
  opacity: 0.4;
}

.drag-handle {
  cursor: grab;
  color: var(--text-muted);
  font-size: 18px;
  padding: 0 4px;
  line-height: 1;
}

.drag-handle:active {
  cursor: grabbing;
}

/* Drag Ghost */
.drag-ghost {
  background: white;
  border-radius: 8px;
  padding: 8px 12px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2);
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  max-width: 200px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* File Drop Area */
.file-drop-area {
  position: relative;
}

.drop-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 48px 24px;
  gap: 8px;
}

.drop-icon {
  font-size: 64px;
  margin-bottom: 8px;
}

.file-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 12px;
  border-radius: 8px;
  background: var(--bg-secondary);
  margin-bottom: 6px;
}

.file-name {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 13px;
}

.file-meta {
  font-size: 11px;
  color: var(--text-muted);
}

.file-remove {
  width: 20px;
  height: 20px;
  border: none;
  background: var(--hover-bg);
  border-radius: 50%;
  cursor: pointer;
  font-size: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
}
```

## สรุป

### รูปแบบ Drag & Drop ใน Electron

1. **HTML5 Drag** - ใช้สำหรับ drag elements ภายใน app
2. **File Drop (OS → App)** - User ลากไฟล์จาก OS เข้ามาใน app
3. **File Drag (App → OS)** - User ลากไฟล์จาก app ออกไปยัง OS (ต้องใช้ `startDrag`)
4. **Cross-window Drag** - ลาก content ระหว่าง windows
5. **Sortable Lists** - จัดเรียง items ด้วย drag

### Best Practices

- ใช้ drag counter เพื่อป้องกัน flickering จาก dragenter/dragleave
- แสดง visual feedback ที่ชัดเจนว่า drop ได้หรือไม่
- Validate ประเภทไฟล์ก่อน accept
- ใช้ `startDrag` ของ Electron สำหรับการ drag ออกไปยัง OS
- Handle cleanup ทุกกรณี (dragend, drop cancelled)
