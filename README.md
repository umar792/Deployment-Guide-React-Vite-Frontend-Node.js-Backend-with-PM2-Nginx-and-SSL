# Deployment-Guide-React-Vite-Frontend-Node.js-Backend-with-PM2-Nginx-and-SSL

This README provides a step-by-step reference for deploying a full-stack application (React/Vite frontend + Node.js backend) on a Linux server (Ubuntu). It includes explanations for every step, common issues, and their resolutions. You can save this in your GitHub repository for future deployments.

Table of Contents

Server Setup

Directory Structure

SSH Key Setup

Backend Setup

Frontend Setup

PM2 Process Management

Nginx Setup

SSL with Let's Encrypt

Stripe Integration / API Testing

Common Issues & Solutions

1️⃣ Server Setup

SSH login to server

ssh root@<SERVER_IP>


Update packages and install dependencies

sudo apt update && sudo apt upgrade -y
sudo apt install nginx git curl build-essential -y


Explanation:

ssh root@... → login to server

apt update/upgrade → ensures system is up-to-date

nginx → web server to serve frontend and proxy backend

git → to clone your project

curl → to test API endpoints

build-essential → needed for Node.js modules compilation

2️⃣ Directory Structure

We create a clean directory structure for frontend and backend:

sudo mkdir -p /var/brooklyngroup/frontend
sudo mkdir -p /var/brooklyngroup/backend


Explanation:

/var → standard folder for web services

Separate frontend and backend avoids conflicts

3️⃣ SSH Key Setup for GitHub

Generate SSH key:

ssh-keygen -t ed25519 -C "your_email@example.com"


Press enter to save at default location: /root/.ssh/id_ed25519

Copy public key:

cat ~/.ssh/id_ed25519.pub


Add this key to your GitHub account → allows server to clone repos without password.

Common Issues:

Passphrase does not match → carefully type the same passphrase twice or leave blank.

Key already exists → you can overwrite or create a new file name.

4️⃣ Backend Setup

Clone backend repo:

cd /var/brooklyngroup/backend
git clone git@github.com:yourusername/backend.git .


Install dependencies:

npm install


Test backend locally:

npm start
# OR
node index.js


Test API:

curl http://127.0.0.1:5000/api


Expected response:

{"success":true,"message":"server is running"}


Common Issues:

502 Bad Gateway → backend not running → start with PM2

Port in use → change backend port in .env

5️⃣ Frontend Setup

Clone frontend repo:

cd /var/brooklyngroup/frontend
git clone git@github.com:yourusername/frontend.git .


Install dependencies:

npm install


Build production:

npm run build
# generates /dist folder


Test dev server with PM2 (optional):

pm2 start npm --name "frontend" -- run dev


Serve production build:

npm install -g serve
pm2 start serve --name "frontend" -- -s /var/brooklyngroup/frontend/dist -l 8080


Explanation:

npm run build → prepares optimized static files

serve → serves production build via PM2 (or use Nginx for production)

6️⃣ PM2 Process Management

Start backend with PM2:

cd /var/brooklyngroup/backend
pm2 start npm --name "backend" -- start


Check status:

pm2 list


Logs:

pm2 logs backend


Auto-start on reboot:

pm2 save
pm2 startup


Common Issues:

pid N/A → backend not actually started, check start script

After logout, PM2 keeps processes running

Always use pm2 save to persist list

7️⃣ Nginx Setup

Default Nginx config:

sudo nano /etc/nginx/sites-available/default


Example config:

server {
    listen 80;
    server_name pay.brooklyngroup.com;

    root /var/brooklyngroup/frontend/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
}


Test Nginx:

sudo nginx -t
sudo systemctl reload nginx


Common Issues:

502 Bad Gateway → backend not running

301 Moved Permanently → trailing slash /api → use /api/ or both blocks

8️⃣ SSL with Let’s Encrypt

Install Certbot:

sudo apt install certbot python3-certbot-nginx -y


Issue certificate:

sudo certbot --nginx -d pay.brooklyngroup.com


Test auto-renew:

sudo certbot renew --dry-run


Explanation:

SSL avoids Mixed Content errors with Stripe payments

Certbot automatically updates Nginx config for HTTPS

9️⃣ Stripe Integration / API Testing

Frontend calls backend:

axios.post("https://pay.brooklyngroup.com/api/v1/create-payment-intent")


Backend example:

app.post("/api/v1/create-payment-intent", async (req, res) => {
  const paymentIntent = await stripe.paymentIntents.create({
    amount: 1000,
    currency: "usd",
  });
  res.json({ clientSecret: paymentIntent.client_secret });
});


Test backend:

curl -X POST https://pay.brooklyngroup.com/api/v1/create-payment-intent

🔧 10️⃣ Common Issues & How to Resolve
Issue	Cause	Solution
502 Bad Gateway	Backend not running	pm2 start backend
301 Moved Permanently	Trailing slash /api	Use /api/ or add both /api and /api/ in Nginx
Mixed Content	Frontend HTTPS calls backend HTTP	Use Nginx proxy + SSL for backend (/api/)
Backend returns 200 but Stripe not working	Placeholder response	Implement /create-payment-intent route
PM2 shows pid N/A	Incorrect start command	Use pm2 start npm --name backend -- start
Processes die on logout	PM2 not set up properly	pm2 save && pm2 startup
✅ Tips for Future Deployments

Always start backend first → then frontend

Always test backend locally (curl http://127.0.0.1:PORT/api)

Nginx proxies must match backend port

SSL must be applied to avoid Mixed Content errors

Save PM2 state (pm2 save) to survive reboots
