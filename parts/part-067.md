# Part 67: Project - System Monitor

## สร้าง System Monitor ด้วย Chart.js

ในบทนี้เราจะสร้าง System Monitor ที่แสดง CPU, RAM, Disk, Network แบบ real-time พร้อม process list และ live charts

---

## 1. System Info Service (Main Process)

```typescript
// src/main/systemInfo.ts
import { ipcMain } from 'electron'
import os from 'os'
import { execSync } from 'child_process'
import { statfs } from 'fs'
import { promisify } from 'util'

const statfsAsync = promisify(statfs)

export interface CpuInfo {
  usage: number        // 0-100
  cores: number
  model: string
  speed: number        // MHz
  perCore: number[]
}

export interface MemoryInfo {
  total: number        // bytes
  used: number
  free: number
  usagePercent: number
  swap?: { total: number; used: number }
}

export interface DiskInfo {
  mountpoint: string
  total: number
  used: number
  free: number
  usagePercent: number
  filesystem: string
}

export interface NetworkInfo {
  interface: string
  rxBytes: number      // cumulative
  txBytes: number
  rxSpeed: number      // bytes/sec
  txSpeed: number
}

export interface ProcessInfo {
  pid: number
  name: string
  cpu: number          // %
  memory: number       // bytes
  status: string
}

// CPU Usage tracker
let lastCpuTimes = os.cpus().map(cpu => ({ ...cpu.times }))

function getCpuUsage(): { overall: number; perCore: number[] } {
  const cpus = os.cpus()
  const perCore: number[] = []

  let totalDiff = 0, idleDiff = 0

  cpus.forEach((cpu, i) => {
    const prev = lastCpuTimes[i]
    const curr = cpu.times

    const prevTotal = Object.values(prev).reduce((a, b) => a + b, 0)
    const currTotal = Object.values(curr).reduce((a, b) => a + b, 0)

    const totalDiffCore = currTotal - prevTotal
    const idleDiffCore = curr.idle - prev.idle

    const usage = totalDiffCore > 0 ? (1 - idleDiffCore / totalDiffCore) * 100 : 0
    perCore.push(Math.max(0, Math.min(100, usage)))
    totalDiff += totalDiffCore
    idleDiff += idleDiffCore
  })

  lastCpuTimes = cpus.map(cpu => ({ ...cpu.times }))

  const overall = totalDiff > 0 ? (1 - idleDiff / totalDiff) * 100 : 0
  return { overall: Math.max(0, Math.min(100, overall)), perCore }
}

// Network speed tracker
let lastNetworkStats: Record<string, { rx: number; tx: number; time: number }> = {}

async function getNetworkInfo(): Promise<NetworkInfo[]> {
  const interfaces = os.networkInterfaces()
  const results: NetworkInfo[] = []

  // อ่าน /proc/net/dev บน Linux
  if (process.platform === 'linux') {
    try {
      const data = require('fs').readFileSync('/proc/net/dev', 'utf-8')
      const lines = data.split('\n').slice(2)

      for (const line of lines) {
        const parts = line.trim().split(/\s+/)
        if (parts.length < 10) continue
        const iface = parts[0].replace(':', '')
        if (iface === 'lo') continue

        const rx = parseInt(parts[1])
        const tx = parseInt(parts[9])
        const now = Date.now()

        if (lastNetworkStats[iface]) {
          const elapsed = (now - lastNetworkStats[iface].time) / 1000
          const rxSpeed = (rx - lastNetworkStats[iface].rx) / elapsed
          const txSpeed = (tx - lastNetworkStats[iface].tx) / elapsed

          results.push({ interface: iface, rxBytes: rx, txBytes: tx, rxSpeed: Math.max(0, rxSpeed), txSpeed: Math.max(0, txSpeed) })
        }

        lastNetworkStats[iface] = { rx, tx, time: now }
      }
    } catch {}
  }

  return results
}

async function getDiskInfo(): Promise<DiskInfo[]> {
  const disks: DiskInfo[] = []
  const mountpoints = process.platform === 'win32' ? ['C:\\'] : ['/']

  for (const mp of mountpoints) {
    try {
      const stats = await statfsAsync(mp)
      const total = stats.blocks * stats.bsize
      const free = stats.bfree * stats.bsize
      const used = total - free

      disks.push({
        mountpoint: mp,
        total,
        used,
        free,
        usagePercent: (used / total) * 100,
        filesystem: 'unknown'
      })
    } catch {}
  }

  return disks
}

function getProcessList(): ProcessInfo[] {
  const processes: ProcessInfo[] = []

  try {
    if (process.platform === 'linux') {
      const result = execSync('ps aux --sort=-%cpu | head -20', { encoding: 'utf-8' })
      const lines = result.split('\n').slice(1)

      for (const line of lines) {
        const parts = line.trim().split(/\s+/)
        if (parts.length < 11) continue
        processes.push({
          pid: parseInt(parts[1]),
          name: parts[10],
          cpu: parseFloat(parts[2]),
          memory: parseInt(parts[5]) * 1024,
          status: parts[7]
        })
      }
    }
  } catch {}

  return processes.slice(0, 20)
}

export function registerSystemHandlers(): void {
  ipcMain.handle('system:get-info', async () => {
    const cpuData = getCpuUsage()
    const totalMem = os.totalmem()
    const freeMem = os.freemem()

    return {
      cpu: {
        usage: cpuData.overall,
        cores: os.cpus().length,
        model: os.cpus()[0]?.model || 'Unknown',
        speed: os.cpus()[0]?.speed || 0,
        perCore: cpuData.perCore
      } as CpuInfo,
      memory: {
        total: totalMem,
        used: totalMem - freeMem,
        free: freeMem,
        usagePercent: ((totalMem - freeMem) / totalMem) * 100
      } as MemoryInfo,
      disk: await getDiskInfo(),
      network: await getNetworkInfo(),
      uptime: os.uptime(),
      platform: process.platform,
      hostname: os.hostname()
    }
  })

  ipcMain.handle('system:get-processes', () => getProcessList())

  ipcMain.handle('system:kill-process', async (_, pid: number) => {
    try {
      process.kill(pid, 'SIGTERM')
      return { success: true }
    } catch (error) {
      return { success: false, error: (error as Error).message }
    }
  })
}
```

---

## 2. Real-time Charts Component

```tsx
// src/renderer/components/ResourceChart.tsx
import { useRef, useEffect } from 'react'
import Chart from 'chart.js/auto'

interface ResourceChartProps {
  data: number[]
  label: string
  color: string
  maxValue?: number
  unit?: string
}

const MAX_DATA_POINTS = 60

export function ResourceChart({ data, label, color, maxValue = 100, unit = '%' }: ResourceChartProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const chartRef = useRef<Chart | null>(null)

  useEffect(() => {
    if (!canvasRef.current) return

    const labels = Array.from({ length: MAX_DATA_POINTS }, (_, i) => `${MAX_DATA_POINTS - i}s`)

    chartRef.current = new Chart(canvasRef.current, {
      type: 'line',
      data: {
        labels,
        datasets: [{
          label,
          data: new Array(MAX_DATA_POINTS).fill(0),
          borderColor: color,
          backgroundColor: color + '20',
          fill: true,
          tension: 0.4,
          pointRadius: 0,
          borderWidth: 2
        }]
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        animation: { duration: 0 },
        scales: {
          x: { display: false },
          y: {
            min: 0,
            max: maxValue,
            ticks: {
              color: '#888',
              callback: (v) => `${v}${unit}`
            },
            grid: { color: 'rgba(255,255,255,0.05)' }
          }
        },
        plugins: {
          legend: { display: false },
          tooltip: {
            callbacks: {
              label: (ctx) => `${ctx.parsed.y.toFixed(1)}${unit}`
            }
          }
        }
      }
    })

    return () => { chartRef.current?.destroy() }
  }, [])

  useEffect(() => {
    if (!chartRef.current) return
    const chart = chartRef.current
    const dataset = chart.data.datasets[0]

    if (!Array.isArray(dataset.data)) return

    // เพิ่มข้อมูลใหม่และลบเก่า
    const currentData = dataset.data as number[]
    const latestValue = data[data.length - 1] ?? 0
    currentData.push(latestValue)
    if (currentData.length > MAX_DATA_POINTS) currentData.shift()

    chart.update('none')
  }, [data])

  const currentValue = data[data.length - 1] ?? 0

  return (
    <div className="resource-chart">
      <div className="chart-header">
        <span className="chart-label">{label}</span>
        <span className="chart-value" style={{ color }}>
          {currentValue.toFixed(1)}{unit}
        </span>
      </div>
      <div className="chart-wrapper">
        <canvas ref={canvasRef} />
      </div>
      <div className="chart-bar">
        <div
          className="chart-fill"
          style={{ width: `${(currentValue / maxValue) * 100}%`, background: color }}
        />
      </div>
    </div>
  )
}
```

---

## 3. Process List Component

```tsx
// src/renderer/components/ProcessList.tsx
import { useState } from 'react'
import { ProcessInfo } from '../../main/systemInfo'

function formatMemory(bytes: number): string {
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(0)} KB`
  return `${(bytes / 1024 / 1024).toFixed(1)} MB`
}

interface ProcessListProps {
  processes: ProcessInfo[]
  onKill: (pid: number) => void
}

export function ProcessList({ processes, onKill }: ProcessListProps) {
  const [sortBy, setSortBy] = useState<'cpu' | 'memory' | 'name'>('cpu')
  const [sortAsc, setSortAsc] = useState(false)
  const [filter, setFilter] = useState('')

  const sorted = [...processes]
    .filter(p => !filter || p.name.toLowerCase().includes(filter.toLowerCase()))
    .sort((a, b) => {
      const cmp = sortBy === 'name' ? a.name.localeCompare(b.name) :
                  sortBy === 'cpu' ? a.cpu - b.cpu : a.memory - b.memory
      return sortAsc ? cmp : -cmp
    })

  const handleSort = (key: typeof sortBy) => {
    if (sortBy === key) setSortAsc(a => !a)
    else { setSortBy(key); setSortAsc(false) }
  }

  const handleKill = (pid: number, name: string) => {
    if (confirm(`ต้องการสิ้นสุดกระบวนการ "${name}" (PID: ${pid})?`)) {
      onKill(pid)
    }
  }

  return (
    <div className="process-list">
      <div className="process-header">
        <h3>กระบวนการ ({processes.length})</h3>
        <input
          type="text"
          placeholder="กรอง..."
          value={filter}
          onChange={e => setFilter(e.target.value)}
          className="process-filter"
        />
      </div>

      <table className="process-table">
        <thead>
          <tr>
            <th onClick={() => handleSort('name')} className="sortable">
              ชื่อ {sortBy === 'name' ? (sortAsc ? '↑' : '↓') : ''}
            </th>
            <th>PID</th>
            <th onClick={() => handleSort('cpu')} className="sortable">
              CPU% {sortBy === 'cpu' ? (sortAsc ? '↑' : '↓') : ''}
            </th>
            <th onClick={() => handleSort('memory')} className="sortable">
              หน่วยความจำ {sortBy === 'memory' ? (sortAsc ? '↑' : '↓') : ''}
            </th>
            <th>การดำเนินการ</th>
          </tr>
        </thead>
        <tbody>
          {sorted.map(proc => (
            <tr key={proc.pid} className={proc.cpu > 50 ? 'high-cpu' : ''}>
              <td className="process-name">{proc.name}</td>
              <td className="process-pid">{proc.pid}</td>
              <td className="process-cpu">
                <div className="usage-bar-wrapper">
                  <div className="usage-bar cpu" style={{ width: `${proc.cpu}%` }} />
                  <span>{proc.cpu.toFixed(1)}%</span>
                </div>
              </td>
              <td className="process-mem">{formatMemory(proc.memory)}</td>
              <td>
                <button
                  className="kill-btn"
                  onClick={() => handleKill(proc.pid, proc.name)}
                  title="สิ้นสุดกระบวนการ"
                >
                  ✕
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}
```

---

## 4. App Component หลัก

```tsx
// src/renderer/App.tsx
import { useState, useEffect, useCallback } from 'react'
import { ResourceChart } from './components/ResourceChart'
import { ProcessList } from './components/ProcessList'
import './styles/app.css'

const HISTORY_SIZE = 60

function formatBytes(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 ** 2) return `${(bytes / 1024).toFixed(1)} KB`
  if (bytes < 1024 ** 3) return `${(bytes / 1024 ** 2).toFixed(1)} MB`
  return `${(bytes / 1024 ** 3).toFixed(2)} GB`
}

export default function App() {
  const [cpuHistory, setCpuHistory] = useState<number[]>([])
  const [ramHistory, setRamHistory] = useState<number[]>([])
  const [netRxHistory, setNetRxHistory] = useState<number[]>([])
  const [netTxHistory, setNetTxHistory] = useState<number[]>([])
  const [currentInfo, setCurrentInfo] = useState<Record<string, unknown> | null>(null)
  const [processes, setProcesses] = useState<unknown[]>([])
  const [activeTab, setActiveTab] = useState<'overview' | 'processes'>('overview')

  const appendHistory = (setter: React.Dispatch<React.SetStateAction<number[]>>, value: number) => {
    setter(prev => {
      const next = [...prev, value]
      if (next.length > HISTORY_SIZE) next.shift()
      return next
    })
  }

  const fetchInfo = useCallback(async () => {
    const info = await window.electronAPI.getSystemInfo()
    setCurrentInfo(info)
    appendHistory(setCpuHistory, info.cpu.usage)
    appendHistory(setRamHistory, info.memory.usagePercent)
    
    if (info.network.length > 0) {
      appendHistory(setNetRxHistory, info.network[0].rxSpeed / 1024) // KB/s
      appendHistory(setNetTxHistory, info.network[0].txSpeed / 1024)
    }
  }, [])

  const fetchProcesses = useCallback(async () => {
    const procs = await window.electronAPI.getProcesses()
    setProcesses(procs)
  }, [])

  useEffect(() => {
    fetchInfo()
    const infoInterval = setInterval(fetchInfo, 1000)
    const procInterval = setInterval(fetchProcesses, 3000)
    fetchProcesses()
    return () => { clearInterval(infoInterval); clearInterval(procInterval) }
  }, [fetchInfo, fetchProcesses])

  if (!currentInfo) return <div className="loading">กำลังโหลด...</div>

  const info = currentInfo as any
  const maxRam = info.memory.total / 1024 / 1024 // MB

  return (
    <div className="app">
      <div className="app-header">
        <h1>📊 System Monitor</h1>
        <div className="header-meta">
          <span>{info.hostname}</span>
          <span>Uptime: {Math.floor(info.uptime / 3600)}h {Math.floor((info.uptime % 3600) / 60)}m</span>
        </div>
        <div className="tabs">
          <button className={activeTab === 'overview' ? 'active' : ''} onClick={() => setActiveTab('overview')}>ภาพรวม</button>
          <button className={activeTab === 'processes' ? 'active' : ''} onClick={() => setActiveTab('processes')}>กระบวนการ</button>
        </div>
      </div>

      {activeTab === 'overview' && (
        <div className="charts-grid">
          <div className="chart-section">
            <div className="section-info">
              <h3>CPU</h3>
              <p>{info.cpu.model}</p>
              <p>{info.cpu.cores} cores @ {info.cpu.speed} MHz</p>
            </div>
            <ResourceChart
              data={cpuHistory} label="CPU" color="#e74c3c" maxValue={100} unit="%" />
            
            {/* Per-core */}
            <div className="core-grid">
              {(info.cpu.perCore as number[]).map((usage: number, i: number) => (
                <div key={i} className="core-item">
                  <span>Core {i}</span>
                  <div className="core-bar">
                    <div className="core-fill" style={{ width: `${usage}%`, background: `hsl(${120 - usage * 1.2}, 70%, 50%)` }} />
                  </div>
                  <span>{usage.toFixed(0)}%</span>
                </div>
              ))}
            </div>
          </div>

          <div className="chart-section">
            <div className="section-info">
              <h3>หน่วยความจำ (RAM)</h3>
              <p>{formatBytes(info.memory.used)} / {formatBytes(info.memory.total)}</p>
            </div>
            <ResourceChart
              data={ramHistory} label="RAM" color="#3498db"
              maxValue={100} unit="%" />
          </div>

          <div className="chart-section">
            <h3>Disk</h3>
            {(info.disk as any[]).map((disk: any) => (
              <div key={disk.mountpoint} className="disk-item">
                <div className="disk-info">
                  <span>{disk.mountpoint}</span>
                  <span>{formatBytes(disk.used)} / {formatBytes(disk.total)}</span>
                </div>
                <div className="disk-bar">
                  <div
                    className="disk-fill"
                    style={{
                      width: `${disk.usagePercent}%`,
                      background: disk.usagePercent > 90 ? '#e74c3c' : disk.usagePercent > 70 ? '#f39c12' : '#2ecc71'
                    }}
                  />
                </div>
                <span>{disk.usagePercent.toFixed(1)}%</span>
              </div>
            ))}
          </div>

          {netRxHistory.length > 0 && (
            <div className="chart-section">
              <h3>เครือข่าย</h3>
              <ResourceChart data={netRxHistory} label="Download" color="#2ecc71" maxValue={1024} unit=" KB/s" />
              <ResourceChart data={netTxHistory} label="Upload" color="#9b59b6" maxValue={1024} unit=" KB/s" />
            </div>
          )}
        </div>
      )}

      {activeTab === 'processes' && (
        <ProcessList
          processes={processes as any[]}
          onKill={async (pid) => {
            const result = await window.electronAPI.killProcess(pid)
            if (!result.success) alert(`ไม่สามารถสิ้นสุดกระบวนการได้: ${result.error}`)
          }}
        />
      )}
    </div>
  )
}
```

---

## 5. CSS

```css
/* styles/app.css */
:root {
  --bg: #0f172a;
  --surface: #1e293b;
  --border: #334155;
  --text: #e2e8f0;
  --text-dim: #94a3b8;
}

.charts-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(380px, 1fr));
  gap: 16px;
  padding: 16px;
}

.chart-section {
  background: var(--surface);
  border-radius: 12px;
  padding: 16px;
  border: 1px solid var(--border);
}

.chart-wrapper { height: 120px; position: relative; margin: 8px 0; }

.chart-bar {
  height: 6px;
  background: var(--border);
  border-radius: 3px;
  overflow: hidden;
}

.chart-fill {
  height: 100%;
  border-radius: 3px;
  transition: width 0.5s ease;
}

.core-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 6px;
  margin-top: 8px;
}

.core-item {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
}

.core-bar { flex: 1; height: 6px; background: var(--border); border-radius: 3px; overflow: hidden; }
.core-fill { height: 100%; border-radius: 3px; transition: width 0.5s; }

/* Process Table */
.process-table { width: 100%; border-collapse: collapse; }
.process-table th, .process-table td { padding: 8px 12px; text-align: left; border-bottom: 1px solid var(--border); }
.process-table th { background: var(--surface); cursor: pointer; font-size: 12px; color: var(--text-dim); }
.process-table th:hover { color: var(--text); }
.high-cpu { background: rgba(231, 76, 60, 0.05); }

.kill-btn {
  background: none; border: 1px solid #e74c3c; color: #e74c3c;
  padding: 2px 8px; border-radius: 4px; cursor: pointer; font-size: 12px;
}
.kill-btn:hover { background: #e74c3c; color: white; }
```

---

## สรุปฟีเจอร์

| ฟีเจอร์ | รายละเอียด |
|---------|------------|
| CPU monitoring | Overall + per-core usage |
| RAM monitoring | Used/Total + history chart |
| Disk monitoring | Usage per partition |
| Network monitoring | Download/Upload speed |
| Process list | Top 20 by CPU |
| Kill process | SIGTERM |
| Live charts | Chart.js 60-second history |
| Auto refresh | ทุก 1 วินาที |
