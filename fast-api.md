# FastAPI Deployment Guide: EC2 + PM2 + Nginx + HTTPS

Quick reference for deploying FastAPI on AWS EC2 with reverse proxy and SSL.

---

## EC2 Setup

**Security Group Rules:**
- SSH (22): Your IP only
- HTTP (80): 0.0.0.0/0
- HTTPS (443): 0.0.0.0/0
- ❌ Don't expose port 8000

**Install Dependencies:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-venv python3-pip nginx git curl
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
```

---

## FastAPI Setup

```bash
git clone <repo_url> fastapi-app
cd fastapi-app
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**Add CORS:**
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://v0-real-intel.vercel.app"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## PM2 Process Management

**Start:**
```bash
pm2 start "venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000" \
  --name fastapi-api --cwd /home/ubuntu/fastapi-app
pm2 save
```

**Manage:**
```bash
pm2 status
pm2 restart fastapi-api
pm2 stop fastapi-api
```

---

## DNS Setup (DuckDNS)

1. Create subdomain: `real-intel-api.duckdns.org`
2. Point A record to EC2 public IP
3. **Verify propagation:**
```bash
nslookup real-intel-api.duckdns.org
curl http://real-intel-api.duckdns.org
```

⚠️ Internal EC2 DNS may show private IP (172.31.x.x) — this is normal.

---

## Nginx Configuration

**Create `/etc/nginx/sites-available/fastapi`:**
```nginx
server {
    listen 80;
    server_name real-intel-api.duckdns.org;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Enable:**
```bash
sudo ln -s /etc/nginx/sites-available/fastapi /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## HTTPS with Let's Encrypt

**Issue Certificate (standalone mode):**
```bash
sudo systemctl stop nginx
sudo certbot certonly --standalone -d real-intel-api.duckdns.org
sudo systemctl start nginx
```

⚠️ **Rate limit:** 5 failed attempts/hour. Verify DNS first!

**Update Nginx (`/etc/nginx/sites-available/fastapi`):**
```nginx
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
```

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## Testing

**Local:**
```bash
curl http://127.0.0.1:8000/docs
```

**Remote:**
```bash
curl https://real-intel-api.duckdns.org
```

---

## Key Takeaways

✅ Bind FastAPI to `127.0.0.1` (not 0.0.0.0)  
✅ Wait for DNS propagation before Certbot  
✅ Use standalone mode if Nginx plugin fails  
✅ Only expose ports 80/443 publicly  
✅ Test locally before PM2/Nginx  
✅ Watch Certbot rate limits
