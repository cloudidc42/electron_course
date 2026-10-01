# Part 041: Electron Forge
## สร้างและ Package แอพด้วย Electron Forge

---

## 🎯 เป้าหมายของบทเรียนนี้

- ตั้งค่า Electron Forge
- Makers (Squirrel, DMG, deb)
- Publishers (GitHub, S3)
- Plugins (Webpack, Vite)
- TypeScript setup
- Hot reload

---

## 1. สร้างโปรเจกต์ใหม่ด้วย Electron Forge

```bash
# สร้างโปรเจกต์ใหม่
npm init electron-app@latest my-app -- --template=webpack

# หรือด้วย TypeScript
npm init electron-app@latest my-app -- --template=webpack-typescript

# หรือด้วย Vite
npm init electron-app@latest my-app -- --template=vite

# cd เข้าโปรเจกต์
cd my-app
npm start
```

---

## 2. เพิ่ม Electron Forge ใน Existing Project

```bash
# ติดตั้ง
npm install --save-dev @electron-forge/cli

# Import config
npx electron-forge import
```

---

## 3. forge.config.js

```javascript
const { FusesPlugin } = require('@electron-forge/plugin-fuses');
const { FuseV1Options, FuseVersion } = require('@electron/fuses');

module.exports = {
  // App metadata
  packagerConfig: {
    asar: true,
    asarUnpack: [
      // Files ที่ต้องแตกออกจาก ASAR
      '**/node_modules/sharp/**/*',
      '**/node_modules/better-sqlite3/**/*',
    ],
    
    // Icons
    icon: './assets/icons/icon',
    
    // macOS specific
    darwinDarkModeSupport: true,
    
    // Windows specific
    win32metadata: {
      CompanyName: 'Your Company',
      FileDescription: 'My Electron App',
      ProductName: 'My Electron App',
      InternalName: 'my-electron-app',
    },
    
    // Ignore files
    ignore: [
      /^\/tests/,
      /^\/\.github/,
      /^\/docs/,
      /\.test\.(js|ts)$/,
      /\.spec\.(js|ts)$/,
    ],
  },
  
  // Rebuild native modules
  rebuildConfig: {},
  
  // Makers (installers/packages)
  makers: [
    // Windows: Squirrel.Windows
    {
      name: '@electron-forge/maker-squirrel',
      config: {
        name: 'MyElectronApp',
        setupIcon: './assets/icons/icon.ico',
        
        // Auto-update server
        remoteReleases: 'https://github.com/yourusername/my-app',
        
        // Certificate
        certificateFile: process.env.CSC_LINK,
        certificatePassword: process.env.CSC_KEY_PASSWORD,
      },
    },
    
    // macOS: DMG
    {
      name: '@electron-forge/maker-dmg',
      config: {
        icon: './assets/icons/icon.icns',
        format: 'ULFO',
        background: './assets/dmg-background.png',
        contents: [
          { x: 130, y: 220, type: 'file' },
          { x: 410, y: 220, type: 'link', path: '/Applications' },
        ],
      },
    },
    
    // macOS: ZIP (for auto-update)
    {
      name: '@electron-forge/maker-zip',
      platforms: ['darwin'],
    },
    
    // Linux: deb
    {
      name: '@electron-forge/maker-deb',
      config: {
        options: {
          name: 'my-electron-app',
          productName: 'My Electron App',
          maintainer: 'Your Name <your@email.com>',
          homepage: 'https://yourwebsite.com',
          description: 'A powerful desktop application',
          categories: ['Utility'],
          icon: './assets/icons/512x512.png',
          depends: ['libgtk-3-0', 'libnotify4'],
        },
      },
    },
    
    // Linux: RPM
    {
      name: '@electron-forge/maker-rpm',
      config: {
        options: {
          name: 'my-electron-app',
          productName: 'My Electron App',
          description: 'A powerful desktop application',
          homepage: 'https://yourwebsite.com',
          icon: './assets/icons/512x512.png',
        },
      },
    },
  ],
  
  // Plugins
  plugins: [
    // Webpack plugin
    {
      name: '@electron-forge/plugin-webpack',
      config: {
        mainConfig: './webpack.main.config.js',
        renderer: {
          config: './webpack.renderer.config.js',
          entryPoints: [
            {
              html: './src/renderer/index.html',
              js: './src/renderer/index.js',
              name: 'main_window',
              preload: {
                js: './src/preload/preload.js',
              },
            },
          ],
        },
        devContentSecurityPolicy: `default-src 'self' 'unsafe-inline' data:; script-src 'self' 'unsafe-eval' 'unsafe-inline' data:`,
        port: 3000,
        loggerPort: 9000,
      },
    },
    
    // Fuses - Security features
    new FusesPlugin({
      version: FuseVersion.V1,
      [FuseV1Options.RunAsNode]: false,
      [FuseV1Options.EnableCookieEncryption]: true,
      [FuseV1Options.EnableNodeOptionsEnvironmentVariable]: false,
      [FuseV1Options.EnableNodeCliInspectArguments]: false,
      [FuseV1Options.EnableEmbeddedAsarIntegrityValidation]: true,
      [FuseV1Options.OnlyLoadAppFromAsar]: true,
    }),
  ],
  
  // Publishers
  publishers: [
    {
      name: '@electron-forge/publisher-github',
      config: {
        repository: {
          owner: 'yourusername',
          name: 'my-app',
        },
        prerelease: false,
        draft: true,
        generateReleaseNotes: true,
      },
    },
  ],
  
  // Hooks
  hooks: {
    generateAssets: async () => {
      // สร้าง assets ก่อน build
      console.log('Generating assets...');
    },
    packageAfterPrune: async (config, buildPath) => {
      // ทำ custom processing หลัง prune
    },
  },
};
```

---

## 4. Webpack Configuration

### webpack.main.config.js

```javascript
const path = require('path');

module.exports = {
  entry: './src/main/index.js',
  mode: 'production',
  
  target: 'electron-main',
  
  output: {
    path: path.join(__dirname, '.webpack/main'),
    filename: 'index.js',
  },
  
  module: {
    rules: [
      {
        test: /\.(js|ts)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env'],
          },
        },
      },
    ],
  },
  
  resolve: {
    extensions: ['.js', '.ts', '.json'],
    alias: {
      '@main': path.resolve(__dirname, 'src/main'),
      '@shared': path.resolve(__dirname, 'src/shared'),
    },
  },
  
  externals: {
    // Native modules ที่ต้อง externalize
    'better-sqlite3': 'commonjs better-sqlite3',
    'sharp': 'commonjs sharp',
  },
  
  node: {
    __dirname: false,
    __filename: false,
  },
};
```

### webpack.renderer.config.js

```javascript
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

const isDev = process.env.NODE_ENV !== 'production';

module.exports = {
  entry: './src/renderer/index.jsx',
  mode: isDev ? 'development' : 'production',
  
  target: 'electron-renderer',
  
  output: {
    path: path.join(__dirname, '.webpack/renderer'),
    filename: '[name].js',
  },
  
  module: {
    rules: [
      // JavaScript/TypeScript
      {
        test: /\.(js|jsx|ts|tsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              '@babel/preset-env',
              '@babel/preset-react',
              '@babel/preset-typescript',
            ],
          },
        },
      },
      
      // CSS
      {
        test: /\.css$/,
        use: [
          isDev ? 'style-loader' : MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader',
        ],
      },
      
      // SCSS/SASS
      {
        test: /\.s[ac]ss$/,
        use: [
          isDev ? 'style-loader' : MiniCssExtractPlugin.loader,
          'css-loader',
          'sass-loader',
        ],
      },
      
      // Images
      {
        test: /\.(png|jpe?g|gif|svg|webp)$/,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024, // 8kb - inline เป็น base64
          },
        },
        generator: {
          filename: 'images/[name].[hash:8][ext]',
        },
      },
      
      // Fonts
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        type: 'asset/resource',
        generator: {
          filename: 'fonts/[name].[hash:8][ext]',
        },
      },
    ],
  },
  
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/renderer/index.html',
      inject: true,
    }),
    
    ...(isDev ? [] : [new MiniCssExtractPlugin({ filename: '[name].css' })]),
  ],
  
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx'],
    alias: {
      '@renderer': path.resolve(__dirname, 'src/renderer'),
      '@components': path.resolve(__dirname, 'src/renderer/components'),
      '@shared': path.resolve(__dirname, 'src/shared'),
    },
  },
  
  devServer: {
    port: 3000,
    hot: true,
  },
  
  optimization: {
    splitChunks: {
      chunks: 'all',
    },
  },
};
```

---

## 5. Vite Plugin

### forge.config.js กับ Vite

```javascript
const { VitePlugin } = require('@electron-forge/plugin-vite');

module.exports = {
  packagerConfig: {
    asar: true,
    icon: './assets/icons/icon',
  },
  
  plugins: [
    new VitePlugin({
      // Main process
      build: [
        {
          entry: 'src/main/index.ts',
          config: 'vite.main.config.ts',
          target: 'main',
        },
        {
          entry: 'src/preload/preload.ts',
          config: 'vite.preload.config.ts',
          target: 'preload',
        },
      ],
      
      // Renderer process
      renderer: [
        {
          name: 'main_window',
          config: 'vite.renderer.config.ts',
        },
      ],
    }),
  ],
};
```

---

## 6. TypeScript Setup

### tsconfig.json (root)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "CommonJS",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "baseUrl": ".",
    "paths": {
      "@main/*": ["src/main/*"],
      "@renderer/*": ["src/renderer/*"],
      "@shared/*": ["src/shared/*"],
      "@preload/*": ["src/preload/*"]
    }
  }
}
```

### tsconfig.main.json

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "outDir": ".webpack/main",
    "lib": ["ES2022"],
    "types": ["node"]
  },
  "include": ["src/main/**/*", "src/shared/**/*"]
}
```

### tsconfig.renderer.json

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "outDir": ".webpack/renderer",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "types": []
  },
  "include": ["src/renderer/**/*", "src/shared/**/*"]
}
```

---

## 7. Hot Reload ใน Development

```javascript
// src/main/index.ts
import { app, BrowserWindow } from 'electron';
import path from 'path';

declare const MAIN_WINDOW_WEBPACK_ENTRY: string;
declare const MAIN_WINDOW_PRELOAD_WEBPACK_ENTRY: string;

function createWindow() {
  const win = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: MAIN_WINDOW_PRELOAD_WEBPACK_ENTRY,
    },
  });

  // Webpack plugin จัดการ HMR ให้อัตโนมัติ
  win.loadURL(MAIN_WINDOW_WEBPACK_ENTRY);

  // เปิด DevTools ใน development
  if (process.env.NODE_ENV === 'development') {
    win.webContents.openDevTools();
  }
}

app.whenReady().then(createWindow);
```

---

## 8. Scripts

```json
{
  "scripts": {
    "start": "electron-forge start",
    "package": "electron-forge package",
    "make": "electron-forge make",
    "publish": "electron-forge publish",
    
    "make:win": "electron-forge make --platform win32",
    "make:mac": "electron-forge make --platform darwin",
    "make:linux": "electron-forge make --platform linux",
    
    "publish:github": "electron-forge publish --target @electron-forge/publisher-github"
  }
}
```

---

## 9. สรุป

### เปรียบเทียบ Electron Forge vs electron-builder

| Feature | Electron Forge | electron-builder |
|---------|----------------|------------------|
| Setup | ง่ายกว่า | ซับซ้อนกว่า |
| Webpack/Vite | มี Plugin ในตัว | ต้องตั้งค่าเอง |
| Makers | Squirrel, DMG, deb | NSIS, DMG, AppImage, deb, rpm |
| Publishers | GitHub, S3 | GitHub, S3 |
| Community | เล็กกว่า | ใหญ่กว่า |
| Documentation | ดี | ดีมาก |

### เมื่อไหร่ใช้ Electron Forge

- โปรเจกต์ใหม่ที่ต้องการ setup เร็ว
- ใช้ Webpack หรือ Vite
- ต้องการ official Electron tool

---

*จบ Part 041 - ต่อไป Part 042: TypeScript Integration*
