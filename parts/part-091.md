# Part 91: Monorepo Setup

## จัดการ Electron Project แบบ Monorepo

ในบทนี้เราจะเรียน Turborepo, pnpm workspaces, shared packages, code sharing ระหว่าง packages และ build pipeline

---

## 1. Monorepo Structure

```
my-electron-monorepo/
├── apps/
│   ├── electron-app/          # Main Electron app
│   │   ├── src/
│   │   ├── package.json
│   │   └── electron-builder.json5
│   └── web-app/               # Web version (ถ้ามี)
│       ├── src/
│       └── package.json
├── packages/
│   ├── ui/                    # Shared UI components
│   │   ├── src/
│   │   └── package.json
│   ├── shared/                # Shared types & utils
│   │   ├── src/
│   │   └── package.json
│   ├── config/                # Shared configuration
│   │   ├── tsconfig/
│   │   ├── eslint/
│   │   └── package.json
│   └── database/              # Database abstraction
│       ├── src/
│       └── package.json
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

---

## 2. pnpm Workspace Configuration

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```json
// package.json (root)
{
  "name": "my-electron-monorepo",
  "private": true,
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "format": "prettier --write \"**/*.{ts,tsx,json,md}\"",
    "type-check": "turbo run type-check"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "prettier": "^3.0.0",
    "@turbo/gen": "^2.0.0"
  },
  "engines": {
    "node": ">=20",
    "pnpm": ">=9"
  }
}
```

---

## 3. Turborepo Configuration

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "globalEnv": ["NODE_ENV", "ELECTRON_IS_DEV"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "tsconfig*.json", "package.json"],
      "outputs": ["dist/**", "out/**", ".next/**"],
      "cache": true
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["build"],
      "inputs": ["src/**", "tests/**"],
      "outputs": ["coverage/**"],
      "cache": true
    },
    "lint": {
      "inputs": ["src/**", ".eslintrc*"],
      "outputs": [],
      "cache": true
    },
    "type-check": {
      "dependsOn": ["^build"],
      "inputs": ["src/**", "tsconfig*.json"],
      "outputs": [],
      "cache": true
    }
  }
}
```

---

## 4. Shared UI Package

```json
// packages/ui/package.json
{
  "name": "@myapp/ui",
  "version": "1.0.0",
  "private": true,
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./styles": "./src/styles/index.css"
  },
  "scripts": {
    "build": "tsc --declaration --emitDeclarationOnly && vite build",
    "dev": "tsc --watch",
    "lint": "eslint src/",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "react": "^18.0.0"
  },
  "devDependencies": {
    "@myapp/config": "workspace:*",
    "typescript": "^5.0.0",
    "vite": "^5.0.0"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  }
}
```

```tsx
// packages/ui/src/index.ts
export { Button } from './components/Button'
export { Input } from './components/Input'
export { Dialog } from './components/Dialog'
export { Tooltip } from './components/Tooltip'
export { ContextMenu } from './components/ContextMenu'
export type { ButtonProps, InputProps, DialogProps } from './types'
```

```tsx
// packages/ui/src/components/Button.tsx
import React from 'react'
import styles from './Button.module.css'

export interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost'
  size?: 'sm' | 'md' | 'lg'
  loading?: boolean
  icon?: React.ReactNode
}

export function Button({
  variant = 'secondary',
  size = 'md',
  loading = false,
  icon,
  children,
  disabled,
  className = '',
  ...props
}: ButtonProps) {
  return (
    <button
      {...props}
      disabled={disabled || loading}
      className={`${styles.btn} ${styles[variant]} ${styles[size]} ${className}`}
      data-loading={loading}
    >
      {loading ? (
        <span className={styles.spinner} aria-hidden="true" />
      ) : icon ? (
        <span className={styles.icon}>{icon}</span>
      ) : null}
      {children}
    </button>
  )
}
```

---

## 5. Shared Types Package

```json
// packages/shared/package.json
{
  "name": "@myapp/shared",
  "version": "1.0.0",
  "private": true,
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "import": "./src/index.ts"
    }
  },
  "scripts": {
    "type-check": "tsc --noEmit"
  },
  "devDependencies": {
    "@myapp/config": "workspace:*",
    "typescript": "^5.0.0"
  }
}
```

```typescript
// packages/shared/src/index.ts
// Types ที่ share ระหว่าง electron-app และ web-app

export interface User {
  id: string
  name: string
  email: string
  createdAt: number
}

export interface FileRecord {
  id: string
  name: string
  path: string
  size: number
  mimeType: string
  createdAt: number
  updatedAt: number
}

export interface AppSettings {
  theme: 'light' | 'dark' | 'system'
  language: string
  autoSave: boolean
  fontSize: number
}

// Shared utility functions
export function formatFileSize(bytes: number): string {
  const units = ['B', 'KB', 'MB', 'GB']
  let size = bytes
  let unitIdx = 0
  while (size >= 1024 && unitIdx < units.length - 1) {
    size /= 1024
    unitIdx++
  }
  return `${size.toFixed(1)} ${units[unitIdx]}`
}

export function debounce<T extends (...args: unknown[]) => void>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timer: ReturnType<typeof setTimeout>
  return (...args: Parameters<T>) => {
    clearTimeout(timer)
    timer = setTimeout(() => fn(...args), delay)
  }
}
```

---

## 6. Shared Config Package

```json
// packages/config/package.json
{
  "name": "@myapp/config",
  "version": "1.0.0",
  "private": true,
  "exports": {
    "./tsconfig": "./tsconfig/base.json",
    "./tsconfig/electron": "./tsconfig/electron.json",
    "./eslint": "./eslint/index.js"
  }
}
```

```json
// packages/config/tsconfig/base.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "display": "Default",
  "compilerOptions": {
    "composite": false,
    "declaration": true,
    "declarationMap": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "inlineSources": false,
    "isolatedModules": true,
    "moduleResolution": "bundler",
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "preserveWatchOutput": true,
    "skipLibCheck": true,
    "strict": true,
    "strictNullChecks": true,
    "target": "ES2022"
  },
  "exclude": ["node_modules"]
}
```

```json
// packages/config/tsconfig/electron.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "display": "Electron",
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["ES2022", "DOM"],
    "module": "ESNext",
    "target": "ES2022",
    "noImplicitAny": true
  }
}
```

---

## 7. Electron App Package.json

```json
// apps/electron-app/package.json
{
  "name": "@myapp/electron-app",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "electron-vite dev",
    "build": "electron-vite build",
    "package": "npm run build && electron-builder",
    "type-check": "tsc --noEmit",
    "lint": "eslint src/"
  },
  "dependencies": {
    "@myapp/shared": "workspace:*",
    "@myapp/ui": "workspace:*",
    "electron-store": "^8.0.0"
  },
  "devDependencies": {
    "@myapp/config": "workspace:*",
    "electron": "^31.0.0",
    "electron-builder": "^24.0.0",
    "electron-vite": "^2.0.0"
  }
}
```

---

## 8. Internal Package Import

```typescript
// apps/electron-app/src/renderer/App.tsx
// ใช้ shared packages ได้เลย

import { Button, Dialog } from '@myapp/ui'
import { formatFileSize, type FileRecord } from '@myapp/shared'
import '@myapp/ui/styles'

export function App() {
  const files: FileRecord[] = []

  return (
    <div>
      {files.map(file => (
        <div key={file.id}>
          <span>{file.name}</span>
          <span>{formatFileSize(file.size)}</span>
        </div>
      ))}
      <Button variant="primary">เปิดไฟล์</Button>
    </div>
  )
}
```

---

## สรุป

| ส่วนประกอบ | เครื่องมือ | หน้าที่ |
|-----------|---------|---------|
| Workspace | pnpm workspaces | จัดการ packages |
| Build pipeline | Turborepo | parallel builds + cache |
| Shared UI | @myapp/ui | reusable components |
| Shared types | @myapp/shared | type definitions |
| Config | @myapp/config | tsconfig/eslint |
| Cache | turbo cache | faster CI builds |
