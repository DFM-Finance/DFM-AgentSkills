---
name: dfm-agent
version: 2.0.0
description: |
  Fully autonomous DTF vault management on Solana. The agent independently researches markets,
  decides vault names/symbols/allocations/policies, and deploys on-chain in a two-step flow —
  no human confirmation required. The policy engine is the guardrail, not a human approval step.
  Use when the user asks to launch a vault, rebalance, check policy, or distribute fees.
homepage: https://dfm.so
license: Proprietary
compatibility: Claude Code, Codex, OpenClaw
metadata: {"category":"defi-agent","auth":"jwt+keypair","author":"dfm","tags":"solana,defi,vault,rebalance,dtf,fund-management,autonomous-agent"}
---

# DFM Agent

DFM Agent is a **fully autonomous** AI skill for DTF (DeFi Token Fund) vault management on Solana. The agent researches markets, decides everything — name, symbol, asset allocations, fee structure, policy configuration — and deploys on-chain. No human-in-the-loop confirmation. The **policy engine** is the guardrail, not a human approval step.

## Core Philosophy

**The agent is the creator. It has full authority over what it launches.**

```
+----------------------------------------------------------------+
|                    AUTONOMOUS AGENT FLOW                        |
|                                                                |
|  1. RESEARCH    Agent analyzes markets, trending tokens,       |
|                 liquidity, volume, and macro conditions         |
|                                                                |
|  2. DECIDE      Agent picks vault name, symbol, assets,        |
|                 allocations, fees, and policy rules             |
|                                                                |
|  3. DEPLOY      Two-step vault creation:                       |
|                 a) POST /launch-dtf -> unsigned transaction     |
|                 b) Agent signs tx & submits on-chain            |
|                 c) POST /dtf-create -> policy + DB persist      |
|                                                                |
|  4. MANAGE      Agent monitors, rebalances, distributes fees   |
|                 -- all autonomously via API calls               |
|                                                                |
|  GUARDRAILS:    Policy engine validates constraints.            |
|                 Invalid payloads return errors with specifics.  |
|                 NO human confirmation step.                     |
+----------------------------------------------------------------+
```

| Principle | Detail |
|---|---|
| **Fully autonomous** | Agent decides everything: name, symbol, assets, allocations, policy, fees. No confirmation prompts. |
| **Two-step vault creation** | `POST /launch-dtf` builds the unsigned tx, agent signs & submits, then `POST /dtf-create` creates policy + persists to DB. |
| **Policy engine as guardrail** | Backend validates the payload and policy config. If valid, it deploys. If not, it returns an error with the specific issue. |
| **Non-custodial** | Agent Wallet private key never leaves the user's machine. Backend never receives secret keys. |
| **Agent = on-chain authority** | The Agent Wallet becomes the permanent on-chain creator/manager of every vault it deploys. |

## Sensitive Data Rules (MANDATORY -- READ FIRST)

**NEVER print, echo, log, or display the values of these environment variables in terminal output:**
- `DFM_AUTH_TOKEN` -- JWT auth token
- `DFM_AGENT_KEYPAIR` -- base58 secret key
- Any private key, secret key, or auth token

**To check if an env var is set, use length check only:**
```bash
node -e 'console.log(process.env.DFM_AUTH_TOKEN ? "set" : "not set")'
```

**NEVER do any of the following:**
```bash
# WRONG - exposes token in terminal output
echo $DFM_AUTH_TOKEN
export DFM_AUTH_TOKEN="eyJ..."   # WRONG - token visible in bash command
curl -H "Authorization: Bearer eyJ..." ...   # WRONG - token embedded in command
curl -H "Authorization: Bearer $DFM_AUTH_TOKEN" ...   # WRONG - shell expands and displays it
```

**ALL API calls MUST use inline `node -e` scripts** that read env vars internally via `process.env`. This prevents sensitive values from appearing in the bash command itself.

**Correct pattern for API calls:**
```bash
node -e '
const http = require("http");
const https = require("https");
const url = new URL((process.env.DFM_API_URL || "http://0.0.0.0:3400") + "/api/v2/agent/dtf/SYMBOL/state");
const client = url.protocol === "https:" ? https : http;
const req = client.get(url, { headers: { "Authorization": "Bearer " + process.env.DFM_AUTH_TOKEN } }, (res) => {
  let data = "";
  res.on("data", (chunk) => data += chunk);
  res.on("end", () => console.log(data));
});
req.on("error", (e) => console.error("Error:", e.message));
'
```

**Do NOT use `curl` for API calls.** Always use `node -e` with `process.env` references so tokens and keys are never visible in the command string.

## Pre-Flight Auth Check (REQUIRED)

**You MUST complete this check before making any API call.** Do not skip this step.

### Step 1: Ensure `.claude/settings.json` has env vars

Claude Code runs bash in a non-interactive subprocess that does NOT source `~/.zshrc` or `~/.bashrc`. Environment variables set via `export` in the user's terminal are NOT available to Claude Code's bash commands. The **only reliable way** to pass env vars to Claude Code is via `.claude/settings.json`.

**On every pre-flight check**, run this script to sync env vars from `~/.zshrc` into `.claude/settings.json`:

```bash
node -e '
const fs = require("fs");
const path = require("path");
const os = require("os");

const settingsPath = path.join(process.cwd(), ".claude", "settings.json");
let settings = {};
try { settings = JSON.parse(fs.readFileSync(settingsPath, "utf8")); } catch {}
if (!settings.env) settings.env = {};

// Read from ~/.zshrc to find exported DFM vars
const zshrc = fs.readFileSync(path.join(os.homedir(), ".zshrc"), "utf8");
const envVars = ["DFM_API_URL", "DFM_AUTH_TOKEN", "DFM_AGENT_KEYPAIR", "SOLANA_RPC_URL", "AGENT_WALLET_PATH"];
for (const v of envVars) {
  // Match export VAR="value" or export VAR=value
  const match = zshrc.match(new RegExp("export\\s+" + v + "=[\"'\\'']?([^\"'\\''\\n]+)[\"'\\'']?"));
  if (match && match[1]) settings.env[v] = match[1];
  // Also check current process.env as fallback
  if (!settings.env[v] && process.env[v]) settings.env[v] = process.env[v];
}

// Set default API URL if not found
if (!settings.env.DFM_API_URL) settings.env.DFM_API_URL = "http://0.0.0.0:3400";

fs.mkdirSync(path.dirname(settingsPath), { recursive: true });
fs.writeFileSync(settingsPath, JSON.stringify(settings, null, 2));

// Report status without exposing values
for (const v of envVars) {
  console.log(v + "=" + (settings.env[v] ? "set" : "NOT SET"));
}
'
```

**If `DFM_AUTH_TOKEN` is NOT SET after this script**, STOP and tell the user:

> Your DFM auth token is not configured. Add it to your `~/.zshrc`:
>
> ```bash
> echo 'export DFM_AUTH_TOKEN="your-jwt-here"' >> ~/.zshrc
> ```
>
> Get the token from the [DFM Dashboard](https://app.dfm.so) (Agent Profile section). Then restart Claude Code.

**If `DFM_AGENT_KEYPAIR` is NOT SET** and the operation requires signing (launch-dtf, distribute-fees), offer to generate a wallet:

> Your Agent Wallet keypair is not configured.
> Say **"Generate a Solana keypair for my DFM Agent wallet"** and I'll create one.

### Step 2: Proceed

If all required vars are set, proceed immediately with the requested operation. **Do not ask for confirmation.**

**IMPORTANT:** After writing to `.claude/settings.json` for the first time, tell the user to restart Claude Code so the env vars are picked up. On subsequent runs, the settings file already exists and env vars are available immediately.

## When To Use

Use this skill when:
- The user asks to launch, create, or deploy a DTF vault
- The user asks to research tokens/markets and create a fund
- The user wants to check vault state, policy, or rebalancing status
- The user asks to rebalance a vault or distribute fees
- The user needs to set up agent auth (wallet generation, token management)

## Autonomous DTF Launch -- How It Works

When the user asks you to launch a DTF (e.g. "Create a blue chip Solana fund" or "Launch a meme token DTF"), follow this autonomous workflow:

### Step 1: Research (Agent decides)

Use `WebSearch` and `WebFetch` tools for ALL DTF-related metadata -- token discovery, prices, market caps, volume, liquidity, mint addresses, trending assets, and market conditions. No exceptions.

Use `WebSearch` to find:
- Top performing Solana tokens by market cap, volume, and price action
- Current market conditions, trends, and sentiment
- Token liquidity and 24h trading volume data

Use `WebFetch` to pull data from:
- Token data aggregators (CoinGecko, CoinMarketCap, Jupiter aggregator, Birdeye, DexScreener)
- Solana token lists and verified registries for mint addresses

Then decide:
- Identify candidate tokens based on the user's intent or strategy
- Check token liquidity, 24h volume, market cap
- Select the best tokens and determine allocations
- Automatically discover each token's Solana `mintAddress` from reliable references (official docs, verified token lists, explorers, major data providers)
- Cross-check mint addresses across multiple references before including them in `underlyingAssets`
- Reject unverified, conflicting, or low-confidence mint mappings and replace them with verified assets

### Step 2: Design (Agent decides)

Based on your research, autonomously decide:
- **Vault name** -- descriptive, catchy, relevant to the strategy
- **Vault symbol** -- short (max 10 chars), unique, memorable
- **Underlying assets** -- pass asset `symbol` or `name` (preferred) with allocation in basis points (must sum to 10000). Backend resolves `mintAddress` from `asset-allocation`.
- **Management fees** -- in basis points (e.g. 200 = 2%)
- **Policy configuration** -- asset limits, rebalance frequency, stablecoin minimums, etc.
- **Tags** -- searchable categories
- **Description** -- strategy summary
- **Launch media fields** -- for DTF launch payloads, set `logoUrl`, `bannerUrl`, and `metadataUri` to empty strings.

### Step 3: Deploy (Two-step flow)

#### 3a. Build the unsigned transaction

Send `POST {DFM_API_URL}/api/v2/agent/launch-dtf` with the vault creation payload. The backend builds and returns an unsigned transaction.

```json
{
  "signerPublicKey": "<public key derived from DFM_AGENT_KEYPAIR>",
  "vaultName": "Solana Blue Chips",
  "vaultSymbol": "SOLBC",
  "underlyingAssets": [
    { "symbol": "SOL", "mintBps": 4000 },
    { "symbol": "JUP", "mintBps": 3000 },
    { "name": "Bonk", "mintBps": 3000 }
  ],
  "managementFees": 200,
  "category": 0,
  "threshold": 500
}
```

The response contains:
```json
{
  "onChain": {
    "transaction": "base64-encoded-unsigned-versioned-transaction...",
    "vaultIndex": 42,
    "vaultPda": "7Xk...def",
    "vaultMintPda": "9Rm...ghi"
  }
}
```

#### 3b. Sign and submit the transaction on-chain

Use the agent's local keypair to sign the returned transaction and submit it to Solana:

```typescript
import { Keypair, VersionedTransaction, Connection } from "@solana/web3.js";
import * as bs58 from "bs58";

const keypair = Keypair.fromSecretKey(bs58.decode(process.env.DFM_AGENT_KEYPAIR!));
const connection = new Connection(process.env.SOLANA_RPC_URL || "https://api.mainnet-beta.solana.com");

// Deserialize the unsigned transaction from the API response
const txBytes = Buffer.from(response.onChain.transaction, "base64");
const tx = VersionedTransaction.deserialize(txBytes);

// Sign with the agent's keypair
tx.sign([keypair]);

// Submit on-chain
const signature = await connection.sendRawTransaction(tx.serialize(), {
  skipPreflight: false,
  preflightCommitment: "confirmed",
});

await connection.confirmTransaction(signature, "confirmed");
```

#### 3c. Create policy and persist to DB

After the on-chain transaction confirms, send `POST {DFM_API_URL}/api/v2/agent/dtf-create` with the transaction signature and policy configuration:

```json
{
  "transactionSignature": "<signature from step 3b>",
  "vaultName": "Solana Blue Chips",
  "vaultSymbol": "SOLBC",
  "vaultType": "DTF",
  "description": "Top-tier Solana ecosystem tokens",
  "tags": ["Blue Chip", "Solana", "DeFi"],
  "logoUrl": "",
  "bannerUrl": "",
  "asset_mode": "OPEN",
  "max_assets": 10,
  "max_asset_pct": 5000,
  "min_asset_pct": 500,
  "min_stablecoin_pct": 2000,
  "min_rebalance_interval_hours": 4,
  "max_rebalances_per_day": 3,
  "fee_locked": true
}
```

#### Backend payload rules

For DTF launch payloads:
- `category: 0` (Manual) for agent-created vaults.
- In `underlyingAssets`, send `symbol` or `name` (preferred). Backend resolves `mintAddress` from `asset-allocation`.
- Hard restriction: never include USDC (`symbol: "USDC"` or `name: "USD Coin"`) in `underlyingAssets`.
- If a candidate list contains USDC, remove it and replace it with another eligible non-USDC asset before sending `launch-dtf`.

For `dtf-create` payloads:
- Set `logoUrl`, `bannerUrl` to empty strings.
- `vaultName` and `vaultSymbol` must match what was used in `launch-dtf`.

#### Signing helper for API calls that return unsigned transactions

Both `launch-dtf` and `distribute-fees` return base64-encoded unsigned `VersionedTransaction`s. Use this pattern to sign and submit:

```typescript
import { Keypair, VersionedTransaction, Connection } from "@solana/web3.js";
import * as bs58 from "bs58";

async function signAndSendTransaction(
  base64Tx: string,
  keypair: Keypair,
  connection: Connection
): Promise<string> {
  const txBytes = Buffer.from(base64Tx, "base64");
  const tx = VersionedTransaction.deserialize(txBytes);
  tx.sign([keypair]);
  const signature = await connection.sendRawTransaction(tx.serialize(), {
    skipPreflight: false,
    preflightCommitment: "confirmed",
  });
  await connection.confirmTransaction(signature, "confirmed");
  return signature;
}

// Usage:
const keypair = Keypair.fromSecretKey(bs58.decode(process.env.DFM_AGENT_KEYPAIR!));
const connection = new Connection(process.env.SOLANA_RPC_URL || "https://api.mainnet-beta.solana.com");
const sig = await signAndSendTransaction(response.onChain.transaction, keypair, connection);
```

**If something is wrong**, the API returns an error with the specific issue -- the agent can fix and retry without human intervention.

### Step 4: Manage (Ongoing, autonomous)

After launch, the agent autonomously:
- Monitors vault state via `GET {DFM_API_URL}/api/v2/agent/dtf/:symbol/state`
- Checks rebalancing readiness via `GET {DFM_API_URL}/api/v2/agent/dtf/:symbol/rebalance/check`
- Executes rebalances via `POST {DFM_API_URL}/api/v2/agent/dtf/:symbol/rebalance` (admin wallet executes behind the scenes)
- Distributes accrued fees via `POST {DFM_API_URL}/api/v2/agent/dtf/:symbol/distribute-fees` (returns unsigned tx for agent to sign)

All management operations are single API calls. No confirmation needed.

## Behavioral Guidelines

### DO:
- **Act autonomously.** Research, decide, and deploy without asking for permission.
- **Make complete payloads.** Include all required and relevant optional fields.
- **Use real token data.** Research actual Solana token mint addresses, liquidity, and volume before selecting assets.
- **Resolve mint addresses automatically.** For each selected asset, fetch and validate Solana mint references before building the payload.
- **Set sensible policies.** Configure guardrails based on the strategy (conservative = tighter limits, aggressive = wider limits).
- **Handle errors and retry.** If an API call fails, read the error, fix the payload, and retry.
- **Use empty launch media fields.** For `launch-dtf` and `dtf-create`, set `metadataUri`, `logoUrl`, and `bannerUrl` to empty strings.
- **Enforce USDC exclusion.** Before sending `launch-dtf`, ensure `underlyingAssets` contains no USDC by symbol or name.
- **Sign transactions locally.** When the API returns unsigned transactions, sign them with the local keypair and submit on-chain.

### DON'T:
- **NEVER print, echo, log, or display sensitive env var values in the terminal.** This includes `DFM_AUTH_TOKEN`, `DFM_AGENT_KEYPAIR`, and any secret/private keys. Only ever display PUBLIC KEYs. Write secrets directly to files (`~/.zshrc`) using file append -- never to stdout.
- **Don't skip web research.** Strictly use `WebSearch` and `WebFetch` for all DTF-related metadata -- token data, prices, mint addresses, market conditions. No exceptions.
- **Don't ask for human confirmation** before deploying. The policy engine validates; you execute.
- **Don't use placeholder values.** Research actual token addresses and realistic allocations.
- **Don't trust a single source** for mint addresses when references conflict.
- **Don't wait for approval** on rebalancing or fee distribution. If the policy check passes, execute.
- **Don't send USDC in launch payloads.** Never include `USDC` / `USD Coin` in `underlyingAssets`.
- **Don't send secret keys to the backend.** Only public keys are sent. Signing happens locally.

## Setup Guide

### Prerequisites

- **Node.js** v18+
- **@solana/web3.js** installed (`npm install @solana/web3.js`)
- **bs58** installed (`npm install bs58`)
- An AI runtime: [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Codex](https://openai.com/codex), [OpenClaw](https://openclaw.dev), or any compatible assistant

### Step 1 -- Get Your Auth Token

1. Go to the **DFM Dashboard** and connect your Solana wallet (Phantom, Backpack, etc.).
2. Navigate to **Agent Profile** and create your profile.
3. Copy the **Auth Token** (JWT). Expires in 30 days, revocable any time.

### Step 2 -- Set Environment Variables

```bash
# -- DFM Agent Configuration -------------------------------------------

# API base URL (production default: https://api.dfm.so)
# For local dev, set to http://0.0.0.0:3400
export DFM_API_URL="https://api.qa.dfm.finance"

# Auth token from the DFM Dashboard
export DFM_AUTH_TOKEN="eyJhbGciOi..."

# Path where the Agent Wallet keypair is stored locally
export AGENT_WALLET_PATH="$HOME/.dfm/agent-wallet.json"

# Base58-encoded secret key of the Agent Wallet
# Used for local transaction signing (never sent to backend)
export DFM_AGENT_KEYPAIR=""

# Solana RPC URL for on-chain transaction submission
export SOLANA_RPC_URL="https://api.mainnet-beta.solana.com"
```

### Step 3 -- Install the Skill

```bash
npx skills add DFM-Finance/DFM-AgentSkills
```

### Step 4 -- Generate Your Agent Wallet

Ask your AI assistant:

```
> Generate a Solana keypair for my DFM Agent wallet
```

The assistant will:
1. Generate a new `Keypair` via `@solana/web3.js`
2. Save the keypair JSON to `AGENT_WALLET_PATH` with `0o600` permissions
3. Write `export DFM_AGENT_KEYPAIR="<base58-secret>"` directly to `~/.zshrc` (silently, via file append)
4. **Only display the PUBLIC KEY in the terminal — NEVER the secret key**

**SECURITY:** The secret key must NEVER appear in terminal output, logs, or chat. It is written directly to the shell profile file and the keypair JSON file — both with restricted permissions. Only the public key is printed.

## Auth

All endpoints are at `{DFM_API_URL}/api/v2/agent/...` and require `Authorization: Bearer <DFM_AUTH_TOKEN>`.

Endpoints marked **[Public]** bypass JWT authentication.

On-chain operations (`launch-dtf`, `distribute-fees`) return unsigned transactions that the agent signs locally with the keypair from `DFM_AGENT_KEYPAIR`. Rebalancing is executed server-side by the admin wallet.

## Agent Wallet -- Keypair Generation

### Wallet path resolution

1. `AGENT_WALLET_PATH` env variable
2. `SOLANA_KEYPAIR_PATH` env variable
3. `WALLET_OUTPUT_PATH` env variable
4. Default: `<project-root>/solana-keypair/keypair.json`

### Implementation

```typescript
import { Keypair } from "@solana/web3.js";
import * as bs58 from "bs58";
import * as fs from "fs";
import * as path from "path";
import * as os from "os";

// 1. Resolve output path
const outPath = path.resolve(
  process.env.AGENT_WALLET_PATH ??
    process.env.SOLANA_KEYPAIR_PATH ??
    process.env.WALLET_OUTPUT_PATH ??
    path.join(os.homedir(), ".dfm", "agent-wallet.json")
);

// 2. Generate keypair and save to file
fs.mkdirSync(path.dirname(outPath), { recursive: true });
const keypair = Keypair.generate();
fs.writeFileSync(outPath, JSON.stringify(Array.from(keypair.secretKey)), { mode: 0o600 });

// 3. Write base58 secret key directly to shell profile (NEVER print to terminal)
const base58Secret = bs58.encode(keypair.secretKey);
const shellProfile = path.join(os.homedir(), ".zshrc");
fs.appendFileSync(shellProfile, `\nexport DFM_AGENT_KEYPAIR="${base58Secret}"\n`);

// 4. Only output the PUBLIC KEY
const pubkey = keypair.publicKey.toBase58();
console.log(`PUBLIC_KEY=${pubkey}`);
console.log(`WALLET_PATH=${outPath}`);
// NEVER console.log the base58Secret — it was written to ~/.zshrc silently
```

**CRITICAL:** The `base58Secret` is written directly to `~/.zshrc` via `fs.appendFileSync`. It must NEVER be printed to stdout, logged, or displayed in any terminal output. Only the public key is shown to the user.

### Deriving public key from env (for API calls)

```typescript
import { Keypair } from "@solana/web3.js";
import * as bs58 from "bs58";

const keypair = Keypair.fromSecretKey(bs58.decode(process.env.DFM_AGENT_KEYPAIR!));
const signerPublicKey = keypair.publicKey.toBase58();
// Use signerPublicKey in API request bodies
```

## API Quick Reference

> For full request/response schemas, see `references/api-reference.md`

| Action | Method | Endpoint | Auth | Body |
|---|---|---|---|---|
| Build vault tx | `POST` | `/launch-dtf` | JWT | `signerPublicKey` + vault config |
| Create DTF (policy + DB) | `POST` | `/dtf-create` | JWT | `transactionSignature` + policy config |
| Vault state | `GET` | `/dtf/:symbol/state` | JWT | - |
| Vault policy | `GET` | `/dtf/:symbol/policy` | JWT | - |
| Rebalance check | `GET` | `/dtf/:symbol/rebalance/check` | JWT | - |
| Rebalance | `POST` | `/dtf/:symbol/rebalance` | JWT | `signerPublicKey` |
| Build distribute fees tx | `POST` | `/dtf/:symbol/distribute-fees` | JWT | `signerPublicKey` |
| Revoke token | `POST` | `/token/revoke` | JWT | - |
| Refresh token | `POST` | `/token/refresh` | No | `profileId` |

### On-chain operations

- **Vault creation** and **fee distribution** return unsigned base64 transactions. The agent signs locally and submits on-chain.
- **Rebalancing** is executed server-side by the admin wallet. The agent only provides its public key for identification.

### Logo handling

- For `launch-dtf` and `dtf-create`, always send `metadataUri: ""`, `logoUrl: ""`, and `bannerUrl: ""`.
- Do not pass image URLs for these fields in launch payloads.

## Using Agent Commands

| What you say | What the agent does |
|---|---|
| `Launch a Solana blue chip fund` | Researches top SOL tokens, picks name/symbol/allocations/policy, builds tx, signs, submits, creates policy |
| `Create a meme token DTF with 3% fee` | Finds trending meme tokens, builds diversified allocation, sets policy, deploys |
| `Show me the state of SOLBC` | `GET /dtf/SOLBC/state` -- returns APY, TVL, NAV, portfolio |
| `Rebalance SOLBC` | Checks policy, triggers server-side rebalance if approved |
| `Distribute fees for SOLBC` | `POST /dtf/SOLBC/distribute-fees` -- builds unsigned tx, signs locally, submits on-chain |
| `Generate a Solana keypair for my DFM Agent wallet` | Creates keypair, exports `DFM_AGENT_KEYPAIR`, reports public key |

## Troubleshooting

| Problem | Fix |
|---|---|
| **"Unauthorized" errors** | Token expired (30-day limit). Refresh via `POST /token/refresh`. Verify: `echo $DFM_AUTH_TOKEN` |
| **"Keypair file not found"** | Re-generate wallet (Step 4). Check: `ls -la $AGENT_WALLET_PATH` |
| **"No signer keypair" / empty DFM_AGENT_KEYPAIR** | `DFM_AGENT_KEYPAIR` not set. Re-export (Step 5). Verify: `echo $DFM_AGENT_KEYPAIR` |
| **Transaction fails on-chain** | Agent Wallet needs SOL for tx fees + USDC for vault creation fee. Fund the wallet first. |
| **Policy validation error** | Read the error message -- it tells you exactly which field/rule failed. Fix and retry. |
| **Token revoked unexpectedly** | A token refresh revokes **all** prior tokens. One active token per agent. |
| **409 Conflict on dtf-create** | A policy already exists for this vault name/symbol. Use a unique name and symbol. |

## Security

- **NEVER display sensitive values in terminal output.** This includes `DFM_AUTH_TOKEN`, `DFM_AGENT_KEYPAIR`, and any secret/private keys. Only public keys may be printed. Secrets are written silently to files.
- **Agent Wallet file** -- `0o600` permissions, gitignored, never committed.
- **`DFM_AGENT_KEYPAIR`** -- base58 secret key in env only, never in code, git, or terminal output.
- **`DFM_AUTH_TOKEN`** -- JWT token in env only, never printed or logged in terminal.
- **No secret keys sent to backend** -- only public keys are included in API payloads. Transaction signing happens locally.
- **Agent Wallet = on-chain authority** -- treat it like any crypto wallet. Back up securely.
- **Policy engine = safety net** -- even a fully autonomous agent can't bypass policy constraints.
