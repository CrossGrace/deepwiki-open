---
layout: default
title: Deployment
---

# Deployment Guide

Complete guide to deploying DeepWiki Open in various environments.

## Table of Contents

- [Local Development](#local-development)
- [Docker Deployment](#docker-deployment)
- [Production Server](#production-server)
- [Cloud Deployment](#cloud-deployment)
- [GitHub Pages](#github-pages)
- [Enterprise Integration](#enterprise-integration)
- [Monitoring & Maintenance](#monitoring--maintenance)

## Local Development

### Prerequisites

- Python 3.11+
- Node.js 18+
- Git
- Enterprise API access

### Setup Steps

1. **Clone repository**:
   ```bash
   git clone https://github.com/CrossGrace/deepwiki-open.git
   cd deepwiki-open
   ```

2. **Install Python dependencies**:
   ```bash
   pip install -r requirements_single.txt
   ```

3. **Install Node dependencies**:
   ```bash
   yarn install
   ```

4. **Configure environment**:
   ```bash
   export GPT_OSS_API_BASE="https://your-company-api.com"
   export GPT_OSS_API_KEY="your-api-key"
   ```

5. **Run development server**:
   ```bash
   yarn dev
   ```

6. **Access application**:
   - Frontend: http://localhost:3000
   - Backend API: http://localhost:8000 (if running separately)

### Development Workflow

```bash
# Terminal 1: Run Next.js dev server
yarn dev

# Terminal 2: Generate wiki via CLI
python deepwiki_single.py \
  --repo https://github.com/your-org/repo \
  --output ./output/wiki

# Terminal 3: Watch logs
tail -f logs/deepwiki.log
```

---

## Docker Deployment

### Basic Docker Setup

1. **Build image**:
   ```bash
   docker build -t deepwiki-open:latest .
   ```

2. **Run container**:
   ```bash
   docker run -d \
     --name deepwiki \
     -p 3000:3000 \
     -e GPT_OSS_API_BASE="https://api.company.com" \
     -e GPT_OSS_API_KEY="your-api-key" \
     -v $(pwd)/output:/app/output \
     deepwiki-open:latest
   ```

3. **View logs**:
   ```bash
   docker logs -f deepwiki
   ```

4. **Stop container**:
   ```bash
   docker stop deepwiki
   docker rm deepwiki
   ```

### Docker Compose

**Create `docker-compose.yml`**:

```yaml
version: '3.8'

services:
  deepwiki:
    build: .
    container_name: deepwiki-open
    ports:
      - "3000:3000"
    environment:
      - GPT_OSS_API_BASE=${GPT_OSS_API_BASE}
      - GPT_OSS_API_KEY=${GPT_OSS_API_KEY}
      - NODE_ENV=production
    volumes:
      - ./output:/app/output
      - ./config:/app/config
      - ./logs:/app/logs
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
```

**Run with Docker Compose**:

```bash
# Start services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

### Multi-Container Setup

**Separate frontend and backend**:

```yaml
version: '3.8'

services:
  frontend:
    build:
      context: .
      dockerfile: Dockerfile.frontend
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:8000
    depends_on:
      - backend

  backend:
    build:
      context: .
      dockerfile: Dockerfile.backend
    ports:
      - "8000:8000"
    environment:
      - GPT_OSS_API_BASE=${GPT_OSS_API_BASE}
      - GPT_OSS_API_KEY=${GPT_OSS_API_KEY}
    volumes:
      - ./output:/app/output
```

**Dockerfile.frontend**:

```dockerfile
FROM node:18-alpine AS deps
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN yarn build

FROM node:18-alpine AS runner
WORKDIR /app
ENV NODE_ENV production
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/node_modules ./node_modules
COPY package.json ./
EXPOSE 3000
CMD ["yarn", "start"]
```

**Dockerfile.backend**:

```dockerfile
FROM python:3.11-slim
WORKDIR /app

# Install dependencies
COPY requirements_single.txt ./
RUN pip install --no-cache-dir -r requirements_single.txt

# Copy source
COPY api ./api
COPY deepwiki_single.py ./

# Expose port
EXPOSE 8000

# Run server
CMD ["python", "-m", "uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Production Server

### Ubuntu/Debian Server Setup

1. **Update system**:
   ```bash
   sudo apt update
   sudo apt upgrade -y
   ```

2. **Install Python 3.11**:
   ```bash
   sudo apt install python3.11 python3.11-venv python3-pip -y
   ```

3. **Install Node.js 18**:
   ```bash
   curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
   sudo apt install nodejs -y
   ```

4. **Install Yarn**:
   ```bash
   npm install -g yarn
   ```

5. **Clone and setup**:
   ```bash
   cd /opt
   sudo git clone https://github.com/CrossGrace/deepwiki-open.git
   cd deepwiki-open
   sudo chown -R $USER:$USER .

   python3.11 -m venv venv
   source venv/bin/activate
   pip install -r requirements_single.txt

   yarn install
   yarn build
   ```

6. **Create systemd service**:

**File**: `/etc/systemd/system/deepwiki.service`

```ini
[Unit]
Description=DeepWiki Open Service
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/deepwiki-open
Environment="GPT_OSS_API_BASE=https://api.company.com"
Environment="GPT_OSS_API_KEY=your-api-key"
Environment="NODE_ENV=production"
ExecStart=/usr/bin/yarn start
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

7. **Enable and start service**:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable deepwiki
   sudo systemctl start deepwiki
   sudo systemctl status deepwiki
   ```

### Nginx Reverse Proxy

**Install Nginx**:

```bash
sudo apt install nginx -y
```

**Configure Nginx**:

**File**: `/etc/nginx/sites-available/deepwiki`

```nginx
server {
    listen 80;
    server_name deepwiki.company.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://localhost:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Increase timeout for long-running operations
    proxy_read_timeout 300;
    proxy_connect_timeout 300;
    proxy_send_timeout 300;
}
```

**Enable site**:

```bash
sudo ln -s /etc/nginx/sites-available/deepwiki /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### SSL with Let's Encrypt

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Obtain certificate
sudo certbot --nginx -d deepwiki.company.com

# Auto-renewal (already enabled by default)
sudo systemctl status certbot.timer
```

**Updated Nginx config with SSL**:

```nginx
server {
    listen 80;
    server_name deepwiki.company.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name deepwiki.company.com;

    ssl_certificate /etc/letsencrypt/live/deepwiki.company.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/deepwiki.company.com/privkey.pem;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://localhost:3000;
        # ... rest of config
    }
}
```

---

## Cloud Deployment

### AWS EC2

1. **Launch EC2 instance**:
   - AMI: Ubuntu 22.04 LTS
   - Instance type: t3.medium (2 vCPU, 4 GB RAM)
   - Storage: 30 GB SSD
   - Security group: Allow ports 22, 80, 443

2. **Connect to instance**:
   ```bash
   ssh -i your-key.pem ubuntu@ec2-xx-xx-xx-xx.compute.amazonaws.com
   ```

3. **Follow production server setup** above

4. **Configure AWS Secrets Manager** (optional):
   ```bash
   # Install AWS CLI
   sudo apt install awscli -y

   # Fetch secrets
   export GPT_OSS_API_KEY=$(aws secretsmanager get-secret-value \
     --secret-id deepwiki/api-key \
     --query SecretString \
     --output text)
   ```

### AWS ECS (Elastic Container Service)

**Create task definition** (`task-definition.json`):

```json
{
  "family": "deepwiki-task",
  "containerDefinitions": [
    {
      "name": "deepwiki",
      "image": "your-ecr-repo/deepwiki-open:latest",
      "memory": 2048,
      "cpu": 1024,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "GPT_OSS_API_BASE",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:deepwiki/api-base"
        },
        {
          "name": "GPT_OSS_API_KEY",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:deepwiki/api-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/deepwiki",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "1024",
  "memory": "2048"
}
```

**Deploy**:

```bash
# Register task definition
aws ecs register-task-definition --cli-input-json file://task-definition.json

# Create service
aws ecs create-service \
  --cluster deepwiki-cluster \
  --service-name deepwiki-service \
  --task-definition deepwiki-task \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}"
```

### Google Cloud Platform (GCP)

**Deploy to Cloud Run**:

```bash
# Build and push image
gcloud builds submit --tag gcr.io/your-project/deepwiki-open

# Deploy to Cloud Run
gcloud run deploy deepwiki \
  --image gcr.io/your-project/deepwiki-open \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars GPT_OSS_API_BASE="https://api.company.com" \
  --set-secrets GPT_OSS_API_KEY=deepwiki-api-key:latest \
  --memory 2Gi \
  --cpu 2 \
  --max-instances 10
```

### Azure Container Instances

```bash
# Create resource group
az group create --name deepwiki-rg --location eastus

# Create container
az container create \
  --resource-group deepwiki-rg \
  --name deepwiki \
  --image your-acr.azurecr.io/deepwiki-open:latest \
  --cpu 2 \
  --memory 4 \
  --ports 3000 \
  --environment-variables \
    NODE_ENV=production \
  --secure-environment-variables \
    GPT_OSS_API_BASE=https://api.company.com \
    GPT_OSS_API_KEY=your-api-key \
  --dns-name-label deepwiki
```

---

## GitHub Pages

### Setup GitHub Pages for Documentation

1. **Enable GitHub Pages**:
   - Go to repository Settings
   - Navigate to Pages section
   - Source: Deploy from branch
   - Branch: `main` or `gh-pages`
   - Folder: `/docs`

2. **Configure Jekyll** (already done):
   - `docs/_config.yml` sets theme and plugins
   - Markdown files automatically converted

3. **Access documentation**:
   - URL: `https://crossgrace.github.io/deepwiki-open`

### Deploy Generated Wiki to GitHub Pages

**Option 1: Manual upload**

```bash
# Generate wiki
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./wiki-output

# Copy to docs/
cp -r ./wiki-output/* ./docs/

# Commit and push
git add docs/
git commit -m "docs: Update wiki pages"
git push origin main
```

**Option 2: Automated with GitHub Actions**

**Create `.github/workflows/generate-wiki.yml`**:

```yaml
name: Generate Wiki

on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly on Sunday
  workflow_dispatch:  # Manual trigger

jobs:
  generate:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements_single.txt

      - name: Generate wiki
        env:
          GPT_OSS_API_BASE: ${{ secrets.GPT_OSS_API_BASE }}
          GPT_OSS_API_KEY: ${{ secrets.GPT_OSS_API_KEY }}
        run: |
          python deepwiki_single.py \
            --repo https://github.com/${{ github.repository }} \
            --output ./docs/wiki

      - name: Commit and push
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add docs/wiki/
          git commit -m "docs: Auto-update wiki [skip ci]" || exit 0
          git push
```

---

## Enterprise Integration

### Internal Network Deployment

1. **Configure internal DNS**:
   ```bash
   # /etc/hosts
   192.168.1.100  deepwiki.internal.company.com
   ```

2. **Use internal API endpoints**:
   ```bash
   export GPT_OSS_API_BASE="http://internal-api.company.com:8080"
   ```

3. **Configure firewall**:
   ```bash
   # Allow internal access only
   sudo ufw allow from 192.168.0.0/16 to any port 3000
   sudo ufw enable
   ```

### SSO Integration

**Example with OAuth2**:

**File**: `src/app/api/auth/[...nextauth]/route.ts`

```typescript
import NextAuth from 'next-auth'
import { OAuthConfig } from 'next-auth/providers'

const handler = NextAuth({
  providers: [
    {
      id: 'company-sso',
      name: 'Company SSO',
      type: 'oauth',
      wellKnown: 'https://sso.company.com/.well-known/openid-configuration',
      authorization: { params: { scope: 'openid email profile' } },
      clientId: process.env.OAUTH_CLIENT_ID,
      clientSecret: process.env.OAUTH_CLIENT_SECRET,
      profile(profile) {
        return {
          id: profile.sub,
          name: profile.name,
          email: profile.email,
        }
      },
    },
  ],
  callbacks: {
    async jwt({ token, account }) {
      if (account) {
        token.accessToken = account.access_token
      }
      return token
    },
  },
})

export { handler as GET, handler as POST }
```

### LDAP Authentication

```python
# api/auth/ldap.py
import ldap

def authenticate_ldap(username, password):
    server = "ldap://ldap.company.com"
    base_dn = "dc=company,dc=com"

    try:
        conn = ldap.initialize(server)
        user_dn = f"uid={username},{base_dn}"
        conn.simple_bind_s(user_dn, password)
        return True
    except ldap.INVALID_CREDENTIALS:
        return False
```

---

## Monitoring & Maintenance

### Logging Configuration

**Python logging**:

```python
# api/logging_config.py
import logging
from logging.handlers import RotatingFileHandler

def setup_logging(log_file='logs/deepwiki.log'):
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            RotatingFileHandler(
                log_file,
                maxBytes=10*1024*1024,  # 10 MB
                backupCount=5
            ),
            logging.StreamHandler()
        ]
    )
```

### Health Checks

**API endpoint**:

```python
# api/health.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "service": "deepwiki-open",
        "version": "1.0.0"
    }
```

**Kubernetes liveness probe**:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10
```

### Performance Monitoring

**Prometheus metrics**:

```python
# api/metrics.py
from prometheus_client import Counter, Histogram

wiki_generations = Counter('wiki_generations_total', 'Total wiki generations')
generation_duration = Histogram('wiki_generation_duration_seconds', 'Wiki generation duration')

@generation_duration.time()
def generate_wiki(repo_url):
    wiki_generations.inc()
    # ... generation logic
```

### Backup Strategy

```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d)
BACKUP_DIR="/backups/deepwiki"

# Backup generated wikis
tar -czf $BACKUP_DIR/wikis-$DATE.tar.gz /app/output

# Backup configuration
tar -czf $BACKUP_DIR/config-$DATE.tar.gz /app/config

# Cleanup old backups (keep 30 days)
find $BACKUP_DIR -type f -mtime +30 -delete
```

**Cron job**:

```bash
# Run daily at 2 AM
0 2 * * * /opt/deepwiki-open/backup.sh
```

### Updates and Maintenance

**Update process**:

```bash
# Stop service
sudo systemctl stop deepwiki

# Pull latest code
cd /opt/deepwiki-open
git pull origin main

# Update dependencies
source venv/bin/activate
pip install --upgrade -r requirements_single.txt
yarn install

# Rebuild frontend
yarn build

# Restart service
sudo systemctl start deepwiki
sudo systemctl status deepwiki
```

### Troubleshooting

**Check service status**:

```bash
sudo systemctl status deepwiki
journalctl -u deepwiki -f
```

**Check logs**:

```bash
tail -f /opt/deepwiki-open/logs/deepwiki.log
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

**Test API connectivity**:

```bash
curl http://localhost:3000/health
curl http://localhost:8000/health
```

**Check resource usage**:

```bash
htop
df -h
free -m
```

---

## Deployment Checklist

### Pre-Deployment

- [ ] Environment variables configured
- [ ] API endpoints accessible
- [ ] Dependencies installed
- [ ] Configuration files reviewed
- [ ] SSL certificates obtained (if applicable)
- [ ] Firewall rules configured
- [ ] DNS records created

### Deployment

- [ ] Application built successfully
- [ ] Service started
- [ ] Health checks passing
- [ ] Logs showing no errors
- [ ] Frontend accessible
- [ ] API endpoints responding

### Post-Deployment

- [ ] Monitoring configured
- [ ] Logging working
- [ ] Backups scheduled
- [ ] Documentation updated
- [ ] Team notified
- [ ] Load testing completed (if applicable)

[← Back to Home](index.md)
