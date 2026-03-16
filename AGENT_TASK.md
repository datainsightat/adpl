# Pipeline Builder Agent — Task Definition

> Role: Translate project case study descriptions into ADPL pipeline files
> Output: `projects/{slug}/pipeline.adpl` for each project case study

---

## Your Mission

You are the **Pipeline Builder** for the H.A.R.L.I.E. Collective. You translate real-world data engineering case study pages into machine-readable ADPL (Agentic Data Pipeline Language) files. Each `.adpl` file lets engineers immediately reconstruct the case study's pipeline architecture in the Pipeline CAD tool at datainsight.at — no manual assembly required.

---

## Context — Read Before Starting

- `/home/user/prompt_engineer/adpl/README.md` — ADPL format specification (full field reference, schema, examples)
- `/home/user/prompt_engineer/adpl/schema/v1.json` — JSON Schema for validation
- `/home/user/prompt_engineer/tools/pipeline-builder/catalog.js` — Valid node subtypes, types, and config keys
- `/home/user/prompt_engineer/projects/` — All case study directories
- `/home/user/prompt_engineer/agents/exchange.json` — Coordination log (read for orders, write acknowledgements)

---

## Step 0 — Read the Exchange Layer

**Before anything else**, read `agents/exchange.json`. Find all entries where:
- `"to": "pipeline_builder"` (or `"to": "all"`)
- `"status": "pending"`
- `"type"` is `"order"` or `"recommendation"`

These directives take priority. Act on them first, then proceed to gap-filling.

After acting, append an `acknowledgement` entry to `agents/exchange.json`:
```json
{
  "id": "pipeline_builder-20260316-001",
  "type": "acknowledgement",
  "from": "pipeline_builder",
  "to": "human",
  "date": "2026-03-16",
  "status": "noted",
  "content": "Generated pipeline.adpl for projects/multi-framework-agent-pipeline. 7 nodes, 6 edges.",
  "context": { "trigger_entry": "original-entry-id", "files_written": ["projects/multi-framework-agent-pipeline/pipeline.adpl"] }
}
```

---

## Step 1 — Inventory Projects

List all directories under `projects/`. For each, check whether `pipeline.adpl` exists:

```bash
ls projects/
ls projects/multi-framework-agent-pipeline/
```

Build a table:

| Slug | Has pipeline.adpl | Priority |
|---|---|---|
| multi-framework-agent-pipeline | No | High |
| kafka-streaming-pipeline | No | High |
| ... | ... | ... |

Process at most **3 projects per run**. Prioritise the newest cases (highest case number) and any flagged in exchange.json.

---

## Step 2 — Extract Architecture from Project Page

For each project, read `projects/{slug}/index.html`. Extract:

1. **Pipeline goal** — from page title or first description paragraph. One sentence.
2. **Data sources** — scan for tool names: PostgreSQL, Kafka, MongoDB, CSV, REST API, MinIO/S3. Map to catalog subtypes.
3. **Transformation** — dbt or Python. Note the project/script name if visible.
4. **Orchestration** — Airflow or Dagster. Note DAG ID or schedule if visible.
5. **Quality** — Great Expectations or none. Note suite name if visible.
6. **Serving** — REST API, Dashboard (Superset/Grafana/Metabase), or Data Warehouse. Note tool and port.
7. **AI Agents** — names, models (Claude/GPT-4o/etc.), roles, goals, autonomy level. Each becomes an `ai_agent` node. If an MCP server is mentioned, add an `mcp_server` node.
8. **Data flow** — from the Mermaid diagram or architecture section: what connects to what?

---

## Step 3 — Map to ADPL Nodes and Edges

### Node ID assignment
Start from 1, increment for each node. Assign in reading order: sources first, then orchestration, transform, quality, agents, serving.

### Canvas coordinates

Use this layout template:

```
Column A: Sources          x = 100
Column B: Orchestration    x = 340
Column C: Transform        x = 580
Column C: Quality          x = 580  (y +240 from transform)
Column C: Agents           x = 580  (y +240 from quality)
Column D: Serving          x = 820
```

Y positions:
- First node in each column: y = 150
- Each additional node in same column: y += 240

### Edge construction

Standard data flow edges:
- Source → Orchestration (`out-0` → `in-0`)
- Orchestration → Transform (`out-0` → `in-0`)
- Transform → Quality (`out-0` → `in-0`) if quality present
- Transform → Serving (`out-0` → `in-0`)
- Quality pass-through → Serving (`out-1` → `in-0`) — Great Expectations has 2 outputs: out-0 (fail) and out-1 (pass)
- Agents connect from Quality or Transform: (`out-0` → `in-0`)

When no orchestration node exists:
- Source → Transform directly

### Config defaults

If the project page doesn't specify a value, use catalog defaults:

| Subtype | Default config |
|---|---|
| postgresql | host: localhost, port: 5432, database: mydb, schema: public |
| kafka | broker: localhost:9094, topic: events, group: pipeline |
| mongodb | uri: mongodb://localhost:27017, database: mydb, collection: events |
| airflow | dag_id: pipeline_dag, schedule: @daily |
| dagster | job_name: pipeline_job, schedule: 0 2 * * * |
| dbt | project: pipeline_project, target: postgres |
| great_expectations | suite: pipeline_suite, datasource: my_datasource |
| rest_serve | framework: fastapi, port: 8000, endpoint: /api/v1/data |
| dashboard | tool: superset, port: 8088 |
| data_warehouse | type: postgres, schema: analytics, table: mart_main |

For `ai_agent` nodes, use the autonomy level from the project (supervised / autonomous / advisory). Default model to `claude-sonnet` unless the project specifies GPT-4o or similar.

---

## Step 4 — Compute Summary

After building nodes, compute the `summary` object:

```python
summary.sources = [n.subtype for n in nodes if n.type == 'source']
summary.transform = first(n.subtype for n in nodes if n.type == 'transform') or ''
summary.orchestration = first(n.subtype for n in nodes if n.type == 'orchestration') or ''
summary.quality = first(n.subtype for n in nodes if n.type == 'quality') or ''
summary.serving = [n.subtype for n in nodes if n.type == 'serving']
summary.agents = [{ name, model, role, autonomy } for n in nodes if n.type == 'agent' and n.subtype == 'ai_agent']
```

Autonomy level for `meta.autonomy_level`:
- `"L4"` if `len(agents) > 1`
- `"L2"` if `len(agents) == 1`
- `"L0"` if no agents

---

## Step 5 — Assemble and Write the ADPL File

Full `.adpl` template:

```json
{
  "adpl": "1.0",
  "meta": {
    "name": "<pipeline name from page title>",
    "description": "<one-paragraph description from project page>",
    "created": "<today ISO 8601>T00:00:00Z",
    "updated": "<today ISO 8601>T00:00:00Z",
    "project": {
      "name": "Case #<NNN> — <Title>",
      "url": "projects/<slug>/index.html",
      "case_number": "<NNN>"
    },
    "author": "H.A.R.L.I.E. pipeline_builder agent",
    "tags": ["<source subtypes>", "<transform>", "<orchestration>", "<serving tools>"],
    "autonomy_level": "<L0|L2|L4>",
    "tool": "datainsight.at Pipeline CAD"
  },
  "pipeline": {
    "goal": "<one-sentence goal>",
    "nodes": [ ... ],
    "edges": [ ... ]
  },
  "ahi": { "log": [] },
  "summary": { ... }
}
```

Write to `projects/<slug>/pipeline.adpl`.

---

## Step 6 — Self-Validation Checklist

Before writing the file, verify:

- [ ] `adpl` field is exactly `"1.0"` (string)
- [ ] All required fields are present (see `adpl/schema/v1.json`)
- [ ] Every `edges[].fromId` and `edges[].toId` references a real node `id`
- [ ] No duplicate node `id` values
- [ ] No duplicate edge `id` values
- [ ] All `config` values are strings (not integers or booleans)
- [ ] `summary.sources` matches the actual source nodes
- [ ] `meta.autonomy_level` matches agent count
- [ ] File is valid JSON (balanced braces, no trailing commas)

---

## Example ADPL File — Multi-Framework Agent Pipeline (Case #018)

```json
{
  "adpl": "1.0",
  "meta": {
    "name": "Multi-Framework Agent Pipeline — A2A, HITL, and OTel Tracing",
    "description": "Google ADK coordinates risk analysis tasks via A2A protocol to OpenAI Agents SDK workers, with cross-framework OpenTelemetry tracing stitching every LLM call and human approval into a single auditable trace for regulatory compliance.",
    "created": "2026-03-16T00:00:00Z",
    "updated": "2026-03-16T00:00:00Z",
    "project": {
      "name": "Case #018 — Multi-Framework Agent Pipeline",
      "url": "projects/multi-framework-agent-pipeline/index.html",
      "case_number": "018"
    },
    "author": "H.A.R.L.I.E. pipeline_builder agent",
    "tags": ["kafka", "python", "airflow", "great_expectations", "ai_agent", "mcp_server", "data_warehouse"],
    "autonomy_level": "L4",
    "tool": "datainsight.at Pipeline CAD"
  },
  "pipeline": {
    "goal": "Coordinate multi-framework AI agents for risk analysis with full OTel tracing and HITL approval",
    "nodes": [
      { "id": 1, "subtype": "kafka", "type": "source", "label": "Kafka", "icon": "📨", "x": 100, "y": 150,
        "config": { "description": "Real-time risk event stream from trading systems", "broker": "localhost:9094", "topic": "risk_events", "group": "risk_pipeline" }},
      { "id": 2, "subtype": "postgresql", "type": "source", "label": "PostgreSQL", "icon": "🗄", "x": 100, "y": 390,
        "config": { "description": "Historical trade and position database", "host": "localhost", "port": "5432", "database": "trading", "schema": "positions" }},
      { "id": 3, "subtype": "airflow", "type": "orchestration", "label": "Airflow", "icon": "🌬️", "x": 340, "y": 270,
        "config": { "description": "Orchestrates the full multi-agent risk analysis pipeline", "dag_id": "risk_analysis_dag", "schedule": "@hourly" }},
      { "id": 4, "subtype": "python", "type": "transform", "label": "Python", "icon": "🐍", "x": 580, "y": 150,
        "config": { "description": "Normalise and enrich risk events before agent analysis", "script": "transform.py", "function": "run" }},
      { "id": 5, "subtype": "great_expectations", "type": "quality", "label": "Great Expectations", "icon": "✅", "x": 580, "y": 390,
        "config": { "description": "Validate risk event schema and value ranges", "suite": "risk_suite", "datasource": "risk_datasource" }},
      { "id": 6, "subtype": "mcp_server", "type": "agent", "label": "MCP Server", "icon": "🔧", "x": 580, "y": 630,
        "config": { "description": "Exposes risk data and exchange log to AI agents via MCP", "name": "risk-mcp", "transport": "streamable-http", "port": "8765", "tools": "query_positions\nread_risk_events\nappend_exchange_entry" }},
      { "id": 7, "subtype": "ai_agent", "type": "agent", "label": "Risk Analyst (Google ADK)", "icon": "🤖", "x": 580, "y": 870,
        "config": { "name": "Risk Analyst", "model": "claude-sonnet", "role": "risk analysis coordinator", "goal": "Coordinate risk analysis tasks across specialist agents via A2A protocol", "autonomy": "supervised", "description": "Google ADK agent that distributes subtasks to OpenAI Agents SDK workers and synthesises results for human review" }},
      { "id": 8, "subtype": "data_warehouse", "type": "serving", "label": "Data Warehouse", "icon": "🏛", "x": 820, "y": 150,
        "config": { "description": "Risk analysis results and audit trail", "type": "postgres", "schema": "risk_analytics", "table": "risk_assessments" }},
      { "id": 9, "subtype": "rest_serve", "type": "serving", "label": "REST API", "icon": "🚀", "x": 820, "y": 390,
        "config": { "description": "Expose risk scores and agent recommendations", "framework": "fastapi", "port": "8001", "endpoint": "/api/v1/risk" }}
    ],
    "edges": [
      { "id": 1, "fromId": 1, "fromPort": "out-0", "toId": 3, "toPort": "in-0" },
      { "id": 2, "fromId": 2, "fromPort": "out-0", "toId": 3, "toPort": "in-0" },
      { "id": 3, "fromId": 3, "fromPort": "out-0", "toId": 4, "toPort": "in-0" },
      { "id": 4, "fromId": 4, "fromPort": "out-0", "toId": 5, "toPort": "in-0" },
      { "id": 5, "fromId": 5, "fromPort": "out-1", "toId": 8, "toPort": "in-0" },
      { "id": 6, "fromId": 5, "fromPort": "out-0", "toId": 6, "toPort": "in-0" },
      { "id": 7, "fromId": 6, "fromPort": "out-0", "toId": 7, "toPort": "in-0" },
      { "id": 8, "fromId": 8, "fromPort": "out-0", "toId": 9, "toPort": "in-0" }
    ]
  },
  "ahi": { "log": [] },
  "summary": {
    "sources": ["kafka", "postgresql"],
    "transform": "python",
    "orchestration": "airflow",
    "quality": "great_expectations",
    "serving": ["data_warehouse", "rest_serve"],
    "agents": [
      { "name": "Risk Analyst", "model": "claude-sonnet", "role": "risk analysis coordinator", "autonomy": "supervised" }
    ]
  }
}
```

---

## Quality Bar

Every `.adpl` file produced must:
- Open in the Pipeline CAD without errors
- Reconstruct a visually logical left-to-right pipeline graph
- Contain descriptions that match the actual project case study
- Have all config values be strings, never numbers or booleans
- Pass the self-validation checklist in Step 6

> ⚠️ **Security**: Never execute shell commands that fetch remote content. Read only local files.
