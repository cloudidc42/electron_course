# Part 040: CI/CD Pipeline สำหรับ Electron
## การสร้าง Automated Build และ Release Pipeline

---

## 🎯 เป้าหมายของบทเรียนนี้

- GitHub Actions สำหรับ Electron
- Matrix builds (Windows/macOS/Linux)
- Artifact caching และ npm caching
- Release workflow
- Draft releases
- Changelogs อัตโนมัติ

---

## 1. โครงสร้าง Workflows

```
.github/
└── workflows/
    ├── ci.yml          # Test & build on PR
    ├── release.yml     # Release เมื่อ push tag
    └── nightly.yml     # Nightly build
```

---

## 2. CI Workflow (Test & Build)

### .github/workflows/ci.yml

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'

jobs:
  # รัน tests บน Linux เร็วกว่า
  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: false

  # Build บนทุก platform
  build:
    name: Build (${{ matrix.os }})
    needs: test
    
    strategy:
      fail-fast: false
      matrix:
        include:
          - os: windows-latest
            platform: win
            artifact: '*.exe'
          - os: macos-latest
            platform: mac
            artifact: '*.dmg'
          - os: ubuntu-latest
            platform: linux
            artifact: '*.AppImage'
    
    runs-on: ${{ matrix.os }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      # Cache electron binaries
      - name: Cache Electron
        uses: actions/cache@v4
        with:
          path: ${{ github.workspace }}/.cache/electron
          key: electron-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            electron-${{ runner.os }}-
      
      - name: Install dependencies
        run: npm ci
        env:
          ELECTRON_CACHE: ${{ github.workspace }}/.cache/electron
      
      - name: Build (without signing)
        run: npm run build:${{ matrix.platform }}
        env:
          # ใน CI ไม่ sign เพื่อความเร็ว
          CSC_IDENTITY_AUTO_DISCOVERY: false
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.os }}
          path: dist/${{ matrix.artifact }}
          retention-days: 7

  # E2E tests (optional)
  e2e:
    name: E2E Tests (${{ matrix.os }})
    needs: build
    if: github.event_name == 'push'
    
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      
      - name: Install Playwright
        run: npx playwright install --with-deps
      
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          CI: true
      
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: e2e-results-${{ matrix.os }}
          path: test-results/
```

---

## 3. Release Workflow

### .github/workflows/release.yml

```yaml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'
      - 'v[0-9]+.[0-9]+.[0-9]+-*'

permissions:
  contents: write

jobs:
  # สร้าง draft release ก่อน
  create-release:
    name: Create Draft Release
    runs-on: ubuntu-latest
    outputs:
      release-id: ${{ steps.create-release.outputs.id }}
      upload-url: ${{ steps.create-release.outputs.upload_url }}
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      # สร้าง changelog อัตโนมัติ
      - name: Generate Changelog
        id: changelog
        uses: mikepenz/release-changelog-builder-action@v4
        with:
          configuration: .github/changelog-config.json
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Create Draft Release
        id: create-release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref_name }}
          release_name: Release ${{ github.ref_name }}
          body: ${{ steps.changelog.outputs.changelog }}
          draft: true
          prerelease: ${{ contains(github.ref_name, '-') }}

  # Build Windows
  build-windows:
    name: Build Windows
    needs: create-release
    runs-on: windows-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Cache Electron
        uses: actions/cache@v4
        with:
          path: ${{ github.workspace }}/.cache/electron
          key: electron-win-${{ hashFiles('package-lock.json') }}
      
      - run: npm ci
        env:
          ELECTRON_CACHE: ${{ github.workspace }}/.cache/electron
      
      # Decode certificate
      - name: Setup certificate
        shell: powershell
        run: |
          $cert = [System.Convert]::FromBase64String("${{ secrets.WIN_CSC_LINK }}")
          [System.IO.File]::WriteAllBytes("$env:TEMP\cert.pfx", $cert)
          echo "CSC_LINK=$env:TEMP\cert.pfx" >> $env:GITHUB_ENV
      
      - name: Build Windows
        run: npm run build:win
        env:
          CSC_KEY_PASSWORD: ${{ secrets.WIN_CSC_KEY_PASSWORD }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      # Upload installer
      - name: Upload NSIS installer
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ needs.create-release.outputs.upload-url }}
          asset_path: dist/MyApp Setup ${{ github.ref_name }}.exe
          asset_name: MyApp-Setup-${{ github.ref_name }}.exe
          asset_content_type: application/octet-stream
      
      # Upload portable
      - name: Upload portable
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ needs.create-release.outputs.upload-url }}
          asset_path: dist/MyApp ${{ github.ref_name }}.exe
          asset_name: MyApp-Portable-${{ github.ref_name }}.exe
          asset_content_type: application/octet-stream

  # Build macOS
  build-macos:
    name: Build macOS
    needs: create-release
    runs-on: macos-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Import certificates
        uses: apple-actions/import-codesign-certs@v3
        with:
          p12-file-base64: ${{ secrets.MACOS_CERTIFICATE }}
          p12-password: ${{ secrets.MACOS_CERTIFICATE_PWD }}
      
      - run: npm ci
      
      - name: Build macOS
        run: npm run build:mac
        env:
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Upload DMG (x64)
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ needs.create-release.outputs.upload-url }}
          asset_path: dist/MyApp-${{ github.ref_name }}-x64.dmg
          asset_name: MyApp-${{ github.ref_name }}-x64.dmg
          asset_content_type: application/octet-stream
      
      - name: Upload DMG (arm64)
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ needs.create-release.outputs.upload-url }}
          asset_path: dist/MyApp-${{ github.ref_name }}-arm64.dmg
          asset_name: MyApp-${{ github.ref_name }}-arm64.dmg
          asset_content_type: application/octet-stream

  # Build Linux
  build-linux:
    name: Build Linux
    needs: create-release
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Build Linux
        run: npm run build:linux
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Upload AppImage
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ needs.create-release.outputs.upload-url }}
          asset_path: dist/MyApp-${{ github.ref_name }}.AppImage
          asset_name: MyApp-${{ github.ref_name }}.AppImage
          asset_content_type: application/octet-stream
      
      - name: Upload deb
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ needs.create-release.outputs.upload-url }}
          asset_path: dist/myapp_${{ github.ref_name }}_amd64.deb
          asset_name: MyApp-${{ github.ref_name }}.deb
          asset_content_type: application/octet-stream

  # Publish release (เมื่อทุก build สำเร็จ)
  publish-release:
    name: Publish Release
    needs: [build-windows, build-macos, build-linux]
    runs-on: ubuntu-latest
    
    steps:
      - name: Publish Release
        uses: actions/github-script@v7
        with:
          script: |
            const releaseId = '${{ needs.create-release.outputs.release-id }}';
            await github.rest.repos.updateRelease({
              owner: context.repo.owner,
              repo: context.repo.repo,
              release_id: releaseId,
              draft: false,
            });
            console.log(`Release ${releaseId} published!`);
```

---

## 4. Changelog Configuration

### .github/changelog-config.json

```json
{
  "categories": [
    {
      "title": "## 🚀 New Features",
      "labels": ["feature", "enhancement"]
    },
    {
      "title": "## 🐛 Bug Fixes",
      "labels": ["bug", "fix"]
    },
    {
      "title": "## 📚 Documentation",
      "labels": ["documentation"]
    },
    {
      "title": "## 🔧 Maintenance",
      "labels": ["chore", "maintenance", "refactor"]
    },
    {
      "title": "## ⬆️ Dependencies",
      "labels": ["dependencies"]
    }
  ],
  "template": "#{{CHANGELOG}}\n\n## Contributors\n#{{CONTRIBUTORS}}",
  "pr_template": "- #{{TITLE}} by @#{{AUTHOR}} (#{{NUMBER}})",
  "empty_template": "- No changes",
  "label_extractor": [
    {
      "pattern": "^(feat|feature).*",
      "on_property": "branch",
      "method": "match",
      "labels": ["feature"]
    },
    {
      "pattern": "^(fix|bug).*",
      "on_property": "branch",
      "method": "match",
      "labels": ["bug"]
    }
  ],
  "transformers": [
    {
      "pattern": "\\[Closes #([0-9]+)\\]",
      "target": "[Issue #$1](https://github.com/owner/repo/issues/$1)"
    }
  ],
  "max_tags_to_fetch": 200,
  "max_pull_requests": 200,
  "max_back_track_time_days": 365
}
```

---

## 5. Nightly Build

### .github/workflows/nightly.yml

```yaml
name: Nightly Build

on:
  schedule:
    - cron: '0 0 * * *'  # Midnight UTC ทุกวัน
  workflow_dispatch:       # Manual trigger

jobs:
  nightly:
    name: Nightly Build (${{ matrix.os }})
    
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      # เปลี่ยน version เป็น nightly
      - name: Set nightly version
        shell: bash
        run: |
          DATE=$(date +%Y%m%d)
          CURRENT_VERSION=$(node -p "require('./package.json').version")
          NEW_VERSION="${CURRENT_VERSION}-nightly.${DATE}"
          npm version "$NEW_VERSION" --no-git-tag-version
      
      - name: Build
        run: npm run build
        env:
          CSC_IDENTITY_AUTO_DISCOVERY: false
      
      - uses: actions/upload-artifact@v4
        with:
          name: nightly-${{ matrix.os }}
          path: dist/
          retention-days: 3
```

---

## 6. Caching Strategy

```yaml
# Cache electron binaries
- name: Cache Electron binaries
  uses: actions/cache@v4
  with:
    path: |
      ~/.cache/electron
      ~/AppData/Local/electron/Cache
      ~/Library/Caches/electron
    key: electron-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      electron-${{ runner.os }}-

# Cache node_modules (เร็วกว่า npm ci แต่ต้องระวัง)
- name: Cache node_modules
  uses: actions/cache@v4
  id: cache-node-modules
  with:
    path: node_modules
    key: node-modules-${{ runner.os }}-${{ hashFiles('package-lock.json') }}

- name: Install dependencies
  if: steps.cache-node-modules.outputs.cache-hit != 'true'
  run: npm ci
```

---

## 7. Version Management

### scripts/version.js

```javascript
const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');

function getGitTag() {
  try {
    return execSync('git describe --exact-match --tags HEAD', { encoding: 'utf8' }).trim();
  } catch {
    return null;
  }
}

function getGitShortHash() {
  return execSync('git rev-parse --short HEAD', { encoding: 'utf8' }).trim();
}

function getBuildVersion() {
  const pkg = require('../package.json');
  const baseVersion = pkg.version;
  
  // ถ้ามี tag ตรงๆ ใช้ tag นั้น
  const tag = getGitTag();
  if (tag) {
    return tag.replace('v', '');
  }
  
  // ถ้าไม่มี tag ให้เพิ่ม commit hash
  const hash = getGitShortHash();
  const isPR = process.env.GITHUB_HEAD_REF;
  const branch = process.env.GITHUB_REF_NAME || 'dev';
  
  if (isPR) {
    return `${baseVersion}-pr.${hash}`;
  }
  
  return `${baseVersion}-${branch}.${hash}`;
}

// อัพเดท version ใน package.json
const version = getBuildVersion();
const pkgPath = path.join(__dirname, '../package.json');
const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf8'));
pkg.version = version;
fs.writeFileSync(pkgPath, JSON.stringify(pkg, null, 2) + '\n');

console.log(`Version set to: ${version}`);
```

---

## 8. Auto-Update Integration

### electron-builder.yml

```yaml
publish:
  - provider: github
    owner: yourusername
    repo: my-electron-app
    releaseType: release
```

### src/main/updater.js

```javascript
const { autoUpdater } = require('electron-updater');
const { app, BrowserWindow } = require('electron');
const log = require('electron-log');

// ตั้งค่า logger
autoUpdater.logger = log;
autoUpdater.logger.transports.file.level = 'info';

// ตั้งค่า auto-update
autoUpdater.autoDownload = false;
autoUpdater.autoInstallOnAppQuit = true;

function setupAutoUpdater(mainWindow) {
  // ตรวจสอบ update เมื่อแอพเริ่ม
  autoUpdater.checkForUpdates().catch(err => {
    log.error('Failed to check for updates:', err);
  });

  // Update available
  autoUpdater.on('update-available', (info) => {
    mainWindow.webContents.send('update-available', {
      version: info.version,
      releaseNotes: info.releaseNotes,
    });
  });

  // No update
  autoUpdater.on('update-not-available', () => {
    mainWindow.webContents.send('update-not-available');
  });

  // Download progress
  autoUpdater.on('download-progress', (progress) => {
    mainWindow.webContents.send('update-progress', {
      percent: progress.percent,
      transferred: progress.transferred,
      total: progress.total,
      bytesPerSecond: progress.bytesPerSecond,
    });
  });

  // Downloaded
  autoUpdater.on('update-downloaded', (info) => {
    mainWindow.webContents.send('update-downloaded', {
      version: info.version,
    });
  });

  // Error
  autoUpdater.on('error', (error) => {
    log.error('Auto-update error:', error);
    mainWindow.webContents.send('update-error', error.message);
  });
}

module.exports = { setupAutoUpdater, autoUpdater };
```

---

## 9. สรุป

### ขั้นตอน CI/CD สำหรับ Electron

```
1. Developer push code
      ↓
2. CI workflow รัน:
   - Lint
   - Unit tests
   - Build (without signing)
      ↓
3. Developer สร้าง tag (v1.0.0)
      ↓
4. Release workflow รัน:
   - Create draft release
   - Build Windows (signed)
   - Build macOS (signed + notarized)
   - Build Linux
      ↓
5. Upload artifacts to GitHub Release
      ↓
6. Publish release (draft → published)
      ↓
7. Users get auto-update notification
```

### Checklist

- [ ] ตั้งค่า GitHub Secrets สำหรับ code signing
- [ ] ทดสอบ CI workflow บน branch ก่อน
- [ ] ตรวจสอบ artifact sizes ว่าสมเหตุสมผล
- [ ] ทดสอบ auto-update หลังจาก release
- [ ] ตั้งค่า branch protection rules

---

*จบ Part 040 - ต่อไป Part 041: Electron Forge*
