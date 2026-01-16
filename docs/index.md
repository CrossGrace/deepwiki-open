---
layout: default
title: Home
---

# DeepWiki Open

> Enterprise GitHub Wiki Generator with AI-Powered Documentation

## Overview

**DeepWiki Open** is an enterprise-grade wiki generator that automatically transforms GitHub repositories into comprehensive, AI-powered documentation. It uses advanced retrieval-augmented generation (RAG) to analyze your codebase and create structured, searchable wiki pages.

### Key Features

- 🤖 **AI-Powered Analysis**: Uses gpt-oss-130b for intelligent documentation generation
- 📊 **RAG-Based Content**: Retrieves relevant code context using BGE-M3 embeddings
- 🏢 **Enterprise Ready**: Designed for internal company use with custom authentication
- 🚀 **Single-Provider Architecture**: Simplified setup with minimal dependencies
- 🌐 **Multi-Language Support**: 14+ languages via i18n
- 🎨 **Modern UI**: Built with React 19 and Next.js 15
- 🔄 **Automated Pipeline**: 7-stage process from ingestion to wiki generation

## Quick Start

### Prerequisites

- Python 3.11+
- Node.js 18+
- Git
- Enterprise API access (gpt-oss-130b + BGE-M3)

### Installation

```bash
# Clone the repository
git clone https://github.com/CrossGrace/deepwiki-open.git
cd deepwiki-open

# Install Python dependencies
pip install -r requirements_single.txt

# Install Node dependencies
yarn install
```

### Configuration

Set up your environment variables:

```bash
export GPT_OSS_API_BASE="https://your-company-api.com"
export GPT_OSS_API_KEY="your-api-key"
```

### Generate a Wiki

```bash
python deepwiki_single.py \
  --repo https://github.com/your-org/your-repo \
  --output ./output/wiki
```

### Run the Web UI

```bash
# Development mode
yarn dev

# Production build
yarn build
yarn start
```

## Architecture

DeepWiki Open follows a **7-stage pipeline architecture**:

1. **Ingest**: Clone and load repository files
2. **Chunk**: Split files into overlapping segments
3. **Plan**: AI-generated wiki structure
4. **Embed**: Generate vector embeddings
5. **Generate**: RAG-based page generation
6. **Write**: Output markdown files
7. **Sidebar**: Auto-generate navigation

```
GitHub Repo → Ingest → Chunk → Plan → Embed → Generate → Write → Wiki Pages
                                  ↓
                            BGE-M3 Embeddings
                                  ↓
                            Vector Retrieval (RAG)
```

## Key Technologies

### Backend
- **Python 3.11+**: Core pipeline
- **httpx**: HTTP client for API calls
- **numpy**: Vector operations for RAG
- **gpt-oss-130b**: Text generation LLM
- **BGE-M3**: 1024-dim embeddings

### Frontend
- **Next.js 15**: React framework
- **React 19**: UI components
- **Tailwind CSS**: Styling
- **Mermaid**: Diagram rendering
- **next-intl**: Internationalization

## Documentation

- [Architecture](architecture.md) - System design and data flow
- [Usage Guide](usage.md) - Detailed usage instructions
- [API Reference](api-reference.md) - API endpoints and clients
- [Configuration](configuration.md) - Configuration options
- [Deployment](deployment.md) - Deployment guides

## Project Statistics

| Metric | Value |
|--------|-------|
| Python Files | 43 |
| TypeScript/TSX Files | 32 |
| Core Pipeline Code | ~1,242 lines |
| Dependencies | 2 Python, 13+ Node.js |
| Supported Languages | 14+ |
| Test Coverage | Unit + Integration |

## Contributing

Contributions are welcome! Please read our contributing guidelines before submitting pull requests.

## License

See [LICENSE](../LICENSE) for details.

## Links

- [GitHub Repository](https://github.com/CrossGrace/deepwiki-open)
- [Report Issues](https://github.com/CrossGrace/deepwiki-open/issues)
- [Enterprise Integration Guide](../ENTERPRISE_INTEGRATION.md)
