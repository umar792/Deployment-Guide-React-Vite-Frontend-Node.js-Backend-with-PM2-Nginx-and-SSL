# React + Node.js Full Deployment Guide

This document provides a **step-by-step guide to deploy a React + Vite frontend and Node.js backend** with PM2, Nginx, and SSL. It includes **server setup, directory structure, SSH setup, PM2 process management, Nginx configuration, SSL setup, Stripe integration, common issues, and solutions**.

---

## Table of Contents

1. Server Setup
2. Directory Structure
3. SSH Key Setup
4. Backend Setup
5. Frontend Setup
6. PM2 Process Management
7. Nginx Setup
8. SSL with Let's Encrypt
9. Stripe Integration / API Testing
10. Common Issues & Solutions
11. Notes on Backend Subdomain

---

## 1️⃣ Server Setup

```bash
ssh root@<SERVER_IP>
```

* Update server:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install nginx git curl build-essential -y
```

**Explanation:**

* `nginx` → web server
* `git` → clone repositories
* `curl` → test APIs
* `build-essential` → needed for Node.js modules

---

## 2️⃣ Directory Structure

```bash
sudo mkdir -p /var/brooklyngroup/frontend
sudo mkdir -p /var/brooklyngroup/backend
```

* `/var/brooklyngroup/frontend` → store frontend build
* `/var/brooklyngroup/backend` → store backend code

---

## 3️⃣ SSH Key Setup for GitHub

Generate SSH key:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Add public key to GitHub:

```bash
cat ~/.ssh/id_ed25519.pub
```

**Common Issues:**

* `Passphrase mismatch` → re-enter carefully
* `Key exists` → overwrite or create new file

---

## 4️⃣ Backend Setup

Clone backend repo:

```bash
cd /var/brooklyngroup/backend
git clone git@github.com:yourusername/backend.git .
npm install
```

Test backend locally:

```bash
npm start
curl http://127.0.0.1:5000/api
```

Expected output:

```json
{"success":true,"message":"server is running"}
```

---

## 5️⃣ Frontend Setup

Clone frontend repo:

```bash
cd /var/brooklyngroup/frontend
git clone git@github.com:yourusername/frontend.git .
npm install
```

Build production:

```bash
npm run build
```

Serve with PM2:

```bash
npm install -g serve
pm2 start serve --name "frontend" -- -s /var/brooklyngroup/frontend/dist -l 8080
```

---

## 6️⃣ PM2 Process Management

Start backend:

```bash
cd /var/brooklyngroup/backend
pm2 start npm --name "backend" -- start
```

Check running processes:

```bash
pm2 list
```

View logs:

```bash
pm2 logs backend
```

Auto-start on reboot:

```bash
pm2 save
pm2 startup
```

---

## 7️⃣ Nginx Setup

Edit default site config:

```bash
sudo nano /etc/nginx/sites-available/default
```

Example configuration:

```nginx
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

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
}
```

Test and reload Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 8️⃣ SSL with Let's Encrypt

Install Certbot:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Issue certificate:

```bash
sudo certbot --nginx -d pay.brooklyngroup.com
```

Test auto-renew:

```bash
sudo certbot renew --dry-run
```

**Explanation:** SSL avoids Mixed Content issues with Stripe.

---

## 9️⃣ Stripe Integration / API Testing

Frontend call:

```ts
axios.post("https://pay.brooklyngroup.com/api/v1/create-payment-intent")
```

Backend route example:

```js
app.post("/api/v1/create-payment-intent", async (req, res) => {
  const paymentIntent = await stripe.paymentIntents.create({
    amount: 1000,
    currency: "usd",
  });
  res.json({ clientSecret: paymentIntent.client_secret });
});
```

Test backend:

```bash
curl -X POST https://pay.brooklyngroup.com/api/v1/create-payment-intent
```

---

## 10️⃣ Common Issues & Solutions

| Issue                                | Cause                             | Solution                                    |
| ------------------------------------ | --------------------------------- | ------------------------------------------- |
| 502 Bad Gateway                      | Backend not running               | Start backend with PM2                      |
| 301 Moved Permanently                | Trailing slash `/api`             | Use `/api/` in Nginx proxy                  |
| Mixed Content                        | Frontend HTTPS calls backend HTTP | Use Nginx proxy + SSL                       |
| Backend returns 200 but Stripe fails | Placeholder route                 | Implement `/create-payment-intent` properly |
| PM2 shows pid N/A                    | Backend not started               | Correct PM2 start command                   |
| Processes die on logout              | PM2 not saved                     | `pm2 save && pm2 startup`                   |

---

## 11️⃣ Notes on Backend Subdomain

* Ideally backend should have a separate subdomain (`api.brooklyngroup.com`).
* If client does not provide it, backend will run on `/api/` path via Nginx reverse proxy.
* All frontend API calls must use `/api/` path.
* PM2 backend process continues running on the assigned port (e.g., 5000).


If your client wants to add a subdomain api.brooklyngroup.com so your backend can run on it, here’s a step-by-step explanation you can give them. This does not require you to touch the frontend yet, only DNS and Nginx configuration.

1️⃣ Add the Subdomain in DNS

Ask your client to log in to their domain registrar (e.g., GoDaddy, Namecheap, Cloudflare).

Navigate to DNS Management / Zone File for brooklyngroup.com.

Add a new A record:

Field	Value
Type	A
Name	api
Value	138.197.93.190 (server IP)
TTL	1 hour (default)

Explanation:

api.brooklyngroup.com will point to the same server as pay.brooklyngroup.com.

Wait for DNS propagation (~5–15 minutes).

2️⃣ Update Nginx for the Subdomain

Once the subdomain is active, SSH into your server and create a new Nginx site configuration for the backend:

sudo nano /etc/nginx/sites-available/api.brooklyngroup.com


Example configuration:

server {
    listen 80;
    server_name api.brooklyngroup.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
}


Save and exit.

3️⃣ Enable the Site
sudo ln -s /etc/nginx/sites-available/api.brooklyngroup.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx


nginx -t → check syntax

systemctl reload nginx → apply configuration

4️⃣ Add SSL with Let’s Encrypt
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d api.brooklyngroup.com


Follow prompts to get SSL certificate

This enables https://api.brooklyngroup.com

5️⃣ Update Frontend API Calls

Change frontend calls from /api/... to:

axios.post("https://api.brooklyngroup.com/v1/create-payment-intent")


Now backend is fully separated on a subdomain, which avoids path conflicts and 502 / 301 issues.

✅ Result:

Frontend: https://pay.brooklyngroup.com

Backend API: https://api.brooklyngroup.com

SSL enabled for both domains

Nginx proxies requests securely
---

This guide can now serve as a **full reference for deploying any React + Node.js project** with SSL and payment integration.

```

---

I can also **generate a downloadable `.md` file** ready to save locally.  

Do you want me to create it so you can **download it directly from this chat**?

```
