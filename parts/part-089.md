# Part 89: Custom Native Menus Advanced

## สร้าง Native Menus ขั้นสูงใน Electron

ในบทนี้เราจะเรียน dynamic menu building, role-based menus, context menus, dock menus (macOS) และ custom menu bars

---

## 1. Dynamic Menu Builder

```typescript
// src/main/menuBuilder.ts
import { Menu, MenuItem, MenuItemConstructorOptions, app, BrowserWindow, shell } from 'electron'

type AppState = {
  hasOpenFile: boolean
  canUndo: boolean
  canRedo: boolean
  isMaximized: boolean
  recentFiles: string[]
  zoom: number
}

export class MenuBuilder {
  private win: BrowserWindow
  private state: AppState

  constructor(win: BrowserWindow) {
    this.win = win
    this.state = {
      hasOpenFile: false,
      canUndo: false,
      canRedo: false,
      isMaximized: false,
      recentFiles: [],
      zoom: 100
    }
  }

  updateState(updates: Partial<AppState>): void {
    this.state = { ...this.state, ...updates }
    this.rebuild()
  }

  rebuild(): void {
    const template = this.buildTemplate()
    const menu = Menu.buildFromTemplate(template)
    Menu.setApplicationMenu(menu)
  }

  private buildTemplate(): MenuItemConstructorOptions[] {
    const isMac = process.platform === 'darwin'

    const template: MenuItemConstructorOptions[] = [
      // macOS App menu
      ...(isMac ? [{
        label: app.name,
        submenu: [
          { role: 'about' as const },
          { type: 'separator' as const },
          {
            label: 'Preferences...',
            accelerator: 'Cmd+,',
            click: () => this.win.webContents.send('menu:preferences')
          },
          { type: 'separator' as const },
          { role: 'services' as const },
          { type: 'separator' as const },
          { role: 'hide' as const },
          { role: 'hideOthers' as const },
          { role: 'unhide' as const },
          { type: 'separator' as const },
          { role: 'quit' as const }
        ]
      }] : []),

      // File menu
      {
        label: 'ไฟล์',
        submenu: [
          {
            label: 'ใหม่',
            accelerator: 'CmdOrCtrl+N',
            click: () => this.win.webContents.send('menu:new-file')
          },
          {
            label: 'เปิด...',
            accelerator: 'CmdOrCtrl+O',
            click: () => this.win.webContents.send('menu:open-file')
          },
          {
            label: 'เปิดล่าสุด',
            submenu: this.buildRecentFilesMenu()
          },
          { type: 'separator' },
          {
            label: 'บันทึก',
            accelerator: 'CmdOrCtrl+S',
            enabled: this.state.hasOpenFile,
            click: () => this.win.webContents.send('menu:save')
          },
          {
            label: 'บันทึกเป็น...',
            accelerator: 'CmdOrCtrl+Shift+S',
            click: () => this.win.webContents.send('menu:save-as')
          },
          { type: 'separator' },
          ...(isMac ? [] : [{ role: 'quit' as const, label: 'ออก' }])
        ]
      },

      // Edit menu
      {
        label: 'แก้ไข',
        submenu: [
          {
            label: 'เลิกทำ',
            accelerator: 'CmdOrCtrl+Z',
            enabled: this.state.canUndo,
            click: () => this.win.webContents.send('menu:undo')
          },
          {
            label: 'ทำซ้ำ',
            accelerator: 'CmdOrCtrl+Shift+Z',
            enabled: this.state.canRedo,
            click: () => this.win.webContents.send('menu:redo')
          },
          { type: 'separator' },
          { role: 'cut', label: 'ตัด' },
          { role: 'copy', label: 'คัดลอก' },
          { role: 'paste', label: 'วาง' },
          { role: 'selectAll', label: 'เลือกทั้งหมด' },
          { type: 'separator' },
          {
            label: 'ค้นหา...',
            accelerator: 'CmdOrCtrl+F',
            click: () => this.win.webContents.send('menu:find')
          },
          {
            label: 'แทนที่...',
            accelerator: 'CmdOrCtrl+H',
            click: () => this.win.webContents.send('menu:replace')
          }
        ]
      },

      // View menu
      {
        label: 'มุมมอง',
        submenu: [
          {
            label: 'ซูมเข้า',
            accelerator: 'CmdOrCtrl+=',
            click: () => this.setZoom(this.state.zoom + 10)
          },
          {
            label: 'ซูมออก',
            accelerator: 'CmdOrCtrl+-',
            click: () => this.setZoom(this.state.zoom - 10)
          },
          {
            label: 'รีเซ็ตขนาด',
            accelerator: 'CmdOrCtrl+0',
            click: () => this.setZoom(100)
          },
          { type: 'separator' },
          {
            label: `ขนาด ${this.state.zoom}%`,
            enabled: false
          },
          { type: 'separator' },
          { role: 'togglefullscreen', label: 'เต็มจอ' },
          { role: 'toggleDevTools', label: 'เครื่องมือนักพัฒนา' }
        ]
      },

      // Help menu
      {
        label: 'ช่วยเหลือ',
        submenu: [
          {
            label: 'เอกสาร',
            click: () => shell.openExternal('https://docs.example.com')
          },
          {
            label: 'รายงานปัญหา',
            click: () => shell.openExternal('https://github.com/example/app/issues')
          },
          ...(isMac ? [] : [
            { type: 'separator' as const },
            { role: 'about' as const, label: 'เกี่ยวกับ' }
          ])
        ]
      }
    ]

    return template
  }

  private buildRecentFilesMenu(): MenuItemConstructorOptions[] {
    if (this.state.recentFiles.length === 0) {
      return [{ label: 'ไม่มีไฟล์ล่าสุด', enabled: false }]
    }

    return [
      ...this.state.recentFiles.slice(0, 10).map(filePath => ({
        label: require('path').basename(filePath),
        sublabel: filePath,
        click: () => this.win.webContents.send('menu:open-recent', { path: filePath })
      })),
      { type: 'separator' as const },
      {
        label: 'ล้างรายการล่าสุด',
        click: () => {
          this.updateState({ recentFiles: [] })
          this.win.webContents.send('menu:clear-recent')
        }
      }
    ]
  }

  private setZoom(zoom: number): void {
    const clamped = Math.max(50, Math.min(200, zoom))
    this.win.webContents.setZoomFactor(clamped / 100)
    this.updateState({ zoom: clamped })
  }
}
```

---

## 2. Context Menu

```typescript
// src/main/contextMenu.ts
import { Menu, MenuItemConstructorOptions, ipcMain, BrowserWindow } from 'electron'

export function registerContextMenuHandlers(win: BrowserWindow): void {
  ipcMain.on('show-context-menu', (event, {
    x, y, type, data
  }: {
    x: number; y: number
    type: 'file' | 'editor' | 'tab' | 'default'
    data?: Record<string, unknown>
  }) => {
    let template: MenuItemConstructorOptions[] = []

    switch (type) {
      case 'file':
        template = buildFileContextMenu(data as { path: string; isDir: boolean }, win)
        break
      case 'editor':
        template = buildEditorContextMenu(win)
        break
      case 'tab':
        template = buildTabContextMenu(data as { tabId: string }, win)
        break
      default:
        template = [
          { role: 'cut', label: 'ตัด' },
          { role: 'copy', label: 'คัดลอก' },
          { role: 'paste', label: 'วาง' }
        ]
    }

    const menu = Menu.buildFromTemplate(template)
    menu.popup({ window: win, x, y })
  })
}

function buildFileContextMenu(
  data: { path: string; isDir: boolean },
  win: BrowserWindow
): MenuItemConstructorOptions[] {
  const { path: filePath, isDir } = data
  const send = (event: string, payload?: unknown) => win.webContents.send(event, payload)

  return [
    ...(isDir ? [
      {
        label: 'สร้างไฟล์ใหม่',
        click: () => send('ctx:new-file', { parent: filePath })
      },
      {
        label: 'สร้างโฟลเดอร์ใหม่',
        click: () => send('ctx:new-folder', { parent: filePath })
      },
      { type: 'separator' as const }
    ] : [
      {
        label: 'เปิด',
        click: () => send('ctx:open', { path: filePath })
      },
      {
        label: 'เปิดด้วยแอปอื่น',
        click: () => require('electron').shell.openPath(filePath)
      },
      { type: 'separator' as const }
    ]),
    {
      label: 'คัดลอก',
      click: () => send('ctx:copy', { path: filePath })
    },
    {
      label: 'ตัด',
      click: () => send('ctx:cut', { path: filePath })
    },
    { type: 'separator' },
    {
      label: 'เปลี่ยนชื่อ',
      click: () => send('ctx:rename', { path: filePath })
    },
    {
      label: 'ลบ (ถังขยะ)',
      click: () => send('ctx:trash', { path: filePath })
    },
    { type: 'separator' },
    {
      label: 'แสดงใน Explorer',
      click: () => require('electron').shell.showItemInFolder(filePath)
    },
    {
      label: 'คัดลอก Path',
      click: () => require('electron').clipboard.writeText(filePath)
    }
  ]
}

function buildEditorContextMenu(win: BrowserWindow): MenuItemConstructorOptions[] {
  return [
    { role: 'cut', label: 'ตัด' },
    { role: 'copy', label: 'คัดลอก' },
    { role: 'paste', label: 'วาง' },
    { type: 'separator' },
    {
      label: 'ค้นหา...',
      click: () => win.webContents.send('menu:find')
    },
    {
      label: 'จัดรูปแบบเอกสาร',
      click: () => win.webContents.send('ctx:format')
    }
  ]
}

function buildTabContextMenu(
  data: { tabId: string },
  win: BrowserWindow
): MenuItemConstructorOptions[] {
  return [
    {
      label: 'ปิดแท็บ',
      click: () => win.webContents.send('ctx:close-tab', data)
    },
    {
      label: 'ปิดแท็บอื่นๆ',
      click: () => win.webContents.send('ctx:close-other-tabs', data)
    },
    {
      label: 'ปิดแท็บทั้งหมด',
      click: () => win.webContents.send('ctx:close-all-tabs')
    },
    { type: 'separator' },
    {
      label: 'คัดลอก Path',
      click: () => win.webContents.send('ctx:copy-path', data)
    }
  ]
}
```

---

## 3. Dock Menu (macOS)

```typescript
// src/main/dockMenu.ts
import { app, Menu } from 'electron'

export function setupDockMenu(createWindow: () => void): void {
  if (process.platform !== 'darwin') return

  const dockMenu = Menu.buildFromTemplate([
    {
      label: 'หน้าต่างใหม่',
      click: () => createWindow()
    },
    { type: 'separator' },
    {
      label: 'ล้าง Recent Files',
      click: () => app.clearRecentDocuments()
    }
  ])

  app.dock.setMenu(dockMenu)
}

// เพิ่ม recent document (macOS)
export function addRecentDocument(filePath: string): void {
  app.addRecentDocument(filePath)
}

// Badge number บน dock icon
export function setDockBadge(count: number): void {
  if (process.platform !== 'darwin') return
  app.dock.setBadge(count > 0 ? String(count) : '')
}
```

---

## 4. Touchbar (macOS)

```typescript
// src/main/touchbar.ts
import { TouchBar, BrowserWindow } from 'electron'

const { TouchBarButton, TouchBarSpacer, TouchBarLabel } = TouchBar

export function setupTouchBar(win: BrowserWindow): void {
  if (process.platform !== 'darwin') return

  const touchBar = new TouchBar({
    items: [
      new TouchBarButton({
        label: '📄 ใหม่',
        click: () => win.webContents.send('menu:new-file')
      }),
      new TouchBarButton({
        label: '📂 เปิด',
        click: () => win.webContents.send('menu:open-file')
      }),
      new TouchBarButton({
        label: '💾 บันทึก',
        click: () => win.webContents.send('menu:save')
      }),
      new TouchBarSpacer({ size: 'flexible' }),
      new TouchBarLabel({ label: 'My Editor' })
    ]
  })

  win.setTouchBar(touchBar)
}
```

---

## สรุป

| ฟีเจอร์ | API | หมายเหตุ |
|--------|-----|---------|
| Application Menu | Menu.buildFromTemplate | ตั้งครั้งเดียว |
| Dynamic rebuild | Menu.setApplicationMenu | เมื่อ state เปลี่ยน |
| Context Menu | menu.popup() | ตาม click position |
| Dock Menu | app.dock.setMenu() | macOS เท่านั้น |
| Recent Documents | app.addRecentDocument() | macOS/Windows |
| TouchBar | new TouchBar() | MacBook Pro |
| Accelerators | CmdOrCtrl+ | cross-platform |
