# Part 87: Real-time Collaboration Features

## สร้างฟีเจอร์ Collaboration แบบ Real-time

ในบทนี้เราจะเรียน CRDT text editing ด้วย Yjs, presence indicators, cursor sharing และ WebRTC peer connections

---

## 1. Yjs Document Sync

```typescript
// src/renderer/collaboration/yjsDoc.ts
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'
import { MonacoBinding } from 'y-monaco'
import * as monaco from 'monaco-editor'

export interface CollaborationSession {
  doc: Y.Doc
  provider: WebsocketProvider
  awareness: WebsocketProvider['awareness']
  text: Y.Text
  disconnect: () => void
}

export function createCollaborationSession(
  roomId: string,
  serverUrl: string,
  userInfo: { name: string; color: string }
): CollaborationSession {
  const doc = new Y.Doc()
  const provider = new WebsocketProvider(serverUrl, roomId, doc, {
    connect: true
  })

  const awareness = provider.awareness

  // ตั้งค่า local user info
  awareness.setLocalState({
    user: userInfo,
    cursor: null,
    selection: null
  })

  const text = doc.getText('content')

  return {
    doc,
    provider,
    awareness,
    text,
    disconnect: () => {
      provider.disconnect()
      doc.destroy()
    }
  }
}

// Bind Yjs text กับ Monaco editor
export function bindMonacoEditor(
  editor: monaco.editor.IStandaloneCodeEditor,
  session: CollaborationSession
): () => void {
  const binding = new MonacoBinding(
    session.text,
    editor.getModel()!,
    new Set([editor]),
    session.awareness
  )

  return () => binding.destroy()
}
```

---

## 2. Presence และ Cursor Sharing

```typescript
// src/renderer/collaboration/presence.ts
import * as Y from 'yjs'
import type { Awareness } from 'y-protocols/awareness'

export interface UserPresence {
  userId: string
  name: string
  color: string
  cursor?: { line: number; column: number }
  selection?: { startLine: number; startColumn: number; endLine: number; endColumn: number }
  lastSeen: number
}

export class PresenceManager {
  private awareness: Awareness
  private listeners: Set<(users: UserPresence[]) => void> = new Set()
  private unsubscribe: (() => void) | null = null

  constructor(awareness: Awareness) {
    this.awareness = awareness
    this.setupListeners()
  }

  private setupListeners(): void {
    const handler = () => {
      this.notifyListeners()
    }

    this.awareness.on('change', handler)
    this.unsubscribe = () => this.awareness.off('change', handler)
  }

  updateCursor(line: number, column: number): void {
    const current = this.awareness.getLocalState() as Record<string, unknown>
    this.awareness.setLocalState({
      ...current,
      cursor: { line, column },
      lastSeen: Date.now()
    })
  }

  updateSelection(
    startLine: number, startColumn: number,
    endLine: number, endColumn: number
  ): void {
    const current = this.awareness.getLocalState() as Record<string, unknown>
    this.awareness.setLocalState({
      ...current,
      selection: { startLine, startColumn, endLine, endColumn },
      lastSeen: Date.now()
    })
  }

  getActiveUsers(): UserPresence[] {
    const users: UserPresence[] = []
    const localId = this.awareness.clientID

    this.awareness.getStates().forEach((state, clientId) => {
      if (clientId === localId) return  // ข้าม local user
      if (state?.user) {
        users.push({
          userId: String(clientId),
          name: state.user.name as string,
          color: state.user.color as string,
          cursor: state.cursor as UserPresence['cursor'],
          selection: state.selection as UserPresence['selection'],
          lastSeen: state.lastSeen as number || Date.now()
        })
      }
    })

    // เรียงตามชื่อ
    return users.sort((a, b) => a.name.localeCompare(b.name))
  }

  onChange(cb: (users: UserPresence[]) => void): () => void {
    this.listeners.add(cb)
    cb(this.getActiveUsers())  // เรียกทันที
    return () => this.listeners.delete(cb)
  }

  private notifyListeners(): void {
    const users = this.getActiveUsers()
    this.listeners.forEach(cb => cb(users))
  }

  dispose(): void {
    this.unsubscribe?.()
    this.listeners.clear()
  }
}
```

---

## 3. Collaboration UI Components

```tsx
// src/renderer/components/CollaborationBar.tsx
import { useState, useEffect } from 'react'
import type { UserPresence } from '../collaboration/presence'

interface CollaborationBarProps {
  users: UserPresence[]
  roomId: string
  connected: boolean
  onShare: () => void
}

export function CollaborationBar({ users, roomId, connected, onShare }: CollaborationBarProps) {
  return (
    <div className="collaboration-bar">
      <div className="status">
        <div className={`dot ${connected ? 'connected' : 'disconnected'}`} />
        <span>{connected ? `${users.length + 1} คน` : 'ไม่ได้เชื่อมต่อ'}</span>
      </div>

      <div className="avatars">
        {users.slice(0, 5).map(user => (
          <UserAvatar key={user.userId} user={user} />
        ))}
        {users.length > 5 && (
          <div className="avatar-overflow">+{users.length - 5}</div>
        )}
      </div>

      <button className="share-btn" onClick={onShare}>
        แชร์
      </button>
    </div>
  )
}

function UserAvatar({ user }: { user: UserPresence }) {
  const initials = user.name.slice(0, 2).toUpperCase()

  return (
    <div
      className="avatar"
      style={{ backgroundColor: user.color }}
      title={`${user.name} - กำลังแก้ไข`}
    >
      {initials}
    </div>
  )
}
```

```tsx
// src/renderer/components/RemoteCursors.tsx
import type { UserPresence } from '../collaboration/presence'

interface RemoteCursorsProps {
  users: UserPresence[]
  lineHeight: number
  charWidth: number
  scrollTop: number
  paddingTop: number
}

export function RemoteCursors({
  users, lineHeight, charWidth, scrollTop, paddingTop
}: RemoteCursorsProps) {
  return (
    <div className="remote-cursors" style={{ position: 'absolute', inset: 0, pointerEvents: 'none' }}>
      {users.map(user => {
        if (!user.cursor) return null
        const top = (user.cursor.line - 1) * lineHeight + paddingTop - scrollTop
        const left = (user.cursor.column - 1) * charWidth

        return (
          <div key={user.userId}>
            {/* Cursor line */}
            <div
              className="remote-cursor"
              style={{
                position: 'absolute',
                top,
                left,
                width: 2,
                height: lineHeight,
                backgroundColor: user.color
              }}
            />
            {/* Name label */}
            <div
              className="cursor-label"
              style={{
                position: 'absolute',
                top: top - 20,
                left,
                backgroundColor: user.color,
                color: '#fff',
                padding: '1px 4px',
                borderRadius: 2,
                fontSize: 11,
                whiteSpace: 'nowrap',
                zIndex: 10
              }}
            >
              {user.name}
            </div>
            {/* Selection */}
            {user.selection && (
              <div
                className="remote-selection"
                style={{
                  position: 'absolute',
                  top: (user.selection.startLine - 1) * lineHeight + paddingTop - scrollTop,
                  left: (user.selection.startColumn - 1) * charWidth,
                  height: (user.selection.endLine - user.selection.startLine + 1) * lineHeight,
                  backgroundColor: user.color + '33',  // 20% opacity
                  minWidth: charWidth
                }}
              />
            )}
          </div>
        )
      })}
    </div>
  )
}
```

---

## 4. Collaboration WebSocket Server

```typescript
// src/server/collabServer.ts
// WebSocket server สำหรับ Yjs sync

import { WebSocketServer, WebSocket } from 'ws'
import * as http from 'http'
import { setupWSConnection } from 'y-websocket/bin/utils'

export function createCollabServer(port: number): { close: () => void } {
  const server = http.createServer()
  const wss = new WebSocketServer({ server })

  wss.on('connection', (ws: WebSocket, req: http.IncomingMessage) => {
    const url = new URL(req.url!, `http://localhost:${port}`)
    const roomId = url.pathname.slice(1)

    console.log(`Client joined room: ${roomId}`)

    setupWSConnection(ws, req, { docName: roomId })
  })

  server.listen(port, () => {
    console.log(`Collaboration server running on ws://localhost:${port}`)
  })

  return {
    close: () => {
      wss.close()
      server.close()
    }
  }
}
```

---

## 5. Share Room Dialog

```tsx
// src/renderer/components/ShareDialog.tsx
import { useState } from 'react'

export function ShareDialog({ roomId, onClose }: { roomId: string; onClose: () => void }) {
  const [copied, setCopied] = useState(false)
  const shareUrl = `myapp://join/${roomId}`

  const copyLink = async () => {
    await navigator.clipboard.writeText(shareUrl)
    setCopied(true)
    setTimeout(() => setCopied(false), 2000)
  }

  return (
    <div className="dialog-overlay" onClick={onClose}>
      <div className="dialog" onClick={e => e.stopPropagation()}>
        <h3>แชร์เอกสาร</h3>
        <p>ส่งลิงก์นี้ให้คนที่ต้องการแก้ไขร่วมกัน</p>

        <div className="share-url">
          <input value={shareUrl} readOnly />
          <button onClick={copyLink} className="btn-primary">
            {copied ? '✓ คัดลอกแล้ว' : 'คัดลอก'}
          </button>
        </div>

        <div className="room-id">
          <span>Room ID: </span>
          <code>{roomId}</code>
        </div>

        <div className="permissions">
          <label>
            <input type="checkbox" defaultChecked />
            ทุกคนที่มีลิงก์แก้ไขได้
          </label>
        </div>

        <button className="btn" onClick={onClose}>ปิด</button>
      </div>
    </div>
  )
}
```

---

## สรุป

| ส่วนประกอบ | เทคโนโลยี | หน้าที่ |
|-----------|---------|---------|
| Document sync | Yjs + y-websocket | CRDT-based text sync |
| Editor binding | y-monaco | Monaco + Yjs |
| Presence | Awareness protocol | cursor/selection sharing |
| Remote cursors | Canvas/DOM overlay | visualize others |
| Collab server | y-websocket server | room-based sync |
| Share UI | Custom dialog | invite collaborators |
