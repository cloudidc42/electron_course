# Part 98: Documentation & Developer Experience

## สร้าง Documentation และปรับปรุง DX

ในบทนี้เราจะเรียน JSDoc, TypeDoc, Storybook สำหรับ UI components, API documentation และ CLAUDE.md

---

## 1. JSDoc Patterns

```typescript
// src/main/fileManager.ts

/**
 * จัดการ file operations สำหรับ Electron main process
 *
 * @example
 * ```typescript
 * const fm = new FileManager()
 * const content = await fm.readFile('/path/to/file.txt')
 * ```
 */
export class FileManager {
  /**
   * อ่านไฟล์และ return เป็น string
   *
   * @param filePath - absolute path ของไฟล์
   * @param encoding - encoding ของไฟล์ (default: utf-8)
   * @returns เนื้อหาของไฟล์
   * @throws {Error} ถ้าไฟล์ไม่มีหรือไม่มีสิทธิ์อ่าน
   *
   * @example
   * ```typescript
   * const content = await fm.readFile('/home/user/notes.txt')
   * console.log(content) // "Hello World"
   * ```
   */
  async readFile(filePath: string, encoding: BufferEncoding = 'utf-8'): Promise<string> {
    const fs = await import('fs/promises')
    const stat = await fs.stat(filePath)

    if (stat.isDirectory()) {
      throw new Error(`Cannot read directory as file: ${filePath}`)
    }

    return fs.readFile(filePath, encoding)
  }

  /**
   * เขียนข้อมูลลงไฟล์ (สร้าง directories ถ้าไม่มี)
   *
   * @param filePath - absolute path ของไฟล์
   * @param content - เนื้อหาที่จะเขียน
   * @param options - options เพิ่มเติม
   * @param options.createDirs - สร้าง parent directories ถ้าไม่มี (default: true)
   * @param options.atomic - เขียนแบบ atomic (default: true)
   */
  async writeFile(
    filePath: string,
    content: string,
    options?: { createDirs?: boolean; atomic?: boolean }
  ): Promise<void> {
    const { createDirs = true, atomic = true } = options ?? {}
    const path = await import('path')
    const fs = await import('fs/promises')

    if (createDirs) {
      await fs.mkdir(path.dirname(filePath), { recursive: true })
    }

    if (atomic) {
      const { atomicWrite } = await import('./atomicWrite')
      await atomicWrite(filePath, content)
    } else {
      await fs.writeFile(filePath, content, 'utf-8')
    }
  }

  /**
   * แสดงรายการไฟล์และโฟลเดอร์ใน directory
   *
   * @param dirPath - path ของ directory
   * @returns รายการ entries พร้อม metadata
   *
   * @remarks
   * ผลลัพธ์จะเรียงตาม: โฟลเดอร์ก่อน, แล้วตามตัวอักษร
   */
  async listDirectory(dirPath: string): Promise<FileEntry[]> {
    const fs = await import('fs/promises')
    const path = await import('path')
    const entries = await fs.readdir(dirPath, { withFileTypes: true })

    const result: FileEntry[] = await Promise.all(
      entries.map(async entry => {
        const entryPath = path.join(dirPath, entry.name)
        const stat = await fs.stat(entryPath)
        return {
          name: entry.name,
          path: entryPath,
          isDirectory: entry.isDirectory(),
          size: stat.size,
          modified: stat.mtime,
          created: stat.birthtime
        }
      })
    )

    return result.sort((a, b) => {
      if (a.isDirectory !== b.isDirectory) return a.isDirectory ? -1 : 1
      return a.name.localeCompare(b.name)
    })
  }
}

/**
 * Metadata ของไฟล์หรือโฟลเดอร์
 */
export interface FileEntry {
  /** ชื่อไฟล์ */
  name: string
  /** absolute path */
  path: string
  /** true ถ้าเป็น directory */
  isDirectory: boolean
  /** ขนาดไฟล์เป็น bytes (0 สำหรับ directories) */
  size: number
  /** วันที่แก้ไขล่าสุด */
  modified: Date
  /** วันที่สร้าง */
  created: Date
}
```

---

## 2. TypeDoc Configuration

```json
// typedoc.json
{
  "$schema": "https://typedoc.org/schema.json",
  "entryPoints": ["src/main/index.ts", "src/preload/index.ts", "src/shared/index.ts"],
  "entryPointStrategy": "expand",
  "out": "docs/api",
  "name": "My Electron App API",
  "includeVersion": true,
  "sort": ["alphabetical"],
  "categorizeByGroup": true,
  "defaultCategory": "General",
  "plugin": ["typedoc-plugin-markdown"],
  "readme": "README.md",
  "exclude": ["**/*.test.ts", "**/node_modules/**"],
  "excludePrivate": true,
  "excludeProtected": false,
  "navigationLinks": {
    "GitHub": "https://github.com/example/my-app",
    "Docs": "https://docs.example.com"
  }
}
```

---

## 3. Storybook สำหรับ UI Components

```typescript
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/react-vite'

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.{ts,tsx}'],
  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
    '@storybook/addon-a11y'
  ],
  framework: {
    name: '@storybook/react-vite',
    options: {}
  },
  docs: {
    autodocs: 'tag'
  }
}

export default config
```

```typescript
// .storybook/preview.tsx
import type { Preview } from '@storybook/react'
import '../src/renderer/styles/theme.css'
import '../src/renderer/styles/global.css'

const preview: Preview = {
  parameters: {
    backgrounds: {
      default: 'dark',
      values: [
        { name: 'dark', value: '#1e1e1e' },
        { name: 'light', value: '#ffffff' }
      ]
    },
    actions: { argTypesRegex: '^on[A-Z].*' },
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/
      }
    }
  }
}

export default preview
```

```tsx
// src/renderer/components/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react'
import { Button } from './Button'

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'danger', 'ghost']
    },
    size: {
      control: 'select',
      options: ['sm', 'md', 'lg']
    }
  }
}

export default meta
type Story = StoryObj<typeof Button>

export const Primary: Story = {
  args: {
    variant: 'primary',
    children: 'คลิกฉัน'
  }
}

export const WithIcon: Story = {
  args: {
    variant: 'primary',
    icon: '📄',
    children: 'เปิดไฟล์'
  }
}

export const Loading: Story = {
  args: {
    variant: 'primary',
    loading: true,
    children: 'กำลังโหลด'
  }
}

export const AllVariants: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: 8, flexWrap: 'wrap' }}>
      {(['primary', 'secondary', 'danger', 'ghost'] as const).map(v => (
        <Button key={v} variant={v}>{v}</Button>
      ))}
    </div>
  )
}
```

---

## 4. CLAUDE.md สำหรับ AI-assisted Development

```markdown
# CLAUDE.md

## Project Overview
Electron app สำหรับ text editing และ file management

## Architecture
- Main process: src/main/ - Node.js, Electron APIs
- Renderer: src/renderer/ - React 18, TypeScript
- Preload: src/preload/ - contextBridge, IPC bridge
- Shared: src/shared/ - Types ที่ใช้ร่วมกัน

## Key Conventions

### IPC Channels
- ชื่อ format: `domain:action` เช่น `file:open`, `settings:get`
- Define ใน src/shared/ipcTypes.ts
- ต้อง type-safe ทั้ง request และ response

### File Structure
- Components: src/renderer/components/[ComponentName]/
- Hooks: src/renderer/hooks/use[Name].ts
- Services: src/renderer/services/[name]Service.ts
- Main handlers: src/main/handlers/[domain]Handlers.ts

### Testing
- Unit tests: vitest (*.test.ts)
- E2E tests: playwright (tests/e2e/)
- ทุก component ต้องมี test
- Coverage เป้าหมาย: 80%+

## Build Commands
- dev: npm run dev
- test: npm run test
- build: npm run build
- package: npm run package

## Common Patterns
- Error handling: tryAsync wrapper สำหรับ async operations
- State: Zustand สำหรับ global, useState สำหรับ local
- Styling: CSS Modules + CSS variables สำหรับ theme

## Electron-specific
- ใช้ contextBridge เสมอ (ห้าม nodeIntegration)
- ตรวจสอบ sender origin ใน IPC handlers
- Cleanup listeners ใน useEffect return
```

---

## 5. Package.json Scripts Documentation

```json
{
  "scripts": {
    "// dev": "=== Development ===",
    "dev": "electron-vite dev",
    "dev:inspect": "electron-vite dev --inspect",

    "// build": "=== Build ===",
    "build": "electron-vite build",
    "build:analyze": "ANALYZE=true electron-vite build",

    "// test": "=== Testing ===",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:headed": "playwright test --headed",

    "// docs": "=== Documentation ===",
    "docs": "typedoc",
    "docs:watch": "typedoc --watch",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build",

    "// quality": "=== Code Quality ===",
    "lint": "eslint src/ --ext .ts,.tsx",
    "lint:fix": "eslint src/ --ext .ts,.tsx --fix",
    "type-check": "tsc --noEmit",
    "format": "prettier --write \"src/**/*.{ts,tsx,css}\"",
    "audit": "ts-node scripts/securityAudit.ts"
  }
}
```

---

## สรุป

| เครื่องมือ | หน้าที่ | Output |
|----------|---------|--------|
| JSDoc | code documentation | IDE tooltips |
| TypeDoc | API reference | HTML/Markdown |
| Storybook | UI component docs | interactive catalog |
| CLAUDE.md | AI context | better AI assistance |
| README | project overview | onboarding |
| Changelog | version history | user-facing |
