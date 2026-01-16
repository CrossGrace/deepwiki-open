---
layout: default
title: Usage Guide
description: Detailed instructions for CLI and web interface usage
prev_page:
  title: Architecture
  url: /architecture.html
next_page:
  title: API Reference
  url: /api-reference.html
---

# Usage Guide

This guide covers how to use DeepWiki Open to generate wiki documentation for your repositories.

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [CLI Usage](#cli-usage)
- [Web UI Usage](#web-ui-usage)
- [Advanced Options](#advanced-options)
- [Troubleshooting](#troubleshooting)

## Installation

### Prerequisites

Before installing DeepWiki Open, ensure you have:

- **Python 3.11 or higher**
- **Node.js 18 or higher**
- **Git**
- **Access to Enterprise API endpoints** (gpt-oss-130b + BGE-M3)

### Step 1: Clone the Repository

```bash
git clone https://github.com/CrossGrace/deepwiki-open.git
cd deepwiki-open
```

### Step 2: Install Python Dependencies

```bash
# Using pip
pip install -r requirements_single.txt

# Or using Poetry
cd api
poetry install
```

**Minimal dependencies**:
- `httpx>=0.24.0` - HTTP client for API calls
- `numpy>=1.24.0` - Vector operations for RAG

### Step 3: Install Node.js Dependencies

```bash
# Using Yarn (recommended)
yarn install

# Or using npm
npm install
```

### Step 4: Verify Installation

```bash
# Test Python CLI
python deepwiki_single.py --help

# Test web server
yarn dev
```

## Configuration

### Environment Variables

DeepWiki Open requires two environment variables:

```bash
# Set API base URL
export GPT_OSS_API_BASE="https://your-company-api.com"

# Set API key
export GPT_OSS_API_KEY="your-api-key-here"
```

**Optional**: Create a `.env` file in the project root:

```bash
# .env
GPT_OSS_API_BASE=https://your-company-api.com
GPT_OSS_API_KEY=your-api-key-here
```

### Configuration Files

DeepWiki Open can use JSON configuration files for advanced settings:

**Generator Config** (`config_examples/enterprise_generator.json`):

```json
{
  "model": "gpt-oss-130b",
  "max_tokens": 4000,
  "temperature": 0.7,
  "api_base": "https://your-company-api.com",
  "api_key": "${GPT_OSS_API_KEY}",
  "timeout": 30,
  "max_retries": 3
}
```

**Embedder Config** (`config_examples/enterprise_embedder.json`):

```json
{
  "model": "BAAI/bge-m3",
  "dimensions": 1024,
  "api_base": "https://your-company-api.com",
  "api_key": "${GPT_OSS_API_KEY}",
  "batch_size": 100,
  "timeout": 60
}
```

## CLI Usage

### Basic Usage

Generate a wiki for a public repository:

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./output/wiki
```

### With Configuration Files

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./output/wiki \
  --generator-config config_examples/enterprise_generator.json \
  --embedder-config config_examples/enterprise_embedder.json
```

### Dry Run Mode

Test the pipeline without writing files:

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./output/wiki \
  --dry-run
```

**Dry run behavior**:
- ✅ Clones repository
- ✅ Chunks files
- ✅ Generates plan
- ✅ Creates embeddings
- ✅ Generates pages
- ❌ Does NOT write markdown files

### Custom Branch

Generate wiki from a specific branch:

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --branch develop \
  --output ./output/wiki
```

### File Filtering

**Include specific extensions**:

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --include-exts .py,.js,.ts,.md \
  --output ./output/wiki
```

**Exclude patterns**:

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --exclude-patterns "test_*,*_test.py,node_modules/*" \
  --output ./output/wiki
```

### CLI Options Reference

| Option | Description | Default |
|--------|-------------|---------|
| `--repo URL` | GitHub repository URL | *Required* |
| `--output PATH` | Output directory for wiki | `./output` |
| `--branch NAME` | Git branch to use | `main` |
| `--generator-config PATH` | LLM config JSON | Uses env vars |
| `--embedder-config PATH` | Embedding config JSON | Uses env vars |
| `--dry-run` | Skip writing files | `false` |
| `--include-exts EXTS` | File extensions to include | `.py,.js,.ts,.md,...` |
| `--exclude-patterns PATTERNS` | Glob patterns to exclude | `test_*,*.test.js,...` |
| `--chunk-size WORDS` | Chunk size in words | `500` |
| `--chunk-overlap WORDS` | Overlap between chunks | `100` |
| `--max-pages NUM` | Maximum wiki pages | `7` |
| `--verbose` | Enable verbose logging | `false` |

## Web UI Usage

### Starting the Development Server

```bash
yarn dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

### Generating a Wiki via UI

1. **Enter Repository URL**
   - Navigate to the home page
   - Enter a GitHub repository URL
   - Example: `https://github.com/facebook/react`

2. **Configure Settings** (Optional)
   - Click "Configure" button
   - Set API endpoints
   - Adjust chunking parameters
   - Set file filters

3. **Generate Wiki**
   - Click "Generate Wiki" button
   - Monitor progress in real-time
   - View logs in console

4. **View Results**
   - Wiki pages appear in sidebar
   - Click to navigate between pages
   - Markdown rendered with syntax highlighting
   - Mermaid diagrams rendered automatically

### UI Features

- **Dark/Light Mode**: Toggle theme in header
- **Multi-Language**: Select language (14+ supported)
- **Live Preview**: See wiki as it generates
- **Mermaid Diagrams**: Automatically rendered
- **Code Highlighting**: Syntax highlighting for all languages
- **Responsive Design**: Works on mobile and desktop

## Advanced Options

### Custom Chunking Strategy

Modify chunking parameters for different repo sizes:

**Small repos** (< 100 files):
```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/small-repo \
  --chunk-size 300 \
  --chunk-overlap 50 \
  --output ./output/wiki
```

**Large repos** (> 1000 files):
```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/large-repo \
  --chunk-size 700 \
  --chunk-overlap 150 \
  --output ./output/wiki
```

### Custom Wiki Structure

Override the default planning with a manual structure:

**Create `custom_plan.json`**:

```json
{
  "pages": [
    {
      "title": "Home",
      "description": "Project overview and quick start guide",
      "file_patterns": ["README.md", "docs/*.md"]
    },
    {
      "title": "Backend API",
      "description": "Python backend architecture and API endpoints",
      "file_patterns": ["api/**/*.py"]
    },
    {
      "title": "Frontend Components",
      "description": "React component library and usage",
      "file_patterns": ["src/components/**/*.tsx"]
    },
    {
      "title": "Configuration",
      "description": "Environment setup and configuration options",
      "file_patterns": ["*.json", "*.yml", "*.env*"]
    }
  ]
}
```

**Use custom plan**:

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --plan custom_plan.json \
  --output ./output/wiki
```

### Incremental Updates

Update an existing wiki without regenerating everything:

```bash
# Initial generation
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./output/wiki \
  --cache-embeddings

# Later update (reuses cached embeddings)
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./output/wiki \
  --use-cache \
  --update-only "Home,API Reference"
```

### Batch Processing

Generate wikis for multiple repositories:

**Create `repos.txt`**:
```
https://github.com/org/repo1
https://github.com/org/repo2
https://github.com/org/repo3
```

**Run batch script**:

```bash
#!/bin/bash
while IFS= read -r repo; do
  echo "Processing $repo..."
  python deepwiki_single.py \
    --repo "$repo" \
    --output "./output/$(basename $repo)" \
    --verbose
done < repos.txt
```

## Pipeline Stages Explained

### Stage 1: Ingest

**What happens**:
- Clones repository to temporary directory
- Scans for source files
- Filters by extension and exclude patterns
- Loads file contents into memory

**Output**: List of source files

**Logs**:
```
[INFO] Cloning repository: https://github.com/your-org/your-repo
[INFO] Loaded 245 source files
[INFO] Filtered to 180 files after exclusions
```

### Stage 2: Chunk

**What happens**:
- Splits each file into 500-word chunks
- Creates 100-word overlap between chunks
- Preserves file path metadata

**Output**: List of text chunks

**Logs**:
```
[INFO] Chunking 180 files...
[INFO] Created 1,234 chunks (avg 6.9 chunks/file)
```

### Stage 3: Plan

**What happens**:
- Analyzes repository structure
- Sends structure to LLM
- LLM generates wiki page plan (3-7 pages)
- Each page gets title, description, file patterns

**Output**: Wiki structure JSON

**Logs**:
```
[INFO] Analyzing repository structure...
[INFO] Repository has 15 directories, 180 files
[INFO] Extensions: .py (120), .js (30), .md (10), .json (20)
[INFO] Generated plan with 5 pages
```

**Example plan**:
```json
{
  "pages": [
    {
      "title": "Home",
      "description": "Project overview",
      "file_patterns": ["README.md"]
    },
    {
      "title": "Architecture",
      "description": "System design",
      "file_patterns": ["src/**/*.py"]
    }
  ]
}
```

### Stage 4: Embed

**What happens**:
- Sends chunks to BGE-M3 embedding API
- Batch size: 100 texts per request
- Each chunk → 1024-dimensional vector

**Output**: Numpy array of embeddings

**Logs**:
```
[INFO] Embedding 1,234 chunks...
[INFO] Batch 1/13: 100 chunks embedded
[INFO] Batch 2/13: 100 chunks embedded
...
[INFO] Embedding complete: (1234, 1024) array
```

### Stage 5: Generate

**What happens** (per page):
1. Match source files by glob patterns
2. Embed page description as query vector
3. Compute cosine similarity: query vs all chunks
4. Retrieve top-10 most relevant chunks
5. Build context (matched files + chunks)
6. Send context to LLM for generation
7. Receive markdown content

**Output**: Markdown content for each page

**Logs**:
```
[INFO] Generating page: Home
[INFO] Matched 3 files by patterns: ['README.md']
[INFO] Retrieved 10 relevant chunks
[INFO] Context size: 5,240 tokens
[INFO] LLM generation complete (1,823 tokens)

[INFO] Generating page: Architecture
[INFO] Matched 45 files by patterns: ['src/**/*.py']
[INFO] Retrieved 10 relevant chunks
[INFO] Context size: 6,000 tokens
[INFO] LLM generation complete (3,456 tokens)
```

### Stage 6: Write

**What happens**:
- Creates output directory
- Writes each page as `{Title}.md`
- Auto-generates `_Sidebar.md` navigation

**Output**: Wiki directory with markdown files

**Logs**:
```
[INFO] Writing 5 pages to ./output/wiki
[INFO] Wrote: Home.md (1,823 words)
[INFO] Wrote: Architecture.md (3,456 words)
[INFO] Wrote: API-Reference.md (2,901 words)
[INFO] Generated _Sidebar.md with 5 entries
[INFO] Wiki generation complete!
```

## Troubleshooting

### Common Issues

#### 1. "API key not set"

**Error**:
```
Error: GPT_OSS_API_KEY environment variable not set
```

**Solution**:
```bash
export GPT_OSS_API_KEY="your-api-key"
```

#### 2. "Failed to clone repository"

**Error**:
```
Error: Failed to clone https://github.com/your-org/private-repo
```

**Solution**: Use SSH URL with authentication:
```bash
# Set up SSH key first
python deepwiki_single.py \
  --repo git@github.com:your-org/private-repo.git \
  --output ./output/wiki
```

#### 3. "HTTP 429: Rate limit exceeded"

**Error**:
```
Error: HTTP 429: Too many requests
```

**Solution**: The CLI automatically retries with exponential backoff. If persistent:
```bash
# Reduce batch size
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --batch-size 50 \
  --output ./output/wiki
```

#### 4. "Out of memory"

**Error**:
```
MemoryError: Unable to allocate array
```

**Solution**: Process in chunks or increase system memory:
```bash
# Reduce chunk size
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --chunk-size 300 \
  --max-pages 5 \
  --output ./output/wiki
```

#### 5. "LLM generation failed"

**Error**:
```
Error: Failed to generate page 'Architecture' after 3 attempts
```

**Solution**: Check logs and retry with verbose mode:
```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --verbose \
  --output ./output/wiki
```

### Debugging Tips

1. **Enable verbose logging**:
   ```bash
   python deepwiki_single.py --verbose --repo URL --output ./wiki
   ```

2. **Use dry-run mode**:
   ```bash
   python deepwiki_single.py --dry-run --repo URL --output ./wiki
   ```

3. **Check API connectivity**:
   ```bash
   curl -H "x-dep-ticket: $GPT_OSS_API_KEY" \
        $GPT_OSS_API_BASE/health
   ```

4. **Inspect generated plan**:
   ```bash
   python deepwiki_single.py --repo URL --output ./wiki --verbose
   # Check logs for plan JSON
   ```

5. **Test embeddings**:
   ```python
   from api.embedding.bge_m3_client import BGEM3Client
   client = BGEM3Client(api_base=API_BASE, api_key=API_KEY)
   embeddings = client.embed_texts(["test text"])
   print(embeddings.shape)  # Should be (1, 1024)
   ```

### Performance Optimization

**For large repositories** (1000+ files):

1. **Filter aggressively**:
   ```bash
   --exclude-patterns "test_*,*.test.*,node_modules/*,dist/*"
   ```

2. **Limit pages**:
   ```bash
   --max-pages 5
   ```

3. **Increase batch size** (if API supports):
   ```bash
   --batch-size 200
   ```

4. **Use faster model** (if available):
   ```json
   {
     "model": "gpt-oss-70b",  // Faster than 130b
     "max_tokens": 2000
   }
   ```

### Getting Help

If you encounter issues:

1. Check [GitHub Issues](https://github.com/CrossGrace/deepwiki-open/issues)
2. Review [Enterprise Integration Guide](../ENTERPRISE_INTEGRATION.md)
3. Enable `--verbose` mode for detailed logs
4. Check API endpoint health

