---
name: qmd
description: Local markdown search engine for knowledge bases, notes, and documentation. Combines BM25 full-text search, vector semantic search, and LLM re-ranking. Use when searching through markdown notes, retrieving documents, or managing knowledge collections.
---

# QMD - Quick Markdown Search

Local, on-device search engine for indexing and retrieving information from markdown notes, documentation, and knowledge bases. Runs entirely locally without cloud dependencies.

## Available MCP Tools

| Tool | Description |
|------|-------------|
| `qmd_search` | Fast BM25 keyword search |
| `qmd_vsearch` | Semantic vector similarity search |
| `qmd_query` | Hybrid search with intelligent re-ranking |
| `qmd_get` | Retrieve document by path or docid |
| `qmd_multi_get` | Batch retrieve with glob patterns |
| `qmd_status` | Index health and collection info |

## Pre-configured Collections

| Collection | Path | Description |
|------------|------|-------------|
| `knowledge` | `/root/clawd/knowledge/` | Personal notes and documentation |
| `skills` | `/root/clawd/skills/` | Skill documentation and references |

## Search Examples

### Keyword Search (Fast)
```
qmd_search("authentication flow")
```

### Semantic Search (Conceptual)
```
qmd_vsearch("how to handle user sessions")
```

### Hybrid Search (Best Results)
```
qmd_query("API rate limiting strategies")
```

### Retrieve Specific Document
```
qmd_get("knowledge/meeting-notes.md")
qmd_get("#abc123")  # By document ID
```

### Batch Retrieve
```
qmd_multi_get("knowledge/*.md")
```

## Managing Collections

### Add New Collection
```bash
qmd collection add /path/to/docs --name mycollection
qmd context add "qmd://mycollection" "Description of the collection"
qmd embed  # Re-index with new collection
```

### List Collections
```bash
qmd collection list
```

### Remove Collection
```bash
qmd collection remove mycollection
```

## Adding Knowledge

To add documents to your knowledge base:

1. **Create markdown files** in `/root/clawd/knowledge/`
2. **Re-embed** to index: `qmd embed`

Example structure:
```
/root/clawd/knowledge/
├── projects/
│   ├── project-a.md
│   └── project-b.md
├── meeting-notes/
│   ├── 2026-01-15.md
│   └── 2026-01-20.md
└── reference/
    ├── api-docs.md
    └── architecture.md
```

## Search Tips

- **Keywords**: Use specific terms for exact matches
- **Semantic**: Use natural language questions for conceptual search
- **Hybrid**: Best for general discovery when you're unsure

## Index Management

```bash
qmd embed          # Full re-index
qmd status         # Check index health
qmd collection list # View collections
```

## Persistence

The qmd index is stored in `/root/.qmd/`. This is backed up to R2 along with other moltbot data.
