# Docker Setup Guide for DeepWiki-Open

Complete guide for running DeepWiki-Open using Docker.

## Table of Contents

- [Quick Start](#quick-start)
- [Two Deployment Options](#two-deployment-options)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Option 1: Full Web Application](#option-1-full-web-application)
- [Option 2: CLI Only](#option-2-cli-only)
- [Environment Variables](#environment-variables)
- [Volume Management](#volume-management)
- [Troubleshooting](#troubleshooting)
- [Advanced Configuration](#advanced-configuration)

---

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/CrossGrace/deepwiki-open.git
cd deepwiki-open
```

### 2. Configure Environment

```bash
# Copy the example environment file
cp .env.example .env

# Edit .env and fill in your API credentials
nano .env  # or use your preferred editor
```

### 3. Choose Your Deployment Option

**Full Web Application:**
```bash
docker-compose up -d
```

**CLI Only:**
```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/your-repo \
  --output /app/output
```

---

## Two Deployment Options

DeepWiki-Open offers two Docker deployment options:

### Option 1: Full Web Application

- **What it includes**: Next.js frontend + FastAPI backend + CLI tool
- **Use case**: Interactive web interface for generating wikis
- **Resource requirements**: ~4-6 GB RAM, 2+ CPU cores
- **Ports**: 3000 (frontend), 8001 (backend API)
- **Docker files**: `Dockerfile`, `docker-compose.yml`

### Option 2: CLI Only

- **What it includes**: Command-line tool only (`deepwiki_single.py`)
- **Use case**: Automated wiki generation, CI/CD pipelines, scheduled tasks
- **Resource requirements**: ~1-2 GB RAM, 1+ CPU cores
- **Ports**: None (CLI only)
- **Docker files**: `Dockerfile.cli`, `docker-compose.cli.yml`

---

## Prerequisites

### Required Software

- **Docker**: Version 20.10 or higher
- **Docker Compose**: Version 1.29 or higher

Install Docker:
- **Ubuntu/Debian**: `sudo apt-get install docker.io docker-compose`
- **macOS**: Install [Docker Desktop](https://www.docker.com/products/docker-desktop)
- **Windows**: Install [Docker Desktop](https://www.docker.com/products/docker-desktop)

### Required API Access

You must have access to:

1. **GPT-OSS-130b LLM API** (OpenAI-compatible endpoint)
   - Base URL
   - Authentication token

2. **BGE-M3 Embedding API** (1024-dimensional embeddings)
   - Base URL
   - Authentication token

### Optional

- **GitHub Personal Access Token** (for private repositories)
  - Generate at: https://github.com/settings/tokens
  - Required scopes: `repo` (for full access to private repos)

---

## Configuration

### Step 1: Create Environment File

Copy the example environment file:

```bash
cp .env.example .env
```

### Step 2: Edit Environment File

Open `.env` in your text editor and configure the required variables:

```bash
# Required: LLM API
DEEPWIKI_LLM_BASE_URL=https://your-llm-api.company.com
DEEPWIKI_LLM_TOKEN=your-llm-auth-token

# Required: Embedding API
DEEPWIKI_EMBEDDING_BASE_URL=https://your-embedding-api.company.com
DEEPWIKI_EMBEDDING_TOKEN=your-embedding-auth-token

# Optional: GitHub token for private repos
GITHUB_TOKEN=ghp_your_github_token_here
```

See [Environment Variables](#environment-variables) section for complete list.

---

## Option 1: Full Web Application

### Build and Start

```bash
# Build and start in detached mode
docker-compose up -d

# Or build first, then start
docker-compose build
docker-compose up -d
```

### Access the Application

- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:8001
- **API Documentation**: http://localhost:8001/docs

### View Logs

```bash
# View all logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f deepwiki
```

### Stop and Remove

```bash
# Stop containers
docker-compose stop

# Stop and remove containers
docker-compose down

# Stop, remove containers, and remove volumes
docker-compose down -v
```

### Using the Web Interface

1. Open http://localhost:3000 in your browser
2. Enter the GitHub repository URL
3. Optionally configure generation settings
4. Click "Generate Wiki"
5. Download or view the generated wiki pages

---

## Option 2: CLI Only

### Build the CLI Image

```bash
docker-compose -f docker-compose.cli.yml build
```

### Run CLI Commands

**Basic usage:**

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/anthropics/anthropic-sdk-python \
  --output /app/output
```

**With GitHub token for private repo:**

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/private-repo \
  --token $GITHUB_TOKEN \
  --output /app/output
```

**Dry run (test without writing):**

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/your-repo \
  --dry-run \
  --debug
```

**Custom workspace:**

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/your-repo \
  --workspace /app/workspace \
  --output /app/output
```

### Alternative: Direct Docker Run

Without docker-compose:

```bash
# Build image
docker build -t deepwiki-cli -f Dockerfile.cli .

# Run with environment variables
docker run --rm \
  -e DEEPWIKI_LLM_BASE_URL="https://your-api.com" \
  -e DEEPWIKI_LLM_TOKEN="your-token" \
  -e DEEPWIKI_EMBEDDING_BASE_URL="https://your-embed-api.com" \
  -e DEEPWIKI_EMBEDDING_TOKEN="your-token" \
  -v $(pwd)/output:/app/output \
  -v $(pwd)/workspace:/app/workspace \
  deepwiki-cli \
  --repo https://github.com/your-org/your-repo \
  --output /app/output
```

### Access Generated Output

The generated wiki files will be in the `./output` directory:

```bash
ls -la ./output/
cat ./output/Home.md
cat ./output/_Sidebar.md
```

---

## Environment Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `DEEPWIKI_LLM_BASE_URL` | Base URL for GPT-OSS-130b API | `https://llm-api.company.com` |
| `DEEPWIKI_LLM_TOKEN` | Auth token for LLM API | `your-secret-token` |
| `DEEPWIKI_EMBEDDING_BASE_URL` | Base URL for BGE-M3 API | `https://embed-api.company.com` |
| `DEEPWIKI_EMBEDDING_TOKEN` | Auth token for embedding API | `your-secret-token` |

### Optional Variables (Application)

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Backend API port | `8001` |
| `NODE_ENV` | Node environment | `production` |
| `LOG_LEVEL` | Logging level | `INFO` |
| `DEEPWIKI_WORKSPACE` | Workspace for cloning repos | `./workspace` |
| `DEEPWIKI_OUTPUT` | Default output directory | `./wiki_output` |
| `GITHUB_TOKEN` | GitHub access token | (empty) |

### Optional Variables (Docker)

| Variable | Description | Default |
|----------|-------------|---------|
| `DOCKER_MEMORY_LIMIT` | Memory limit for container | `6g` |
| `DOCKER_MEMORY_RESERVATION` | Memory reservation | `2g` |
| `CUSTOM_CERT_DIR` | Custom certificate directory | `certs` |

### Multi-Provider Mode (Optional)

Only needed if using the full multi-provider version:

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | OpenAI API key |
| `GOOGLE_API_KEY` | Google API key |

---

## Volume Management

### Volumes in Full Web Application

The `docker-compose.yml` mounts several volumes:

```yaml
volumes:
  - deepwiki-data:/root/.adalflow  # Persistent data
  - ./output:/app/output           # Generated wikis
  - ./workspace:/app/workspace     # Cloned repos
  - ./api/logs:/app/api/logs       # Application logs
```

### Accessing Generated Files

**From host machine:**

```bash
# View generated wiki files
ls -la ./output/

# View logs
tail -f ./api/logs/application.log

# View cloned repositories
ls -la ./workspace/
```

**From inside container:**

```bash
# Execute bash inside container
docker-compose exec deepwiki bash

# Navigate to output directory
cd /app/output
ls -la
```

### Cleaning Up Volumes

```bash
# Remove all stopped containers and volumes
docker-compose down -v

# Remove specific volume
docker volume rm deepwiki-open_deepwiki-data

# Remove all unused volumes
docker volume prune
```

---

## Troubleshooting

### Issue: Container fails to start

**Check logs:**

```bash
docker-compose logs deepwiki
```

**Common causes:**
1. Missing environment variables in `.env`
2. Invalid API credentials
3. Port conflicts (3000 or 8001 already in use)

**Solutions:**

```bash
# Verify .env file exists and has required variables
cat .env

# Check for port conflicts
sudo lsof -i :3000
sudo lsof -i :8001

# Use different ports
PORT=8002 docker-compose up
```

### Issue: API authentication failures

**Error message:** `HTTP 401 Unauthorized`

**Solutions:**

1. Verify API tokens are correct in `.env`
2. Check if tokens have expired
3. Test API connectivity:

```bash
# Test LLM API
curl -H "x-dep-ticket: your-token" \
     https://your-llm-api.com/health

# Test Embedding API
curl -H "x-dep-ticket: your-token" \
     https://your-embed-api.com/health
```

### Issue: Out of memory errors

**Error message:** `Container killed due to memory limit`

**Solutions:**

1. Increase memory limit in `.env`:

```bash
DOCKER_MEMORY_LIMIT=8g
DOCKER_MEMORY_RESERVATION=4g
```

2. Or modify `docker-compose.yml`:

```yaml
mem_limit: 8g
mem_reservation: 4g
```

3. Restart with new limits:

```bash
docker-compose down
docker-compose up -d
```

### Issue: Permission denied on volumes

**Error message:** `Permission denied` when writing to volumes

**Solutions:**

```bash
# Create directories with correct permissions
mkdir -p output workspace api/logs
chmod 777 output workspace api/logs

# Or run container as specific user
docker-compose run --user $(id -u):$(id -g) deepwiki-cli ...
```

### Issue: Build failures

**Solutions:**

```bash
# Clean build cache
docker-compose build --no-cache

# Remove old images
docker image prune -a

# Check Docker disk space
docker system df
docker system prune
```

### Issue: Health check failures

**Check health status:**

```bash
docker-compose ps
docker inspect deepwiki-open | grep Health
```

**Test health endpoint manually:**

```bash
curl http://localhost:8001/health
```

---

## Advanced Configuration

### Custom Certificates (Enterprise)

If your enterprise uses custom CA certificates:

1. **Create certificates directory:**

```bash
mkdir -p certs
```

2. **Copy your CA certificates:**

```bash
cp /path/to/company-ca.crt certs/
```

3. **Uncomment volume mount in `docker-compose.yml`:**

```yaml
volumes:
  - ./certs:/app/certs:ro
```

4. **Set environment variable:**

```bash
export CUSTOM_CERT_DIR=certs
```

5. **Rebuild and restart:**

```bash
docker-compose down
docker-compose build
docker-compose up -d
```

### Running Behind Corporate Proxy

Add proxy configuration to `docker-compose.yml`:

```yaml
services:
  deepwiki:
    environment:
      - HTTP_PROXY=http://proxy.company.com:8080
      - HTTPS_PROXY=http://proxy.company.com:8080
      - NO_PROXY=localhost,127.0.0.1
```

### Persistent Development Mode

For development with hot-reload:

```bash
# Override with development compose file
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
```

Create `docker-compose.dev.yml`:

```yaml
version: '3.8'

services:
  deepwiki:
    volumes:
      - ./src:/app/src
      - ./api:/app/api
    environment:
      - NODE_ENV=development
      - LOG_LEVEL=DEBUG
    command: yarn dev
```

### Scheduled Wiki Generation (Cron)

Run wiki generation on a schedule using cron:

```bash
# Edit crontab
crontab -e

# Add scheduled job (daily at 2 AM)
0 2 * * * cd /path/to/deepwiki-open && docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli --repo https://github.com/your-org/repo --output /app/output >> /var/log/deepwiki-cron.log 2>&1
```

### Docker Swarm / Kubernetes

For production orchestration, see:

- **Docker Swarm**: Modify `docker-compose.yml` to use `deploy` instead of `mem_limit`
- **Kubernetes**: Convert with `kompose convert` or use provided k8s manifests (if available)

---

## Best Practices

### Security

1. **Never commit `.env` file** - it contains sensitive credentials
2. **Use secrets management** in production (Docker secrets, Vault, etc.)
3. **Regularly rotate API tokens**
4. **Run containers as non-root user** when possible
5. **Scan images for vulnerabilities**: `docker scan deepwiki-open`

### Performance

1. **Adjust memory limits** based on repository size:
   - Small repos (<100 files): 2GB
   - Medium repos (100-500 files): 4GB
   - Large repos (500+ files): 6-8GB

2. **Use volume caching** for faster rebuilds
3. **Optimize for multi-stage builds** (already done in Dockerfile)

### Maintenance

1. **Regular updates:**

```bash
git pull origin main
docker-compose build --no-cache
docker-compose up -d
```

2. **Monitor logs:**

```bash
docker-compose logs -f --tail=100
```

3. **Backup generated wikis:**

```bash
tar -czf wiki-backup-$(date +%Y%m%d).tar.gz ./output
```

4. **Clean up periodically:**

```bash
docker system prune -a --volumes
```

---

## Examples

### Example 1: Generate Wiki for Public Repository

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/anthropics/anthropic-sdk-python \
  --output /app/output
```

### Example 2: Generate Wiki for Private Repository

```bash
docker-compose -f docker-compose.cli.yml run --rm \
  -e GITHUB_TOKEN=ghp_your_token_here \
  deepwiki-cli \
  --repo https://github.com/your-company/private-repo \
  --token $GITHUB_TOKEN \
  --output /app/output
```

### Example 3: Dry Run with Debug Logging

```bash
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli \
  --repo https://github.com/your-org/your-repo \
  --dry-run \
  --debug
```

### Example 4: Multiple Repositories in Batch

Create a script `generate-wikis.sh`:

```bash
#!/bin/bash

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

echo "All wikis generated!"
```

Run it:

```bash
chmod +x generate-wikis.sh
./generate-wikis.sh
```

---

## Support

For issues or questions:

1. Check this documentation thoroughly
2. Review error messages in logs: `docker-compose logs -f`
3. Enable debug mode: `--debug` flag or `LOG_LEVEL=DEBUG`
4. Check existing issues: https://github.com/CrossGrace/deepwiki-open/issues
5. Create a new issue with:
   - Docker version: `docker --version`
   - Docker Compose version: `docker-compose --version`
   - Error logs
   - Steps to reproduce

---

## Quick Reference

### Common Commands

```bash
# Full web application
docker-compose up -d                    # Start
docker-compose logs -f                  # View logs
docker-compose stop                     # Stop
docker-compose down                     # Stop and remove
docker-compose down -v                  # Stop, remove, and delete volumes

# CLI only
docker-compose -f docker-compose.cli.yml build                                # Build
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli --help         # Help
docker-compose -f docker-compose.cli.yml run --rm deepwiki-cli [options]      # Run

# Maintenance
docker-compose build --no-cache         # Rebuild
docker-compose exec deepwiki bash       # Shell access
docker system prune -a                  # Clean up
```

### File Overview

```
deepwiki-open/
├── .env.example              # Example environment configuration
├── .env                      # Your environment configuration (create this)
├── Dockerfile                # Full web application image
├── Dockerfile.cli            # CLI-only lightweight image
├── docker-compose.yml        # Full web application compose
├── docker-compose.cli.yml    # CLI-only compose
├── DOCKER_SETUP.md           # This file
└── output/                   # Generated wiki files (mounted volume)
```

---

**Last Updated**: January 2026

**Version**: 1.0
