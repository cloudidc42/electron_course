# Part 66: Project - Password Manager

## สร้าง Password Manager พร้อม Encryption

ในบทนี้เราจะสร้าง Password Manager ที่มี encrypted storage, AES-256 encryption, password generator และ secure clipboard

---

## หลักการความปลอดภัย

```
┌─────────────────────────────────────────────────────┐
│  Master Password                                      │
│       │                                              │
│       ▼                                              │
│  PBKDF2/Argon2 Key Derivation                        │
│       │                                              │
│       ▼                                              │
│  AES-256-GCM Encryption Key                          │
│       │                                              │
│       ▼                                              │
│  Encrypted Vault (.vault file)                       │
│  ┌─────────────────────────────┐                    │
│  │ { entries: [...] }          │                    │
│  └─────────────────────────────┘                    │
└─────────────────────────────────────────────────────┘
```

---

## 1. Encryption Service

```typescript
// src/main/encryptionService.ts
import { createCipheriv, createDecipheriv, randomBytes, createHash, pbkdf2Sync, timingSafeEqual } from 'crypto'

const ALGORITHM = 'aes-256-gcm'
const KEY_LENGTH = 32
const IV_LENGTH = 16
const SALT_LENGTH = 32
const AUTH_TAG_LENGTH = 16
const PBKDF2_ITERATIONS = 310_000

export interface EncryptedData {
  salt: string
  iv: string
  authTag: string
  data: string
  version: number
}

export class EncryptionService {
  // derive encryption key จาก master password
  deriveKey(password: string, salt: Buffer): Buffer {
    return pbkdf2Sync(password, salt, PBKDF2_ITERATIONS, KEY_LENGTH, 'sha512')
  }

  // เข้ารหัส
  encrypt(plaintext: string, password: string): EncryptedData {
    const salt = randomBytes(SALT_LENGTH)
    const iv = randomBytes(IV_LENGTH)
    const key = this.deriveKey(password, salt)

    const cipher = createCipheriv(ALGORITHM, key, iv, { authTagLength: AUTH_TAG_LENGTH })
    
    let encrypted = cipher.update(plaintext, 'utf8', 'hex')
    encrypted += cipher.final('hex')
    const authTag = cipher.getAuthTag()

    return {
      salt: salt.toString('hex'),
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex'),
      data: encrypted,
      version: 1
    }
  }

  // ถอดรหัส
  decrypt(encrypted: EncryptedData, password: string): string {
    const salt = Buffer.from(encrypted.salt, 'hex')
    const iv = Buffer.from(encrypted.iv, 'hex')
    const authTag = Buffer.from(encrypted.authTag, 'hex')
    const key = this.deriveKey(password, salt)

    const decipher = createDecipheriv(ALGORITHM, key, iv, { authTagLength: AUTH_TAG_LENGTH })
    decipher.setAuthTag(authTag)

    let decrypted = decipher.update(encrypted.data, 'hex', 'utf8')
    decrypted += decipher.final('utf8')

    return decrypted
  }

  // Hash master password สำหรับ verification (ไม่เก็บ password ตรงๆ)
  hashMasterPassword(password: string): string {
    const salt = 'pm-verify-salt-v1'
    return createHash('sha256').update(password + salt).digest('hex')
  }

  // ตรวจสอบ master password
  verifyMasterPassword(password: string, hash: string): boolean {
    const computed = this.hashMasterPassword(password)
    try {
      return timingSafeEqual(Buffer.from(computed, 'hex'), Buffer.from(hash, 'hex'))
    } catch {
      return false
    }
  }
}

export const encryptionService = new EncryptionService()
```

---

## 2. Vault Manager

```typescript
// src/main/vaultManager.ts
import { app, safeStorage } from 'electron'
import { readFile, writeFile, access } from 'fs/promises'
import { join } from 'path'
import { encryptionService, EncryptedData } from './encryptionService'

export interface PasswordEntry {
  id: string
  title: string
  username: string
  password: string
  url: string
  notes: string
  category: string
  tags: string[]
  createdAt: number
  updatedAt: number
  passwordHistory: { password: string; changedAt: number }[]
  isFavorite: boolean
  strength: number
}

export interface Vault {
  entries: PasswordEntry[]
  categories: string[]
  createdAt: number
}

export class VaultManager {
  private vault: Vault | null = null
  private masterPassword: string | null = null
  private vaultPath: string

  constructor() {
    this.vaultPath = join(app.getPath('userData'), 'vault.encrypted')
  }

  // เปิด vault (unlock)
  async unlock(masterPassword: string): Promise<{ success: boolean; error?: string }> {
    try {
      const exists = await access(this.vaultPath).then(() => true).catch(() => false)

      if (!exists) {
        // สร้าง vault ใหม่
        this.vault = { entries: [], categories: ['ทั่วไป', 'งาน', 'ธนาคาร', 'โซเชียล'], createdAt: Date.now() }
        this.masterPassword = masterPassword
        await this.save()
        return { success: true }
      }

      // อ่าน vault ที่มีอยู่
      const fileContent = await readFile(this.vaultPath, 'utf-8')
      const { hash, encrypted }: { hash: string; encrypted: EncryptedData } = JSON.parse(fileContent)

      // ตรวจสอบ password
      if (!encryptionService.verifyMasterPassword(masterPassword, hash)) {
        return { success: false, error: 'รหัสผ่านไม่ถูกต้อง' }
      }

      // ถอดรหัส vault
      const decrypted = encryptionService.decrypt(encrypted, masterPassword)
      this.vault = JSON.parse(decrypted)
      this.masterPassword = masterPassword

      return { success: true }
    } catch (error) {
      return { success: false, error: 'ไม่สามารถเปิด vault ได้' }
    }
  }

  // ล็อค vault
  lock(): void {
    this.vault = null
    this.masterPassword = null
  }

  // บันทึก vault
  async save(): Promise<void> {
    if (!this.vault || !this.masterPassword) throw new Error('Vault is locked')

    const plaintext = JSON.stringify(this.vault)
    const encrypted = encryptionService.encrypt(plaintext, this.masterPassword)
    const hash = encryptionService.hashMasterPassword(this.masterPassword)

    await writeFile(this.vaultPath, JSON.stringify({ hash, encrypted }), 'utf-8')
  }

  // CRUD Operations
  getEntries(): PasswordEntry[] {
    if (!this.vault) throw new Error('Vault is locked')
    return this.vault.entries
  }

  addEntry(entry: Omit<PasswordEntry, 'id' | 'createdAt' | 'updatedAt' | 'passwordHistory' | 'strength'>): PasswordEntry {
    if (!this.vault) throw new Error('Vault is locked')

    const newEntry: PasswordEntry = {
      ...entry,
      id: `${Date.now()}-${Math.random().toString(36).slice(2)}`,
      createdAt: Date.now(),
      updatedAt: Date.now(),
      passwordHistory: [],
      strength: this.calculateStrength(entry.password)
    }

    this.vault.entries.push(newEntry)
    this.save()
    return newEntry
  }

  updateEntry(id: string, updates: Partial<PasswordEntry>): PasswordEntry | null {
    if (!this.vault) throw new Error('Vault is locked')

    const idx = this.vault.entries.findIndex(e => e.id === id)
    if (idx === -1) return null

    const existing = this.vault.entries[idx]

    // บันทึก password เก่าใน history
    if (updates.password && updates.password !== existing.password) {
      if (!existing.passwordHistory) existing.passwordHistory = []
      existing.passwordHistory.unshift({ password: existing.password, changedAt: Date.now() })
      existing.passwordHistory = existing.passwordHistory.slice(0, 10) // เก็บแค่ 10 รายการ
      updates.strength = this.calculateStrength(updates.password)
    }

    this.vault.entries[idx] = { ...existing, ...updates, updatedAt: Date.now() }
    this.save()
    return this.vault.entries[idx]
  }

  deleteEntry(id: string): boolean {
    if (!this.vault) throw new Error('Vault is locked')
    const before = this.vault.entries.length
    this.vault.entries = this.vault.entries.filter(e => e.id !== id)
    if (this.vault.entries.length < before) { this.save(); return true }
    return false
  }

  // คำนวณความแข็งแกร่งของ password (0-100)
  calculateStrength(password: string): number {
    let score = 0
    if (password.length >= 8) score += 20
    if (password.length >= 12) score += 20
    if (password.length >= 16) score += 10
    if (/[A-Z]/.test(password)) score += 10
    if (/[a-z]/.test(password)) score += 10
    if (/[0-9]/.test(password)) score += 10
    if (/[^A-Za-z0-9]/.test(password)) score += 20
    return Math.min(100, score)
  }

  isLocked(): boolean {
    return this.vault === null
  }
}

export const vaultManager = new VaultManager()
```

---

## 3. Password Generator

```typescript
// src/renderer/utils/passwordGenerator.ts
export interface GeneratorOptions {
  length: number
  uppercase: boolean
  lowercase: boolean
  numbers: boolean
  symbols: boolean
  excludeAmbiguous: boolean  // ไม่ใช้ 0, O, l, I
  customSymbols?: string
}

const CHARS = {
  uppercase: 'ABCDEFGHIJKLMNOPQRSTUVWXYZ',
  lowercase: 'abcdefghijklmnopqrstuvwxyz',
  numbers: '0123456789',
  symbols: '!@#$%^&*()_+-=[]{}|;:,.<>?',
  ambiguous: '0Ol1I'
}

export function generatePassword(options: GeneratorOptions): string {
  const {
    length = 16,
    uppercase = true,
    lowercase = true,
    numbers = true,
    symbols = true,
    excludeAmbiguous = false,
    customSymbols
  } = options

  let charset = ''
  const required: string[] = []

  if (uppercase) {
    let chars = CHARS.uppercase
    if (excludeAmbiguous) chars = chars.split('').filter(c => !CHARS.ambiguous.includes(c)).join('')
    charset += chars
    required.push(chars[Math.floor(Math.random() * chars.length)])
  }
  if (lowercase) {
    let chars = CHARS.lowercase
    if (excludeAmbiguous) chars = chars.split('').filter(c => !CHARS.ambiguous.includes(c)).join('')
    charset += chars
    required.push(chars[Math.floor(Math.random() * chars.length)])
  }
  if (numbers) {
    let chars = CHARS.numbers
    if (excludeAmbiguous) chars = chars.split('').filter(c => !CHARS.ambiguous.includes(c)).join('')
    charset += chars
    required.push(chars[Math.floor(Math.random() * chars.length)])
  }
  if (symbols) {
    const chars = customSymbols || CHARS.symbols
    charset += chars
    required.push(chars[Math.floor(Math.random() * chars.length)])
  }

  if (!charset) return ''

  // สุ่ม password แบบ cryptographically secure
  const array = new Uint32Array(length)
  crypto.getRandomValues(array)

  let password = Array.from(array).map(n => charset[n % charset.length]).join('')

  // ใส่ required characters
  const positions = Array.from({ length: required.length }, (_, i) =>
    Math.floor(Math.random() * length)
  )

  const chars = password.split('')
  required.forEach((char, i) => { chars[positions[i]] = char })
  password = chars.join('')

  return password
}

export function getStrengthLabel(score: number): { label: string; color: string } {
  if (score < 30) return { label: 'อ่อนแอมาก', color: '#e74c3c' }
  if (score < 50) return { label: 'อ่อน', color: '#e67e22' }
  if (score < 70) return { label: 'ปานกลาง', color: '#f1c40f' }
  if (score < 90) return { label: 'แข็งแกร่ง', color: '#2ecc71' }
  return { label: 'แข็งแกร่งมาก', color: '#27ae60' }
}

export function checkBreached(password: string): Promise<boolean> {
  // ตรวจสอบกับ HaveIBeenPwned API (k-Anonymity model)
  // ส่งแค่ 5 ตัวแรกของ SHA1 hash - ปลอดภัย
  return crypto.subtle.digest('SHA-1', new TextEncoder().encode(password))
    .then(hashBuffer => {
      const hashHex = Array.from(new Uint8Array(hashBuffer))
        .map(b => b.toString(16).padStart(2, '0'))
        .join('')
        .toUpperCase()
      
      const prefix = hashHex.slice(0, 5)
      const suffix = hashHex.slice(5)
      
      return fetch(`https://api.pwnedpasswords.com/range/${prefix}`)
        .then(r => r.text())
        .then(text => text.includes(suffix))
        .catch(() => false)
    })
}
```

---

## 4. Main App Component

```tsx
// src/renderer/App.tsx
import { useState, useEffect, useMemo } from 'react'
import { LockScreen } from './components/LockScreen'
import { EntryList } from './components/EntryList'
import { EntryForm } from './components/EntryForm'
import { PasswordGenerator } from './components/PasswordGenerator'
import { PasswordEntry } from '../main/vaultManager'

type View = 'list' | 'add' | 'edit' | 'generator'

export default function App() {
  const [isLocked, setIsLocked] = useState(true)
  const [entries, setEntries] = useState<PasswordEntry[]>([])
  const [view, setView] = useState<View>('list')
  const [editingEntry, setEditingEntry] = useState<PasswordEntry | null>(null)
  const [searchQuery, setSearchQuery] = useState('')
  const [selectedCategory, setSelectedCategory] = useState<string>('all')
  const [copiedField, setCopiedField] = useState<string | null>(null)
  const [autoLockTimer, setAutoLockTimer] = useState<ReturnType<typeof setTimeout> | null>(null)

  // Auto-lock หลังจากไม่ใช้งาน 5 นาที
  const resetAutoLock = () => {
    if (autoLockTimer) clearTimeout(autoLockTimer)
    const timer = setTimeout(() => {
      handleLock()
    }, 5 * 60 * 1000)
    setAutoLockTimer(timer)
  }

  const handleUnlock = async (password: string) => {
    const result = await window.electronAPI.vaultUnlock(password)
    if (result.success) {
      const entries = await window.electronAPI.vaultGetEntries()
      setEntries(entries)
      setIsLocked(false)
      resetAutoLock()
    }
    return result
  }

  const handleLock = () => {
    window.electronAPI.vaultLock()
    setIsLocked(true)
    setEntries([])
    if (autoLockTimer) clearTimeout(autoLockTimer)
  }

  const handleSave = async (data: Partial<PasswordEntry>) => {
    if (editingEntry) {
      const updated = await window.electronAPI.vaultUpdateEntry(editingEntry.id, data)
      if (updated) {
        setEntries(prev => prev.map(e => e.id === updated.id ? updated : e))
      }
    } else {
      const newEntry = await window.electronAPI.vaultAddEntry(data)
      setEntries(prev => [...prev, newEntry])
    }
    setView('list')
    setEditingEntry(null)
  }

  const handleCopyPassword = async (entry: PasswordEntry) => {
    await window.electronAPI.copySecure(entry.password)
    setCopiedField(entry.id)
    // Auto-clear clipboard หลัง 30 วินาที
    setTimeout(async () => {
      await window.electronAPI.clearClipboard()
      setCopiedField(null)
    }, 30_000)
  }

  const filteredEntries = useMemo(() =>
    entries.filter(e => {
      const matchesSearch = !searchQuery ||
        e.title.toLowerCase().includes(searchQuery.toLowerCase()) ||
        e.username.toLowerCase().includes(searchQuery.toLowerCase()) ||
        e.url.toLowerCase().includes(searchQuery.toLowerCase())
      
      const matchesCategory = selectedCategory === 'all' || e.category === selectedCategory
      
      return matchesSearch && matchesCategory
    }), [entries, searchQuery, selectedCategory])

  if (isLocked) {
    return <LockScreen onUnlock={handleUnlock} />
  }

  return (
    <div className="app" onMouseMove={resetAutoLock} onKeyDown={resetAutoLock}>
      <div className="sidebar">
        <div className="sidebar-header">
          <h2>🔐 Passwords</h2>
          <button onClick={handleLock} title="ล็อค" className="lock-btn">🔒</button>
        </div>

        <input
          type="search"
          placeholder="ค้นหา..."
          value={searchQuery}
          onChange={e => setSearchQuery(e.target.value)}
          className="search-input"
        />

        <div className="category-list">
          {['all', 'ทั่วไป', 'งาน', 'ธนาคาร', 'โซเชียล'].map(cat => (
            <button
              key={cat}
              className={`category-btn ${selectedCategory === cat ? 'active' : ''}`}
              onClick={() => setSelectedCategory(cat)}
            >
              {cat === 'all' ? '📋 ทั้งหมด' :
               cat === 'ธนาคาร' ? '🏦 ธนาคาร' :
               cat === 'โซเชียล' ? '👤 โซเชียล' :
               cat === 'งาน' ? '💼 งาน' : '📌 ทั่วไป'}
              <span className="count">
                {cat === 'all' ? entries.length : entries.filter(e => e.category === cat).length}
              </span>
            </button>
          ))}
        </div>

        <div className="sidebar-actions">
          <button className="btn-primary" onClick={() => { setEditingEntry(null); setView('add') }}>
            + เพิ่ม
          </button>
          <button onClick={() => setView('generator')}>⚙️ Generator</button>
        </div>
      </div>

      <div className="main-content">
        {view === 'list' && (
          <EntryList
            entries={filteredEntries}
            copiedField={copiedField}
            onEdit={(entry) => { setEditingEntry(entry); setView('edit') }}
            onCopyPassword={handleCopyPassword}
            onDelete={async (id) => {
              if (confirm('ลบรายการนี้?')) {
                await window.electronAPI.vaultDeleteEntry(id)
                setEntries(prev => prev.filter(e => e.id !== id))
              }
            }}
          />
        )}
        {(view === 'add' || view === 'edit') && (
          <EntryForm
            entry={editingEntry}
            onSave={handleSave}
            onCancel={() => { setView('list'); setEditingEntry(null) }}
          />
        )}
        {view === 'generator' && (
          <PasswordGenerator onUse={(password) => {
            if (editingEntry) {
              setEditingEntry({ ...editingEntry, password })
              setView('edit')
            } else {
              setView('add')
            }
          }} />
        )}
      </div>
    </div>
  )
}
```

---

## 5. Entry List Component

```tsx
// src/renderer/components/EntryList.tsx
import { useState } from 'react'
import { PasswordEntry } from '../../main/vaultManager'
import { getStrengthLabel } from '../utils/passwordGenerator'

interface EntryListProps {
  entries: PasswordEntry[]
  copiedField: string | null
  onEdit: (entry: PasswordEntry) => void
  onCopyPassword: (entry: PasswordEntry) => void
  onDelete: (id: string) => void
}

export function EntryList({ entries, copiedField, onEdit, onCopyPassword, onDelete }: EntryListProps) {
  const [visiblePasswords, setVisiblePasswords] = useState<Set<string>>(new Set())
  const [expandedEntry, setExpandedEntry] = useState<string | null>(null)

  const togglePasswordVisibility = (id: string) => {
    setVisiblePasswords(prev => {
      const next = new Set(prev)
      if (next.has(id)) next.delete(id)
      else next.add(id)
      return next
    })
  }

  if (entries.length === 0) {
    return (
      <div className="empty-state">
        <div className="empty-icon">🔐</div>
        <p>ไม่มีรายการ</p>
        <p className="empty-hint">คลิก "+ เพิ่ม" เพื่อเพิ่ม password แรก</p>
      </div>
    )
  }

  return (
    <div className="entry-list">
      {entries.map(entry => {
        const strength = getStrengthLabel(entry.strength || 0)
        const isExpanded = expandedEntry === entry.id

        return (
          <div key={entry.id} className="entry-card">
            <div className="entry-header" onClick={() => setExpandedEntry(isExpanded ? null : entry.id)}>
              <div className="entry-icon">
                {entry.url ? (
                  <img
                    src={`https://www.google.com/s2/favicons?domain=${entry.url}&sz=32`}
                    alt=""
                    onError={e => { (e.target as HTMLImageElement).style.display = 'none' }}
                  />
                ) : '🔑'}
              </div>
              <div className="entry-info">
                <div className="entry-title">{entry.title}</div>
                <div className="entry-username">{entry.username}</div>
              </div>
              <div className="entry-meta">
                <div
                  className="strength-bar"
                  style={{ '--strength': entry.strength + '%', '--color': strength.color } as React.CSSProperties}
                  title={`ความแข็งแกร่ง: ${strength.label}`}
                />
                {entry.isFavorite && <span>⭐</span>}
              </div>
            </div>

            {isExpanded && (
              <div className="entry-details">
                {/* Password */}
                <div className="field-row">
                  <label>Password</label>
                  <div className="field-value">
                    <span className="password-field">
                      {visiblePasswords.has(entry.id) ? entry.password : '••••••••••••'}
                    </span>
                    <button onClick={() => togglePasswordVisibility(entry.id)} title="แสดง/ซ่อน">
                      {visiblePasswords.has(entry.id) ? '🙈' : '👁'}
                    </button>
                    <button
                      onClick={() => onCopyPassword(entry)}
                      className={copiedField === entry.id ? 'copied' : ''}
                      title="คัดลอก (auto-clear 30s)"
                    >
                      {copiedField === entry.id ? '✓ คัดลอกแล้ว' : '📋'}
                    </button>
                  </div>
                </div>

                {/* URL */}
                {entry.url && (
                  <div className="field-row">
                    <label>URL</label>
                    <span className="url-field">{entry.url}</span>
                  </div>
                )}

                {/* Notes */}
                {entry.notes && (
                  <div className="field-row">
                    <label>Notes</label>
                    <span className="notes-field">{entry.notes}</span>
                  </div>
                )}

                <div className="entry-actions">
                  <button onClick={() => onEdit(entry)}>✏️ แก้ไข</button>
                  <button onClick={() => onDelete(entry.id)} className="btn-danger">🗑 ลบ</button>
                  <span className="entry-date">
                    อัพเดท: {new Date(entry.updatedAt).toLocaleDateString('th-TH')}
                  </span>
                </div>
              </div>
            )}
          </div>
        )
      })}
    </div>
  )
}
```

---

## 6. Preload Security

```typescript
// src/preload/index.ts
import { contextBridge, ipcRenderer, clipboard } from 'electron'

contextBridge.exposeInMainWorld('electronAPI', {
  // Vault operations
  vaultUnlock: (password: string) => ipcRenderer.invoke('vault:unlock', password),
  vaultLock: () => ipcRenderer.send('vault:lock'),
  vaultGetEntries: () => ipcRenderer.invoke('vault:get-entries'),
  vaultAddEntry: (entry: unknown) => ipcRenderer.invoke('vault:add-entry', entry),
  vaultUpdateEntry: (id: string, data: unknown) => ipcRenderer.invoke('vault:update-entry', id, data),
  vaultDeleteEntry: (id: string) => ipcRenderer.invoke('vault:delete-entry', id),

  // Secure clipboard - copy แล้วล้างอัตโนมัติ
  copySecure: (text: string) => ipcRenderer.invoke('clipboard:copy-secure', text),
  clearClipboard: () => ipcRenderer.invoke('clipboard:clear')
})
```

---

## สรุปฟีเจอร์ความปลอดภัย

| ฟีเจอร์ | การ Implement |
|---------|--------------|
| AES-256-GCM | Node.js crypto module |
| Key Derivation | PBKDF2 SHA-512, 310,000 iterations |
| Password Verification | Timing-safe comparison |
| Auto-lock | 5 นาที ไม่ใช้งาน |
| Clipboard auto-clear | 30 วินาที |
| Password generator | Cryptographically secure random |
| Strength meter | Rule-based scoring |
| Password history | เก็บ 10 รายการล่าสุด |
| HaveIBeenPwned | k-Anonymity model check |
| Encrypted storage | ไฟล์ .vault เข้ารหัสทั้งหมด |
