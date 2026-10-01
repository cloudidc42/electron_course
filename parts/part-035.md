# Part 35: Accessibility ใน Electron

## บทนำ

Accessibility (a11y) ทำให้แอปพลิเคชันใช้ได้กับทุกคน รวมถึงผู้ที่ใช้ screen reader, keyboard navigation, หรือต้องการ visual adjustments ใน Electron เราต้องดูแลทั้ง HTML/CSS และ Electron-specific APIs

## ARIA Attributes

```jsx
// src/components/accessible/Button.jsx

// ❌ ไม่ถูกต้อง - ไม่มี semantic หรือ ARIA
function BadButton({ onClick, children }) {
  return (
    <div onClick={onClick} style={{ cursor: 'pointer' }}>
      {children}
    </div>
  )
}

// ✅ ถูกต้อง - ใช้ <button> และ ARIA ที่เหมาะสม
function GoodButton({ 
  onClick, 
  children, 
  disabled = false,
  loading = false,
  ariaLabel,
  ariaExpanded,
  ariaControls,
  variant = 'primary'
}) {
  return (
    <button
      onClick={onClick}
      disabled={disabled || loading}
      aria-label={ariaLabel}
      aria-expanded={ariaExpanded}
      aria-controls={ariaControls}
      aria-busy={loading}
      className={`btn btn-${variant} ${loading ? 'btn-loading' : ''}`}
    >
      {loading && (
        <span className="spinner" aria-hidden="true" />
      )}
      <span className={loading ? 'sr-only' : ''}>
        {children}
      </span>
      {loading && <span className="sr-only">Loading...</span>}
    </button>
  )
}

// Dropdown ที่ accessible
function AccessibleDropdown({ label, options, value, onChange }) {
  const [isOpen, setIsOpen] = useState(false)
  const buttonRef = useRef(null)
  const listRef = useRef(null)
  const dropdownId = useId()

  const selectedOption = options.find(o => o.value === value)

  const handleKeyDown = (e) => {
    switch (e.key) {
      case 'Enter':
      case ' ':
        setIsOpen(!isOpen)
        break
      case 'Escape':
        setIsOpen(false)
        buttonRef.current?.focus()
        break
      case 'ArrowDown':
        e.preventDefault()
        if (!isOpen) setIsOpen(true)
        else focusNextOption()
        break
      case 'ArrowUp':
        e.preventDefault()
        if (isOpen) focusPrevOption()
        break
    }
  }

  const focusNextOption = () => {
    const items = listRef.current?.querySelectorAll('[role="option"]')
    if (!items) return
    const focused = document.activeElement
    const index = Array.from(items).indexOf(focused)
    const next = items[Math.min(index + 1, items.length - 1)]
    next?.focus()
  }

  const focusPrevOption = () => {
    const items = listRef.current?.querySelectorAll('[role="option"]')
    if (!items) return
    const focused = document.activeElement
    const index = Array.from(items).indexOf(focused)
    const prev = items[Math.max(index - 1, 0)]
    prev?.focus()
  }

  return (
    <div className="dropdown-container">
      <label id={`${dropdownId}-label`}>{label}</label>
      
      <button
        ref={buttonRef}
        role="combobox"
        aria-haspopup="listbox"
        aria-expanded={isOpen}
        aria-labelledby={`${dropdownId}-label`}
        aria-controls={`${dropdownId}-list`}
        onKeyDown={handleKeyDown}
        onClick={() => setIsOpen(!isOpen)}
        className="dropdown-button"
      >
        {selectedOption?.label || 'Select...'}
        <span aria-hidden="true">▾</span>
      </button>

      {isOpen && (
        <ul
          ref={listRef}
          id={`${dropdownId}-list`}
          role="listbox"
          aria-labelledby={`${dropdownId}-label`}
          className="dropdown-list"
        >
          {options.map(option => (
            <li
              key={option.value}
              role="option"
              aria-selected={option.value === value}
              tabIndex={0}
              onClick={() => {
                onChange(option.value)
                setIsOpen(false)
                buttonRef.current?.focus()
              }}
              onKeyDown={(e) => {
                if (e.key === 'Enter' || e.key === ' ') {
                  onChange(option.value)
                  setIsOpen(false)
                  buttonRef.current?.focus()
                }
              }}
              className={`dropdown-item ${option.value === value ? 'selected' : ''}`}
            >
              {option.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

## Keyboard Navigation

```jsx
// src/hooks/useKeyboardNavigation.js
import { useEffect, useCallback, useRef } from 'react'

/**
 * Hook สำหรับ arrow key navigation ใน list
 */
export function useArrowKeyNavigation(containerRef, options = {}) {
  const {
    selector = '[data-navigable]',
    orientation = 'vertical',  // 'vertical', 'horizontal', 'both'
    loop = true,
    onSelect
  } = options

  const getItems = useCallback(() => {
    if (!containerRef.current) return []
    return Array.from(containerRef.current.querySelectorAll(selector))
      .filter(el => !el.disabled && !el.getAttribute('aria-disabled'))
  }, [containerRef, selector])

  const handleKeyDown = useCallback((e) => {
    const items = getItems()
    if (items.length === 0) return

    const currentIndex = items.indexOf(document.activeElement)
    let nextIndex = currentIndex

    const isVertical = orientation === 'vertical' || orientation === 'both'
    const isHorizontal = orientation === 'horizontal' || orientation === 'both'

    switch (e.key) {
      case 'ArrowDown':
        if (!isVertical) return
        e.preventDefault()
        nextIndex = currentIndex < items.length - 1 
          ? currentIndex + 1 
          : loop ? 0 : currentIndex
        break
      case 'ArrowUp':
        if (!isVertical) return
        e.preventDefault()
        nextIndex = currentIndex > 0 
          ? currentIndex - 1 
          : loop ? items.length - 1 : currentIndex
        break
      case 'ArrowRight':
        if (!isHorizontal) return
        e.preventDefault()
        nextIndex = currentIndex < items.length - 1 
          ? currentIndex + 1 
          : loop ? 0 : currentIndex
        break
      case 'ArrowLeft':
        if (!isHorizontal) return
        e.preventDefault()
        nextIndex = currentIndex > 0 
          ? currentIndex - 1 
          : loop ? items.length - 1 : currentIndex
        break
      case 'Home':
        e.preventDefault()
        nextIndex = 0
        break
      case 'End':
        e.preventDefault()
        nextIndex = items.length - 1
        break
      case 'Enter':
      case ' ':
        if (onSelect && currentIndex >= 0) {
          e.preventDefault()
          onSelect(items[currentIndex], currentIndex)
        }
        return
      default:
        return
    }

    items[nextIndex]?.focus()
  }, [getItems, orientation, loop, onSelect])

  useEffect(() => {
    const container = containerRef.current
    if (!container) return

    container.addEventListener('keydown', handleKeyDown)
    return () => container.removeEventListener('keydown', handleKeyDown)
  }, [containerRef, handleKeyDown])
}

/**
 * Accessible list component
 */
export function AccessibleList({ items, onSelect, label }) {
  const containerRef = useRef(null)
  const [selectedIndex, setSelectedIndex] = useState(-1)

  useArrowKeyNavigation(containerRef, {
    selector: '[role="option"]',
    onSelect: (el, index) => {
      setSelectedIndex(index)
      onSelect?.(items[index], index)
    }
  })

  return (
    <ul
      ref={containerRef}
      role="listbox"
      aria-label={label}
      aria-activedescendant={selectedIndex >= 0 ? `item-${selectedIndex}` : undefined}
      className="accessible-list"
    >
      {items.map((item, index) => (
        <li
          key={item.id || index}
          id={`item-${index}`}
          role="option"
          aria-selected={index === selectedIndex}
          tabIndex={index === 0 ? 0 : -1}
          onClick={() => {
            setSelectedIndex(index)
            onSelect?.(item, index)
          }}
          className={`list-item ${index === selectedIndex ? 'selected' : ''}`}
          data-navigable
        >
          {item.label}
        </li>
      ))}
    </ul>
  )
}
```

## Focus Management

```jsx
// src/hooks/useFocusTrap.js
import { useEffect, useRef } from 'react'

const FOCUSABLE_SELECTORS = [
  'a[href]',
  'button:not([disabled])',
  'input:not([disabled])',
  'select:not([disabled])',
  'textarea:not([disabled])',
  '[tabindex]:not([tabindex="-1"])',
  '[contenteditable]'
].join(', ')

/**
 * Focus trap สำหรับ modal dialogs
 */
export function useFocusTrap(containerRef, isActive = true) {
  const previouslyFocusedRef = useRef(null)

  useEffect(() => {
    if (!isActive) return

    // เก็บ element ที่ focus อยู่ก่อน trap
    previouslyFocusedRef.current = document.activeElement

    // Focus element แรกใน container
    const container = containerRef.current
    if (!container) return

    const focusableElements = container.querySelectorAll(FOCUSABLE_SELECTORS)
    const firstFocusable = focusableElements[0]
    const lastFocusable = focusableElements[focusableElements.length - 1]

    if (firstFocusable) {
      firstFocusable.focus()
    }

    const handleKeyDown = (e) => {
      if (e.key !== 'Tab') return

      if (e.shiftKey) {
        // Shift+Tab: ย้อนกลับ
        if (document.activeElement === firstFocusable) {
          e.preventDefault()
          lastFocusable?.focus()
        }
      } else {
        // Tab: ไปข้างหน้า
        if (document.activeElement === lastFocusable) {
          e.preventDefault()
          firstFocusable?.focus()
        }
      }
    }

    document.addEventListener('keydown', handleKeyDown)

    return () => {
      document.removeEventListener('keydown', handleKeyDown)
      // คืน focus ไปที่ element เดิม
      if (previouslyFocusedRef.current) {
        previouslyFocusedRef.current.focus()
      }
    }
  }, [isActive, containerRef])
}

/**
 * Accessible Modal
 */
export function AccessibleModal({ isOpen, onClose, title, children }) {
  const modalRef = useRef(null)
  const titleId = useId()

  useFocusTrap(modalRef, isOpen)

  useEffect(() => {
    const handleEscape = (e) => {
      if (e.key === 'Escape' && isOpen) {
        onClose()
      }
    }
    document.addEventListener('keydown', handleEscape)
    return () => document.removeEventListener('keydown', handleEscape)
  }, [isOpen, onClose])

  if (!isOpen) return null

  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-labelledby={titleId}
      className="modal-overlay"
      onClick={(e) => e.target === e.currentTarget && onClose()}
    >
      <div ref={modalRef} className="modal-content">
        <div className="modal-header">
          <h2 id={titleId}>{title}</h2>
          <button
            onClick={onClose}
            aria-label="Close dialog"
            className="modal-close"
          >
            ×
          </button>
        </div>
        <div className="modal-body">
          {children}
        </div>
      </div>
    </div>
  )
}
```

## High Contrast Mode

```css
/* src/styles/accessibility.css */

/* Screen reader only - ซ่อน visually แต่ screen reader ยังอ่านได้ */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}

/* Focus indicators ที่ชัดเจน */
:focus-visible {
  outline: 3px solid #2563eb;
  outline-offset: 2px;
  border-radius: 2px;
}

/* Skip navigation link */
.skip-nav {
  position: absolute;
  top: -100%;
  left: 0;
  padding: 8px 16px;
  background: #1e40af;
  color: white;
  text-decoration: none;
  z-index: 9999;
  border-radius: 0 0 4px 0;
}

.skip-nav:focus {
  top: 0;
}

/* High contrast mode */
@media (forced-colors: active) {
  /* Windows High Contrast Mode */
  :root {
    --color-text: ButtonText;
    --color-bg: ButtonFace;
    --color-border: ButtonBorder;
    --color-link: LinkText;
    --color-focus: Highlight;
  }
  
  button, input, select, textarea {
    forced-color-adjust: none;
    border: 2px solid ButtonBorder;
  }
  
  .btn-primary {
    background: ButtonText;
    color: ButtonFace;
  }
  
  /* ไม่ใช้ color เพียงอย่างเดียวในการสื่อความหมาย */
  .status-error::before {
    content: '⚠ ';
  }
  
  .status-success::before {
    content: '✓ ';
  }
}

/* Reduced motion */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

## Font Size Scaling

```jsx
// src/hooks/useFontScale.js
import { useState, useEffect, useCallback } from 'react'

const FONT_SIZES = {
  small: 12,
  normal: 16,
  large: 20,
  xlarge: 24
}

export function useFontScale() {
  const [scale, setScale] = useState(() => {
    const saved = localStorage.getItem('font-scale')
    return saved ? parseFloat(saved) : 1.0
  })

  useEffect(() => {
    document.documentElement.style.fontSize = `${16 * scale}px`
    localStorage.setItem('font-scale', scale.toString())
  }, [scale])

  const increase = useCallback(() => {
    setScale(s => Math.min(s + 0.1, 2.0))
  }, [])

  const decrease = useCallback(() => {
    setScale(s => Math.max(s - 0.1, 0.7))
  }, [])

  const reset = useCallback(() => {
    setScale(1.0)
  }, [])

  return { scale, increase, decrease, reset }
}

// Font Size Controls
export function FontSizeControls() {
  const { scale, increase, decrease, reset } = useFontScale()
  const percent = Math.round(scale * 100)

  return (
    <div
      role="group"
      aria-label="Font size controls"
      className="font-size-controls"
    >
      <button
        onClick={decrease}
        aria-label={`Decrease font size (current: ${percent}%)`}
        disabled={scale <= 0.7}
        className="font-btn"
      >
        A−
      </button>
      
      <output
        aria-live="polite"
        aria-label="Current font size"
        className="font-display"
      >
        {percent}%
      </output>
      
      <button
        onClick={increase}
        aria-label={`Increase font size (current: ${percent}%)`}
        disabled={scale >= 2.0}
        className="font-btn"
      >
        A+
      </button>
      
      <button
        onClick={reset}
        aria-label="Reset font size to 100%"
        className="font-btn"
      >
        Reset
      </button>
    </div>
  )
}
```

## Electron Accessibility APIs

```javascript
// electron/main.js
const { app, systemPreferences } = require('electron')

app.whenReady().then(() => {
  // ตรวจสอบ accessibility features ของระบบ
  checkSystemAccessibility()
  
  // Enable accessibility support
  app.setAccessibilitySupportEnabled(true)
})

function checkSystemAccessibility() {
  // macOS
  if (process.platform === 'darwin') {
    const isAccessibilityEnabled = app.isAccessibilitySupportEnabled()
    console.log('Accessibility support:', isAccessibilityEnabled)
    
    // ตรวจสอบ reduce motion
    const reduceMotion = systemPreferences.getAnimationSettings()
    console.log('Reduce motion:', reduceMotion.prefersReducedMotion)
    
    // ตรวจสอบ high contrast
    const isHighContrast = systemPreferences.getColor('window') !== null
    
    // ส่งข้อมูลไปยัง renderer
    if (mainWindow) {
      mainWindow.webContents.send('a11y:settings', {
        reduceMotion: reduceMotion.prefersReducedMotion,
        highContrast: isHighContrast
      })
    }
  }
}

// Listen for accessibility changes
if (process.platform === 'darwin') {
  systemPreferences.subscribeNotification(
    'NSWorkspaceAccessibilityDisplayOptionsDidChangeNotification',
    () => {
      const settings = systemPreferences.getAnimationSettings()
      if (mainWindow && !mainWindow.isDestroyed()) {
        mainWindow.webContents.send('a11y:settings-changed', {
          reduceMotion: settings.prefersReducedMotion
        })
      }
    }
  )
}
```

```jsx
// src/hooks/useSystemA11y.js
import { useState, useEffect } from 'react'

export function useSystemAccessibility() {
  const [settings, setSettings] = useState({
    reduceMotion: false,
    highContrast: false
  })

  useEffect(() => {
    // รับ initial settings
    if (window.electronAPI) {
      window.electronAPI.invoke('a11y:get-settings').then(s => {
        if (s) setSettings(s)
      }).catch(() => {})
    }

    // CSS media queries
    const motionQuery = window.matchMedia('(prefers-reduced-motion: reduce)')
    const contrastQuery = window.matchMedia('(forced-colors: active)')

    const handleMotion = (e) => {
      setSettings(prev => ({ ...prev, reduceMotion: e.matches }))
    }
    const handleContrast = (e) => {
      setSettings(prev => ({ ...prev, highContrast: e.matches }))
    }

    setSettings({
      reduceMotion: motionQuery.matches,
      highContrast: contrastQuery.matches
    })

    motionQuery.addEventListener('change', handleMotion)
    contrastQuery.addEventListener('change', handleContrast)

    // Listen for Electron updates
    let cleanup = () => {}
    if (window.electronAPI?.bus) {
      cleanup = window.electronAPI.bus.on('a11y:settings-changed', (s) => {
        setSettings(prev => ({ ...prev, ...s }))
      })
    }

    return () => {
      motionQuery.removeEventListener('change', handleMotion)
      contrastQuery.removeEventListener('change', handleContrast)
      cleanup()
    }
  }, [])

  return settings
}
```

## Live Regions

```jsx
// src/components/LiveRegion.jsx
import { useEffect, useRef } from 'react'

/**
 * ARIA Live Region สำหรับแจ้ง screen readers เกี่ยวกับ dynamic content
 */
export function LiveRegion({ message, politeness = 'polite', atomic = true }) {
  const regionRef = useRef(null)

  useEffect(() => {
    // Clear และ set ใหม่เพื่อ trigger announcement ทุกครั้ง
    if (regionRef.current && message) {
      regionRef.current.textContent = ''
      setTimeout(() => {
        if (regionRef.current) {
          regionRef.current.textContent = message
        }
      }, 100)
    }
  }, [message])

  return (
    <div
      ref={regionRef}
      role={politeness === 'assertive' ? 'alert' : 'status'}
      aria-live={politeness}
      aria-atomic={atomic}
      className="sr-only"
    />
  )
}

// Status Bar ที่ announce changes
export function StatusBar({ status }) {
  return (
    <div className="status-bar" role="status" aria-live="polite">
      <span aria-hidden="true" className={`status-icon ${status.type}`} />
      <span>{status.message}</span>
    </div>
  )
}

// Progress ที่ announce percentage
export function AccessibleProgress({ value, max = 100, label }) {
  const percent = Math.round((value / max) * 100)
  
  return (
    <div className="progress-container">
      <label id="progress-label">{label}</label>
      <progress
        value={value}
        max={max}
        aria-labelledby="progress-label"
        aria-valuenow={value}
        aria-valuemin={0}
        aria-valuemax={max}
        aria-valuetext={`${percent}% complete`}
        className="progress-bar"
      />
      <span aria-hidden="true">{percent}%</span>
      <LiveRegion
        message={percent % 25 === 0 ? `${percent}% complete` : ''}
        politeness="polite"
      />
    </div>
  )
}
```

## Accessibility Checker

```javascript
// electron/utils/a11yAudit.js
const { BrowserWindow } = require('electron')

class AccessibilityAuditor {
  /**
   * Run axe-core audit บน window
   */
  async audit(win, options = {}) {
    if (!win || win.isDestroyed()) {
      throw new Error('Window not available')
    }

    const results = await win.webContents.executeJavaScript(`
      (async () => {
        // Check if axe is loaded
        if (typeof axe === 'undefined') {
          return { error: 'axe-core not loaded' }
        }
        
        try {
          const results = await axe.run(document, ${JSON.stringify(options)})
          return {
            violations: results.violations.map(v => ({
              id: v.id,
              impact: v.impact,
              description: v.description,
              help: v.help,
              helpUrl: v.helpUrl,
              nodes: v.nodes.map(n => ({
                html: n.html,
                target: n.target,
                failureSummary: n.failureSummary
              }))
            })),
            passes: results.passes.length,
            incomplete: results.incomplete.length
          }
        } catch (err) {
          return { error: err.message }
        }
      })()
    `)

    return results
  }

  /**
   * ตรวจสอบ basic a11y issues โดยไม่ใช้ axe
   */
  async basicCheck(win) {
    if (!win || win.isDestroyed()) return null

    return await win.webContents.executeJavaScript(`
      (() => {
        const issues = []

        // Images without alt text
        document.querySelectorAll('img:not([alt])').forEach(img => {
          issues.push({
            type: 'missing-alt',
            severity: 'error',
            element: img.outerHTML.slice(0, 100),
            message: 'Image missing alt attribute'
          })
        })

        // Form inputs without labels
        document.querySelectorAll('input, select, textarea').forEach(input => {
          const id = input.id
          const hasLabel = id && document.querySelector('label[for="' + id + '"]')
          const hasAriaLabel = input.getAttribute('aria-label') || input.getAttribute('aria-labelledby')
          const hasTitle = input.getAttribute('title')
          
          if (!hasLabel && !hasAriaLabel && !hasTitle) {
            issues.push({
              type: 'missing-label',
              severity: 'error',
              element: input.outerHTML.slice(0, 100),
              message: 'Form control missing label'
            })
          }
        })

        // Buttons without accessible name
        document.querySelectorAll('button').forEach(btn => {
          const hasText = btn.textContent.trim()
          const hasAriaLabel = btn.getAttribute('aria-label')
          const hasAriaLabelledBy = btn.getAttribute('aria-labelledby')
          const hasTitle = btn.getAttribute('title')
          
          if (!hasText && !hasAriaLabel && !hasAriaLabelledBy && !hasTitle) {
            issues.push({
              type: 'empty-button',
              severity: 'error',
              element: btn.outerHTML.slice(0, 100),
              message: 'Button has no accessible name'
            })
          }
        })

        // Low contrast - basic check (requires actual contrast calculation for full check)
        document.querySelectorAll('*').forEach(el => {
          const style = window.getComputedStyle(el)
          const fontSize = parseFloat(style.fontSize)
          // Flag very small text as potential issue
          if (fontSize < 11 && el.textContent.trim()) {
            issues.push({
              type: 'small-text',
              severity: 'warning',
              message: 'Text may be too small: ' + fontSize + 'px'
            })
          }
        })

        // Heading hierarchy
        const headings = document.querySelectorAll('h1, h2, h3, h4, h5, h6')
        let prevLevel = 0
        headings.forEach(h => {
          const level = parseInt(h.tagName[1])
          if (level > prevLevel + 1) {
            issues.push({
              type: 'heading-skip',
              severity: 'warning',
              element: h.outerHTML.slice(0, 100),
              message: 'Heading level skipped from h' + prevLevel + ' to h' + level
            })
          }
          prevLevel = level
        })

        return {
          issues,
          summary: {
            errors: issues.filter(i => i.severity === 'error').length,
            warnings: issues.filter(i => i.severity === 'warning').length,
            total: issues.length
          }
        }
      })()
    `)
  }
}

module.exports = new AccessibilityAuditor()
```

## Skip Navigation

```jsx
// src/components/SkipNavigation.jsx

/**
 * Skip navigation links - ช่วยผู้ใช้ keyboard/screen reader ข้ามไปยัง content หลัก
 */
export default function SkipNavigation() {
  return (
    <div className="skip-nav-container" aria-label="Skip navigation">
      <a href="#main-content" className="skip-nav">
        Skip to main content
      </a>
      <a href="#main-nav" className="skip-nav">
        Skip to navigation
      </a>
    </div>
  )
}

// App Layout
function AppLayout({ children }) {
  return (
    <>
      <SkipNavigation />
      
      <header role="banner">
        {/* header content */}
      </header>
      
      <nav id="main-nav" role="navigation" aria-label="Main navigation">
        {/* navigation */}
      </nav>
      
      <main id="main-content" role="main" tabIndex={-1}>
        {children}
      </main>
      
      <footer role="contentinfo">
        {/* footer */}
      </footer>
    </>
  )
}
```

## CSS สำหรับ Accessibility

```css
/* src/styles/a11y.css */

/* Visible focus styles */
*:focus-visible {
  outline: 3px solid var(--color-focus, #2563eb);
  outline-offset: 2px;
  border-radius: 3px;
}

/* Remove default outline แต่มี focus-visible แทน */
*:focus:not(:focus-visible) {
  outline: none;
}

/* Skip navigation */
.skip-nav {
  position: absolute;
  top: -100%;
  left: 8px;
  padding: 8px 16px;
  background: #1e40af;
  color: #fff;
  text-decoration: none;
  font-weight: 600;
  border-radius: 0 0 4px 4px;
  z-index: 9999;
  transition: top 0.1s;
}

.skip-nav:focus {
  top: 0;
}

/* Screen reader only */
.sr-only {
  position: absolute !important;
  width: 1px !important;
  height: 1px !important;
  padding: 0 !important;
  margin: -1px !important;
  overflow: hidden !important;
  clip: rect(0, 0, 0, 0) !important;
  white-space: nowrap !important;
  border: 0 !important;
}

/* คงไว้เมื่อ focus สำหรับ skip links */
.sr-only-focusable:focus {
  position: static !important;
  width: auto !important;
  height: auto !important;
  padding: inherit !important;
  margin: inherit !important;
  overflow: visible !important;
  clip: auto !important;
  white-space: normal !important;
}

/* Keyboard focus indicator สำหรับ interactive elements */
.keyboard-focus-ring {
  position: relative;
}

.keyboard-focus-ring:focus-visible::after {
  content: '';
  position: absolute;
  inset: -3px;
  border: 3px solid #2563eb;
  border-radius: inherit;
  pointer-events: none;
}

/* High contrast adjustments */
@media (forced-colors: active) {
  .skip-nav {
    border: 2px solid ButtonText;
  }
  
  *:focus-visible {
    outline: 3px solid Highlight;
  }
}

/* Reduced motion */
@media (prefers-reduced-motion: reduce) {
  .skip-nav {
    transition: none;
  }
  
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

## สรุป

### Accessibility Checklist

| หมวด | รายการ |
|------|--------|
| **Keyboard** | ทุก interactive element focus ได้ด้วย Tab |
| **Keyboard** | มี visible focus indicator |
| **Keyboard** | ไม่มี keyboard trap (ยกเว้น modals) |
| **Screen Reader** | Images มี alt text ที่เหมาะสม |
| **Screen Reader** | Form controls มี labels |
| **Screen Reader** | ARIA live regions สำหรับ dynamic content |
| **Visual** | Text contrast ratio ≥ 4.5:1 (normal), ≥ 3:1 (large) |
| **Visual** | ไม่ใช้ color เพียงอย่างเดียวในการสื่อความหมาย |
| **Visual** | รองรับ 200% zoom โดยไม่เสีย layout |
| **Structure** | Heading hierarchy ถูกต้อง (h1 → h2 → h3) |
| **Structure** | Landmark roles (main, nav, banner, contentinfo) |
| **Structure** | Skip navigation links |

### เครื่องมือทดสอบ

- **axe-core** - automated a11y testing
- **VoiceOver (macOS)** - screen reader ทดสอบ
- **NVDA (Windows)** - screen reader ทดสอบ
- **Chrome DevTools Accessibility panel** - ดู accessibility tree
- **Lighthouse** - audit รวม a11y score
