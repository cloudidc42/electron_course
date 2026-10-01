# Part 037: Testing with Jest & Playwright
## การทดสอบแอพ Electron อย่างครอบคลุม

---

## 🎯 เป้าหมายของบทเรียนนี้

- Unit testing สำหรับ Main Process ด้วย Jest
- การ mock Electron APIs ใน tests
- Integration tests
- End-to-End (E2E) tests ด้วย Playwright
- ตั้งค่า CI testing

---

## 1. โครงสร้างโปรเจกต์สำหรับ Testing

```
my-electron-app/
├── src/
│   ├── main/
│   │   ├── index.js
│   │   ├── database.js
│   │   └── fileManager.js
│   └── renderer/
│       └── app.js
├── tests/
│   ├── unit/
│   │   ├── main/
│   │   │   ├── database.test.js
│   │   │   └── fileManager.test.js
│   │   └── renderer/
│   │       └── app.test.js
│   ├── integration/
│   │   └── ipc.test.js
│   └── e2e/
│       ├── app.spec.ts
│       └── fixtures/
│           └── test-app.ts
├── jest.config.js
├── playwright.config.ts
└── package.json
```

---

## 2. ติดตั้ง Dependencies

```bash
# Jest สำหรับ unit tests
npm install --save-dev jest @types/jest ts-jest

# Playwright สำหรับ E2E
npm install --save-dev @playwright/test playwright

# Electron testing utilities
npm install --save-dev electron-mocker jest-environment-jsdom

# Coverage
npm install --save-dev @jest/coverage-provider-v8
```

---

## 3. ตั้งค่า Jest

### jest.config.js

```javascript
module.exports = {
  projects: [
    // Main process tests (Node.js environment)
    {
      displayName: 'main',
      testMatch: ['<rootDir>/tests/unit/main/**/*.test.js'],
      testEnvironment: 'node',
      moduleNameMapper: {
        // Mock electron module
        '^electron$': '<rootDir>/tests/__mocks__/electron.js',
      },
      transform: {
        '^.+\\.tsx?$': 'ts-jest',
      },
      collectCoverageFrom: [
        'src/main/**/*.{js,ts}',
        '!src/main/index.js', // exclude entry point
      ],
    },
    // Renderer process tests (jsdom environment)
    {
      displayName: 'renderer',
      testMatch: ['<rootDir>/tests/unit/renderer/**/*.test.js'],
      testEnvironment: 'jsdom',
      setupFilesAfterFramework: ['<rootDir>/tests/setup/renderer.js'],
      moduleNameMapper: {
        '\\.(css|less|scss)$': '<rootDir>/tests/__mocks__/styleMock.js',
      },
    },
  ],
  
  // Global coverage settings
  collectCoverage: false,
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

---

## 4. Mock Electron APIs

### tests/__mocks__/electron.js

```javascript
const { EventEmitter } = require('events');

// Mock app
const app = {
  getPath: jest.fn((name) => {
    const paths = {
      userData: '/tmp/test-userData',
      appData: '/tmp/test-appData',
      temp: '/tmp',
      desktop: '/tmp/test-desktop',
      documents: '/tmp/test-documents',
    };
    return paths[name] || '/tmp/test';
  }),
  getVersion: jest.fn(() => '1.0.0-test'),
  getName: jest.fn(() => 'TestApp'),
  getLocale: jest.fn(() => 'en-US'),
  isPackaged: false,
  quit: jest.fn(),
  on: jest.fn(),
  whenReady: jest.fn(() => Promise.resolve()),
};

// Mock BrowserWindow
class BrowserWindow extends EventEmitter {
  constructor(options = {}) {
    super();
    this.id = Math.random();
    this.options = options;
    this.webContents = {
      send: jest.fn(),
      on: jest.fn(),
      executeJavaScript: jest.fn(),
      session: {
        clearCache: jest.fn(),
        clearStorageData: jest.fn(),
      },
    };
  }
  
  loadFile = jest.fn(() => Promise.resolve());
  loadURL = jest.fn(() => Promise.resolve());
  show = jest.fn();
  hide = jest.fn();
  close = jest.fn();
  destroy = jest.fn();
  isDestroyed = jest.fn(() => false);
  isVisible = jest.fn(() => true);
  setTitle = jest.fn();
  getTitle = jest.fn(() => 'Test Window');
  setSize = jest.fn();
  getSize = jest.fn(() => [800, 600]);
  focus = jest.fn();
  
  static getAllWindows = jest.fn(() => []);
  static getFocusedWindow = jest.fn(() => null);
}

// Mock ipcMain
const ipcMain = new EventEmitter();
ipcMain.handle = jest.fn((channel, handler) => {
  ipcMain._handlers = ipcMain._handlers || {};
  ipcMain._handlers[channel] = handler;
});
ipcMain.removeHandler = jest.fn();

// Helper สำหรับ trigger ipcMain handler ใน tests
ipcMain.triggerHandler = async (channel, ...args) => {
  const handler = ipcMain._handlers?.[channel];
  if (!handler) throw new Error(`No handler for channel: ${channel}`);
  const event = { sender: { send: jest.fn() } };
  return handler(event, ...args);
};

// Mock ipcRenderer
const ipcRenderer = {
  invoke: jest.fn(),
  on: jest.fn(),
  once: jest.fn(),
  removeListener: jest.fn(),
  removeAllListeners: jest.fn(),
  send: jest.fn(),
};

// Mock dialog
const dialog = {
  showOpenDialog: jest.fn(() => Promise.resolve({ canceled: false, filePaths: ['/test/file.txt'] })),
  showSaveDialog: jest.fn(() => Promise.resolve({ canceled: false, filePath: '/test/output.txt' })),
  showMessageBox: jest.fn(() => Promise.resolve({ response: 0 })),
  showErrorBox: jest.fn(),
};

// Mock shell
const shell = {
  openExternal: jest.fn(() => Promise.resolve()),
  openPath: jest.fn(() => Promise.resolve('')),
  showItemInFolder: jest.fn(),
};

// Mock Menu
const Menu = {
  buildFromTemplate: jest.fn((template) => ({ template })),
  setApplicationMenu: jest.fn(),
  getApplicationMenu: jest.fn(() => null),
};

// Mock Notification
class Notification {
  constructor(options) {
    this.options = options;
  }
  show = jest.fn();
  static isSupported = jest.fn(() => true);
}

// Mock nativeTheme
const nativeTheme = {
  themeSource: 'system',
  shouldUseDarkColors: false,
  on: jest.fn(),
};

// Mock contextBridge
const contextBridge = {
  exposeInMainWorld: jest.fn(),
};

// Mock screen
const screen = {
  getPrimaryDisplay: jest.fn(() => ({
    workAreaSize: { width: 1920, height: 1080 },
    scaleFactor: 1,
    bounds: { x: 0, y: 0, width: 1920, height: 1080 },
  })),
  getAllDisplays: jest.fn(() => []),
};

// Mock Tray
class Tray {
  constructor(image) {
    this.image = image;
  }
  setToolTip = jest.fn();
  setContextMenu = jest.fn();
  on = jest.fn();
  destroy = jest.fn();
}

module.exports = {
  app,
  BrowserWindow,
  ipcMain,
  ipcRenderer,
  dialog,
  shell,
  Menu,
  Notification,
  nativeTheme,
  contextBridge,
  screen,
  Tray,
};
```

---

## 5. Unit Tests - Main Process

### src/main/fileManager.js (ฟังก์ชันที่จะ test)

```javascript
const fs = require('fs').promises;
const path = require('path');

class FileManager {
  constructor(basePath) {
    this.basePath = basePath;
  }

  // อ่านไฟล์
  async readFile(filename) {
    const filePath = path.join(this.basePath, filename);
    try {
      const content = await fs.readFile(filePath, 'utf-8');
      return { success: true, content };
    } catch (error) {
      if (error.code === 'ENOENT') {
        return { success: false, error: 'File not found' };
      }
      throw error;
    }
  }

  // เขียนไฟล์
  async writeFile(filename, content) {
    const filePath = path.join(this.basePath, filename);
    await fs.writeFile(filePath, content, 'utf-8');
    return { success: true, path: filePath };
  }

  // ลบไฟล์
  async deleteFile(filename) {
    const filePath = path.join(this.basePath, filename);
    await fs.unlink(filePath);
    return { success: true };
  }

  // รายการไฟล์
  async listFiles(extension = null) {
    const files = await fs.readdir(this.basePath);
    if (extension) {
      return files.filter(f => f.endsWith(extension));
    }
    return files;
  }

  // ตรวจสอบไฟล์มีอยู่หรือไม่
  async fileExists(filename) {
    try {
      await fs.access(path.join(this.basePath, filename));
      return true;
    } catch {
      return false;
    }
  }
}

module.exports = FileManager;
```

### tests/unit/main/fileManager.test.js

```javascript
const fs = require('fs').promises;
const path = require('path');
const FileManager = require('../../../src/main/fileManager');

// Mock fs module
jest.mock('fs', () => ({
  promises: {
    readFile: jest.fn(),
    writeFile: jest.fn(),
    unlink: jest.fn(),
    readdir: jest.fn(),
    access: jest.fn(),
  },
}));

describe('FileManager', () => {
  let fileManager;
  const basePath = '/test/files';

  beforeEach(() => {
    fileManager = new FileManager(basePath);
    jest.clearAllMocks();
  });

  describe('readFile', () => {
    test('should read file successfully', async () => {
      const content = 'Hello, World!';
      fs.readFile.mockResolvedValue(content);

      const result = await fileManager.readFile('test.txt');

      expect(result).toEqual({ success: true, content });
      expect(fs.readFile).toHaveBeenCalledWith(
        path.join(basePath, 'test.txt'),
        'utf-8'
      );
    });

    test('should return error when file not found', async () => {
      const error = new Error('ENOENT');
      error.code = 'ENOENT';
      fs.readFile.mockRejectedValue(error);

      const result = await fileManager.readFile('missing.txt');

      expect(result).toEqual({ success: false, error: 'File not found' });
    });

    test('should throw error for other file errors', async () => {
      const error = new Error('Permission denied');
      error.code = 'EACCES';
      fs.readFile.mockRejectedValue(error);

      await expect(fileManager.readFile('protected.txt')).rejects.toThrow('Permission denied');
    });
  });

  describe('writeFile', () => {
    test('should write file and return path', async () => {
      fs.writeFile.mockResolvedValue();

      const result = await fileManager.writeFile('output.txt', 'content');

      expect(result).toEqual({
        success: true,
        path: path.join(basePath, 'output.txt'),
      });
      expect(fs.writeFile).toHaveBeenCalledWith(
        path.join(basePath, 'output.txt'),
        'content',
        'utf-8'
      );
    });
  });

  describe('listFiles', () => {
    test('should list all files when no extension filter', async () => {
      const files = ['file1.txt', 'file2.js', 'file3.txt'];
      fs.readdir.mockResolvedValue(files);

      const result = await fileManager.listFiles();

      expect(result).toEqual(files);
    });

    test('should filter files by extension', async () => {
      const files = ['file1.txt', 'file2.js', 'file3.txt'];
      fs.readdir.mockResolvedValue(files);

      const result = await fileManager.listFiles('.txt');

      expect(result).toEqual(['file1.txt', 'file3.txt']);
    });
  });

  describe('fileExists', () => {
    test('should return true when file exists', async () => {
      fs.access.mockResolvedValue();

      const result = await fileManager.fileExists('existing.txt');

      expect(result).toBe(true);
    });

    test('should return false when file does not exist', async () => {
      fs.access.mockRejectedValue(new Error('ENOENT'));

      const result = await fileManager.fileExists('missing.txt');

      expect(result).toBe(false);
    });
  });
});
```

---

## 6. Unit Tests - IPC Handlers

### tests/unit/main/ipcHandlers.test.js

```javascript
const { ipcMain, dialog } = require('electron');
const FileManager = require('../../../src/main/fileManager');

// Mock dependencies
jest.mock('../../../src/main/fileManager');

// โหลด IPC handlers
let handlers;
beforeAll(() => {
  // โหลด module ที่มี ipcMain.handle calls
  handlers = require('../../../src/main/ipcHandlers');
});

describe('IPC Handlers', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('file:open', () => {
    test('should open file dialog and return content', async () => {
      dialog.showOpenDialog.mockResolvedValue({
        canceled: false,
        filePaths: ['/test/file.txt'],
      });
      
      FileManager.prototype.readFile.mockResolvedValue({
        success: true,
        content: 'file content',
      });

      const result = await ipcMain.triggerHandler('file:open');

      expect(result).toEqual({
        success: true,
        content: 'file content',
        path: '/test/file.txt',
      });
    });

    test('should return null when dialog canceled', async () => {
      dialog.showOpenDialog.mockResolvedValue({
        canceled: true,
        filePaths: [],
      });

      const result = await ipcMain.triggerHandler('file:open');

      expect(result).toBeNull();
    });
  });

  describe('file:save', () => {
    test('should save file successfully', async () => {
      FileManager.prototype.writeFile.mockResolvedValue({
        success: true,
        path: '/test/output.txt',
      });

      const result = await ipcMain.triggerHandler('file:save', {
        content: 'hello world',
        filename: 'output.txt',
      });

      expect(result).toEqual({ success: true, path: '/test/output.txt' });
    });
  });
});
```

---

## 7. E2E Tests ด้วย Playwright

### playwright.config.ts

```typescript
import { defineConfig } from '@playwright/test';
import path from 'path';

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 30000,
  
  use: {
    // Electron-specific config
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  
  projects: [
    {
      name: 'electron',
      use: {
        // custom fixture ที่จะ launch Electron
      },
    },
  ],
  
  // Reporters
  reporter: [
    ['html', { outputFolder: 'test-results/html' }],
    ['junit', { outputFile: 'test-results/junit.xml' }],
    ['line'],
  ],
  
  outputDir: 'test-results/artifacts',
});
```

### tests/e2e/fixtures/electron-fixture.ts

```typescript
import { test as base, ElectronApplication, Page, _electron as electron } from '@playwright/test';
import path from 'path';

// Custom fixture สำหรับ Electron
interface ElectronFixtures {
  electronApp: ElectronApplication;
  mainPage: Page;
}

export const test = base.extend<ElectronFixtures>({
  // Launch Electron application
  electronApp: async ({}, use) => {
    const app = await electron.launch({
      args: [path.join(__dirname, '../../../src/main/index.js')],
      env: {
        ...process.env,
        NODE_ENV: 'test',
      },
    });
    
    // รอให้แอพ ready
    await app.evaluate(({ app }) => app.whenReady());
    
    await use(app);
    
    // Cleanup
    await app.close();
  },
  
  // Get main window page
  mainPage: async ({ electronApp }, use) => {
    const page = await electronApp.firstWindow();
    
    // รอให้ page โหลดเสร็จ
    await page.waitForLoadState('domcontentloaded');
    
    await use(page);
  },
});

export { expect } from '@playwright/test';
```

### tests/e2e/app.spec.ts

```typescript
import { test, expect } from './fixtures/electron-fixture';

test.describe('Main Application', () => {
  test('should launch and show main window', async ({ mainPage }) => {
    // ตรวจสอบ title
    await expect(mainPage).toHaveTitle(/Electron/);
    
    // ตรวจสอบ elements หลัก
    await expect(mainPage.locator('h1')).toBeVisible();
  });

  test('should have working navigation', async ({ mainPage }) => {
    // คลิกปุ่มและตรวจสอบ
    const navButton = mainPage.locator('[data-testid="nav-home"]');
    await expect(navButton).toBeVisible();
    await navButton.click();
    
    // ตรวจสอบว่าไปถึงหน้าที่ถูกต้อง
    await expect(mainPage.locator('[data-testid="home-content"]')).toBeVisible();
  });

  test('should handle file operations', async ({ electronApp, mainPage }) => {
    // Mock dialog ผ่าน evaluate
    await electronApp.evaluate(({ dialog }) => {
      dialog.showOpenDialog = () => Promise.resolve({
        canceled: false,
        filePaths: ['/test/mock-file.txt'],
      });
    });
    
    // คลิกปุ่ม Open
    await mainPage.locator('[data-testid="btn-open"]').click();
    
    // ตรวจสอบว่า file dialog ถูกเรียก
    // (ตรวจสอบ state ที่เปลี่ยนไปแทน)
    await expect(mainPage.locator('[data-testid="file-path"]')).toContainText('/test/mock-file.txt');
  });

  test('should change theme', async ({ mainPage }) => {
    const themeToggle = mainPage.locator('[data-testid="theme-toggle"]');
    await themeToggle.click();
    
    // ตรวจสอบ class ที่เปลี่ยนไป
    await expect(mainPage.locator('body')).toHaveClass(/dark/);
    
    // Toggle กลับ
    await themeToggle.click();
    await expect(mainPage.locator('body')).not.toHaveClass(/dark/);
  });

  test('should display correct version', async ({ electronApp, mainPage }) => {
    const version = await electronApp.evaluate(({ app }) => app.getVersion());
    
    const versionElement = mainPage.locator('[data-testid="app-version"]');
    await expect(versionElement).toContainText(version);
  });
});

test.describe('Settings', () => {
  test('should open settings panel', async ({ mainPage }) => {
    await mainPage.locator('[data-testid="btn-settings"]').click();
    await expect(mainPage.locator('[data-testid="settings-panel"]')).toBeVisible();
  });

  test('should save and load settings', async ({ mainPage }) => {
    // เปิด settings
    await mainPage.locator('[data-testid="btn-settings"]').click();
    
    // เปลี่ยนค่า
    const checkbox = mainPage.locator('[data-testid="setting-notifications"]');
    const initialState = await checkbox.isChecked();
    await checkbox.click();
    
    // บันทึก
    await mainPage.locator('[data-testid="btn-save-settings"]').click();
    
    // ปิดและเปิดใหม่
    await mainPage.locator('[data-testid="btn-close-settings"]').click();
    await mainPage.locator('[data-testid="btn-settings"]').click();
    
    // ตรวจสอบว่าค่าถูกบันทึก
    expect(await checkbox.isChecked()).toBe(!initialState);
  });
});

test.describe('IPC Communication', () => {
  test('should receive events from main process', async ({ electronApp, mainPage }) => {
    // ส่ง event จาก main process
    await electronApp.evaluate(({ BrowserWindow }) => {
      const win = BrowserWindow.getAllWindows()[0];
      win.webContents.send('test-event', { data: 'hello' });
    });
    
    // ตรวจสอบว่า renderer รับ event
    await expect(mainPage.locator('[data-testid="event-receiver"]')).toContainText('hello');
  });
});
```

---

## 8. Integration Tests

### tests/integration/ipc.test.js

```javascript
const { app, BrowserWindow } = require('electron');

// Integration test ที่ launch จริง แต่ใน headless mode
describe('IPC Integration', () => {
  let win;

  beforeAll(async () => {
    await app.whenReady();
    
    win = new BrowserWindow({
      show: false,
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
        preload: require('path').join(__dirname, '../../src/preload/preload.js'),
      },
    });
    
    await win.loadFile(require('path').join(__dirname, '../../src/renderer/index.html'));
  });

  afterAll(() => {
    win.destroy();
  });

  test('should handle file read via IPC', async () => {
    const result = await win.webContents.executeJavaScript(`
      window.electronAPI.readFile('/tmp/test.txt')
    `);
    
    expect(result).toHaveProperty('success');
  });

  test('should send and receive custom events', async () => {
    // Setup listener ใน renderer
    await win.webContents.executeJavaScript(`
      new Promise((resolve) => {
        window.addEventListener('test:message', (e) => resolve(e.detail));
      })
    `);
    
    // ส่งจาก main
    win.webContents.send('test:message', { data: 'test' });
  });
});
```

---

## 9. Test Scripts ใน package.json

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:unit": "jest --testPathPattern=tests/unit",
    "test:integration": "jest --testPathPattern=tests/integration",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:debug": "playwright test --debug",
    "test:all": "npm run test:unit && npm run test:e2e",
    "test:ci": "npm run test:unit -- --ci && npm run test:e2e -- --reporter=junit"
  }
}
```

---

## 10. CI Testing Setup

### .github/workflows/test.yml

```yaml
name: Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit -- --ci --coverage
      - uses: codecov/codecov-action@v3

  e2e-tests:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          CI: true
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report-${{ matrix.os }}
          path: test-results/
```

---

## 11. สรุป

Testing strategy สำหรับ Electron:

1. **Unit Tests (Jest)** - ทดสอบ business logic แยกจาก Electron APIs
2. **Mock Electron APIs** - ใช้ jest.mock() สำหรับ electron module
3. **Integration Tests** - ทดสอบ IPC communication
4. **E2E Tests (Playwright)** - ทดสอบ full app flow

### Tips

- แยก business logic ออกจาก Electron-specific code เพื่อให้ test ง่ายขึ้น
- ใช้ `data-testid` attributes ใน HTML สำหรับ E2E tests
- Run E2E tests ใน headless mode สำหรับ CI
- ใช้ Playwright screenshots และ traces เมื่อ test fail

---

*จบ Part 037 - ต่อไป Part 038: electron-builder Packaging*
