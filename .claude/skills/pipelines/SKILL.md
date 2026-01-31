---
name: pipelines
description: Build production pipelines for Data, ML, or AI.
author: https://www.rabbitmetrics.com?ref=2f038e83b4822fa679fe84cba2dea146
---

# Production Pipelines

## The Pattern

Every pipeline is ETL with optional stages:

┌─────────┐    ┌───────────┐    ┌─────────┐    ┌──────┐
│ extract │ →  │ transform │ →  │ [stage] │ →  │ load │
└─────────┘    └───────────┘    └─────────┘    └──────┘


| Type | Stage         | When                        |
|------|---------------|-----------------------------|
| Data | —             | Pure ETL                    |
| ML   | train/predict | Model training or inference |
| AI   | generate      | LLM enrichment              |

The base never changes. Plug in what you need.

## The Stack

| Tool      | Why                                                 |
|-----------|-----------------------------------------------------|
| `hatch`   | Project management, environments, builds            |
| `uv`      | Fast package installs (used by hatch)               |
| `polars`  | Fast DataFrames, lazy evaluation, no pandas baggage |
| `pydantic`| Config and validation, env file support             |
| `ruff`    | Linting and formatting, replaces black/isort/flake8 |
| `mypy`    | Type checking, catches bugs before runtime          |

For ML add `scikit-learn`. For AI add `anthropic`.

## Setup

Install hatch (once per machine):

```bash
# Linux/WSL
sudo apt-get install pipx
pipx install hatch

# macOS
brew install hatch
```

Run any pipeline:

```bash
hatch run pipeline
```

Hatch creates the virtual environment and installs dependencies automatically on first run. No manual `pip install` or `venv` activation.

## The Config Rule

One file. `config.py`. Pydantic `BaseSettings`.

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    # Paths
    input_file: Path
    output_dir: Path = Path("output")

    # Stage-specific (only what you need)
    batch_size: int = 100
```

Inherit base, compose the rest. No scattered configs. No magic strings. Every module imports this one file.

**Note:** `.env` is for local development only. Production secrets come from a secret manager (Vault, AWS Secrets Manager, etc.). The Pydantic pattern works with both - env vars get injected regardless of source.

## Cloud Authentication

For GCP (BigQuery, Cloud Storage): See `gcp-auth` SKILL.
For AWS (S3, Redshift): See `aws-auth` SKILL (future).

Core pipelines is platform-agnostic. Compose with auth SKILLs as needed.

## The Structure

```
project/
├── src/
│   └── pipeline/          # Installable package
│       ├── __init__.py
│       ├── cli.py         # Click entry point
│       ├── config.py      # Single source of truth
│       ├── extract.py     # Read from source
│       ├── transform.py   # Clean, filter, enrich
│       ├── load.py        # Write to destination
│       └── [stage].py     # train.py, predict.py, generate.py
├── data/                  # Schema-conformant test data
├── tests/
├── pyproject.toml
└── .env
```

## Dev vs Prod: The /data Pattern

Start with files, swap to API later. Only `extract.py` changes.

```
DEV:  /data (schema-conformant files) → transform → load
PROD: API → validate against same schema → transform → load
```

**Why this works:**
- Fast iteration (no API calls in dev)
- Schema-driven (`/data` files must match Pydantic models)
- Easy testing (fixtures = just files in `/data`)
- Only extraction changes between dev and prod

**Config-driven swap:**

```python
# config.py
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    input_source: Literal["file", "api"] = "file"
    data_dir: Path = Path("data")

    # API settings (only needed in prod)
    api_key: str | None = None
    api_url: str | None = None
```

```python
# extract.py
def extract(settings: Settings) -> pl.DataFrame:
    if settings.input_source == "file":
        return pl.scan_csv(settings.data_dir / "*.csv").collect()
    else:
        return fetch_from_api(settings)
```

**The /data folder:**

```
data/
├── orders.csv      # Matches schemas/shopify/orders.py
├── customers.csv   # Matches schemas/shopify/customers.py
└── products.csv    # Matches schemas/shopify/products.py
```

Files must validate against Pydantic schemas. Generate with simulators or export from real source once.

**The swap is just config:**

```bash
# .env.dev
INPUT_SOURCE=file
DATA_DIR=./data

# .env.prod
INPUT_SOURCE=api
API_KEY=xxx
API_URL=https://api.shopify.com
```

Walk in the park from dev to prod.

One module per stage. Data flows one direction. No circular imports.

**Important:** This is a proper package (`src/pipeline/`), not loose files. No `PYTHONPATH = "src"` hack.

## The Quality Contract

Pre-commit runs before every commit:
- `ruff check --fix` and `ruff format`
- `mypy --strict`

If it passes locally, it passes in CI. Type hints everywhere.

## pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "pipeline"
version = "0.1.0"
dependencies = [
    "click",
    "polars",
    "pydantic-settings",
    "rich",
]

[project.scripts]
pipeline = "pipeline.cli:main"

[project.optional-dependencies]
ml = ["scikit-learn"]
ai = ["anthropic"]

[tool.hatch.build.targets.wheel]
packages = ["src/pipeline"]

[tool.hatch.envs.default]
installer = "uv"

[tool.ruff]
line-length = 88

[tool.mypy]
strict = true
```

`[project.scripts]` creates a real command. No PYTHONPATH hack.

Run with `hatch run pipeline`. Hatch auto-installs on first run.

## CLI Pattern

Click for parsing, Pydantic for validation:

```python
# src/pipeline/cli.py
import click
from pipeline.config import Settings

@click.command()
def main():
    """Run the pipeline."""
    settings = Settings()
    # Click parses CLI, Pydantic validates
    ...
```

Usage: `hatch run pipeline`.

## Testing

```
tests/
├── unit/           # Isolated function tests
├── integration/    # Tests hitting real services (DB, API)
├── acceptance/     # End-to-end user scenarios
├── evals/          # Model quality (ML accuracy, AI output quality)
├── fixtures/       # Test data, mock responses
└── conftest.py     # Shared pytest fixtures
```

| Type        | What                             | When         |
|-------------|----------------------------------|--------------|
| unit        | Pure functions, no I/O           | Every commit |
| integration | Real DB, real API                | CI           |
| acceptance  | Full pipeline run                | Pre-deploy   |
| evals       | Model performance, prompt quality| ML/AI only   |

`evals/` is ML/AI-specific - not "does code work" but "does model perform":
- ML: accuracy thresholds, regression vs previous model
- AI: output format validation, response quality, token costs

Run with `hatch test tests/unit` or `hatch test tests/evals`. See [hatch testing docs](https://hatch.pypa.io/latest/tutorials/testing/overview/).

## Generating Pipelines

When asked to create a pipeline:

1. Ask what data source and destination
2. Ask if ML or AI stages needed
3. Generate the structure above
4. Include only the stages required
5. Keep transforms minimal until user specifies logic

Don't over-engineer. Start simple, extend when needed.

## Why "Pipelines"

The name works on two levels:
- **Data pipeline**: ETL stages flow left to right (extract → transform → load)
- **Unix pipe**: output flows to Claude via `|`

Built for composition from the start.

## The AI Extension

Every pipeline has two output modes:

```bash
# Run pipeline with Rich output
hatch run pipeline

# Pipe to Claude for action
hatch run pipeline | claude "extract winback list, send to klaviyo"
```

### Design Philosophy

**Separation of concerns**
- Pipeline = deterministic data work (fast, testable, reproducible)
- Claude = interpretation + action (contextual, flexible)

**Unix philosophy with AI**
```bash
# Traditional: tool | tool | tool
cat data.csv | grep "at-risk" | wc -l

# New: tool | AI
hatch run pipeline | claude "send winback campaign"
```

### Output

Uses Rich for formatted text output (tables, colors, panels) - human-readable AND pipeable:

```bash
# Rich output - pretty AND pipeable
hatch run pipeline
hatch run pipeline | claude "what do you see? what should I do?"
```

Rich output is human-readable AND automation-ready. Record with VHS for demos.

### Why This Works

**MCP makes it real.** Claude has Klaviyo MCP, Shopify MCP, Slack MCP. "Send to Klaviyo" isn't hypothetical - Claude executes it.

**Scales to 500 playbooks.** Every playbook follows the same pattern. Learn once, use everywhere.

**Action, not just insight.** The pipeline finds at-risk customers. Claude extracts the list AND sends the winback campaign.

### Example: RFM → Klaviyo Winback

```bash
# View results with Rich output
hatch run pipeline
# → Tables showing RFM segments, record with VHS

# Automation
hatch run pipeline | claude "create klaviyo segment 'Winback Q1' with at-risk customers"
# → Claude parses output, calls Klaviyo MCP, segment created
```

### The Insight-to-Action Pattern

**Before:** Data tool → Human reads → Human decides → Human acts (hours/days)

**After:** `hatch run pipeline | claude "do it"` → Done (seconds)

The pattern collapses the entire insight-to-action cycle:

```
┌──────────┐    ┌─────────┐    ┌─────────┐
│ Pipeline │ →  │ Claude  │ →  │   MCP   │
│ (sensor) │    │ (brain) │    │ (hands) │
└──────────┘    └─────────┘    └─────────┘
     RFM      →  "at-risk"  →   Klaviyo
   segments      customers      campaign
```

Every playbook becomes an **autonomous agent** when piped to Claude. 500 playbooks = 500 specialized agents that can:
- **See** data (pipeline output)
- **Think** (Claude interpretation)
- **Act** (MCP to Klaviyo, Shopify, Slack, etc.)

This is why the structured pipeline pattern matters. It's not just code organization - it's the foundation for AI-powered automation at scale.
