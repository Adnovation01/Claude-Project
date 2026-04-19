# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment

- **OS:** Windows Server 2025, running inside Git Bash / bash shell
- **IDE:** Antigravity (VS Code fork)
- **Shell note:** Always use Unix shell syntax in Bash tool calls. PowerShell is used in the terminal by the user.

## PATH Requirements

Git and GitHub CLI are installed but may not be in the bash PATH. Prefix commands with:

```bash
export PATH="$PATH:/c/Program Files/GitHub CLI:/c/Program Files/Git/cmd"
```

Or run `git`/`gh` commands via PowerShell if bash PATH is missing them.

## Git & GitHub Workflow

Every project must:
1. Be initialized with `git init`
2. Have a matching GitHub repo created via `gh repo create`
3. Be linked to the remote: `git remote add origin https://github.com/Adnovation01/<repo-name>.git`
4. Receive clean, descriptive commits and be pushed to GitHub after every meaningful change

**GitHub account:** Adnovation01  
**Git identity:** adnovation / adnovationllp@gmail.com  
**Current repo:** https://github.com/Adnovation01/Claude-Project

## Commit & Push Rules (MANDATORY)

These rules apply to **every project and task — current and future — without exception**:

- **Commit and push after every meaningful unit of work**: new file, feature, fix, config change, or documentation update. Never leave work uncommitted at the end of a session.
- **Stage only relevant files**: never use `git add .` blindly — always add specific files to avoid committing system files, secrets, or unrelated changes.
- **Push immediately after committing**: `git push origin main` every time. Do not accumulate unpushed commits.
- **Keep CLAUDE.md current**: update this file whenever the project structure, workflow, commands, or context changes — then commit and push the update as its own clean commit.
- **Never skip this process**: not for small edits, not for "temporary" changes, not for quick experiments. Every change that matters gets committed and pushed.

## Commit Style

- Use present tense, imperative mood: `Add feature`, `Fix bug`, `Update config`
- Subject line under 72 characters
- Always include `Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>`

## Key Projects

### Supertrend Algo (`Downloads/supertrend-algo-exness-master/`)

A Flask web app bridging TradingView webhook alerts → MetaTrader 5 trade execution. Architecture:

```
TradingView Pine Script alert → POST /tvwebhook → Flask app → MT5 worker process (per user) → broker
```

- Multi-user: each user has their own MT5 background worker process
- Entry point: `main.py`
- Web routes in `web/`, utilities in `utils/`
- **Requires MT5 terminal running on the same Windows machine** (MetaTrader5 Python package is Windows-only)

Run the app:
```bat
pip install -r requirements.txt
python main.py
```

Or use the provided scripts: `execute_main.bat` (Windows) / `execute_main.sh` (bash).

The app is deployed to a VPS (see `Documents/Trading Terminal.txt` for endpoints).

### MCP MetaTrader 5 Server (`Downloads/mcp-metatrader5-server/`)

A FastMCP server exposing MT5 trading functions to AI assistants via the Model Context Protocol.

- Source: `src/mcp_mt5/main.py`
- Uses `uv` for dependency management (`pyproject.toml`, `uv.lock`)
- Python 3.11+ required

Run (stdio mode for MCP clients):
```bash
uv run mt5mcp
```

Run (HTTP mode for development — set `MT5_MCP_TRANSPORT=http` in `.env`):
```bash
uv run mt5mcp
```

Tests:
```bash
uv run pytest
```

Install into Claude Code:
```bash
uv run fastmcp install claude-code src/mcp_mt5/main.py
```
