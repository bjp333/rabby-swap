# FlowRouter Swap API v1 — partner guide

Robinhood Chain (chain id 4663). Base URL: `https://api.flowrouter.xyz`.

This guide is for developers who integrate FlowRouter swaps into a wallet, an aggregator or a backend.
It describes the API source at `eea86314b1b897c19aab66b80963d49fce648e43`. Every field below was checked against that source.
No live host build was inspected. This source pin does not prove the current running API build.
Historical commit `880c2a251` is an ancestor of this source, not a current deployment receipt.
Section 12 lists what we could not check.

## 1. Quick facts

| Item | Value |
| --- | --- |
| Chain | Robinhood Chain, id `4663` |
| Router (= `transaction.to` = token spender) | `0x8Ad60fAE44DD6aaFD8A98B7f25EaD026013efdF2` |
| Native ETH | `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` (the API also accepts `0x0000000000000000000000000000000000000000`) |
| Authentication | Header `X-API-Key` on every request |
| Methods | `GET` only. No request body. |
| Amounts | Base-10 integer strings in the token's smallest unit (wei for ETH) |
| Quote lifetime | 30 seconds or less (`expiresAt`) |
| Order type | Exact input (sell a fixed amount). The taker receives the output. |
| Calls from browsers | Refused. Call from your server or from your own proxy. |

## 2. Authentication

**HUMAN OPERATOR ONLY:** Credential creation, file inspection, use, cleanup, revocation, rotation and hand-over
are human tasks. Agents must not read, handle or create credential or header files. The authenticated examples
in this guide are for a human-operated partner server. They are not agent run commands.

- Send your key in the `X-API-Key` header. Send it once. Do not put it in the URL.
- A key has the form `fr_live_<24 hex>_<64 hex>`. We give you the key through a private channel.
- A key has a lifetime of 366 days or less. We give you a new key before the old one expires.
- We store only a SHA-256 hash of your key. If you lose the key, we issue a new one.
- Each key is bound to one FlowRouter referral code on chain. The code sets your fee or your surplus share
  (section 8). You do not send the code in requests.
- Browser calls are refused. A request with an `Origin` header gets `403 FORBIDDEN`. There is no CORS.
  Keep the key on your server.

Missing key, a wrong key, an expired key, a revoked key, or two `X-API-Key` headers: `401 UNAUTHORIZED`.
A key without the scope for the endpoint: `403 FORBIDDEN`.

## 3. Endpoints

| Endpoint | Purpose |
| --- | --- |
| `GET /v1/4663/price` | Indicative price. No transaction. Use it to show a price before the user connects a wallet. |
| `GET /v1/4663/quote` | Firm quote for one taker, with a transaction that we simulated. |
| `GET /v1/4663/tokens` | The tokens that we route now. |
| `GET /v1/health` | Is the service ready? |

All endpoints need the key. Any other path under `/v1/` gets `404 NOT_FOUND`.

### 3.1 Request parameters (`/price` and `/quote`)

| Parameter | Required | Rules |
| --- | --- | --- |
| `sellToken` | yes | Token address (any letter case), or the native ETH address. |
| `buyToken` | yes | Token address, or the native ETH address. Must differ from `sellToken`. |
| `sellAmount` | yes | Integer string, 1 to 78 digits, no leading zero. Smallest unit of `sellToken`. |
| `slippageBps` | no | Integer 1 to 1000, no leading zero. Default `50` (0.50%). |
| `taker` | `/quote`: yes. `/price`: optional | The address that sends the transaction and receives the output. `/price` accepts it but does not use it. |
| `recipient` | `/quote` only, optional | Must be equal to `taker`. A different recipient is not supported in v1. |

Strict rules (each one gives `400 INVALID_REQUEST`):

- An unknown parameter is refused. Examples: `slippagePercentage`, `rfq`, `excludedSources`, `feeBps`.
- A parameter that is sent twice, or with an empty value, is refused.
- The query string must be 1,024 bytes or less.
- `taker` must not be the zero address or the native ETH address.

A token that is not in the current token list gives `400 TOKEN_NOT_SUPPORTED`.
`/tokens` and `/health` take no parameters. Any parameter there gives `400 INVALID_REQUEST`.

### 3.2 `/price` and `/quote` — common response fields

| Field | Type | Meaning |
| --- | --- | --- |
| `chainId` | number | Always `4663`. |
| `blockNumber` | string | The block of the chain state that the route was priced on. |
| `sellToken` | string | Your `sellToken`, in the spelling that you sent. |
| `buyToken` | string | Your `buyToken`, in the spelling that you sent. |
| `sellAmount` | string | Your `sellAmount`. In `/quote` it is also the exact `inputAmount` in the calldata. |
| `buyAmount` | string | The amount the taker receives if the swap does at least as well as the quote. It is the MOST the taker receives: any surplus above it does not go to the taker (section 8.3). Net of your fee. |
| `minBuyAmount` | string | The on-chain minimum. The transaction reverts if the taker would receive less. It follows from `slippageBps`. Net of your fee. |
| `allowanceTarget` | string or null | The address to approve for `sellToken`: the router. `null` when you sell native ETH. |
| `fees.integratorFee` | object or null | Your fee (section 8.1). `null` when your code has no fee. |
| `fees.flowRouterFee` | object or null | FlowRouter's part of that fee. `null` when your code has no fee. |
| `route.fills[]` | array | The route: one entry per pool step (see below). |
| `gas` | string | The gas that we expect the swap to use (our simulation of this route). |
| `gasPrice` | string | Our gas price for this quote, in wei per gas (the node's `eth_gasPrice` that the solver used). It is an estimate, not a guaranteed price. |
| `totalNetworkFee` | string | `gas` × `gasPrice`, in wei. The expected network cost. |
| `liquidityAvailable` | boolean | Always `true` in a 200 response. When there is no route you get an error instead. |
| `requestId` | string | Our id for this request. It is also in the `X-Request-Id` header. Send it to us when you report a problem. |

`route.fills[]` entries:

| Field | Meaning |
| --- | --- |
| `source` | The venue id, for example `uniswap-v4`, `uniswap-v3`, `wrapped-native`. |
| `from` | The input token of this step, lower case. `native` for ETH. |
| `to` | The output token of this step, lower case. `native` for ETH. |
| `proportionBps` | The share of all `from` tokens on the route that this step takes, in basis points. The steps that start from the same token add up to 10000. A simple multi-hop route has 10000 on every step. |

### 3.3 `/price` only

| Field | Meaning |
| --- | --- |
| `indicative` | Always `true`. A price is not firm. |

A price runs the same solver as a quote, but without a taker. It does not ask market makers for firm
quotes, so a later `/quote` can differ. We may give the same price to equal requests (same tokens, amount
and slippage) for up to 2 seconds.

### 3.4 `/quote` only

| Field | Meaning |
| --- | --- |
| `quoteId` | Our id for this quote. |
| `transaction.to` | The router `0x8Ad60fAE44DD6aaFD8A98B7f25EaD026013efdF2`. Check this value. |
| `transaction.data` | The calldata: `swap((address,uint256,address,uint256,uint256,address),bytes,address,uint32)`, selector `0xd46cadbc`. |
| `transaction.value` | Wei to send. Equal to `sellAmount` when you sell native ETH, else `"0"`. |
| `transaction.gas` | The gas LIMIT to send (section 7). Send it as it is. |
| `transaction.gasPrice` | The highest fee per gas that the transaction may pay (a fee cap). It is not the expected price. |
| `expiresAt` | Epoch time in milliseconds. Send the transaction before this time. |
| `quoteValidUntil` | The same value as `expiresAt`. |
| `issues.allowance` | `null`, or `{ "actual", "spender" }` when the taker's allowance for the router is less than `sellAmount`. |
| `issues.balance` | `null`, or `{ "token", "actual", "expected" }` when the taker holds less than `sellAmount`. For native ETH this compares the ETH balance with `sellAmount` only (not with gas). |
| `issues.simulationIncomplete` | Always `false` in v1. |
| `issues.invalidSourcesPassed` | Always `[]` in v1. |
| `simulation.status` | Always `"success"`. A failed simulation gives an error, not a quote. |
| `simulation.blockNumber` | The block of the simulation. |
| `simulation.funding` | Always `"canonical_overrides"`: the simulation gives the taker the sell amount and the allowance with state overrides. So you get a quote even when the taker has no funds yet. Read `issues` for what the taker lacks. |

The calldata fields decode as follows. `tokenInfo = (inputToken, inputAmount, outputToken, outputQuote,
outputMin, outputReceiver)`:

| Calldata field | API field |
| --- | --- |
| `inputToken` | `sellToken` (address(0) for native ETH) |
| `inputAmount` | `sellAmount` (exact) |
| `outputToken` | `buyToken` (address(0) for native ETH) |
| `outputQuote` | `buyAmount` |
| `outputMin` | `minBuyAmount` |
| `outputReceiver` | `taker` |
| `referralCode` (last argument) | your referral code |

Note: on FlowRouter, `outputReceiver = address(0)` does NOT mean "the sender". We never build such a
transaction. If you decode our calldata, treat a zero receiver as invalid.

### 3.5 `/tokens`

```json
{
  "chainId": 4663,
  "nativeToken": "0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE",
  "tokens": [
    { "address": "0x…", "symbol": "USDG", "name": "…", "decimals": 6, "logoURI": null, "provenance": "curated" }
  ]
}
```

- `tokens` holds ERC-20 tokens, sorted by address. Native ETH is not in the list. It is in `nativeToken`.
- `address` is EIP-55 checksummed. `logoURI` is always `null` in v1.
- `provenance` is `curated` or `automatic-admission` (a token that our checks admitted automatically).
- The list changes over time. Read it again from time to time (for example every 10 minutes).
- A live call on 2026-10-16 returned 5,676 tokens.

### 3.6 `/health`

```json
{ "chainId": 4663, "ready": true, "router": "0x8Ad60fAE44DD6aaFD8A98B7f25EaD026013efdF2" }
```

`ready: false` means that quotes are not available now. The HTTP status is 200 in both cases.

## 4. Errors

Every error from the API has this body and an HTTP status:

```json
{ "code": "NO_ROUTE", "message": "No executable route is available right now.", "requestId": "…" }
```

| HTTP | `code` | Meaning | What to do |
| --- | --- | --- | --- |
| 400 | `INVALID_REQUEST` | A parameter is missing, unknown, sent twice or out of range; `recipient` ≠ `taker`; same buy and sell token. | Fix the request. |
| 400 | `TOKEN_NOT_SUPPORTED` | A token is not in our current list. | Check `/tokens`. |
| 401 | `UNAUTHORIZED` | No key, a wrong key, an expired or revoked key, or two keys. | Check the header. |
| 403 | `FORBIDDEN` | The key does not have this scope, or the request has an `Origin` header (browser call). | Call from a server. |
| 404 | `NOT_FOUND` | Unknown path. | |
| 405 | `METHOD_NOT_ALLOWED` | Only `GET`. (Through the public URL, most other methods are stopped earlier with an HTML 403; see below.) | Use `GET`. |
| 422 | `NO_ROUTE` | No executable route now. | Try later or another amount. |
| 422 | `INSUFFICIENT_LIQUIDITY` | The route cannot take the full sell amount. We never return a partial fill. | Use a smaller amount. |
| 422 | `SIMULATION_FAILED` | The transaction simulation did not succeed, or the quote went stale. | Request a new quote. |
| 429 | `RATE_LIMITED` | Your key's request limit was reached. Has `Retry-After` (seconds). | Wait `Retry-After`. |
| 503 | `SERVICE_BUSY` | The service is at its limit now. Has `Retry-After`. There is no queue. | Wait and retry. |
| 503 | `SERVICE_UNAVAILABLE` | The service cannot quote now (for example a stale chain state, or your referral code is not active yet). Often has `Retry-After`. | Wait and retry. |
| 500 | `INTERNAL` | We could not validate our own response. | Retry once. Report the `requestId`. |

Errors that do not come from the API (no JSON body from us):

- `429` with an HTML body: our edge limit per client IP (10 requests per second, burst 20).
- `403` with an HTML body: a method other than `GET`/`HEAD` (stopped by our proxy).
- `502` / `504`: the proxy could not reach the API in time (it waits 15 s).
- Cloudflare error pages (for example 520-524).

Treat any non-JSON body as a temporary failure.

## 5. Limits

| Limit | Default |
| --- | --- |
| Per key, `/price` | 5 requests per second, burst 10 |
| Per key, `/quote` | 2 requests per second, burst 5 |
| Per key, `/tokens` + `/health` | together 5 per second, burst 10 |
| Per client IP (edge) | 10 per second, burst 20 |
| All partners together | `/price` 10 per second, `/quote` 4 per second; a small number of requests at the same time |
| Time to answer | `/price` up to about 9 s, `/quote` up to about 14 s (p50 today: about 1–2 s in the saved records) |

We can change your key's limits. Ask us. The limit on parallel requests is low today; a request over it
gets `503 SERVICE_BUSY` at once (no queue). Send requests one after another, and use `Retry-After`.

## 6. Quote expiry

- `expiresAt` = the earliest of: 30 s after our simulation, 30 s after the route was priced, and the
  earliest deadline of a market maker on the route.
- After `expiresAt`, do not send the transaction. Get a new quote.
- A slow signing flow (hardware wallet, multisig) needs a new quote when the old one expires.

## 7. Gas

| Field | What it is | Use it for |
| --- | --- | --- |
| `gas` | Expected gas used by the swap (our simulation). | Ranking quotes: output minus `gas` × your gas price. |
| `gasPrice` | Our gas price estimate for this quote (wei per gas). | Ranking, if you have no gas price of your own. |
| `totalNetworkFee` | `gas` × `gasPrice` (wei). | The expected network cost. The real cost on the receipt can differ. |
| `transaction.gas` | The gas LIMIT for the transaction. | Send it unchanged. |
| `transaction.gasPrice` | The fee cap (2 × base fee + priority fee at quote time). | `maxFeePerGas` of an EIP-1559 transaction. |

Rules for `transaction.gas`:

- Most routes: `eth_estimateGas` of the taker's exact transaction × 1.3 + 50,000 (never less than the limit
  that our own web app would use).
- Routes through a pool that behaves differently when less gas is available (some Uniswap v4 hooks):
  `transaction.gas` is the chain maximum, **32,000,000**. With a lower limit such a pool can give less
  output or revert. Do not lower the limit, and do not replace it with your own `eth_estimateGas`.
- Only the gas used is charged. Example: a swap with `transaction.gas` 263,503 used 160,348 gas.
- But the wallet must hold `transaction.gas` × fee cap + `transaction.value` to send the transaction.
  At a fee cap of 40,088,000 wei, a 32,000,000 limit needs about 0.00128 ETH of gas funds (plus the value).
- We simulate the transaction at the limit that we return.

## 8. Fees and surplus

### 8.1 Partner fee (fee codes)

- For key kind `fee`, choose a strictly positive fee R: 0% < R ≤ 2%. We register it on your referral code.
- A zero-fee partner uses key kind `surplus` or `attribution`, as appropriate. It must not use kind `fee`.
- **The user pays R.** `buyAmount` and `minBuyAmount` are already net of R.
- **You receive 80% of R.** FlowRouter keeps 20%. The router pays your share to your beneficiary address in
  the same transaction.
- The fee is taken from the output token.

Response fields when your code has a fee:

```json
"fees": {
  "integratorFee": { "type": "volume", "token": "<buyToken>", "amount": "9677", "bps": "25", "beneficiaryShare": "0.8" },
  "flowRouterFee": { "type": "volume", "token": "<buyToken>", "amount": "2420" }
}
```

- `integratorFee.bps` = R in basis points, as an exact decimal string (for example `"25"` or `"12.5"`).
- `integratorFee.amount` = your 80% share, computed on the quoted gross output.
- `flowRouterFee.amount` = gross − `buyAmount` − your share.
- These amounts are for the quoted output. On chain the fee is taken from the real output, so it can be
  higher when the swap does better than the quote.
- The example above is computed with the API's formula for R = 25 bps and a gross output of 4,838,627 (not a
  captured response): `buyAmount` 4,826,530, your share 9,677, FlowRouter 2,420.

We read your code's fee from chain every 5 minutes. If we cannot read it for 15 minutes, your `/price`
and `/quote` requests get `503 SERVICE_UNAVAILABLE` until a read succeeds. We never quote with a fee
that we did not read from chain.

### 8.2 Surplus share (zero-fee codes)

A partner without a fee can get a share of the positive surplus instead. Example: DefiLlama Swap, 50%.
The router pays the share to your surplus address in the same transaction. The `fees` field is always an object.
Both fee fields are null for zero-fee `surplus` and `attribution` codes:

```json
{"fees":{"integratorFee":null,"flowRouterFee":null}}
```
A code cannot have both a fee and a surplus share.

### 8.3 Positive surplus

If the swap gives more than `buyAmount`, the taker receives exactly `buyAmount`. The surplus goes to the
protocol, or is split with a zero-fee partner by its on-chain surplus share. `buyAmount` is the most the
taker receives. `minBuyAmount` is the least.

## 9. Send the swap

1. Get a quote with `taker` = the user's address.
2. Check `transaction.to` = `0x8Ad60fAE44DD6aaFD8A98B7f25EaD026013efdF2`.
3. If `sellToken` is an ERC-20 and `issues.allowance` is not null: approve `allowanceTarget` for at least
   `sellAmount`. If the quote expires while the approval confirms, get a new quote.
4. Send `{ to, data, value, gas: transaction.gas }` from `taker`, before `expiresAt`. Use
   `transaction.gasPrice` as the fee cap (EIP-1559 `maxFeePerGas`). On the captured swap the receipt's
   effective gas price was the base fee (20,000,000 wei) with a fee cap of 40,088,000 wei.
5. Do not change `data` or `value`. Do not lower `gas`.

## 10. Examples

The responses below were captured from the API on 2026-10-16/17 (no key inside). `transaction.data` is shortened.
Authenticated commands below are **HUMAN OPERATOR ONLY**. The header-file path is a placeholder. An agent must
not create, inspect, use or clean up that file. This package contains no credential or header file.

### 10.1 Price: 0.001 ETH → USDG

```bash
curl -s -H @/operator-only/flowrouter.hdr \
  'https://api.flowrouter.xyz/v1/4663/price?sellToken=0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE&buyToken=0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168&sellAmount=1000000000000000&slippageBps=50'
```

```json
{
  "allowanceTarget": null,
  "blockNumber": "90450199",
  "buyAmount": "2422869",
  "buyToken": "0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168",
  "chainId": 4663,
  "fees": { "flowRouterFee": null, "integratorFee": null },
  "gas": "182211",
  "gasPrice": "20272000",
  "indicative": true,
  "liquidityAvailable": true,
  "minBuyAmount": "2410754",
  "requestId": "ce48246a-e7d5-4114-9793-cfb1f883da59",
  "route": { "fills": [
    { "from": "native", "proportionBps": 10000, "source": "uniswap-v4", "to": "0x5fc5360d0400a0fd4f2af552add042d716f1d168" }
  ] },
  "sellAmount": "1000000000000000",
  "sellToken": "0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE",
  "totalNetworkFee": "3693781392000"
}
```

### 10.2 Quote: 0.002 ETH → USDG (this quote was sent on chain)

```bash
curl -s -H @/operator-only/flowrouter.hdr \
  'https://api.flowrouter.xyz/v1/4663/quote?sellToken=0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE&buyToken=0x5fc5360d0400a0fd4f2af552add042d716f1d168&sellAmount=2000000000000000&slippageBps=50&taker=0xe911f518449ba0011d84b047b4cde50daa081ec1'
```

```json
{
  "allowanceTarget": null,
  "blockNumber": "90572140",
  "buyAmount": "4838627",
  "buyToken": "0x5fc5360d0400a0fd4f2af552add042d716f1d168",
  "chainId": 4663,
  "expiresAt": 1792203723810,
  "fees": { "flowRouterFee": null, "integratorFee": null },
  "gas": "160355",
  "gasPrice": "20022000",
  "issues": { "allowance": null, "balance": null, "invalidSourcesPassed": [], "simulationIncomplete": false },
  "liquidityAvailable": true,
  "minBuyAmount": "4814433",
  "quoteId": "0x237f283af8c7cedc2c553e36eeed8c3ab9c2f1554d5754790498b0644de6f519",
  "quoteValidUntil": 1792203723810,
  "requestId": "e3a24c8c-e0d4-4750-8a45-fb7c53f4e975",
  "route": { "fills": [
    { "from": "native", "proportionBps": 10000, "source": "uniswap-v4", "to": "0x5fc5360d0400a0fd4f2af552add042d716f1d168" }
  ] },
  "sellAmount": "2000000000000000",
  "sellToken": "0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE",
  "simulation": { "blockNumber": "90572162", "funding": "canonical_overrides", "status": "success" },
  "totalNetworkFee": "3210627810000",
  "transaction": {
    "data": "0xd46cadbc…",
    "gas": "263503",
    "gasPrice": "40088000",
    "to": "0x8Ad60fAE44DD6aaFD8A98B7f25EaD026013efdF2",
    "value": "2000000000000000"
  }
}
```

On chain (tx `0xd8560ab493ca97fdd5798a273e2ba55ae270e5d4ce50e0829d7a1d6403b327fb`, block 90,572,206): status 1,
gas used 160,348 (quote `gas` 160,355), effective gas price 20,000,000 wei, received 4,838,627 = `buyAmount`.

### 10.3 Quote: 5 USDG → ETH (ERC-20 sell, three steps)

Captured on 2026-10-16 with an OLDER build. In that build `gas`, `gasPrice` and `totalNetworkFee` had another
meaning (gas = `eth_estimateGas`, price = fee cap). In the documented source, `gas` is the simulated gas used, and `gasPrice` is
on the same basis as `/price`. The other fields have the same meaning today.

```json
{
  "allowanceTarget": "0x8Ad60fAE44DD6aaFD8A98B7f25EaD026013efdF2",
  "buyAmount": "2083281964837324",
  "buyToken": "0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE",
  "minBuyAmount": "2072865555013137",
  "route": { "fills": [
    { "from": "0x5fc5360d0400a0fd4f2af552add042d716f1d168", "proportionBps": 10000, "source": "uniswap-v4", "to": "0x05b37fb53a299a1b874a619e1c4c404d52c36f4c" },
    { "from": "0x05b37fb53a299a1b874a619e1c4c404d52c36f4c", "proportionBps": 10000, "source": "uniswap-v3", "to": "0x0bd7d308f8e1639fab988df18a8011f41eacad73" },
    { "from": "0x0bd7d308f8e1639fab988df18a8011f41eacad73", "proportionBps": 10000, "source": "wrapped-native", "to": "native" }
  ] },
  "sellAmount": "5000000",
  "sellToken": "0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168",
  "transaction": { "data": "0xd46cadbc…", "to": "0x8Ad60fAE44DD6aaFD8A98B7f25EaD026013efdF2", "value": "0" }
}
```

### 10.4 Error (shape from the code, not captured)

```bash
curl -s -i 'https://api.flowrouter.xyz/v1/health'
```

```
HTTP/2 401
x-request-id: 2b0f…
{"code":"UNAUTHORIZED","message":"A valid X-API-Key header is required.","requestId":"2b0f…"}
```

### 10.5 JavaScript (Node 18+, human-operated server only)

**HUMAN OPERATOR ONLY:** This example uses a server credential. Agents must not run it or read the credential.
The setting shown is an example for the partner; this workspace does not authorize any agent to load it.

```js
const API = "https://api.flowrouter.xyz";
const ROUTER = "0x8ad60fae44dd6aafd8a98b7f25ead026013efdf2";
const NATIVE = "0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE";

async function flowrouter(path, params) {
  const url = `${API}${path}?${new URLSearchParams(params)}`;
  const response = await fetch(url, { headers: { "X-API-Key": process.env.FLOWROUTER_API_KEY } });
  const text = await response.text();
  let body;
  try { body = JSON.parse(text); } catch { throw new Error(`HTTP ${response.status} (not from the API)`); }
  if (!response.ok) {
    const error = new Error(`${body.code}: ${body.message} (requestId ${body.requestId})`);
    error.retryAfter = Number(response.headers.get("retry-after")) || undefined;
    throw error;
  }
  return body;
}

// 1. Price (no wallet yet)
const price = await flowrouter("/v1/4663/price", {
  sellToken: NATIVE, buyToken: "0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168",
  sellAmount: "1000000000000000", slippageBps: "50",
});

// 2. Quote for the user, then send it with the user's wallet (ethers v6 signer here)
const quote = await flowrouter("/v1/4663/quote", {
  sellToken: NATIVE, buyToken: "0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168",
  sellAmount: "1000000000000000", slippageBps: "50", taker: userAddress,
});
if (quote.transaction.to.toLowerCase() !== ROUTER) throw new Error("unexpected router");
if (quote.allowanceTarget !== null && quote.allowanceTarget.toLowerCase() !== ROUTER) throw new Error("unexpected spender");
if (quote.chainId !== 4663 || !signer.provider) throw new Error("unexpected chain");
const from = await signer.getAddress();
const network = await signer.provider.getNetwork();
if (from.toLowerCase() !== userAddress.toLowerCase() || network.chainId !== 4663n) throw new Error("wallet changed: request a new quote");
const request = {
  from,
  chainId: 4663,
  to: quote.transaction.to,
  data: quote.transaction.data,
  value: BigInt(quote.transaction.value),
  gasLimit: BigInt(quote.transaction.gas),          // send as it is: can be 32,000,000
  maxFeePerGas: BigInt(quote.transaction.gasPrice), // the fee cap
};
if (!Number.isSafeInteger(quote.expiresAt) || Date.now() >= quote.expiresAt) throw new Error("quote expired: request a new one");
const tx = await signer.sendTransaction(request);
// If wallet signing can cross expiry, the human-operated send flow must check it again before broadcast.
```

## 11. Not in v1

Exact output (buy a fixed amount); a recipient other than the taker; `rfq=false` (on-chain liquidity only);
excluded sources; Permit2 quotes; a keyless tier; CORS.

## 12. What we could not verify

- The live host configuration was not read for this guide. The limits in section 5 are the code defaults
  and the decisions on record (D-793: at most 1 price and 1 quote at the same time, 2 in total). Your key
  can have other limits.
- `gasPrice`: in the code it is the solver's gas price when the solver returns an execution envelope. Without
  one, the code falls back to the fee cap (2 × base fee + priority fee). All captured responses show the
  solver basis (about the base fee). We did not prove that every route returns an envelope.
- All captured responses come from our attribution test key. Each has
  `fees: {integratorFee: null, flowRouterFee: null}`. The fee example in 8.1 is computed with the code's
  formula, not captured. No live fee/surplus-key response or settlement has been added.
- The error example in 10.4 shows the shape from the code. It was not captured for this guide.
- Example 10.3 is from an older build (see its note). We have no saved USDG → ETH quote from the documented source build.
- The token count (5,676) is from one call on 2026-10-16 (D-787). It changes.
