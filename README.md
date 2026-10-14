# semantic_modeling_demo

SML semantic models built by an LLM agent (Claude Code + the AtScale MCP) straight from a warehouse
schema. The demo: point the agent at a database and have it build, validate, commit, push and deploy a
semantic model you can query.

## Current model

**Solar Panel Operations** - solar plant fleet operations, from a local Postgres database.

| Item | Value |
|---|---|
| AtScale connection | `bird_solar_panel` |
| Database / schema | `solar_panel` / `public` (Postgres on `localhost:5433`) |
| Grain | one row per plant performance snapshot |
| Directory | `solar_panel_operations/` (created by the agent) |
| Catalog | `comodel_demo` |

Context source is the warehouse only. There is no Atlan or other data catalog, and the agent does not
read any existing solar panel model, so descriptions, joins and metrics all come from what it finds in
the database.

## Repository layout

The root holds `catalog.yml`, this README and one directory per model. Each model directory holds its
own `connections/`, `datasets/`, `dimensions/`, `metrics/`, `calculations/` and `models/`. `sml-cli`
recurses, so one `catalog.yml` covers every model beneath it. `unique_name` and `label` are unique
repo-wide, so qualify names in a new model (`Snapshot Year`, not `Year`).

```
catalog.yml
README.md
solar_panel_operations/{connections,datasets,dimensions,metrics,calculations,models}/
```

## Model Memory (`ai_context`)

`catalog.yml` carries an `ai_context` array that tells the agent where context lives and what rules
bind it before it touches the warehouse. Each `models/*.yml` can carry its own `ai_context` recording
what that model was derived from. Entries look like:

```yaml
ai_context:
  - source_type: mcp | git | document | url | report | warehouse | catalog | other
    source: <locator - URL, path, asset name>
    instruction: <what to fetch from it and how to use it while modeling>
```

Declared today: the repo layout convention, the Postgres warehouse (the only data source), and the
build and quality rules.

## Prerequisites

- Postgres with the `solar_panel` database on `localhost:5433`, and the AtScale connection
  `bird_solar_panel` already created and reachable by the engine.
- The AtScale MCP server connected to Claude Code (`atscale-local`).
- `sml-cli` on PATH.
- This repo registered in AtScale as a `catalog` repository, with the URL matching `origin` exactly.
- A `.env` file (gitignored) with `ATSCALE_API_URL` (the SML gateway, port 3001, base URL only) and
  `ATSCALE_API_TOKEN` (a token minted on that same instance).
- Push access to `origin` (a fine-grained PAT or `gh auth login`, limited to this repo), and a git
  `user.name` / `user.email`.

## Validate / deploy

`atscale-deploy` is **git-based**: it finds the AtScale repository by this project's `origin` remote
and AtScale then pulls from GitHub, so a local commit is not enough. The commit must be pushed before
deploying, or the old model deploys and looks like success.

```bash
sml-cli validate .
git add -A && git commit -m "..." && git push origin main
set -a && . ./.env && set +a
sml-cli atscale-deploy . --catalog-name=comodel_demo
```

Check the schema the deploy prints and confirm `list_models` returns the new model. A working
`run_query` alone does not prove a healthy deploy.

After every deploy:

1. Confirm `list_models` and `explore_columns` both succeed for the model.
2. Run at least one live `run_query` per dimension. Derived-dataset SQL is not parsed until the first
   query, so a syntax error passes validate and deploy and fails here.
3. Validate every `metric_calc` expression with `validate_mdx_expression`.

## Authoring notes

- A **`description:` value must be quoted**, plain ASCII. An unquoted scalar containing a colon and
  space is read as a nested mapping and the object loses its description or fails to parse.
- A deployed column takes its name from **`unique_name`**, not `label`. Keep `unique_name` label-based.
- A model's **`dimensions:` list holds degenerate dimensions only**. Everything else goes through a
  model `relationships:` entry.
- **Qualify every table in a derived (`sql:`) dataset** (`public.plant_record`). Physical `table:`
  datasets take the bare table name from the connection.
- Most all-time running-total MDX is rejected at publish, not at validate. Check any such measure
  against the period-by-period series after deploying.
