# 🔥 Configuração Firebase - Loja 3F

## 📋 Pré-requisitos

1. Criar projeto no [Firebase Console](https://console.firebase.google.com/)
2. Ativar Authentication, Firestore Database e Storage

## 🔐 Configuração de Autenticação

```javascript
// Adicionar no arquivo js/firebase-config.js

import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getFirestore } from 'firebase/firestore';
import { getStorage } from 'firebase/storage';

const firebaseConfig = {
  apiKey: "SUA_API_KEY",
  authDomain: "SEU_PROJETO.firebaseapp.com",
  projectId: "SEU_PROJETO_ID",
  storageBucket: "SEU_PROJETO.appspot.com",
  messagingSenderId: "SEU_SENDER_ID",
  appId: "SEU_APP_ID"
};

const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);
export const db = getFirestore(app);
export const storage = getStorage(app);
```

## 📊 Estrutura do Firestore

### Coleção: `products`
```json
{
  "id": "number",
  "name": "string",
  "category": "string",
  "price": "number",
  "description": "string",
  "mainImage": "string (Storage URL)",
  "stockBySizes": {
    "P": "number",
    "M": "number",
    "G": "number",
    "GG": "number"
  },
  "createdAt": "timestamp"
}
```

### Coleção: `categories`
```json
{
  "id": "number",
  "name": "string",
  "slug": "string"
}
```

### Coleção: `orders`
```json
{
  "id": "string",
  "number": "string",
  "date": "timestamp",
  "status": "string",
  "customer": {
    "name": "string",
    "email": "string",
    "phone": "string"
  },
  "items": "array",
  "total": "number"
}
```

### Coleção: `promotions`
```json
{
  "id": "string",
  "productId": "number",
  "discountPercent": "number",
  "active": "boolean",
  "startDate": "timestamp",
  "endDate": "timestamp"
}
```

### Coleção: `settings`
```json
{
  "payment": {
    "installments": "number",
    "pixDiscount": "number"
  },
  "stock": {
    "lowStockAlert": "number"
  },
  "slider": "array"
}
```

## 🔒 Regras de Segurança Firestore

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    // Produtos - leitura pública, escrita apenas admin
    match /products/{productId} {
      allow read: if true;
      allow write: if request.auth != null && 
                     get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }
    
    // Categorias - leitura pública, escrita apenas admin
    match /categories/{categoryId} {
      allow read: if true;
      allow write: if request.auth != null && 
                     get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }
    
    // Pedidos - apenas admin
    match /orders/{orderId} {
      allow read, write: if request.auth != null && 
                           get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }
    
    // Promoções - leitura pública, escrita apenas admin
    match /promotions/{promoId} {
      allow read: if true;
      allow write: if request.auth != null && 
                     get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }
    
    // Configurações - leitura pública, escrita apenas admin
    match /settings/{settingId} {
      allow read: if true;
      allow write: if request.auth != null && 
                     get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }
    
    // Usuários
    match /users/{userId} {
      allow read: if request.auth != null && request.auth.uid == userId;
      allow write: if request.auth != null && 
                     get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
    }
  }
}
```

## 📦 Regras de Segurança Storage

```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /products/{allPaths=**} {
      allow read: if true;
      allow write: if request.auth != null;
    }
    
    match /slider/{allPaths=**} {
      allow read: if true;
      allow write: if request.auth != null;
    }
  }
}
```

## 🔄 Migração de LocalStorage para Firebase

### Substituir em `js/products.js`:
```javascript
// Antes
function getAllProducts() {
    const saved = localStorage.getItem(PRODUCTS_KEY);
    return saved ? JSON.parse(saved) : mockProducts;
}

// Depois
import { db } from './firebase-config.js';
import { collection, getDocs } from 'firebase/firestore';

async function getAllProducts() {
    const snapshot = await getDocs(collection(db, 'products'));
    return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}
```

### Substituir em `js/orders.js`:
```javascript
// Antes
function getAllOrders() {
    const saved = localStorage.getItem(ORDERS_KEY);
    return saved ? JSON.parse(saved) : [];
}

// Depois
import { db } from './firebase-config.js';
import { collection, getDocs, query, orderBy } from 'firebase/firestore';

async function getAllOrders() {
    const q = query(collection(db, 'orders'), orderBy('date', 'desc'));
    const snapshot = await getDocs(q);
    return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}
```

## 📸 Upload de Imagens para Storage

```javascript
import { storage } from './firebase-config.js';
import { ref, uploadBytes, getDownloadURL } from 'firebase/storage';

async function uploadProductImage(file) {
    const storageRef = ref(storage, `products/${Date.now()}_${file.name}`);
    const snapshot = await uploadBytes(storageRef, file);
    const downloadURL = await getDownloadURL(snapshot.ref);
    return downloadURL;
}
```

## 📊 Relatório de Vendas com Dados Reais

```javascript
import { db } from './firebase-config.js';
import { collection, query, where, getDocs, Timestamp } from 'firebase/firestore';

async function getSalesReport(period) {
    const now = new Date();
    let startDate;
    
    switch(period) {
        case 'yesterday':
            startDate = new Date(now.setDate(now.getDate() - 1));
            break;
        case '7days':
            startDate = new Date(now.setDate(now.getDate() - 7));
            break;
        case '30days':
            startDate = new Date(now.setDate(now.getDate() - 30));
            break;
    }
    
    const q = query(
        collection(db, 'orders'),
        where('date', '>=', Timestamp.fromDate(startDate)),
        where('status', '!=', 'cancelled')
    );
    
    const snapshot = await getDocs(q);
    const orders = snapshot.docs.map(doc => doc.data());
    
    return {
        total: orders.reduce((sum, order) => sum + order.total, 0) / 100,
        orders: orders.length
    };
}
```

## ✅ Checklist de Implementação

- [ ] Criar projeto no Firebase
- [ ] Ativar Authentication (Email/Password)
- [ ] Ativar Firestore Database
- [ ] Ativar Storage
- [ ] Configurar regras de segurança
- [ ] Adicionar firebase-config.js
- [ ] Migrar funções de products.js
- [ ] Migrar funções de orders.js
- [ ] Migrar funções de categories.js
- [ ] Migrar funções de promotions.js
- [ ] Migrar funções de settings.js
- [ ] Implementar upload de imagens
- [ ] Testar autenticação admin
- [ ] Testar CRUD de produtos
- [ ] Testar relatório de vendas

## 🚀 Deploy

```bash
# Instalar Firebase CLI
npm install -g firebase-tools

# Login
firebase login

# Inicializar projeto
firebase init

# Deploy
firebase deploy
```

---

**Após conectar ao Firebase, todas as funcionalidades do painel admin estarão 100% funcionais com dados persistentes!** 🔥
