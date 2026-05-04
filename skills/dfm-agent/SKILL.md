---
name: dfm-agent
version: 2.0.0
description: |
  Fully autonomous DTF vault management on Solana. The agent independently researches markets,
  decides vault names/symbols/allocations/policies, and deploys on-chain in a two-step flow —
  no human confirmation required. The policy engine is the guardrail, not a human approval step.
  Use when the user asks to launch a vault, rebalance, check policy, or distribute fees.
homepage: https://qa.dfm.finance
license: MIT
compatibility: Claude Code, Codex, OpenClaw
metadata: {"category":"defi-agent","auth":"jwt+keypair","author":"dfm","tags":"solana,defi,vault,rebalance,dtf,fund-management,autonomous-agent"}
---

# DFM Agent

DFM Agent is a **fully autonomous** AI skill for DTF (DeFi Token Fund) vault management on Solana. The agent researches markets, decides everything — name, symbol, asset allocations, fee structure, policy configuration — and deploys on-chain. No human-in-the-loop confirmation. The **policy engine** is the guardrail, not a human approval step.

## Core Philosophy

**The agent is the creator. It has full authority over what it launches.**

```
+-------------------------------------------------------------------+
|                    AUTONOMOUS AGENT FLOW                           |
|                                                                   |
|  1. RESEARCH    Agent analyzes markets, picks candidate tokens    |
|                 via WebSearch. Then GET /market-metrics for       |
|                 authoritative liquidity/volume numbers (the       |
|                 exact values the policy engine enforces against). |
|                                                                   |
|  2. DECIDE      Agent picks vault name, symbol, assets,           |
|                 allocations, fees, and policy thresholds.         |
|                                                                   |
|  3. VALIDATE    POST /policy/dry-run (loop) -- simulate the       |
|                 basket+policy against all pre-launch rules.       |
|                 No on-chain cost. If violations, adjust policy    |
|                 OR basket, retry until clean.                     |
|                                                                   |
|  4. DEPLOY      Two-step vault creation (policy-gated):           |
|                 a) POST /launch-dtf {basket + policy} ->          |
|                    basket-vs-policy validated server-side.        |
|                    Policy committed. Unsigned tx returned.        |
|                 b) Agent signs tx & submits on-chain.             |
|                 c) POST /dtf-create {tx signature + metadata} ->  |
|                    finalize vault (metadata only, no policy).     |
|                                                                   |
|  5. MANAGE      Agent monitors, rebalances, distributes fees.     |
|                                                                   |
|  GUARDRAILS:    Policy is law before creation. /launch-dtf        |
|                 refuses to build a tx for any basket that         |
|                 violates the agent's own declared policy.         |
|                 NO human confirmation step.                       |
+-------------------------------------------------------------------+
```

| Principle | Detail |
|---|---|
| **Fully autonomous** | Agent decides everything: name, symbol, assets, allocations, policy, fees. No confirmation prompts. |
| **Policy is law before creation** | The `policy` object ships **inside the `/launch-dtf` request body**. Backend runs `evaluatePreCreation` against the basket + policy before building the tx. Violations return `400` with a full `violations[]` array — nothing lands on-chain. |
| **Pre-flight loop via dry-run** | `POST /policy/dry-run` returns the same evaluation without committing anything. Use it to iterate on policy/basket combinations for free before calling `/launch-dtf`. |
| **Metrics source of truth** | `GET /market-metrics` returns the exact `liquidity_usd` / `volume_24h_usd` numbers the policy engine will enforce. Use these values (not aggregator numbers scraped from the web) when choosing `min_amm_liquidity_usd` / `min_24h_volume_usd`. |
| **Two-step vault creation** | `POST /launch-dtf` validates + commits policy + builds unsigned tx. Agent signs + submits. `POST /dtf-create` persists on-chain metadata to DB (no policy — that was already committed and is linked automatically by the chain-event pipeline). |
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
const url = new URL(process.env.DFM_API_URL + "/api/v2/agent/dtf/SYMBOL/state");
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

**IMPORTANT: No timeout on API calls.** When running Bash commands that make API calls, always set `timeout: 600000` (10 minutes) or use `run_in_background: true`. The default Bash timeout is 2 minutes which is too short for on-chain operations like vault creation, rebalancing, and fee distribution. Never let an API call get killed by a timeout.

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

// Reject sentinel/placeholder values that should never be honoured.
const isInvalid = (v) => {
  if (!v || typeof v !== "string") return true;
  const t = v.trim();
  if (!t) return true;
  if (t === "+token+" || t === "<token>" || t.startsWith("+")) return true;
  if (t.startsWith("\"") || t.endsWith("\"")) return true; // stray quotes from bad templating
  return false;
};

let zshrc = "";
try { zshrc = fs.readFileSync(path.join(os.homedir(), ".zshrc"), "utf8"); } catch {}

const envVars = ["DFM_API_URL", "DFM_AUTH_TOKEN", "DFM_AGENT_KEYPAIR", "SOLANA_RPC_URL", "AGENT_WALLET_PATH"];
for (const v of envVars) {
  // 1) Keep settings.json value if already valid (it is updated in-process by refresh / launch scripts).
  if (!isInvalid(settings.env[v])) continue;

  // 2) Pull from ~/.zshrc. Use a global regex and take the LAST match — newest export wins
  //    when the file accumulates multiple `export VAR=...` lines.
  const re = new RegExp("export\\s+" + v + "=[\"\\047]?([^\"\\047\\n]+)[\"\\047]?", "g");
  let lastMatch = null, m;
  while ((m = re.exec(zshrc)) !== null) lastMatch = m;
  if (lastMatch && !isInvalid(lastMatch[1])) {
    settings.env[v] = lastMatch[1];
    continue;
  }

  // 3) Final fallback: current process env
  if (!isInvalid(process.env[v])) settings.env[v] = process.env[v];
  else delete settings.env[v]; // ensure no garbage placeholder lingers
}

fs.mkdirSync(path.dirname(settingsPath), { recursive: true });
fs.writeFileSync(settingsPath, JSON.stringify(settings, null, 2));

for (const v of envVars) {
  console.log(v + "=" + (!isInvalid(settings.env[v]) ? "set" : "NOT SET"));
}
'
```

**If `DFM_AUTH_TOKEN` is SET but any API call returns 401 (Unauthorized / token expired):**

1. Run the **token refresh script below**. The script derives the **agent wallet address** from `DFM_AGENT_KEYPAIR` (no user prompt required), calls `POST {DFM_API_URL}/api/v2/agent/token/refresh-by-wallet` with the agent wallet address, writes the new JWT to `.claude/settings.json`, and **replaces** any existing `export DFM_AUTH_TOKEN=` line in `~/.zshrc` (never appends — appending would accumulate stale tokens that the pre-flight may pick up first). The token value is **never printed**.
2. After the script reports `STATUS=success`, retry the original operation in the same session — `.claude/settings.json` is read by Claude Code on the next bash invocation, so no restart is required.

**DO NOT improvise the refresh.** Earlier improvised attempts have written literal placeholder strings (e.g. `+token+`) into `settings.json`. Always use this exact script.

Write the script once to `.claude/refresh-token.js`, then run it with `node .claude/refresh-token.js`:

```javascript
const http = require("http");
const https = require("https");
const fs = require("fs");
const path = require("path");
const os = require("os");
const { Keypair } = require("@solana/web3.js");
const bs58 = require("bs58").default || require("bs58");

const apiUrl = process.env.DFM_API_URL;
const agentSecret = process.env.DFM_AGENT_KEYPAIR;
if (!apiUrl) { console.log("ERROR: DFM_API_URL not set"); process.exit(1); }
if (!agentSecret) {
  console.log("ERROR: DFM_AGENT_KEYPAIR not set — cannot derive agent wallet for refresh");
  process.exit(1);
}

// Derive the agent's on-chain wallet address from the keypair env var.
const agentWalletAddress = Keypair.fromSecretKey(bs58.decode(agentSecret)).publicKey.toBase58();

const payload = JSON.stringify({ agentWalletAddress });
const url = new URL(apiUrl + "/api/v2/agent/token/refresh-by-wallet");
const client = url.protocol === "https:" ? https : http;

const req = client.request(url, {
  method: "POST",
  headers: { "Content-Type": "application/json", "Content-Length": Buffer.byteLength(payload) }
}, (res) => {
  let data = "";
  res.on("data", (c) => data += c);
  res.on("end", () => {
    try {
      const json = JSON.parse(data);
      // Backend response shape: { status, message, data: { token, expires, expiresPrettyPrint, expiresAt } }
      const token = json.data?.token || json.token;
      if (!token || typeof token !== "string" || token.startsWith("+")) {
        console.log("ERROR: No valid token in response: " + data);
        process.exit(1);
      }

      // (1) Update .claude/settings.json
      const sp = path.join(process.cwd(), ".claude", "settings.json");
      let s = {}; try { s = JSON.parse(fs.readFileSync(sp, "utf8")); } catch {}
      if (!s.env) s.env = {};
      s.env.DFM_AUTH_TOKEN = token;
      fs.mkdirSync(path.dirname(sp), { recursive: true });
      fs.writeFileSync(sp, JSON.stringify(s, null, 2));

      // (2) REPLACE any existing DFM_AUTH_TOKEN export in ~/.zshrc; never append duplicates.
      const zshrcPath = path.join(os.homedir(), ".zshrc");
      let zshrc = "";
      try { zshrc = fs.readFileSync(zshrcPath, "utf8"); } catch {}
      const newLine = "export DFM_AUTH_TOKEN=\"" + token + "\"";
      const lineRe = /^\s*export\s+DFM_AUTH_TOKEN=.*$/gm;
      if (lineRe.test(zshrc)) {
        zshrc = zshrc.replace(lineRe, newLine);
      } else {
        if (zshrc.length && !zshrc.endsWith("\n")) zshrc += "\n";
        zshrc += newLine + "\n";
      }
      fs.writeFileSync(zshrcPath, zshrc);

      // Output ONLY safe info — NEVER the token
      console.log("STATUS=success");
      console.log("AGENT_WALLET=" + agentWalletAddress);
      console.log("DFM_AUTH_TOKEN=set");
    } catch (e) {
      console.log("ERROR: " + e.message);
      process.exit(1);
    }
  });
});
req.on("error", (e) => { console.log("ERROR: " + e.message); process.exit(1); });
req.write(payload);
req.end();
```

**If `DFM_AUTH_TOKEN` is NOT SET after this script**, run the automated agent profile creation flow:

1. Ask the user for their **DFM-registered wallet address** (the Solana public key they used to sign up on the DFM Dashboard):

   > To set up your DFM Agent, I need the **Solana wallet address** you registered on the DFM Dashboard (https://qa.dfm.finance).
   > Please paste your wallet public key.

2. **Ensure the agent wallet exists** — the profile launch requires the agent's wallet public key.
   - If `DFM_AGENT_KEYPAIR` is already set, derive the public key from it (do NOT generate a new one).
   - If a keypair file exists at `AGENT_WALLET_PATH` (default `~/.dfm/agent-wallet.json`), load it and derive the public key.
   - Only if neither exists, generate a new keypair. See "Agent Wallet -- Keypair Generation" section below.

3. Once the user provides the wallet address and the agent keypair exists, **auto-generate** the agent profile name and username:
   - Name: generate a creative agent name (e.g. "Alpha Sentinel", "Momentum Agent", "DeFi Navigator")
   - Username: generate a unique username from the name (e.g. "alpha_sentinel", "momentum_agent")
   - **Don't worry about pre-checking uniqueness** — the `profile-launch.js` script auto-retries with a random 4-char hex suffix on any `409 "Username is already taken"` from the backend (up to 5 attempts). Just pass a sensible base name/username; the script handles collisions silently.

4. **Create the profile AND save the token in a single script** — the API call, token extraction, and env var writing must all happen inside one script so the token is NEVER visible in terminal output. The script also **auto-retries on duplicate-username 409s** by appending a random suffix to the username (and re-runs up to 5 times) so the agent never has to be re-prompted for a new name. Write a script file and execute it:

   Write a file called `.claude/profile-launch.js` with this content, then run it with `node .claude/profile-launch.js`:

   ```javascript
   const http = require("http");
   const https = require("https");
   const fs = require("fs");
   const path = require("path");
   const os = require("os");
   const crypto = require("crypto");
   const { Keypair } = require("@solana/web3.js");
   const bs58 = require("bs58").default || require("bs58");

   const apiUrl = process.env.DFM_API_URL;
   const walletAddress = process.argv[2];
   const baseName = process.argv[3];
   const baseUsername = process.argv[4];

   if (!apiUrl) { console.log("ERROR: DFM_API_URL not set"); process.exit(1); }
   if (!walletAddress || !baseName || !baseUsername) {
     console.log("ERROR: usage: node profile-launch.js <walletAddress> <name> <username>");
     process.exit(1);
   }

   // Derive agent wallet public key from DFM_AGENT_KEYPAIR
   const agentKeypair = Keypair.fromSecretKey(bs58.decode(process.env.DFM_AGENT_KEYPAIR));
   const agentWalletAddress = agentKeypair.publicKey.toBase58();

   const MAX_ATTEMPTS = 5;
   const UNAME_RX = /username/i; // disambiguates duplicate-username vs. wallet-already-has-agent

   // Sanitize base username to allowed charset (alphanumeric + underscore)
   const sanitize = (s) => s.toLowerCase().replace(/[^a-z0-9_]/g, "_").replace(/_+/g, "_").replace(/^_|_$/g, "");
   const cleanBase = sanitize(baseUsername) || "agent";

   // Suffix generator: deterministic-looking but unique. 4 hex chars = 65k combos.
   const suffix = () => crypto.randomBytes(2).toString("hex");

   function attempt(usernameToTry, agentNameToTry, n) {
     const payload = JSON.stringify({
       userPublicKey: walletAddress,
       agentWalletAddress: agentWalletAddress,
       name: agentNameToTry,
       username: usernameToTry,
       metadata: [{ key: "created_by", value: "dfm-agent-skill" }]
     });

     const url = new URL(apiUrl + "/api/v2/agent/profile-launch");
     const client = url.protocol === "https:" ? https : http;

     const req = client.request(url, {
       method: "POST",
       headers: { "Content-Type": "application/json", "Content-Length": Buffer.byteLength(payload) }
     }, (res) => {
       let data = "";
       res.on("data", (chunk) => data += chunk);
       res.on("end", () => {
         let json = null;
         try { json = JSON.parse(data); } catch { /* fall through */ }
         const message = json?.message || json?.error || data;

         // 409 + duplicate-username -> regenerate username, retry. Do NOT retry on
         // "agent profile already exists for this wallet" — that is unrecoverable here.
         if (res.statusCode === 409 && UNAME_RX.test(String(message)) && !/wallet/i.test(String(message))) {
           if (n >= MAX_ATTEMPTS) {
             console.log("ERROR: username conflicts after " + MAX_ATTEMPTS + " attempts. Last tried: " + usernameToTry);
             process.exit(1);
           }
           const nextUsername = cleanBase + "_" + suffix();
           const nextName = baseName; // keep display name; only username needs to be unique
           console.log("RETRY=username_taken attempt=" + (n + 1) + " next_username=" + nextUsername);
           return attempt(nextUsername, nextName, n + 1);
         }

         if (res.statusCode === 409) {
           console.log("ERROR: 409 Conflict (not username-related): " + message);
           process.exit(1);
         }

         if (res.statusCode < 200 || res.statusCode >= 300) {
           console.log("ERROR: HTTP " + res.statusCode + ": " + message);
           process.exit(1);
         }

         const token = json?.data?.token?.token;
         if (!token) { console.log("ERROR: No token in response. Response: " + data); process.exit(1); }

         // Write to .claude/settings.json
         const sp = path.join(process.cwd(), ".claude", "settings.json");
         let s = {}; try { s = JSON.parse(fs.readFileSync(sp, "utf8")); } catch {}
         if (!s.env) s.env = {};
         s.env.DFM_AUTH_TOKEN = token;
         fs.writeFileSync(sp, JSON.stringify(s, null, 2));

         // REPLACE any existing DFM_AUTH_TOKEN export in ~/.zshrc; never append duplicates.
         const zshrcPath = path.join(os.homedir(), ".zshrc");
         let zshrc = "";
         try { zshrc = fs.readFileSync(zshrcPath, "utf8"); } catch {}
         const newLine = "export DFM_AUTH_TOKEN=\"" + token + "\"";
         const lineRe = /^\s*export\s+DFM_AUTH_TOKEN=.*$/gm;
         if (lineRe.test(zshrc)) {
           zshrc = zshrc.replace(lineRe, newLine);
         } else {
           if (zshrc.length && !zshrc.endsWith("\n")) zshrc += "\n";
           zshrc += newLine + "\n";
         }
         fs.writeFileSync(zshrcPath, zshrc);

         // Only output safe info — NEVER the token
         const profileName = json.data?.agentProfile?.name || agentNameToTry;
         const profileUsername = json.data?.agentProfile?.username || usernameToTry;
         console.log("STATUS=success");
         console.log("AGENT_NAME=" + profileName);
         console.log("AGENT_USERNAME=" + profileUsername);
         console.log("AGENT_WALLET=" + agentWalletAddress);
         console.log("ATTEMPTS=" + n);
         console.log("DFM_AUTH_TOKEN=set");
       });
     });
     req.on("error", (e) => { console.log("ERROR: " + e.message); process.exit(1); });
     req.write(payload);
     req.end();
   }

   attempt(cleanBase, baseName, 1);
   ```

   Run it as: `node .claude/profile-launch.js <WALLET_ADDRESS> "<AGENT_NAME>" "<AGENT_USERNAME>"`

   The script derives `agentWalletAddress` from `DFM_AGENT_KEYPAIR` automatically and includes it in the payload. It outputs ONLY `STATUS=success`, `AGENT_NAME=...`, `AGENT_USERNAME=...`, `AGENT_WALLET=...`, `ATTEMPTS=<n>`, and `DFM_AUTH_TOKEN=set`. The actual token value is written directly to `.claude/settings.json` and `~/.zshrc` — it NEVER appears in terminal output.

   **Username conflict retry behavior:**
   - On HTTP `409` whose message references "username" (e.g. `"Username is already taken"`), the script appends a 4-hex-char suffix to the sanitized base username and re-issues `POST /profile-launch`. Up to **5 attempts**.
   - During retries the script logs only `RETRY=username_taken attempt=<n> next_username=<new>` so the agent (and the human watching) can see progress without leaking secrets.
   - The display `name` is preserved across retries — only `username` changes.
   - On HTTP `409` whose message references "wallet" (e.g. `"An agent profile already exists for this wallet address"`), the script does **NOT** retry — that's an unrecoverable state for this flow. The agent should call `/token/refresh-by-wallet` instead.
   - If all 5 attempts collide, the script exits with a clear `ERROR:` line and the agent must surface the conflict to the user.

5. Tell the user: "Agent profile created! Restart your AI agent to pick up the auth token, then you're ready to go."

**If `DFM_AGENT_KEYPAIR` is NOT SET** and the operation requires signing (launch-dtf, distribute-fees), auto-generate a wallet (see "Agent Wallet -- Keypair Generation" section below). Do not ask — just generate it.

### Step 2: Proceed

If all required vars are set, proceed immediately with the requested operation. **Do not ask for confirmation.**

**IMPORTANT:** After writing to `.claude/settings.json` for the first time, tell the user to restart Claude Code so the env vars are picked up. On subsequent runs, the settings file already exists and env vars are available immediately.

## When To Use

Use this skill when:
- The user asks to launch, create, or deploy a DTF vault
- The user asks to research tokens/markets and create a fund
- The user wants to check vault state, policy, or rebalancing status
- The user asks to rebalance a vault or distribute fees
- The user asks to **deposit USDC into a vault** or **redeem vault tokens** (capital flows are agent-bound — see "Step 7: Capital Flows")
- The user needs to set up agent auth (wallet generation, token management)

## Autonomous DTF Launch -- How It Works

When the user asks you to launch a DTF (e.g. "Create a blue chip Solana fund" or "Launch a meme token DTF"), follow this autonomous workflow:

### Step 1: Research (Agent decides)

Use `WebSearch` and `WebFetch` for **token discovery** — prices, market caps, mint addresses, trending assets, macro conditions. Then use `GET /market-metrics` as the **authoritative source** for the two numbers the policy engine actually enforces (Rule 2 liquidity, Rule 3 24h volume). Aggregator scrape values often disagree with what the backend sees; `/market-metrics` is Jupiter data queried through the same pipeline the evaluator uses, so the numbers match exactly.

Use `WebSearch` to find:
- Top performing Solana tokens by market cap, volume, and price action
- Current market conditions, trends, and sentiment
- **For yield DTFs**: Solana LSTs (liquid staking tokens) and yield-bearing tokens — mSOL, jitoSOL, bSOL, INF, hSOL, stSOL, and their current APYs, TVLs, and staking yields

Use `WebFetch` to pull data from:
- Token data aggregators (CoinGecko, CoinMarketCap, Jupiter aggregator, Birdeye, DexScreener)
- Solana token lists and verified registries for mint addresses
- **For yield DTFs**: Staking yield aggregators, LST protocol sites (Marinade, Jito, BlazeStake, Sanctum), and DeFi yield dashboards

**Then call `GET {DFM_API_URL}/api/v2/agent/market-metrics`** with the candidate assets (mints, symbols, or names) to get the exact policy-relevant numbers:

```
GET /api/v2/agent/market-metrics?symbols=SOL,JUP&names=Bonk
Authorization: Bearer <DFM_AUTH_TOKEN>
```

Response:
```json
{
  "metrics": [
    {
      "mintAddress": "So11111111111111111111111111111111111111112",
      "symbol": "SOL",
      "name": "Wrapped SOL",
      "liquidity_usd": 691807448.19,
      "volume_24h_usd": 14648499808.98,
      "price_usd": 85.33,
      "holder_count": 3820662,
      "policyRelevant": {
        "liquidity_usd": 691807448.19,
        "volume_24h_usd": 14648499808.98
      }
    }
  ],
  "unresolved": []
}
```

Use these numbers to:
- **Drop candidates that don't meet the strategy's floor** (e.g. reject an asset with `liquidity_usd < 50000` for an aggressive fund).
- **Calibrate `min_amm_liquidity_usd` and `min_24h_volume_usd`** in the policy with a **safety buffer of 30–50% below the weakest selected asset's `/market-metrics` number**. Jupiter snapshots fluctuate intraday — setting the floor *at* the weakest asset's current value guarantees the asset will dip below the floor on a bad snapshot and become "locked" in the basket (the agent can no longer remove it via `/update-assets-tx`, because any intermediate basket that still contains the locked asset fails the volume gate). Concrete rule: **floor = floor(weakest_asset_snapshot × 0.5)**, rounded down to a clean number.
- **Null values** in `policyRelevant` indicate a transient Jupiter fetch miss; retry after a cache warm-up (the endpoint caches per mint).

Then decide:
- Identify candidate tokens based on the user's intent or strategy
- **For yield DTFs**: Prioritize LSTs and yield-bearing assets with highest APY and deepest liquidity
- Select the final basket and determine allocations
- Automatically discover each token's Solana `mintAddress` from reliable references (official docs, verified token lists, explorers, major data providers)
- Cross-check mint addresses across multiple references before including them in `underlyingAssets`
- Reject unverified, conflicting, or low-confidence mint mappings and replace them with verified assets

### Step 2: Design (Agent decides)

Based on your research, autonomously decide:
- **Vault type** -- `"DTF"` for standard diversified token funds, `"YIELD_DTF"` for yield-bearing / LST-focused funds. Set in the `dtf-create` payload.
- **Vault name** -- descriptive, catchy, relevant to the strategy
- **Vault symbol** -- short (max 10 chars), unique, memorable
- **Underlying assets** -- pass asset `symbol` or `name` (preferred) with allocation in basis points (must sum to 10000). Backend resolves `mintAddress` from `asset-allocation`.
  - For **DTF**: standard tokens (SOL, JUP, Bonk, RAY, etc.)
  - For **YIELD_DTF**: LSTs and yield-bearing tokens (mSOL, jitoSOL, bSOL, INF, etc.)
- **Management fees** -- in basis points (e.g. 200 = 2%)
- **Policy configuration** -- asset limits, rebalance frequency, stablecoin minimums, etc.
- **Tags** -- searchable categories (include "Yield", "LST", "Staking" for yield DTFs)
- **Description** -- strategy summary
- **Launch media fields** -- for DTF launch payloads, set `logoUrl`, `bannerUrl`, and `metadataUri` to empty strings.

### Step 3: Validate (Pre-flight dry-run)

Before calling `/launch-dtf`, run the proposed basket + policy through `POST /policy/dry-run`. This is **free** (no DB write, no on-chain cost) and returns every violation at once so the agent can fix them in one pass. **Loop until `ok: true` or `violations: []`.**

```bash
POST {DFM_API_URL}/api/v2/agent/policy/dry-run
Authorization: Bearer <DFM_AUTH_TOKEN>

{
  "underlyingAssets": [
    { "symbol": "SOL",  "pct_bps": 4000 },
    { "symbol": "JUP",  "pct_bps": 3000 },
    { "name":   "Bonk", "pct_bps": 3000 }
  ],
  "policy": {
    "asset_mode": "OPEN",
    "min_amm_liquidity_usd": 100000,
    "min_24h_volume_usd": 500000,
    "min_assets": 3,
    "max_assets": 12,
    "max_asset_pct": 4000,
    "min_asset_pct": 500
  }
}
```

Clean response (proceed to Step 4):
```json
{ "ok": true, "policyCheck": { "ok": true, "violations": [] } }
```

Flagged response — **all** violations returned together:
```json
{
  "ok": false,
  "policyCheck": {
    "ok": false,
    "violations": [
      {
        "violationCode": "rule2MinAmmLiquidity",
        "message": "Mint Bonk... has $30000 AMM liquidity; policy requires at least $100000.",
        "details": { "mint": "Dez...", "observedUsd": 30000, "minUsd": 100000 }
      },
      {
        "violationCode": "rule4MinMaxAssetCount",
        "message": "Proposed 3 distinct assets; policy requires between 5 and 12.",
        "details": { "distinctAssetCount": 3, "minAllowed": 5, "maxAllowed": 12 }
      }
    ]
  }
}
```

**Fix strategy (priority order):**
1. `rule2MinAmmLiquidity` / `rule3Min24hVolume` → lower the policy threshold (if the asset is strategy-critical) OR swap the asset out. Don't set the threshold above what `/market-metrics` reports for any included asset.
2. `rule4MinMaxAssetCount` → adjust basket size OR the `min_assets`/`max_assets` bounds.
3. `rule5MaxPctPerAsset` / `rule6MinPctPerAssetIfHeld` → rebalance the `pct_bps` allocations.
4. `rule1WhitelistBlacklist` / `assetModeViolation` → fix `asset_mode`, `asset_whitelist`, or `asset_blacklist`.

**Re-run dry-run after every fix.** Only proceed to `/launch-dtf` once dry-run returns clean. Max ~3 iterations in practice; if you can't converge, surface the blocker to the user before incurring on-chain costs.

### Step 4: Deploy (Two-step flow — policy-gated)

#### 4a. Build the unsigned transaction (with policy)

Send `POST {DFM_API_URL}/api/v2/agent/launch-dtf` with the vault creation payload **AND the policy**. Backend runs the same pre-creation evaluation as `/policy/dry-run`, commits the policy (unlinked, keyed by vault_name + vault_symbol), and returns the unsigned tx. If any rule violates, it returns `400` with the full `violations[]` — **no tx is built and nothing lands on-chain**, so the agent can fix and retry.

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
  "threshold": 500,
  "policy": {
    "asset_mode": "OPEN",
    "asset_whitelist": [],
    "asset_blacklist": [],
    "min_amm_liquidity_usd": 100000,
    "min_24h_volume_usd": 500000,
    "min_assets": 2,
    "max_assets": 8,
    "max_asset_pct": 4000,
    "min_asset_pct": 500,
    "min_stablecoin_pct": 0,
    "max_rebalance_pct": 7500,
    "min_rebalance_interval_hours": 4,
    "max_rebalances_per_day": 3,
    "max_rebalances_per_week": 14,
    "launch_blackout_hours": 24,
    "fee_locked": true,
    "notes": "Auto-generated policy for blue chip strategy"
  }
}
```

**Headroom rationale (do not copy values blindly):** the launch-time policy must allow the *future* basket migrations the agent will need to perform. See "Future-proofing checklist" below — the agent must run that checklist against the proposed policy *before* sending `/launch-dtf`.

Successful response:
```json
{
  "onChain": {
    "transaction": "base64-encoded-unsigned-versioned-transaction...",
    "vaultIndex": 42,
    "vaultPda": "7Xk...def",
    "vaultMintPda": "9Rm...ghi"
  },
  "policyId": "665c..."
}
```

Policy-violation response (400) — **same shape as dry-run**:
```json
{
  "statusCode": 400,
  "message": "Policy validation failed",
  "ok": false,
  "violations": [
    { "violationCode": "rule2MinAmmLiquidity", "message": "...", "details": {...} }
  ]
}
```

#### 4b. Sign and submit the transaction on-chain

Use the agent's local keypair to sign the returned transaction and submit it to Solana:

```typescript
import { Keypair, VersionedTransaction, Connection } from "@solana/web3.js";
const bs58 = require("bs58").default || require("bs58");

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

#### 4c. Persist vault metadata to DB

After the on-chain transaction confirms, send `POST {DFM_API_URL}/api/v2/agent/dtf-create` with the transaction signature and **metadata only**. Policy was already committed during `/launch-dtf` and is linked to the new vault automatically by the chain-event pipeline. Do **not** send policy fields here — they are rejected.

```json
{
  "transactionSignature": "<signature from step 4b>",
  "vaultName": "Solana Blue Chips",
  "vaultSymbol": "SOLBC",
  "vaultType": "DTF",
  "description": "Top-tier Solana ecosystem tokens",
  "tags": ["Blue Chip", "Solana", "DeFi"],
  "logoUrl": "",
  "bannerUrl": "",
  "noRebalance": false
}
```

Successful response:
```json
{
  "vault": [
    {
      "eventType": "VaultCreated",
      "vault": {
        "vaultName": "Solana Blue Chips",
        "vaultSymbol": "SOLBC",
        "vaultIndex": 42,
        "status": "active"
      }
    }
  ],
  "policyId": "665c..."
}
```

**Example: WHITELIST mode (for curated LST yield fund)** — the policy fields move to `/launch-dtf`. `/dtf-create` still only carries metadata:

Step 4a `/launch-dtf` body (policy for curated LST fund):
```json
{
  "signerPublicKey": "<agent pubkey>",
  "vaultName": "Solana LST Yield",
  "vaultSymbol": "SLSTY",
  "underlyingAssets": [
    { "symbol": "mSOL", "mintBps": 4000 },
    { "symbol": "jitoSOL", "mintBps": 3500 },
    { "symbol": "bSOL", "mintBps": 2500 }
  ],
  "managementFees": 150,
  "category": 0,
  "policy": {
    "asset_mode": "WHITELIST_ONLY",
    "asset_whitelist": [
      "mSoLzYCxHdYgdzU16g5QSh3i5K3z3KZK7ytfqcJm7So",
      "J1toso1uCk3RLmjorhTtrVwY9HJ7X8V9yYac6Y7kGCPn",
      "bSo13r4TkiE4KumL71LsHTPpL2euBYLFx6h9HP3piy1"
    ],
    "asset_blacklist": [],
    "min_amm_liquidity_usd": 500000,
    "min_24h_volume_usd": 500000,
    "min_assets": 2,
    "max_assets": 8,
    "max_asset_pct": 5000,
    "min_asset_pct": 1000,
    "min_stablecoin_pct": 0,
    "max_rebalance_pct": 2000,
    "min_rebalance_interval_hours": 12,
    "max_rebalances_per_day": 1,
    "max_rebalances_per_week": 5,
    "launch_blackout_hours": 24,
    "fee_locked": true,
    "notes": "Whitelisted LST-only yield fund"
  }
}
```

Step 4c `/dtf-create` body (metadata only):
```json
{
  "transactionSignature": "<signature>",
  "vaultName": "Solana LST Yield",
  "vaultSymbol": "SLSTY",
  "vaultType": "YIELD_DTF",
  "description": "Diversified Solana liquid staking token yield fund",
  "tags": ["Yield", "LST", "Staking", "Solana"],
  "logoUrl": "",
  "bannerUrl": "",
  "noRebalance": false
}
```

#### Policy field decision guide

The agent MUST decide ALL policy values based on the vault strategy. **These values go into the `policy` sub-object of `/launch-dtf` (Step 4a) — not into `/dtf-create`.** Calibrate every threshold with **headroom for future basket migrations** — a vault whose own policy makes its basket immutable cannot be rescued by the agent. See "Future-proofing checklist" below before sending `/launch-dtf`.

| Strategy Type | `max_asset_pct` | `min_asset_pct` | `min_amm_liquidity_usd` | `min_24h_volume_usd` | `max_rebalances_per_day` | `min_rebalance_interval_hours` |
|---|---|---|---|---|---|---|
| **Conservative** (blue chip, index) | 3000-4000 | 500-1000 | 500000 | 1000000 | 2 | 6 |
| **Moderate** (mixed, ecosystem) | 4000-5000 | 500 | 100000 | 500000 | 3 | 4 |
| **Aggressive** (meme, trending) | 5000-6000 | 300 | 50000 | 100000 | 4 | 2 |
| **Yield** (LSTs, staking, yield) | 4000-5000 | 500-1000 | 500000 | 500000 | 1 | 12 |

The liquidity/volume floors above are **strategy ceilings, not target values**. Always also apply the **0.5× safety buffer** rule against the weakest included asset's actual `/market-metrics` snapshot — pick whichever is *lower* of (strategy table value, weakest_asset_snapshot × 0.5).

Always set:
- `asset_mode`: choose based on the vault strategy:
  - `"OPEN"` — any asset can be added. No restrictions. Use for broad market / index / aggressive strategies.
  - `"WHITELIST_ONLY"` — only assets in `asset_whitelist` are allowed. Use for curated funds (e.g. "only blue chips", "only LSTs"). Set `asset_whitelist` to the mint addresses of the selected assets.
  - `"OPEN_BLACKLIST"` — all assets allowed except those in `asset_blacklist`. Use when you want to exclude specific risky assets. Set `asset_blacklist` to the mint addresses to exclude.
  - `"WHITELIST_BLACKLIST"` — only whitelisted assets allowed, with additional blacklist exclusions. Use for strict curated funds with explicit exclusions. Set both `asset_whitelist` and `asset_blacklist`.
  - **Decision rule:** If the user asks for a specific category fund (e.g. "LST fund", "blue chip only", "top 5 DeFi tokens"), use `WHITELIST_ONLY`. If the user asks for broad exposure, use `OPEN`. If the user says "exclude meme coins" or similar, use `OPEN_BLACKLIST`.
- `min_assets`: **`max(1, launch_basket_count − 1)`**. Never set `min_assets == launch_basket_count` — that traps the basket at exactly that size, with no room to drop a single asset on a future migration. (TOP3S launched with `min_assets: 3, max_assets: 3` — locked forever to a 3-asset basket and any single-asset swap exceeds the per-update cap.)
- `max_assets`: **`min(12, launch_basket_count + 3)`**, with `12` as the hard ceiling. Always leave room to grow the basket by 2–3 assets.
- `max_rebalance_pct`: **`5000`–`7500` (50–75%)** for any basket of ≤4 assets; **`3000`–`5000`** is acceptable only for baskets of ≥6 assets where individual weights are already small. **Never set below the value of `2 × max_asset_pct`** — replacing one asset moves *both* the outgoing weight and the incoming weight, so the aggregate movement is roughly twice the swapped weight. Setting `max_rebalance_pct < 2 × max_asset_pct` makes any single-asset swap structurally impossible. (TOP3S launched with `max_rebalance_pct: 2500` and `max_asset_pct: 5500` — every JUP→ETH swap is mathematically blocked.)
- `max_rebalances_per_week`: `max_rebalances_per_day * 7` or less
- `launch_blackout_hours`: `24` (prevent rebalancing in first 24h)
- `fee_locked`: `true` for index / yield / curated mandates where the management-fee promise is part of the product. Set `false` if the strategy may need to adjust fees as TVL grows (you can always lock later via a policy update; you cannot unlock if the fee was locked at launch).
- `notes`: brief description of the strategy and policy rationale

#### Future-proofing checklist (run BEFORE `/launch-dtf`)

The agent must mentally tick each of these against the proposed `policy` + basket. Failing any single one is grounds to **adjust the policy and re-run dry-run** — never launch a vault that fails this checklist. The cost of getting it wrong is **terminal**: a misconfigured constitutional policy cannot be relaxed by the agent and effectively bricks the vault for migrations.

1. **Asset-count headroom** — Is `max_assets ≥ launch_basket_count + 2` AND `min_assets ≤ launch_basket_count − 1`? If no → widen the bounds.
2. **Single-swap feasibility** — Is `max_rebalance_pct ≥ 2 × max_asset_pct`? Equivalent question: if I had to replace the largest-weight asset tomorrow, would the resulting allocation movement fit under `max_rebalance_pct`? If no → raise `max_rebalance_pct` (preferred) or lower `max_asset_pct`.
3. **Liquidity/volume buffer** — Is `min_amm_liquidity_usd ≤ weakest_asset_snapshot.liquidity_usd × 0.5` AND `min_24h_volume_usd ≤ weakest_asset_snapshot.volume_24h_usd × 0.5`? If no → halve the floors. Snapshots fluctuate; the floor must absorb a 50% intraday dip without locking the asset.
4. **Removability** — For every asset currently in the basket, simulate a basket without that asset: does the simulated basket still satisfy `min_assets`, `max_asset_pct`, and `min_asset_pct`? If any asset is non-removable, the vault is one bad volume snapshot away from being permanently stuck.
5. **Catch-22 defence** — If any included asset's `/market-metrics` snapshot is within 30% of the proposed `min_24h_volume_usd` or `min_amm_liquidity_usd`, **lower the floor further or swap the asset**. An asset that hovers near the floor will eventually fall below it, and then no intermediate basket containing it can pass the volume gate, blocking the migration entirely.

If the user's stated strategy (e.g. "exactly 3 concentrated picks") seems to want a rigid policy, do not encode rigidity by tightening the policy bounds. The strategy *intent* belongs in the basket and `notes`; the policy *bounds* belong wide enough to keep the vault manageable.

#### Backend payload rules

For `/launch-dtf` payloads:
- `category: 0` (Manual) for agent-created vaults.
- In `underlyingAssets`, send `symbol` or `name` (preferred). Backend resolves `mintAddress` from `asset-allocation`.
- Hard restriction: never include USDC (`symbol: "USDC"` or `name: "USD Coin"`) in `underlyingAssets`.
- If a candidate list contains USDC, remove it and replace it with another eligible non-USDC asset before sending `launch-dtf`.
- **Asset count: minimum 1, maximum 12 assets** in `underlyingAssets`. The backend rejects payloads outside this range.
- **`policy` object is REQUIRED**. Include every field the agent wants enforced. Omitted fields default to 0/disabled.
- The basket in `underlyingAssets` is evaluated against `policy` server-side (same rules as `/policy/dry-run`). If any rule fails, `/launch-dtf` returns `400` with `violations[]` and no tx is built.

For `/dtf-create` payloads:
- **METADATA ONLY.** Do NOT send any policy fields (`asset_mode`, `asset_whitelist`, `min_amm_liquidity_usd`, etc.). The policy is already committed during `/launch-dtf` and linked automatically.
- Keep only: `transactionSignature`, `vaultName`, `vaultSymbol`, `vaultType`, `description`, `tags`, `logoUrl`, `bannerUrl`, `noRebalance`.
- Set `vaultType`: `"DTF"` for standard funds, `"YIELD_DTF"` for yield/LST funds.
- Set `logoUrl`, `bannerUrl` to empty strings.
- `vaultName` and `vaultSymbol` must match what was used in `launch-dtf`.

#### Signing helper for API calls that return unsigned transactions

Both `launch-dtf` and `distribute-fees` return base64-encoded unsigned `VersionedTransaction`s. Use this pattern to sign and submit:

```typescript
import { Keypair, VersionedTransaction, Connection } from "@solana/web3.js";
const bs58 = require("bs58").default || require("bs58");

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

**CRITICAL ERROR HANDLING RULES for vault deployment:**

1. **NEVER call `launch-dtf` again after a transaction has been signed and submitted on-chain.** The on-chain vault creation is irreversible and costs USDC. If `dtf-create` fails, only retry `dtf-create` with the SAME transaction signature, vault name, and symbol. Do NOT generate a new name/symbol or call `launch-dtf` again.

2. **If `launch-dtf` returns 400 with policy `violations[]`** (before any on-chain submission): no policy was committed, no tx was built, no on-chain cost was incurred. Fix the basket OR policy per the violation codes (same guidance as Step 3 dry-run) and retry `launch-dtf`. Consider running `/policy/dry-run` first to iterate cheaply.

3. **If `launch-dtf` fails for other reasons** (e.g. validation error, asset not found, insufficient USDC): fix the payload and retry `launch-dtf` — still no on-chain cost since the tx hasn't been built.

4. **If signing/submission fails**: you MAY retry `launch-dtf` to get a new unsigned transaction. The policy was already committed on the first call — **reuse the exact same `vaultName` and `vaultSymbol`** so the existing unlinked policy is picked up (changing them would commit a second policy with a different name).

5. **If `dtf-create` fails** (after successful on-chain submission): ONLY retry `dtf-create` with the exact same `transactionSignature`, `vaultName`, and `vaultSymbol`. NEVER change these values. NEVER call `launch-dtf` again. The policy is already committed and will be linked by the chain-event pipeline regardless.

6. **Keep the transaction signature**: after a successful on-chain submission, store the signature and reuse it for all `dtf-create` retries. This is the link between the on-chain vault and the database record.

### Step 5: Manage (Ongoing, autonomous)

After launch, the agent autonomously:
- Lists the user's vaults via `GET {DFM_API_URL}/api/v2/agent/vaults/user?page=1&limit=10&vaultType=dtf&includeTvl=true` (paginated; switch `vaultType=yield_dtf` for yield funds)
- Browses platform-featured vaults via `GET {DFM_API_URL}/api/v2/agent/vaults/featured/list?page=1&limit=10&vaultType=dtf&includeTvl=true`
- Monitors vault state via `GET {DFM_API_URL}/api/v2/agent/dtf/:symbol/state`
- Checks rebalancing readiness via `GET {DFM_API_URL}/api/v2/agent/dtf/:symbol/rebalance/check`
- Executes rebalances via `POST {DFM_API_URL}/api/v2/agent/dtf/:symbol/rebalance` (admin wallet executes behind the scenes)
- Distributes accrued fees via `POST {DFM_API_URL}/api/v2/agent/dtf/:symbol/distribute-fees` (returns unsigned tx for agent to sign)

**Listing vaults (`/vaults/user`, `/vaults/featured/list`):** both endpoints take the same four query params — `page`, `limit`, `vaultType` (`dtf` | `yield_dtf`), `includeTvl`. Use `/vaults/user` to enumerate the caller's own vaults with pagination and type filtering (preferred over the legacy unpaginated `/dtf/my-vaults`); use `/vaults/featured/list` to surface featured vaults across the platform. Always send `includeTvl=true` — both endpoints must return `totalValueLocked` and `sharePrice`. Iterate `page` until `pagination.hasNext` is `false` (or `pagination.page >= pagination.totalPages`).

**Mapping natural-language requests to `page`:** translate the user's phrasing literally into the `page` query param.

| User says | Send |
|---|---|
| "show me my vaults" / "list my DTFs" | `?page=1&limit=10&...` (always start at page 1) |
| "show me the **second page**" / "page 2" | `?page=2&limit=10&...` |
| "page 5 of featured vaults" | `?page=5&limit=10&...` (against `/vaults/featured/list`) |
| "**next page**" / "show me more" | `?page=<lastShown + 1>&...`. Refuse and tell the user "you're on the last page" if the previous response had `pagination.hasNext === false`. |
| "**previous page**" / "go back" | `?page=<lastShown - 1>&...`. If `lastShown` was already 1, return the same page-1 results (don't let `page` go below 1). |
| "first page" / "go back to the start" | `?page=1&...` |
| "last page" | First call `page=1` to read `pagination.totalPages`, then call `?page=<totalPages>&...` |
| "show me 25 per page" / "fetch 50 at a time" | Forward `limit` as given (clamp to a sensible max like 100 if abused). Reset `page` to 1 when `limit` changes. |

**Remembering pagination state across turns:** the skill is stateless — there's no built-in "current page" memory. Track the last `pagination.page` and `pagination.totalPages` you returned **in your own conversation context** so that "next page" / "previous page" requests resolve correctly. If the user switches filters (`vaultType`, `search`, `limit`), reset to `page=1` because the result set has changed and the old page index no longer maps to the same data.

**After fetching a page**, surface the navigation footer to the user: e.g. "Page 2 of 5 — say 'next page' for more, or 'page N' to jump." Use `pagination.hasNext` / `pagination.hasPrev` to decide which controls to mention. Never hide pagination from the user — if `pagination.totalPages > 1` they need to know more pages exist.

**Response shape (both endpoints):** `{ data: Vault[], pagination: { page, limit, total, totalPages, hasNext, hasPrev } }`. Each `Vault` carries `vaultName`, `vaultSymbol`, `vaultAddress`, `description`, `vaultIndex`, `tags[]`, `feeConfig.managementFeeBps`, `underlyingAssets[]` (each with nested `assetAllocation: { name, symbol, logoUrl }` and `pct_bps`), `creator` (rich profile object: `name`, `walletAddress`, `avatar`, `twitter_username`), `category: { name }`, `totalValueLocked`, `sharePrice`, `vaultApy`, `performance7d`, plus string-typed `nav` and `totalSupply`. See `references/api-reference.md` § 12a for the full field table.

**When displaying vault lists to the user, surface the readable fields** — `vaultName` / `vaultSymbol`, `description`, the asset basket as a comma-separated `symbol pct%` summary (divide `pct_bps` by 100), `totalValueLocked` formatted as USD, `performance7d` as a percentage, `feeConfig.managementFeeBps / 100` as the fee %, and `creator.name` (or `creator.twitter_username`) as the author. **Skip `_id`, `id`, raw `pct_bps`, internal Mongo fields, and `daoconfig: null`.** `nav` and `totalSupply` are decimal-safe strings — parse with `Number()` before any math, and treat `"0"` / `null` `vaultApy` / `null` `performance7d` as "no data yet" for new vaults.

All management operations are single API calls. No confirmation needed.

### Step 6: Update Underlying Assets (Four-Phase Flow)

When the user says **"update underlying"**, **"change the basket"**, **"swap assets"**, **"rebalance to X"**, or any phrasing that means "replace the vault's `underlyingAssets` allocations", run this four-phase flow. **Do not ask for confirmation between Phases 1–3.** Phase 4 (rebalance) is mandatory after a basket change, but ask the user once before executing it.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  UPDATE UNDERLYING — four-phase flow                                      │
│                                                                          │
│  Phase 1: DECIDE THE NEW BASKET                                          │
│    - WebSearch / WebFetch + GET /market-metrics → candidate assets       │
│    - Match against the vault's existing constitutional policy            │
│      (GET /dtf/:symbol/policy → asset_mode, whitelist, min liquidity)    │
│    - Choose `mintBps` allocations summing to 10000                       │
│    - Prefer `symbol` / `name` identifiers — backend resolves to mints    │
│                                                                          │
│  Phase 2: BUILD ON-CHAIN TX (POLICY-GATED, SERVER-SIDE)                  │
│    POST /api/v2/agent/vaults/:symbol/update-assets-tx                    │
│      { signerPublicKey, underlyingAssets: [{ symbol, mintBps }, ...] }   │
│                                                                          │
│      ├─ 400 + violations[] ─▶ Phase 1 with adjustments (loop)            │
│      └─ 201 + base64 tx     ─▶ Phase 3                                   │
│                                                                          │
│    Sign locally with DFM_AGENT_KEYPAIR → submit on-chain → confirm       │
│                                                                          │
│  Phase 3: SYNC DB                                                        │
│    PATCH /api/v2/agent/vaults/:symbol/underlying-assets-by-mint          │
│      { underlyingAssets: [{ mintAddress, pct_bps }, ...] }               │
│      (Note: pct_bps, NOT mintBps — different field name)                 │
│      Auto-flushes agent:vaults:* caches.                                 │
│                                                                          │
│  Phase 4: REBALANCE (MANDATORY — ASK USER FIRST)                          │
│    Ask the user: "Rebalancing is required to apply the new basket.        │
│      Confirm to execute now?"                                            │
│    On confirm:                                                           │
│      POST /api/v2/agent/dtf/:symbol/rebalance                            │
│        { signerPublicKey }                                               │
│      Admin wallet executes swaps server-side.                            │
│    On decline: warn the user that the on-chain basket targets the         │
│      new allocations but the vault's holdings still match the old        │
│      basket; drift persists until they trigger a rebalance later.        │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Phase 1 — Decide the new basket

Identify which assets to add/remove/rebalance based on the user's intent. Use the same research tools as launch:

- `WebSearch` / `WebFetch` for token discovery (price action, narratives, market caps).
- `GET /market-metrics?symbols=A,B,C` for the **authoritative** Jupiter liquidity / 24h volume — same numbers the policy engine enforces against Rules 2 / 3.
- `GET /dtf/:symbol/policy` to read the vault's existing `asset_mode`, `asset_whitelist`, `asset_blacklist`, `min_amm_liquidity_usd`, `min_24h_volume_usd`, `max_asset_pct`, `min_asset_pct`, `min_stablecoin_pct`. Calibrate the new basket so the policy gate in Phase 2 doesn't reject it. **The policy is fixed; only the basket is editable in this flow.**

Pick `mintBps` for each asset such that all values sum to **exactly 10000**. Round half-away-from-zero if needed and absorb the remainder into the largest allocation.

**You can pass `symbol` or `name` instead of `mintAddress`** — the backend resolves them via the `asset-allocation` collection (same path as `/launch-dtf`). Resolving server-side is cheaper and avoids the agent maintaining its own mint-address map.

#### Phase 2 — Build the on-chain tx (policy-gated)

```bash
# Inline node script — never echo the JWT or the keypair to terminal
node -e '
const http = require("http");
const https = require("https");
const url = new URL(process.env.DFM_API_URL + "/api/v2/agent/vaults/" + process.argv[1] + "/update-assets-tx"); // process.argv[1] = vaultSymbol (e.g. ALPHA)
const client = url.protocol === "https:" ? https : http;

const { Keypair, VersionedTransaction, Connection } = require("@solana/web3.js");
const bs58 = require("bs58").default || require("bs58");
const keypair = Keypair.fromSecretKey(bs58.decode(process.env.DFM_AGENT_KEYPAIR));

const payload = JSON.stringify({
  signerPublicKey: keypair.publicKey.toBase58(),
  underlyingAssets: [
    { symbol: "SOL", mintBps: 5000 },
    { symbol: "JUP", mintBps: 3000 },
    { name: "Bonk", mintBps: 2000 }
  ]
});

const req = client.request(url, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Content-Length": Buffer.byteLength(payload),
    "Authorization": "Bearer " + process.env.DFM_AUTH_TOKEN
  }
}, (res) => {
  let data = ""; res.on("data", (c) => data += c);
  res.on("end", async () => {
    const json = JSON.parse(data);
    if (res.statusCode === 201) {
      // Sign locally + submit on-chain
      const conn = new Connection(process.env.SOLANA_RPC_URL || "https://api.mainnet-beta.solana.com");
      const tx = VersionedTransaction.deserialize(Buffer.from(json.transaction, "base64"));
      tx.sign([keypair]);
      const sig = await conn.sendRawTransaction(tx.serialize(), { preflightCommitment: "confirmed" });
      await conn.confirmTransaction(sig, "confirmed");
      console.log("ON_CHAIN_OK signature=" + sig + " vaultIndex=" + json.vaultIndex);
    } else if (res.statusCode === 400 && json.ok === false) {
      // Policy violation — every violated rule in violations[]
      console.log("POLICY_VIOLATION");
      for (const v of json.violations || []) {
        console.log("  " + v.violationCode + ": " + v.message);
      }
    } else {
      console.log("ERROR " + res.statusCode + ": " + data);
    }
  });
});
req.on("error", (e) => console.log("ERROR: " + e.message));
req.write(payload);
req.end();
' <vaultSymbol>
```

Run with `timeout: 600000` (10 minutes) — on-chain confirmation can take time.

**On policy violation (HTTP 400 + `{ ok: false, violations[] }`):** read every entry in `violations[]`, summarise the codes to the user as a one-line "blocked because", then **automatically retry** Phase 1 with adjustments:

| Violation code | What to change |
|---|---|
| `rule2MinAmmLiquidity` / `rule3Min24hVolume` | Drop the offending asset OR swap it for a more-liquid alternative — the vault's policy threshold is fixed, so the basket must adapt. |
| `rule4MinMaxAssetCount` | The proposed basket size doesn't fit `min_assets`/`max_assets`. The bounds are fixed — either reshape the basket OR if the vault was launched with `min_assets == max_assets`, see "structurally locked policy" below. |
| `rule5MaxPctPerAsset` | Reduce that asset's `mintBps` and redistribute to others. |
| `rule6MinPctPerAssetIfHeld` | Either raise the asset above the floor OR drop it from the basket entirely. |
| `rule7MinStablecoinFloor` | Add a stablecoin allocation (must be one already in `asset-allocation` and permitted by `asset_mode`). |
| `rule1WhitelistBlacklist` / `assetModeViolation` | Remove the asset (it's blacklisted or not whitelisted in the vault's existing policy). |
| **Per-update movement cap** (rejection mentions `max_rebalance_pct`) | Aggregate movement = sum of `|new_bps - old_bps|` across every mint. If this exceeds `max_rebalance_pct`, the swap won't fit. Try a smaller-step migration (move a fraction of the desired weight first, repeat after `min_rebalance_interval_hours`). If even a single-asset swap exceeds the cap, the policy is structurally locked — see below. |

Loop Phase 1 → Phase 2 up to 3 times. If still failing after 3 attempts, surface the unresolved violations to the user.

**Diagnosing a structurally locked policy:** if the same `rule4MinMaxAssetCount` keeps firing because `min_assets == max_assets`, OR every single-asset swap exceeds `max_rebalance_pct` (test: `2 × max_asset_pct > max_rebalance_pct`), OR an included asset's volume sits within 30% of `min_24h_volume_usd` and `rule3Min24hVolume` blocks every migration that still contains it — the vault was misconfigured at launch and the agent cannot self-heal. State the diagnosis to the user with the specific bound that needs to move, and stop. Don't loop indefinitely on a vault that math says is unfixable.

**Signing:** use the existing `signAndSend` helper from the Pre-Flight Auth section. The signed tx hits Solana RPC; wait for `confirmed` commitment before moving to Phase 3.

#### Phase 3 — Sync DB

After the on-chain tx confirms, `PATCH /api/v2/agent/vaults/:symbol/underlying-assets-by-mint` to update the DB record. **Note the field-name difference:**

- Phase 2 (`/update-assets-tx`) uses `mintBps` (matches the on-chain instruction).
- Phase 3 (`/underlying-assets-by-mint`) uses `pct_bps` (matches the DB schema).

Same numeric values, different keys. Always include `mintAddress` here (no symbol/name resolution at this endpoint).

```bash
node -e '
const http = require("http");
const https = require("https");
const url = new URL(process.env.DFM_API_URL + "/api/v2/agent/vaults/" + process.argv[1] + "/underlying-assets-by-mint"); // process.argv[1] = vaultSymbol (e.g. ALPHA)
const client = url.protocol === "https:" ? https : http;

const payload = JSON.stringify({
  underlyingAssets: [
    { mintAddress: "So11111111111111111111111111111111111111112", pct_bps: 5000 },
    { mintAddress: "JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN", pct_bps: 3000 },
    { mintAddress: "DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263", pct_bps: 2000 }
  ]
});

const req = client.request(url, {
  method: "PATCH",
  headers: {
    "Content-Type": "application/json",
    "Content-Length": Buffer.byteLength(payload),
    "Authorization": "Bearer " + process.env.DFM_AUTH_TOKEN
  }
}, (res) => {
  let data = ""; res.on("data", (c) => data += c);
  res.on("end", () => console.log("DB_SYNC " + res.statusCode));
});
req.on("error", (e) => console.log("ERROR: " + e.message));
req.write(payload);
req.end();
' <vaultSymbol>
```

**Why both phases?** Phase 2 mutates on-chain state; Phase 3 makes the change visible to read endpoints (`/vaults/user`, `/vaults/featured/list`) immediately. The chain-event pipeline does eventually backfill the DB on its own, but PATCH gives synchronous visibility — the user expects "show me my vault" to reflect the new basket immediately after they say "update underlying".

#### Phase 4 — Rebalance (mandatory, with user confirmation)

After Phase 3 completes, the on-chain basket targets the new allocations but the vault still holds the old asset mix. Rebalancing is **required** to bring actual holdings in line with the new basket — without it, the vault drifts.

**Ask the user once** in plain language before triggering it: e.g. *"The basket update is on-chain and synced. Rebalancing to the new allocations is required — confirm to execute now?"* Then call:

`POST {DFM_API_URL}/api/v2/agent/dtf/:symbol/rebalance` with `{ signerPublicKey }`.

This is the same `/rebalance` endpoint used in the rebalance section — the admin wallet executes the swaps server-side, so the agent only sends the request and surfaces the result. Use the `vaultSymbol` (not `vaultId`) returned from Phase 2 / Phase 3.

If the user **declines**, surface a one-line warning: the on-chain basket targets the new allocations but actual holdings still match the old basket; the drift persists until they later say "rebalance the vault".

#### After all phases succeed

Surface a one-line summary to the user: "Updated `<vaultName>` basket to `<asset1 pct1%, asset2 pct2%, ...>` and rebalanced. On-chain signatures — update: `<updateSig>`, rebalance: `<rebalanceSig>`." The agent should NEVER expose endpoint names, payload shapes, or HTTP methods in user-facing messages — only the outcome.

#### CRITICAL ERROR HANDLING for update underlying

1. **`/update-assets-tx` returns 400 with `violations[]`** (before any signing): policy gate failed — nothing on-chain. Loop Phase 1 with adjustments. Free.
2. **`/update-assets-tx` returns 400 with asset-not-found message** ("The following assets are not available in the platform: ..."): the symbol/name didn't resolve. Pick a different asset whose symbol IS in `asset-allocation` (you can verify against `/market-metrics`).
3. **Signing/submission fails on-chain**: you MAY retry `/update-assets-tx` to get a fresh blockhash + fresh policy check (policy may have drifted in the meantime). Same `signerPublicKey` and same basket — no other changes needed.
4. **On-chain submitted but Phase 3 (PATCH) fails**: do NOT re-run `/update-assets-tx` (the on-chain change already happened — don't double-update). Retry the PATCH with the **same body**. If PATCH keeps failing, surface the on-chain signature to the user; the chain-event pipeline will eventually sync the DB on its own.
5. **`/update-assets-tx` returns 404 "No constitutional policy found"**: the vault was created outside the agent flow (no `/launch-dtf`). Update is blocked. Surface this to the user — there's nothing to fix from the agent side.
6. **Phase 4 (rebalance) fails after Phase 3 succeeded**: do NOT re-run `/update-assets-tx` or PATCH (the basket change is already complete on-chain and in the DB). Retry `POST /dtf/:symbol/rebalance` with the same body. If it keeps failing, surface to the user: the basket has been updated but the vault's holdings still match the old basket — they should retry the rebalance later via "rebalance the vault".
7. **Any phase returns HTTP 403 / "Only the vault creator can perform this action"**: the agent wallet (`signerPublicKey`) does not match the vault's `creatorAddress`. **STOP — do NOT retry, do NOT try a different vault symbol or id, do NOT loop through `/vaults/user` or `/vaults/featured/list` looking for matches.** Surface to the user: *"That vault belongs to a different wallet. Only its creator can update it."* End the turn.

### Step 7: Capital Flows — Deposit & Redeem

When the user says **"deposit USDC into SOLBC"**, **"buy SOLBC shares"**, **"redeem 1.5 SOLBC"**, **"sell my SOLBC tokens"**, or any phrasing meaning "move capital in or out of a vault using the agent wallet", run one of the two flows below.

```
+--------------------------------------------------------------------+
|  CAPITAL FLOWS — agent-bound deposit (2 steps) + redeem (5 steps)  |
|                                                                    |
|  DEPOSIT (vault USDC fans out into underlyings):                   |
|    1. POST /vaults/:symbol/check-min-deposit    (MANDATORY GATE)   |
|         -> abort flow on 400. Do NOT call deposit-tx if the        |
|         -> threshold is not met.                                   |
|    2. POST /vaults/:symbol/deposit-tx                              |
|         -> base64 unsigned VersionedTransaction                    |
|    3. agent signs locally + submits on-chain                       |
|    4. POST /deposit-transaction                                    |
|         -> agentSwap (vault USDC -> underlyings via Jupiter) +     |
|            depositTransaction (parses logs, persists 4 records)    |
|                                                                    |
|  REDEEM (queue-serialised; underlyings fan in to vault USDC):      |
|    1. POST /check-min-redeem                    (MANDATORY GATE)   |
|         -> reads `isValid` (returns 200 either way). Abort flow    |
|         -> when isValid=false. Do NOT call request-ticket.         |
|    2. POST /redeem/request-ticket               (queue entry)      |
|    3. poll v1 GET /api/v1/tx-event-management/redeem/ticket-status |
|         /:ticketId until isReady=true (agent has no clone)         |
|    4. POST /redeem/execute/:ticketId            (backend swap fan- |
|         in: underlyings -> USDC inside the vault)                  |
|    5. POST /vaults/:symbol/redeem-tx                               |
|         -> base64 unsigned finalizeRedeem tx                       |
|    6. agent signs locally + submits on-chain                       |
|    7. POST /redeem-transaction { transactionSignature, ticketId }  |
|         -> records redeem + auto-confirms ticket (non-fatal)       |
|                                                                    |
|  IDENTITY:                                                         |
|    Deposit-tx takes an explicit `userPublicKey` body field — the   |
|    depositor wallet may or may not be the agent. Records are       |
|    attributed via JWT (agentProfile + agentAddress).               |
|    Redeem-tx is bound to the agent wallet (JWT canonical, body     |
|    `agentWallet` accepted only as fallback).                       |
|                                                                    |
|  PRE-FLIGHT GATES ARE MANDATORY:                                   |
|    Both flows MUST start with the corresponding check. Skipping    |
|    them sends underspec'd amounts to the on-chain ix and produces  |
|    confusing failures (post-fee dust, 0-share mints, etc.).        |
+--------------------------------------------------------------------+
```

**When to run which flow:**

| User intent | Flow | Notes |
|---|---|---|
| "Deposit X USDC into <symbol>" | Deposit | Set `userPublicKey` to the **agent wallet** by default unless the user explicitly names a different depositor wallet. |
| "Redeem X tokens from <symbol>" / "Sell my <symbol>" | Redeem | Always uses the agent wallet (JWT). The agent must already hold a position keyed on its `agentProfile`. |
| "Add liquidity to <symbol>" / "Buy more <symbol>" | Deposit | Same as deposit. |
| "Withdraw from <symbol>" / "Cash out <symbol>" | Redeem | Same as redeem. |

**Pre-flight (both flows):** before issuing build-tx requests, surface a one-line summary to the user (vault, amount, expected fee). The flows below do **not** ask for further confirmation between sub-steps.

**Mandatory minimum-amount gate:** every deposit run MUST begin with `POST /vaults/:symbol/check-min-deposit`, and every redeem run MUST begin with `POST /check-min-redeem`. These are not optional — they are gates on the rest of the flow:
- `check-min-deposit` throws `400` (`Minimum deposit should be at least $<n> USDC`) when the proposed amount is below threshold. **Do not proceed to `/deposit-tx` on 400** — surface the threshold to the user verbatim, ask them for a larger amount, then re-gate.
- `check-min-redeem` returns `200` with `isValid: false` when the proposed amount is below threshold (it does **not** throw). **Read `isValid` and abort if `false`** — do not call `/redeem/request-ticket` on a failed check.

The reason the gates are mandatory: skipping them lets through too-small amounts that produce confusing on-chain failures (post-fee dust, zero-share mints, and — for redeem — fractional swaps Jupiter rejects). Catching the issue at the validator endpoint gives the user a clean, actionable error before any on-chain or queue interaction.

**Phase ordering contract (READ FIRST — the most-violated invariant in this skill):**

Every phase of both flows is **strictly sequential**. The agent MUST `await` the previous phase's API response (and verify its status) before issuing the next phase's call. The scripts below are written as a single async chain so this is automatic — but if the agent ever runs phases as separate `node -e` invocations, or runs anything in parallel, the flow will silently corrupt:

| Risk | What goes wrong |
|---|---|
| Skipping `/deposit-transaction` after the on-chain submit confirms | The deposit tx lands on-chain (shares are minted to the depositor) **but no `UserVaultPosition` is recorded under `agentProfile`**. The user can't redeem via the agent flow later because `/redeem-transaction` will throw `No position found for agent in this vault`. |
| Reporting deposit success to the user before `/deposit-transaction` returns 200 | Same as above — the on-chain side is done, the DB side isn't. The user thinks they're holding shares; the agent's later P&L breakdown can't see the deposit. |
| Calling `/redeem/execute/:ticketId` before `isReady=true` | Backend rejects with `Failed to process ticket` and **auto-cancels the ticket** — the user has to start the whole redeem flow over. |
| Building `/redeem-tx` before `/redeem/execute` finishes the swap fan-in | The vault doesn't have enough USDC to settle, so `finalizeRedeem` fails on-chain. The signed price expires; the agent must re-call `/redeem-tx` for a fresh price. |
| Calling `/redeem-transaction` before the on-chain `finalizeRedeem` confirms | Backend can't fetch the tx (`Transaction not found`); the redeem is on-chain but no DB records exist. Re-call after `confirmTransaction` resolves. |

**The success indicator is the final `*_OK` JSON line printed by each script** (`DEPOSIT_OK { ... }` or `REDEEM_OK { ... }`). Any earlier `PHASE_N_OK` log is intermediate progress only — **do not surface success to the user until you see the final OK line on stdout**. If the script exits without the final OK line (or with `MIN_*_FAIL`, `BUILD_ERROR`, `EXECUTE_ERROR`, `RECORD_ERROR`, `FATAL`), surface the documented failure-message template and stop.

#### 7a. Deposit Flow (2 sub-steps after pre-flight)

**Step 7a-0 — Mandatory pre-flight: `/check-min-deposit`**

Always call `POST {DFM_API_URL}/api/v2/agent/vaults/:symbol/check-min-deposit` with `{ minDeposit: <UI USDC> }` **before** building the deposit tx. On `400`, do **not** proceed — surface the threshold message to the user and ask for a larger amount. The consolidated script in Step 7a-1 below runs this check internally and aborts on failure.

**Step 7a-1 — End-to-end deposit (single script, four sequential phases)**

The script below runs **all four deposit phases in one async chain** — each phase awaits the previous phase's API response before starting:

| Phase | Action | Indicator on stdout |
|---|---|---|
| 1 | `POST /vaults/:symbol/check-min-deposit` (mandatory gate) | `PHASE_1_OK gate=passed` |
| 2 | `POST /vaults/:symbol/deposit-tx` (build unsigned tx) | `PHASE_2_OK vaultIndex=<n>` |
| 3 | Sign locally with `DFM_AGENT_KEYPAIR` + `sendRawTransaction` + `confirmTransaction` | `PHASE_3_OK onChainSig=<sig>` |
| 4 | `POST /deposit-transaction` (swap fan-out + DB record persist) | `DEPOSIT_OK { … }` (final success — JSON line) |

**Do not run these phases as separate `node -e` invocations** — keep the chain inside a single script so the agent can't accidentally proceed before a phase resolves. **Do not surface success to the user** until the final `DEPOSIT_OK { … }` JSON line appears on stdout. A `PHASE_3_OK` (on-chain confirmed) **is not enough** — without `DEPOSIT_OK`, the `UserVaultPosition` keyed on `agentProfile` was never written, and the user's later redeem will fail with `No position found for agent in this vault`.

```bash
node -e '
const http = require("http");
const https = require("https");
const { Keypair, VersionedTransaction, Connection } = require("@solana/web3.js");
const bs58 = require("bs58").default || require("bs58");

const keypair = Keypair.fromSecretKey(bs58.decode(process.env.DFM_AGENT_KEYPAIR));
const symbol = process.argv[1];                 // e.g. ALPHA
const depositAmount = Number(process.argv[2]);  // UI USDC, e.g. 5

function call(path, method, body) {
  return new Promise((resolve, reject) => {
    const url = new URL(process.env.DFM_API_URL + path);
    const client = url.protocol === "https:" ? https : http;
    const payload = body ? JSON.stringify(body) : null;
    const headers = { "Authorization": "Bearer " + process.env.DFM_AUTH_TOKEN };
    if (payload) {
      headers["Content-Type"] = "application/json";
      headers["Content-Length"] = Buffer.byteLength(payload);
    }
    const req = client.request(url, { method, headers }, (res) => {
      let data = ""; res.on("data", (c) => data += c);
      res.on("end", () => {
        try { resolve({ status: res.statusCode, body: JSON.parse(data || "{}") }); }
        catch { resolve({ status: res.statusCode, body: data }); }
      });
    });
    req.on("error", reject);
    if (payload) req.write(payload);
    req.end();
  });
}

(async () => {
  // PHASE 1 — Mandatory gate
  const gate = await call(
    "/api/v2/agent/vaults/" + symbol + "/check-min-deposit",
    "POST",
    { minDeposit: depositAmount }
  );
  if (gate.status !== 200) {
    console.log("MIN_DEPOSIT_FAIL " + gate.status + ": " + (gate.body?.message || JSON.stringify(gate.body)));
    return;
  }
  console.log("PHASE_1_OK gate=passed");

  // PHASE 2 — Build the unsigned deposit tx (await response BEFORE signing)
  const build = await call(
    "/api/v2/agent/vaults/" + symbol + "/deposit-tx",
    "POST",
    {
      userPublicKey: keypair.publicKey.toBase58(), // agent wallet as depositor (default)
      depositAmount,
    }
  );
  if (build.status !== 201) {
    console.log("BUILD_ERROR " + build.status + ": " + JSON.stringify(build.body));
    return;
  }
  console.log("PHASE_2_OK vaultIndex=" + build.body.vaultIndex);

  // PHASE 3 — Sign locally + submit on-chain (await CONFIRMATION before recording)
  const conn = new Connection(process.env.SOLANA_RPC_URL || "https://api.mainnet-beta.solana.com");
  const tx = VersionedTransaction.deserialize(Buffer.from(build.body.transaction, "base64"));
  tx.sign([keypair]);
  const onChainSig = await conn.sendRawTransaction(tx.serialize(), { preflightCommitment: "confirmed" });
  await conn.confirmTransaction(onChainSig, "confirmed");
  console.log("PHASE_3_OK onChainSig=" + onChainSig);

  // PHASE 4 — Record (await /deposit-transaction response before declaring success)
  // Backend runs two server-side sub-operations:
  //   (a) agentSwap — vault USDC -> underlyings via Jupiter
  //   (b) depositTransaction — parses VaultDeposited logs, writes
  //       DepositTransaction + UserVaultPosition (keyed on agentProfile) +
  //       DepositRecord + History
  // Failure modes are surfaced INSIDE the 200 response (swap.failedSwapsInfo,
  // deposit.events[].vaultDepositUpdateError) — neither aborts the flow.
  const rec = await call("/api/v2/agent/deposit-transaction", "POST", {
    transactionSignature: onChainSig,
    vaultIndex: build.body.vaultIndex,
    etfSharePriceRaw: build.body.etfSharePriceRaw,
    amountInRaw: build.body.amountInRaw,
    slippage: 200,
  });
  if (rec.status !== 200) {
    console.log("RECORD_ERROR " + rec.status + ": " + JSON.stringify(rec.body));
    return;
  }

  // FINAL — emit consolidated JSON for the user-facing summary + memory.
  // Agent must wait for THIS line before reporting deposit success.
  const ev = rec.body.deposit?.events?.[0] || {};
  const sharePriceDepositUsd = build.body.etfSharePriceRaw
    ? Number(build.body.etfSharePriceRaw) / 1e6
    : null;
  console.log("DEPOSIT_OK " + JSON.stringify({
    event: ev.eventType,
    vaultName: ev.vaultName,
    vaultSymbol: ev.vaultSymbol,
    grossUsdcRaw: ev.amount,                          // 6 decimals
    entryFeeRaw: ev.entryFee,                         // 6 decimals
    managementFeeRaw: ev.managementFee,               // 6 decimals (typically 0 at deposit)
    netUsdcRaw: ev.netAmount,                         // amount that actually became shares
    sharesMintedRaw: ev.vaultTokensMinted,            // 6 decimals
    sharePriceDepositUsd,                             // share price baked into THIS deposit
    onChainSig,
    failedSwaps: rec.body.swap?.failedSwapsInfo || null,
    recordError: ev.vaultDepositUpdateError || null,
  }));
})().catch(e => console.log("FATAL " + (e?.message || String(e))));
' <vaultSymbol> <depositAmount>
```

Run with `timeout: 600000` (10 minutes). The four `*_OK` markers print as each phase resolves; the agent should follow them but report success **only when `DEPOSIT_OK { … }` appears**. On any other terminal line (`MIN_DEPOSIT_FAIL`, `BUILD_ERROR`, `RECORD_ERROR`, `FATAL`), surface the documented failure-message template and stop.

#### Deposit completion messaging — MANDATORY format

**Symmetric to the redeem messaging rules.** When summarising a deposit, the agent must read `entryFee` and `sharePriceDepositUsd` from the script's JSON output, surface them transparently, and **remember them in conversation memory** so a future redeem call can show a true P&L breakdown instead of guessing.

**Required user-facing summary (template):**
```
Deposit complete — `<vaultName>` (`<vaultSymbol>`).

• USDC deposited (gross): <grossUsdcRaw / 1e6> USDC
• Entry fee: <entryFeeRaw / 1e6> USDC
• USDC that purchased shares: <netUsdcRaw / 1e6> USDC
• Shares minted: <sharesMintedRaw / 1e6> <vaultSymbol>
• Share price at deposit: $<sharePriceDepositUsd>
• On-chain signature: <onChainSig>
```

If `failedSwaps` is present, append exactly: *"`<failedCount>` underlying swap(s) failed and the USDC was returned to the vault — no funds lost."* Do not invent any other failure narrative.

If `recordError` is present, append exactly: *"On-chain deposit succeeded, but the database record-write failed — the chain-event pipeline will reconcile shortly. Your shares are minted on-chain regardless."*

**Remember-for-later (conversation context):** stash `{ vaultSymbol, grossUsdcRaw, entryFeeRaw, sharesMintedRaw, sharePriceDepositUsd, onChainSig }` so when the same user later redeems from the same vault, the redeem-completion summary can include the deposit-side fee + a real NAV-change calculation.

#### 7b. Redeem Flow (4 sub-steps after pre-flight)

The redeem flow has more steps because (a) it queues to serialise vault liquidation, and (b) the swap fan-in must happen **before** the on-chain `finalizeRedeem` so the vault has enough USDC to settle.

**Step 7b-0 — Mandatory pre-flight: `/check-min-redeem`**

Always call `POST {DFM_API_URL}/api/v2/agent/check-min-redeem` with `{ minRedeem: <UI vault tokens> }` **before** requesting a ticket. Unlike `check-min-deposit`, this endpoint **always returns 200** with `{ isValid, message }` — read `isValid` rather than the HTTP status. When `isValid: false`, do **not** call `/redeem/request-ticket`; surface the threshold to the user and stop. The combined script in Step 7b-5 below runs this check internally and aborts on failure.

**Step 7b-1 — Request a redeem ticket**

`POST {DFM_API_URL}/api/v2/agent/redeem/request-ticket` with `{ vaultSymbol, vaultTokenAmount, etfSharePriceRaw?, slippage? }`. `vaultTokenAmount` is **raw u64 as a string** (6 decimals — multiply UI tokens by 1e6 first). `etfSharePriceRaw` is optional; backend reads on-chain when omitted.

Returns: `{ ticketId, position, estimatedWaitSeconds, isReady }`.

**Step 7b-2 — Poll until ready (use v1 endpoint)**

The agent module has no `/ticket-status` clone. Poll `GET {DFM_API_URL}/api/v1/tx-event-management/redeem/ticket-status/:ticketId` every 3-5 seconds with the same `Authorization` header. Stop when `status.isReady === true`. Tickets expire ~3 minutes after becoming ready — don't dawdle.

**Step 7b-3 — Execute the swap fan-in**

`POST {DFM_API_URL}/api/v2/agent/redeem/execute/:ticketId` with `{ slippage? }`. All other parameters (vault, amount, share price) are read from the ticket itself. Backend transitions the ticket to `PROCESSING`, runs `redeemAgentSwapAdmin` (vault underlyings → USDC), and returns swap signatures.

**On execute failure**, the ticket is **auto-cancelled** by the backend before the error rethrows — the agent does **not** need to call `/cancel`. Surface the error to the user and stop.

**Step 7b-4 — Build the unsigned `finalizeRedeem` tx**

`POST {DFM_API_URL}/api/v2/agent/vaults/:symbol/redeem-tx` with `{ vaultTokenAmount }` (UI vault tokens, e.g. `1.5`). The agent wallet is read from the JWT (body `agentWallet` is fallback only).

Backend resolves the vault, derives PDAs, fetches a fresh KMS-signed share price, conditionally adds 4 ATA-creation ixs (agent vault-token + USDC, fee-recipient USDC, vault-admin USDC), and emits the Ed25519 verify ix + `finalizeRedeem(...)` ix. Returns `{ transaction, vaultIndex, etfSharePriceRaw, priceTimestamp, vaultTokenAmountRaw, ... }`.

**Step 7b-5 — Sign locally, submit, record + auto-confirm ticket**

Sign the returned tx with `DFM_AGENT_KEYPAIR`, submit on-chain, then call `POST {DFM_API_URL}/api/v2/agent/redeem-transaction` with `{ transactionSignature, vaultIndex, etfSharePriceRaw, signatureArray, slippage, ticketId }`.

Including `ticketId` triggers the backend to **auto-confirm** the redeem queue ticket after the record is persisted. The auto-confirm is **non-fatal** — failure surfaces on the response as `ticketConfirm: { ok: false, message }` but does not abort. The redeem is already on-chain and DB-recorded.

**End-to-end redeem (single script, seven sequential phases).** Each phase awaits its API response before the next:

| Phase | Action | Indicator on stdout |
|---|---|---|
| 1 | `POST /check-min-redeem` (mandatory gate; **read `isValid`**, not status) | `PHASE_1_OK gate=passed` |
| 2 | `POST /redeem/request-ticket` (queue entry) | `PHASE_2_OK ticketId=<id> ready=<bool>` |
| 3 | Poll v1 `GET /redeem/ticket-status/:ticketId` until `isReady=true` (max 5 min) | `PHASE_3_OK ticket=ready` |
| 4 | `POST /redeem/execute/:ticketId` (backend swap fan-in) | `PHASE_4_OK swaps=<n>` |
| 5 | `POST /vaults/:symbol/redeem-tx` (build unsigned `finalizeRedeem`) | `PHASE_5_OK vaultIndex=<n>` |
| 6 | Sign locally with `DFM_AGENT_KEYPAIR` + `sendRawTransaction` + `confirmTransaction` | `PHASE_6_OK onChainSig=<sig>` |
| 7 | `POST /redeem-transaction` (record + auto-confirm ticket) | `REDEEM_OK { … }` (final success — JSON line) |

**Do not surface success to the user** until the final `REDEEM_OK { … }` line appears on stdout. `PHASE_6_OK` (on-chain confirmed) **is not enough** — without `REDEEM_OK`, the position state is out of sync with the on-chain redeem and the History row was never written.

```bash
# End-to-end redeem — single script, seven sequential phases.
node -e '
const http = require("http");
const https = require("https");
const { Keypair, VersionedTransaction, Connection } = require("@solana/web3.js");
const bs58 = require("bs58").default || require("bs58");

const keypair = Keypair.fromSecretKey(bs58.decode(process.env.DFM_AGENT_KEYPAIR));
const symbol = process.argv[1];
const uiAmount = Number(process.argv[2]); // e.g. 1.5

function call(path, method, body) {
  return new Promise((resolve, reject) => {
    const url = new URL(process.env.DFM_API_URL + path);
    const client = url.protocol === "https:" ? https : http;
    const payload = body ? JSON.stringify(body) : null;
    const headers = {
      "Authorization": "Bearer " + process.env.DFM_AUTH_TOKEN
    };
    if (payload) {
      headers["Content-Type"] = "application/json";
      headers["Content-Length"] = Buffer.byteLength(payload);
    }
    const req = client.request(url, { method, headers }, (res) => {
      let data = ""; res.on("data", (c) => data += c);
      res.on("end", () => {
        try { resolve({ status: res.statusCode, body: JSON.parse(data || "{}") }); }
        catch { resolve({ status: res.statusCode, body: data }); }
      });
    });
    req.on("error", reject);
    if (payload) req.write(payload);
    req.end();
  });
}

(async () => {
  const rawAmount = String(Math.round(uiAmount * 1e6));

  // PHASE 1 — Mandatory gate: /check-min-redeem
  // Returns 200 with isValid; do NOT trust HTTP status alone — read the boolean.
  const gate = await call("/api/v2/agent/check-min-redeem", "POST", { minRedeem: uiAmount });
  if (gate.status !== 200 || gate.body?.isValid !== true) {
    console.log("MIN_REDEEM_FAIL: " + (gate.body?.message || JSON.stringify(gate.body)));
    return;
  }
  console.log("PHASE_1_OK gate=passed");

  // PHASE 2 — Request a redeem ticket (await response BEFORE polling)
  const ticket = await call("/api/v2/agent/redeem/request-ticket", "POST", {
    vaultSymbol: symbol,
    vaultTokenAmount: rawAmount,
    slippage: 200
  });
  if (ticket.status !== 200) { console.log("TICKET_ERROR " + ticket.status + ": " + JSON.stringify(ticket.body)); return; }
  const { ticketId } = ticket.body;
  console.log("PHASE_2_OK ticketId=" + ticketId + " ready=" + ticket.body.isReady);

  // PHASE 3 — Poll v1 ticket-status until ready (max 5 minutes)
  // Each poll awaits the response before sleeping for the next attempt.
  // Do NOT call /redeem/execute until isReady=true.
  const start = Date.now();
  while (Date.now() - start < 5 * 60 * 1000) {
    if (ticket.body.isReady) break;
    await new Promise(r => setTimeout(r, 3000));
    const s = await call("/api/v1/tx-event-management/redeem/ticket-status/" + ticketId, "GET");
    if (s.body?.isReady) { ticket.body.isReady = true; break; }
    if (["EXPIRED", "CANCELLED", "COMPLETED"].includes(s.body?.status)) {
      console.log("TICKET_TERMINAL status=" + s.body.status); return;
    }
  }
  if (!ticket.body.isReady) { console.log("TICKET_TIMEOUT"); return; }
  console.log("PHASE_3_OK ticket=ready");

  // PHASE 4 — Execute the backend swap fan-in (await 200 response BEFORE building tx)
  // On failure here, the backend AUTO-CANCELS the ticket — do not call /cancel.
  const exec = await call("/api/v2/agent/redeem/execute/" + ticketId, "POST", { slippage: 200 });
  if (exec.status !== 200) { console.log("EXECUTE_ERROR " + exec.status + ": " + JSON.stringify(exec.body)); return; }
  const swapSigs = (exec.body.swaps || []).map(s => s.swapSig || s.sig).filter(Boolean);
  console.log("PHASE_4_OK swaps=" + swapSigs.length);

  // PHASE 5 — Build the unsigned finalizeRedeem tx (await response BEFORE signing)
  const build = await call("/api/v2/agent/vaults/" + symbol + "/redeem-tx", "POST", {
    vaultTokenAmount: uiAmount
  });
  if (build.status !== 201) { console.log("BUILD_ERROR " + build.status + ": " + JSON.stringify(build.body)); return; }
  console.log("PHASE_5_OK vaultIndex=" + build.body.vaultIndex);

  // PHASE 6 — Sign + submit on-chain (await CONFIRMATION before recording)
  const conn = new Connection(process.env.SOLANA_RPC_URL || "https://api.mainnet-beta.solana.com");
  const tx = VersionedTransaction.deserialize(Buffer.from(build.body.transaction, "base64"));
  tx.sign([keypair]);
  const onChainSig = await conn.sendRawTransaction(tx.serialize(), { preflightCommitment: "confirmed" });
  await conn.confirmTransaction(onChainSig, "confirmed");
  console.log("PHASE_6_OK onChainSig=" + onChainSig);

  // PHASE 7 — Record + auto-confirm ticket (await /redeem-transaction response)
  // Including ticketId triggers the backend to confirmRedeemTicket after the
  // record is persisted. Failure of the confirm step is non-fatal and surfaces
  // on the response as ticketConfirm.ok=false.
  const rec = await call("/api/v2/agent/redeem-transaction", "POST", {
    transactionSignature: onChainSig,
    vaultIndex: build.body.vaultIndex,
    etfSharePriceRaw: build.body.etfSharePriceRaw,
    signatureArray: swapSigs,
    slippage: 200,
    ticketId
  });
  if (rec.status !== 200) { console.log("RECORD_ERROR " + rec.status + ": " + JSON.stringify(rec.body)); return; }

  // FINAL — emit REDEEM_OK { ... } only after /redeem-transaction returns 200.
  // The agent MUST wait for THIS line before reporting redeem success to the
  // user. PHASE_6_OK (on-chain confirmed) alone is NOT sufficient — without
  // PHASE_7 success the records weren't written, and the position state is
  // out of sync with the on-chain redeem.
  const ev = rec.body.events?.[0] || {};
  const sharePriceRedeemUsd = build.body.etfSharePriceRaw
    ? Number(build.body.etfSharePriceRaw) / 1e6
    : null;
  console.log("REDEEM_OK " + JSON.stringify({
    event: ev.eventType,
    vaultName: ev.vaultName,
    vaultSymbol: ev.vaultSymbol,
    sharesRedeemedRaw: ev.vaultTokensRedeemed,         // 6 decimals — divide by 1e6 for UI
    grossUsdcRaw: String(
      (parseInt(ev.netStablecoinAmount || "0")
        + parseInt(ev.exitFee || "0")
        + parseInt(ev.managementFee || "0"))
    ),                                                  // pre-fee USDC
    netUsdcRaw: ev.netStablecoinAmount,                // post-fee USDC
    exitFeeRaw: ev.exitFee,                             // 6 decimals
    managementFeeRaw: ev.managementFee,                 // 6 decimals
    totalFeesRaw: ev.totalFees,                         // 6 decimals
    sharePriceRedeemUsd,                                // share price baked into THIS redeem
    onChainSig,
    ticketConfirm: rec.body.ticketConfirm,
  }));
})().catch(e => console.log("FATAL " + (e?.message || String(e))));
' <vaultSymbol> <uiAmount>
```

Run with `timeout: 600000` (10 minutes). The seven `PHASE_N_OK` markers print as each phase resolves; report success **only when `REDEEM_OK { … }` appears**. On any other terminal line (`MIN_REDEEM_FAIL`, `TICKET_ERROR`, `TICKET_TERMINAL`, `TICKET_TIMEOUT`, `EXECUTE_ERROR`, `BUILD_ERROR`, `RECORD_ERROR`, `FATAL`), surface the documented failure-message template and stop. The fields needed for the user-facing summary are described in "Redeem completion messaging" below.

#### Redeem completion messaging — MANDATORY format

The delta between what the user receives in USDC and what they originally deposited is **almost always fees, not market loss**. The vault collects an entry fee on deposit and an exit fee + management fee on redeem; together those bps add up to the difference. Misattributing this to "market discount", "loss from inception price", or "asset price drop" is **wrong** unless the share price has actually moved between deposit and redeem — and even then, fees are the larger driver for short holding periods.

**What the agent must read from the redeem response:**
- `events[].vaultTokensRedeemed` (raw, 6 dec) — divide by `1e6` for UI shares.
- `events[].netStablecoinAmount` (raw, 6 dec) — net USDC paid out, **after** exit fee + management fee.
- `events[].exitFee` (raw, 6 dec) — exit fee charged on this redeem.
- `events[].managementFee` (raw, 6 dec) — management fee charged on this redeem.
- `events[].totalFees` (raw, 6 dec) — convenience sum of the above two.
- `etfSharePriceRaw` from the build-tx response (raw u64, 6 dec) — share price at time of redeem. Convert via `/ 1e6` for USD.

**Required JSON the script outputs has all of this — use it directly. Do NOT compute the user-facing summary from `netStablecoinAmount` alone.**

**Decomposing the delta:**
1. `exitFee_usdc = exitFeeRaw / 1e6`
2. `mgmtFee_usdc = managementFeeRaw / 1e6`
3. `redeemSideFees_usdc = exitFee_usdc + mgmtFee_usdc` (= `totalFees / 1e6`)
4. **The deposit-side entry fee was charged earlier and is NOT part of this redeem's response.** If the agent has the original deposit's `entryFee` cached in conversation memory (from the `/deposit-transaction` step in the same session), include it; otherwise omit it from the breakdown rather than inventing a number.
5. **NAV change** is `(sharePriceRedeemUsd − sharePriceDepositUsd) × sharesRedeemed`. Only mention this if the agent has both share prices and they actually differ. If you don't have the deposit-time share price, **do not** describe the delta as market movement.

**Required user-facing summary (template):**
```
Redeem complete — `<vaultSymbol>`.

• Shares redeemed: <sharesRedeemedRaw / 1e6> <vaultSymbol>
• USDC received (net): <netUsdcRaw / 1e6> USDC
• Fees on this redeem: <totalFeesRaw / 1e6> USDC (exit: <exitFeeRaw / 1e6>, management: <managementFeeRaw / 1e6>)
• Share price at redemption: $<sharePriceRedeemUsd>
• On-chain signature: <onChainSig>
```

If the agent has the original deposit recorded in this session, append:
```
• Compared to your original deposit of <depositGrossUsdc> USDC:
    – Entry fee paid at deposit: <entryFeeUsdc> USDC
    – Exit + management fees on this redeem: <totalFeesUsdc> USDC
    – NAV change: <navChangeUsdc> USDC (positive = vault gained, negative = vault lost)
    – Net P&L: <netUsdcRaw / 1e6 − depositGrossUsdc> USDC
```

**Forbidden phrasings (the agent MUST NOT use these unless it has actual share-price comparison data showing real NAV movement):**
- "small loss from market discount on JUP / RAY / <asset>" — the underlyings' on-chain price doesn't pass through to the vault NAV in real time; only `sharePrice` does.
- "loss from the vault's inception price" — there is no "inception price" tracked in the response.
- "the assets are worth less than when you deposited" — read `sharePriceRedeemUsd` and prove this before saying it.
- "tracking error" / "slippage from your basket" — the on-chain `finalizeRedeem` doesn't slip the user; slippage was in the prior `redeemAgentSwapAdmin` step (server-side, vault-internal).

If `netUsdcRaw < depositGrossUsdc` (user got back less than they put in) and the agent **doesn't have** the deposit-time share price, the correct phrasing is:
> *"You received `<netUsdcRaw / 1e6>` USDC after entry, exit, and management fees. The full breakdown of fees: …"*

Not:
> *"…a small loss reflecting the current discount on JUP and RAY from the vault's inception price."*

#### CRITICAL ERROR HANDLING for capital flows

| Symptom | What to do |
|---|---|
| Script ends without `DEPOSIT_OK { … }` (deposit) | The deposit was **not** fully completed — even if `PHASE_3_OK` (on-chain confirmed) was logged, the DB records (`DepositTransaction`, `UserVaultPosition`, `DepositRecord`, History) were not written. Do NOT report deposit success to the user. The chain-event pipeline may eventually reconcile, but for the agent's purposes the deposit is "in flight". Surface the last error line to the user and stop. |
| Script ends without `REDEEM_OK { … }` (redeem) | The redeem was **not** fully completed. If `PHASE_6_OK` was logged but `PHASE_7` failed, the on-chain `finalizeRedeem` already happened (USDC paid out to the agent wallet) but no `RedeemTransaction` / `UserVaultPosition` decrement / `RedeemRecord` / History was written, and the queue ticket was not auto-confirmed. Do NOT report redeem success. Surface the last error line and stop; the operator can manually re-call `/redeem-transaction` with the same `transactionSignature` (the duplicate-signature guard makes that idempotent). |
| `/check-min-deposit` returns `400 Minimum deposit should be at least $<n> USDC` | **Hard gate — do NOT proceed to `/deposit-tx`.** Surface the threshold verbatim to the user, ask them for a larger amount, then re-run the gate. The gate must pass before any further deposit calls. |
| `/check-min-redeem` returns `200 { isValid: false, ... }` | **Hard gate — do NOT proceed to `/redeem/request-ticket`.** Surface `message` to the user, ask for a larger amount, then re-run the gate. Note: the endpoint returns HTTP 200 even on fail — read `isValid`, not the status code. |
| `/deposit-tx` returns `400 Vault "<symbol>" has no on-chain vaultIndex` | The vault hasn't been created on-chain yet — `/launch-dtf` was called but `/dtf-create` (and the on-chain submit between them) never landed. Surface to the user. |
| `/deposit-tx` returns `400 Signed price vaultPubkey ... does not match derived vault PDA` | KMS signer is mis-configured for this vault. **Do NOT retry** — surface to the user; this is an operator-side fix. |
| `/deposit-transaction` returns `200` with `swap.failedSwapsInfo` populated | Per-asset Jupiter swap failed and the backend already returned the USDC to the vault. Treat the deposit as recorded; warn the user about the failed swap count. **Do NOT re-call** `/deposit-transaction` — the deposit record already exists. |
| `/deposit-transaction` returns `200` with `deposit.events[].vaultDepositUpdateError` | The deposit record-write failed but the on-chain swap succeeded. Surface to user. The chain-event pipeline will eventually reconcile. **Do NOT re-call** with the same signature (idempotency guard will surface the existing record). |
| `/redeem/execute/:ticketId` throws | The ticket is **auto-cancelled** by the backend. Do NOT call `/cancel`. Surface error to user and end. |
| Ticket polling times out (`isReady` never true within 5 min) | Cancel manually via v1 `DELETE /api/v1/tx-event-management/redeem/cancel/:ticketId` and surface to user. Then call `/redeem/request-ticket` fresh if they want to retry. |
| `/redeem-tx` returns `400 InvalidPriceSignature` (after signing/submit) | The KMS-signed price expired between build and submit. Re-call `/redeem-tx` to get a fresh price; do NOT submit the stale tx. |
| `/redeem-transaction` returns `400 No position found for agent in this vault` | The agent has no recorded `UserVaultPosition` for this vault. They must `/deposit` under the agent flow first — a vault funded outside the agent flow won't have a position keyed on `agentProfile`. |
| `/redeem-transaction` returns `400 Insufficient shares. Have X, trying to redeem Y` | The on-chain redeem already happened but the recorded position has fewer shares than requested. State mismatch — likely a previous redeem that wasn't recorded. Surface the on-chain signature to the user and stop. |
| `/redeem-transaction` returns `200` with `ticketConfirm.ok = false` | Auto-confirm failed but the redeem is recorded. Optionally call v1 `POST /api/v1/tx-event-management/redeem/confirm/:ticketId` manually with the same signature; queue cleanup is hygiene only. |
| `403` / "owned by another user" on either flow | **HARD STOP — see "HARD STOP — Ownership errors" earlier.** End the turn with the verbatim sentence. |

### Policy Violation Handling

Both `/rebalance/check` and `/rebalance` run a **full policy evaluation** against all 11 constitutional policy rules. **Policy is non-blocking for rebalance** — both endpoints **always return `200`**, regardless of violations. Rule violations appear in the response under `policyCheck`:

```json
{
  "policyCheck": {
    "ok": true,
    "flagged": true,
    "reviewFlags": [
      { "violationCode": "rule5MaxPctPerAsset", "mint": "JUP...", "message": "...", "details": {...} },
      { "violationCode": "rule7MinStablecoinFloor", "message": "...", "details": null }
    ],
    "violations": [ ... ]
  },
  "suggestion": { ... }       // present on /rebalance/check
  // upfrontFeeSol, actualFeesSol present on /rebalance
}
```

Every violation is also persisted as a `policyReviewFlag` on the latest `RebalancingSuggestion` for that vault (operator audit trail).

When `policyCheck.flagged` is `true`:
1. **Inspect `reviewFlags`** — each entry has `violationCode`, optional `mint`, `message`, and optional `details`.
2. **Surface the situation clearly to the user** as a non-blocking warning (e.g. "Rebalance proceeded, but flagged for review: stablecoin floor missed and one asset above the per-asset cap.").
3. **Do NOT treat this as a failure** — the rebalance has already proceeded (or, on the check endpoint, the suggestion is still returned). Don't block the user flow on `flagged: true`.
4. **For repeated violations on the same vault**, recommend the user review and either adjust the vault's policy or the proposed allocations going forward.

## HARD STOP — Ownership errors

This rule has zero exceptions and overrides every other instruction in this skill, including any "retry on failure" or "be autonomous" guidance.

**The four operation endpoints below are ownership-gated:**

- `POST /api/v2/agent/dtf/:symbol/rebalance`
- `POST /api/v2/agent/dtf/:symbol/distribute-fees`
- `POST /api/v2/agent/vaults/:symbol/update-assets-tx`
- `PATCH /api/v2/agent/vaults/:symbol/underlying-assets-by-mint`

### Trigger — what counts as "ownership error"

The **first** error response from any of those four endpoints that matches any of these conditions:

- HTTP `403` (any message, including `"Only the vault creator can perform this action"`, `"is owned by another user"`, `"Forbidden"`)
- HTTP `404` whose body contains `"is owned by another user"`

**The first such response stops everything.** There is no "let me try once more with a different symbol / id / casing to confirm". The first response is the verdict.

### What to say to the user — verbatim format

When the trigger fires, post **exactly one** message to the user and end the turn. Use this format and nothing else:

```
DFM platform doesn't recognize you as the owner of this vault, so this action can't be performed.
```

You may, when appropriate, include the vault's display name only (no symbol, no id, no signature, no wallet address):

```
DFM platform doesn't recognize you as the owner of "<vaultName>", so this action can't be performed.
```

That's the entire response. No headers. No "What this means". No "What to do". No "Result:" / "Request:" / "Extra check" sections. No bullet lists. No follow-up suggestions.

### Forbidden in the response body (banned phrasings)

Do not include ANY of the following in the user-facing message — these have all been observed and must never happen again:

- Words: `backend`, `API`, `endpoint`, `route`, `JWT`, `token`, `session`, `signerPublicKey`, `DFM_AGENT_KEYPAIR`, `DFM_API_URL`, `wallet address`, `keypair`, `creatorAddress`, `signature`, `Mongo`, `id`, `index`, `vault index`, `on-chain creator`
- HTTP status codes or names: `403`, `404`, `Forbidden`, `Not Found`, `Unauthorized`
- Method/path snippets: `POST /…`, `GET /…`, `/dtf/...`, `/vaults/...`, `/rebalance`, `/distribute-fees`
- JSON: any `{ … }` or quoted server message
- Self-referential rule mentions: ❌ "per the agent rules", "I did not retry further", "ownership rules require…", "as documented in the skill"
- Diagnostic / advice sections: ❌ "What this means:", "What to do:", "What you can do next:", "Here's what I tried:", "If you want…", "Try the same call against QA", "Refresh the token", "Use a different keypair", "Have the vault transferred"
- Announcements that imply more work is coming: ❌ "Let me check…", "I'll verify…", "One moment while…"

### Forbidden tool calls after the trigger

After surfacing the message, **do not call any tool**. Specifically forbidden:

- Retrying the same call with the same `signerPublicKey`
- Retrying with a different `vaultSymbol` casing, hyphenation, or aliasing — e.g. **`POP-DTF` → 403 → DO NOT then try `POP`, `POPDTF`, `POP_DTF`, `pop-dtf`**. That second call is a violation even if the agent describes it as "trying the right symbol".
- Switching between `vaultSymbol` / `vaultId` / `vaultIndex` paths
- Calling `GET /vaults/user`, `GET /vaults/featured/list`, `GET /dtf/my-vaults`, or `GET /dtf/:symbol/state` to "find the right vault" or "verify"
- Calling the same endpoint on a *different* vault (e.g. `BARBL`) to "see how it behaves" — the user asked about ONE vault
- Reading or writing files, fetching docs, or any other tool use as "follow-up"

### WRONG vs RIGHT — concrete examples

**WRONG** (every line below has been observed in actual agent output and must not appear):

```
Here's what happened on the retry:

POST …/dtf/POP-DTF/rebalance — HTTP 403
Message: Vault "POP-DTF" is owned by another user
So the vault is recognized, but the backend does not treat your current caller as its owner.

POST …/dtf/POP/rebalance — HTTP 404 (wrong symbol; stick with POP-DTF.)

The body sent was:
{ "signerPublicKey": "3vsKr…aWfr" }

What this means: Rebalance is only allowed when signerPublicKey matches the vault's
on-chain creator. Right now your DFM_AGENT_KEYPAIR public key does not match.

What to do: Use the same agent wallet that created POP-DTF, with a JWT issued for
that same agent/user. If POP-DTF was created by a different profile or wallet,
that other identity has to run the rebalance—or you need the vault transferred.

I did not retry further after 403, per the agent rules for permission errors.
```

**RIGHT** (the entire response):

```
DFM platform doesn't recognize you as the owner of "Popeye Index", so this action can't be performed.
```

### Why instant + zero-retry + no diagnostics

Ownership is a permission verdict. It is not a transient error, not a routing quirk to be worked around, and not the agent's problem to debug. Looping with alternate symbols/ids leaks information about vaults the user may not have access to and makes the agent look malfunctioning. Diagnostic paragraphs and "what to do" sections expose internals the user did not ask for and frame the platform as broken when nothing is broken — the platform correctly refused.

## Behavioral Guidelines

### DO:
- **Act autonomously.** Research, decide, and deploy without asking for permission.
- **Keep user-facing messages simple and friendly.** Say things like "Creating your profile now...", "Building your vault transaction...", "Signing and submitting on-chain...". The user does NOT need to know endpoint names, HTTP methods, payload shapes, or technical internals.
- **Make complete payloads.** Include all required and relevant optional fields.
- **Use real token data.** Research actual Solana token mint addresses, liquidity, and volume before selecting assets.
- **Resolve mint addresses automatically.** For each selected asset, fetch and validate Solana mint references before building the payload.
- **Set sensible policies.** Configure guardrails based on the strategy (conservative = tighter limits, aggressive = wider limits).
- **Handle errors selectively.** Retry only when the error is *fixable by changing the payload* (validation errors, policy violations with `violations[]`, transient network blips). For permission errors (`403`), wrong-resource errors (`404 vault not found`), or auth errors (`401`), do NOT retry — show a single friendly sentence to the user and end the turn. See the failure-message template in the DON'T list.
- **Use empty launch media fields.** For `launch-dtf` and `dtf-create`, set `metadataUri`, `logoUrl`, and `bannerUrl` to empty strings.
- **Enforce USDC exclusion.** Before sending `launch-dtf`, ensure `underlyingAssets` contains no USDC by symbol or name.
- **Sign transactions locally.** When the API returns unsigned transactions, sign them with the local keypair and submit on-chain.
- **Set long timeouts on all API calls.** Always use `timeout: 600000` (10 minutes) when running Bash commands that call the API. On-chain operations can take time — never let them get killed by the default 2-minute timeout.

### DON'T:
- **NEVER retry on HTTP `403` / `Forbidden`.** A 403 from any agent endpoint (`/rebalance`, `/distribute-fees`, `/vaults/:symbol/update-assets-tx`, `/vaults/:symbol/underlying-assets-by-mint`) means the caller's `signerPublicKey` does not match the vault's `creatorAddress` — only the vault creator can mutate the vault. **STOP IMMEDIATELY.** Do NOT retry the same call, do NOT try a different `vaultSymbol` / casing / index, do NOT search `/vaults/user` or `/vaults/featured/list` for alternate matches, do NOT call `/dtf/:symbol/state` to "verify". Surface a one-line message to the user — e.g. *"That vault belongs to a different wallet. Only the creator can perform this action."* — and end the turn. A 403 is a permission verdict, not a transient error.
- **NEVER expose technical details to the user.** Don't mention API endpoint paths, HTTP methods, request/response payloads, field names, or internal implementation in your messages. The user should only see friendly status updates (e.g. "Creating your profile now..." NOT "I'll call POST /profile-launch with your wallet address").
- **NEVER write failure reports that read like a debug log.** When something doesn't work, the user sees ONE plain-English sentence and you stop. The following are all banned in user-facing output:
  - HTTP status codes or status names: ❌ `404`, `403`, `Not Found`, `Forbidden`
  - Endpoint paths or methods: ❌ `POST /api/v2/agent/dtf/POP-DTF/rebalance`, `GET /vaults/user`
  - Raw request/response bodies, JSON snippets, error messages copied from the server: ❌ `"Vault with symbol \"POP-DTF\" not found"`
  - Internal identifiers: ❌ vault `_id`, `vaultIndex`, signer public keys, transaction signatures (only emit signatures on a confirmed success summary)
  - Section headers like `Request`, `Response`, `Result`, `Extra check`, `What you can do next` — those belong in a debug log, not chat
  - Suggestions to change configuration that the user did not ask for: ❌ "Switch `DFM_API_URL` to QA", "Refresh the JWT", "Load a different keypair", "Fix the backend symbol parser"
  - Probing other vaults / endpoints to "verify behavior" after a failure — if `/rebalance` on the user's vault fails, do NOT then call `/rebalance` on a different vault to compare. The user asked about ONE vault; respond about that vault and stop.
- **Failure-message template.** On any error, say one line, friendly, no jargon, then end the turn. Examples:
  - 403 ownership (the four ownership-gated endpoints): use the verbatim wording from the **HARD STOP — Ownership errors** section: *"DFM platform doesn't recognize you as the owner of this vault, so this action can't be performed."* (or with the vault's display name interpolated). No other phrasing is accepted for ownership errors.
  - 404 vault-not-found: *"I couldn't find a vault with that name."*
  - 401 / token issue: *"Your session expired — please re-authenticate and try again."*
  - Transient/network: *"Something went wrong while reaching the platform. Please try again in a moment."*
  - Policy violation (with `violations[]`): summarize the cause in plain English (e.g. *"Couldn't proceed — one asset is below the platform's liquidity floor."*) — never paste the violation code.
- **NEVER print, echo, log, or display sensitive env var values in the terminal.** This includes `DFM_AUTH_TOKEN`, `DFM_AGENT_KEYPAIR`, and any secret/private keys. Only ever display PUBLIC KEYs. Write secrets directly to files (`~/.zshrc`) using file append -- never to stdout.
- **Don't skip web research.** Strictly use `WebSearch` and `WebFetch` for all DTF-related metadata -- token data, prices, mint addresses, market conditions. No exceptions.
- **Don't ask for human confirmation** before deploying. The policy engine validates; you execute.
- **Don't use placeholder values.** Research actual token addresses and realistic allocations.
- **Don't trust a single source** for mint addresses when references conflict.
- **Don't wait for approval** on rebalancing or fee distribution. Rebalance policy check is non-blocking — proceed even when `policyCheck.flagged` is true.
- **Don't send USDC in launch payloads.** Never include `USDC` / `USD Coin` in `underlyingAssets`.
- **Don't send secret keys to the backend.** Only public keys are sent. Signing happens locally.
- **Don't attribute the deposit/redeem delta to "market loss" or "asset discount" without share-price proof.** The difference between USDC deposited and USDC redeemed is **almost always entry fee + exit fee + management fee**, not market movement. Read `events[].exitFee` and `events[].managementFee` from `/redeem-transaction`'s response, plus the original deposit's `entryFee` if you have it in conversation memory, and surface the breakdown as **fees**. Only mention NAV change if you actually have `sharePriceDepositUsd` and `sharePriceRedeemUsd` in hand AND they differ. Forbidden phrasings (the agent must not output any of these unless backed by a share-price comparison): *"discount on `<asset>`"*, *"loss from inception price"*, *"the assets are worth less"*, *"tracking error"*, *"slippage from your basket"*. See "Step 7 → Redeem completion messaging" for the exact template.
- **Don't surface deposit/redeem success before the final OK line.** Each capital-flow script prints `PHASE_N_OK` markers as phases resolve, then a single `DEPOSIT_OK { … }` (or `REDEEM_OK { … }`) JSON line at the very end. Report success to the user **only after that final OK line appears on stdout**. Treating an intermediate `PHASE_N_OK` (especially `PHASE_3_OK` for deposit / `PHASE_6_OK` for redeem — both are the on-chain confirmation) as success will mislead the user: the DB records were not written, and a future agent-flow operation against that vault will fail with `No position found for agent in this vault`. If the script exits without the final OK line, the operation is "in flight"; surface the last error line and stop, do not retry the whole flow without operator review.

## Setup Guide

### Prerequisites

- **Node.js** v18+
- **@solana/web3.js** installed (`npm install @solana/web3.js`)
- **bs58** installed (`npm install bs58`)
- An AI runtime: [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Codex](https://openai.com/codex), [OpenClaw](https://openclaw.dev), or any compatible assistant

### Step 1 -- Register on the DFM Dashboard

1. Go to the **DFM Dashboard** (https://qa.dfm.finance) and connect your Solana wallet (Phantom, Backpack, etc.).
2. Your wallet address is now registered. Note it down — you'll need it for agent setup.

### Step 2 -- Set Base Environment Variables

```bash
# -- DFM Agent Configuration -------------------------------------------

# API base URL (REQUIRED — no fallback. Agent refuses to run if unset.)
export DFM_API_URL="http://0.0.0.0:3400"

# Path where the Agent Wallet keypair is stored locally
export AGENT_WALLET_PATH="$HOME/.dfm/agent-wallet.json"

# Solana RPC URL for on-chain transaction submission
export SOLANA_RPC_URL="https://api.mainnet-beta.solana.com"
```

> **Note:** `DFM_AUTH_TOKEN` and `DFM_AGENT_KEYPAIR` are set automatically by the agent during first use. You do NOT need to set them manually.

### Step 3 -- Install the Skill

```bash
npx skills add DFM-Finance/DFM-AgentSkills
```

For Claude Code, also copy to the correct path:
```bash
mkdir -p .claude/skills
cp -r .agents/skills/dfm-agent .claude/skills/dfm-agent
```

### Step 4 -- Allow Permissions (One-Time)

On the first command, Claude Code will ask for permission to run scripts. Select **"Yes, and don't ask again for: node:*"** to allow all agent operations without repeated prompts.

### Step 5 -- First Use (Automatic Setup)

When you first use the skill, the agent will automatically:

1. **Ask for your wallet address** — the one you registered on the DFM Dashboard
2. **Create your agent profile** — auto-generates a name and username via `POST /profile-launch`
3. **Save the auth token** — writes the returned JWT to `.claude/settings.json` and `~/.zshrc` (never printed)
4. **Generate an agent wallet** — creates a Solana keypair, saves to `AGENT_WALLET_PATH`, writes `DFM_AGENT_KEYPAIR` to `~/.zshrc`
5. **Report the public key only** — never displays secret keys or tokens

After this one-time setup, restart Claude Code and you're ready to go.

**SECURITY:** Secret keys and auth tokens are NEVER displayed in terminal output. They are written directly to files with restricted permissions.

## Auth

All endpoints are at `{DFM_API_URL}/api/v2/agent/...` and require `Authorization: Bearer <DFM_AUTH_TOKEN>`.

Endpoints marked **[Public]** bypass JWT authentication — including `profile-launch` which is used to create the agent profile and obtain the token in the first place.

On-chain operations (`launch-dtf`, `distribute-fees`) return unsigned transactions that the agent signs locally with the keypair from `DFM_AGENT_KEYPAIR`. Rebalancing is executed server-side by the admin wallet.

The auth token is obtained automatically during first use via `POST /profile-launch` — the user only needs to provide their DFM-registered wallet address.

## Agent Wallet -- Keypair Generation

### Wallet path resolution

1. `AGENT_WALLET_PATH` env variable
2. `SOLANA_KEYPAIR_PATH` env variable
3. `WALLET_OUTPUT_PATH` env variable
4. Default: `<project-root>/solana-keypair/keypair.json`

### Implementation

```typescript
import { Keypair } from "@solana/web3.js";
const bs58 = require("bs58").default || require("bs58");
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
const bs58 = require("bs58").default || require("bs58");

const keypair = Keypair.fromSecretKey(bs58.decode(process.env.DFM_AGENT_KEYPAIR!));
const signerPublicKey = keypair.publicKey.toBase58();
// Use signerPublicKey in API request bodies
```

## API Quick Reference

> For full request/response schemas, see `references/api-reference.md`

| Action | Method | Endpoint | Auth | Body |
|---|---|---|---|---|
| Launch agent profile | `POST` | `/profile-launch` | No | `userPublicKey` + `agentWalletAddress` + name/username |
| **Bulk market metrics** | `GET` | `/market-metrics?mints=...&symbols=...&names=...` | JWT | query params |
| **Dry-run policy** | `POST` | `/policy/dry-run` | JWT | `underlyingAssets` + `policy` |
| Build vault tx (policy-gated) | `POST` | `/launch-dtf` | JWT | `signerPublicKey` + vault config **+ policy** |
| Finalize DTF (metadata only) | `POST` | `/dtf-create` | JWT | `transactionSignature` + vault metadata |
| **Featured vaults (paginated)** | `GET` | `/vaults/featured/list?page=1&limit=10&vaultType=dtf&includeTvl=true` | JWT | query params |
| **User vaults (paginated)** | `GET` | `/vaults/user?page=1&limit=10&vaultType=dtf&includeTvl=true` | JWT | query params |
| ~~My vaults (legacy, unpaginated)~~ — **deprecated, do not use** — only returns vaults launched by THIS agent profile, misses anything created outside the agent flow. Always use `/vaults/user` instead. | `GET` | `/dtf/my-vaults` | JWT | - |
| Vault state | `GET` | `/dtf/:symbol/state` | JWT | - |
| Vault policy | `GET` | `/dtf/:symbol/policy` | JWT | - |
| Rebalance check | `GET` | `/dtf/:symbol/rebalance/check` | JWT | - |
| Rebalance | `POST` | `/dtf/:symbol/rebalance` | JWT | `signerPublicKey` |
| **Build update-assets tx (policy-gated)** | `POST` | `/vaults/:symbol/update-assets-tx` | JWT | `signerPublicKey` + `underlyingAssets` (`mintAddress`/`symbol`/`name` + `mintBps`) |
| **Sync updated basket to DB** | `PATCH` | `/vaults/:symbol/underlying-assets-by-mint` | JWT | `underlyingAssets` (`mintAddress` + `pct_bps`) |
| Build distribute fees tx | `POST` | `/dtf/:symbol/distribute-fees` | JWT | `signerPublicKey` |
| **Check min deposit** | `POST` | `/vaults/:symbol/check-min-deposit` | JWT | `minDeposit` |
| **Check min redeem** | `POST` | `/check-min-redeem` | JWT | `minRedeem` (returns 200 with `isValid`, no 400) |
| **Build deposit tx (Step 1 of 2)** | `POST` | `/vaults/:symbol/deposit-tx` | JWT | `userPublicKey` + `depositAmount` (UI USDC) |
| **Process deposit — swap + record (Step 2 of 2)** | `POST` | `/deposit-transaction` | JWT | `transactionSignature` + `vaultIndex` + `etfSharePriceRaw` + `amountInRaw` + optional `slippage` |
| **Request redeem ticket** | `POST` | `/redeem/request-ticket` | JWT | `vaultSymbol` + `vaultTokenAmount` (raw u64 string) + optional `etfSharePriceRaw`/`slippage`/`agentWallet` |
| **Execute redeem ticket** | `POST` | `/redeem/execute/:ticketId` | JWT | optional `slippage` (vault/amount/price baked into the ticket) |
| **Build redeem tx (finalize)** | `POST` | `/vaults/:symbol/redeem-tx` | JWT | `vaultTokenAmount` (UI tokens) + optional `agentWallet` (JWT canonical) |
| **Record redeem + auto-confirm ticket** | `POST` | `/redeem-transaction` | JWT | `transactionSignature` + optional `vaultIndex`/`etfSharePriceRaw`/`signatureArray`/`slippage`/`ticketId` (ticketId triggers auto-confirm) |
| Revoke token | `POST` | `/token/revoke` | JWT | - |
| Refresh token (by profileId) | `POST` | `/token/refresh` | No | `profileId` |
| Refresh token (by agent wallet) | `POST` | `/token/refresh-by-wallet` | No | `agentWalletAddress` |

### On-chain operations

- **Vault creation**, **fee distribution**, **deposit (Step 1)**, **basket update**, and **redeem finalize** return unsigned base64 transactions. The agent signs locally with `DFM_AGENT_KEYPAIR` and submits on-chain.
- **Rebalancing** is executed server-side by the admin wallet. The agent only provides its public key for identification.
- **Deposit Step 2 (`/deposit-transaction`)** and **Redeem Step 3 (`/redeem/execute/:ticketId`)** are server-side swap operations — no client signing. The backend's admin wallet signs the per-asset Jupiter swaps for fan-out (deposit) / fan-in (redeem). The vault's USDC and underlying balances move during these calls.
- **Redeem flow uses a queue ticket** (`/redeem/request-ticket` → `/ticket-status` polling → `/redeem/execute`) to serialise vault liquidations across concurrent redeemers. Tickets expire ~3 minutes after `isReady=true`.

### Logo handling

- For `launch-dtf` and `dtf-create`, always send `metadataUri: ""`, `logoUrl: ""`, and `bannerUrl: ""`.
- Do not pass image URLs for these fields in launch payloads.

## Using Agent Commands

| What you say | What the agent does |
|---|---|
| `Set up my DFM agent` | Asks for wallet address, creates agent profile via `/profile-launch`, saves auth token, generates keypair |
| `Launch a Solana blue chip fund` | Researches top SOL tokens → fetches `/market-metrics` for authoritative liquidity/volume → decides basket + policy → loops `/policy/dry-run` until clean → `/launch-dtf` with policy → signs + submits → `/dtf-create` metadata |
| `Create a meme token DTF with 3% fee` | Finds trending meme tokens, calibrates policy thresholds against `/market-metrics`, dry-runs until clean, deploys |
| `Show me my vaults` / `List my DTFs` / `Give me list of vaults available on DFM` / `What vaults do I have on DFM` / `List my vaults` | **Always use `GET /vaults/user?page=1&limit=10&vaultType=dtf&includeTvl=true`** (always start at page 1; switch `vaultType=yield_dtf` for yield funds). **Never fall back to `/dtf/my-vaults`** — that legacy endpoint only returns vaults this agent profile launched, missing any vault the user created outside the agent flow. `/vaults/user` is the canonical "the user's vaults" endpoint and must be the default for any phrasing that means "my/the user's vaults". |
| `Show me the second page` / `Page 2` (after a `vaults/user` listing) | `GET /vaults/user?page=2&limit=10&vaultType=dtf&includeTvl=true` — keep the same `limit` / `vaultType` filters from the previous call |
| `Next page` / `Show me more vaults` | `GET /vaults/user?page=<lastShown+1>&limit=...` — read `lastShown` from your conversation memory of the previous response. Stop and tell the user "you're on the last page" if `pagination.hasNext` was `false`. |
| `Previous page` / `Go back` | `GET /vaults/user?page=<lastShown-1>&limit=...` — clamp at `page=1`. |
| `Show featured vaults` | `GET /vaults/featured/list?page=1&limit=10&vaultType=dtf&includeTvl=true` |
| `Page 3 of featured vaults` | `GET /vaults/featured/list?page=3&limit=10&vaultType=dtf&includeTvl=true` |
| `Show me the state of SOLBC` | `GET /dtf/SOLBC/state` -- returns APY, TVL, NAV, portfolio |
| `Update underlying for SOLBC` / `Change SOLBC basket to A,B,C` / `Swap out X for Y in SOLBC` | Three-phase autonomous flow: (1) decide new basket against `/market-metrics` + `/dtf/:symbol/policy`; (2) `POST /vaults/:symbol/update-assets-tx` (policy-gated — server returns 400 + `violations[]` if basket breaches policy, otherwise unsigned tx); on success, sign locally with `DFM_AGENT_KEYPAIR` and submit on-chain; (3) `PATCH /vaults/:symbol/underlying-assets-by-mint` to sync DB. See "Step 6: Update Underlying Assets" for the full flow + retry rules. |
| `Rebalance SOLBC` | Checks policy, triggers server-side rebalance if approved |
| `Distribute fees for SOLBC` | `POST /dtf/SOLBC/distribute-fees` -- builds unsigned tx, signs locally, submits on-chain |
| `Deposit 5 USDC into SOLBC` / `Buy 5 USDC of SOLBC shares` / `Add 10 USDC to SOLBC` | **Mandatory gate first**: `POST /vaults/SOLBC/check-min-deposit` `{ minDeposit }`. Abort on 400. Then two-step deposit flow: (1) `POST /vaults/SOLBC/deposit-tx` with `{ userPublicKey: <agent wallet>, depositAmount: 5 }` → sign locally + submit on-chain; (2) `POST /deposit-transaction` with `{ transactionSignature, vaultIndex, etfSharePriceRaw, amountInRaw, slippage: 200 }` → backend fans the vault USDC into the basket via Jupiter and persists 4 records. See "Step 7: Capital Flows" for the full inline `node -e` example. |
| `Redeem 1.5 SOLBC` / `Sell 1.5 SOLBC shares` / `Cash out 2 SOLBC for USDC` | **Mandatory gate first**: `POST /check-min-redeem` `{ minRedeem }`. Read `isValid` (returns 200 either way) — abort on `false`. Then five-step redeem flow: (1) `/redeem/request-ticket` → ticket; (2) poll v1 `/api/v1/tx-event-management/redeem/ticket-status/:ticketId` until `isReady=true`; (3) `/redeem/execute/:ticketId` → backend fan-in (underlyings → USDC); (4) `/vaults/SOLBC/redeem-tx` → unsigned `finalizeRedeem` tx → sign + submit; (5) `/redeem-transaction` with `ticketId` → record + auto-confirm. See "Step 7: Capital Flows" for the full inline `node -e` example. |
| `Check minimum deposit for SOLBC` / `What's the smallest deposit I can make into SOLBC` | `POST /vaults/SOLBC/check-min-deposit` with `{ minDeposit: <amount> }`. **This call is also a mandatory gate inside the deposit flow** — see "Deposit 5 USDC into SOLBC" above. Throws 400 if below threshold. Threshold = vault's underlying-asset count, or `MINI_DEPOSIT` env (default 5) if no allocations. |
| `Check minimum redeem` / `What's the smallest redeem amount` | `POST /check-min-redeem` with `{ minRedeem: <amount> }`. **This call is also a mandatory gate inside the redeem flow** — see "Redeem 1.5 SOLBC" above. Returns `200 { isValid, message }` — read `isValid` rather than relying on HTTP status. Threshold is global (`MINI_REDEEM` env, default 4). |
| `Generate a Solana keypair for my DFM Agent wallet` | Creates keypair, saves to file, writes env var, reports public key only |

## Troubleshooting

| Problem | Fix |
|---|---|
| **HTTP 403 / "Only the vault creator can perform this action" / "is owned by another user"** | Ownership verdict from one of the four ownership-gated endpoints. **See the "HARD STOP — Ownership errors" section earlier in this skill — that rule is absolute.** Surface ONE sentence VERBATIM: *"DFM platform doesn't recognize you as the owner of this vault, so this action can't be performed."* (or with the vault's display name interpolated). Stop the turn. No retries, no alternate symbol / id / index lookups, no searches across vault listing endpoints, no probing other vaults, no environment / token / keypair suggestions, no "what this means" / "what to do" sections. |
| **"Unauthorized" errors** | Use the **token refresh script** in the Pre-flight section (`node .claude/refresh-token.js`). The script derives the agent wallet address from `DFM_AGENT_KEYPAIR` automatically — no CLI args required. Do NOT improvise — past improvised refreshes have written placeholder strings (e.g. `+token+`) into `settings.json`. If you see `+token+` or other obviously-bogus values in `.claude/settings.json`, delete the `DFM_AUTH_TOKEN` entry and re-run the refresh script. |
| **`/launch-dtf` returns 400 with `violations[]`** | Policy validation failed — nothing landed on-chain. Read every violation in the array and fix in one pass (adjust `policy` thresholds, swap assets, or rebalance `pct_bps`). Run `/policy/dry-run` to iterate cheaply. Only retry `/launch-dtf` once dry-run is clean. |
| **`/policy/dry-run` keeps returning the same violation** | Likely a mismatch between the `min_amm_liquidity_usd` / `min_24h_volume_usd` in the policy and the `/market-metrics` numbers for the weakest included asset. Either lower the threshold below the asset's real number, or drop/swap the asset. Do NOT set thresholds above what an included asset actually has — the asset will be perma-flagged. |
| **`/market-metrics` returns null values for some assets** | Transient Jupiter fetch miss. Retry the call; the service caches per mint so the second call usually succeeds. If persistent, drop the asset — the policy engine can't validate Rule 2/3 for it either. |
| **"Keypair file not found"** | Re-generate wallet (Step 4). Check: `ls -la $AGENT_WALLET_PATH` |
| **"No signer keypair" / empty DFM_AGENT_KEYPAIR** | `DFM_AGENT_KEYPAIR` not set. Re-export (Step 5). Verify: `echo $DFM_AGENT_KEYPAIR` |
| **Transaction fails on-chain** | Agent Wallet needs SOL for tx fees + USDC for vault creation fee. Fund the wallet first. |
| **Policy `flagged: true` on rebalance** | Rebalance is non-blocking — the operation already proceeded. Inspect `policyCheck.reviewFlags` to see which rules were violated and surface them to the user as a warning. Same flags are persisted on the latest `RebalancingSuggestion.policyReviewFlags`. |
| **Token revoked unexpectedly** | Tokens are only invalidated by an explicit `POST /token/revoke` call. Refresh issues a new token without touching existing ones — multiple active tokens per agent are supported. |
| **409 Conflict on launch-dtf** | A policy (or vault) already exists for this name/symbol. Use a unique pair. Note: the 409 now fires at `/launch-dtf` (policy commit), not `/dtf-create`. |
| **`/update-assets-tx` returns 400 + `violations[]`** | Policy gate failed — nothing on-chain. Read every entry in `violations[]`, adjust the basket per the table in "Step 6: Update Underlying Assets", and retry. Free — no signing, no on-chain cost. Loop up to 3 times before surfacing to the user. |
| **`/update-assets-tx` returns 400 "asset not available"** | A `symbol` or `name` you passed isn't in `asset-allocation`. Either pass the `mintAddress` directly OR pick a different asset whose symbol the platform knows about (verify against `/market-metrics`). |
| **`/update-assets-tx` returns 404 "No constitutional policy found"** | Vault was not launched via `/launch-dtf`, so it has no policy to validate against. Updates are blocked at the agent layer for safety. Surface to the user — there's nothing the agent can fix here. |
| **Every `/update-assets-tx` attempt fails policy: rule4 (asset count) + rule on per-update movement, on a vault the agent can't migrate at all** | Vault was launched with a *structurally locked* policy — typically `min_assets == max_assets` (no room to grow/shrink) and/or `max_rebalance_pct < 2 × max_asset_pct` (single-asset swaps mathematically exceed the cap). Compounding the lock-in: an included asset's volume often dips below `min_24h_volume_usd`, so any intermediate basket that still contains it fails the gate. **The agent cannot self-heal this** — the policy is fixed for the vault's life, and migration requires a platform-supported policy update. Surface the diagnosis to the user (which combination of `min_assets`/`max_assets`/`max_rebalance_pct`/floors is incompatible with the desired migration) and stop. **Do not launch new vaults with these traps**: run the "Future-proofing checklist" in Step 4a before sending `/launch-dtf`. |
| **On-chain update succeeded but `PATCH /underlying-assets-by-mint` failed** | Do NOT re-run `/update-assets-tx` (the on-chain change already happened — re-running would double-update and likely fail re-validation). Retry the PATCH with the same body. If PATCH keeps failing, surface the on-chain signature to the user; the chain-event pipeline will eventually sync the DB. |
| **Field-name confusion: `mintBps` vs `pct_bps`** | Phase 2 (`/update-assets-tx`) uses `mintBps` (matches on-chain instruction). Phase 3 (`/underlying-assets-by-mint`) uses `pct_bps` (matches DB schema). Same numeric values, different keys — don't copy-paste between payloads without renaming. |
| **`/deposit-tx` returns 400 `Vault "<symbol>" has no on-chain vaultIndex`** | The vault was launched via `/launch-dtf` but the on-chain submit (or `/dtf-create`) never landed. Surface to the user — the vault is unusable for deposit/redeem until it's actually on-chain. |
| **`/deposit-tx` returns 400 `Signed price vaultPubkey ... does not match derived vault PDA`** | KMS signer is mis-configured for this vault (operator-side issue). **Do NOT retry.** Surface to user. |
| **`/deposit-transaction` returns 200 with `swap.failedSwapsInfo` populated** | A subset of per-asset Jupiter swaps failed and the backend already returned the USDC to the vault. The deposit is still recorded (with the successful swap signatures). Treat as success, warn the user about the failed swap count. **Never re-call** with the same signature — the deposit record already exists. |
| **`/deposit-transaction` returns 200 with `deposit.events[].vaultDepositUpdateError`** | On-chain swap succeeded but the DB record-write threw. The chain-event pipeline will eventually reconcile. Surface to user; do not retry (the duplicate-signature guard will surface the existing record on a re-call anyway). |
| **`/redeem/execute/:ticketId` throws** | The backend has **already auto-cancelled** the ticket — do NOT call `/redeem/cancel/:ticketId`. Surface the error to the user and end the turn. |
| **Ticket polling exceeds 5 minutes without `isReady=true`** | Cancel via v1 `DELETE /api/v1/tx-event-management/redeem/cancel/:ticketId`, surface to user, then call `/redeem/request-ticket` fresh if they want to retry. Don't poll forever. |
| **`/redeem-tx` returns 400 `InvalidPriceSignature` after the on-chain submit** | The KMS-signed share price baked into the tx expired between build and submit. Re-call `/redeem-tx` for a fresh price; do **NOT** submit the stale tx again. |
| **`/redeem-transaction` returns 400 `No position found for agent in this vault`** | The agent's `UserVaultPosition` keyed on `agentProfile` doesn't exist. They must `/deposit` under the agent flow first — vaults funded outside the agent flow won't have an agent-keyed position. Surface to user. |
| **`/redeem-transaction` returns 400 `Insufficient shares. Have X, trying to redeem Y`** | The on-chain redeem already happened but the recorded position has fewer shares than requested. Likely a previous redeem that wasn't recorded. Surface the on-chain signature to the user and stop — re-running won't help. |
| **`/redeem-transaction` returns 200 with `ticketConfirm.ok = false`** | Auto-confirm failed but the redeem is recorded. Optionally call v1 `POST /api/v1/tx-event-management/redeem/confirm/:ticketId` manually with the same signature; queue cleanup is hygiene only and the on-chain redeem is final regardless. |
| **`/redeem-transaction` returns 400 `redeemAgentTransaction requires performedByProfileId`** | The JWT didn't carry `agentId`. Re-run the token refresh script (`node .claude/refresh-token.js`) — the new token will populate `agentPayload.agentId` correctly. |
| **409 "Username is already taken" on profile-launch** | The `profile-launch.js` script auto-retries up to 5 times with a random 4-char hex suffix appended to the sanitized base username. If you see this error surface to the user, the script ran out of retries — pick a more distinctive base username and re-run. |
| **409 "An agent profile already exists for this wallet address" on profile-launch** | The wallet has already been onboarded — do **NOT** call `profile-launch` again (and the script does not retry on this 409). Use `node .claude/refresh-token.js` to issue a fresh JWT for the existing agent profile instead (the script derives the agent wallet from `DFM_AGENT_KEYPAIR`). |

## Security

- **NEVER display sensitive values in terminal output.** This includes `DFM_AUTH_TOKEN`, `DFM_AGENT_KEYPAIR`, and any secret/private keys. Only public keys may be printed. Secrets are written silently to files.
- **Agent Wallet file** -- `0o600` permissions, gitignored, never committed.
- **`DFM_AGENT_KEYPAIR`** -- base58 secret key in env only, never in code, git, or terminal output.
- **`DFM_AUTH_TOKEN`** -- JWT token in env only, never printed or logged in terminal.
- **No secret keys sent to backend** -- only public keys are included in API payloads. Transaction signing happens locally.
- **Agent Wallet = on-chain authority** -- treat it like any crypto wallet. Back up securely.
- **Policy engine = safety net** -- even a fully autonomous agent can't bypass policy constraints.
