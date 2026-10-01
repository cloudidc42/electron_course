# Part 74: Electron + WebAssembly

## ใช้ WebAssembly ใน Electron App

ในบทนี้เราจะเรียนวิธี load WASM modules, ใช้ WASM ใน main/renderer/worker, Emscripten output และ Rust to WASM

---

## 1. Loading WASM Modules ใน Renderer

```typescript
// src/renderer/wasm/imageProcessor.ts

// สร้าง WASM module สำหรับ image processing
// ไฟล์นี้ถูก compile จาก C/Rust

export interface ImageProcessorWASM {
  memory: WebAssembly.Memory
  exports: {
    alloc: (size: number) => number
    dealloc: (ptr: number, size: number) => void
    grayscale: (ptr: number, width: number, height: number) => void
    blur: (ptr: number, width: number, height: number, radius: number) => void
    brighten: (ptr: number, width: number, height: number, factor: number) => void
    threshold: (ptr: number, width: number, height: number, thresh: number) => void
    edge_detect: (ptr: number, width: number, height: number) => void
  }
}

let wasmInstance: ImageProcessorWASM | null = null

export async function loadImageProcessor(): Promise<ImageProcessorWASM> {
  if (wasmInstance) return wasmInstance

  // โหลด WASM module
  const wasmUrl = new URL('./image_processor.wasm', import.meta.url)
  const response = await fetch(wasmUrl.href)
  const wasmBuffer = await response.arrayBuffer()

  const { instance } = await WebAssembly.instantiate(wasmBuffer, {
    env: {
      memory: new WebAssembly.Memory({ initial: 256, maximum: 4096 }),
      abort: (msg: number, file: number, line: number, col: number) => {
        console.error(`WASM abort: ${msg} at ${file}:${line}:${col}`)
      }
    }
  })

  wasmInstance = {
    memory: (instance.exports.memory as WebAssembly.Memory),
    exports: instance.exports as unknown as ImageProcessorWASM['exports']
  }

  return wasmInstance
}

// High-level API สำหรับใช้งาน
export async function processImage(
  imageData: ImageData,
  operation: 'grayscale' | 'blur' | 'brighten' | 'threshold' | 'edge',
  params?: { radius?: number; factor?: number; threshold?: number }
): Promise<ImageData> {
  const wasm = await loadImageProcessor()
  const { memory, exports } = wasm

  const { width, height, data } = imageData
  const byteLength = data.length // width * height * 4 (RGBA)

  // Allocate WASM memory
  const ptr = exports.alloc(byteLength)

  // Copy image data into WASM memory
  new Uint8Array(memory.buffer, ptr, byteLength).set(data)

  // Process
  switch (operation) {
    case 'grayscale':
      exports.grayscale(ptr, width, height)
      break
    case 'blur':
      exports.blur(ptr, width, height, params?.radius ?? 3)
      break
    case 'brighten':
      exports.brighten(ptr, width, height, params?.factor ?? 1.2)
      break
    case 'threshold':
      exports.threshold(ptr, width, height, params?.threshold ?? 128)
      break
    case 'edge':
      exports.edge_detect(ptr, width, height)
      break
  }

  // Copy result back
  const result = new Uint8ClampedArray(memory.buffer, ptr, byteLength).slice()
  exports.dealloc(ptr, byteLength)

  return new ImageData(result, width, height)
}
```

---

## 2. C Code (Emscripten)

```c
// src/wasm/image_processor.c
#include <stdint.h>
#include <stdlib.h>
#include <math.h>

// Export declarations
#define EXPORT __attribute__((visibility("default")))

EXPORT void* alloc(int size) {
    return malloc(size);
}

EXPORT void dealloc(void* ptr, int size) {
    free(ptr);
}

// Grayscale conversion
EXPORT void grayscale(uint8_t* data, int width, int height) {
    int len = width * height * 4;
    for (int i = 0; i < len; i += 4) {
        uint8_t r = data[i];
        uint8_t g = data[i + 1];
        uint8_t b = data[i + 2];
        // Luminance formula
        uint8_t gray = (uint8_t)(0.299 * r + 0.587 * g + 0.114 * b);
        data[i] = gray;
        data[i + 1] = gray;
        data[i + 2] = gray;
    }
}

// Box blur
EXPORT void blur(uint8_t* data, int width, int height, int radius) {
    int size = width * height * 4;
    uint8_t* temp = (uint8_t*)malloc(size);
    
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            int sumR = 0, sumG = 0, sumB = 0, count = 0;
            
            for (int dy = -radius; dy <= radius; dy++) {
                for (int dx = -radius; dx <= radius; dx++) {
                    int nx = x + dx;
                    int ny = y + dy;
                    if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
                        int idx = (ny * width + nx) * 4;
                        sumR += data[idx];
                        sumG += data[idx + 1];
                        sumB += data[idx + 2];
                        count++;
                    }
                }
            }
            
            int idx = (y * width + x) * 4;
            temp[idx] = sumR / count;
            temp[idx + 1] = sumG / count;
            temp[idx + 2] = sumB / count;
            temp[idx + 3] = data[idx + 3];
        }
    }
    
    for (int i = 0; i < size; i++) data[i] = temp[i];
    free(temp);
}

// Brighten
EXPORT void brighten(uint8_t* data, int width, int height, float factor) {
    int len = width * height * 4;
    for (int i = 0; i < len; i += 4) {
        data[i] = (uint8_t)fmin(255, data[i] * factor);
        data[i+1] = (uint8_t)fmin(255, data[i+1] * factor);
        data[i+2] = (uint8_t)fmin(255, data[i+2] * factor);
    }
}

// Threshold
EXPORT void threshold(uint8_t* data, int width, int height, int thresh) {
    int len = width * height * 4;
    for (int i = 0; i < len; i += 4) {
        uint8_t gray = (uint8_t)(0.299 * data[i] + 0.587 * data[i+1] + 0.114 * data[i+2]);
        uint8_t val = gray > thresh ? 255 : 0;
        data[i] = val;
        data[i+1] = val;
        data[i+2] = val;
    }
}

// Edge detection (Sobel)
EXPORT void edge_detect(uint8_t* data, int width, int height) {
    int size = width * height * 4;
    uint8_t* temp = (uint8_t*)malloc(size);
    
    // Convert to grayscale first
    for (int i = 0; i < size; i += 4) {
        uint8_t gray = (uint8_t)(0.299 * data[i] + 0.587 * data[i+1] + 0.114 * data[i+2]);
        temp[i] = temp[i+1] = temp[i+2] = gray;
        temp[i+3] = data[i+3];
    }
    
    // Sobel operator
    int Gx[3][3] = {{-1,0,1},{-2,0,2},{-1,0,1}};
    int Gy[3][3] = {{-1,-2,-1},{0,0,0},{1,2,1}};
    
    for (int y = 1; y < height - 1; y++) {
        for (int x = 1; x < width - 1; x++) {
            int gx = 0, gy = 0;
            
            for (int dy = -1; dy <= 1; dy++) {
                for (int dx = -1; dx <= 1; dx++) {
                    int px = temp[((y+dy)*width+(x+dx))*4];
                    gx += Gx[dy+1][dx+1] * px;
                    gy += Gy[dy+1][dx+1] * px;
                }
            }
            
            int mag = (int)sqrt(gx*gx + gy*gy);
            if (mag > 255) mag = 255;
            
            int idx = (y * width + x) * 4;
            data[idx] = data[idx+1] = data[idx+2] = (uint8_t)mag;
        }
    }
    
    free(temp);
}
```

---

## 3. Rust to WASM

```rust
// src/wasm/rust_processor/src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn fibonacci(n: u32) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => {
            let mut a: u64 = 0;
            let mut b: u64 = 1;
            for _ in 2..=n {
                let c = a + b;
                a = b;
                b = c;
            }
            b
        }
    }
}

#[wasm_bindgen]
pub fn sort_numbers(arr: &mut [f64]) {
    arr.sort_by(|a, b| a.partial_cmp(b).unwrap());
}

#[wasm_bindgen]
pub struct TextProcessor {
    text: String,
}

#[wasm_bindgen]
impl TextProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new(text: String) -> TextProcessor {
        TextProcessor { text }
    }

    pub fn word_count(&self) -> usize {
        self.text.split_whitespace().count()
    }

    pub fn char_count(&self) -> usize {
        self.text.chars().count()
    }

    pub fn to_uppercase(&self) -> String {
        self.text.to_uppercase()
    }

    pub fn contains(&self, query: &str) -> bool {
        self.text.contains(query)
    }

    pub fn replace_all(&self, from: &str, to: &str) -> String {
        self.text.replace(from, to)
    }
}
```

```typescript
// src/renderer/wasm/rustProcessor.ts
import init, { fibonacci, TextProcessor } from './rust_processor/pkg'

let initialized = false

export async function initRustWasm(): Promise<void> {
  if (initialized) return
  await init()
  initialized = true
}

export async function computeFibonacci(n: number): Promise<bigint> {
  await initRustWasm()
  return fibonacci(n)
}

export async function createTextProcessor(text: string) {
  await initRustWasm()
  const processor = new TextProcessor(text)

  return {
    wordCount: () => processor.word_count(),
    charCount: () => processor.char_count(),
    toUppercase: () => processor.to_uppercase(),
    contains: (query: string) => processor.contains(query),
    replaceAll: (from: string, to: string) => processor.replace_all(from, to),
    free: () => processor.free()  // สำคัญ! ต้อง free memory
  }
}
```

---

## 4. WASM ใน Web Worker

```typescript
// src/renderer/workers/wasmWorker.ts
// Worker ที่ run WASM เพื่อไม่ block main thread

import { processImage } from '../wasm/imageProcessor'

self.onmessage = async (e: MessageEvent) => {
  const { id, type, payload } = e.data

  try {
    let result: unknown

    switch (type) {
      case 'process-image': {
        const { imageData, operation, params } = payload
        result = await processImage(imageData, operation, params)
        break
      }
      case 'fibonacci': {
        const { n } = payload
        // Heavy computation ใน worker
        let a = 0n, b = 1n
        for (let i = 0; i < n; i++) {
          [a, b] = [b, a + b]
        }
        result = String(a)
        break
      }
    }

    self.postMessage({ id, success: true, result })
  } catch (error) {
    self.postMessage({ id, success: false, error: (error as Error).message })
  }
}
```

```typescript
// src/renderer/hooks/useWasmWorker.ts
import { useRef, useCallback } from 'react'

type WorkerRequest = {
  type: string
  payload: Record<string, unknown>
}

export function useWasmWorker() {
  const workerRef = useRef<Worker | null>(null)
  const pendingRef = useRef<Map<string, {
    resolve: (v: unknown) => void
    reject: (e: Error) => void
  }>>(new Map())

  const getWorker = useCallback(() => {
    if (!workerRef.current) {
      workerRef.current = new Worker(new URL('../workers/wasmWorker.ts', import.meta.url), {
        type: 'module'
      })

      workerRef.current.onmessage = (e) => {
        const { id, success, result, error } = e.data
        const pending = pendingRef.current.get(id)
        if (pending) {
          pendingRef.current.delete(id)
          if (success) pending.resolve(result)
          else pending.reject(new Error(error))
        }
      }
    }
    return workerRef.current
  }, [])

  const invoke = useCallback(<T>(request: WorkerRequest): Promise<T> => {
    const id = `${Date.now()}-${Math.random()}`
    const worker = getWorker()

    return new Promise((resolve, reject) => {
      pendingRef.current.set(id, { resolve: resolve as (v: unknown) => void, reject })
      worker.postMessage({ id, ...request })
    })
  }, [getWorker])

  return { invoke }
}
```

---

## 5. Performance Comparison

```tsx
// src/renderer/components/WasmBenchmark.tsx
import { useState } from 'react'

export function WasmBenchmark() {
  const [results, setResults] = useState<Record<string, number>>({})
  const [running, setRunning] = useState(false)

  const runBenchmark = async () => {
    setRunning(true)
    const benchResults: Record<string, number> = {}

    // JavaScript implementation
    const t1 = performance.now()
    let sum = 0
    for (let i = 0; i < 10_000_000; i++) sum += Math.sqrt(i)
    const t2 = performance.now()
    benchResults['JavaScript (10M sqrt)'] = t2 - t1

    // WASM implementation (สมมติว่ามี)
    const t3 = performance.now()
    // const wasmResult = await wasmSqrtSum(10_000_000)
    const t4 = performance.now()
    benchResults['WASM (10M sqrt)'] = t4 - t3

    // Fibonacci
    const t5 = performance.now()
    let a = 0n, b = 1n
    for (let i = 0; i < 10000; i++) [a, b] = [b, a + b]
    const t6 = performance.now()
    benchResults['JS Fibonacci(10000)'] = t6 - t5

    setResults(benchResults)
    setRunning(false)
  }

  return (
    <div className="benchmark">
      <h3>WASM vs JavaScript Benchmark</h3>
      <button onClick={runBenchmark} disabled={running}>
        {running ? 'กำลังทดสอบ...' : 'เริ่ม Benchmark'}
      </button>
      <div className="results">
        {Object.entries(results).map(([name, ms]) => (
          <div key={name} className="result-row">
            <span>{name}</span>
            <span className="time">{ms.toFixed(2)} ms</span>
          </div>
        ))}
      </div>
    </div>
  )
}
```

---

## 6. Build Configuration

```typescript
// electron.vite.config.ts (updated สำหรับ WASM)
import { defineConfig } from 'electron-vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  renderer: {
    plugins: [react()],
    build: {
      rollupOptions: {
        output: {
          // WASM files
          assetFileNames: 'assets/[name]-[hash][extname]'
        }
      }
    },
    optimizeDeps: {
      exclude: ['@sqlite.org/sqlite-wasm']  // ตัวอย่าง WASM package
    },
    // Enable WASM
    assetsInclude: ['**/*.wasm']
  }
})
```

```bash
# Build Rust to WASM
# ติดตั้ง wasm-pack ก่อน
cargo install wasm-pack

# Build
wasm-pack build src/wasm/rust_processor --target web --out-dir src/renderer/wasm/rust_processor/pkg

# Build C ด้วย Emscripten
emcc src/wasm/image_processor.c \
  -o src/renderer/wasm/image_processor.wasm \
  -O3 \
  -s WASM=1 \
  -s EXPORTED_FUNCTIONS='["_alloc","_dealloc","_grayscale","_blur","_brighten","_threshold","_edge_detect"]' \
  -s ALLOW_MEMORY_GROWTH=1 \
  -s INITIAL_MEMORY=16777216
```

---

## สรุป

| ประเด็น | รายละเอียด |
|---------|-----------|
| C/Emscripten | Image processing แบบ pixel-level |
| Rust/wasm-bindgen | High-level API ที่ clean |
| Web Workers | ไม่ block UI thread |
| Memory management | ต้อง alloc/dealloc ใน C WASM |
| wasm-bindgen | จัดการ memory อัตโนมัติ |
| Performance | เหมาะกับ CPU-intensive tasks |
| Use cases | Image/video processing, crypto, compression |
