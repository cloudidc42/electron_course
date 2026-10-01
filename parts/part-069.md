# Part 69: Project - LAN Chat App

## สร้าง Local Network Chat ด้วย WebSocket

ในบทนี้เราจะสร้าง Chat App ที่ทำงานบน LAN โดยใช้ WebSocket Server ใน Electron, รองรับ rooms, file sharing และ encrypted messages

---

## สถาปัตยกรรม

```
┌──────────────────────────────────────────────────────┐
│  Electron App A (Host)                                │
│  ┌─────────────────┐   ┌──────────────────────────┐  │
│  │  Main Process   │   │  Renderer                │  │
│  │  WebSocket      │◄──│  Chat UI                 │  │
│  │  Server :8765   │   └──────────────────────────┘  │
│  └────────┬────────┘                                  │
└───────────┼──────────────────────────────────────────┘
            │ WebSocket connections
    ┌───────┴──────────────────────┐
    │                              │
┌───┴──────┐                ┌─────┴────┐
│ Client B  │                │ Client C │
└──────────┘                └──────────┘
```

---

## 1. WebSocket Server (Main Process)

```typescript
// src/main/chatServer.ts
import { WebSocketServer, WebSocket } from 'ws'
import { createServer } from 'http'
import { networkInterfaces } from 'os'
import { BrowserWindow, ipcMain } from 'electron'
import { createHash, randomBytes } from 'crypto'

export interface ChatUser {
  id: string
  username: string
  avatar: string
  ws: WebSocket
  rooms: Set<string>
  joinedAt: number
}

export interface ChatMessage {
  id: string
  type: 'text' | 'file' | 'system' | 'typing'
  content: string
  senderId: string
  senderName: string
  roomId: string
  timestamp: number
  encrypted?: boolean
  fileInfo?: { name: string; size: number; mimeType: string }
  reactions?: Record<string, string[]>  // emoji -> userIds
}

export interface ChatRoom {
  id: string
  name: string
  createdBy: string
  members: Set<string>
  messages: ChatMessage[]
  isPrivate: boolean
}

export class ChatServer {
  private wss: WebSocketServer | null = null
  private users: Map<string, ChatUser> = new Map()
  private rooms: Map<string, ChatRoom> = new Map()
  private win: BrowserWindow

  constructor(win: BrowserWindow) {
    this.win = win
    // สร้าง default room
    this.rooms.set('general', {
      id: 'general',
      name: 'ทั่วไป',
      createdBy: 'system',
      members: new Set(),
      messages: [],
      isPrivate: false
    })
  }

  start(port: number = 8765): Promise<void> {
    return new Promise((resolve, reject) => {
      const server = createServer()
      this.wss = new WebSocketServer({ server })

      this.wss.on('connection', (ws: WebSocket) => {
        this.handleConnection(ws)
      })

      server.on('error', reject)
      server.listen(port, () => {
        console.log(`Chat server listening on port ${port}`)
        resolve()
      })
    })
  }

  private handleConnection(ws: WebSocket): void {
    let userId: string | null = null

    ws.on('message', (data: Buffer) => {
      try {
        const message = JSON.parse(data.toString())
        this.handleMessage(ws, message, userId, (id) => { userId = id })
      } catch (error) {
        this.sendError(ws, 'Invalid message format')
      }
    })

    ws.on('close', () => {
      if (userId) this.handleDisconnect(userId)
    })

    ws.on('error', (error) => {
      console.error('WebSocket error:', error)
    })
  }

  private handleMessage(
    ws: WebSocket,
    message: { type: string; data: Record<string, unknown> },
    userId: string | null,
    setUserId: (id: string) => void
  ): void {
    switch (message.type) {
      case 'join':
        this.handleJoin(ws, message.data as { username: string; avatar: string }, setUserId)
        break

      case 'message':
        if (userId) this.handleChatMessage(userId, message.data as { content: string; roomId: string; encrypted?: boolean })
        break

      case 'join-room':
        if (userId) this.handleJoinRoom(userId, message.data.roomId as string)
        break

      case 'create-room':
        if (userId) this.handleCreateRoom(userId, message.data as { name: string; isPrivate: boolean })
        break

      case 'typing':
        if (userId) this.handleTyping(userId, message.data.roomId as string)
        break

      case 'reaction':
        if (userId) this.handleReaction(userId, message.data as { messageId: string; roomId: string; emoji: string })
        break

      case 'file':
        if (userId) this.handleFile(userId, message.data as { roomId: string; name: string; data: string; mimeType: string })
        break
    }
  }

  private handleJoin(
    ws: WebSocket,
    data: { username: string; avatar: string },
    setUserId: (id: string) => void
  ): void {
    const userId = randomBytes(8).toString('hex')
    const user: ChatUser = {
      id: userId,
      username: data.username || `ผู้ใช้_${userId.slice(0, 4)}`,
      avatar: data.avatar || this.generateAvatar(userId),
      ws,
      rooms: new Set(['general']),
      joinedAt: Date.now()
    }

    this.users.set(userId, user)
    setUserId(userId)

    // เข้าร่วม general room
    const generalRoom = this.rooms.get('general')!
    generalRoom.members.add(userId)

    // ส่งข้อมูลกลับ
    this.send(ws, {
      type: 'joined',
      data: {
        userId,
        rooms: this.getRoomList(),
        users: this.getUserList()
      }
    })

    // แจ้งผู้ใช้คนอื่น
    const systemMessage = this.createSystemMessage(
      `${user.username} เข้าร่วมห้องสนทนา`,
      'general'
    )
    this.broadcastToRoom('general', { type: 'message', data: systemMessage }, userId)

    // อัพเดท UI ของ host
    this.win.webContents.send('chat:user-joined', this.getUserList())
  }

  private handleChatMessage(
    userId: string,
    data: { content: string; roomId: string; encrypted?: boolean }
  ): void {
    const user = this.users.get(userId)
    if (!user) return

    const room = this.rooms.get(data.roomId)
    if (!room || !room.members.has(userId)) return

    const message: ChatMessage = {
      id: randomBytes(8).toString('hex'),
      type: 'text',
      content: data.content,
      senderId: userId,
      senderName: user.username,
      roomId: data.roomId,
      timestamp: Date.now(),
      encrypted: data.encrypted,
      reactions: {}
    }

    room.messages.push(message)
    if (room.messages.length > 500) room.messages.shift()

    this.broadcastToRoom(data.roomId, { type: 'message', data: message })
    this.win.webContents.send('chat:message', message)
  }

  private handleJoinRoom(userId: string, roomId: string): void {
    const user = this.users.get(userId)
    const room = this.rooms.get(roomId)
    if (!user || !room) return

    user.rooms.add(roomId)
    room.members.add(userId)

    // ส่งประวัติข้อความ
    this.send(user.ws, {
      type: 'room-history',
      data: { roomId, messages: room.messages.slice(-50) }
    })

    const systemMsg = this.createSystemMessage(`${user.username} เข้าร่วมห้อง ${room.name}`, roomId)
    this.broadcastToRoom(roomId, { type: 'message', data: systemMsg })
  }

  private handleCreateRoom(userId: string, data: { name: string; isPrivate: boolean }): void {
    const roomId = randomBytes(8).toString('hex')
    const user = this.users.get(userId)
    if (!user) return

    const room: ChatRoom = {
      id: roomId,
      name: data.name,
      createdBy: userId,
      members: new Set([userId]),
      messages: [],
      isPrivate: data.isPrivate
    }

    this.rooms.set(roomId, room)
    user.rooms.add(roomId)

    this.broadcast({ type: 'room-created', data: { room: this.getRoomInfo(room) } })
  }

  private handleTyping(userId: string, roomId: string): void {
    const user = this.users.get(userId)
    if (!user) return

    this.broadcastToRoom(roomId, {
      type: 'typing',
      data: { userId, username: user.username, roomId }
    }, userId)
  }

  private handleReaction(userId: string, data: { messageId: string; roomId: string; emoji: string }): void {
    const room = this.rooms.get(data.roomId)
    if (!room) return

    const message = room.messages.find(m => m.id === data.messageId)
    if (!message) return

    if (!message.reactions) message.reactions = {}
    if (!message.reactions[data.emoji]) message.reactions[data.emoji] = []

    const reactions = message.reactions[data.emoji]
    const idx = reactions.indexOf(userId)
    if (idx === -1) reactions.push(userId)
    else reactions.splice(idx, 1)

    this.broadcastToRoom(data.roomId, {
      type: 'reaction-updated',
      data: { messageId: data.messageId, reactions: message.reactions }
    })
  }

  private handleFile(userId: string, data: { roomId: string; name: string; data: string; mimeType: string }): void {
    const user = this.users.get(userId)
    const room = this.rooms.get(data.roomId)
    if (!user || !room) return

    const message: ChatMessage = {
      id: randomBytes(8).toString('hex'),
      type: 'file',
      content: data.data,  // base64
      senderId: userId,
      senderName: user.username,
      roomId: data.roomId,
      timestamp: Date.now(),
      fileInfo: {
        name: data.name,
        size: Buffer.from(data.data, 'base64').length,
        mimeType: data.mimeType
      }
    }

    room.messages.push(message)
    this.broadcastToRoom(data.roomId, { type: 'message', data: message })
  }

  private handleDisconnect(userId: string): void {
    const user = this.users.get(userId)
    if (!user) return

    user.rooms.forEach(roomId => {
      const room = this.rooms.get(roomId)
      if (room) {
        room.members.delete(userId)
        const msg = this.createSystemMessage(`${user.username} ออกจากห้อง`, roomId)
        this.broadcastToRoom(roomId, { type: 'message', data: msg })
      }
    })

    this.users.delete(userId)
    this.broadcast({ type: 'user-left', data: { userId, users: this.getUserList() } })
    this.win.webContents.send('chat:user-left', this.getUserList())
  }

  private broadcastToRoom(roomId: string, message: object, excludeUserId?: string): void {
    const room = this.rooms.get(roomId)
    if (!room) return

    room.members.forEach(userId => {
      if (userId === excludeUserId) return
      const user = this.users.get(userId)
      if (user && user.ws.readyState === WebSocket.OPEN) {
        this.send(user.ws, message)
      }
    })
  }

  private broadcast(message: object, excludeUserId?: string): void {
    this.users.forEach((user, userId) => {
      if (userId === excludeUserId) return
      if (user.ws.readyState === WebSocket.OPEN) {
        this.send(user.ws, message)
      }
    })
  }

  private send(ws: WebSocket, data: object): void {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(data))
    }
  }

  private sendError(ws: WebSocket, message: string): void {
    this.send(ws, { type: 'error', data: { message } })
  }

  private createSystemMessage(text: string, roomId: string): ChatMessage {
    return {
      id: randomBytes(4).toString('hex'),
      type: 'system',
      content: text,
      senderId: 'system',
      senderName: 'System',
      roomId,
      timestamp: Date.now()
    }
  }

  private generateAvatar(userId: string): string {
    const colors = ['#e74c3c', '#3498db', '#2ecc71', '#f39c12', '#9b59b6']
    const color = colors[parseInt(userId.slice(0, 2), 16) % colors.length]
    return `${color}:${userId.slice(0, 2).toUpperCase()}`
  }

  private getRoomList() {
    return Array.from(this.rooms.values()).map(r => this.getRoomInfo(r))
  }

  private getRoomInfo(room: ChatRoom) {
    return { id: room.id, name: room.name, memberCount: room.members.size, isPrivate: room.isPrivate }
  }

  private getUserList() {
    return Array.from(this.users.values()).map(u => ({
      id: u.id, username: u.username, avatar: u.avatar, joinedAt: u.joinedAt
    }))
  }

  getServerIPs(): string[] {
    const interfaces = networkInterfaces()
    const ips: string[] = []
    for (const [, iface] of Object.entries(interfaces)) {
      iface?.forEach(addr => {
        if (addr.family === 'IPv4' && !addr.internal) ips.push(addr.address)
      })
    }
    return ips
  }
}
```

---

## 2. Chat Client Hook

```typescript
// src/renderer/hooks/useChatClient.ts
import { useRef, useState, useCallback, useEffect } from 'react'
import { ChatMessage } from '../../main/chatServer'

interface Room { id: string; name: string; memberCount: number }
interface User { id: string; username: string; avatar: string }

export function useChatClient(serverUrl: string, username: string, avatar: string) {
  const wsRef = useRef<WebSocket | null>(null)
  const [connected, setConnected] = useState(false)
  const [userId, setUserId] = useState<string | null>(null)
  const [rooms, setRooms] = useState<Room[]>([])
  const [users, setUsers] = useState<User[]>([])
  const [messages, setMessages] = useState<Record<string, ChatMessage[]>>({})
  const [activeRoom, setActiveRoom] = useState('general')
  const [typingUsers, setTypingUsers] = useState<Record<string, { username: string; timeout: ReturnType<typeof setTimeout> }>>({})

  const connect = useCallback(() => {
    const ws = new WebSocket(serverUrl)
    wsRef.current = ws

    ws.onopen = () => {
      setConnected(true)
      ws.send(JSON.stringify({ type: 'join', data: { username, avatar } }))
    }

    ws.onmessage = (e) => {
      const { type, data } = JSON.parse(e.data)

      switch (type) {
        case 'joined':
          setUserId(data.userId)
          setRooms(data.rooms)
          setUsers(data.users)
          break

        case 'message':
          setMessages(prev => ({
            ...prev,
            [data.roomId]: [...(prev[data.roomId] || []), data]
          }))
          // Clear typing indicator
          if (data.senderId !== 'system') {
            setTypingUsers(prev => {
              const next = { ...prev }
              delete next[data.senderId]
              return next
            })
          }
          break

        case 'room-history':
          setMessages(prev => ({ ...prev, [data.roomId]: data.messages }))
          break

        case 'typing':
          if (data.roomId === activeRoom) {
            setTypingUsers(prev => {
              const next = { ...prev }
              if (next[data.userId]?.timeout) clearTimeout(next[data.userId].timeout)
              const timeout = setTimeout(() => {
                setTypingUsers(p => { const n = { ...p }; delete n[data.userId]; return n })
              }, 3000)
              next[data.userId] = { username: data.username, timeout }
              return next
            })
          }
          break

        case 'room-created':
          setRooms(prev => [...prev, data.room])
          break

        case 'user-left':
          setUsers(data.users)
          break

        case 'reaction-updated':
          setMessages(prev => {
            const roomMessages = prev[activeRoom] || []
            return {
              ...prev,
              [activeRoom]: roomMessages.map(m =>
                m.id === data.messageId ? { ...m, reactions: data.reactions } : m
              )
            }
          })
          break
      }
    }

    ws.onclose = () => {
      setConnected(false)
      setTimeout(() => connect(), 3000)
    }

    ws.onerror = () => ws.close()
  }, [serverUrl, username, avatar])

  useEffect(() => {
    connect()
    return () => wsRef.current?.close()
  }, [connect])

  const sendMessage = useCallback((content: string, encrypted = false) => {
    wsRef.current?.send(JSON.stringify({
      type: 'message',
      data: { content, roomId: activeRoom, encrypted }
    }))
  }, [activeRoom])

  const sendTyping = useCallback(() => {
    wsRef.current?.send(JSON.stringify({ type: 'typing', data: { roomId: activeRoom } }))
  }, [activeRoom])

  const joinRoom = useCallback((roomId: string) => {
    wsRef.current?.send(JSON.stringify({ type: 'join-room', data: { roomId } }))
    setActiveRoom(roomId)
  }, [])

  const sendFile = useCallback(async (file: File) => {
    const reader = new FileReader()
    reader.onload = (e) => {
      const base64 = (e.target?.result as string).split(',')[1]
      wsRef.current?.send(JSON.stringify({
        type: 'file',
        data: { roomId: activeRoom, name: file.name, data: base64, mimeType: file.type }
      }))
    }
    reader.readAsDataURL(file)
  }, [activeRoom])

  const sendReaction = useCallback((messageId: string, emoji: string) => {
    wsRef.current?.send(JSON.stringify({
      type: 'reaction',
      data: { messageId, roomId: activeRoom, emoji }
    }))
  }, [activeRoom])

  return {
    connected, userId, rooms, users, messages, activeRoom, typingUsers,
    sendMessage, sendTyping, joinRoom, sendFile, sendReaction, setActiveRoom
  }
}
```

---

## 3. Chat UI Component

```tsx
// src/renderer/components/ChatWindow.tsx
import { useState, useRef, useEffect, KeyboardEvent } from 'react'
import { useChatClient } from '../hooks/useChatClient'
import { MessageBubble } from './MessageBubble'

const EMOJI_LIST = ['👍', '❤️', '😄', '😮', '😢', '😡']

interface ChatWindowProps {
  serverUrl: string
  username: string
  avatar: string
}

export function ChatWindow({ serverUrl, username, avatar }: ChatWindowProps) {
  const [inputText, setInputText] = useState('')
  const [showEmojiPicker, setShowEmojiPicker] = useState(false)
  const messagesEndRef = useRef<HTMLDivElement>(null)
  const fileInputRef = useRef<HTMLInputElement>(null)
  const typingTimeoutRef = useRef<ReturnType<typeof setTimeout>>()

  const {
    connected, userId, rooms, users, messages, activeRoom, typingUsers,
    sendMessage, sendTyping, joinRoom, sendFile, sendReaction
  } = useChatClient(serverUrl, username, avatar)

  const currentMessages = messages[activeRoom] || []
  const typingList = Object.values(typingUsers).map(t => t.username)

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [currentMessages])

  const handleSend = () => {
    const text = inputText.trim()
    if (!text || !connected) return
    sendMessage(text)
    setInputText('')
  }

  const handleKeyDown = (e: KeyboardEvent<HTMLTextAreaElement>) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault()
      handleSend()
    } else {
      // Typing indicator
      clearTimeout(typingTimeoutRef.current)
      sendTyping()
    }
  }

  const handleFileSelect = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0]
    if (file) sendFile(file)
  }

  return (
    <div className="chat-window">
      {/* Sidebar */}
      <div className="chat-sidebar">
        <div className="connection-status">
          <span className={`status-dot ${connected ? 'online' : 'offline'}`} />
          {connected ? 'เชื่อมต่อแล้ว' : 'กำลังเชื่อมต่อ...'}
        </div>

        <div className="rooms-section">
          <h4>ห้อง</h4>
          {rooms.map(room => (
            <button
              key={room.id}
              className={`room-btn ${activeRoom === room.id ? 'active' : ''}`}
              onClick={() => joinRoom(room.id)}
            >
              # {room.name}
              <span className="member-count">{room.memberCount}</span>
            </button>
          ))}
        </div>

        <div className="users-section">
          <h4>ออนไลน์ ({users.length})</h4>
          {users.map(user => (
            <div key={user.id} className="user-item">
              <div className="user-avatar" style={{ background: user.avatar.split(':')[0] }}>
                {user.avatar.split(':')[1]}
              </div>
              <span>{user.username}</span>
              {user.id === userId && <span className="you-label">(คุณ)</span>}
            </div>
          ))}
        </div>
      </div>

      {/* Main chat */}
      <div className="chat-main">
        <div className="chat-header">
          <h3># {rooms.find(r => r.id === activeRoom)?.name || activeRoom}</h3>
        </div>

        <div className="messages-container">
          {currentMessages.map(msg => (
            <MessageBubble
              key={msg.id}
              message={msg}
              isOwn={msg.senderId === userId}
              onReact={(emoji) => sendReaction(msg.id, emoji)}
              emojis={EMOJI_LIST}
            />
          ))}

          {typingList.length > 0 && (
            <div className="typing-indicator">
              <span className="typing-dots">
                <span>•</span><span>•</span><span>•</span>
              </span>
              {typingList.join(', ')} กำลังพิมพ์...
            </div>
          )}

          <div ref={messagesEndRef} />
        </div>

        <div className="input-area">
          <button
            className="attach-btn"
            onClick={() => fileInputRef.current?.click()}
            title="แนบไฟล์"
          >📎</button>
          <input
            type="file"
            ref={fileInputRef}
            style={{ display: 'none' }}
            onChange={handleFileSelect}
          />

          <textarea
            value={inputText}
            onChange={e => setInputText(e.target.value)}
            onKeyDown={handleKeyDown}
            placeholder="พิมพ์ข้อความ... (Enter ส่ง, Shift+Enter ขึ้นบรรทัด)"
            className="message-input"
            rows={1}
            disabled={!connected}
          />

          <button onClick={handleSend} disabled={!connected || !inputText.trim()} className="send-btn">
            ส่ง
          </button>
        </div>
      </div>
    </div>
  )
}
```

---

## 4. Message Bubble Component

```tsx
// src/renderer/components/MessageBubble.tsx
import { useState } from 'react'
import { ChatMessage } from '../../main/chatServer'

interface MessageBubbleProps {
  message: ChatMessage
  isOwn: boolean
  onReact: (emoji: string) => void
  emojis: string[]
}

export function MessageBubble({ message, isOwn, onReact, emojis }: MessageBubbleProps) {
  const [showReactions, setShowReactions] = useState(false)

  if (message.type === 'system') {
    return (
      <div className="system-message">
        <span>{message.content}</span>
        <span className="msg-time">{new Date(message.timestamp).toLocaleTimeString('th-TH')}</span>
      </div>
    )
  }

  return (
    <div className={`message-bubble ${isOwn ? 'own' : 'other'}`}
      onMouseEnter={() => setShowReactions(true)}
      onMouseLeave={() => setShowReactions(false)}
    >
      {!isOwn && (
        <div className="sender-name">{message.senderName}</div>
      )}

      {message.type === 'text' && (
        <div className="message-content">
          {message.content}
          {message.encrypted && <span className="encrypted-badge" title="เข้ารหัส">🔒</span>}
        </div>
      )}

      {message.type === 'file' && message.fileInfo && (
        <div className="file-message">
          {message.fileInfo.mimeType.startsWith('image/') ? (
            <img
              src={`data:${message.fileInfo.mimeType};base64,${message.content}`}
              alt={message.fileInfo.name}
              className="file-image"
            />
          ) : (
            <a
              href={`data:${message.fileInfo.mimeType};base64,${message.content}`}
              download={message.fileInfo.name}
              className="file-download"
            >
              📄 {message.fileInfo.name} ({(message.fileInfo.size / 1024).toFixed(1)} KB)
            </a>
          )}
        </div>
      )}

      {/* Reactions */}
      {message.reactions && Object.entries(message.reactions).filter(([, users]) => users.length > 0).length > 0 && (
        <div className="reactions">
          {Object.entries(message.reactions).filter(([, u]) => u.length > 0).map(([emoji, userIds]) => (
            <button key={emoji} className="reaction-btn" onClick={() => onReact(emoji)}>
              {emoji} {userIds.length}
            </button>
          ))}
        </div>
      )}

      <span className="msg-time">{new Date(message.timestamp).toLocaleTimeString('th-TH')}</span>

      {showReactions && (
        <div className="reaction-picker">
          {emojis.map(emoji => (
            <button key={emoji} onClick={() => onReact(emoji)}>{emoji}</button>
          ))}
        </div>
      )}
    </div>
  )
}
```

---

## สรุปฟีเจอร์

| ฟีเจอร์ | รายละเอียด |
|---------|------------|
| WebSocket Server | ws library ใน Electron main |
| Multi-room chat | สร้างห้องใหม่ได้ |
| User presence | แสดงรายชื่อออนไลน์ |
| File sharing | ส่งรูปภาพและไฟล์ |
| Typing indicator | แสดงเมื่อกำลังพิมพ์ |
| Emoji reactions | React ต่อข้อความ |
| Message history | เก็บ 500 ข้อความล่าสุด |
| Auto-reconnect | กลับมาเชื่อมต่อเมื่อหลุด |
| System messages | แจ้งเข้า/ออกห้อง |
