# Context Lattice (memMCP)

Private, reusable context infrastructure for AI agents.

[![HTTP MCP](https://img.shields.io/badge/MCP-HTTP%20only-0f766e)](https://modelcontextprotocol.io/)
[![Local First](https://img.shields.io/badge/Mode-local--first-166534)](#quickstart)
[![Compose](https://img.shields.io/badge/Deploy-Docker%20Compose-1d4ed8)](#quickstart)
[![Dashboard](https://img.shields.io/badge/UI-Next.js%20Dashboard-111827)](#operator-dashboard)

> Public overview note: this repository is private. Its `README.md` is synced to `sheawinkler/memmcp-overview` (public brand: Context Lattice).
> Public web pages (`index.html`, `updates.html`) are sourced from `docs/public_overview/` and synced with `scripts/sync_public_overview.sh`.

## Why Context Lattice

Most agent stacks leak reliability and budget through context drift:
- too much long-context prompt stuffing
- inconsistent retrieval quality across tools
- memory fanout failures that silently reduce durability
- runaway storage growth during bursty write periods

Context Lattice is a local-first memory fabric that gives agents one coherent system for:
- write durability
- retrieval quality
- queue health
- retention controls

## What Is New (2026)

- Fanout coalescer window that collapses repeated hot writes before they amplify outbox pressure.
- Backlog-aware Letta admission control that drops low-value fanout writes under pressure while preserving core durability.
- In-process low-value sink retention worker for Qdrant and Letta, with optional Mongo pruning.
- Shared HTTP client pools + batched fanout writes for lower overhead.
- Federated retrieval across Qdrant, Mongo raw, MindsDB, Letta, and memory-bank lexical fallback.
- Telemetry endpoints for fanout, retention, queue health, and rollup behavior.

## Core Capabilities

- HTTP-only MCP endpoint for memory service integration.
- Durable fanout outbox with retry semantics (`sqlite` default, `mongo` backend option).
- Multi-sink architecture:
  - Qdrant for vector recall
  - Mongo raw as source-of-truth write log
  - MindsDB autosync for SQL-side analytics/search workflows
  - Letta archival memory for agent recall
- Topic-path indexing and retrieval scoping.
- Learning loop support for preference-aware rerank.
- High-risk task gating for sensitive operations.

## Architecture At A Glance

```text
Client / Agent
  -> HTTP MCP (/mcp)
  -> Orchestrator (/memory/write, /memory/search)
      -> memory-bank (durable write path)
      -> fanout outbox (retry + dedupe + coalescing)
          -> Qdrant
          -> Mongo raw
          -> MindsDB
          -> Letta
      -> telemetry + retention workers
```

## Quickstart

### Prerequisites
- Docker Desktop (Compose v2)
- `gmake`, `jq`, `rg`, `python3`, `curl`
- macOS 13+ (tested)

### 1) Configure env

```bash
cp .env.example .env
ln -svf ../../.env infra/compose/.env
```

### 2) Start stack

```bash
gmake mem
gmake mem-ps
gmake mem-logs
```

Profile shortcuts:

```bash
gmake mem-mode-core
gmake mem-mode-full
gmake mem-up-lite
```

### 3) Verify

```bash
curl -fsS http://127.0.0.1:8075/health | jq
curl -fsS http://127.0.0.1:8075/telemetry/fanout | jq
curl -fsS http://127.0.0.1:8075/telemetry/retention | jq
```

## Operator Dashboard

- URL: `http://localhost:3000`
- Setup page: `http://localhost:3000/setup`
- Status page: `http://localhost:3000/status`

Local dashboard dev:

```bash
cd memmcp-dashboard
npm install
MEMMCP_ORCHESTRATOR_URL=http://127.0.0.1:8075 npm run dev
```

## Reliability And Storage Controls

### Queue and fanout
- `FANOUT_COALESCE_ENABLED=true`
- `FANOUT_COALESCE_WINDOW_SECS=6`
- `FANOUT_COALESCE_TARGETS=qdrant,mindsdb,letta,langfuse`
- `LETTA_ADMISSION_ENABLED=true`
- `LETTA_ADMISSION_BACKLOG_SOFT_LIMIT=800`
- `LETTA_ADMISSION_BACKLOG_HARD_LIMIT=2500`

### Retention
- `SINK_RETENTION_ENABLED=true`
- `SINK_RETENTION_INTERVAL_SECS=2100`
- `QDRANT_LOW_VALUE_RETENTION_HOURS=72`
- `LETTA_LOW_VALUE_RETENTION_HOURS=72`
- `MONGO_RAW_LOW_VALUE_RETENTION_HOURS=0` (disabled by default)

### Existing retention jobs
- `scripts/retention_runner.sh`
- `scripts/fanout_outbox_gc.py`
- `scripts/install_retention_runner.sh`

## HTTP Endpoints (selected)

- `POST /memory/write`
- `POST /memory/search`
- `GET /telemetry/memory`
- `GET /telemetry/fanout`
- `GET /telemetry/retention`
- `POST /telemetry/retention/run`
- `GET /status/ui`
- `GET /pilot`

## Troubleshooting

- Disk pressure: see `docs/storage_and_retention.md`.
- Fanout backlog: check `GET /telemetry/fanout`.
- Retention behavior: check `GET /telemetry/retention` and trigger `POST /telemetry/retention/run`.
- Compose env loading: ensure `infra/compose/.env` symlink points to project `.env`.

## Docs

- `docs/orchestrator_enhancements.md`
- `docs/performance.md`
- `docs/retention_ops.md`
- `docs/onprem_full_runbook.md`
- `docs/launch_checklist.md`
- `docs/pilot_offer.md`

## Business And Licensing

- Open for local/self-hosted use.
- Context Lattice Cloud planned for managed enterprise deployments (SSO/SAML, RBAC, quotas, billing, SLAs).
- License: Business Source License 1.1 with change-date transition to Apache-2.0 (see `LICENSE`).

## Pilot CTA

Run a 2-4 week context-cost audit:
- baseline token + reliability profile
- deploy and tune retrieval + fanout
- produce ROI and rollout recommendation

Contact via `PILOT_CONTACT_EMAIL` or `PILOT_CONTACT_URL` in orchestrator config.
