# VaultPilot Setup — agent-guided `/setup` skill for `vaultpilot-mcp`

Conversational onboarding for [`vaultpilot-mcp`](https://github.com/szhygulin/vaultpilot-mcp).
When the user types `/setup` (or asks "how do I get started"), this skill
tells the agent exactly how to guide them through the minimum required
configuration — zero JSON editing, no preemptive key collection.

## What it replaces

Before this skill, a new user had to:
1. Hunt for the right `claude_desktop_config.json` path on their OS
2. Edit it without breaking the surrounding JSON
3. Run a terminal readline wizard prompting for 4–6 API keys upfront,
   whether or not they'd use them
4. Cross-reference Ledger Live pairing docs for the current app version
5. Decide between Helius / Alchemy / Infura with no context

With this skill installed, the agent:

- Calls `get_vaultpilot_config_status` to see what's already configured
- Asks one question to classify the use case (read-only / EVM signing /
  Solana signing / TRON signing)
- Collects only the keys that use case actually needs
- Validates each pasted key via a cheap read-only tool call
- Ends with a concrete working example (not "you're all set")

## Install

```bash
git clone https://github.com/szhygulin/vaultpilot-setup-skill.git \
  ~/.claude/skills/vaultpilot-setup
```

Restart Claude Code so the skill is discovered. When the user types
`/setup` or asks a first-run question, the agent will follow the flow
in [`SKILL.md`](./SKILL.md).

## Update

```bash
cd ~/.claude/skills/vaultpilot-setup
git pull --ff-only
```

## Relationship to `vaultpilot-preflight`

These are two different skills with two different jobs:

- **`vaultpilot-preflight`** — invariant: every state-changing tx gets
  bytes-verified and hash-matched before signing, even if the MCP omits
  its own instructions. Loaded on every signing flow.
- **`vaultpilot-setup`** (this repo) — invariant: first-run users never
  have to edit JSON config files by hand or guess which keys they need.
  Loaded only on `/setup` and setup-adjacent questions.

Both skills are **intentionally independent repos**. An attacker who
compromises the MCP's release pipeline cannot weaken either — their trust
roots are the user's own clones of these repositories.

## License

MIT. See [`LICENSE`](./LICENSE).
