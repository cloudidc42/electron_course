# Part 79: Testing Best Practices

## การทดสอบ Electron App อย่างมืออาชีพ

ในบทนี้เราจะเรียน unit testing ด้วย Vitest, E2E testing ด้วย Playwright, page objects, mocking IPC และ CI integration

---

## 1. Project Structure สำหรับ Testing

```
src/
├── main/
│   ├── fileManager.ts
│   └── __tests__/
│       └── fileManager.test.ts
├── renderer/
│   ├── components/
│   │   └── FileList.tsx
│   └── __tests__/
│       └── FileList.test.tsx
└── shared/
    ├── utils.ts
    └── __tests__/
        └── utils.test.ts
tests/
└── e2e/
    ├── fixtures/
    ├── pages/
    │   ├── MainPage.ts
    │   └── SettingsPage.ts
    └── specs/
        ├── app.spec.ts
        └── file-operations.spec.ts
```

---

## 2. Vitest Setup

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import * as path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      include: ['src/**/*.{ts,tsx}'],
      exclude: ['src/**/*.d.ts', 'src/test/**']
    }
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src/renderer'),
      '@shared': path.resolve(__dirname, 'src/shared')
    }
  }
})
```

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom'
import { vi } from 'vitest'

// Mock window.electronAPI
Object.defineProperty(window, 'electronAPI', {
  value: {
    invoke: vi.fn(),
    send: vi.fn(),
    on: vi.fn().mockReturnValue(() => {}),
  },
  writable: true
})

// Mock matchMedia
Object.defineProperty(window, 'matchMedia', {
  value: vi.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  }))
})
```

---

## 3. Unit Tests สำหรับ Main Process

```typescript
// src/main/__tests__/fileManager.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import * as fs from 'fs/promises'
import * as path from 'path'
import * as os from 'os'

// Mock electron
vi.mock('electron', () => ({
  app: {
    getPath: vi.fn((name: string) => {
      if (name === 'userData') return '/tmp/test-user-data'
      return '/tmp'
    })
  },
  ipcMain: {
    handle: vi.fn()
  }
}))

import { FileManager } from '../fileManager'

describe('FileManager', () => {
  let fm: FileManager
  let tempDir: string

  beforeEach(async () => {
    tempDir = await fs.mkdtemp(path.join(os.tmpdir(), 'fm-test-'))
    fm = new FileManager()
  })

  afterEach(async () => {
    await fs.rm(tempDir, { recursive: true, force: true })
  })

  describe('readFile', () => {
    it('should read existing file', async () => {
      const filePath = path.join(tempDir, 'test.txt')
      await fs.writeFile(filePath, 'hello world')

      const content = await fm.readFile(filePath)
      expect(content).toBe('hello world')
    })

    it('should throw on non-existent file', async () => {
      await expect(fm.readFile('/nonexistent/path')).rejects.toThrow()
    })

    it('should throw on directory', async () => {
      await expect(fm.readFile(tempDir)).rejects.toThrow()
    })
  })

  describe('writeFile', () => {
    it('should write file', async () => {
      const filePath = path.join(tempDir, 'output.txt')
      await fm.writeFile(filePath, 'content')

      const content = await fs.readFile(filePath, 'utf-8')
      expect(content).toBe('content')
    })

    it('should create directories if needed', async () => {
      const filePath = path.join(tempDir, 'sub', 'dir', 'file.txt')
      await fm.writeFile(filePath, 'test')

      expect(await fs.access(filePath).then(() => true).catch(() => false)).toBe(true)
    })
  })

  describe('listDirectory', () => {
    it('should list files in directory', async () => {
      await fs.writeFile(path.join(tempDir, 'a.txt'), '')
      await fs.writeFile(path.join(tempDir, 'b.txt'), '')
      await fs.mkdir(path.join(tempDir, 'subdir'))

      const entries = await fm.listDirectory(tempDir)
      expect(entries).toHaveLength(3)
      expect(entries.find(e => e.name === 'a.txt')).toBeTruthy()
      expect(entries.find(e => e.name === 'subdir')?.isDirectory).toBe(true)
    })

    it('should return empty array for empty directory', async () => {
      const entries = await fm.listDirectory(tempDir)
      expect(entries).toHaveLength(0)
    })
  })
})
```

---

## 4. React Component Tests

```tsx
// src/renderer/__tests__/FileList.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import { vi, describe, it, expect, beforeEach } from 'vitest'
import userEvent from '@testing-library/user-event'
import { FileList } from '../components/FileList'

const mockFiles = [
  { name: 'document.txt', path: '/home/user/document.txt', size: 1024, isDirectory: false, modified: new Date() },
  { name: 'images', path: '/home/user/images', size: 0, isDirectory: true, modified: new Date() },
  { name: 'photo.jpg', path: '/home/user/photo.jpg', size: 204800, isDirectory: false, modified: new Date() }
]

describe('FileList', () => {
  const mockOnOpen = vi.fn()
  const mockOnSelect = vi.fn()

  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('renders all files', () => {
    render(<FileList files={mockFiles} onOpen={mockOnOpen} onSelect={mockOnSelect} />)

    expect(screen.getByText('document.txt')).toBeInTheDocument()
    expect(screen.getByText('images')).toBeInTheDocument()
    expect(screen.getByText('photo.jpg')).toBeInTheDocument()
  })

  it('calls onOpen when double-clicking a file', async () => {
    render(<FileList files={mockFiles} onOpen={mockOnOpen} onSelect={mockOnSelect} />)

    const docItem = screen.getByText('document.txt').closest('[data-testid="file-item"]')!
    await userEvent.dblClick(docItem)

    expect(mockOnOpen).toHaveBeenCalledWith(mockFiles[0])
  })

  it('calls onSelect on single click', async () => {
    render(<FileList files={mockFiles} onOpen={mockOnOpen} onSelect={mockOnSelect} />)

    const docItem = screen.getByText('document.txt').closest('[data-testid="file-item"]')!
    await userEvent.click(docItem)

    expect(mockOnSelect).toHaveBeenCalledWith([mockFiles[0]])
  })

  it('shows directory icon for directories', () => {
    render(<FileList files={mockFiles} onOpen={mockOnOpen} onSelect={mockOnSelect} />)

    const dirItem = screen.getByText('images').closest('[data-testid="file-item"]')!
    expect(dirItem.querySelector('[data-icon="directory"]')).toBeInTheDocument()
  })

  it('formats file size correctly', () => {
    render(<FileList files={mockFiles} onOpen={mockOnOpen} onSelect={mockOnSelect} />)

    expect(screen.getByText('1.0 KB')).toBeInTheDocument()
    expect(screen.getByText('200.0 KB')).toBeInTheDocument()
  })

  it('supports keyboard navigation', async () => {
    render(<FileList files={mockFiles} onOpen={mockOnOpen} onSelect={mockOnSelect} />)

    const list = screen.getByRole('list')
    list.focus()

    fireEvent.keyDown(list, { key: 'ArrowDown' })
    expect(mockOnSelect).toHaveBeenLastCalledWith([mockFiles[0]])

    fireEvent.keyDown(list, { key: 'ArrowDown' })
    expect(mockOnSelect).toHaveBeenLastCalledWith([mockFiles[1]])

    fireEvent.keyDown(list, { key: 'Enter' })
    expect(mockOnOpen).toHaveBeenCalledWith(mockFiles[1])
  })
})
```

---

## 5. E2E Tests ด้วย Playwright

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: 'tests/e2e/specs',
  timeout: 30_000,
  expect: { timeout: 5_000 },
  reporter: [['html', { outputFolder: 'tests/e2e/report' }]],
  use: {
    // สำหรับ Electron ใช้ custom launcher
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'on-first-retry'
  }
})
```

```typescript
// tests/e2e/fixtures/electron.ts
import { test as base, ElectronApplication, Page, _electron as electron } from '@playwright/test'
import * as path from 'path'

type ElectronFixtures = {
  electronApp: ElectronApplication
  mainPage: Page
}

export const test = base.extend<ElectronFixtures>({
  electronApp: async ({}, use) => {
    const app = await electron.launch({
      args: [path.join(__dirname, '../../../out/main/index.js')],
      env: {
        ...process.env,
        NODE_ENV: 'test',
        ELECTRON_IS_TEST: '1'
      }
    })

    await use(app)
    await app.close()
  },

  mainPage: async ({ electronApp }, use) => {
    const page = await electronApp.firstWindow()
    await page.waitForLoadState('domcontentloaded')
    await use(page)
  }
})

export { expect } from '@playwright/test'
```

---

## 6. Page Object Pattern

```typescript
// tests/e2e/pages/MainPage.ts
import { Page, Locator } from '@playwright/test'

export class MainPage {
  readonly page: Page
  readonly newFileBtn: Locator
  readonly openFileBtn: Locator
  readonly saveBtn: Locator
  readonly editor: Locator
  readonly statusBar: Locator
  readonly tabBar: Locator

  constructor(page: Page) {
    this.page = page
    this.newFileBtn = page.locator('[data-testid="new-file"]')
    this.openFileBtn = page.locator('[data-testid="open-file"]')
    this.saveBtn = page.locator('[data-testid="save-file"]')
    this.editor = page.locator('.monaco-editor')
    this.statusBar = page.locator('[data-testid="status-bar"]')
    this.tabBar = page.locator('[data-testid="tab-bar"]')
  }

  async typeInEditor(text: string): Promise<void> {
    await this.editor.click()
    await this.page.keyboard.type(text)
  }

  async pressShortcut(keys: string): Promise<void> {
    await this.page.keyboard.press(keys)
  }

  async getEditorContent(): Promise<string> {
    return this.page.evaluate(() => {
      // Monaco Editor API
      const editor = (window as Window & { monaco?: { editor: { getModels: () => { getValue: () => string }[] } } }).monaco?.editor?.getModels()[0]
      return editor?.getValue() ?? ''
    })
  }

  async getCurrentFile(): Promise<string | null> {
    return this.statusBar.locator('[data-testid="current-file"]').textContent()
  }

  async getTabCount(): Promise<number> {
    return this.tabBar.locator('[data-testid="tab"]').count()
  }
}
```

```typescript
// tests/e2e/specs/app.spec.ts
import { test, expect } from '../fixtures/electron'
import { MainPage } from '../pages/MainPage'

test.describe('App Launch', () => {
  test('should show main window', async ({ mainPage }) => {
    expect(await mainPage.title()).toBeTruthy()
    await expect(mainPage).toHaveTitle(/My App/)
  })

  test('should have correct window size', async ({ electronApp }) => {
    const window = await electronApp.firstWindow()
    const size = await window.evaluate(() => ({
      width: window.innerWidth,
      height: window.innerHeight
    }))
    expect(size.width).toBeGreaterThan(800)
    expect(size.height).toBeGreaterThan(500)
  })
})

test.describe('File Operations', () => {
  test('should create new file', async ({ mainPage }) => {
    const page = new MainPage(mainPage)

    await page.newFileBtn.click()
    expect(await page.getTabCount()).toBe(1)
    expect(await page.getCurrentFile()).toBe('Untitled')
  })

  test('should save file with Ctrl+S', async ({ mainPage }) => {
    const page = new MainPage(mainPage)

    await page.newFileBtn.click()
    await page.typeInEditor('const x = 42')
    await page.pressShortcut('Control+s')

    // Verify save dialog appeared (or file was saved)
    const title = await mainPage.title()
    expect(title).not.toContain('●')  // ● แสดงว่ายังไม่ได้ save
  })
})
```

---

## 7. IPC Mock Helper

```typescript
// src/test/ipcMock.ts
import { vi } from 'vitest'

type MockHandlers = Record<string, (...args: unknown[]) => unknown>

export function createIPCMock(handlers: MockHandlers = {}) {
  const mockInvoke = vi.fn((channel: string, ...args: unknown[]) => {
    const handler = handlers[channel]
    if (handler) return Promise.resolve(handler(...args))
    return Promise.resolve(null)
  })

  const listeners = new Map<string, Set<(data: unknown) => void>>()

  const mockOn = vi.fn((channel: string, cb: (data: unknown) => void) => {
    if (!listeners.has(channel)) listeners.set(channel, new Set())
    listeners.get(channel)!.add(cb)
    return () => listeners.get(channel)?.delete(cb)
  })

  // Helper: simulate event from main
  const emit = (channel: string, data: unknown) => {
    listeners.get(channel)?.forEach(cb => cb(data))
  }

  Object.defineProperty(window, 'electronAPI', {
    value: { invoke: mockInvoke, on: mockOn, send: vi.fn() },
    writable: true
  })

  return { mockInvoke, mockOn, emit }
}

// Usage in test:
// const { mockInvoke, emit } = createIPCMock({
//   'file:open': () => ({ path: '/test.txt', content: 'hello' })
// })
```

---

## สรุป

| ประเภท Test | เครื่องมือ | ครอบคลุม |
|-----------|---------|---------|
| Unit (Main) | Vitest + fs mocks | Business logic |
| Unit (Renderer) | Vitest + RTL | Components |
| Integration | Vitest + IPC mocks | IPC flows |
| E2E | Playwright Electron | Full app behavior |
| Page Objects | Playwright | Maintainable selectors |
| Coverage | V8 + lcov | Code coverage report |
