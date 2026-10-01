# Part 055: System Notifications Advanced
## การแสดง System Notifications ขั้นสูง

---

## 🎯 เป้าหมายของบทเรียนนี้

- Notification grouping
- Notification actions (buttons)
- Persistent notifications
- Notification categories (macOS)
- Windows Toast notifications

---

## 1. Basic Notifications

```typescript
import { Notification, app } from 'electron';

function showNotification(title: string, body: string): void {
  if (!Notification.isSupported()) {
    console.warn('Notifications not supported');
    return;
  }
  
  const notification = new Notification({
    title,
    body,
    icon: join(__dirname, '../../assets/icons/icon.png'),
    silent: false,
  });
  
  notification.on('click', () => {
    console.log('Notification clicked');
    // Focus app window
    BrowserWindow.getAllWindows().forEach(win => {
      if (win.isMinimized()) win.restore();
      win.focus();
    });
  });
  
  notification.show();
}
```

---

## 2. Notification กับ Actions (macOS)

```typescript
// macOS Notification Actions
function showNotificationWithActions(): void {
  const notification = new Notification({
    title: 'New Message',
    body: 'You have a new message from John',
    icon: join(__dirname, '../../assets/icons/icon.png'),
    
    // Actions (macOS only)
    actions: [
      {
        type: 'button',
        text: 'Reply',
      },
      {
        type: 'button', 
        text: 'Mark as Read',
      },
    ],
    
    // Reply field (macOS only)
    hasReply: true,
    replyPlaceholder: 'Type your reply...',
    
    // Close button text
    closeButtonText: 'Dismiss',
    
    // urgency
    urgency: 'normal', // 'low' | 'normal' | 'critical'
  });
  
  // Handle action button click
  notification.on('action', (event, index) => {
    if (index === 0) {
      console.log('Reply button clicked');
      // Open reply window
    } else if (index === 1) {
      console.log('Mark as read clicked');
      // Mark message as read
    }
  });
  
  // Handle reply (macOS)
  notification.on('reply', (event, reply) => {
    console.log('User replied:', reply);
    // Send reply
  });
  
  notification.on('click', () => {
    console.log('Notification body clicked');
  });
  
  notification.on('close', () => {
    console.log('Notification dismissed');
  });
  
  notification.show();
}
```

---

## 3. Notification Manager

### src/main/notifications/NotificationManager.ts

```typescript
import { Notification, BrowserWindow, app } from 'electron';
import { join } from 'path';

interface NotificationOptions {
  id: string;
  title: string;
  body: string;
  type?: 'info' | 'success' | 'warning' | 'error';
  icon?: string;
  sound?: boolean;
  persistent?: boolean;
  actions?: Array<{ text: string; handler: () => void }>;
  onClick?: () => void;
  onClose?: () => void;
  groupId?: string;
  timeout?: number; // ms (0 = no timeout)
  data?: any;
}

class NotificationManager {
  private notifications: Map<string, {
    notification: Notification;
    options: NotificationOptions;
  }> = new Map();
  
  private groups: Map<string, string[]> = new Map();
  
  // Icon ตาม type
  private getIcon(type: string = 'info'): string {
    const iconMap: Record<string, string> = {
      info: 'icon.png',
      success: 'success.png',
      warning: 'warning.png',
      error: 'error.png',
    };
    
    return join(
      app.isPackaged ? process.resourcesPath : join(__dirname, '../../../'),
      'assets/icons',
      iconMap[type] || 'icon.png'
    );
  }
  
  show(options: NotificationOptions): void {
    if (!Notification.isSupported()) return;
    
    // ปิด notification เดิมถ้ามี id เดียวกัน
    this.close(options.id);
    
    const notification = new Notification({
      title: options.title,
      body: options.body,
      icon: options.icon || this.getIcon(options.type),
      silent: !options.sound,
      urgency: options.type === 'error' ? 'critical' : 
               options.type === 'warning' ? 'normal' : 'low',
      
      // macOS actions
      actions: options.actions?.map(a => ({
        type: 'button' as const,
        text: a.text,
      })),
      
      timeoutType: options.persistent ? 'never' : 'default',
    });
    
    // Event handlers
    notification.on('click', () => {
      options.onClick?.();
      this.focusApp();
      
      if (!options.persistent) {
        this.notifications.delete(options.id);
      }
    });
    
    notification.on('close', () => {
      options.onClose?.();
      this.notifications.delete(options.id);
    });
    
    notification.on('action', (_, index) => {
      options.actions?.[index]?.handler();
    });
    
    // จัดการ group
    if (options.groupId) {
      this.addToGroup(options.groupId, options.id);
    }
    
    this.notifications.set(options.id, { notification, options });
    notification.show();
    
    // Auto-close
    if (options.timeout && options.timeout > 0) {
      setTimeout(() => this.close(options.id), options.timeout);
    }
  }
  
  close(id: string): void {
    const item = this.notifications.get(id);
    if (item) {
      item.notification.close();
      this.notifications.delete(id);
    }
  }
  
  closeGroup(groupId: string): void {
    const ids = this.groups.get(groupId) || [];
    ids.forEach(id => this.close(id));
    this.groups.delete(groupId);
  }
  
  closeAll(): void {
    this.notifications.forEach((_, id) => this.close(id));
  }
  
  private addToGroup(groupId: string, notificationId: string): void {
    const group = this.groups.get(groupId) || [];
    group.push(notificationId);
    this.groups.set(groupId, group);
  }
  
  private focusApp(): void {
    BrowserWindow.getAllWindows().forEach(win => {
      if (win.isMinimized()) win.restore();
      win.focus();
    });
    app.focus({ steal: true });
  }
  
  // Grouped notification (แสดงแค่ 1 notification สำหรับหลาย items)
  showGrouped(groupId: string, items: string[], title: string): void {
    const count = items.length;
    
    this.show({
      id: `group-${groupId}`,
      title,
      body: count === 1 
        ? items[0] 
        : `${items[0]} and ${count - 1} more`,
      groupId,
      onClick: () => {
        BrowserWindow.getAllWindows()[0]?.webContents.send('notification:group-clicked', {
          groupId,
          items,
        });
      },
    });
  }
}

export const notificationManager = new NotificationManager();
```

---

## 4. macOS Notification Categories

```typescript
// macOS ใช้ UNUserNotificationCenter (ผ่าน native addon)
// electron-macos-notifications หรือ node-mac-usernotifications

// ตัวอย่างด้วย native code (Swift wrapper)
// หรือใช้ basic Notification API ที่มีอยู่แล้ว

// macOS Notification Center Integration
function requestNotificationPermission(): void {
  if (process.platform !== 'darwin') return;
  
  // ใน Electron ไม่ต้อง request permission โดยตรง
  // แต่ถ้า app ถูก sandbox (Mac App Store) ต้องใช้ NSUserNotificationCenter
  
  const notification = new Notification({
    title: 'Test',
    body: 'Testing notification',
  });
  notification.show();
}
```

---

## 5. Windows Toast Notifications

```typescript
// Windows Toast Notifications
// ใช้ node-notifier หรือ custom native module

import notifier from 'node-notifier';
import path from 'path';
import { app } from 'electron';

function showWindowsToast(options: {
  title: string;
  message: string;
  icon?: string;
  sound?: boolean;
  wait?: boolean;
}): Promise<{ action: string }> {
  return new Promise((resolve) => {
    notifier.notify(
      {
        title: options.title,
        message: options.message,
        icon: options.icon || path.join(app.getPath('userData'), 'icon.png'),
        sound: options.sound !== false,
        wait: options.wait || false,
        appID: app.getAppUserModelId(),
        
        // Windows-specific
        ...(process.platform === 'win32' ? {
          toastType: 'remind',
          actions: 'Reply|Dismiss',
          reply: true,
        } : {}),
      },
      (error, response, metadata) => {
        if (error) {
          console.error('Notification error:', error);
          resolve({ action: 'error' });
          return;
        }
        
        resolve({ action: response });
      }
    );
    
    notifier.on('click', () => {
      // Focus app
    });
    
    notifier.on('reply', (_, options, metadata) => {
      console.log('Reply:', metadata?.activationValue);
    });
  });
}
```

---

## 6. Progress Notifications

```typescript
// แสดง progress ใน notification (macOS/Linux)
function showProgressNotification(
  title: string,
  initialMessage: string
): {
  update: (progress: number, message?: string) => void;
  complete: (message?: string) => void;
  error: (message: string) => void;
} {
  const id = `progress-${Date.now()}`;
  
  notificationManager.show({
    id,
    title,
    body: initialMessage,
    persistent: true,
  });
  
  return {
    update(progress: number, message?: string) {
      notificationManager.close(id);
      notificationManager.show({
        id,
        title,
        body: message || `${Math.round(progress)}%`,
        persistent: true,
      });
    },
    
    complete(message?: string) {
      notificationManager.close(id);
      notificationManager.show({
        id: `${id}-complete`,
        title,
        body: message || 'Complete!',
        type: 'success',
        timeout: 5000,
      });
    },
    
    error(message: string) {
      notificationManager.close(id);
      notificationManager.show({
        id: `${id}-error`,
        title,
        body: message,
        type: 'error',
        timeout: 10000,
      });
    },
  };
}

// ตัวอย่าง
async function downloadFile(url: string, destPath: string): Promise<void> {
  const progress = showProgressNotification('Downloading...', 'Starting...');
  
  try {
    const response = await fetch(url);
    const total = parseInt(response.headers.get('content-length') || '0');
    let downloaded = 0;
    
    const reader = response.body!.getReader();
    
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      
      downloaded += value!.length;
      const percent = total > 0 ? (downloaded / total) * 100 : 0;
      progress.update(percent, `${Math.round(percent)}%`);
    }
    
    progress.complete('Download complete!');
  } catch (error: any) {
    progress.error(`Download failed: ${error.message}`);
    throw error;
  }
}
```

---

## 7. IPC Handlers

```typescript
// src/main/ipc-handlers.ts
import { ipcMain } from 'electron';
import { notificationManager } from './notifications/NotificationManager';

ipcMain.handle('notification:show', (_, options) => {
  notificationManager.show({
    ...options,
    onClick: () => {
      // Notify renderer
      BrowserWindow.getAllWindows()[0]?.webContents.send(
        'notification:clicked', 
        { id: options.id }
      );
    },
  });
});

ipcMain.handle('notification:close', (_, id: string) => {
  notificationManager.close(id);
});

ipcMain.handle('notification:closeAll', () => {
  notificationManager.closeAll();
});
```

---

## 8. สรุป

### Notification Features ตาม Platform

| Feature | Windows | macOS | Linux |
|---------|---------|-------|-------|
| Basic | ✅ | ✅ | ✅ |
| Actions | ❌ | ✅ | ⚠️ |
| Reply | ❌ | ✅ | ❌ |
| Persistent | ✅ | ✅ | ⚠️ |
| Progress | ❌ | ❌ | ❌ |
| Grouping | ✅ | ✅ | ❌ |
| Sound | ✅ | ✅ | ✅ |

### Best Practices

1. ตรวจสอบ `Notification.isSupported()` ก่อนแสดง
2. ใช้ unique IDs เพื่อจัดการ notifications
3. อย่าแสดง notifications มากเกินไป
4. ขอ permission ถ้า app อยู่ใน sandbox
5. Test บน target OS

---

*จบ Part 055 - ต่อไป Part 056: Menu Bar App (macOS)*
