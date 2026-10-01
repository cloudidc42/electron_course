# Part 043: Webpack Configuration
## ตั้งค่า Webpack สำหรับ Electron

---

## 🎯 เป้าหมายของบทเรียนนี้

- Webpack สำหรับ main/renderer process
- Hot Module Replacement (HMR)
- Asset handling
- CSS/SCSS
- Dev server config
- Production optimization
- Code splitting

---

## 1. ติดตั้ง Dependencies

```bash
npm install --save-dev webpack webpack-cli webpack-dev-server
npm install --save-dev babel-loader @babel/core @babel/preset-env @babel/preset-react @babel/preset-typescript
npm install --save-dev css-loader style-loader sass-loader sass
npm install --save-dev html-webpack-plugin mini-css-extract-plugin
npm install --save-dev file-loader url-loader
npm install --save-dev cross-env concurrently wait-on electron-reload
```

---

## 2. โครงสร้างไฟล์

```
webpack/
├── webpack.common.js
├── webpack.main.js
├── webpack.renderer.js
├── webpack.preload.js
└── webpack.dev.js
```

---

## 3. webpack.common.js (Shared config)

```javascript
const path = require('path');
const { DefinePlugin } = require('webpack');

module.exports = {
  // Common resolve
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx', '.json'],
    alias: {
      '@main': path.resolve(__dirname, '../src/main'),
      '@renderer': path.resolve(__dirname, '../src/renderer'),
      '@preload': path.resolve(__dirname, '../src/preload'),
      '@shared': path.resolve(__dirname, '../src/shared'),
      '@assets': path.resolve(__dirname, '../src/assets'),
    },
  },
  
  // Common plugins
  plugins: [
    new DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV || 'development'),
      '__DEV__': JSON.stringify(process.env.NODE_ENV !== 'production'),
    }),
  ],
  
  // Common module rules
  module: {
    rules: [
      // TypeScript/JavaScript
      {
        test: /\.(js|jsx|ts|tsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            cacheDirectory: true,
            presets: [
              ['@babel/preset-env', { targets: { node: 'current' } }],
              ['@babel/preset-react', { runtime: 'automatic' }],
              '@babel/preset-typescript',
            ],
            plugins: [
              ['@babel/plugin-transform-runtime'],
            ],
          },
        },
      },
    ],
  },
};
```

---

## 4. webpack.main.js

```javascript
const path = require('path');
const { merge } = require('webpack-merge');
const common = require('./webpack.common');

const isDev = process.env.NODE_ENV !== 'production';

module.exports = merge(common, {
  mode: isDev ? 'development' : 'production',
  
  target: 'electron-main',
  
  entry: {
    main: './src/main/index.ts',
  },
  
  output: {
    path: path.resolve(__dirname, '../dist/main'),
    filename: '[name].js',
    libraryTarget: 'commonjs2',
  },
  
  // ไม่ให้ bundle node_modules
  externals: {
    // Native modules
    'better-sqlite3': 'commonjs better-sqlite3',
    'sharp': 'commonjs sharp',
    'canvas': 'commonjs canvas',
    
    // Electron modules
    'electron': 'commonjs electron',
  },
  
  module: {
    rules: [
      // Node modules ที่มี native code
      {
        test: /\.node$/,
        use: 'node-loader',
      },
    ],
  },
  
  node: {
    __dirname: false,
    __filename: false,
  },
  
  // Source maps
  devtool: isDev ? 'cheap-source-map' : false,
  
  // Optimization สำหรับ production
  optimization: {
    minimize: !isDev,
    nodeEnv: false,
  },
  
  // Stats
  stats: {
    colors: true,
    hash: false,
    version: false,
    timings: true,
    assets: true,
    chunks: false,
    modules: false,
    reasons: false,
    children: false,
    source: false,
    errors: true,
    errorDetails: true,
    warnings: true,
    publicPath: false,
  },
});
```

---

## 5. webpack.renderer.js

```javascript
const path = require('path');
const { merge } = require('webpack-merge');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const TerserPlugin = require('terser-webpack-plugin');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
const common = require('./webpack.common');

const isDev = process.env.NODE_ENV !== 'production';
const analyzeBundle = process.env.ANALYZE === 'true';

module.exports = merge(common, {
  mode: isDev ? 'development' : 'production',
  
  target: 'electron-renderer',
  
  entry: {
    renderer: './src/renderer/index.tsx',
  },
  
  output: {
    path: path.resolve(__dirname, '../dist/renderer'),
    filename: isDev ? '[name].js' : '[name].[contenthash:8].js',
    chunkFilename: isDev ? '[name].chunk.js' : '[name].[contenthash:8].chunk.js',
    assetModuleFilename: 'assets/[name].[hash:8][ext]',
    clean: true,
  },
  
  module: {
    rules: [
      // CSS
      {
        test: /\.css$/,
        use: [
          isDev ? 'style-loader' : MiniCssExtractPlugin.loader,
          {
            loader: 'css-loader',
            options: {
              modules: {
                auto: true, // *.module.css จะใช้ CSS Modules
                localIdentName: isDev 
                  ? '[name]__[local]--[hash:base64:5]'
                  : '[hash:base64:8]',
              },
              sourceMap: isDev,
            },
          },
          'postcss-loader',
        ],
      },
      
      // SCSS/SASS
      {
        test: /\.s[ac]ss$/,
        use: [
          isDev ? 'style-loader' : MiniCssExtractPlugin.loader,
          {
            loader: 'css-loader',
            options: {
              modules: {
                auto: /\.module\.s[ac]ss$/,
              },
              sourceMap: isDev,
            },
          },
          'postcss-loader',
          {
            loader: 'sass-loader',
            options: {
              sourceMap: isDev,
              sassOptions: {
                includePaths: ['src/styles'],
              },
              additionalData: `@import "@styles/variables";`,
            },
          },
        ],
      },
      
      // Images
      {
        test: /\.(png|jpe?g|gif|webp|avif)$/,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 10 * 1024, // 10kb
          },
        },
        generator: {
          filename: 'images/[name].[hash:8][ext]',
        },
      },
      
      // SVG - SVGR (import as React component)
      {
        test: /\.svg$/,
        oneOf: [
          // import { ReactComponent as Icon } from './icon.svg'
          {
            issuer: /\.[jt]sx?$/,
            resourceQuery: /\?react/,
            use: ['@svgr/webpack'],
          },
          // import iconUrl from './icon.svg'
          {
            type: 'asset/resource',
            generator: {
              filename: 'images/[name].[hash:8][ext]',
            },
          },
        ],
      },
      
      // Fonts
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        type: 'asset/resource',
        generator: {
          filename: 'fonts/[name].[hash:8][ext]',
        },
      },
      
      // Videos
      {
        test: /\.(mp4|webm|ogg|mp3|wav|flac|aac)$/,
        type: 'asset/resource',
        generator: {
          filename: 'media/[name].[hash:8][ext]',
        },
      },
    ],
  },
  
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/renderer/index.html',
      filename: 'index.html',
      inject: true,
      minify: !isDev ? {
        removeComments: true,
        collapseWhitespace: true,
        removeRedundantAttributes: true,
        useShortDoctype: true,
        removeEmptyAttributes: true,
        removeStyleLinkTypeAttributes: true,
        keepClosingSlash: true,
        minifyJS: true,
        minifyCSS: true,
        minifyURLs: true,
      } : false,
    }),
    
    // CSS extraction (production only)
    ...(isDev ? [] : [
      new MiniCssExtractPlugin({
        filename: 'css/[name].[contenthash:8].css',
        chunkFilename: 'css/[name].[contenthash:8].chunk.css',
      }),
    ]),
    
    // Bundle analyzer
    ...(analyzeBundle ? [
      new BundleAnalyzerPlugin({
        analyzerMode: 'static',
        reportFilename: '../bundle-analysis.html',
      }),
    ] : []),
  ],
  
  optimization: {
    minimize: !isDev,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            ecma: 2020,
            drop_console: true,
            drop_debugger: true,
            pure_funcs: ['console.log', 'console.info'],
          },
          format: {
            comments: false,
          },
        },
        extractComments: false,
      }),
      new CssMinimizerPlugin(),
    ],
    
    // Code splitting
    splitChunks: {
      chunks: 'all',
      minSize: 20000,
      maxSize: 250000,
      minChunks: 1,
      maxAsyncRequests: 30,
      maxInitialRequests: 30,
      cacheGroups: {
        defaultVendors: {
          test: /[\\/]node_modules[\\/]/,
          priority: -10,
          reuseExistingChunk: true,
          name(module) {
            const packageName = module.context.match(
              /[\\/]node_modules[\\/](.*?)([\\/]|$)/
            )[1];
            return `vendor.${packageName.replace('@', '')}`;
          },
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true,
        },
        // แยก React เป็น chunk พิเศษ
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'react',
          priority: 10,
          chunks: 'all',
        },
      },
    },
    
    // Runtime chunk แยกต่างหาก
    runtimeChunk: {
      name: 'runtime',
    },
  },
  
  devtool: isDev ? 'cheap-module-source-map' : false,
  
  // Dev server สำหรับ HMR
  devServer: {
    port: 3000,
    hot: true,
    historyApiFallback: true,
    static: {
      directory: path.join(__dirname, '../dist/renderer'),
    },
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
  },
  
  performance: {
    hints: isDev ? false : 'warning',
    maxEntrypointSize: 512000,
    maxAssetSize: 512000,
  },
});
```

---

## 6. webpack.preload.js

```javascript
const path = require('path');
const { merge } = require('webpack-merge');
const common = require('./webpack.common');

module.exports = merge(common, {
  mode: process.env.NODE_ENV !== 'production' ? 'development' : 'production',
  
  target: 'electron-preload',
  
  entry: {
    preload: './src/preload/preload.ts',
  },
  
  output: {
    path: path.resolve(__dirname, '../dist/preload'),
    filename: '[name].js',
  },
  
  externals: {
    electron: 'commonjs electron',
  },
  
  node: {
    __dirname: false,
    __filename: false,
  },
  
  devtool: process.env.NODE_ENV !== 'production' ? 'cheap-source-map' : false,
});
```

---

## 7. package.json scripts

```json
{
  "scripts": {
    "dev": "concurrently \"npm run dev:renderer\" \"npm run dev:main\" \"npm run dev:electron\"",
    "dev:renderer": "cross-env NODE_ENV=development webpack serve --config webpack/webpack.renderer.js",
    "dev:main": "cross-env NODE_ENV=development webpack --config webpack/webpack.main.js --watch",
    "dev:preload": "cross-env NODE_ENV=development webpack --config webpack/webpack.preload.js --watch",
    "dev:electron": "wait-on http://localhost:3000 && electron dist/main/main.js",
    
    "build": "npm run build:main && npm run build:preload && npm run build:renderer",
    "build:main": "cross-env NODE_ENV=production webpack --config webpack/webpack.main.js",
    "build:renderer": "cross-env NODE_ENV=production webpack --config webpack/webpack.renderer.js",
    "build:preload": "cross-env NODE_ENV=production webpack --config webpack/webpack.preload.js",
    
    "analyze": "cross-env ANALYZE=true npm run build:renderer"
  }
}
```

---

## 8. postcss.config.js

```javascript
module.exports = {
  plugins: [
    require('autoprefixer'),
    require('postcss-preset-env')({
      stage: 3,
      features: {
        'nesting-rules': true,
        'custom-properties': true,
      },
    }),
    ...(process.env.NODE_ENV === 'production' ? [require('cssnano')] : []),
  ],
};
```

---

## 9. สรุป

| Feature | Config |
|---------|--------|
| Main process | `target: 'electron-main'` |
| Renderer process | `target: 'electron-renderer'` |
| Preload script | `target: 'electron-preload'` |
| HMR | `webpack-dev-server` + hot |
| CSS Modules | `css-loader modules: { auto: true }` |
| Code splitting | `optimization.splitChunks` |

---

*จบ Part 043 - ต่อไป Part 044: Vite + Electron*
