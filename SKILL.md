# OpenClaw Memory System / OpenClaw 记忆系统

> **让 OpenClaw Agent 拥有持久记忆。**
>
> 自动同步 Obsidian 知识库 → Agent 可搜索。会话观察零成本压缩 → 不丢失。向量库 + 嵌入服务 + 同步链路 → 自动健康监控。

## 关键词

记忆系统、长期记忆、向量检索、知识库同步、Obsidian vault、Qdrant、embedding、agent memory、零 LLM 压缩、自愈监控、heartbeat、vault 同步、概念聚合、记忆蒸馏、健康检查

## 这是什么

一个 OpenClaw 的记忆增强技能。解决了三个问题：

1. **Vault 内容进不了记忆** — 你用 Obsidian 记了大量笔记，但 Agent 的 `memory_search` 搜不到
2. **会话记忆丢失** — 工具调用、决策、发现等有价值信息，对话结束就没了
3. **记忆系统维护成本高** — 向量库、嵌入服务、同步链路，哪个断了都不知道

## 安装

```bash
cd ~/.openclaw/workspace/skills
git clone https://github.com/yxyujian98-png/openclaw-memory-system.git
cd openclaw-memory-system
pip install -r requirements.txt
docker-compose up -d                              # 启动 Qdrant
python scripts/setup.py --vault-dir /path/to/vault  # 7 步自动配置
```

## 你需要什么

| 组件 | 必需 | 说明 |
|------|:----:|------|
| Python 3.10+ | ✅ | 脚本运行环境 |
| Qdrant | ✅ | 向量数据库（docker-compose 一行启动） |
| 嵌入服务 | ✅ | LM Studio / Ollama / 任何 OpenAI 兼容的 embedding 端点 |
| Obsidian Vault | ✅ | 你的 Markdown 知识库 |
| LLM API | 可选 | 只有高重要性记忆才需要（DeepSeek / OpenAI 等） |

## 它做了什么

```
你的 Obsidian Vault                          OpenClaw memory_search
  ├── 01-日记/     ── sync_vault_memory.py ──→  memory/*.md (SQLite)
  ├── 02-知识/     ── vault_to_qdrant.py   ──→  Qdrant (向量库)
  ├── 04-教训/
  └── 07-项目/

Agent 工具调用     ── observe.py + compress.py ──→ Qdrant (结构化观察)

每 45 分钟自动：vault 健康检查 → 增量同步 → 记忆压缩 → 健康报告
```

**零 LLM 成本**：compress.py 用规则驱动（不调 LLM）把工具调用结构化为类型/概念/重要性。
**三级嵌入降级**：LM Studio → 本地 ONNX → numpy 哈希，嵌入服务挂了也能跑。
**版本追踪**：Qdrant 向量带 version/is_latest/supersedes，知识演化不冲突。

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 1: OpenClaw Built-in                    │
│                                                                 │
│  session-memory hook ──→ memory/YYYY-MM-DD-HHMM.md             │
│  memory-compact hook ──→ extract on compaction                  │
│  memory-extract hook ──→ extract on /new, /reset                │
│                          │                                      │
│                          ▼                                      │
│  memory_search ◄── SQLite (main.sqlite)                        │
│                  ├── FTS5 (BM25 keyword)                        │
│                  ├── sqlite-vec (vector similarity)             │
│                  └── hybrid merge                               │
│                  ├── indexes: memory/*.md, MEMORY.md            │
│                  └── extraPaths: skills.memory.md, self-improving│
└─────────────────────────────────────────────────────────────────┘
         │ sync_vault_memory.py (vault → memory/)
         │ vault_to_qdrant.py (vault → Qdrant)
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 2: Custom Scripts                       │
│                                                                 │
│  Cron (heartbeat every 45m):                                    │
│    maintenance_orchestrator --cycle light --parallel            │
│      ├── vault_guardian.py      (vault health + incremental sync)│
│      ├── vault_to_qdrant.py    (vault → Qdrant vector sync)    │
│      ├── extract_memories.py   (session → compress → Qdrant)   │
│      ├── memory_health.py      (4-chain health check)          │
│      ├── lmstudio_guardian.py  (embedding server health)       │
│      └── ... (14 more tasks in DAG order)                      │
│                                                                 │
│  Cron (heavy every 6h):                                         │
│    maintenance_orchestrator --cycle heavy --parallel            │
│      ├── vault_to_qdrant.py    (full sync)                     │
│      ├── extract_memories.py --full (consolidate + distill)    │
│      ├── build_project_profile.py                              │
│      └── session_cleaner.py                                    │
│                                                                 │
│  Qdrant (knowledge_base):                                       │
│    ├── vault_sync pipeline (vault markdown → chunks → embed)   │
│    ├── compress pipeline (tool calls → structured observations) │
│    └── consolidate pipeline (concepts → fused knowledge)       │
└─────────────────────────────────────────────────────────────────┘
```

## Data Flow (Runtime)

### Flow 1: User Chat → Memory (OpenClaw Built-in)

```
User sends message
  → Agent processes in session
  → User issues /new or /reset
    → session-memory hook fires (background)
      → Extracts last 15 user/assistant messages
      → Saves to memory/YYYY-MM-DD-HHMM.md
    → memory-extract hook fires
      → Extracts valuable memories
  → OpenClaw file watcher detects new memory/*.md
    → Debounced reindex (1.5s)
    → Chunks (~400 tokens, 80-token overlap)
    → Embeds via LM Studio (nomic-embed-text-v1.5)
    → Stores in SQLite (main.sqlite)
      → FTS5 index (keyword)
      → sqlite-vec (vector)
```

### Flow 2: Vault → Memory (Custom Scripts)

```
User edits vault file (Obsidian)
  → vault_watcher.py detects mtime change (every 600s scan)
    → Calls vault_to_qdrant.py
      → Chunks markdown by headings (15-line segments)
      → Embeds via LM Studio
      → Stores in Qdrant (knowledge_base collection)
      → Version tracking (version, is_latest, supersedes)

  → vault_guardian.py runs (heartbeat, every 45m)
    → Scans vault for stale files (>3 days with unresolved markers)
    → Syncs changed vault files to memory/ (incremental)
    → Checks Qdrant + LM Studio health
    → Reports broken [[wiki links]]
```

### Flow 3: Session → Compress → Qdrant (Custom Scripts)

```
Agent makes tool calls (read, edit, exec, search, ...)
  → observe.py registers observation to queue (~/.openclaw/observe_queue.jsonl)

Heartbeat (every 45m):
  → extract_memories.py runs
    → Reads recent session dumps from memory/
    → Parses user:/assistant: lines into observations
    → Also scans trajectory JSONL files for tool calls
    → compress.py structures each observation (0 LLM token):
        - derive_type (file_read, command_run, error, decision, ...)
        - extract_concepts (tech keywords)
        - score_importance (rules: error=7, decision=8, ...)
        - quality_gate filter (reject low-quality)
    → Indexes to Qdrant

  → extract_memories.py --full (heavy, every 6h):
    → Step 2: Concept consolidation (LLM)
        - Groups Qdrant observations by concept
        - Fuses related observations via LLM (ReMe architecture)
        - Writes fused knowledge to vault/02-知识/
    → Step 3: Vault distillation (LLM)
        - Clusters vault files by similarity
        - Distills clusters into summary documents
```

### Flow 4: Search (Both Layers)

```
Agent calls memory_search(query):
  → OpenClaw searches SQLite (main.sqlite):
    → FTS5 keyword search (BM25)
    → sqlite-vec vector search (cosine similarity)
    → Hybrid merge with configurable weights
  → Results from memory/*.md, MEMORY.md, extraPaths
  → This is the PRIMARY search path (built-in)

Agent calls unified_memory.py search (custom, optional):
  → Searches Qdrant knowledge_base (custom vectors)
  → Searches Mem0 (if configured)
  → PRISM intent routing (factual/procedural/reflective/recency)
  → This is a SUPPLEMENTARY search path
```

## What This Skill Covers

### Core Scripts (included)

| Script | Runs When | Purpose |
|--------|-----------|---------|
| `shared_config.py` | Imported by all | Centralized config (env > config.json > defaults) |
| `qdrant_utils.py` | Imported by all | Qdrant CRUD operations |
| `embedder.py` | On embed call | 3-level fallback: LM Studio → ONNX → numpy hash |
| `compress.py` | Heartbeat | Zero-LLM observation structuring |
| `observe.py` | After tool calls | Observation queue registration |
| `sync_vault_memory.py` | Heartbeat | vault → workspace/memory/ sync |
| `vault_to_qdrant.py` | Heartbeat/heavy | vault → Qdrant vector sync with versioning |
| `extract_memories.py` | Heartbeat | Session → compress → Qdrant; --full: consolidate + distill |
| `vault_guardian.py` | Heartbeat | Vault health + stale detection + incremental sync |
| `memory_health.py` | Heartbeat | 4-chain health check (vault→memory→qdrant→embedding) |
| `lmstudio_guardian.py` | Heartbeat | Embedding server health + degraded mode |
| `unified_memory.py` | Manual/cron | Mem0 + Qdrant unified search + PRISM routing |
| `maintenance_orchestrator.py` | Cron | DAG-based task scheduler |
| `setup.py` | Once | Initial setup (config, deps, Qdrant collection) |

### What OpenClaw Provides (not included, built-in)

- `session-memory` hook → auto-saves session to memory/
- `memory-compact` hook → extracts on compaction
- `memory-extract` hook → extracts on /new, /reset
- `memory_search` tool → hybrid SQLite search
- `memory_get` tool → read specific memory files
- SQLite index (main.sqlite) → FTS5 + sqlite-vec
- File watcher → debounced reindex on memory/*.md change
- Dreaming system → background promotion to MEMORY.md

### Integration Points

1. **vault → memory/**: `sync_vault_memory.py` copies vault .md to workspace/memory/, making them searchable by OpenClaw's built-in `memory_search`
2. **vault → Qdrant**: `vault_to_qdrant.py` indexes vault content in Qdrant for custom semantic search
3. **session → Qdrant**: `extract_memories.py` compresses session observations into Qdrant
4. **heartbeat**: Cron triggers `maintenance_orchestrator.py` which runs all scripts in DAG order
5. **memory_search extraPaths**: OpenClaw indexes `data/skills.memory.md` and `~/self-improving/` alongside memory/

## Quick Start

```bash
# 1. Install
cd ~/.openclaw/workspace/skills
git clone https://github.com/your-org/openclaw-memory-system.git

# 2. Setup
cd openclaw-memory-system
python scripts/setup.py --vault-dir /path/to/vault

# 3. Configure (edit scripts/config.json with your API keys)

# 4. Add heartbeat tasks to HEARTBEAT.md:
#    python scripts/vault_guardian.py
#    python scripts/extract_memories.py
#    python scripts/memory_health.py
```

## Prerequisites

| Component | Required | Purpose |
|-----------|:--------:|---------|
| Python 3.10+ | Yes | Script runtime |
| Qdrant | Yes | Custom vector store |
| LM Studio / embedding server | Yes | Text embeddings |
| Obsidian vault | Yes | Knowledge source |
| LLM API (DeepSeek etc) | Optional | Memory distillation (high-importance only) |
