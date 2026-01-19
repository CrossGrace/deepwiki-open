---
layout: default
title: Deployment
description: Docker, cloud platforms, and production deployment guides
prev_page:
  title: Configuration
  url: /configuration.html
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

DeepWiki-Open provides two Docker deployment options for different use cases.

> **📚 Complete Docker Documentation**: See [DOCKER_SETUP.md](../DOCKER_SETUP.md) for comprehensive guide with troubleshooting and advanced configuration.

### Quick Start

**Prerequisites**:
- Docker 20.10+
- Docker Compose 1.29+
- Access to GPT-OSS-130b and BGE-M3 APIs

**Setup**:

1. **Configure environment**:
   ```bash
   # Copy example environment file
   cp .env.example .env

   # Edit .env and add your API credentials
   nano .env
   ```

2. **Choose deployment option**:

   **Option A: Full Web Application**
   ```bash
   docker-compose up -d
   # Access at http://localhost:3000
   ```

   **Option B: CLI Only**
   ```bash
   docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
     --repo https://github.com/your-org/your-repo \
     --output /app/output
   ```

### Deployment Options

#### Option 1: Full Web Application

**What's included**:
- Next.js frontend (port 3000)
- FastAPI backend (port 8001)
- CLI tool (`deepwiki_single.py`)

**Resource requirements**:
- RAM: 4-6 GB
- CPU: 2+ cores
- Disk: 10+ GB

**Configuration**:

The `docker-compose.yml` is pre-configured with:
- Environment variable management
- Persistent volumes for data, workspace, and logs
- Health checks and auto-restart
- Resource limits (configurable)
- Network isolation

**Commands**:

```bash
# Start services in background
docker-compose up -d

# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f deepwiki

# Check status
docker-compose ps

# Stop services
docker-compose stop

# Stop and remove containers
docker-compose down

# Stop and remove volumes (⚠️ deletes data)
docker-compose down -v

# Rebuild after code changes
docker-compose build --no-cache
docker-compose up -d
```

**Access the application**:
- Frontend: http://localhost:3000
- Backend API: http://localhost:8001
- API Docs: http://localhost:8001/docs

#### Option 2: CLI Only (Lightweight)

**What's included**:
- Command-line tool only (`deepwiki_single.py`)
- No web interface

**Resource requirements**:
- RAM: 1-2 GB
- CPU: 1+ cores
- Disk: 5+ GB

**Use cases**:
- Automated wiki generation
- CI/CD pipelines
- Scheduled tasks (cron jobs)
- Batch processing

**Configuration**:

Uses `docker-compose.cli.yml` and `Dockerfile.cli` for minimal footprint.

**Commands**:

```bash
# Build CLI image
docker-compose -f docker-compose.cli.yml build

# Generate wiki for public repo
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/anthropics/anthropic-sdk-python \
  --output /app/output

# Generate wiki for private repo
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/private-repo \
  --token $GITHUB_TOKEN \
  --output /app/output

# Dry run (test without writing files)
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/your-repo \
  --dry-run \
  --debug

# Access generated output
ls -la ./output/
cat ./output/Home.md
```

**Alternative: Direct Docker run**:

```bash
# Build image
docker build -t deepwiki-cli -f Dockerfile.cli .

# Run with environment variables
docker run --rm \
  -e DEEPWIKI_LLM_BASE_URL="https://your-llm-api.com" \
  -e DEEPWIKI_LLM_TOKEN="your-token" \
  -e DEEPWIKI_EMBEDDING_BASE_URL="https://your-embed-api.com" \
  -e DEEPWIKI_EMBEDDING_TOKEN="your-token" \
  -v $(pwd)/output:/app/output \
  -v $(pwd)/workspace:/app/workspace \
  deepwiki-cli \
  --repo https://github.com/your-org/your-repo \
  --output /app/output
```

### Environment Configuration

**Required variables** (in `.env` file):

```bash
# LLM API (gpt-oss-130b)
DEEPWIKI_LLM_BASE_URL=https://your-llm-api.company.com
DEEPWIKI_LLM_TOKEN=your-llm-auth-token

# Embedding API (BGE-M3)
DEEPWIKI_EMBEDDING_BASE_URL=https://your-embedding-api.company.com
DEEPWIKI_EMBEDDING_TOKEN=your-embedding-auth-token
```

**Optional variables**:

```bash
# Application settings
PORT=8001
NODE_ENV=production
LOG_LEVEL=INFO
DEEPWIKI_WORKSPACE=./workspace
DEEPWIKI_OUTPUT=./wiki_output

# GitHub token (for private repos)
GITHUB_TOKEN=ghp_your_token_here

# Docker resource limits
DOCKER_MEMORY_LIMIT=6g
DOCKER_MEMORY_RESERVATION=2g
```

See [.env.example](../.env.example) for complete template.

### Volume Management

**Persistent data locations**:

```
./output/          # Generated wiki files
./workspace/       # Cloned repositories
./api/logs/        # Application logs
deepwiki-data      # Named volume for embeddings
```

**Access volumes**:

```bash
# From host
ls -la ./output/
tail -f ./api/logs/application.log

# From container
docker-compose exec deepwiki bash
cd /app/output
```

**Backup volumes**:

```bash
# Backup generated wikis
tar -czf wikis-backup-$(date +%Y%m%d).tar.gz ./output

# Backup workspace
tar -czf workspace-backup-$(date +%Y%m%d).tar.gz ./workspace
```

**Clean up volumes**:

```bash
# Remove all volumes (⚠️ deletes all data)
docker-compose down -v

# Remove specific volume
docker volume rm deepwiki-open_deepwiki-data

# Clean unused volumes
docker volume prune
```

### Production Deployment with Docker

**Best practices**:

1. **Use environment files**:
   ```bash
   # Don't commit .env to version control
   echo ".env" >> .gitignore

   # Use secrets management in production
   docker secret create deepwiki_llm_token /path/to/token
   ```

2. **Configure resource limits**:
   ```yaml
   # docker-compose.yml
   services:
     deepwiki:
       mem_limit: 8g
       mem_reservation: 4g
       cpus: 2.0
   ```

3. **Enable health checks**:
   ```bash
   # Check health status
   docker inspect deepwiki-open | grep Health

   # Test health endpoint
   curl http://localhost:8001/health
   ```

4. **Set up logging**:
   ```yaml
   # docker-compose.yml
   services:
     deepwiki:
       logging:
         driver: "json-file"
         options:
           max-size: "10m"
           max-file: "3"
   ```

5. **Configure restart policy**:
   ```yaml
   services:
     deepwiki:
       restart: unless-stopped  # or always
   ```

**Docker Swarm deployment**:

```bash
# Initialize swarm
docker swarm init

# Deploy stack
docker stack deploy -c docker-compose.yml deepwiki

# Check services
docker service ls
docker service logs deepwiki_deepwiki

# Scale service
docker service scale deepwiki_deepwiki=3

# Remove stack
docker stack rm deepwiki
```

### Enterprise Docker Configuration

**Custom certificates**:

```bash
# Create certificates directory
mkdir -p certs

# Copy CA certificates
cp /path/to/company-ca.crt certs/

# Uncomment volume mount in docker-compose.yml
# volumes:
#   - ./certs:/app/certs:ro

# Set environment variable
export CUSTOM_CERT_DIR=certs

# Rebuild and restart
docker-compose down
docker-compose build
docker-compose up -d
```

**Corporate proxy**:

```yaml
# docker-compose.yml
services:
  deepwiki:
    environment:
      - HTTP_PROXY=http://proxy.company.com:8080
      - HTTPS_PROXY=http://proxy.company.com:8080
      - NO_PROXY=localhost,127.0.0.1
```

**Private registry**:

```bash
# Build and tag
docker build -t registry.company.com/deepwiki-open:latest .

# Login to registry
docker login registry.company.com

# Push image
docker push registry.company.com/deepwiki-open:latest

# Update docker-compose.yml
# image: registry.company.com/deepwiki-open:latest
```

### Troubleshooting Docker Deployment

**Container won't start**:

```bash
# Check logs
docker-compose logs deepwiki

# Common issues:
# 1. Missing environment variables
cat .env | grep DEEPWIKI

# 2. Port conflicts
sudo lsof -i :3000
sudo lsof -i :8001

# 3. Permission issues
sudo chown -R $USER:$USER output/ workspace/ api/logs/
```

**API authentication failures**:

```bash
# Test API connectivity
curl -H "x-dep-ticket: your-token" \
     https://your-llm-api.com/health

# Verify environment variables are loaded
docker-compose exec deepwiki env | grep DEEPWIKI
```

**Out of memory**:

```bash
# Increase memory limit in .env
echo "DOCKER_MEMORY_LIMIT=8g" >> .env
echo "DOCKER_MEMORY_RESERVATION=4g" >> .env

# Restart
docker-compose down
docker-compose up -d

# Monitor resource usage
docker stats deepwiki-open
```

**Build failures**:

```bash
# Clean build cache
docker-compose build --no-cache

# Remove old images
docker image prune -a

# Check disk space
docker system df
docker system prune
```

### Automated Deployment Examples

**Cron job for scheduled wiki generation**:

```bash
# Edit crontab
crontab -e

# Add daily wiki generation at 2 AM
0 2 * * * cd /opt/deepwiki-open && docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli --repo https://github.com/your-org/repo --output /app/output >> /var/log/deepwiki-cron.log 2>&1
```

**Batch processing script**:

```bash
#!/bin/bash
# generate-wikis.sh

REPOS=(
  "https://github.com/org/repo1"
  "https://github.com/org/repo2"
  "https://github.com/org/repo3"
)

for repo in "${REPOS[@]}"; do
  echo "Generating wiki for $repo..."
  docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
    --repo "$repo" \
    --output /app/output
done
```

### Additional Resources

- **Complete Docker Guide**: [DOCKER_SETUP.md](../DOCKER_SETUP.md) (English)
- **Korean Docker Guide**: [DOCKER_SETUP.kr.md](../DOCKER_SETUP.kr.md) (한글)
- **Environment Template**: [.env.example](../.env.example)
- **Docker Compose Config**: [docker-compose.yml](../docker-compose.yml)
- **CLI Docker Config**: [docker-compose.cli.yml](../docker-compose.cli.yml)

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

