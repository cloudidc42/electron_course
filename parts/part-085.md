# Part 85: End-to-End Encryption

## ระบบเข้ารหัสแบบ End-to-End ใน Electron App

ในบทนี้เราจะเรียน libsodium, Argon2 password hashing, key exchange, encrypted messaging และ secure storage

---

## 1. ติดตั้งและ Setup libsodium

```bash
npm install libsodium-wrappers libsodium-wrappers-sumo
npm install -D @types/libsodium-wrappers
```

```typescript
// src/renderer/crypto/sodium.ts
import _sodium from 'libsodium-wrappers'

let ready = false
let sodium: typeof _sodium

export async function getSodium(): Promise<typeof _sodium> {
  if (!ready) {
    await _sodium.ready
    sodium = _sodium
    ready = true
  }
  return sodium
}
```

---

## 2. Key Generation และ Key Exchange

```typescript
// src/renderer/crypto/keyExchange.ts
import { getSodium } from './sodium'

export interface KeyPair {
  publicKey: Uint8Array
  privateKey: Uint8Array
}

export interface EncryptedMessage {
  ciphertext: Uint8Array
  nonce: Uint8Array
  senderPublicKey: Uint8Array
}

// สร้าง asymmetric key pair สำหรับ X25519 key exchange
export async function generateKeyPair(): Promise<KeyPair> {
  const sodium = await getSodium()
  const kp = sodium.crypto_box_keypair()
  return {
    publicKey: kp.publicKey,
    privateKey: kp.privateKey
  }
}

// Encrypt message สำหรับ recipient โดยใช้ public key ของเขา
export async function encryptMessage(
  message: string | Uint8Array,
  recipientPublicKey: Uint8Array,
  senderPrivateKey: Uint8Array,
  senderPublicKey: Uint8Array
): Promise<EncryptedMessage> {
  const sodium = await getSodium()

  const msgBytes = typeof message === 'string'
    ? sodium.from_string(message)
    : message

  const nonce = sodium.randombytes_buf(sodium.crypto_box_NONCEBYTES)

  const ciphertext = sodium.crypto_box_easy(
    msgBytes,
    nonce,
    recipientPublicKey,
    senderPrivateKey
  )

  return { ciphertext, nonce, senderPublicKey }
}

// Decrypt message
export async function decryptMessage(
  encrypted: EncryptedMessage,
  recipientPrivateKey: Uint8Array
): Promise<string> {
  const sodium = await getSodium()

  const plaintext = sodium.crypto_box_open_easy(
    encrypted.ciphertext,
    encrypted.nonce,
    encrypted.senderPublicKey,
    recipientPrivateKey
  )

  return sodium.to_string(plaintext)
}

// Encode/decode keys as base64 สำหรับ storage/transmission
export async function encodeKey(key: Uint8Array): Promise<string> {
  const sodium = await getSodium()
  return sodium.to_base64(key, sodium.base64_variants.URLSAFE_NO_PADDING)
}

export async function decodeKey(encoded: string): Promise<Uint8Array> {
  const sodium = await getSodium()
  return sodium.from_base64(encoded, sodium.base64_variants.URLSAFE_NO_PADDING)
}
```

---

## 3. Argon2 Password Hashing

```typescript
// src/main/argon2.ts
// ใน main process ใช้ argon2 native module

import * as argon2 from 'argon2'
import { ipcMain } from 'electron'
import * as crypto from 'crypto'

export async function hashPassword(password: string): Promise<string> {
  return argon2.hash(password, {
    type: argon2.argon2id,      // argon2id (ปลอดภัยสุด)
    memoryCost: 65536,           // 64 MB
    timeCost: 3,                 // 3 iterations
    parallelism: 4,              // 4 threads
    saltLength: 32,
    hashLength: 32
  })
}

export async function verifyPassword(hash: string, password: string): Promise<boolean> {
  try {
    return await argon2.verify(hash, password)
  } catch {
    return false
  }
}

// Derive encryption key จาก password
export async function deriveKey(password: string, salt: Buffer): Promise<Buffer> {
  return argon2.hash(password, {
    type: argon2.argon2id,
    memoryCost: 65536,
    timeCost: 3,
    parallelism: 4,
    salt,
    hashLength: 32,
    raw: true  // return raw bytes
  }) as unknown as Buffer
}

export function registerArgon2Handlers(): void {
  ipcMain.handle('crypto:hash-password', async (_, { password }: { password: string }) => {
    return hashPassword(password)
  })

  ipcMain.handle('crypto:verify-password', async (_, { hash, password }: {
    hash: string; password: string
  }) => {
    return verifyPassword(hash, password)
  })

  ipcMain.handle('crypto:derive-key', async (_, { password, salt }: {
    password: string; salt: string
  }) => {
    const saltBuf = Buffer.from(salt, 'base64')
    const key = await deriveKey(password, saltBuf)
    return key.toString('base64')
  })
}
```

---

## 4. Symmetric Encryption (XChaCha20-Poly1305)

```typescript
// src/renderer/crypto/symmetricCrypto.ts
import { getSodium } from './sodium'

export interface EncryptedData {
  ciphertext: string   // base64
  nonce: string        // base64
}

// Encrypt data ด้วย secret key
export async function encryptData(
  data: string | object,
  keyBase64: string
): Promise<EncryptedData> {
  const sodium = await getSodium()

  const key = sodium.from_base64(keyBase64, sodium.base64_variants.URLSAFE_NO_PADDING)
  const message = typeof data === 'string' ? sodium.from_string(data) : sodium.from_string(JSON.stringify(data))
  const nonce = sodium.randombytes_buf(sodium.crypto_secretbox_NONCEBYTES)

  const ciphertext = sodium.crypto_secretbox_easy(message, nonce, key)

  return {
    ciphertext: sodium.to_base64(ciphertext, sodium.base64_variants.URLSAFE_NO_PADDING),
    nonce: sodium.to_base64(nonce, sodium.base64_variants.URLSAFE_NO_PADDING)
  }
}

// Decrypt data
export async function decryptData(
  encrypted: EncryptedData,
  keyBase64: string
): Promise<string> {
  const sodium = await getSodium()

  const key = sodium.from_base64(keyBase64, sodium.base64_variants.URLSAFE_NO_PADDING)
  const ciphertext = sodium.from_base64(encrypted.ciphertext, sodium.base64_variants.URLSAFE_NO_PADDING)
  const nonce = sodium.from_base64(encrypted.nonce, sodium.base64_variants.URLSAFE_NO_PADDING)

  const plaintext = sodium.crypto_secretbox_open_easy(ciphertext, nonce, key)

  return sodium.to_string(plaintext)
}

// Encrypt ไฟล์
export async function encryptFile(
  fileBuffer: ArrayBuffer,
  keyBase64: string
): Promise<EncryptedData> {
  const sodium = await getSodium()
  const key = sodium.from_base64(keyBase64, sodium.base64_variants.URLSAFE_NO_PADDING)
  const data = new Uint8Array(fileBuffer)
  const nonce = sodium.randombytes_buf(sodium.crypto_secretbox_NONCEBYTES)
  const ciphertext = sodium.crypto_secretbox_easy(data, nonce, key)

  return {
    ciphertext: sodium.to_base64(ciphertext, sodium.base64_variants.URLSAFE_NO_PADDING),
    nonce: sodium.to_base64(nonce, sodium.base64_variants.URLSAFE_NO_PADDING)
  }
}
```

---

## 5. Secure Vault Storage

```typescript
// src/main/secureVault.ts
import * as crypto from 'crypto'
import * as fs from 'fs/promises'
import * as path from 'path'
import { app, ipcMain } from 'electron'
import * as argon2 from 'argon2'

interface VaultEntry {
  id: string
  title: string
  data: string    // encrypted JSON
  createdAt: number
  updatedAt: number
}

interface VaultFile {
  version: number
  salt: string
  entries: VaultEntry[]
}

export class SecureVault {
  private vaultPath: string
  private key: Buffer | null = null
  private vault: VaultFile | null = null

  constructor() {
    this.vaultPath = path.join(app.getPath('userData'), 'vault.enc')
  }

  async unlock(masterPassword: string): Promise<boolean> {
    try {
      const vaultData = await fs.readFile(this.vaultPath, 'utf-8')
      const vault: VaultFile = JSON.parse(vaultData)

      const salt = Buffer.from(vault.salt, 'base64')
      this.key = await argon2.hash(masterPassword, {
        type: argon2.argon2id,
        memoryCost: 65536,
        timeCost: 3,
        parallelism: 4,
        salt,
        hashLength: 32,
        raw: true
      }) as unknown as Buffer

      this.vault = vault
      return true
    } catch {
      // vault ไม่มีหรือ password ผิด
      return false
    }
  }

  async create(masterPassword: string): Promise<void> {
    const salt = crypto.randomBytes(32)
    this.key = await argon2.hash(masterPassword, {
      type: argon2.argon2id,
      memoryCost: 65536,
      timeCost: 3,
      parallelism: 4,
      salt,
      hashLength: 32,
      raw: true
    }) as unknown as Buffer

    this.vault = {
      version: 1,
      salt: salt.toString('base64'),
      entries: []
    }

    await this.save()
  }

  lock(): void {
    this.key = null
    this.vault = null
  }

  async addEntry(title: string, data: object): Promise<string> {
    if (!this.key || !this.vault) throw new Error('Vault locked')

    const id = crypto.randomUUID()
    const encrypted = this.encrypt(JSON.stringify(data))
    const entry: VaultEntry = {
      id,
      title,
      data: encrypted,
      createdAt: Date.now(),
      updatedAt: Date.now()
    }

    this.vault.entries.push(entry)
    await this.save()
    return id
  }

  async getEntry(id: string): Promise<object | null> {
    if (!this.key || !this.vault) throw new Error('Vault locked')

    const entry = this.vault.entries.find(e => e.id === id)
    if (!entry) return null

    const decrypted = this.decrypt(entry.data)
    return JSON.parse(decrypted)
  }

  private encrypt(plaintext: string): string {
    if (!this.key) throw new Error('No key')
    const iv = crypto.randomBytes(12)
    const cipher = crypto.createCipheriv('aes-256-gcm', this.key, iv)
    const encrypted = Buffer.concat([cipher.update(plaintext, 'utf-8'), cipher.final()])
    const tag = cipher.getAuthTag()
    return Buffer.concat([iv, tag, encrypted]).toString('base64')
  }

  private decrypt(encoded: string): string {
    if (!this.key) throw new Error('No key')
    const buf = Buffer.from(encoded, 'base64')
    const iv = buf.subarray(0, 12)
    const tag = buf.subarray(12, 28)
    const data = buf.subarray(28)
    const decipher = crypto.createDecipheriv('aes-256-gcm', this.key, iv)
    decipher.setAuthTag(tag)
    return decipher.update(data).toString('utf-8') + decipher.final('utf-8')
  }

  private async save(): Promise<void> {
    if (!this.vault) return
    await fs.writeFile(this.vaultPath, JSON.stringify(this.vault), 'utf-8')
  }
}
```

---

## สรุป

| เทคโนโลยี | Algorithm | ใช้กับ |
|----------|---------|-------|
| libsodium | X25519 + XSalsa20 | Key exchange + asymmetric |
| libsodium | XChaCha20-Poly1305 | Symmetric encryption |
| Argon2id | Memory-hard KDF | Password hashing |
| AES-256-GCM | AEAD | Node.js symmetric |
| Node.js crypto | PBKDF2/random | Key derivation |
| Vault pattern | Encrypted JSON | Secure storage |
