# Part 83: Electron App Themes

## ระบบ Theme แบบ Dynamic ใน Electron App

ในบทนี้เราจะเรียน CSS custom properties, nativeTheme API, theme switching animation และ user-defined themes

---

## 1. Theme System Architecture

```typescript
// src/shared/themes.ts
export interface ThemeTokens {
  // Colors
  colorBackground: string
  colorSurface: string
  colorSurfaceHover: string
  colorBorder: string
  colorText: string
  colorTextMuted: string
  colorTextDisabled: string
  colorAccent: string
  colorAccentHover: string
  colorAccentText: string
  colorDanger: string
  colorWarning: string
  colorSuccess: string
  colorInfo: string
  // Sidebar
  colorSidebar: string
  colorSidebarText: string
  colorSidebarActive: string
  // Shadows
  shadowSm: string
  shadowMd: string
  shadowLg: string
  // Typography
  fontFamily: string
  fontMono: string
  fontSize: string
  lineHeight: string
  // Spacing
  borderRadius: string
  borderRadiusLg: string
  // Scrollbar
  colorScrollbar: string
  colorScrollbarThumb: string
}

export interface Theme {
  id: string
  name: string
  dark: boolean
  tokens: ThemeTokens
}

export const themes: Theme[] = [
  {
    id: 'dark',
    name: 'Dark (Default)',
    dark: true,
    tokens: {
      colorBackground: '#1e1e1e',
      colorSurface: '#252526',
      colorSurfaceHover: '#2a2d2e',
      colorBorder: '#3c3c3c',
      colorText: '#cccccc',
      colorTextMuted: '#858585',
      colorTextDisabled: '#5a5a5a',
      colorAccent: '#0078d4',
      colorAccentHover: '#1084d8',
      colorAccentText: '#ffffff',
      colorDanger: '#f85149',
      colorWarning: '#cca700',
      colorSuccess: '#3fb950',
      colorInfo: '#388bfd',
      colorSidebar: '#333333',
      colorSidebarText: '#cccccc',
      colorSidebarActive: '#094771',
      shadowSm: '0 1px 2px rgba(0,0,0,0.3)',
      shadowMd: '0 4px 8px rgba(0,0,0,0.3)',
      shadowLg: '0 8px 24px rgba(0,0,0,0.4)',
      fontFamily: '"Segoe UI", system-ui, sans-serif',
      fontMono: '"JetBrains Mono", "Cascadia Code", monospace',
      fontSize: '13px',
      lineHeight: '1.6',
      borderRadius: '4px',
      borderRadiusLg: '8px',
      colorScrollbar: '#1e1e1e',
      colorScrollbarThumb: '#424242'
    }
  },
  {
    id: 'light',
    name: 'Light',
    dark: false,
    tokens: {
      colorBackground: '#ffffff',
      colorSurface: '#f3f3f3',
      colorSurfaceHover: '#e8e8e8',
      colorBorder: '#e0e0e0',
      colorText: '#1f1f1f',
      colorTextMuted: '#6e6e6e',
      colorTextDisabled: '#a8a8a8',
      colorAccent: '#0067b8',
      colorAccentHover: '#005a9e',
      colorAccentText: '#ffffff',
      colorDanger: '#d93025',
      colorWarning: '#b45309',
      colorSuccess: '#1a7f37',
      colorInfo: '#0550ae',
      colorSidebar: '#f8f8f8',
      colorSidebarText: '#333333',
      colorSidebarActive: '#dce9f8',
      shadowSm: '0 1px 2px rgba(0,0,0,0.1)',
      shadowMd: '0 4px 8px rgba(0,0,0,0.1)',
      shadowLg: '0 8px 24px rgba(0,0,0,0.15)',
      fontFamily: '"Segoe UI", system-ui, sans-serif',
      fontMono: '"JetBrains Mono", "Cascadia Code", monospace',
      fontSize: '13px',
      lineHeight: '1.6',
      borderRadius: '4px',
      borderRadiusLg: '8px',
      colorScrollbar: '#f3f3f3',
      colorScrollbarThumb: '#c5c5c5'
    }
  },
  {
    id: 'monokai',
    name: 'Monokai',
    dark: true,
    tokens: {
      colorBackground: '#272822',
      colorSurface: '#2d2e27',
      colorSurfaceHover: '#35362e',
      colorBorder: '#49483e',
      colorText: '#f8f8f2',
      colorTextMuted: '#75715e',
      colorTextDisabled: '#4a4940',
      colorAccent: '#a6e22e',
      colorAccentHover: '#b8f03e',
      colorAccentText: '#272822',
      colorDanger: '#f92672',
      colorWarning: '#fd971f',
      colorSuccess: '#a6e22e',
      colorInfo: '#66d9e8',
      colorSidebar: '#1e1f1a',
      colorSidebarText: '#f8f8f2',
      colorSidebarActive: '#3e3d31',
      shadowSm: '0 1px 2px rgba(0,0,0,0.4)',
      shadowMd: '0 4px 8px rgba(0,0,0,0.4)',
      shadowLg: '0 8px 24px rgba(0,0,0,0.5)',
      fontFamily: '"Segoe UI", system-ui, sans-serif',
      fontMono: '"JetBrains Mono", "Cascadia Code", monospace',
      fontSize: '13px',
      lineHeight: '1.6',
      borderRadius: '4px',
      borderRadiusLg: '8px',
      colorScrollbar: '#272822',
      colorScrollbarThumb: '#49483e'
    }
  }
]
```

---

## 2. Theme Service

```typescript
// src/renderer/services/themeService.ts
import { themes, Theme, ThemeTokens } from '../../shared/themes'
import ElectronStore from 'electron-store'

class ThemeService {
  private currentTheme: Theme
  private styleEl: HTMLStyleElement | null = null
  private listeners: Set<(theme: Theme) => void> = new Set()

  constructor() {
    this.currentTheme = themes[0]
  }

  init(): void {
    // สร้าง style element
    this.styleEl = document.createElement('style')
    this.styleEl.id = 'theme-vars'
    document.head.appendChild(this.styleEl)

    // โหลด theme ที่บันทึกไว้
    const savedId = localStorage.getItem('theme-id') || 'dark'
    const saved = themes.find(t => t.id === savedId) || themes[0]
    this.applyTheme(saved, false)

    // ฟัง nativeTheme changes
    window.electronAPI.on('theme:changed', ({ shouldUseDark }: { shouldUseDark: boolean }) => {
      const autoTheme = themes.find(t => t.dark === shouldUseDark) || themes[0]
      this.applyTheme(autoTheme, true)
    })
  }

  applyTheme(theme: Theme, animate = true): void {
    if (animate) {
      document.documentElement.classList.add('theme-transitioning')
      setTimeout(() => document.documentElement.classList.remove('theme-transitioning'), 300)
    }

    this.currentTheme = theme
    this.updateCSSVariables(theme.tokens)
    document.documentElement.setAttribute('data-theme', theme.id)
    document.documentElement.classList.toggle('dark', theme.dark)

    localStorage.setItem('theme-id', theme.id)
    this.listeners.forEach(cb => cb(theme))

    // แจ้ง main process ด้วย
    window.electronAPI.send('theme:apply', { themeId: theme.id, dark: theme.dark })
  }

  private updateCSSVariables(tokens: ThemeTokens): void {
    if (!this.styleEl) return

    const cssVars = Object.entries(tokens)
      .map(([key, value]) => `  --${camelToKebab(key)}: ${value};`)
      .join('\n')

    this.styleEl.textContent = `:root {\n${cssVars}\n}`
  }

  setThemeById(id: string): void {
    const theme = themes.find(t => t.id === id)
    if (theme) this.applyTheme(theme)
  }

  getCurrentTheme(): Theme {
    return this.currentTheme
  }

  getAvailableThemes(): Theme[] {
    return [...themes]
  }

  onChange(cb: (theme: Theme) => void): () => void {
    this.listeners.add(cb)
    return () => this.listeners.delete(cb)
  }

  // Custom theme ที่ผู้ใช้สร้างเอง
  createCustomTheme(name: string, tokens: Partial<ThemeTokens>): Theme {
    const base = themes.find(t => t.id === 'dark')!
    const custom: Theme = {
      id: `custom-${Date.now()}`,
      name,
      dark: true,
      tokens: { ...base.tokens, ...tokens }
    }
    themes.push(custom)
    return custom
  }
}

function camelToKebab(str: string): string {
  return str.replace(/([A-Z])/g, (m) => `-${m.toLowerCase()}`)
}

export const themeService = new ThemeService()
```

---

## 3. CSS Theme Foundation

```css
/* src/renderer/styles/theme.css */

/* Transition animation สำหรับ theme switching */
.theme-transitioning,
.theme-transitioning * {
  transition:
    background-color 250ms ease,
    color 200ms ease,
    border-color 200ms ease,
    box-shadow 200ms ease !important;
}

/* Base styles ที่ใช้ CSS variables */
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  background-color: var(--color-background);
  color: var(--color-text);
  font-family: var(--font-family);
  font-size: var(--font-size);
  line-height: var(--line-height);
}

/* Scrollbar theming */
::-webkit-scrollbar {
  width: 10px;
  height: 10px;
}

::-webkit-scrollbar-track {
  background: var(--color-scrollbar);
}

::-webkit-scrollbar-thumb {
  background: var(--color-scrollbar-thumb);
  border-radius: 5px;
}

::-webkit-scrollbar-thumb:hover {
  background: var(--color-text-muted);
}

/* Component base styles */
.surface {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--border-radius);
}

.btn {
  background: var(--color-surface);
  color: var(--color-text);
  border: 1px solid var(--color-border);
  border-radius: var(--border-radius);
  padding: 6px 14px;
  cursor: pointer;
  font-size: var(--font-size);
  transition: background 150ms, border-color 150ms;
}

.btn:hover { background: var(--color-surface-hover); }

.btn-primary {
  background: var(--color-accent);
  color: var(--color-accent-text);
  border-color: var(--color-accent);
}

.btn-primary:hover { background: var(--color-accent-hover); }

.btn-danger {
  background: transparent;
  color: var(--color-danger);
  border-color: var(--color-danger);
}
```

---

## 4. Theme Picker Component

```tsx
// src/renderer/components/ThemePicker.tsx
import { useState, useEffect } from 'react'
import { themeService } from '../services/themeService'
import { Theme } from '../../shared/themes'

export function ThemePicker() {
  const [currentTheme, setCurrentTheme] = useState(themeService.getCurrentTheme())
  const themes = themeService.getAvailableThemes()

  useEffect(() => {
    return themeService.onChange(setCurrentTheme)
  }, [])

  return (
    <div className="theme-picker">
      <h3>ธีม</h3>
      <div className="theme-grid">
        {themes.map(theme => (
          <ThemeCard
            key={theme.id}
            theme={theme}
            active={theme.id === currentTheme.id}
            onClick={() => themeService.setThemeById(theme.id)}
          />
        ))}
      </div>
    </div>
  )
}

function ThemeCard({ theme, active, onClick }: {
  theme: Theme
  active: boolean
  onClick: () => void
}) {
  const { tokens } = theme

  return (
    <div
      className={`theme-card ${active ? 'active' : ''}`}
      onClick={onClick}
      style={{
        background: tokens.colorBackground,
        border: `2px solid ${active ? tokens.colorAccent : tokens.colorBorder}`,
        borderRadius: tokens.borderRadius,
        overflow: 'hidden',
        cursor: 'pointer'
      }}
    >
      {/* Preview */}
      <div className="preview" style={{ padding: 8 }}>
        <div style={{
          background: tokens.colorSidebar,
          height: 40,
          borderRadius: 2,
          marginBottom: 4,
          display: 'flex', alignItems: 'center', paddingLeft: 8, gap: 4
        }}>
          {[...Array(3)].map((_, i) => (
            <div key={i} style={{
              width: 6, height: 6, borderRadius: '50%',
              background: [tokens.colorDanger, tokens.colorWarning, tokens.colorSuccess][i]
            }} />
          ))}
        </div>
        <div style={{
          background: tokens.colorSurface,
          height: 24,
          borderRadius: 2,
          marginBottom: 4
        }} />
        <div style={{
          background: tokens.colorAccent,
          height: 16,
          width: '60%',
          borderRadius: 2
        }} />
      </div>
      <div style={{
        padding: '4px 8px 8px',
        background: tokens.colorSurface,
        color: tokens.colorText,
        fontSize: 11
      }}>
        {theme.name}
      </div>
    </div>
  )
}
```

---

## 5. nativeTheme Integration (Main Process)

```typescript
// src/main/nativeThemeHandler.ts
import { nativeTheme, ipcMain, BrowserWindow } from 'electron'

export function setupNativeTheme(win: BrowserWindow): void {
  // ฟัง system theme change
  nativeTheme.on('updated', () => {
    win.webContents.send('theme:changed', {
      shouldUseDark: nativeTheme.shouldUseDarkColors
    })
  })

  // Handle theme mode requests จาก renderer
  ipcMain.handle('theme:set-mode', (_, mode: 'light' | 'dark' | 'system') => {
    nativeTheme.themeSource = mode
    return nativeTheme.shouldUseDarkColors
  })

  ipcMain.handle('theme:get-system', () => ({
    shouldUseDark: nativeTheme.shouldUseDarkColors,
    mode: nativeTheme.themeSource
  }))
}
```

---

## สรุป

| ส่วนประกอบ | รายละเอียด |
|-----------|-----------|
| ThemeTokens | CSS custom properties ทั้งหมด |
| Theme definitions | dark, light, monokai + custom |
| ThemeService | apply, persist, onChange |
| CSS transitions | smooth theme switching |
| nativeTheme API | follow system preference |
| ThemePicker UI | visual theme selector |
