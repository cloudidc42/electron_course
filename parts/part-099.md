# Part 99: Deploying & Releasing

## กระบวนการ Deploy และ Release Electron App แบบมืออาชีพ

ในบทนี้เราจะเรียน semantic versioning, GitHub Releases automation, release notes generation, code signing และ distribution pipeline ครบวงจร

---

## 1. Semantic Versioning Strategy

```
Version Format:  MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]

MAJOR = breaking changes
MINOR = new features (backward compatible)
PATCH = bug fixes

ตัวอย่าง:
  1.0.0        → initial release
  1.1.0        → new feature
  1.1.1        → bug fix
  2.0.0        → breaking change
  2.0.0-beta.1 → pre-release beta
  2.0.0-rc.1   → release candidate
```

```json
// package.json
{
  "version": "1.5.2",
  "scripts": {
    "version:patch":  "npm version patch --no-git-tag-version",
    "version:minor":  "npm version minor --no-git-tag-version",
    "version:major":  "npm version major --no-git-tag-version",
    "version:beta":   "npm version prerelease --preid=beta --no-git-tag-version",
    "version:rc":     "npm version prerelease --preid=rc --no-git-tag-version",
    "release":        "ts-node scripts/release.ts"
  }
}
```

---

## 2. Changelog Management

```typescript
// scripts/changelog.ts
// สร้าง CHANGELOG.md โดยอัตโนมัติจาก git commits

import { execSync } from 'child_process'
import * as fs from 'fs'
import * as path from 'path'

interface CommitInfo {
  hash: string
  type: string
  scope?: string
  message: string
  breaking: boolean
  author: string
}

// Conventional Commits parser
function parseCommit(line: string): CommitInfo | null {
  // format: <hash> <type>(<scope>)!: <message>
  const match = line.match(/^([a-f0-9]+) (feat|fix|docs|style|refactor|perf|test|chore|build|ci|revert)(\([^)]+\))?(!)?: (.+)$/)
  if (!match) return null

  return {
    hash:     match[1].slice(0, 7),
    type:     match[2],
    scope:    match[3]?.slice(1, -1),
    message:  match[5],
    breaking: match[4] === '!',
    author:   ''
  }
}

function getCommitsSince(tag: string): CommitInfo[] {
  const range = tag ? `${tag}..HEAD` : 'HEAD'
  const log = execSync(
    `git log ${range} --format="%H %s" --no-merges`,
    { encoding: 'utf-8' }
  ).trim()

  if (!log) return []

  return log.split('\n')
    .map(parseCommit)
    .filter((c): c is CommitInfo => c !== null)
}

function getLatestTag(): string {
  try {
    return execSync('git describe --tags --abbrev=0', { encoding: 'utf-8' }).trim()
  } catch {
    return ''
  }
}

function groupCommits(commits: CommitInfo[]): Record<string, CommitInfo[]> {
  const groups: Record<string, CommitInfo[]> = {
    'Breaking Changes': [],
    'Features':         [],
    'Bug Fixes':        [],
    'Performance':      [],
    'Refactoring':      [],
    'Other':            []
  }

  for (const commit of commits) {
    if (commit.breaking) {
      groups['Breaking Changes'].push(commit)
    } else if (commit.type === 'feat') {
      groups['Features'].push(commit)
    } else if (commit.type === 'fix') {
      groups['Bug Fixes'].push(commit)
    } else if (commit.type === 'perf') {
      groups['Performance'].push(commit)
    } else if (commit.type === 'refactor') {
      groups['Refactoring'].push(commit)
    } else if (!['docs', 'style', 'test', 'ci'].includes(commit.type)) {
      groups['Other'].push(commit)
    }
  }

  return groups
}

function formatCommit(commit: CommitInfo, repoUrl: string): string {
  const scope = commit.scope ? `**${commit.scope}**: ` : ''
  const link = `[\`${commit.hash}\`](${repoUrl}/commit/${commit.hash})`
  return `- ${scope}${commit.message} ${link}`
}

export function generateChangelog(version: string, repoUrl: string): string {
  const latestTag = getLatestTag()
  const commits = getCommitsSince(latestTag)
  const groups = groupCommits(commits)
  const date = new Date().toISOString().split('T')[0]

  let changelog = `## [${version}] - ${date}\n\n`

  for (const [section, sectionCommits] of Object.entries(groups)) {
    if (sectionCommits.length === 0) continue
    changelog += `### ${section}\n\n`
    for (const commit of sectionCommits) {
      changelog += formatCommit(commit, repoUrl) + '\n'
    }
    changelog += '\n'
  }

  return changelog
}

export function updateChangelogFile(newEntry: string): void {
  const changelogPath = path.join(process.cwd(), 'CHANGELOG.md')
  const existing = fs.existsSync(changelogPath)
    ? fs.readFileSync(changelogPath, 'utf-8')
    : '# Changelog\n\nAll notable changes to this project will be documented in this file.\n\n'

  const insertAt = existing.indexOf('## [')
  const updated = insertAt >= 0
    ? existing.slice(0, insertAt) + newEntry + existing.slice(insertAt)
    : existing + newEntry

  fs.writeFileSync(changelogPath, updated, 'utf-8')
  console.log('CHANGELOG.md updated')
}
```

---

## 3. Release Script

```typescript
// scripts/release.ts
// สร้าง release ครบขั้นตอน

import { execSync } from 'child_process'
import * as fs from 'fs'
import * as readline from 'readline'
import { generateChangelog, updateChangelogFile } from './changelog'

type ReleaseType = 'patch' | 'minor' | 'major' | 'beta' | 'rc'

async function promptUser(question: string): Promise<string> {
  const rl = readline.createInterface({ input: process.stdin, output: process.stdout })
  return new Promise(resolve => {
    rl.question(question, answer => {
      rl.close()
      resolve(answer.trim())
    })
  })
}

function getCurrentVersion(): string {
  const pkg = JSON.parse(fs.readFileSync('package.json', 'utf-8'))
  return pkg.version
}

function bumpVersion(type: ReleaseType): string {
  const preid = type === 'beta' ? 'beta' : type === 'rc' ? 'rc' : undefined
  const npmType = ['beta', 'rc'].includes(type) ? 'prerelease' : type
  const preidArg = preid ? `--preid=${preid}` : ''

  execSync(`npm version ${npmType} ${preidArg} --no-git-tag-version`, { stdio: 'inherit' })
  return getCurrentVersion()
}

function runChecks(): void {
  console.log('\n--- Running pre-release checks ---')

  // Type check
  execSync('npx tsc --noEmit', { stdio: 'inherit' })
  console.log('TypeScript: OK')

  // Tests
  execSync('npm run test', { stdio: 'inherit' })
  console.log('Tests: OK')

  // Lint
  execSync('npm run lint', { stdio: 'inherit' })
  console.log('Lint: OK')
}

async function main(): Promise<void> {
  const currentVersion = getCurrentVersion()
  console.log(`\nCurrent version: ${currentVersion}`)

  const typeStr = await promptUser('Release type (patch/minor/major/beta/rc): ')
  const type = typeStr as ReleaseType

  if (!['patch', 'minor', 'major', 'beta', 'rc'].includes(type)) {
    console.error('Invalid release type')
    process.exit(1)
  }

  // Run checks
  try {
    runChecks()
  } catch {
    console.error('\nPre-release checks failed!')
    process.exit(1)
  }

  // Bump version
  const newVersion = bumpVersion(type)
  console.log(`\nNew version: ${newVersion}`)

  // Generate changelog
  const repoUrl = 'https://github.com/example/my-app'
  const changelogEntry = generateChangelog(newVersion, repoUrl)
  console.log('\nChangelog entry:')
  console.log(changelogEntry)

  const confirm = await promptUser('\nProceed with release? (y/N): ')
  if (confirm.toLowerCase() !== 'y') {
    console.log('Release cancelled')
    process.exit(0)
  }

  // Update changelog file
  updateChangelogFile(changelogEntry)

  // Git commit and tag
  execSync('git add package.json CHANGELOG.md', { stdio: 'inherit' })
  execSync(`git commit -m "chore: release v${newVersion}"`, { stdio: 'inherit' })
  execSync(`git tag -a "v${newVersion}" -m "Release v${newVersion}"`, { stdio: 'inherit' })

  console.log(`\nRelease v${newVersion} committed and tagged!`)
  console.log('Run `git push && git push --tags` to trigger CI/CD')
}

main().catch(err => {
  console.error(err)
  process.exit(1)
})
```

---

## 4. GitHub Actions Release Pipeline

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  # Job 1: Build artifacts สำหรับทุก platform
  build:
    strategy:
      matrix:
        include:
          - os: windows-latest
            platform: win
            ext: exe
          - os: macos-latest
            platform: mac
            ext: dmg
          - os: ubuntu-latest
            platform: linux
            ext: AppImage

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      # macOS code signing
      - name: Setup macOS signing
        if: matrix.platform == 'mac'
        env:
          CERTIFICATE_P12: ${{ secrets.MACOS_CERTIFICATE_P12 }}
          CERTIFICATE_PASSWORD: ${{ secrets.MACOS_CERTIFICATE_PASSWORD }}
        run: |
          echo "$CERTIFICATE_P12" | base64 --decode > certificate.p12
          security create-keychain -p "" build.keychain
          security default-keychain -s build.keychain
          security unlock-keychain -p "" build.keychain
          security import certificate.p12 -k build.keychain -P "$CERTIFICATE_PASSWORD" -T /usr/bin/codesign
          security set-key-partition-list -S apple-tool:,apple: -s -k "" build.keychain

      - name: Build and package
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
          WIN_CSC_LINK: ${{ secrets.WIN_CERTIFICATE_P12 }}
          WIN_CSC_KEY_PASSWORD: ${{ secrets.WIN_CERTIFICATE_PASSWORD }}
        run: npm run package

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist-${{ matrix.platform }}
          path: |
            dist/*.exe
            dist/*.dmg
            dist/*.AppImage
            dist/*.deb
            dist/*.rpm
            dist/*.zip
            dist/latest*.yml
          retention-days: 5

  # Job 2: สร้าง GitHub Release
  create-release:
    needs: build
    runs-on: ubuntu-latest

    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Download all artifacts
        uses: actions/download-artifact@v4
        with:
          path: dist-all
          merge-multiple: true

      - name: Extract version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF_NAME#v}" >> $GITHUB_OUTPUT

      - name: Extract changelog
        id: changelog
        run: |
          # ดึง changelog สำหรับ version นี้
          VERSION="${{ steps.version.outputs.VERSION }}"
          NOTES=$(awk "/## \[$VERSION\]/{flag=1; next} /## \[/{flag=0} flag" CHANGELOG.md)
          echo "NOTES<<EOF" >> $GITHUB_OUTPUT
          echo "$NOTES" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          name: "v${{ steps.version.outputs.VERSION }}"
          body: ${{ steps.changelog.outputs.NOTES }}
          draft: false
          prerelease: ${{ contains(github.ref_name, 'beta') || contains(github.ref_name, 'rc') || contains(github.ref_name, 'alpha') }}
          files: dist-all/**/*
          generate_release_notes: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # Job 3: อัปเดต update server
  update-server:
    needs: create-release
    runs-on: ubuntu-latest
    if: "!contains(github.ref_name, 'alpha')"

    steps:
      - name: Notify update server
        run: |
          curl -X POST https://releases.example.com/webhook \
            -H "Authorization: Bearer ${{ secrets.RELEASE_WEBHOOK_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"version":"${{ github.ref_name }}", "channel":"${{ contains(github.ref_name, '"'"'beta'"'"') && '"'"'beta'"'"' || '"'"'stable'"'"' }}"}'
```

---

## 5. Code Signing

```typescript
// scripts/signWindows.ts
// Windows code signing ด้วย SignTool (Authenticode)

import { execSync } from 'child_process'
import * as path from 'path'
import * as fs from 'fs'

interface SignOptions {
  file: string
  certificatePath?: string
  certificatePassword?: string
  timestampUrl?: string
  description?: string
}

export function signWindowsExecutable(options: SignOptions): void {
  const {
    file,
    certificatePath = process.env.WIN_CSC_LINK,
    certificatePassword = process.env.WIN_CSC_KEY_PASSWORD,
    timestampUrl = 'http://timestamp.digicert.com',
    description = 'My Electron App'
  } = options

  if (!fs.existsSync(file)) {
    throw new Error(`File not found: ${file}`)
  }

  // ค้นหา signtool.exe
  const signtoolPaths = [
    'C:\\Program Files (x86)\\Windows Kits\\10\\bin\\10.0.19041.0\\x64\\signtool.exe',
    'C:\\Program Files (x86)\\Windows Kits\\10\\bin\\x64\\signtool.exe'
  ]

  const signtool = signtoolPaths.find(p => fs.existsSync(p))
  if (!signtool) {
    throw new Error('signtool.exe not found. Install Windows SDK.')
  }

  const args = [
    'sign',
    '/f', certificatePath!,
    '/p', certificatePassword!,
    '/t', timestampUrl,
    '/d', `"${description}"`,
    '/v',
    `"${file}"`
  ]

  console.log(`Signing: ${path.basename(file)}`)
  execSync(`"${signtool}" ${args.join(' ')}`, { stdio: 'inherit' })
  console.log(`Signed: ${path.basename(file)}`)
}
```

---

## 6. Release Notes Template

```markdown
<!-- .github/release-template.md -->
## What's New in v{{VERSION}}

{{FEATURES_SECTION}}

## Bug Fixes

{{BUGFIX_SECTION}}

## Breaking Changes

{{BREAKING_SECTION}}

## Download

| Platform | File | Size |
|----------|------|------|
| Windows  | `{{APP_NAME}}-{{VERSION}}-Setup.exe` | ~{{WIN_SIZE}} |
| macOS    | `{{APP_NAME}}-{{VERSION}}.dmg` | ~{{MAC_SIZE}} |
| Linux    | `{{APP_NAME}}-{{VERSION}}.AppImage` | ~{{LINUX_SIZE}} |

## Upgrade Notes

If you're upgrading from v{{PREV_MAJOR}}.x, please read the [Migration Guide](MIGRATION.md).

## Checksums (SHA-256)

```
{{CHECKSUMS}}
```

---
Full changelog: [CHANGELOG.md](CHANGELOG.md)
```

---

## 7. Multi-environment Config

```typescript
// src/main/environment.ts
// จัดการ config ตาม environment

export type AppEnvironment = 'development' | 'staging' | 'production'

interface EnvironmentConfig {
  apiBaseUrl: string
  updateServerUrl: string
  analyticsEnabled: boolean
  logLevel: 'debug' | 'info' | 'warn' | 'error'
  crashReportUrl: string
}

const configs: Record<AppEnvironment, EnvironmentConfig> = {
  development: {
    apiBaseUrl: 'http://localhost:3000',
    updateServerUrl: 'http://localhost:8080/updates',
    analyticsEnabled: false,
    logLevel: 'debug',
    crashReportUrl: 'http://localhost:9000/crashes'
  },
  staging: {
    apiBaseUrl: 'https://staging-api.example.com',
    updateServerUrl: 'https://staging-releases.example.com',
    analyticsEnabled: true,
    logLevel: 'info',
    crashReportUrl: 'https://staging-crashes.example.com'
  },
  production: {
    apiBaseUrl: 'https://api.example.com',
    updateServerUrl: 'https://releases.example.com',
    analyticsEnabled: true,
    logLevel: 'warn',
    crashReportUrl: 'https://crashes.example.com'
  }
}

export function getEnvironment(): AppEnvironment {
  if (process.env.NODE_ENV === 'development') return 'development'
  if (process.env.APP_ENV === 'staging') return 'staging'
  return 'production'
}

export function getConfig(): EnvironmentConfig {
  return configs[getEnvironment()]
}
```

---

## 8. Pre-release Checklist Script

```typescript
// scripts/preReleaseCheck.ts
// ตรวจสอบ checklist ก่อน release

import * as fs from 'fs'
import * as path from 'path'
import { execSync } from 'child_process'

interface CheckResult {
  name: string
  passed: boolean
  message?: string
}

const checks: Array<() => CheckResult> = [
  () => {
    const pkg = JSON.parse(fs.readFileSync('package.json', 'utf-8'))
    const valid = /^\d+\.\d+\.\d+(-[\w.]+)?$/.test(pkg.version)
    return { name: 'Version format valid', passed: valid, message: pkg.version }
  },

  () => {
    const pkg = JSON.parse(fs.readFileSync('package.json', 'utf-8'))
    const required = ['name', 'version', 'description', 'author', 'license']
    const missing = required.filter(f => !pkg[f])
    return {
      name: 'package.json complete',
      passed: missing.length === 0,
      message: missing.length > 0 ? `Missing: ${missing.join(', ')}` : undefined
    }
  },

  () => {
    const hasChangelog = fs.existsSync('CHANGELOG.md')
    return { name: 'CHANGELOG.md exists', passed: hasChangelog }
  },

  () => {
    const iconPaths = ['build/icon.ico', 'build/icon.icns', 'build/icon.png']
    const missing = iconPaths.filter(p => !fs.existsSync(p))
    return {
      name: 'App icons present',
      passed: missing.length === 0,
      message: missing.length > 0 ? `Missing: ${missing.join(', ')}` : undefined
    }
  },

  () => {
    try {
      execSync('npx tsc --noEmit', { stdio: 'pipe' })
      return { name: 'TypeScript compilation', passed: true }
    } catch (e: unknown) {
      return { name: 'TypeScript compilation', passed: false, message: String(e) }
    }
  },

  () => {
    try {
      const result = execSync('npm test -- --reporter=verbose 2>&1', { encoding: 'utf-8' })
      return { name: 'All tests passing', passed: true }
    } catch {
      return { name: 'All tests passing', passed: false }
    }
  },

  () => {
    const envVars = ['APPLE_ID', 'APPLE_TEAM_ID', 'WIN_CSC_LINK']
    const missing = envVars.filter(v => !process.env[v])
    return {
      name: 'Signing credentials set',
      passed: missing.length === 0,
      message: missing.length > 0 ? `Missing env: ${missing.join(', ')}` : undefined
    }
  }
]

function runChecks(): boolean {
  console.log('\nPre-release Checklist\n' + '='.repeat(40))

  let allPassed = true
  for (const check of checks) {
    const result = check()
    const icon = result.passed ? 'PASS' : 'FAIL'
    console.log(`[${icon}] ${result.name}${result.message ? ': ' + result.message : ''}`)
    if (!result.passed) allPassed = false
  }

  console.log('='.repeat(40))
  console.log(allPassed ? '\nAll checks passed!' : '\nSome checks FAILED.')
  return allPassed
}

const ok = runChecks()
process.exit(ok ? 0 : 1)
```

---

## สรุป Release Pipeline

```
Developer Local
  ↓  git commit (conventional commits)
  ↓  npm run release (bumps version, updates CHANGELOG, tags)
  ↓  git push && git push --tags

GitHub Actions (triggered by tag push)
  ↓  Build on Windows, macOS, Linux (matrix)
  ↓  Code sign each platform
  ↓  Upload artifacts
  ↓  Create GitHub Release with notes
  ↓  Upload update YML files to release
  ↓  Notify update server

User Machines (electron-updater)
  ↓  Check update server periodically
  ↓  Download update in background
  ↓  Prompt user to install
  ↓  Quit and install
```

| ขั้นตอน | เครื่องมือ | หมายเหตุ |
|---------|-----------|---------|
| Versioning | npm version | Conventional Commits |
| Changelog | custom script | auto-generated |
| Build | electron-builder | matrix: Win/Mac/Linux |
| Signing (Win) | signtool.exe | Authenticode |
| Signing (Mac) | codesign + notarize | Apple Developer |
| Release | GitHub Actions | softprops/action-gh-release |
| Distribution | GitHub Releases | electron-updater reads YML |
| Update delivery | electron-updater | staged rollout |
