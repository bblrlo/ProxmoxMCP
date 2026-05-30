# ProxmoxMCP-Plus — Agent Guide

## Branch Rule
- Never commit to `main`. All modifications in separate branches only.

## Project Identity
- PyPI: `proxmox-mcp-plus`, requires Python >=3.11
- Build backend: `hatchling`; source in `src/proxmox_mcp/`
- Package entrypoint: `proxmox_mcp.server:main` (aliased as `proxmox-mcp` and `proxmox-mcp-plus`)
- Bootstrap entrypoint: `main.py` (inserts `src/` into `sys.path`, then calls `ProxmoxMCPServer`)
- Docker entrypoint: `proxmox_mcp.docker_entrypoint` (selects OpenAPI vs MCP HTTP mode via `PROXMOX_MCP_MODE`)

## Runtime Modes
- **OpenAPI** (default in Docker): port 8811, requires `PROXMOX_API_KEY` env var. Local unauth: `PROXMOX_ALLOW_NO_AUTH=true`
- **MCP HTTP**: set `PROXMOX_MCP_MODE=mcp-http`, port 8000, transport `STREAMABLE_HTTP`
- **Stdio**: used directly by `uvx proxmox-mcp-plus` or `pip install proxmox-mcp-plus && proxmox-mcp-plus`

## Dev Setup
```bash
uv venv && uv pip install -e ".[dev]"
cp proxmox-config/config.example.json proxmox-config/config.json
```
`conftest.py` automatically inserts `src/` into `sys.path` so tests always use the repo source.

## All Verification Commands (CI order)
```bash
pytest -q --cov=proxmox_mcp --cov-report=term-missing --cov-fail-under=75
ruff check .
mypy src --ignore-missing-imports
python -m build && twine check dist/*
pip-audit -r requirements.txt
jq empty manifest.json
```

## Config Quirks
- `verify_ssl: false` is blocked unless `security.dev_mode: true` — enforced by the config loader.
- Env vars override JSON file config for most fields. Key env vars: `PROXMOX_HOST`, `PROXMOX_USER`, `PROXMOX_TOKEN_NAME`, `PROXMOX_TOKEN_VALUE`, `MCP_HOST`, `MCP_PORT`, `MCP_TRANSPORT`, `MCP_DNS_REBINDING_PROTECTION`, `MCP_ALLOWED_HOSTS`, `MCP_ALLOWED_ORIGINS`, `PROXMOX_JOBS_SQLITE_PATH`, `COMMAND_POLICY_MODE`.
- MCP transport `STREAMABLE_HTTP` normalizes to `STREAMABLE` internally.

## SSH Is Optional
- `execute_container_command` and `update_container_ssh_keys` are only registered when the config contains an `ssh` block. Tests verify absence vs presence.

## Job Store
- SQLite-backed, default `proxmox-jobs.sqlite3` in CWD. Configurable via `jobs.sqlite_path` or `PROXMOX_JOBS_SQLITE_PATH` env var.

## API Tunnel
- `api_tunnel` config section can tunnel ProxmoxAPI through SSH (tested in `test_server.py`).

## Tool Architecture
- Registration: `src/proxmox_mcp/services/builtin_tool_plugins.py`
- Implementations: `src/proxmox_mcp/tools/` (one file per domain)
- Descriptions + typed schemas: `src/proxmox_mcp/tools/definitions.py`
- Mutations that call async Proxmox tasks must return a stable `job_id` + store retry recipe.

## Release Metadata Alignment
Version must match across all five files:
- `pyproject.toml` → `setup.py` → `src/proxmox_mcp/__init__.py` → `manifest.json` → `server.json`
(`test_release_metadata.py` enforces this.)

## Live E2E
- Script: `tests/scripts/run_real_e2e.py`
- Prefers `proxmox-config/config.live.json`, falls back to `PROXMOX_MCP_E2E_CONFIG` env var.
- Refuses to run against the default `proxmox-config/config.json` to avoid accidental live damage.
