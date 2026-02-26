# code-review-graph

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/)
[![MCP](https://img.shields.io/badge/MCP-compatible-green.svg)](https://modelcontextprotocol.io/)
[![Tests](https://img.shields.io/badge/tests-30%2F30%20passing-brightgreen.svg)](#testing)

**Persistent incremental knowledge graph for token-efficient, context-aware code reviews with Claude Code.**

Stop re-scanning your entire codebase on every review. `code-review-graph` builds a structural graph of your code using Tree-sitter, tracks it incrementally, and gives Claude Code the context it needs to review only what changed — and everything affected by those changes.

## Why?

| Without graph | With graph |
|---|---|
| Full repo scan every review | Only changed + impacted files |
| No blast-radius awareness | Automatic impact analysis |
| Token-heavy (entire codebase) | 5-10x fewer tokens per review |
| Manual "what else does this affect?" | Graph-powered dependency tracing |

## Installation

```bash
# Clone and install
git clone https://github.com/tirth8205/code-review-graph.git
cd code-review-graph
python3 -m venv .venv && source .venv/bin/activate
pip install -e .
```

Then add to your Claude Code project's `.mcp.json`:

```json
{
  "mcpServers": {
    "code-review-graph": {
      "command": "/path/to/code-review-graph/.venv/bin/python",
      "args": ["-m", "server.main"],
      "cwd": "/path/to/code-review-graph"
    }
  }
}
```

## Quickstart

### 1. Build the graph (first time)

```
/code-review-graph:build-graph
```

This parses your entire codebase and creates the knowledge graph. Takes ~10s for a 500-file project.

### 2. Review your changes

```
/code-review-graph:review-delta
```

Only reviews files changed since your last commit, plus everything impacted by those changes.

**Before**: Claude reads 200 files, uses ~150k tokens.
**After**: Claude reads 8 changed + 12 impacted files, uses ~25k tokens.

### 3. Review a PR

```
/code-review-graph:review-pr
```

Full structural review of a branch diff with blast-radius analysis, test coverage gaps, and actionable recommendations.

## Architecture

```
┌─────────────────────────────────────────────┐
│                Claude Code                   │
│  ┌─────────┐  ┌──────────┐  ┌────────────┐ │
│  │  Skills  │  │  Hooks   │  │   Agent    │ │
│  └────┬────┘  └────┬─────┘  └─────┬──────┘ │
│       │            │               │         │
│       └────────────┼───────────────┘         │
│                    │                         │
│              ┌─────▼─────┐                   │
│              │ MCP Server │                  │
│              └─────┬─────┘                   │
└────────────────────┼────────────────────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
    ┌────▼───┐  ┌───▼────┐  ┌──▼──────────┐
    │ Parser │  │ Graph  │  │ Incremental │
    │(sitter)│  │(SQLite)│  │  (git diff) │
    └────────┘  └────────┘  └─────────────┘
```

**Components**:
- **Parser** (`server/parser.py`): Tree-sitter multi-language AST parser. Extracts structural nodes and relationships.
- **Graph** (`server/graph.py`): SQLite-backed knowledge graph with NetworkX for traversal queries.
- **Incremental** (`server/incremental.py`): Git-aware delta detection. Re-parses only changed files + their dependents.
- **MCP Server** (`server/main.py`): Exposes 6 tools to Claude Code via the Model Context Protocol.
- **Skills**: Three review workflows (`build-graph`, `review-delta`, `review-pr`).
- **Hooks**: Auto-updates the graph on file edits and git commits.

## Graph Schema

### Nodes

| Kind | Properties |
|------|-----------|
| **File** | path, language, last_parsed_hash, size |
| **Class** | name, file, line_start, line_end, modifiers |
| **Function** | name, file, class (nullable), line_start, line_end, params, return_type, is_test |
| **Type** | name, file, kind (enum, interface, etc.) |
| **Test** | name, file, tested_function |

### Edges

| Kind | Direction | Meaning |
|------|-----------|---------|
| **CALLS** | Function -> Function | Function calls another function |
| **IMPORTS_FROM** | File -> File/Module | File imports from another |
| **INHERITS** | Class -> Class | Class extends another |
| **IMPLEMENTS** | Class -> Interface | Class implements an interface |
| **CONTAINS** | File/Class -> Function/Class | Containment hierarchy |
| **TESTED_BY** | Function -> Test | Function has a test |
| **DEPENDS_ON** | Node -> Node | General dependency |

## MCP Tools

| Tool | Description |
|------|-------------|
| `build_or_update_graph_tool` | Full or incremental graph build |
| `get_impact_radius_tool` | Blast radius analysis for changed files |
| `query_graph_tool` | Predefined relationship queries (callers, callees, tests, imports) |
| `get_review_context_tool` | Token-optimized review context with source snippets |
| `semantic_search_nodes_tool` | Search code entities by name/keyword |
| `list_graph_stats_tool` | Graph statistics and health check |

## Supported Languages

| Language | Extensions | Status |
|----------|-----------|--------|
| Python | `.py` | Full support |
| TypeScript | `.ts`, `.tsx` | Full support |
| JavaScript | `.js`, `.jsx` | Full support |
| Go | `.go` | Full support |
| Rust | `.rs` | Full support |
| Java | `.java` | Full support |
| C# | `.cs` | Full support |
| Ruby | `.rb` | Full support |
| Kotlin | `.kt` | Full support |
| Swift | `.swift` | Full support |
| PHP | `.php` | Full support |
| C/C++ | `.c`, `.h`, `.cpp`, `.hpp` | Full support |

## Configuration

Create a `.code-review-graphignore` file in your repo root to exclude paths:

```
# Ignore generated files
generated/**
*.generated.ts
*.pb.go

# Ignore vendor
vendor/**
third_party/**
```

## Troubleshooting

### Database lock errors
The graph uses SQLite with WAL mode. If you see lock errors, ensure only one build process runs at a time. The database auto-recovers.

### Large repositories (>10k files)
First build may take 30-60 seconds. Subsequent incremental updates are fast (<2s). Consider adding more ignore patterns for generated/vendor code.

### Missing nodes after build
Check that the file's language is supported and the file isn't in an ignore pattern. Run with `full_rebuild=True` to force a complete re-parse.

### Graph seems stale
Hooks auto-update on edit/commit. If the graph is stale, run `/code-review-graph:build-graph` manually.

## Comparison

| Feature | code-review-graph | code-graph-rag | CocoIndex |
|---------|:-:|:-:|:-:|
| Review-first design | Yes | No | No |
| Claude Code integration | Native | No | No |
| Incremental updates | Yes | Partial | Yes |
| No external DB needed | Yes (SQLite) | No (Neo4j) | No |
| Auto-update hooks | Yes | No | No |
| Impact/blast radius | Yes | No | No |
| Multi-language | 12+ languages | Python only | Varies |
| Token-efficient reviews | Yes | No | No |

## Contributing

### Adding a new language

1. Add the extension mapping in `server/parser.py` → `EXTENSION_TO_LANGUAGE`
2. Add node type mappings in `_CLASS_TYPES`, `_FUNCTION_TYPES`, `_IMPORT_TYPES`, `_CALL_TYPES`
3. Test with a sample file in that language
4. Submit a PR

### Development

```bash
# Install dev dependencies
pip install -e ".[dev]"

# Run tests
pytest

# Lint
ruff check server/
```

## License

MIT
