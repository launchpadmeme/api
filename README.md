<a href="https://launchpad.meme"> <img src="https://launchpad.meme/assets/launchpad-logo.png" width="370" alt="" data-canonical-src="https://launchpad.meme/assets/launchpad-logo.png"></a>  &nbsp;&nbsp;

# launchpad.meme Public API

The launchpad.meme public API lets apps, dashboards, launch tools, and partner integrations create Solana token launches without automating the website UI.

Base URL:

```text
https://launchpad.meme/api/v1
```

Docs and developer portal:

```text
https://github.com/launchpadmeme/api
https://launchpad.meme/developers
```

## Quick start in 5 minutes

1. Connect your wallet on `/developers` and create an API key.
2. Create a draft with `POST /coins/create-draft`.
3. Create a transaction with `POST /coins/create-transaction`.
4. Sign and send the returned `transaction_base64` with the launch wallet.
5. Finalize with `POST /coins/finalize` and the Solana `tx_signature`.
6. Open the returned coin URL or poll `GET /coins/status`.

## Authentication

Send your API key as a bearer token on protected endpoints:

```http
Authorization: Bearer lp_live_xxxxxxxxxxxxxxxxxxxxx
```

API keys are displayed once. Store them in your backend environment. Do not expose keys in browser code, public repositories, frontend bundles, mobile apps, screenshots, or client-side logs.

## Automated launch wallet mode

New API keys use **automated launch wallet mode** by default.

- The wallet connected on `/developers` is the launch wallet for that key.
- `creator_wallet` in create requests must match that launch wallet.
- The launch wallet signs the prepared Solana transaction.
- The launch wallet funds token setup, Solana network fees, priority fees, and optional initial buys.
- API keys cannot launch tokens for arbitrary wallets.
- `finalize` verifies the submitted Solana transaction before the coin page becomes live.

## Endpoint overview

| Endpoint | Auth | Purpose |
| --- | --- | --- |
| `GET /api/v1` | Public | API discovery JSON. |
| `GET /api/v1/health` | Public | Public API health check. |
| `POST /api/v1/coins/create-draft` | Bearer key | Validate and create a draft. |
| `POST /api/v1/coins/create-transaction` | Bearer key | Prepare the Solana transaction. |
| `POST /api/v1/coins/finalize` | Bearer key | Verify and publish after transaction send. |
| `GET /api/v1/coins/status?draft_id=...` | Bearer key | Check draft/finalize status. |

## API root

```bash
curl -sS https://launchpad.meme/api/v1 | jq .
```

Example response shape:

```json
{
  "ok": true,
  "name": "Launchpad.meme API",
  "version": "v1",
  "status": "online",
  "docs": "https://github.com/launchpadmeme/api",
  "authentication": "Bearer API key"
}
```

## Health

```bash
curl -sS https://launchpad.meme/api/v1/health | jq .
```

Example response shape:

```json
{
  "ok": true,
  "status": "online",
  "api": "v1",
  "service": "launchpad-api",
  "version": "v1",
  "docs": "https://github.com/launchpadmeme/api"
}
```

The health response is public and developer-safe. It does not expose infrastructure, wallet, database, or admin details.

## Full curl flow

### 1. Create draft

```bash
API_KEY="lp_live_..."
LAUNCH_WALLET="YOUR_CONNECTED_WALLET"

curl -sS -X POST https://launchpad.meme/api/v1/coins/create-draft \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"chain\": \"solana\",
    \"token_standard\": \"token_2022\",
    \"creator_wallet\": \"${LAUNCH_WALLET}\",
    \"name\": \"Example Coin\",
    \"symbol\": \"EXAMPLE\",
    \"description\": \"Created through the launchpad.meme public API.\",
    \"image_url\": \"https://example.com/logo.webp\",
    \"website_url\": \"https://example.com\",
    \"twitter_url\": \"https://x.com/example\",
    \"telegram_url\": \"https://t.me/example\",
    \"initial_buy_sol\": \"0.01\",
    \"slippage\": \"2\",
    \"priority_speed\": \"fast\"
  }" | jq .
```

Example response shape:

```json
{
  "ok": true,
  "draft": {
    "draft_id": "draft_...",
    "status": "draft",
    "expires_in": 900
  }
}
```

### 2. Create transaction

```bash
DRAFT_ID="draft_..."

curl -sS -X POST https://launchpad.meme/api/v1/coins/create-transaction \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{\"draft_id\":\"${DRAFT_ID}\"}" | jq .
```

Example response shape:

```json
{
  "ok": true,
  "result": {
    "status": "transaction_ready",
    "transaction_request_id": "txreq_...",
    "blockchain_transaction_ready": true,
    "transaction_base64": "..."
  }
}
```

### 3. Sign and send transaction

Use your own Solana signer or wallet infrastructure. The signer must control the launch wallet connected when the API key was created.

```text
transaction_base64 -> deserialize Solana transaction -> sign with launch wallet -> send to Solana RPC -> tx_signature
```

### 4. Finalize

```bash
TXREQ_ID="txreq_..."
REAL_SIG="REAL_SOLANA_SIGNATURE"

curl -sS -X POST https://launchpad.meme/api/v1/coins/finalize \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"draft_id\": \"${DRAFT_ID}\",
    \"transaction_request_id\": \"${TXREQ_ID}\",
    \"tx_signature\": \"${REAL_SIG}\"
  }" | jq .
```

Successful finalize responses include the public coin URL and verification status.

### 5. Status check

```bash
curl -sS "https://launchpad.meme/api/v1/coins/status?draft_id=${DRAFT_ID}" \
  -H "Authorization: Bearer ${API_KEY}" | jq .
```

Common lifecycle states:

| State | Meaning |
| --- | --- |
| `draft` | Draft was accepted and is waiting for transaction preparation. |
| `transaction_ready` | A Solana transaction was prepared and should be signed by the launch wallet. |
| `finalized` | The transaction was verified and the coin URL is available. |
| `failed` | The request could not complete. Read `message`, fix the issue, and retry safely. |

## More draft examples

### Minimal draft without initial buy

```bash
curl -sS -X POST https://launchpad.meme/api/v1/coins/create-draft \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"chain\": \"solana\",
    \"token_standard\": \"token_2022\",
    \"creator_wallet\": \"${LAUNCH_WALLET}\",
    \"name\": \"No Buy Test\",
    \"symbol\": \"NOBUY\",
    \"description\": \"Draft without initial buy.\",
    \"image_url\": \"https://example.com/logo.webp\"
  }" | jq .
```

### Token-2022 with initial buy

```bash
curl -sS -X POST https://launchpad.meme/api/v1/coins/create-draft \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"chain\": \"solana\",
    \"token_standard\": \"token_2022\",
    \"creator_wallet\": \"${LAUNCH_WALLET}\",
    \"name\": \"First Buy Test\",
    \"symbol\": \"FBT\",
    \"description\": \"Automated create plus initial buy.\",
    \"image_url\": \"https://example.com/logo.webp\",
    \"initial_buy_sol\": \"0.01\",
    \"slippage\": \"2\",
    \"priority_speed\": \"fast\"
  }" | jq .
```

### Legacy SPL token standard

```bash
curl -sS -X POST https://launchpad.meme/api/v1/coins/create-draft \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"chain\": \"solana\",
    \"token_standard\": \"spl_tokenkeg\",
    \"creator_wallet\": \"${LAUNCH_WALLET}\",
    \"name\": \"Legacy SPL Test\",
    \"symbol\": \"SPLT\",
    \"description\": \"Legacy SPL launch test.\",
    \"image_url\": \"https://example.com/logo.webp\",
    \"initial_buy_sol\": \"0.01\"
  }" | jq .
```

## Node.js finalize example

```js
const API_KEY = process.env.LAUNCHPAD_API_KEY;
const draftId = process.env.DRAFT_ID;
const txRequestId = process.env.TXREQ_ID;
const txSignature = process.env.TX_SIGNATURE;

const res = await fetch('https://launchpad.meme/api/v1/coins/finalize', {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${API_KEY}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    draft_id: draftId,
    transaction_request_id: txRequestId,
    tx_signature: txSignature,
  }),
});

const data = await res.json();
console.log(data);
```

## Example files

This repo can include ready-to-copy examples:

```text
examples/create-coin-curl.sh
examples/create-coin-node.mjs
examples/status-check.sh
```

## Validation rules

- `chain`: currently `solana`
- `token_standard`: `token_2022` or `spl_tokenkeg`
- `creator_wallet`: must match the API key launch wallet
- `name`: 1–32 characters
- `symbol`: 1–10 alphanumeric characters
- `image_url`: HTTPS PNG/JPG/JPEG/WEBP/GIF URL
- `website_url`, `twitter_url`, `telegram_url`: HTTPS URLs when provided
- `initial_buy_sol`: optional positive SOL amount as a string or number
- `slippage`: optional percentage
- `priority_speed`: optional priority preset, for example `fast`

## Rate limits and retry behavior

Default self-service limits:

```text
60 requests/minute
500 drafts/day
500 finalized coins/day
Maximum 5 active API keys per wallet
```

On `429`, back off before retrying. For temporary `5xx` responses, retry safely and keep your own request logs. Do not retry finalize with a different transaction signature for the same transaction request.

## Developer security basics

- Keep API keys server-side.
- Never commit API keys, wallet secrets, or signed transactions to public repositories.
- Use one API key per app or integration.
- Rotate any key that may have been exposed.
- Verify finalized coin URLs and transaction signatures in your own application logs.
- Fund launch wallets according to your app’s operating needs and risk limits.

## Common API responses

| HTTP status / message | Meaning | Recommended handling |
| --- | --- | --- |
| `400` | Bad JSON or invalid request shape. | Fix the request body and retry. |
| `401` | Missing, invalid, or disabled API key. | Check the `Authorization: Bearer` header and key status. |
| `403` | Wallet or origin is not allowed for the key. | Use the launch wallet connected when the API key was created. |
| `404` | Endpoint or resource was not found. | Check the route and request method. |
| `409` | Draft or transaction state conflict. | Query status and continue from the latest state. |
| `422` | Draft validation failed. | Show the validation message and correct the input. |
| `429` | Rate limit reached. | Back off and retry later. |
| `5xx` | Temporary service problem. | Retry with safe backoff and keep your own request logs. |
| `creator_wallet must match launch wallet` | The key is locked to automated launch wallet mode. | Set `creator_wallet` to the wallet connected when the API key was created. |
| `finalize rejected tx_signature` | The submitted signature does not match the prepared transaction/draft. | Send the exact transaction returned by `create-transaction`, then finalize with that signature. |
| `draft expired` | Draft or transaction request is too old. | Create a new draft and transaction request. |
