# AGENTS.md — src/memory

**Parent**: `./AGENTS.md` (read first)

## OVERVIEW
Memory persistence backends — SQLite, PostgreSQL, Lucid, Markdown, None.

## STRUCTURE
```
src/memory/
├── traits.rs       # Memory trait definition
├── mod.rs         # Factory + register_memory! macro
├── sqlite.rs      # SQLite hybrid search
├── postgres.rs    # PostgreSQL backend
├── lucid.rs       # Lucid bridge
├── markdown.rs    # Markdown file backend
├── none.rs        # No-op backend
├── vector.rs      # Vector embeddings + FTS5
└── *.rs           # Storage implementations
```

## WHERE TO LOOK
| Task | Location |
|------|----------|
| Add new backend | Implement `Memory` trait → register in `mod.rs` |
| Memory trait | `traits.rs` — `store()`, `recall()`, `forget()` |
| Vector search | `vector.rs` — cosine similarity, FTS5 hybrid |
| Embeddings | `EmbeddingProvider` trait in traits.rs |

## CONVENTIONS
- Memory impl: `struct FooMemory; impl Memory for FooMemory { ... }`
- Store: `async fn store(&self, context: MemoryContext) -> Result<()>`
- Recall: `async fn recall(&self, query: &str, limit: usize) -> Result<Vec<MemoryEntry>>`
- Hybrid merge: weighted vector + keyword (default 0.7/0.3)

## ANTI-PATTERNS
- **NEVER** skip index rebuilding for FTS5
- **NEVER** leak sensitive data in memory entries
- **NEVER** skip chunk size limits

## RISK: MEDIUM
Data persistence module. Changes require:
- Persistence tests
- Migration path documentation
