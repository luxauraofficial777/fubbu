# fubbu
FUBBU Telemetric/Topological Agentic Orchestration Layer for VW NEXUS &amp; Liminal Link
# FUBBU

**Fully Unmanaged Backbone Bridge Unit — local multi-agent orchestration sidecar.**

FUBBU is a self-hosted, local multi-agent sidecar that runs as an independent OS process alongside VW Nexus. It implements a **Router-Worker-Verifier loop** using local open-weight models (Ollama) for heavy low-level tasks: MIPS analysis, PSX emulator debugging, binary diffing, code review, and architecture analysis.

---

## What It Does

FUBBU receives tasks from VW Nexus, decomposes them, retrieves context via RAG, delegates to a worker model for inference, verifies the output through structural checks and an advisory model, and posts verified results back to Nexus. Failed verification triggers feedback to the worker for retry.

### Pipeline Stages

```
1. THINK:    Nexus → Router     (task received)
2. ROUTE:    Router → RAG       (context retrieval via Nexus RAG API)
3. DELEGATE: Router → Worker    (task assigned with context + constraints)
4. EXECUTE:  Worker → Ollama    (inference request to local model)
5. VERIFY:   Worker → Verifier  (structural checks + advisory model)
6. FEEDBACK: Verifier → Worker  (rejection with constraints, if failed)
7. COMPLETE: Verifier → Nexus   (verified result posted back)
```

---

## Models

| Role | Model | Purpose |
|------|-------|---------|
| **Router** | `qwen2.5-coder:7b` | Small, fast — task decomposition + RAG + worker assignment |
| **Worker** | `deepseek-coder:6.7b` | Large — actual debug analysis and inference |
| **Verifier** | `qwen2.5-coder:7b` | Small — structural + advisory verification |

Models are configurable in `fubbu_config.yaml`.

---

## Verifier Design

**Two-tier verification:**

1. **Structural checks (hard gate)** — Immediate rejection if any fail:
   - `mips_address_range` — MIPS addresses in valid range
   - `file_exists` — Referenced files exist
   - `json_valid` — JSON output is well-formed
   - `diff_applicable` — Diffs are applicable
   - `no_hallucination_markers` — No hallucination patterns detected

2. **Advisory model (soft gate)** — 0.0-1.0 confidence score, threshold 0.7:
   - Structural failure = immediate rejection, skip advisory
   - Advisory failure = feedback to worker with rejection reason
   - Advisory pass = result accepted, posted to Nexus

---

## Architecture

- **Process isolation** — Runs as subprocess, never imported into Nexus
- **State containment** — Separate SQLite DBs (`fubbu_state.db`, `fubbu_metrics.db`, `fubbu_topography.db`), never writes to `nexus_state.db`
- **API-only Nexus communication** — All Nexus interaction via HTTP on `:8651`
- **File lock** — `fubbu.lock` prevents dual instances
- **Heartbeat** — Every 30s via `POST /agents/{id}/heartbeat`
- **Telemetry mirroring** — Events mirrored to Nexus event bus (batched, every 10s)

---

## Modules

| File | Purpose |
|------|---------|
| `nexus_router_bridge.py` | Main sidecar entry point + orchestration loop |
| `fubbu_router.py` | Router: task decomposition + RAG + worker assignment |
| `fubbu_worker.py` | Worker: Ollama inference client with retry |
| `fubbu_verifier.py` | Verifier: structural checks + advisory model |
| `fubbu_state.py` | Isolated SQLite state (5 tables) |
| `fubbu_nexus_client.py` | Thin HTTP client for Nexus API |
| `fubbu_telemetry.py` | Step tracking + state mirroring + metrics DB |
| `fubbu_topography.py` | Agent communication topology (JSON + DOT export) |
| `fubbu_config.yaml` | Configuration (models, ports, thresholds) |
| `__main__.py` | CLI: start, status, topology, test-connection |

---

## CLI

```powershell
# Start the sidecar
python -m fubbu start

# Show current status
python -m fubbu status

# Export topology graph
python -m fubbu topology

# Test Nexus + Ollama connectivity
python -m fubbu test-connection
```

---

## Configuration

`fubbu_config.yaml`:

```yaml
nexus:
  url: "http://127.0.0.1:8651"
  poll_interval: 5

models:
  router:
    provider: ollama
    model: qwen2.5-coder:7b
    temperature: 0.3
  worker:
    provider: ollama
    model: deepseek-coder:6.7b
    temperature: 0.4
    max_retries: 3
  verifier:
    provider: ollama
    model: qwen2.5-coder:7b
    temperature: 0.2
    advisory_threshold: 0.7

capabilities:
  - low_level_debug
  - mips_analysis
  - binary_diff
  - psx_emulator_debug
  - code_review
  - architecture_analysis
```

---

## Capabilities

FUBBU claims tasks from Nexus matching these capabilities:

- `low_level_debug` — General low-level debugging
- `mips_analysis` — MIPS disassembly and analysis
- `binary_diff` — Binary comparison and diffing
- `psx_emulator_debug` — PSX emulator troubleshooting
- `code_review` — Code review and quality analysis
- `architecture_analysis` — Architecture assessment

---

## Integration

- **VW Nexus** — Registers as agent `fubbu_router`, claims tasks, posts results, mirrors telemetry
- **VW RAG** — Router queries RAG for context before delegating to worker
- **Ollama** — All three models (router, worker, verifier) run via Ollama
- **VW Deck** — Agent card visible in deck UI with state, capabilities, current task

---

<div align="center">

**LUX AURA**

*Autonomous agentic systems for the curious.*

[🌐 Bandcamp](https://luxaura.bandcamp.com) · [📘 Facebook](https://facebook.com/LuxAuraOfficial) · [💻 GitHub](https://github.com/luxauraofficial777) · [▶️ YouTube](https://youtube.com/c/LuxAuraOfficial)

</div>
