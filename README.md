# DFM Agent

Fully autonomous AI agent skill for DTF (DeFi Token Fund) vault management on Solana. The agent independently researches markets, decides vault names/symbols/allocations/policies, and deploys on-chain -- no human confirmation required.

## Quick Start

### 1. Register on the DFM Dashboard

Go to the [DFM Dashboard](https://app.dfm.so) and connect your Solana wallet (Phantom, Backpack, etc.). Note down your wallet address.

### 2. Set base environment variables

```bash
export DFM_API_URL="<your-dfm-api-url>"
export AGENT_WALLET_PATH="$HOME/.dfm/agent-wallet.json"
export SOLANA_RPC_URL="https://api.mainnet-beta.solana.com"
```

> `DFM_AUTH_TOKEN` and `DFM_AGENT_KEYPAIR` are set automatically during agent profile launch. You do NOT need to set them manually.

### 3. Install the skill

```bash
npx skills add DFM-Finance/DFM-AgentSkills
```

Select your agents from the interactive prompt. For Claude Code, also copy to the `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp -r .agents/skills/dfm-agent .claude/skills/dfm-agent
```

### 4. Restart Claude Code and verify

Restart Claude Code (`/exit` and reopen), then check the skill is loaded:

```
/skills
```

You should see `dfm-agent` listed.

### 5. Launch your Agent Profile

Tell Claude Code:

```
Launch my agent profile
```

The agent will:
1. Ask for your **DFM-registered wallet address** (the Solana public key you connected on the Dashboard)
2. Auto-generate an agent profile name and username
3. Call the `POST /profile-launch` API to create your agent profile
4. Save the returned **auth token** to `.claude/settings.json` and `~/.zshrc` (never printed in terminal)
5. Auto-generate a **Solana keypair** for your agent wallet
6. Save the keypair and export `DFM_AGENT_KEYPAIR` to `~/.zshrc` (never printed in terminal)
7. Report only your **public key**

After this, restart Claude Code once more and you're fully set up.

### 6. Fund the wallet

Send SOL (for tx fees) and USDC (for vault creation fees) to your agent's public key.

### 7. Start using

```
Launch a Solana blue chip fund
Show me the state of SOLBC
Rebalance SOLBC
Distribute fees for SOLBC
```

## Already have an auth token?

If you already have a `DFM_AUTH_TOKEN` from the Dashboard, set it directly:

```bash
export DFM_AUTH_TOKEN="<your-jwt-here>"
```

The agent will skip the profile launch step and use this token for all API calls.

## What it does

- **Autonomous DTF launch** -- agent researches markets, picks tokens, decides allocations/fees/policy, builds unsigned tx, signs locally, submits on-chain, then creates policy + persists to DB
- **Backend-resolved assets** -- for launch payloads, agent sends asset `symbol`/`name`; backend resolves `mintAddress` from `asset-allocation`
- **Portfolio rebalancing** -- monitors vaults, checks policy compliance, triggers server-side rebalancing by admin wallet
- **Fee distribution** -- builds unsigned fee distribution tx, agent signs locally and submits on-chain
- **Policy enforcement** -- the policy engine validates constraints; if the payload is valid, it deploys; if not, it returns the specific error

## Design philosophy

The agent is the creator. It has full authority over what it launches. No human-in-the-loop confirmation. The policy engine is the guardrail, not a human approval step. No secret keys are sent to the backend -- all transaction signing happens locally.

## Compatible runtimes

- Claude Code (requires manual copy to `.claude/skills/` -- see step 3)
- Codex
- OpenClaw
- Cursor, Cline, and other supported agents (auto-installed by `npx skills add`)

## Docs

- Skill guide: `skills/dfm-agent/SKILL.md`
- Full API reference: `skills/dfm-agent/references/api-reference.md`

## License

Proprietary. Copyright DFM.
