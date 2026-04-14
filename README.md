# DFM Agent

Fully autonomous AI agent skill for DTF (DeFi Token Fund) vault management on Solana. The agent independently researches markets, decides vault names/symbols/allocations/policies, and deploys on-chain -- no human confirmation required.

## Install

```bash
npx skills add DFM-Finance/DFM-AgentSkills
```

> **Prerequisites:** You need a JWT auth token from the [DFM Dashboard](https://app.dfm.so) and a locally generated Solana keypair before first use.

## What it does

- **Autonomous DTF launch** -- agent researches markets, picks tokens, decides allocations/fees/policy, builds unsigned tx, signs locally, submits on-chain, then creates policy + persists to DB
- **Backend-resolved assets** -- for launch payloads, agent sends asset `symbol`/`name`; backend resolves `mintAddress` from `asset-allocation`
- **Portfolio rebalancing** -- monitors vaults, checks policy compliance, triggers server-side rebalancing by admin wallet
- **Fee distribution** -- builds unsigned fee distribution tx, agent signs locally and submits on-chain
- **Policy enforcement** -- the policy engine validates constraints; if the payload is valid, it deploys; if not, it returns the specific error

## Design philosophy

The agent is the creator. It has full authority over what it launches. No human-in-the-loop confirmation. The policy engine is the guardrail, not a human approval step. No secret keys are sent to the backend -- all transaction signing happens locally.

## Compatible runtimes

- Claude Code (set to full permission mode for fully autonomous operation)
- Codex
- OpenClaw

## Auth

```bash
# API base URL (production default: https://api.qa.dfm.finance)
# For local dev, set to http://0.0.0.0:3400
export DFM_API_URL="https://api.qa.dfm.finance"

# JWT from DFM Dashboard (30-day expiry, revocable)
export DFM_AUTH_TOKEN="eyJhbGciOi..."

# Base58-encoded Agent Wallet secret key (used for local signing, never sent to backend)
export DFM_AGENT_KEYPAIR="<base58-secret-key>"

# Path to store the keypair JSON file
export AGENT_WALLET_PATH="$HOME/.dfm/agent-wallet.json"

# Solana RPC URL for on-chain transaction submission
export SOLANA_RPC_URL="https://api.mainnet-beta.solana.com"
```

## Docs

- Skill guide: `skills/dfm-agent/SKILL.md`
- Full API reference: `skills/dfm-agent/references/api-reference.md`

## License

Proprietary. Copyright DFM.
