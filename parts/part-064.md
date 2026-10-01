# Part 64: Project - Pomodoro Timer

## สร้าง Pomodoro Timer แบบ Professional

ในบทนี้เราจะสร้าง Pomodoro Timer ที่มีระบบนับถอยหลัง, สถิติ, system notifications, tray integration และ GitHub-style contribution graph

---

## โครงสร้างโปรเจค

```
pomodoro-timer/
├── src/
│   ├── main/
│   │   ├── index.ts
│   │   ├── trayManager.ts
│   │   └── notificationManager.ts
│   ├── renderer/
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── Timer.tsx
│   │   │   ├── SessionList.tsx
│   │   │   ├── ContributionGraph.tsx
│   │   │   ├── Settings.tsx
│   │   │   └── Stats.tsx
│   │   └── store/
│   │       └── pomodoroStore.ts
│   └── preload/index.ts
```

---

## 1. Main Process

```typescript
// src/main/index.ts
import { app, BrowserWindow, ipcMain, Notification, nativeImage } from 'electron'
import { join } from 'path'
import { setupTray } from './trayManager'
import Store from 'electron-store'

interface SessionRecord {
  id: string
  type: 'work' | 'short-break' | 'long-break'
  duration: number
  completedAt: number
  date: string
}

const store = new Store<{ sessions: SessionRecord[] }>({
  defaults: { sessions: [] }
})

let mainWindow: BrowserWindow | null = null

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 420,
    height: 680,
    resizable: false,
    titleBarStyle: 'hiddenInset',
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      nodeIntegration: false,
      contextIsolation: true
    },
    backgroundColor: '#1a1a2e'
  })

  setupTray(mainWindow)
  registerHandlers()

  if (process.env.ELECTRON_RENDERER_URL) {
    mainWindow.loadURL(process.env.ELECTRON_RENDERER_URL)
  } else {
    mainWindow.loadFile(join(__dirname, '../renderer/index.html'))
  }
}

function registerHandlers() {
  // บันทึก session ที่เสร็จแล้ว
  ipcMain.handle('session:save', (_, session: SessionRecord) => {
    const sessions = store.get('sessions', [])
    sessions.push(session)
    // เก็บแค่ 365 วันล่าสุด
    const cutoff = Date.now() - 365 * 24 * 60 * 60 * 1000
    const filtered = sessions.filter(s => s.completedAt > cutoff)
    store.set('sessions', filtered)
    return filtered
  })

  // ดึงข้อมูล sessions
  ipcMain.handle('session:get-all', () => {
    return store.get('sessions', [])
  })

  // ส่ง notification
  ipcMain.handle('notification:send', (_, { title, body, sound }: { title: string; body: string; sound?: string }) => {
    if (!Notification.isSupported()) return

    const notification = new Notification({
      title,
      body,
      silent: false,
      urgency: 'normal'
    })
    notification.show()
  })

  // อัพเดทชื่อ window สำหรับ timer
  ipcMain.on('timer:update', (_, { remaining, type }: { remaining: string; type: string }) => {
    const emoji = type === 'work' ? '🍅' : '☕'
    mainWindow?.setTitle(`${emoji} ${remaining}`)
  })
}

app.whenReady().then(createWindow)
app.on('window-all-closed', () => app.quit())
```

---

## 2. Tray Manager

```typescript
// src/main/trayManager.ts
import { Tray, Menu, BrowserWindow, app, nativeImage } from 'electron'
import { join } from 'path'

let tray: Tray | null = null
let timerState: { status: 'idle' | 'running' | 'paused'; remaining: string; type: string } = {
  status: 'idle', remaining: '25:00', type: 'work'
}

export function setupTray(win: BrowserWindow): void {
  const iconPath = join(__dirname, '../assets/tray-icon.png')
  tray = new Tray(nativeImage.createFromPath(iconPath).resize({ width: 16, height: 16 }))

  updateTrayMenu(win)
  
  tray.on('click', () => {
    win.isVisible() ? win.hide() : win.show()
  })

  // รับ state update จาก renderer
  const { ipcMain } = require('electron')
  ipcMain.on('tray:update-state', (_, state: typeof timerState) => {
    timerState = state
    updateTrayMenu(win)
    
    // อัพเดท tray title (macOS)
    if (process.platform === 'darwin' && timerState.status === 'running') {
      tray?.setTitle(`${timerState.remaining}`)
    } else {
      tray?.setTitle('')
    }
  })
}

function updateTrayMenu(win: BrowserWindow): void {
  const menu = Menu.buildFromTemplate([
    {
      label: timerState.status === 'running'
        ? `⏱ ${timerState.remaining} - กำลังทำงาน`
        : timerState.status === 'paused'
        ? `⏸ ${timerState.remaining} - หยุดชั่วคราว`
        : '🍅 Pomodoro Timer',
      enabled: false
    },
    { type: 'separator' },
    {
      label: timerState.status === 'running' ? 'หยุดชั่วคราว' : 'เริ่ม',
      click: () => win.webContents.send('tray:toggle')
    },
    {
      label: 'รีเซ็ต',
      click: () => win.webContents.send('tray:reset')
    },
    { type: 'separator' },
    {
      label: 'แสดงหน้าต่าง',
      click: () => { win.show(); win.focus() }
    },
    { type: 'separator' },
    { label: 'ออก', click: () => app.quit() }
  ])

  tray?.setContextMenu(menu)
  tray?.setToolTip(`Pomodoro Timer - ${timerState.remaining}`)
}
```

---

## 3. Pomodoro Store

```typescript
// src/renderer/store/pomodoroStore.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

export type SessionType = 'work' | 'short-break' | 'long-break'

export interface PomodoroSettings {
  workDuration: number      // นาที
  shortBreakDuration: number
  longBreakDuration: number
  sessionsUntilLongBreak: number
  autoStartBreaks: boolean
  autoStartWork: boolean
  soundEnabled: boolean
  notificationsEnabled: boolean
}

export interface Session {
  id: string
  type: SessionType
  duration: number
  completedAt: number
  date: string
}

interface PomodoroStore {
  settings: PomodoroSettings
  currentType: SessionType
  timeRemaining: number    // วินาที
  isRunning: boolean
  sessionsCompleted: number
  totalSessions: number
  sessions: Session[]

  startTimer: () => void
  pauseTimer: () => void
  resetTimer: () => void
  tick: () => void
  skipSession: () => void
  completeSession: () => void
  updateSettings: (settings: Partial<PomodoroSettings>) => void
  loadSessions: (sessions: Session[]) => void
}

const DEFAULT_SETTINGS: PomodoroSettings = {
  workDuration: 25,
  shortBreakDuration: 5,
  longBreakDuration: 15,
  sessionsUntilLongBreak: 4,
  autoStartBreaks: true,
  autoStartWork: false,
  soundEnabled: true,
  notificationsEnabled: true
}

function getDuration(type: SessionType, settings: PomodoroSettings): number {
  switch (type) {
    case 'work': return settings.workDuration * 60
    case 'short-break': return settings.shortBreakDuration * 60
    case 'long-break': return settings.longBreakDuration * 60
  }
}

export const usePomodoroStore = create<PomodoroStore>()(
  persist(
    immer((set, get) => ({
      settings: DEFAULT_SETTINGS,
      currentType: 'work',
      timeRemaining: DEFAULT_SETTINGS.workDuration * 60,
      isRunning: false,
      sessionsCompleted: 0,
      totalSessions: 0,
      sessions: [],

      startTimer: () => set(state => { state.isRunning = true }),
      pauseTimer: () => set(state => { state.isRunning = false }),

      resetTimer: () => set(state => {
        state.isRunning = false
        state.timeRemaining = getDuration(state.currentType, state.settings)
      }),

      tick: () => {
        const { timeRemaining, isRunning } = get()
        if (!isRunning) return
        
        if (timeRemaining <= 1) {
          get().completeSession()
        } else {
          set(state => { state.timeRemaining -= 1 })
        }
      },

      skipSession: () => {
        const { currentType, settings, sessionsCompleted } = get()
        let nextType: SessionType = 'work'
        
        if (currentType === 'work') {
          const nextCompleted = sessionsCompleted + 1
          nextType = nextCompleted % settings.sessionsUntilLongBreak === 0
            ? 'long-break'
            : 'short-break'
        }

        set(state => {
          state.isRunning = false
          state.currentType = nextType
          state.timeRemaining = getDuration(nextType, state.settings)
        })
      },

      completeSession: () => {
        const { currentType, settings, sessionsCompleted } = get()

        // บันทึก session
        const session: Session = {
          id: `${Date.now()}-${Math.random()}`,
          type: currentType,
          duration: getDuration(currentType, settings),
          completedAt: Date.now(),
          date: new Date().toISOString().split('T')[0]
        }

        // ส่งไป main process
        window.electronAPI.saveSession(session)

        // ส่ง notification
        if (settings.notificationsEnabled) {
          const isWork = currentType === 'work'
          window.electronAPI.sendNotification({
            title: isWork ? '🎉 Pomodoro สำเร็จ!' : '⏰ หมดเวลาพัก',
            body: isWork ? 'ได้เวลาพักแล้ว' : 'กลับมาทำงานกันเถอะ!'
          })
        }

        // กำหนด session ถัดไป
        let nextType: SessionType = 'work'
        let newSessionsCompleted = sessionsCompleted

        if (currentType === 'work') {
          newSessionsCompleted += 1
          nextType = newSessionsCompleted % settings.sessionsUntilLongBreak === 0
            ? 'long-break'
            : 'short-break'
        }

        set(state => {
          state.sessions.push(session)
          state.totalSessions += 1
          state.sessionsCompleted = newSessionsCompleted
          state.currentType = nextType
          state.timeRemaining = getDuration(nextType, state.settings)
          state.isRunning = (currentType === 'work' && state.settings.autoStartBreaks) ||
                           (currentType !== 'work' && state.settings.autoStartWork)
        })
      },

      updateSettings: (newSettings) => {
        set(state => {
          state.settings = { ...state.settings, ...newSettings }
          // รีเซ็ต timer ถ้าเปลี่ยน duration
          if (!state.isRunning) {
            state.timeRemaining = getDuration(state.currentType, state.settings)
          }
        })
      },

      loadSessions: (sessions) => set(state => { state.sessions = sessions })
    })),
    { name: 'pomodoro-settings', partialize: (state) => ({ settings: state.settings }) }
  )
)
```

---

## 4. Timer Component

```tsx
// src/renderer/components/Timer.tsx
import { useEffect, useRef, useCallback } from 'react'
import { usePomodoroStore, SessionType } from '../store/pomodoroStore'

const SESSION_COLORS: Record<SessionType, string> = {
  'work': '#e74c3c',
  'short-break': '#2ecc71',
  'long-break': '#3498db'
}

const SESSION_LABELS: Record<SessionType, string> = {
  'work': '🍅 โฟกัส',
  'short-break': '☕ พักสั้น',
  'long-break': '🌴 พักยาว'
}

export function Timer() {
  const { currentType, timeRemaining, isRunning, sessionsCompleted, settings,
          startTimer, pauseTimer, resetTimer, tick, skipSession } = usePomodoroStore()
  
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null)
  const totalDuration = settings[
    currentType === 'work' ? 'workDuration' :
    currentType === 'short-break' ? 'shortBreakDuration' : 'longBreakDuration'
  ] * 60

  // Timer tick
  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => tick(), 1000)
    } else {
      if (intervalRef.current) clearInterval(intervalRef.current)
    }
    return () => { if (intervalRef.current) clearInterval(intervalRef.current) }
  }, [isRunning, tick])

  // อัพเดท title bar และ tray
  useEffect(() => {
    const minutes = Math.floor(timeRemaining / 60).toString().padStart(2, '0')
    const seconds = (timeRemaining % 60).toString().padStart(2, '0')
    const formatted = `${minutes}:${seconds}`
    
    window.electronAPI.updateTimer({ remaining: formatted, type: currentType })
    window.electronAPI.updateTrayState({
      status: isRunning ? 'running' : timeRemaining < totalDuration ? 'paused' : 'idle',
      remaining: formatted,
      type: currentType
    })
  }, [timeRemaining, isRunning, currentType])

  const minutes = Math.floor(timeRemaining / 60).toString().padStart(2, '0')
  const seconds = (timeRemaining % 60).toString().padStart(2, '0')
  const progress = 1 - timeRemaining / totalDuration

  // SVG Circle progress
  const radius = 120
  const circumference = 2 * Math.PI * radius
  const strokeDashoffset = circumference * (1 - progress)
  const color = SESSION_COLORS[currentType]

  // Session dots (4 dots per cycle)
  const dots = Array.from({ length: settings.sessionsUntilLongBreak }, (_, i) => ({
    filled: i < (sessionsCompleted % settings.sessionsUntilLongBreak)
  }))

  return (
    <div className="timer-container">
      {/* Session Type Selector */}
      <div className="session-type-selector">
        {(['work', 'short-break', 'long-break'] as SessionType[]).map(type => (
          <button
            key={type}
            className={`type-btn ${currentType === type ? 'active' : ''}`}
            onClick={() => !isRunning && usePomodoroStore.getState().skipSession()}
            style={{ '--accent': SESSION_COLORS[type] } as React.CSSProperties}
          >
            {SESSION_LABELS[type]}
          </button>
        ))}
      </div>

      {/* SVG Circle Timer */}
      <div className="timer-circle-wrapper">
        <svg className="timer-svg" viewBox="0 0 300 300">
          {/* Background circle */}
          <circle
            cx="150" cy="150" r={radius}
            fill="none"
            stroke="rgba(255,255,255,0.1)"
            strokeWidth="8"
          />
          {/* Progress circle */}
          <circle
            cx="150" cy="150" r={radius}
            fill="none"
            stroke={color}
            strokeWidth="8"
            strokeLinecap="round"
            strokeDasharray={circumference}
            strokeDashoffset={strokeDashoffset}
            transform="rotate(-90 150 150)"
            style={{ transition: 'stroke-dashoffset 1s linear' }}
          />
        </svg>

        {/* Time Display */}
        <div className="timer-display">
          <div className="timer-time" style={{ color }}>
            {minutes}:{seconds}
          </div>
          <div className="timer-label">{SESSION_LABELS[currentType]}</div>
        </div>
      </div>

      {/* Session Progress Dots */}
      <div className="session-dots">
        {dots.map((dot, i) => (
          <div key={i} className={`session-dot ${dot.filled ? 'filled' : ''}`}
            style={{ '--color': color } as React.CSSProperties} />
        ))}
      </div>

      {/* Controls */}
      <div className="timer-controls">
        <button className="ctrl-btn secondary" onClick={resetTimer} title="รีเซ็ต">
          ↺
        </button>
        <button
          className="ctrl-btn primary"
          onClick={isRunning ? pauseTimer : startTimer}
          style={{ '--accent': color } as React.CSSProperties}
        >
          {isRunning ? '⏸' : '▶'}
        </button>
        <button className="ctrl-btn secondary" onClick={skipSession} title="ข้าม">
          ⏭
        </button>
      </div>
    </div>
  )
}
```

---

## 5. Contribution Graph (GitHub-style)

```tsx
// src/renderer/components/ContributionGraph.tsx
import { useMemo } from 'react'
import { Session } from '../store/pomodoroStore'

interface ContributionGraphProps {
  sessions: Session[]
  weeks?: number
}

export function ContributionGraph({ sessions, weeks = 16 }: ContributionGraphProps) {
  const graph = useMemo(() => {
    // สร้าง map ของจำนวน pomodoro ต่อวัน
    const workSessions = sessions.filter(s => s.type === 'work')
    const countByDate: Record<string, number> = {}

    workSessions.forEach(session => {
      const date = session.date
      countByDate[date] = (countByDate[date] || 0) + 1
    })

    // สร้าง grid 7 x weeks
    const today = new Date()
    const cells = []
    const totalDays = weeks * 7

    for (let i = totalDays - 1; i >= 0; i--) {
      const date = new Date(today)
      date.setDate(date.getDate() - i)
      const dateStr = date.toISOString().split('T')[0]
      const count = countByDate[dateStr] || 0
      
      cells.push({
        date: dateStr,
        count,
        level: count === 0 ? 0 : count < 2 ? 1 : count < 4 ? 2 : count < 6 ? 3 : 4,
        dayOfWeek: date.getDay()
      })
    }

    return cells
  }, [sessions, weeks])

  // จัดเรียงเป็น columns (7 rows ต่อ column)
  const columns = []
  for (let i = 0; i < graph.length; i += 7) {
    columns.push(graph.slice(i, i + 7))
  }

  const COLORS = ['#161b22', '#0e4429', '#006d32', '#26a641', '#39d353']
  const DAYS = ['อา', 'จ', 'อ', 'พ', 'พฤ', 'ศ', 'ส']
  const totalPomodoros = sessions.filter(s => s.type === 'work').length

  return (
    <div className="contribution-graph">
      <div className="graph-header">
        <h3>สถิติการทำงาน</h3>
        <span>{totalPomodoros} Pomodoro ใน {weeks} สัปดาห์</span>
      </div>

      <div className="graph-container">
        {/* Day labels */}
        <div className="day-labels">
          {DAYS.map((day, i) => (
            <span key={i} className="day-label" style={{ visibility: i % 2 === 0 ? 'visible' : 'hidden' }}>
              {day}
            </span>
          ))}
        </div>

        {/* Grid */}
        <div className="graph-grid">
          {columns.map((col, colIdx) => (
            <div key={colIdx} className="graph-column">
              {col.map(cell => (
                <div
                  key={cell.date}
                  className="graph-cell"
                  style={{ backgroundColor: COLORS[cell.level] }}
                  title={`${cell.date}: ${cell.count} Pomodoro${cell.count !== 1 ? 's' : ''}`}
                />
              ))}
            </div>
          ))}
        </div>
      </div>

      {/* Legend */}
      <div className="graph-legend">
        <span>น้อย</span>
        {COLORS.map((color, i) => (
          <div key={i} className="legend-cell" style={{ backgroundColor: color }} />
        ))}
        <span>มาก</span>
      </div>
    </div>
  )
}
```

---

## 6. Stats Component

```tsx
// src/renderer/components/Stats.tsx
import { useMemo } from 'react'
import { Session } from '../store/pomodoroStore'

interface StatsProps {
  sessions: Session[]
}

export function Stats({ sessions }: StatsProps) {
  const stats = useMemo(() => {
    const workSessions = sessions.filter(s => s.type === 'work')
    const today = new Date().toISOString().split('T')[0]
    const week = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString().split('T')[0]

    const todaySessions = workSessions.filter(s => s.date === today).length
    const weekSessions = workSessions.filter(s => s.date >= week).length
    const totalFocusMinutes = Math.round(
      workSessions.reduce((sum, s) => sum + s.duration, 0) / 60
    )

    const hoursByDay: Record<string, number> = {}
    workSessions.forEach(s => {
      const hour = new Date(s.completedAt).getHours()
      hoursByDay[hour] = (hoursByDay[hour] || 0) + 1
    })
    const peakHour = Object.entries(hoursByDay).sort(([,a],[,b]) => b-a)[0]

    return { todaySessions, weekSessions, totalFocusMinutes, peakHour }
  }, [sessions])

  const statCards = [
    { label: 'วันนี้', value: stats.todaySessions, unit: 'Pomodoro', emoji: '🍅' },
    { label: '7 วัน', value: stats.weekSessions, unit: 'Pomodoro', emoji: '📅' },
    { label: 'เวลาโฟกัสรวม', value: stats.totalFocusMinutes, unit: 'นาที', emoji: '⏱' },
    { label: 'ชั่วโมงสูงสุด', value: stats.peakHour ? `${stats.peakHour[0]}:00` : '-', unit: 'น.', emoji: '🕐' }
  ]

  return (
    <div className="stats-grid">
      {statCards.map(card => (
        <div key={card.label} className="stat-card">
          <div className="stat-emoji">{card.emoji}</div>
          <div className="stat-value">{card.value}</div>
          <div className="stat-unit">{card.unit}</div>
          <div className="stat-label">{card.label}</div>
        </div>
      ))}
    </div>
  )
}
```

---

## 7. CSS Styles

```css
/* src/renderer/styles/app.css */
:root {
  --bg: #1a1a2e;
  --surface: #16213e;
  --surface-2: #0f3460;
  --text: #e2e8f0;
  --text-dim: #94a3b8;
}

body { background: var(--bg); color: var(--text); font-family: 'Inter', sans-serif; }

/* Timer */
.timer-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 24px;
  gap: 20px;
}

.session-type-selector {
  display: flex;
  gap: 8px;
  background: var(--surface);
  padding: 4px;
  border-radius: 12px;
}

.type-btn {
  padding: 8px 16px;
  border: none;
  border-radius: 8px;
  background: transparent;
  color: var(--text-dim);
  cursor: pointer;
  font-size: 13px;
  transition: all 0.2s;
}
.type-btn.active { background: var(--accent, #e74c3c); color: white; }

.timer-circle-wrapper {
  position: relative;
  width: 280px;
  height: 280px;
}

.timer-svg { position: absolute; width: 100%; height: 100%; }

.timer-display {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.timer-time {
  font-size: 56px;
  font-weight: 300;
  font-variant-numeric: tabular-nums;
  letter-spacing: -2px;
}

.timer-controls { display: flex; gap: 16px; align-items: center; }

.ctrl-btn {
  border: none;
  border-radius: 50%;
  cursor: pointer;
  font-size: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.1s;
}

.ctrl-btn.primary {
  width: 72px;
  height: 72px;
  background: var(--accent, #e74c3c);
  color: white;
  font-size: 28px;
}

.ctrl-btn.secondary {
  width: 48px;
  height: 48px;
  background: var(--surface-2);
  color: var(--text);
}

.ctrl-btn:hover { transform: scale(1.05); }
.ctrl-btn:active { transform: scale(0.95); }

/* Session dots */
.session-dots { display: flex; gap: 8px; }
.session-dot {
  width: 10px; height: 10px;
  border-radius: 50%;
  background: var(--surface-2);
  transition: background 0.3s;
}
.session-dot.filled { background: var(--color, #e74c3c); }

/* Contribution Graph */
.graph-grid { display: flex; gap: 2px; }
.graph-column { display: flex; flex-direction: column; gap: 2px; }
.graph-cell {
  width: 12px; height: 12px;
  border-radius: 2px;
  cursor: pointer;
}
.graph-cell:hover { outline: 1px solid rgba(255,255,255,0.3); }

/* Stats */
.stats-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  padding: 16px;
}
.stat-card {
  background: var(--surface);
  border-radius: 12px;
  padding: 16px;
  text-align: center;
}
.stat-value { font-size: 28px; font-weight: 700; margin: 8px 0 4px; }
.stat-unit { font-size: 11px; color: var(--text-dim); }
.stat-label { font-size: 12px; color: var(--text-dim); margin-top: 4px; }
```

---

## 8. Preload

```typescript
// src/preload/index.ts
import { contextBridge, ipcRenderer } from 'electron'

contextBridge.exposeInMainWorld('electronAPI', {
  saveSession: (session: unknown) => ipcRenderer.invoke('session:save', session),
  getSessions: () => ipcRenderer.invoke('session:get-all'),
  sendNotification: (data: unknown) => ipcRenderer.invoke('notification:send', data),
  updateTimer: (data: unknown) => ipcRenderer.send('timer:update', data),
  updateTrayState: (data: unknown) => ipcRenderer.send('tray:update-state', data),
  
  onTrayToggle: (cb: () => void) => ipcRenderer.on('tray:toggle', cb),
  onTrayReset: (cb: () => void) => ipcRenderer.on('tray:reset', cb)
})
```

---

## สรุปฟีเจอร์

| ฟีเจอร์ | รายละเอียด |
|---------|------------|
| Timer countdown | Circle progress animation |
| Work/Break sessions | Work 25min, Short break 5min, Long break 15min |
| Custom intervals | ปรับได้ใน settings |
| System notifications | Electron Notification API |
| Tray integration | แสดงเวลาใน tray, menu controls |
| Statistics | วันนี้, 7 วัน, รวม |
| Contribution graph | GitHub-style heatmap |
| Session history | บันทึกทุก session ลง storage |
| Auto-start | ตั้งค่าให้เริ่มอัตโนมัติ |
| Keyboard shortcuts | Space=toggle, R=reset, S=skip |
