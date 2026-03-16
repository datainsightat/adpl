# ADPL — Agentic Data Pipeline Language

**Version:** 1.1
**File extension:** `.adpl`
**MIME type:** `application/vnd.adpl+json`

ADPL is an open JSON-based file format for describing, sharing, and reconstructing agentic data pipelines. A single `.adpl` file captures everything the **Pipeline CAD** tool at [datainsight.at](https://datainsight.at) needs to reconstruct a full pipeline graph — nodes, edges, agent configurations, AHI log entries, and human-readable metadata. Starting with v1.1, it also embeds the full system prompts for the **DE Setup Agent** and all **Pipeline Monitor Agents**.

---

## Design Goals

| Goal | Description |
|---|---|
| **Portable** | One file fully describes a pipeline. No external references required to open it. |
| **Reproducible** | Importing an `.adpl` file produces an identical canvas state. |
| **Human-readable** | Valid JSON, logically organized, self-documenting field names. |
| **Code-generation-ready** | Contains all config values needed for infrastructure code generation (Compose, Airflow DAGs, dbt projects, agent configs). |
| **Agent-authored** | Designed to be written by the `pipeline_builder` H.A.R.L.I.E. agent from natural-language project descriptions. |
| **Self-operating** | v1.1 embeds the full agent prompts needed to set up, test, and monitor the pipeline with zero additional configuration. |

---

## File Structure

```
pipeline.adpl
├── adpl          — format version ("1.1")
├── meta          — human-readable metadata
│   ├── name
│   ├── description
│   ├── created   — ISO 8601 timestamp
│   ├── updated   — ISO 8601 timestamp
│   ├── project   — originating case study reference (optional)
│   ├── author
│   ├── tags
│   ├── autonomy_level — L0 / L2 / L4
│   └── tool
│
├── pipeline      — the graph
│   ├── goal
│   ├── nodes[]   — canvas nodes (sources, transforms, agents, etc.)
│   └── edges[]   — directed connections between node ports
│
├── agents        — ★ NEW in v1.1 — embedded agent prompts
│   ├── setup     — DE Setup Agent: sets up and tests the pipeline
│   └── monitors[] — Monitor Agents: watch the pipeline in production
│
├── ahi           — Agent-Human Interface
│   └── log[]     — coordination entries
│
└── summary       — computed, human-readable snapshot
    ├── sources[]
    ├── transform
    ├── orchestration
    ├── quality
    ├── serving[]
    └── agents[]
```

---

## Full Schema Reference

### Root

| Field | Type | Required | Description |
|---|---|---|---|
| `adpl` | string | ✅ | Format version. Must be `"1.1"`. |
| `meta` | object | ✅ | File metadata. |
| `pipeline` | object | ✅ | The pipeline graph. |
| `agents` | object | ✅ | Embedded agent definitions and system prompts. |
| `ahi` | object | ✅ | Agent-Human Interface log. |
| `summary` | object | ✅ | Computed summary for display. |

---

### `meta`

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✅ | Short pipeline name, e.g. `"Real-time Order Analytics"`. |
| `description` | string | ✅ | One-paragraph description of what this pipeline does. |
| `created` | string | ✅ | ISO 8601 creation timestamp. |
| `updated` | string | ✅ | ISO 8601 last-update timestamp. |
| `project` | object | — | Reference to originating case study. |
| `project.name` | string | — | Case study name. |
| `project.url` | string | — | Relative URL to case study page. |
| `project.case_number` | string | — | Zero-padded case number, e.g. `"018"`. |
| `author` | string | ✅ | Creator identity, e.g. `"H.A.R.L.I.E. pipeline_builder agent"`. |
| `tags` | string[] | ✅ | Technology tags. |
| `autonomy_level` | string | ✅ | `"L0"` (no agents), `"L2"` (supervised agents), `"L4"` (multi-agent). |
| `tool` | string | ✅ | Always `"datainsight.at Pipeline CAD"`. |

---

### `pipeline.nodes[]`

Each node represents one block on the canvas.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | integer | ✅ | Unique numeric ID within this file. |
| `subtype` | string | ✅ | Node subtype key (see **Node Subtypes** table below). |
| `type` | string | ✅ | Node category: `source` / `transform` / `orchestration` / `agent` / `quality` / `serving` / `human`. |
| `label` | string | ✅ | Display label shown on canvas. |
| `icon` | string | ✅ | Emoji icon. |
| `x` | number | ✅ | Canvas X position in pixels. |
| `y` | number | ✅ | Canvas Y position in pixels. |
| `config` | object | ✅ | Node-specific configuration. All values are strings. |

#### Node Subtypes

| `subtype` | `type` | Config keys |
|---|---|---|
| `postgresql` | `source` | `description`, `host`, `port`, `database`, `schema` |
| `kafka` | `source` | `description`, `broker`, `topic`, `group` |
| `mongodb` | `source` | `description`, `uri`, `database`, `collection` |
| `csv` | `source` | `description`, `path`, `delimiter` |
| `rest_api` | `source` | `description`, `url`, `path` |
| `minio` | `source` | `description`, `endpoint`, `bucket` |
| `dbt` | `transform` | `description`, `project`, `target` |
| `python` | `transform` | `description`, `script`, `function` |
| `airflow` | `orchestration` | `description`, `dag_id`, `schedule` |
| `dagster` | `orchestration` | `description`, `job_name`, `schedule` |
| `mcp_server` | `agent` | `description`, `name`, `transport`, `port`, `tools` |
| `ai_agent` | `agent` | `name`, `model`, `role`, `goal`, `autonomy`, `description` |
| `great_expectations` | `quality` | `description`, `suite`, `datasource` |
| `rest_serve` | `serving` | `description`, `framework`, `port`, `endpoint` |
| `dashboard` | `serving` | `description`, `tool`, `port` |
| `data_warehouse` | `serving` | `description`, `type`, `schema`, `table` |

---

### `pipeline.edges[]`

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | integer | ✅ | Unique numeric ID. |
| `fromId` | integer | ✅ | Source node `id`. |
| `fromPort` | string | ✅ | Output port key, e.g. `"out-0"`. |
| `toId` | integer | ✅ | Target node `id`. |
| `toPort` | string | ✅ | Input port key, e.g. `"in-0"`. |

---

### `agents` ★ New in v1.1

The `agents` section embeds complete, ready-to-use system prompts for two classes of operational agent. Unlike the in-pipeline `ai_agent` nodes (which model agents that run *inside* the data flow), these agents operate *around* the pipeline — setting it up, testing it, and watching it in production.

#### `agents.setup` — DE Setup Agent

One setup agent per pipeline. Responsible for initialising infrastructure, running integration tests, and declaring the pipeline ready for production.

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✅ | Agent display name, e.g. `"DE Setup Agent"`. |
| `model` | string | ✅ | LLM model ID, e.g. `"claude-sonnet-4-6"`. |
| `role` | string | ✅ | Short role description. |
| `goal` | string | ✅ | One-sentence goal. |
| `autonomy` | string | ✅ | `"supervised"` / `"autonomous"` / `"advisory"`. |
| `triggers` | string[] | ✅ | When to invoke: `"on_deploy"`, `"on_schema_change"`, `"on_demand"`. |
| `tools` | string[] | ✅ | Tool names available to this agent. |
| `output_channel` | string | ✅ | Where output is written: `"exchange"`. |
| `system_prompt` | string | ✅ | Full system prompt text (Markdown). |

#### `agents.monitors[]` — Pipeline Monitor Agents

Zero or more monitor agents. Each watches a specific operational concern (orchestration health, data quality, agent output, data freshness).

Same field structure as `agents.setup`.

Standard monitor types (include whichever apply to this pipeline's stack):

| Monitor type | `role` | Trigger | Typical tools |
|---|---|---|---|
| Orchestration Monitor | `"orchestration health monitor"` | `"on_schedule"`, `"on_failure"` | `check_dag_status`, `query_run_history`, `append_exchange_entry` |
| Data Quality Monitor | `"data quality monitor"` | `"on_quality_run"`, `"on_schedule"` | `read_ge_results`, `query_expectations`, `append_exchange_entry` |
| Agent Output Monitor | `"agent output reviewer"` | `"on_agent_completion"`, `"on_schedule"` | `read_exchange`, `query_agent_outputs`, `append_exchange_entry` |
| Data Freshness Monitor | `"data freshness monitor"` | `"on_schedule"` | `query_row_counts`, `check_source_timestamps`, `append_exchange_entry` |

---

### `ahi.log[]`

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✅ | Unique entry ID. |
| `type` | string | ✅ | `observation` / `recommendation` / `alert` / `acknowledgement` / `order` / `approval` / `override` |
| `from` | string | ✅ | Sender. |
| `to` | string | ✅ | Recipient or `"all"`. |
| `date` | string | ✅ | ISO 8601 date. |
| `status` | string | ✅ | `noted` / `pending` / `acted` |
| `content` | string | ✅ | Message body. |

---

### `summary`

| Field | Type | Description |
|---|---|---|
| `sources` | string[] | Source subtype keys. |
| `transform` | string | Transform subtype key or `""`. |
| `orchestration` | string | Orchestration subtype key or `""`. |
| `quality` | string | Quality subtype key or `""`. |
| `serving` | string[] | Serving subtype keys. |
| `agents` | object[] | `{ name, model, role, autonomy }` — in-pipeline AI agents only. |

---

## Agent Prompt Conventions

System prompts in `agents.setup.system_prompt` and `agents.monitors[].system_prompt` follow this structure:

```markdown
# {Agent Name} — {Pipeline Name}

## Role
One-sentence role description.

## Pipeline Context
Stack overview, goal, key config values (host, DAG ID, schedule, etc.)

## Your Responsibilities
Numbered list of concrete checks and actions.

## Output Protocol
How to write to exchange.json (type, status, content format).

## Error Handling
What to do when checks fail — escalation rules, log locations.
```

All values referenced in prompts (host names, ports, DAG IDs, suite names) must match the `pipeline.nodes[].config` values in the same file.

---

## Minimal Valid Example (v1.1)

```json
{
  "adpl": "1.1",
  "meta": {
    "name": "Hello Pipeline",
    "description": "Minimal single-source pipeline for demonstration.",
    "created": "2026-03-16T00:00:00Z",
    "updated": "2026-03-16T00:00:00Z",
    "author": "human",
    "tags": ["postgresql", "dbt"],
    "autonomy_level": "L0",
    "tool": "datainsight.at Pipeline CAD"
  },
  "pipeline": {
    "goal": "Extract orders from PostgreSQL and transform with dbt",
    "nodes": [
      {
        "id": 1, "subtype": "postgresql", "type": "source",
        "label": "PostgreSQL", "icon": "🗄",
        "x": 100, "y": 200,
        "config": { "description": "Order database", "host": "localhost", "port": "5432", "database": "orders", "schema": "public" }
      },
      {
        "id": 2, "subtype": "dbt", "type": "transform",
        "label": "dbt", "icon": "🧱",
        "x": 380, "y": 200,
        "config": { "description": "Transform raw orders into analytics mart", "project": "orders_pipeline", "target": "postgres" }
      }
    ],
    "edges": [
      { "id": 1, "fromId": 1, "fromPort": "out-0", "toId": 2, "toPort": "in-0" }
    ]
  },
  "agents": {
    "setup": {
      "name": "DE Setup Agent",
      "model": "claude-sonnet-4-6",
      "role": "pipeline setup and testing",
      "goal": "Initialise the Hello Pipeline infrastructure, verify all services start cleanly, and confirm dbt models build successfully.",
      "autonomy": "supervised",
      "triggers": ["on_deploy", "on_demand"],
      "tools": ["run_compose", "run_health_checks", "run_dbt", "append_exchange_entry"],
      "output_channel": "exchange",
      "system_prompt": "# DE Setup Agent — Hello Pipeline\n\n## Role\nYou set up and test the Hello Pipeline infrastructure before it goes live.\n\n## Pipeline Context\n- **Goal**: Extract orders from PostgreSQL and transform with dbt\n- **Stack**: PostgreSQL → dbt\n- **Database**: localhost:5432 / orders / public schema\n- **dbt project**: orders_pipeline (target: postgres)\n\n## Your Responsibilities\n1. Start services: `podman-compose up -d`\n2. Wait for PostgreSQL to be ready: `make wait-db`\n3. Run schema and seed: `make init-db`\n4. Run dbt: `make dbt-run`\n5. Verify mart tables are populated\n6. Report result to exchange.json\n\n## Output Protocol\nAppend to `agents/exchange.json`:\n- `acknowledgement` (status: noted) if all checks pass\n- `alert` (status: pending) if any check fails — include exact error\n\n## Error Handling\n- Container fails to start: `podman logs <service>` for root cause\n- dbt error: check `dbt/logs/dbt.log` for failing model"
    },
    "monitors": [
      {
        "name": "Data Freshness Monitor",
        "model": "claude-sonnet-4-6",
        "role": "data freshness monitor",
        "goal": "Verify PostgreSQL source data and dbt mart tables are refreshed on schedule.",
        "autonomy": "autonomous",
        "triggers": ["on_schedule:0 8 * * *"],
        "tools": ["query_row_counts", "check_source_timestamps", "append_exchange_entry"],
        "output_channel": "exchange",
        "system_prompt": "# Data Freshness Monitor — Hello Pipeline\n\n## Role\nYou verify that data in PostgreSQL and the dbt mart is fresh and updated on schedule.\n\n## Pipeline Context\n- **Source**: PostgreSQL (localhost:5432 / orders / public)\n- **Transform**: dbt mart table in orders_pipeline\n- **Expected refresh**: daily\n\n## Your Responsibilities\n1. Query `max(updated_at)` from PostgreSQL source tables\n2. Query `max(updated_at)` from dbt mart table\n3. Check both are within the last 25 hours\n4. Report result to exchange.json\n\n## Output Protocol\n- `observation` if data is fresh\n- `alert` if any table is stale beyond 25 hours"
      }
    ]
  },
  "ahi": { "log": [] },
  "summary": {
    "sources": ["postgresql"],
    "transform": "dbt",
    "orchestration": "",
    "quality": "",
    "serving": [],
    "agents": []
  }
}
```

---

## How the Pipeline CAD Tool Uses ADPL

### Import
Open [Pipeline CAD](https://datainsight.at) → **Import** → select a `.adpl` file. The tool reads `pipeline.nodes`, `pipeline.edges`, `pipeline.goal`, and `ahi.log` to reconstruct the canvas. The `agents` section is preserved in the exported file on round-trip but not rendered on canvas.

### Export
Click **Export ADPL** to download the current canvas as a `.adpl` file. The tool auto-generates agent system prompts based on the detected pipeline stack.

---

## How the `pipeline_builder` Agent Creates ADPL Files

The `pipeline_builder` H.A.R.L.I.E. agent reads project descriptions from `projects/{slug}/index.html` and writes `projects/{slug}/pipeline.adpl`. See `agents/skills/pipeline_builder/TASK.md` for the full protocol including agent prompt generation rules.

---

## Versioning

| Version | Date | Notes |
|---|---|---|
| `1.0` | 2026-03-16 | Initial release |
| `1.1` | 2026-03-16 | Added `agents` section with embedded DE Setup Agent and Monitor Agent system prompts |

**Backward compatibility**: v1.0 files import correctly into tools that support v1.1. The `agents` section is optional during import — tools fill it with generated defaults if absent.

---

## License

ADPL format specification is released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) — use freely, no attribution required.
