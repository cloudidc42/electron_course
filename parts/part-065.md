# Part 65: Project - Screenshot Tool

## สร้าง Screenshot Tool แบบ Professional

ในบทนี้เราจะสร้าง Screenshot Tool ที่มี screen capture, annotation tools, upload to imgur, tray menu และ hotkey trigger

---

## โครงสร้างโปรเจค

```
screenshot-tool/
├── src/
│   ├── main/
│   │   ├── index.ts
│   │   ├── captureManager.ts
│   │   ├── trayManager.ts
│   │   └── overlayWindow.ts
│   ├── renderer/
│   │   ├── capture/
│   │   │   └── index.html     (overlay for region selection)
│   │   ├── editor/
│   │   │   ├── App.tsx        (annotation editor)
│   │   │   └── components/
│   │   └── gallery/
│   │       └── App.tsx        (screenshot gallery)
│   └── preload/index.ts
```

---

## 1. Main Process - Capture Manager

```typescript
// src/main/captureManager.ts
import {
  BrowserWindow, desktopCapturer, screen,
  ipcMain, clipboard, nativeImage, shell
} from 'electron'
import { writeFile, mkdir } from 'fs/promises'
import { join } from 'path'
import os from 'os'

const SAVE_DIR = join(os.homedir(), 'Pictures', 'Screenshots')

export interface CaptureResult {
  dataUrl: string
  filePath: string
  timestamp: number
}

export class CaptureManager {
  private overlayWindows: Map<number, BrowserWindow> = new Map()

  // Capture full screen
  async captureFullScreen(displayId?: number): Promise<CaptureResult> {
    const displays = screen.getAllDisplays()
    const target = displayId
      ? displays.find(d => d.id === displayId) || displays[0]
      : screen.getPrimaryDisplay()

    const sources = await desktopCapturer.getSources({
      types: ['screen'],
      thumbnailSize: {
        width: target.bounds.width * target.scaleFactor,
        height: target.bounds.height * target.scaleFactor
      }
    })

    const source = sources.find(s => s.display_id === target.id.toString()) || sources[0]
    const dataUrl = source.thumbnail.toDataURL()

    const filePath = await this.saveScreenshot(dataUrl)
    return { dataUrl, filePath, timestamp: Date.now() }
  }

  // Capture region (ผ่าน overlay window)
  async captureRegion(): Promise<CaptureResult | null> {
    return new Promise((resolve) => {
      const display = screen.getPrimaryDisplay()

      const overlay = new BrowserWindow({
        x: display.bounds.x,
        y: display.bounds.y,
        width: display.bounds.width,
        height: display.bounds.height,
        transparent: true,
        frame: false,
        alwaysOnTop: true,
        skipTaskbar: true,
        cursor: 'crosshair',
        webPreferences: {
          preload: join(__dirname, '../preload/index.js'),
          nodeIntegration: false,
          contextIsolation: true
        }
      })

      overlay.setFullScreen(true)
      overlay.loadFile(join(__dirname, '../renderer/capture/index.html'))

      overlay.once('ready-to-show', () => overlay.show())

      // รับผลจาก overlay
      ipcMain.once('capture:region-selected', async (_, rect: { x: number; y: number; width: number; height: number }) => {
        overlay.close()

        if (!rect || rect.width < 5 || rect.height < 5) {
          resolve(null)
          return
        }

        // Capture full screen แล้ว crop
        const fullCapture = await this.captureFullScreen()
        const img = nativeImage.createFromDataURL(fullCapture.dataUrl)
        const cropped = img.crop(rect)
        const dataUrl = cropped.toDataURL()
        const filePath = await this.saveScreenshot(dataUrl)

        resolve({ dataUrl, filePath, timestamp: Date.now() })
      })

      ipcMain.once('capture:cancelled', () => {
        overlay.close()
        resolve(null)
      })
    })
  }

  // Capture with delay
  async captureWithDelay(seconds: number): Promise<CaptureResult> {
    return new Promise((resolve) => {
      setTimeout(async () => {
        const result = await this.captureFullScreen()
        resolve(result)
      }, seconds * 1000)
    })
  }

  // Save screenshot
  private async saveScreenshot(dataUrl: string): Promise<string> {
    await mkdir(SAVE_DIR, { recursive: true })
    
    const timestamp = new Date().toISOString()
      .replace(/[:.]/g, '-')
      .replace('T', '_')
      .slice(0, 19)
    const fileName = `screenshot_${timestamp}.png`
    const filePath = join(SAVE_DIR, fileName)

    const base64 = dataUrl.replace(/^data:image\/\w+;base64,/, '')
    const buffer = Buffer.from(base64, 'base64')
    await writeFile(filePath, buffer)

    return filePath
  }

  // Copy to clipboard
  copyToClipboard(dataUrl: string): void {
    const img = nativeImage.createFromDataURL(dataUrl)
    clipboard.writeImage(img)
  }
}

export const captureManager = new CaptureManager()
```

---

## 2. Region Selection Overlay

```html
<!-- src/renderer/capture/index.html -->
<!DOCTYPE html>
<html>
<head>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      width: 100vw; height: 100vh;
      cursor: crosshair;
      background: rgba(0,0,0,0.3);
      user-select: none;
    }
    #selection {
      position: fixed;
      border: 2px solid #00aaff;
      box-shadow: 0 0 0 9999px rgba(0,0,0,0.4);
      display: none;
    }
    #size-indicator {
      position: fixed;
      background: rgba(0,0,0,0.8);
      color: white;
      padding: 4px 8px;
      border-radius: 4px;
      font: 12px monospace;
      pointer-events: none;
    }
    #instructions {
      position: fixed;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      background: rgba(0,0,0,0.8);
      color: white;
      padding: 12px 20px;
      border-radius: 8px;
      text-align: center;
      font: 14px sans-serif;
      pointer-events: none;
    }
  </style>
</head>
<body>
  <div id="instructions">
    ลากเพื่อเลือกพื้นที่ · Escape เพื่อยกเลิก
  </div>
  <div id="selection"></div>
  <div id="size-indicator"></div>

  <script>
    const selection = document.getElementById('selection')
    const sizeIndicator = document.getElementById('size-indicator')
    const instructions = document.getElementById('instructions')

    let isDrawing = false
    let startX = 0, startY = 0

    document.addEventListener('mousedown', (e) => {
      if (e.button !== 0) return
      isDrawing = true
      startX = e.clientX
      startY = e.clientY
      selection.style.display = 'block'
      instructions.style.display = 'none'
    })

    document.addEventListener('mousemove', (e) => {
      if (!isDrawing) return

      const x = Math.min(e.clientX, startX)
      const y = Math.min(e.clientY, startY)
      const width = Math.abs(e.clientX - startX)
      const height = Math.abs(e.clientY - startY)

      selection.style.left = x + 'px'
      selection.style.top = y + 'px'
      selection.style.width = width + 'px'
      selection.style.height = height + 'px'

      sizeIndicator.textContent = `${width} × ${height}`
      sizeIndicator.style.left = (e.clientX + 10) + 'px'
      sizeIndicator.style.top = (e.clientY + 10) + 'px'
    })

    document.addEventListener('mouseup', (e) => {
      if (!isDrawing) return
      isDrawing = false

      const x = Math.min(e.clientX, startX)
      const y = Math.min(e.clientY, startY)
      const width = Math.abs(e.clientX - startX)
      const height = Math.abs(e.clientY - startY)

      window.electronAPI.regionSelected({ x, y, width, height })
    })

    document.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') {
        window.electronAPI.cancelCapture()
      }
    })
  </script>
</body>
</html>
```

---

## 3. Annotation Editor Component

```tsx
// src/renderer/editor/components/AnnotationCanvas.tsx
import { useRef, useState, useEffect, useCallback } from 'react'

type Tool = 'arrow' | 'text' | 'rectangle' | 'circle' | 'blur' | 'pen' | 'highlight'

interface Annotation {
  id: string
  tool: Tool
  color: string
  strokeWidth: number
  data: Record<string, unknown>
}

interface AnnotationCanvasProps {
  imageUrl: string
  onExport: (dataUrl: string) => void
}

export function AnnotationCanvas({ imageUrl, onExport }: AnnotationCanvasProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const overlayRef = useRef<HTMLCanvasElement>(null)
  const [tool, setTool] = useState<Tool>('arrow')
  const [color, setColor] = useState('#ff0000')
  const [strokeWidth, setStrokeWidth] = useState(3)
  const [annotations, setAnnotations] = useState<Annotation[]>([])
  const [isDrawing, setIsDrawing] = useState(false)
  const [startPos, setStartPos] = useState({ x: 0, y: 0 })
  const imageRef = useRef<HTMLImageElement | null>(null)

  // โหลดรูปภาพ
  useEffect(() => {
    const img = new Image()
    img.onload = () => {
      imageRef.current = img
      const canvas = canvasRef.current!
      canvas.width = img.width
      canvas.height = img.height
      const overlayCanvas = overlayRef.current!
      overlayCanvas.width = img.width
      overlayCanvas.height = img.height
      renderAll()
    }
    img.src = imageUrl
  }, [imageUrl])

  const renderAll = useCallback(() => {
    const canvas = canvasRef.current
    const img = imageRef.current
    if (!canvas || !img) return

    const ctx = canvas.getContext('2d')!
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    ctx.drawImage(img, 0, 0)

    // วาด annotations ทั้งหมด
    annotations.forEach(ann => renderAnnotation(ctx, ann))
  }, [annotations])

  useEffect(() => { renderAll() }, [annotations, renderAll])

  const renderAnnotation = (ctx: CanvasRenderingContext2D, ann: Annotation) => {
    ctx.save()
    ctx.strokeStyle = ann.color
    ctx.fillStyle = ann.color
    ctx.lineWidth = ann.strokeWidth

    switch (ann.tool) {
      case 'arrow': {
        const { x1, y1, x2, y2 } = ann.data as { x1: number; y1: number; x2: number; y2: number }
        drawArrow(ctx, x1, y1, x2, y2)
        break
      }
      case 'rectangle': {
        const { x, y, w, h } = ann.data as { x: number; y: number; w: number; h: number }
        ctx.strokeRect(x, y, w, h)
        break
      }
      case 'text': {
        const { x, y, text } = ann.data as { x: number; y: number; text: string }
        ctx.font = `bold ${ann.strokeWidth * 8}px sans-serif`
        ctx.fillText(text, x, y)
        break
      }
      case 'blur': {
        const { x, y, w, h } = ann.data as { x: number; y: number; w: number; h: number }
        ctx.filter = 'blur(8px)'
        ctx.drawImage(canvasRef.current!, x, y, w, h, x, y, w, h)
        ctx.filter = 'none'
        break
      }
    }
    ctx.restore()
  }

  const drawArrow = (ctx: CanvasRenderingContext2D, x1: number, y1: number, x2: number, y2: number) => {
    const angle = Math.atan2(y2 - y1, x2 - x1)
    const headLength = 15

    ctx.beginPath()
    ctx.moveTo(x1, y1)
    ctx.lineTo(x2, y2)
    ctx.stroke()

    ctx.beginPath()
    ctx.moveTo(x2, y2)
    ctx.lineTo(x2 - headLength * Math.cos(angle - Math.PI / 6), y2 - headLength * Math.sin(angle - Math.PI / 6))
    ctx.lineTo(x2 - headLength * Math.cos(angle + Math.PI / 6), y2 - headLength * Math.sin(angle + Math.PI / 6))
    ctx.closePath()
    ctx.fill()
  }

  const handleMouseDown = (e: React.MouseEvent) => {
    const rect = canvasRef.current!.getBoundingClientRect()
    const scaleX = canvasRef.current!.width / rect.width
    const scaleY = canvasRef.current!.height / rect.height
    const x = (e.clientX - rect.left) * scaleX
    const y = (e.clientY - rect.top) * scaleY

    setIsDrawing(true)
    setStartPos({ x, y })

    if (tool === 'text') {
      const text = prompt('ใส่ข้อความ:')
      if (text) {
        setAnnotations(prev => [...prev, {
          id: Date.now().toString(),
          tool: 'text', color, strokeWidth,
          data: { x, y, text }
        }])
      }
      setIsDrawing(false)
    }
  }

  const handleMouseUp = (e: React.MouseEvent) => {
    if (!isDrawing) return
    setIsDrawing(false)

    const rect = canvasRef.current!.getBoundingClientRect()
    const scaleX = canvasRef.current!.width / rect.width
    const scaleY = canvasRef.current!.height / rect.height
    const x = (e.clientX - rect.left) * scaleX
    const y = (e.clientY - rect.top) * scaleY

    const w = x - startPos.x
    const h = y - startPos.y

    if (Math.abs(w) < 3 && Math.abs(h) < 3) return

    let data: Record<string, unknown> = {}
    switch (tool) {
      case 'arrow': data = { x1: startPos.x, y1: startPos.y, x2: x, y2: y }; break
      case 'rectangle': data = { x: startPos.x, y: startPos.y, w, h }; break
      case 'blur': data = { x: Math.min(startPos.x, x), y: Math.min(startPos.y, y), w: Math.abs(w), h: Math.abs(h) }; break
    }

    setAnnotations(prev => [...prev, { id: Date.now().toString(), tool, color, strokeWidth, data }])
  }

  const handleExport = () => {
    const dataUrl = canvasRef.current!.toDataURL('image/png')
    onExport(dataUrl)
  }

  const TOOLS = [
    { id: 'arrow', icon: '↗', label: 'ลูกศร' },
    { id: 'rectangle', icon: '□', label: 'สี่เหลี่ยม' },
    { id: 'text', icon: 'T', label: 'ข้อความ' },
    { id: 'blur', icon: '⊘', label: 'เบลอ' },
    { id: 'pen', icon: '✏', label: 'ปากกา' }
  ] as const

  const COLORS = ['#ff0000', '#ff6600', '#ffff00', '#00ff00', '#0066ff', '#9900ff', '#ffffff', '#000000']

  return (
    <div className="annotation-editor">
      <div className="annotation-toolbar">
        <div className="tool-group">
          {TOOLS.map(t => (
            <button
              key={t.id}
              className={`tool-btn ${tool === t.id ? 'active' : ''}`}
              onClick={() => setTool(t.id as Tool)}
              title={t.label}
            >
              {t.icon}
            </button>
          ))}
        </div>

        <div className="color-picker">
          {COLORS.map(c => (
            <button
              key={c}
              className={`color-btn ${color === c ? 'active' : ''}`}
              style={{ background: c }}
              onClick={() => setColor(c)}
            />
          ))}
        </div>

        <input
          type="range"
          min={1} max={10}
          value={strokeWidth}
          onChange={e => setStrokeWidth(+e.target.value)}
          title="ขนาดเส้น"
          className="stroke-width"
        />

        <div className="toolbar-spacer" />

        <button onClick={() => { setAnnotations([]); renderAll() }} title="ล้างทั้งหมด">🗑</button>
        <button onClick={() => setAnnotations(a => a.slice(0, -1))} title="เลิกทำ (Ctrl+Z)">↩</button>
        <button onClick={handleExport} className="export-btn">💾 บันทึก</button>
        <button onClick={() => {
          const dataUrl = canvasRef.current!.toDataURL()
          window.electronAPI.copyToClipboard(dataUrl)
        }} className="copy-btn">📋 คัดลอก</button>
      </div>

      <div className="canvas-container">
        <canvas
          ref={canvasRef}
          className="annotation-canvas"
          onMouseDown={handleMouseDown}
          onMouseUp={handleMouseUp}
          style={{ cursor: tool === 'text' ? 'text' : 'crosshair' }}
        />
        <canvas ref={overlayRef} className="overlay-canvas" />
      </div>
    </div>
  )
}
```

---

## 4. Main IPC Handlers

```typescript
// src/main/index.ts
import { app, BrowserWindow, ipcMain, globalShortcut, Tray, Menu, nativeImage } from 'electron'
import { join } from 'path'
import { captureManager } from './captureManager'
import axios from 'axios'
import FormData from 'form-data'

let mainWindow: BrowserWindow | null = null
let tray: Tray | null = null

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 900,
    height: 700,
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })

  registerHandlers()
  registerShortcuts()
  setupTray()

  if (process.env.ELECTRON_RENDERER_URL) {
    mainWindow.loadURL(process.env.ELECTRON_RENDERER_URL)
  } else {
    mainWindow.loadFile(join(__dirname, '../renderer/editor/index.html'))
  }
}

function registerHandlers() {
  // Capture full screen
  ipcMain.handle('capture:fullscreen', async (_, displayId?: number) => {
    return captureManager.captureFullScreen(displayId)
  })

  // Capture region
  ipcMain.handle('capture:region', async () => {
    return captureManager.captureRegion()
  })

  // Capture with delay
  ipcMain.handle('capture:delayed', async (_, seconds: number) => {
    return captureManager.captureWithDelay(seconds)
  })

  // Copy to clipboard
  ipcMain.handle('capture:copy-clipboard', (_, dataUrl: string) => {
    captureManager.copyToClipboard(dataUrl)
  })

  // Upload to Imgur
  ipcMain.handle('capture:upload-imgur', async (_, dataUrl: string) => {
    try {
      const base64 = dataUrl.replace(/^data:image\/\w+;base64,/, '')
      const form = new FormData()
      form.append('image', base64)
      form.append('type', 'base64')

      const response = await axios.post('https://api.imgur.com/3/image', form, {
        headers: {
          Authorization: 'Client-ID YOUR_IMGUR_CLIENT_ID',
          ...form.getHeaders()
        }
      })

      return { success: true, url: response.data.data.link }
    } catch (error) {
      return { success: false, error: (error as Error).message }
    }
  })

  // Region selection handlers
  ipcMain.on('capture:region-selected', () => {}) // handled in captureManager
  ipcMain.on('capture:cancelled', () => {})
}

function registerShortcuts() {
  // Ctrl+Shift+S = Screenshot ทันที
  globalShortcut.register('CommandOrControl+Shift+S', async () => {
    const result = await captureManager.captureFullScreen()
    mainWindow?.webContents.send('capture:new', result)
    mainWindow?.show()
    mainWindow?.focus()
  })

  // Ctrl+Shift+R = Region capture
  globalShortcut.register('CommandOrControl+Shift+R', async () => {
    const result = await captureManager.captureRegion()
    if (result) {
      mainWindow?.webContents.send('capture:new', result)
      mainWindow?.show()
      mainWindow?.focus()
    }
  })
}

function setupTray() {
  const icon = nativeImage.createFromPath(join(__dirname, '../assets/icon.png')).resize({ width: 16, height: 16 })
  tray = new Tray(icon)

  const menu = Menu.buildFromTemplate([
    {
      label: 'จับภาพหน้าจอ (Ctrl+Shift+S)',
      click: async () => {
        const result = await captureManager.captureFullScreen()
        mainWindow?.webContents.send('capture:new', result)
        mainWindow?.show()
      }
    },
    {
      label: 'เลือกพื้นที่ (Ctrl+Shift+R)',
      click: async () => {
        const result = await captureManager.captureRegion()
        if (result) {
          mainWindow?.webContents.send('capture:new', result)
          mainWindow?.show()
        }
      }
    },
    {
      label: 'จับภาพหลังจาก...',
      submenu: [3, 5, 10].map(s => ({
        label: `${s} วินาที`,
        click: async () => {
          const result = await captureManager.captureWithDelay(s)
          mainWindow?.webContents.send('capture:new', result)
          mainWindow?.show()
        }
      }))
    },
    { type: 'separator' },
    { label: 'ออก', click: () => app.quit() }
  ])

  tray.setContextMenu(menu)
  tray.setToolTip('Screenshot Tool')
  tray.on('click', () => mainWindow?.show())
}

app.whenReady().then(createWindow)
app.on('will-quit', () => globalShortcut.unregisterAll())
```

---

## 5. Gallery Component

```tsx
// src/renderer/gallery/App.tsx
import { useState, useEffect } from 'react'

interface Screenshot {
  filePath: string
  dataUrl: string
  timestamp: number
}

export function Gallery() {
  const [screenshots, setScreenshots] = useState<Screenshot[]>([])
  const [selected, setSelected] = useState<Screenshot | null>(null)

  useEffect(() => {
    // รับ screenshot ใหม่จาก main process
    window.electronAPI.onNewCapture((result: Screenshot) => {
      setScreenshots(prev => [result, ...prev])
      setSelected(result)
    })
  }, [])

  const handleCopy = (screenshot: Screenshot) => {
    window.electronAPI.copyToClipboard(screenshot.dataUrl)
  }

  const handleUpload = async (screenshot: Screenshot) => {
    const result = await window.electronAPI.uploadImgur(screenshot.dataUrl)
    if (result.success) {
      alert(`อัพโหลดสำเร็จ!\nURL: ${result.url}`)
      navigator.clipboard.writeText(result.url)
    }
  }

  return (
    <div className="gallery">
      <div className="gallery-sidebar">
        <h3>ภาพทั้งหมด ({screenshots.length})</h3>
        <div className="thumbnail-grid">
          {screenshots.map(s => (
            <div
              key={s.timestamp}
              className={`thumbnail ${selected?.timestamp === s.timestamp ? 'selected' : ''}`}
              onClick={() => setSelected(s)}
            >
              <img src={s.dataUrl} alt="" />
              <div className="thumbnail-time">
                {new Date(s.timestamp).toLocaleTimeString('th-TH')}
              </div>
            </div>
          ))}
        </div>
      </div>

      <div className="gallery-main">
        {selected ? (
          <>
            <div className="preview-area">
              <img src={selected.dataUrl} alt="screenshot" className="preview-image" />
            </div>
            <div className="action-bar">
              <button onClick={() => handleCopy(selected)}>📋 คัดลอก</button>
              <button onClick={() => handleUpload(selected)}>☁️ อัพโหลด Imgur</button>
              <span className="file-path">{selected.filePath}</span>
            </div>
          </>
        ) : (
          <div className="empty-state">
            <p>ใช้ Ctrl+Shift+S เพื่อจับภาพหน้าจอ</p>
            <p>ใช้ Ctrl+Shift+R เพื่อเลือกพื้นที่</p>
          </div>
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
| Full screen capture | จับภาพทั้งหน้าจอ |
| Region selection | เลือกพื้นที่ด้วย overlay |
| Delay capture | รอ 3/5/10 วินาที |
| Arrow annotation | วาดลูกศรบนภาพ |
| Text annotation | เพิ่มข้อความ |
| Blur/Censor | เบลอพื้นที่ sensitive |
| Copy to clipboard | คัดลอกเป็น image |
| Save to disk | บันทึก PNG |
| Upload to Imgur | อัพโหลดและได้ URL |
| Tray menu | เข้าถึงได้จาก tray |
| Global hotkeys | Ctrl+Shift+S/R |
