---
name: dfm-agent
description: >-
  DFM Agent skill for autonomous DTF vault management on Solana. Generates an
  Agent Wallet keypair (non-custodial, stored locally), authenticates with a
  DFM auth token from the dashboard, and provides full vault lifecycle: launch
  DTF vaults, check policies, rebalance, and distribute fees — all via the
  DFM Agent API. Use when the user asks to set up a DFM agent, generate an
  agent wallet, launch a vault, rebalance, check policy, or distribute fees.
---

# DFM Agent Skill

## Overview

This skill lets an AI assistant (Claude CLI, Openclaw, or similar) operate as
a **DFM Agent** — creating and managing DTF (DeFi Token Fund) vaults on Solana
through the DFM platform's Agent API.

**Non-custodial by design:** the Agent Wallet private key is generated and
stored **only on the user's machine**. The DFM platform never sees the secret
key — it only receives signed transactions and the public key.

---

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│  1. USER DASHBOARD (Frontend)                               │
│     User connects their main wallet, creates an Agent       │
│     Profile, and receives a JWT auth token                  │
│     (30-day expiry, revocable any time).                    │
├─────────────────────────────────────────────────────────────┤
│  2. ENVIRONMENT SETUP                                       │
│     User sets DFM_AUTH_TOKEN and AGENT_WALLET_PATH           │
│     as environment variables in their terminal.             │
├─────────────────────────────────────────────────────────────┤
│  3. GENERATE AGENT WALLET (on user's machine)               │
│     AI assistant generates a Solana keypair and stores      │
│     it at AGENT_WALLET_PATH. The platform NEVER sees        │
│     the secret key — fully non-custodial.                   │
├─────────────────────────────────────────────────────────────┤
│  4. AGENT API CALLS                                         │
│     AI assistant calls DFM Agent API endpoints using:       │
│       - Auth Token (Bearer JWT) for identity proof          │
│       - Agent Wallet for signing on-chain transactions      │
│     The Agent Wallet becomes the permanent on-chain         │
│     creator/manager of any vaults it deploys.               │
├─────────────────────────────────────────────────────────────┤
│  5. VAULT MANAGEMENT                                        │
│     Launch DTFs, check policies, rebalance, distribute      │
│     fees — all signed with the Agent Wallet.                │
│     Backend tracks which user created each vault via        │
│     the auth token; Agent Wallet remains the on-chain       │
│     management authority.                                   │
└─────────────────────────────────────────────────────────────┘
```

| Principle | Detail |
|---|---|
| **Non-custodial** | Agent Wallet private key never leaves the user's machine. |
| **Auth token as identity** | JWT ties API calls to the user's profile; 30-day expiry, revocable. |
| **Agent Wallet as authority** | On-chain, the Agent Wallet is the permanent creator/manager of vaults it deploys. |
| **User attribution** | Backend maps each Agent Wallet action to the originating user via auth token. |

---

## Setup Guide

### Prerequisites

- **Node.js** v18+
- **@solana/web3.js** installed (`npm install @solana/web3.js`)
- An AI coding tool:
  - [Claude CLI](https://docs.anthropic.com/en/docs/claude-code) (`npm install -g @anthropic-ai/claude-code`), **or**
  - [Openclaw](https://openclaw.dev), **or**
  - Any AI assistant that supports custom skills

### Step 1 — Get Your Auth Token

1. Go to the **DFM Dashboard** and connect your Solana wallet (Phantom, Backpack, etc.).
2. Navigate to **Agent Profile** and create your profile (name, username, metadata).
3. After creation, copy the **Auth Token** (JWT) shown on screen.
   - Expires in **30 days**.
   - Revocable any time from the dashboard.

### Step 2 — Set Environment Variables

Add these to your shell profile (`~/.zshrc` or `~/.bashrc`):

```bash
# ── DFM Agent Configuration ──────────────────────────────────

# Auth token from the DFM Dashboard (paste your JWT here)
export DFM_AUTH_TOKEN="eyJhbGciOi..."

# Path where the Agent Wallet keypair will be stored locally
export AGENT_WALLET_PATH="$HOME/.dfm/agent-wallet.json"
```

Reload and verify:

```bash
source ~/.zshrc
echo $DFM_AUTH_TOKEN
echo $AGENT_WALLET_PATH
```

### Step 3 — Add the DFM Agent Skill

**Claude CLI:** Ensure the `skills/` folder with this `SKILL.md` is in your project. Claude CLI auto-discovers it. Verify by asking:

```
> What DFM agent skills are available?
```

**Openclaw:** Point to the `skills/SKILL.md` file per Openclaw's custom skill docs.

### Step 4 — Generate Your Agent Wallet

Ask your AI assistant:

```
> Generate a Solana keypair for my DFM Agent wallet
```

The assistant will:
1. Generate a new Solana `Keypair` via `@solana/web3.js`
2. Save the 64-byte secret key (JSON array) to `AGENT_WALLET_PATH` (default: `~/.dfm/agent-wallet.json`)
3. Set file permissions to `0o600` (owner-only)
4. Report the **public key** only

Example output:

```
Agent Wallet created successfully.
Public key: 7xKp...3nQm
Saved to: /Users/you/.dfm/agent-wallet.json
```

**You are now ready to use the Agent API.**

---

## Agent Wallet — Keypair Generation

### Wallet path resolution (checked in order)

1. `AGENT_WALLET_PATH` env variable
2. `SOLANA_KEYPAIR_PATH` env variable
3. `WALLET_OUTPUT_PATH` env variable
4. Default: `<project-root>/solana-keypair/keypair.json`

### Implementation

```typescript
import { Keypair } from "@solana/web3.js";
import * as fs from "fs";
import * as path from "path";

const defaultOutPath = path.join(process.cwd(), "solana-keypair", "keypair.json");
const outPath = path.resolve(
  process.env.AGENT_WALLET_PATH ??
    process.env.SOLANA_KEYPAIR_PATH ??
    process.env.WALLET_OUTPUT_PATH ??
    defaultOutPath
);

fs.mkdirSync(path.dirname(outPath), { recursive: true });

const keypair = Keypair.generate();
fs.writeFileSync(outPath, JSON.stringify(Array.from(keypair.secretKey)), {
  mode: 0o600,
});

const pubkey = keypair.publicKey.toBase58();
// Report pubkey to user; do NOT echo the secret key.
```

### Loading the keypair

```typescript
import { Keypair } from "@solana/web3.js";
import * as fs from "fs";

const raw = JSON.parse(fs.readFileSync(outPath, "utf8"));
const keypair = Keypair.fromSecretKey(Uint8Array.from(raw));
```

---

## Agent API Reference

Base URL: `/api/v2/agent`

> For full request/response schemas, see
> [`DFM-Backend/src/modules/agent-profile/AGENT-API.md`](../../DFM-Backend/src/modules/agent-profile/AGENT-API.md)

### Authentication

All endpoints require a JWT token (obtained from the DFM Dashboard after profile creation):

```
Authorization: Bearer <DFM_AUTH_TOKEN>
```

Endpoints marked **[Public]** bypass authentication.

### Launch DTF Vault [Authenticated]

```
POST /api/v2/agent/launch-dtf
Authorization: Bearer <DFM_AUTH_TOKEN>
```

Body (required: `vaultName`, `vaultSymbol`, `underlyingAssets`, `managementFees`):
```json
{
  "vaultName": "Alpha Fund",
  "vaultSymbol": "ALPHA",
  "underlyingAssets": [
    { "mintAddress": "So11111111111111111111111111111111111111112", "mintBps": 5000 },
    { "mintAddress": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v", "mintBps": 5000 }
  ],
  "managementFees": 200,
  "vaultType": "DTF",
  "description": "An alpha-seeking vault",
  "asset_mode": "OPEN",
  "max_assets": 10,
  "max_asset_pct": 4000,
  "min_rebalance_interval_hours": 24
}
```

The Agent Wallet signs the on-chain vault creation transaction and becomes the permanent creator/manager.

---

### Get Vault State [Authenticated]

```
GET /api/v2/agent/dtf/:symbol/state
Authorization: Bearer <DFM_AUTH_TOKEN>
```

Returns vault details including APY, share price, NAV, TVL, portfolio, user holdings, and rebalance history.

---

### Get Vault Policy [Authenticated]

```
GET /api/v2/agent/dtf/:symbol/policy
Authorization: Bearer <DFM_AUTH_TOKEN>
```

Returns the full constitutional policy rule set for the vault.

---

### Rebalance Check — Dry Run [Authenticated]

```
GET /api/v2/agent/dtf/:symbol/rebalance/check
Authorization: Bearer <DFM_AUTH_TOKEN>
```

Validates rebalancing against constitutional policy without executing. Returns approved with suggestion, or rejection with the specific rule that failed (`PolicyViolation` + `violationCode`). No state change.

---

### Execute Rebalance [Authenticated]

```
POST /api/v2/agent/dtf/:symbol/rebalance
Authorization: Bearer <DFM_AUTH_TOKEN>
```

Executes vault rebalancing using the Agent Wallet. Runs sell phase then buy phase on-chain.

**Errors:** `400` Policy violated or rebalancing already in progress | `404` Vault not found

---

### Distribute Fees [Authenticated]

```
POST /api/v2/agent/dtf/:symbol/distribute-fees
Authorization: Bearer <DFM_AUTH_TOKEN>
```

Distributes accrued management fees for a vault on-chain.

---

### Revoke Token [Authenticated]

```
POST /api/v2/agent/token/revoke
Authorization: Bearer <DFM_AUTH_TOKEN>
```

Immediately invalidates the current auth token. Get a new one from the dashboard.

**Errors:** `401` Invalid or already revoked token

---

### Refresh Token [Public]

```
POST /api/v2/agent/token/refresh
```

Body:
```json
{ "profileId": "<agent-profile-id>" }
```

Issues a new token and revokes **all** existing tokens for the agent. Use when the current token has expired.

**Errors:** `404` No agent profile found for this profileId

---

## Using Agent Commands

Once set up, give natural-language commands to your AI assistant:

| What you say | What happens |
|---|---|
| `Generate a Solana keypair for my DFM Agent wallet` | Creates keypair, saves to `AGENT_WALLET_PATH`, reports public key |
| `Launch a new DTF vault called "Beta Fund" with symbol BETA, 50% SOL and 50% USDC, 2% management fee` | Calls `POST /launch-dtf`, Agent Wallet signs on-chain |
| `Show me the state of the ALPHA vault` | Calls `GET /dtf/ALPHA/state` — returns APY, TVL, NAV |
| `What are the policy rules for the ALPHA vault?` | Calls `GET /dtf/ALPHA/policy` — returns rule set |
| `Can I rebalance the ALPHA vault right now?` | Calls `GET /dtf/ALPHA/rebalance/check` — dry run, no changes |
| `Rebalance the ALPHA vault` | Calls `POST /dtf/ALPHA/rebalance` — signs and submits on-chain |
| `Distribute fees for the ALPHA vault` | Calls `POST /dtf/ALPHA/distribute-fees` |
| `Revoke my current DFM agent token` | Calls `POST /token/revoke` — invalidates JWT |
| `Refresh my DFM agent token` | Calls `POST /token/refresh` — new token, old ones revoked |

---

## API Quick Reference

| Action | Method | Endpoint | Auth |
|---|---|---|---|
| Launch DTF | `POST` | `/api/v2/agent/launch-dtf` | Yes |
| Vault state | `GET` | `/api/v2/agent/dtf/:symbol/state` | Yes |
| Vault policy | `GET` | `/api/v2/agent/dtf/:symbol/policy` | Yes |
| Rebalance check | `GET` | `/api/v2/agent/dtf/:symbol/rebalance/check` | Yes |
| Rebalance | `POST` | `/api/v2/agent/dtf/:symbol/rebalance` | Yes |
| Distribute fees | `POST` | `/api/v2/agent/dtf/:symbol/distribute-fees` | Yes |
| Revoke token | `POST` | `/api/v2/agent/token/revoke` | Yes |
| Refresh token | `POST` | `/api/v2/agent/token/refresh` | No |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| **"Unauthorized" errors** | Token expired (30-day limit). Refresh via dashboard or `Refresh my DFM agent token`. Verify: `echo $DFM_AUTH_TOKEN` |
| **"Keypair file not found"** | Re-run wallet generation (Step 4). Check path: `ls -la $AGENT_WALLET_PATH` |
| **"Permission denied" on keypair** | File is `0o600` (owner-only). Run as the same OS user who created it. |
| **Token revoked unexpectedly** | A token refresh revokes **all** prior tokens. One active token per agent at a time. |

---

## Security

- **Agent Wallet file** — `0o600` permissions, gitignored, never committed.
- **Auth token** — stored as env variable, never hardcoded in code.
- **Non-custodial** — DFM platform never sees your private key.
- **Revocation** — revoke your auth token instantly from the dashboard if compromised.
- **Agent Wallet = on-chain authority** — if compromised, attacker can manage your vaults. Back it up securely and treat it like any crypto wallet private key.
