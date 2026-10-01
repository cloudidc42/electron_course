# Part 63: Project - File Manager

## สร้าง File Manager แบบ Dual-Pane

ในบทนี้เราจะสร้าง File Manager แบบมืออาชีพที่มี dual-pane layout, breadcrumb navigation, file operations, preview panel และอื่นๆ

---

## สถาปัตยกรรมโดยรวม

```
┌────────────────────────────────────────────────────────┐
│  Toolbar: [New Folder] [Copy] [Move] [Delete] [Search] │
├──────────────────────┬─────────────────────────────────┤
│  Left Pane           │  Right Pane                     │
│  ┌─────────────────┐ │  ┌──────────────────────────┐   │
│  │ Breadcrumb nav  │ │  │  Breadcrumb nav           │   │
│  │─────────────────│ │  │──────────────────────────│   │
│  │ 📁 Documents    │ │  │  📄 report.pdf  1.2 MB   │   │
│  │ 📄 resume.docx  │ │  │  📷 photo.jpg   2.4 MB   │   │
│  │ 📷 photo.jpg    │ │  │  📁 Archive/             │   │
│  └─────────────────┘ │  └──────────────────────────┘   │
├──────────────────────┴─────────────────────────────────┤
│  Preview Panel: [ไอคอน/รูปภาพ/PDF preview]            │
├────────────────────────────────────────────────────────┤
│  Status: 3 items selected · 5.2 MB                     │
└────────────────────────────────────────────────────────┘
```

---

## 1. Main Process - File System Operations

```typescript
// src/main/fileSystemHandlers.ts
import { ipcMain, dialog, shell, clipboard } from 'electron'
import {
  readdir, stat, mkdir, rename, copyFile,
  rm, unlink, access, constants
} from 'fs/promises'
import { createReadStream, createWriteStream } from 'fs'
import { join, dirname, basename, extname } from 'path'
import { pipeline } from 'stream/promises'
import os from 'os'

export interface FileEntry {
  name: string
  path: string
  type: 'file' | 'directory' | 'symlink'
  size: number
  modified: number
  created: number
  extension: string
  hidden: boolean
  permissions: { read: boolean; write: boolean; execute: boolean }
}

export function registerFileSystemHandlers(): void {
  // อ่านรายการไฟล์ในโฟลเดอร์
  ipcMain.handle('fs:list', async (_, dirPath: string) => {
    try {
      const entries = await readdir(dirPath, { withFileTypes: true })
      const files: FileEntry[] = []

      for (const entry of entries) {
        const fullPath = join(dirPath, entry.name)
        try {
          const stats = await stat(fullPath)
          const canRead = await access(fullPath, constants.R_OK).then(() => true).catch(() => false)
          const canWrite = await access(fullPath, constants.W_OK).then(() => true).catch(() => false)

          files.push({
            name: entry.name,
            path: fullPath,
            type: entry.isDirectory() ? 'directory' : entry.isSymbolicLink() ? 'symlink' : 'file',
            size: stats.size,
            modified: stats.mtimeMs,
            created: stats.birthtimeMs,
            extension: extname(entry.name).slice(1).toLowerCase(),
            hidden: entry.name.startsWith('.'),
            permissions: { read: canRead, write: canWrite, execute: false }
          })
        } catch {
          // Skip inaccessible files
        }
      }

      return { success: true, files }
    } catch (error) {
      return { success: false, error: (error as Error).message }
    }
  })

  // สร้างโฟลเดอร์ใหม่
  ipcMain.handle('fs:mkdir', async (_, dirPath: string) => {
    try {
      await mkdir(dirPath, { recursive: true })
      return { success: true }
    } catch (error) {
      return { success: false, error: (error as Error).message }
    }
  })

  // เปลี่ยนชื่อ
  ipcMain.handle('fs:rename', async (_, { oldPath, newName }: { oldPath: string; newName: string }) => {
    const newPath = join(dirname(oldPath), newName)
    try {
      await rename(oldPath, newPath)
      return { success: true, newPath }
    } catch (error) {
      return { success: false, error: (error as Error).message }
    }
  })

  // คัดลอกไฟล์/โฟลเดอร์
  ipcMain.handle('fs:copy', async (_, { sources, destination }: { sources: string[]; destination: string }) => {
    const results: { path: string; success: boolean; error?: string }[] = []

    for (const source of sources) {
      const destPath = join(destination, basename(source))
      try {
        const stats = await stat(source)
        if (stats.isDirectory()) {
          await copyDirectory(source, destPath)
        } else {
          await copyFile(source, destPath)
        }
        results.push({ path: source, success: true })
      } catch (error) {
        results.push({ path: source, success: false, error: (error as Error).message })
      }
    }

    return results
  })

  // ย้ายไฟล์
  ipcMain.handle('fs:move', async (_, { sources, destination }: { sources: string[]; destination: string }) => {
    const results: { path: string; success: boolean; newPath?: string; error?: string }[] = []

    for (const source of sources) {
      const destPath = join(destination, basename(source))
      try {
        await rename(source, destPath)
        results.push({ path: source, success: true, newPath: destPath })
      } catch (error) {
        // ถ้าข้าม drive ให้ copy แล้ว delete
        try {
          const stats = await stat(source)
          if (stats.isDirectory()) await copyDirectory(source, destPath)
          else await copyFile(source, destPath)
          await rm(source, { recursive: true })
          results.push({ path: source, success: true, newPath: destPath })
        } catch (err) {
          results.push({ path: source, success: false, error: (err as Error).message })
        }
      }
    }

    return results
  })

  // ลบไฟล์ (ส่งไปถังขยะ)
  ipcMain.handle('fs:delete', async (_, paths: string[]) => {
    const results: { path: string; success: boolean; error?: string }[] = []

    for (const filePath of paths) {
      try {
        await shell.trashItem(filePath)
        results.push({ path: filePath, success: true })
      } catch (error) {
        results.push({ path: filePath, success: false, error: (error as Error).message })
      }
    }

    return results
  })

  // ดึงข้อมูล drives/bookmarks
  ipcMain.handle('fs:get-places', () => {
    const home = os.homedir()
    const places = [
      { name: 'Home', path: home, icon: '🏠' },
      { name: 'Desktop', path: join(home, 'Desktop'), icon: '🖥' },
      { name: 'Documents', path: join(home, 'Documents'), icon: '📄' },
      { name: 'Downloads', path: join(home, 'Downloads'), icon: '⬇️' },
      { name: 'Pictures', path: join(home, 'Pictures'), icon: '🖼' },
      { name: 'Music', path: join(home, 'Music'), icon: '🎵' }
    ]
    return places
  })

  // Search ไฟล์
  ipcMain.handle('fs:search', async (_, { root, query }: { root: string; query: string }) => {
    const results: FileEntry[] = []
    await searchFiles(root, query.toLowerCase(), results, 0)
    return results.slice(0, 200) // จำกัด 200 ผลลัพธ์
  })

  // เปิดใน system file manager
  ipcMain.handle('fs:show-in-finder', async (_, filePath: string) => {
    shell.showItemInFolder(filePath)
  })

  // คัดลอก path
  ipcMain.handle('fs:copy-path', (_, filePath: string) => {
    clipboard.writeText(filePath)
  })

  // ดึงข้อมูลสำหรับ preview
  ipcMain.handle('fs:get-preview', async (_, filePath: string) => {
    const stats = await stat(filePath)
    const ext = extname(filePath).slice(1).toLowerCase()
    
    const textExts = ['txt', 'md', 'js', 'ts', 'py', 'css', 'html', 'json', 'yaml', 'xml', 'csv', 'log']
    const imageExts = ['jpg', 'jpeg', 'png', 'gif', 'webp', 'svg', 'bmp']
    
    if (imageExts.includes(ext)) {
      return { type: 'image', path: `file://${filePath}` }
    }
    
    if (textExts.includes(ext) && stats.size < 1024 * 1024) { // < 1MB
      const { readFile } = await import('fs/promises')
      const content = await readFile(filePath, 'utf-8')
      return { type: 'text', content: content.slice(0, 5000), ext }
    }
    
    return { type: 'info', size: stats.size, modified: stats.mtimeMs }
  })
}

async function copyDirectory(src: string, dest: string): Promise<void> {
  await mkdir(dest, { recursive: true })
  const entries = await readdir(src, { withFileTypes: true })
  
  for (const entry of entries) {
    const srcPath = join(src, entry.name)
    const destPath = join(dest, entry.name)
    
    if (entry.isDirectory()) {
      await copyDirectory(srcPath, destPath)
    } else {
      await copyFile(srcPath, destPath)
    }
  }
}

async function searchFiles(dir: string, query: string, results: FileEntry[], depth: number): Promise<void> {
  if (depth > 5 || results.length >= 200) return
  
  try {
    const entries = await readdir(dir, { withFileTypes: true })
    
    for (const entry of entries) {
      if (entry.name.startsWith('.')) continue
      
      const fullPath = join(dir, entry.name)
      
      if (entry.name.toLowerCase().includes(query)) {
        try {
          const stats = await stat(fullPath)
          results.push({
            name: entry.name,
            path: fullPath,
            type: entry.isDirectory() ? 'directory' : 'file',
            size: stats.size,
            modified: stats.mtimeMs,
            created: stats.birthtimeMs,
            extension: extname(entry.name).slice(1).toLowerCase(),
            hidden: false,
            permissions: { read: true, write: true, execute: false }
          })
        } catch {}
      }
      
      if (entry.isDirectory()) {
        await searchFiles(fullPath, query, results, depth + 1)
      }
    }
  } catch {}
}
```

---

## 2. File Pane Component

```tsx
// src/renderer/components/FilePane.tsx
import { useState, useEffect, useCallback, useRef } from 'react'
import { FileEntry } from '../../main/fileSystemHandlers'
import { BreadcrumbNav } from './BreadcrumbNav'
import { FileGrid } from './FileGrid'
import { FileList } from './FileList'

interface FilePaneProps {
  paneSide: 'left' | 'right'
  isActive: boolean
  onActivate: () => void
  onSelectionChange: (files: FileEntry[]) => void
}

type SortKey = 'name' | 'size' | 'modified' | 'type'
type ViewMode = 'list' | 'grid'

export function FilePane({ paneSide, isActive, onActivate, onSelectionChange }: FilePaneProps) {
  const [currentPath, setCurrentPath] = useState<string>('')
  const [files, setFiles] = useState<FileEntry[]>([])
  const [selected, setSelected] = useState<Set<string>>(new Set())
  const [sortBy, setSortBy] = useState<SortKey>('name')
  const [sortAsc, setSortAsc] = useState(true)
  const [viewMode, setViewMode] = useState<ViewMode>('list')
  const [showHidden, setShowHidden] = useState(false)
  const [loading, setLoading] = useState(false)
  const [filter, setFilter] = useState('')
  const [history, setHistory] = useState<string[]>([])
  const [historyIndex, setHistoryIndex] = useState(-1)
  const lastClickRef = useRef<{ path: string; time: number } | null>(null)

  // โหลดรายการไฟล์
  const loadDirectory = useCallback(async (path: string, addToHistory = true) => {
    setLoading(true)
    setSelected(new Set())
    setFilter('')

    const result = await window.electronAPI.fsList(path)
    if (result.success) {
      setFiles(result.files)
      setCurrentPath(path)

      if (addToHistory) {
        setHistory(h => [...h.slice(0, historyIndex + 1), path])
        setHistoryIndex(i => i + 1)
      }
    }
    setLoading(false)
  }, [historyIndex])

  // โหลด home directory ตอนเริ่มต้น
  useEffect(() => {
    window.electronAPI.fsGetPlaces().then(places => {
      if (places[0]) loadDirectory(places[0].path)
    })
  }, [])

  // Navigate back/forward
  const goBack = () => {
    if (historyIndex > 0) {
      const path = history[historyIndex - 1]
      setHistoryIndex(i => i - 1)
      loadDirectory(path, false)
    }
  }

  const goForward = () => {
    if (historyIndex < history.length - 1) {
      const path = history[historyIndex + 1]
      setHistoryIndex(i => i + 1)
      loadDirectory(path, false)
    }
  }

  const goUp = () => {
    const parent = currentPath.split('/').slice(0, -1).join('/') || '/'
    loadDirectory(parent)
  }

  // เรียงและกรองไฟล์
  const displayFiles = [...files]
    .filter(f => showHidden || !f.hidden)
    .filter(f => !filter || f.name.toLowerCase().includes(filter.toLowerCase()))
    .sort((a, b) => {
      // โฟลเดอร์ก่อน
      if (a.type === 'directory' && b.type !== 'directory') return -1
      if (a.type !== 'directory' && b.type === 'directory') return 1

      let cmp = 0
      switch (sortBy) {
        case 'name': cmp = a.name.localeCompare(b.name); break
        case 'size': cmp = a.size - b.size; break
        case 'modified': cmp = a.modified - b.modified; break
        case 'type': cmp = a.extension.localeCompare(b.extension); break
      }

      return sortAsc ? cmp : -cmp
    })

  // Click handler
  const handleFileClick = (file: FileEntry, e: React.MouseEvent) => {
    onActivate()
    const now = Date.now()

    // Double click
    if (lastClickRef.current?.path === file.path && now - lastClickRef.current.time < 400) {
      if (file.type === 'directory') {
        loadDirectory(file.path)
      } else {
        window.electronAPI.fsShowInFinder(file.path)
      }
      lastClickRef.current = null
      return
    }

    lastClickRef.current = { path: file.path, time: now }

    // Multi-select
    if (e.shiftKey && selected.size > 0) {
      const allPaths = displayFiles.map(f => f.path)
      const lastSelected = [...selected].pop()!
      const lastIdx = allPaths.indexOf(lastSelected)
      const currentIdx = allPaths.indexOf(file.path)
      const [start, end] = [Math.min(lastIdx, currentIdx), Math.max(lastIdx, currentIdx)]
      setSelected(new Set(allPaths.slice(start, end + 1)))
    } else if (e.ctrlKey || e.metaKey) {
      const next = new Set(selected)
      if (next.has(file.path)) next.delete(file.path)
      else next.add(file.path)
      setSelected(next)
    } else {
      setSelected(new Set([file.path]))
    }
  }

  useEffect(() => {
    const selectedFiles = files.filter(f => selected.has(f.path))
    onSelectionChange(selectedFiles)
  }, [selected])

  const handleSort = (key: SortKey) => {
    if (sortBy === key) setSortAsc(a => !a)
    else { setSortBy(key); setSortAsc(true) }
  }

  return (
    <div className={`file-pane ${isActive ? 'active' : ''} ${paneSide}`} onClick={onActivate}>
      {/* Header */}
      <div className="pane-header">
        <div className="nav-controls">
          <button onClick={goBack} disabled={historyIndex <= 0} title="ย้อนกลับ">←</button>
          <button onClick={goForward} disabled={historyIndex >= history.length - 1} title="ไปหน้า">→</button>
          <button onClick={goUp} title="ขึ้นระดับ">↑</button>
        </div>

        <BreadcrumbNav path={currentPath} onNavigate={loadDirectory} />

        <div className="pane-controls">
          <input
            type="text"
            placeholder="กรอง..."
            value={filter}
            onChange={e => setFilter(e.target.value)}
            className="filter-input"
          />
          <button
            className={`icon-btn ${showHidden ? 'active' : ''}`}
            onClick={() => setShowHidden(!showHidden)}
            title="แสดงไฟล์ซ่อน"
          >👁</button>
          <button
            className={`icon-btn ${viewMode === 'grid' ? 'active' : ''}`}
            onClick={() => setViewMode(v => v === 'list' ? 'grid' : 'list')}
            title="สลับมุมมอง"
          >{viewMode === 'list' ? '⊞' : '≡'}</button>
        </div>
      </div>

      {/* Content */}
      <div className="pane-content">
        {loading ? (
          <div className="loading">กำลังโหลด...</div>
        ) : viewMode === 'list' ? (
          <FileList
            files={displayFiles}
            selected={selected}
            onFileClick={handleFileClick}
            onSort={handleSort}
            sortBy={sortBy}
            sortAsc={sortAsc}
          />
        ) : (
          <FileGrid
            files={displayFiles}
            selected={selected}
            onFileClick={handleFileClick}
          />
        )}
      </div>

      {/* Footer */}
      <div className="pane-footer">
        <span>{displayFiles.length} รายการ</span>
        {selected.size > 0 && (
          <span>{selected.size} เลือก</span>
        )}
      </div>
    </div>
  )
}
```

---

## 3. File List Component

```tsx
// src/renderer/components/FileList.tsx
import { FileEntry } from '../../main/fileSystemHandlers'

const FILE_ICONS: Record<string, string> = {
  // Documents
  pdf: '📕', doc: '📘', docx: '📘', xls: '📗', xlsx: '📗', ppt: '📙', pptx: '📙',
  txt: '📄', md: '📝', csv: '📊',
  // Images
  jpg: '🖼', jpeg: '🖼', png: '🖼', gif: '🎞', svg: '🎨', psd: '🎨',
  // Code
  js: '📜', ts: '📜', py: '🐍', rb: '💎', rs: '🦀', go: '🔵',
  html: '🌐', css: '🎨', json: '📋', xml: '📋',
  // Archive
  zip: '📦', tar: '📦', gz: '📦', rar: '📦',
  // Media
  mp3: '🎵', wav: '🎵', mp4: '🎬', mov: '🎬', avi: '🎬',
  // System
  exe: '⚙️', app: '⚙️', dmg: '💿', iso: '💿'
}

function getIcon(file: FileEntry): string {
  if (file.type === 'directory') return '📁'
  return FILE_ICONS[file.extension] || '📄'
}

function formatSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  if (bytes < 1024 * 1024 * 1024) return `${(bytes / 1024 / 1024).toFixed(1)} MB`
  return `${(bytes / 1024 / 1024 / 1024).toFixed(2)} GB`
}

function formatDate(ms: number): string {
  return new Date(ms).toLocaleString('th-TH', {
    year: 'numeric', month: '2-digit', day: '2-digit',
    hour: '2-digit', minute: '2-digit'
  })
}

interface FileListProps {
  files: FileEntry[]
  selected: Set<string>
  onFileClick: (file: FileEntry, e: React.MouseEvent) => void
  onSort: (key: 'name' | 'size' | 'modified' | 'type') => void
  sortBy: string
  sortAsc: boolean
}

export function FileList({ files, selected, onFileClick, onSort, sortBy, sortAsc }: FileListProps) {
  const SortIndicator = ({ col }: { col: string }) => {
    if (sortBy !== col) return <span className="sort-indicator neutral">↕</span>
    return <span className="sort-indicator active">{sortAsc ? '↑' : '↓'}</span>
  }

  return (
    <table className="file-list">
      <thead>
        <tr>
          <th onClick={() => onSort('name')} className="sortable">
            ชื่อ <SortIndicator col="name" />
          </th>
          <th onClick={() => onSort('size')} className="sortable">
            ขนาด <SortIndicator col="size" />
          </th>
          <th onClick={() => onSort('type')} className="sortable">
            ประเภท <SortIndicator col="type" />
          </th>
          <th onClick={() => onSort('modified')} className="sortable">
            แก้ไขล่าสุด <SortIndicator col="modified" />
          </th>
        </tr>
      </thead>
      <tbody>
        {files.length === 0 ? (
          <tr><td colSpan={4} className="empty-state">โฟลเดอร์ว่าง</td></tr>
        ) : (
          files.map(file => (
            <tr
              key={file.path}
              className={`file-row ${selected.has(file.path) ? 'selected' : ''}`}
              onClick={(e) => onFileClick(file, e)}
              onContextMenu={(e) => {
                e.preventDefault()
                // Context menu
              }}
            >
              <td className="file-name">
                <span className="file-icon">{getIcon(file)}</span>
                <span className={file.hidden ? 'hidden-file' : ''}>{file.name}</span>
              </td>
              <td className="file-size">
                {file.type === 'directory' ? '—' : formatSize(file.size)}
              </td>
              <td className="file-type">
                {file.type === 'directory' ? 'โฟลเดอร์' : (file.extension.toUpperCase() || 'ไฟล์')}
              </td>
              <td className="file-date">{formatDate(file.modified)}</td>
            </tr>
          ))
        )}
      </tbody>
    </table>
  )
}
```

---

## 4. Breadcrumb Navigation

```tsx
// src/renderer/components/BreadcrumbNav.tsx
interface BreadcrumbNavProps {
  path: string
  onNavigate: (path: string) => void
}

export function BreadcrumbNav({ path, onNavigate }: BreadcrumbNavProps) {
  const parts = path.split('/').filter(Boolean)
  
  const getParts = () => {
    const result = [{ name: '/', path: '/' }]
    let current = ''
    for (const part of parts) {
      current += '/' + part
      result.push({ name: part, path: current })
    }
    return result
  }

  const breadcrumbs = getParts()

  return (
    <div className="breadcrumb">
      {breadcrumbs.map((crumb, idx) => (
        <span key={crumb.path} className="breadcrumb-item">
          {idx > 0 && <span className="separator">/</span>}
          <button
            className={`crumb ${idx === breadcrumbs.length - 1 ? 'current' : ''}`}
            onClick={() => onNavigate(crumb.path)}
            title={crumb.path}
          >
            {crumb.name}
          </button>
        </span>
      ))}
    </div>
  )
}
```

---

## 5. Preview Panel

```tsx
// src/renderer/components/PreviewPanel.tsx
import { useState, useEffect } from 'react'
import { FileEntry } from '../../main/fileSystemHandlers'

interface PreviewPanelProps {
  file: FileEntry | null
}

interface PreviewData {
  type: 'image' | 'text' | 'info' | 'none'
  path?: string
  content?: string
  ext?: string
  size?: number
  modified?: number
}

export function PreviewPanel({ file }: PreviewPanelProps) {
  const [preview, setPreview] = useState<PreviewData>({ type: 'none' })
  const [loading, setLoading] = useState(false)

  useEffect(() => {
    if (!file || file.type === 'directory') {
      setPreview({ type: 'none' })
      return
    }

    setLoading(true)
    window.electronAPI.fsGetPreview(file.path)
      .then(data => setPreview(data))
      .finally(() => setLoading(false))
  }, [file?.path])

  if (!file) {
    return (
      <div className="preview-panel empty">
        <span>เลือกไฟล์เพื่อดูตัวอย่าง</span>
      </div>
    )
  }

  return (
    <div className="preview-panel">
      <div className="preview-header">
        <strong>{file.name}</strong>
        <span className="preview-size">
          {file.type === 'file' ? formatSize(file.size) : 'โฟลเดอร์'}
        </span>
      </div>

      <div className="preview-content">
        {loading ? (
          <div className="preview-loading">กำลังโหลด...</div>
        ) : preview.type === 'image' ? (
          <img src={preview.path} alt={file.name} className="preview-image" />
        ) : preview.type === 'text' ? (
          <div className="preview-text">
            <pre className={`language-${preview.ext}`}>{preview.content}</pre>
          </div>
        ) : (
          <div className="preview-info">
            <div className="info-icon">
              {file.type === 'directory' ? '📁' : '📄'}
            </div>
            <table className="info-table">
              <tbody>
                <tr><td>ชื่อ:</td><td>{file.name}</td></tr>
                <tr><td>ประเภท:</td><td>{file.extension.toUpperCase() || 'ไฟล์'}</td></tr>
                <tr><td>ขนาด:</td><td>{formatSize(file.size)}</td></tr>
                <tr><td>แก้ไขล่าสุด:</td><td>{new Date(file.modified).toLocaleString('th-TH')}</td></tr>
                <tr><td>Path:</td><td className="path-cell">{file.path}</td></tr>
              </tbody>
            </table>
          </div>
        )}
      </div>
    </div>
  )
}

function formatSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  if (bytes < 1024 * 1024 * 1024) return `${(bytes / 1024 / 1024).toFixed(1)} MB`
  return `${(bytes / 1024 / 1024 / 1024).toFixed(2)} GB`
}
```

---

## 6. App Component หลัก

```tsx
// src/renderer/App.tsx
import { useState } from 'react'
import { FilePane } from './components/FilePane'
import { PreviewPanel } from './components/PreviewPanel'
import { OperationsToolbar } from './components/OperationsToolbar'
import { FileEntry } from '../main/fileSystemHandlers'
import './styles/app.css'

export default function App() {
  const [activePane, setActivePane] = useState<'left' | 'right'>('left')
  const [leftSelection, setLeftSelection] = useState<FileEntry[]>([])
  const [rightSelection, setRightSelection] = useState<FileEntry[]>([])
  const [showPreview, setShowPreview] = useState(true)

  const activeSelection = activePane === 'left' ? leftSelection : rightSelection
  const previewFile = activeSelection.length === 1 ? activeSelection[0] : null

  const handleCopy = async () => {
    if (activeSelection.length === 0) return
    // copy ไปยัง pane อื่น
    alert(`คัดลอก ${activeSelection.length} ไฟล์`)
  }

  const handleMove = async () => {
    if (activeSelection.length === 0) return
    alert(`ย้าย ${activeSelection.length} ไฟล์`)
  }

  const handleDelete = async () => {
    if (activeSelection.length === 0) return
    if (!confirm(`ลบ ${activeSelection.length} รายการ?`)) return
    const paths = activeSelection.map(f => f.path)
    await window.electronAPI.fsDelete(paths)
  }

  return (
    <div className="app">
      <OperationsToolbar
        selectedCount={activeSelection.length}
        onCopy={handleCopy}
        onMove={handleMove}
        onDelete={handleDelete}
        onTogglePreview={() => setShowPreview(!showPreview)}
        showPreview={showPreview}
      />

      <div className="main-layout">
        <div className="dual-pane">
          <FilePane
            paneSide="left"
            isActive={activePane === 'left'}
            onActivate={() => setActivePane('left')}
            onSelectionChange={setLeftSelection}
          />
          <div className="pane-divider" />
          <FilePane
            paneSide="right"
            isActive={activePane === 'right'}
            onActivate={() => setActivePane('right')}
            onSelectionChange={setRightSelection}
          />
        </div>

        {showPreview && (
          <PreviewPanel file={previewFile} />
        )}
      </div>

      <div className="status-bar">
        <span>Active: {activePane === 'left' ? 'ซ้าย' : 'ขวา'}</span>
        {activeSelection.length > 0 && (
          <span>เลือก {activeSelection.length} รายการ</span>
        )}
      </div>
    </div>
  )
}
```

---

## สรุปฟีเจอร์

| ฟีเจอร์ | รายละเอียด |
|---------|------------|
| Dual-pane layout | สองหน้าต่างแบบ Norton Commander |
| Breadcrumb navigation | นำทางแบบ breadcrumb |
| File operations | Copy, Move, Delete (ถังขยะ) |
| Sort & Filter | เรียงตาม name/size/type/date, กรอง inline |
| Preview panel | รูปภาพ, text, file info |
| Hidden files | แสดง/ซ่อนได้ |
| Search | ค้นหาแบบ recursive |
| List/Grid view | สลับมุมมองได้ |
| History navigation | ← → history |
| Multi-select | Shift+Click, Ctrl+Click |
