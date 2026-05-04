# DFM Agent API Reference

Base URL: `{DFM_API_URL}/api/v2/agent`

The `DFM_API_URL` environment variable defines the API host. All endpoints below are relative to this base URL.

`DFM_API_URL` must be set in the environment. There is no default — if not set, STOP and ask the user to set it.

## Authentication

All endpoints are protected by `AgentAuthGuard` and require a valid JWT token in the `Authorization` header:

```
Authorization: Bearer <DFM_AUTH_TOKEN>
```

Endpoints marked **[Public]** bypass authentication.

| Detail | Value |
|--------|-------|
| Token type | JWT (Bearer) |
| Expiry | 30 days |
| Revocable | Yes -- via dashboard or `POST /token/revoke` |
| Request identity | Guard decodes JWT and attaches `agentPayload`, `agentId`, `userId` to request |

### On-chain signing

Endpoints that build on-chain transactions (`launch-dtf`, `distribute-fees`) return unsigned base64-encoded `VersionedTransaction`s. The agent signs them locally using the keypair from `DFM_AGENT_KEYPAIR` and submits on-chain. **No secret keys are sent to the backend.**

Rebalancing (`rebalance`) is executed server-side by the admin wallet. The agent only provides its public key for identification.

### Signing and submitting unsigned transactions

```typescript
import { Keypair, VersionedTransaction, Connection } from "@solana/web3.js";
const bs58 = require("bs58").default || require("bs58");

const keypair = Keypair.fromSecretKey(bs58.decode(process.env.DFM_AGENT_KEYPAIR!));
const connection = new Connection(process.env.SOLANA_RPC_URL || "https://api.mainnet-beta.solana.com");

async function signAndSend(base64Tx: string): Promise<string> {
  const tx = VersionedTransaction.deserialize(Buffer.from(base64Tx, "base64"));
  tx.sign([keypair]);
  const sig = await connection.sendRawTransaction(tx.serialize(), {
    skipPreflight: false, preflightCommitment: "confirmed",
  });
  await connection.confirmTransaction(sig, "confirmed");
  return sig;
}
```

---

## 1. POST `/profile-launch` - Launch Agent Profile [Public]

Creates an agent profile by looking up the user via wallet public key. Requires the agent's Solana wallet address. Returns agent profile and JWT token.

**Request Body:**
```json
{
  "userPublicKey": "YourDFMWalletHere",
  "agentWalletAddress": "YourAgentWalletPublicKeyHere",
  "name": "Alpha Agent",
  "username": "agent_alpha",
  "metadata": [{ "key": "strategy", "value": "momentum" }]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `userPublicKey` | string | Yes | DFM-registered wallet address |
| `agentWalletAddress` | string | Yes | Agent's Solana wallet public key (for on-chain vault creation) |
| `name` | string | Yes | Display name |
| `username` | string | Yes | Unique username (alphanumeric + underscores) |
| `metadata` | array | No | Arbitrary key-value metadata |

**Response (201):**
```json
{
  "agentProfile": {
    "_id": "664a...",
    "profileId": "60f7...",
    "agentWalletAddress": "YourAgentWalletPublicKeyHere",
    "name": "Alpha Agent",
    "username": "agent_alpha"
  },
  "token": { "token": "eyJhbGciOi...", "expiresIn": "30d" }
}
```

**Errors:**
- `400` Validation error (missing required field, or `username` violates `/^[a-zA-Z0-9_]+$/`)
- `404` No profile found for the given `userPublicKey`
- `409` `"Username is already taken"` — `profile-launch.js` auto-retries with a random suffix (matched via `UNAME_RX = /username/i`)
- `409` `"An agent profile already exists for this wallet address"` — script does NOT retry; agent should call `/token/refresh-by-wallet` instead

---

## 2. POST `/profile` - Create Agent Profile (by profileId) [Public]

Creates a new agent profile linked to an existing user profile. Returns a JWT auth token for subsequent API calls.

**Request Body:**
```json
{
  "profileId": "60f7b3b3b3b3b3b3b3b3b3b3",
  "name": "Alpha Agent",
  "username": "agent_alpha",
  "metadata": [{ "key": "strategy", "value": "momentum" }]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `profileId` | string (MongoId) | No | Link to existing user profile |
| `name` | string | Yes | Display name |
| `username` | string | Yes | Unique username (alphanumeric + underscores) |
| `metadata` | array | No | Arbitrary key-value metadata |

**Response (201):**
```json
{
  "_id": "664a1b2c3d4e5f6a7b8c9d0e",
  "profileId": "60f7b3b3b3b3b3b3b3b3b3b3",
  "name": "Alpha Agent",
  "username": "agent_alpha",
  "metadata": [{ "key": "strategy", "value": "momentum" }],
  "isDeleted": false,
  "createdAt": "2026-04-10T12:00:00.000Z",
  "updatedAt": "2026-04-10T12:00:00.000Z"
}
```

**Errors:** `400` Validation error | `404` Profile not found for given profileId | `409` Username taken or agent profile already exists

---

## 3. POST `/launch-dtf` - Build Vault Creation Transaction (Policy-Gated) [Authenticated]

Builds an unsigned vault creation transaction AND commits the constitutional policy in one call. Before producing the transaction, the backend runs the proposed basket against the supplied policy (same rules as `/policy/dry-run`, Section 14). If any rule violates, returns `400` with `violations[]` and **nothing is committed or built** — safe to fix and retry with no on-chain cost.

**Request Body:**
```json
{
  "signerPublicKey": "<public key derived from DFM_AGENT_KEYPAIR>",
  "vaultName": "Blue Chip Fund",
  "vaultSymbol": "BCF",
  "underlyingAssets": [
    { "symbol": "SOL", "mintBps": 5000 },
    { "name": "Bonk", "mintBps": 5000 }
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
    "max_assets": 12,
    "max_asset_pct": 6000,
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

**Required fields:** `signerPublicKey`, `vaultName`, `vaultSymbol`, `underlyingAssets`, `managementFees`, `policy`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `signerPublicKey` | string | Yes | Base58-encoded public key of the signer (fee payer) |
| `vaultName` | string | Yes | Vault name (max 32 chars) |
| `vaultSymbol` | string | Yes | Vault symbol (max 10 chars) |
| `underlyingAssets` | array | Yes | Assets with allocations (symbol/name preferred, backend resolves mintAddress) |
| `managementFees` | number | Yes | Management fees in basis points (0-10000) |
| `metadataUri` | string | No | IPFS metadata URI (default: `""`) |
| `category` | number | No | For agent API launches, use `0` (Manual) only |
| `threshold` | number/null | No | Rebalance threshold in bps (default: `null`) |
| `policy` | object | Yes | Constitutional policy (see fields below) |

**Policy sub-object fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `asset_mode` | `"OPEN"` / `"WHITELIST_ONLY"` / `"OPEN_BLACKLIST"` / `"WHITELIST_BLACKLIST"` | `"OPEN"` | Asset restriction mode |
| `asset_whitelist` | string[] | `[]` | Whitelisted mint addresses |
| `asset_blacklist` | string[] | `[]` | Blacklisted mint addresses |
| `min_amm_liquidity_usd` | number | `0` | Min AMM liquidity (0=disabled) |
| `min_24h_volume_usd` | number | `0` | Min 24h volume (0=disabled) |
| `min_assets` / `max_assets` | number | `0` | Asset count bounds (min 1, max 12; 0=disabled) |
| `max_asset_pct` / `min_asset_pct` | number | `0` | Allocation bounds in bps |
| `min_stablecoin_pct` | number | `0` | Min stablecoin % in bps |
| `max_rebalance_pct` | number | `0` | Max rebalance % in bps |
| `min_rebalance_interval_hours` | number | `0` | Hours between rebalances |
| `max_rebalances_per_day` | number | `0` | Daily rebalance cap (0=unlimited) |
| `max_rebalances_per_week` | number | `0` | Weekly rebalance cap |
| `launch_blackout_hours` | number | `0` | Blackout after launch |
| `fee_locked` | boolean | `true` | Prevent fee changes |
| `notes` | string | - | Policy notes |

**Underlying asset rules:**
- Pass `symbol` or `name` in each `underlyingAssets[]` item (preferred). Backend resolves `mintAddress` from `asset-allocation`.
- **Minimum 1, maximum 12 assets** in `underlyingAssets`. Payloads outside this range are rejected.
- Do not include USDC (`symbol: "USDC"` or `name: "USD Coin"`), as backend blocks USDC for agent-created vault launches.
- Each asset's liquidity/volume (per Jupiter) must meet `policy.min_amm_liquidity_usd` / `policy.min_24h_volume_usd`. Use `GET /market-metrics` (Section 13) to check these numbers before submitting.

**Response (201) - Success:**
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

| Field | Description |
|-------|-------------|
| `transaction` | Base64-encoded unsigned `VersionedTransaction` -- sign locally and submit on-chain |
| `vaultIndex` | The vault index assigned by the factory |
| `vaultPda` | Vault PDA address |
| `vaultMintPda` | Vault mint PDA address |
| `policyId` | Mongo ID of the committed (unlinked) policy |

**Response (400) - Policy Violation:**
```json
{
  "statusCode": 400,
  "message": "Policy validation failed",
  "ok": false,
  "violations": [
    {
      "violationCode": "rule2MinAmmLiquidity",
      "message": "Mint Bonk... has $30000 AMM liquidity; policy requires at least $100000.",
      "details": { "mint": "Dez...", "observedUsd": 30000, "minUsd": 100000 }
    },
    {
      "violationCode": "rule5MaxPctPerAsset",
      "message": "Mint JUP... proposed at 80%; policy max is 60%.",
      "details": { "mint": "JUP..." }
    }
  ]
}
```

**Every applicable rule is evaluated in a single pass** — all violations are returned together so the agent can fix them in one iteration. See Section 14 for the full list of violation codes and debugging strategy.

**After receiving a successful response:** Sign the transaction with your local keypair and submit it on-chain. Use the resulting transaction signature in `POST /dtf-create`.

**Errors:** `400` Validation error, policy violations (`violations[]`), USDC blocked, insufficient balance | `404` Asset not found | `409` Policy (or vault) already exists for this vault name/symbol

---

## 4. POST `/dtf-create` - Persist Vault to DB (Metadata-Only) [Authenticated]

Called after the agent has signed and submitted the vault creation transaction on-chain. Reads the on-chain event, persists the vault's metadata to the database, and links the policy that was already committed during `/launch-dtf`. **Sends no policy fields.**

**Request Body:**
```json
{
  "transactionSignature": "5KzR8vN3xY7mW2pQ...",
  "vaultName": "Blue Chip Fund",
  "vaultSymbol": "BCF",
  "vaultType": "DTF",
  "description": "A diversified blue chip Solana fund",
  "tags": ["DeFi", "Blue Chip"],
  "logoUrl": "",
  "bannerUrl": "",
  "noRebalance": false
}
```

**Required fields:** `transactionSignature`, `vaultName`, `vaultSymbol`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transactionSignature` | string | Yes | On-chain transaction signature from the signed vault creation tx |
| `vaultName` | string | Yes | Must match the name used in `launch-dtf` |
| `vaultSymbol` | string | Yes | Must match the symbol used in `launch-dtf` |

**Optional DB fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `vaultType` | `"DTF"` / `"YIELD_DTF"` | `"DTF"` | Vault type |
| `logoUrl` | string | `""` | Logo image URL (set to `""` for agent launches) |
| `bannerUrl` | string | `""` | Banner image URL (set to `""` for agent launches) |
| `description` | string | - | Vault description |
| `noRebalance` | boolean | `false` | Disable auto-rebalancing |
| `tags` | string[] | `[]` | Searchable tags |

**Do NOT send policy fields here.** `asset_mode`, `asset_whitelist`, `min_amm_liquidity_usd`, etc. all belong in `/launch-dtf` and are ignored/rejected here.

**Response (201):**
```json
{
  "vault": [
    {
      "eventType": "VaultCreated",
      "vault": {
        "vaultName": "Blue Chip Fund",
        "vaultSymbol": "BCF",
        "vaultIndex": 42,
        "status": "active"
      }
    }
  ],
  "policyId": "665c..."
}
```

The returned `policyId` is the policy committed during `/launch-dtf`, now linked to the on-chain vault by the chain-event pipeline.

**Errors:** `400` Validation error, transaction not found, vault DB persist failed | `409` No unlinked policy found for this vault (was `/launch-dtf` called first?)

---

## 4. GET `/dtf/my-vaults` - List My Vaults [Authenticated, **DEPRECATED**]

> **Do not call this from the agent.** Use **§ 12b `GET /vaults/user`** instead. This legacy endpoint only returns vaults launched by THIS agent profile (filters on `agentCreated=true`), so it misses any vault the user created outside the agent flow (web dashboard, another agent, etc.). For every "list my vaults" / "vaults on DFM" / "what do I own" phrasing, route to `/vaults/user`.

Returns all DTF vaults created by the authenticated agent's wallet. No request body — wallet is extracted from the JWT token.

**Response (200):**
```json
{
  "vaults": [
    {
      "_id": "69e095fd...",
      "vaultName": "Solana Top 3 Alpha",
      "vaultSymbol": "SOLT3",
      "vaultType": "DTF",
      "status": "active",
      "vaultIndex": 64,
      "creatorAddress": "YourWalletHere",
      "agentCreated": true,
      "createdAt": "2026-04-16T07:55:41.319Z"
    }
  ],
  "total": 1
}
```

---

## 5. GET `/dtf/:symbol/state` - Get Vault State [Authenticated]

Returns full vault details with portfolio, user holdings, and rebalance history.

**Path params:** `symbol` - Vault symbol (e.g. `BCF`)

**Response (200):**
```json
{
  "vault": {
    "_id": "664a...",
    "vaultName": "Blue Chip Fund",
    "vaultSymbol": "BCF",
    "vaultType": "DTF",
    "status": "active",
    "vaultIndex": 42,
    "feeConfig": { "managementFeeBps": 200 },
    "underlyingAssets": [
      {
        "assetAllocation": {
          "mintAddress": "So111...",
          "name": "Wrapped SOL",
          "symbol": "SOL",
          "logoUrl": "https://..."
        },
        "pct_bps": 5000
      }
    ],
    "apy": 12.5,
    "sharePrice": 1.05,
    "nav": 105000,
    "totalTokens": 100000,
    "totalValueLocked": 105000
  },
  "portfolio": {
    "totalValueUsd": 105000,
    "assets": ["..."]
  },
  "userHoldings": ["..."],
  "rebalanceHistory": {
    "data": ["..."],
    "pagination": { "page": 1, "limit": 10, "total": 5 }
  }
}
```

---

## 5. GET `/dtf/:symbol/policy` - Get Vault Policy [Authenticated]

Returns the constitutional policy rule set for a vault, looked up directly by `vault_symbol`.

**Path params:** `symbol` - Vault symbol

**Response (200):**
```json
{
  "_id": "665c...",
  "asset_mode": "OPEN",
  "asset_whitelist": [],
  "asset_blacklist": [],
  "min_amm_liquidity_usd": 0,
  "min_24h_volume_usd": 0,
  "min_assets": 0,
  "max_assets": 0,
  "max_asset_pct": 0,
  "min_asset_pct": 0,
  "min_stablecoin_pct": 0,
  "max_rebalance_pct": 0,
  "min_rebalance_interval_hours": 4,
  "max_rebalances_per_day": 3,
  "max_rebalances_per_week": 0,
  "launch_blackout_hours": 0,
  "fee_locked": true,
  "vault_name": "Blue Chip Fund",
  "vault_symbol": "BCF",
  "agentDTFPolicy": true,
  "notes": "Happy path",
  "asset_whitelist_details": [],
  "asset_blacklist_details": []
}
```

---

## 6. GET `/dtf/:vaultSymbol/rebalance/check` - Dry-Run Rebalance Check [Authenticated]

Runs a non-blocking policy evaluation and creates/refreshes the rebalancing suggestion. **Always returns 200** — rebalance is never blocked by constitutional policy. Any rule violations are persisted as `policyReviewFlags` on the latest `RebalancingSuggestion` and surfaced in the response under `policyCheck.flagged` / `policyCheck.reviewFlags`.

**Path params:** `vaultSymbol` - Vault symbol

**Response (200) - Clean (no violations):**
```json
{
  "policyCheck": {
    "ok": true,
    "flagged": false
  },
  "suggestion": {
    "vaultId": "664a...",
    "vaultSymbol": "BCF",
    "analysisTimestamp": "2026-04-10T12:00:00.000Z",
    "status": "suggestion",
    "suggestedActions": [
      {
        "action": "reduce",
        "symbol": "SOL",
        "mintAddress": "So111...",
        "currentAllocationBps": 5500,
        "suggestedAllocationBps": 5000,
        "allocationChangeBps": -500,
        "rebalanceAmountUsd": 525,
        "reason": "Over-allocated by 5%"
      }
    ]
  }
}
```

**Response (200) - Flagged (violations present, rebalance still proceeds):**
```json
{
  "policyCheck": {
    "ok": true,
    "flagged": true,
    "reviewFlags": [
      {
        "violationCode": "rule5MaxPctPerAsset",
        "mint": "JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN",
        "message": "Mint JUP... proposed at 60%; policy max is 30%.",
        "details": { "mint": "JUP..." }
      },
      {
        "violationCode": "rule7MinStablecoinFloor",
        "message": "Proposed stablecoin share is 0%; policy requires at least 10%.",
        "details": null
      }
    ],
    "violations": [
      { "violationCode": "rule5MaxPctPerAsset", "message": "...", "details": {...} },
      { "violationCode": "rule7MinStablecoinFloor", "message": "...", "details": null }
    ]
  },
  "suggestion": { ... }
}
```

**`policyCheck` fields:**

| Field | Type | Description |
|-------|------|-------------|
| `ok` | `true` | Always true — rebalance is non-blocking. |
| `flagged` | `boolean` | `true` if any rule violation was raised (and persisted as a `policyReviewFlag` on the latest suggestion). |
| `reviewFlags` | `Array<{violationCode, mint?, message, details?}>` | Present only when `flagged: true`. Mirrors `RebalancingSuggestion.policyReviewFlags`. |
| `violations` | `RuleEvaluationResult[]` | Full list of rule violations. Empty/absent when none. |

**Violation codes:**

| Code | Description |
|------|-------------|
| `assetModeViolation` | Asset not allowed by vault's asset mode |
| `rule1WhitelistBlacklist` | Blacklisted or not whitelisted |
| `rule2MinAmmLiquidity` | Below min AMM liquidity |
| `rule3Min24hVolume` | Below min 24h volume |
| `platformMaxAssetsExceeded` | Exceeds platform max (15) |
| `rule4MinMaxAssetCount` | Asset count out of bounds |
| `rule5MaxPctPerAsset` | Single asset over max % |
| `rule6MinPctPerAssetIfHeld` | Held asset under min % |
| `rule7MinStablecoinFloor` | Stablecoin below floor |
| `rule8MaxPctRebalancedPerTx` | Rebalance % too large |
| `rule9MinTimeBetweenRebalances` | Too soon since last rebalance |
| `rule10MaxRebalancesDayWeek` | Daily/weekly cap exceeded |
| `rule11LaunchBlackout` | In launch blackout period |

**Errors:** `404` Vault or constitutional policy not found.

---

## 7. POST `/dtf/:symbol/rebalance` - Execute Rebalancing [Authenticated]

Triggers vault rebalancing. **Ownership-gated** — `signerPublicKey` must match the vault's `creatorAddress`; otherwise the request fails with `403 Forbidden` ("Only the vault creator can perform this action") *before* any rebalance work runs. The agent must NOT retry on 403 or try alternate vault lookups — surface to the user and stop. **Policy is non-blocking** — any rule violations are persisted as `policyReviewFlags` on the latest `RebalancingSuggestion` and surfaced under `policyCheck` in the response, but rebalancing still proceeds. Rebalancing is executed server-side by the admin wallet. Runs sell phase then buy phase on-chain.

**Path params:** `symbol` - Vault symbol

**Request Body:**
```json
{
  "signerPublicKey": "<public key derived from DFM_AGENT_KEYPAIR>"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `signerPublicKey` | string | Yes | Base58-encoded public key of the caller |

**Response (200) - Clean (no policy violations):**
```json
{
  "ok": true,
  "vaultId": "664a1b2c3d4e5f6a7b8c9d0e",
  "vaultSymbol": "BCF",
  "upfrontFeeSol": 0.005,
  "actualFeesSol": 0.0042,
  "policyCheck": {
    "ok": true,
    "flagged": false
  }
}
```

**Response (200) - Flagged (rebalance proceeded with policy review flags):**
```json
{
  "ok": true,
  "vaultId": "664a1b2c3d4e5f6a7b8c9d0e",
  "vaultSymbol": "BCF",
  "upfrontFeeSol": 0.005,
  "actualFeesSol": 0.0042,
  "policyCheck": {
    "ok": true,
    "flagged": true,
    "reviewFlags": [
      {
        "violationCode": "rule9MinTimeBetweenRebalances",
        "message": "Last rebalance was 2.00h ago; policy requires at least 6h.",
        "details": { "lastRebalanceAt": "2026-04-20T08:30:00.000Z", "requiredHours": 6 }
      }
    ],
    "violations": [
      { "violationCode": "rule9MinTimeBetweenRebalances", "message": "...", "details": {...} }
    ]
  }
}
```

**Response fields:**

| Field | Type | Description |
|-------|------|-------------|
| `ok` | `true` | Operation completed (rebalance was attempted). |
| `vaultId` / `vaultSymbol` | string | Vault identifiers. |
| `upfrontFeeSol` / `actualFeesSol` | number | SOL fees from the execution record. |
| `policyCheck` | object | Same shape as Section 6 — `{ ok, flagged, reviewFlags?, violations? }`. |

**Errors:** `400` rebalancing already in progress | `404` Vault not found | `500` Execution failure (e.g. insufficient SOL, on-chain error). Policy violations no longer return 400 — they appear as `policyCheck.flagged` in the 200 response.

---

## 8. POST `/dtf/:symbol/distribute-fees` - Build Distribute Fees Transactions [Authenticated]

Builds unsigned transaction(s) for distributing accrued management fees. Fees are recalculated server-side from trusted sources. Returns 1-2 base64-encoded unsigned transactions for the agent to sign locally and submit on-chain.

**Path params:** `symbol` - Vault symbol

**Request Body:**
```json
{
  "signerPublicKey": "<public key derived from DFM_AGENT_KEYPAIR>"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `signerPublicKey` | string | Yes | Base58-encoded public key of the signer (fee payer) |

**Response (200):**
```json
{
  "transactions": [
    "base64-encoded-unsigned-distribute-fees-tx..."
  ],
  "vaultIndex": 42,
  "date": "2026-04-10",
  "managementFeesAmountRaw": 1500000,
  "sharePriceRaw": 1050000
}
```

| Field | Description |
|-------|-------------|
| `transactions` | Array of base64-encoded unsigned `VersionedTransaction`s -- sign and submit **in order** |
| `vaultIndex` | Vault index |
| `date` | Distribution date |
| `managementFeesAmountRaw` | Server-calculated fee amount in raw USDC (6 decimals) |
| `sharePriceRaw` | Server-calculated share price in raw format (6 decimals) |

**After receiving the response:** Sign each transaction in order with your local keypair, submit on-chain, and wait for confirmation before submitting the next.

**Errors:** `400` Fees zero or negative, distribution failed | `404` Vault not found

---

## 9. POST `/token/revoke` - Revoke Agent Token [Authenticated]

Revokes the current agent auth token immediately. The token in the `Authorization` header is the one that gets revoked.

**Request Body:** None

**Response (200):**
```json
{
  "message": "Token revoked successfully"
}
```

**Errors:** `401` Invalid or already revoked token

---

## 10. POST `/token/refresh` - Refresh Agent Token [Public]

Issues a new agent token using the agent's `profileId`. **Existing tokens are NOT revoked** — multiple active tokens per agent are supported (each lives for 30 days then is auto-removed by the TTL index). To explicitly invalidate a specific token, use `POST /token/revoke`.

**Request Body:**
```json
{
  "profileId": "664a1b2c3d4e5f6a7b8c9d0e"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `profileId` | string | Yes | The agent profile ID to refresh token for |

**Response (200):**
```json
{
  "token": "eyJhbGciOi...",
  "expiresIn": "30d"
}
```

**Errors:** `404` No agent profile found for this profileId

---

## 11. POST `/token/refresh-by-wallet` - Refresh Agent Token by Wallet [Public]

Issues a new agent token scoped to a specific agent profile, identified by its on-chain agent wallet address. **Existing tokens are NOT revoked** — multiple active tokens per agent are supported. The issued JWT is bound to that single agent profile, even if the underlying user has multiple agents linked to their account. Use this when you don't have the `profileId` to hand — the agent wallet is derivable from `DFM_AGENT_KEYPAIR`.

**Request Body:**
```json
{
  "agentWalletAddress": "AgentSolanaWalletPublicKeyHere"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `agentWalletAddress` | string | Yes | Base58-encoded public key of the agent's Solana wallet (on-chain signer, derivable from `DFM_AGENT_KEYPAIR`) |

**Response (200):**
```json
{
  "token": "eyJhbGciOi...",
  "expiresIn": "30d"
}
```

**Errors:** `404` No agent profile found for this agent wallet address

---

## 13. GET `/market-metrics` - Bulk Market Metrics [Authenticated]

Returns Jupiter-sourced market metrics for a set of assets — the **authoritative source** for the numbers the policy engine enforces against Rules 2 & 3 (`min_amm_liquidity_usd`, `min_24h_volume_usd`). Use this before building a policy so the thresholds you choose are grounded in the same data the backend sees.

Accepts any combination of `mints`, `symbols`, and `names` as comma-separated query params. Non-mint identifiers are resolved via `asset-allocation`. USDC is blocked.

**Query params:**
```
GET /market-metrics?mints=<m1>,<m2>&symbols=SOL,JUP&names=Bonk
Authorization: Bearer <DFM_AUTH_TOKEN>
```

| Param | Type | Description |
|-------|------|-------------|
| `mints` | CSV of strings | Solana mint addresses (direct — no lookup) |
| `symbols` | CSV of strings | Token symbols (resolved via `asset-allocation`) |
| `names` | CSV of strings | Token names (resolved via `asset-allocation`) |

At least one identifier must be provided. Identifiers that fail to resolve are returned in `unresolved[]` — a single bad identifier does not fail the whole request.

**Response (200):**
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
    },
    {
      "mintAddress": "DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263",
      "symbol": "Bonk",
      "name": "Bonk",
      "liquidity_usd": null,
      "volume_24h_usd": null,
      "price_usd": null,
      "holder_count": null,
      "policyRelevant": { "liquidity_usd": null, "volume_24h_usd": null }
    }
  ],
  "unresolved": [
    { "identifier": "FOO", "reason": "Asset not found in asset-allocation" }
  ]
}
```

| Field | Description |
|-------|-------------|
| `liquidity_usd` | Jupiter AMM liquidity in USD. Compared to `policy.min_amm_liquidity_usd` (Rule 2). |
| `volume_24h_usd` | Sum of `stats24h.buyVolume + stats24h.sellVolume` from Jupiter. Compared to `policy.min_24h_volume_usd` (Rule 3). |
| `price_usd` | Current USD price (informational). |
| `holder_count` | Jupiter holder count (informational). |
| `policyRelevant` | Subset of the above that the policy engine actually reads — use for policy threshold calibration. |

**`null` values:** indicate a transient Jupiter fetch miss for that mint. Responses are cached per-mint in Redis with a configurable TTL, so retrying the call usually returns live data on the next request.

**Errors:** `400` No identifiers provided | `401` Invalid or missing JWT

---

## 12a. GET `/vaults/featured/list` - List Featured Vaults [Authenticated]

Returns a paginated list of featured vaults across the platform. Filterable by vault type, with optional TVL data hydration.

**Query params:**

```
GET /vaults/featured/list?page=1&limit=10&vaultType=dtf&includeTvl=true
Authorization: Bearer <DFM_AUTH_TOKEN>
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | number | `1` | Page number (1-indexed) |
| `limit` | number | `10` | Items per page |
| `vaultType` | `"dtf"` / `"yield_dtf"` | `"dtf"` | Filter by vault type |
| `includeTvl` | boolean | `true` | Hydrate response with `totalValueLocked`, `nav`, `sharePrice`. Agents must always send `true`. |

**Response (200):**
```json
{
  "data": [
    {
      "_id": "6983c090ff8f3d8a0f99626f",
      "vaultName": "Agentonomy",
      "vaultSymbol": "AGE-DTF",
      "vaultAddress": "FdxUo6tGcpAAaDcB9771X4dywjFVJG8o972xSKoKtp8B",
      "description": "AI agent tokens basket designed for narrative exposure...",
      "vaultIndex": 53,
      "tags": ["AI-agents", "agent-economy", "high-beta"],
      "nav": "0",
      "totalSupply": "0",
      "feeConfig": { "managementFeeBps": 150 },
      "underlyingAssets": [
        {
          "assetAllocation": {
            "_id": "68ca86b973a7083867c6d386",
            "name": "Virtual Protocol",
            "symbol": "VIRTUAL",
            "logoUrl": "https://ipfs.io/ipfs/bafkre...",
            "id": "68ca86b973a7083867c6d386"
          },
          "pct_bps": 4000,
          "_id": "6983c090ff8f3d8a0f996270",
          "id": "6983c090ff8f3d8a0f996270"
        }
      ],
      "creator": {
        "_id": "68e6e163bcfe1ea703b7fa28",
        "name": "Sample Creator",
        "email": "creator@example.com",
        "avatar": "https://s3.blocsys.com/.../avatar.png",
        "walletAddress": "YourWalletPublicKeyHere",
        "twitter_username": "samplecreator",
        "useTwitterImage": false,
        "useAvatarImage": true,
        "socialLinks": [{}],
        "id": "68e6e163bcfe1ea703b7fa28"
      },
      "category": {
        "_id": "691ace106d9f63ddb538f633",
        "name": "Manual"
      },
      "totalValueLocked": 14.808095,
      "sharePrice": 1.3484338807683582,
      "daoconfig": null,
      "vaultApy": null,
      "performance7d": 13.01
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 2,
    "totalPages": 1,
    "hasNext": false,
    "hasPrev": false
  }
}
```

**Vault item fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_id` | string | Mongo document id |
| `vaultName` | string | Display name |
| `vaultSymbol` | string | Symbol (e.g. `AGE-DTF`) |
| `vaultAddress` | string | On-chain vault PDA address |
| `description` | string | Strategy description |
| `vaultIndex` | number | Factory-assigned vault index |
| `tags` | string[] | Searchable tags |
| `nav` | string | Net asset value (decimal-as-string; `"0"` until first deposit) |
| `totalSupply` | string | Total share supply (decimal-as-string) |
| `feeConfig.managementFeeBps` | number | Management fee in basis points |
| `underlyingAssets[]` | array | Asset allocations — each item has `assetAllocation` (`name`, `symbol`, `logoUrl`, `_id`) and `pct_bps` |
| `creator` | object | Creator profile — `name`, `email`, `avatar`, `walletAddress`, `twitter_username`, etc. |
| `category` | object | `{ _id, name }` (e.g. `"Manual"`) |
| `totalValueLocked` | number \| null | TVL in USD. Present when `includeTvl=true`. |
| `sharePrice` | number \| null | Share price in USD. Present when `includeTvl=true`. |
| `daoconfig` | object \| null | DAO configuration; `null` for non-DAO vaults |
| `vaultApy` | number \| null | APY (`null` until enough history) |
| `performance7d` | number \| null | 7-day performance % |

**Pagination fields:** `page`, `limit`, `total`, `totalPages`, `hasNext`, `hasPrev`.

**Note:** `nav` and `totalSupply` are returned as strings (decimal-safe representation). Parse with `Number()` or a big-number library before arithmetic. `vaultApy` and `performance7d` may be `null` for new vaults with insufficient history.

**Errors:** `400` Invalid query params | `401` Invalid or missing JWT

---

## 12b. GET `/vaults/user` - List User Vaults [Authenticated]

Returns a paginated list of vaults owned by the authenticated user (resolved from the JWT). Filterable by vault type, with optional TVL data hydration. Use this in preference to `/dtf/my-vaults` when you need pagination or `vaultType` filtering.

**Query params:**

```
GET /vaults/user?page=1&limit=10&vaultType=dtf&includeTvl=true
Authorization: Bearer <DFM_AUTH_TOKEN>
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | number | `1` | Page number (1-indexed) |
| `limit` | number | `10` | Items per page |
| `vaultType` | `"dtf"` / `"yield_dtf"` | `"dtf"` | Filter by vault type |
| `includeTvl` | boolean | `true` | Hydrate `totalValueLocked` / `sharePrice`. Agents must always send `true`. |

**Response (200):** Same shape as `/vaults/featured/list` — `{ data: Vault[], pagination: { page, limit, total, totalPages, hasNext, hasPrev } }`. See the field table above for every property the agent should read.

**Errors:** `400` Invalid query params | `401` Invalid or missing JWT

---

## 12c. POST `/vaults/:symbol/update-assets-tx` - Build Update-Assets Transaction (Policy-Gated, On-Chain) [Authenticated]

**Two-step gated build, server-side:**

1. **Policy validation (always first).** Backend runs the vault's constitutional-policy `UPDATE_UNDERLYING` check against the proposed basket — same evaluator as `POST /policy/:vaultId/check`. If any rule violates, returns `400` with `{ ok: false, violationCode, message, violations[] }` and **NO transaction is built**. Fix the basket and retry.
2. **Transaction build (only on a clean policy pass).** Builds an unsigned `updateUnderlyingAssets(vaultIndex, underlyingAssets)` Solana `VersionedTransaction` for client-side signing. The agent signs locally with `DFM_AGENT_KEYPAIR` and submits on-chain.

The endpoint resolves each identifier (`mintAddress` / `symbol` / `name`) against the asset-allocation collection — agents can pass `{ symbol: "SOL", mintBps: 5000 }` instead of looking up mint addresses themselves.

**Path Params:**

| Param | Description |
|-------|-------------|
| `symbol` | Vault symbol (e.g. `ALPHA`). Case-insensitive match against `VaultFactory.vaultSymbol`. Get from `/vaults/user` (`data[].vaultSymbol`) or `/vaults/featured/list`. |

**Request Body:**
```json
{
  "signerPublicKey": "<public key derived from DFM_AGENT_KEYPAIR>",
  "underlyingAssets": [
    { "symbol": "SOL", "mintBps": 5000 },
    { "name": "USD Coin", "mintBps": 3000 },
    { "mintAddress": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v", "mintBps": 2000 }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `signerPublicKey` | string | Yes | Base58 public key of the vault admin / fee payer (derived from `DFM_AGENT_KEYPAIR`). Must match the vault's on-chain `admin` or the program rejects the tx. |
| `underlyingAssets[].mintAddress` | string | One-of | Solana mint address (direct, no DB lookup) |
| `underlyingAssets[].symbol` | string | One-of | Asset symbol (e.g. `SOL`, `JUP`). Resolved against asset-allocation. |
| `underlyingAssets[].name` | string | One-of | Asset name (e.g. `"Wrapped SOL"`). Resolved against asset-allocation. |
| `underlyingAssets[].mintBps` | number (0-10000) | Yes | Allocation in basis points. All entries must sum to **10000**. |

**Response (201) - Clean policy pass:**
```json
{
  "transaction": "base64-encoded-unsigned-versioned-transaction...",
  "vaultIndex": 42,
  "vaultPda": "7Xk...def"
}
```

| Field | Description |
|-------|-------------|
| `transaction` | Base64-encoded unsigned `VersionedTransaction` — sign locally and submit on-chain |
| `vaultIndex` | The vault's on-chain index (sanity-check) |
| `vaultPda` | The derived vault PDA address |

**Response (400) - Policy violation (no transaction built):**
```json
{
  "statusCode": 400,
  "ok": false,
  "violationCode": "rule5MaxPctPerAsset",
  "message": "Mint JUP... proposed at 60%; policy max is 30%.",
  "violations": [
    {
      "violationCode": "rule5MaxPctPerAsset",
      "message": "Mint JUP... proposed at 60%; policy max is 30%.",
      "details": { "mint": "JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN" }
    },
    {
      "violationCode": "rule7MinStablecoinFloor",
      "message": "Proposed stablecoin share is 0%; policy requires at least 10%.",
      "details": null
    }
  ]
}
```

The `violations[]` array carries every violated rule in one pass — fix all of them before retrying. Same violation codes as Section 6 (`/dtf/:vaultSymbol/rebalance/check`).

**After a successful response:** sign the transaction with `DFM_AGENT_KEYPAIR` and submit on-chain. Use the existing `signAndSend` helper from the Authentication section. After on-chain confirmation, call `PATCH /vaults/:symbol/underlying-assets-by-mint` (Section 12d) to sync the DB record, then trigger a rebalance via `POST /dtf/:symbol/rebalance` (Section 7) — rebalancing is required after a basket change so actual holdings match the new allocations. Ask the user to confirm execution before calling `/rebalance`.

**Errors:** `400` policy violation, validation error, asset not found in asset-allocation, mintBps don't sum to 10000 | `401` Invalid or missing JWT | `403` Only the vault creator can perform this action | `404` Vault not found for the given symbol | `404` No constitutional policy found for this vault (vault was not launched via `/launch-dtf`)

---

## 12d. PATCH `/vaults/:symbol/underlying-assets-by-mint` - Persist Updated Basket to DB [Authenticated]

After the on-chain `updateUnderlyingAssets` tx confirms, call this endpoint to sync the DB record. **Metadata-only — no on-chain interaction.** The DTO uses `pct_bps` (DB field name), NOT `mintBps`.

The chain-event pipeline will eventually sync the DB on its own as on-chain events are processed, so this endpoint is needed only when synchronous DB consistency is required (e.g. agent immediately reads `/vaults/user` after the on-chain submit and expects the new basket to appear).

**Path Params:**

| Param | Description |
|-------|-------------|
| `symbol` | Vault symbol (e.g. `ALPHA`) — same symbol used in Section 12c |

**Request Body:**
```json
{
  "underlyingAssets": [
    { "mintAddress": "So11111111111111111111111111111111111111112", "pct_bps": 5000 },
    { "mintAddress": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v", "pct_bps": 5000 }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `underlyingAssets[].mintAddress` | string | Yes | Solana mint address (no symbol/name resolution here — pass mint directly) |
| `underlyingAssets[].pct_bps` | number (0-10000) | Yes | Allocation in basis points. Must sum to **10000**. |

**Response (200):** Returns the updated `VaultFactory` document with the new `underlyingAssets` array. The agent's vault-list caches (`agent:vaults:*`) are flushed automatically as part of this request — the next `/vaults/user` or `/vaults/featured/list` call reflects the new basket immediately.

**Errors:** `400` invalid mint / pct_bps don't sum to 10000 | `401` Invalid or missing JWT | `403` Only the vault creator can perform this action | `404` Vault not found for the given symbol

---

## 14. POST `/policy/dry-run` - Simulate Policy Evaluation [Authenticated]

Runs a proposed basket + policy against all pre-launch policy rules (asset mode, whitelist/blacklist, min liquidity, min 24h volume, asset count bounds, per-asset allocation bounds) and returns **every** violation in one pass. No DB write, no on-chain cost — use in a tight loop to iterate on basket/policy combinations before calling `/launch-dtf`.

Returns the same evaluation the backend will run server-side during `/launch-dtf`. A clean dry-run means the subsequent `/launch-dtf` with the same basket + policy will pass.

**Request Body:**
```json
{
  "underlyingAssets": [
    { "symbol": "SOL",  "pct_bps": 4000 },
    { "symbol": "JUP",  "pct_bps": 3000 },
    { "name":   "Bonk", "pct_bps": 3000 }
  ],
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
    "fee_locked": true
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `underlyingAssets` | array | Yes | Proposed basket. Each item has `mintAddress` or `symbol` or `name`, plus `pct_bps`. Must sum to 10000. |
| `policy` | object | Yes | Same shape as the `policy` sub-object in `/launch-dtf`. |

**Response (200) - Clean:**
```json
{
  "ok": true,
  "policyCheck": { "ok": true, "violations": [] }
}
```

**Response (200) - Violations (note: 200, not 400 — dry-run is evaluation, not rejection):**
```json
{
  "ok": false,
  "policyCheck": {
    "ok": false,
    "violationCode": "rule2MinAmmLiquidity",
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

**Rules evaluated at pre-launch:**

| Code | Rule | Fix |
|------|------|-----|
| `assetModeViolation` | Policy `asset_mode` is invalid | Set to one of `OPEN`, `WHITELIST_ONLY`, `OPEN_BLACKLIST`, `WHITELIST_BLACKLIST`. |
| `rule1WhitelistBlacklist` | Asset blocked by whitelist/blacklist | Add to `asset_whitelist` or remove from `asset_blacklist`; or swap the asset. |
| `rule2MinAmmLiquidity` | Asset liquidity < threshold | Lower `min_amm_liquidity_usd` OR swap asset. Check `/market-metrics` for the real number. |
| `rule3Min24hVolume` | Asset 24h volume < threshold | Lower `min_24h_volume_usd` OR swap asset. |
| `platformMaxAssetsExceeded` | Over 15 distinct assets | Platform hard cap — reduce basket size. |
| `rule4MinMaxAssetCount` | Count outside `min_assets` / `max_assets` | Adjust bounds OR basket size. |
| `rule5MaxPctPerAsset` | A `pct_bps` > `max_asset_pct` | Reduce the allocation OR raise `max_asset_pct`. |
| `rule6MinPctPerAssetIfHeld` | A held asset's `pct_bps` < `min_asset_pct` | Raise the allocation OR drop the asset. |

Rules 7–11 (stablecoin floor, rebalance limits, launch blackout) apply post-launch and are NOT evaluated here.

**`marketMetricsUnavailable`:** if `min_amm_liquidity_usd > 0` or `min_24h_volume_usd > 0` and Jupiter is unreachable, strict mode fails Rules 2/3 closed. Surface the issue to the user and retry; the Redis cache warms quickly.

**Errors:** `400` Validation error, bps not summing to 10000, USDC in basket | `401` Invalid or missing JWT | `404` Asset not found in asset-allocation

---

## 15. Capital Flows — Deposit & Redeem (overview)

The agent namespace exposes two capital-flow pipelines:

- **Deposit (2 steps):** `/vaults/:symbol/deposit-tx` builds an unsigned `deposit` tx; the agent signs locally + submits; `/deposit-transaction` then runs the backend swap fan-out (vault USDC → underlyings via Jupiter) and persists the deposit records.
- **Redeem (5 steps):** queue-serialised. `/redeem/request-ticket` queues, `/redeem/execute/:ticketId` runs the backend swap fan-in (underlyings → vault USDC), `/vaults/:symbol/redeem-tx` builds an unsigned `finalizeRedeem` tx, the agent signs + submits, then `/redeem-transaction` persists the records and (with `ticketId`) auto-confirms the queue ticket.

Both flows resolve `vaultSymbol → vaultIndex` server-side, bake a fresh KMS-signed share price into the on-chain tx, and attribute persisted records to the agent identity (`agentProfile` + `agentAddress`) via the JWT. **Backend never sees a private key** — the agent signs locally with `DFM_AGENT_KEYPAIR`.

| Sub-section | Endpoint | Step |
|---|---|---|
| **15a** | **`POST /vaults/:symbol/check-min-deposit`** | **Deposit pre-flight gate (MANDATORY)** |
| **15b** | **`POST /check-min-redeem`** | **Redeem pre-flight gate (MANDATORY)** |
| 15c | `POST /vaults/:symbol/deposit-tx` | Deposit Step 1 — build unsigned tx |
| 15d | `POST /deposit-transaction` | Deposit Step 2 — swap + record |
| 15e | `POST /redeem/request-ticket` | Redeem Step 1 — queue entry |
| 15f | `POST /redeem/execute/:ticketId` | Redeem Step 2 — backend swap fan-in |
| 15g | `POST /vaults/:symbol/redeem-tx` | Redeem Step 3 — build unsigned `finalizeRedeem` |
| 15h | `POST /redeem-transaction` | Redeem Step 4 — record + auto-confirm ticket |

**Mandatory gates (15a + 15b):** every deposit run MUST start with 15a; every redeem run MUST start with 15b. Skipping the gate lets through too-small amounts that cause confusing on-chain failures (post-fee dust, zero-share mints, fractional swaps Jupiter rejects). The gates are agent-side enforced — neither `/deposit-tx` nor `/redeem/request-ticket` re-checks the threshold themselves, so the agent is the only thing standing between the user and a bad amount. **15a returns 400 on fail (HTTP-status-driven). 15b returns 200 with `isValid: false` on fail (read the boolean).**

**Polling endpoint (not cloned in the agent namespace):** clients poll `GET /api/v1/tx-event-management/redeem/ticket-status/:ticketId` to detect when the queue ticket is ready. The agent module did not get its own clone — use the v1 path with the agent JWT.

---

## 15a. POST `/vaults/:symbol/check-min-deposit` - Validate Minimum Deposit [Authenticated]

**MANDATORY pre-flight gate for the deposit flow** — every `/deposit-tx` call must be preceded by a successful `/check-min-deposit`. Pure validation — no on-chain action, no DB write. Required threshold = the vault's underlying-asset count (one USDC per asset). When the vault has no allocations, falls back to `MINI_DEPOSIT` env (default `5`). Throws `400` if `minDeposit` is below threshold; otherwise `200 { isValid: true, message }`.

**On 400, the agent must abort the deposit flow** — surface the threshold message to the user and ask for a larger amount. Do NOT call `/deposit-tx`.

**Path Params:** `symbol` — vault symbol (case-insensitive).

**Request Body:**
```json
{ "minDeposit": 5 }
```

**Response (200):**
```json
{ "isValid": true, "message": "Minimum deposit requirement of $5 USDC met" }
```

**Errors:** `400` `Minimum deposit should be at least $<n> USDC` | `401` Invalid or missing JWT | `403` Vault is owned by another user | `404` Vault not found

---

## 15b. POST `/check-min-redeem` - Validate Minimum Redeem [Authenticated]

**MANDATORY pre-flight gate for the redeem flow** — every `/redeem/request-ticket` call must be preceded by a successful `/check-min-redeem`. Pure validation against the global `MINI_REDEEM` env (default `4`). **Behavioral difference from `check-min-deposit`:** always returns `200` with an `isValid` boolean — does **not** throw 400 on fail. **Read `isValid` rather than relying on HTTP status** — and abort the redeem flow when `isValid` is `false`. No vault parameter (threshold is global).

**On `isValid: false`, the agent must abort the redeem flow** — surface the threshold message to the user and ask for a larger amount. Do NOT call `/redeem/request-ticket`.

**Request Body:**
```json
{ "minRedeem": 4 }
```

**Response (200):**
```json
{ "isValid": true, "message": "Minimum redeem requirement of 4 shares met" }
```

**Errors:** `401` Invalid or missing JWT

---

## 15c. POST `/vaults/:symbol/deposit-tx` - Build Deposit Transaction (Step 1 of 2) [Authenticated]

Resolves the vault by symbol → `vaultIndex`, derives PDAs (`factory_v2`, `vault`, `vault_mint`, `vault_stablecoin_account`), conditionally adds 4 ATA-creation ixs (depositor USDC + vault-token, fee-recipient USDC, admin USDC), requests a fresh KMS-signed share price via `PriceSignerService.signSharePrice(vaultIndex)`, and emits the Ed25519 verify ix + the on-chain `deposit(vaultIndex, amount, etfSharePrice, priceTimestamp)` ix.

The depositor signs locally with their wallet and submits on-chain — **the backend never sees a private key**. The response carries the same field names (`vaultIndex`, `etfSharePriceRaw`, `amountInRaw`) the merged Step 2 endpoint expects, so the client can forward the response straight through (after attaching the resulting `transactionSignature`).

**Path Params:** `symbol` — vault symbol (case-insensitive).

**Request Body:**
```json
{
  "userPublicKey": "7xKp9mN3vR5tQ8wY2zA4bC6dF1eG3hI5jK7lM9oP1qS",
  "depositAmount": 5
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `userPublicKey` | string | Yes | Base58 Solana address of the depositor (signer + fee payer of the built tx). For agent-driven deposits this is typically the agent wallet, but any wallet may be supplied. |
| `depositAmount` | number | Yes | USDC amount in UI (decimal) units, e.g. `5` means 5 USDC. Must be a positive number. |

**Response (201):**
```json
{
  "transaction": "<base64 unsigned VersionedTransaction>",
  "vaultIndex": 0,
  "vaultPda": "...",
  "vaultMintPda": "...",
  "etfSharePriceRaw": "1053000",
  "priceTimestamp": 1730000000,
  "amountInRaw": "5000000"
}
```

| Field | Description |
|-------|-------------|
| `transaction` | Base64-encoded unsigned v0 `VersionedTransaction`. The depositor wallet signs locally and submits on-chain. |
| `vaultIndex` | u32 — forward into Step 2 unchanged. |
| `etfSharePriceRaw` | Signed share price raw u64 (6 decimals). Forward into Step 2 unchanged. |
| `priceTimestamp` | Unix seconds — already baked into the tx; included for client-side sanity checks. |
| `amountInRaw` | Gross USDC raw u64 (6 decimals). Forward into Step 2 unchanged — the swap step clamps to vault balance, so passing the gross is safe. |

**Errors:** `400` invalid `userPublicKey` / non-positive `depositAmount` / vault has no on-chain `vaultIndex` / signed-price PDA mismatch / factory account fetch failed | `401` Invalid or missing JWT | `404` Vault not found

---

## 15d. POST `/deposit-transaction` - Process Deposit — Swap then Record (Step 2 of 2) [Authenticated]

Single call that runs two sequential operations:

**(a) `agentSwap`** — fans the vault's USDC into its underlying assets via Jupiter using the same `vaultIndex` / `amountInRaw` / `etfSharePriceRaw` from Step 1. Transfers vault USDC to the admin wallet, executes per-asset Jupiter swaps according to the vault's `underlyingAssets` bps weights, and returns USDC back to the vault for any failed swap.

**(b) `depositTransaction`** — fetches the on-chain `VaultDeposited` transaction by signature, parses program logs (emoji-prefixed `Program log:` lines), and writes `DepositTransaction` + `UserVaultPosition` (keyed on `agentProfile`) + `DepositRecord` + History row. Forwards the successful swap signatures from (a) as `signatureArray`.

If (a) produces no successful swaps, (b) is still attempted (records the deposit with an empty `signatureArray`). Both sub-step failures are surfaced **inside the 200 response** rather than thrown — see "failure modes" below.

**Request Body:**
```json
{
  "transactionSignature": "5JYQDd2…",
  "vaultIndex": 0,
  "etfSharePriceRaw": "1053000",
  "amountInRaw": "5000000",
  "slippage": 200
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transactionSignature` | string | Yes | On-chain signature of the confirmed `deposit` tx from Step 1. |
| `vaultIndex` | number | Yes | u32 from Step 1's response. |
| `etfSharePriceRaw` | string | Yes | Raw u64 (6 decimals) from Step 1's response. |
| `amountInRaw` | string | Yes | Raw u64 (6 decimals) from Step 1's response. |
| `slippage` | number | No | Basis points, range `50-500` (0.5%-5%). Falls back to `SLIPPAGE` env or 200 bps default. |

**Response (200):** `{ swap, deposit }` — both sub-step results returned together.

```json
{
  "swap": {
    "vaultIndex": 0,
    "amountRequested": "5000000",
    "amountUsed": "5000000",
    "vaultUsdcBalance": "...",
    "etfSharePriceRaw": "1053000",
    "swaps": [
      { "assetMint": "...", "usdcPortion": "...", "transferSig": "...", "swapSig": "..." }
    ]
  },
  "deposit": {
    "events": [
      {
        "eventType": "VaultDeposited",
        "vaultName": "ALPHA",
        "vaultSymbol": "ALPHA",
        "vaultIndex": 0,
        "user": "<depositor>",
        "amount": "5000000",
        "entryFee": "...",
        "netAmount": "...",
        "vaultTokensMinted": "...",
        "newTotalAssets": "...",
        "newTotalSupply": "...",
        "updatedVaultDepositId": "<ObjectId>"
      }
    ]
  }
}
```

**Failure modes (surfaced inside 200):**
- `swap.failedSwapsInfo` — present when one or more per-asset swaps failed. Includes `count`, `totalFailedUSDC`, `returnTransferSignature` (if the backend successfully returned USDC to the vault), and a manual-recovery note if the return tx itself failed.
- `deposit.events[].vaultDepositUpdateError` — present when the deposit-record save threw (logged + surfaced, but the swap result is still returned).

**Errors:** `400` Transaction not found / no deposit logs / agent profile not found for address / insufficient admin SOL (≥ 0.1 needed for tx fees) / no underlying assets configured | `401` Invalid or missing JWT

---

## 15e. POST `/redeem/request-ticket` - Request Redeem Ticket [Authenticated]

Queues an agent-bound redeem request. Resolves `vaultSymbol → vaultIndex` server-side, computes required USDC, and creates a ticket on the redeem queue. Calculates the share price from on-chain data when `etfSharePriceRaw` is omitted.

The redeem queue serialises vault liquidations to prevent races between concurrent redeemers. The frontend should poll **v1 `GET /api/v1/tx-event-management/redeem/ticket-status/:ticketId`** to detect when the ticket is ready (the agent namespace has no `/ticket-status` clone).

**Wallet binding:** the canonical agent wallet is read from the JWT (`agentPayload.agentWalletAddress`). A body-level `agentWallet` is accepted only as a fallback if the JWT field is missing.

**Request Body:**
```json
{
  "vaultSymbol": "ALPHA",
  "vaultTokenAmount": "2005023",
  "etfSharePriceRaw": "997692",
  "slippage": 200
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `vaultSymbol` | string | Yes | Case-insensitive vault symbol. |
| `vaultTokenAmount` | string | Yes | Vault token amount to redeem, **raw u64 as string** (6 decimals — multiply UI tokens by 1e6). |
| `etfSharePriceRaw` | string | No | Raw u64 (6 decimals). Backend reads on-chain when omitted/`"0"`. |
| `slippage` | number | No | Basis points, range `50-500`. |
| `agentWallet` | string | No | Fallback only — JWT canonical. |

**Response (200):**
```json
{
  "ticketId": "ticket_1704067200000_abc123xyz",
  "position": 1,
  "estimatedWaitSeconds": 0,
  "isReady": true
}
```

**Errors:** `400` Invalid request / vault has no on-chain `vaultIndex` | `401` Invalid or missing JWT | `404` Vault not found

---

## 15f. POST `/redeem/execute/:ticketId` - Execute Redeem Ticket [Authenticated]

Should be called when `/ticket-status` reports `isReady=true`. Marks the ticket as `PROCESSING` (locks the queue position), runs the redeem fan-in via `redeemAgentSwapAdmin` using the parameters captured at ticket creation (vault, amount, price), and returns swap results.

**Time bound:** the ticket must be executed within ~3 minutes of becoming ready or it expires.

**On execution failure**, the ticket is **auto-cancelled** by the backend (queue slot released) and the original error is rethrown — the client does **not** need to call `/cancel`.

**Path Params:** `ticketId` — id returned by `/redeem/request-ticket`.

**Request Body:**
```json
{ "slippage": 200 }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `slippage` | number | No | Basis points, range `50-500`. Precedence: `dto.slippage ?? ticket.slippage ?? 200`. |

**Response (200):**
```json
{
  "vaultIndex": 0,
  "vaultTokenAmount": "2005023",
  "etfSharePriceRaw": "997692",
  "swaps": [
    { "assetMint": "...", "amountIn": "...", "amountOut": "...", "swapSig": "..." }
  ],
  "ticketId": "ticket_1704067200000_abc123xyz",
  "ticketStatus": "PROCESSING",
  "message": "Swaps completed. Please sign and submit the on-chain transaction, then call /confirm/:ticketId"
}
```

| Field | Description |
|-------|-------------|
| `swaps[]` | Per-asset swap results (mint + amounts + signature). Collect successful `swapSig`s for `signatureArray` in Step 4. |
| `ticketStatus` | Always `PROCESSING` on a successful 200 — the ticket is locked until `/redeem-transaction` records and auto-confirms it. |
| `message` | Human-readable instruction for the next step. |

**Errors:** `400` Failed to process ticket (already processing/expired/missing) / Redeem swap failure (auto-cancelled) | `401` Invalid or missing JWT

---

## 15g. POST `/vaults/:symbol/redeem-tx` - Build Redeem Transaction (Finalize) [Authenticated]

Builds an unsigned `finalizeRedeem` `VersionedTransaction` for client-side signing. Mirrors the deposit-tx pattern: backend resolves all on-chain accounts and bakes a fresh KMS-signed share price into the tx; the agent signs locally and submits.

Intended to follow the swap fan-in produced by `/redeem/execute/:ticketId` — by then the vault holds enough USDC to settle the redemption.

**Wallet binding:** the redeemer is the agent wallet, read from the JWT. Body-level `agentWallet` is fallback only.

**Server-side build steps:**
1. Resolve vault by symbol → `vaultIndex`.
2. Derive `factory_v2`, `vault`, `vault_mint`, `vault_stablecoin_account` PDAs.
3. Fetch the on-chain factory account for `admin` + `feeRecipient`; read the vault account for its admin (falls back to factory admin).
4. Conditionally push 4 ATA-creation ixs: agent vault-token, agent USDC, fee-recipient USDC, vault-admin USDC.
5. Call `priceSignerService.signSharePrice(vaultIndex)` — refuses to build (`400`) if the signed price's `vaultPubkey` ≠ derived vault PDA.
6. Emit Ed25519 verify ix + Anchor `finalizeRedeem(vaultIndex, vaultTokenAmount, etfSharePrice, priceTimestamp)` ix (no Jupiter program, unlike deposit).
7. Compile a v0 message with the agent as fee payer; return base64.

**Path Params:** `symbol` — vault symbol.

**Request Body:**
```json
{ "vaultTokenAmount": 1.5 }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `vaultTokenAmount` | number | Yes | Vault-token amount to redeem in **UI units** (e.g. `1.5` = 1.5 vault tokens). Backend converts to raw u64 (6 decimals). Must be positive. |
| `agentWallet` | string | No | Fallback only — JWT canonical. |

**Response (201):**
```json
{
  "transaction": "<base64 unsigned VersionedTransaction>",
  "vaultIndex": 0,
  "vaultPda": "...",
  "vaultMintPda": "...",
  "etfSharePriceRaw": "1053000",
  "priceTimestamp": 1730000000,
  "vaultTokenAmountRaw": "1500000"
}
```

Forward `transactionSignature` (after on-chain submit), `vaultIndex`, `etfSharePriceRaw`, and the swap signatures from Step 15f into Step 15h.

**Errors:** `400` invalid `vaultTokenAmount` / invalid agent wallet / vault has no `vaultIndex` / signed-price PDA mismatch / factory fetch failed | `401` Invalid or missing JWT | `404` Vault not found

---

## 15h. POST `/redeem-transaction` - Record Redeem + Auto-Confirm Ticket [Authenticated]

Records an agent redeem after the client has signed and submitted the on-chain `finalizeRedeem` tx built by `/vaults/:symbol/redeem-tx`. Mirrors the v1 redeem-transaction route but binds **all** persisted records to the agent identity (`agentProfile` + `agentAddress`) resolved from the JWT instead of a user `Profile`.

**Identity guard:** the legacy user-side Profile lookup is **not** performed. `agentProfileId` and `agentAddress` come off the JWT (`agentPayload`). The service hard-fails with `400` if `performedByProfileId` is missing — this endpoint cannot be called without a valid agent JWT.

**What gets persisted:**
1. **`RedeemTransaction`** (legacy collection) — written with `agentProfile` + `agentAddress` (no user fields).
2. **`UserVaultPosition`** — looked up by `{ agentProfile, vaultFactory }`, decremented (`currentShares -= sharesRedeemed`), cost basis reduced proportionally, and **closed** (`status: "closed"`) when shares hit 0. Throws `400` if no position exists or shares insufficient.
3. **`RedeemRecord`** — fresh record under that position with agent fields, `sharesRedeemed`, `usdcReceived` (net), `pricePerShare`, fees, `metadata: { signatureArray, slippage }`.
4. **`History`** — `redeem_completed` row attributed via the trailing `agentProfile` / `agentAddress` params.

**Auto-confirm queue ticket (optional):** when `ticketId` is supplied, the controller calls `confirmRedeemTicket(ticketId, transactionSignature)` immediately after the four writes. **Non-fatal** — failures are logged at `warn` and surfaced as `ticketConfirm: { ok: false, message }` but do not abort the response. When `ticketId` is omitted, `ticketConfirm` is `null`.

**Request Body:**
```json
{
  "transactionSignature": "5JYQDd2…",
  "vaultIndex": 0,
  "etfSharePriceRaw": "997692",
  "signatureArray": ["redeemSwapSig1"],
  "slippage": 200,
  "ticketId": "ticket_1704067200000_abc123xyz"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transactionSignature` | string | Yes | Signature of the confirmed on-chain `finalizeRedeem` tx. |
| `vaultAddress` | string | No | Vault-PDA hint — improves vault association when log parse is ambiguous. |
| `vaultIndex` | number | No | vaultIndex hint — overrides log-parsed value. |
| `etfSharePriceRaw` | string | No | Forwarded to position update for P&L tracking. |
| `signatureArray` | string[] | No | Redeem-side swap signatures captured at `/redeem/execute/:ticketId`. Recorded on the History row. |
| `slippage` | number | No | Basis points, range `50-500`. |
| `ticketId` | string | No | Redeem queue ticket id. When present, the backend auto-confirms the ticket after the redeem is recorded. Failures are non-fatal. |

**Response (200):**
```json
{
  "events": [
    {
      "eventType": "VaultRedeemed",
      "vaultName": "ALPHA",
      "vaultSymbol": "ALPHA",
      "vaultIndex": 0,
      "user": "<agent wallet>",
      "vaultTokensRedeemed": "1500000",
      "exitFee": "1500",
      "managementFee": "0",
      "totalFees": "1500",
      "netStablecoinAmount": "1497000",
      "newTotalAssets": "...",
      "newTotalSupply": "...",
      "updatedVaultDepositId": "<ObjectId>"
    }
  ],
  "ticketConfirm": {
    "ok": true,
    "message": "Ticket ticket_… completed successfully. Transaction: 5JYQDd2…"
  }
}
```

| Field | Description |
|-------|-------------|
| `events[].user` | On-chain `user` from program logs — for the agent flow this is the agent wallet. |
| `events[].vaultTokensRedeemed` | Vault-token amount burned (raw u64, 6 decimals). |
| `events[].netStablecoinAmount` | USDC paid out to the agent wallet after fees (raw u64, 6 decimals). |
| `events[].updatedVaultDepositId` | ObjectId of the updated VaultDeposit — useful for client-side correlation. |
| `ticketConfirm` | `null` when no `ticketId` was supplied. `{ ok: true, message }` on success. `{ ok: false, message }` when confirm threw — redeem still recorded; client may re-call v1 `/redeem/confirm/:ticketId` manually. |

**Errors:** `400` Transaction not found / no redeem logs / `redeemAgentTransaction requires performedByProfileId` / Invalid `netStablecoinAmount` or `vaultTokensToRedeem` / No position found for agent in this vault / Insufficient shares | `401` Invalid or missing JWT

**Side effects:** clears `agent:vaults:*` Redis keys.

---

## 15i. GET `/vaults/:symbol/shares` - Get Vault Share Balance (on-chain read) [Authenticated]

Reads an agent wallet's vault-token (share) balance directly from on-chain. **No DB read** — every value comes from the Solana RPC. Useful as a sanity check before a redeem ("how many shares do I actually hold?") and for portfolio displays that need live data rather than the agent's cached `UserVaultPosition`.

**Wallet resolution:** the wallet is canonically read from the JWT (`agentPayload.agentWalletAddress`). The optional `?wallet=<pubkey>` query param is accepted only as a fallback — pass it explicitly when the agent wants to inspect a different wallet's balance.

**Path Params:** `symbol` — vault symbol (case-insensitive).

**Query Params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `wallet` | string | No | Base58 Solana public key. Defaults to the agent wallet from the JWT. |

**Example:**
```
# Default — reads the agent's own balance from JWT
GET /api/v2/agent/vaults/POP-DTF/shares
Authorization: Bearer <token>

# Explicit — read a different wallet's balance
GET /api/v2/agent/vaults/POP-DTF/shares?wallet=5HvUiPYGJ9wshjzDHSb1tYASmF1eomFeYo3zCjMPA7B8
Authorization: Bearer <token>
```

**Response (200):** wrapped in the global `{ status, message, data }` envelope. The inner `data`:

```json
{
  "vaultSymbol": "POP-DTF",
  "vaultIndex": 68,
  "vaultMintPda": "...",
  "wallet": "5HvUiPYGJ9wshjzDHSb1tYASmF1eomFeYo3zCjMPA7B8",
  "ata": "...",
  "sharesRaw": "2543171",
  "sharesUi": "2.543171",
  "exists": true,
  "decimals": 6
}
```

| Field (under `data`) | Description |
|-------|-------------|
| `vaultIndex` | u32 — on-chain vault index resolved from the symbol. |
| `vaultMintPda` | Vault-mint PDA. The SPL token mint that represents the vault's shares. |
| `wallet` | The wallet whose balance was read (JWT-resolved or `?wallet=` query). |
| `ata` | The wallet's associated token account address for the vault mint. |
| `sharesRaw` | Raw u64 share balance as a string (6 decimals). `"0"` when `exists: false`. |
| `sharesUi` | `sharesRaw` converted to UI units, formatted to `decimals` decimal places. |
| `exists` | `true` when the ATA has been created (balance may be 0 or positive). `false` when the ATA does not exist — normal for wallets that have never deposited into this vault. |
| `decimals` | Vault-token decimals. `6` for all DFM vaults today. |

**Errors:** `400` Invalid wallet base58 / vault has no `vaultIndex` / no agent wallet available | `401` Invalid or missing JWT | `404` Vault not found.

---

## Architecture

```
AgentProfileController
    |
    v
AgentProfileService
    |
    +-- AgentVaultService (builds unsigned vault creation transactions)
    +-- PolicyEngineService (constitutional policy commit + evaluation;
    |                        invoked from launchDTF pre-tx, and from dryRun)
    +-- JupiterMarketMetricsService (Jupiter data + Redis cache; powers
    |                                /market-metrics and Rules 2/3)
    +-- AgentRebalanceService (rebalancing executed by admin wallet)
    +-- VaultFactoryService (DB vault operations + builds unsigned fee distribution txs)
    +-- VaultManagementFeesService (fee calculation)
    +-- VaultInsightsService (portfolio & holdings)
    +-- VaultRebalancingService (suggestions & history)
    +-- TxEventManagementService (on-chain tx parsing; auto-links the
                                   pre-committed policy to the new vault
                                   via linkPolicyToVaultByVaultNameAndSymbol)
```

**Deploy flow (policy-gated):**
```
agent basket+policy ──► /policy/dry-run ──(loop)──► /launch-dtf ──► sign+submit ──► /dtf-create
                           free eval              validates + commits    on-chain      metadata only
                                                   policy + builds tx                  (policy auto-linked)
```

**Signing:**
- Vault creation, fee distribution, basket update, **deposit (Step 1)**, and **redeem finalize** all return unsigned transactions for local signing — no secret keys sent to backend.
- Rebalancing, **deposit Step 2 swap fan-out**, and **redeem Step 2 swap fan-in** are executed server-side by the admin wallet; caller only provides public key / JWT for identification.
- `AgentPriceSignerService` (KMS) signs share-price messages used by `/launch-dtf`, fee distribution, **`/deposit-tx`**, and **`/redeem-tx`**. The verify ix is appended to each on-chain tx; the program rejects on signer/vault PDA mismatch.
