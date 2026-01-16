---
layout: default
title: Configuration
description: Environment variables, LLM settings, and pipeline configuration
prev_page:
  title: API Reference
  url: /api-reference.html
next_page:
  title: Deployment
  url: /deployment.html
---

# Configuration Guide

Complete guide to configuring DeepWiki Open for your enterprise environment.

## Table of Contents

- [Environment Variables](#environment-variables)
- [Configuration Files](#configuration-files)
- [LLM Configuration](#llm-configuration)
- [Embedding Configuration](#embedding-configuration)
- [Pipeline Settings](#pipeline-settings)
- [Frontend Configuration](#frontend-configuration)
- [Docker Configuration](#docker-configuration)

## Environment Variables

### Required Variables

DeepWiki Open requires two environment variables to function:

```bash
# Enterprise API base URL
export GPT_OSS_API_BASE="https://your-company-api.com"

# API authentication key
export GPT_OSS_API_KEY="your-api-key-here"
```

### Optional Variables

```bash
# Enable verbose logging
export DEEPWIKI_VERBOSE=true

# Custom temporary directory
export DEEPWIKI_TEMP_DIR="/tmp/deepwiki"

# Maximum concurrent API requests
export DEEPWIKI_MAX_CONCURRENT=5

# Default output directory
export DEEPWIKI_OUTPUT_DIR="./output/wiki"
```

### Setting Variables

**Linux/macOS**:

```bash
# Temporary (current session)
export GPT_OSS_API_BASE="https://api.company.com"
export GPT_OSS_API_KEY="abc123xyz"

# Permanent (add to ~/.bashrc or ~/.zshrc)
echo 'export GPT_OSS_API_BASE="https://api.company.com"' >> ~/.bashrc
echo 'export GPT_OSS_API_KEY="abc123xyz"' >> ~/.bashrc
source ~/.bashrc
```

**Windows**:

```powershell
# PowerShell
$env:GPT_OSS_API_BASE = "https://api.company.com"
$env:GPT_OSS_API_KEY = "abc123xyz"

# Permanent (System Properties > Environment Variables)
setx GPT_OSS_API_BASE "https://api.company.com"
setx GPT_OSS_API_KEY "abc123xyz"
```

### Using .env File

Create a `.env` file in the project root:

```bash
# .env
GPT_OSS_API_BASE=https://your-company-api.com
GPT_OSS_API_KEY=your-api-key-here
DEEPWIKI_VERBOSE=true
DEEPWIKI_OUTPUT_DIR=./wiki-output
```

**Load with Python**:

```python
from dotenv import load_dotenv
load_dotenv()
```

**Load in Shell**:

```bash
export $(cat .env | xargs)
```

---

## Configuration Files

### Generator Configuration

**Location**: `config_examples/enterprise_generator.json`

Complete LLM configuration:

```json
{
  "model": "gpt-oss-130b",
  "api_base": "https://your-company-api.com",
  "api_key": "${GPT_OSS_API_KEY}",
  "timeout": 30,
  "max_retries": 3,
  "retry_delays": [2, 4, 8],
  "generation": {
    "temperature": 0.7,
    "max_tokens": 4000,
    "top_p": 1.0,
    "frequency_penalty": 0.0,
    "presence_penalty": 0.0
  },
  "planning": {
    "temperature": 0.5,
    "max_tokens": 2000
  }
}
```

**Parameter Reference**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | `"gpt-oss-130b"` | LLM model identifier |
| `api_base` | string | *Required* | Enterprise API base URL |
| `api_key` | string | *Required* | API key (supports `${VAR}` substitution) |
| `timeout` | number | `30` | Request timeout in seconds |
| `max_retries` | number | `3` | Maximum retry attempts |
| `retry_delays` | array | `[2, 4, 8]` | Exponential backoff delays (seconds) |
| `generation.temperature` | number | `0.7` | Sampling temperature for page generation |
| `generation.max_tokens` | number | `4000` | Max tokens per page |
| `generation.top_p` | number | `1.0` | Nucleus sampling parameter |
| `planning.temperature` | number | `0.5` | Temperature for wiki planning (lower = more focused) |

**Using the config**:

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/repo \
  --generator-config config_examples/enterprise_generator.json \
  --output ./wiki
```

---

### Embedder Configuration

**Location**: `config_examples/enterprise_embedder.json`

Complete embedding configuration:

```json
{
  "model": "BAAI/bge-m3",
  "api_base": "https://your-company-api.com",
  "api_key": "${GPT_OSS_API_KEY}",
  "dimensions": 1024,
  "batch_size": 100,
  "timeout": 60,
  "max_retries": 3,
  "retry_delays": [2, 4, 8],
  "fallback_on_batch_failure": true,
  "encoding_format": "float"
}
```

**Parameter Reference**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | `"BAAI/bge-m3"` | Embedding model identifier |
| `api_base` | string | *Required* | Enterprise API base URL |
| `api_key` | string | *Required* | API key |
| `dimensions` | number | `1024` | Embedding vector dimensions |
| `batch_size` | number | `100` | Texts per batch request |
| `timeout` | number | `60` | Request timeout in seconds |
| `max_retries` | number | `3` | Maximum retry attempts |
| `fallback_on_batch_failure` | boolean | `true` | Retry individual texts if batch fails |
| `encoding_format` | string | `"float"` | Embedding format (`float` or `base64`) |

**Performance Tuning**:

```json
{
  "batch_size": 200,  // Increase for faster processing (if API supports)
  "timeout": 120,     // Increase for large batches
  "max_retries": 5    // Increase for unreliable networks
}
```

---

## LLM Configuration

### Temperature Settings

Temperature controls randomness in generation:

| Use Case | Temperature | Description |
|----------|-------------|-------------|
| Planning | `0.3-0.5` | Focused, consistent structure |
| Technical Docs | `0.5-0.7` | Balanced creativity and accuracy |
| Creative Content | `0.8-1.0` | More varied language |
| Deterministic | `0.0` | Always same output (not recommended) |

**Example**:

```json
{
  "planning": {
    "temperature": 0.4  // Consistent wiki structure
  },
  "generation": {
    "temperature": 0.7  // Natural documentation style
  }
}
```

### Token Limits

Control output length:

```json
{
  "planning": {
    "max_tokens": 2000  // Planning JSON (typically < 1000 tokens)
  },
  "generation": {
    "max_tokens": 4000  // Page content (1-4k tokens per page)
  }
}
```

**Recommendations**:
- **Small pages** (500-1000 words): `max_tokens: 2000`
- **Medium pages** (1000-2000 words): `max_tokens: 4000`
- **Large pages** (2000+ words): `max_tokens: 6000`

### Retry Configuration

Customize retry behavior:

```json
{
  "max_retries": 4,
  "retry_delays": [2, 4, 8, 16],  // Exponential backoff
  "retry_on_status": [429, 500, 502, 503, 504],
  "retry_on_timeout": true
}
```

**Aggressive Retry** (poor network):
```json
{
  "max_retries": 5,
  "retry_delays": [1, 2, 4, 8, 16],
  "timeout": 60
}
```

**Conservative Retry** (good network):
```json
{
  "max_retries": 2,
  "retry_delays": [2, 4],
  "timeout": 15
}
```

---

## Embedding Configuration

### Batch Size Tuning

Adjust batch size based on API limits and performance:

```json
{
  "batch_size": 50   // Conservative (slower, more reliable)
}
```

```json
{
  "batch_size": 200  // Aggressive (faster, may hit rate limits)
}
```

**Recommendations**:

| Repository Size | Batch Size | Rationale |
|----------------|-----------|-----------|
| < 100 files | 50 | Small repos don't benefit from large batches |
| 100-500 files | 100 | Balanced performance |
| 500-1000 files | 150 | Faster processing |
| 1000+ files | 200 | Maximum throughput (if API supports) |

### Fallback Strategy

Configure fallback behavior:

```json
{
  "fallback_on_batch_failure": true,   // Retry individual texts
  "use_zero_vector_on_failure": true,  // Use zero vector if text fails
  "skip_failed_chunks": false          // Or skip entirely
}
```

**Strategies**:

1. **Aggressive** (complete all embeddings):
   ```json
   {
     "fallback_on_batch_failure": true,
     "use_zero_vector_on_failure": true,
     "max_retries": 5
   }
   ```

2. **Conservative** (fail fast):
   ```json
   {
     "fallback_on_batch_failure": false,
     "use_zero_vector_on_failure": false,
     "max_retries": 2
   }
   ```

---

## Pipeline Settings

### Chunking Configuration

Control how files are split:

```python
# Small chunks (better for precise retrieval)
chunker = TextChunker(chunk_size=300, chunk_overlap=50)

# Medium chunks (balanced)
chunker = TextChunker(chunk_size=500, chunk_overlap=100)

# Large chunks (more context per chunk)
chunker = TextChunker(chunk_size=800, chunk_overlap=200)
```

**Command line**:

```bash
python deepwiki_single.py \
  --repo URL \
  --chunk-size 500 \
  --chunk-overlap 100 \
  --output ./wiki
```

**Recommendations**:

| Content Type | Chunk Size | Overlap | Rationale |
|-------------|-----------|---------|-----------|
| Code files | 400-600 | 100-150 | Preserve function context |
| Documentation | 600-800 | 150-200 | Natural paragraph breaks |
| Configuration | 200-400 | 50-100 | Small, focused chunks |

### File Filtering

**Include extensions**:

```bash
--include-exts .py,.js,.ts,.tsx,.jsx,.md,.json,.yml,.yaml
```

**Exclude patterns**:

```bash
--exclude-patterns "test_*,*.test.*,node_modules/*,dist/*,build/*,.git/*"
```

**Configuration file** (`pipeline_config.json`):

```json
{
  "ingest": {
    "include_extensions": [
      ".py", ".js", ".ts", ".tsx", ".jsx",
      ".md", ".json", ".yml", ".yaml",
      ".go", ".rs", ".java", ".cpp", ".h"
    ],
    "exclude_patterns": [
      "test_*",
      "*.test.*",
      "*_test.py",
      "node_modules/*",
      "dist/*",
      "build/*",
      ".git/*",
      "*.pyc",
      "__pycache__/*"
    ],
    "max_file_size_mb": 1
  }
}
```

### Planning Configuration

Control wiki structure:

```json
{
  "plan": {
    "max_pages": 7,
    "min_pages": 3,
    "required_pages": ["Home"],
    "page_types": [
      "Overview",
      "Architecture",
      "API Reference",
      "Configuration",
      "Deployment"
    ]
  }
}
```

**Command line**:

```bash
python deepwiki_single.py \
  --repo URL \
  --max-pages 5 \
  --output ./wiki
```

### Generation Configuration

Control RAG retrieval:

```json
{
  "generate": {
    "top_k_chunks": 10,           // Chunks to retrieve per page
    "max_context_tokens": 6000,   // Max context for LLM
    "max_matched_files": 10,      // Max files to include
    "chars_per_file": 2000,       // Chars to include per file
    "chars_per_chunk": 1000       // Chars to include per chunk
  }
}
```

**Command line**:

```bash
python deepwiki_single.py \
  --repo URL \
  --top-k 15 \
  --max-context 8000 \
  --output ./wiki
```

---

## Frontend Configuration

### Next.js Configuration

**File**: `next.config.ts`

```typescript
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // API rewrites
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'http://localhost:8000/:path*',  // Python backend
      },
    ]
  },

  // Environment variables
  env: {
    GPT_OSS_API_BASE: process.env.GPT_OSS_API_BASE,
    GPT_OSS_API_KEY: process.env.GPT_OSS_API_KEY,
  },

  // Build optimization
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },

  // Image optimization
  images: {
    domains: ['github.com', 'raw.githubusercontent.com'],
  },
}

export default nextConfig
```

### Internationalization

**File**: `src/i18n.ts`

```typescript
export const locales = [
  'en', 'zh', 'ja', 'ko', 'es', 'fr', 'de', 'it', 'pt', 'ru',
  'ar', 'hi', 'tr', 'pl'
] as const

export type Locale = typeof locales[number]

export const defaultLocale: Locale = 'en'
```

**Add new language**:

1. Create `src/messages/{locale}.json`:
   ```json
   {
     "common": {
       "title": "DeepWiki Open",
       "subtitle": "Enterprise Wiki Generator"
     }
   }
   ```

2. Update `src/i18n.ts`:
   ```typescript
   export const locales = [..., 'nl'] as const
   ```

### Theme Configuration

**File**: `tailwind.config.js`

```javascript
module.exports = {
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          500: '#3b82f6',
          900: '#1e3a8a',
        },
      },
    },
  },
}
```

---

## Docker Configuration

### Dockerfile

**Multi-stage build**:

```dockerfile
# Stage 1: Node dependencies
FROM node:18-alpine AS node-deps
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

# Stage 2: Next.js build
FROM node:18-alpine AS next-build
WORKDIR /app
COPY --from=node-deps /app/node_modules ./node_modules
COPY . .
RUN yarn build

# Stage 3: Python base
FROM python:3.11-slim AS python-base
WORKDIR /app
COPY requirements_single.txt ./
RUN pip install --no-cache-dir -r requirements_single.txt

# Stage 4: Final image
FROM python:3.11-slim
WORKDIR /app

# Copy Python dependencies
COPY --from=python-base /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages

# Copy Next.js build
COPY --from=next-build /app/.next ./.next
COPY --from=next-build /app/public ./public
COPY --from=next-build /app/node_modules ./node_modules

# Copy source code
COPY . .

# Expose ports
EXPOSE 3000 8000

# Start both servers
CMD ["sh", "-c", "yarn start & python deepwiki_single.py --serve"]
```

### Docker Compose

**File**: `docker-compose.yml`

```yaml
version: '3.8'

services:
  deepwiki:
    build: .
    ports:
      - "3000:3000"  # Frontend
      - "8000:8000"  # Backend API
    environment:
      - GPT_OSS_API_BASE=${GPT_OSS_API_BASE}
      - GPT_OSS_API_KEY=${GPT_OSS_API_KEY}
      - NODE_ENV=production
    volumes:
      - ./output:/app/output  # Persist generated wikis
      - ./config:/app/config  # Configuration files
    restart: unless-stopped
```

**Run with Docker Compose**:

```bash
# Build and start
docker-compose up -d

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

### Environment-Specific Configs

**Development** (`.env.development`):

```bash
NODE_ENV=development
GPT_OSS_API_BASE=https://dev-api.company.com
DEEPWIKI_VERBOSE=true
```

**Production** (`.env.production`):

```bash
NODE_ENV=production
GPT_OSS_API_BASE=https://api.company.com
DEEPWIKI_VERBOSE=false
```

**Load specific env**:

```bash
# Development
docker-compose --env-file .env.development up

# Production
docker-compose --env-file .env.production up
```

---

## Configuration Examples

### Example 1: High-Quality Documentation

Optimize for best quality:

```json
{
  "generator": {
    "temperature": 0.5,
    "max_tokens": 6000
  },
  "embedder": {
    "batch_size": 50
  },
  "pipeline": {
    "chunk_size": 600,
    "chunk_overlap": 150,
    "top_k_chunks": 15,
    "max_context_tokens": 8000
  }
}
```

### Example 2: Fast Processing

Optimize for speed:

```json
{
  "generator": {
    "temperature": 0.7,
    "max_tokens": 2000,
    "timeout": 15
  },
  "embedder": {
    "batch_size": 200,
    "timeout": 30
  },
  "pipeline": {
    "chunk_size": 400,
    "chunk_overlap": 50,
    "top_k_chunks": 5,
    "max_pages": 5
  }
}
```

### Example 3: Large Repository

Handle 1000+ files:

```json
{
  "pipeline": {
    "chunk_size": 500,
    "chunk_overlap": 100,
    "max_file_size_mb": 2
  },
  "embedder": {
    "batch_size": 150,
    "timeout": 90
  },
  "ingest": {
    "exclude_patterns": [
      "test_*",
      "*.test.*",
      "node_modules/*",
      "dist/*",
      "vendor/*"
    ]
  }
}
```

---

## Validation

### Test Configuration

```bash
# Test environment variables
python -c "import os; print(os.environ['GPT_OSS_API_BASE'])"

# Test API connectivity
curl -H "x-dep-ticket: $GPT_OSS_API_KEY" \
     $GPT_OSS_API_BASE/health

# Test with dry-run
python deepwiki_single.py \
  --repo https://github.com/your-org/small-repo \
  --dry-run \
  --verbose
```

### Debug Mode

Enable maximum logging:

```bash
export DEEPWIKI_VERBOSE=true
export DEEPWIKI_DEBUG=true

python deepwiki_single.py \
  --repo URL \
  --verbose \
  --output ./wiki
```

