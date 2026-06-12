# OpenClaw Memory System

> **让 OpenClaw Agent 拥有持久记忆。**
>
> 自动同步 Obsidian 知识库 → Agent 可搜索。会话观察零成本压缩 → 不丢失。向量库 + 嵌入服务 + 同步链路 → 自动健康监控。

## 它解决什么问题

你用 Obsidian 记了大量笔记，但 OpenClaw Agent 的 `memory_search` 搜不到。
你和 Agent 聊了半天，做了很多决策和发现，对话结束就丢了。
你的 Qdrant 向量库、LM Studio 嵌入服务、vault 同步链路，哪个断了都不知道。

这个技能把这三件事自动化了。

## 快速开始

```bash
# 1. 克隆到 OpenClaw 技能目录
cd ~/.openclaw/workspace/skills
git clone https://github.com/yxyujian98-png/openclaw-memory-system.git
cd openclaw-memory-system

# 2. 安装依赖
pip install -r requirements.txt

# 3. 启动 Qdrant
docker-compose up -d

# 4. 运行安装向导（自动检查依赖、创建配置、初始化 Qdrant）
python scripts/setup.py --vault-dir /path/to/your/vault

# 5. 按 setup.py 输出的指引配置 hooks 和 cron
```

## Configuration

### config.json

```json
{
  "vault_dir": "/path/to/vault",
  "llm": {
    "baseUrl": "https://api.deepseek.com/v1",
    "apiKey": "***",
    "model": "deepseek-chat"
  },
  "embedder": {
    "baseUrl": "http://localhost:1234/v1",
    "apiKey": "***",
    "model": "text-embedding-nomic-embed-text-v1.5",
    "embeddingDims": 768
  },
  "qdrant": {
    "host": "localhost",
    "port": 6333,
    "collection": "knowledge_base"
  },
  "sync_dirs": ["01-日记", "02-知识", "04-教训", "07-项目"]
}
```

### Environment Variables (alternative)

```bash
OPENCLAW_VAULT_DIR="/path/to/vault"
OPENCLAW_LMSTUDIO_URL="http://localhost:1234/v1"
OPENCLAW_LMSTUDIO_KEY="***"
OPENCLAW_QDRANT_HOST="localhost"
OPENCLAW_QDRANT_PORT=6333
```

## Scripts

### Core Pipeline

| Script | Token Cost | What It Does |
|--------|:----------:|-------------|
| `shared_config.py` | 0 | Centralized config (env > config.json > defaults) |
| `qdrant_utils.py` | 0 | Qdrant CRUD (search, scroll, upsert, update_payload) |
| `embedder.py` | 0 | 3-level embedding fallback + LRU cache (500 entries, 5min TTL) |
| `compress.py` | 0 | Rule-based observation structuring (type, concepts, importance, narrative) |
| `observe.py` | 0 | Tool call observation queue (append-only JSONL, 1MB rotation) |
| `sync_vault_memory.py` | 0 | vault → workspace/memory/ (incremental, by mtime) |
| `vault_to_qdrant.py` | 0 | vault → Qdrant (chunk by heading, embed, version track) |
| `extract_memories.py` | LLM for high | Step 1: session→compress→Qdrant; Step 2: concept fusion; Step 3: distillation |
| `unified_memory.py` | LLM for classify | Mem0 + Qdrant search + PRISM intent routing |

### Health & Maintenance

| Script | What It Does |
|--------|-------------|
| `vault_guardian.py` | Vault health + stale detection + incremental sync + broken link check |
| `memory_health.py` | 4-chain check: vault→memory, memory→qdrant, LM Studio, Qdrant |
| `lmstudio_guardian.py` | Embedding server health + degraded mode flag |
| `context_snapshot.py` | Pre-compaction context backup to vault |
| `compress_to_rule.py` | Execution patterns → antibody/rule candidates |
| `maintenance_orchestrator.py` | DAG scheduler with topological sort + parallel execution |

## Key Design Decisions

### 1. Dual-Layer Architecture

OpenClaw's built-in memory (SQLite) is the **primary** search path. Custom scripts (Qdrant) are **supplementary**:
- `memory_search` → SQLite (always available, hybrid search)
- `unified_memory.py` → Qdrant (custom pipelines, observation analysis)

The bridge: `sync_vault_memory.py` copies vault files to `memory/`, so they appear in `memory_search`.

### 2. Zero-Token-First

`compress.py` structures raw observations without LLM:
- Rule-based type derivation (file_read, command_run, error, decision, ...)
- Keyword-based concept extraction
- Rule-based importance scoring (error=7, decision=8, file_read=2, ...)
- Quality gate rejects low-importance observations

LLM is only used for high-importance observations (importance ≥ 8 or type=decision/discovery).

### 3. 3-Level Embedding Fallback

```
Level 1: LM Studio (nomic-embed-text-v1.5) — best quality
Level 2: Local ONNX (all-MiniLM-L6-v2) — offline fallback
Level 3: Numpy hash — degraded but functional, Qdrant can still retrieve
```

Plus LRU cache (500 entries, 5min TTL) to avoid redundant network calls.

### 4. Version Tracking in Qdrant

Every Qdrant point carries:
- `version`: integer, incremented on each update
- `is_latest`: boolean, only newest is True
- `supersedes`: list of replaced point IDs
- `deleted`: boolean, soft-delete for vault file removal

### 5. PRISM Intent Routing

`unified_memory.py` classifies queries into intent types and routes to optimal search:

| Intent | Keywords | Strategy |
|--------|----------|----------|
| factual | 什么, 谁, 哪, API, 命令 | keywords_first → vector fallback |
| procedural | 怎么, 如何, 步骤, 配置 | path_pattern → vector fallback |
| reflective | 为什么, 原因, 分析, 看法 | vector direct |
| recency | 最近, 昨天, 上次 | recency_first → vector fallback |

### 6. Antibody System

Error patterns from session logs are stored as "antibodies" in `data/antibodies.json`:
- `pattern`: error string to match
- `fix`: human-readable fix
- `auto_fix`: PowerShell command for automatic repair
- `hits`: match count
- `success_rate`: fix success tracking

## Troubleshooting

```bash
# Full health check
python scripts/memory_health.py

# Check OpenClaw memory index
openclaw memory status
openclaw memory status --deep

# Check Qdrant
curl http://localhost:6333/collections/knowledge_base

# Test embedding
python scripts/embedder.py "test text"

# Force vault sync
python scripts/vault_to_qdrant.py

# Check config
python scripts/shared_config.py

# Force reindex
openclaw memory index --force

# Check hooks
openclaw hooks list

# Check cron
openclaw cron list
```

## License

MIT
