# Part 27: WebContents & BrowserView ใน Electron

## บทนำ

`webContents` เป็น module ที่ทรงพลังใน Electron สำหรับควบคุม web content ส่วน `BrowserView` และ `WebContentsView` ช่วยให้เราฝัง web pages เพิ่มเติมใน window ได้ ในบทนี้จะครอบคลุม APIs, navigation, JavaScript execution, find-in-page, printing, screenshots, และ content injection

## webContents API พื้นฐาน

### การเข้าถึง webContents

```javascript
// electron/main.js
const { BrowserWindow, webContents } = require('electron')

// จาก window
const mainWindow = new BrowserWindow({...})
const wc = mainWindow.webContents

// ดึง webContents ทั้งหมด
const allWebContents = webContents.getAllWebContents()

// ดึง focused webContents
const focused = webContents.getFocusedWebContents()

// ดึงจาก ID
const specificWC = webContents.fromId(1)
```

### Navigation Control

```javascript
// electron/services/navigationService.js
const { BrowserWindow, ipcMain } = require('electron')

class NavigationService {
  constructor(mainWindow) {
    this.window = mainWindow
    this.wc = mainWindow.webContents
    this.setupNavigationHandlers()
  }

  setupNavigationHandlers() {
    const wc = this.wc

    // ติดตาม navigation events
    wc.on('did-start-navigation', (event, url, isInPlace, isMainFrame) => {
      if (isMainFrame) {
        this.onNavigationStart(url)
      }
    })

    wc.on('did-navigate', (event, url, httpResponseCode) => {
      this.onNavigationComplete(url, httpResponseCode)
    })

    wc.on('did-navigate-in-page', (event, url, isMainFrame) => {
      if (isMainFrame) {
        this.onNavigationComplete(url, 200)
      }
    })

    wc.on('did-fail-load', (event, errorCode, errorDescription, validatedURL) => {
      this.onNavigationError(validatedURL, errorCode, errorDescription)
    })

    wc.on('dom-ready', () => {
      this.onDomReady()
    })

    wc.on('page-title-updated', (event, title) => {
      this.window.setTitle(title)
    })

    wc.on('page-favicon-updated', (event, favicons) => {
      // อัปเดต favicon
      if (favicons.length > 0) {
        this.currentFavicon = favicons[0]
      }
    })
  }

  onNavigationStart(url) {
    console.log('[Nav] Starting:', url)
    this.window.webContents.send('nav:loading', { url, loading: true })
  }

  onNavigationComplete(url, statusCode) {
    console.log('[Nav] Completed:', url, statusCode)
    this.window.webContents.send('nav:complete', {
      url,
      statusCode,
      loading: false,
      canGoBack: this.wc.navigationHistory.canGoBack(),
      canGoForward: this.wc.navigationHistory.canGoForward()
    })
  }

  onNavigationError(url, errorCode, description) {
    console.error('[Nav] Error:', url, errorCode, description)
    this.window.webContents.send('nav:error', { url, errorCode, description })
  }

  onDomReady() {
    this.window.webContents.send('nav:domReady')
  }

  // Navigation methods
  navigate(url) {
    this.wc.loadURL(url)
  }

  goBack() {
    if (this.wc.navigationHistory.canGoBack()) {
      this.wc.navigationHistory.goBack()
    }
  }

  goForward() {
    if (this.wc.navigationHistory.canGoForward()) {
      this.wc.navigationHistory.goForward()
    }
  }

  reload(ignoreCache = false) {
    if (ignoreCache) {
      this.wc.reloadIgnoringCache()
    } else {
      this.wc.reload()
    }
  }

  stop() {
    this.wc.stop()
  }

  getCurrentURL() {
    return this.wc.getURL()
  }

  getHistoryEntries() {
    const history = this.wc.navigationHistory
    return {
      entries: history.getAllEntries?.() || [],
      currentIndex: history.getActiveIndex?.() || 0,
      canGoBack: history.canGoBack(),
      canGoForward: history.canGoForward()
    }
  }
}

module.exports = NavigationService
```

## Execute JavaScript

```javascript
// electron/services/scriptInjector.js
const { ipcMain } = require('electron')

class ScriptInjector {
  constructor(webContentsRef) {
    this.wc = webContentsRef
    this.setupHandlers()
  }

  /**
   * Execute JavaScript ใน renderer context
   */
  async executeScript(script, userGesture = false) {
    try {
      const result = await this.wc.executeJavaScript(script, userGesture)
      return { success: true, result }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  /**
   * Inject CSS
   */
  async injectCSS(css) {
    try {
      const key = await this.wc.insertCSS(css)
      return { success: true, key }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  /**
   * Remove injected CSS
   */
  async removeCSS(key) {
    await this.wc.removeInsertedCSS(key)
  }

  /**
   * ดึงข้อมูลจาก DOM
   */
  async getDOMData() {
    return this.executeScript(`
      JSON.stringify({
        title: document.title,
        url: window.location.href,
        bodyText: document.body.innerText.substring(0, 1000),
        links: Array.from(document.querySelectorAll('a')).slice(0, 20).map(a => ({
          text: a.textContent.trim(),
          href: a.href
        })),
        images: Array.from(document.querySelectorAll('img')).slice(0, 10).map(img => img.src),
        meta: Array.from(document.querySelectorAll('meta')).reduce((acc, meta) => {
          const name = meta.getAttribute('name') || meta.getAttribute('property')
          if (name) acc[name] = meta.getAttribute('content')
          return acc
        }, {})
      })
    `).then(({ result }) => JSON.parse(result))
  }

  /**
   * Scroll ไปยัง element
   */
  async scrollToElement(selector) {
    return this.executeScript(`
      const el = document.querySelector(${JSON.stringify(selector)})
      if (el) {
        el.scrollIntoView({ behavior: 'smooth', block: 'center' })
        true
      } else {
        false
      }
    `)
  }

  /**
   * Click element
   */
  async clickElement(selector) {
    return this.executeScript(`
      const el = document.querySelector(${JSON.stringify(selector)})
      if (el) {
        el.click()
        true
      } else {
        false
      }
    `, true) // userGesture: true
  }

  /**
   * Fill form field
   */
  async fillInput(selector, value) {
    return this.executeScript(`
      const el = document.querySelector(${JSON.stringify(selector)})
      if (el) {
        const nativeInputValueSetter = Object.getOwnPropertyDescriptor(
          window.HTMLInputElement.prototype, 'value'
        ).set
        nativeInputValueSetter.call(el, ${JSON.stringify(value)})
        el.dispatchEvent(new Event('input', { bubbles: true }))
        el.dispatchEvent(new Event('change', { bubbles: true }))
        true
      } else {
        false
      }
    `)
  }

  /**
   * Extract เนื้อหา
   */
  async extractContent(options = {}) {
    const { selectors = {}, includeImages = false } = options
    
    return this.executeScript(`
      (function() {
        const result = {
          title: document.title,
          url: window.location.href
        }
        
        const selectors = ${JSON.stringify(selectors)}
        for (const [key, selector] of Object.entries(selectors)) {
          const el = document.querySelector(selector)
          result[key] = el ? el.textContent.trim() : null
        }
        
        if (${includeImages}) {
          result.images = Array.from(document.images).map(img => ({
            src: img.src,
            alt: img.alt,
            width: img.naturalWidth,
            height: img.naturalHeight
          }))
        }
        
        return JSON.stringify(result)
      })()
    `).then(({ result }) => result ? JSON.parse(result) : null)
  }

  setupHandlers() {
    ipcMain.handle('webcontents:executeScript', async (event, script) => {
      return this.executeScript(script)
    })

    ipcMain.handle('webcontents:injectCSS', async (event, css) => {
      return this.injectCSS(css)
    })

    ipcMain.handle('webcontents:extractContent', async (event, options) => {
      return this.extractContent(options)
    })
  }
}

module.exports = ScriptInjector
```

## Find in Page

```javascript
// electron/services/findInPage.js

class FindInPage {
  constructor(webContentsRef) {
    this.wc = webContentsRef
    this.searchText = ''
    this.results = { activeMatchOrdinal: 0, matches: 0 }
    this.setupEventListeners()
  }

  setupEventListeners() {
    this.wc.on('found-in-page', (event, result) => {
      this.results = {
        activeMatchOrdinal: result.activeMatchOrdinal,
        matches: result.matches,
        finalUpdate: result.finalUpdate
      }
      
      // ส่งผลลัพธ์ไปยัง renderer
      if (!this.wc.isDestroyed()) {
        this.wc.send('findInPage:result', this.results)
      }
    })
  }

  /**
   * ค้นหาข้อความ
   */
  find(text, options = {}) {
    if (!text) {
      this.stopFind()
      return
    }

    this.searchText = text
    
    this.wc.findInPage(text, {
      forward: options.forward !== false,
      findNext: options.findNext || false,
      matchCase: options.matchCase || false,
      ...options
    })
  }

  /**
   * ค้นหาถัดไป
   */
  findNext() {
    if (this.searchText) {
      this.find(this.searchText, { findNext: true, forward: true })
    }
  }

  /**
   * ค้นหาก่อนหน้า
   */
  findPrevious() {
    if (this.searchText) {
      this.find(this.searchText, { findNext: true, forward: false })
    }
  }

  /**
   * หยุดค้นหา
   */
  stopFind(clearSelection = true) {
    this.wc.stopFindInPage(clearSelection ? 'clearSelection' : 'keepSelection')
    this.searchText = ''
    this.results = { activeMatchOrdinal: 0, matches: 0 }
  }

  getResults() {
    return this.results
  }
}

module.exports = FindInPage
```

## Print & PDF

```javascript
// electron/services/printService.js
const { ipcMain, dialog } = require('electron')
const path = require('path')
const fs = require('fs')

class PrintService {
  constructor(mainWindow) {
    this.window = mainWindow
    this.setupHandlers()
  }

  /**
   * Print หน้าปัจจุบัน
   */
  async print(options = {}) {
    const printOptions = {
      silent: false,              // แสดง print dialog
      printBackground: true,      // Print background
      deviceName: '',             // ชื่อ printer (ว่าง = default)
      color: true,
      margins: {
        marginType: 'printableArea'  // 'default' | 'none' | 'printableArea' | 'custom'
      },
      landscape: false,
      scaleFactor: 100,
      pagesPerSheet: 1,
      copies: 1,
      pageRanges: [],             // เช่น [{from: 0, to: 1}]
      duplexMode: 'simplex',      // 'simplex' | 'shortEdge' | 'longEdge'
      ...options
    }

    return new Promise((resolve, reject) => {
      this.window.webContents.print(printOptions, (success, failureReason) => {
        if (success) {
          resolve({ success: true })
        } else {
          resolve({ success: false, reason: failureReason })
        }
      })
    })
  }

  /**
   * Export เป็น PDF
   */
  async exportToPDF(filePath, options = {}) {
    const pdfOptions = {
      pageSize: 'A4',             // 'A3' | 'A4' | 'A5' | 'Legal' | 'Letter' | 'Tabloid'
      printBackground: true,
      landscape: false,
      marginsType: 0,             // 0=default | 1=no margins | 2=min margins
      scaleFactor: 100,
      pageRanges: '',
      headerTemplate: '',
      footerTemplate: '',
      displayHeaderFooter: false,
      ...options
    }

    try {
      const data = await this.window.webContents.printToPDF(pdfOptions)
      
      if (filePath) {
        await fs.promises.writeFile(filePath, data)
        return { success: true, path: filePath, size: data.length }
      }
      
      return { success: true, data }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  /**
   * ดึงรายการ printers
   */
  async getPrinters() {
    return this.window.webContents.getPrintersAsync()
  }

  /**
   * Print พร้อม dialog เพื่อเลือก path
   */
  async printToPDFWithDialog() {
    const savePath = await dialog.showSaveDialog(this.window, {
      defaultPath: 'document.pdf',
      filters: [{ name: 'PDF Files', extensions: ['pdf'] }]
    })

    if (savePath.canceled) return null

    return this.exportToPDF(savePath.filePath)
  }

  setupHandlers() {
    ipcMain.handle('print:print', (event, options) => {
      return this.print(options)
    })

    ipcMain.handle('print:toPDF', async (event, options) => {
      return this.printToPDFWithDialog()
    })

    ipcMain.handle('print:getPrinters', () => {
      return this.getPrinters()
    })
  }
}

module.exports = PrintService
```

## Screenshot / Capture

```javascript
// electron/services/captureService.js
const { ipcMain, nativeImage, clipboard } = require('electron')
const path = require('path')
const fs = require('fs')

class CaptureService {
  constructor(mainWindow) {
    this.window = mainWindow
    this.setupHandlers()
  }

  /**
   * Capture หน้าจอทั้งหมด
   */
  async captureFullPage() {
    try {
      const image = await this.window.webContents.capturePage()
      return {
        success: true,
        width: image.getSize().width,
        height: image.getSize().height,
        dataURL: image.toDataURL(),
        buffer: image.toPNG()
      }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  /**
   * Capture เฉพาะ region
   */
  async captureRegion(rect) {
    try {
      const image = await this.window.webContents.capturePage({
        x: rect.x,
        y: rect.y,
        width: rect.width,
        height: rect.height
      })
      
      return {
        success: true,
        dataURL: image.toDataURL(),
        buffer: image.toPNG()
      }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  /**
   * บันทึก screenshot เป็นไฟล์
   */
  async saveScreenshot(filePath, rect = null) {
    const capture = rect 
      ? await this.captureRegion(rect)
      : await this.captureFullPage()
    
    if (!capture.success) return capture

    try {
      await fs.promises.writeFile(filePath, capture.buffer)
      return { success: true, path: filePath }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  /**
   * Copy screenshot ไปยัง clipboard
   */
  async copyToClipboard(rect = null) {
    const capture = rect
      ? await this.captureRegion(rect)
      : await this.captureFullPage()
    
    if (!capture.success) return capture

    try {
      const image = nativeImage.createFromBuffer(capture.buffer)
      clipboard.writeImage(image)
      return { success: true }
    } catch (error) {
      return { success: false, error: error.message }
    }
  }

  setupHandlers() {
    ipcMain.handle('capture:fullPage', () => {
      return this.captureFullPage()
    })

    ipcMain.handle('capture:region', (event, rect) => {
      return this.captureRegion(rect)
    })

    ipcMain.handle('capture:save', async (event, options = {}) => {
      const { dialog } = require('electron')
      
      const savePath = await dialog.showSaveDialog(this.window, {
        defaultPath: `screenshot-${Date.now()}.png`,
        filters: [{ name: 'PNG Images', extensions: ['png'] }]
      })
      
      if (savePath.canceled) return null
      return this.saveScreenshot(savePath.filePath, options.rect)
    })

    ipcMain.handle('capture:clipboard', (event, rect) => {
      return this.copyToClipboard(rect)
    })
  }
}

module.exports = CaptureService
```

## BrowserView (Legacy) / WebContentsView

```javascript
// electron/views/browserViewManager.js
const { BrowserView, BrowserWindow, WebContentsView } = require('electron')

class BrowserViewManager {
  constructor(mainWindow) {
    this.mainWindow = mainWindow
    this.views = new Map()
    this.activeViewId = null
  }

  /**
   * สร้าง WebContentsView ใหม่ (API ใหม่)
   */
  createWebContentsView(id, url, bounds) {
    const view = new WebContentsView({
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
        sandbox: true
      }
    })

    view.webContents.loadURL(url)
    view.setBounds(bounds)
    
    // เพิ่ม view เข้า window
    this.mainWindow.contentView.addChildView(view)
    
    this.views.set(id, view)
    this.activeViewId = id

    // Setup events
    this.setupViewEvents(id, view)
    
    return view
  }

  /**
   * สร้าง BrowserView (Legacy API)
   */
  createBrowserView(id, url, bounds) {
    const view = new BrowserView({
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
        preload: path.join(__dirname, '../preload.js')
      }
    })

    this.mainWindow.addBrowserView(view)
    view.setBounds(bounds)
    view.setAutoResize({
      width: true,
      height: true,
      horizontal: false,
      vertical: false
    })

    view.webContents.loadURL(url)
    
    this.views.set(id, view)
    this.activeViewId = id

    return view
  }

  /**
   * Setup events สำหรับ view
   */
  setupViewEvents(id, view) {
    view.webContents.on('did-navigate', (event, url) => {
      this.mainWindow.webContents.send('view:navigated', { id, url })
    })

    view.webContents.on('page-title-updated', (event, title) => {
      this.mainWindow.webContents.send('view:titleUpdated', { id, title })
    })

    view.webContents.on('did-fail-load', (event, errorCode, description) => {
      this.mainWindow.webContents.send('view:loadError', { id, errorCode, description })
    })

    view.webContents.on('will-navigate', (event, url) => {
      // ตรวจสอบ URL ก่อน navigation
      if (!this.isAllowedURL(url)) {
        event.preventDefault()
        console.warn('[BrowserView] Blocked navigation to:', url)
      }
    })
  }

  /**
   * Switch ระหว่าง views
   */
  switchView(id) {
    const view = this.views.get(id)
    if (!view) return false

    // ซ่อน views ทั้งหมด
    this.views.forEach((v, viewId) => {
      if (viewId !== id) {
        v.setBounds({ x: 0, y: 0, width: 0, height: 0 })
      }
    })

    // แสดง view ที่เลือก
    const bounds = this.getContentBounds()
    view.setBounds(bounds)
    
    this.activeViewId = id
    return true
  }

  /**
   * ลบ view
   */
  destroyView(id) {
    const view = this.views.get(id)
    if (!view) return

    if (view instanceof BrowserView) {
      this.mainWindow.removeBrowserView(view)
    } else if (view instanceof WebContentsView) {
      this.mainWindow.contentView.removeChildView(view)
    }
    
    view.webContents.destroy()
    this.views.delete(id)
    
    if (this.activeViewId === id) {
      const remaining = [...this.views.keys()]
      this.activeViewId = remaining[0] || null
    }
  }

  /**
   * ดึง bounds สำหรับ content area
   */
  getContentBounds() {
    const windowBounds = this.mainWindow.getBounds()
    const headerHeight = 60 // height ของ navigation bar
    const sidebarWidth = 220 // width ของ sidebar
    
    return {
      x: sidebarWidth,
      y: headerHeight,
      width: windowBounds.width - sidebarWidth,
      height: windowBounds.height - headerHeight
    }
  }

  /**
   * ตรวจสอบ URL ที่อนุญาต
   */
  isAllowedURL(url) {
    const allowedPatterns = [
      /^https?:\/\//,    // HTTP/HTTPS
      /^about:blank/     // Empty page
    ]
    
    return allowedPatterns.some(pattern => pattern.test(url))
  }

  /**
   * Navigate view
   */
  navigateView(id, url) {
    const view = this.views.get(id)
    if (view) {
      view.webContents.loadURL(url)
    }
  }

  /**
   * Execute script ใน view
   */
  async executeInView(id, script) {
    const view = this.views.get(id)
    if (!view) return null
    return view.webContents.executeJavaScript(script)
  }

  /**
   * Screenshot ของ view
   */
  async captureView(id) {
    const view = this.views.get(id)
    if (!view) return null
    
    const image = await view.webContents.capturePage()
    return image.toDataURL()
  }

  getViewIds() {
    return [...this.views.keys()]
  }

  getActiveView() {
    return this.views.get(this.activeViewId)
  }
}

module.exports = BrowserViewManager
```

## Browser App สำหรับ Demo

```jsx
// src/views/BrowserView.jsx
import { useState, useEffect, useRef, useCallback } from 'react'

export default function BrowserView() {
  const [url, setUrl] = useState('https://example.com')
  const [inputUrl, setInputUrl] = useState('')
  const [loading, setLoading] = useState(false)
  const [navInfo, setNavInfo] = useState({ canGoBack: false, canGoForward: false })
  const [findText, setFindText] = useState('')
  const [findResults, setFindResults] = useState({ matches: 0, active: 0 })
  const [viewId] = useState('main-browser')
  const contentRef = useRef(null)

  useEffect(() => {
    if (!window.electronAPI) return

    // สร้าง BrowserView
    const bounds = calculateBounds()
    window.electronAPI.invoke('browserview:create', viewId, url, bounds)

    // Listen for events
    const cleanups = [
      window.electronAPI.on('view:navigated', ({ id, url: newUrl }) => {
        if (id === viewId) {
          setUrl(newUrl)
          setInputUrl(newUrl)
          setLoading(false)
        }
      }),
      window.electronAPI.on('nav:loading', ({ loading: isLoading }) => {
        setLoading(isLoading)
      }),
      window.electronAPI.on('nav:complete', (info) => {
        setNavInfo(info)
        setLoading(false)
      }),
      window.electronAPI.on('findInPage:result', (result) => {
        setFindResults({
          matches: result.matches,
          active: result.activeMatchOrdinal
        })
      })
    ]

    return () => {
      cleanups.forEach(cleanup => cleanup?.())
      window.electronAPI.invoke('browserview:destroy', viewId)
    }
  }, [viewId])

  const calculateBounds = () => {
    const toolbar = document.querySelector('.browser-toolbar')
    const toolbarHeight = toolbar?.offsetHeight || 60
    return {
      x: 220, // sidebar width
      y: toolbarHeight + 32, // titlebar + toolbar
      width: window.innerWidth - 220,
      height: window.innerHeight - toolbarHeight - 32
    }
  }

  const navigate = useCallback((targetUrl) => {
    if (!window.electronAPI) return
    setLoading(true)
    window.electronAPI.invoke('browserview:navigate', viewId, targetUrl)
  }, [viewId])

  const handleNavigate = (e) => {
    e.preventDefault()
    let targetUrl = inputUrl.trim()
    if (!targetUrl.startsWith('http')) targetUrl = 'https://' + targetUrl
    navigate(targetUrl)
  }

  const handleFind = () => {
    if (!window.electronAPI) return
    window.electronAPI.invoke('findInPage:find', findText)
  }

  const handleFindNext = () => {
    window.electronAPI?.invoke('findInPage:next')
  }

  const handleStopFind = () => {
    window.electronAPI?.invoke('findInPage:stop')
    setFindText('')
    setFindResults({ matches: 0, active: 0 })
  }

  const handleCapture = async () => {
    const result = await window.electronAPI?.invoke('capture:fullPage')
    if (result?.success) {
      const link = document.createElement('a')
      link.href = result.dataURL
      link.download = `screenshot-${Date.now()}.png`
      link.click()
    }
  }

  return (
    <div className="browser-view" ref={contentRef}>
      <div className="browser-toolbar">
        <div className="nav-controls">
          <button 
            onClick={() => window.electronAPI?.invoke('nav:back')}
            disabled={!navInfo.canGoBack}
            title="ย้อนกลับ"
          >◀</button>
          <button 
            onClick={() => window.electronAPI?.invoke('nav:forward')}
            disabled={!navInfo.canGoForward}
            title="ไปข้างหน้า"
          >▶</button>
          <button 
            onClick={() => window.electronAPI?.invoke('nav:reload')}
            title="โหลดใหม่"
          >{loading ? '✕' : '↺'}</button>
        </div>

        <form className="url-bar" onSubmit={handleNavigate}>
          {loading && <div className="loading-indicator" />}
          <input
            type="text"
            value={inputUrl}
            onChange={e => setInputUrl(e.target.value)}
            placeholder="ใส่ URL..."
          />
          <button type="submit">ไป</button>
        </form>

        <div className="browser-actions">
          <button onClick={() => setFindText(prev => prev ? '' : ' ')} title="ค้นหา">🔍</button>
          <button onClick={handleCapture} title="Screenshot">📸</button>
          <button 
            onClick={() => window.electronAPI?.invoke('print:toPDF')}
            title="Export PDF"
          >📄</button>
        </div>
      </div>

      {findText !== '' && (
        <div className="find-bar">
          <input
            type="text"
            value={findText}
            onChange={e => setFindText(e.target.value)}
            onKeyDown={e => e.key === 'Enter' && handleFindNext()}
            placeholder="ค้นหา..."
            autoFocus
          />
          <span className="find-count">
            {findResults.active}/{findResults.matches}
          </span>
          <button onClick={handleFind}>ค้นหา</button>
          <button onClick={handleFindNext}>ถัดไป ▼</button>
          <button onClick={() => window.electronAPI?.invoke('findInPage:prev')}>
            ก่อนหน้า ▲
          </button>
          <button onClick={handleStopFind}>✕</button>
        </div>
      )}

      {/* WebContentsView จะ render ที่นี่ผ่าน Electron BrowserView API */}
      <div className="browser-content-placeholder">
        {loading && (
          <div className="loading-overlay">
            <div className="spinner-large" />
          </div>
        )}
      </div>
    </div>
  )
}
```

## สรุป

### เมื่อไหรใช้อะไร

- **webContents API** - ควบคุม content ใน BrowserWindow หลัก
- **BrowserView** - ฝัง web view ใน window (legacy แต่ยังใช้ได้)
- **WebContentsView** - API ใหม่แทน BrowserView ใน Electron 30+
- **executeJavaScript** - Inject code หรือ extract data จาก web page
- **capturePage** - Screenshot สำหรับ export หรือ testing

### Best Practices

1. ใช้ `sandbox: true` สำหรับ untrusted content
2. Validate URLs ก่อน navigation
3. จำกัด JavaScript execution จาก user input
4. Cleanup views เมื่อไม่ใช้งาน
