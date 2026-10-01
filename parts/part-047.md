# Part 047: Electron with Python
## การใช้ Python ร่วมกับ Electron

---

## 🎯 เป้าหมายของบทเรียนนี้

- รัน Python จาก Electron
- python-shell npm package
- Packaging Python ด้วย PyInstaller
- IPC กับ Python subprocess
- REST API จาก Python
- Data Science integration

---

## 1. วิธีที่ 1: child_process โดยตรง

### src/main/python-runner.ts

```typescript
import { spawn, ChildProcess } from 'child_process';
import { join } from 'path';
import { app } from 'electron';

class PythonRunner {
  private process: ChildProcess | null = null;
  
  // หา path ของ Python
  getPythonPath(): string {
    if (app.isPackaged) {
      // ใน production - ใช้ bundled Python
      const pythonExecutable = process.platform === 'win32' 
        ? 'python.exe' 
        : 'python3';
      
      return join(process.resourcesPath, 'python', pythonExecutable);
    }
    
    // ใน development - ใช้ system Python
    return process.platform === 'win32' ? 'python' : 'python3';
  }
  
  // รัน Python script
  async run(scriptPath: string, args: string[] = []): Promise<string> {
    return new Promise((resolve, reject) => {
      const python = this.getPythonPath();
      
      const proc = spawn(python, [scriptPath, ...args], {
        env: {
          ...process.env,
          PYTHONUNBUFFERED: '1', // Disable buffering
        },
      });
      
      let stdout = '';
      let stderr = '';
      
      proc.stdout.on('data', (data) => {
        stdout += data.toString();
      });
      
      proc.stderr.on('data', (data) => {
        stderr += data.toString();
      });
      
      proc.on('close', (code) => {
        if (code !== 0) {
          reject(new Error(`Python exited with code ${code}: ${stderr}`));
        } else {
          resolve(stdout.trim());
        }
      });
      
      proc.on('error', reject);
    });
  }
  
  // รัน Python code โดยตรง
  async runCode(code: string): Promise<string> {
    return new Promise((resolve, reject) => {
      const python = this.getPythonPath();
      const proc = spawn(python, ['-c', code]);
      
      let stdout = '';
      let stderr = '';
      
      proc.stdout.on('data', (data) => { stdout += data.toString(); });
      proc.stderr.on('data', (data) => { stderr += data.toString(); });
      
      proc.on('close', (code) => {
        if (code !== 0) reject(new Error(stderr));
        else resolve(stdout.trim());
      });
      
      proc.on('error', reject);
    });
  }
  
  // รัน Python ด้วย stdin/stdout communication
  startPersistentProcess(scriptPath: string): ChildProcess {
    const python = this.getPythonPath();
    
    this.process = spawn(python, [scriptPath], {
      stdio: ['pipe', 'pipe', 'pipe'],
      env: { ...process.env, PYTHONUNBUFFERED: '1' },
    });
    
    this.process.on('error', (err) => {
      console.error('Python process error:', err);
    });
    
    this.process.on('exit', (code) => {
      console.log('Python process exited:', code);
      this.process = null;
    });
    
    return this.process;
  }
  
  // ส่งคำสั่งไปยัง Python process
  sendCommand(command: object): Promise<any> {
    return new Promise((resolve, reject) => {
      if (!this.process) {
        reject(new Error('Python process not running'));
        return;
      }
      
      const json = JSON.stringify(command) + '\n';
      this.process.stdin!.write(json);
      
      // รอ response
      const handleResponse = (data: Buffer) => {
        try {
          const response = JSON.parse(data.toString().trim());
          this.process!.stdout!.off('data', handleResponse);
          resolve(response);
        } catch (e) {
          // ยังไม่ได้รับ complete JSON - รอต่อ
        }
      };
      
      this.process.stdout!.on('data', handleResponse);
      
      // Timeout
      setTimeout(() => {
        this.process!.stdout!.off('data', handleResponse);
        reject(new Error('Python command timeout'));
      }, 10000);
    });
  }
  
  stop(): void {
    this.process?.kill('SIGTERM');
    this.process = null;
  }
}

export const pythonRunner = new PythonRunner();
```

---

## 2. Python Script - Simple

### python/scripts/data_processor.py

```python
import sys
import json
import pandas as pd
import numpy as np

def process_data(data):
    """Process data and return statistics."""
    df = pd.DataFrame(data)
    
    result = {
        'count': len(df),
        'columns': list(df.columns),
        'dtypes': {col: str(dtype) for col, dtype in df.dtypes.items()},
        'stats': {}
    }
    
    # คำนวณ statistics สำหรับ numeric columns
    numeric_cols = df.select_dtypes(include=[np.number]).columns
    for col in numeric_cols:
        result['stats'][col] = {
            'mean': float(df[col].mean()),
            'std': float(df[col].std()),
            'min': float(df[col].min()),
            'max': float(df[col].max()),
            'median': float(df[col].median()),
        }
    
    return result

if __name__ == '__main__':
    # รับ input จาก command line argument
    if len(sys.argv) > 1:
        input_data = json.loads(sys.argv[1])
        result = process_data(input_data)
        print(json.dumps(result))
    else:
        print(json.dumps({'error': 'No input provided'}))
```

---

## 3. Python Script - Persistent (stdin/stdout)

### python/scripts/worker.py

```python
import sys
import json
import numpy as np
from sklearn.linear_model import LinearRegression

# ปิด stdout buffering
sys.stdout.reconfigure(line_buffering=True)

class MLWorker:
    def __init__(self):
        self.model = None
    
    def train(self, X, y):
        """Train linear regression model."""
        self.model = LinearRegression()
        X_array = np.array(X).reshape(-1, 1)
        self.model.fit(X_array, y)
        return {
            'success': True,
            'coef': float(self.model.coef_[0]),
            'intercept': float(self.model.intercept_),
            'score': float(self.model.score(X_array, y))
        }
    
    def predict(self, X):
        """Make predictions."""
        if self.model is None:
            return {'error': 'Model not trained'}
        
        X_array = np.array(X).reshape(-1, 1)
        predictions = self.model.predict(X_array).tolist()
        return {'predictions': predictions}
    
    def process_command(self, command):
        """Process incoming command."""
        action = command.get('action')
        
        if action == 'train':
            return self.train(command['X'], command['y'])
        elif action == 'predict':
            return self.predict(command['X'])
        elif action == 'ping':
            return {'pong': True}
        else:
            return {'error': f'Unknown action: {action}'}

worker = MLWorker()

# Main loop - อ่าน JSON จาก stdin
for line in sys.stdin:
    line = line.strip()
    if not line:
        continue
    
    try:
        command = json.loads(line)
        result = worker.process_command(command)
        
        # ส่ง response เป็น JSON
        print(json.dumps(result), flush=True)
    except json.JSONDecodeError as e:
        print(json.dumps({'error': f'Invalid JSON: {str(e)}'}), flush=True)
    except Exception as e:
        print(json.dumps({'error': str(e)}), flush=True)
```

---

## 4. วิธีที่ 2: python-shell

```bash
npm install python-shell
npm install --save-dev @types/python-shell
```

### src/main/python-shell-runner.ts

```typescript
import { PythonShell, Options } from 'python-shell';
import { join } from 'path';
import { app } from 'electron';

const SCRIPTS_DIR = app.isPackaged
  ? join(process.resourcesPath, 'python', 'scripts')
  : join(__dirname, '../../python/scripts');

class PythonShellRunner {
  // รัน script และรับผลลัพธ์
  async runScript(scriptName: string, args?: string[]): Promise<string[]> {
    const options: Options = {
      mode: 'text',
      pythonPath: 'python3',
      pythonOptions: ['-u'],
      scriptPath: SCRIPTS_DIR,
      args,
    };
    
    return new Promise((resolve, reject) => {
      PythonShell.run(scriptName, options, (err, results) => {
        if (err) reject(err);
        else resolve(results || []);
      });
    });
  }
  
  // รัน script แบบ JSON mode
  async runJSONScript<T>(scriptName: string, data?: any): Promise<T> {
    const options: Options = {
      mode: 'json',
      pythonPath: 'python3',
      pythonOptions: ['-u'],
      scriptPath: SCRIPTS_DIR,
      args: data ? [JSON.stringify(data)] : undefined,
    };
    
    return new Promise((resolve, reject) => {
      PythonShell.run(scriptName, options, (err, results) => {
        if (err) reject(err);
        else resolve(results?.[0] as T);
      });
    });
  }
  
  // Streaming mode
  streamScript(
    scriptName: string, 
    onMessage: (message: string) => void,
    onClose: () => void
  ): PythonShell {
    const shell = new PythonShell(scriptName, {
      mode: 'text',
      pythonPath: 'python3',
      pythonOptions: ['-u'],
      scriptPath: SCRIPTS_DIR,
    });
    
    shell.on('message', onMessage);
    shell.on('close', onClose);
    shell.on('pythonError', (err) => console.error('Python error:', err));
    
    return shell;
  }
}

export const pythonShellRunner = new PythonShellRunner();
```

---

## 5. วิธีที่ 3: REST API จาก Python

### python/api/app.py

```python
from flask import Flask, jsonify, request
from flask_cors import CORS
import pandas as pd
import json

app = Flask(__name__)
CORS(app)  # อนุญาต CORS สำหรับ Electron

@app.route('/api/process', methods=['POST'])
def process_data():
    """Process data endpoint."""
    data = request.json
    
    try:
        df = pd.DataFrame(data.get('data', []))
        result = {
            'rows': len(df),
            'columns': list(df.columns),
            'summary': df.describe().to_dict()
        }
        return jsonify({'success': True, 'result': result})
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500

@app.route('/api/analyze', methods=['POST'])
def analyze():
    """Analysis endpoint."""
    data = request.json
    text = data.get('text', '')
    
    # Simple analysis
    words = text.split()
    result = {
        'word_count': len(words),
        'char_count': len(text),
        'sentence_count': text.count('.') + text.count('!') + text.count('?'),
    }
    
    return jsonify({'success': True, 'result': result})

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'ok'})

if __name__ == '__main__':
    import sys
    port = int(sys.argv[1]) if len(sys.argv) > 1 else 5000
    app.run(port=port, debug=False)
```

### src/main/python-api.ts

```typescript
import { ChildProcess, spawn } from 'child_process';
import { app } from 'electron';
import { join } from 'path';
import * as net from 'net';

class PythonAPIServer {
  private process: ChildProcess | null = null;
  private port: number = 0;
  private baseUrl: string = '';

  // หา port ที่ว่าง
  private async findFreePort(): Promise<number> {
    return new Promise((resolve) => {
      const server = net.createServer();
      server.listen(0, () => {
        const port = (server.address() as net.AddressInfo).port;
        server.close(() => resolve(port));
      });
    });
  }

  async start(): Promise<string> {
    this.port = await this.findFreePort();
    
    const pythonPath = app.isPackaged
      ? join(process.resourcesPath, 'python', 'python3')
      : 'python3';
    
    const scriptPath = app.isPackaged
      ? join(process.resourcesPath, 'python', 'api', 'app.py')
      : join(__dirname, '../../python/api/app.py');
    
    this.process = spawn(pythonPath, [scriptPath, String(this.port)], {
      env: { ...process.env },
    });
    
    this.process.stderr!.on('data', (data) => {
      console.log('Python API:', data.toString());
    });
    
    // รอให้ server start
    await this.waitForServer();
    
    this.baseUrl = `http://localhost:${this.port}`;
    console.log(`Python API server started at ${this.baseUrl}`);
    
    return this.baseUrl;
  }

  private waitForServer(timeout = 10000): Promise<void> {
    return new Promise((resolve, reject) => {
      const start = Date.now();
      
      const check = () => {
        const client = net.createConnection({ port: this.port }, () => {
          client.destroy();
          resolve();
        });
        
        client.on('error', () => {
          if (Date.now() - start > timeout) {
            reject(new Error('Python API server timeout'));
          } else {
            setTimeout(check, 500);
          }
        });
      };
      
      setTimeout(check, 500);
    });
  }

  async call(endpoint: string, data?: any): Promise<any> {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: data ? 'POST' : 'GET',
      headers: data ? { 'Content-Type': 'application/json' } : {},
      body: data ? JSON.stringify(data) : undefined,
    });
    
    return response.json();
  }

  stop(): void {
    this.process?.kill('SIGTERM');
    this.process = null;
  }

  getBaseUrl(): string {
    return this.baseUrl;
  }
}

export const pythonAPIServer = new PythonAPIServer();
```

---

## 6. Packaging Python ด้วย PyInstaller

### package-python.sh

```bash
#!/bin/bash
# Script สำหรับ bundle Python

cd python

# สร้าง virtual environment
python3 -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate  # Windows

# ติดตั้ง dependencies
pip install -r requirements.txt
pip install pyinstaller

# Build executable
pyinstaller \
  --onedir \
  --name python-worker \
  --distpath ../resources/python \
  --add-data "scripts:scripts" \
  api/app.py

deactivate
```

### electron-builder.yml

```yaml
extraResources:
  - from: resources/python/
    to: python/
    filter:
      - "**/*"
```

---

## 7. IPC Handlers

```typescript
// src/main/ipc-handlers.ts
import { ipcMain } from 'electron';
import { pythonRunner } from './python-runner';
import { pythonAPIServer } from './python-api';

// รัน Python script
ipcMain.handle('python:runScript', async (_, { script, args }) => {
  try {
    const result = await pythonRunner.run(script, args);
    return { success: true, result };
  } catch (error: any) {
    return { success: false, error: error.message };
  }
});

// เรียก Python API
ipcMain.handle('python:apiCall', async (_, { endpoint, data }) => {
  try {
    const result = await pythonAPIServer.call(endpoint, data);
    return { success: true, result };
  } catch (error: any) {
    return { success: false, error: error.message };
  }
});
```

---

## 8. สรุป

| วิธี | ข้อดี | ข้อเสีย |
|------|-------|---------|
| child_process | ง่าย, ควบคุมได้ | รัน/หยุดทุกครั้ง |
| python-shell | API ง่ายกว่า | dependency เพิ่ม |
| Persistent process | เร็ว | ซับซ้อนกว่า |
| REST API | สะดวก, debug ง่าย | overhead มากกว่า |

### Best Practices

1. ใช้ PyInstaller bundle Python สำหรับ production
2. ตั้งค่า PYTHONUNBUFFERED=1 เพื่อ real-time output
3. Handle errors จาก Python อย่างถูกต้อง
4. ใช้ JSON เพื่อ data exchange ระหว่าง JS และ Python
5. Test บน target OS เสมอ (Python compatibility)

---

*จบ Part 047 - ต่อไป Part 048: In-App Purchases (macOS)*
