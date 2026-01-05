# LionMCP

> Local MCP server for semantic code search across Lion Parcel microservices.

## Overview

LionMCP is a developer productivity tool that:
- **Indexes** all Lion Parcel repositories into a local PostgreSQL + pgvector database
- **Exposes** semantic search via MCP server (SSE transport)
- **Syncs** automatically with GitHub using OAuth authentication
- **Runs** as a native macOS menubar app

## Quick Start

### Prerequisites

- macOS 13+ (Ventura or later)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Node.js 20+](https://nodejs.org/) (or [Bun](https://bun.sh/))
- [pnpm](https://pnpm.io/)
- [Gemini API Key](https://makersuite.google.com/app/apikey)

### Installation

```bash
# Clone the repository
git clone https://github.com/lionparcel/lionmcp.git
cd lionmcp

# Install dependencies
pnpm install

# Copy environment template
cp .env.example .env
# Edit .env and add your GEMINI_API_KEY

# Start PostgreSQL
docker compose -f docker/docker-compose.yml up -d

# Build all packages
pnpm build

# Start MCP server
pnpm dev:mcp
```

### VS Code Configuration

Add to your VS Code settings:

```json
{
  "mcp.servers": {
    "lionmcp": {
      "transport": "streamable-http",
      "url": "http://localhost:19848/mcp"
    }
  }
}
```

## Documentation

- [SPEC.md](docs/SPEC.md) - Full specification
- [IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md) - Development roadmap
- [ARCHITECTURE.md](docs/ARCHITECTURE.md) - System architecture
- [STATUS.md](docs/STATUS.md) - Current implementation status

## Project Structure

```
lionmcp/
├── apps/menubar/         # Swift macOS menubar app
├── packages/
│   ├── mcp-server/       # MCP server (SSE)
│   ├── embeddings/       # Gemini embedding pipeline
│   ├── chunker/          # AST-aware code chunking
│   └── db/               # PostgreSQL + pgvector
├── cli/                  # Command-line tool
├── docker/               # Docker configurations
└── docs/                 # Documentation
```

## Features

### MCP Tools

| Tool | Description |
|------|-------------|
| `search_code` | Semantic search across all repos |
| `find_symbol` | Find function/class definitions |
| `get_file_context` | Retrieve file content with metadata |
| `get_ast_outline` | Get file structure (functions, classes) |
| `list_directory` | Browse repository directories |
| `find_references` | Find all usages of a symbol |
| `get_dependency_graph` | Visualize imports/exports |
| `explain_logic` | AI-powered code explanation |
| `list_repos` | Show indexed repositories |
| `get_documentation` | Retrieve README, AGENTS.md, etc. |

### Menubar App

- One-click GitHub OAuth
- Docker container management
- Hourly automatic sync
- Manual sync trigger
- Per-repo status display
- Settings UI for configuration

## Development

```bash
# Run tests
pnpm test

# Type check
pnpm typecheck

# Lint
pnpm lint

# Build menubar app (requires Xcode)
cd apps/menubar && swift build
```

## License

Proprietary - Lion Parcel Internal Use Only
