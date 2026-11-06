
# 🚀 End-to-End Full-Stack Deployment on AWS EC2

### From Zero to Production: A Beginner-Friendly DevOps Guide

Ever wondered how a frontend, backend, database, and domain work together when deploying an application to the cloud?

This repository explains the complete deployment process using AWS EC2, Ubuntu, Next.js, Node.js/FastAPI, Nginx, PM2, DNS, and SSL.

No complicated theory. Just practical concepts, architecture, commands, and deployment steps.

## 🎯 What You'll Build

A production-style full-stack application accessible through:

- https://example.com → Next.js Frontend
- https://api.example.com → Backend API

## 🏗️ 1. Understand the Architecture

```text
                 USER / BROWSER
                       |
                       v
                 DOMAIN / DNS
                       |
                       v
                 AWS EC2 SERVER
                    Ubuntu
                       |
                       v
                    NGINX
                 Reverse Proxy
                  /        \
                 /          \
                v            v
          FRONTEND        BACKEND
          Next.js         Node.js
          Port 3000       Port 5000
                              |
                              v
                          DATABASE
                       PostgreSQL /
                       MongoDB /
                       Supabase
```

### What does each component do?

| Technology | Responsibility |
|---|---|
| AWS EC2 | Provides a cloud server |
| Ubuntu | Server operating system |
| Git/GitHub | Source code management |
| Node.js | JavaScript runtime |
| Python | Backend runtime |
| Next.js | Frontend framework |
| PM2 | Node.js process manager |
| systemd | Linux service manager |
| Nginx | Reverse proxy |
| DNS | Domain-to-IP resolution |
| Let's Encrypt | Free SSL certificates |
| Certbot | SSL certificate automation |

## ☁️ 2. Create an AWS EC2 Instance

1. Open AWS Console.
2. Navigate to EC2.
3. Click Launch Instance.
4. Select Ubuntu Server 24.04 LTS.
5. Choose an appropriate instance type.
6. Create an RSA key pair.
7. Configure storage (20–30 GB gp3 is a reasonable starting point).
8. Configure Security Groups.

### Security Group Inbound Rules

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | Your IP |
| HTTP | 80 | 0.0.0.0/0 |
| HTTPS | 443 | 0.0.0.0/0 |

Never expose application or database ports publicly without a specific security requirement.

### Important: Elastic IP

Allocate and associate an Elastic IP with your EC2 instance.

Why?

A normal public IP can change after stopping and starting an instance. A stable IP makes DNS management easier.

## 🔐 3. Connect to EC2 Using SSH

Windows PowerShell:

```bash
ssh -i "my-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

Mac/Linux:

```bash
chmod 400 my-key.pem
ssh -i my-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

You are now connected to your remote Ubuntu server.

## ⚙️ 4. Prepare the Server

Update Ubuntu:

```bash
sudo apt update
sudo apt upgrade -y
```

Install essential tools:

```bash
sudo apt install -y git curl wget unzip nginx
```

Install Node.js using NVM:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

source ~/.bashrc

nvm install --lts
nvm use --lts
```

Verify:

```bash
node -v
npm -v
git --version
```

## 📂 5. Clone Your Private GitHub Repositories

Create application directories:

```bash
sudo mkdir -p /var/www
sudo chown -R ubuntu:ubuntu /var/www
cd /var/www
```

Generate an SSH deploy key:

```bash
ssh-keygen -t ed25519 -C "ec2-production"
```

Display the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Add it to:

GitHub → Repository Settings → Deploy Keys.

Clone your repositories:

```bash
git clone git@github.com:USERNAME/FRONTEND.git my-frontend

git clone git@github.com:USERNAME/BACKEND.git my-backend
```

Use read-only deploy keys wherever possible.

## 🖥️ 6. Deploy the Node.js Backend

```bash
cd /var/www/my-backend

npm ci --omit=dev
```

Create environment variables:

```bash
nano .env
```

Example:

```env
NODE_ENV=production
PORT=5000
DATABASE_URL=your_database_url
JWT_SECRET=your_secret
FRONTEND_URL=https://example.com
```

Secure the environment file:

```bash
chmod 600 .env
```

Start the application:

```bash
npm start
```

Test locally:

```bash
curl http://127.0.0.1:5000/health
```

### Introduce PM2

Install:

```bash
sudo npm install -g pm2
```

Start backend:

```bash
pm2 start npm --name my-backend -- start
```

Save process configuration:

```bash
pm2 save
pm2 startup
```

Execute the startup command printed by PM2.

Useful commands:

```bash
pm2 list
pm2 logs
pm2 monit
pm2 restart my-backend
```

PM2 helps keep Node.js applications running and restarts crashed processes.

## ⚛️ 7. Deploy the Next.js Frontend

```bash
cd /var/www/my-frontend

npm ci
```

Create production environment configuration:

```bash
nano .env.production
```

Example:

```env
NEXT_PUBLIC_API_URL=https://api.example.com/api
```

Build:

```bash
npm run build
```

Start:

```bash
pm2 start npm --name my-frontend -- start
```

Next.js normally runs on port 3000.

Verify:

```bash
curl -I http://127.0.0.1:3000
```

Important:

- `npm run build` creates the production build.
- `npm run start` runs the production server.
- Never use `npm run dev` as your production server.
- NEXT_PUBLIC variables are exposed to browser code.
- Rebuild after changing build-time public environment variables.

## 🐍 8. What If Your Backend Uses FastAPI?

The overall architecture stays the same.

Install Python:

```bash
sudo apt install -y python3 python3-venv
```

Create virtual environment:

```bash
cd /var/www/my-backend

python3 -m venv venv

source venv/bin/activate

pip install -r requirements.txt
```

Install Uvicorn:

```bash
pip install "uvicorn[standard]"
```

Run:

```bash
uvicorn app.main:app --host 127.0.0.1 --port 8000
```

For production, configure a systemd service to manage Uvicorn.

Example:

```ini
[Unit]
Description=FastAPI Backend
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/var/www/my-backend
EnvironmentFile=/var/www/my-backend/.env
ExecStart=/var/www/my-backend/venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Save as:

`/etc/systemd/system/my-backend.service`

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now my-backend
```

Check logs:

```bash
sudo journalctl -u my-backend -f
```

## 🌐 9. Configure Nginx (Reverse Proxy)

Why Nginx?

Your applications run on internal ports, but users should access them through standard HTTP/HTTPS domains.

Create:

```bash
sudo nano /etc/nginx/sites-available/my-application
```

Frontend configuration:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Backend configuration:

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:5000;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

For FastAPI, replace port 5000 with 8000.

Enable configuration:

```bash
sudo ln -s /etc/nginx/sites-available/my-application /etc/nginx/sites-enabled/
```

Test:

```bash
sudo nginx -t
```

Reload:

```bash
sudo systemctl reload nginx
```

Now Nginx routes:

- example.com → Next.js
- api.example.com → Backend

## 🌍 10. Configure DNS

Open your domain provider, such as GoDaddy.

Create these DNS records using your EC2 Elastic IP:

| Type | Host | Value |
|---|---|---|
| A | @ | YOUR_EC2_ELASTIC_IP |
| A | www | YOUR_EC2_ELASTIC_IP |
| A | api | YOUR_EC2_ELASTIC_IP |

Check:

```bash
nslookup example.com
```

Remember:

DNS maps your domain to your server IP. It does not deploy your application.

## 🔒 11. Configure HTTPS Using Let's Encrypt

Install Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Request SSL:

```bash
sudo certbot --nginx \
-d example.com \
-d www.example.com \
-d api.example.com
```

Test automatic renewal:

```bash
sudo certbot renew --dry-run
```

Now your applications can be accessed securely using HTTPS.

## 🗄️ 12. Connect Your Database

Recommended managed database options:

- AWS RDS (PostgreSQL/MySQL)
- Supabase (PostgreSQL)
- MongoDB Atlas

For RDS:

1. Create a database.
2. Prefer private networking.
3. Configure its Security Group.
4. Allow the database port only from your EC2 Security Group.
5. Configure DATABASE_URL securely.
6. Run database migrations.

Never expose production database ports publicly without a compelling and properly secured requirement.

## 🔄 13. Update Your Application

Backend:

```bash
cd /var/www/my-backend

git pull origin main

npm ci --omit=dev

pm2 restart my-backend --update-env
```

Frontend:

```bash
cd /var/www/my-frontend

git pull origin main

npm ci

npm run build

pm2 restart my-frontend --update-env
```

FastAPI:

```bash
cd /var/www/my-backend

git pull origin main

source venv/bin/activate

pip install -r requirements.txt

sudo systemctl restart my-backend
```

Always verify health and logs after deployment.

For production, plan migrations, rollback and zero-downtime deployment separately.

## 🛠️ 14. Essential Troubleshooting Commands

| Command | Purpose |
|---|---|
| `pm2 list` | Running Node processes |
| `pm2 logs` | Application logs |
| `sudo nginx -t` | Validate Nginx |
| `sudo systemctl status nginx` | Nginx status |
| `sudo ss -tulpn` | Listening ports |
| `free -h` | RAM usage |
| `df -h` | Disk usage |
| `top` | CPU/process monitoring |
| `sudo journalctl -u my-backend -f` | FastAPI logs |
| `curl -I URL` | HTTP response check |
| `nslookup example.com` | DNS check |

### Common Production Errors

| Error | First thing to investigate |
|---|---|
| 502 Bad Gateway | Application process and upstream |
| 504 Gateway Timeout | Slow upstream or timeout |
| 403 Forbidden | Permissions and Nginx rules |
| DNS not resolving | DNS records |
| SSL error | Certificate and domain |
| CORS error | Backend CORS policy |
| Database connection failed | Credentials and networking |

## 🔐 15. Production Security Checklist

- Restrict SSH access.
- Use Elastic IP.
- Keep application ports private.
- Use read-only GitHub deploy keys.
- Never commit `.env` files.
- Secure database access.
- Configure HTTPS.
- Configure process auto-restart.
- Configure database backups.
- Monitor CPU, RAM and disk.
- Maintain deployment rollback procedures.

## 🚀 16. What To Learn Next?

Once you understand manual deployment, move towards automation.

| Technology | Purpose |
|---|---|
| Docker | Containerization |
| Docker Compose | Multi-container applications |
| GitHub Actions | CI/CD |
| AWS ECR | Container image registry |
| AWS ALB | Load balancing |
| Auto Scaling | Horizontal scaling |
| Terraform | Infrastructure as Code |
| CloudWatch | Monitoring and logs |
| AWS Secrets Manager | Secret management |
| CloudFront | CDN and caching |

### CI/CD Workflow

```text
Developer Pushes Code
        |
        v
   GitHub Actions
        |
        v
 Install Dependencies
        |
        v
    Run Tests
        |
        v
    Build App
        |
        v
   Deploy to EC2
        |
        v
   Health Checks
        |
        v
 Production Updated
```

## 🧠 The Deployment Flow To Remember

1. Domain
2. Security
3. EC2
4. Operating System
5. Repository
6. Runtime
7. Process Manager
8. Nginx
9. DNS
10. SSL
11. Database
12. Monitoring

### Final Thoughts

Cloud deployment is not magic.

Once you understand what each component does and how requests travel through the infrastructure, deploying a full-stack application becomes much easier.

Start with one EC2 instance. Deploy manually. Understand every command. Then automate everything using Docker, GitHub Actions and Terraform.

**Learn by building, not just by watching tutorials.**

If this guide helps you, ⭐ star this repository and share it with someone learning DevOps!

---

### Disclaimer
This is a learning-oriented deployment walkthrough. Instance sizing, secrets handling, networking, backups, monitoring, migrations and availability strategies must be adapted to your production workload.

