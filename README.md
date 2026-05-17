<img src="https://launchpad.meme/assets/launchpad-logo.png" width="500" alt="" data-canonical-src="https://launchpad.meme/assets/launchpad-logo.png">  &nbsp;&nbsp;

# launchpad.meme Public API

The launchpad.meme public API lets apps, launch tools, dashboards, and partner integrations create Solana token launches without automating the website UI.

Base URL:

```text
https://launchpad.meme/api/v1
```

## Authentication

Send your API key as a bearer token:

```http
Authorization: Bearer lp_live_xxxxxxxxxxxxxxxxxxxxx
```

Create self-service keys at:

```text
https://launchpad.meme/developers
```

API keys are displayed once. Store them securely in your backend environment and never expose them in browser code, frontend bundles, mobile apps, public repositories, screenshots, or client-side logs.

## Automated launch wallet mode

New API keys use **automated launch wallet mode** by default.

That means:

- The wallet connected on `/developers` is the launch wallet for that API key.
- `creator_wallet` in API requests must match that launch wallet.
- The launch wallet funds token setup, Solana network fees, priority fees, and optional initial buys.
- API keys cannot launch tokens for arbitrary wallets.
- `create-transaction` returns a Solana transaction for the launch wallet to sign and send.
- `finalize` verifies the submitted Solana transaction before the coin page becomes live.

## Flow overview

```text
1. Create an API key on /developers
2. POST /coins/create-draft
3. POST /coins/create-transaction
4. Sign and send transaction_base64 with the launch wallet
5. POST /coins/finalize with tx_signature
6. Open the returned coin URL or query /coins/status
```

## Endpoints

### Health

```bash
curl -sS https://launchpad.meme/api/v1/health | jq .
```

### Create draft

Creates and validates a coin draft. The coin page is not published until finalize succeeds.

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

Example response:

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

### Create transaction

Prepares the Solana create transaction.

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
    "wallet_action": {
      "type": "automated_launch_wallet"
    },
    "blockchain_transaction_ready": true,
    "transaction_base64": "..."
  }
}
```

### Sign and send the transaction

Use your own Solana signer or wallet infrastructure. The signer must control the launch wallet connected when the API key was created.

Pseudo-code:

```text
transaction_base64 -> deserialize Solana transaction -> sign with launch wallet -> send to Solana RPC -> tx_signature
```

### Finalize

Finalize verifies the draft, transaction request, and submitted Solana transaction before publishing the coin page.

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

Successful finalize responses include the coin URL and verification status.

### Status

```bash
curl -sS "https://launchpad.meme/api/v1/coins/status?draft_id=${DRAFT_ID}" \
  -H "Authorization: Bearer ${API_KEY}" | jq .
```

## More examples

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

### Node.js finalize example

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

## Rate limits

Default self-service limits:

```text
60 requests/minute
500 drafts/day
500 finalized coins/day
Maximum 5 active API keys per wallet
```

## Developer security basics

- Keep API keys server-side.
- Never commit API keys, wallet secrets, or signed transactions to public repositories.
- Use one API key per app or integration.
- Rotate any key that may have been exposed.
- Verify finalized coin URLs and transaction signatures in your own application logs.
- Fund launch wallets according to your app’s operating needs and risk limits.

## Common errors

`creator_wallet must match launch wallet`  
Use the wallet connected when the API key was created.

`transaction_base64 missing`  
Check whether `initial_buy_sol` is present and positive.

`finalize rejected tx_signature`  
Make sure the launch wallet signed and sent the exact `transaction_base64` from `create-transaction`.

`draft expired`  
Create a new draft and transaction request.
