# Part 93: Desktop App UX Patterns

## UX Patterns สำหรับ Desktop App ที่ดี

ในบทนี้เราจะเรียน onboarding flow, command palette, keyboard-first design, drag & drop, undo/redo stack และ user feedback patterns

---

## 1. Onboarding Flow

```tsx
// src/renderer/components/Onboarding.tsx
import { useState } from 'react'

interface OnboardingStep {
  id: string
  title: string
  description: string
  illustration: React.ReactNode
  action?: { label: string; onClick: () => void }
  skip?: boolean
}

interface OnboardingProps {
  onComplete: () => void
}

export function Onboarding({ onComplete }: OnboardingProps) {
  const [currentStep, setCurrentStep] = useState(0)
  const [completed, setCompleted] = useState<Set<string>>(new Set())

  const steps: OnboardingStep[] = [
    {
      id: 'welcome',
      title: 'ยินดีต้อนรับสู่ My App!',
      description: 'แอปที่จะช่วยให้คุณทำงานได้มีประสิทธิภาพมากขึ้น มาเริ่มต้นด้วยการตั้งค่าพื้นฐานกันเลย',
      illustration: <WelcomeIllustration />
    },
    {
      id: 'theme',
      title: 'เลือกธีมที่คุณชอบ',
      description: 'คุณสามารถเปลี่ยนธีมได้ตลอดเวลาในการตั้งค่า',
      illustration: <ThemeSelector />,
      skip: true
    },
    {
      id: 'shortcuts',
      title: 'Keyboard Shortcuts ที่ควรรู้',
      description: 'เรียนรู้ shortcuts เหล่านี้เพื่อทำงานได้เร็วขึ้น',
      illustration: <ShortcutsList />,
      skip: true
    },
    {
      id: 'ready',
      title: 'พร้อมแล้ว!',
      description: 'คุณพร้อมที่จะเริ่มใช้งาน My App แล้ว',
      illustration: <ReadyIllustration />,
      action: { label: 'เริ่มใช้งาน', onClick: onComplete }
    }
  ]

  const step = steps[currentStep]
  const isLast = currentStep === steps.length - 1
  const progress = ((currentStep + 1) / steps.length) * 100

  const next = () => {
    setCompleted(prev => new Set([...prev, step.id]))
    if (isLast) { onComplete(); return }
    setCurrentStep(prev => prev + 1)
  }

  return (
    <div className="onboarding">
      <div className="onboarding-card">
        {/* Progress */}
        <div className="progress-bar">
          <div className="progress-fill" style={{ width: `${progress}%` }} />
        </div>

        {/* Content */}
        <div className="step-content">
          <div className="illustration">{step.illustration}</div>
          <h2>{step.title}</h2>
          <p>{step.description}</p>
        </div>

        {/* Navigation */}
        <div className="step-nav">
          {currentStep > 0 && (
            <button className="btn" onClick={() => setCurrentStep(prev => prev - 1)}>
              ย้อนกลับ
            </button>
          )}
          <div className="dots">
            {steps.map((s, i) => (
              <div key={s.id} className={`dot ${i === currentStep ? 'active' : ''} ${completed.has(s.id) ? 'done' : ''}`} />
            ))}
          </div>
          {step.skip && (
            <button className="btn-ghost" onClick={next}>ข้าม</button>
          )}
          <button className="btn-primary" onClick={step.action?.onClick ?? next}>
            {step.action?.label ?? (isLast ? 'เสร็จ' : 'ถัดไป')}
          </button>
        </div>
      </div>
    </div>
  )
}

function WelcomeIllustration() {
  return <div className="illustration-placeholder" style={{ fontSize: 80 }}>👋</div>
}
function ThemeSelector() {
  return <div className="illustration-placeholder" style={{ fontSize: 80 }}>🎨</div>
}
function ShortcutsList() {
  const shortcuts = [
    { keys: 'Ctrl+N', action: 'สร้างไฟล์ใหม่' },
    { keys: 'Ctrl+O', action: 'เปิดไฟล์' },
    { keys: 'Ctrl+S', action: 'บันทึก' },
    { keys: 'Ctrl+P', action: 'Command Palette' },
    { keys: 'Ctrl+Z', action: 'เลิกทำ' }
  ]
  return (
    <div className="shortcuts-list">
      {shortcuts.map(s => (
        <div key={s.keys} className="shortcut-row">
          <kbd>{s.keys}</kbd>
          <span>{s.action}</span>
        </div>
      ))}
    </div>
  )
}
function ReadyIllustration() {
  return <div className="illustration-placeholder" style={{ fontSize: 80 }}>🚀</div>
}
```

---

## 2. Command Palette

```tsx
// src/renderer/components/CommandPalette.tsx
import { useState, useEffect, useRef, useMemo } from 'react'

export interface Command {
  id: string
  title: string
  description?: string
  shortcut?: string
  icon?: React.ReactNode
  category?: string
  action: () => void
  keywords?: string[]
}

interface CommandPaletteProps {
  commands: Command[]
  onClose: () => void
}

export function CommandPalette({ commands, onClose }: CommandPaletteProps) {
  const [query, setQuery] = useState('')
  const [selected, setSelected] = useState(0)
  const inputRef = useRef<HTMLInputElement>(null)
  const listRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    inputRef.current?.focus()
  }, [])

  const filtered = useMemo(() => {
    if (!query.trim()) return commands.slice(0, 20)

    const q = query.toLowerCase()
    return commands
      .filter(cmd => {
        const text = [cmd.title, cmd.description, ...(cmd.keywords ?? [])].join(' ').toLowerCase()
        return text.includes(q)
      })
      .sort((a, b) => {
        // Prioritize title matches
        const aTitle = a.title.toLowerCase().includes(q) ? 0 : 1
        const bTitle = b.title.toLowerCase().includes(q) ? 0 : 1
        return aTitle - bTitle
      })
      .slice(0, 20)
  }, [query, commands])

  // Reset selection เมื่อ results เปลี่ยน
  useEffect(() => {
    setSelected(0)
  }, [filtered.length])

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault()
        setSelected(s => Math.min(s + 1, filtered.length - 1))
        break
      case 'ArrowUp':
        e.preventDefault()
        setSelected(s => Math.max(s - 1, 0))
        break
      case 'Enter':
        e.preventDefault()
        if (filtered[selected]) {
          filtered[selected].action()
          onClose()
        }
        break
      case 'Escape':
        onClose()
        break
    }
  }

  // Scroll selected item into view
  useEffect(() => {
    const item = listRef.current?.children[selected] as HTMLElement
    item?.scrollIntoView({ block: 'nearest' })
  }, [selected])

  // Group by category
  const grouped = filtered.reduce((acc, cmd) => {
    const cat = cmd.category || 'ทั่วไป'
    if (!acc[cat]) acc[cat] = []
    acc[cat].push(cmd)
    return acc
  }, {} as Record<string, Command[]>)

  return (
    <div className="palette-overlay" onClick={onClose}>
      <div className="palette" onClick={e => e.stopPropagation()}>
        <div className="palette-input-wrap">
          <span className="search-icon">🔍</span>
          <input
            ref={inputRef}
            value={query}
            onChange={e => setQuery(e.target.value)}
            onKeyDown={handleKeyDown}
            placeholder="พิมพ์คำสั่ง..."
            className="palette-input"
          />
          <kbd className="esc-hint">Esc</kbd>
        </div>

        <div className="palette-results" ref={listRef}>
          {filtered.length === 0 ? (
            <div className="palette-empty">ไม่พบคำสั่ง "{query}"</div>
          ) : (
            Object.entries(grouped).map(([category, cmds]) => (
              <div key={category} className="palette-group">
                <div className="group-label">{category}</div>
                {cmds.map(cmd => {
                  const globalIdx = filtered.indexOf(cmd)
                  return (
                    <div
                      key={cmd.id}
                      className={`palette-item ${globalIdx === selected ? 'selected' : ''}`}
                      onClick={() => { cmd.action(); onClose() }}
                      onMouseEnter={() => setSelected(globalIdx)}
                    >
                      {cmd.icon && <span className="item-icon">{cmd.icon}</span>}
                      <div className="item-text">
                        <span className="item-title">{cmd.title}</span>
                        {cmd.description && <span className="item-desc">{cmd.description}</span>}
                      </div>
                      {cmd.shortcut && <kbd className="item-shortcut">{cmd.shortcut}</kbd>}
                    </div>
                  )
                })}
              </div>
            ))
          )}
        </div>
      </div>
    </div>
  )
}
```

---

## 3. Undo/Redo Stack

```typescript
// src/renderer/hooks/useUndoRedo.ts
import { useState, useCallback } from 'react'

interface UndoRedoState<T> {
  past: T[]
  present: T
  future: T[]
}

export function useUndoRedo<T>(initialState: T, maxHistory = 100) {
  const [state, setState] = useState<UndoRedoState<T>>({
    past: [],
    present: initialState,
    future: []
  })

  const set = useCallback((newPresent: T | ((prev: T) => T)) => {
    setState(s => {
      const resolved = typeof newPresent === 'function'
        ? (newPresent as (prev: T) => T)(s.present)
        : newPresent

      const past = [...s.past, s.present].slice(-maxHistory)
      return {
        past,
        present: resolved,
        future: []
      }
    })
  }, [maxHistory])

  const undo = useCallback(() => {
    setState(s => {
      if (s.past.length === 0) return s
      const past = [...s.past]
      const previous = past.pop()!
      return {
        past,
        present: previous,
        future: [s.present, ...s.future]
      }
    })
  }, [])

  const redo = useCallback(() => {
    setState(s => {
      if (s.future.length === 0) return s
      const [next, ...future] = s.future
      return {
        past: [...s.past, s.present],
        present: next,
        future
      }
    })
  }, [])

  const reset = useCallback((newState: T) => {
    setState({ past: [], present: newState, future: [] })
  }, [])

  return {
    state: state.present,
    set,
    undo,
    redo,
    reset,
    canUndo: state.past.length > 0,
    canRedo: state.future.length > 0,
    historySize: state.past.length
  }
}
```

---

## 4. Drag and Drop

```tsx
// src/renderer/hooks/useDragAndDrop.ts
import { useRef, useState, useCallback, DragEvent } from 'react'

interface DragDropOptions<T> {
  onDrop: (items: T[], position: { x: number; y: number }) => void
  accepts?: string[]  // file types
}

export function useFileDrop<T = File>(options: DragDropOptions<T>) {
  const [isDragging, setIsDragging] = useState(false)
  const counterRef = useRef(0)

  const onDragEnter = useCallback((e: DragEvent) => {
    e.preventDefault()
    counterRef.current++
    setIsDragging(true)
  }, [])

  const onDragLeave = useCallback(() => {
    counterRef.current--
    if (counterRef.current === 0) setIsDragging(false)
  }, [])

  const onDragOver = useCallback((e: DragEvent) => {
    e.preventDefault()
    e.dataTransfer.dropEffect = 'copy'
  }, [])

  const onDrop = useCallback((e: DragEvent) => {
    e.preventDefault()
    counterRef.current = 0
    setIsDragging(false)

    const files = Array.from(e.dataTransfer.files) as T[]
    const filtered = options.accepts
      ? files.filter(f => options.accepts!.some(type =>
          (f as unknown as File).type.includes(type) ||
          (f as unknown as File).name.endsWith(type)
        ))
      : files

    if (filtered.length > 0) {
      options.onDrop(filtered, { x: e.clientX, y: e.clientY })
    }
  }, [options])

  return {
    isDragging,
    dropHandlers: { onDragEnter, onDragLeave, onDragOver, onDrop }
  }
}
```

---

## 5. Toast Notifications

```tsx
// src/renderer/components/Toast.tsx
import { useState, useCallback, createContext, useContext } from 'react'

interface Toast {
  id: string
  message: string
  type: 'success' | 'error' | 'warning' | 'info'
  duration?: number
  action?: { label: string; onClick: () => void }
}

interface ToastContextValue {
  show: (toast: Omit<Toast, 'id'>) => string
  dismiss: (id: string) => void
}

const ToastContext = createContext<ToastContextValue | null>(null)

export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([])

  const show = useCallback((toast: Omit<Toast, 'id'>) => {
    const id = `toast-${Date.now()}-${Math.random()}`
    const duration = toast.duration ?? 4000

    setToasts(prev => [...prev, { ...toast, id }])

    if (duration > 0) {
      setTimeout(() => dismiss(id), duration)
    }

    return id
  }, [])

  const dismiss = useCallback((id: string) => {
    setToasts(prev => prev.filter(t => t.id !== id))
  }, [])

  return (
    <ToastContext.Provider value={{ show, dismiss }}>
      {children}
      <div className="toast-container">
        {toasts.map(toast => (
          <ToastItem key={toast.id} toast={toast} onDismiss={() => dismiss(toast.id)} />
        ))}
      </div>
    </ToastContext.Provider>
  )
}

function ToastItem({ toast, onDismiss }: { toast: Toast; onDismiss: () => void }) {
  const icons = { success: '✅', error: '❌', warning: '⚠️', info: 'ℹ️' }

  return (
    <div className={`toast toast-${toast.type}`} role="alert">
      <span className="toast-icon">{icons[toast.type]}</span>
      <span className="toast-message">{toast.message}</span>
      {toast.action && (
        <button className="toast-action" onClick={() => { toast.action!.onClick(); onDismiss() }}>
          {toast.action.label}
        </button>
      )}
      <button className="toast-close" onClick={onDismiss}>×</button>
    </div>
  )
}

export function useToast(): ToastContextValue {
  const ctx = useContext(ToastContext)
  if (!ctx) throw new Error('useToast must be used within ToastProvider')
  return ctx
}
```

---

## สรุป

| Pattern | Component | UX Benefit |
|---------|---------|-----------|
| Onboarding | Onboarding wizard | ลด time-to-value |
| Command Palette | Ctrl+P popup | keyboard-first power users |
| Undo/Redo | History stack | ความมั่นใจในการแก้ไข |
| Drag and Drop | Drop zone | intuitive file input |
| Toast | Notification stack | non-intrusive feedback |
| Progress indicators | Spinner/bar | reduce perceived wait |
