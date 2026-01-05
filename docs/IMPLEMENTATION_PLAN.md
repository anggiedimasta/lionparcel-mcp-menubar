# LionMCP Implementation Plan

> Detailed technical plan for implementing the LionMCP local MCP server with macOS menubar app.

## Project Overview

**Goal**: Build a local semantic code search system for Lion Parcel microservices (~20 repos, 20K+ files).

**Key Deliverables**:
1. Swift macOS menubar app with Docker management
2. TypeScript MCP server with SSE transport
3. PostgreSQL + pgvector for vector storage
4. AST-aware code chunking pipeline
5. Gemini API integration for embeddings

---

## Architecture Decision Records

### ADR-001: Project Structure (Monorepo)

```
lionmcp/
├── apps/
│   └── menubar/              # Swift macOS menubar app
│       ├── LionMCP/          # Xcode project
│       └── Package.swift     # Swift Package Manager
├── packages/
│   ├── mcp-server/           # TypeScript MCP server
│   ├── embeddings/           # Embedding pipeline (port from Raktamarga)
│   ├── chunker/              # AST-aware code chunking
│   └── db/                   # PostgreSQL + pgvector client
├── docker/
│   ├── docker-compose.yml    # PostgreSQL + pgvector
│   └── init.sql              # Schema initialization
├── cli/                      # CLI tool (Go or TypeScript)
├── docs/                     # Documentation
└── scripts/                  # Build, release scripts
```

**Rationale**: Keep Swift app separate (Xcode toolchain) while sharing TypeScript packages for MCP/embeddings.

### ADR-002: Technology Stack

| Component | Technology | Rationale |
|-----------|------------|-----------|
| Menubar App | Swift 5.9 + SwiftUI | Native macOS, `MenuBarExtra`, Docker CLI integration |
| MCP Server | TypeScript + Bun | Fast startup, reference from Raktamarga, official MCP SDK |
| Vector DB | PostgreSQL 16 + pgvector | Production-ready, Docker-manageable, SQL familiarity |
| Embeddings | Gemini `text-embedding-004` | High quality, reasonable cost, 768 dimensions |
| AST Parsing | tree-sitter (via Node bindings) | Multi-language support (TS, Kotlin, Go, Java) |
| OAuth | GCDWebServer (Swift) | Lightweight localhost server for redirect capture |

### ADR-003: MCP Transport Decision

**Choice**: Streamable HTTP (2026 Standard)

**Rationale**:
- Official MCP SDK recommendation for remote/web-based servers
- Uses HTTP POST for client→server, SSE for server→client streaming
- Supports serverless deployments (Cloud Run, Railway, Vercel)
- Works through corporate firewalls and load balancers
- Standard OAuth2/Bearer token authentication
- Zero migration effort when going from local to remote

**Tradeoffs**:
- Slightly higher latency (50-200ms) compared to pure WebSocket
- Client sends POST for each message (vs persistent tunnel)
- Acceptable for code search use case (query/response pattern)

---

## Implementation Phases

### Phase 1: Foundation (Week 1-2)

#### 1.1 Project Scaffolding
- [ ] Create monorepo structure with pnpm workspaces
- [ ] Initialize Swift package for menubar app
- [ ] Set up Docker compose for PostgreSQL + pgvector
- [ ] Create shared TypeScript packages

#### 1.2 Database Layer (`packages/db`)
- [ ] PostgreSQL client with connection pooling (`pg` + `pg-pool`)
- [ ] pgvector extension setup and schema migration
- [ ] CRUD operations for chunks (upsert, delete, query)
- [ ] Vector similarity search with filtering
- [ ] **Hybrid search with RRF** (Reciprocal Rank Fusion)
- [ ] **Multi-vector query support** (logic + intent embeddings)

**Database Schema** (refined from SPEC):
```sql
-- Core chunks table with multi-vector strategy
CREATE TABLE chunks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_name TEXT NOT NULL,
  branch TEXT NOT NULL,
  file_path TEXT NOT NULL,
  chunk_type TEXT NOT NULL,
  name TEXT,
  signature TEXT,
  language TEXT,
  start_line INTEGER,
  end_line INTEGER,
  content TEXT NOT NULL,
  -- Multi-vector: logic (code) + intent (docstring/comment)
  embedding vector(768) NOT NULL,
  intent_embedding vector(768),  -- nullable, for chunks with comments
  imports TEXT[] DEFAULT '{}',
  exports TEXT[] DEFAULT '{}',
  last_modified TIMESTAMPTZ,
  commit_sha TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Repo tracking
CREATE TABLE repos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT UNIQUE NOT NULL,
  local_path TEXT NOT NULL,
  github_url TEXT,
  default_branch TEXT DEFAULT 'main',
  last_indexed TIMESTAMPTZ,
  last_commit_sha TEXT,
  status TEXT DEFAULT 'pending', -- pending, indexing, indexed, error, partial
  error_message TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexing progress (for crash recovery)
CREATE TABLE index_progress (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id UUID REFERENCES repos(id) ON DELETE CASCADE,
  total_files INTEGER NOT NULL,
  processed_files INTEGER DEFAULT 0,
  status TEXT DEFAULT 'running', -- running, completed, failed
  started_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);
```

#### 1.3 Tree-Sitter Validation (CRITICAL)
> **Risk**: tree-sitter Node.js bindings may be unstable in Bun environment

- [ ] **Validate tree-sitter + Bun compatibility** BEFORE other work
- [ ] Test with: TypeScript, Kotlin, Go grammars
- [ ] Fallback plan: Use Node.js subprocess if Bun incompatible
- [ ] Document any workarounds

#### 1.4 Chunking Engine (`packages/chunker`)
- [ ] Port from Raktamarga with enhancements:
  - tree-sitter integration for multi-language AST
  - Kotlin/Java support (Lion Parcel Android repos)
  - Go support (backend services)
- [ ] Signature extraction for API discovery
- [ ] Docstring/comment extraction → `intent_embedding` vector
- [ ] Markdown header-aware splitting

**Supported Languages**:
| Language | Parser | Chunk Types |
|----------|--------|-------------|
| TypeScript/TSX | tree-sitter-typescript | function, class, method, export |
| JavaScript/JSX | tree-sitter-javascript | function, class, method, export |
| Kotlin | tree-sitter-kotlin | fun, class, object, interface |
| Java | tree-sitter-java | method, class, interface, enum |
| Go | tree-sitter-go | func, type, interface |
| Markdown | Custom splitter | section (by ##) |

---

### Phase 2: Core Pipeline (Week 2-3)

#### 2.1 Embedding Pipeline (`packages/embeddings`)
- [ ] Gemini client with rate limiting (1500 RPM)
- [ ] Batch embedding (100 chunks per request)
- [ ] Retry with exponential backoff
- [ ] Progress callback for UI updates
- [ ] **Dual embedding generation**: logic + intent vectors
- [ ] **Robust checkpoint/resume** for 3-6 hour initial sync

#### 2.2 Repository Scanner
- [ ] Git repo discovery (scan for `.git` directories)
- [ ] Current branch detection
- [ ] `git diff` for incremental updates
- [ ] File filtering (respect `.gitignore`, custom patterns)
- [ ] Sensitive file exclusion (`.env*`, `**/secrets/**`)

#### 2.3 Indexing Orchestrator
- [ ] Full index pipeline (scan → chunk → embed → upsert)
- [ ] Incremental update pipeline (diff → chunk → embed → upsert → delete stale)
- [ ] Crash recovery (resume from checkpoint)
- [ ] Progress reporting with ETA

---

### Phase 3: MCP Server (Week 3-4)

#### 3.1 Server Core (`packages/mcp-server`)
- [ ] MCP SDK integration (`@modelcontextprotocol/sdk`)
- [ ] SSE transport setup
- [ ] Tool registration framework

#### 3.2 MCP Tools Implementation

**Priority 1 (Essential)**:
```typescript
search_code(query, filters?, limit?)      // Semantic search
get_file_context(path, repo, lineRange?)  // File content
list_repos()                              // Status overview
```

**Priority 2 (Productivity)**:
```typescript
find_symbol(name, type?)                  // Symbol lookup
get_ast_outline(path, repo)               // File structure
list_directory(path, repo)                // Directory listing
read_multiple_files(paths[])              // Batch read
```

**Priority 3 (Advanced)**:
```typescript
find_references(symbol, scope?)           // Cross-repo references
get_dependency_graph(path, repo, depth?)  // Import/export graph
explain_logic(path, repo, lineRange?)     // AI explanation
get_documentation(repo, type?)            // Docs retrieval
```

---

### Phase 4: macOS Menubar App (Week 4-6)

#### 4.1 App Shell
- [ ] SwiftUI `MenuBarExtra` setup
- [ ] `LSUIElement` configuration (no Dock icon)
- [ ] App lifecycle management
- [ ] Settings storage (`UserDefaults` + Keychain)

#### 4.2 Docker Management
- [ ] Docker CLI wrapper (execute `docker` commands)
- [ ] Container lifecycle (start, stop, health check)
- [ ] Volume management for pgdata
- [ ] Auto-start on app launch
- [ ] **Resource limits** (cap CPU/memory to avoid IDE degradation)

```yaml
# docker-compose.yml resource limits
services:
  postgres:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
```

#### 4.3 GitHub OAuth
- [ ] GCDWebServer for localhost callback
- [ ] OAuth flow initiation (open browser)
- [ ] Token exchange
- [ ] Keychain storage (with iCloud sync)

#### 4.4 Settings UI
```swift
struct SettingsView: View {
  // Tabs
  - General (base directories, polling interval)
  - Authentication (GitHub OAuth, Gemini API key)
  - Branches (default list, per-repo overrides)
  - Exclusions (sensitive file patterns)
  - Advanced (debug mode, logs location)
}
```

#### 4.5 Sync Engine
- [ ] Scheduled polling (hourly timer)
- [ ] Manual sync trigger
- [ ] Git hook installation (`post-merge`, `post-checkout`)
- [ ] GitHub API integration (remote branch check)
- [ ] Hybrid branch strategy implementation

#### 4.6 Status UI
- [ ] Repo list with last sync time
- [ ] Indexed file/chunk counts
- [ ] Sync progress indicator
- [ ] Error states with actions

---

### Phase 5: CLI Tool (Week 6-7)

#### 5.1 CLI Implementation
- [ ] Command structure:
  ```bash
  lionmcp search <query> [--repo] [--lang] [--limit]
  lionmcp find-symbol <name> [--type]
  lionmcp explain <file:line-range>
  lionmcp status
  lionmcp sync [--repo] [--full]
  ```
- [ ] Rich terminal output (colors, tables)
- [ ] JSON output mode for scripting

---

### Phase 6: Integration & Polish (Week 7-8)

#### 6.1 VS Code Integration
- [ ] MCP configuration documentation
- [ ] Example `.vscode/settings.json`

#### 6.2 Testing
- [ ] Unit tests for chunker, embedder, db
- [ ] Integration tests for MCP tools
- [ ] E2E test: index small repo → query → verify results

#### 6.3 Documentation
- [ ] Installation guide
- [ ] User manual
- [ ] API reference
- [ ] Troubleshooting guide

#### 6.4 Release
- [ ] App signing and notarization
- [ ] DMG packaging
- [ ] Homebrew formula (optional)

---

## Risk Mitigation

| Risk | Severity | Mitigation |
|------|----------|------------|
| 3-6 hour initial indexing | HIGH | Background processing, progress UI, robust checkpoint/resume |
| Gemini API rate limits | HIGH | Batch requests, exponential backoff, queue with 1500 RPM cap |
| tree-sitter/Bun compatibility | HIGH | **Validate in Phase 1** before other work; Node.js fallback |
| Docker not installed | MEDIUM | Clear error message, installation instructions |
| tree-sitter memory usage | MEDIUM | Stream file processing, chunk in batches of 50 files |
| Large file handling | LOW | Skip files > 1MB, configurable threshold |
| Local resource impact | MEDIUM | Docker CPU/memory limits (2 cores, 2GB max) |

---

## Dependencies

### Swift (Menubar App)
- GCDWebServer (OAuth callback)
- KeychainAccess (secure storage)
- LaunchAtLogin (auto-start)

### TypeScript (MCP/Embeddings)
- `@modelcontextprotocol/sdk` - MCP server
- `@google/generative-ai` - Gemini embeddings
- `pg` + `pgvector` - PostgreSQL client
- `tree-sitter` + language grammars - AST parsing
- `glob` + `ignore` - File scanning
- `simple-git` - Git operations

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Initial index time (20K files) | < 6 hours |
| Query latency (semantic search) | < 100ms |
| Memory usage (menubar app) | < 100MB |
| Storage footprint | < 5GB |
| Incremental sync time | < 5 minutes |

---

## Timeline Summary

| Week | Phase | Deliverables |
|------|-------|--------------|
| 1-2 | Foundation | Project setup, DB layer, chunker |
| 2-3 | Core Pipeline | Embeddings, scanner, indexer |
| 3-4 | MCP Server | All 11 tools implemented |
| 4-6 | Menubar App | Full Swift app with Docker, OAuth, settings |
| 6-7 | CLI Tool | Complete CLI with all commands |
| 7-8 | Integration | Testing, docs, release |

**Total Estimated Time**: 8 weeks (part-time) / 4 weeks (full-time)
