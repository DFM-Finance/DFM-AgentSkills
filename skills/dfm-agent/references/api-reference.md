# DFM Agent API Reference

Base URL: `{DFM_API_URL}/api/v2/agent`

The `DFM_API_URL` environment variable defines the API host. All endpoints below are relative to this base URL.

| Environment | `DFM_API_URL` |
|---|---|
| **Production** | `https://api.qa.dfm.finance` |
| **Local dev** | `http://0.0.0.0:3400` |

If `DFM_API_URL` is not set, default to `https://api.qa.dfm.finance`.

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
| Revocable | Yes — via dashboard or `POST /token/revoke` |
| Request identity | Guard decodes JWT and attaches `agentPayload`, `agentId`, `userId` to request |

### On-chain signer (`popeyeSecret`)

Endpoints that perform on-chain transactions (`launch-dtf`, `rebalance`, `distribute-fees`) require a `popeyeSecret` field in the request body. This is the base58-encoded Agent Wallet secret key, read from the `DFM_AGENT_KEYPAIR` environment variable.

---

## 1. POST `/profile` - Create Agent Profile [Public]

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

## 2. POST `/launch-dtf` - Launch DTF Vault [Authenticated]

4-step flow: create constitutional policy, deploy vault on-chain using a caller-provided Solana keypair, persist to DB, mark as agent-created.

**Request Body:**
```json
{
  "popeyeSecret": "<DFM_AGENT_KEYPAIR env variable>",
  "vaultName": "Blue Chip Fund",
  "vaultSymbol": "BCF",
  "underlyingAssets": [
    { "mintAddress": "So11111111111111111111111111111111111111112", "mintBps": 5000 },
    { "mintAddress": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v", "mintBps": 5000 }
  ],
  "managementFees": 200,
  "metadataUri": "",
  "category": 1,
  "threshold": 500,
  "vaultType": "DTF",
  "logoUrl": "",
  "bannerUrl": "",
  "description": "A diversified blue chip Solana fund",
  "noRebalance": false,
  "tags": ["DeFi", "Blue Chip"],
  "asset_mode": "OPEN",
  "fee_locked": true,
  "max_rebalances_per_day": 3,
  "min_rebalance_interval_hours": 4
}
```

**Required fields:** `popeyeSecret`, `vaultName`, `vaultSymbol`, `underlyingAssets`, `managementFees`

**On-chain signer:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `popeyeSecret` | string | Yes | Base58-encoded signer credential from `DFM_AGENT_KEYPAIR` env var |

**Launch media field rule:**
- For DTF launch requests, set `metadataUri`, `logoUrl`, and `bannerUrl` to empty strings (`""`).

**Optional on-chain fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `metadataUri` | string | `""` | IPFS metadata URI |
| `category` | number | `0` | 0=Manual, 1=Automatic, 2=DAO |
| `threshold` | number/null | `null` | Rebalance threshold in bps |

**Optional DB fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `vaultType` | `"DTF"` / `"YIELD_DTF"` | `"DTF"` | Vault type |
| `logoUrl` | string | - | Logo image URL |
| `bannerUrl` | string | - | Banner image URL |
| `description` | string | - | Vault description |
| `noRebalance` | boolean | `false` | Disable auto-rebalancing |
| `tags` | string[] | - | Searchable tags |

**Optional policy fields** (defaults from `policy.json` if omitted):

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `asset_mode` | `"OPEN"` / `"WHITELIST"` / `"BLACKLIST"` | `"OPEN"` | Asset restriction mode |
| `asset_whitelist` | string[] | `[]` | Whitelisted mint addresses |
| `asset_blacklist` | string[] | `[]` | Blacklisted mint addresses |
| `min_amm_liquidity_usd` | number | `0` | Min AMM liquidity (0=disabled) |
| `min_24h_volume_usd` | number | `0` | Min 24h volume (0=disabled) |
| `min_assets` / `max_assets` | number | `0` | Asset count bounds (0=disabled) |
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
  "onChain": {
    "signature": "5Kz...abc",
    "vaultIndex": 42,
    "vaultPda": "7Xk...def",
    "vaultMintPda": "9Rm...ghi"
  },
  "vault": {
    "vaultName": "Blue Chip Fund",
    "vaultSymbol": "BCF",
    "vaultIndex": 42,
    "status": "active",
    "transactionSignature": "5Kz...abc"
  }
}
```

---

## 3. GET `/dtf/:symbol/state` - Get Vault State [Authenticated]

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
    "data": [
      {
        "_id": "665b...",
        "vaultSymbol": "BCF",
        "status": "completed",
        "startTime": "2026-04-09T10:00:00.000Z",
        "endTime": "2026-04-09T10:02:30.000Z",
        "sellPhaseTransactions": ["..."],
        "buyPhaseTransactions": ["..."],
        "totalUsdcReceived": 5000,
        "totalUsdcSpent": 4950,
        "executionDurationMs": 150000
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total": 5,
      "totalPages": 1,
      "hasNext": false,
      "hasPrev": false
    }
  }
}
```

---

## 4. GET `/dtf/:symbol/policy` - Get Vault Policy [Authenticated]

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

## 5. GET `/dtf/:vaultSymbol/rebalance/check` - Dry-Run Rebalance Check [Authenticated]

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
```json
{
  "statusCode": 400,
  "error": "PolicyViolation",
  "violationCode": "RULE_9_MIN_INTERVAL",
  "message": "Rebalancing too soon. Minimum interval is 4 hours."
}
```

---

## 6. POST `/dtf/:symbol/rebalance` - Execute Rebalancing [Authenticated]

Executes vault rebalancing using a caller-provided Solana keypair. Runs sell phase then buy phase on-chain.

**Path params:** `symbol` - Vault symbol

**Request Body:**
```json
{
  "popeyeSecret": "<DFM_AGENT_KEYPAIR env variable>"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `popeyeSecret` | string | Yes | Base58-encoded signer credential from `DFM_AGENT_KEYPAIR` env var |

**Response (200):**
```json
{
  "ok": true,
  "vaultId": "664a1b2c3d4e5f6a7b8c9d0e",
  "vaultSymbol": "BCF"
}
```

**Errors:** `400` Policy violated or rebalancing already in progress | `404` Vault not found

---

## 7. POST `/dtf/:symbol/distribute-fees` - Distribute Management Fees [Authenticated]

Distributes accrued management fees for a vault on-chain using a caller-provided Solana keypair. Fees are recalculated server-side from trusted sources.

**Path params:** `symbol` - Vault symbol

**Request Body:**
```json
{
  "popeyeSecret": "<DFM_AGENT_KEYPAIR env variable>"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `popeyeSecret` | string | Yes | Base58-encoded signer credential from `DFM_AGENT_KEYPAIR` env var |

**Response (200):**
```json
{
  "status": "success",
  "message": "OK",
  "transactionSignature": "3Rz...xyz",
  "data": {
    "transactionSignature": "3Rz...xyz",
    "vaultIndex": 42,
    "date": "2026-04-10"
  }
}
```

---

## 8. POST `/token/revoke` - Revoke Agent Token [Authenticated]

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

## 9. POST `/token/refresh` - Refresh Agent Token [Public]

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

## Architecture

```
AgentProfileController
    |
    v
AgentProfileService
    |
    +-- AgentVaultService (on-chain vault creation via caller-provided keypair)
    +-- PolicyEngineService (constitutional policy CRUD)
    +-- AgentRebalanceService (rebalancing via caller-provided keypair)
    +-- VaultFactoryService (DB vault operations + fee distribution)
    +-- VaultManagementFeesService (fee calculation)
    +-- VaultInsightsService (portfolio & holdings)
    +-- VaultRebalancingService (suggestions & history)
    +-- TxEventManagementService (on-chain tx parsing & DB persistence)
```

**Agent Signing:**
- Vault creation, rebalancing, and fee distribution use a caller-provided Solana keypair (`popeyeSecret`) for on-chain transaction signing
- `AgentPriceSignerService` - Price message signing for fee distribution (server-side KMS, not caller-provided)
