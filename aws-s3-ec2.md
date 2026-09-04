# Bookkeepers Marketplace — AWS Production Deployment Guide

This README documents the production deployment setup for the **Bookkeepers Marketplace** platform using AWS, GoDaddy, Docker, Nginx, S3, CloudFront, MongoDB, and Stripe.

> **Important:** Never commit passwords, `.pem` files, AWS secret keys, Stripe secret keys, database credentials, or `.env` files to GitHub.

---

## 1. Production Architecture

```text
                         Internet
                            │
                            ▼
                         GoDaddy
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
bookkeepersmarketplace.store   admin.bookkeepersmarketplace.store   api.bookkeepersmarketplace.store
          │                 │                 │
          └─────────────────┴─────────────────┘
                            │
                        Elastic IP
                            │
                            ▼
                           EC2
                            │
                           Nginx
                 ┌──────────┼──────────┐
                 │          │          │
                 ▼          ▼          ▼
              Frontend     Admin     Backend
              Next.js      Next.js    NestJS
               :3000        :3001      :5000
                                         │
                           ┌─────────────┼─────────────┐
                           │             │             │
                           ▼             ▼             ▼
                       MongoDB         Stripe         S3
                                                     │
                                                     ▼
                                                 CloudFront
```

---

## 2. Domains

| Application | Domain |
|---|---|
| Website | `https://bookkeepersmarketplace.store` |
| Website WWW | `https://www.bookkeepersmarketplace.store` |
| Admin Dashboard | `https://admin.bookkeepersmarketplace.store` |
| Backend API | `https://api.bookkeepersmarketplace.store` |

---

## 3. AWS Account Security

Recommended AWS account setup:

1. Create the AWS account using the business/company email.
2. Complete AWS billing setup.
3. Enable MFA for the AWS root account.
4. Do **not** create root access keys.
5. Create a separate IAM developer user for infrastructure management.
6. Use temporary `AdministratorAccess` only during initial setup.
7. Reduce permissions after the infrastructure is ready.

### IAM Console Login

The developer should use:

```text
AWS IAM sign-in URL
IAM username
Temporary password
```

The root account password, MFA codes, and billing credentials must remain private.

---

## 4. IAM User for S3 / Backend Access

For application-level S3 access, create a dedicated IAM user or preferably use an EC2 IAM Role.

### If Access Keys Are Required

Go to:

```text
IAM
→ Users
→ Select user
→ Security credentials
→ Access keys
→ Create access key
```

Example environment variables:

```env
AWS_ACCESS_KEY_ID=AKIAxxxxxxxxxxxxxxxx
AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AWS_BUCKET_NAME=your-s3-bucket-name
AWS_REGION=us-east-2
```

### Recommended for EC2

For production on EC2, prefer an **IAM Role** attached to the EC2 instance instead of long-lived access keys.

---

## 5. S3 Setup

Create a private S3 bucket for:

- Images
- Videos
- Documents
- Generated PDFs
- Profile files

Example:

```text
bookkeepers-marketplace-prod-storage
```

Recommended settings:

```text
Bucket type: General purpose
Object Ownership: Bucket owner enforced
ACLs: Disabled
Block Public Access: ON
Default encryption: Enabled
```

Recommended object structure:

```text
uploads/
profiles/
documents/
videos/
pdfs/
```

---

## 6. CloudFront Setup

Keep the S3 bucket private and serve files through CloudFront.

Create a CloudFront distribution with:

```text
Origin type: Amazon S3
Origin: Your private S3 bucket
Private S3 access: Enabled
Origin path: Empty
Origin settings: Recommended
Cache settings: Recommended
```

Use Origin Access Control (OAC).

Example:

```env
AWS_PUBLIC_BASE_URL=https://YOUR_CLOUDFRONT_DOMAIN.cloudfront.net
```

Example object:

```text
uploads/profile.jpg
```

Public URL:

```text
https://YOUR_CLOUDFRONT_DOMAIN.cloudfront.net/uploads/profile.jpg
```

### Best Database Pattern

Store only the object key:

```text
uploads/profile.jpg
```

Generate the full URL at response time:

```text
AWS_PUBLIC_BASE_URL + objectKey
```

---

## 7. EC2 Server Setup

Recommended initial configuration:

```text
Region: us-east-2 (Ohio)
OS: Ubuntu Server 26.04 LTS
Architecture: 64-bit (x86)
Instance type: t3.small
vCPU: 2
RAM: 2 GB
Storage: 30 GB gp3
```

For more traffic or heavier workloads, upgrade later to:

```text
t3.medium
```

### Security Group

Inbound rules:

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | My IP |
| HTTP | 80 | Anywhere |
| HTTPS | 443 | Anywhere |

Do **not** publicly expose:

```text
3000
3001
5000
27017
6379
```

---

## 8. Elastic IP

Allocate an Elastic IP:

```text
EC2
→ Elastic IP addresses
→ Allocate Elastic IP address
```

Then associate it with the production EC2 instance.

Use the Elastic IP in GoDaddy DNS.

---

## 9. GoDaddy DNS Setup

Example DNS records:

```text
A      @       → YOUR_ELASTIC_IP
A      admin   → YOUR_ELASTIC_IP
A      api     → YOUR_ELASTIC_IP
CNAME  www     → bookkeepersmarketplace.store
```

Do not modify GoDaddy system records such as:

```text
NS
SOA
_domainconnect
_dmarc
```

---

## 10. SSH Login

Ubuntu EC2 instances do not normally use a root password.

Use the `.pem` key:

```bash
ssh -i "./bookkeepers-marketplace-key.pem" ubuntu@YOUR_ELASTIC_IP
```

### Windows `.pem` Permission Fix

If you get:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
```

Run in CMD:

```cmd
icacls ".\bookkeepers-marketplace-key.pem" /inheritance:r
icacls ".\bookkeepers-marketplace-key.pem" /remove "NT AUTHORITY\Authenticated Users"
icacls ".\bookkeepers-marketplace-key.pem" /remove "BUILTIN\Users"
icacls ".\bookkeepers-marketplace-key.pem" /grant:r "%USERDOMAIN%\%USERNAME%:(R)"
```

Check:

```cmd
icacls ".\bookkeepers-marketplace-key.pem"
```

Then reconnect:

```cmd
ssh -i ".\bookkeepers-marketplace-key.pem" ubuntu@YOUR_ELASTIC_IP
```

### Root Shell

After SSH login:

```bash
sudo -i
```

Usually, normal `ubuntu` + `sudo` is preferred.

---

## 11. Initial Ubuntu Setup

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git curl unzip nginx
```

Enable Nginx:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Check:

```bash
sudo systemctl status nginx
```

---

## 12. Install Docker

```bash
sudo apt install -y docker.io docker-compose-v2
```

Enable Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Allow the Ubuntu user to run Docker:

```bash
sudo usermod -aG docker ubuntu
```

Logout and reconnect:

```bash
exit
```

Then verify:

```bash
docker --version
docker compose version
docker ps
```

---

## 13. Add Swap

For a `t3.small` instance, adding 2 GB swap is recommended.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Make it permanent:

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Check:

```bash
free -h
```

---

## 14. Production Project Structure

Recommended server directory:

```text
/home/ubuntu/bookkeepers-marketplace/
│
├── frontend/
├── admin/
├── backend/
└── docker-compose.yml
```

Create:

```bash
mkdir -p ~/bookkeepers-marketplace
cd ~/bookkeepers-marketplace
```

---

## 15. Clone Git Repositories

Example:

```bash
git clone FRONTEND_REPOSITORY_URL frontend
git clone ADMIN_REPOSITORY_URL admin
git clone BACKEND_REPOSITORY_URL backend
```

For private repositories, use GitHub Deploy Keys or another secure authentication method.

---

## 16. Environment Variables

### Frontend

`frontend/.env.production`

```env
NEXT_PUBLIC_API_URL=https://api.bookkeepersmarketplace.store
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_xxxxxxxxx
```

### Admin

`admin/.env.production`

```env
NEXT_PUBLIC_API_URL=https://api.bookkeepersmarketplace.store
NEXTAUTH_URL=https://admin.bookkeepersmarketplace.store
NEXTAUTH_SECRET=CHANGE_ME
```

### Backend

`backend/.env`

```env
NODE_ENV=production
PORT=5000

MONGODB_URI=CHANGE_ME
JWT_SECRET=CHANGE_ME

FRONTEND_URL=https://bookkeepersmarketplace.store
ADMIN_URL=https://admin.bookkeepersmarketplace.store

AWS_BUCKET_NAME=bookkeepers-marketplace-prod-storage
AWS_REGION=us-east-2
AWS_PUBLIC_BASE_URL=https://YOUR_CLOUDFRONT_DOMAIN.cloudfront.net

STRIPE_SECRET_KEY=sk_test_xxxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxx
```

If access keys are being used:

```env
AWS_ACCESS_KEY_ID=CHANGE_ME
AWS_SECRET_ACCESS_KEY=CHANGE_ME
```

> Never commit `.env` files to Git.

---

## 17. `.gitignore`

Add:

```gitignore
.env
.env.*
*.pem
node_modules/
dist/
.next/
```

---

## 18. Docker Compose

Example root `docker-compose.yml`:

```yaml
services:
  frontend:
    build: ./frontend
    container_name: bkm-frontend
    restart: unless-stopped
    env_file:
      - ./frontend/.env.production
    ports:
      - "127.0.0.1:3000:3000"

  admin:
    build: ./admin
    container_name: bkm-admin
    restart: unless-stopped
    env_file:
      - ./admin/.env.production
    ports:
      - "127.0.0.1:3001:3000"

  backend:
    build: ./backend
    container_name: bkm-backend
    restart: unless-stopped
    env_file:
      - ./backend/.env
    ports:
      - "127.0.0.1:5000:5000"
```

Start:

```bash
docker compose up -d --build
```

Check:

```bash
docker ps
```

Logs:

```bash
docker compose logs -f
```

---

## 19. Nginx — Website

Create:

```bash
sudo nano /etc/nginx/sites-available/bookkeepersmarketplace.store
```

```nginx
server {
    listen 80;
    server_name bookkeepersmarketplace.store www.bookkeepersmarketplace.store;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 20. Nginx — Admin

```bash
sudo nano /etc/nginx/sites-available/admin.bookkeepersmarketplace.store
```

```nginx
server {
    listen 80;
    server_name admin.bookkeepersmarketplace.store;

    location / {
        proxy_pass http://127.0.0.1:3001;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 21. Nginx — Backend API

```bash
sudo nano /etc/nginx/sites-available/api.bookkeepersmarketplace.store
```

```nginx
server {
    listen 80;
    server_name api.bookkeepersmarketplace.store;

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:5000;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 22. Enable Nginx Sites

```bash
sudo ln -s /etc/nginx/sites-available/bookkeepersmarketplace.store /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/admin.bookkeepersmarketplace.store /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/api.bookkeepersmarketplace.store /etc/nginx/sites-enabled/
```

Remove default config:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

Test:

```bash
sudo nginx -t
```

Reload:

```bash
sudo systemctl reload nginx
```

---

## 23. SSL / HTTPS

Install Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Website:

```bash
sudo certbot --nginx \
-d bookkeepersmarketplace.store \
-d www.bookkeepersmarketplace.store
```

Admin:

```bash
sudo certbot --nginx \
-d admin.bookkeepersmarketplace.store
```

Backend:

```bash
sudo certbot --nginx \
-d api.bookkeepersmarketplace.store
```

Test renewal:

```bash
sudo certbot renew --dry-run
```

Final URLs:

```text
https://bookkeepersmarketplace.store
https://www.bookkeepersmarketplace.store
https://admin.bookkeepersmarketplace.store
https://api.bookkeepersmarketplace.store
```

---

## 24. MongoDB Atlas

Recommended:

```text
MongoDB Atlas
→ Network Access
→ Add IP Address
→ EC2 Elastic IP /32
```

Avoid `0.0.0.0/0` in production when possible.

---

## 25. Stripe Setup

Client responsibilities:

```text
Business verification
Legal information
Business representative
Identity verification
Business address
Website information
Bank account
Payout information
Tax information
2FA
```

Developer integration:

```env
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_xxxxxxxxx
STRIPE_SECRET_KEY=sk_test_xxxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxx
```

Production:

```env
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_xxxxxxxxx
STRIPE_SECRET_KEY=sk_live_xxxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxx
```

Webhook example:

```text
https://api.bookkeepersmarketplace.store/api/v1/stripe/webhook
```

Create separate webhook configuration for live mode.

---

## 26. CORS

Backend should allow only the required frontend origins.

Example NestJS:

```ts
app.enableCors({
  origin: [
    'https://bookkeepersmarketplace.store',
    'https://www.bookkeepersmarketplace.store',
    'https://admin.bookkeepersmarketplace.store',
  ],
  credentials: true,
});
```

---

## 27. Deployment Update Workflow

When new code is pushed:

```bash
cd ~/bookkeepers-marketplace
```

Update repositories:

```bash
cd frontend && git pull && cd ..
cd admin && git pull && cd ..
cd backend && git pull && cd ..
```

Rebuild:

```bash
docker compose up -d --build
```

Check:

```bash
docker ps
docker compose logs --tail=100
```

---

## 28. Useful Server Commands

Memory:

```bash
free -h
```

Disk:

```bash
df -h
```

Processes:

```bash
top
```

Docker usage:

```bash
docker stats
```

Containers:

```bash
docker ps
```

Logs:

```bash
docker compose logs -f
```

Nginx:

```bash
sudo systemctl status nginx
sudo nginx -t
sudo systemctl reload nginx
```

---

## 29. Common Problems

### SSH timeout

Check:

```text
EC2 instance is running
Elastic IP is associated
Security Group allows port 22 from current IP
```

### `.pem` bad permissions

Fix with `icacls` as shown above.

### S3 `AccessDenied`

Keep S3 private and verify CloudFront OAC + S3 bucket policy.

### CloudFront NXDOMAIN

Wait until CloudFront distribution is fully deployed and copy the exact distribution domain.

### Domain not resolving

Verify GoDaddy DNS and test:

```bash
nslookup bookkeepersmarketplace.store
nslookup admin.bookkeepersmarketplace.store
nslookup api.bookkeepersmarketplace.store
```

### Nginx 502 Bad Gateway

Check containers:

```bash
docker ps
```

Test locally:

```bash
curl http://127.0.0.1:3000
curl http://127.0.0.1:3001
curl http://127.0.0.1:5000
```

Then inspect logs.

---

## 30. Production Security Checklist

- [ ] AWS root MFA enabled
- [ ] Root access keys disabled
- [ ] IAM developer account used
- [ ] IAM permissions reduced after setup
- [ ] SSH restricted to trusted IP
- [ ] EC2 application ports are not public
- [ ] Elastic IP attached
- [ ] S3 Block Public Access enabled
- [ ] CloudFront OAC configured
- [ ] `.env` ignored by Git
- [ ] `.pem` ignored by Git
- [ ] MongoDB restricted to trusted IPs
- [ ] HTTPS enabled
- [ ] Stripe secrets backend-only
- [ ] Production Stripe webhook configured
- [ ] Docker restart policy enabled
- [ ] Backups and monitoring configured

---

## 31. Full Setup Order

```text
AWS Account
   ↓
IAM
   ↓
S3
   ↓
CloudFront
   ↓
EC2
   ↓
Security Group
   ↓
Elastic IP
   ↓
GoDaddy DNS
   ↓
SSH
   ↓
Ubuntu Setup
   ↓
Docker + Nginx
   ↓
Frontend + Admin + Backend
   ↓
MongoDB
   ↓
SSL / HTTPS
   ↓
Stripe Webhook
   ↓
Final Production Testing
```

---

## Notes

This guide is intended to be reusable for future AWS deployments using a similar architecture:

```text
Next.js + Next.js Admin + NestJS
AWS EC2 + S3 + CloudFront
Docker + Nginx
GoDaddy DNS
MongoDB
Stripe
```

Replace all project-specific domains, bucket names, regions, repository URLs, and environment variables when using it for another project.
