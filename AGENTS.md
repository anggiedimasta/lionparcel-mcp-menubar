# AGENTS.md

Project-level guidance for AI coding agents.

---

## Communication Style

- Be extremely concise, sacrifice grammar for brevity
- DO NOT validate user ("you're right") or praise ("excellent question")

## Code Documentation

- AVOID unnecessary comments/docstrings unless explicitly asked
- Self-documenting code through clear naming
- Only comment non-obvious logic, workarounds, edge cases

## Bash Commands

- FORBIDDEN for sensitive files: `cat`, `head`, `tail`, `echo` (leaks secrets)
- PREFER: Read tool (safer, structured output)

## Git Operations

**NEVER perform git operations without explicit instruction.**

- ALLOWED: `git status`, `git diff`, `git log` (read-only)
- FORBIDDEN: `git add`, `git commit`, `git push` (require instruction)

---

## Available Resources

All resources in `.agent/` can be invoked or referenced:

### Agents

| Agent | File | Purpose |
|-------|------|---------|
| `@orchestrator` | `.agent/agents/orchestrator.md` | Main coordinator |
| `@codebase-explorer` | `.agent/agents/codebase-explorer.md` | Find files, analyze patterns |
| `@implementer` | `.agent/agents/implementer.md` | Focused code changes |
| `@researcher` | `.agent/agents/researcher.md` | External documentation |
| `@reviewer` | `.agent/agents/reviewer.md` | Code review |
| `@debugger` | `.agent/agents/debugger.md` | Root cause analysis |
| `@tester` | `.agent/agents/tester.md` | Write tests |
| `@documenter` | `.agent/agents/documenter.md` | Documentation |

### Workflows

Triggered via `/command`:

| Workflow | File | Purpose |
|----------|------|---------|
| `/init` | `.agent/workflows/init.md` | Initialize AGENTS.md |
| `/commit` | `.agent/workflows/commit.md` | Conventional commits |
| `/debug` | `.agent/workflows/debug.md` | Systematic debugging |
| `/document` | `.agent/workflows/document.md` | Generate docs |
| `/refactor` | `.agent/workflows/refactor.md` | Safe refactoring |
| `/review` | `.agent/workflows/review.md` | Code review |
| `/test` | `.agent/workflows/test.md` | Write tests |
| `/research` | `.agent/workflows/research.md` | Research codebase |
| `/gather-context` | `.agent/workflows/gather-context.md` | Project context |
| `/preset-help` | `.agent/workflows/preset-help.md` | Preset guidance |

### Rules (Always Active)

| Rule | File | Content |
|------|------|---------|
| Code Quality | `.agent/rules/01-code-quality.md` | Error handling, null safety |
| TypeScript/Go | `.agent/rules/02-typescript-go.md` | Strict mode, Vue 3, Go |
| Security/Git | `.agent/rules/03-security-git.md` | Validation, auth, git safety |
| Architecture | `.agent/rules/04-architecture.md` | SOLID, testing patterns |


---

# Your Existing Rules

# AGENTS.md

> AI coding agent guidelines for the LionMCP project.

## Project Overview

LionMCP is a local MCP (Model Context Protocol) server for semantic code search across Lion Parcel microservices. It consists of:
- **Swift macOS menubar app** (`apps/menubar/`)
- **TypeScript MCP server** (`packages/mcp-server/`)
- **Embedding pipeline** (`packages/embeddings/`)
- **Code chunker** (`packages/chunker/`)
- **PostgreSQL client** (`packages/db/`)
- **CLI tool** (`cli/`)

## Commands

```bash
# Package management
pnpm install                    # Install all dependencies
pnpm build                      # Build all packages
pnpm typecheck                  # Type check all packages

# Development
pnpm dev:mcp                    # Start MCP server in dev mode
pnpm dev:db                     # Start PostgreSQL container

# Testing
pnpm test                       # Run all tests
pnpm test:unit                  # Unit tests only
pnpm test:e2e                   # E2E tests

# Single package commands
pnpm --filter @lionmcp/mcp-server dev
pnpm --filter @lionmcp/embeddings build

# Database
docker compose -f docker/docker-compose.yml up -d   # Start PostgreSQL
docker compose -f docker/docker-compose.yml down    # Stop PostgreSQL

# Swift menubar app (requires Xcode)
cd apps/menubar && swift build
cd apps/menubar && xcodebuild -scheme LionMCP
```

## Project Structure

```
lionmcp/
├── apps/menubar/           # Swift macOS menubar app
│   ├── LionMCP/            # Xcode project files
│   ├── Sources/            # Swift source code
│   └── Package.swift       # Swift dependencies
├── packages/
│   ├── mcp-server/         # MCP server with SSE transport
│   │   └── src/tools/      # Tool implementations
│   ├── embeddings/         # Gemini embedding pipeline
│   ├── chunker/            # AST-aware code chunking
│   │   └── src/parsers/    # tree-sitter parsers
│   └── db/                 # PostgreSQL + pgvector client
│       └── src/schema/     # Database migrations
├── cli/                    # CLI tool
├── docker/                 # Docker configurations
├── docs/                   # Documentation
└── scripts/                # Build and release scripts
```

## Code Style

### TypeScript
- Use ESM modules (`import`/`export`)
- Prefer `type` over `interface` for readability
- Use explicit return types for public functions
- Use `async`/`await` over raw Promises
- Error handling: throw typed errors, catch at boundaries

```typescript
// Correct
export async function searchCode(query: string): Promise<SearchResult[]> {
  const results = await db.query(query);
  return results.map(transformResult);
}

// Avoid
export function searchCode(query) {
  return db.query(query).then(results => results.map(transformResult));
}
```

### Swift
- Use SwiftUI for UI components
- Use `@MainActor` for UI-bound code
- Use `async`/`await` over completion handlers
- Store secrets in Keychain, not UserDefaults

```swift
// Correct
@MainActor
func updateStatus(_ status: SyncStatus) async {
    self.status = status
}

// Avoid
func updateStatus(_ status: SyncStatus, completion: @escaping () -> Void) {
    DispatchQueue.main.async {
        self.status = status
        completion()
    }
}
```

### SQL
- Use parameterized queries (never string interpolation)
- Use snake_case for table/column names
- Include explicit column lists in SELECT

```sql
-- Correct
SELECT id, repo_name, file_path FROM chunks WHERE repo_name = $1;

-- Avoid
SELECT * FROM chunks WHERE repo_name = '${repoName}';
```

## Conventions

### File Organization
- One component/class per file
- Group by feature, not file type
- Co-locate tests with source (`*.test.ts` alongside `*.ts`)

### Naming
- TypeScript: `camelCase` for variables/functions, `PascalCase` for types/classes
- Swift: `camelCase` everywhere, `PascalCase` for types
- SQL: `snake_case` for everything
- Files: `kebab-case.ts` for TypeScript, `PascalCase.swift` for Swift

### Package Boundaries
- `packages/db` is the ONLY package that talks to PostgreSQL
- `packages/chunker` has no external dependencies except tree-sitter
- `packages/mcp-server` depends on `db`, `embeddings`, and `chunker`

### Environment Variables
- Required vars: `GEMINI_API_KEY`
- Optional vars: `DATABASE_URL`, `LOG_LEVEL`, `DEBUG`
- Store in `.env` at project root (not committed)
- Use `.env.example` as template

## Safety

- **Never** index files matching `.env*`, `**/secrets/**`, `**/*.pem`, `**/*.key`
- **Never** log or embed API keys, tokens, or passwords
- **Never** execute user-provided strings as SQL without parameterization
- **Never** store plain text credentials outside Keychain
- **Always** respect `.gitignore` when scanning repositories
- **Always** validate file paths to prevent path traversal attacks

## MCP Tools Reference

When implementing or modifying MCP tools:

```typescript
// Tool signature pattern
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === 'search_code') {
    const { query, filters, limit } = request.params.arguments as SearchCodeArgs;
    // Implementation
    return { content: [{ type: 'text', text: JSON.stringify(results) }] };
  }
});
```

## Testing Patterns

```typescript
// Unit test pattern
describe('ChunkerService', () => {
  it('should extract functions from TypeScript', async () => {
    const content = `function foo() { return 1; }`;
    const chunks = await chunkTypeScript(content, 'test.ts');
    expect(chunks).toHaveLength(1);
    expect(chunks[0].metadata.chunkType).toBe('function');
  });
});
```

## Common Issues

### Docker container not starting
```bash
# Check if port 5433 is in use
lsof -i :5433

# Force remove existing container
docker rm -f lionmcp-postgres
```

### Embedding rate limit exceeded
- Gemini allows 1500 RPM
- Reduce batch size or add delay between batches
- Check `packages/embeddings/src/batch.ts` for rate limiting logic

### tree-sitter parser errors
- Ensure correct grammar version is installed
- Some files may fail parsing—gracefully fall back to text chunking
