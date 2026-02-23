# StarkBot Hackathon Samples

Sample **skill** and **module** for [StarkBot](https://github.com/anthropics/starkbot). Use these as templates to build your own extensions.

## What's Inside

```
skills/
  linear/              # Sample Skill — Linear issue tracker integration
    linear.md          # Skill manifest + prompt (YAML frontmatter + Markdown)
    scripts/
      linear.sh        # Bash script called by the skill

modules/
  discord_tipping/     # Sample Module — Discord user profile & wallet linking
    module.toml        # Module manifest (TOML)
    service.py         # Python microservice (Flask)
  starkbot_sdk/        # Shared SDK used by Python modules
```

---

## Skills vs Modules

| | Skill | Module |
|---|---|---|
| **What it is** | A prompt template + optional scripts | A standalone microservice |
| **Format** | Markdown file with YAML frontmatter | `module.toml` manifest + service code |
| **Runtime** | Runs inside the bot's LLM context | Runs as a separate HTTP process |
| **State** | Stateless (talks to external APIs) | Can have its own database, dashboard, etc. |
| **Best for** | API integrations, workflows, automations | Stateful services, dashboards, complex tools |

---

## Sample Skill: Linear

The `linear` skill lets the bot query and manage Linear issues via GraphQL.

### Skill Structure

A skill is a directory containing a Markdown file with YAML frontmatter:

```yaml
---
name: linear
description: "Query and manage Linear issues, projects, and team workflows."
version: 1.0.0
author: starkbot
requires_tools: [run_skill_script]
requires_binaries: [curl, jq]
scripts: [linear.sh]
requires_api_keys:
  LINEAR_API_KEY:
    description: "Linear API key"
    secret: true
tags: [linear, project-management]
arguments:
  action:
    description: "Action to perform"
    required: false
---

# Prompt content goes here...
```

**Key fields:**

- `name` — Unique skill identifier
- `requires_tools` — Bot tools the skill needs (e.g. `run_skill_script` to execute bundled scripts)
- `requires_binaries` — System binaries the script depends on
- `scripts` — Script files in the `scripts/` subdirectory
- `requires_api_keys` — Environment variables needed (prompted on first use)
- `arguments` — Arguments the LLM can pass when invoking the skill
- `tags` — Used for search and discovery

The Markdown body after the frontmatter is the prompt template injected into the LLM context when the skill is active.

### Installing the Skill

Copy the `linear/` directory into your StarkBot skills folder:

```bash
cp -r skills/linear ~/.starkbot/skills/
```

Or install via the bot:

```
> install the linear skill
```

---

## Sample Module: Discord Tipping

The `discord_tipping` module runs a Python Flask microservice that manages Discord user profiles and linked wallet addresses.

### Module Structure

A module is a directory with a `module.toml` manifest and service code:

```toml
[module]
name = "discord_tipping"
version = "1.0.0"
description = "Discord user profile & wallet address linking for tipping"
author = "starkbot"

[service]
command = "uv run service.py"       # How to start the service
default_port = 9101
port_env_var = "DISCORD_TIPPING_PORT"
has_dashboard = true
health_endpoint = "/rpc/status"
backup_endpoint = "/rpc/backup/export"
restore_endpoint = "/rpc/backup/restore"

[[tools]]                           # Tools exposed to the bot
name = "discord_tipping_profile"
description = "Manage Discord user profiles and linked wallet addresses"
group = "social"
rpc_method = "POST"
rpc_endpoint = "/rpc/profile"

[tools.parameters.action]
type = "string"
description = "Action to perform"
required = true
enum = ["get_or_create", "get", "register", "unregister", "list", "stats"]
```

**Key sections:**

- `[module]` — Name, version, description, author
- `[service]` — How to run the service, port config, health/backup endpoints
- `[[tools]]` — Tools the module exposes to the bot (become callable LLM tools)
- `[tools.parameters.*]` — Parameter schemas for each tool

The service itself is a standard HTTP server (Flask, Express, etc.) that implements the RPC endpoints declared in the manifest.

### RPC Protocol

All module endpoints use a standard JSON envelope:

```json
// Success
{"success": true, "data": { ... }}

// Error
{"success": false, "error": "description"}
```

The `/rpc/status` endpoint is required and used for health checks.

### Installing the Module

Copy the `discord_tipping/` directory (and `starkbot_sdk/` if using Python) into your StarkBot modules folder:

```bash
cp -r modules/discord_tipping ~/.starkbot/modules/
cp -r modules/starkbot_sdk ~/.starkbot/modules/
```

Or install via the bot:

```
> install the discord_tipping module
```

---

## Building Your Own

### Creating a Skill

1. Create a directory: `my_skill/`
2. Add `my_skill.md` with YAML frontmatter (see the linear example)
3. Optionally add scripts in `my_skill/scripts/`
4. Copy to `~/.starkbot/skills/`

### Creating a Module

1. Create a directory: `my_module/`
2. Add `module.toml` with your manifest (see the discord_tipping example)
3. Implement your service with at least `/rpc/status`
4. Define `[[tools]]` in the manifest for any tools you want the bot to use
5. Copy to `~/.starkbot/modules/`

### Python Module SDK

The included `starkbot_sdk` package provides helpers for Python modules:

```python
from starkbot_sdk import create_app, success, error

app = create_app("my_module")

@app.route("/rpc/my_tool", methods=["POST"])
def my_tool():
    data = request.get_json()
    # ... do work ...
    return success({"result": "done"})
```

Install it locally for development:

```bash
cd modules/starkbot_sdk
pip install -e .
```

---

## Reference

| Skill Field | Description |
|---|---|
| `name` | Unique identifier |
| `description` | Human-readable description |
| `version` | Semver version string |
| `author` | Author name |
| `requires_tools` | List of bot tools needed |
| `requires_binaries` | System binaries needed |
| `scripts` | Script files in `scripts/` dir |
| `requires_api_keys` | Env vars with descriptions |
| `arguments` | Named arguments for the skill |
| `tags` | Search/discovery tags |

| Module Manifest Section | Description |
|---|---|
| `[module]` | Name, version, description, author |
| `[service]` | Command, port, endpoints |
| `[service.env_vars]` | Required/optional env vars |
| `[[tools]]` | Tool definitions exposed to bot |
| `[tools.parameters.*]` | Parameter schemas |
| `[[ext_endpoints]]` | Public HTTP endpoints (optional) |

---

## How to Install Skills & Modules into StarkBot

There are multiple ways to install. **No zipping required** — plain directories work fine.

### Installing a Skill

**Option 1: Copy the directory** (simplest)

```bash
cp -r skills/linear ~/.starkbot/skills/
# Restart StarkBot — it auto-discovers skills on startup
```

**Option 2: Install from raw markdown** (via the bot)

Tell StarkBot to install it and paste the `.md` content:

```
> install this skill: <paste contents of linear.md>
```

The bot's `manage_skills` tool accepts raw markdown via the `markdown` parameter.

**Option 3: Install from URL** (once hosted, e.g. on GitHub)

```
> install skill from https://raw.githubusercontent.com/you/repo/main/skills/linear/linear.md
```

The bot's `manage_skills` tool accepts a `url` parameter pointing to a raw `.md` file.

**Option 4: ZIP upload** (via the StarkBot UI)

Upload a `.zip` file through `POST /api/skills/upload`. The ZIP must contain a `SKILL.md` (or `{name}.md`) at the root or one level deep, plus optional `scripts/` and `abis/` directories. Max 10MB.

### Installing a Module

**Option 1: Copy the directory** (simplest)

```bash
# The SDK is needed if your module uses it
cp -r modules/starkbot_sdk ~/.starkbot/modules/
cp -r modules/discord_tipping ~/.starkbot/modules/
# Restart StarkBot — it auto-discovers modules on startup
```

**Option 2: ZIP upload** (via bot or UI)

Tell the bot:

```
> import module from zip at /path/to/discord_tipping.zip
```

Or upload through `POST /api/modules/upload`. The ZIP must contain a `module.toml` at the root or one level deep. Max 10MB.

**Option 3: Install from StarkHub** (remote registry)

```
> install remote module @username/discord-tipping
```

Downloads and verifies (SHA-256) from StarkHub automatically.

### Summary

| Method | Skills | Modules |
|---|---|---|
| **Copy directory to `~/.starkbot/`** | `~/.starkbot/skills/{name}/` | `~/.starkbot/modules/{name}/` |
| **Raw markdown / paste** | Yes | No |
| **URL to `.md` file** | Yes | No |
| **ZIP upload** | Yes (HTTP) | Yes (HTTP or bot tool) |
| **StarkHub remote** | No | Yes (`@user/slug`) |

The **directory copy** approach is the easiest for hackathon use — no packaging step needed. Just clone this repo, copy the folders, and restart StarkBot.
