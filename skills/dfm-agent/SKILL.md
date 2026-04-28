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

1. Ask the user for their **DFM-registered wallet address** (only if you don't already know it).
2. Run the **token refresh script below** with the wallet address. The script calls `POST {DFM_API_URL}/api/v2/agent/token/refresh-by-wallet`, writes the new JWT to `.claude/settings.json`, and **replaces** any existing `export DFM_AUTH_TOKEN=` line in `~/.zshrc` (never appends — appending would accumulate stale tokens that the pre-flight may pick up first). The token value is **never printed**.
3. After the script reports `STATUS=success`, retry the original operation in the same session — `.claude/settings.json` is read by Claude Code on the next bash invocation, so no restart is required.

**DO NOT improvise the refresh.** Earlier improvised attempts have written literal placeholder strings (e.g. `+token+`) into `settings.json`. Always use this exact script.

Write the script once to `.claude/refresh-token.js`, then run it with `node .claude/refresh-token.js <WALLET_ADDRESS>`:

```javascript
const http = require("http");
const https = require("https");
const fs = require("fs");
const path = require("path");
const os = require("os");

const apiUrl = process.env.DFM_API_URL;
const walletAddress = process.argv[2];
if (!apiUrl) { console.log("ERROR: DFM_API_URL not set"); process.exit(1); }
if (!walletAddress) { console.log("ERROR: usage: node refresh-token.js <walletAddress>"); process.exit(1); }

const payload = JSON.stringify({ walletAddress });
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
- **Calibrate `min_amm_liquidity_usd` and `min_24h_volume_usd`** in the policy below the weakest selected asset's real number — setting the floor above an included asset guarantees a policy violation the agent can't self-heal later (the asset gets "locked" in the basket).
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
    "min_assets": 3,
    "max_assets": 12,
    "max_asset_pct": 4000,
    "min_asset_pct": 500,
    "min_stablecoin_pct": 0,
    "max_rebalance_pct": 2500,
    "min_rebalance_interval_hours": 4,
    "max_rebalances_per_day": 3,
    "max_rebalances_per_week": 14,
    "launch_blackout_hours": 24,
    "fee_locked": true,
    "notes": "Auto-generated policy for blue chip strategy"
  }
}
```

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

The agent MUST decide ALL policy values based on the vault strategy. **These values go into the `policy` sub-object of `/launch-dtf` (Step 4a) — not into `/dtf-create`.** Calibrate the liquidity/volume floors below the weakest selected asset's `/market-metrics` number to avoid locking an asset in violation of the vault's own policy.

| Strategy Type | `max_asset_pct` | `min_asset_pct` | `min_amm_liquidity_usd` | `min_24h_volume_usd` | `max_rebalances_per_day` | `min_rebalance_interval_hours` |
|---|---|---|---|---|---|---|
| **Conservative** (blue chip, index) | 3000-4000 | 500-1000 | 500000 | 1000000 | 2 | 6 |
| **Moderate** (mixed, ecosystem) | 4000-5000 | 500 | 100000 | 500000 | 3 | 4 |
| **Aggressive** (meme, trending) | 5000-6000 | 300 | 50000 | 100000 | 4 | 2 |
| **Yield** (LSTs, staking, yield) | 4000-5000 | 500-1000 | 500000 | 500000 | 1 | 12 |

Always set:
- `asset_mode`: choose based on the vault strategy:
  - `"OPEN"` — any asset can be added. No restrictions. Use for broad market / index / aggressive strategies.
  - `"WHITELIST_ONLY"` — only assets in `asset_whitelist` are allowed. Use for curated funds (e.g. "only blue chips", "only LSTs"). Set `asset_whitelist` to the mint addresses of the selected assets.
  - `"OPEN_BLACKLIST"` — all assets allowed except those in `asset_blacklist`. Use when you want to exclude specific risky assets. Set `asset_blacklist` to the mint addresses to exclude.
  - `"WHITELIST_BLACKLIST"` — only whitelisted assets allowed, with additional blacklist exclusions. Use for strict curated funds with explicit exclusions. Set both `asset_whitelist` and `asset_blacklist`.
  - **Decision rule:** If the user asks for a specific category fund (e.g. "LST fund", "blue chip only", "top 5 DeFi tokens"), use `WHITELIST_ONLY`. If the user asks for broad exposure, use `OPEN`. If the user says "exclude meme coins" or similar, use `OPEN_BLACKLIST`.
- `min_assets`: set to the number of assets in the vault (or lower)
- `max_assets`: `12` (hard max)
- `max_rebalance_pct`: `2000`-`3000` (20-30% max change per rebalance)
- `max_rebalances_per_week`: `max_rebalances_per_day * 7` or less
- `launch_blackout_hours`: `24` (prevent rebalancing in first 24h)
- `fee_locked`: `true` (always lock fees)
- `notes`: brief description of the strategy and policy rationale

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
- Monitors vault state via `GET {DFM_API_URL}/api/v2/agent/dtf/:symbol/state`
- Checks rebalancing readiness via `GET {DFM_API_URL}/api/v2/agent/dtf/:symbol/rebalance/check`
- Executes rebalances via `POST {DFM_API_URL}/api/v2/agent/dtf/:symbol/rebalance` (admin wallet executes behind the scenes)
- Distributes accrued fees via `POST {DFM_API_URL}/api/v2/agent/dtf/:symbol/distribute-fees` (returns unsigned tx for agent to sign)

All management operations are single API calls. No confirmation needed.

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

## Behavioral Guidelines

### DO:
- **Act autonomously.** Research, decide, and deploy without asking for permission.
- **Keep user-facing messages simple and friendly.** Say things like "Creating your profile now...", "Building your vault transaction...", "Signing and submitting on-chain...". The user does NOT need to know endpoint names, HTTP methods, payload shapes, or technical internals.
- **Make complete payloads.** Include all required and relevant optional fields.
- **Use real token data.** Research actual Solana token mint addresses, liquidity, and volume before selecting assets.
- **Resolve mint addresses automatically.** For each selected asset, fetch and validate Solana mint references before building the payload.
- **Set sensible policies.** Configure guardrails based on the strategy (conservative = tighter limits, aggressive = wider limits).
- **Handle errors and retry.** If an API call fails, read the error, fix the payload, and retry.
- **Use empty launch media fields.** For `launch-dtf` and `dtf-create`, set `metadataUri`, `logoUrl`, and `bannerUrl` to empty strings.
- **Enforce USDC exclusion.** Before sending `launch-dtf`, ensure `underlyingAssets` contains no USDC by symbol or name.
- **Sign transactions locally.** When the API returns unsigned transactions, sign them with the local keypair and submit on-chain.
- **Set long timeouts on all API calls.** Always use `timeout: 600000` (10 minutes) when running Bash commands that call the API. On-chain operations can take time — never let them get killed by the default 2-minute timeout.

### DON'T:
- **NEVER expose technical details to the user.** Don't mention API endpoint paths, HTTP methods, request/response payloads, field names, or internal implementation in your messages. The user should only see friendly status updates (e.g. "Creating your profile now..." NOT "I'll call POST /profile-launch with your wallet address").
- **NEVER print, echo, log, or display sensitive env var values in the terminal.** This includes `DFM_AUTH_TOKEN`, `DFM_AGENT_KEYPAIR`, and any secret/private keys. Only ever display PUBLIC KEYs. Write secrets directly to files (`~/.zshrc`) using file append -- never to stdout.
- **Don't skip web research.** Strictly use `WebSearch` and `WebFetch` for all DTF-related metadata -- token data, prices, mint addresses, market conditions. No exceptions.
- **Don't ask for human confirmation** before deploying. The policy engine validates; you execute.
- **Don't use placeholder values.** Research actual token addresses and realistic allocations.
- **Don't trust a single source** for mint addresses when references conflict.
- **Don't wait for approval** on rebalancing or fee distribution. Rebalance policy check is non-blocking — proceed even when `policyCheck.flagged` is true.
- **Don't send USDC in launch payloads.** Never include `USDC` / `USD Coin` in `underlyingAssets`.
- **Don't send secret keys to the backend.** Only public keys are sent. Signing happens locally.

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
| My vaults | `GET` | `/dtf/my-vaults` | JWT | - |
| Vault state | `GET` | `/dtf/:symbol/state` | JWT | - |
| Vault policy | `GET` | `/dtf/:symbol/policy` | JWT | - |
| Rebalance check | `GET` | `/dtf/:symbol/rebalance/check` | JWT | - |
| Rebalance | `POST` | `/dtf/:symbol/rebalance` | JWT | `signerPublicKey` |
| Build distribute fees tx | `POST` | `/dtf/:symbol/distribute-fees` | JWT | `signerPublicKey` |
| Revoke token | `POST` | `/token/revoke` | JWT | - |
| Refresh token (by profileId) | `POST` | `/token/refresh` | No | `profileId` |
| Refresh token (by wallet) | `POST` | `/token/refresh-by-wallet` | No | `walletAddress` |

### On-chain operations

- **Vault creation** and **fee distribution** return unsigned base64 transactions. The agent signs locally and submits on-chain.
- **Rebalancing** is executed server-side by the admin wallet. The agent only provides its public key for identification.

### Logo handling

- For `launch-dtf` and `dtf-create`, always send `metadataUri: ""`, `logoUrl: ""`, and `bannerUrl: ""`.
- Do not pass image URLs for these fields in launch payloads.

## Using Agent Commands

| What you say | What the agent does |
|---|---|
| `Set up my DFM agent` | Asks for wallet address, creates agent profile via `/profile-launch`, saves auth token, generates keypair |
| `Launch a Solana blue chip fund` | Researches top SOL tokens → fetches `/market-metrics` for authoritative liquidity/volume → decides basket + policy → loops `/policy/dry-run` until clean → `/launch-dtf` with policy → signs + submits → `/dtf-create` metadata |
| `Create a meme token DTF with 3% fee` | Finds trending meme tokens, calibrates policy thresholds against `/market-metrics`, dry-runs until clean, deploys |
| `Show me the state of SOLBC` | `GET /dtf/SOLBC/state` -- returns APY, TVL, NAV, portfolio |
| `Rebalance SOLBC` | Checks policy, triggers server-side rebalance if approved |
| `Distribute fees for SOLBC` | `POST /dtf/SOLBC/distribute-fees` -- builds unsigned tx, signs locally, submits on-chain |
| `Generate a Solana keypair for my DFM Agent wallet` | Creates keypair, saves to file, writes env var, reports public key only |

## Troubleshooting

| Problem | Fix |
|---|---|
| **"Unauthorized" errors** | Use the **token refresh script** in the Pre-flight section (`node .claude/refresh-token.js <wallet>`). Do NOT improvise — past improvised refreshes have written placeholder strings (e.g. `+token+`) into `settings.json`. If you see `+token+` or other obviously-bogus values in `.claude/settings.json`, delete the `DFM_AUTH_TOKEN` entry and re-run the refresh script. |
| **`/launch-dtf` returns 400 with `violations[]`** | Policy validation failed — nothing landed on-chain. Read every violation in the array and fix in one pass (adjust `policy` thresholds, swap assets, or rebalance `pct_bps`). Run `/policy/dry-run` to iterate cheaply. Only retry `/launch-dtf` once dry-run is clean. |
| **`/policy/dry-run` keeps returning the same violation** | Likely a mismatch between the `min_amm_liquidity_usd` / `min_24h_volume_usd` in the policy and the `/market-metrics` numbers for the weakest included asset. Either lower the threshold below the asset's real number, or drop/swap the asset. Do NOT set thresholds above what an included asset actually has — the asset will be perma-flagged. |
| **`/market-metrics` returns null values for some assets** | Transient Jupiter fetch miss. Retry the call; the service caches per mint so the second call usually succeeds. If persistent, drop the asset — the policy engine can't validate Rule 2/3 for it either. |
| **"Keypair file not found"** | Re-generate wallet (Step 4). Check: `ls -la $AGENT_WALLET_PATH` |
| **"No signer keypair" / empty DFM_AGENT_KEYPAIR** | `DFM_AGENT_KEYPAIR` not set. Re-export (Step 5). Verify: `echo $DFM_AGENT_KEYPAIR` |
| **Transaction fails on-chain** | Agent Wallet needs SOL for tx fees + USDC for vault creation fee. Fund the wallet first. |
| **Policy `flagged: true` on rebalance** | Rebalance is non-blocking — the operation already proceeded. Inspect `policyCheck.reviewFlags` to see which rules were violated and surface them to the user as a warning. Same flags are persisted on the latest `RebalancingSuggestion.policyReviewFlags`. |
| **Token revoked unexpectedly** | Tokens are only invalidated by an explicit `POST /token/revoke` call. Refresh issues a new token without touching existing ones — multiple active tokens per agent are supported. |
| **409 Conflict on launch-dtf** | A policy (or vault) already exists for this name/symbol. Use a unique pair. Note: the 409 now fires at `/launch-dtf` (policy commit), not `/dtf-create`. |
| **409 "Username is already taken" on profile-launch** | The `profile-launch.js` script auto-retries up to 5 times with a random 4-char hex suffix appended to the sanitized base username. If you see this error surface to the user, the script ran out of retries — pick a more distinctive base username and re-run. |
| **409 "An agent profile already exists for this wallet address" on profile-launch** | The wallet has already been onboarded — do **NOT** call `profile-launch` again (and the script does not retry on this 409). Use `node .claude/refresh-token.js <WALLET_ADDRESS>` to issue a fresh JWT for the existing agent profile instead. |

## Security

- **NEVER display sensitive values in terminal output.** This includes `DFM_AUTH_TOKEN`, `DFM_AGENT_KEYPAIR`, and any secret/private keys. Only public keys may be printed. Secrets are written silently to files.
- **Agent Wallet file** -- `0o600` permissions, gitignored, never committed.
- **`DFM_AGENT_KEYPAIR`** -- base58 secret key in env only, never in code, git, or terminal output.
- **`DFM_AUTH_TOKEN`** -- JWT token in env only, never printed or logged in terminal.
- **No secret keys sent to backend** -- only public keys are included in API payloads. Transaction signing happens locally.
- **Agent Wallet = on-chain authority** -- treat it like any crypto wallet. Back up securely.
- **Policy engine = safety net** -- even a fully autonomous agent can't bypass policy constraints.
