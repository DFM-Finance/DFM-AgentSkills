# DFM Agent

Fully autonomous AI agent skill for DTF (DeFi Token Fund) vault management on Solana. The agent independently researches markets, decides vault names/symbols/allocations/policies, and deploys on-chain -- no human confirmation required.

## Quick Start

### 1. Set environment variables

```bash
export DFM_API_URL="https://api.qa.dfm.finance"    # or http://0.0.0.0:3400 for local dev
export DFM_AUTH_TOKEN="<JWT from DFM Dashboard>"     # 30-day expiry, revocable
export AGENT_WALLET_PATH="$HOME/.dfm/agent-wallet.json"
export SOLANA_RPC_URL="https://api.mainnet-beta.solana.com"
```

Get your auth token from the [DFM Dashboard](https://app.dfm.so) — connect wallet, create Agent Profile, copy token.

### 2. Install the skill

```bash
npx skills add DFM-Finance/DFM-AgentSkills
```

Select your agents from the interactive prompt. For Claude Code, also copy to the `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp -r .agents/skills/dfm-agent .claude/skills/dfm-agent
```

### 3. Restart Claude Code and verify

Restart Claude Code (`/exit` and reopen), then check the skill is loaded:

```
/skills
```

You should see `dfm-agent` listed.

### 4. Generate your Agent Wallet

Ask Claude Code:

```
Generate a Solana keypair for my DFM Agent wallet
```

The agent will generate a keypair, save it to `AGENT_WALLET_PATH`, write `DFM_AGENT_KEYPAIR` to your shell profile, and report only the public key.

### 5. Fund the wallet

Send SOL (for tx fees) and USDC (for vault creation fees) to your agent's public key.

### 6. Start using

```
Launch a Solana blue chip fund
Show me the state of SOLBC
Rebalance SOLBC
```

## What it does

- **Autonomous DTF launch** -- agent researches markets, picks tokens, decides allocations/fees/policy, builds unsigned tx, signs locally, submits on-chain, then creates policy + persists to DB
- **Backend-resolved assets** -- for launch payloads, agent sends asset `symbol`/`name`; backend resolves `mintAddress` from `asset-allocation`
- **Portfolio rebalancing** -- monitors vaults, checks policy compliance, triggers server-side rebalancing by admin wallet
- **Fee distribution** -- builds unsigned fee distribution tx, agent signs locally and submits on-chain
- **Policy enforcement** -- the policy engine validates constraints; if the payload is valid, it deploys; if not, it returns the specific error

## Design philosophy

The agent is the creator. It has full authority over what it launches. No human-in-the-loop confirmation. The policy engine is the guardrail, not a human approval step. No secret keys are sent to the backend -- all transaction signing happens locally.

## Compatible runtimes

- Claude Code (requires manual copy to `.claude/skills/` -- see step 2)
- Codex
- OpenClaw
- Cursor, Cline, and other supported agents (auto-installed by `npx skills add`)

## Docs

- Skill guide: `skills/dfm-agent/SKILL.md`
- Full API reference: `skills/dfm-agent/references/api-reference.md`

## License

Proprietary. Copyright DFM.
