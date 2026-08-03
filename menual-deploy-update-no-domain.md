# 🚀 Backend Deployment Guide (VPS + Ubuntu)

> Works whether or not you have a domain. If you don't have one yet, use your **server's public IP** — everything below still applies, just skip the SSL/domain steps (marked ⚠️ **Domain only**).

---

## 📌 Overview

| Item | Value |
|---|---|
| App | Backend API (Node.js/NestJS/Express) |
| Port | `5000` (change if yours differs) |
| Access without domain | `http://YOUR_SERVER_IP:5000` or `http://YOUR_SERVER_IP` (via Nginx) |
| Access with domain | `https://api.yourdomain.com` |

---

## ✅ Before You Start — Domain or No Domain?

You only need **three things**, domain or not:

1. Server has a **public IP**.
2. The port your app runs on is **open in the firewall / security group**.
3. Your app listens on `0.0.0.0`, not `localhost`:
   ```ts
   app.listen(5000, '0.0.0.0');
   ```

If you have a domain, add an **A record** pointing it to your server IP before running Certbot. If you don't, just skip that — you'll use the IP directly.

---

## 🔧 Step 1 — Initial Server Setup (One Time)

```bash
ssh root@YOUR_SERVER_IP
sudo apt update && sudo apt upgrade -y
sudo apt install curl git -y
```

### Install Node.js (via NVM)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install --lts
node -v
npm -v
```

### Install PM2, Nginx

```bash
npm install -g pm2
sudo apt install nginx -y
```

⚠️ **Domain only** — install Certbot for SSL (skip if using only an IP):

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo apt install cron -y
sudo systemctl enable cron --now
crontab -e
# add this line at the bottom:
0 3 * * * certbot renew --quiet
```

---

## 🗄️ Step 2 — Deploy the Backend

### Clone & configure

```bash
cd ~
git clone https://github.com/your-org/your-backend.git
cd your-backend
nano .env
```

Paste your environment variables (Mongo URI, JWT secrets, Cloudinary, Stripe, email, etc.), then save with `Ctrl+X` → `Y` → `Enter`.

> If you don't have a domain yet, set `FRONTEND_URL` / `BACKEND_URL` to your IP for now (e.g. `http://YOUR_SERVER_IP:5000`), and update it later once you get a domain.

### Install, build, move to /var/www

```bash
npm install
npm run build
cd ~
sudo mkdir -p /var/www
sudo cp -r your-backend /var/www/
```

### Run with PM2

```bash
cd /var/www/your-backend
pm2 start npm --name "backend" -- run start:prod
pm2 save
pm2 startup
# copy-paste the command it prints
```

At this point your API is already reachable at:

```
http://YOUR_SERVER_IP:5000
```

Make sure port `5000` is open (e.g. `sudo ufw allow 5000` if using UFW, or open it in your cloud provider's firewall/security group).

---

## 🌐 Step 3 — Put Nginx in Front (Recommended, Domain or Not)

This lets you drop the `:5000` and just use the IP (or domain) directly on port 80.

```bash
sudo vim /etc/nginx/sites-available/backend
```

Press `i`, paste:

**No domain — use `server_name _;` to catch all requests to this IP:**

```nginx
server {
    listen 80;
    server_name _;
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

**With domain — use your real domain instead:**

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

Save: `Esc` → `:wq` → `Enter`, then enable it:

```bash
sudo ln -s /etc/nginx/sites-available/backend /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

Now the API is reachable at:

```
http://YOUR_SERVER_IP
```

⚠️ **Domain only** — get SSL:

```bash
sudo certbot --nginx -d api.yourdomain.com
```

> Let's Encrypt/Certbot **requires a domain** — it cannot issue a certificate for a bare IP. Without a domain, your API will only be available over `http://`, not `https://`. That's fine for development/testing; for production, a domain (even a cheap one, or a free one like `.eu.org` / a subdomain from `duckdns.org` / `nip.io`) is strongly recommended so you can enable HTTPS.

✅ **Backend live at:**
- No domain: `http://YOUR_SERVER_IP` (or `:5000` without Nginx)
- With domain: `https://api.yourdomain.com`

---

## 🔄 Redeploy After Code Changes

```bash
cd /var/www/your-backend
git pull
npm install
npm run build
pm2 restart backend
```

---

## 📊 Useful Commands

```bash
pm2 list                       # see running apps
pm2 logs backend --lines 50    # view logs
pm2 restart backend            # restart
pm2 delete backend             # remove (e.g. if port is stuck)

sudo nginx -t                  # test nginx config
sudo systemctl restart nginx
sudo systemctl status nginx

sudo certbot certificates      # check SSL certs (domain only)
sudo certbot renew             # renew manually (domain only)
```

---

## ⚠️ Common Issues & Fixes

| Problem | Fix |
|---|---|
| `502 Bad Gateway` | App isn't running on the expected port — check `pm2 list` and the `proxy_pass` port in your Nginx config |
| Can't reach `http://SERVER_IP:5000` | Port not open in firewall/security group, or app is bound to `localhost` instead of `0.0.0.0` |
| Certbot fails | You need a real domain with an A record pointing to this server — it will not work with a bare IP |
| `sudo cd` doesn't work | Use `cd` without `sudo`; use `sudo mkdir` separately |
| App crashes after reboot | Run `pm2 startup` and paste the command it outputs |
| Port already in use | `pm2 delete backend` then start it again |

---

## 🗂️ Getting a Domain Later

Once you're ready:

1. Buy a domain (Namecheap, GoDaddy, Cloudflare) — or get a free one (`duckdns.org`, `.eu.org`, `is-a.dev`, etc.).
2. Add an **A record** pointing it to `YOUR_SERVER_IP`.
3. Wait 5–30 minutes for DNS to propagate.
4. Change `server_name _;` to `server_name api.yourdomain.com;` in your Nginx config, then `sudo nginx -t && sudo systemctl restart nginx`.
5. Run `sudo certbot --nginx -d api.yourdomain.com` to get HTTPS.
