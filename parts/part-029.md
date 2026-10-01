# Part 29: Child Process & Worker Threads ใน Electron

## บทนำ

สำหรับ CPU-intensive tasks เช่น image processing, data analysis, หรือ file compression การรัน code ใน main thread จะทำให้ UI ค้างได้ Electron รองรับทั้ง `child_process` และ `worker_threads` เพื่อแก้ปัญหานี้ ในบทนี้จะครอบคลุมทั้งสองวิธี รวมถึง progress reporting และ cancellation

## Child Process

### child_process.fork

```javascript
// electron/workers/forkExample.js - ไฟล์ worker
process.on('message', async (message) => {
  const { type, data, id } = message
  
  try {
    let result
    
    switch (type) {
      case 'PROCESS_DATA':
        result = await processLargeDataset(data)
        break
      case 'COMPRESS_FILES':
        result = await compressFiles(data)
        break
      default:
        throw new Error(`Unknown task type: ${type}`)
    }
    
    process.send({ id, type: 'SUCCESS', result })
  } catch (error) {
    process.send({ id, type: 'ERROR', error: error.message })
  }
})

async function processLargeDataset(data) {
  const { items } = data
  const results = []
  
  for (let i = 0; i < items.length; i++) {
    // Simulate heavy computation
    const processed = heavyComputation(items[i])
    results.push(processed)
    
    // Report progress
    if (i % 100 === 0) {
      const percent = (i / items.length) * 100
      process.send({ 
        type: 'PROGRESS', 
        percent,
        processed: i,
        total: items.length 
      })
    }
  }
  
  return results
}

function heavyComputation(item) {
  // Simulate CPU work
  let result = 0
  for (let i = 0; i < 100000; i++) {
    result += Math.sqrt(i * item)
  }
  return result
}

async function compressFiles(data) {
  const { files, outputDir } = data
  // Implementation...
  return { compressed: files.length, outputDir }
}

process.on('SIGTERM', () => {
  process.exit(0)
})
```

```javascript
// electron/services/childProcessManager.js
const { fork, spawn, exec } = require('child_process')
const path = require('path')
const { EventEmitter } = require('events')

class ChildProcessManager extends EventEmitter {
  constructor() {
    super()
    this.processes = new Map()
    this.pendingTasks = new Map()
    this.taskCounter = 0
  }

  /**
   * สร้าง fork process
   */
  createForkProcess(workerPath, options = {}) {
    const proc = fork(workerPath, [], {
      stdio: ['pipe', 'pipe', 'pipe', 'ipc'],
      env: { ...process.env, WORKER: 'true' },
      ...options
    })

    const processId = `fork-${Date.now()}`
    this.processes.set(processId, proc)

    proc.on('message', (message) => {
      const { id, type, result, error, percent, processed, total } = message
      
      if (type === 'PROGRESS') {
        this.emit('progress', { processId, percent, processed, total })
        
        // Notify task callbacks
        const task = this.pendingTasks.get(id)
        if (task?.onProgress) {
          task.onProgress({ percent, processed, total })
        }
        return
      }
      
      const task = this.pendingTasks.get(id)
      if (!task) return
      
      if (type === 'SUCCESS') {
        task.resolve(result)
      } else if (type === 'ERROR') {
        task.reject(new Error(error))
      }
      
      this.pendingTasks.delete(id)
    })

    proc.on('error', (error) => {
      console.error(`[ChildProcess ${processId}] Error:`, error)
      this.emit('error', { processId, error })
    })

    proc.on('exit', (code, signal) => {
      console.log(`[ChildProcess ${processId}] Exited:`, code, signal)
      this.processes.delete(processId)
      
      // Reject pending tasks
      this.pendingTasks.forEach((task, taskId) => {
        task.reject(new Error(`Process exited with code ${code}`))
        this.pendingTasks.delete(taskId)
      })
    })

    proc.stderr.on('data', (data) => {
      console.error(`[Worker ${processId}] STDERR:`, data.toString())
    })

    proc.stdout.on('data', (data) => {
      console.log(`[Worker ${processId}] STDOUT:`, data.toString())
    })

    return processId
  }

  /**
   * ส่ง task ไปยัง process
   */
  sendTask(processId, type, data, options = {}) {
    const proc = this.processes.get(processId)
    if (!proc) throw new Error(`Process ${processId} not found`)

    const taskId = `task-${++this.taskCounter}`

    return new Promise((resolve, reject) => {
      const timeout = options.timeout || 30000
      
      this.pendingTasks.set(taskId, {
        resolve,
        reject,
        onProgress: options.onProgress,
        processId,
        startTime: Date.now()
      })

      // Timeout
      const timer = setTimeout(() => {
        if (this.pendingTasks.has(taskId)) {
          this.pendingTasks.delete(taskId)
          reject(new Error(`Task ${taskId} timed out after ${timeout}ms`))
        }
      }, timeout)

      proc.send({ id: taskId, type, data })
        .catch((err) => {
          clearTimeout(timer)
          this.pendingTasks.delete(taskId)
          reject(err)
        })
    })
  }

  /**
   * Kill process
   */
  killProcess(processId) {
    const proc = this.processes.get(processId)
    if (proc) {
      proc.kill('SIGTERM')
      this.processes.delete(processId)
    }
  }

  /**
   * Kill ทุก processes
   */
  killAll() {
    this.processes.forEach((proc) => {
      proc.kill('SIGTERM')
    })
    this.processes.clear()
    this.pendingTasks.clear()
  }

  getActiveProcessCount() {
    return this.processes.size
  }
}

module.exports = new ChildProcessManager()
```

### child_process.spawn สำหรับ External Programs

```javascript
// electron/services/externalProcess.js
const { spawn } = require('child_process')
const { EventEmitter } = require('events')

class ExternalProcess extends EventEmitter {
  constructor() {
    super()
    this.runningProcesses = new Map()
  }

  /**
   * Run external command ด้วย streaming output
   */
  run(command, args = [], options = {}) {
    const {
      cwd = process.cwd(),
      env = process.env,
      timeout = 0,
      onData = null,
      onError = null
    } = options

    return new Promise((resolve, reject) => {
      const proc = spawn(command, args, {
        cwd,
        env,
        shell: process.platform === 'win32'
      })

      const processId = proc.pid?.toString() || Date.now().toString()
      this.runningProcesses.set(processId, proc)

      let stdout = ''
      let stderr = ''

      proc.stdout.on('data', (data) => {
        const text = data.toString()
        stdout += text
        if (onData) onData(text)
        this.emit('data', { processId, type: 'stdout', text })
      })

      proc.stderr.on('data', (data) => {
        const text = data.toString()
        stderr += text
        if (onError) onError(text)
        this.emit('data', { processId, type: 'stderr', text })
      })

      proc.on('close', (code) => {
        this.runningProcesses.delete(processId)
        
        if (code === 0) {
          resolve({ stdout, stderr, code })
        } else {
          reject(new Error(`Process exited with code ${code}\n${stderr}`))
        }
      })

      proc.on('error', (error) => {
        this.runningProcesses.delete(processId)
        reject(error)
      })

      // Timeout
      if (timeout > 0) {
        setTimeout(() => {
          if (this.runningProcesses.has(processId)) {
            proc.kill()
            reject(new Error(`Process timed out after ${timeout}ms`))
          }
        }, timeout)
      }
    })
  }

  /**
   * ตัวอย่าง: รัน Python script
   */
  async runPython(scriptPath, args = []) {
    const pythonCmd = process.platform === 'win32' ? 'python' : 'python3'
    return this.run(pythonCmd, [scriptPath, ...args])
  }

  /**
   * ตัวอย่าง: รัน FFmpeg สำหรับ video processing
   */
  async convertVideo(inputPath, outputPath, onProgress) {
    return new Promise((resolve, reject) => {
      const args = [
        '-i', inputPath,
        '-c:v', 'libx264',
        '-c:a', 'aac',
        '-progress', 'pipe:1',
        outputPath
      ]

      const proc = spawn('ffmpeg', args)
      
      // Parse FFmpeg progress output
      proc.stdout.on('data', (data) => {
        const text = data.toString()
        const timeMatch = text.match(/out_time_ms=(\d+)/)
        if (timeMatch && onProgress) {
          const ms = parseInt(timeMatch[1]) / 1000
          onProgress({ currentTime: ms })
        }
      })

      proc.on('close', (code) => {
        if (code === 0) resolve({ outputPath })
        else reject(new Error(`FFmpeg failed with code ${code}`))
      })

      proc.on('error', reject)
    })
  }

  killProcess(processId) {
    const proc = this.runningProcesses.get(processId)
    if (proc) {
      proc.kill('SIGTERM')
    }
  }
}

module.exports = new ExternalProcess()
```

## Worker Threads

### Basic Worker Thread

```javascript
// electron/workers/heavyTask.worker.js
const { workerData, parentPort, isMainThread } = require('worker_threads')

if (!isMainThread) {
  const { taskType, data } = workerData
  
  async function main() {
    try {
      let result
      
      switch (taskType) {
        case 'SORT':
          result = await performSort(data)
          break
        case 'SEARCH':
          result = await performSearch(data)
          break
        case 'HASH':
          result = await generateHashes(data)
          break
        default:
          throw new Error(`Unknown task: ${taskType}`)
      }
      
      parentPort.postMessage({ type: 'COMPLETE', result })
    } catch (error) {
      parentPort.postMessage({ type: 'ERROR', error: error.message })
    }
  }
  
  main()
}

async function performSort(data) {
  const { items, algorithm = 'quicksort' } = data
  
  // Update progress periodically
  const reportProgress = (percent) => {
    parentPort.postMessage({ type: 'PROGRESS', percent })
  }

  // Chunked sort สำหรับ large arrays
  const chunkSize = 10000
  const totalChunks = Math.ceil(items.length / chunkSize)
  const results = []

  for (let i = 0; i < totalChunks; i++) {
    const chunk = items.slice(i * chunkSize, (i + 1) * chunkSize)
    const sorted = chunk.sort((a, b) => a - b)
    results.push(...sorted)
    
    reportProgress((i + 1) / totalChunks * 100)
  }

  // Final merge sort
  return results.sort((a, b) => a - b)
}

async function performSearch(data) {
  const { items, query, fields = [] } = data
  const queryLower = query.toLowerCase()
  
  return items.filter(item => {
    if (fields.length === 0) {
      return JSON.stringify(item).toLowerCase().includes(queryLower)
    }
    return fields.some(field => {
      const value = item[field]
      return value && String(value).toLowerCase().includes(queryLower)
    })
  })
}

async function generateHashes(data) {
  const { items } = data
  const crypto = require('crypto')
  
  return items.map(item => ({
    ...item,
    hash: crypto.createHash('sha256')
      .update(JSON.stringify(item))
      .digest('hex')
  }))
}
```

### Worker Thread Manager

```javascript
// electron/services/workerThreadManager.js
const { Worker, isMainThread, workerData, parentPort, receiveMessageOnPort, MessageChannel } = require('worker_threads')
const path = require('path')
const os = require('os')

class WorkerThreadManager {
  constructor(maxWorkers = null) {
    this.maxWorkers = maxWorkers || os.cpus().length
    this.workers = []
    this.taskQueue = []
    this.pendingTasks = new Map()
    this.taskCounter = 0
    
    // สร้าง worker pool
    this.initializePool()
  }

  /**
   * สร้าง worker pool
   */
  initializePool(workerScript) {
    // Worker pool จะสร้าง on-demand แทน pre-create
    this.workerScript = workerScript || path.join(__dirname, '../workers/pool.worker.js')
  }

  /**
   * Execute task ใน worker
   */
  execute(taskType, data, options = {}) {
    return new Promise((resolve, reject) => {
      const taskId = ++this.taskCounter
      const onProgress = options.onProgress || null
      const timeout = options.timeout || 60000

      // สร้าง worker สำหรับ task นี้
      const worker = new Worker(
        options.workerScript || this.workerScript,
        {
          workerData: { taskType, data, taskId }
        }
      )

      const timeoutTimer = timeout > 0
        ? setTimeout(() => {
            worker.terminate()
            reject(new Error(`Task ${taskId} timed out after ${timeout}ms`))
          }, timeout)
        : null

      worker.on('message', (message) => {
        const { type, result, error, percent } = message

        if (type === 'PROGRESS' && onProgress) {
          onProgress({ percent, taskId })
          return
        }

        if (timeoutTimer) clearTimeout(timeoutTimer)

        if (type === 'COMPLETE') {
          resolve(result)
        } else if (type === 'ERROR') {
          reject(new Error(error))
        }

        worker.terminate()
      })

      worker.on('error', (err) => {
        if (timeoutTimer) clearTimeout(timeoutTimer)
        reject(err)
        worker.terminate()
      })

      worker.on('exit', (code) => {
        if (timeoutTimer) clearTimeout(timeoutTimer)
        if (code !== 0) {
          reject(new Error(`Worker exited with code ${code}`))
        }
      })
    })
  }

  /**
   * Execute tasks หลายๆ อัน แบบ parallel
   */
  async executeAll(tasks, options = {}) {
    const {
      maxConcurrency = this.maxWorkers,
      onProgress = null
    } = options

    const results = []
    const chunks = []
    
    for (let i = 0; i < tasks.length; i += maxConcurrency) {
      chunks.push(tasks.slice(i, i + maxConcurrency))
    }

    let completed = 0
    
    for (const chunk of chunks) {
      const chunkResults = await Promise.allSettled(
        chunk.map(task => this.execute(
          task.type, 
          task.data,
          {
            ...options,
            onProgress: (progress) => {
              if (onProgress) {
                onProgress({
                  ...progress,
                  total: tasks.length,
                  completed: completed + chunk.indexOf(task)
                })
              }
            }
          }
        ))
      )
      
      completed += chunk.length
      
      chunkResults.forEach((result, i) => {
        results.push({
          task: chunk[i],
          status: result.status,
          value: result.value,
          error: result.reason?.message
        })
      })
      
      if (onProgress) {
        onProgress({
          percent: (completed / tasks.length) * 100,
          completed,
          total: tasks.length
        })
      }
    }

    return results
  }

  /**
   * Execute ด้วย SharedArrayBuffer สำหรับ cancellation
   */
  executeWithCancellation(taskType, data) {
    const sharedBuffer = new SharedArrayBuffer(4)
    const cancelFlag = new Int32Array(sharedBuffer)
    
    let workerRef = null

    const promise = new Promise((resolve, reject) => {
      const worker = new Worker(
        path.join(__dirname, '../workers/cancellable.worker.js'),
        {
          workerData: { taskType, data, cancelBuffer: sharedBuffer }
        }
      )
      
      workerRef = worker

      worker.on('message', (message) => {
        if (message.type === 'COMPLETE') {
          resolve(message.result)
        } else if (message.type === 'CANCELLED') {
          reject(new Error('Task cancelled'))
        } else if (message.type === 'ERROR') {
          reject(new Error(message.error))
        }
        worker.terminate()
      })

      worker.on('error', reject)
    })

    const cancel = () => {
      // Set cancel flag
      Atomics.store(cancelFlag, 0, 1)
      Atomics.notify(cancelFlag, 0)
    }

    return { promise, cancel }
  }
}

module.exports = new WorkerThreadManager()
```

### Cancellable Worker

```javascript
// electron/workers/cancellable.worker.js
const { workerData, parentPort } = require('worker_threads')

const { taskType, data, cancelBuffer } = workerData
const cancelFlag = new Int32Array(cancelBuffer)

function checkCancelled() {
  return Atomics.load(cancelFlag, 0) === 1
}

async function main() {
  try {
    const result = await performTask(taskType, data)
    
    if (checkCancelled()) {
      parentPort.postMessage({ type: 'CANCELLED' })
      return
    }
    
    parentPort.postMessage({ type: 'COMPLETE', result })
  } catch (error) {
    parentPort.postMessage({ type: 'ERROR', error: error.message })
  }
}

async function performTask(type, data) {
  switch (type) {
    case 'PROCESS_FILES':
      return processFiles(data.files)
    default:
      throw new Error(`Unknown task: ${type}`)
  }
}

async function processFiles(files) {
  const results = []
  
  for (let i = 0; i < files.length; i++) {
    // Check cancellation ทุกๆ file
    if (checkCancelled()) {
      throw new Error('Cancelled')
    }
    
    const result = await processFile(files[i])
    results.push(result)
    
    parentPort.postMessage({
      type: 'PROGRESS',
      percent: ((i + 1) / files.length) * 100,
      current: i + 1,
      total: files.length
    })
  }
  
  return results
}

async function processFile(file) {
  // Simulate processing
  await new Promise(resolve => setTimeout(resolve, 10))
  return { file, processed: true }
}

main()
```

## IPC Handlers สำหรับ Tasks

```javascript
// electron/ipc/taskHandlers.js
const { ipcMain } = require('electron')
const childProcessManager = require('../services/childProcessManager')
const workerThreadManager = require('../services/workerThreadManager')

const activeTasks = new Map()

function setupTaskHandlers(mainWindow) {
  
  // Worker Thread tasks
  ipcMain.handle('task:runInWorker', async (event, { taskType, data, options = {} }) => {
    const taskId = `wt-${Date.now()}`
    
    const sendProgress = (progress) => {
      if (!mainWindow.isDestroyed()) {
        mainWindow.webContents.send(`task:progress:${taskId}`, progress)
      }
    }

    try {
      activeTasks.set(taskId, { type: 'worker', status: 'running' })
      
      const result = await workerThreadManager.execute(taskType, data, {
        ...options,
        onProgress: sendProgress,
        timeout: options.timeout || 60000
      })
      
      activeTasks.delete(taskId)
      return { success: true, taskId, result }
    } catch (error) {
      activeTasks.delete(taskId)
      return { success: false, taskId, error: error.message }
    }
  })

  // Cancellable task
  ipcMain.handle('task:runCancellable', async (event, { taskType, data }) => {
    const taskId = `ct-${Date.now()}`
    
    const { promise, cancel } = workerThreadManager.executeWithCancellation(taskType, data)
    
    activeTasks.set(taskId, { 
      type: 'cancellable', 
      status: 'running',
      cancel 
    })
    
    try {
      const result = await promise
      activeTasks.delete(taskId)
      return { success: true, taskId, result }
    } catch (error) {
      activeTasks.delete(taskId)
      if (error.message === 'Task cancelled') {
        return { success: false, taskId, cancelled: true }
      }
      return { success: false, taskId, error: error.message }
    }
  })

  // Cancel task
  ipcMain.handle('task:cancel', (event, taskId) => {
    const task = activeTasks.get(taskId)
    if (task?.cancel) {
      task.cancel()
      return { success: true }
    }
    return { success: false, error: 'Task not found or not cancellable' }
  })

  // Batch tasks
  ipcMain.handle('task:runBatch', async (event, { tasks, maxConcurrency }) => {
    const batchId = `batch-${Date.now()}`
    
    const sendProgress = (progress) => {
      if (!mainWindow.isDestroyed()) {
        mainWindow.webContents.send(`task:batchProgress:${batchId}`, progress)
      }
    }
    
    try {
      const results = await workerThreadManager.executeAll(tasks, {
        maxConcurrency,
        onProgress: sendProgress
      })
      
      return { success: true, batchId, results }
    } catch (error) {
      return { success: false, batchId, error: error.message }
    }
  })

  // Child process tasks
  ipcMain.handle('task:runInChildProcess', async (event, { workerScript, taskType, data }) => {
    const processId = childProcessManager.createForkProcess(workerScript)
    
    const taskId = `cp-${Date.now()}`
    activeTasks.set(taskId, { type: 'childProcess', processId, status: 'running' })
    
    childProcessManager.on('progress', ({ processId: pid, percent }) => {
      if (pid === processId && !mainWindow.isDestroyed()) {
        mainWindow.webContents.send(`task:progress:${taskId}`, { percent })
      }
    })
    
    try {
      const result = await childProcessManager.sendTask(processId, taskType, data, {
        onProgress: (progress) => {
          if (!mainWindow.isDestroyed()) {
            mainWindow.webContents.send(`task:progress:${taskId}`, progress)
          }
        }
      })
      
      activeTasks.delete(taskId)
      childProcessManager.killProcess(processId)
      
      return { success: true, taskId, result }
    } catch (error) {
      activeTasks.delete(taskId)
      childProcessManager.killProcess(processId)
      return { success: false, taskId, error: error.message }
    }
  })

  // Get active tasks
  ipcMain.handle('task:getActive', () => {
    const active = []
    activeTasks.forEach((task, id) => {
      active.push({ id, type: task.type, status: task.status })
    })
    return active
  })
}

module.exports = { setupTaskHandlers }
```

## React Hook สำหรับ Tasks

```jsx
// src/hooks/useBackgroundTask.js
import { useState, useCallback, useEffect, useRef } from 'react'

/**
 * Hook สำหรับรัน background tasks ด้วย progress reporting
 */
export function useBackgroundTask() {
  const [tasks, setTasks] = useState(new Map())
  const cleanupFns = useRef(new Map())

  const runTask = useCallback(async (taskType, data, options = {}) => {
    const taskId = `task-${Date.now()}`
    
    setTasks(prev => new Map(prev.set(taskId, {
      id: taskId,
      type: taskType,
      status: 'running',
      progress: 0,
      result: null,
      error: null,
      startTime: Date.now()
    })))

    // Setup progress listener
    const handleProgress = (progress) => {
      setTasks(prev => {
        const updated = new Map(prev)
        const task = updated.get(taskId)
        if (task) {
          updated.set(taskId, { ...task, progress: progress.percent || 0 })
        }
        return updated
      })
      options.onProgress?.(progress)
    }

    if (window.electronAPI?.on) {
      const cleanup = window.electronAPI.on(
        `task:progress:${taskId}`, 
        handleProgress
      )
      cleanupFns.current.set(taskId, cleanup)
    }

    try {
      let response
      
      if (options.cancellable) {
        response = await window.electronAPI.invoke('task:runCancellable', {
          taskType, data
        })
      } else {
        response = await window.electronAPI.invoke('task:runInWorker', {
          taskType, data, options
        })
      }

      setTasks(prev => {
        const updated = new Map(prev)
        const task = updated.get(taskId)
        if (task) {
          updated.set(taskId, {
            ...task,
            status: response.success ? 'completed' : 'failed',
            progress: 100,
            result: response.result,
            error: response.error,
            endTime: Date.now()
          })
        }
        return updated
      })

      return response
    } finally {
      // Cleanup listener
      const cleanup = cleanupFns.current.get(taskId)
      cleanup?.()
      cleanupFns.current.delete(taskId)
    }
  }, [])

  const cancelTask = useCallback(async (taskId) => {
    const response = await window.electronAPI?.invoke('task:cancel', taskId)
    
    if (response?.success) {
      setTasks(prev => {
        const updated = new Map(prev)
        const task = updated.get(taskId)
        if (task) {
          updated.set(taskId, { ...task, status: 'cancelled' })
        }
        return updated
      })
    }
    
    return response
  }, [])

  const clearTask = useCallback((taskId) => {
    setTasks(prev => {
      const updated = new Map(prev)
      updated.delete(taskId)
      return updated
    })
  }, [])

  const clearCompletedTasks = useCallback(() => {
    setTasks(prev => {
      const updated = new Map(prev)
      updated.forEach((task, id) => {
        if (['completed', 'failed', 'cancelled'].includes(task.status)) {
          updated.delete(id)
        }
      })
      return updated
    })
  }, [])

  useEffect(() => {
    return () => {
      cleanupFns.current.forEach(fn => fn?.())
    }
  }, [])

  return {
    tasks: [...tasks.values()],
    runTask,
    cancelTask,
    clearTask,
    clearCompletedTasks
  }
}
```

## Task Manager UI Component

```jsx
// src/components/TaskManager.jsx
import { useBackgroundTask } from '../hooks/useBackgroundTask'

export default function TaskManager() {
  const { tasks, runTask, cancelTask, clearCompletedTasks } = useBackgroundTask()

  const handleRunHeavyTask = async () => {
    const data = {
      items: Array.from({ length: 100000 }, (_, i) => i)
    }

    const result = await runTask('SORT', data, {
      onProgress: (p) => console.log('Progress:', p.percent)
    })

    if (result.success) {
      alert(`เสร็จแล้ว! ข้อมูล ${result.result?.length} รายการ`)
    }
  }

  const handleRunCancellable = async () => {
    const data = { files: Array.from({ length: 50 }, (_, i) => `file-${i}.txt`) }
    const result = await runTask('PROCESS_FILES', data, { cancellable: true })
    
    if (result.cancelled) {
      alert('Task ถูกยกเลิก')
    }
  }

  const getStatusColor = (status) => {
    const colors = {
      running: '#3b82f6',
      completed: '#10b981',
      failed: '#ef4444',
      cancelled: '#f59e0b'
    }
    return colors[status] || '#6b7280'
  }

  return (
    <div className="task-manager">
      <div className="task-header">
        <h2>Background Tasks</h2>
        <div className="task-actions">
          <button onClick={handleRunHeavyTask} className="btn-primary">
            🚀 Run Heavy Task
          </button>
          <button onClick={handleRunCancellable} className="btn-secondary">
            ⏱️ Run Cancellable
          </button>
          <button onClick={clearCompletedTasks} className="btn-outline">
            🗑️ Clear Done
          </button>
        </div>
      </div>

      {tasks.length === 0 ? (
        <div className="empty-tasks">
          <p>ไม่มี tasks ที่กำลังทำงาน</p>
        </div>
      ) : (
        <div className="task-list">
          {tasks.map(task => (
            <div key={task.id} className="task-item">
              <div className="task-info">
                <span className="task-type">{task.type}</span>
                <span className="task-status" style={{ color: getStatusColor(task.status) }}>
                  {task.status}
                </span>
                {task.startTime && (
                  <span className="task-duration">
                    {task.endTime 
                      ? `${((task.endTime - task.startTime) / 1000).toFixed(1)}s`
                      : `${((Date.now() - task.startTime) / 1000).toFixed(0)}s`
                    }
                  </span>
                )}
              </div>

              {task.status === 'running' && (
                <>
                  <div className="task-progress">
                    <div 
                      className="task-progress-bar"
                      style={{ width: `${task.progress}%` }}
                    />
                  </div>
                  <span className="task-percent">{task.progress.toFixed(1)}%</span>
                  <button 
                    onClick={() => cancelTask(task.id)}
                    className="btn-cancel"
                  >
                    ✕ ยกเลิก
                  </button>
                </>
              )}

              {task.error && (
                <p className="task-error">{task.error}</p>
              )}
            </div>
          ))}
        </div>
      )}
    </div>
  )
}
```

## สรุป

### เมื่อไหรใช้อะไร

| วิธี | เหมาะสำหรับ | ข้อดี | ข้อเสีย |
|------|-------------|-------|---------|
| Worker Threads | CPU-intensive JS | Share memory, เร็ว | เฉพาะ JS/WASM |
| child_process.fork | Node.js scripts | Isolated, reliable | ช้ากว่า |
| child_process.spawn | External programs | รันโปรแกรมอื่นได้ | Overhead สูง |

### Best Practices

1. **ใช้ Worker Threads** สำหรับ CPU-intensive JavaScript tasks
2. **ใช้ child_process.fork** สำหรับ Node.js scripts แยก
3. **ใช้ child_process.spawn** สำหรับ external programs
4. **เสมอมี timeout** เพื่อป้องกัน hanging tasks
5. **Implement cancellation** สำหรับ long-running tasks
6. **Report progress** เพื่อให้ user ทราบสถานะ
7. **Cleanup workers** เมื่อไม่ใช้งานแล้ว
