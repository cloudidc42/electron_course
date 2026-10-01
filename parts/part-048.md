# Part 048: In-App Purchases (macOS)
## การทำ In-App Purchases ใน Electron บน macOS

---

## 🎯 เป้าหมายของบทเรียนนี้

- inAppPurchase module ของ Electron
- แสดงรายการ products
- Payment queue
- ตรวจสอบ receipts
- Sandbox testing
- Subscription management

---

## 1. ข้อกำหนดเบื้องต้น

```
1. macOS แอพที่ distribute ผ่าน Mac App Store เท่านั้น
2. Apple Developer Account
3. App ต้องมี App ID ใน App Store Connect
4. ต้องตั้งค่า In-App Purchases ใน App Store Connect
5. ต้องผ่าน App Review
```

> **หมายเหตุ**: inAppPurchase API ใน Electron ทำงานได้เฉพาะแอพที่ distribute ผ่าน Mac App Store เท่านั้น ไม่ใช่ distribute ด้วยตัวเอง

---

## 2. ตั้งค่าใน App Store Connect

### ประเภท In-App Purchase

```
1. Consumable
   - ซื้อได้หลายครั้ง
   - ตัวอย่าง: เหรียญ, credits, tokens
   
2. Non-Consumable
   - ซื้อครั้งเดียว ใช้ได้ตลอด
   - ตัวอย่าง: premium features, remove ads
   
3. Auto-Renewable Subscriptions
   - ต่ออายุอัตโนมัติ
   - ตัวอย่าง: monthly/yearly subscription
   
4. Non-Renewing Subscriptions
   - Subscription แบบไม่ต่ออายุ
   - ตัวอย่าง: 1-month access (ต้อง renew เอง)
```

---

## 3. Main Process - In-App Purchase Handler

### src/main/iap.ts

```typescript
import { inAppPurchase, app, BrowserWindow } from 'electron';
import { ipcMain } from 'electron';

// Product IDs จาก App Store Connect
const PRODUCT_IDS = [
  'com.yourcompany.app.premium',
  'com.yourcompany.app.credits100',
  'com.yourcompany.app.subscription_monthly',
  'com.yourcompany.app.subscription_yearly',
];

export class InAppPurchaseManager {
  private purchaseListeners: Array<() => void> = [];
  
  constructor() {
    this.setupTransactionObserver();
  }

  // ==========================================
  // ตรวจสอบว่า IAP พร้อมใช้งาน
  // ==========================================
  
  canMakePurchases(): boolean {
    return inAppPurchase.canMakePayments();
  }
  
  // ==========================================
  // ดึงรายการ Products
  // ==========================================
  
  async getProducts(productIds: string[] = PRODUCT_IDS): Promise<Electron.Product[]> {
    if (!this.canMakePurchases()) {
      throw new Error('In-app purchases are not available');
    }
    
    return new Promise((resolve, reject) => {
      inAppPurchase.getProducts(productIds, (products) => {
        if (!products || products.length === 0) {
          reject(new Error('No products found'));
          return;
        }
        
        resolve(products);
      });
    });
  }
  
  // ==========================================
  // ซื้อ Product
  // ==========================================
  
  async purchaseProduct(productId: string, quantity: number = 1): Promise<boolean> {
    if (!this.canMakePurchases()) {
      throw new Error('Purchases not available');
    }
    
    return new Promise((resolve, reject) => {
      const success = inAppPurchase.purchaseProduct(productId, quantity, (isValid) => {
        resolve(isValid);
      });
      
      if (!success) {
        reject(new Error('Failed to initiate purchase'));
      }
    });
  }
  
  // ==========================================
  // Transaction Observer
  // ==========================================
  
  private setupTransactionObserver(): void {
    // Listen for transaction updates
    inAppPurchase.on('transactions-updated', (event, transactions) => {
      transactions.forEach(transaction => {
        this.handleTransaction(transaction);
      });
    });
  }
  
  private async handleTransaction(transaction: Electron.Transaction): Promise<void> {
    const { transactionState, payment } = transaction;
    const productId = payment.productIdentifier;
    
    console.log(`Transaction: ${productId} - State: ${transactionState}`);
    
    switch (transactionState) {
      case 'purchasing':
        // กำลังซื้อ
        this.notifyRenderers('iap:purchasing', { productId });
        break;
        
      case 'purchased':
        // ซื้อสำเร็จ
        try {
          // ตรวจสอบ receipt
          const isValid = await this.verifyReceipt();
          
          if (isValid) {
            // Deliver product
            await this.deliverProduct(productId);
            this.notifyRenderers('iap:purchased', { productId, success: true });
          } else {
            this.notifyRenderers('iap:purchased', { 
              productId, 
              success: false,
              error: 'Receipt validation failed' 
            });
          }
          
          // Finish transaction
          inAppPurchase.finishTransactionByDate(transaction.transactionDate);
        } catch (error) {
          console.error('Purchase handling error:', error);
        }
        break;
        
      case 'failed':
        // ซื้อล้มเหลว
        const error = transaction.errorCode 
          ? `Error ${transaction.errorCode}` 
          : 'Purchase failed';
        this.notifyRenderers('iap:failed', { productId, error });
        
        // Finish failed transaction
        inAppPurchase.finishTransactionByDate(transaction.transactionDate);
        break;
        
      case 'restored':
        // Restore ซื้อเก่า
        try {
          await this.deliverProduct(productId);
          this.notifyRenderers('iap:restored', { productId });
          inAppPurchase.finishTransactionByDate(transaction.transactionDate);
        } catch (error) {
          console.error('Restore handling error:', error);
        }
        break;
        
      case 'deferred':
        // รออนุมัติ (เด็กต้องขออนุญาต parent)
        this.notifyRenderers('iap:deferred', { productId });
        break;
    }
  }
  
  // ==========================================
  // Receipt Verification
  // ==========================================
  
  async verifyReceipt(): Promise<boolean> {
    const receiptURL = inAppPurchase.getReceiptURL();
    
    if (!receiptURL) {
      return false;
    }
    
    try {
      // อ่าน receipt file
      const fs = require('fs');
      const receiptData = fs.readFileSync(receiptURL);
      const receiptBase64 = receiptData.toString('base64');
      
      // ส่งไปตรวจสอบกับ Apple (หรือ server ของคุณ)
      const isValid = await this.validateReceiptWithServer(receiptBase64);
      
      return isValid;
    } catch (error) {
      console.error('Receipt verification error:', error);
      return false;
    }
  }
  
  private async validateReceiptWithServer(receiptData: string): Promise<boolean> {
    // ส่งไปยัง backend server ของคุณ
    // NEVER validate directly with Apple from client (security risk)
    try {
      const response = await fetch('https://api.yourapp.com/verify-receipt', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.getAuthToken()}`,
        },
        body: JSON.stringify({
          receipt: receiptData,
          isSandbox: this.isSandboxMode(),
        }),
      });
      
      const result = await response.json();
      return result.valid === true;
    } catch (error) {
      console.error('Server validation error:', error);
      return false;
    }
  }
  
  // ==========================================
  // Deliver Product
  // ==========================================
  
  private async deliverProduct(productId: string): Promise<void> {
    // บันทึก purchase ใน local storage
    const store = require('electron-store');
    const purchaseStore = new store({ name: 'purchases' });
    
    const purchases = purchaseStore.get('purchases', {}) as Record<string, any>;
    
    if (productId.includes('subscription')) {
      // Subscription
      purchases[productId] = {
        active: true,
        purchasedAt: new Date().toISOString(),
        expiresAt: this.calculateExpiry(productId),
      };
    } else if (productId.includes('credits')) {
      // Consumable - เพิ่ม credits
      const amount = this.getCreditsAmount(productId);
      purchases.credits = (purchases.credits || 0) + amount;
    } else {
      // Non-consumable
      purchases[productId] = {
        purchased: true,
        purchasedAt: new Date().toISOString(),
      };
    }
    
    purchaseStore.set('purchases', purchases);
  }
  
  // ==========================================
  // Restore Purchases
  // ==========================================
  
  restorePurchases(): void {
    inAppPurchase.restoreCompletedTransactions();
  }
  
  // ==========================================
  // Sandbox Mode Detection
  // ==========================================
  
  isSandboxMode(): boolean {
    const receiptURL = inAppPurchase.getReceiptURL();
    return receiptURL?.includes('sandboxReceipt') || !app.isPackaged;
  }
  
  // ==========================================
  // Helper Methods
  // ==========================================
  
  private notifyRenderers(channel: string, data: any): void {
    BrowserWindow.getAllWindows().forEach(win => {
      win.webContents.send(channel, data);
    });
  }
  
  private getAuthToken(): string {
    return process.env.API_TOKEN || '';
  }
  
  private calculateExpiry(productId: string): string {
    const now = new Date();
    if (productId.includes('yearly')) {
      now.setFullYear(now.getFullYear() + 1);
    } else {
      now.setMonth(now.getMonth() + 1);
    }
    return now.toISOString();
  }
  
  private getCreditsAmount(productId: string): number {
    const match = productId.match(/credits(\d+)/);
    return match ? parseInt(match[1]) : 0;
  }
}

// Singleton
export const iapManager = new InAppPurchaseManager();
```

---

## 4. IPC Handlers

```typescript
// src/main/index.ts
import { ipcMain } from 'electron';
import { iapManager } from './iap';

// ตรวจสอบว่า IAP พร้อมใช้งาน
ipcMain.handle('iap:canMakePurchases', () => {
  return iapManager.canMakePurchases();
});

// ดึงรายการ products
ipcMain.handle('iap:getProducts', async (_, productIds?: string[]) => {
  try {
    const products = await iapManager.getProducts(productIds);
    return { success: true, products };
  } catch (error: any) {
    return { success: false, error: error.message };
  }
});

// ซื้อ product
ipcMain.handle('iap:purchase', async (_, { productId, quantity }) => {
  try {
    const success = await iapManager.purchaseProduct(productId, quantity);
    return { success };
  } catch (error: any) {
    return { success: false, error: error.message };
  }
});

// Restore purchases
ipcMain.handle('iap:restore', () => {
  iapManager.restorePurchases();
  return { success: true };
});

// ตรวจสอบ receipt
ipcMain.handle('iap:verifyReceipt', async () => {
  const isValid = await iapManager.verifyReceipt();
  return { valid: isValid };
});
```

---

## 5. Renderer - Store Component

### src/renderer/components/Store.tsx

```tsx
import React, { useState, useEffect } from 'react';

interface Product {
  productIdentifier: string;
  localizedTitle: string;
  localizedDescription: string;
  formattedPrice: string;
  priceLocale: string;
  subscriptionPeriod?: string;
}

function Store() {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  const [purchasing, setPurchasing] = useState<string | null>(null);
  const [canPurchase, setCanPurchase] = useState(false);

  useEffect(() => {
    async function init() {
      // ตรวจสอบ
      const canBuy = await window.electronAPI.invoke('iap:canMakePurchases');
      setCanPurchase(canBuy);
      
      if (canBuy) {
        try {
          const { products } = await window.electronAPI.invoke('iap:getProducts');
          setProducts(products);
        } catch (error) {
          console.error('Failed to load products:', error);
        }
      }
      
      setLoading(false);
    }
    
    init();
    
    // Listen for purchase events
    window.electronAPI.on('iap:purchased', (data: any) => {
      setPurchasing(null);
      if (data.success) {
        alert(`Purchase successful: ${data.productId}`);
      } else {
        alert(`Purchase failed: ${data.error}`);
      }
    });
    
    window.electronAPI.on('iap:failed', (data: any) => {
      setPurchasing(null);
      alert(`Purchase failed: ${data.error}`);
    });
    
    window.electronAPI.on('iap:restored', (data: any) => {
      alert(`Restored: ${data.productId}`);
    });
  }, []);

  const handlePurchase = async (productId: string) => {
    setPurchasing(productId);
    
    try {
      await window.electronAPI.invoke('iap:purchase', { productId, quantity: 1 });
    } catch (error) {
      setPurchasing(null);
      alert(`Error: ${error}`);
    }
  };

  const handleRestore = async () => {
    await window.electronAPI.invoke('iap:restore');
    alert('Restoring purchases...');
  };

  if (loading) return <div>Loading store...</div>;
  
  if (!canPurchase) {
    return <div>In-app purchases are not available on your device.</div>;
  }

  return (
    <div style={{ padding: '20px' }}>
      <h2>Store</h2>
      
      {products.map(product => (
        <div key={product.productIdentifier} style={{
          border: '1px solid #ddd',
          padding: '16px',
          borderRadius: '8px',
          marginBottom: '12px',
        }}>
          <h3>{product.localizedTitle}</h3>
          <p>{product.localizedDescription}</p>
          <p style={{ fontWeight: 'bold', color: '#2196F3' }}>{product.formattedPrice}</p>
          {product.subscriptionPeriod && (
            <p style={{ color: '#666', fontSize: '12px' }}>
              Per {product.subscriptionPeriod}
            </p>
          )}
          
          <button
            onClick={() => handlePurchase(product.productIdentifier)}
            disabled={purchasing !== null}
          >
            {purchasing === product.productIdentifier ? 'Purchasing...' : 'Buy'}
          </button>
        </div>
      ))}
      
      <button
        onClick={handleRestore}
        style={{ marginTop: '16px', opacity: 0.7 }}
      >
        Restore Purchases
      </button>
    </div>
  );
}

export default Store;
```

---

## 6. Sandbox Testing

### สร้าง Sandbox Test Account

1. ไปที่ App Store Connect → Users and Access → Sandbox
2. สร้าง sandbox tester account
3. ใน Xcode หรือ simulator → Sign out จาก App Store
4. Sign in ด้วย sandbox account

### ตรวจสอบ Sandbox Mode

```typescript
// ตรวจสอบว่าอยู่ใน sandbox หรือไม่
const isSandbox = inAppPurchase.getReceiptURL()?.includes('sandboxReceipt');

// ใน development mode มักจะเป็น sandbox เสมอ
if (!app.isPackaged) {
  console.log('Running in sandbox/development mode');
}
```

---

## 7. Server-side Receipt Validation

### Node.js Backend

```javascript
// backend/receipt-validator.js
const axios = require('axios');

const APPLE_VERIFY_URL = 'https://buy.itunes.apple.com/verifyReceipt';
const APPLE_SANDBOX_URL = 'https://sandbox.itunes.apple.com/verifyReceipt';

async function verifyReceipt(receiptData, isSandbox = false) {
  const url = isSandbox ? APPLE_SANDBOX_URL : APPLE_VERIFY_URL;
  
  try {
    const response = await axios.post(url, {
      'receipt-data': receiptData,
      'password': process.env.APP_SHARED_SECRET, // จาก App Store Connect
      'exclude-old-transactions': true,
    });
    
    const { status, receipt, latest_receipt_info } = response.data;
    
    // status 0 = valid
    if (status === 21007) {
      // Sandbox receipt ส่งไป production → ลอง sandbox แทน
      return verifyReceipt(receiptData, true);
    }
    
    if (status !== 0) {
      return { valid: false, error: `Apple status: ${status}` };
    }
    
    // ตรวจสอบ subscription expiry
    const activeSubscriptions = (latest_receipt_info || [])
      .filter(info => {
        const expiryDate = new Date(parseInt(info.expires_date_ms));
        return expiryDate > new Date();
      });
    
    return {
      valid: true,
      receipt,
      activeSubscriptions,
      hasActiveSubscription: activeSubscriptions.length > 0,
    };
  } catch (error) {
    return { valid: false, error: error.message };
  }
}

module.exports = { verifyReceipt };
```

---

## 8. สรุป

### IAP Flow

```
1. แสดง products ที่ได้จาก App Store Connect
2. User เลือกซื้อ
3. Apple แสดง payment dialog
4. Transaction ถูกส่งกลับมา
5. ตรวจสอบ receipt กับ Apple/server
6. Deliver product ถ้า valid
7. Finish transaction
```

### ข้อควรระวัง

- ทดสอบด้วย Sandbox account เสมอก่อน production
- ตรวจสอบ receipt ที่ server เสมอ (ไม่ใช่ client)
- Handle network errors อย่างถูกต้อง
- เก็บ purchases ใน secure storage
- Test restore functionality

---

*จบ Part 048 - ต่อไป Part 049: Monitoring & Logging*
