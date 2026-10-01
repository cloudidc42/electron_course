# Part 96: Advanced Auto-Update Strategies

## Auto-update แบบขั้นสูงใน Electron

ในบทนี้เราจะเรียน staged rollouts, delta updates, background download, channel switching และ rollback mechanisms

---

## 1. electron-updater Setup

```typescript
// src/main/updater.ts
import { autoUpdater, UpdateInfo, ProgressInfo } from 'electron-updater'
import { BrowserWindow, ipcMain, dialog, app } from 'electron'
import log from 'electron-log'

// ตั้งค่า logging
autoUpdater.logger = log
autoUpdater.autoDownload = false    // download ด้วยตัวเอง
autoUpdater.autoInstallOnAppQuit = true

export interface UpdateStatus {
  state: 'idle' | 'checking' | 'available' | 'not-available' | 'downloading' | 'ready' | 'error'
  info?: UpdateInfo
  progress?: ProgressInfo
  error?: string
}

export class AppUpdater {
  private status: UpdateStatus = { state: 'idle' }
  private win: BrowserWindow

  constructor(win: BrowserWindow) {
    this.win = win
    this.setupEvents()
    this.setupIPC()
  }

  private setupEvents(): void {
    autoUpdater.on('checking-for-update', () => {
      this.setStatus({ state: 'checking' })
    })

    autoUpdater.on('update-available', (info: UpdateInfo) => {
      this.setStatus({ state: 'available', info })
      this.notifyUser(info)
    })

    autoUpdater.on('update-not-available', (info: UpdateInfo) => {
      this.setStatus({ state: 'not-available', info })
    })

    autoUpdater.on('download-progress', (progress: ProgressInfo) => {
      this.setStatus({ state: 'downloading', progress })
    })

    autoUpdater.on('update-downloaded', (info: UpdateInfo) => {
      this.setStatus({ state: 'ready', info })
      this.promptInstall(info)
    })

    autoUpdater.on('error', (err: Error) => {
      this.setStatus({ state: 'error', error: err.message })
      log.error('Update error:', err)
    })
  }

  private setupIPC(): void {
    ipcMain.handle('updater:check', () => this.checkForUpdates())
    ipcMain.handle('updater:download', () => this.downloadUpdate())
    ipcMain.handle('updater:install', () => this.installUpdate())
    ipcMain.handle('updater:status', () => this.status)
    ipcMain.handle('updater:get-channel', () => autoUpdater.channel)
    ipcMain.handle('updater:set-channel', async (_, { channel }: { channel: string }) => {
      autoUpdater.channel = channel
      return this.checkForUpdates()
    })
  }

  async checkForUpdates(): Promise<void> {
    try {
      await autoUpdater.checkForUpdates()
    } catch (error) {
      log.error('Check for updates failed:', error)
    }
  }

  async downloadUpdate(): Promise<void> {
    await autoUpdater.downloadUpdate()
  }

  installUpdate(): void {
    autoUpdater.quitAndInstall(false, true)  // isSilent=false, isForceRunAfter=true
  }

  private setStatus(status: UpdateStatus): void {
    this.status = status
    this.win.webContents.send('updater:status-changed', status)
  }

  private notifyUser(info: UpdateInfo): void {
    // แสดง notification ใน app
    this.win.webContents.send('updater:update-available', {
      version: info.version,
      releaseNotes: info.releaseNotes
    })
  }

  private promptInstall(info: UpdateInfo): void {
    dialog.showMessageBox(this.win, {
      type: 'info',
      title: 'อัปเดตพร้อมแล้ว',
      message: `อัปเดต v${info.version} ดาวน์โหลดเสร็จแล้ว`,
      detail: 'ติดตั้งเดี๋ยวนี้หรือรอ?',
      buttons: ['ติดตั้งเดี๋ยวนี้', 'รอก่อน']
    }).then(({ response }) => {
      if (response === 0) this.installUpdate()
    })
  }
}
```

---

## 2. Staged Rollouts

```json
// latest.yml (GitHub Releases format พร้อม staged rollout)
{
  "version": "2.1.0",
  "releaseDate": "2025-10-01T00:00:00.000Z",
  "path": "MyApp-2.1.0.exe",
  "sha512": "...",
  "size": 85000000,
  "stagingPercentage": 10,
  "releaseNotes": "## What's New\n- Bug fixes\n- Performance improvements"
}
```

```typescript
// src/main/stagedRollout.ts
// จัดการ staged rollout เอง ถ้า provider ไม่รองรับ

import * as crypto from 'crypto'
import ElectronStore from 'electron-store'

const store = new ElectronStore<{ rolloutId: string }>()

// สร้าง stable rollout ID สำหรับ device นี้
function getRolloutId(): string {
  let id = store.get('rolloutId')
  if (!id) {
    id = crypto.randomUUID()
    store.set('rolloutId', id)
  }
  return id
}

// ตรวจสอบว่า device นี้อยู่ใน rollout percentage
export function isInRollout(percentage: number): boolean {
  if (percentage >= 100) return true
  if (percentage <= 0) return false

  const id = getRolloutId()
  const hash = crypto.createHash('md5').update(id).digest('hex')
  const bucket = parseInt(hash.slice(0, 8), 16) % 100

  return bucket < percentage
}

// ตรวจสอบกับ update manifest
interface UpdateManifest {
  version: string
  stagingPercentage?: number
  minVersion?: string
  releaseNotes?: string
}

export async function checkStagedUpdate(manifestUrl: string): Promise<UpdateManifest | null> {
  const response = await fetch(manifestUrl)
  const manifest: UpdateManifest = await response.json()

  const percentage = manifest.stagingPercentage ?? 100

  if (!isInRollout(percentage)) {
    console.log(`Update v${manifest.version} not applicable (${percentage}% rollout, not selected)`)
    return null
  }

  return manifest
}
```

---

## 3. Release Channels

```typescript
// src/main/updateChannels.ts
import { autoUpdater } from 'electron-updater'
import ElectronStore from 'electron-store'

export type UpdateChannel = 'stable' | 'beta' | 'alpha' | 'nightly'

const store = new ElectronStore<{ updateChannel: UpdateChannel }>()

export function getCurrentChannel(): UpdateChannel {
  return store.get('updateChannel', 'stable')
}

export function setChannel(channel: UpdateChannel): void {
  store.set('updateChannel', channel)
  autoUpdater.channel = channel
  console.log(`Update channel set to: ${channel}`)
}

// ตั้งค่า feed URL ตาม channel
export function configureFeedURL(): void {
  const channel = getCurrentChannel()
  const baseUrl = 'https://releases.example.com'

  const feedUrls: Record<UpdateChannel, string> = {
    stable: `${baseUrl}/latest`,
    beta: `${baseUrl}/beta`,
    alpha: `${baseUrl}/alpha`,
    nightly: `${baseUrl}/nightly`
  }

  autoUpdater.setFeedURL({
    provider: 'generic',
    url: feedUrls[channel],
    useMultipleRangeRequest: false  // สำหรับ delta updates
  })
}

// Channel selector UI
export function getChannelInfo(channel: UpdateChannel): {
  label: string; description: string; stable: boolean
} {
  const info: Record<UpdateChannel, { label: string; description: string; stable: boolean }> = {
    stable: { label: 'Stable', description: 'เวอร์ชั่นที่มีเสถียรภาพสูง', stable: true },
    beta: { label: 'Beta', description: 'ฟีเจอร์ใหม่ที่ยังทดสอบอยู่', stable: false },
    alpha: { label: 'Alpha', description: 'เวอร์ชั่น development สำหรับผู้กล้า', stable: false },
    nightly: { label: 'Nightly', description: 'Build ล่าสุดทุกคืน อาจไม่เสถียร', stable: false }
  }
  return info[channel]
}
```

---

## 4. Background Download UI

```tsx
// src/renderer/components/UpdateBanner.tsx
import { useState, useEffect } from 'react'
import type { UpdateStatus } from '../../main/updater'

export function UpdateBanner() {
  const [status, setStatus] = useState<UpdateStatus>({ state: 'idle' })
  const [dismissed, setDismissed] = useState(false)

  useEffect(() => {
    // รับ status updates
    const cleanup = window.electronAPI.on('updater:status-changed', (s: UpdateStatus) => {
      setStatus(s as UpdateStatus)
      setDismissed(false)  // reset dismissed เมื่อ state เปลี่ยน
    })

    // Check for updates เมื่อ start
    setTimeout(() => {
      window.electronAPI.invoke('updater:check')
    }, 5000)  // รอ 5 วินาทีหลัง app launch

    return cleanup as (() => void)
  }, [])

  if (dismissed || status.state === 'idle' || status.state === 'not-available') return null

  return (
    <div className={`update-banner update-${status.state}`}>
      {status.state === 'checking' && (
        <span>กำลังตรวจสอบอัปเดต...</span>
      )}

      {status.state === 'available' && (
        <>
          <span>
            🎉 มีอัปเดต v{status.info?.version} พร้อมแล้ว
          </span>
          <button
            className="btn-primary btn-sm"
            onClick={() => window.electronAPI.invoke('updater:download')}
          >
            ดาวน์โหลด
          </button>
          <button className="btn-ghost btn-sm" onClick={() => setDismissed(true)}>
            ภายหลัง
          </button>
        </>
      )}

      {status.state === 'downloading' && status.progress && (
        <>
          <div className="update-progress">
            <div
              className="update-progress-fill"
              style={{ width: `${status.progress.percent}%` }}
            />
          </div>
          <span>
            ดาวน์โหลด {status.progress.percent.toFixed(0)}% (
            {(status.progress.bytesPerSecond / 1024 / 1024).toFixed(1)} MB/s)
          </span>
        </>
      )}

      {status.state === 'ready' && (
        <>
          <span>✅ อัปเดตพร้อมติดตั้ง</span>
          <button
            className="btn-primary btn-sm"
            onClick={() => window.electronAPI.invoke('updater:install')}
          >
            ติดตั้งและ Restart
          </button>
          <button className="btn-ghost btn-sm" onClick={() => setDismissed(true)}>
            หลังปิดแอป
          </button>
        </>
      )}

      {status.state === 'error' && (
        <>
          <span>❌ อัปเดตล้มเหลว: {status.error}</span>
          <button className="btn-ghost btn-sm" onClick={() => setDismissed(true)}>
            ปิด
          </button>
        </>
      )}
    </div>
  )
}
```

---

## 5. Rollback Mechanism

```typescript
// src/main/rollback.ts
import * as path from 'path'
import * as fs from 'fs'
import { app } from 'electron'
import ElectronStore from 'electron-store'

interface VersionRecord {
  version: string
  installedAt: number
  path: string
  isWorking: boolean
}

const store = new ElectronStore<{ versions: VersionRecord[]; failCount: number }>()

export function recordSuccessfulLaunch(): void {
  const versions = store.get('versions', [])
  const current = versions.find(v => v.version === app.getVersion())

  if (current) {
    current.isWorking = true
    store.set('versions', versions)
  } else {
    versions.push({
      version: app.getVersion(),
      installedAt: Date.now(),
      path: app.getPath('exe'),
      isWorking: true
    })
    store.set('versions', versions.slice(-5))  // เก็บ 5 เวอร์ชั่นล่าสุด
  }

  store.set('failCount', 0)
}

export function recordFailedLaunch(): number {
  const count = (store.get('failCount', 0)) + 1
  store.set('failCount', count)
  return count
}

export function shouldRollback(): boolean {
  return store.get('failCount', 0) >= 3
}

export function getLastWorkingVersion(): VersionRecord | null {
  const versions = store.get('versions', [])
  const current = app.getVersion()

  return versions
    .filter(v => v.version !== current && v.isWorking)
    .sort((a, b) => b.installedAt - a.installedAt)[0] ?? null
}
```

---

## สรุป

| ฟีเจอร์ | Implementation | ประโยชน์ |
|--------|---------------|---------|
| Auto-download | autoUpdater.autoDownload | background prep |
| Staged rollout | percentage-based bucket | risk mitigation |
| Channels | stable/beta/alpha/nightly | user choice |
| Background download | ProgressInfo events | ไม่รบกวนผู้ใช้ |
| Delta updates | binary diff patches | ลด download size |
| Rollback | fail count detection | safety net |
