---
layout: default
title: Architecture
---

# Architecture

## System Overview

DeepWiki Open is built on a **single-provider, enterprise-focused architecture** that simplifies wiki generation while maintaining powerful AI capabilities. The system is divided into three main layers:

1. **Backend Pipeline (Python)**: Core wiki generation engine
2. **Frontend UI (Next.js/React)**: Web interface for configuration and viewing
3. **Enterprise API Layer**: LLM and embedding services

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     User Browser                            │
│  - Repository URL Input                                     │
│  - Configuration Modal                                      │
│  - Wiki Viewer                                             │
└─────────────────┬───────────────────────────────────────────┘
                  │ HTTP/HTTPS
                  ▼
┌─────────────────────────────────────────────────────────────┐
│              Next.js Frontend (Port 3000)                   │
│  - React 19 Components                                      │
│  - API Routes (/api/wiki, /api/chat)                       │
│  - Static Asset Serving                                     │
└─────────────────┬───────────────────────────────────────────┘
                  │ Internal API Calls
                  ▼
┌─────────────────────────────────────────────────────────────┐
│              Python Backend Pipeline                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Stage 1: INGEST                                    │   │
│  │  - Clone GitHub repository                          │   │
│  │  - Load source files                                │   │
│  │  - Filter by extension                              │   │
│  └─────────────────┬───────────────────────────────────┘   │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Stage 2: CHUNK                                     │   │
│  │  - Split files into 500-word segments               │   │
│  │  - Create overlapping chunks (100 words)            │   │
│  └─────────────────┬───────────────────────────────────┘   │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Stage 3: PLAN                                      │   │
│  │  - Analyze repository structure                     │   │
│  │  - Generate wiki page layout (LLM)                  │   │
│  │  - Define page topics and file patterns             │   │
│  └─────────────────┬───────────────────────────────────┘   │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Stage 4: EMBED                                     │   │
│  │  - Generate BGE-M3 embeddings (1024-dim)            │   │
│  │  - Batch processing (100 texts/batch)               │   │
│  └─────────────────┬───────────────────────────────────┘   │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Stage 5: GENERATE (per page)                       │   │
│  │  - Match files by glob patterns                     │   │
│  │  - Embed page description as query                  │   │
│  │  - Retrieve top-10 relevant chunks (cosine sim)     │   │
│  │  - Build context for LLM                            │   │
│  │  - Generate markdown content                        │   │
│  └─────────────────┬───────────────────────────────────┘   │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Stage 6: WRITE                                     │   │
│  │  - Write markdown files                             │   │
│  │  - Generate _Sidebar.md navigation                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────┬───────────────────────────────────────────┘
                  │ API Calls (httpx)
                  ▼
┌─────────────────────────────────────────────────────────────┐
│              Enterprise API Layer                           │
│                                                             │
│  ┌───────────────────────────────────────────────────┐     │
│  │  LLM Endpoint (gpt-oss-130b)                      │     │
│  │  POST /{model}/v1/chat/completions                │     │
│  │  - Text generation                                 │     │
│  │  - Wiki planning                                   │     │
│  │  - Page content generation                         │     │
│  └───────────────────────────────────────────────────┘     │
│                                                             │
│  ┌───────────────────────────────────────────────────┐     │
│  │  Embedding Endpoint (BGE-M3)                      │     │
│  │  POST /v1/embeddings                               │     │
│  │  - 1024-dimensional vectors                        │     │
│  │  - Batch processing support                        │     │
│  └───────────────────────────────────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Directory Structure

```
deepwiki-open/
├── api/                    # Backend Python code
│   ├── llm/               # LLM client
│   │   └── gpt_oss_client.py
│   ├── embedding/         # Embedding client
│   │   └── bge_m3_client.py
│   ├── pipeline/          # Pipeline stages
│   │   ├── ingest.py     # Stage 1: Repository ingestion
│   │   ├── chunk.py      # Stage 2: Text chunking
│   │   ├── plan.py       # Stage 3: Wiki planning
│   │   └── generate.py   # Stage 5: Page generation
│   ├── wiki/             # Output generation
│   │   └── writer.py     # Stage 6: Markdown writer
│   └── config/           # Configuration files
│
├── src/                   # Frontend Next.js code
│   ├── app/              # Next.js app directory
│   │   ├── page.tsx      # Main UI
│   │   ├── layout.tsx    # Root layout
│   │   └── api/          # API routes
│   ├── components/       # React components
│   ├── contexts/         # React contexts
│   └── types/            # TypeScript types
│
├── deepwiki_single.py     # CLI entry point
├── requirements_single.txt # Python dependencies
├── package.json          # Node dependencies
└── docs/                 # GitHub Pages documentation
```

## Pipeline Stages in Detail

### Stage 1: Ingest (`api/pipeline/ingest.py`)

**Purpose**: Load repository files into memory

**Process**:
1. Clone GitHub repository using `git clone`
2. Walk directory tree
3. Filter files by:
   - Allowed extensions (.py, .js, .ts, .md, etc.)
   - Exclude patterns (.git/, node_modules/, etc.)
4. Load file contents into memory

**Output**: List of `SourceFile` objects with path and content

```python
class SourceFile:
    path: str          # Relative path from repo root
    content: str       # File contents
```

**Key Implementation**: `api/pipeline/ingest.py:130`

### Stage 2: Chunk (`api/pipeline/chunk.py`)

**Purpose**: Split large files into manageable segments for embedding

**Algorithm**:
- **Chunk Size**: 500 words
- **Overlap**: 100 words
- **Method**: Word-based splitting (not character-based)

**Example**:
```
File: 1500 words

Chunk 1: words 0-500
Chunk 2: words 400-900   (100-word overlap with Chunk 1)
Chunk 3: words 800-1300  (100-word overlap with Chunk 2)
Chunk 4: words 1200-1500 (100-word overlap with Chunk 3)
```

**Output**: List of `TextChunk` objects

```python
class TextChunk:
    file_path: str     # Source file path
    content: str       # Chunk text
    start_word: int    # Starting word index
    end_word: int      # Ending word index
```

**Key Implementation**: `api/pipeline/chunk.py:88`

### Stage 3: Plan (`api/pipeline/plan.py`)

**Purpose**: Generate wiki structure using LLM analysis

**Input**: Repository metadata
- Directory structure
- File extensions
- File sizes
- README content (if exists)

**LLM Prompt**:
```
Analyze this repository and create a wiki structure with 3-7 pages.

Repository structure:
{directory tree}

File statistics:
{extension counts, total lines}

Return JSON with pages:
[
  {
    "title": "Home",
    "description": "Overview and getting started",
    "file_patterns": ["README.md", "*.md"]
  },
  ...
]
```

**Output**: Wiki structure JSON

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

**Key Implementation**: `api/pipeline/plan.py:233`

### Stage 4: Embed (`api/embedding/bge_m3_client.py`)

**Purpose**: Convert all chunks to vector embeddings

**Process**:
1. Batch chunks into groups of 100
2. For each batch:
   - Send POST request to BGE-M3 endpoint
   - Receive 1024-dimensional vectors
3. Fallback: If batch fails, retry individual texts
4. Store embeddings in numpy array

**API Call**:
```python
POST /v1/embeddings
{
  "input": ["text1", "text2", ...],
  "model": "BAAI/bge-m3"
}

Response:
{
  "data": [
    {"embedding": [0.1, 0.2, ..., 0.9]},  # 1024 dims
    ...
  ]
}
```

**Output**: Numpy array of shape `(num_chunks, 1024)`

**Key Implementation**: `api/embedding/bge_m3_client.py:195`

### Stage 5: Generate (`api/pipeline/generate.py`)

**Purpose**: Generate each wiki page using RAG

**RAG Process** (per page):

1. **File Matching**:
   - Match source files using glob patterns from plan
   - Example: `src/**/*.py` matches all Python files in src/

2. **Query Embedding**:
   - Embed page description using BGE-M3
   - Example: "Architecture and design patterns" → [0.3, 0.7, ..., 0.4]

3. **Chunk Retrieval**:
   - Compute cosine similarity: `query · chunk / (||query|| * ||chunk||)`
   - Retrieve top-10 most similar chunks
   - De-duplicate by file path

4. **Context Building**:
   ```
   === Matched Files ===
   File: src/main.py (first 2000 chars)
   ...

   === Retrieved Code Chunks ===
   Chunk from src/utils.py (1000 chars)
   ...

   Total context: ~6000 tokens (~24,000 chars)
   ```

5. **LLM Generation**:
   ```
   System: You are a technical documentation writer.

   User: Generate a wiki page titled "Architecture"
   Description: System design and patterns

   Context: {matched files + retrieved chunks}

   Requirements:
   - Use markdown format
   - Include code examples
   - Add diagrams if helpful
   ```

**Output**: Markdown content for each page

**Key Implementation**: `api/pipeline/generate.py:290`

### Stage 6: Write (`api/wiki/writer.py`)

**Purpose**: Write markdown files to disk

**Process**:
1. Create output directory
2. Write each page as `{title}.md`
3. Auto-generate `_Sidebar.md`:
   ```markdown
   ## Navigation
   - [Home](Home)
   - [Architecture](Architecture)
   - [API Reference](API-Reference)
   ```

**Output**: Wiki directory with markdown files

**Key Implementation**: `api/wiki/writer.py:125`

## Component Relationships

```
deepwiki_single.py
    │
    ├─→ GPTOSSClient (api/llm/gpt_oss_client.py)
    │   ├─ chat_completion()
    │   ├─ retry logic (3 attempts, exponential backoff)
    │   └─ httpx.Client
    │
    ├─→ BGEM3Client (api/embedding/bge_m3_client.py)
    │   ├─ embed_texts()
    │   ├─ batch processing (100 texts/batch)
    │   └─ httpx.Client
    │
    ├─→ RepositoryIngester (api/pipeline/ingest.py)
    │   ├─ clone_repository()
    │   └─ load_files()
    │
    ├─→ TextChunker (api/pipeline/chunk.py)
    │   └─ chunk_files()
    │
    ├─→ WikiPlanner (api/pipeline/plan.py)
    │   ├─ analyze_repository()
    │   └─ generate_plan() → GPTOSSClient
    │
    ├─→ PageGenerator (api/pipeline/generate.py)
    │   ├─ generate_page() → GPTOSSClient
    │   ├─ embed_query() → BGEM3Client
    │   ├─ retrieve_chunks() → numpy.dot()
    │   └─ match_files() → fnmatch
    │
    └─→ WikiWriter (api/wiki/writer.py)
        ├─ write_page()
        └─ generate_sidebar()
```

## Data Flow

```
Input: GitHub URL
    │
    ▼
[Clone Repo] → SourceFile[]
    │
    ▼
[Chunk] → TextChunk[]
    │
    ▼
[Plan] → WikiStructure { pages: [...] }
    │
    ▼
[Embed] → ChunkEmbeddings (numpy array)
    │
    ▼
[Generate] → For each page:
    │              │
    │              ├─→ Match files by glob
    │              ├─→ Embed page description
    │              ├─→ Retrieve top-10 chunks (cosine similarity)
    │              ├─→ Build context
    │              └─→ LLM generates markdown
    │
    ▼
[Write] → Wiki directory with .md files
```

## Error Handling Strategy

### LLM Retry Logic

```python
def chat_completion_with_retry(messages):
    max_attempts = 3
    backoff_delays = [2, 4, 8]  # seconds

    for attempt in range(max_attempts):
        try:
            response = httpx.post(
                f"{API_BASE}/{model}/v1/chat/completions",
                json={"messages": messages},
                timeout=30.0
            )
            response.raise_for_status()
            return response.json()
        except (httpx.HTTPError, httpx.TimeoutException) as e:
            if attempt < max_attempts - 1:
                time.sleep(backoff_delays[attempt])
                continue
            raise
```

### Page-Level Isolation

```python
def generate_all_pages(wiki_structure):
    pages = []
    for page_config in wiki_structure["pages"]:
        try:
            content = generate_page(page_config)
            pages.append(content)
        except Exception as e:
            # Log error but continue with other pages
            logger.error(f"Failed to generate {page_config['title']}: {e}")
            pages.append(generate_error_page(page_config))
    return pages
```

### Batch Embedding Fallback

```python
def embed_texts_with_fallback(texts):
    try:
        # Try batch embedding (100 texts)
        return embed_batch(texts)
    except Exception:
        # Fallback: embed one-by-one
        embeddings = []
        for text in texts:
            try:
                embeddings.append(embed_single(text))
            except Exception:
                # Use zero vector as last resort
                embeddings.append(np.zeros(1024))
        return np.array(embeddings)
```

## Performance Characteristics

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| Ingest | O(n) | n = number of files |
| Chunk | O(n × m) | m = average file size |
| Plan | O(1) | Single LLM call |
| Embed | O(c / b) | c = chunks, b = batch size (100) |
| Generate (per page) | O(f + k) | f = file matches, k = chunk retrievals (10) |
| Write | O(p) | p = number of pages |

**Total Pipeline Time**:
- Small repo (100 files): ~2-3 minutes
- Medium repo (1000 files): ~10-15 minutes
- Large repo (5000+ files): ~30-60 minutes

**Bottlenecks**:
1. LLM calls (page generation)
2. Embedding API calls
3. Git clone (for large repos)

**Optimization Strategies**:
- Batch embeddings (100 texts at once)
- Parallel page generation (future improvement)
- Incremental updates (cache embeddings)

## Security Considerations

1. **API Authentication**: Uses `x-dep-ticket` custom header
2. **Input Validation**: Validates repository URLs and paths
3. **Sandboxing**: Git clone to temporary directory
4. **Secrets Management**: Environment variables only (no hardcoded keys)
5. **Rate Limiting**: Respects 429 responses with exponential backoff

## Scalability

**Current Limitations**:
- Single-threaded pipeline
- In-memory chunk storage
- Synchronous API calls

**Future Improvements**:
- Async/await for API calls
- Database for chunk/embedding storage
- Distributed processing for large repos
- Incremental wiki updates

## Technology Stack Summary

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **CLI** | Python 3.11+ | Main orchestration |
| **HTTP Client** | httpx 0.24+ | API calls |
| **Vector Ops** | numpy 1.24+ | Cosine similarity |
| **LLM** | gpt-oss-130b | Text generation |
| **Embeddings** | BGE-M3 (1024-dim) | Semantic search |
| **Frontend** | Next.js 15 | Web UI |
| **UI Library** | React 19 | Components |
| **Styling** | Tailwind CSS | Responsive design |
| **Markdown** | react-markdown | Rendering |
| **Diagrams** | Mermaid | Visual documentation |

[← Back to Home](index.md) | [Next: Usage Guide →](usage.md)
