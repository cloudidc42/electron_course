# Part 68: Project - Music Player

## สร้าง Music Player ด้วย Web Audio API

ในบทนี้เราจะสร้าง Music Player ที่มี waveform visualization, equalizer, lyrics display และ mini player mode

---

## 1. Main Process - File & Metadata Handler

```typescript
// src/main/musicHandlers.ts
import { ipcMain, dialog, app } from 'electron'
import { readdir, stat, readFile } from 'fs/promises'
import { join, extname, basename } from 'path'
import * as mm from 'music-metadata'

export interface TrackMetadata {
  filePath: string
  title: string
  artist: string
  album: string
  year?: number
  genre?: string[]
  duration: number
  trackNumber?: number
  coverArt?: string    // base64 data URL
  bitrate?: number
  sampleRate?: number
}

const AUDIO_EXTENSIONS = ['.mp3', '.flac', '.ogg', '.wav', '.m4a', '.aac', '.opus']

export function registerMusicHandlers(): void {
  // เปิดไฟล์เพลง
  ipcMain.handle('music:open-files', async () => {
    const { canceled, filePaths } = await dialog.showOpenDialog({
      properties: ['openFile', 'multiSelections'],
      filters: [{ name: 'Audio', extensions: ['mp3', 'flac', 'ogg', 'wav', 'm4a', 'aac'] }]
    })
    if (canceled) return []
    return await Promise.all(filePaths.map(readMetadata))
  })

  // เปิดโฟลเดอร์เพลง
  ipcMain.handle('music:open-folder', async () => {
    const { canceled, filePaths } = await dialog.showOpenDialog({
      properties: ['openDirectory']
    })
    if (canceled) return []

    const folderPath = filePaths[0]
    const files = await scanAudioFolder(folderPath)
    return await Promise.all(files.map(readMetadata))
  })

  // อ่าน metadata จากไฟล์เดียว
  ipcMain.handle('music:get-metadata', async (_, filePath: string) => {
    return readMetadata(filePath)
  })

  // อ่านไฟล์ lyrics (.lrc)
  ipcMain.handle('music:get-lyrics', async (_, audioPath: string) => {
    const lrcPath = audioPath.replace(/\.[^.]+$/, '.lrc')
    try {
      const content = await readFile(lrcPath, 'utf-8')
      return parseLrc(content)
    } catch {
      return null
    }
  })
}

async function readMetadata(filePath: string): Promise<TrackMetadata> {
  try {
    const meta = await mm.parseFile(filePath, { duration: true })
    const { common, format } = meta

    // แปลง cover art เป็น base64
    let coverArt: string | undefined
    if (common.picture && common.picture.length > 0) {
      const pic = common.picture[0]
      coverArt = `data:${pic.format};base64,${Buffer.from(pic.data).toString('base64')}`
    }

    return {
      filePath,
      title: common.title || basename(filePath, extname(filePath)),
      artist: common.artist || 'Unknown Artist',
      album: common.album || 'Unknown Album',
      year: common.year,
      genre: common.genre,
      duration: format.duration || 0,
      trackNumber: common.track?.no ?? undefined,
      coverArt,
      bitrate: format.bitrate,
      sampleRate: format.sampleRate
    }
  } catch {
    return {
      filePath,
      title: basename(filePath, extname(filePath)),
      artist: 'Unknown',
      album: 'Unknown',
      duration: 0
    }
  }
}

async function scanAudioFolder(folderPath: string): Promise<string[]> {
  const files: string[] = []
  const entries = await readdir(folderPath, { withFileTypes: true })

  for (const entry of entries) {
    const fullPath = join(folderPath, entry.name)
    if (entry.isFile() && AUDIO_EXTENSIONS.includes(extname(entry.name).toLowerCase())) {
      files.push(fullPath)
    } else if (entry.isDirectory()) {
      const subFiles = await scanAudioFolder(fullPath)
      files.push(...subFiles)
    }
  }

  return files.sort()
}

interface LrcLine { time: number; text: string }

function parseLrc(lrcContent: string): LrcLine[] {
  const lines: LrcLine[] = []
  const pattern = /\[(\d{2}):(\d{2})\.(\d{2,3})\](.*)/

  for (const line of lrcContent.split('\n')) {
    const match = line.match(pattern)
    if (match) {
      const minutes = parseInt(match[1])
      const seconds = parseInt(match[2])
      const ms = parseInt(match[3].padEnd(3, '0').slice(0, 3))
      const time = minutes * 60 + seconds + ms / 1000
      lines.push({ time, text: match[4].trim() })
    }
  }

  return lines.sort((a, b) => a.time - b.time)
}
```

---

## 2. Audio Player Hook (Web Audio API)

```typescript
// src/renderer/hooks/useAudioPlayer.ts
import { useRef, useState, useCallback, useEffect } from 'react'

interface EqBand {
  frequency: number
  gain: number
  label: string
}

const DEFAULT_EQ_BANDS: EqBand[] = [
  { frequency: 60, gain: 0, label: '60Hz' },
  { frequency: 170, gain: 0, label: '170Hz' },
  { frequency: 310, gain: 0, label: '310Hz' },
  { frequency: 600, gain: 0, label: '600Hz' },
  { frequency: 1000, gain: 0, label: '1kHz' },
  { frequency: 3000, gain: 0, label: '3kHz' },
  { frequency: 6000, gain: 0, label: '6kHz' },
  { frequency: 12000, gain: 0, label: '12kHz' },
  { frequency: 14000, gain: 0, label: '14kHz' },
  { frequency: 16000, gain: 0, label: '16kHz' }
]

export function useAudioPlayer() {
  const audioContextRef = useRef<AudioContext | null>(null)
  const sourceNodeRef = useRef<MediaElementAudioSourceNode | null>(null)
  const gainNodeRef = useRef<GainNode | null>(null)
  const analyserRef = useRef<AnalyserNode | null>(null)
  const eqFiltersRef = useRef<BiquadFilterNode[]>([])
  const audioElementRef = useRef<HTMLAudioElement | null>(null)

  const [isPlaying, setIsPlaying] = useState(false)
  const [currentTime, setCurrentTime] = useState(0)
  const [duration, setDuration] = useState(0)
  const [volume, setVolumeState] = useState(0.8)
  const [isMuted, setIsMuted] = useState(false)
  const [eqBands, setEqBands] = useState<EqBand[]>(DEFAULT_EQ_BANDS)
  const [currentTrackPath, setCurrentTrackPath] = useState<string | null>(null)

  // สร้าง Audio Context และ node graph
  const initAudioContext = useCallback(() => {
    if (audioContextRef.current) return

    const ctx = new AudioContext()
    audioContextRef.current = ctx

    // Gain node สำหรับ volume
    const gainNode = ctx.createGain()
    gainNode.gain.value = volume
    gainNodeRef.current = gainNode

    // Analyser สำหรับ visualization
    const analyser = ctx.createAnalyser()
    analyser.fftSize = 2048
    analyser.smoothingTimeConstant = 0.8
    analyserRef.current = analyser

    // EQ Filters
    const filters = DEFAULT_EQ_BANDS.map(band => {
      const filter = ctx.createBiquadFilter()
      filter.type = 'peaking'
      filter.frequency.value = band.frequency
      filter.gain.value = band.gain
      filter.Q.value = 1.4
      return filter
    })
    eqFiltersRef.current = filters

    // Chain: source -> EQ filters -> gain -> analyser -> destination
    let prevNode: AudioNode = gainNode
    filters.forEach(filter => {
      prevNode.connect(filter)
      prevNode = filter
    })
    prevNode.connect(analyser)
    analyser.connect(ctx.destination)
  }, [])

  // โหลดและเล่นเพลง
  const loadTrack = useCallback(async (filePath: string) => {
    initAudioContext()
    const ctx = audioContextRef.current!

    // Disconnect เก่า
    if (sourceNodeRef.current) {
      sourceNodeRef.current.disconnect()
      audioElementRef.current?.pause()
    }

    const audio = new Audio(`file://${filePath}`)
    audioElementRef.current = audio

    const source = ctx.createMediaElementSource(audio)
    source.connect(gainNodeRef.current!)
    sourceNodeRef.current = source

    audio.addEventListener('timeupdate', () => setCurrentTime(audio.currentTime))
    audio.addEventListener('loadedmetadata', () => setDuration(audio.duration))
    audio.addEventListener('ended', () => setIsPlaying(false))

    if (ctx.state === 'suspended') await ctx.resume()

    await audio.play()
    setIsPlaying(true)
    setCurrentTrackPath(filePath)
  }, [initAudioContext])

  const play = async () => {
    if (!audioElementRef.current) return
    const ctx = audioContextRef.current!
    if (ctx.state === 'suspended') await ctx.resume()
    await audioElementRef.current.play()
    setIsPlaying(true)
  }

  const pause = () => {
    audioElementRef.current?.pause()
    setIsPlaying(false)
  }

  const seek = (time: number) => {
    if (audioElementRef.current) {
      audioElementRef.current.currentTime = time
      setCurrentTime(time)
    }
  }

  const setVolume = (vol: number) => {
    setVolumeState(vol)
    if (gainNodeRef.current) gainNodeRef.current.gain.value = vol
    if (audioElementRef.current) audioElementRef.current.volume = vol
  }

  const toggleMute = () => {
    if (!audioElementRef.current) return
    const newMuted = !isMuted
    audioElementRef.current.muted = newMuted
    setIsMuted(newMuted)
  }

  const updateEqBand = (index: number, gain: number) => {
    const filters = eqFiltersRef.current
    if (filters[index]) {
      filters[index].gain.value = gain
    }
    setEqBands(prev => prev.map((band, i) => i === index ? { ...band, gain } : band))
  }

  // Waveform data สำหรับ visualization
  const getWaveformData = useCallback((): Uint8Array | null => {
    const analyser = analyserRef.current
    if (!analyser) return null
    const data = new Uint8Array(analyser.frequencyBinCount)
    analyser.getByteFrequencyData(data)
    return data
  }, [])

  const getTimeDomainData = useCallback((): Float32Array | null => {
    const analyser = analyserRef.current
    if (!analyser) return null
    const data = new Float32Array(analyser.fftSize)
    analyser.getFloatTimeDomainData(data)
    return data
  }, [])

  return {
    isPlaying, currentTime, duration, volume, isMuted, eqBands, currentTrackPath,
    loadTrack, play, pause, seek, setVolume, toggleMute, updateEqBand,
    getWaveformData, getTimeDomainData
  }
}
```

---

## 3. Waveform Visualizer

```tsx
// src/renderer/components/WaveformVisualizer.tsx
import { useRef, useEffect } from 'react'

interface WaveformVisualizerProps {
  getFrequencyData: () => Uint8Array | null
  isPlaying: boolean
  type?: 'bars' | 'wave' | 'circle'
  color?: string
}

export function WaveformVisualizer({ getFrequencyData, isPlaying, type = 'bars', color = '#1db954' }: WaveformVisualizerProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const animFrameRef = useRef<number>()

  useEffect(() => {
    const canvas = canvasRef.current
    if (!canvas) return

    const ctx = canvas.getContext('2d')!

    const draw = () => {
      animFrameRef.current = requestAnimationFrame(draw)

      const data = getFrequencyData()
      if (!data) return

      const { width, height } = canvas
      ctx.clearRect(0, 0, width, height)
      ctx.fillStyle = 'transparent'

      if (type === 'bars') {
        const barWidth = width / data.length * 2.5
        let x = 0

        for (let i = 0; i < data.length; i++) {
          const barHeight = (data[i] / 255) * height * 0.9
          const hue = (i / data.length) * 120 + 180
          ctx.fillStyle = `hsl(${hue}, 80%, 60%)`
          ctx.fillRect(x, height - barHeight, barWidth - 1, barHeight)
          x += barWidth + 1
          if (x > width) break
        }
      } else if (type === 'wave') {
        ctx.beginPath()
        ctx.strokeStyle = color
        ctx.lineWidth = 2

        const sliceWidth = width / data.length
        let x = 0

        for (let i = 0; i < data.length; i++) {
          const v = data[i] / 128.0
          const y = (v * height) / 2

          if (i === 0) ctx.moveTo(x, y)
          else ctx.lineTo(x, y)
          x += sliceWidth
        }

        ctx.lineTo(width, height / 2)
        ctx.stroke()
      } else if (type === 'circle') {
        const centerX = width / 2
        const centerY = height / 2
        const radius = Math.min(width, height) * 0.3

        ctx.beginPath()
        ctx.strokeStyle = color
        ctx.lineWidth = 2

        for (let i = 0; i < data.length; i++) {
          const angle = (i / data.length) * Math.PI * 2
          const amplitude = (data[i] / 255) * radius * 0.5
          const x1 = centerX + Math.cos(angle) * radius
          const y1 = centerY + Math.sin(angle) * radius
          const x2 = centerX + Math.cos(angle) * (radius + amplitude)
          const y2 = centerY + Math.sin(angle) * (radius + amplitude)

          ctx.moveTo(x1, y1)
          ctx.lineTo(x2, y2)
        }
        ctx.stroke()
      }
    }

    if (isPlaying) {
      draw()
    } else {
      ctx.clearRect(0, 0, canvas.width, canvas.height)
    }

    return () => {
      if (animFrameRef.current) cancelAnimationFrame(animFrameRef.current)
    }
  }, [isPlaying, type, color, getFrequencyData])

  return (
    <canvas
      ref={canvasRef}
      className="waveform-canvas"
      width={600}
      height={120}
    />
  )
}
```

---

## 4. Equalizer Component

```tsx
// src/renderer/components/Equalizer.tsx
interface EqualizerProps {
  bands: { frequency: number; gain: number; label: string }[]
  onBandChange: (index: number, gain: number) => void
}

const EQ_PRESETS: Record<string, number[]> = {
  flat: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
  bass: [8, 7, 4, 2, 0, -1, -2, -3, -4, -4],
  treble: [-4, -4, -3, -2, -1, 0, 2, 4, 7, 8],
  vocal: [-2, -3, -3, 0, 3, 5, 3, 0, -2, -3],
  rock: [5, 3, 0, -2, -3, -2, 0, 3, 5, 6],
  electronic: [6, 4, 0, -2, -4, 0, 3, 4, 5, 6],
  classical: [0, 0, 0, 0, 0, 0, -2, -3, -4, -4]
}

export function Equalizer({ bands, onBandChange }: EqualizerProps) {
  const applyPreset = (presetName: string) => {
    const preset = EQ_PRESETS[presetName]
    if (preset) {
      preset.forEach((gain, i) => onBandChange(i, gain))
    }
  }

  return (
    <div className="equalizer">
      <div className="eq-header">
        <h3>Equalizer</h3>
        <div className="eq-presets">
          {Object.keys(EQ_PRESETS).map(preset => (
            <button key={preset} onClick={() => applyPreset(preset)} className="preset-btn">
              {preset === 'flat' ? 'Flat' :
               preset === 'bass' ? 'Bass Boost' :
               preset === 'treble' ? 'Treble' :
               preset === 'vocal' ? 'Vocal' :
               preset === 'rock' ? 'Rock' :
               preset === 'electronic' ? 'Electronic' : 'Classical'}
            </button>
          ))}
        </div>
      </div>

      <div className="eq-bands">
        {bands.map((band, i) => (
          <div key={i} className="eq-band">
            <input
              type="range"
              min={-12} max={12} step={0.5}
              value={band.gain}
              onChange={e => onBandChange(i, parseFloat(e.target.value))}
              className="eq-slider"
              orient="vertical"
            />
            <span className="eq-value">{band.gain > 0 ? '+' : ''}{band.gain.toFixed(1)}</span>
            <span className="eq-label">{band.label}</span>
          </div>
        ))}
      </div>
    </div>
  )
}
```

---

## 5. Player Controls Component

```tsx
// src/renderer/components/PlayerControls.tsx
import { useState } from 'react'

interface PlayerControlsProps {
  isPlaying: boolean
  currentTime: number
  duration: number
  volume: number
  isMuted: boolean
  onPlayPause: () => void
  onPrevious: () => void
  onNext: () => void
  onSeek: (time: number) => void
  onVolumeChange: (vol: number) => void
  onToggleMute: () => void
  onShuffle: () => void
  onRepeat: () => void
  shuffle: boolean
  repeatMode: 'off' | 'one' | 'all'
}

function formatTime(seconds: number): string {
  if (!seconds || isNaN(seconds)) return '0:00'
  const m = Math.floor(seconds / 60)
  const s = Math.floor(seconds % 60)
  return `${m}:${s.toString().padStart(2, '0')}`
}

export function PlayerControls({
  isPlaying, currentTime, duration, volume, isMuted,
  onPlayPause, onPrevious, onNext, onSeek, onVolumeChange, onToggleMute,
  onShuffle, onRepeat, shuffle, repeatMode
}: PlayerControlsProps) {
  const progress = duration > 0 ? (currentTime / duration) * 100 : 0

  return (
    <div className="player-controls">
      {/* Progress bar */}
      <div className="progress-container">
        <span className="time">{formatTime(currentTime)}</span>
        <input
          type="range"
          min={0} max={duration || 0} step={0.1}
          value={currentTime}
          onChange={e => onSeek(parseFloat(e.target.value))}
          className="progress-bar"
          style={{ '--progress': `${progress}%` } as React.CSSProperties}
        />
        <span className="time">{formatTime(duration)}</span>
      </div>

      {/* Controls */}
      <div className="controls-row">
        <button
          className={`ctrl-btn ${shuffle ? 'active' : ''}`}
          onClick={onShuffle}
          title="สุ่ม"
        >🔀</button>

        <button className="ctrl-btn" onClick={onPrevious} title="เพลงก่อนหน้า">⏮</button>

        <button
          className="ctrl-btn play-btn"
          onClick={onPlayPause}
          title={isPlaying ? 'หยุด' : 'เล่น'}
        >
          {isPlaying ? '⏸' : '▶'}
        </button>

        <button className="ctrl-btn" onClick={onNext} title="เพลงถัดไป">⏭</button>

        <button
          className={`ctrl-btn ${repeatMode !== 'off' ? 'active' : ''}`}
          onClick={onRepeat}
          title="ทำซ้ำ"
        >
          {repeatMode === 'one' ? '🔂' : '🔁'}
        </button>
      </div>

      {/* Volume */}
      <div className="volume-control">
        <button onClick={onToggleMute} className="mute-btn">
          {isMuted ? '🔇' : volume > 0.5 ? '🔊' : '🔉'}
        </button>
        <input
          type="range"
          min={0} max={1} step={0.01}
          value={isMuted ? 0 : volume}
          onChange={e => onVolumeChange(parseFloat(e.target.value))}
          className="volume-slider"
        />
        <span className="volume-value">{Math.round((isMuted ? 0 : volume) * 100)}%</span>
      </div>
    </div>
  )
}
```

---

## 6. Lyrics Display Component

```tsx
// src/renderer/components/LyricsDisplay.tsx
import { useEffect, useRef } from 'react'

interface LrcLine { time: number; text: string }

interface LyricsDisplayProps {
  lyrics: LrcLine[]
  currentTime: number
}

export function LyricsDisplay({ lyrics, currentTime }: LyricsDisplayProps) {
  const containerRef = useRef<HTMLDivElement>(null)
  const activeLineRef = useRef<HTMLDivElement>(null)

  // หาบรรทัดที่กำลังเล่น
  const activeIdx = lyrics.reduce((acc, line, i) => {
    if (line.time <= currentTime) return i
    return acc
  }, -1)

  // Scroll ไปยังบรรทัดที่กำลังเล่น
  useEffect(() => {
    activeLineRef.current?.scrollIntoView({
      behavior: 'smooth',
      block: 'center'
    })
  }, [activeIdx])

  if (!lyrics.length) {
    return (
      <div className="lyrics-container empty">
        <p>ไม่มีเนื้อเพลง</p>
        <p className="hint">วางไฟล์ .lrc ในโฟลเดอร์เดียวกับเพลง</p>
      </div>
    )
  }

  return (
    <div className="lyrics-container" ref={containerRef}>
      {lyrics.map((line, i) => (
        <div
          key={i}
          ref={i === activeIdx ? activeLineRef : undefined}
          className={`lyrics-line ${i === activeIdx ? 'active' : ''} ${i < activeIdx ? 'past' : ''}`}
        >
          {line.text || ' '}
        </div>
      ))}
    </div>
  )
}
```

---

## 7. CSS

```css
/* styles/app.css */
:root { --bg: #121212; --surface: #1e1e1e; --green: #1db954; --text: #fff; }

.player-container {
  display: grid;
  grid-template-rows: 1fr auto;
  height: 100vh;
  background: var(--bg);
  color: var(--text);
}

.now-playing-art {
  width: 200px; height: 200px;
  border-radius: 12px;
  object-fit: cover;
  box-shadow: 0 20px 60px rgba(0,0,0,0.5);
}

.progress-bar {
  flex: 1;
  appearance: none;
  height: 4px;
  border-radius: 2px;
  background: linear-gradient(to right, var(--green) var(--progress), #555 var(--progress));
  cursor: pointer;
}

.play-btn {
  width: 60px; height: 60px;
  border-radius: 50%;
  background: var(--green);
  color: black;
  font-size: 24px;
  border: none;
  cursor: pointer;
  display: flex; align-items: center; justify-content: center;
}

.lyrics-line {
  text-align: center;
  padding: 8px 20px;
  font-size: 16px;
  color: rgba(255,255,255,0.4);
  transition: all 0.3s;
}

.lyrics-line.active {
  color: white;
  font-size: 20px;
  font-weight: 600;
  transform: scale(1.05);
}

.eq-band {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.eq-slider {
  writing-mode: vertical-lr;
  direction: rtl;
  height: 100px;
  appearance: auto;
}
```

---

## สรุปฟีเจอร์

| ฟีเจอร์ | รายละเอียด |
|---------|------------|
| Local file playback | mp3, flac, ogg, wav, m4a |
| Playlist management | เพิ่ม/ลบ/เรียง |
| Waveform visualization | Bars, Wave, Circle |
| 10-band Equalizer | EQ Presets |
| Lyrics display | .lrc sync |
| Metadata reading | music-metadata |
| Cover art display | Embedded / external |
| Shuffle & Repeat | Off/One/All |
| Mini player mode | Compact mode |
| Volume control | ปรับ + mute |
