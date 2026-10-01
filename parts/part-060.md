# Part 060: Electron Security Audit
## การตรวจสอบและเพิ่มความปลอดภัยให้ Electron App

---

## เป้าหมายของบทเรียนนี้

- ASAR packaging และ ASAR integrity
- ElectronFuses (security switches)
- ปิด dangerous features
- Sandboxing
- Network security
- Dependency auditing

---

## 1. ASAR Packaging

```typescript
// electron-builder.yml
asar: true  # บังคับเปลี่ยนเป็น ASAR (ค่า default)

# เพิ่ม ASAR integrity (Electron >= 30)
asarIntegrity: true
```

### ตรวจสอบ ASAR Integrity

```typescript
// src/main/index.ts
import { app } from 'electron';
import integrity from '@electron/asar-integrity';

// ตรวจสอบ ASAR ก่อน app ready
if (app.isPackaged) {
  integrity.verifyPackageIntegrity().catch((err) => {
    console.error('ASAR integrity check failed:', err);
    app.exit(1); // ออกทันทีถ้า ASAR ถูกแก้ไข
  });
}
```

---

## 2. ElectronFuses

```typescript
// scripts/fuses.ts
// รัน script นี้หลัง build เพื่อ flip security fuses

import { flipFuses, FuseVersion, FuseV1Options } from '@electron/fuses';
import { join } from 'path';
import { globSync } from 'glob';

async function applyFuses(): Promise<void> {
  const platform = process.platform;
  
  // หา Electron binary
  let electronPaths: string[] = [];
  
  if (platform === 'darwin') {
    electronPaths = globSync('dist/mac*/**/*.app/Contents/MacOS/*');
  } else if (platform === 'win32') {
    electronPaths = globSync('dist/win-unpacked/*.exe');
  } else {
    electronPaths = globSync('dist/linux-unpacked/my-app');
  }
  
  for (const electronPath of electronPaths) {
    console.log(`Applying fuses to: ${electronPath}`);
    
    await flipFuses(electronPath, {
      version: FuseVersion.V1,
      
      // ปิด Node.js CLI flags
      [FuseV1Options.RunAsNode]: false,
      
      // ปิด --inspect และ --inspect-brk
      [FuseV1Options.EnableNodeOptionsEnvironmentVariable]: false,
      
      // ปิด ELECTRON_RUN_AS_NODE
      [FuseV1Options.EnableEmbeddedAsarIntegrityValidation]: true,
      
      // บังคับ Context Isolation
      [FuseV1Options.OnlyLoadAppFromAsar]: true,
      
      // ปิด remote module
      [FuseV1Options.EnableCookieEncryption]: true,
      
      // ปิด NODE_OPTIONS environment variable
      [FuseV1Options.EnableNodeCliInspectArguments]: false,
      
      // Grant file:// URLs access to filesystem
      [FuseV1Options.GrantFileProtocolExtraPrivileges]: false,
    });
    
    console.log(`Fuses applied to: ${electronPath}`);
  }
}

applyFuses().catch(console.error);
```

---

## 3. Secure BrowserWindow Configuration

```typescript
// src/main/security/secure-window.ts
import { BrowserWindow, session } from 'electron';
import { join } from 'path';

export function createSecureWindow(): BrowserWindow {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    
    webPreferences: {
      // ต้องเปิดเสมอ
      contextIsolation: true,        // default true ใน Electron >= 12
      nodeIntegration: false,         // default false ใน Electron >= 5
      sandbox: true,                  // เปิด Chromium sandbox
      
      // ปิด features อันตราย
      enableRemoteModule: false,       // deprecated, ปิดเสมอ
      allowRunningInsecureContent: false, // ห้าม HTTP content ใน HTTPS
      experimentalFeatures: false,
      
      // WebSecurity
      webSecurity: true,              // อย่าปิด!
      
      // Preload script เท่านั้น
      preload: join(__dirname, '../preload/index.js'),
      
      // ปิด spell check ถ้าไม่ต้องการ
      // spellcheck: false,
      
      // Dev tools
      devTools: !app.isPackaged,
    },
  });
  
  return win;
}
```

---

## 4. Content Security Policy (CSP)

```typescript
// src/main/security/csp.ts
import { session } from 'electron';

export function setupCSP(): void {
  session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
    // สร้าง CSP header
    const csp = buildCSP({
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],       // ไม่อนุญาต inline scripts
      styleSrc: ["'self'", "'unsafe-inline'"], // บางครั้งต้องอนุญาต
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.example.com"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"],
      formAction: ["'self'"],
      baseUri: ["'self'"],
      upgradeInsecureRequests: true,
    });
    
    callback({
      responseHeaders: {
        ...details.responseHeaders,
        'Content-Security-Policy': [csp],
        'X-Content-Type-Options': ['nosniff'],
        'X-Frame-Options': ['DENY'],
        'X-XSS-Protection': ['1; mode=block'],
        'Referrer-Policy': ['strict-origin-when-cross-origin'],
      },
    });
  });
}

interface CSPOptions {
  defaultSrc?: string[];
  scriptSrc?: string[];
  styleSrc?: string[];
  imgSrc?: string[];
  connectSrc?: string[];
  fontSrc?: string[];
  objectSrc?: string[];
  mediaSrc?: string[];
  frameSrc?: string[];
  formAction?: string[];
  baseUri?: string[];
  upgradeInsecureRequests?: boolean;
}

function buildCSP(options: CSPOptions): string {
  const directives: string[] = [];
  
  const addDirective = (name: string, values?: string[]) => {
    if (values && values.length > 0) {
      directives.push(`${name} ${values.join(' ')}`);
    }
  };
  
  addDirective('default-src', options.defaultSrc);
  addDirective('script-src', options.scriptSrc);
  addDirective('style-src', options.styleSrc);
  addDirective('img-src', options.imgSrc);
  addDirective('connect-src', options.connectSrc);
  addDirective('font-src', options.fontSrc);
  addDirective('object-src', options.objectSrc);
  addDirective('media-src', options.mediaSrc);
  addDirective('frame-src', options.frameSrc);
  addDirective('form-action', options.formAction);
  addDirective('base-uri', options.baseUri);
  
  if (options.upgradeInsecureRequests) {
    directives.push('upgrade-insecure-requests');
  }
  
  return directives.join('; ');
}
```

---

## 5. Navigation Security

```typescript
// src/main/security/navigation.ts
import { app, shell } from 'electron';

// ป้องกัน navigation ไปยัง URLs ที่ไม่รู้จัก
export function setupNavigationSecurity(win: Electron.BrowserWindow): void {
  const allowedOrigins = new Set([
    'http://localhost:5173',         // dev server
    'app://localhost',               // production
    'https://api.example.com',      // allowed API
  ]);
  
  // ป้องกัน will-navigate
  win.webContents.on('will-navigate', (event, url) => {
    const origin = new URL(url).origin;
    
    if (!allowedOrigins.has(origin)) {
      event.preventDefault();
      
      // เปิด external links ใน browser
      if (url.startsWith('https://') || url.startsWith('http://')) {
        shell.openExternal(url);
      }
      
      console.warn(`Blocked navigation to: ${url}`);
    }
  });
  
  // ป้องกัน new window
  win.webContents.setWindowOpenHandler(({ url }) => {
    const origin = new URL(url).origin;
    
    if (allowedOrigins.has(origin)) {
      // อนุญาต เปิดใน electron
      return { action: 'allow' };
    }
    
    // เปิดใน browser แทน
    shell.openExternal(url);
    return { action: 'deny' };
  });
  
  // ป้องกัน webview
  app.on('web-contents-created', (_, contents) => {
    if (contents.getType() === 'webview') {
      // Block webview navigation ด้วย
      contents.on('will-navigate', (event) => {
        event.preventDefault();
      });
    }
  });
}
```

---

## 6. Permission Request Handler

```typescript
// src/main/security/permissions.ts
import { session } from 'electron';

type Permission = 
  | 'media' 
  | 'geolocation' 
  | 'notifications'
  | 'midiSysex'
  | 'pointerLock'
  | 'fullscreen'
  | 'openExternal';

interface PermissionConfig {
  allowed: Permission[];
  requireOrigin?: string[]; // เฉพาะ origins เหล่านี้เท่านั้น
}

export function setupPermissionHandler(config: PermissionConfig = { allowed: [] }): void {
  session.defaultSession.setPermissionRequestHandler(
    (webContents, permission, callback, details) => {
      const url = webContents.getURL();
      const origin = new URL(url).origin;
      
      // ตรวจสอบ origin
      if (config.requireOrigin && !config.requireOrigin.includes(origin)) {
        console.warn(`Permission "${permission}" denied for origin: ${origin}`);
        callback(false);
        return;
      }
      
      // ตรวจสอบ permission
      if (config.allowed.includes(permission as Permission)) {
        console.log(`Permission "${permission}" granted for: ${origin}`);
        callback(true);
      } else {
        console.warn(`Permission "${permission}" denied`);
        callback(false);
      }
    }
  );
  
  // ตรวจสอบ permission ที่มีอยู่แล้ว
  session.defaultSession.setPermissionCheckHandler(
    (webContents, permission) => {
      return config.allowed.includes(permission as Permission);
    }
  );
}
```

---

## 7. Dependency Security Audit

```typescript
// scripts/security-audit.ts
import { exec } from 'child_process';
import { promisify } from 'util';
import { writeFileSync } from 'fs';

const execAsync = promisify(exec);

interface AuditResult {
  vulnerabilities: {
    critical: number;
    high: number;
    moderate: number;
    low: number;
  };
  packages: Array<{
    name: string;
    severity: string;
    description: string;
    fixedIn?: string;
  }>;
}

export async function runSecurityAudit(): Promise<AuditResult> {
  try {
    const { stdout } = await execAsync('npm audit --json');
    const auditData = JSON.parse(stdout);
    
    const result: AuditResult = {
      vulnerabilities: {
        critical: auditData.metadata?.vulnerabilities?.critical || 0,
        high: auditData.metadata?.vulnerabilities?.high || 0,
        moderate: auditData.metadata?.vulnerabilities?.moderate || 0,
        low: auditData.metadata?.vulnerabilities?.low || 0,
      },
      packages: [],
    };
    
    // Parse vulnerabilities
    if (auditData.vulnerabilities) {
      for (const [name, vuln] of Object.entries(auditData.vulnerabilities as any)) {
        const v = vuln as any;
        result.packages.push({
          name,
          severity: v.severity,
          description: v.via?.[0]?.title || 'Unknown',
          fixedIn: v.fixAvailable?.version,
        });
      }
    }
    
    return result;
  } catch (error: any) {
    // npm audit exits with non-zero if vulnerabilities found
    if (error.stdout) {
      const auditData = JSON.parse(error.stdout);
      return {
        vulnerabilities: auditData.metadata?.vulnerabilities || {},
        packages: [],
      };
    }
    throw error;
  }
}

// รัน audit และแสดงผล
async function main(): Promise<void> {
  console.log('Running security audit...\n');
  
  const result = await runSecurityAudit();
  
  const { critical, high, moderate, low } = result.vulnerabilities;
  
  console.log('Vulnerability Summary:');
  console.log(`  Critical: ${critical}`);
  console.log(`  High:     ${high}`);
  console.log(`  Moderate: ${moderate}`);
  console.log(`  Low:      ${low}`);
  
  if (result.packages.length > 0) {
    console.log('\nVulnerable Packages:');
    result.packages.forEach(pkg => {
      console.log(`  ${pkg.name} (${pkg.severity}): ${pkg.description}`);
      if (pkg.fixedIn) {
        console.log(`    Fix: upgrade to ${pkg.fixedIn}`);
      }
    });
  }
  
  // Fail CI ถ้ามี critical หรือ high
  if (critical > 0 || high > 0) {
    console.error('\nFAIL: Critical or high vulnerabilities found!');
    process.exit(1);
  }
  
  console.log('\nAudit complete.');
}

if (require.main === module) {
  main();
}
```

---

## 8. Security Checklist Script

```typescript
// scripts/security-check.ts
import { readFileSync, existsSync } from 'fs';
import { join } from 'path';

interface CheckResult {
  name: string;
  status: 'pass' | 'fail' | 'warn';
  message: string;
}

async function runSecurityChecks(): Promise<void> {
  const results: CheckResult[] = [];
  
  // Check 1: package.json
  const pkg = JSON.parse(readFileSync('package.json', 'utf-8'));
  
  results.push({
    name: 'Node version',
    status: process.versions.node >= '18' ? 'pass' : 'warn',
    message: `Node ${process.versions.node}`,
  });
  
  // Check 2: Electron version
  const electronVersion = pkg.devDependencies?.electron || pkg.dependencies?.electron;
  const major = parseInt(electronVersion?.replace(/[\^~]/, '').split('.')[0] || '0');
  results.push({
    name: 'Electron version',
    status: major >= 30 ? 'pass' : major >= 25 ? 'warn' : 'fail',
    message: `Electron ${electronVersion} (recommend >= 30)`,
  });
  
  // Check 3: contextIsolation
  const mainContent = readFileSync('src/main/index.ts', 'utf-8');
  results.push({
    name: 'contextIsolation',
    status: !mainContent.includes('contextIsolation: false') ? 'pass' : 'fail',
    message: mainContent.includes('contextIsolation: false') 
      ? 'contextIsolation is disabled!' 
      : 'contextIsolation enabled',
  });
  
  // Check 4: nodeIntegration
  results.push({
    name: 'nodeIntegration',
    status: !mainContent.includes('nodeIntegration: true') ? 'pass' : 'fail',
    message: mainContent.includes('nodeIntegration: true')
      ? 'nodeIntegration is enabled!'
      : 'nodeIntegration disabled',
  });
  
  // Check 5: webSecurity
  results.push({
    name: 'webSecurity',
    status: !mainContent.includes('webSecurity: false') ? 'pass' : 'fail',
    message: mainContent.includes('webSecurity: false')
      ? 'webSecurity is disabled!'
      : 'webSecurity enabled',
  });
  
  // Check 6: Fuses script
  results.push({
    name: 'ElectronFuses',
    status: existsSync('scripts/fuses.ts') ? 'pass' : 'warn',
    message: existsSync('scripts/fuses.ts')
      ? 'Fuses script found'
      : 'No fuses script found (recommended)',
  });
  
  // Check 7: CSP
  results.push({
    name: 'Content Security Policy',
    status: mainContent.includes('Content-Security-Policy') ? 'pass' : 'warn',
    message: mainContent.includes('Content-Security-Policy')
      ? 'CSP configured'
      : 'CSP not found (recommended)',
  });
  
  // Check 8: ASAR
  const builderConfig = existsSync('electron-builder.yml')
    ? readFileSync('electron-builder.yml', 'utf-8')
    : '';
  results.push({
    name: 'ASAR packaging',
    status: !builderConfig.includes('asar: false') ? 'pass' : 'fail',
    message: builderConfig.includes('asar: false')
      ? 'ASAR is disabled!'
      : 'ASAR enabled',
  });
  
  // แสดงผล
  console.log('\n=== Electron Security Check ===\n');
  
  let hasFailures = false;
  
  results.forEach(result => {
    const icon = result.status === 'pass' ? '✓' : result.status === 'fail' ? '✗' : '!';
    console.log(`${icon} ${result.name}: ${result.message}`);
    if (result.status === 'fail') hasFailures = true;
  });
  
  const passed = results.filter(r => r.status === 'pass').length;
  const failed = results.filter(r => r.status === 'fail').length;
  const warned = results.filter(r => r.status === 'warn').length;
  
  console.log(`\nResults: ${passed} passed, ${failed} failed, ${warned} warnings`);
  
  if (hasFailures) {
    console.error('\nSecurity check failed!');
    process.exit(1);
  }
}

runSecurityChecks();
```

---

## 9. สรุป Security Best Practices

### สิ่งที่ต้องทำ (Must Do)

```typescript
// ✅ เปิด Context Isolation
contextIsolation: true

// ✅ ปิด Node Integration
nodeIntegration: false

// ✅ ปิด Remote Module
enableRemoteModule: false

// ✅ เปิด Sandbox
sandbox: true

// ✅ ไม่ปิด WebSecurity
webSecurity: true

// ✅ ใช้ ASAR packaging
// electron-builder: asar: true

// ✅ ตั้ง CSP headers
session.defaultSession.webRequest.onHeadersReceived(...)

// ✅ ตรวจสอบ navigation
win.webContents.on('will-navigate', ...)

// ✅ ใช้ ElectronFuses
flipFuses(electronPath, { ... })
```

### Security Checklist

| Item | สำคัญ |
|------|-------|
| contextIsolation: true | Critical |
| nodeIntegration: false | Critical |
| sandbox: true | High |
| webSecurity: true | Critical |
| CSP headers | High |
| Navigation guard | High |
| ElectronFuses | High |
| ASAR integrity | Medium |
| npm audit | Medium |
| Permission handler | Medium |

---

*จบ Part 060 - ครบทั้ง 25 ไฟล์แล้ว! (Part 036-060)*
