---
name: vaultpilot-setup
description: Use when the user invokes `/setup`, asks "how do I get started with VaultPilot", is hitting a first-run error that looks like a missing API key / RPC / Ledger pairing, or asks which keys they still need. Guides them through the minimum required setup conversationally — read-only portfolio reads need zero keys; signing needs chain-specific pairing and, for some chains, one API key.
---

<!-- VAULTPILOT_SETUP_SKILL_v1 -->

# VaultPilot setup — conversational configuration guide

You are helping a user finish configuring `vaultpilot-mcp` so they can
actually use it. **Do not ask them to paste or edit their
`claude_desktop_config.json` / `.claude.json` / `mcp.json` file by hand.**
The server's installer already wrote that file for them. Your job is to
fill in the parts the installer cannot auto-resolve: API keys (where
needed), Ledger pairings, and in-chat validation.

## Step 1 — always start by snapshotting current state

Call `mcp__vaultpilot-mcp__get_vaultpilot_config_status` with no arguments.
This is read-only, never leaks secret values, and returns:

- `configPath` / `configFileExists` — where the installer wrote the config
- `serverVersion` — so you can match advice against the user's version
- `rpc[chain].source` — one of `env-var` / `provider-key` / `custom-url` /
  `public-fallback` per chain. `public-fallback` means PublicNode defaults
  are in use — fine for read-only, rate-limited for heavy use.
- `apiKeys[service].set` + `source` — `etherscan`, `oneInch`, `tronGrid`,
  `walletConnectProjectId`. Boolean + source only; values never leak.
- `pairings.walletConnect.sessionTopicSuffix` (last 8 chars if paired)
- `pairings.solana.count`, `pairings.tron.count` — paired-account counts
- `preflightSkill.installed` — whether `~/.claude/skills/vaultpilot-preflight/`
  is on disk. If false, flag it; without that skill the agent-side
  bytes-verification invariants stop firing.

Don't list everything you found back at the user. Pick what's relevant
to their ask.

## Step 2 — classify the use case, then ask only for what's needed

Most questions from a new user fall into one of four buckets. Ask one
short question to disambiguate if you can't tell, then branch:

| Use case | Needs |
|---|---|
| **Read balances / portfolio / history** (no signing) | Nothing. PublicNode RPCs + anonymous TronGrid ship by default. Skip to Step 5 — offer a sample tool call like `get_portfolio_summary` to prove it works. |
| **Sign EVM txs** (Ethereum / Arbitrum / Polygon / Base / Optimism) | WalletConnect project ID + `pair_ledger_live` once. No Etherscan / 1inch key required until they hit a tool that needs one. |
| **Sign Solana txs** | Optional but recommended: a Helius RPC URL (public RPC is rate-limited and drops writes under load). Required: `pair_ledger_solana`. |
| **Sign TRON txs** | Optional TronGrid key for headroom. Required: `pair_ledger_tron`. |

If they say "all of it" / "everything", proceed in that order: EVM → Solana
→ TRON. Don't front-load the TRON step on a user who's asking about Solana.

## Step 3 — for each needed piece, drive the exact shortest path

### 3a. WalletConnect project ID (EVM signing)

If `apiKeys.walletConnectProjectId.set === false`:

1. Open `https://cloud.reown.com/app/new-project` for them (use your
   client's URL-open capability; do not just print the link raw — use a
   markdown hyperlink if printing).
2. Tell them the minimum form: project name = anything (e.g. "VaultPilot
   personal"), project type = Wallet, Advanced → leave defaults.
3. After they create it, they'll see a **Project ID** (36-char UUID). Ask
   them to paste it back in chat.
4. Save it: instruct them to run the server's setup wizard with
   `WALLETCONNECT_PROJECT_ID=<uuid> npx vaultpilot-mcp-setup --save`
   (or whatever their install method was — the wizard path is documented
   in `get_vaultpilot_config_status`'s `configPath`).
5. After they confirm saved, call
   `mcp__vaultpilot-mcp__pair_ledger_live` and relay the QR URI + mobile-
   app path (Ledger Live → Connect).

### 3b. Solana signing

Required: `mcp__vaultpilot-mcp__pair_ledger_solana` (USB, blind-sign
enabled). Before calling it, call
`mcp__vaultpilot-mcp__get_ledger_device_info` so you can say "I see your
Bitcoin app is open — switch to Solana" instead of a generic
"open the Solana app" hint.

Recommended: Helius RPC for reliability.

1. If `rpc.solana.source === "public-fallback"`, tell them: "Solana sends
   and fast portfolio reads work better with a free Helius key. Takes ~30
   seconds. Want me to walk you through?". If yes:
2. Open `https://dashboard.helius.dev/api-keys`. Free tier is fine.
3. They copy the API key. You add it as an RPC URL:
   `https://mainnet.helius-rpc.com/?api-key=<key>`. Save via the wizard
   (`SOLANA_RPC_URL=<url> npx vaultpilot-mcp-setup --save`).
4. Re-call `get_vaultpilot_config_status` and confirm `rpc.solana.source`
   is no longer `public-fallback`.

The first Solana send auto-bundles a one-time nonce setup (~0.00144 SOL).
They do NOT need to run `prepare_solana_nonce_init` first.

### 3c. TRON signing

Required: `mcp__vaultpilot-mcp__pair_ledger_tron`.

Optional: TronGrid key. Free tier at
`https://www.trongrid.io/dashboard/apikeys`. If they hit rate-limit errors
("503" / "rate limit"), that's the fix.

## Step 4 — validate every key the user pastes

Do NOT just save the key and move on. After saving, fire a cheap
read-only tool to prove it works:

- WalletConnect ID → there's no "ping" — validation happens on
  `pair_ledger_live`. If the pair fails with `project id not found`, the
  ID was wrong.
- Helius → `mcp__vaultpilot-mcp__get_portfolio_summary` with
  `solanaAddress` set. If it returns 401 / 403, the key is malformed or
  unpaid-tier-overage.
- TronGrid → `mcp__vaultpilot-mcp__get_tron_staking` or a `get_token_balance`
  with `chain: "tron"`. 401 → bad key; 503 → rate limit (rare for free tier).
- Etherscan → validated the first time they hit `get_tx_verification` or
  `get_transaction_history` on EVM.

On 401: tell them "the key I got back returned 401 — most likely a copy
error (missing a character) or pasted the project name instead of the
actual key." Offer to re-open the dashboard.

On 429 / 503: tell them it's rate limiting — the key itself is fine, they
just hit the free tier ceiling. Suggest waiting a minute and retrying,
not re-creating the key.

## Step 5 — prove it works

End every setup branch with a concrete working example. Not "you're all
set" — an actual tool call that demonstrates the thing they wanted:

- Read-only: `get_portfolio_summary` with whichever addresses they've
  pasted (from `get_ledger_status`'s paired list if available, else ask).
- EVM signing: propose `preview_send` on a tiny self-transfer (e.g.
  0.001 ETH back to themselves). Don't actually send without their explicit
  go-ahead — just show them the preview.
- Solana: propose `prepare_solana_native_send` of 0.001 SOL to themselves.
  Surface the auto-bundled nonce-setup note so they're not surprised.
- TRON: propose `prepare_tron_native_send` of 1 TRX to themselves.

## Step 6 — flag the preflight skill if missing

If `preflightSkill.installed === false`, tell the user **after** the
setup flow completes (don't block on it):

> Heads up: the agent-side preflight skill isn't installed at
> `~/.claude/skills/vaultpilot-preflight/`. Without it, a compromised MCP
> could silently drop the bytes-verification instructions I run before
> signing. Install with:
>
> ```
> git clone https://github.com/szhygulin/vaultpilot-skill.git \
>   ~/.claude/skills/vaultpilot-preflight
> ```
>
> Then restart Claude Code. Safe to skip for read-only use, recommended
> before signing.

## Anti-patterns — things NOT to do

- Do NOT ask the user to edit `claude_desktop_config.json` / `mcp.json` /
  `.claude.json` directly. The server's installer auto-registered itself
  (PR #147). If they claim it didn't, re-run the installer — don't
  improvise JSON surgery.
- Do NOT ask for every API key upfront. Ask for the minimum for their
  use case and let them add more later when a tool prompts for it.
- Do NOT paste the API key back in chat to "confirm you got it right" —
  that writes it into chat history. Validate via the matching read-only
  tool call instead.
- Do NOT treat `public-fallback` as broken. It's fine for discovery /
  read-only. Only upgrade to a provider key when the user is doing
  heavy-write flows (Solana signing, frequent polling).

## Deep-link reference

Exact URLs per provider (avoid dashboard landing pages; go straight to
new-key forms):

| Provider | Deep link |
|---|---|
| Reown / WalletConnect | `https://cloud.reown.com/app/new-project` |
| Helius | `https://dashboard.helius.dev/api-keys` |
| TronGrid | `https://www.trongrid.io/dashboard/apikeys` |
| Etherscan (V2, multichain) | `https://etherscan.io/myapikey` |
| Infura | `https://app.infura.io/register` |
| Alchemy | `https://dashboard.alchemy.com/apps/new` |

Etherscan / Infura / Alchemy are all **optional** and should be deferred
until a tool actually needs them. Don't collect them preemptively.
