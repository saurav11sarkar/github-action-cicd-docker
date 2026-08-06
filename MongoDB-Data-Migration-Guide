# MongoDB Data Migration Guide

**Developer MongoDB → Client MongoDB**

এই guide ব্যবহার করে Developer MongoDB-এর সম্পূর্ণ database data নিরাপদভাবে Client MongoDB-তে migrate করা যাবে।

---

## 🔄 Migration Flow

```text
Developer MongoDB
       │
       │ mongodump
       ▼
Local Backup
       │
       │ mongorestore
       ▼
Client MongoDB
       │
       ▼
Update .env
       │
       ▼
Restart & Test Project
```

---

## 1. MongoDB Database Tools Install

MongoDB migration করার জন্য `mongodump` এবং `mongorestore` প্রয়োজন।

### Official Download

[MongoDB Database Tools — Official Download](https://www.mongodb.com/try/download/database-tools)

### Windows-এর জন্য

Download page থেকে:

```text
Platform → Windows
Package  → MSI
```

তারপর `.msi` file install করো। Install করার পর PowerShell বন্ধ করে আবার open করো।

### Check Installation

```powershell
mongodump --version
mongorestore --version
```

Version দেখালে installation complete। ✅

### যদি `mongodump is not recognized` দেখায়

MongoDB Tools কোথায় install হয়েছে সেটা খুঁজে বের করো:

```powershell
Get-ChildItem "C:\Program Files\MongoDB" -Recurse -Filter mongodump.exe -ErrorAction SilentlyContinue
```

সাধারণত path হবে:

```text
C:\Program Files\MongoDB\Tools\100\bin\mongodump.exe
```

সরাসরি test করতে:

```powershell
& "C:\Program Files\MongoDB\Tools\100\bin\mongodump.exe" --version
```

Version দেখালে MongoDB Tools ঠিকমতো install হয়েছে — শুধু Windows PATH-এ add করা নেই।

---

## 2. Developer & Client MongoDB URI সংগ্রহ

Migration-এর জন্য দুইটি MongoDB URI লাগবে।

**Developer MongoDB**

```text
<DEVELOPER_MONGO_URI>
```

**Client MongoDB**

```text
<CLIENT_MONGO_URI>
```

উদাহরণ ফরম্যাট:

```text
mongodb+srv://USERNAME:PASSWORD@CLUSTER/DATABASE
```

> ⚠️ **Important:** Real username/password কখনো README, GitHub বা public জায়গায় রাখবে না।

---

## 3. Developer MongoDB থেকে Backup নেওয়া

প্রথমে Developer MongoDB-এর সব data local computer-এ backup নিতে হবে।

```powershell
& "C:\Program Files\MongoDB\Tools\100\bin\mongodump.exe" --uri="<DEVELOPER_MONGO_URI>" --out=.\mongo-backup
```

উদাহরণ:

```powershell
& "C:\Program Files\MongoDB\Tools\100\bin\mongodump.exe" --uri="mongodb+srv://USERNAME:PASSWORD@CLUSTER/DATABASE" --out=.\mongo-backup
```

---

## 4. Backup Successfully হয়েছে কিনা Check

Backup successful হলে terminal-এ এরকম message দেখতে পারো:

```text
done dumping `database.users`
done dumping `database.projects`
done dumping `database.services`
```

এবং current folder-এ তৈরি হবে:

```text
mongo-backup/
└── DATABASE_NAME/
```

যেমন:

```text
mongo-backup/
└── peterbasillios/
    ├── users.bson
    ├── projects.bson
    ├── services.bson
    ├── contacts.bson
    └── notifications.bson
```

Backup check করতে:

```powershell
Get-ChildItem .\mongo-backup
Get-ChildItem .\mongo-backup\<DATABASE_NAME>
```

`.bson` files দেখতে পেলে backup ঠিকভাবে তৈরি হয়েছে। ✅

---

## 5. Client MongoDB Check

Restore করার আগে Client MongoDB check করা **খুব গুরুত্বপূর্ণ**।

- Client MongoDB empty হলে → সরাসরি restore করা যাবে।
- Client MongoDB-তে data থাকলে → আগে check করো সেই data দরকার আছে কিনা।
  - Important data থাকলে → `--drop` ব্যবহার করবে না।
  - Test/old data হলে → `--drop` ব্যবহার করে replace করা যেতে পারে।

---

## 6. Client MongoDB-তে Data Restore

Client MongoDB empty থাকলে:

```powershell
& "C:\Program Files\MongoDB\Tools\100\bin\mongorestore.exe" --uri="<CLIENT_MONGO_URI>" .\mongo-backup\<DATABASE_NAME>
```

উদাহরণ:

```powershell
& "C:\Program Files\MongoDB\Tools\100\bin\mongorestore.exe" --uri="mongodb+srv://USERNAME:PASSWORD@CLUSTER/DATABASE" .\mongo-backup\DATABASE_NAME
```

এতে Developer-এর backup Client MongoDB-তে চলে যাবে।

---

## 7. Client-এর Existing Data Replace করতে চাইলে

Client database-এর পুরোনো/test data remove করে Developer-এর data বসাতে চাইলে:

```powershell
& "C:\Program Files\MongoDB\Tools\100\bin\mongorestore.exe" --drop --uri="<CLIENT_MONGO_URI>" .\mongo-backup\<DATABASE_NAME>
```

### `--drop` কী করে?

```text
Existing Client Collection
          ↓
       Remove
          ↓
Developer Backup
          ↓
       Restore
```

অর্থাৎ একই নামের existing collection আগে remove হবে, তারপর backup-এর collection restore হবে।

> ⚠️ **সতর্কতা:** Client database-এ important/production data থাকলে confirmation ছাড়া `--drop` ব্যবহার করবে না।

---

## 8. Migration Verify করা

Restore শেষ হলে MongoDB Atlas বা MongoDB Compass থেকে Client database open করো। Developer এবং Client-এর collections compare করো।

```text
Developer              Client
---------              ------
users           →      users
projects        →      projects
services        →      services
contacts        →      contacts
notifications   →      notifications
payments        →      payments
subscribes      →      subscribes
```

### Document Count মিলিয়ে দেখো

```text
Developer                Client
---------                ------
users = 1          →     users = 1
projects = 5       →     projects = 5
services = 5       →     services = 5
contacts = 7       →     contacts = 7
notifications = 6  →     notifications = 6
```

যে collections migrate করেছো, সেগুলোর data count মিলছে কিনা check করো।

---

## 9. Project-এর `.env` Update

Migration complete হওয়ার পরে project-এর `.env` file-এ Developer MongoDB-এর পরিবর্তে Client MongoDB URI দিতে হবে।

আগে:

```env
MONGO_URI=<DEVELOPER_MONGO_URI>
```

পরে:

```env
MONGO_URI=<CLIENT_MONGO_URI>
```

Source code-এর ভিতরে MongoDB URI লিখবে না।

---

## 10. Backend Restart

`.env` update করার পরে backend restart করতে হবে।

**Development**

```bash
npm run dev
```

**Production**

```bash
npm run build
npm start
```

**PM2**

```bash
pm2 restart <app-name>
```

**Docker**

```bash
docker compose up -d --build
```

Project অনুযায়ী যেটা applicable সেটা ব্যবহার করবে।

---

## 11. Application Test

Client MongoDB connect হওয়ার পরে application-এর গুরুত্বপূর্ণ functionality test করো:

- [ ] Login
- [ ] Register
- [ ] User data
- [ ] Profile
- [ ] Projects
- [ ] Services
- [ ] Contacts
- [ ] Notifications
- [ ] Payments
- [ ] অন্যান্য database-related APIs

Existing data ঠিকভাবে load হচ্ছে কিনা এবং নতুন data create/update হচ্ছে কিনা check করো।

---

## 12. Final Migration Checklist

- [ ] MongoDB Database Tools install করা হয়েছে
- [ ] `mongodump` কাজ করছে
- [ ] `mongorestore` কাজ করছে
- [ ] Developer MongoDB connection ঠিক আছে
- [ ] Client MongoDB connection ঠিক আছে
- [ ] Developer database backup নেওয়া হয়েছে
- [ ] Backup files verify করা হয়েছে
- [ ] Client database আগে check করা হয়েছে
- [ ] প্রয়োজন অনুযায়ী restore করা হয়েছে
- [ ] `--drop` শুধুমাত্র প্রয়োজন হলে ব্যবহার করা হয়েছে
- [ ] Developer ও Client collection compare করা হয়েছে
- [ ] Document count compare করা হয়েছে
- [ ] Project `.env` update করা হয়েছে
- [ ] Backend restart করা হয়েছে
- [ ] Login test করা হয়েছে
- [ ] Existing data test করা হয়েছে
- [ ] Main APIs test করা হয়েছে
- [ ] Application successfully running

---

## 🚀 Quick Migration Process

Future project-এ পুরো process সংক্ষেপে:

**Step 1 — Tools Install**

[Download MongoDB Database Tools](https://www.mongodb.com/try/download/database-tools)

**Step 2 — Developer Backup**

```powershell
& "C:\Program Files\MongoDB\Tools\100\bin\mongodump.exe" --uri="<DEVELOPER_MONGO_URI>" --out=.\mongo-backup
```

**Step 3 — Backup Check**

```powershell
Get-ChildItem .\mongo-backup\<DATABASE_NAME>
```

**Step 4 — Client Restore**

```powershell
& "C:\Program Files\MongoDB\Tools\100\bin\mongorestore.exe" --uri="<CLIENT_MONGO_URI>" .\mongo-backup\<DATABASE_NAME>
```

**Step 5 — Existing Client Data Replace করতে হলে**

```powershell
& "C:\Program Files\MongoDB\Tools\100\bin\mongorestore.exe" --drop --uri="<CLIENT_MONGO_URI>" .\mongo-backup\<DATABASE_NAME>
```

**Step 6 — `.env` Update**

```env
MONGO_URI=<CLIENT_MONGO_URI>
```

**Step 7 — Backend Restart & Test**

```text
Restart → Login → Existing Data Check → API Test → Done ✅
```

---

## 🔐 Security

Real MongoDB credentials কখনো:

- GitHub-এ রাখবে না
- README-তে রাখবে না
- Source code-এ রাখবে না
- Public screenshot-এ রাখবে না
- Client-এর সাথে প্রয়োজন ছাড়া share করবে না

README-তে সবসময় placeholder ব্যবহার করবে:

```text
<DEVELOPER_MONGO_URI>
<CLIENT_MONGO_URI>
```

Actual credentials `.env` অথবা secure secret manager-এ রাখবে। যদি MongoDB username/password accidentally public হয়ে যায়, password **immediately change/rotate** করবে।

---

## ✅ Final Summary

MongoDB migration-এর মূল বিষয় খুব simple:

```text
Developer MongoDB
        ↓
     mongodump
        ↓
   Local Backup
        ↓
   mongorestore
        ↓
   Client MongoDB
        ↓
 Update .env
        ↓
 Restart Backend
        ↓
      Test
        ↓
   Migration Done ✅
```

**মনে রাখবে:**

- `mongodump` = Developer MongoDB থেকে data বের করে backup নেওয়া।
- `mongorestore` = সেই backup Client MongoDB-তে নিয়ে যাওয়া।
- `--drop` = Client-এর একই নামের পুরোনো collection remove করে backup-এর data বসানো।
