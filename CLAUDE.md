# Repo conventions for Claude Code

> **Generic process rules live in `~/.claude/CLAUDE.md`** (auto-loaded by Claude Code from the private [claude-md-global](https://github.com/szhygulin/claude-md-global) repo). The rules below are project-specific.

## This skill ships zero buildable artifacts

`vaultpilot-setup-skill` is **doc-only by design**. Do not add:

- `bin/` executables (Node CLIs, shell scripts, etc.)
- `package.json` / `package-lock.json` / runtime dependencies
- Any compiled or buildable artifact

The trust root is the user's own `git clone` of this repo. A doc-only tree is auditable line-by-line; a Node bin pulls in a transitive dependency graph (npm supply chain), which is exactly the kind of surface a skill is designed to *defend against*. Every dep added here erodes the "the skill cannot itself be the attack vector" property.

If a verification or setup helper is genuinely needed, it lives in `vaultpilot-mcp` (or a separate purpose-built package the user opts into), not here. Closing the agent-side gap is done via prose guidance in `SKILL.md`, not by shipping code.

## What this repo *does* contain

`SKILL.md` (the agent-side setup flow), `README.md`, `LICENSE`. That's the whole shape — keep PRs within it.
