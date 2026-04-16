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

**Errors:** `400` Validation error | `404` No profile found for wallet | `409` Username taken or agent already exists

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

## 2. POST `/launch-dtf` - Build Vault Creation Transaction [Authenticated]

Builds an unsigned vault creation transaction and returns it as base64. The agent signs it locally, submits on-chain, then calls `POST /dtf-create` with the resulting transaction signature.

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
  "threshold": 500
}
```

**Required fields:** `signerPublicKey`, `vaultName`, `vaultSymbol`, `underlyingAssets`, `managementFees`

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

**Underlying asset rules:**
- Pass `symbol` or `name` in each `underlyingAssets[]` item (preferred). Backend resolves `mintAddress` from `asset-allocation`.
- **Minimum 1, maximum 12 assets** in `underlyingAssets`. Payloads outside this range are rejected.
- Do not include USDC (`symbol: "USDC"` or `name: "USD Coin"`), as backend blocks USDC for agent-created vault launches.

**Response (201):**
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

| Field | Description |
|-------|-------------|
| `transaction` | Base64-encoded unsigned `VersionedTransaction` -- sign locally and submit on-chain |
| `vaultIndex` | The vault index assigned by the factory |
| `vaultPda` | Vault PDA address |
| `vaultMintPda` | Vault mint PDA address |

**After receiving the response:** Sign the transaction with your local keypair and submit it on-chain. Use the resulting transaction signature in `POST /dtf-create`.

**Errors:** `400` Validation error, USDC blocked, insufficient balance | `404` Asset not found

---

## 3. POST `/dtf-create` - Create DTF (Policy + DB Persist) [Authenticated]

Called after the agent has signed and submitted the vault creation transaction on-chain. Creates the constitutional policy and persists the vault to the database.

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
  "asset_mode": "OPEN",
  "max_asset_pct": 6000,
  "min_rebalance_interval_hours": 4,
  "max_rebalances_per_day": 3,
  "fee_locked": true
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
| `logoUrl` | string | - | Logo image URL (set to `""` for agent launches) |
| `bannerUrl` | string | - | Banner image URL (set to `""` for agent launches) |
| `description` | string | - | Vault description |
| `noRebalance` | boolean | `false` | Disable auto-rebalancing |
| `tags` | string[] | - | Searchable tags |

**Optional policy fields** (defaults from `policy.json` if omitted):

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

**Response (201):**
```json
{
  "policy": {
    "_id": "665c...",
    "vault_name": "Blue Chip Fund",
    "vault_symbol": "BCF",
    "asset_mode": "OPEN",
    "agentDTFPolicy": true
  },
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
  ]
}
```

**Errors:** `400` Validation error | `409` Policy already exists for this vault name/symbol

---

## 4. GET `/dtf/my-vaults` - List My Vaults [Authenticated]

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

Validates rebalancing against the constitutional policy without executing. Returns the suggestion if approved.

**Path params:** `vaultSymbol` - Vault symbol

**Response (200) - Approved:**
```json
{
  "policyCheck": { "ok": true },
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

**Response (400) - Policy Violated:**

Returns ALL violated policies. The `violations` array lists every rule that failed with its code, message, and optional details.

```json
{
  "statusCode": 400,
  "error": "PolicyViolation",
  "message": "Must wait 4h between rebalances | Max 3 rebalances per day exceeded",
  "violations": [
    {
      "violationCode": "rule9MinTimeBetweenRebalances",
      "message": "Must wait 4h between rebalances",
      "details": { "lastRebalanceAt": "2026-04-10T11:30:00.000Z", "minIntervalHours": 4 }
    },
    {
      "violationCode": "rule10MaxRebalancesDayWeek",
      "message": "Max 3 rebalances per day exceeded",
      "details": { "rebalanceCountDay": 3, "maxPerDay": 3 }
    }
  ]
}
```

**Violation codes:**

| Code | Description |
|------|-------------|
| `assetModeViolation` | Asset not allowed by vault's asset mode |
| `rule1WhitelistBlacklist` | Blacklisted or not whitelisted |
| `rule2MinAmmLiquidity` | Below min AMM liquidity |
| `rule3Min24hVolume` | Below min 24h volume |
| `platformMaxAssetsExceeded` | Exceeds platform max (12) |
| `rule4MinMaxAssetCount` | Asset count out of bounds |
| `rule5MaxPctPerAsset` | Single asset over max % |
| `rule6MinPctPerAssetIfHeld` | Held asset under min % |
| `rule7MinStablecoinFloor` | Stablecoin below floor |
| `rule8MaxPctRebalancedPerTx` | Rebalance % too large |
| `rule9MinTimeBetweenRebalances` | Too soon since last rebalance |
| `rule10MaxRebalancesDayWeek` | Daily/weekly cap exceeded |
| `rule11LaunchBlackout` | In launch blackout period |

---

## 7. POST `/dtf/:symbol/rebalance` - Execute Rebalancing [Authenticated]

Triggers vault rebalancing. Runs full policy evaluation before executing — if any policy rule is violated, returns all violations. The caller provides their public key for identification; rebalancing is executed server-side by the admin wallet. Runs sell phase then buy phase on-chain.

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

**Response (200):**
```json
{
  "ok": true,
  "vaultId": "664a1b2c3d4e5f6a7b8c9d0e",
  "vaultSymbol": "BCF"
}
```

**Response (400) - Policy Violated:**

Same format as rebalance check — all violated policies returned:

```json
{
  "statusCode": 400,
  "error": "PolicyViolation",
  "message": "Policy violated: [rule9MinTimeBetweenRebalances] Must wait 4h between rebalances; [rule11LaunchBlackout] Vault in 24h launch blackout",
  "violations": [
    { "violationCode": "rule9MinTimeBetweenRebalances", "message": "Must wait 4h between rebalances" },
    { "violationCode": "rule11LaunchBlackout", "message": "Vault in 24h launch blackout" }
  ]
}
```

**Errors:** `400` Policy violated (with `violations` array) or rebalancing already in progress | `404` Vault not found

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

Re-issues a new agent token using the agent's `profileId`. Revokes **all** existing tokens for the agent. Use this when the current token has expired.

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

Re-issues a new agent token using the wallet address registered on the DFM Dashboard. Revokes **all** existing tokens for the agent. Use this when the current token has expired and you don't have the profileId.

**Request Body:**
```json
{
  "walletAddress": "YourWalletPublicKeyHere"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `walletAddress` | string | Yes | Base58-encoded public key of the DFM-registered wallet |

**Response (200):**
```json
{
  "token": "eyJhbGciOi...",
  "expiresIn": "30d"
}
```

**Errors:** `404` No profile or agent found for this wallet address

---

## Architecture

```
AgentProfileController
    |
    v
AgentProfileService
    |
    +-- AgentVaultService (builds unsigned vault creation transactions)
    +-- PolicyEngineService (constitutional policy CRUD)
    +-- AgentRebalanceService (rebalancing executed by admin wallet)
    +-- VaultFactoryService (DB vault operations + builds unsigned fee distribution txs)
    +-- VaultManagementFeesService (fee calculation)
    +-- VaultInsightsService (portfolio & holdings)
    +-- VaultRebalancingService (suggestions & history)
    +-- TxEventManagementService (on-chain tx parsing & DB persistence)
```

**Signing:**
- Vault creation and fee distribution return unsigned transactions for local signing (no secret keys sent to backend)
- Rebalancing is executed server-side by the admin wallet; caller provides public key for identification
- `AgentPriceSignerService` -- server-side KMS price message signing for fee distribution (not caller-provided)
