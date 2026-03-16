# ADPL — Agentic Data Pipeline Language

**Version:** 1.0
**File extension:** `.adpl`
**MIME type:** `application/vnd.adpl+json`

ADPL is an open JSON-based file format for describing, sharing, and reconstructing agentic data pipelines. A single `.adpl` file captures everything the **Pipeline CAD** tool at [datainsight.at](https://datainsight.at) needs to reconstruct a full pipeline graph — nodes, edges, agent configurations, AHI log entries, and human-readable metadata.

---

## Design Goals

| Goal | Description |
|---|---|
| **Portable** | One file fully describes a pipeline. No external references required to open it. |
| **Reproducible** | Importing an `.adpl` file produces an identical canvas state. |
| **Human-readable** | Valid JSON, logically organized, self-documenting field names. |
| **Code-generation-ready** | Contains all config values needed for infrastructure code generation (Compose, Airflow DAGs, dbt projects, agent configs). |
| **Agent-authored** | Designed to be written by the `pipeline_builder` H.A.R.L.I.E. agent from natural-language project descriptions. |

---

## File Structure

```
pipeline.adpl
├── adpl          — format version ("1.0")
├── meta          — human-readable metadata
│   ├── name      — pipeline name
│   ├── description
│   ├── created   — ISO 8601 timestamp
│   ├── updated   — ISO 8601 timestamp
│   ├── project   — originating case study reference (optional)
│   ├── author    — who or what created this file
│   ├── tags      — searchable technology labels
│   ├── autonomy_level — L0 / L2 / L4
│   └── tool      — "datainsight.at Pipeline CAD"
│
├── pipeline      — the graph
│   ├── goal      — one-sentence pipeline purpose
│   ├── nodes[]   — canvas nodes (sources, transforms, agents, etc.)
│   └── edges[]   — directed connections between node ports
│
├── ahi           — Agent-Human Interface
│   └── log[]     — coordination entries (observations, orders, approvals…)
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
| `adpl` | string | ✅ | Format version. Must be `"1.0"`. |
| `meta` | object | ✅ | File metadata. |
| `pipeline` | object | ✅ | The pipeline graph. |
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
| `project.name` | string | — | Case study name, e.g. `"Case #018 — Multi-Framework Agent Pipeline"`. |
| `project.url` | string | — | Relative URL to case study page. |
| `project.case_number` | string | — | Zero-padded case number, e.g. `"018"`. |
| `author` | string | ✅ | Creator identity, e.g. `"H.A.R.L.I.E. pipeline_builder agent"`. |
| `tags` | string[] | ✅ | Technology tags, e.g. `["kafka","dbt","airflow","ai-agent"]`. |
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
| `config` | object | ✅ | Node-specific configuration. Keys vary by subtype (see below). |

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

### `ahi.log[]`

Each entry is a coordination message between agents and humans.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✅ | Unique entry ID, e.g. `"agent-20260316-001"`. |
| `type` | string | ✅ | `observation` / `recommendation` / `alert` / `acknowledgement` / `order` / `approval` / `override` |
| `from` | string | ✅ | Sender, e.g. `"pipeline_monitor"` or `"human"`. |
| `to` | string | ✅ | Recipient or `"all"`. |
| `date` | string | ✅ | ISO 8601 date. |
| `status` | string | ✅ | `noted` / `pending` / `acted` |
| `content` | string | ✅ | Message body. |

---

### `summary`

Flattened representation for display and quick inspection. Computed from the graph, not authoritative.

| Field | Type | Description |
|---|---|---|
| `sources` | string[] | List of source subtype keys. |
| `transform` | string | Transform subtype key (`"dbt"`, `"python"`, or `""`). |
| `orchestration` | string | Orchestration subtype key (`"airflow"`, `"dagster"`, or `""`). |
| `quality` | string | Quality subtype key (`"great_expectations"`, or `""`). |
| `serving` | string[] | List of serving subtype keys. |
| `agents` | object[] | Each: `{ name, model, role, autonomy }`. |

---

## Minimal Valid Example

```json
{
  "adpl": "1.0",
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
Open [Pipeline CAD](https://datainsight.at) → **Import** → select a `.adpl` file. The tool reads `pipeline.nodes`, `pipeline.edges`, `pipeline.goal`, and `ahi.log` to reconstruct the full canvas state. All node positions, configurations, and AHI entries are preserved.

### Export
Click **Export ADPL** to download the current canvas as a `.adpl` file, including all node configs, edge connections, and AHI log entries. The tool populates `summary` automatically.

---

## How the `pipeline_builder` Agent Creates ADPL Files

The `pipeline_builder` H.A.R.L.I.E. agent reads project descriptions from `projects/{slug}/index.html` and translates the architecture into an ADPL file saved as `projects/{slug}/pipeline.adpl`. See `agents/skills/pipeline_builder/TASK.md` for the full agent protocol.

---

## Versioning

The `adpl` field in the root identifies the format version. Future versions will be backward-compatible for import (older files always open in newer tools). Breaking changes increment the major version.

| Version | Date | Notes |
|---|---|---|
| `1.0` | 2026-03-16 | Initial release |

---

## License

ADPL format specification is released under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/) — use freely, no attribution required.
