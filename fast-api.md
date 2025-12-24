# FastAPI Deployment on EC2 with PM2, Nginx, and HTTPS

This is a comprehensive guide and summary of deploying a FastAPI backend on an AWS EC2 instance, managing it with PM2, exposing it via Nginx, securing it with HTTPS using Let's Encrypt, and integrating with a Vercel frontend.

---

## 1️⃣ EC2 Setup

- Launch an Ubuntu EC2 instance.
- **Security group inbound rules:**

| Type  | Protocol | Port | Source       |
|-------|---------|------|-------------|
| SSH   | TCP     | 22   | Your IP only |
| HTTP  | TCP     | 80   | 0.0.0.0/0   |
| HTTPS | TCP     | 443  | 0.0.0.0/0   |

- **Do not expose Uvicorn port 8000** publicly.
- Install system packages and PM2:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-venv python3-pip nginx git curl
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2


2️⃣ FastAPI App Setup

Clone your repository and create a Python virtual environment:

git clone <repo_url> fastapi-app
cd fastapi-app
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt


Add CORS middleware to allow frontend access:

from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://v0-real-intel.vercel.app"],
    allow_methods=["*"],
    allow_headers=["*"],
)


Test locally:

uvicorn main:app --host 127.0.0.1 --port 8000

3️⃣ PM2 Management

Start FastAPI with PM2:

pm2 start "venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000" \
  --name fastapi-api \
  --cwd /home/ubuntu/fastapi-app
pm2 save
pm2 status


Useful PM2 commands:

pm2 restart fastapi-api
pm2 stop fastapi-api
pm2 delete fastapi-api


Notes:

Bind to 127.0.0.1 so Nginx can proxy traffic.

Always test the app locally before using PM2.

4️⃣ DNS Setup (DuckDNS)

Create a free subdomain: real-intel-api.duckdns.org.

Update the A record to point to your EC2 public IPv4.

Verify propagation before running Certbot:

nslookup real-intel-api.duckdns.org  # should return public IP
ping -c 1 real-intel-api.duckdns.org
curl http://real-intel-api.duckdns.org


Lesson: Internal EC2 DNS may resolve the domain to private IP (172.31.x.x) due to split-horizon DNS — normal.

5️⃣ Nginx Reverse Proxy

Create Nginx config (/etc/nginx/sites-available/fastapi):

server {
    listen 80;
    server_name real-intel-api.duckdns.org;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}


Enable and test:

sudo ln -s /etc/nginx/sites-available/fastapi /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

6️⃣ HTTPS via Let's Encrypt

Issue: Certbot Nginx plugin can fail if internal EC2 DNS resolves domain to private IP.

Solution: Use standalone mode after DNS propagation:

sudo systemctl stop nginx
sudo certbot certonly --standalone -d real-intel-api.duckdns.org
sudo systemctl start nginx


Avoid multiple failed attempts → Let’s Encrypt rate limit: 5 failed authorizations per hour.

Configure Nginx with SSL certificates:

server {
    listen 443 ssl;
    server_name real-intel-api.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/real-intel-api.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/real-intel-api.duckdns.org/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name real-intel-api.duckdns.org;
    return 301 https://$host$request_uri;
}


Reload Nginx:

sudo nginx -t
sudo systemctl restart nginx

7️⃣ Testing

From EC2:

curl http://127.0.0.1:8000/docs
curl https://real-intel-api.duckdns.org


From browser/frontend:

fetch("https://real-intel-api.duckdns.org/your-endpoint")


Confirm CORS allows your frontend: https://v0-real-intel.vercel.app.

8️⃣ Key Lessons Learned

DNS propagation matters: Wait until external NS lookup shows the public IP before running Certbot.

Internal EC2 DNS may show private IP: Normal behavior that can confuse Certbot’s Nginx plugin.

PM2 host: Always bind to 127.0.0.1; Nginx proxies external traffic.

Security: Only expose ports 80/443; keep SSH restricted.

Certbot rate limits: Don’t retry multiple times until DNS/configuration are correct.

Test locally first: Ensure FastAPI runs via Uvicorn before PM2 or Nginx.
