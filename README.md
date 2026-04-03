# 🚀 NestJS Backend — Production Ready Docker + CI/CD (A to Z)

> **NestJS + Prisma + PostgreSQL + MongoDB + Redis** এর জন্য সম্পূর্ণ Production Ready Setup।
> প্রত্যেক `git push` এ `main` ব্রাঞ্চে অটোমেটিক **Docker Build → Push → VPS Deploy** হয়।
> এই README শুধু একবার পড়ো — তারপর যেকোনো প্রজেক্টে Docker দিয়ে deploy করতে পারবে।

---

## 📋 Table of Contents

1. [প্রত্যেকটা জিনিস কী কাজ করে?](#1-প্রত্যেকটা-জিনিস-কী-কাজ-করে)
2. [প্রজেক্ট ফাইল স্ট্রাকচার](#2-প্রজেক্ট-ফাইল-স্ট্রাকচার)
3. [Prerequisites — আগে কী কী লাগবে?](#3-prerequisites)
4. [.env ফাইল এবং Config Setup](#4-env-ফাইল-এবং-config-setup)
5. [Dockerfile — কীভাবে তৈরি করব?](#5-dockerfile)
6. [.dockerignore — কী বাদ দেব?](#6-dockerignore)
7. [docker-compose.yml — সব একসাথে চালানো](#7-docker-composeyml)
8. [Docker Commands — সব দরকারি কমান্ড](#8-docker-commands)
9. [Local এ Docker দিয়ে Test করা](#9-local-এ-docker-দিয়ে-test-করা)
10. [Docker Hub Setup](#10-docker-hub-setup)
11. [VPS Server Setup — সব ধাপে ধাপে](#11-vps-server-setup)
12. [GitHub Repository Setup](#12-github-repository-setup)
13. [GitHub Secrets Setup](#13-github-secrets-setup)
14. [GitHub Actions CI/CD Pipeline](#14-github-actions-cicd-pipeline)
15. [প্রথম Deploy করা](#15-প্রথম-deploy)
16. [CI/CD কীভাবে কাজ করে?](#16-cicd-কীভাবে-কাজ-করে)
17. [Server এ Useful Commands](#17-server-এ-useful-commands)
18. [Troubleshooting — সমস্যা সমাধান](#18-troubleshooting)
19. [Security Best Practices](#19-security-best-practices)
20. [Production Go-Live Checklist](#20-production-go-live-checklist)

---

## 1. প্রত্যেকটা জিনিস কী কাজ করে?

### 🐳 Docker কী?
Docker হলো একটা tool যা তোমার অ্যাপকে একটা **container** এ চালায়। Container মানে একটা ছোট বাক্স যেখানে তোমার অ্যাপ + সব দরকারি সফটওয়্যার (Node.js, OS libraries) একসাথে থাকে। ফলে তোমার লোকালে যাই চলুক, server-এও একইভাবে চলবে।

| জিনিস | কাজ | কোথায় থাকে |
|---|---|---|
| **Dockerfile** | তোমার NestJS কোড থেকে Docker Image বানায় | প্রজেক্ট রুটে |
| **docker-compose.yml** | App + PostgreSQL + MongoDB + Redis একসাথে চালায় | প্রজেক্ট রুটে |
| **.dockerignore** | Image-এ অপ্রয়োজনীয় ফাইল না নেওয়ার তালিকা | প্রজেক্ট রুটে |
| **GitHub Actions** | Push হলে automatically build + deploy করে | `.github/workflows/` ফোল্ডারে |
| **Docker Hub** | তোমার built image সংরক্ষণ করে (cloud) | hub.docker.com |
| **VPS** | তোমার অ্যাপ চালানোর Linux server | DigitalOcean/Hetzner/AWS |

### 🗄️ Database গুলো কী কাজ করে?

| Database | কাজ | কোন port |
|---|---|---|
| **PostgreSQL** | Relational data — users, orders, products (Prisma দিয়ে ব্যবহার করবে) | 5432 |
| **MongoDB** | NoSQL data — logs, files, unstructured documents | 27017 |
| **Redis** | Caching, rate limiting, session store (খুব দ্রুত) | 6379 |

---

## 2. প্রজেক্ট ফাইল স্ট্রাকচার

```
backend/
├── .github/
│   └── workflows/
│       └── nestjs-cicd.yml          ← GitHub Actions CI/CD Pipeline
│
├── src/
│   ├── app/
│   │   ├── config/
│   │   │   └── index.ts             ← .env ভ্যালু এখান থেকে পড়া হয়
│   │   ├── module/                  ← Feature modules (auth, user, etc.)
│   │   ├── prisma/                  ← PrismaModule + PrismaService
│   │   └── utils/
│   └── main.ts
│
├── prisma/
│   ├── schema.prisma                ← Database schema
│   └── migrations/                  ← Migration history
│
├── .dockerignore                    ← Docker image এ কী যাবে না
├── .env                             ← Local secrets (git এ দেবে না!)
├── .env.example                     ← .env এর template (git এ দেবে)
├── .gitignore
├── docker-compose.yml               ← সব service একসাথে চালানো
├── Dockerfile                       ← Image build করার নির্দেশনা
├── package.json
├── tsconfig.json
└── README.md
```

---

## 3. Prerequisites

তোমার **লোকাল PC** তে এগুলো থাকতে হবে:

| Tool | Download | Check করো |
|---|---|---|
| **Node.js 20+** | https://nodejs.org | `node -v` |
| **Docker Desktop** | https://www.docker.com/products/docker-desktop | `docker -v` |
| **Git** | https://git-scm.com | `git -v` |

> **Windows ব্যবহারকারী:** Docker Desktop install করলে Docker Compose আপনাআপনি আসে।

---

## 4. .env ফাইল এবং Config Setup

### 📁 Config ফাইলের ব্যাখ্যা

তুমি `.env` ফাইলের ভ্যালু `src/app/config/index.ts` থেকে পড়ো:

```typescript
// src/app/config/index.ts
import path from 'path';
import dotenv from 'dotenv';

// .env ফাইল লোড করা — process.cwd() মানে প্রজেক্টের root folder
dotenv.config({ path: path.join(process.cwd(), '.env') });

export default {
  port: process.env.PORT || 3000,
  env: process.env.NODE_ENV || 'development',
  bcryptSaltRounds: process.env.BCRYPT_SALT_ROUNDS,

  jwt: {
    accessTokenSecret: process.env.ACCESS_TOKEN_SECRET,
    accessTokenExpires: process.env.ACCESS_TOKEN_EXPIRES,
    refreshTokenSecret: process.env.REFRESH_TOKEN_SECRET,
    refreshTokenExpires: process.env.REFRESH_TOKEN_EXPIRES,
  },

  cloudinary: {
    name: process.env.CLOUDINARY_CLOUD_NAME,
    apiKey: process.env.CLOUDINARY_API_KEY,
    apiSecret: process.env.CLOUDINARY_API_SECRET,
  },

  email: {
    expires: process.env.EMAIL_EXPIRES,
    host: process.env.EMAIL_HOST,
    port: process.env.EMAIL_PORT,
    address: process.env.EMAIL_ADDRESS,
    pass: process.env.EMAIL_PASS,
    from: process.env.EMAIL_FROM,
    to: process.env.EMAIL_TO,
    admin: process.env.ADMIN_EMAIL,
  },

  stripe: {
    publicKey: process.env.STRIPE_PUBLISHABLE_KEY,
    secretKey: process.env.STRIPE_SECRET_KEY,
    webhookSecret: process.env.STRIPE_WEBHOOK_SECRET,
  },

  frontendUrl: process.env.FRONTEND_URL,
};
```

> **গুরুত্বপূর্ণ:** Docker container চালু হলে `.env` ফাইলের ভ্যালু পাবে কারণ `docker-compose.yml` এ `env_file: .env` দেওয়া আছে।

### 📄 .env.example ফাইল (এটা git-এ রাখো)

প্রজেক্ট রুটে `.env.example` নামে ফাইল তৈরি করো:

```env
# ========================================
# Application
# ========================================
NODE_ENV=development
PORT=5000

# ========================================
# PostgreSQL (Prisma)
# ========================================
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/mydb?schema=public
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=mydb

# ========================================
# MongoDB
# ========================================
MONGO_URI=mongodb://localhost:27017/mydatabase
MONGO_DB=mydatabase

# ========================================
# Redis
# ========================================
REDIS_URL=redis://localhost:6379

# ========================================
# Security
# ========================================
BCRYPT_SALT_ROUNDS=10
ACCESS_TOKEN_SECRET=your_very_long_random_secret_here
ACCESS_TOKEN_EXPIRES=7d
REFRESH_TOKEN_SECRET=another_very_long_random_secret_here
REFRESH_TOKEN_EXPIRES=90d

# ========================================
# Cloudinary (File Upload)
# ========================================
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

# ========================================
# Email (SMTP)
# ========================================
EMAIL_EXPIRES=900000
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_ADDRESS=your_email@gmail.com
EMAIL_PASS=your_gmail_app_password
EMAIL_FROM=your_email@gmail.com
EMAIL_TO=
ADMIN_EMAIL=admin@example.com

# ========================================
# Stripe (Payment)
# ========================================
STRIPE_PUBLISHABLE_KEY=pk_test_xxx
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# ========================================
# Google OAuth
# ========================================
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret

# ========================================
# URLs
# ========================================
FRONTEND_URL=http://localhost:3000
BACKEND_URL=http://localhost:5000/api/v1

# ========================================
# Rate Limiting
# ========================================
RATE_LIMIT_WINDOW=15m
RATE_LIMIT_MAX=100
RATE_LIMIT_DELAY=50
```

> **⚠️ সতর্কতা:** `.env` ফাইলে সত্যিকারের ভ্যালু দেবে। `.env.example` এ শুধু template রাখবে।
> `.env` কখনো git-এ push করবে না — `.gitignore` এ আছে কিনা চেক করো।

### Docker এ Database URL কীভাবে লিখবে?

```
# লোকাল development (Docker ছাড়া)
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/mydb

# Docker Compose এর ভেতরে (service name ব্যবহার করতে হবে!)
DATABASE_URL=postgresql://postgres:postgres@postgres:5432/mydb

# MongoDB লোকাল
MONGO_URI=mongodb://localhost:27017/mydatabase

# MongoDB Docker Compose এর ভেতরে
MONGO_URI=mongodb://mongo:27017/mydatabase
```

> **কেন `@postgres` এবং `@mongo`?** Docker Compose এ সব service একটা internal network এ থাকে। তখন `localhost` কাজ করে না — service-এর নাম ব্যবহার করতে হয়।

---

## 5. Dockerfile

প্রজেক্ট রুটে `Dockerfile` নামে ফাইল তৈরি করো:

```dockerfile
# ================================================================
# Stage 1: Builder
# কাজ: সব dependency ইনস্টল করা + Prisma generate + code build করা
# এই stage টা শুধু build এর জন্য — production image এ যাবে না
# ================================================================
FROM node:20-alpine AS builder

# Working directory সেট করা
WORKDIR /usr/src/app

# প্রথমে শুধু package files কপি করা
# কেন? এতে npm install step cache হয় — code না বদলালে পুনরায় install হয় না
COPY package*.json ./

# Prisma schema কপি করা — generate করতে লাগবে
COPY prisma ./prisma/

# সব dependency install (dev সহ — build এর জন্য দরকার)
RUN npm ci

# Prisma Client generate করা
# কেন? Prisma TypeScript types ও DB client তৈরি হয় — না করলে app crash করবে
RUN npx prisma generate

# বাকি সব source code কপি করা
COPY . .

# TypeScript → JavaScript build করা
RUN npm run build

# ================================================================
# Stage 2: Production
# কাজ: শুধু দরকারি জিনিস রেখে ছোট, fast, নিরাপদ image বানানো
# ================================================================
FROM node:20-alpine AS production

WORKDIR /usr/src/app

# Production environment বলে দেওয়া
ENV NODE_ENV=production

# শুধু package files কপি
COPY package*.json ./

# শুধু production dependency install (dev packages বাদ — image ছোট হয়)
RUN npm ci --only=production && npm cache clean --force

# Builder stage থেকে Prisma files কপি করা
COPY --from=builder /usr/src/app/prisma ./prisma
COPY --from=builder /usr/src/app/node_modules/.prisma ./node_modules/.prisma

# Builder stage থেকে compiled dist folder কপি
COPY --from=builder /usr/src/app/dist ./dist

# Security: root user এর বদলে non-root user তৈরি করা
# কেন? Container hack হলেও server root access পাবে না
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001
USER nestjs

# কোন port এ app চলবে তা বলে দেওয়া (documentation মাত্র, আসলে port খোলে না)
EXPOSE 5000

# Container চালু হলে কী চলবে:
# 1. Prisma migration deploy করবে (নতুন migration থাকলে apply হবে)
# 2. তারপর app চালু হবে
CMD ["sh", "-c", "npx prisma migrate deploy && node dist/src/main"]
```

### Multi-stage build কেন?

```
Builder stage:  ~800MB (সব dev tools সহ)
        ↓
Production stage: ~200MB (শুধু দরকারি জিনিস)

ফলাফল: Image 4x ছোট → faster pull → কম server storage
```

---

## 6. .dockerignore

প্রজেক্ট রুটে `.dockerignore` নামে ফাইল তৈরি করো:

```
# Dependencies — image এ নিজে install হবে
node_modules

# Build output — builder stage এ তৈরি হবে
dist

# Secrets — কখনো image এ নেবে না!
.env
.env.*
*.local

# Logs
*.log
npm-debug.log*
yarn-debug.log*

# Git
.git
.gitignore
.gitattributes

# Documentation
README.md
CHANGELOG.md
docs/

# Test files — production এ দরকার নেই
coverage/
test/
tests/
**/*.spec.ts
**/*.test.ts

# Code quality tools
.eslintrc*
.eslintignore
.prettierrc*
.editorconfig

# Docker files নিজেই — image এর ভেতরে দরকার নেই
docker-compose*.yml
Dockerfile*

# IDE/Editor
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db
```

---

## 7. docker-compose.yml

প্রজেক্ট রুটে `docker-compose.yml` নামে ফাইল তৈরি করো:

```yaml
version: '3.9'

services:

  # ================================================
  # NestJS Application
  # ================================================
  app:
    build:
      context: .                    # Dockerfile কোথায় আছে
      dockerfile: Dockerfile        # কোন Dockerfile ব্যবহার করব
      target: production            # Multi-stage এর কোন stage পর্যন্ত build করব
    container_name: nestjs_app
    restart: unless-stopped         # Crash হলে auto restart, কিন্তু manually stop করলে restart না
    ports:
      - "${PORT:-5000}:${PORT:-5000}"  # host:container — বাইরে থেকে access এর জন্য
    env_file:
      - .env                        # .env ফাইলের সব variable container এ পাঠাবে
    depends_on:
      postgres:
        condition: service_healthy  # PostgreSQL ready না হলে app চালু হবে না
      mongo:
        condition: service_started  # MongoDB চালু হলেই app শুরু হবে
      redis:
        condition: service_healthy  # Redis ready না হলে app চালু হবে না
    networks:
      - backend_network

  # ================================================
  # PostgreSQL Database
  # ================================================
  postgres:
    image: postgres:16-alpine       # Official PostgreSQL image (alpine = ছোট size)
    container_name: postgres_db
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5432:5432"                 # লোকাল থেকে pgAdmin দিয়ে connect করতে পারবে
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Data persist করার জন্য
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s                 # প্রতি 10 সেকেন্ডে check করবে
      timeout: 5s
      retries: 5                    # 5 বার fail হলে unhealthy
    networks:
      - backend_network

  # ================================================
  # MongoDB
  # ================================================
  mongo:
    image: mongo:7                  # Official MongoDB image
    container_name: mongodb
    restart: always
    ports:
      - "27017:27017"               # লোকাল থেকে MongoDB Compass দিয়ে connect করতে পারবে
    environment:
      MONGO_INITDB_DATABASE: ${MONGO_DB}
    volumes:
      - mongo_data:/data/db         # Data persist করার জন্য
    networks:
      - backend_network

  # ================================================
  # Redis Cache
  # ================================================
  redis:
    image: redis:7-alpine
    container_name: redis_cache
    restart: always
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes  # Data disk এ save করবে (restart এ হারাবে না)
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - redis_data:/data
    networks:
      - backend_network

# ================================================
# Internal Network
# সব service এই network এ — একে অপরকে name দিয়ে call করতে পারে
# যেমন: app থেকে postgres connect করতে host = "postgres"
# ================================================
networks:
  backend_network:
    driver: bridge

# ================================================
# Persistent Volumes
# Container মুছে গেলেও data থাকে
# ================================================
volumes:
  postgres_data:
  mongo_data:
  redis_data:
```

---

## 8. Docker Commands

### 🔵 Image সম্পর্কিত

```bash
# Docker image দেখা
docker images
docker image ls

# Image build করা (current folder এর Dockerfile দিয়ে)
docker build -t myapp:1.0 .

# Image build করা (specific stage পর্যন্ত)
docker build --target production -t myapp:prod .

# Image মুছে ফেলা
docker rmi myapp:1.0

# ব্যবহার না হওয়া সব image মুছে ফেলা
docker image prune -a

# Image Docker Hub এ push করা
docker push username/myapp:latest

# Docker Hub থেকে image নামানো
docker pull username/myapp:latest
```

### 🟢 Container সম্পর্কিত

```bash
# চলমান container দেখা
docker ps

# সব container দেখা (বন্ধ সহ)
docker ps -a

# Container চালানো (background এ)
docker run -d --name myweb -p 5000:5000 myapp:1.0

# Container চালানো (interactive terminal সহ)
docker run -it --name myweb myapp:1.0 sh

# .env ফাইল দিয়ে container চালানো
docker run -d --name myweb --env-file .env -p 5000:5000 myapp:1.0

# Container বন্ধ করা
docker stop myweb

# Container চালু করা (আগে বন্ধ করা)
docker start myweb

# Container restart
docker restart myweb

# Container মুছে ফেলা
docker rm myweb

# জোর করে মুছে ফেলা (চলমান থাকলেও)
docker rm -f myweb

# Log দেখা
docker logs myweb

# Live log দেখা (Ctrl+C তে বের হবে)
docker logs -f myweb

# Container এর ভেতরে shell খোলা (debugging এর জন্য)
docker exec -it myweb sh
docker exec -it myweb bash

# Container এর ভেতরে specific command চালানো
docker exec -it myweb npx prisma migrate deploy
docker exec -it myweb npx prisma studio
```

### 🟡 Docker Compose সম্পর্কিত

```bash
# সব service build + চালু (প্রথমবার)
docker compose up -d --build

# শুধু চালু (আগে build করা থাকলে)
docker compose up -d

# একটা specific service চালু
docker compose up -d postgres

# সব service বন্ধ করা
docker compose down

# Volume সহ মুছে ফেলা (database data হারাবে!)
docker compose down -v

# Log দেখা
docker compose logs -f
docker compose logs -f app

# চলমান service দেখা
docker compose ps

# Service restart
docker compose restart app

# Image rebuild করে চালু করা
docker compose up -d --build app
```

### 🔴 Cleanup Commands

```bash
# Disk usage দেখা
docker system df

# সব বন্ধ container, unused image, network মুছে ফেলা
docker system prune

# সব কিছু মুছে ফেলা (volume সহ) — সাবধান!
docker system prune -a --volumes

# সব container force stop + remove
docker rm -f $(docker ps -aq)

# সব unused image মুছে ফেলা
docker image prune -a -f
```

### 🔑 Docker run এর গুরুত্বপূর্ণ Flags

| Flag | মানে | উদাহরণ |
|---|---|---|
| `-d` | Background (detached) এ চালাও | `docker run -d nginx` |
| `-it` | Interactive terminal খোলো | `docker run -it ubuntu sh` |
| `--name` | Container এর নাম দাও | `docker run --name myapp` |
| `-p` | Port mapping (host:container) | `docker run -p 5000:5000` |
| `-v` | Volume/folder mount | `docker run -v /host:/container` |
| `-e` | Environment variable দাও | `docker run -e PORT=5000` |
| `--env-file` | .env ফাইল থেকে variable নাও | `docker run --env-file .env` |
| `--rm` | Container বন্ধ হলে auto delete | `docker run --rm ubuntu` |
| `--network` | Specific network এ যোগ দাও | `docker run --network mynet` |
| `-m` | Memory limit | `docker run -m 512m` |
| `--restart` | Restart policy | `docker run --restart unless-stopped` |

---

## 9. Local এ Docker দিয়ে Test করা

### Step 1 — .env তৈরি করো (Docker Compose এর জন্য)

```bash
cp .env.example .env
```

`.env` ফাইলে Docker Compose এর জন্য এই ভ্যালু দাও:

```env
NODE_ENV=development
PORT=5000

# Docker Compose এ service name ব্যবহার করবে (localhost না!)
DATABASE_URL=postgresql://postgres:postgres@postgres:5432/mydb?schema=public
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=mydb

MONGO_URI=mongodb://mongo:27017/mydatabase
MONGO_DB=mydatabase

REDIS_URL=redis://redis:6379
```

### Step 2 — Build করে সব চালাও

```bash
docker compose up -d --build
```

**এই command এ কী হয়:**
```
1. Dockerfile পড়ে image build করে
2. PostgreSQL, MongoDB, Redis image download করে
3. সব container চালু করে
4. Internal network তৈরি করে
5. Volumes তৈরি করে (data সংরক্ষণের জন্য)
```

### Step 3 — Log দেখে check করো

```bash
# সব service এর log একসাথে
docker compose logs -f

# শুধু app এর log
docker compose logs -f app
```

### Step 4 — Test করো

```bash
# App চলছে কিনা দেখো
curl http://localhost:5000

# অথবা browser এ যাও
http://localhost:5000/api/v1
```

### Step 5 — Database check করো

```bash
# PostgreSQL এ connect করো (পাসওয়ার্ড: postgres)
docker exec -it postgres_db psql -U postgres -d mydb

# MongoDB তে connect করো
docker exec -it mongodb mongosh mydatabase

# Redis check করো
docker exec -it redis_cache redis-cli ping
# Output: PONG
```

### Step 6 — Prisma migration চালাও

```bash
# Container এর ভেতরে migration চালানো
docker exec -it nestjs_app npx prisma migrate deploy

# অথবা Prisma Studio (visual DB editor)
docker exec -it nestjs_app npx prisma studio
# এরপর browser এ http://localhost:5555 যাও
```

### Step 7 — বন্ধ করো

```bash
# সব বন্ধ করো (data থাকবে)
docker compose down

# সব বন্ধ + data মুছে ফেলো (fresh start এর জন্য)
docker compose down -v
```

---

## 10. Docker Hub Setup

### Step 1 — অ্যাকাউন্ট তৈরি করো

1. **https://hub.docker.com** এ যাও
2. **Sign Up** করো (ফ্রি)
3. Email verify করো

### Step 2 — Repository তৈরি করো

1. **Create Repository** button এ click করো
2. **Name:** `nestjs-app`
3. **Visibility:** Public
4. **Create** button এ click করো

তোমার image এর full name হবে: `your-username/nestjs-app:latest`

### Step 3 — Access Token তৈরি করো (পাসওয়ার্ড না!)

1. **Account Settings** এ যাও (top right এ তোমার username → Settings)
2. **Security** tab এ click করো
3. **New Access Token** button এ click করো
4. **Token Name:** `github-actions`
5. **Access permissions:** Read, Write, Delete
6. **Generate** করো
7. **Token টা এখনই কপি করো** — পরে দেখা যাবে না!

---

## 11. VPS Server Setup

> এই কাজ শুধু একবারই করতে হবে।

### কোথায় VPS পাবে?

| Provider | Price | বিশেষত্ব |
|---|---|---|
| **Hetzner** | ~$4/month থেকে | সস্তা, fast |
| **DigitalOcean** | $6/month থেকে | সহজ UI |
| **Vultr** | $5/month থেকে | বাংলাদেশ থেকে card দিয়ে কিনতে পারবে |
| **AWS Lightsail** | $5/month থেকে | AWS এর সহজ VPS |

**Recommended:** Ubuntu 22.04 LTS, minimum 1 CPU + 1GB RAM

---

### Step 1 — VPS এ SSH Login করো

```bash
# Windows (PowerShell বা Git Bash)
ssh root@YOUR_SERVER_IP

# Linux/macOS
ssh root@YOUR_SERVER_IP
```

---

### Step 2 — System Update করো

```bash
apt update && apt upgrade -y
```

---

### Step 3 — Docker Install করো

```bash
# Docker install script (official)
curl -fsSL https://get.docker.com | sh

# Docker service চালু করো
systemctl enable docker
systemctl start docker

# Verify
docker --version
```

---

### Step 4 — Docker Compose Install করো

```bash
apt-get install -y docker-compose-plugin

# Verify
docker compose version
```

---

### Step 5 — Firewall Setup করো

```bash
# UFW install করো (যদি না থাকে)
apt install -y ufw

# Default rules
ufw default deny incoming
ufw default allow outgoing

# SSH allow করো (এটা না করলে lock out হয়ে যাবে!)
ufw allow ssh

# তোমার app port allow করো
ufw allow 5000/tcp

# Firewall চালু করো
ufw enable

# Status check
ufw status
```

---

### Step 6 — Non-root User তৈরি করো (Recommended)

```bash
# নতুন user তৈরি করো
adduser appuser

# sudo permission দাও
usermod -aG sudo appuser

# Docker permission দাও (sudo ছাড়া docker চালাতে পারবে)
usermod -aG docker appuser

# নতুন user এ switch করো
su - appuser
```

---

### Step 7 — App Directory তৈরি করো

```bash
mkdir -p /app
```

---

### Step 8 — SSH Key তৈরি করো (GitHub Actions এর জন্য)

এই কাজ তোমার **লোকাল PC** তে করো (server এ না):

```bash
# Linux/macOS
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions

# Windows PowerShell
ssh-keygen -t ed25519 -C "github-actions" -f "$env:USERPROFILE\.ssh\github_actions"
```

`Enter` চাপো (passphrase দরকার নেই)।

এতে দুটো file তৈরি হবে:
- `github_actions` — **private key** → GitHub Secrets এ যাবে
- `github_actions.pub` — **public key** → Server এ যাবে

---

### Step 9 — Public Key Server এ যোগ করো

```bash
# Linux/macOS — automatic
ssh-copy-id -i ~/.ssh/github_actions.pub root@YOUR_SERVER_IP

# Windows — manually
# প্রথমে public key দেখো:
type "$env:USERPROFILE\.ssh\github_actions.pub"

# Output copy করো, তারপর server এ:
echo "PASTE_YOUR_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

---

### Step 10 — Connection Test করো

```bash
# Local PC থেকে key দিয়ে test
ssh -i ~/.ssh/github_actions root@YOUR_SERVER_IP

# সফল হলে server এ login হবে
```

---

### Step 11 — Private Key content কপি করো

GitHub Secrets এ `SERVER_SSH_KEY` হিসেবে দেবে:

```bash
# Linux/macOS
cat ~/.ssh/github_actions

# Windows
Get-Content "$env:USERPROFILE\.ssh\github_actions"
```

**পুরো output কপি করো** — `-----BEGIN OPENSSH PRIVATE KEY-----` থেকে `-----END OPENSSH PRIVATE KEY-----` পর্যন্ত।

---

## 12. GitHub Repository Setup

### Step 1 — Repository তৈরি করো

1. **https://github.com** এ যাও
2. **New repository** তৈরি করো
3. **Private** রাখো (recommended)
4. README যোগ করো না (নিজে করবে)
5. **Create repository**

### Step 2 — Code Push করো

```bash
cd your-project-folder
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git push -u origin main
```

### Step 3 — Workflow folder তৈরি করো

```bash
mkdir -p .github/workflows
```

---

## 13. GitHub Secrets Setup

GitHub Repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

**প্রত্যেকটা আলাদাভাবে যোগ করো:**

### 🐳 Docker Hub

| Secret Name | Value |
|---|---|
| `DOCKER_USERNAME` | তোমার Docker Hub username |
| `DOCKER_PASSWORD` | Docker Hub Access Token (Step 10.3 এ তৈরি করেছিলে) |

### 🖥️ VPS Server

| Secret Name | Value |
|---|---|
| `SERVER_HOST` | তোমার VPS IP address (যেমন: `123.45.67.89`) |
| `SERVER_USER` | `root` বা `appuser` |
| `SERVER_SSH_KEY` | Private key এর পুরো content (Step 11.11) |

### 🔧 Application

| Secret Name | Value |
|---|---|
| `PORT` | `5000` |
| `NODE_ENV` | `production` |
| `DATABASE_URL` | `postgresql://postgres:YOUR_PASS@postgres:5432/mydb?schema=public` |
| `POSTGRES_USER` | `postgres` |
| `POSTGRES_PASSWORD` | একটা strong password |
| `POSTGRES_DB` | `mydb` |
| `MONGO_URI` | `mongodb://mongo:27017/mydatabase` |
| `MONGO_DB` | `mydatabase` |
| `REDIS_URL` | `redis://redis:6379` |
| `BCRYPT_SALT_ROUNDS` | `10` |
| `ACCESS_TOKEN_SECRET` | একটা long random string |
| `ACCESS_TOKEN_EXPIRES` | `7d` |
| `REFRESH_TOKEN_SECRET` | আরেকটা long random string |
| `REFRESH_TOKEN_EXPIRES` | `90d` |
| `CLOUDINARY_CLOUD_NAME` | Cloudinary dashboard থেকে |
| `CLOUDINARY_API_KEY` | Cloudinary dashboard থেকে |
| `CLOUDINARY_API_SECRET` | Cloudinary dashboard থেকে |
| `EMAIL_EXPIRES` | `900000` |
| `EMAIL_HOST` | `smtp.gmail.com` |
| `EMAIL_PORT` | `587` |
| `EMAIL_ADDRESS` | তোমার Gmail |
| `EMAIL_PASS` | Gmail App Password (login password না!) |
| `EMAIL_FROM` | তোমার Gmail |
| `EMAIL_TO` | (খালি রাখো) |
| `ADMIN_EMAIL` | admin email |
| `STRIPE_PUBLISHABLE_KEY` | Stripe dashboard থেকে |
| `STRIPE_SECRET_KEY` | Stripe dashboard থেকে |
| `STRIPE_WEBHOOK_SECRET` | Stripe dashboard থেকে |
| `GOOGLE_CLIENT_ID` | Google Cloud Console থেকে |
| `GOOGLE_CLIENT_SECRET` | Google Cloud Console থেকে |
| `FRONTEND_URL` | `https://your-frontend.com` |
| `BACKEND_URL` | `http://YOUR_SERVER_IP:5000/api/v1` |
| `RATE_LIMIT_WINDOW` | `15m` |
| `RATE_LIMIT_MAX` | `100` |
| `RATE_LIMIT_DELAY` | `50` |

> **Gmail App Password কীভাবে পাবে?**
> Gmail → Manage Account → Security → 2-Step Verification ON করো → App Passwords → নতুন password তৈরি করো

---

## 14. GitHub Actions CI/CD Pipeline

ফাইল: `.github/workflows/nestjs-cicd.yml`

```yaml
name: NestJS CI/CD Pipeline

# কখন চলবে — main branch এ push হলে
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest        # GitHub এর free Linux server এ চলবে

    steps:
      # =============================
      # Step 1: Code checkout
      # =============================
      - name: Checkout code
        uses: actions/checkout@v4

      # =============================
      # Step 2: Node.js setup
      # =============================
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'            # npm cache করবে — পরের run এ faster হবে

      # =============================
      # Step 3: Dependencies install
      # =============================
      - name: Install dependencies
        run: npm ci               # ci = clean install, package-lock.json অনুসারে exact install

      # =============================
      # Step 4: Build
      # =============================
      - name: Build NestJS App
        run: npm run build

      # =============================
      # Step 5: Docker Hub login
      # =============================
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      # =============================
      # Step 6: Docker image build + push
      # =============================
      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          target: production       # Multi-stage এর production stage পর্যন্ত
          push: true               # Docker Hub এ push করবে
          tags: ${{ secrets.DOCKER_USERNAME }}/nestjs-app:latest
          cache-from: type=registry,ref=${{ secrets.DOCKER_USERNAME }}/nestjs-app:buildcache
          cache-to: type=inline    # Build cache save করবে — পরের build faster হবে

      # =============================
      # Step 7: VPS তে deploy
      # =============================
      - name: Deploy to VPS Server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            echo "========================================="
            echo "🚀 Starting deployment..."
            echo "========================================="

            # App directory নিশ্চিত করো
            mkdir -p /app
            cd /app

            # =============================
            # .env ফাইল তৈরি করো
            # GitHub Secrets থেকে সব ভ্যালু নেওয়া হচ্ছে
            # =============================
            cat > /app/.env << 'ENVEOF'
            NODE_ENV=production
            PORT=${{ secrets.PORT }}
            DATABASE_URL=${{ secrets.DATABASE_URL }}
            POSTGRES_USER=${{ secrets.POSTGRES_USER }}
            POSTGRES_PASSWORD=${{ secrets.POSTGRES_PASSWORD }}
            POSTGRES_DB=${{ secrets.POSTGRES_DB }}
            MONGO_URI=${{ secrets.MONGO_URI }}
            MONGO_DB=${{ secrets.MONGO_DB }}
            REDIS_URL=${{ secrets.REDIS_URL }}
            BCRYPT_SALT_ROUNDS=${{ secrets.BCRYPT_SALT_ROUNDS }}
            ACCESS_TOKEN_SECRET=${{ secrets.ACCESS_TOKEN_SECRET }}
            ACCESS_TOKEN_EXPIRES=${{ secrets.ACCESS_TOKEN_EXPIRES }}
            REFRESH_TOKEN_SECRET=${{ secrets.REFRESH_TOKEN_SECRET }}
            REFRESH_TOKEN_EXPIRES=${{ secrets.REFRESH_TOKEN_EXPIRES }}
            CLOUDINARY_CLOUD_NAME=${{ secrets.CLOUDINARY_CLOUD_NAME }}
            CLOUDINARY_API_KEY=${{ secrets.CLOUDINARY_API_KEY }}
            CLOUDINARY_API_SECRET=${{ secrets.CLOUDINARY_API_SECRET }}
            EMAIL_EXPIRES=${{ secrets.EMAIL_EXPIRES }}
            EMAIL_HOST=${{ secrets.EMAIL_HOST }}
            EMAIL_PORT=${{ secrets.EMAIL_PORT }}
            EMAIL_ADDRESS=${{ secrets.EMAIL_ADDRESS }}
            EMAIL_PASS=${{ secrets.EMAIL_PASS }}
            EMAIL_FROM=${{ secrets.EMAIL_FROM }}
            EMAIL_TO=${{ secrets.EMAIL_TO }}
            ADMIN_EMAIL=${{ secrets.ADMIN_EMAIL }}
            STRIPE_PUBLISHABLE_KEY=${{ secrets.STRIPE_PUBLISHABLE_KEY }}
            STRIPE_SECRET_KEY=${{ secrets.STRIPE_SECRET_KEY }}
            STRIPE_WEBHOOK_SECRET=${{ secrets.STRIPE_WEBHOOK_SECRET }}
            GOOGLE_CLIENT_ID=${{ secrets.GOOGLE_CLIENT_ID }}
            GOOGLE_CLIENT_SECRET=${{ secrets.GOOGLE_CLIENT_SECRET }}
            FRONTEND_URL=${{ secrets.FRONTEND_URL }}
            BACKEND_URL=${{ secrets.BACKEND_URL }}
            RATE_LIMIT_WINDOW=${{ secrets.RATE_LIMIT_WINDOW }}
            RATE_LIMIT_MAX=${{ secrets.RATE_LIMIT_MAX }}
            RATE_LIMIT_DELAY=${{ secrets.RATE_LIMIT_DELAY }}
            ENVEOF

            # =============================
            # docker-compose.yml তৈরি করো
            # =============================
            cat > /app/docker-compose.yml << 'COMPOSEEOF'
            version: '3.9'

            services:
              app:
                image: ${{ secrets.DOCKER_USERNAME }}/nestjs-app:latest
                container_name: nestjs_app
                restart: unless-stopped
                ports:
                  - "${{ secrets.PORT }}:${{ secrets.PORT }}"
                env_file:
                  - .env
                depends_on:
                  postgres:
                    condition: service_healthy
                  mongo:
                    condition: service_started
                  redis:
                    condition: service_healthy
                networks:
                  - backend_network

              postgres:
                image: postgres:16-alpine
                container_name: postgres_db
                restart: always
                environment:
                  POSTGRES_USER: ${{ secrets.POSTGRES_USER }}
                  POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD }}
                  POSTGRES_DB: ${{ secrets.POSTGRES_DB }}
                ports:
                  - "5432:5432"
                volumes:
                  - postgres_data:/var/lib/postgresql/data
                healthcheck:
                  test: ["CMD-SHELL", "pg_isready -U ${{ secrets.POSTGRES_USER }}"]
                  interval: 10s
                  timeout: 5s
                  retries: 5
                networks:
                  - backend_network

              mongo:
                image: mongo:7
                container_name: mongodb
                restart: always
                ports:
                  - "27017:27017"
                environment:
                  MONGO_INITDB_DATABASE: ${{ secrets.MONGO_DB }}
                volumes:
                  - mongo_data:/data/db
                networks:
                  - backend_network

              redis:
                image: redis:7-alpine
                container_name: redis_cache
                restart: always
                ports:
                  - "6379:6379"
                command: redis-server --appendonly yes
                healthcheck:
                  test: ["CMD", "redis-cli", "ping"]
                  interval: 10s
                  timeout: 5s
                  retries: 5
                volumes:
                  - redis_data:/data
                networks:
                  - backend_network

            networks:
              backend_network:
                driver: bridge

            volumes:
              postgres_data:
              mongo_data:
              redis_data:
            COMPOSEEOF

            echo "📥 Pulling latest Docker image..."
            docker pull ${{ secrets.DOCKER_USERNAME }}/nestjs-app:latest

            echo "⏹️  Stopping old containers..."
            docker compose down

            echo "▶️  Starting new containers..."
            docker compose up -d

            echo "🧹 Cleaning up old images..."
            docker image prune -f

            echo "========================================="
            echo "✅ Deployment completed successfully!"
            echo "========================================="
```

---

## 15. প্রথম Deploy করা

### Step 1 — সব ফাইল commit করো

```bash
git add .
git commit -m "feat: add docker and cicd setup"
git push origin main
```

### Step 2 — GitHub Actions এ দেখো

1. GitHub Repo → **Actions** tab
2. চলমান workflow দেখবে
3. প্রতিটা step এ click করে log দেখতে পারবে
4. সব ✅ হলে deploy সফল

### Step 3 — Server এ verify করো

```bash
ssh root@YOUR_SERVER_IP

# Container চলছে কিনা দেখো
docker compose -f /app/docker-compose.yml ps

# App log দেখো
docker compose -f /app/docker-compose.yml logs app
```

### Step 4 — API test করো

```bash
curl http://YOUR_SERVER_IP:5000

# অথবা browser এ
http://YOUR_SERVER_IP:5000/api/v1
```

---

## 16. CI/CD কীভাবে কাজ করে?

```
তুমি: git push origin main
         │
         ▼
GitHub Actions শুরু হয় (ubuntu server এ)
         │
         ├─► npm ci + npm run build (code check)
         │
         ├─► Docker image build (Dockerfile অনুসারে)
         │
         ├─► Docker Hub এ push (username/nestjs-app:latest)
         │
         └─► SSH দিয়ে VPS তে যাও
                  │
                  ├─► /app/.env লেখো (secrets থেকে)
                  ├─► /app/docker-compose.yml লেখো
                  ├─► docker pull (নতুন image নামাও)
                  ├─► docker compose down (পুরনো বন্ধ)
                  ├─► docker compose up -d (নতুন চালু)
                  │      └─► prisma migrate deploy (migration)
                  │      └─► node dist/src/main (app start)
                  └─► docker image prune (পুরনো image মুছো)

মোট সময়: প্রায় 3-6 মিনিট
```

---

## 17. Server এ Useful Commands

```bash
# App directory তে যাও
cd /app

# সব container এর status দেখো
docker compose ps

# App এর live log দেখো
docker compose logs -f app

# Postgres log দেখো
docker compose logs -f postgres

# App container এর ভেতরে যাও
docker exec -it nestjs_app sh

# Prisma migration manually চালাও
docker exec -it nestjs_app npx prisma migrate deploy

# Prisma Studio চালাও (browser এ DB দেখতে)
docker exec -it nestjs_app npx prisma studio
# Browser: http://YOUR_SERVER_IP:5555

# Database সরাসরি দেখো
docker exec -it postgres_db psql -U postgres -d mydb

# Redis check করো
docker exec -it redis_cache redis-cli ping

# সব restart করো
docker compose restart

# শুধু app restart
docker compose restart app

# সব বন্ধ করো
docker compose down

# পুরনো image মুছো (disk space বাঁচাতে)
docker image prune -f

# Disk usage দেখো
docker system df
df -h
```

---

## 18. Troubleshooting

### ❌ Database connection error

**Error:** `Can't connect to database`

```bash
# Solution 1: service name check করো
# DATABASE_URL এ localhost না দিয়ে service name দাও
DATABASE_URL=postgresql://postgres:pass@postgres:5432/mydb  # ✅
DATABASE_URL=postgresql://postgres:pass@localhost:5432/mydb  # ❌ Docker এ কাজ করবে না

# Solution 2: SSL error হলে
DATABASE_URL=postgresql://user:pass@host:5432/db?sslmode=no-verify
```

### ❌ Container বারবার restart হচ্ছে

```bash
# Log দেখো
docker compose logs app

# Common causes:
# 1. .env ফাইল নেই বা ভুল ভ্যালু
# 2. Database এখনো ready না (depends_on check করো)
# 3. Port already in use
```

### ❌ Prisma migration fail

```bash
# Container এর ভেতরে manually চালাও
docker exec -it nestjs_app npx prisma migrate deploy

# Prisma generate করো
docker exec -it nestjs_app npx prisma generate
```

### ❌ GitHub Actions fail

**SSH connection failed:**
- `SERVER_HOST`, `SERVER_USER`, `SERVER_SSH_KEY` secrets check করো
- Public key server এ আছে কিনা: `cat ~/.ssh/authorized_keys`

**Docker build fail:**
- লোকালে test করো: `docker build --target production -t test .`

**Docker push fail:**
- `DOCKER_USERNAME`, `DOCKER_PASSWORD` secrets check করো

### ❌ Port already in use

```bash
# কোন process port ব্যবহার করছে দেখো
lsof -i :5000
# অথবা
ss -tlnp | grep 5000

# Process kill করো
kill -9 PID_NUMBER
```

### ❌ .env server এ ঠিক নেই কিনা দেখো

```bash
ssh root@YOUR_SERVER_IP
cat /app/.env
```

---

## 19. Security Best Practices

```
✅ .env কখনো git এ commit করবে না
✅ .gitignore এ .env আছে কিনা verify করো
✅ Docker Hub এ password না দিয়ে Access Token ব্যবহার করো
✅ App non-root user (USER nestjs) দিয়ে চালাও
✅ Production এ Redis তে password সেট করো
✅ Database port (5432, 27017) public এ expose করবে না
✅ GitHub Actions এর জন্য আলাদা SSH key ব্যবহার করো
✅ Secrets rotate করো নিয়মিত (leaked হলে তাড়াতাড়ি বদলাও)
✅ Firewall (ufw) দিয়ে শুধু দরকারি port খোলো
```

---

## 20. Production Go-Live Checklist

```
PC Setup:
  ☐ Node.js 20+ installed
  ☐ Docker Desktop installed
  ☐ Git installed

Docker Hub:
  ☐ Account created
  ☐ Repository "nestjs-app" created
  ☐ Access Token created and saved

VPS:
  ☐ Ubuntu VPS provisioned
  ☐ Docker installed
  ☐ Docker Compose installed
  ☐ Firewall configured (port 5000 open)
  ☐ /app folder created
  ☐ SSH key generated (local PC)
  ☐ Public key added to server

GitHub:
  ☐ Repository created
  ☐ Code pushed to main branch
  ☐ .github/workflows/nestjs-cicd.yml created
  ☐ All secrets added (check সব গুলো আছে)

Deploy:
  ☐ First git push triggered
  ☐ GitHub Actions all green ✅
  ☐ docker compose ps shows all running
  ☐ API responds on http://YOUR_IP:5000
```

---

## Quick Reference Card

```bash
# Local development
npm run start:dev

# Local Docker test
docker compose up -d --build
docker compose logs -f app
docker compose down

# Deploy (automatic)
git add . && git commit -m "your message" && git push origin main

# Server debug
ssh root@YOUR_IP
docker compose -f /app/docker-compose.yml logs -f app
docker exec -it nestjs_app sh
```

---

> **Author:** Saurav
> **Stack:** NestJS + Prisma + PostgreSQL + MongoDB + Redis + Docker + GitHub Actions
> **Updated:** April 2026
