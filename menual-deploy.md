# Deployment Guide
Here is a brief instruction of hosting a MERN application in a VPS. Basically I am going to share the steps of hosting any MERN application. 

## Step-1
Connect with the VPS using ssh. In this step you need to connect the remote server with the local terminal or other cli tool.

```bash
  ssh username@ip_address_of_the_vps_server
```
Note: First time you need to type `yes` for confirmation the fingerprint. 

## Step-2
Update and upgrade of the packages of the application
```bash
sudo apt update && sudo apt upgrade -y
```

## Step-3
Install curl in the server 
```bash
sudo apt install curl
```

## Step-4
Install the nodejs in the server 
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash

\. "$HOME/.nvm/nvm.sh"

nvm install 22

node -v
nvm current
npm -v
```

## Step-5 
Install git 
```bash
sudo apt install git 
```

## Step-6 
Install pm2
```bash
npm install pm2 -g
```

## Step-7
Clone the repository from the github 
```bash
git clone https://github-repository-link 
```

## Step-8
Navigate to the directory and isntall the packages
```bash
cd directoryName
npm install 
npm run build
```

## Step-9
Create a directory in the following directory /var & navigate to the directory 
```bash
sudo cd /var
sudo mkdir www
sudo cd www
```

## Step-10
Copy the code directory in the in the www directory 
```bash
sudo cp -r ~/directoyName ./
```

## Step-11
Navigate to the directory 
```bash
sudo cd directoryName
```

## Step-12
Start the applciation using the process manager
```bash
pm2 start npm --name "app-name" -- start

```

## Step-13
Save the pm2 process 
```bash
pm2 save
pm2 startup
```
## Step-14
Install nginx
```bash
sudo apt install nginx -y
```

## Step-15
Create the nginx config file
```bash
sudo vim /etc/nginx/sites-available/appName 
```
Edit the file and insert the following configuration to the config file
```bash
server {
    listen 80;
    server_name your-domain.com;
   
    client_max_body_size 100M;

    location / {
        proxy_pass http://localhost:applicationPort;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

## Step-16
Enable the nginx configuration
```bash
sudo ln -s /etc/nginx/sites-available/appName /etc/nginx/sites-enabled/
```

## Step-17
Restart the nginx server 
```bash
sudo systemctl restart nginx
```

## Step-18
Install certbot 
```bash
sudo apt install certbot python3-certbot-nginx -y
```

## Step-19
Configure https with certbot ssl
```bash
certbot --nginx -d your-domain.com -d www.your-domain.com
```

## Step-20
Install cron 
```bash
sudo apt install cron
```

## Step-21
Enable crontab and and run the cron service
```bash
sudo systemctl enable cron
sudo systemctl start cron
```

## Step-22
Add contab in the system
```bash
crontab -e
```
Add the following line to the crontab config file
```bash
0 3 * * * certbot renew --quiet
```

## Additional Setup
1. Restart the app: `pm2 restart app-name`
2. Run two next app using pm2: `pm2 start npm --name "app-name" -- start -- -p 3001`
3. setup redirection the site to another domain
```bash
# Redirect non-www → www
server {
    listen 80;
    server_name your-domain.com;
    return 301 http://www.your-domain.com$request_uri;
}

# Main app on www
server {
    listen 80;
    server_name www.your-domain.com;
    
    client_max_body_size 100M;

    location / {
        proxy_pass http://localhost:applicationPort;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
