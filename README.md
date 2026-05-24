# vertrectdirectory

A local RAG pipeline over all Vertical Rectangle and Epsilver project documentation, exposed as an MCP tool for Claude Code.

Ask Claude anything about any VR project and it searches the actual docs instead of guessing.

---

## What it does

Chunks and embeds every markdown file across all VR/Epsilver repos into a local Chroma vector store. A lightweight MCP server exposes a `search_docs` tool so Claude Code can semantically search project documentation mid-conversation — no copy-pasting, no context stuffing.

Projects indexed:
- Pop Maker Studio
- Silvertune / Silvertune Web / Silvertune Pedal
- Lyric Video Blender
- Wickrunner
- Cultural Extremity Index (CEI + cei-cli)
- keybpm
- epsilver-site

---

## Stack

- **Embedding model** — `all-MiniLM-L6-v2` via sentence-transformers (runs fully locally, ~80MB)
- **Vector store** — Chroma (persistent, on disk, no server needed)
- **MCP layer** — same `mcp` SDK as Pop Maker Studio's MCP server

No API keys. No cloud. No subscription.

---

## Setup

**1. Clone all VR/Epsilver repos into `~/dev/`**

```bash
git clone https://github.com/verticalrectangle/pop-maker-studio
git clone https://github.com/verticalrectangle/silvertune-web
# ... etc
```

**2. Install dependencies**

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

**3. Build the index**

```bash
.venv/bin/python3 ingest.py
```

Re-run any time docs change.

**4. Wire into Claude Code**

Add to `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "vr-docs": {
      "command": "/absolute/path/to/vertrectdirectory/.venv/bin/python3",
      "args": ["/absolute/path/to/vertrectdirectory/server.py"]
    }
  }
}
```

Restart Claude Code. The `search_docs` tool is now available.

---

## How it works

**Ingestion (`ingest.py`)**

Each markdown file is split on headers — every section becomes one chunk. Chunks are embedded with `all-MiniLM-L6-v2` and stored in Chroma with project and filename metadata. The embedding step runs once; results persist to `chroma_db/` on disk.

**Search (`server.py`)**

At query time, the same embedding model converts the query to a vector. Chroma finds the N closest chunks by cosine similarity. Results are returned with project attribution and similarity score so Claude knows where the information came from.

**Why not an LLM for retrieval?**

Embedding models are purpose-built for similarity search — small, fast, and cheap to run. The LLM (Claude) only runs once per query, reading the retrieved chunks to synthesize an answer. Keeping the two roles separate is standard RAG practice and means the heavy inference only happens when it needs to.

---

## MCP tool

| Tool | Description |
|------|-------------|
| `search_docs(query, n?)` | Semantic search over all project docs. Returns top N chunks with project/file attribution and similarity score. |
