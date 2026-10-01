# Part 82: Advanced Window Management

## จัดการ Windows แบบขั้นสูง

ในบทนี้เราจะเรียน multi-monitor support, window snap layouts, floating windows, PiP (Picture-in-Picture) และ window state persistence

---

## 1. Multi-Monitor Window Management

```typescript
// src/main/windowManager.ts
import { BrowserWindow, screen, Rectangle } from 'electron'
import ElectronStore from 'electron-store'

interface WindowState {
  x: number
  y: number
  width: number
  height: number
  maximized: boolean
  displayId: number
}

const store = new ElectronStore<{ windowStates: Record<string, WindowState> }>()

export class WindowManager {
  private windows = new Map<string, BrowserWindow>()

  // สร้าง window ที่จำ position
  createWindow(id: string, options: Electron.BrowserWindowConstructorOptions): BrowserWindow {
    const savedState = this.getSavedState(id)
    const validBounds = this.ensureVisibleOnScreen(savedState)

    const win = new BrowserWindow({
      ...options,
      ...validBounds,
      show: false  // ไม่แสดงก่อน ready
    })

    if (savedState?.maximized) {
      win.maximize()
    }

    win.once('ready-to-show', () => win.show())

    // บันทึก state เมื่อเปลี่ยนแปลง
    this.trackWindowState(id, win)

    this.windows.set(id, win)
    win.on('closed', () => this.windows.delete(id))

    return win
  }

  private getSavedState(id: string): WindowState | undefined {
    const states = store.get('windowStates', {})
    return states[id]
  }

  private saveState(id: string, state: WindowState): void {
    const states = store.get('windowStates', {})
    states[id] = state
    store.set('windowStates', states)
  }

  // ตรวจสอบว่า window ยังอยู่บน screen
  private ensureVisibleOnScreen(state?: WindowState): Partial<WindowState> {
    if (!state) return {}

    const displays = screen.getAllDisplays()

    // ตรวจสอบว่า window ยังอยู่บน display ที่มีอยู่
    const targetDisplay = displays.find(d => d.id === state.displayId)

    if (targetDisplay) {
      const { bounds } = targetDisplay.workArea as unknown as { bounds: Rectangle }
      const workArea = targetDisplay.workArea

      // ตรวจสอบว่าอยู่ใน work area
      if (
        state.x >= workArea.x &&
        state.y >= workArea.y &&
        state.x + state.width <= workArea.x + workArea.width &&
        state.y + state.height <= workArea.y + workArea.height
      ) {
        return state
      }
    }

    // Default ไปยัง primary display center
    const primary = screen.getPrimaryDisplay()
    const { width, height } = state
    return {
      x: Math.floor((primary.workArea.width - width) / 2) + primary.workArea.x,
      y: Math.floor((primary.workArea.height - height) / 2) + primary.workArea.y,
      width,
      height
    }
  }

  private trackWindowState(id: string, win: BrowserWindow): void {
    let saveTimeout: NodeJS.Timeout | null = null

    const saveState = () => {
      if (saveTimeout) clearTimeout(saveTimeout)
      saveTimeout = setTimeout(() => {
        if (win.isDestroyed()) return
        const [x, y] = win.getPosition()
        const [width, height] = win.getSize()
        const display = screen.getDisplayNearestPoint({ x, y })

        this.saveState(id, {
          x, y, width, height,
          maximized: win.isMaximized(),
          displayId: display.id
        })
      }, 500)
    }

    win.on('resize', saveState)
    win.on('move', saveState)
    win.on('maximize', saveState)
    win.on('unmaximize', saveState)
  }

  // ย้าย window ไปยัง display อื่น
  moveToDisplay(windowId: string, displayId: number): void {
    const win = this.windows.get(windowId)
    if (!win) return

    const display = screen.getAllDisplays().find(d => d.id === displayId)
    if (!display) return

    const { workArea } = display
    const [width, height] = win.getSize()

    win.setPosition(
      Math.floor(workArea.x + (workArea.width - width) / 2),
      Math.floor(workArea.y + (workArea.height - height) / 2)
    )
  }

  // Snap window ไปยังตำแหน่งต่างๆ
  snapWindow(windowId: string, position: 'left' | 'right' | 'top' | 'bottom' | 'center' | 'maximize'): void {
    const win = this.windows.get(windowId)
    if (!win) return

    const display = screen.getDisplayNearestPoint(win.getBounds())
    const { x, y, width, height } = display.workArea

    const snaps: Record<typeof position, Rectangle> = {
      left: { x, y, width: Math.floor(width / 2), height },
      right: { x: x + Math.floor(width / 2), y, width: Math.floor(width / 2), height },
      top: { x, y, width, height: Math.floor(height / 2) },
      bottom: { x, y: y + Math.floor(height / 2), width, height: Math.floor(height / 2) },
      center: {
        x: x + Math.floor(width * 0.1),
        y: y + Math.floor(height * 0.1),
        width: Math.floor(width * 0.8),
        height: Math.floor(height * 0.8)
      },
      maximize: { x, y, width, height }
    }

    const target = snaps[position]
    win.setBounds(target, true)  // animate = true
  }

  getWindow(id: string): BrowserWindow | undefined {
    return this.windows.get(id)
  }

  getAllWindows(): Map<string, BrowserWindow> {
    return new Map(this.windows)
  }
}

export const windowManager = new WindowManager()
```

---

## 2. Picture-in-Picture (Floating Window)

```typescript
// src/main/pipWindow.ts
import { BrowserWindow, screen } from 'electron'
import * as path from 'path'

let pipWindow: BrowserWindow | null = null

export function createPIPWindow(options: {
  width?: number
  height?: number
  corner?: 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right'
}): BrowserWindow {
  if (pipWindow && !pipWindow.isDestroyed()) {
    pipWindow.focus()
    return pipWindow
  }

  const { width = 320, height = 180, corner = 'bottom-right' } = options
  const display = screen.getPrimaryDisplay()
  const { workArea } = display
  const margin = 20

  // คำนวณ position ตาม corner
  const positions: Record<typeof corner, { x: number; y: number }> = {
    'top-left': { x: workArea.x + margin, y: workArea.y + margin },
    'top-right': { x: workArea.x + workArea.width - width - margin, y: workArea.y + margin },
    'bottom-left': { x: workArea.x + margin, y: workArea.y + workArea.height - height - margin },
    'bottom-right': { x: workArea.x + workArea.width - width - margin, y: workArea.y + workArea.height - height - margin }
  }

  const { x, y } = positions[corner]

  pipWindow = new BrowserWindow({
    width,
    height,
    x,
    y,
    alwaysOnTop: true,
    frame: false,
    transparent: true,
    resizable: true,
    skipTaskbar: true,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true
    }
  })

  pipWindow.setAspectRatio(16 / 9)
  pipWindow.setVisibleOnAllWorkspaces(true, { visibleOnFullScreen: true })

  pipWindow.loadURL(
    process.env.NODE_ENV === 'development'
      ? 'http://localhost:5173/pip'
      : `file://${path.join(__dirname, '../renderer/index.html')}#/pip`
  )

  pipWindow.on('closed', () => { pipWindow = null })

  return pipWindow
}

export function closePIPWindow(): void {
  pipWindow?.close()
  pipWindow = null
}
```

---

## 3. Window Snap Overlay (Windows 11 style)

```tsx
// src/renderer/components/SnapOverlay.tsx
import { useState, useCallback } from 'react'

type SnapZone = 'left' | 'right' | 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right' | 'center'

const snapZones: { id: SnapZone; style: React.CSSProperties }[] = [
  { id: 'left', style: { left: 0, top: 0, width: '50%', height: '100%' } },
  { id: 'right', style: { right: 0, top: 0, width: '50%', height: '100%' } },
  { id: 'top-left', style: { left: 0, top: 0, width: '50%', height: '50%' } },
  { id: 'top-right', style: { right: 0, top: 0, width: '50%', height: '50%' } },
  { id: 'bottom-left', style: { left: 0, bottom: 0, width: '50%', height: '50%' } },
  { id: 'bottom-right', style: { right: 0, bottom: 0, width: '50%', height: '50%' } },
  { id: 'center', style: { left: '25%', top: '10%', width: '50%', height: '80%' } }
]

export function SnapOverlay({ visible, onSnap }: {
  visible: boolean
  onSnap: (zone: SnapZone) => void
}) {
  const [hoveredZone, setHoveredZone] = useState<SnapZone | null>(null)

  if (!visible) return null

  return (
    <div className="snap-overlay">
      {snapZones.map(zone => (
        <div
          key={zone.id}
          className={`snap-zone ${hoveredZone === zone.id ? 'active' : ''}`}
          style={{ position: 'absolute', ...zone.style }}
          onMouseEnter={() => setHoveredZone(zone.id)}
          onMouseLeave={() => setHoveredZone(null)}
          onClick={() => onSnap(zone.id)}
        />
      ))}
    </div>
  )
}
```

---

## 4. Multi-Window IPC Setup

```typescript
// src/main/multiWindowIPC.ts
import { ipcMain, BrowserWindow } from 'electron'
import { windowManager } from './windowManager'

export function registerWindowHandlers(): void {
  ipcMain.handle('window:snap', (event, { position }: { position: string }) => {
    const win = BrowserWindow.fromWebContents(event.sender)
    if (!win) return

    const winId = getWindowId(win)
    if (winId) {
      windowManager.snapWindow(winId, position as Parameters<typeof windowManager.snapWindow>[1])
    }
  })

  ipcMain.handle('window:move-to-display', (event, { displayId }: { displayId: number }) => {
    const win = BrowserWindow.fromWebContents(event.sender)
    if (!win) return

    const winId = getWindowId(win)
    if (winId) windowManager.moveToDisplay(winId, displayId)
  })

  ipcMain.handle('window:get-displays', () => {
    const { screen } = require('electron')
    return screen.getAllDisplays().map((d: Electron.Display) => ({
      id: d.id,
      bounds: d.bounds,
      workArea: d.workArea,
      scaleFactor: d.scaleFactor,
      primary: d.id === screen.getPrimaryDisplay().id,
      label: d.label || `Display ${d.id}`
    }))
  })

  ipcMain.handle('window:open-pip', (_, opts) => {
    const { createPIPWindow } = require('./pipWindow')
    createPIPWindow(opts)
  })
}

function getWindowId(win: BrowserWindow): string | undefined {
  const all = windowManager.getAllWindows()
  for (const [id, w] of all) {
    if (w === win) return id
  }
  return undefined
}
```

---

## 5. Window State Hook

```typescript
// src/renderer/hooks/useWindowState.ts
import { useEffect, useState, useCallback } from 'react'

interface WindowState {
  isMaximized: boolean
  isMinimized: boolean
  isFullScreen: boolean
  isFocused: boolean
}

export function useWindowState(): WindowState & {
  minimize: () => void
  maximize: () => void
  restore: () => void
  close: () => void
  toggleFullScreen: () => void
} {
  const [state, setState] = useState<WindowState>({
    isMaximized: false,
    isMinimized: false,
    isFullScreen: false,
    isFocused: true
  })

  useEffect(() => {
    const handlers = [
      window.electronAPI.on('window:maximize', () => setState(s => ({ ...s, isMaximized: true }))),
      window.electronAPI.on('window:unmaximize', () => setState(s => ({ ...s, isMaximized: false }))),
      window.electronAPI.on('window:minimize', () => setState(s => ({ ...s, isMinimized: true }))),
      window.electronAPI.on('window:restore', () => setState(s => ({ ...s, isMinimized: false }))),
      window.electronAPI.on('window:focus', () => setState(s => ({ ...s, isFocused: true }))),
      window.electronAPI.on('window:blur', () => setState(s => ({ ...s, isFocused: false })))
    ]

    return () => handlers.forEach(cleanup => cleanup())
  }, [])

  return {
    ...state,
    minimize: useCallback(() => window.electronAPI.send('window:minimize'), []),
    maximize: useCallback(() => window.electronAPI.send('window:maximize'), []),
    restore: useCallback(() => window.electronAPI.send('window:restore'), []),
    close: useCallback(() => window.electronAPI.send('window:close'), []),
    toggleFullScreen: useCallback(() => window.electronAPI.send('window:toggle-fullscreen'), [])
  }
}
```

---

## สรุป

| ฟีเจอร์ | วิธีการ | Note |
|--------|---------|------|
| Window state persistence | electron-store | จำ position/size |
| Multi-monitor | screen.getAllDisplays() | ensureVisibleOnScreen |
| Snap layouts | setBounds() + workArea | animate = true |
| PiP window | alwaysOnTop + transparent | setAspectRatio |
| All workspaces | setVisibleOnAllWorkspaces | macOS |
| Frame-less | frame: false | custom titlebar |
