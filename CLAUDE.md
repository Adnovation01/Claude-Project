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

## Commit Style

- Use present tense, imperative mood: `Add feature`, `Fix bug`, `Update config`
- Subject line under 72 characters
- Always include `Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>`
