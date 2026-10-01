# Part 25: Auto Updater ใน Electron

## บทนำ

Auto Updater เป็นฟีเจอร์สำคัญที่ทำให้ user ได้รับ version ล่าสุดของแอปโดยอัตโนมัติ Electron มี `autoUpdater` module built-in แต่ `electron-updater` จาก electron-builder มีความสามารถมากกว่า รองรับ GitHub Releases, S3, และ custom update server

## การติดตั้ง

```bash
npm install electron-updater
npm install --save-dev electron-builder
```

## GitHub Releases Auto Updater

### ตั้งค่า electron-builder.json

```json
{
  "appId": "com.yourcompany.myapp",
  "productName": "My App",
  "publish": [
    {
      "provider": "github",
      "owner": "yourusername",
      "repo": "your-app-repo",
      "private": false,
      "releaseType": "release"
    }
  ],
  "win": {
    "target": ["nsis"],
    "publisherName": "Your Company"
  },
  "mac": {
    "target": ["dmg", "zip"],
    "category": "public.app-category.productivity"
  },
  "linux": {
    "target": ["AppImage"],
    "category": "Utility"
  }
}
```

### Auto Updater Service

```javascript
// electron/services/updater.js
const { autoUpdater } = require('electron-updater')
const { app, BrowserWindow, dialog, ipcMain } = require('electron')
const log = require('electron-log')
const path = require('path')

// ตั้งค่า logging
autoUpdater.logger = log
autoUpdater.logger.transports.file.level = 'info'

class UpdaterService {
  constructor() {
    this.mainWindow = null
    this.updateAvailable = false
    this.updateDownloaded = false
    this.updateInfo = null
    this.isChecking = false
    
    this.setupAutoUpdater()
  }

  /**
   * ตั้งค่า updater
   */
  initialize(mainWindow) {
    this.mainWindow = mainWindow
    
    // ตั้งค่า update feed
    autoUpdater.autoDownload = false          // ไม่ download อัตโนมัติ
    autoUpdater.autoInstallOnAppQuit = true   // Install เมื่อปิดแอป
    autoUpdater.allowPrerelease = false       // ไม่ใช้ pre-release
    autoUpdater.allowDowngrade = false        // ไม่ downgrade
    
    // Check update เมื่อ startup (หลังจาก 5 วินาที)
    if (app.isPackaged) {
      setTimeout(() => {
        this.checkForUpdates()
      }, 5000)
      
      // Check ทุก 4 ชั่วโมง
      setInterval(() => {
        this.checkForUpdates()
      }, 4 * 60 * 60 * 1000)
    }
  }

  /**
   * Setup event listeners
   */
  setupAutoUpdater() {
    autoUpdater.on('checking-for-update', () => {
      log.info('[Updater] Checking for updates...')
      this.isChecking = true
      this.sendStatusToWindow('checking-for-update', {
        message: 'กำลังตรวจสอบการอัปเดต...'
      })
    })

    autoUpdater.on('update-available', (info) => {
      log.info('[Updater] Update available:', info.version)
      this.isChecking = false
      this.updateAvailable = true
      this.updateInfo = info
      
      this.sendStatusToWindow('update-available', {
        version: info.version,
        releaseDate: info.releaseDate,
        releaseNotes: info.releaseNotes,
        message: `มีอัปเดตใหม่: v${info.version}`
      })
      
      // แสดง notification
      this.showUpdateNotification(info)
    })

    autoUpdater.on('update-not-available', (info) => {
      log.info('[Updater] No update available. Current version:', info.version)
      this.isChecking = false
      this.updateAvailable = false
      
      this.sendStatusToWindow('update-not-available', {
        version: info.version,
        message: 'แอปเป็นเวอร์ชันล่าสุดแล้ว'
      })
    })

    autoUpdater.on('error', (err) => {
      log.error('[Updater] Error:', err.message)
      this.isChecking = false
      
      this.sendStatusToWindow('error', {
        error: err.message,
        message: `เกิดข้อผิดพลาด: ${err.message}`
      })
    })

    autoUpdater.on('download-progress', (progressObj) => {
      const logMessage = [
        `Download speed: ${this.formatBytes(progressObj.bytesPerSecond)}/s`,
        `- Downloaded ${progressObj.percent.toFixed(1)}%`,
        `(${this.formatBytes(progressObj.transferred)}/${this.formatBytes(progressObj.total)})`
      ].join(' ')
      
      log.info('[Updater]', logMessage)
      
      this.sendStatusToWindow('download-progress', {
        percent: progressObj.percent,
        bytesPerSecond: progressObj.bytesPerSecond,
        transferred: progressObj.transferred,
        total: progressObj.total,
        message: `กำลังดาวน์โหลด: ${progressObj.percent.toFixed(1)}%`
      })
    })

    autoUpdater.on('update-downloaded', (info) => {
      log.info('[Updater] Update downloaded:', info.version)
      this.updateDownloaded = true
      
      this.sendStatusToWindow('update-downloaded', {
        version: info.version,
        releaseNotes: info.releaseNotes,
        message: `ดาวน์โหลด v${info.version} เสร็จแล้ว`
      })
      
      // แจ้งให้ user ทราบ
      this.promptInstallUpdate(info)
    })
  }

  /**
   * ตรวจสอบการอัปเดต
   */
  async checkForUpdates(silent = true) {
    if (this.isChecking) return
    if (!app.isPackaged) {
      log.info('[Updater] Skipping update check in development mode')
      if (!silent) {
        this.sendStatusToWindow('update-not-available', {
          message: 'ไม่ตรวจสอบในโหมด development'
        })
      }
      return
    }

    try {
      this.isChecking = true
      await autoUpdater.checkForUpdates()
    } catch (error) {
      log.error('[Updater] Check failed:', error)
      this.isChecking = false
      if (!silent) {
        this.sendStatusToWindow('error', {
          error: error.message,
          message: `ตรวจสอบล้มเหลว: ${error.message}`
        })
      }
    }
  }

  /**
   * ดาวน์โหลด update
   */
  async downloadUpdate() {
    if (!this.updateAvailable) return
    
    try {
      await autoUpdater.downloadUpdate()
    } catch (error) {
      log.error('[Updater] Download failed:', error)
      this.sendStatusToWindow('error', {
        error: error.message,
        message: `ดาวน์โหลดล้มเหลว: ${error.message}`
      })
    }
  }

  /**
   * Install update และ restart
   */
  installUpdate() {
    if (!this.updateDownloaded) return
    
    // บันทึกข้อมูลก่อน restart
    autoUpdater.quitAndInstall(false, true)
  }

  /**
   * แสดง notification เมื่อมี update
   */
  showUpdateNotification(info) {
    if (!this.mainWindow) return
    
    // ส่ง notification ผ่าน IPC
    this.mainWindow.webContents.send('app:notification', {
      type: 'update',
      title: 'มีอัปเดตใหม่',
      body: `v${info.version} พร้อมให้ดาวน์โหลดแล้ว`,
      action: 'download-update'
    })
  }

  /**
   * ถามว่าต้องการ install ทันทีหรือไม่
   */
  async promptInstallUpdate(info) {
    if (!this.mainWindow) return
    
    const response = await dialog.showMessageBox(this.mainWindow, {
      type: 'info',
      title: 'อัปเดตพร้อมแล้ว',
      message: `v${info.version} พร้อม Install แล้ว`,
      detail: `${this.formatReleaseNotes(info.releaseNotes)}\n\nต้องการ restart เพื่อ install ตอนนี้หรือไม่?`,
      buttons: ['Restart ทันที', 'ทีหลัง'],
      defaultId: 0,
      cancelId: 1
    })

    if (response.response === 0) {
      this.installUpdate()
    }
  }

  /**
   * ส่งสถานะไปยัง renderer
   */
  sendStatusToWindow(status, data = {}) {
    if (!this.mainWindow || this.mainWindow.isDestroyed()) return
    
    this.mainWindow.webContents.send('updater:status', {
      status,
      ...data,
      timestamp: new Date().toISOString()
    })
  }

  /**
   * Helper functions
   */
  formatBytes(bytes) {
    if (bytes === 0) return '0 B'
    const k = 1024
    const sizes = ['B', 'KB', 'MB', 'GB']
    const i = Math.floor(Math.log(bytes) / Math.log(k))
    return `${parseFloat((bytes / Math.pow(k, i)).toFixed(2))} ${sizes[i]}`
  }

  formatReleaseNotes(notes) {
    if (!notes) return ''
    if (Array.isArray(notes)) {
      return notes.map(n => n.note || n).join('\n')
    }
    // Strip HTML tags
    return notes.replace(/<[^>]*>/g, '').trim()
  }

  /**
   * ดึงสถานะปัจจุบัน
   */
  getStatus() {
    return {
      updateAvailable: this.updateAvailable,
      updateDownloaded: this.updateDownloaded,
      updateInfo: this.updateInfo,
      isChecking: this.isChecking,
      currentVersion: app.getVersion()
    }
  }
}

const updaterService = new UpdaterService()

/**
 * Setup IPC handlers
 */
function setupUpdaterHandlers() {
  ipcMain.handle('updater:check', () => {
    return updaterService.checkForUpdates(false)
  })

  ipcMain.handle('updater:download', () => {
    return updaterService.downloadUpdate()
  })

  ipcMain.handle('updater:install', () => {
    updaterService.installUpdate()
  })

  ipcMain.handle('updater:status', () => {
    return updaterService.getStatus()
  })

  ipcMain.handle('updater:skipVersion', (event, version) => {
    // บันทึก version ที่ user ต้องการ skip
    const store = require('../store/appStore')
    store.set('skippedVersion', version)
  })
}

module.exports = { updaterService, setupUpdaterHandlers }
```

## S3 Update Server

```javascript
// electron/services/s3Updater.js
const { autoUpdater } = require('electron-updater')

/**
 * ตั้งค่า S3 as update provider
 */
function setupS3Updater() {
  // ตั้งค่า feed URL สำหรับ S3
  autoUpdater.setFeedURL({
    provider: 's3',
    bucket: 'your-bucket-name',
    region: 'ap-southeast-1',
    path: '/updates/',
    // ACL: 'public-read'
  })
  
  // หรือใช้ generic HTTP provider
  // autoUpdater.setFeedURL({
  //   provider: 'generic',
  //   url: 'https://your-update-server.com/updates',
  //   channel: 'latest'
  // })
}

module.exports = { setupS3Updater }
```

## Update Notification UI (React)

```jsx
// src/components/UpdateNotification.jsx
import { useState, useEffect } from 'react'
import './UpdateNotification.css'

function UpdateNotification() {
  const [updateState, setUpdateState] = useState({
    status: 'idle', // idle | checking | available | downloading | downloaded | error
    version: null,
    progress: 0,
    releaseNotes: '',
    error: null
  })
  const [visible, setVisible] = useState(false)
  const [minimized, setMinimized] = useState(false)

  useEffect(() => {
    if (!window.electronAPI) return

    // Listen for updater events
    const cleanup = window.electronAPI.on('updater:status', (data) => {
      setUpdateState(prev => ({
        ...prev,
        status: data.status,
        version: data.version || prev.version,
        progress: data.percent || prev.progress,
        releaseNotes: data.releaseNotes || prev.releaseNotes,
        error: data.error || null
      }))

      // แสดง notification สำหรับ events สำคัญ
      if (['available', 'downloaded', 'error'].includes(data.status)) {
        setVisible(true)
        setMinimized(false)
      }
    })

    return cleanup
  }, [])

  const handleDownload = async () => {
    setUpdateState(prev => ({ ...prev, status: 'downloading' }))
    await window.electronAPI.invoke('updater:download')
  }

  const handleInstall = async () => {
    await window.electronAPI.invoke('updater:install')
  }

  const handleCheck = async () => {
    setUpdateState(prev => ({ ...prev, status: 'checking' }))
    await window.electronAPI.invoke('updater:check')
  }

  const handleDismiss = () => {
    setVisible(false)
  }

  const handleSkip = async () => {
    if (updateState.version) {
      await window.electronAPI.invoke('updater:skipVersion', updateState.version)
    }
    setVisible(false)
  }

  if (!visible) {
    return (
      <button 
        className="update-check-btn"
        onClick={handleCheck}
        title="ตรวจสอบการอัปเดต"
      >
        🔄
      </button>
    )
  }

  return (
    <div className={`update-notification ${minimized ? 'minimized' : ''}`}>
      <div className="update-header">
        <span className="update-icon">
          {updateState.status === 'error' ? '❌' : 
           updateState.status === 'downloaded' ? '✅' : '🔄'}
        </span>
        <span className="update-title">
          {getStatusTitle(updateState.status)}
        </span>
        <div className="update-actions-header">
          <button onClick={() => setMinimized(!minimized)}>
            {minimized ? '▲' : '▼'}
          </button>
          <button onClick={handleDismiss}>✕</button>
        </div>
      </div>

      {!minimized && (
        <div className="update-body">
          {updateState.status === 'checking' && (
            <div className="update-checking">
              <div className="spinner" />
              <p>กำลังตรวจสอบการอัปเดต...</p>
            </div>
          )}

          {updateState.status === 'available' && (
            <div className="update-available">
              <p className="version-badge">v{updateState.version}</p>
              {updateState.releaseNotes && (
                <div className="release-notes">
                  <strong>สิ่งที่ใหม่:</strong>
                  <p>{updateState.releaseNotes}</p>
                </div>
              )}
              <div className="update-buttons">
                <button 
                  className="btn-primary"
                  onClick={handleDownload}
                >
                  ⬇️ ดาวน์โหลด
                </button>
                <button 
                  className="btn-secondary"
                  onClick={handleSkip}
                >
                  ข้ามเวอร์ชันนี้
                </button>
              </div>
            </div>
          )}

          {updateState.status === 'downloading' && (
            <div className="update-downloading">
              <div className="progress-bar">
                <div 
                  className="progress-fill"
                  style={{ width: `${updateState.progress}%` }}
                />
              </div>
              <p>{updateState.progress.toFixed(1)}% กำลังดาวน์โหลด...</p>
            </div>
          )}

          {updateState.status === 'downloaded' && (
            <div className="update-downloaded">
              <p>✅ ดาวน์โหลดเสร็จแล้ว! พร้อม Install</p>
              <div className="update-buttons">
                <button 
                  className="btn-primary"
                  onClick={handleInstall}
                >
                  🚀 Restart & Install
                </button>
                <button 
                  className="btn-secondary"
                  onClick={handleDismiss}
                >
                  ทีหลัง
                </button>
              </div>
            </div>
          )}

          {updateState.status === 'error' && (
            <div className="update-error">
              <p className="error-message">❌ {updateState.error}</p>
              <button 
                className="btn-secondary"
                onClick={handleCheck}
              >
                ลองใหม่
              </button>
            </div>
          )}

          {updateState.status === 'up-to-date' && (
            <p>✅ แอปเป็นเวอร์ชันล่าสุดแล้ว</p>
          )}
        </div>
      )}
    </div>
  )
}

function getStatusTitle(status) {
  const titles = {
    idle: 'อัปเดต',
    checking: 'กำลังตรวจสอบ',
    available: 'มีอัปเดตใหม่',
    downloading: 'กำลังดาวน์โหลด',
    downloaded: 'พร้อม Install',
    error: 'เกิดข้อผิดพลาด',
    'up-to-date': 'เป็นเวอร์ชันล่าสุด'
  }
  return titles[status] || 'อัปเดต'
}

export default UpdateNotification
```

## CSS สำหรับ Update Notification

```css
/* src/components/UpdateNotification.css */
.update-notification {
  position: fixed;
  bottom: 20px;
  right: 20px;
  width: 340px;
  background: var(--bg-surface, #ffffff);
  border: 1px solid var(--border-color, #e2e8f0);
  border-radius: 12px;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
  z-index: 9999;
  overflow: hidden;
  transition: all 0.3s ease;
}

.update-notification.minimized {
  height: 44px;
}

.update-header {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 16px;
  background: var(--bg-subtle, #f8fafc);
  border-bottom: 1px solid var(--border-color, #e2e8f0);
}

.update-title {
  flex: 1;
  font-weight: 600;
  font-size: 14px;
}

.update-body {
  padding: 16px;
}

.version-badge {
  display: inline-block;
  background: #6366f1;
  color: white;
  padding: 2px 10px;
  border-radius: 20px;
  font-size: 13px;
  font-weight: 600;
  margin-bottom: 10px;
}

.progress-bar {
  height: 6px;
  background: var(--border-color, #e2e8f0);
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 8px;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #6366f1, #818cf8);
  border-radius: 3px;
  transition: width 0.3s ease;
}

.update-buttons {
  display: flex;
  gap: 8px;
  margin-top: 12px;
}

.btn-primary, .btn-secondary {
  flex: 1;
  padding: 8px 12px;
  border: none;
  border-radius: 8px;
  font-size: 13px;
  cursor: pointer;
  font-weight: 500;
}

.btn-primary {
  background: #6366f1;
  color: white;
}

.btn-secondary {
  background: var(--bg-subtle, #f1f5f9);
  color: var(--text-primary, #1e293b);
}

.spinner {
  width: 20px;
  height: 20px;
  border: 2px solid #e2e8f0;
  border-top-color: #6366f1;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  margin: 0 auto 8px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.error-message {
  color: #ef4444;
  font-size: 13px;
}

.release-notes {
  background: var(--bg-subtle, #f8fafc);
  border-radius: 8px;
  padding: 10px;
  margin: 8px 0;
  font-size: 13px;
  max-height: 100px;
  overflow-y: auto;
}
```

## Differential Updates

```javascript
// electron/services/differentialUpdater.js
/**
 * Differential updates ช่วยลดขนาด download
 * โดยดาวน์โหลดเฉพาะส่วนที่เปลี่ยนแปลง
 * electron-builder รองรับ NSIS delta updates บน Windows
 */

const { autoUpdater } = require('electron-updater')

function configureDifferentialUpdate() {
  // electron-updater รองรับ blockmap-based differential updates
  // โดยอัตโนมัติเมื่อ publish ด้วย electron-builder
  
  // ตั้งค่าให้ use differential updates
  autoUpdater.setFeedURL({
    provider: 'github',
    owner: 'yourusername',
    repo: 'your-repo',
    // Differential updates จะทำงานอัตโนมัติ
    // ต้องมี .blockmap files ใน release
  })

  // Listen for download-progress เพื่อแสดง differential stats
  autoUpdater.on('download-progress', (progressObj) => {
    console.log(`
      Total: ${formatBytes(progressObj.total)}
      Downloaded: ${formatBytes(progressObj.transferred)}
      Speed: ${formatBytes(progressObj.bytesPerSecond)}/s
      Progress: ${progressObj.percent.toFixed(1)}%
    `)
  })
}

function formatBytes(bytes) {
  if (bytes === 0) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return `${(bytes / Math.pow(k, i)).toFixed(1)} ${sizes[i]}`
}
```

## Rollback Mechanism

```javascript
// electron/services/rollback.js
const { app } = require('electron')
const path = require('path')
const fs = require('fs')
const Store = require('electron-store')

const store = new Store()

class RollbackManager {
  constructor() {
    this.backupDir = path.join(app.getPath('userData'), 'backups')
    this.maxBackups = 3
  }

  /**
   * บันทึก backup ก่อน update
   */
  async createBackup() {
    const version = app.getVersion()
    const timestamp = Date.now()
    const backupName = `v${version}-${timestamp}`
    
    // บันทึก version info
    store.set('lastBackup', {
      version,
      timestamp,
      backupName
    })

    console.log(`[Rollback] Created backup: ${backupName}`)
    return backupName
  }

  /**
   * Rollback ไปยัง version ก่อนหน้า
   * หมายเหตุ: การ rollback ใน Electron ทำได้จำกัด
   * ส่วนใหญ่ต้องให้ user download version เก่าเอง
   */
  async rollback() {
    const lastBackup = store.get('lastBackup')
    if (!lastBackup) {
      throw new Error('ไม่มี backup สำหรับ rollback')
    }

    console.log('[Rollback] Rolling back to:', lastBackup.version)
    
    // Log rollback event
    store.set('rollbackEvent', {
      fromVersion: app.getVersion(),
      toVersion: lastBackup.version,
      timestamp: Date.now()
    })
    
    return lastBackup
  }

  /**
   * ตรวจสอบว่า update สำเร็จหรือไม่
   * เรียกหลังจาก app restart
   */
  verifyUpdate(expectedVersion) {
    const currentVersion = app.getVersion()
    const success = currentVersion === expectedVersion
    
    if (success) {
      console.log('[Rollback] Update verified successfully:', currentVersion)
      store.delete('pendingUpdate')
    } else {
      console.error('[Rollback] Update verification failed!')
      console.error('Expected:', expectedVersion, 'Got:', currentVersion)
    }
    
    return success
  }
}

module.exports = new RollbackManager()
```

## Settings View สำหรับ Update

```jsx
// src/views/UpdateSettings.jsx
import { useState, useEffect } from 'react'

export default function UpdateSettings() {
  const [currentVersion, setCurrentVersion] = useState('')
  const [updateStatus, setUpdateStatus] = useState('idle')
  const [autoCheck, setAutoCheck] = useState(true)
  const [channel, setChannel] = useState('stable')

  useEffect(() => {
    if (!window.electronAPI) return
    
    window.electronAPI.app.getVersion().then(setCurrentVersion)
    
    // ดึงสถานะ updater
    window.electronAPI.invoke('updater:status').then((status) => {
      setUpdateStatus(status.updateAvailable ? 'available' : 'idle')
    })

    // Listen สำหรับ events
    const cleanup = window.electronAPI.on('updater:status', (data) => {
      setUpdateStatus(data.status)
    })

    return cleanup
  }, [])

  const checkForUpdates = async () => {
    setUpdateStatus('checking')
    await window.electronAPI.invoke('updater:check')
  }

  return (
    <div className="update-settings">
      <h2>การอัปเดต</h2>

      <div className="setting-group">
        <div className="setting-item">
          <label>เวอร์ชันปัจจุบัน</label>
          <span className="version-display">v{currentVersion}</span>
        </div>

        <div className="setting-item">
          <label>Release Channel</label>
          <select 
            value={channel}
            onChange={(e) => setChannel(e.target.value)}
          >
            <option value="stable">Stable</option>
            <option value="beta">Beta</option>
          </select>
        </div>

        <div className="setting-item">
          <label>ตรวจสอบอัตโนมัติ</label>
          <input 
            type="checkbox"
            checked={autoCheck}
            onChange={(e) => setAutoCheck(e.target.checked)}
          />
        </div>
      </div>

      <div className="update-action">
        <button 
          className="btn-primary"
          onClick={checkForUpdates}
          disabled={updateStatus === 'checking' || updateStatus === 'downloading'}
        >
          {updateStatus === 'checking' ? '🔄 กำลังตรวจสอบ...' : '🔍 ตรวจสอบการอัปเดต'}
        </button>

        {updateStatus === 'available' && (
          <div className="update-available-notice">
            <p>🎉 มีอัปเดตใหม่พร้อม</p>
            <button 
              className="btn-primary"
              onClick={() => window.electronAPI.invoke('updater:download')}
            >
              ⬇️ ดาวน์โหลด
            </button>
          </div>
        )}

        {updateStatus === 'downloaded' && (
          <div className="update-ready-notice">
            <p>✅ พร้อม Install แล้ว</p>
            <button 
              className="btn-success"
              onClick={() => window.electronAPI.invoke('updater:install')}
            >
              🚀 Install & Restart
            </button>
          </div>
        )}
      </div>

      <div className="update-history">
        <h3>ประวัติการอัปเดต</h3>
        <p className="text-muted">ดูประวัติการเปลี่ยนแปลงบน GitHub Releases</p>
        <button 
          className="btn-link"
          onClick={() => window.electronAPI?.shell.openExternal(
            'https://github.com/yourusername/your-repo/releases'
          )}
        >
          🔗 ดูบน GitHub
        </button>
      </div>
    </div>
  )
}
```

## สรุป

### Best Practices สำหรับ Auto Updater

1. **ไม่ Auto-download** ใน production - ให้ user เลือก
2. **แสดง Release Notes** เพื่อให้ user รู้ว่าอัปเดตนั้นมีอะไร
3. **Background download** - ดาวน์โหลดในพื้นหลังโดยไม่ block UI
4. **Verify update** หลัง install
5. **Log ทุกขั้นตอน** สำหรับ debugging
6. **Handle errors gracefully** - แสดงข้อผิดพลาดที่เข้าใจได้
7. **Skip version** - ให้ user ข้าม version ที่ไม่ต้องการได้
