# 🚀 Full Stack Deployment Guide (VPS + Ubuntu)

> **Example Project:** CarCheckAI  
> Replace domain names and GitHub links with your own project details.

---

## 📌 Project Overview

| App | GitHub Repo | Domain | Port |
|-----|-------------|--------|------|
| Backend API | `https://github.com/your-org/your-backend.git` | `api.yourdomain.com` | `5000` |
| Admin Dashboard | `https://github.com/your-org/your-admin.git` | `admin.yourdomain.com` | `3001` |
| Website | `https://github.com/your-org/your-website.git` | `yourdomain.com` / `www.yourdomain.com` | `3000` |

> ⚠️ **Before starting:** Go to your domain registrar (Namecheap, GoDaddy, Cloudflare etc.) and add **A records** pointing all domains to your VPS IP address.

---

## 🔧 Initial Server Setup (One Time Only)

### Step 1 — SSH into the server
```bash
ssh root@YOUR_VPS_IP
```
> First time: type `yes` to confirm the fingerprint.

---

### Step 2 — Update & upgrade packages
```bash
sudo apt update && sudo apt upgrade -y
```

---

### Step 3 — Install curl
```bash
sudo apt install curl -y
```

---

### Step 4 — Install latest Node.js via NVM

> NVM lets you install and switch Node versions easily. Always install the latest LTS.

```bash
# Install NVM (check latest version at https://github.com/nvm-sh/nvm/releases)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash

# Load NVM into current session
\. "$HOME/.nvm/nvm.sh"

# Install latest LTS Node (currently v22)
nvm install --lts

# Verify installation
node -v
npm -v
```

> 💡 To always use the latest in the future:
> ```bash
> nvm install --lts
> nvm use --lts
> ```

---

### Step 5 — Update npm to latest
```bash
npm install -g npm@latest
```

---

### Step 6 — Install Git
```bash
sudo apt install git -y
```

---

### Step 7 — Install PM2 (Process Manager)
```bash
npm install -g pm2
```

---

### Step 8 — Install Nginx
```bash
sudo apt install nginx -y
```

---

### Step 9 — Install Certbot (SSL)
```bash
sudo apt install certbot python3-certbot-nginx -y
```

---

### Step 10 — Setup Cron for auto SSL renewal
```bash
sudo apt install cron -y
sudo systemctl enable cron
sudo systemctl start cron
crontab -e
```
When asked, choose `1` (nano). Add this line at the bottom of the file:
```
0 3 * * * certbot renew --quiet
```
Save: `Ctrl+X` → `Y` → `Enter`

---

## 🗄️ Deploy Backend API

### Step 1 — Clone the repository
```bash
cd ~
git clone https://github.com/your-org/your-backend.git
cd your-backend
```

### Step 2 — Create .env file
```bash
nano .env
```
Paste your environment variables:
```env
NODE_ENV=production
PORT=5000
MONGO_URI=mongodb+srv://username:password@cluster.mongodb.net/dbname

JWT_SECRET=your_jwt_secret
JWT_EXPIRE=1h
BCRYPT_SALT_ROUNDS=10

ACCESS_TOKEN_SECRET=your_access_token_secret
ACCESS_TOKEN_EXPIRES=7d
REFRESH_TOKEN_SECRET=your_refresh_token_secret
REFRESH_TOKEN_EXPIRES=90d

CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_ADDRESS=your@gmail.com
EMAIL_PASS=your_app_password
EMAIL_FROM=your@gmail.com
ADMIN_EMAIL=admin@gmail.com

STRIPE_PUBLISHABLE_KEY=pk_live_xxxx
STRIPE_SECRET_KEY=sk_live_xxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxx

GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret

FRONTEND_URL=https://yourdomain.com
BACKEND_URL=https://api.yourdomain.com/api/v1

RATE_LIMIT_WINDOW=15m
RATE_LIMIT_MAX=100
RATE_LIMIT_DELAY=50
```
Save: `Ctrl+X` → `Y` → `Enter`

### Step 3 — Install & build
```bash
npm install
npm run build
```

### Step 4 — Copy to /var/www
```bash
cd ~
sudo mkdir -p /var/www
sudo cp -r your-backend /var/www/
```

### Step 5 — Start with PM2
```bash
cd /var/www/your-backend
pm2 start npm --name "backend" -- run start:prod
pm2 save
pm2 startup
```
> Copy the command that `pm2 startup` outputs and run it.

### Step 6 — Create Nginx config
```bash
sudo vim /etc/nginx/sites-available/backend
```
Press `i` to enter insert mode, paste:
```nginx
server {
    listen 80;
    server_name api.yourdomain.com;

    client_max_body_size 100M;
    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
Save: `Esc` → `:wq` → `Enter`

### Step 7 — Enable config & restart Nginx
```bash
sudo ln -s /etc/nginx/sites-available/backend /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Step 8 — SSL Certificate
```bash
sudo certbot --nginx -d api.yourdomain.com
```
- Enter your email
- Type `Y` to agree to Terms of Service
- Type `Y` or `N` for EFF newsletter

✅ **Backend live at:** `https://api.yourdomain.com`  
✅ **Swagger Docs:** `https://api.yourdomain.com/api/docs`

---

## 🖥️ Deploy Admin Dashboard (Next.js — Port 3001)

### Step 1 — Clone the repository
```bash
cd ~
git clone https://github.com/your-org/your-admin.git
cd your-admin
```

### Step 2 — Create .env file
```bash
nano .env
```
```env
NEXT_PUBLIC_API_URL=https://api.yourdomain.com/api/v1
NEXTAUTH_SECRET=your_nextauth_secret
NEXTAUTH_URL=https://admin.yourdomain.com
```
Save: `Ctrl+X` → `Y` → `Enter`

### Step 3 — Install & build
```bash
npm install
npm run build
```

### Step 4 — Copy to /var/www
```bash
cd ~
sudo cp -r your-admin /var/www/
```

### Step 5 — Start with PM2 on port 3001
```bash
cd /var/www/your-admin
pm2 start npm --name "admin" -- start -- -p 3001
pm2 save
```

### Step 6 — Create Nginx config
```bash
sudo vim /etc/nginx/sites-available/admin
```
Press `i`, paste:
```nginx
server {
    listen 80;
    server_name admin.yourdomain.com;

    client_max_body_size 100M;
    location / {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
Save: `Esc` → `:wq` → `Enter`

### Step 7 — Enable config & restart Nginx
```bash
sudo ln -s /etc/nginx/sites-available/admin /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Step 8 — SSL Certificate
```bash
sudo certbot --nginx -d admin.yourdomain.com
```

✅ **Admin Dashboard live at:** `https://admin.yourdomain.com`

---

## 🌐 Deploy Website (Next.js — Port 3000)

### Step 1 — Clone the repository
```bash
cd ~
git clone https://github.com/your-org/your-website.git
cd your-website
```

### Step 2 — Create .env file
```bash
nano .env
```
```env
NEXT_PUBLIC_API_URL=https://api.yourdomain.com/api/v1
NEXTAUTH_SECRET=your_nextauth_secret
NEXTAUTH_URL=https://yourdomain.com
```
Save: `Ctrl+X` → `Y` → `Enter`

### Step 3 — Install & build
```bash
npm install
npm run build
```

### Step 4 — Copy to /var/www
```bash
cd ~
sudo cp -r your-website /var/www/
```

### Step 5 — Start with PM2 on port 3000
```bash
cd /var/www/your-website
pm2 start npm --name "website" -- start
pm2 save
```

### Step 6 — Create Nginx config
```bash
sudo vim /etc/nginx/sites-available/website
```
Press `i`, paste:
```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    client_max_body_size 100M;
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
Save: `Esc` → `:wq` → `Enter`

### Step 7 — Enable config & restart Nginx
```bash
sudo ln -s /etc/nginx/sites-available/website /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Step 8 — SSL Certificate
```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

✅ **Website live at:** `https://yourdomain.com`

---

## 🔄 Update / Redeploy (After Code Changes)

### Backend:
```bash
cd /var/www/your-backend
git pull
npm install
npm run build
pm2 restart backend
```

### Admin Dashboard:
```bash
cd /var/www/your-admin
git pull
npm install
npm run build
pm2 restart admin
```

### Website:
```bash
cd /var/www/your-website
git pull
npm install
npm run build
pm2 restart website
```

---

## 📊 Useful Commands

```bash
# See all running apps
pm2 list

# View live logs
pm2 logs backend --lines 50
pm2 logs admin --lines 50
pm2 logs website --lines 50

# Restart apps
pm2 restart backend
pm2 restart admin
pm2 restart website

# Check Nginx
sudo nginx -t
sudo systemctl status nginx
sudo systemctl restart nginx

# Check SSL certificates
sudo certbot certificates

# Renew SSL manually
sudo certbot renew
```

---

## ⚠️ Common Issues & Fixes

| Problem | Fix |
|---------|-----|
| `502 Bad Gateway` | App not running on correct port. Check `pm2 list` and nginx `proxy_pass` port |
| `sudo cd` not working | Use `cd` without sudo. Use `sudo mkdir` separately |
| Certbot fails | DNS A record not pointing to VPS IP yet. Wait 5-30 min |
| App crashes on restart | Run `pm2 startup` and copy-paste the output command |
| Port already in use | Run `pm2 delete app-name` then restart |

---

## 🗂️ Port & Domain Summary

| App | PM2 Name | Port | Domain |
|-----|----------|------|--------|
| Backend API | `backend` | `5000` | `api.yourdomain.com` |
| Admin Dashboard | `admin` | `3001` | `admin.yourdomain.com` |
| Website | `website` | `3000` | `yourdomain.com` |
