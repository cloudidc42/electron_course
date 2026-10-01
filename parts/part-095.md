# Part 95: Electron App Analytics

## ระบบ Analytics แบบ Privacy-respecting

ในบทนี้เราจะเรียนการสร้าง analytics ที่เคารพ privacy ของผู้ใช้ มีระบบ opt-in/opt-out และเก็บข้อมูลแบบ anonymous

---

## 1. Analytics Service Design

```typescript
// src/shared/analytics.ts

export interface AnalyticsEvent {
  name: string
  properties?: Record<string, string | number | boolean>
  timestamp?: number
}

export interface UserConsent {
  analytics: boolean
  crashReports: boolean
  updatedAt: number
}

export interface AnalyticsSession {
  sessionId: string
  startedAt: number
  events: AnalyticsEvent[]
}
```

```typescript
// src/renderer/analytics/analyticsService.ts

import { AnalyticsEvent, UserConsent } from '../../shared/analytics'

class AnalyticsService {
  private sessionId: string
  private queue: AnalyticsEvent[] = []
  private consent: UserConsent | null = null
  private flushTimer: NodeJS.Timeout | null = null
  private readonly FLUSH_INTERVAL = 30_000
  private readonly MAX_QUEUE_SIZE = 100

  constructor() {
    this.sessionId = this.generateSessionId()
  }

  private generateSessionId(): string {
    // Anonymous session ID (ไม่ผูกกับ user)
    return `${Date.now()}-${Math.random().toString(36).slice(2, 11)}`
  }

  // โหลด consent จาก storage
  async loadConsent(): Promise<void> {
    const stored = localStorage.getItem('analytics-consent')
    if (stored) {
      this.consent = JSON.parse(stored)
    }
    // ถ้ายังไม่ได้ตัดสินใจ = null (ต้องถามผู้ใช้ก่อน)
  }

  setConsent(consent: UserConsent): void {
    this.consent = consent
    localStorage.setItem('analytics-consent', JSON.stringify(consent))

    if (consent.analytics) {
      this.startFlushTimer()
    } else {
      this.stopFlushTimer()
      this.queue = []
    }
  }

  getConsent(): UserConsent | null {
    return this.consent
  }

  // Track event
  track(name: string, properties?: Record<string, string | number | boolean>): void {
    if (!this.consent?.analytics) return

    const event: AnalyticsEvent = {
      name,
      properties: this.sanitizeProperties(properties),
      timestamp: Date.now()
    }

    this.queue.push(event)

    if (this.queue.length >= this.MAX_QUEUE_SIZE) {
      this.flush()
    }
  }

  // ลบ personal data จาก properties
  private sanitizeProperties(props?: Record<string, string | number | boolean>): Record<string, string | number | boolean> | undefined {
    if (!props) return undefined

    const sanitized: Record<string, string | number | boolean> = {}
    const blocklist = ['email', 'name', 'username', 'password', 'token', 'key', 'secret']

    for (const [key, value] of Object.entries(props)) {
      if (blocklist.some(b => key.toLowerCase().includes(b))) continue
      if (typeof value === 'string' && value.includes('@')) continue  // อาจเป็น email

      sanitized[key] = value
    }

    return sanitized
  }

  // Track page view / screen
  screen(screenName: string, properties?: Record<string, string | number | boolean>): void {
    this.track('screen_view', { screen: screenName, ...properties })
  }

  // Track feature usage
  feature(featureName: string, action = 'used'): void {
    this.track('feature', { feature: featureName, action })
  }

  // Track timing
  timing(category: string, variable: string, durationMs: number): void {
    this.track('timing', { category, variable, duration_ms: Math.round(durationMs) })
  }

  private startFlushTimer(): void {
    if (this.flushTimer) return
    this.flushTimer = setInterval(() => this.flush(), this.FLUSH_INTERVAL)
  }

  private stopFlushTimer(): void {
    if (this.flushTimer) {
      clearInterval(this.flushTimer)
      this.flushTimer = null
    }
  }

  async flush(): Promise<void> {
    if (this.queue.length === 0) return

    const events = [...this.queue]
    this.queue = []

    try {
      await this.sendEvents(events)
    } catch (error) {
      // ถ้าส่งไม่ได้ ใส่กลับเข้า queue
      this.queue = [...events.slice(-50), ...this.queue]  // เก็บ 50 events ล่าสุด
      console.error('Analytics flush failed:', error)
    }
  }

  private async sendEvents(events: AnalyticsEvent[]): Promise<void> {
    const payload = {
      session_id: this.sessionId,
      app_version: await window.electronAPI.invoke('app:version') as string,
      platform: navigator.platform,
      language: navigator.language.split('-')[0],  // เฉพาะ language code
      events
    }

    // ส่งไปยัง analytics server
    await fetch('https://analytics.example.com/collect', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    })
  }

  // Export data (GDPR right to access)
  exportData(): string {
    return JSON.stringify({
      sessionId: this.sessionId,
      queuedEvents: this.queue,
      consent: this.consent
    }, null, 2)
  }

  // Delete all data (GDPR right to erasure)
  deleteAllData(): void {
    this.queue = []
    this.consent = null
    localStorage.removeItem('analytics-consent')
  }
}

export const analytics = new AnalyticsService()
```

---

## 2. Consent UI

```tsx
// src/renderer/components/AnalyticsConsent.tsx
import { useState } from 'react'
import { analytics } from '../analytics/analyticsService'
import type { UserConsent } from '../../shared/analytics'

interface ConsentDialogProps {
  onDecide: (consent: UserConsent) => void
}

export function AnalyticsConsentDialog({ onDecide }: ConsentDialogProps) {
  const [analyticsEnabled, setAnalyticsEnabled] = useState(false)
  const [crashReportsEnabled, setCrashReportsEnabled] = useState(true)

  const handleDecide = (accept: boolean) => {
    const consent: UserConsent = {
      analytics: accept ? analyticsEnabled : false,
      crashReports: accept ? crashReportsEnabled : false,
      updatedAt: Date.now()
    }
    analytics.setConsent(consent)
    onDecide(consent)
  }

  return (
    <div className="consent-dialog" role="dialog" aria-modal="true">
      <div className="consent-content">
        <h2>ช่วยเราปรับปรุง My App</h2>
        <p>เราต้องการข้อมูลการใช้งานเพื่อปรับปรุงแอป ข้อมูลทั้งหมดจะถูกเก็บแบบ anonymous</p>

        <div className="consent-options">
          <ConsentOption
            title="Usage Analytics"
            description="เก็บข้อมูลฟีเจอร์ที่คุณใช้ เพื่อปรับปรุง UX (ไม่มีข้อมูลส่วนตัว)"
            checked={analyticsEnabled}
            onChange={setAnalyticsEnabled}
          />
          <ConsentOption
            title="Crash Reports"
            description="รายงานเมื่อแอป crash เพื่อให้เราแก้ bug ได้เร็วขึ้น"
            checked={crashReportsEnabled}
            onChange={setCrashReportsEnabled}
          />
        </div>

        <div className="consent-info">
          <p>
            ข้อมูลที่เก็บ: เวอร์ชั่นแอป, OS, ฟีเจอร์ที่ใช้, session duration
          </p>
          <p>
            ข้อมูลที่ <strong>ไม่เก็บ</strong>: ชื่อ, email, เนื้อหาไฟล์, location
          </p>
          <a href="#" onClick={() => window.electronAPI.send('open:privacy-policy')}>
            นโยบายความเป็นส่วนตัว →
          </a>
        </div>

        <div className="consent-actions">
          <button className="btn" onClick={() => handleDecide(false)}>
            ไม่ยินยอม
          </button>
          <button className="btn-primary" onClick={() => handleDecide(true)}>
            ยินยอม
          </button>
        </div>
      </div>
    </div>
  )
}

function ConsentOption({
  title, description, checked, onChange
}: {
  title: string; description: string; checked: boolean; onChange: (v: boolean) => void
}) {
  return (
    <label className="consent-option">
      <input
        type="checkbox"
        checked={checked}
        onChange={e => onChange(e.target.checked)}
      />
      <div className="option-text">
        <strong>{title}</strong>
        <span>{description}</span>
      </div>
    </label>
  )
}
```

---

## 3. Usage in App

```typescript
// src/renderer/App.tsx (ตัวอย่างการใช้ analytics)
import { useEffect } from 'react'
import { analytics } from './analytics/analyticsService'

// Track app startup
analytics.screen('main')
analytics.track('app_launched', {
  version: window.__APP_VERSION__
})

// Track feature usage
function handleOpenFile() {
  const start = performance.now()
  // ... open file logic
  analytics.timing('file_operations', 'open_file', performance.now() - start)
  analytics.feature('file_open')
}

// Track errors
window.addEventListener('error', (e) => {
  analytics.track('js_error', {
    message: e.message.slice(0, 100),  // จำกัดความยาว
    source: e.filename?.split('/').pop() ?? 'unknown'
  })
})
```

---

## 4. Privacy Settings UI

```tsx
// src/renderer/components/PrivacySettings.tsx
import { useState, useEffect } from 'react'
import { analytics } from '../analytics/analyticsService'

export function PrivacySettings() {
  const [consent, setConsent] = useState(analytics.getConsent())

  const handleToggle = (key: 'analytics' | 'crashReports', value: boolean) => {
    const newConsent = {
      ...(consent ?? { analytics: false, crashReports: false, updatedAt: 0 }),
      [key]: value,
      updatedAt: Date.now()
    }
    analytics.setConsent(newConsent)
    setConsent(newConsent)
  }

  const handleDeleteData = () => {
    if (confirm('ลบข้อมูลการใช้งานทั้งหมด?')) {
      analytics.deleteAllData()
      setConsent(null)
    }
  }

  const handleExportData = () => {
    const data = analytics.exportData()
    const blob = new Blob([data], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = 'my-app-data.json'
    a.click()
    URL.revokeObjectURL(url)
  }

  return (
    <div className="privacy-settings">
      <h3>ความเป็นส่วนตัว</h3>

      <div className="settings-group">
        <ToggleSetting
          title="Usage Analytics"
          description="ช่วยพัฒนา app ด้วยข้อมูล anonymous"
          checked={consent?.analytics ?? false}
          onChange={v => handleToggle('analytics', v)}
        />
        <ToggleSetting
          title="Crash Reports"
          description="รายงาน crashes อัตโนมัติ"
          checked={consent?.crashReports ?? false}
          onChange={v => handleToggle('crashReports', v)}
        />
      </div>

      <div className="data-rights">
        <h4>สิทธิ์ของคุณ (GDPR)</h4>
        <button className="btn" onClick={handleExportData}>
          ดาวน์โหลดข้อมูลของฉัน
        </button>
        <button className="btn-danger" onClick={handleDeleteData}>
          ลบข้อมูลทั้งหมด
        </button>
      </div>
    </div>
  )
}

function ToggleSetting({ title, description, checked, onChange }: {
  title: string; description: string; checked: boolean; onChange: (v: boolean) => void
}) {
  return (
    <div className="toggle-setting">
      <div className="setting-text">
        <strong>{title}</strong>
        <p>{description}</p>
      </div>
      <label className="toggle-switch">
        <input type="checkbox" checked={checked} onChange={e => onChange(e.target.checked)} />
        <span className="switch-slider" />
      </label>
    </div>
  )
}
```

---

## สรุป

| หลักการ | Implementation |
|--------|---------------|
| Opt-in consent | Dialog ก่อนเก็บข้อมูล |
| Data minimization | เก็บเฉพาะที่จำเป็น |
| Anonymization | ไม่มี user ID |
| Sanitization | ลบ PII ออกอัตโนมัติ |
| Right to access | Export JSON |
| Right to erasure | Delete all data |
| Offline queue | Retry เมื่อ network กลับมา |
