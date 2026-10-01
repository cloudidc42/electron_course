# Part 76: Electron + AI/ML Integration

## ผสาน AI/ML เข้ากับ Electron App

ในบทนี้เราจะเรียนการใช้ Ollama (local LLM), TensorFlow.js, whisper.cpp สำหรับ speech-to-text และ streaming responses

---

## 1. Ollama Integration (Local LLM)

```typescript
// src/main/ollama.ts
import { BrowserWindow } from 'electron'
import http from 'http'

export interface OllamaMessage {
  role: 'user' | 'assistant' | 'system'
  content: string
}

export interface OllamaModel {
  name: string
  size: number
  modified_at: string
  digest: string
}

const OLLAMA_BASE = 'http://localhost:11434'

// ตรวจสอบว่า Ollama รันอยู่หรือไม่
export async function checkOllamaHealth(): Promise<boolean> {
  return new Promise(resolve => {
    http.get(`${OLLAMA_BASE}/`, res => {
      resolve(res.statusCode === 200)
    }).on('error', () => resolve(false))
  })
}

// ดึงรายการ models ที่ติดตั้ง
export async function listModels(): Promise<OllamaModel[]> {
  const response = await fetch(`${OLLAMA_BASE}/api/tags`)
  const data = await response.json() as { models: OllamaModel[] }
  return data.models || []
}

// Chat with streaming
export async function* chatStream(
  model: string,
  messages: OllamaMessage[],
  options?: { temperature?: number; top_p?: number }
): AsyncGenerator<string> {
  const response = await fetch(`${OLLAMA_BASE}/api/chat`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      model,
      messages,
      stream: true,
      options: {
        temperature: options?.temperature ?? 0.7,
        top_p: options?.top_p ?? 0.9
      }
    })
  })

  if (!response.ok || !response.body) {
    throw new Error(`Ollama error: ${response.statusText}`)
  }

  const reader = response.body.getReader()
  const decoder = new TextDecoder()

  while (true) {
    const { done, value } = await reader.read()
    if (done) break

    const chunk = decoder.decode(value, { stream: true })
    const lines = chunk.split('\n').filter(Boolean)

    for (const line of lines) {
      try {
        const data = JSON.parse(line) as {
          message?: { content: string }
          done: boolean
        }
        if (data.message?.content) {
          yield data.message.content
        }
        if (data.done) return
      } catch {
        // skip invalid JSON
      }
    }
  }
}

// Streaming ผ่าน IPC ไปยัง renderer
export function registerOllamaHandlers(win: BrowserWindow): void {
  const { ipcMain } = require('electron')

  ipcMain.handle('ollama:health', () => checkOllamaHealth())
  ipcMain.handle('ollama:models', () => listModels())

  ipcMain.handle('ollama:chat-start', async (_, { model, messages, requestId }: {
    model: string
    messages: OllamaMessage[]
    requestId: string
  }) => {
    try {
      const stream = chatStream(model, messages)
      for await (const token of stream) {
        win.webContents.send('ollama:token', { requestId, token })
      }
      win.webContents.send('ollama:done', { requestId })
    } catch (error) {
      win.webContents.send('ollama:error', { requestId, error: (error as Error).message })
    }
  })
}
```

---

## 2. Chat UI Component

```tsx
// src/renderer/components/AIChat.tsx
import { useState, useRef, useEffect, useCallback } from 'react'

interface Message {
  id: string
  role: 'user' | 'assistant'
  content: string
  streaming?: boolean
}

export function AIChat() {
  const [messages, setMessages] = useState<Message[]>([])
  const [input, setInput] = useState('')
  const [model, setModel] = useState('llama3.2')
  const [models, setModels] = useState<string[]>([])
  const [isStreaming, setIsStreaming] = useState(false)
  const messagesEndRef = useRef<HTMLDivElement>(null)
  const abortRef = useRef<(() => void) | null>(null)

  useEffect(() => {
    loadModels()
  }, [])

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [messages])

  // ฟัง streaming tokens จาก main process
  useEffect(() => {
    const cleanup1 = window.electronAPI.on('ollama:token', ({ requestId, token }: {
      requestId: string; token: string
    }) => {
      setMessages(prev => prev.map(m =>
        m.id === requestId
          ? { ...m, content: m.content + token }
          : m
      ))
    })

    const cleanup2 = window.electronAPI.on('ollama:done', ({ requestId }: { requestId: string }) => {
      setMessages(prev => prev.map(m =>
        m.id === requestId ? { ...m, streaming: false } : m
      ))
      setIsStreaming(false)
    })

    const cleanup3 = window.electronAPI.on('ollama:error', ({ requestId, error }: {
      requestId: string; error: string
    }) => {
      setMessages(prev => prev.map(m =>
        m.id === requestId
          ? { ...m, content: `Error: ${error}`, streaming: false }
          : m
      ))
      setIsStreaming(false)
    })

    return () => { cleanup1(); cleanup2(); cleanup3() }
  }, [])

  const loadModels = async () => {
    const healthy = await window.electronAPI.invoke('ollama:health')
    if (!healthy) return
    const modelList = await window.electronAPI.invoke('ollama:models') as Array<{ name: string }>
    setModels(modelList.map(m => m.name))
    if (modelList.length > 0) setModel(modelList[0].name)
  }

  const sendMessage = useCallback(async () => {
    if (!input.trim() || isStreaming) return

    const userMsg: Message = {
      id: `user-${Date.now()}`,
      role: 'user',
      content: input.trim()
    }

    const requestId = `assistant-${Date.now()}`
    const assistantMsg: Message = {
      id: requestId,
      role: 'assistant',
      content: '',
      streaming: true
    }

    const newMessages = [...messages, userMsg]
    setMessages([...newMessages, assistantMsg])
    setInput('')
    setIsStreaming(true)

    // ส่ง message history ทั้งหมดให้ AI
    const history = newMessages.map(m => ({
      role: m.role,
      content: m.content
    }))

    await window.electronAPI.invoke('ollama:chat-start', {
      model,
      messages: history,
      requestId
    })
  }, [input, isStreaming, messages, model])

  return (
    <div className="ai-chat">
      <div className="chat-header">
        <select value={model} onChange={e => setModel(e.target.value)}>
          {models.map(m => <option key={m} value={m}>{m}</option>)}
        </select>
        <button onClick={() => setMessages([])}>ล้างประวัติ</button>
      </div>

      <div className="chat-messages">
        {messages.map(msg => (
          <div key={msg.id} className={`message ${msg.role}`}>
            <div className="message-role">{msg.role === 'user' ? 'คุณ' : 'AI'}</div>
            <div className="message-content">
              {msg.content}
              {msg.streaming && <span className="cursor">▋</span>}
            </div>
          </div>
        ))}
        <div ref={messagesEndRef} />
      </div>

      <div className="chat-input">
        <textarea
          value={input}
          onChange={e => setInput(e.target.value)}
          onKeyDown={e => {
            if (e.key === 'Enter' && !e.shiftKey) {
              e.preventDefault()
              sendMessage()
            }
          }}
          placeholder="พิมพ์ข้อความ... (Enter ส่ง, Shift+Enter ขึ้นบรรทัด)"
          disabled={isStreaming}
          rows={3}
        />
        <button
          onClick={sendMessage}
          disabled={isStreaming || !input.trim()}
          className="send-btn"
        >
          {isStreaming ? 'กำลังตอบ...' : 'ส่ง'}
        </button>
      </div>
    </div>
  )
}
```

---

## 3. TensorFlow.js Object Detection

```typescript
// src/renderer/ml/objectDetection.ts
import * as tf from '@tensorflow/tfjs'
import * as cocoSsd from '@tensorflow-models/coco-ssd'

let model: cocoSsd.ObjectDetection | null = null

export async function loadModel(): Promise<void> {
  if (model) return
  // โหลด model (cached หลัง download ครั้งแรก)
  model = await cocoSsd.load({
    base: 'lite_mobilenet_v2'   // เบากว่า mobilenet_v2
  })
  console.log('COCO-SSD model loaded')
}

export async function detectObjects(
  imageElement: HTMLImageElement | HTMLVideoElement | HTMLCanvasElement
): Promise<cocoSsd.DetectedObject[]> {
  if (!model) await loadModel()
  return model!.detect(imageElement)
}

// วาด bounding boxes
export function drawDetections(
  canvas: HTMLCanvasElement,
  detections: cocoSsd.DetectedObject[]
): void {
  const ctx = canvas.getContext('2d')!
  ctx.clearRect(0, 0, canvas.width, canvas.height)

  for (const detection of detections) {
    const [x, y, width, height] = detection.bbox
    const score = Math.round(detection.score * 100)

    // Draw box
    ctx.strokeStyle = '#00ff00'
    ctx.lineWidth = 2
    ctx.strokeRect(x, y, width, height)

    // Draw label
    const label = `${detection.class} ${score}%`
    ctx.fillStyle = 'rgba(0, 255, 0, 0.8)'
    ctx.fillRect(x, y - 20, label.length * 8, 20)
    ctx.fillStyle = '#000'
    ctx.font = '14px monospace'
    ctx.fillText(label, x + 2, y - 4)
  }
}
```

---

## 4. Whisper Speech-to-Text

```typescript
// src/main/whisper.ts
// ใช้ whisper.cpp ผ่าน child_process

import { spawn } from 'child_process'
import * as path from 'path'
import * as fs from 'fs'
import * as os from 'os'

export async function transcribeAudio(
  audioPath: string,
  language = 'th',
  modelSize: 'tiny' | 'base' | 'small' = 'base'
): Promise<string> {
  const whisperBin = getWhisperBinary()
  const modelPath = path.join(process.resourcesPath, 'models', `ggml-${modelSize}.bin`)

  return new Promise((resolve, reject) => {
    const args = [
      '-m', modelPath,
      '-f', audioPath,
      '-l', language,
      '--output-txt',
      '--no-timestamps',
      '-t', String(os.cpus().length)   // จำนวน threads
    ]

    const proc = spawn(whisperBin, args)
    let stdout = ''
    let stderr = ''

    proc.stdout.on('data', (data: Buffer) => { stdout += data.toString() })
    proc.stderr.on('data', (data: Buffer) => { stderr += data.toString() })

    proc.on('close', (code) => {
      if (code === 0) {
        // Whisper output มี timestamps, clean ออก
        const text = stdout
          .split('\n')
          .filter(line => !line.startsWith('['))
          .join('\n')
          .trim()
        resolve(text)
      } else {
        reject(new Error(`Whisper failed: ${stderr}`))
      }
    })
  })
}

function getWhisperBinary(): string {
  const platform = process.platform
  const binName = platform === 'win32' ? 'whisper.exe' : 'whisper'
  return path.join(process.resourcesPath, 'bin', binName)
}

// Record audio และ transcribe
export function registerWhisperHandlers(): void {
  const { ipcMain } = require('electron')

  ipcMain.handle('whisper:transcribe', async (_, { audioPath, language }: {
    audioPath: string
    language?: string
  }) => {
    try {
      const text = await transcribeAudio(audioPath, language)
      return { success: true, text }
    } catch (error) {
      return { success: false, error: (error as Error).message }
    }
  })
}
```

---

## 5. Voice Input Component

```tsx
// src/renderer/components/VoiceInput.tsx
import { useState, useRef } from 'react'

export function VoiceInput({ onText }: { onText: (text: string) => void }) {
  const [recording, setRecording] = useState(false)
  const [transcribing, setTranscribing] = useState(false)
  const mediaRecorderRef = useRef<MediaRecorder | null>(null)
  const chunksRef = useRef<Blob[]>([])

  const startRecording = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
    const recorder = new MediaRecorder(stream, { mimeType: 'audio/webm' })
    mediaRecorderRef.current = recorder
    chunksRef.current = []

    recorder.ondataavailable = (e) => chunksRef.current.push(e.data)
    recorder.onstop = async () => {
      stream.getTracks().forEach(t => t.stop())
      await processRecording()
    }

    recorder.start(100)
    setRecording(true)
  }

  const stopRecording = () => {
    mediaRecorderRef.current?.stop()
    setRecording(false)
  }

  const processRecording = async () => {
    setTranscribing(true)
    try {
      const blob = new Blob(chunksRef.current, { type: 'audio/webm' })
      const arrayBuffer = await blob.arrayBuffer()

      // บันทึก audio file ชั่วคราว
      const tempPath = await window.electronAPI.invoke('file:write-temp', {
        data: Array.from(new Uint8Array(arrayBuffer)),
        ext: 'webm'
      })

      const result = await window.electronAPI.invoke('whisper:transcribe', {
        audioPath: tempPath,
        language: 'th'
      }) as { success: boolean; text?: string; error?: string }

      if (result.success && result.text) {
        onText(result.text)
      } else {
        console.error('Transcription failed:', result.error)
      }
    } finally {
      setTranscribing(false)
    }
  }

  return (
    <div className="voice-input">
      {transcribing ? (
        <div className="transcribing">
          <div className="spinner" />
          <span>กำลังแปลงเสียง...</span>
        </div>
      ) : (
        <button
          className={`record-btn ${recording ? 'recording' : ''}`}
          onMouseDown={startRecording}
          onMouseUp={stopRecording}
          onTouchStart={startRecording}
          onTouchEnd={stopRecording}
        >
          {recording ? '🔴 บันทึกอยู่... (ปล่อยเพื่อหยุด)' : '🎤 กดค้างเพื่อพูด'}
        </button>
      )}
    </div>
  )
}
```

---

## สรุป

| เทคโนโลยี | Use Case | วิธีการ |
|----------|---------|---------|
| Ollama | Local LLM chat | HTTP API + streaming |
| TensorFlow.js | Object detection in-browser | COCO-SSD model |
| whisper.cpp | Speech to text | Child process binary |
| Streaming IPC | Real-time token delivery | IPC event stream |
| Privacy | ทำงาน offline ทั้งหมด | ไม่ส่งข้อมูลออก internet |
