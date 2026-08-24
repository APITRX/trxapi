**English** | [简体中文](node-api.zh-CN.md)

# TRON Node API

> Online version: https://www.trxapi.io/docs/node.html · API version v1

Query on-chain data straight from our TRON node: blocks, accounts, transactions, resources, contracts. This is a **transparent proxy** — requests are forwarded verbatim and responses match the standard java-tron HTTP API exactly, so code written against the standard endpoints works by swapping the host and appending your apikey.

- **Free**: registered customers get 1M requests/day at 5 req/s, sponsored by the fintech firm CoinePay
- **Two endpoint families**: `/node/wallet/{method}` (fullnode, latest data) and `/node/walletsolidity/{method}` (solidity, finalized irreversible data)
- **GET / POST both work**: JSON bodies forwarded verbatim

## About the node

The node serving you is no ordinary self-hosted node: it is operated by the fintech firm CoinePay as a **validator (witness) registered on TRON**, ranked within the top 127 as a Super Representative Partner. Node address `TCoinepWiWHaeyB7tCnebrmuB9k1WeJrhu` — verify it on [Tronscan](https://tronscan.org/#/address/TCoinepWiWHaeyB7tCnebrmuB9k1WeJrhu). Deposit verification for CoinePay's own platform wallets runs on this very node — its stability and security are proven 24/7 by real money flows.

## Quickstart

```bash
# Replace the host in any standard node endpoint path and append your apikey:
curl "https://api.trxapi.io/api/v1/openapi/node/wallet/getnowblock?apikey=YOUR_API_KEY"

# POST (JSON forwarded verbatim): account on the solidity node
curl -X POST "https://api.trxapi.io/api/v1/openapi/node/walletsolidity/getaccount?apikey=YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"address":"TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t","visible":true}'
```

`{method}` is the standard java-tron HTTP API endpoint name (the last path segment). The [official TRON developer docs](https://developers.tron.network/) are authoritative for the full list.

## fullnode or solidity?

- **Latest state** (balances, resources, newest block) → `wallet` (fullnode)
- **Irreversibility** (deposit confirmation, reconciliation) → `walletsolidity` (solidity)
- **Anything money-related must trust the solidity data**

## Common methods

### Blocks

| method | HTTP | Description |
|---|---|---|
| `getnowblock` | GET/POST | Latest block |
| `getblockbynum` | POST | Block by height (`{"num": 12345}`) |
| `getblockbyid` | POST | Block by hash |
| `getblockbylatestnum` | POST | Most recent N blocks |

### Accounts & resources

| method | HTTP | Description |
|---|---|---|
| `getaccount` | POST | Account info and TRX balance (`{"address":"T...","visible":true}`) |
| `getaccountresource` | POST | Energy / bandwidth remaining — the go-to for watching delegated energy arrive |
| `getaccountnet` | POST | Bandwidth usage |
| `getdelegatedresourcev2` | POST | Resource delegation between two addresses (Stake 2.0) |
| `validateaddress` | POST | Validate an address format |

### Transactions

| method | HTTP | Description |
|---|---|---|
| `gettransactionbyid` | POST | Transaction by hash (`{"value":"txid"}`) |
| `gettransactioninfobyid` | POST | Execution result, containing block, energy usage — the go-to for verifying order txids |
| `gettransactioninfobyblocknum` | POST | Execution info for a whole block |
| `broadcasttransaction` | POST | Broadcast a locally signed transaction |
| `broadcasthex` | POST | Broadcast signed raw hex |
| `createtransaction` | POST | Build an unsigned TRX transfer (sign locally, then broadcast) |

### Contracts & chain parameters

| method | HTTP | Description |
|---|---|---|
| `triggerconstantcontract` | POST | Read-only contract call (free) — common for TRC20 balances |
| `triggersmartcontract` | POST | Build a contract-call transaction |
| `getcontract` | POST | Contract info and ABI |
| `getchainparameters` | GET/POST | Chain parameters (incl. energy price) |
| `getenergyprices` | GET/POST | Historical energy prices |
| `listwitnesses` | GET/POST | All witnesses (you can find our node here) |
| `getnodeinfo` | GET/POST | Node runtime status |

## Use cases

1. **Deposit confirmation**: a txid returned by the solidity node's `gettransactioninfobyid` is irreversible; for TRC20 also check `receipt.result` is SUCCESS. Batch-reconcile by walking heights with `getblockbynum` + `gettransactioninfobyblocknum`
2. **Energy delivery monitoring** (pairs with the Energy API): check `EnergyLimit` / `EnergyUsed` via `getaccountresource`; keep the order txid receipt via `gettransactioninfobyid`
3. **TRC20 balance checks**: read-only `balanceOf(address)` via `triggerconstantcontract`; parse transfer details from the `log` (Transfer events) in `gettransactioninfobyid`
4. **Self-custodial wallet / payout pipeline**: build with `createtransaction` or `triggersmartcontract` → sign locally → `broadcasttransaction`. Keys never leave your servers
5. **Cost models**: live energy prices via `getchainparameters` / `getenergyprices`

## Limits & error handling

- **Success**: the node-native response is returned verbatim — no extra envelope
- **429**: per-second rate or daily quota exceeded — back off exponentially (1s → 2s → 4s) and check for runaway polling; TRON produces a block ~every 3s, so polling faster is pointless
- **501**: apikey missing, invalid, or disabled
- Node errors for invalid methods/params are returned verbatim

> Most SDKs (TronWeb included) append their own path after fullHost and cannot carry the query auth parameter — call the HTTP endpoints directly, or add a small server-side proxy that injects the apikey:

```js
// Node.js 18+ minimal wrapper
const BASE = 'https://api.trxapi.io/api/v1/openapi/node'
const KEY = process.env.TRXAPI_KEY

async function node(path, body) {
  const res = await fetch(`${BASE}/${path}?apikey=${KEY}`, body ? {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  } : undefined)
  return res.json()
}

const block = await node('wallet/getnowblock')
const acct = await node('walletsolidity/getaccount', { address: 'TR7NHqjeKQx...', visible: true })
```

Quota increases & support: admin@coinepay.cc
