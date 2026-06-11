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

## Installed Plugins & Skills — When and How to Use Them

Claude Code has the following plugins installed (user scope). Use them **proactively** at the trigger points below — don't wait to be asked.

| Plugin | Trigger / When to use | What it does |
|---|---|---|
| **superpowers** | Start of any non-trivial feature, bugfix, or refactor (more than a one-line change) | Run `/brainstorm` → `/write-plan` → `/execute-plan`. Enforces TDD (red/green), YAGNI, DRY, and systematic debugging instead of ad-hoc edits. |
| **context7** | Before writing code against any library/framework/API you're not 100% certain about the current version's syntax for (Flask, MetaTrader5, FastMCP, requests, etc.) | Pulls live, version-accurate docs and code examples into context — avoids hallucinated APIs. |
| **github** | Any time you need to create/update issues, PRs, review diffs, or check CI status on `Adnovation01/*` repos | Native GitHub MCP — issues, PRs, reviews, repo search, full API access. |
| **commit-commands** | After every meaningful unit of work (per Commit & Push Rules below) | Use `/commit` and `/push` for consistent, well-formatted commits and immediate pushes. |
| **claude-md-management** | Whenever project structure, workflow, or commands change; periodically audit this file | Audits CLAUDE.md for staleness, captures session learnings, keeps this file accurate. |
| **code-review** | Before considering a feature/fix "done", especially before pushing to `main` or opening a PR | Multi-agent automated review with confidence-scored findings — catches bugs/security issues a single pass misses. |
| **code-simplifier** | Immediately after writing/editing code, as a final pass on just-modified files | Cleans up duplication and clarifies logic in recently changed code without altering behavior. |
| **feature-dev** | For larger features needing exploration + architecture + review (e.g. new endpoints, new MT5 worker logic) | Structured workflow: explore codebase → design architecture → implement → quality review. |
| **chrome-devtools-mcp** | Testing the Supertrend Flask dashboard (BTC paper trade dashboard, webhook UI, login pages) | Drives a real Chrome browser — inspect network requests, console errors, screenshots, performance traces. Always verify UI changes this way before reporting done. |
| **frontend-design** | Building or improving any HTML/CSS/JS UI (trading dashboards, status pages) | Generates polished, distinctive frontend code — avoid generic/bland AI-default styling. |
| **hookify** | When the user gives a recurring instruction ("always do X", "never do Y", "stop doing Z") | Convert it into a markdown-defined hook so the behavior is enforced automatically going forward, instead of relying on memory alone. |

### Recommended workflow for a typical task

1. **Plan** (superpowers `/brainstorm` + `/write-plan`) for anything beyond a trivial fix.
2. **Research** unfamiliar APIs via context7 before coding.
3. **Implement** (feature-dev for larger features; direct edits for small ones).
4. **Clean up** with code-simplifier on changed files.
5. **Verify**: run tests; for any web/dashboard change, drive it with chrome-devtools-mcp.
6. **Review** with code-review before finalizing.
7. **Commit & push** via commit-commands (`/commit`, `/push`) — per the mandatory rules below.
8. **Update CLAUDE.md** via claude-md-management if anything structural changed, then commit that too.
9. If the user states a standing preference/rule mid-task, capture it with **hookify** so it's enforced automatically next time.

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
