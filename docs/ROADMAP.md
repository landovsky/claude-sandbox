# Roadmap: Tech-Agnostic Claude Sandbox

## Current State

The tool is **Rails-first but not Rails-only**. Node.js is already a first-class citizen in service detection and dependency installation. The infrastructure layer (S3 caching, secrets management, git safety, Kubernetes orchestration) is fully generic.

### What's Ruby/Rails-Specific Today

- **Dockerfile** installs Ruby via `ruby-install`, plus `gem install bundler`
- **entrypoint.sh** runs `bundle install`, `rails db:prepare` unconditionally for Rails projects
- **`RAILS_ENV: development`** hardcoded in `docker-compose.yml` and K8s YAML
- **`ruby-versions.yaml`** is required — build fails without it
- **Image build/push system** is organized around Ruby versions
- **`auto_detect_ruby_version()`** runs for every `local` and `remote` command

### What's Already Language-Agnostic

- `lib/cache-manager.sh` — has stubs for `pip`, `cargo`, `gomod`
- Service detection — scans both `Gemfile` and `package.json`
- S3 caching, SOPS secrets, git safety, K8s orchestration, Telegram notifications
- `docs/EXTENDING.md` documents how to add new ecosystems

## Competitive Landscape

None of the existing AI agent sandbox tools cover the full loop of *detect services → provision them → cache dependencies → manage secrets → run agent → notify*. They all stop at "here's an isolated box."

| Tool | What It Does | What It Lacks |
|------|-------------|---------------|
| [Docker Sandboxes](https://docs.docker.com/ai/sandboxes) | Official Docker microVM isolation, works with Claude Code | No service orchestration, no auto-detection, no dep caching, no secrets |
| [Daytona](https://www.daytona.io/) | 27ms sandbox spin-up, API-first, agent-agnostic | No opinion on app services (Postgres, Redis, etc.) |
| [E2B](https://www.docker.com/blog/docker-e2b-building-the-future-of-trusted-ai/) | Cloud sandboxes for AI agents | Focused on code execution, not full-stack app environments |
| [Agent Sandbox (K8s)](https://www.infoq.com/news/2025/12/agent-sandbox-kubernetes/) | Open-source K8s controller for isolated pods | No service detection or dependency management |
| [claudebox](https://github.com/RchGrav/claudebox) | Docker dev environment for Claude | No service auto-detection or K8s support |

**Our unique value: the application-aware orchestration layer** — detecting what services an app needs, provisioning them, caching dependencies, managing secrets, running the agent, and notifying on completion.

**What's redundant:** The sandboxing/isolation itself. Docker Sandboxes or Daytona could replace the Dockerfile + Docker Compose layer and provide better isolation (microVMs) for free.

## Roadmap to Tech-Agnostic

### Phase 1: Formalize the Stack Plugin Interface

Each language/framework becomes a plugin (e.g. `plugins/ruby.sh`, `plugins/python.sh`) that implements:

| Function | Purpose | Ruby Example | Python Example |
|----------|---------|-------------|----------------|
| `detect` | Does this repo use me? | Check for `Gemfile` | Check for `requirements.txt`, `pyproject.toml` |
| `runtime` | What runtime is needed? | Ruby 3.4 | Python 3.12 |
| `deps_install` | Install dependencies | `bundle install` | `pip install -r requirements.txt` |
| `deps_cache` | Lockfile + target dir | `Gemfile.lock` → `vendor/bundle` | `requirements.txt` → `.venv` |
| `db_setup` | DB migration step | `rails db:prepare` | `alembic upgrade head` |
| `services` | What infra is needed? | Scan Gemfile for `pg`, `redis` | Scan requirements for `psycopg2`, `redis` |
| `env_vars` | Framework env vars | `RAILS_ENV=development` | `FLASK_ENV=development` |

### Phase 2: Extract Ruby/Rails Into the First Plugin

- Move all Ruby-specific logic out of `entrypoint.sh` and `bin/claude-sandbox` into `plugins/ruby.sh`
- The entrypoint becomes a loop: detect which plugins match → run them
- Ruby behavior stays identical — this is a refactor, not a feature change

### Phase 3: Replace the Monolithic Dockerfile

Options (pick one):

1. **Base image + runtime installation at boot** — slower but flexible, S3 cache mitigates startup time
2. **Multi-stage builds** with optional language layers
3. **Multiple tagged images per stack** — extend what we already do for Ruby versions

### Phase 4: Generalize Configuration

- `ruby-versions.yaml` → `stack-versions.yaml`:
  ```yaml
  ruby:
    versions: { "3.3": "3.3.6", "3.4": "3.4.7" }
    default: "3.4"
  python:
    versions: { "3.11": "3.11.9", "3.12": "3.12.4" }
    default: "3.12"
  ```
- Remove hardcoded `RAILS_ENV` — let each plugin set its own env vars
- Extend `detect-services.sh` to check `requirements.txt` for `psycopg2`, `go.mod` for `pgx`, etc.

### Phase 5: Add New Stack Plugins

Priority order based on ecosystem size and AI agent use cases:

1. **Python** — `requirements.txt`/`pyproject.toml`, pip/uv, alembic/django migrations
2. **Go** — `go.mod`, `go install`, goose migrations
3. **Rust** — `Cargo.toml`, `cargo build`, diesel migrations
4. **Elixir** — `mix.exs`, `mix deps.get`, `mix ecto.migrate`

## Strategic Consideration

Before investing in full generalization, consider: **should the isolation layer be replaced with Docker Sandboxes or Daytona**, focusing this tool purely on the orchestration/detection/caching layer that none of them provide? This would:

- Drop Dockerfile maintenance burden
- Get microVM isolation for free
- Keep the part that's actually unique (application-aware orchestration)
