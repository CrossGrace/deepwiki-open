# DeepWiki-Open: Single-Provider Edition

![DeepWiki Banner](screenshots/Deepwiki.png)

**Simplified GitHub Wiki Generator for Enterprise**

This is a **radically simplified** version of DeepWiki-Open, redesigned as a focused, single-provider GitHub Wiki generator for internal enterprise use. It uses **ONLY**:
- **LLM**: `gpt-oss-130b` (internal OpenAI-compatible API)
- **Embedding**: `BGE-M3` (1024-dimensional, internal API)

> **📌 Note**: This is NOT the original multi-provider DeepWiki. For the full-featured version with web UI, multiple LLM providers, and extensive configurability, see the original [DeepWiki-Open](https://github.com/deepwiki-io/deepwiki-open).

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Directory Structure](#-directory-structure)
- [Core Modules](#-core-modules)
- [Installation](#-installation)
- [Usage](#-usage)
- [API Requirements](#-api-requirements)
- [Environment Variables](#-environment-variables)
- [Code Structure Deep Dive](#-code-structure-deep-dive)
- [Error Handling](#-error-handling)
- [Extending the System](#-extending-the-system)
- [Troubleshooting](#-troubleshooting)
- [Migration Guide](#-migration-guide)

---

## 🎯 Overview

### What This Is

A **single-purpose tool** that transforms GitHub repositories into comprehensive Wiki documentation:

```
GitHub Repository → Analysis → Planning → Generation → GitHub Wiki
```

### Key Characteristics

- ✅ **Single LLM**: gpt-oss-130b only (no provider abstraction)
- ✅ **Single Embedder**: BGE-M3 only (1024 dimensions)
- ✅ **No Configuration Files**: Environment variables only
- ✅ **Linear Pipeline**: Clear, predictable flow
- ✅ **Page-by-Page Generation**: Isolated failures, controlled output
- ✅ **~90% Smaller**: ~1,500 lines vs ~15,000 lines

### What This Is NOT

- ❌ Multi-provider LLM platform
- ❌ General-purpose documentation tool
- ❌ Autonomous agent system
- ❌ Web application with UI
- ❌ Configurable via JSON files

---

## 🏗️ Architecture

### High-Level Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                        1. INGEST PHASE                          │
│  Clone GitHub repo → Load source files → Filter by extension   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        2. CHUNK PHASE                           │
│     Split files into overlapping text segments (500 words)     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        3. PLAN PHASE                            │
│   LLM analyzes repo structure → Generates wiki page layout     │
│   (NO content generation, NO autonomous exploration)           │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        4. EMBED PHASE                           │
│     BGE-M3 computes 1024-dim embeddings for all chunks         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     5. GENERATE PHASE                           │
│  For each page:                                                 │
│    → Retrieve relevant chunks (RAG)                             │
│    → Match files from plan                                      │
│    → Build context (max 6000 tokens)                            │
│    → LLM generates markdown content                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        6. WRITE PHASE                           │
│    Write .md files → Generate _Sidebar.md → Output to disk     │
└─────────────────────────────────────────────────────────────────┘
```

### Key Design Principles

1. **Single Responsibility**: Each module does ONE thing
2. **No Abstractions**: Hardcoded to gpt-oss-130b + BGE-M3
3. **Fail Gracefully**: Page errors don't crash pipeline
4. **Token Budget**: Strict limits prevent overflow
5. **RAG-First**: Context retrieval before generation

---

## 📁 Directory Structure

```
deepwiki-open/
│
├── api/                                    # Core API modules
│   │
│   ├── llm/                                # LLM client (ONLY gpt-oss-130b)
│   │   ├── __init__.py
│   │   └── gpt_oss_client.py              # GPTOSSClient class
│   │
│   ├── embedding/                          # Embedding client (ONLY BGE-M3)
│   │   ├── __init__.py
│   │   └── bge_m3_client.py               # BGEM3Client class
│   │
│   ├── pipeline/                           # Pipeline modules
│   │   ├── __init__.py
│   │   ├── ingest.py                      # RepositoryIngester class
│   │   ├── chunk.py                       # TextChunker class
│   │   ├── plan.py                        # WikiPlanner class
│   │   └── generate.py                    # PageGenerator class
│   │
│   └── wiki/                               # Wiki writer
│       ├── __init__.py
│       └── writer.py                      # WikiWriter class
│
├── deepwiki_single.py                      # Main entry point (CLI)
│
├── requirements_single.txt                 # Minimal dependencies
│
├── README.md                               # This file
├── README_SINGLE_PROVIDER.md               # Detailed usage guide
├── SINGLE_PROVIDER_MIGRATION.md            # Migration summary
└── DELETED_FILES_LIST.md                   # Cleanup instructions

# Legacy files (unused by deepwiki_single.py)
├── api/config.py                          # Multi-provider config (UNUSED)
├── api/openai_client.py                   # Old clients (UNUSED)
├── api/data_pipeline.py                   # Complex pipeline (UNUSED)
├── app.py                                 # FastAPI server (UNUSED)
└── main.py                                # Old entry point (UNUSED)
```

---

## 🔧 Core Modules

### 1. LLM Client (`api/llm/gpt_oss_client.py`)

**Purpose**: Single LLM client for gpt-oss-130b

**Class**: `GPTOSSClient`

**Key Methods**:
```python
def __init__(base_url, token, timeout=30.0, max_retries=3)
def chat(messages: List[Dict], temperature=0.7, max_tokens=None) -> str
def chat_with_system(system: str, user: str) -> str
```

**Features**:
- OpenAI-compatible Chat Completions API
- Custom `x-dep-ticket` authentication header
- Exponential backoff retry (2s, 4s, 8s, 16s)
- Automatic retry on HTTP 429 and 5xx errors
- 30s timeout (configurable)

**HTTP Call Pattern**:
```python
endpoint = f"/gpt-oss-130b/v1/chat/completions"
payload = {
    "model": "openai/gpt-oss-130b",
    "messages": [{"role": "user", "content": "..."}],
    "temperature": 0.7
}
response = httpx_client.post(endpoint, json=payload)
```

---

### 2. Embedding Client (`api/embedding/bge_m3_client.py`)

**Purpose**: Single embedding client for BGE-M3

**Class**: `BGEM3Client`

**Key Methods**:
```python
def __init__(base_url, token, batch_size=100, timeout=60.0)
def embed(texts: List[str]) -> List[List[float]]  # Returns 1024-dim vectors
```

**Features**:
- 1024-dimensional embeddings (BGE-M3 standard)
- Batch processing (default: 100 texts per batch)
- Partial failure recovery (retries individual texts on batch failure)
- Zero vector fallback for failed embeddings
- Custom `x-dep-ticket` authentication

**HTTP Call Pattern**:
```python
endpoint = "/v1/embeddings"
payload = {
    "model": "bge-m3",
    "input": ["text1", "text2", ...]
}
response = httpx_client.post(endpoint, json=payload)
```

---

### 3. Repository Ingester (`api/pipeline/ingest.py`)

**Purpose**: Clone repository and load source files

**Class**: `RepositoryIngester`

**Key Methods**:
```python
def __init__(workspace_dir="./workspace")
def clone_repo(repo_url: str, access_token: str = None) -> str
def load_files(repo_path: str) -> List[Dict[str, str]]
```

**Features**:
- Git shallow clone (depth=1, single branch)
- GitHub access token support for private repos
- Simple file filtering (excludes binaries, build artifacts)
- Returns list of {path, content, size} dictionaries

**Excluded Patterns**:
```python
EXCLUDED_DIRS = [".git", ".venv", "node_modules", "__pycache__", ...]
EXCLUDED_EXTENSIONS = [".pyc", ".so", ".dll", ".zip", ".jpg", ...]
```

---

### 4. Text Chunker (`api/pipeline/chunk.py`)

**Purpose**: Split files into overlapping text segments

**Class**: `TextChunker`

**Key Methods**:
```python
def __init__(chunk_size=500, overlap=100)  # Words, not tokens
def chunk_files(files: List[Dict]) -> List[Dict[str, any]]
```

**Features**:
- Word-based chunking (approximates token splitting)
- Configurable chunk size and overlap
- Preserves file metadata (path, chunk_id, source)

**Output Format**:
```python
[
    {
        'text': 'chunk content...',
        'file': 'src/main.py',
        'chunk_id': 0,
        'source': 'src/main.py#chunk0'
    },
    ...
]
```

---

### 5. Wiki Planner (`api/pipeline/plan.py`)

**Purpose**: Plan wiki page structure (NO content generation)

**Class**: `WikiPlanner`

**Key Methods**:
```python
def __init__(llm_client: GPTOSSClient)
def plan_wiki_structure(files: List[Dict], repo_name: str) -> List[Dict]
```

**CRITICAL CONSTRAINTS**:
- ❌ Does NOT generate documentation content
- ❌ Does NOT summarize code
- ❌ Does NOT explore autonomously
- ✅ ONLY analyzes structure and creates page layout

**Output Format**:
```python
[
    {
        'page': 'Home',
        'files': ['README.md'],
        'description': 'Project overview'
    },
    {
        'page': 'Architecture',
        'files': ['src/**/*.py', 'core/*.py'],
        'description': 'System architecture and design'
    },
    ...
]
```

**LLM Prompt Pattern**:
```
System: You are a documentation planner. Create page structure ONLY.
User: Repository structure: {directories, file types, ...}
      Create a wiki page structure. Return JSON.
```

**Fallback**: If LLM fails, uses rule-based plan (Home, Architecture, API Reference)

---

### 6. Page Generator (`api/pipeline/generate.py`)

**Purpose**: Generate wiki pages using LLM with RAG

**Class**: `PageGenerator`

**Key Methods**:
```python
def __init__(llm_client, embedder_client, max_context_tokens=6000)
def prepare_embeddings(chunks: List[Dict])
def generate_page(page_plan: Dict, files: List[Dict]) -> str
```

**Features**:
- **RAG-based context**: Retrieve relevant chunks using cosine similarity
- **File matching**: Match files based on glob patterns from plan
- **Token budget**: Maximum 6000 tokens context (~24,000 chars)
- **Per-page isolation**: Page errors don't crash pipeline

**Generation Flow**:
```
1. Match files based on page plan patterns
2. Embed page description as query
3. Retrieve top-k relevant chunks (k=10)
4. Build context: matched files + retrieved chunks
5. Truncate context to token budget
6. Call LLM to generate markdown content
7. Return generated page or error page
```

**Context Structure**:
```
=== Relevant Files ===
--- src/app.py ---
<file content truncated to 2000 chars>

=== Retrieved Context ===
[src/core/engine.py#chunk3]
<chunk content truncated to 1000 chars>
```

**Retrieval Algorithm**:
```python
# Cosine similarity
query_embedding = embedder.embed([query])[0]
similarities = chunk_embeddings @ query_embedding
top_k_indices = argsort(similarities)[-10:]
```

---

### 7. Wiki Writer (`api/wiki/writer.py`)

**Purpose**: Write markdown files to disk

**Class**: `WikiWriter`

**Key Methods**:
```python
def __init__(output_dir: str, dry_run: bool = False)
def write_wiki(pages: List[Dict[str, str]])
```

**Features**:
- Individual .md files per page
- Auto-generated `_Sidebar.md` for navigation
- Filename sanitization (spaces → hyphens)
- Dry-run mode (prints instead of writing)

**Output Format**:
```
wiki_output/
├── Home.md
├── Architecture.md
├── API-Reference.md
└── _Sidebar.md
```

**Sidebar Format**:
```markdown
# Wiki Navigation

* [Home](Home)
* [Architecture](Architecture)
* [API Reference](API-Reference)
```

---

### 8. Main Entry Point (`deepwiki_single.py`)

**Purpose**: CLI interface for pipeline execution

**Usage**:
```bash
python deepwiki_single.py \
  --repo https://github.com/org/repo \
  --output ./wiki \
  --token GITHUB_TOKEN \
  --dry-run \
  --debug
```

**Execution Flow**:
```python
[1/7] Initialize clients (LLM + Embedder)
[2/7] Ingest repository (clone + load files)
[3/7] Chunk files (split into segments)
[4/7] Plan wiki structure (LLM decides pages)
[5/7] Compute embeddings (BGE-M3, 1024-dim)
[6/7] Generate pages (page-by-page with RAG)
[7/7] Write wiki files (markdown + sidebar)
```

**Options**:
- `--repo`: GitHub repository URL (required)
- `--output`: Output directory (default: ./wiki_output)
- `--token`: GitHub access token for private repos
- `--workspace`: Workspace for cloning (default: ./workspace)
- `--dry-run`: Test mode, don't write files
- `--debug`: Enable debug logging

---

## 💻 Installation

### Prerequisites

- **Python 3.11+**
- **Git** (for cloning repositories)
- **Access to enterprise APIs**:
  - gpt-oss-130b LLM endpoint
  - BGE-M3 embedding endpoint

### Dependencies

Only 2 core dependencies:

```txt
httpx>=0.24.0    # HTTP client for API calls
numpy>=1.24.0    # Vector operations for retrieval
```

### Setup

```bash
# Clone repository
git clone https://github.com/CrossGrace/deepwiki-open.git
cd deepwiki-open

# Install dependencies
pip install -r requirements_single.txt

# Or install manually
pip install httpx numpy
```

---

## 🚀 Usage

### 1. Set Environment Variables

```bash
# LLM API (gpt-oss-130b)
export DEEPWIKI_LLM_BASE_URL="https://your-llm-api.company.com"
export DEEPWIKI_LLM_TOKEN="your-llm-auth-token"

# Embedding API (BGE-M3)
export DEEPWIKI_EMBEDDING_BASE_URL="https://your-embedding-api.company.com"
export DEEPWIKI_EMBEDDING_TOKEN="your-embedding-auth-token"
```

### 2. Run DeepWiki

**Basic Usage**:
```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./wiki
```

**With GitHub Token** (for private repos):
```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/private-repo \
  --output ./wiki \
  --token YOUR_GITHUB_TOKEN
```

**Dry Run** (test without writing):
```bash
python deepwiki_single.py \
  --repo https://github.com/anthropics/anthropic-sdk-python \
  --dry-run \
  --debug
```

### 3. View Output

```bash
ls -lh wiki_output/
cat wiki_output/Home.md
cat wiki_output/_Sidebar.md
```

---

## 🔌 API Requirements

### LLM API (gpt-oss-130b)

**Endpoint Pattern**:
```
POST /{model}/v1/chat/completions
```

**Full URL Example**:
```
https://your-llm-api.company.com/gpt-oss-130b/v1/chat/completions
```

**Request Headers**:
```json
{
  "Content-Type": "application/json",
  "Accept": "application/json",
  "x-dep-ticket": "your-auth-token"
}
```

**Request Body**:
```json
{
  "model": "openai/gpt-oss-130b",
  "messages": [
    {"role": "system", "content": "You are a technical writer..."},
    {"role": "user", "content": "Write documentation for..."}
  ],
  "temperature": 0.7,
  "max_tokens": 4000
}
```

**Response Format** (OpenAI-compatible):
```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Generated documentation text..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 1000,
    "completion_tokens": 500,
    "total_tokens": 1500
  }
}
```

---

### Embedding API (BGE-M3)

**Endpoint**:
```
POST /v1/embeddings
```

**Full URL Example**:
```
https://your-embedding-api.company.com/v1/embeddings
```

**Request Headers**:
```json
{
  "Content-Type": "application/json",
  "Accept": "application/json",
  "x-dep-ticket": "your-auth-token"
}
```

**Request Body**:
```json
{
  "model": "bge-m3",
  "input": [
    "First text to embed",
    "Second text to embed",
    "..."
  ]
}
```

**Response Format**:
```json
{
  "embeddings": [
    [0.123, 0.456, ..., 0.789],  // 1024-dimensional vector
    [0.234, 0.567, ..., 0.890]
  ],
  "model": "bge-m3"
}
```

**Alternative Format** (OpenAI-compatible):
```json
{
  "data": [
    {
      "embedding": [0.123, 0.456, ..., 0.789],
      "index": 0
    },
    {
      "embedding": [0.234, 0.567, ..., 0.890],
      "index": 1
    }
  ]
}
```

**Critical**: Embeddings MUST be **1024-dimensional** (BGE-M3 standard)

---

## 🔐 Environment Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `DEEPWIKI_LLM_BASE_URL` | Base URL for gpt-oss-130b API | `https://llm-api.company.com` |
| `DEEPWIKI_LLM_TOKEN` | Authentication token for LLM API | `your-secret-token` |
| `DEEPWIKI_EMBEDDING_BASE_URL` | Base URL for BGE-M3 API | `https://embed-api.company.com` |
| `DEEPWIKI_EMBEDDING_TOKEN` | Authentication token for embedding API | `your-secret-token` |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DEEPWIKI_WORKSPACE` | Directory for cloning repos | `./workspace` |
| `DEEPWIKI_OUTPUT` | Default output directory | `./wiki_output` |

### Setting Variables

**Linux/Mac**:
```bash
export DEEPWIKI_LLM_BASE_URL="https://llm-api.company.com"
export DEEPWIKI_LLM_TOKEN="your-token"
```

**Windows (PowerShell)**:
```powershell
$env:DEEPWIKI_LLM_BASE_URL="https://llm-api.company.com"
$env:DEEPWIKI_LLM_TOKEN="your-token"
```

**Using .env file** (requires python-dotenv):
```bash
# Create .env file
cat > .env <<EOF
DEEPWIKI_LLM_BASE_URL=https://llm-api.company.com
DEEPWIKI_LLM_TOKEN=your-token
DEEPWIKI_EMBEDDING_BASE_URL=https://embed-api.company.com
DEEPWIKI_EMBEDDING_TOKEN=your-token
EOF

# Load in Python
from dotenv import load_dotenv
load_dotenv()
```

---

## 🔍 Code Structure Deep Dive

### Data Flow

```python
# 1. Repository Data
{
    'path': 'src/main.py',
    'content': 'def main():\n    ...',
    'size': 1234
}

# 2. Chunks
{
    'text': 'def main():\n    print("Hello")',
    'file': 'src/main.py',
    'chunk_id': 0,
    'source': 'src/main.py#chunk0'
}

# 3. Embeddings
[
    [0.123, 0.456, ..., 0.789],  # 1024-dimensional vector
    [0.234, 0.567, ..., 0.890],
    ...
]

# 4. Page Plans
{
    'page': 'Architecture',
    'files': ['src/**/*.py'],
    'description': 'System architecture'
}

# 5. Generated Pages
{
    'name': 'Architecture',
    'content': '# Architecture\n\n...'
}
```

### Key Algorithms

#### 1. Cosine Similarity Retrieval

```python
def retrieve_chunks(query: str, chunks: List[Dict], embeddings: np.ndarray, top_k: int = 10):
    """Retrieve most relevant chunks using cosine similarity."""
    # Embed query
    query_embedding = embedder.embed([query])[0]
    query_vec = np.array(query_embedding)

    # Normalize vectors
    query_norm = query_vec / (np.linalg.norm(query_vec) + 1e-10)
    chunk_norms = embeddings / (np.linalg.norm(embeddings, axis=1, keepdims=True) + 1e-10)

    # Compute similarities
    similarities = chunk_norms @ query_norm

    # Get top-k indices
    top_indices = np.argsort(similarities)[-top_k:][::-1]

    return [chunks[i] for i in top_indices]
```

#### 2. Token Budget Management

```python
def truncate_context(context: str, max_tokens: int = 6000) -> str:
    """Truncate context to token budget (approximation)."""
    # Rough approximation: 4 chars = 1 token
    max_chars = max_tokens * 4

    if len(context) <= max_chars:
        return context

    return context[:max_chars] + "\n\n... (truncated)"
```

#### 3. Exponential Backoff Retry

```python
def call_with_retry(func, max_retries: int = 3):
    """Call function with exponential backoff."""
    for attempt in range(max_retries):
        try:
            return func()
        except (HTTPStatusError, TimeoutException) as e:
            if attempt == max_retries - 1:
                raise

            wait = 2 ** attempt  # 2, 4, 8 seconds
            time.sleep(wait)

    raise RuntimeError(f"Failed after {max_retries} attempts")
```

### Class Relationships

```
deepwiki_single.py (main)
    │
    ├── GPTOSSClient (llm)
    │   └── httpx.Client
    │
    ├── BGEM3Client (embedding)
    │   └── httpx.Client
    │
    ├── RepositoryIngester (pipeline)
    │   └── subprocess (git)
    │
    ├── TextChunker (pipeline)
    │
    ├── WikiPlanner (pipeline)
    │   └── GPTOSSClient
    │
    ├── PageGenerator (pipeline)
    │   ├── GPTOSSClient
    │   ├── BGEM3Client
    │   └── numpy (for retrieval)
    │
    └── WikiWriter (wiki)
```

### State Management

```python
# Global state in PageGenerator
class PageGenerator:
    def __init__(self, llm, embedder):
        self.llm = llm
        self.embedder = embedder
        self._chunks = None           # Cached chunks
        self._chunk_embeddings = None # Cached embeddings

    def prepare_embeddings(self, chunks):
        """Compute and cache embeddings once."""
        self._chunks = chunks
        texts = [c['text'] for c in chunks]
        self._chunk_embeddings = self.embedder.embed(texts)

    def generate_page(self, page_plan, files):
        """Use cached embeddings for retrieval."""
        relevant = self._retrieve_chunks(page_plan['description'])
        ...
```

---

## ⚠️ Error Handling

### LLM Call Failures

**Scenarios**:
- HTTP 429 (rate limit)
- HTTP 500-504 (server errors)
- Timeout (> 30s)
- Network errors

**Handling**:
```python
# In GPTOSSClient.chat()
for attempt in range(max_retries):
    try:
        response = client.post(endpoint, json=payload)
        response.raise_for_status()
        return response.json()
    except HTTPStatusError as e:
        if e.response.status_code in (429, 500, 502, 503, 504):
            if attempt < max_retries - 1:
                wait = 2 ** attempt
                time.sleep(wait)
                continue
        raise
```

**Impact**: Page generation fails, but pipeline continues

---

### Embedding Failures

**Scenarios**:
- Batch too large (HTTP 413)
- Bad request (HTTP 400)
- Rate limit (HTTP 429)
- Individual text errors

**Handling**:
```python
# In BGEM3Client.embed()
try:
    # Try full batch
    return self._embed_batch(texts)
except HTTPStatusError as e:
    if status in (400, 413) and len(texts) > 1:
        # Retry individual texts
        return self._embed_individual(texts)
    raise
```

**Fallback**: Zero vectors inserted for failed texts

**Impact**: Retrieval may be less accurate, but pipeline continues

---

### Page Generation Failures

**Scenarios**:
- No matched files
- LLM error
- Context too large
- JSON parsing error

**Handling**:
```python
# In PageGenerator.generate_page()
try:
    content = self._generate_content(page_plan, context)
    return content
except Exception as e:
    logger.error(f"Failed to generate {page_name}: {e}")
    return self._generate_error_page(page_plan, str(e))
```

**Impact**: Error page written, other pages unaffected

---

### Wiki Write Failures

**Scenarios**:
- Permission denied
- Disk full
- Invalid filename

**Handling**:
```python
# In WikiWriter.write_wiki()
for page in pages:
    try:
        self._write_page(page['name'], page['content'])
    except IOError as e:
        logger.error(f"Failed to write {page['name']}: {e}")
        continue  # Skip page, continue with others
```

**Impact**: Failed pages skipped, successful pages written

---

## 🛠️ Extending the System

### Adding New Pipeline Stage

**Example**: Add code analysis stage

```python
# api/pipeline/analyze.py
class CodeAnalyzer:
    """Analyze code complexity, dependencies, etc."""

    def analyze_files(self, files: List[Dict]) -> Dict[str, any]:
        analysis = {
            'total_files': len(files),
            'languages': self._detect_languages(files),
            'complexity': self._compute_complexity(files),
        }
        return analysis
```

**Integrate in main**:
```python
# In deepwiki_single.py
from api.pipeline.analyze import CodeAnalyzer

analyzer = CodeAnalyzer()
analysis = analyzer.analyze_files(files)
logger.info(f"Analysis: {analysis}")
```

---

### Customizing Page Templates

**Example**: Add custom markdown header

```python
# In PageGenerator._generate_content()
system_prompt = """You are a technical documentation writer.

Use this template:
# {page_name}

> **Category**: {category}
> **Last Updated**: {date}

## Overview
...
"""
```

---

### Switching Retrieval Strategy

**Example**: Use BM25 instead of cosine similarity

```python
# Install: pip install rank-bm25
from rank_bm25 import BM25Okapi

class PageGenerator:
    def _build_bm25_index(self, chunks):
        """Build BM25 index instead of embeddings."""
        corpus = [c['text'].split() for c in chunks]
        self.bm25 = BM25Okapi(corpus)

    def _retrieve_chunks_bm25(self, query, top_k=10):
        """Retrieve using BM25 instead of embeddings."""
        tokenized_query = query.split()
        scores = self.bm25.get_scores(tokenized_query)
        top_indices = np.argsort(scores)[-top_k:][::-1]
        return [self._chunks[i] for i in top_indices]
```

---

### Adding Caching

**Example**: Cache embeddings to disk

```python
import pickle
from pathlib import Path

class PageGenerator:
    def prepare_embeddings(self, chunks, cache_dir='./cache'):
        cache_file = Path(cache_dir) / 'embeddings.pkl'

        if cache_file.exists():
            logger.info("Loading embeddings from cache")
            with open(cache_file, 'rb') as f:
                cached = pickle.load(f)
                self._chunks = cached['chunks']
                self._chunk_embeddings = cached['embeddings']
                return

        # Compute embeddings
        logger.info("Computing embeddings")
        texts = [c['text'] for c in chunks]
        embeddings = self.embedder.embed(texts)

        # Save to cache
        cache_dir = Path(cache_dir)
        cache_dir.mkdir(exist_ok=True)
        with open(cache_file, 'wb') as f:
            pickle.dump({'chunks': chunks, 'embeddings': embeddings}, f)

        self._chunks = chunks
        self._chunk_embeddings = embeddings
```

---

## 🐛 Troubleshooting

### "DEEPWIKI_LLM_BASE_URL must be set"

**Cause**: Environment variable not set

**Solution**:
```bash
export DEEPWIKI_LLM_BASE_URL="https://your-api.company.com"
export DEEPWIKI_LLM_TOKEN="your-token"
```

---

### "HTTP 401 Unauthorized"

**Cause**: Invalid API token

**Solution**:
1. Check token validity
2. Verify `x-dep-ticket` header is supported by API
3. Test with curl:
```bash
curl -H "x-dep-ticket: your-token" \
     https://your-api.company.com/health
```

---

### "Embedding dimension mismatch"

**Cause**: API returns vectors ≠ 1024 dimensions

**Solution**:
1. Verify BGE-M3 API configuration
2. Check response format:
```python
embeddings = response.json()['embeddings']
print(f"Dimension: {len(embeddings[0])}")  # Should be 1024
```

---

### "No files matched for page"

**Cause**: Glob patterns in plan don't match actual files

**Solution**:
1. Enable debug logging: `--debug`
2. Check matched files in logs
3. Adjust patterns in planner or use fallback plan

---

### "Page generation timeout"

**Cause**: LLM response too slow

**Solution**:
1. Increase timeout:
```python
llm = GPTOSSClient(timeout=60.0)  # 60 seconds instead of 30
```
2. Reduce context size:
```python
generator = PageGenerator(max_context_tokens=4000)  # 4000 instead of 6000
```

---

### "Repository clone failed"

**Cause**: Invalid URL, network error, or authentication

**Solution**:
1. Verify repo URL is correct
2. For private repos, provide GitHub token:
```bash
python deepwiki_single.py --repo URL --token GITHUB_TOKEN
```
3. Check network connectivity:
```bash
git clone https://github.com/org/repo test_clone
```

---

## 🔄 Migration Guide

### From Original DeepWiki-Open

**Old** (multi-provider):
```bash
# Complex configuration
export DEEPWIKI_EMBEDDER_TYPE=enterprise_bge
# Edit api/config/generator.json
# Edit api/config/embedder.json

python main.py --repo-url https://github.com/org/repo
```

**New** (single-provider):
```bash
# Simple environment variables
export DEEPWIKI_LLM_BASE_URL="https://llm-api.company.com"
export DEEPWIKI_LLM_TOKEN="your-token"
export DEEPWIKI_EMBEDDING_BASE_URL="https://embed-api.company.com"
export DEEPWIKI_EMBEDDING_TOKEN="your-token"

python deepwiki_single.py --repo https://github.com/org/repo
```

---

### Key Differences

| Feature | Original | Single-Provider |
|---------|----------|-----------------|
| **Providers** | Multiple (Google, OpenAI, Ollama, etc.) | Single (gpt-oss-130b + BGE-M3) |
| **Configuration** | JSON files | Environment variables only |
| **Web UI** | Yes (FastAPI) | No (CLI only) |
| **Agent Mode** | Autonomous exploration | Constrained planning |
| **Code Size** | ~15,000 lines | ~1,500 lines |
| **Dependencies** | 20+ packages | 2 packages (httpx, numpy) |
| **Streaming** | Yes | No |
| **Multi-language** | Yes | English only |

---

## 📚 Additional Resources

### Documentation Files

- **`README_SINGLE_PROVIDER.md`**: Comprehensive user guide
- **`SINGLE_PROVIDER_MIGRATION.md`**: Detailed migration summary
- **`DELETED_FILES_LIST.md`**: Cleanup instructions for unused files

### Example Repositories

Test with these public repos:
```bash
# Small repo (~100 files)
python deepwiki_single.py \
  --repo https://github.com/anthropics/anthropic-sdk-python

# Medium repo (~500 files)
python deepwiki_single.py \
  --repo https://github.com/fastapi/fastapi

# Large repo (~1000+ files)
python deepwiki_single.py \
  --repo https://github.com/django/django
```

---

## 🙏 Credits

Based on [DeepWiki-Open](https://github.com/deepwiki-io/deepwiki-open)

Simplified for single-provider enterprise use.

---

## 📄 License

MIT License (inherited from DeepWiki-Open)

---

## 📞 Support

For issues or questions:
1. Read this README thoroughly
2. Check `README_SINGLE_PROVIDER.md` for usage details
3. Enable `--debug` mode for detailed logs
4. Review error messages and troubleshooting section
5. Contact repository maintainers for API-specific issues

---

**Last Updated**: January 2026

**Version**: 2.0 (Single-Provider Edition)
