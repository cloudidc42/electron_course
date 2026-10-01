# Part 62: Project - Markdown Editor

## สร้าง Markdown Editor พร้อม Live Preview และ Export

ในบทนี้เราจะสร้าง Markdown Editor แบบมืออาชีพที่มี live preview, split view, export PDF/HTML, custom themes และฟีเจอร์ครบครัน

---

## โครงสร้างโปรเจค

```
markdown-editor/
├── src/
│   ├── main/
│   │   ├── index.ts
│   │   ├── exportHandlers.ts
│   │   └── ipcHandlers.ts
│   ├── renderer/
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── MarkdownEditor.tsx
│   │   │   ├── MarkdownPreview.tsx
│   │   │   ├── Toolbar.tsx
│   │   │   ├── ThemeSelector.tsx
│   │   │   ├── TableEditor.tsx
│   │   │   └── FrontmatterPanel.tsx
│   │   └── utils/
│   │       ├── markdownProcessor.ts
│   │       └── themes.ts
│   └── preload/index.ts
└── package.json
```

---

## 1. Main Process

```typescript
// src/main/index.ts
import { app, BrowserWindow, ipcMain, dialog } from 'electron'
import { join } from 'path'
import { readFile, writeFile } from 'fs/promises'
import { registerExportHandlers } from './exportHandlers'

function createWindow() {
  const win = new BrowserWindow({
    width: 1400,
    height: 900,
    titleBarStyle: process.platform === 'darwin' ? 'hiddenInset' : 'default',
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      nodeIntegration: false,
      contextIsolation: true
    },
    backgroundColor: '#ffffff'
  })

  // IPC Handlers
  ipcMain.handle('file:open', async () => {
    const { canceled, filePaths } = await dialog.showOpenDialog(win, {
      filters: [{ name: 'Markdown', extensions: ['md', 'markdown', 'mdx'] }],
      properties: ['openFile']
    })
    if (canceled) return null
    const content = await readFile(filePaths[0], 'utf-8')
    return { path: filePaths[0], content }
  })

  ipcMain.handle('file:save', async (_, { path, content }: { path: string; content: string }) => {
    if (!path) {
      const { canceled, filePath } = await dialog.showSaveDialog(win, {
        filters: [{ name: 'Markdown', extensions: ['md'] }]
      })
      if (canceled || !filePath) return null
      await writeFile(filePath, content, 'utf-8')
      return filePath
    }
    await writeFile(path, content, 'utf-8')
    return path
  })

  registerExportHandlers(win)

  if (process.env.ELECTRON_RENDERER_URL) {
    win.loadURL(process.env.ELECTRON_RENDERER_URL)
  } else {
    win.loadFile(join(__dirname, '../renderer/index.html'))
  }
}

app.whenReady().then(createWindow)
```

---

## 2. Export Handlers (PDF & HTML)

```typescript
// src/main/exportHandlers.ts
import { ipcMain, dialog, BrowserWindow } from 'electron'
import { writeFile, readFile } from 'fs/promises'
import { join, basename, dirname } from 'path'

export function registerExportHandlers(win: BrowserWindow): void {
  // Export to PDF ผ่าน Electron's printToPDF
  ipcMain.handle('export:pdf', async (_, { html, fileName }: { html: string; fileName: string }) => {
    const { canceled, filePath } = await dialog.showSaveDialog(win, {
      defaultPath: fileName.replace('.md', '.pdf'),
      filters: [{ name: 'PDF', extensions: ['pdf'] }]
    })
    if (canceled || !filePath) return { success: false }

    // สร้าง hidden window สำหรับ print
    const printWin = new BrowserWindow({
      width: 800,
      height: 600,
      show: false,
      webPreferences: { javascript: false }
    })

    // โหลด HTML ที่มี styles ครบ
    const styledHtml = `<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    @page { margin: 2cm; }
    body { font-family: Georgia, serif; line-height: 1.6; color: #333; max-width: 800px; margin: 0 auto; }
    h1, h2, h3 { color: #1a1a1a; margin-top: 1.5em; }
    code { background: #f5f5f5; padding: 2px 6px; border-radius: 3px; font-family: monospace; }
    pre { background: #f5f5f5; padding: 16px; border-radius: 6px; overflow: auto; }
    blockquote { border-left: 4px solid #ccc; padding-left: 16px; color: #666; }
    table { border-collapse: collapse; width: 100%; }
    th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
    th { background: #f5f5f5; }
    img { max-width: 100%; height: auto; }
    a { color: #0066cc; }
  </style>
</head>
<body>${html}</body>
</html>`

    await printWin.loadURL(`data:text/html;charset=utf-8,${encodeURIComponent(styledHtml)}`)

    const pdfBuffer = await printWin.webContents.printToPDF({
      printBackground: true,
      pageSize: 'A4',
      margins: { top: 1, bottom: 1, left: 1.5, right: 1.5 }
    })

    printWin.close()
    await writeFile(filePath, pdfBuffer)
    return { success: true, path: filePath }
  })

  // Export to HTML
  ipcMain.handle('export:html', async (_, { html, fileName }: { html: string; fileName: string }) => {
    const { canceled, filePath } = await dialog.showSaveDialog(win, {
      defaultPath: fileName.replace('.md', '.html'),
      filters: [{ name: 'HTML', extensions: ['html'] }]
    })
    if (canceled || !filePath) return { success: false }

    const fullHtml = `<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>${basename(fileName, '.md')}</title>
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Noto Sans Thai', sans-serif;
      line-height: 1.7;
      color: #333;
      max-width: 860px;
      margin: 40px auto;
      padding: 0 20px;
    }
    h1 { border-bottom: 2px solid #eee; padding-bottom: 12px; }
    h2 { border-bottom: 1px solid #eee; padding-bottom: 8px; }
    code {
      background: #f6f8fa;
      padding: 2px 6px;
      border-radius: 4px;
      font-family: 'JetBrains Mono', Consolas, monospace;
      font-size: 90%;
    }
    pre {
      background: #f6f8fa;
      padding: 16px;
      border-radius: 6px;
      overflow: auto;
      border: 1px solid #e1e4e8;
    }
    pre code { background: none; padding: 0; }
    blockquote {
      margin: 0;
      padding: 8px 16px;
      border-left: 4px solid #0366d6;
      background: #f1f8ff;
      border-radius: 0 4px 4px 0;
    }
    table { border-collapse: collapse; width: 100%; margin: 16px 0; }
    th { background: #f6f8fa; font-weight: 600; }
    th, td { border: 1px solid #dfe2e5; padding: 8px 12px; }
    tr:nth-child(even) { background: #f9f9f9; }
    img { max-width: 100%; border-radius: 4px; }
    a { color: #0366d6; text-decoration: none; }
    a:hover { text-decoration: underline; }
    .highlight { background: #fff3cd; padding: 2px 4px; border-radius: 3px; }
  </style>
</head>
<body>
${html}
</body>
</html>`

    await writeFile(filePath, fullHtml, 'utf-8')
    return { success: true, path: filePath }
  })
}
```

---

## 3. Markdown Processor

```typescript
// src/renderer/utils/markdownProcessor.ts
import { marked } from 'marked'
import { markedHighlight } from 'marked-highlight'
import hljs from 'highlight.js'
import DOMPurify from 'dompurify'
import yaml from 'js-yaml'

// ตั้งค่า syntax highlighting
marked.use(markedHighlight({
  langPrefix: 'hljs language-',
  highlight(code, lang) {
    const language = hljs.getLanguage(lang) ? lang : 'plaintext'
    return hljs.highlight(code, { language }).value
  }
}))

// ตั้งค่า marked options
marked.setOptions({
  gfm: true,      // GitHub Flavored Markdown
  breaks: true,   // แปลง \n เป็น <br>
  pedantic: false
})

// Custom renderer สำหรับฟีเจอร์พิเศษ
const renderer = new marked.Renderer()

// Task list items
renderer.listitem = (text, task, checked) => {
  if (task) {
    return `<li class="task-item">
      <input type="checkbox" ${checked ? 'checked' : ''} disabled> ${text}
    </li>`
  }
  return `<li>${text}</li>`
}

// External links เปิดใน browser
renderer.link = (href, title, text) => {
  const isExternal = href?.startsWith('http')
  return `<a href="${href}" ${title ? `title="${title}"` : ''} ${isExternal ? 'target="_blank" rel="noopener"' : ''}>${text}</a>`
}

marked.use({ renderer })

export interface FrontmatterData {
  title?: string
  date?: string
  tags?: string[]
  author?: string
  description?: string
  [key: string]: unknown
}

export interface ProcessedMarkdown {
  html: string
  frontmatter: FrontmatterData | null
  toc: TocItem[]
  wordCount: number
  readTime: number
}

export interface TocItem {
  id: string
  text: string
  level: number
}

export function processMarkdown(raw: string): ProcessedMarkdown {
  let content = raw
  let frontmatter: FrontmatterData | null = null

  // Parse frontmatter
  const fmMatch = content.match(/^---\n([\s\S]*?)\n---\n?/)
  if (fmMatch) {
    try {
      frontmatter = yaml.load(fmMatch[1]) as FrontmatterData
    } catch {}
    content = content.slice(fmMatch[0].length)
  }

  // Extract TOC
  const toc: TocItem[] = []
  const tocRenderer = new marked.Renderer()
  tocRenderer.heading = (text, level) => {
    const id = text.toLowerCase()
      .replace(/[^\w\s-ก-๙]/g, '')
      .replace(/\s+/g, '-')
    toc.push({ id, text, level })
    return `<h${level} id="${id}">${text}</h${level}>`
  }
  marked.use({ renderer: tocRenderer })

  // แปลง Markdown เป็น HTML
  const rawHtml = marked.parse(content) as string

  // Sanitize HTML
  const html = DOMPurify.sanitize(rawHtml, {
    ALLOWED_TAGS: ['h1','h2','h3','h4','h5','h6','p','br','strong','em','del','code','pre',
      'blockquote','ul','ol','li','table','thead','tbody','tr','th','td','a','img',
      'input','hr','div','span'],
    ALLOWED_ATTR: ['href', 'src', 'alt', 'title', 'class', 'id', 'type', 'checked',
      'disabled', 'target', 'rel']
  })

  // Word count
  const words = content.replace(/[#*`_\[\]]/g, '').split(/\s+/).filter(Boolean)
  const wordCount = words.length
  const readTime = Math.ceil(wordCount / 200)

  return { html, frontmatter, toc, wordCount, readTime }
}

export function insertMarkdownSyntax(
  text: string,
  selStart: number,
  selEnd: number,
  syntax: string
): { newText: string; newStart: number; newEnd: number } {
  const selected = text.slice(selStart, selEnd)
  
  const syntaxMap: Record<string, [string, string]> = {
    bold: ['**', '**'],
    italic: ['*', '*'],
    strikethrough: ['~~', '~~'],
    code: ['`', '`'],
    codeblock: ['```\n', '\n```'],
    link: ['[', '](url)'],
    image: ['![alt text](', ')'],
    blockquote: ['> ', ''],
    heading1: ['# ', ''],
    heading2: ['## ', ''],
    heading3: ['### ', ''],
    table: ['\n| Column 1 | Column 2 |\n| --- | --- |\n| Cell 1 | Cell 2 |\n', '']
  }

  const [prefix, suffix] = syntaxMap[syntax] || ['', '']
  const newSelected = selected || syntax
  const newText = text.slice(0, selStart) + prefix + newSelected + suffix + text.slice(selEnd)
  const newStart = selStart + prefix.length
  const newEnd = newStart + newSelected.length

  return { newText, newStart, newEnd }
}
```

---

## 4. Markdown Editor Component

```tsx
// src/renderer/components/MarkdownEditor.tsx
import { useRef, useEffect, useCallback } from 'react'
import { useDropzone } from 'react-dropzone'
import { insertMarkdownSyntax } from '../utils/markdownProcessor'

interface MarkdownEditorProps {
  value: string
  onChange: (value: string) => void
  onImagePaste?: (file: File) => Promise<string>
  fontSize?: number
}

export function MarkdownEditor({ value, onChange, onImagePaste, fontSize = 14 }: MarkdownEditorProps) {
  const textareaRef = useRef<HTMLTextAreaElement>(null)

  // Handle paste - รองรับการวางรูปภาพ
  const handlePaste = async (e: React.ClipboardEvent) => {
    const items = Array.from(e.clipboardData.items)
    const imageItem = items.find(item => item.type.startsWith('image/'))
    
    if (imageItem && onImagePaste) {
      e.preventDefault()
      const file = imageItem.getAsFile()
      if (!file) return
      
      const url = await onImagePaste(file)
      const ta = textareaRef.current!
      const { selectionStart, selectionEnd } = ta
      const imgMarkdown = `![${file.name}](${url})`
      const newValue = value.slice(0, selectionStart) + imgMarkdown + value.slice(selectionEnd)
      onChange(newValue)
      
      // Move cursor after image syntax
      setTimeout(() => {
        ta.setSelectionRange(selectionStart + imgMarkdown.length, selectionStart + imgMarkdown.length)
        ta.focus()
      }, 0)
    }
  }

  // Handle keyboard shortcuts
  const handleKeyDown = (e: React.KeyboardEvent<HTMLTextAreaElement>) => {
    const ta = textareaRef.current!
    const { selectionStart, selectionEnd } = ta
    const mod = e.ctrlKey || e.metaKey

    // Bold: Ctrl+B
    if (mod && e.key === 'b') {
      e.preventDefault()
      applySyntax('bold')
      return
    }

    // Italic: Ctrl+I
    if (mod && e.key === 'i') {
      e.preventDefault()
      applySyntax('italic')
      return
    }

    // Tab key -> insert 2 spaces
    if (e.key === 'Tab') {
      e.preventDefault()
      const newValue = value.slice(0, selectionStart) + '  ' + value.slice(selectionEnd)
      onChange(newValue)
      setTimeout(() => ta.setSelectionRange(selectionStart + 2, selectionStart + 2), 0)
      return
    }

    // Auto-list continuation
    if (e.key === 'Enter') {
      const lineStart = value.lastIndexOf('\n', selectionStart - 1) + 1
      const currentLine = value.slice(lineStart, selectionStart)
      
      // Check for list patterns
      const bulletMatch = currentLine.match(/^(\s*)([-*+])\s/)
      const numberedMatch = currentLine.match(/^(\s*)(\d+)\.\s/)
      const taskMatch = currentLine.match(/^(\s*)([-*+])\s\[[ x]\]\s/)
      
      if (taskMatch) {
        e.preventDefault()
        const continuation = `\n${taskMatch[1]}${taskMatch[2]} [ ] `
        insertAtCursor(continuation)
        return
      }
      
      if (bulletMatch) {
        // ถ้าบรรทัดว่าง ให้หยุด list
        if (currentLine.trim() === bulletMatch[2]) {
          e.preventDefault()
          const removeList = value.slice(0, lineStart) + value.slice(selectionStart)
          onChange(removeList)
          return
        }
        e.preventDefault()
        insertAtCursor(`\n${bulletMatch[1]}${bulletMatch[2]} `)
        return
      }

      if (numberedMatch) {
        e.preventDefault()
        insertAtCursor(`\n${numberedMatch[1]}${parseInt(numberedMatch[2]) + 1}. `)
        return
      }
    }
  }

  const insertAtCursor = (text: string) => {
    const ta = textareaRef.current!
    const { selectionStart, selectionEnd } = ta
    const newValue = value.slice(0, selectionStart) + text + value.slice(selectionEnd)
    onChange(newValue)
    setTimeout(() => {
      ta.setSelectionRange(selectionStart + text.length, selectionStart + text.length)
      ta.focus()
    }, 0)
  }

  const applySyntax = (syntax: string) => {
    const ta = textareaRef.current!
    const { selectionStart, selectionEnd } = ta
    const { newText, newStart, newEnd } = insertMarkdownSyntax(value, selectionStart, selectionEnd, syntax)
    onChange(newText)
    setTimeout(() => {
      ta.setSelectionRange(newStart, newEnd)
      ta.focus()
    }, 0)
  }

  // Drag and drop files
  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    noClick: true,
    accept: { 'image/*': ['.png', '.jpg', '.jpeg', '.gif', '.webp'] },
    onDrop: async (files) => {
      if (onImagePaste && files.length > 0) {
        const url = await onImagePaste(files[0])
        insertAtCursor(`![${files[0].name}](${url})\n`)
      }
    }
  })

  return (
    <div className={`markdown-editor-wrapper ${isDragActive ? 'drag-active' : ''}`} {...getRootProps()}>
      <input {...getInputProps()} />
      {isDragActive && (
        <div className="drag-overlay">
          <span>วางรูปภาพที่นี่</span>
        </div>
      )}
      <textarea
        ref={textareaRef}
        className="markdown-textarea"
        value={value}
        onChange={e => onChange(e.target.value)}
        onKeyDown={handleKeyDown}
        onPaste={handlePaste}
        spellCheck={false}
        style={{ fontSize }}
        placeholder="เริ่มเขียน Markdown ที่นี่...\n\n# หัวข้อ\n\nเขียนเนื้อหาของคุณที่นี่..."
      />
    </div>
  )
}
```

---

## 5. Markdown Preview Component

```tsx
// src/renderer/components/MarkdownPreview.tsx
import { useEffect, useRef } from 'react'
import { ProcessedMarkdown } from '../utils/markdownProcessor'

interface MarkdownPreviewProps {
  processed: ProcessedMarkdown
  theme: string
  scrollSync?: boolean
  editorScrollPercent?: number
}

export function MarkdownPreview({ processed, theme, scrollSync, editorScrollPercent }: MarkdownPreviewProps) {
  const previewRef = useRef<HTMLDivElement>(null)

  // Scroll sync
  useEffect(() => {
    if (scrollSync && previewRef.current && editorScrollPercent !== undefined) {
      const el = previewRef.current
      const maxScroll = el.scrollHeight - el.clientHeight
      el.scrollTop = maxScroll * editorScrollPercent
    }
  }, [editorScrollPercent, scrollSync])

  // Handle link clicks
  const handleClick = (e: React.MouseEvent) => {
    const target = e.target as HTMLElement
    if (target.tagName === 'A') {
      const href = (target as HTMLAnchorElement).href
      if (href.startsWith('http')) {
        e.preventDefault()
        window.electronAPI.openExternal(href)
      }
    }
  }

  return (
    <div className={`markdown-preview theme-${theme}`} ref={previewRef}>
      {/* Frontmatter display */}
      {processed.frontmatter && (
        <div className="frontmatter-badge">
          {processed.frontmatter.tags?.map(tag => (
            <span key={tag} className="tag">#{tag}</span>
          ))}
          {processed.frontmatter.date && (
            <span className="date">{processed.frontmatter.date}</span>
          )}
        </div>
      )}

      {/* Stats */}
      <div className="preview-stats">
        <span>{processed.wordCount} คำ</span>
        <span>อ่าน {processed.readTime} นาที</span>
      </div>

      {/* Table of Contents */}
      {processed.toc.length > 2 && (
        <details className="toc-panel">
          <summary>สารบัญ</summary>
          <ul>
            {processed.toc.map((item) => (
              <li key={item.id} style={{ marginLeft: (item.level - 1) * 16 }}>
                <a href={`#${item.id}`}>{item.text}</a>
              </li>
            ))}
          </ul>
        </details>
      )}

      {/* Main Content */}
      <div
        className="markdown-content"
        dangerouslySetInnerHTML={{ __html: processed.html }}
        onClick={handleClick}
      />
    </div>
  )
}
```

---

## 6. Toolbar Component

```tsx
// src/renderer/components/Toolbar.tsx
interface ToolbarProps {
  onFormat: (syntax: string) => void
  onInsert: (type: string) => void
  viewMode: 'edit' | 'preview' | 'split'
  onViewModeChange: (mode: 'edit' | 'preview' | 'split') => void
  onExport: (type: 'pdf' | 'html') => void
  onOpen: () => void
  onSave: () => void
}

const TOOLBAR_GROUPS = [
  {
    name: 'format',
    items: [
      { icon: 'B', label: 'Bold (Ctrl+B)', action: 'bold', style: { fontWeight: 'bold' } },
      { icon: 'I', label: 'Italic (Ctrl+I)', action: 'italic', style: { fontStyle: 'italic' } },
      { icon: 'S̶', label: 'Strikethrough', action: 'strikethrough' },
      { icon: '`', label: 'Inline Code', action: 'code' }
    ]
  },
  {
    name: 'heading',
    items: [
      { icon: 'H1', label: 'Heading 1', action: 'heading1' },
      { icon: 'H2', label: 'Heading 2', action: 'heading2' },
      { icon: 'H3', label: 'Heading 3', action: 'heading3' }
    ]
  },
  {
    name: 'insert',
    items: [
      { icon: '🔗', label: 'Link', action: 'link' },
      { icon: '🖼', label: 'Image', action: 'image' },
      { icon: '❝', label: 'Blockquote', action: 'blockquote' },
      { icon: '</>', label: 'Code Block', action: 'codeblock' },
      { icon: '⊞', label: 'Table', action: 'table' }
    ]
  }
]

export function Toolbar({ onFormat, onInsert, viewMode, onViewModeChange, onExport, onOpen, onSave }: ToolbarProps) {
  return (
    <div className="toolbar">
      <div className="toolbar-section file-actions">
        <button onClick={onOpen} title="เปิดไฟล์ (Ctrl+O)">📁</button>
        <button onClick={onSave} title="บันทึก (Ctrl+S)">💾</button>
      </div>

      <div className="toolbar-divider" />

      {TOOLBAR_GROUPS.map(group => (
        <div key={group.name} className="toolbar-section">
          {group.items.map(item => (
            <button
              key={item.action}
              onClick={() => onFormat(item.action)}
              title={item.label}
              style={item.style}
              className="toolbar-btn"
            >
              {item.icon}
            </button>
          ))}
          <div className="toolbar-divider" />
        </div>
      ))}

      <div className="toolbar-section view-controls">
        {(['edit', 'split', 'preview'] as const).map(mode => (
          <button
            key={mode}
            className={`toolbar-btn ${viewMode === mode ? 'active' : ''}`}
            onClick={() => onViewModeChange(mode)}
            title={mode === 'edit' ? 'แก้ไขอย่างเดียว' : mode === 'preview' ? 'ดูตัวอย่างอย่างเดียว' : 'แยกครึ่ง'}
          >
            {mode === 'edit' ? '✏️' : mode === 'preview' ? '👁' : '⊟'}
          </button>
        ))}
      </div>

      <div className="toolbar-spacer" />

      <div className="toolbar-section export-actions">
        <button onClick={() => onExport('html')} title="Export เป็น HTML">HTML</button>
        <button onClick={() => onExport('pdf')} title="Export เป็น PDF">PDF</button>
      </div>
    </div>
  )
}
```

---

## 7. Table Editor Component

```tsx
// src/renderer/components/TableEditor.tsx
import { useState } from 'react'

interface TableData {
  headers: string[]
  rows: string[][]
  alignments: ('left' | 'center' | 'right')[]
}

interface TableEditorProps {
  onInsert: (markdown: string) => void
  onClose: () => void
}

export function TableEditor({ onInsert, onClose }: TableEditorProps) {
  const [table, setTable] = useState<TableData>({
    headers: ['คอลัมน์ 1', 'คอลัมน์ 2', 'คอลัมน์ 3'],
    rows: [['', '', ''], ['', '', '']],
    alignments: ['left', 'left', 'left']
  })

  const addColumn = () => {
    setTable(t => ({
      ...t,
      headers: [...t.headers, `คอลัมน์ ${t.headers.length + 1}`],
      rows: t.rows.map(row => [...row, '']),
      alignments: [...t.alignments, 'left']
    }))
  }

  const addRow = () => {
    setTable(t => ({
      ...t,
      rows: [...t.rows, new Array(t.headers.length).fill('')]
    }))
  }

  const updateHeader = (idx: number, value: string) => {
    setTable(t => {
      const headers = [...t.headers]
      headers[idx] = value
      return { ...t, headers }
    })
  }

  const updateCell = (rowIdx: number, colIdx: number, value: string) => {
    setTable(t => {
      const rows = t.rows.map(r => [...r])
      rows[rowIdx][colIdx] = value
      return { ...t, rows }
    })
  }

  const toggleAlignment = (idx: number) => {
    const cycle: ('left' | 'center' | 'right')[] = ['left', 'center', 'right']
    setTable(t => {
      const alignments = [...t.alignments]
      const current = cycle.indexOf(alignments[idx])
      alignments[idx] = cycle[(current + 1) % 3]
      return { ...t, alignments }
    })
  }

  const generateMarkdown = () => {
    const alignStr = (a: string) => a === 'center' ? ':---:' : a === 'right' ? '---:' : '---'
    
    const header = `| ${table.headers.join(' | ')} |`
    const separator = `| ${table.alignments.map(alignStr).join(' | ')} |`
    const rows = table.rows.map(row => `| ${row.join(' | ')} |`)
    
    return [header, separator, ...rows].join('\n')
  }

  const handleInsert = () => {
    onInsert('\n' + generateMarkdown() + '\n')
    onClose()
  }

  return (
    <div className="table-editor-modal">
      <div className="table-editor">
        <h3>สร้างตาราง</h3>
        <div className="table-preview">
          <table>
            <thead>
              <tr>
                {table.headers.map((h, i) => (
                  <th key={i}>
                    <input value={h} onChange={e => updateHeader(i, e.target.value)} />
                    <button
                      className="align-btn"
                      onClick={() => toggleAlignment(i)}
                      title="เปลี่ยนการจัดวาง"
                    >
                      {table.alignments[i] === 'left' ? '⬅' : table.alignments[i] === 'center' ? '⬛' : '➡'}
                    </button>
                  </th>
                ))}
                <th><button onClick={addColumn}>+</button></th>
              </tr>
            </thead>
            <tbody>
              {table.rows.map((row, ri) => (
                <tr key={ri}>
                  {row.map((cell, ci) => (
                    <td key={ci}>
                      <input value={cell} onChange={e => updateCell(ri, ci, e.target.value)} />
                    </td>
                  ))}
                  <td />
                </tr>
              ))}
              <tr>
                <td colSpan={table.headers.length + 1}>
                  <button onClick={addRow}>+ เพิ่มแถว</button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        <div className="table-editor-preview">
          <pre>{generateMarkdown()}</pre>
        </div>
        <div className="table-editor-actions">
          <button onClick={handleInsert} className="btn-primary">แทรกตาราง</button>
          <button onClick={onClose}>ยกเลิก</button>
        </div>
      </div>
    </div>
  )
}
```

---

## 8. App Component หลัก

```tsx
// src/renderer/App.tsx
import { useState, useEffect, useMemo, useCallback } from 'react'
import { MarkdownEditor } from './components/MarkdownEditor'
import { MarkdownPreview } from './components/MarkdownPreview'
import { Toolbar } from './components/Toolbar'
import { TableEditor } from './components/TableEditor'
import { FrontmatterPanel } from './components/FrontmatterPanel'
import { processMarkdown, insertMarkdownSyntax } from './utils/markdownProcessor'
import { THEMES } from './utils/themes'
import './styles/app.css'

const DEFAULT_CONTENT = `---
title: เอกสารตัวอย่าง
date: ${new Date().toISOString().split('T')[0]}
tags: [markdown, electron, example]
---

# ยินดีต้อนรับสู่ Markdown Editor

เขียน **Markdown** ได้อย่างสะดวกสบายพร้อม _live preview_

## ฟีเจอร์เด่น

- ✅ Live Preview แบบ split view
- ✅ Syntax Highlighting
- ✅ Export PDF และ HTML
- ✅ รองรับ Frontmatter
- ✅ Table Editor แบบ visual
- ✅ วางรูปภาพได้โดยตรง

## ตัวอย่าง Code

\`\`\`typescript
const greeting = (name: string): string => {
  return \`สวัสดี, \${name}!\`
}
console.log(greeting('นักพัฒนา'))
\`\`\`
`

export default function App() {
  const [content, setContent] = useState(DEFAULT_CONTENT)
  const [filePath, setFilePath] = useState<string | null>(null)
  const [viewMode, setViewMode] = useState<'edit' | 'preview' | 'split'>('split')
  const [theme, setTheme] = useState('github')
  const [fontSize, setFontSize] = useState(15)
  const [isDirty, setIsDirty] = useState(false)
  const [showTableEditor, setShowTableEditor] = useState(false)
  const [editorScrollPercent, setEditorScrollPercent] = useState(0)

  const processed = useMemo(() => processMarkdown(content), [content])

  const handleContentChange = (newContent: string) => {
    setContent(newContent)
    setIsDirty(true)
  }

  const handleFormat = useCallback((syntax: string) => {
    if (syntax === 'table') {
      setShowTableEditor(true)
      return
    }
    // Dispatch to editor component
    window.dispatchEvent(new CustomEvent('editor:format', { detail: syntax }))
  }, [])

  const handleOpen = async () => {
    if (isDirty && !confirm('มีการเปลี่ยนแปลงที่ยังไม่ได้บันทึก ต้องการเปิดไฟล์ใหม่หรือไม่?')) return
    const result = await window.electronAPI.openFile()
    if (result) {
      setContent(result.content)
      setFilePath(result.path)
      setIsDirty(false)
    }
  }

  const handleSave = async () => {
    const savedPath = await window.electronAPI.saveFile({ path: filePath, content })
    if (savedPath) {
      setFilePath(savedPath)
      setIsDirty(false)
    }
  }

  const handleExport = async (type: 'pdf' | 'html') => {
    const fileName = filePath ? filePath.split('/').pop() || 'document.md' : 'document.md'
    if (type === 'pdf') {
      await window.electronAPI.exportPdf({ html: processed.html, fileName })
    } else {
      await window.electronAPI.exportHtml({ html: processed.html, fileName })
    }
  }

  const handleImagePaste = async (file: File): Promise<string> => {
    // บันทึกรูปภาพลง disk แล้วคืน path
    const buffer = await file.arrayBuffer()
    const result = await window.electronAPI.saveImage({ buffer: Array.from(new Uint8Array(buffer)), name: file.name })
    return result.path
  }

  // Update window title
  useEffect(() => {
    const name = filePath ? filePath.split('/').pop() : 'Untitled'
    document.title = `${isDirty ? '● ' : ''}${name} - Markdown Editor`
  }, [filePath, isDirty])

  return (
    <div className="app">
      <Toolbar
        onFormat={handleFormat}
        onInsert={handleFormat}
        viewMode={viewMode}
        onViewModeChange={setViewMode}
        onExport={handleExport}
        onOpen={handleOpen}
        onSave={handleSave}
      />

      <div className={`editor-layout mode-${viewMode}`}>
        {(viewMode === 'edit' || viewMode === 'split') && (
          <MarkdownEditor
            value={content}
            onChange={handleContentChange}
            onImagePaste={handleImagePaste}
            fontSize={fontSize}
          />
        )}

        {viewMode === 'split' && <div className="split-divider" />}

        {(viewMode === 'preview' || viewMode === 'split') && (
          <MarkdownPreview
            processed={processed}
            theme={theme}
            scrollSync={viewMode === 'split'}
            editorScrollPercent={editorScrollPercent}
          />
        )}
      </div>

      {showTableEditor && (
        <TableEditor
          onInsert={(md) => setContent(c => c + '\n' + md)}
          onClose={() => setShowTableEditor(false)}
        />
      )}
    </div>
  )
}
```

---

## 9. CSS Themes

```typescript
// src/renderer/utils/themes.ts
export const THEMES: Record<string, string> = {
  github: `
    .markdown-content { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; }
    .markdown-content h1 { border-bottom: 2px solid #e1e4e8; }
    .markdown-content pre { background: #f6f8fa; border: 1px solid #e1e4e8; }
    .markdown-content code { background: rgba(27,31,35,.05); color: #e36209; }
  `,
  dark: `
    .markdown-preview { background: #1e1e1e; color: #d4d4d4; }
    .markdown-content h1, .markdown-content h2 { color: #4fc1ff; border-color: #3e3e42; }
    .markdown-content pre { background: #2d2d30; border-color: #3e3e42; }
    .markdown-content code { background: #2d2d30; color: #ce9178; }
    .markdown-content a { color: #4fc1ff; }
    .markdown-content blockquote { border-left-color: #007acc; background: #1b2a3b; }
  `,
  solarized: `
    .markdown-preview { background: #fdf6e3; color: #657b83; }
    .markdown-content h1, h2, h3 { color: #268bd2; }
    .markdown-content pre { background: #eee8d5; }
    .markdown-content code { color: #dc322f; background: #eee8d5; }
    .markdown-content a { color: #2aa198; }
    .markdown-content blockquote { border-left-color: #cb4b16; }
  `,
  minimal: `
    .markdown-preview { background: #fff; color: #333; font-family: Georgia, serif; }
    .markdown-content pre { background: #f5f5f5; border-left: 3px solid #ccc; }
    .markdown-content code { background: #f0f0f0; }
    .markdown-content blockquote { color: #666; border-left-color: #ccc; }
  `
}
```

---

## สรุปฟีเจอร์

| ฟีเจอร์ | รายละเอียด |
|---------|------------|
| Live Preview | แสดงผล Markdown แบบ real-time |
| Split View | แบ่งครึ่งหน้าจอ edit/preview |
| GitHub Flavored Markdown | รองรับ tables, task lists, strikethrough |
| Syntax Highlighting | 100+ ภาษา ผ่าน highlight.js |
| Export PDF | ผ่าน Electron printToPDF |
| Export HTML | พร้อม embedded styles |
| Frontmatter | YAML frontmatter support |
| Image Paste | วางรูปภาพจาก clipboard |
| Table Editor | Visual table builder |
| Multiple Themes | GitHub, Dark, Solarized, Minimal |
| Word Count & Read Time | แสดงข้อมูล statistics |
| Table of Contents | auto-generated จาก headings |
| Auto-list continuation | กด Enter ต่อ list อัตโนมัติ |
