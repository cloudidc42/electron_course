# Part 70: Project - Video Player

## สร้าง Video Player ด้วย HTML5 Video + mpv Integration

ในบทนี้เราจะสร้าง Video Player ที่มี playlist, subtitles, keyboard controls, speed control, screenshot frame และ always-on-top mini player

---

## 1. Main Process

```typescript
// src/main/index.ts
import { app, BrowserWindow, ipcMain, dialog, globalShortcut, screen } from 'electron'
import { join } from 'path'
import { stat } from 'fs/promises'

let mainWindow: BrowserWindow | null = null
let miniPlayerWindow: BrowserWindow | null = null

function createMainWindow() {
  mainWindow = new BrowserWindow({
    width: 1280,
    height: 800,
    backgroundColor: '#000000',
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })

  if (process.env.ELECTRON_RENDERER_URL) {
    mainWindow.loadURL(process.env.ELECTRON_RENDERER_URL)
  } else {
    mainWindow.loadFile(join(__dirname, '../renderer/index.html'))
  }
}

function createMiniPlayer() {
  const display = screen.getPrimaryDisplay()

  miniPlayerWindow = new BrowserWindow({
    width: 400,
    height: 260,
    x: display.workAreaSize.width - 420,
    y: display.workAreaSize.height - 280,
    resizable: true,
    minimizable: false,
    maximizable: false,
    alwaysOnTop: true,
    frame: false,
    titleBarStyle: 'hiddenInset',
    transparent: true,
    backgroundColor: '#00000000',
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),
      nodeIntegration: false,
      contextIsolation: true
    }
  })

  miniPlayerWindow.loadFile(join(__dirname, '../renderer/mini.html'))
  return miniPlayerWindow
}

ipcMain.handle('player:open-files', async () => {
  const { canceled, filePaths } = await dialog.showOpenDialog({
    properties: ['openFile', 'multiSelections'],
    filters: [
      {
        name: 'Video',
        extensions: ['mp4', 'mkv', 'avi', 'mov', 'webm', 'm4v', 'flv', 'wmv', 'ts', 'mpg']
      }
    ]
  })
  if (canceled) return []

  const files = await Promise.all(filePaths.map(async (path) => {
    const s = await stat(path)
    return { path, size: s.size, name: path.split('/').pop() || path }
  }))

  return files
})

ipcMain.handle('player:open-folder', async () => {
  const { canceled, filePaths } = await dialog.showOpenDialog({
    properties: ['openDirectory']
  })
  if (canceled) return []

  const { readdir } = await import('fs/promises')
  const extensions = ['.mp4', '.mkv', '.avi', '.mov', '.webm', '.m4v']
  const files = await readdir(filePaths[0], { withFileTypes: true })
  return files
    .filter(f => f.isFile() && extensions.some(ext => f.name.toLowerCase().endsWith(ext)))
    .map(f => ({ path: join(filePaths[0], f.name), name: f.name }))
})

ipcMain.handle('player:open-subtitles', async () => {
  const { canceled, filePaths } = await dialog.showOpenDialog({
    filters: [{ name: 'Subtitles', extensions: ['srt', 'vtt', 'ass', 'ssa', 'sub'] }]
  })
  if (canceled) return null
  const { readFile } = await import('fs/promises')
  const content = await readFile(filePaths[0], 'utf-8')
  return { path: filePaths[0], content, name: filePaths[0].split('/').pop() }
})

ipcMain.handle('player:take-screenshot', async (_, dataUrl: string) => {
  const { writeFile, mkdir } = await import('fs/promises')
  const os = await import('os')
  const screenshotDir = join(os.homedir(), 'Pictures', 'VideoScreenshots')
  await mkdir(screenshotDir, { recursive: true })

  const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19)
  const filePath = join(screenshotDir, `screenshot_${timestamp}.png`)
  const base64 = dataUrl.replace(/^data:image\/\w+;base64,/, '')
  await writeFile(filePath, Buffer.from(base64, 'base64'))
  return filePath
})

ipcMain.handle('player:toggle-mini', async () => {
  if (miniPlayerWindow && !miniPlayerWindow.isDestroyed()) {
    const state = { isOpen: true }
    miniPlayerWindow.close()
    miniPlayerWindow = null
    return state
  } else {
    createMiniPlayer()
    return { isOpen: false }
  }
})

app.whenReady().then(createMainWindow)
```

---

## 2. Video Player Store

```typescript
// src/renderer/store/playerStore.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

export interface VideoFile {
  path: string
  name: string
  size?: number
}

export interface SubtitleTrack {
  path: string
  name: string
  content: string
  language?: string
}

interface PlayerStore {
  playlist: VideoFile[]
  currentIndex: number
  isPlaying: boolean
  currentTime: number
  duration: number
  volume: number
  isMuted: boolean
  playbackSpeed: number
  isFullscreen: boolean
  isMiniPlayer: boolean
  subtitles: SubtitleTrack[]
  activeSubtitleIndex: number
  subtitleDelay: number
  subtitleSize: number
  videoFit: 'contain' | 'cover'
  brightness: number
  contrast: number
  saturation: number
  recentFiles: VideoFile[]
  
  setPlaylist: (files: VideoFile[]) => void
  addToPlaylist: (files: VideoFile[]) => void
  playIndex: (index: number) => void
  nextTrack: () => void
  prevTrack: () => void
  setPlaying: (playing: boolean) => void
  setCurrentTime: (time: number) => void
  setDuration: (duration: number) => void
  setVolume: (vol: number) => void
  toggleMute: () => void
  setSpeed: (speed: number) => void
  setSubtitles: (tracks: SubtitleTrack[]) => void
  setActiveSubtitle: (index: number) => void
  setSubtitleDelay: (delay: number) => void
  addRecentFile: (file: VideoFile) => void
}

export const usePlayerStore = create<PlayerStore>()(
  persist(
    immer((set, get) => ({
      playlist: [],
      currentIndex: 0,
      isPlaying: false,
      currentTime: 0,
      duration: 0,
      volume: 1,
      isMuted: false,
      playbackSpeed: 1,
      isFullscreen: false,
      isMiniPlayer: false,
      subtitles: [],
      activeSubtitleIndex: -1,
      subtitleDelay: 0,
      subtitleSize: 20,
      videoFit: 'contain',
      brightness: 100,
      contrast: 100,
      saturation: 100,
      recentFiles: [],

      setPlaylist: (files) => set(s => { s.playlist = files; s.currentIndex = 0 }),
      addToPlaylist: (files) => set(s => { s.playlist.push(...files) }),
      
      playIndex: (index) => set(s => {
        if (index >= 0 && index < s.playlist.length) {
          s.currentIndex = index
          s.isPlaying = true
          s.currentTime = 0
          const file = s.playlist[index]
          if (!s.recentFiles.some(r => r.path === file.path)) {
            s.recentFiles.unshift(file)
            s.recentFiles = s.recentFiles.slice(0, 20)
          }
        }
      }),

      nextTrack: () => {
        const { currentIndex, playlist } = get()
        if (currentIndex < playlist.length - 1) {
          get().playIndex(currentIndex + 1)
        }
      },

      prevTrack: () => {
        const { currentIndex } = get()
        if (currentIndex > 0) get().playIndex(currentIndex - 1)
      },

      setPlaying: (playing) => set(s => { s.isPlaying = playing }),
      setCurrentTime: (time) => set(s => { s.currentTime = time }),
      setDuration: (duration) => set(s => { s.duration = duration }),
      setVolume: (vol) => set(s => { s.volume = vol }),
      toggleMute: () => set(s => { s.isMuted = !s.isMuted }),
      setSpeed: (speed) => set(s => { s.playbackSpeed = speed }),
      setSubtitles: (tracks) => set(s => { s.subtitles = tracks; s.activeSubtitleIndex = tracks.length > 0 ? 0 : -1 }),
      setActiveSubtitle: (index) => set(s => { s.activeSubtitleIndex = index }),
      setSubtitleDelay: (delay) => set(s => { s.subtitleDelay = delay }),
      addRecentFile: (file) => set(s => {
        s.recentFiles = [file, ...s.recentFiles.filter(r => r.path !== file.path)].slice(0, 20)
      })
    })),
    {
      name: 'video-player',
      partialize: (state) => ({
        volume: state.volume,
        playbackSpeed: state.playbackSpeed,
        subtitleSize: state.subtitleSize,
        recentFiles: state.recentFiles,
        videoFit: state.videoFit
      })
    }
  )
)
```

---

## 3. Video Player Component

```tsx
// src/renderer/components/VideoPlayer.tsx
import { useRef, useEffect, useCallback, useState } from 'react'
import { usePlayerStore } from '../store/playerStore'

interface SubtitleEntry { start: number; end: number; text: string }

function parseSRT(content: string): SubtitleEntry[] {
  const entries: SubtitleEntry[] = []
  const blocks = content.trim().split(/\n\n+/)

  for (const block of blocks) {
    const lines = block.split('\n')
    if (lines.length < 3) continue

    const timeMatch = lines[1].match(
      /(\d{2}):(\d{2}):(\d{2}),(\d{3}) --> (\d{2}):(\d{2}):(\d{2}),(\d{3})/
    )
    if (!timeMatch) continue

    const toSeconds = (h: string, m: string, s: string, ms: string) =>
      parseInt(h) * 3600 + parseInt(m) * 60 + parseInt(s) + parseInt(ms) / 1000

    entries.push({
      start: toSeconds(timeMatch[1], timeMatch[2], timeMatch[3], timeMatch[4]),
      end: toSeconds(timeMatch[5], timeMatch[6], timeMatch[7], timeMatch[8]),
      text: lines.slice(2).join('\n')
        .replace(/<[^>]+>/g, '')  // ลบ HTML tags
    })
  }

  return entries
}

function parseVTT(content: string): SubtitleEntry[] {
  const entries: SubtitleEntry[] = []
  const blocks = content.replace(/WEBVTT.*\n/, '').trim().split(/\n\n+/)

  for (const block of blocks) {
    const lines = block.split('\n').filter(l => !l.match(/^\d+$/))
    if (lines.length < 2) continue

    const timeMatch = lines[0].match(
      /(\d{2}):(\d{2}):(\d{2})\.(\d{3}) --> (\d{2}):(\d{2}):(\d{2})\.(\d{3})/
    )
    if (!timeMatch) continue

    const toSeconds = (h: string, m: string, s: string, ms: string) =>
      parseInt(h) * 3600 + parseInt(m) * 60 + parseInt(s) + parseInt(ms) / 1000

    entries.push({
      start: toSeconds(timeMatch[1], timeMatch[2], timeMatch[3], timeMatch[4]),
      end: toSeconds(timeMatch[5], timeMatch[6], timeMatch[7], timeMatch[8]),
      text: lines.slice(1).join('\n')
    })
  }

  return entries
}

export function VideoPlayer() {
  const videoRef = useRef<HTMLVideoElement>(null)
  const containerRef = useRef<HTMLDivElement>(null)
  const [controlsVisible, setControlsVisible] = useState(true)
  const hideControlsTimer = useRef<ReturnType<typeof setTimeout>>()
  const [currentSubtitle, setCurrentSubtitle] = useState<string>('')
  const subtitleEntriesRef = useRef<SubtitleEntry[]>([])
  const [isDragging, setIsDragging] = useState(false)

  const {
    playlist, currentIndex, isPlaying, volume, isMuted, playbackSpeed,
    subtitles, activeSubtitleIndex, subtitleDelay, subtitleSize,
    videoFit, brightness, contrast, saturation,
    setPlaying, setCurrentTime, setDuration, setVolume,
    nextTrack, prevTrack
  } = usePlayerStore()

  const currentFile = playlist[currentIndex]

  // โหลด video
  useEffect(() => {
    const video = videoRef.current
    if (!video || !currentFile) return

    video.src = `file://${currentFile.path}`
    video.load()
    if (isPlaying) video.play().catch(() => {})
  }, [currentIndex, currentFile?.path])

  // Sync state กับ video element
  useEffect(() => {
    const video = videoRef.current
    if (!video) return

    if (isPlaying) video.play().catch(() => {})
    else video.pause()
  }, [isPlaying])

  useEffect(() => {
    if (videoRef.current) {
      videoRef.current.volume = isMuted ? 0 : volume
    }
  }, [volume, isMuted])

  useEffect(() => {
    if (videoRef.current) videoRef.current.playbackRate = playbackSpeed
  }, [playbackSpeed])

  // Event listeners
  useEffect(() => {
    const video = videoRef.current
    if (!video) return

    const handlers = {
      timeupdate: () => {
        setCurrentTime(video.currentTime)
        updateSubtitle(video.currentTime)
      },
      loadedmetadata: () => setDuration(video.duration),
      ended: () => nextTrack(),
      play: () => setPlaying(true),
      pause: () => setPlaying(false)
    }

    Object.entries(handlers).forEach(([event, handler]) => {
      video.addEventListener(event, handler)
    })

    return () => {
      Object.entries(handlers).forEach(([event, handler]) => {
        video.removeEventListener(event, handler)
      })
    }
  }, [])

  // Load subtitles
  useEffect(() => {
    if (activeSubtitleIndex >= 0 && subtitles[activeSubtitleIndex]) {
      const track = subtitles[activeSubtitleIndex]
      const ext = track.name.split('.').pop()?.toLowerCase()

      if (ext === 'srt') {
        subtitleEntriesRef.current = parseSRT(track.content)
      } else if (ext === 'vtt') {
        subtitleEntriesRef.current = parseVTT(track.content)
      }
    } else {
      subtitleEntriesRef.current = []
      setCurrentSubtitle('')
    }
  }, [activeSubtitleIndex, subtitles])

  const updateSubtitle = (time: number) => {
    const adjustedTime = time - subtitleDelay
    const entry = subtitleEntriesRef.current.find(
      e => adjustedTime >= e.start && adjustedTime <= e.end
    )
    setCurrentSubtitle(entry?.text || '')
  }

  // Keyboard controls
  useEffect(() => {
    const handleKey = (e: KeyboardEvent) => {
      const video = videoRef.current
      if (!video) return

      switch (e.key) {
        case ' ':
        case 'k':
          e.preventDefault()
          setPlaying(!isPlaying)
          break
        case 'ArrowLeft':
          e.preventDefault()
          video.currentTime = Math.max(0, video.currentTime - (e.shiftKey ? 30 : 5))
          break
        case 'ArrowRight':
          e.preventDefault()
          video.currentTime = Math.min(video.duration, video.currentTime + (e.shiftKey ? 30 : 5))
          break
        case 'ArrowUp':
          e.preventDefault()
          setVolume(Math.min(1, volume + 0.1))
          break
        case 'ArrowDown':
          e.preventDefault()
          setVolume(Math.max(0, volume - 0.1))
          break
        case 'f':
          e.preventDefault()
          if (!document.fullscreenElement) containerRef.current?.requestFullscreen()
          else document.exitFullscreen()
          break
        case 'm':
          e.preventDefault()
          usePlayerStore.getState().toggleMute()
          break
        case 'n':
          nextTrack()
          break
        case 'p':
          prevTrack()
          break
        case ',':
          usePlayerStore.getState().setSpeed(Math.max(0.25, playbackSpeed - 0.25))
          break
        case '.':
          usePlayerStore.getState().setSpeed(Math.min(4, playbackSpeed + 0.25))
          break
      }
    }

    window.addEventListener('keydown', handleKey)
    return () => window.removeEventListener('keydown', handleKey)
  }, [isPlaying, volume, playbackSpeed])

  // Auto-hide controls
  const showControls = () => {
    setControlsVisible(true)
    clearTimeout(hideControlsTimer.current)
    if (isPlaying) {
      hideControlsTimer.current = setTimeout(() => setControlsVisible(false), 3000)
    }
  }

  const handleScreenshot = async () => {
    const video = videoRef.current
    if (!video) return

    const canvas = document.createElement('canvas')
    canvas.width = video.videoWidth
    canvas.height = video.videoHeight
    canvas.getContext('2d')?.drawImage(video, 0, 0)
    const dataUrl = canvas.toDataURL('image/png')
    const path = await window.electronAPI.takeScreenshot(dataUrl)
    alert(`บันทึกภาพ: ${path}`)
  }

  const videoStyle = {
    filter: `brightness(${brightness}%) contrast(${contrast}%) saturate(${saturation}%)`,
    objectFit: videoFit
  }

  return (
    <div
      ref={containerRef}
      className="video-container"
      onMouseMove={showControls}
      onClick={() => setPlaying(!isPlaying)}
      style={{ cursor: controlsVisible ? 'default' : 'none' }}
    >
      <video
        ref={videoRef}
        style={videoStyle as React.CSSProperties}
        className="video-element"
        onClick={e => e.stopPropagation()}
      />

      {/* Subtitles */}
      {currentSubtitle && (
        <div
          className="subtitle-display"
          style={{ fontSize: subtitleSize }}
          dangerouslySetInnerHTML={{ __html: currentSubtitle.replace(/\n/g, '<br>') }}
        />
      )}

      {/* Controls overlay */}
      {controlsVisible && <VideoControls onScreenshot={handleScreenshot} />}
    </div>
  )
}
```

---

## 4. Video Controls Component

```tsx
// src/renderer/components/VideoControls.tsx
import { useRef } from 'react'
import { usePlayerStore } from '../store/playerStore'

function formatTime(s: number): string {
  if (!s || isNaN(s)) return '0:00'
  const h = Math.floor(s / 3600)
  const m = Math.floor((s % 3600) / 60)
  const sec = Math.floor(s % 60)
  return h > 0 ? `${h}:${m.toString().padStart(2,'0')}:${sec.toString().padStart(2,'0')}`
               : `${m}:${sec.toString().padStart(2,'0')}`
}

const SPEEDS = [0.25, 0.5, 0.75, 1, 1.25, 1.5, 1.75, 2, 3, 4]

interface VideoControlsProps {
  onScreenshot: () => void
}

export function VideoControls({ onScreenshot }: VideoControlsProps) {
  const {
    isPlaying, currentTime, duration, volume, isMuted, playbackSpeed,
    subtitles, activeSubtitleIndex, playlist, currentIndex,
    setPlaying, setCurrentTime: seek, setVolume, toggleMute, setSpeed,
    setActiveSubtitle, nextTrack, prevTrack
  } = usePlayerStore()

  const progressPercent = duration > 0 ? (currentTime / duration) * 100 : 0

  return (
    <div className="video-controls" onClick={e => e.stopPropagation()}>
      {/* Progress bar */}
      <div className="progress-bar-container">
        <input
          type="range"
          min={0} max={duration || 0} step={0.1}
          value={currentTime}
          onChange={e => {
            const video = document.querySelector('video')
            if (video) video.currentTime = parseFloat(e.target.value)
            seek(parseFloat(e.target.value))
          }}
          className="progress-bar"
          style={{ '--progress': `${progressPercent}%` } as React.CSSProperties}
        />
      </div>

      {/* Controls row */}
      <div className="controls-row">
        <div className="controls-left">
          <button onClick={prevTrack} title="ก่อนหน้า (P)">⏮</button>
          <button onClick={() => setPlaying(!isPlaying)} title="เล่น/หยุด (Space)">
            {isPlaying ? '⏸' : '▶'}
          </button>
          <button onClick={nextTrack} title="ถัดไป (N)">⏭</button>

          <div className="volume-control">
            <button onClick={toggleMute} title="ปิดเสียง (M)">
              {isMuted ? '🔇' : volume > 0.5 ? '🔊' : '🔉'}
            </button>
            <input
              type="range" min={0} max={1} step={0.05}
              value={isMuted ? 0 : volume}
              onChange={e => setVolume(parseFloat(e.target.value))}
              className="volume-slider"
            />
          </div>

          <span className="time-display">
            {formatTime(currentTime)} / {formatTime(duration)}
          </span>
        </div>

        <div className="controls-right">
          {/* Speed */}
          <select
            value={playbackSpeed}
            onChange={e => setSpeed(parseFloat(e.target.value))}
            className="speed-select"
            title="ความเร็ว (, และ .)"
          >
            {SPEEDS.map(s => (
              <option key={s} value={s}>{s}x</option>
            ))}
          </select>

          {/* Subtitles */}
          <select
            value={activeSubtitleIndex}
            onChange={e => setActiveSubtitle(parseInt(e.target.value))}
            className="subtitle-select"
            title="คำบรรยาย"
          >
            <option value={-1}>ปิด</option>
            {subtitles.map((s, i) => (
              <option key={i} value={i}>{s.name}</option>
            ))}
          </select>

          <button onClick={onScreenshot} title="ถ่ายภาพ">📷</button>

          <button
            onClick={() => {
              if (!document.fullscreenElement) document.documentElement.requestFullscreen()
              else document.exitFullscreen()
            }}
            title="เต็มหน้าจอ (F)"
          >⛶</button>

          <button
            onClick={() => window.electronAPI.toggleMiniPlayer()}
            title="Mini Player"
          >⊟</button>
        </div>
      </div>
    </div>
  )
}
```

---

## 5. Playlist Component

```tsx
// src/renderer/components/Playlist.tsx
import { usePlayerStore } from '../store/playerStore'

export function Playlist() {
  const { playlist, currentIndex, playIndex, addToPlaylist } = usePlayerStore()

  const handleDrop = async (e: React.DragEvent) => {
    e.preventDefault()
    const paths = Array.from(e.dataTransfer.files)
      .filter(f => /\.(mp4|mkv|avi|mov|webm|m4v)$/i.test(f.name))
      .map(f => ({ path: f.path, name: f.name }))
    addToPlaylist(paths)
  }

  return (
    <div
      className="playlist"
      onDragOver={e => e.preventDefault()}
      onDrop={handleDrop}
    >
      <div className="playlist-header">
        <h3>เพลย์ลิสต์ ({playlist.length})</h3>
        <div className="playlist-actions">
          <button onClick={async () => {
            const files = await window.electronAPI.openFiles()
            if (files.length) addToPlaylist(files)
          }}>+ เพิ่ม</button>
        </div>
      </div>

      {playlist.length === 0 ? (
        <div className="playlist-empty">
          <p>ลากไฟล์วิดีโอมาวางที่นี่</p>
        </div>
      ) : (
        <div className="playlist-items">
          {playlist.map((item, i) => (
            <div
              key={`${item.path}-${i}`}
              className={`playlist-item ${i === currentIndex ? 'active' : ''}`}
              onClick={() => playIndex(i)}
              title={item.path}
            >
              <span className="item-index">{i + 1}</span>
              <span className="item-name">{item.name}</span>
              {i === currentIndex && <span className="playing-icon">♪</span>}
            </div>
          ))}
        </div>
      )}
    </div>
  )
}
```

---

## 6. CSS

```css
/* styles/app.css */
:root { --bg: #000; --controls-bg: rgba(0,0,0,0.8); }

.video-container {
  position: relative;
  flex: 1;
  background: #000;
  overflow: hidden;
}

.video-element {
  width: 100%;
  height: 100%;
  display: block;
}

.subtitle-display {
  position: absolute;
  bottom: 80px;
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
  color: white;
  text-shadow: 2px 2px 4px black, -2px -2px 4px black;
  max-width: 80%;
  pointer-events: none;
  line-height: 1.5;
  padding: 4px 8px;
  background: rgba(0,0,0,0.3);
  border-radius: 4px;
}

.video-controls {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 8px 16px 12px;
  background: linear-gradient(transparent, rgba(0,0,0,0.9));
  transition: opacity 0.3s;
}

.progress-bar {
  width: 100%;
  appearance: none;
  height: 4px;
  border-radius: 2px;
  background: linear-gradient(to right, #e74c3c var(--progress), rgba(255,255,255,0.3) var(--progress));
  cursor: pointer;
  margin-bottom: 8px;
}

.controls-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  color: white;
}

.controls-left, .controls-right {
  display: flex;
  align-items: center;
  gap: 12px;
}

.controls-left button, .controls-right button {
  background: none;
  border: none;
  color: white;
  font-size: 18px;
  cursor: pointer;
  padding: 4px;
  opacity: 0.9;
}

.controls-left button:hover, .controls-right button:hover { opacity: 1; }

.speed-select, .subtitle-select {
  background: rgba(255,255,255,0.1);
  border: none;
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 13px;
  cursor: pointer;
}
```

---

## สรุปฟีเจอร์

| ฟีเจอร์ | รายละเอียด |
|---------|------------|
| Video playback | HTML5 Video (mp4, mkv, webm ฯลฯ) |
| Playlist | เพิ่ม/เล่น/เรียง |
| Subtitles | SRT, VTT พร้อม delay adjust |
| Speed control | 0.25x ถึง 4x |
| Screenshot | บันทึก frame เป็น PNG |
| Keyboard shortcuts | Space/Arrow keys/F/M |
| Mini player | Always-on-top compact mode |
| Drag & Drop | ลากไฟล์เข้า playlist |
| Recent files | เก็บประวัติ 20 รายการ |
| Video filters | Brightness/Contrast/Saturation |
