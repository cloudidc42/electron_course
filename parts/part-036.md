# Part 036: Internationalization (i18n) ใน Electron.js
## การทำระบบหลายภาษา

---

## 🎯 เป้าหมายของบทเรียนนี้

หลังจากเรียนจบบทนี้ คุณจะสามารถ:
- ติดตั้งและใช้งาน i18next กับ Electron
- ตรวจจับภาษาของผู้ใช้อัตโนมัติ
- รองรับภาษา RTL (Right-to-Left)
- จัดรูปแบบตัวเลขและวันที่ตามภาษา
- เปลี่ยนภาษาแบบ Dynamic โดยไม่ต้อง restart แอพ
- แปลภาษาใน Electron Menu

---

## 1. ติดตั้ง Dependencies

```bash
npm install i18next i18next-fs-backend i18next-electron-fs-backend
npm install --save-dev @types/i18next
```

### โครงสร้างโปรเจกต์

```
my-electron-app/
├── src/
│   ├── main/
│   │   ├── index.js
│   │   └── i18n.js
│   ├── renderer/
│   │   ├── index.html
│   │   ├── app.js
│   │   └── i18n.js
│   └── preload/
│       └── preload.js
├── locales/
│   ├── en/
│   │   └── translation.json
│   ├── th/
│   │   └── translation.json
│   └── ar/
│       └── translation.json
└── package.json
```

---

## 2. สร้างไฟล์ภาษา (Translation Files)

### locales/en/translation.json

```json
{
  "app": {
    "title": "My Electron App",
    "welcome": "Welcome to {{name}}!",
    "description": "This is a cross-platform desktop application"
  },
  "menu": {
    "file": "File",
    "edit": "Edit",
    "view": "View",
    "help": "Help",
    "new": "New",
    "open": "Open",
    "save": "Save",
    "saveAs": "Save As...",
    "exit": "Exit",
    "preferences": "Preferences",
    "about": "About"
  },
  "buttons": {
    "ok": "OK",
    "cancel": "Cancel",
    "yes": "Yes",
    "no": "No",
    "save": "Save",
    "delete": "Delete",
    "edit": "Edit",
    "add": "Add New"
  },
  "messages": {
    "confirmDelete": "Are you sure you want to delete {{item}}?",
    "saveSuccess": "File saved successfully",
    "saveError": "Failed to save file: {{error}}",
    "loading": "Loading...",
    "noData": "No data available"
  },
  "errors": {
    "notFound": "Item not found",
    "networkError": "Network connection failed",
    "permissionDenied": "Permission denied"
  },
  "settings": {
    "language": "Language",
    "theme": "Theme",
    "notifications": "Notifications",
    "autoUpdate": "Auto Update"
  },
  "formats": {
    "date": "MM/DD/YYYY",
    "time": "hh:mm A",
    "currency": "${{amount}}"
  }
}
```

### locales/th/translation.json

```json
{
  "app": {
    "title": "แอพ Electron ของฉัน",
    "welcome": "ยินดีต้อนรับสู่ {{name}}!",
    "description": "แอพพลิเคชันเดสก์ท็อปแบบ cross-platform"
  },
  "menu": {
    "file": "ไฟล์",
    "edit": "แก้ไข",
    "view": "มุมมอง",
    "help": "ช่วยเหลือ",
    "new": "ใหม่",
    "open": "เปิด",
    "save": "บันทึก",
    "saveAs": "บันทึกเป็น...",
    "exit": "ออก",
    "preferences": "การตั้งค่า",
    "about": "เกี่ยวกับ"
  },
  "buttons": {
    "ok": "ตกลง",
    "cancel": "ยกเลิก",
    "yes": "ใช่",
    "no": "ไม่",
    "save": "บันทึก",
    "delete": "ลบ",
    "edit": "แก้ไข",
    "add": "เพิ่มใหม่"
  },
  "messages": {
    "confirmDelete": "คุณแน่ใจหรือไม่ที่จะลบ {{item}}?",
    "saveSuccess": "บันทึกไฟล์สำเร็จ",
    "saveError": "ไม่สามารถบันทึกไฟล์ได้: {{error}}",
    "loading": "กำลังโหลด...",
    "noData": "ไม่มีข้อมูล"
  },
  "errors": {
    "notFound": "ไม่พบรายการ",
    "networkError": "การเชื่อมต่อเครือข่ายล้มเหลว",
    "permissionDenied": "ปฏิเสธสิทธิ์"
  },
  "settings": {
    "language": "ภาษา",
    "theme": "ธีม",
    "notifications": "การแจ้งเตือน",
    "autoUpdate": "อัพเดทอัตโนมัติ"
  },
  "formats": {
    "date": "DD/MM/YYYY",
    "time": "HH:mm น.",
    "currency": "{{amount}} บาท"
  }
}
```

### locales/ar/translation.json (Arabic - RTL)

```json
{
  "app": {
    "title": "تطبيق Electron الخاص بي",
    "welcome": "مرحباً بك في {{name}}!",
    "description": "تطبيق سطح المكتب متعدد المنصات"
  },
  "menu": {
    "file": "ملف",
    "edit": "تعديل",
    "view": "عرض",
    "help": "مساعدة",
    "new": "جديد",
    "open": "فتح",
    "save": "حفظ",
    "exit": "خروج"
  },
  "buttons": {
    "ok": "موافق",
    "cancel": "إلغاء",
    "save": "حفظ",
    "delete": "حذف"
  }
}
```

---

## 3. ตั้งค่า i18next ใน Main Process

### src/main/i18n.js

```javascript
const i18next = require('i18next');
const Backend = require('i18next-fs-backend');
const path = require('path');
const { app } = require('electron');

// RTL languages list
const RTL_LANGUAGES = ['ar', 'he', 'fa', 'ur'];

class I18nManager {
  constructor() {
    this.i18n = i18next.createInstance();
    this.isInitialized = false;
  }

  async init() {
    const localesPath = path.join(
      app.isPackaged 
        ? process.resourcesPath 
        : path.join(__dirname, '../../'),
      'locales'
    );

    await this.i18n
      .use(Backend)
      .init({
        lng: this.getSavedLanguage() || this.detectSystemLanguage(),
        fallbackLng: 'en',
        debug: !app.isPackaged,
        
        backend: {
          loadPath: path.join(localesPath, '{{lng}}/{{ns}}.json'),
          addPath: path.join(localesPath, '{{lng}}/{{ns}}.missing.json'),
        },
        
        ns: ['translation'],
        defaultNS: 'translation',
        
        interpolation: {
          escapeValue: false,
        },
        
        // รองรับ plural forms
        pluralSeparator: '_',
        contextSeparator: '_',
      });

    this.isInitialized = true;
    console.log(`i18n initialized with language: ${this.i18n.language}`);
    return this;
  }

  // ตรวจจับภาษาของระบบ
  detectSystemLanguage() {
    const locale = app.getLocale(); // เช่น 'th-TH', 'en-US', 'ar-SA'
    const lang = locale.split('-')[0]; // เอาแค่ 'th', 'en', 'ar'
    
    // รายการภาษาที่รองรับ
    const supportedLangs = ['en', 'th', 'ar', 'ja', 'zh', 'ko', 'fr', 'de', 'es'];
    
    return supportedLangs.includes(lang) ? lang : 'en';
  }

  // โหลดภาษาที่บันทึกไว้
  getSavedLanguage() {
    try {
      const Store = require('electron-store');
      const store = new Store();
      return store.get('language');
    } catch (e) {
      return null;
    }
  }

  // เปลี่ยนภาษา
  async changeLanguage(lang) {
    await this.i18n.changeLanguage(lang);
    
    // บันทึกภาษาที่เลือก
    try {
      const Store = require('electron-store');
      const store = new Store();
      store.set('language', lang);
    } catch (e) {
      console.error('Failed to save language preference:', e);
    }
    
    return this.i18n.language;
  }

  // แปลข้อความ
  t(key, options = {}) {
    return this.i18n.t(key, options);
  }

  // ตรวจสอบว่าภาษาปัจจุบันเป็น RTL หรือไม่
  isRTL() {
    return RTL_LANGUAGES.includes(this.i18n.language);
  }

  // ดึงภาษาปัจจุบัน
  getCurrentLanguage() {
    return this.i18n.language;
  }

  // ดึงรายการภาษาที่รองรับ
  getSupportedLanguages() {
    return [
      { code: 'en', name: 'English', nativeName: 'English' },
      { code: 'th', name: 'Thai', nativeName: 'ภาษาไทย' },
      { code: 'ar', name: 'Arabic', nativeName: 'العربية', rtl: true },
      { code: 'ja', name: 'Japanese', nativeName: '日本語' },
      { code: 'zh', name: 'Chinese', nativeName: '中文' },
      { code: 'ko', name: 'Korean', nativeName: '한국어' },
      { code: 'fr', name: 'French', nativeName: 'Français' },
      { code: 'de', name: 'German', nativeName: 'Deutsch' },
    ];
  }
}

module.exports = new I18nManager();
```

---

## 4. Main Process - ตั้งค่า IPC และ Menu

### src/main/index.js

```javascript
const { app, BrowserWindow, ipcMain, Menu } = require('electron');
const path = require('path');
const i18nManager = require('./i18n');

let mainWindow;

async function createWindow() {
  // รอให้ i18n initialize ก่อน
  await i18nManager.init();
  
  const { t, isRTL } = i18nManager;

  mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    title: i18nManager.t('app.title'),
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, '../preload/preload.js'),
    },
  });

  // สร้าง Menu ที่แปลภาษาแล้ว
  createMenu();

  mainWindow.loadFile(path.join(__dirname, '../renderer/index.html'));
}

function createMenu() {
  const t = (key) => i18nManager.t(key);
  
  const template = [
    {
      label: t('menu.file'),
      submenu: [
        { label: t('menu.new'), accelerator: 'CmdOrCtrl+N' },
        { label: t('menu.open'), accelerator: 'CmdOrCtrl+O' },
        { type: 'separator' },
        { label: t('menu.save'), accelerator: 'CmdOrCtrl+S' },
        { label: t('menu.saveAs'), accelerator: 'CmdOrCtrl+Shift+S' },
        { type: 'separator' },
        { 
          label: t('menu.exit'), 
          accelerator: process.platform === 'darwin' ? 'Cmd+Q' : 'Alt+F4',
          click: () => app.quit()
        },
      ],
    },
    {
      label: t('menu.edit'),
      submenu: [
        { label: 'Undo', accelerator: 'CmdOrCtrl+Z', role: 'undo' },
        { label: 'Redo', accelerator: 'CmdOrCtrl+Y', role: 'redo' },
        { type: 'separator' },
        { role: 'cut' },
        { role: 'copy' },
        { role: 'paste' },
      ],
    },
    {
      label: t('settings.language'),
      submenu: i18nManager.getSupportedLanguages().map(lang => ({
        label: `${lang.nativeName} (${lang.name})`,
        type: 'radio',
        checked: lang.code === i18nManager.getCurrentLanguage(),
        click: async () => {
          await i18nManager.changeLanguage(lang.code);
          
          // แจ้ง renderer ให้อัพเดทภาษา
          mainWindow.webContents.send('language-changed', {
            language: lang.code,
            isRTL: i18nManager.isRTL(),
          });
          
          // Rebuild menu
          createMenu();
        },
      })),
    },
    {
      label: t('menu.help'),
      submenu: [
        { label: t('menu.about') },
      ],
    },
  ];

  const menu = Menu.buildFromTemplate(template);
  Menu.setApplicationMenu(menu);
}

// IPC Handlers
ipcMain.handle('i18n:translate', async (event, key, options) => {
  return i18nManager.t(key, options);
});

ipcMain.handle('i18n:changeLanguage', async (event, lang) => {
  const newLang = await i18nManager.changeLanguage(lang);
  
  // Rebuild menu
  createMenu();
  
  // แจ้ง window อื่นๆ
  BrowserWindow.getAllWindows().forEach(win => {
    win.webContents.send('language-changed', {
      language: newLang,
      isRTL: i18nManager.isRTL(),
    });
  });
  
  return { language: newLang, isRTL: i18nManager.isRTL() };
});

ipcMain.handle('i18n:getCurrentLanguage', () => {
  return {
    language: i18nManager.getCurrentLanguage(),
    isRTL: i18nManager.isRTL(),
    supportedLanguages: i18nManager.getSupportedLanguages(),
  };
});

app.whenReady().then(createWindow);
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit();
});
```

---

## 5. Preload Script

### src/preload/preload.js

```javascript
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('i18nAPI', {
  // แปลข้อความ
  translate: (key, options) => ipcRenderer.invoke('i18n:translate', key, options),
  
  // เปลี่ยนภาษา
  changeLanguage: (lang) => ipcRenderer.invoke('i18n:changeLanguage', lang),
  
  // ดึงภาษาปัจจุบัน
  getCurrentLanguage: () => ipcRenderer.invoke('i18n:getCurrentLanguage'),
  
  // รับ event เมื่อภาษาเปลี่ยน
  onLanguageChanged: (callback) => {
    ipcRenderer.on('language-changed', (event, data) => callback(data));
  },
  
  // ลบ listener
  removeLanguageListener: () => {
    ipcRenderer.removeAllListeners('language-changed');
  },
});
```

---

## 6. Renderer Process - i18n Client

### src/renderer/i18n.js

```javascript
class I18nClient {
  constructor() {
    this.currentLanguage = 'en';
    this.isRTL = false;
    this.listeners = [];
    this.setupLanguageChangeListener();
  }

  async init() {
    const { language, isRTL } = await window.i18nAPI.getCurrentLanguage();
    this.currentLanguage = language;
    this.isRTL = isRTL;
    
    // ตั้งค่า document direction
    this.updateDocumentDirection();
    
    return this;
  }

  // แปลข้อความ (async)
  async t(key, options = {}) {
    return await window.i18nAPI.translate(key, options);
  }

  // แปลข้อความ (sync สำหรับ HTML template)
  // ใช้ data attribute แทน
  async translatePage() {
    const elements = document.querySelectorAll('[data-i18n]');
    
    for (const element of elements) {
      const key = element.getAttribute('data-i18n');
      const options = {};
      
      // อ่าน interpolation values จาก data attributes
      for (const attr of element.attributes) {
        if (attr.name.startsWith('data-i18n-')) {
          const paramName = attr.name.replace('data-i18n-', '');
          options[paramName] = attr.value;
        }
      }
      
      const translation = await this.t(key, options);
      
      if (element.getAttribute('data-i18n-attr')) {
        // แปล attribute
        const attrName = element.getAttribute('data-i18n-attr');
        element.setAttribute(attrName, translation);
      } else {
        // แปล text content
        element.textContent = translation;
      }
    }
  }

  // เปลี่ยนภาษา
  async changeLanguage(lang) {
    const result = await window.i18nAPI.changeLanguage(lang);
    this.currentLanguage = result.language;
    this.isRTL = result.isRTL;
    this.updateDocumentDirection();
    await this.translatePage();
    
    // แจ้ง listeners
    this.listeners.forEach(fn => fn(result));
    
    return result;
  }

  // อัพเดท document direction
  updateDocumentDirection() {
    document.documentElement.dir = this.isRTL ? 'rtl' : 'ltr';
    document.documentElement.lang = this.currentLanguage;
    
    // เพิ่ม/ลบ class สำหรับ RTL styling
    if (this.isRTL) {
      document.body.classList.add('rtl');
    } else {
      document.body.classList.remove('rtl');
    }
  }

  // ฟังการเปลี่ยนภาษา
  setupLanguageChangeListener() {
    window.i18nAPI.onLanguageChanged(async (data) => {
      this.currentLanguage = data.language;
      this.isRTL = data.isRTL;
      this.updateDocumentDirection();
      await this.translatePage();
      this.listeners.forEach(fn => fn(data));
    });
  }

  // เพิ่ม listener
  onLanguageChange(callback) {
    this.listeners.push(callback);
    return () => {
      this.listeners = this.listeners.filter(fn => fn !== callback);
    };
  }

  destroy() {
    window.i18nAPI.removeLanguageListener();
  }
}

// Export instance
window.i18n = new I18nClient();
```

---

## 7. HTML Template

### src/renderer/index.html

```html
<!DOCTYPE html>
<html lang="en" dir="ltr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Electron i18n Demo</title>
  <style>
    /* Base styles */
    * { box-sizing: border-box; }
    
    body {
      font-family: 'Segoe UI', Tahoma, sans-serif;
      margin: 0;
      padding: 20px;
      background: #f5f5f5;
    }
    
    /* RTL Support */
    body.rtl {
      font-family: 'Arial', sans-serif;
    }
    
    /* RTL ปรับ margin/padding */
    body.rtl .card {
      text-align: right;
    }
    
    body.rtl .btn-group {
      flex-direction: row-reverse;
    }
    
    .container {
      max-width: 800px;
      margin: 0 auto;
    }
    
    .header {
      background: #2196F3;
      color: white;
      padding: 20px;
      border-radius: 8px;
      margin-bottom: 20px;
    }
    
    .lang-selector {
      display: flex;
      gap: 10px;
      flex-wrap: wrap;
      margin-bottom: 20px;
    }
    
    .lang-btn {
      padding: 8px 16px;
      border: 2px solid #2196F3;
      border-radius: 4px;
      cursor: pointer;
      background: white;
      transition: all 0.2s;
    }
    
    .lang-btn.active {
      background: #2196F3;
      color: white;
    }
    
    .card {
      background: white;
      padding: 20px;
      border-radius: 8px;
      margin-bottom: 16px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    
    .btn-group {
      display: flex;
      gap: 10px;
    }
    
    .info-grid {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 16px;
    }
    
    .info-item {
      padding: 12px;
      background: #f9f9f9;
      border-radius: 4px;
    }
    
    .info-label {
      font-size: 12px;
      color: #666;
      margin-bottom: 4px;
    }
    
    .info-value {
      font-size: 18px;
      font-weight: bold;
      color: #333;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1 data-i18n="app.title">My Electron App</h1>
      <p data-i18n="app.welcome" data-i18n-name="World">Welcome to World!</p>
    </div>
    
    <!-- Language Selector -->
    <div class="card">
      <h3 data-i18n="settings.language">Language</h3>
      <div class="lang-selector" id="langSelector">
        <!-- สร้างด้วย JavaScript -->
      </div>
    </div>
    
    <!-- Content -->
    <div class="card">
      <h2 data-i18n="app.description">Description</h2>
      <div class="btn-group">
        <button class="lang-btn" data-i18n="buttons.save">Save</button>
        <button class="lang-btn" data-i18n="buttons.cancel">Cancel</button>
        <button class="lang-btn" data-i18n="buttons.delete">Delete</button>
      </div>
    </div>
    
    <!-- Formatted Data -->
    <div class="card">
      <h3>Number & Date Formatting</h3>
      <div class="info-grid">
        <div class="info-item">
          <div class="info-label">Date</div>
          <div class="info-value" id="formattedDate">-</div>
        </div>
        <div class="info-item">
          <div class="info-label">Number</div>
          <div class="info-value" id="formattedNumber">-</div>
        </div>
        <div class="info-item">
          <div class="info-label">Currency</div>
          <div class="info-value" id="formattedCurrency">-</div>
        </div>
        <div class="info-item">
          <div class="info-label">Current Language</div>
          <div class="info-value" id="currentLang">-</div>
        </div>
      </div>
    </div>
  </div>

  <script src="i18n.js"></script>
  <script src="app.js"></script>
</body>
</html>
```

---

## 8. Renderer App Logic

### src/renderer/app.js

```javascript
// Number & Date Formatter
class LocaleFormatter {
  constructor(locale) {
    this.locale = locale;
  }

  // จัดรูปแบบตัวเลข
  formatNumber(number) {
    return new Intl.NumberFormat(this.locale).format(number);
  }

  // จัดรูปแบบตัวเลขทศนิยม
  formatDecimal(number, decimals = 2) {
    return new Intl.NumberFormat(this.locale, {
      minimumFractionDigits: decimals,
      maximumFractionDigits: decimals,
    }).format(number);
  }

  // จัดรูปแบบเงิน
  formatCurrency(amount, currency = 'USD') {
    // Map language to currency
    const currencyMap = {
      th: 'THB',
      en: 'USD',
      ar: 'SAR',
      ja: 'JPY',
      zh: 'CNY',
      ko: 'KRW',
      fr: 'EUR',
      de: 'EUR',
    };
    
    const lang = this.locale.split('-')[0];
    const curr = currency === 'USD' ? (currencyMap[lang] || 'USD') : currency;
    
    return new Intl.NumberFormat(this.locale, {
      style: 'currency',
      currency: curr,
    }).format(amount);
  }

  // จัดรูปแบบวันที่
  formatDate(date, style = 'long') {
    return new Intl.DateTimeFormat(this.locale, {
      dateStyle: style,
    }).format(date);
  }

  // จัดรูปแบบเวลา
  formatTime(date, style = 'short') {
    return new Intl.DateTimeFormat(this.locale, {
      timeStyle: style,
    }).format(date);
  }

  // จัดรูปแบบวันที่และเวลา
  formatDateTime(date) {
    return new Intl.DateTimeFormat(this.locale, {
      dateStyle: 'medium',
      timeStyle: 'short',
    }).format(date);
  }

  // จัดรูปแบบ relative time
  formatRelative(date) {
    const rtf = new Intl.RelativeTimeFormat(this.locale, { numeric: 'auto' });
    const diff = date - new Date();
    const seconds = Math.round(diff / 1000);
    const minutes = Math.round(seconds / 60);
    const hours = Math.round(minutes / 60);
    const days = Math.round(hours / 24);

    if (Math.abs(seconds) < 60) return rtf.format(seconds, 'second');
    if (Math.abs(minutes) < 60) return rtf.format(minutes, 'minute');
    if (Math.abs(hours) < 24) return rtf.format(hours, 'hour');
    return rtf.format(days, 'day');
  }

  // อัพเดท locale
  setLocale(locale) {
    this.locale = locale;
  }
}

// Map language code to locale
function langToLocale(lang) {
  const localeMap = {
    en: 'en-US',
    th: 'th-TH',
    ar: 'ar-SA',
    ja: 'ja-JP',
    zh: 'zh-CN',
    ko: 'ko-KR',
    fr: 'fr-FR',
    de: 'de-DE',
  };
  return localeMap[lang] || lang;
}

// Global formatter
let formatter = new LocaleFormatter('en-US');

// อัพเดทข้อมูล formatted
function updateFormattedData(lang) {
  const locale = langToLocale(lang);
  formatter.setLocale(locale);
  
  const now = new Date();
  const bigNumber = 1234567.89;
  
  document.getElementById('formattedDate').textContent = formatter.formatDate(now);
  document.getElementById('formattedNumber').textContent = formatter.formatNumber(bigNumber);
  document.getElementById('formattedCurrency').textContent = formatter.formatCurrency(bigNumber);
  document.getElementById('currentLang').textContent = `${lang} (${locale})`;
}

// สร้างปุ่มเลือกภาษา
async function buildLanguageSelector() {
  const { supportedLanguages, language } = await window.i18nAPI.getCurrentLanguage();
  const container = document.getElementById('langSelector');
  
  supportedLanguages.forEach(lang => {
    const btn = document.createElement('button');
    btn.className = `lang-btn ${lang.code === language ? 'active' : ''}`;
    btn.textContent = lang.nativeName;
    btn.title = lang.name;
    btn.dataset.lang = lang.code;
    
    btn.addEventListener('click', async () => {
      await window.i18n.changeLanguage(lang.code);
      
      // อัพเดทปุ่ม active
      document.querySelectorAll('.lang-btn').forEach(b => b.classList.remove('active'));
      btn.classList.add('active');
      
      // อัพเดทข้อมูล
      updateFormattedData(lang.code);
    });
    
    container.appendChild(btn);
  });
}

// Initialize
async function init() {
  await window.i18n.init();
  await window.i18n.translatePage();
  
  const { language } = await window.i18nAPI.getCurrentLanguage();
  updateFormattedData(language);
  
  await buildLanguageSelector();
  
  // Listen for language changes
  window.i18n.onLanguageChange(({ language }) => {
    updateFormattedData(language);
    
    // อัพเดทปุ่ม active
    document.querySelectorAll('.lang-btn').forEach(btn => {
      btn.classList.toggle('active', btn.dataset.lang === language);
    });
  });
}

// Start
init().catch(console.error);
```

---

## 9. RTL Support CSS

### src/renderer/rtl.css

```css
/* RTL Support */
[dir="rtl"] {
  /* Flip margins and paddings */
}

[dir="rtl"] .card {
  text-align: right;
}

[dir="rtl"] .btn-group {
  flex-direction: row-reverse;
}

[dir="rtl"] .lang-selector {
  flex-direction: row-reverse;
  justify-content: flex-end;
}

[dir="rtl"] .info-grid {
  direction: rtl;
}

/* Logical properties (modern way) */
.container {
  margin-inline: auto;
  padding-inline: 20px;
}

.card {
  padding-block: 16px;
  padding-inline: 20px;
}

/* RTL-specific font */
[lang="ar"] * {
  font-family: 'Segoe UI', 'Arabic Typesetting', 'Simplified Arabic', sans-serif;
}

[lang="th"] * {
  font-family: 'Segoe UI', 'Tahoma', 'Sarabun', sans-serif;
}

[lang="ja"] * {
  font-family: 'Yu Gothic', 'Meiryo', sans-serif;
}

[lang="zh"] * {
  font-family: 'Microsoft YaHei', 'SimSun', sans-serif;
}
```

---

## 10. Dynamic Language Switching ใน Vue/React

### สำหรับ React

```jsx
// src/renderer/hooks/useTranslation.js
import { useState, useEffect, useCallback } from 'react';

export function useTranslation() {
  const [language, setLanguage] = useState('en');
  const [isRTL, setIsRTL] = useState(false);

  useEffect(() => {
    // โหลดภาษาปัจจุบัน
    window.i18nAPI.getCurrentLanguage().then(({ language, isRTL }) => {
      setLanguage(language);
      setIsRTL(isRTL);
    });

    // Listen for changes
    const cleanup = window.i18nAPI.onLanguageChanged(({ language, isRTL }) => {
      setLanguage(language);
      setIsRTL(isRTL);
    });

    return cleanup;
  }, []);

  const t = useCallback(async (key, options) => {
    return window.i18nAPI.translate(key, options);
  }, []);

  const changeLanguage = useCallback(async (lang) => {
    return window.i18nAPI.changeLanguage(lang);
  }, []);

  return { t, language, isRTL, changeLanguage };
}

// Component ที่ใช้ hook
function WelcomeMessage() {
  const { t, language, isRTL, changeLanguage } = useTranslation();
  const [greeting, setGreeting] = useState('');

  useEffect(() => {
    t('app.welcome', { name: 'World' }).then(setGreeting);
  }, [language, t]);

  return (
    <div dir={isRTL ? 'rtl' : 'ltr'}>
      <p>{greeting}</p>
      <button onClick={() => changeLanguage('th')}>ภาษาไทย</button>
      <button onClick={() => changeLanguage('en')}>English</button>
      <button onClick={() => changeLanguage('ar')}>العربية</button>
    </div>
  );
}
```

---

## 11. Plural Forms

### locales/en/translation.json (plural)

```json
{
  "items": {
    "count_one": "{{count}} item",
    "count_other": "{{count}} items"
  },
  "notifications": {
    "new_one": "You have {{count}} new notification",
    "new_other": "You have {{count}} new notifications"
  }
}
```

### locales/th/translation.json (plural - Thai ไม่มี plural form)

```json
{
  "items": {
    "count": "{{count}} รายการ"
  },
  "notifications": {
    "new": "คุณมี {{count}} การแจ้งเตือนใหม่"
  }
}
```

### การใช้งาน plural

```javascript
// ใน main process
i18nManager.t('items.count', { count: 1 });  // "1 item"
i18nManager.t('items.count', { count: 5 });  // "5 items"

// ใน renderer (async)
await window.i18nAPI.translate('items.count', { count: 1 });
await window.i18nAPI.translate('items.count', { count: 5 });
```

---

## 12. สรุป

ในบทนี้เราได้เรียนรู้:

1. **i18next Integration** - ติดตั้งและตั้งค่า i18next กับ File System Backend
2. **Language Detection** - ตรวจจับภาษาจาก `app.getLocale()`
3. **RTL Support** - รองรับภาษาที่เขียนจากขวาไปซ้าย (Arabic, Hebrew)
4. **Number/Date Formatting** - ใช้ `Intl` API สำหรับ format ข้อมูลตามภาษา
5. **Dynamic Language Switching** - เปลี่ยนภาษาโดยไม่ต้อง restart
6. **Menu Localization** - แปลภาษาใน Application Menu

### Best Practices

- เก็บ translation files ใน `locales/` folder แยกเป็นภาษา
- ใช้ `app.getLocale()` สำหรับ default language detection
- รองรับ fallback language (ปกติใช้ English)
- ใช้ `Intl` API สำหรับ number/date formatting แทนการ hardcode
- ทดสอบ RTL layout ด้วย Arabic หรือ Hebrew
- ใช้ CSS logical properties (`margin-inline`, `padding-block`) แทน physical properties

---

*จบ Part 036 - ต่อไป Part 037: Testing with Jest & Playwright*
