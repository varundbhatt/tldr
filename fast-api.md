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
