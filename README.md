# DFM Agent

Fully autonomous AI agent skill for DTF (DeFi Token Fund) vault management on Solana. The agent independently researches markets, decides vault names/symbols/allocations/policies, and deploys on-chain in a single API call — no human confirmation required.

## Install

```bash
npx skills add DFM-Finance/DFM-AgentSkills
```

> **Prerequisites:** You need a JWT auth token from the [DFM Dashboard](https://app.dfm.so) and a locally generated Solana keypair before first use.

## What it does

- **Autonomous DTF launch** — agent researches markets, picks tokens, decides allocations/fees/policy, deploys in one call
- **Portfolio rebalancing** — monitors vaults, checks policy compliance, executes sell/buy phases on-chain
- **Fee distribution** — claims accrued management fees on-chain
- **Policy enforcement** — the policy engine validates constraints; if the payload is valid, it deploys; if not, it returns the specific error

## Design philosophy

The agent is the creator. It has full authority over what it launches. No human-in-the-loop confirmation. The policy engine is the guardrail, not a human approval step.

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

# Base58-encoded Agent Wallet secret key (generated locally)
export DFM_AGENT_KEYPAIR="<base58-secret-key>"

# Path to store the keypair JSON file
export AGENT_WALLET_PATH="$HOME/.dfm/agent-wallet.json"
```

## Docs

- Skill guide: `skills/dfm-agent/SKILL.md`
- Full API reference: `skills/dfm-agent/references/api-reference.md`

## License

Proprietary. Copyright DFM.
