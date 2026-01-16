---
layout: default
title: API Reference
description: Complete Python API and Enterprise endpoint documentation
prev_page:
  title: Usage Guide
  url: /usage.html
next_page:
  title: Configuration
  url: /configuration.html
---

# API Reference

Complete reference for DeepWiki Open's Python API and enterprise endpoints.

## Table of Contents

- [Python API](#python-api)
- [Enterprise API Endpoints](#enterprise-api-endpoints)
- [Data Models](#data-models)
- [Client Classes](#client-classes)
- [Pipeline Modules](#pipeline-modules)

## Python API

### Main Entry Point

#### `deepwiki_single.py`

Main CLI script that orchestrates the wiki generation pipeline.

```python
def main(
    repo_url: str,
    output_dir: str,
    branch: str = "main",
    generator_config: Optional[str] = None,
    embedder_config: Optional[str] = None,
    dry_run: bool = False,
    **kwargs
) -> None:
    """
    Generate wiki documentation for a GitHub repository.

    Args:
        repo_url: GitHub repository URL
        output_dir: Directory to write wiki files
        branch: Git branch to clone (default: "main")
        generator_config: Path to LLM config JSON
        embedder_config: Path to embedding config JSON
        dry_run: If True, skip writing files

    Raises:
        ValueError: Invalid repository URL or configuration
        RuntimeError: Pipeline stage failure
    """
```

**Example**:

```python
from deepwiki_single import main

main(
    repo_url="https://github.com/your-org/your-repo",
    output_dir="./output/wiki",
    branch="main",
    dry_run=False
)
```

## Client Classes

### GPTOSSClient

LLM client for gpt-oss-130b text generation.

**Location**: `api/llm/gpt_oss_client.py`

#### Constructor

```python
class GPTOSSClient:
    def __init__(
        self,
        api_base: str,
        api_key: str,
        model: str = "gpt-oss-130b",
        timeout: float = 30.0,
        max_retries: int = 3
    ):
        """
        Initialize GPT-OSS client.

        Args:
            api_base: Base URL for API (e.g., "https://api.company.com")
            api_key: API key for authentication (x-dep-ticket header)
            model: Model identifier (default: "gpt-oss-130b")
            timeout: Request timeout in seconds (default: 30.0)
            max_retries: Maximum retry attempts (default: 3)
        """
```

#### Methods

##### `chat_completion()`

Generate text completion from messages.

```python
def chat_completion(
    self,
    messages: List[Dict[str, str]],
    temperature: float = 0.7,
    max_tokens: int = 4000,
    **kwargs
) -> str:
    """
    Generate chat completion.

    Args:
        messages: List of message dicts with 'role' and 'content'
        temperature: Sampling temperature (0.0-2.0, default: 0.7)
        max_tokens: Maximum tokens to generate (default: 4000)

    Returns:
        Generated text content

    Raises:
        httpx.HTTPError: API request failed after retries
        ValueError: Invalid message format
    """
```

**Example**:

```python
from api.llm.gpt_oss_client import GPTOSSClient

client = GPTOSSClient(
    api_base="https://api.company.com",
    api_key="your-api-key"
)

messages = [
    {"role": "system", "content": "You are a technical writer."},
    {"role": "user", "content": "Write a README for a Python project."}
]

response = client.chat_completion(messages, temperature=0.7)
print(response)
```

##### `chat_completion_with_retry()`

Internal method with exponential backoff retry logic.

```python
def _chat_completion_with_retry(
    self,
    messages: List[Dict[str, str]],
    **kwargs
) -> Dict[str, Any]:
    """
    Chat completion with retry logic.

    Retry strategy:
    - Attempt 1: Immediate
    - Attempt 2: After 2 seconds
    - Attempt 3: After 4 seconds
    - Attempt 4: After 8 seconds (if max_retries > 3)

    Retries on:
    - HTTP 429 (rate limit)
    - HTTP 5xx (server errors)
    - Timeout exceptions
    - Network errors

    Returns:
        Full API response JSON

    Raises:
        httpx.HTTPError: All retries exhausted
    """
```

---

### BGEM3Client

Embedding client for BGE-M3 (1024-dimensional vectors).

**Location**: `api/embedding/bge_m3_client.py`

#### Constructor

```python
class BGEM3Client:
    def __init__(
        self,
        api_base: str,
        api_key: str,
        model: str = "BAAI/bge-m3",
        dimensions: int = 1024,
        batch_size: int = 100,
        timeout: float = 60.0
    ):
        """
        Initialize BGE-M3 embedding client.

        Args:
            api_base: Base URL for API
            api_key: API key for authentication
            model: Model identifier (default: "BAAI/bge-m3")
            dimensions: Embedding dimensions (default: 1024)
            batch_size: Texts per batch request (default: 100)
            timeout: Request timeout in seconds (default: 60.0)
        """
```

#### Methods

##### `embed_texts()`

Generate embeddings for multiple texts.

```python
def embed_texts(
    self,
    texts: List[str],
    batch_size: Optional[int] = None
) -> np.ndarray:
    """
    Generate embeddings for texts with batch processing.

    Args:
        texts: List of text strings to embed
        batch_size: Override default batch size (optional)

    Returns:
        Numpy array of shape (len(texts), 1024)

    Raises:
        httpx.HTTPError: API request failed
        ValueError: Empty text list or invalid batch size

    Notes:
        - Automatically batches texts (default: 100 per batch)
        - Falls back to individual embedding on batch failure
        - Returns zero vectors for failed individual texts
    """
```

**Example**:

```python
from api.embedding.bge_m3_client import BGEM3Client

client = BGEM3Client(
    api_base="https://api.company.com",
    api_key="your-api-key",
    batch_size=100
)

texts = [
    "First document to embed",
    "Second document to embed",
    "Third document to embed"
]

embeddings = client.embed_texts(texts)
print(embeddings.shape)  # (3, 1024)
```

##### `embed_query()`

Convenience method for single query embedding.

```python
def embed_query(self, query: str) -> np.ndarray:
    """
    Embed a single query text.

    Args:
        query: Query string

    Returns:
        Numpy array of shape (1024,)
    """
```

---

## Pipeline Modules

### RepositoryIngester

**Location**: `api/pipeline/ingest.py`

Clone and load repository files.

```python
class RepositoryIngester:
    def __init__(
        self,
        include_extensions: Optional[List[str]] = None,
        exclude_patterns: Optional[List[str]] = None
    ):
        """
        Initialize repository ingester.

        Args:
            include_extensions: File extensions to include
                (default: ['.py', '.js', '.ts', '.tsx', '.md', ...])
            exclude_patterns: Glob patterns to exclude
                (default: ['test_*', '*.test.*', 'node_modules/*', ...])
        """

    def ingest(
        self,
        repo_url: str,
        branch: str = "main",
        clone_dir: Optional[str] = None
    ) -> List[SourceFile]:
        """
        Clone repository and load source files.

        Args:
            repo_url: GitHub repository URL
            branch: Git branch (default: "main")
            clone_dir: Directory to clone into (default: temp dir)

        Returns:
            List of SourceFile objects

        Raises:
            subprocess.CalledProcessError: Git clone failed
            OSError: File system error
        """
```

---

### TextChunker

**Location**: `api/pipeline/chunk.py`

Split files into overlapping chunks.

```python
class TextChunker:
    def __init__(
        self,
        chunk_size: int = 500,
        chunk_overlap: int = 100
    ):
        """
        Initialize text chunker.

        Args:
            chunk_size: Chunk size in words (default: 500)
            chunk_overlap: Overlap in words (default: 100)
        """

    def chunk_files(
        self,
        files: List[SourceFile]
    ) -> List[TextChunk]:
        """
        Chunk all files into overlapping segments.

        Args:
            files: List of SourceFile objects

        Returns:
            List of TextChunk objects

        Algorithm:
            1. Split file by whitespace into words
            2. Create chunks of chunk_size words
            3. Overlap by chunk_overlap words
            4. Preserve file path metadata
        """
```

---

### WikiPlanner

**Location**: `api/pipeline/plan.py`

Generate wiki structure using LLM.

```python
class WikiPlanner:
    def __init__(self, llm_client: GPTOSSClient):
        """
        Initialize wiki planner.

        Args:
            llm_client: GPT-OSS client for planning
        """

    def plan(
        self,
        files: List[SourceFile],
        max_pages: int = 7
    ) -> Dict[str, Any]:
        """
        Generate wiki page structure.

        Args:
            files: List of source files
            max_pages: Maximum pages to generate (default: 7)

        Returns:
            Wiki structure dict:
            {
                "pages": [
                    {
                        "title": str,
                        "description": str,
                        "file_patterns": List[str]
                    },
                    ...
                ]
            }

        Process:
            1. Analyze repository structure
            2. Build prompt with file statistics
            3. Send to LLM for planning
            4. Parse JSON response
            5. Validate structure
        """
```

---

### PageGenerator

**Location**: `api/pipeline/generate.py`

Generate wiki pages using RAG.

```python
class PageGenerator:
    def __init__(
        self,
        llm_client: GPTOSSClient,
        embedding_client: BGEM3Client,
        chunk_embeddings: np.ndarray,
        chunks: List[TextChunk],
        files: List[SourceFile]
    ):
        """
        Initialize page generator.

        Args:
            llm_client: GPT-OSS client
            embedding_client: BGE-M3 client
            chunk_embeddings: Precomputed embeddings (shape: [num_chunks, 1024])
            chunks: List of text chunks
            files: List of source files
        """

    def generate_page(
        self,
        page_config: Dict[str, Any],
        top_k: int = 10,
        max_context_tokens: int = 6000
    ) -> str:
        """
        Generate single wiki page with RAG.

        Args:
            page_config: Page configuration from wiki plan
                {
                    "title": str,
                    "description": str,
                    "file_patterns": List[str]
                }
            top_k: Number of chunks to retrieve (default: 10)
            max_context_tokens: Max context tokens (default: 6000)

        Returns:
            Generated markdown content

        Process:
            1. Match files by glob patterns
            2. Embed page description as query
            3. Compute cosine similarity with all chunks
            4. Retrieve top-k chunks
            5. Build context (files + chunks)
            6. Generate page with LLM
        """
```

#### Helper Methods

##### `match_files()`

```python
def match_files(
    self,
    patterns: List[str]
) -> List[SourceFile]:
    """
    Match files by glob patterns.

    Args:
        patterns: List of glob patterns (e.g., ["src/**/*.py"])

    Returns:
        List of matched SourceFile objects
    """
```

##### `retrieve_chunks()`

```python
def retrieve_chunks(
    self,
    query_embedding: np.ndarray,
    top_k: int = 10
) -> List[Tuple[TextChunk, float]]:
    """
    Retrieve most similar chunks via cosine similarity.

    Args:
        query_embedding: Query vector (shape: [1024])
        top_k: Number of chunks to return

    Returns:
        List of (chunk, similarity_score) tuples
        Sorted by similarity (highest first)

    Algorithm:
        similarity = query · chunk / (||query|| * ||chunk||)
    """
```

---

### WikiWriter

**Location**: `api/wiki/writer.py`

Write markdown files to disk.

```python
class WikiWriter:
    def __init__(self, output_dir: str):
        """
        Initialize wiki writer.

        Args:
            output_dir: Directory to write files
        """

    def write_page(
        self,
        title: str,
        content: str
    ) -> None:
        """
        Write single wiki page.

        Args:
            title: Page title (used as filename)
            content: Markdown content

        Output:
            Creates {output_dir}/{title}.md
        """

    def generate_sidebar(
        self,
        page_titles: List[str]
    ) -> None:
        """
        Generate _Sidebar.md navigation.

        Args:
            page_titles: List of page titles

        Output:
            Creates {output_dir}/_Sidebar.md with:
            ## Navigation
            - [Home](Home)
            - [Architecture](Architecture)
            ...
        """
```

---

## Data Models

### SourceFile

```python
@dataclass
class SourceFile:
    path: str          # Relative path from repo root
    content: str       # File contents
    size: int          # File size in bytes
    extension: str     # File extension (e.g., ".py")
```

### TextChunk

```python
@dataclass
class TextChunk:
    file_path: str     # Source file path
    content: str       # Chunk text
    start_word: int    # Starting word index in file
    end_word: int      # Ending word index in file
    embedding: Optional[np.ndarray] = None  # 1024-dim vector
```

### WikiPage

```python
@dataclass
class WikiPage:
    title: str              # Page title
    description: str        # Page description
    file_patterns: List[str]  # Glob patterns for file matching
    content: Optional[str] = None  # Generated markdown
```

---

## Enterprise API Endpoints

### LLM Endpoint (gpt-oss-130b)

**Endpoint**: `POST /{model}/v1/chat/completions`

**Full URL**: `{API_BASE}/gpt-oss-130b/v1/chat/completions`

#### Request

```json
{
  "model": "gpt-oss-130b",
  "messages": [
    {
      "role": "system",
      "content": "You are a technical documentation writer."
    },
    {
      "role": "user",
      "content": "Generate a wiki page about system architecture."
    }
  ],
  "temperature": 0.7,
  "max_tokens": 4000,
  "top_p": 1.0,
  "frequency_penalty": 0.0,
  "presence_penalty": 0.0
}
```

#### Headers

```
Content-Type: application/json
x-dep-ticket: {API_KEY}
```

#### Response

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "gpt-oss-130b",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "# Architecture\n\nThis system follows..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 1234,
    "completion_tokens": 567,
    "total_tokens": 1801
  }
}
```

#### Error Responses

**429 Rate Limit**:
```json
{
  "error": {
    "message": "Rate limit exceeded",
    "type": "rate_limit_error",
    "code": 429
  }
}
```

**500 Server Error**:
```json
{
  "error": {
    "message": "Internal server error",
    "type": "server_error",
    "code": 500
  }
}
```

---

### Embedding Endpoint (BGE-M3)

**Endpoint**: `POST /v1/embeddings`

**Full URL**: `{API_BASE}/v1/embeddings`

#### Request

```json
{
  "input": [
    "First text to embed",
    "Second text to embed",
    "Third text to embed"
  ],
  "model": "BAAI/bge-m3",
  "encoding_format": "float"
}
```

#### Headers

```
Content-Type: application/json
x-dep-ticket: {API_KEY}
```

#### Response

```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "index": 0,
      "embedding": [0.123, 0.456, ..., 0.789]  // 1024 floats
    },
    {
      "object": "embedding",
      "index": 1,
      "embedding": [0.234, 0.567, ..., 0.890]
    },
    {
      "object": "embedding",
      "index": 2,
      "embedding": [0.345, 0.678, ..., 0.901]
    }
  ],
  "model": "BAAI/bge-m3",
  "usage": {
    "prompt_tokens": 150,
    "total_tokens": 150
  }
}
```

#### Rate Limits

| Endpoint | Rate Limit | Batch Size |
|----------|-----------|------------|
| Chat Completions | 60 req/min | 1 request |
| Embeddings | 100 req/min | 100 texts/request |

---

## Usage Examples

### Complete Pipeline Example

```python
import os
from api.llm.gpt_oss_client import GPTOSSClient
from api.embedding.bge_m3_client import BGEM3Client
from api.pipeline.ingest import RepositoryIngester
from api.pipeline.chunk import TextChunker
from api.pipeline.plan import WikiPlanner
from api.pipeline.generate import PageGenerator
from api.wiki.writer import WikiWriter

# Initialize clients
llm_client = GPTOSSClient(
    api_base=os.environ["GPT_OSS_API_BASE"],
    api_key=os.environ["GPT_OSS_API_KEY"]
)

embedding_client = BGEM3Client(
    api_base=os.environ["GPT_OSS_API_BASE"],
    api_key=os.environ["GPT_OSS_API_KEY"]
)

# Stage 1: Ingest
ingester = RepositoryIngester()
files = ingester.ingest("https://github.com/your-org/your-repo")
print(f"Loaded {len(files)} files")

# Stage 2: Chunk
chunker = TextChunker(chunk_size=500, chunk_overlap=100)
chunks = chunker.chunk_files(files)
print(f"Created {len(chunks)} chunks")

# Stage 3: Plan
planner = WikiPlanner(llm_client)
wiki_structure = planner.plan(files, max_pages=7)
print(f"Generated plan with {len(wiki_structure['pages'])} pages")

# Stage 4: Embed
chunk_texts = [chunk.content for chunk in chunks]
embeddings = embedding_client.embed_texts(chunk_texts)
print(f"Embeddings shape: {embeddings.shape}")

# Stage 5: Generate
generator = PageGenerator(
    llm_client=llm_client,
    embedding_client=embedding_client,
    chunk_embeddings=embeddings,
    chunks=chunks,
    files=files
)

pages = []
for page_config in wiki_structure["pages"]:
    content = generator.generate_page(page_config)
    pages.append({"title": page_config["title"], "content": content})
    print(f"Generated: {page_config['title']}")

# Stage 6: Write
writer = WikiWriter(output_dir="./output/wiki")
for page in pages:
    writer.write_page(page["title"], page["content"])
writer.generate_sidebar([p["title"] for p in pages])
print("Wiki generation complete!")
```

### RAG Retrieval Example

```python
import numpy as np
from api.embedding.bge_m3_client import BGEM3Client

# Initialize client
client = BGEM3Client(
    api_base="https://api.company.com",
    api_key="your-api-key"
)

# Embed documents
documents = [
    "Python is a high-level programming language.",
    "JavaScript is used for web development.",
    "Go is a compiled language designed at Google."
]
doc_embeddings = client.embed_texts(documents)

# Embed query
query = "Tell me about programming languages"
query_embedding = client.embed_query(query)

# Compute cosine similarity
def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

scores = [
    cosine_similarity(query_embedding, doc_emb)
    for doc_emb in doc_embeddings
]

# Get top result
top_idx = np.argmax(scores)
print(f"Most relevant: {documents[top_idx]}")
print(f"Similarity: {scores[top_idx]:.4f}")
```

