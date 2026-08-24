**English** | [简体中文](README.zh-CN.md)

# TRXAPI Developer Documentation

TRON Energy Delegation & Node API Service · [www.trxapi.io](https://www.trxapi.io/) · Operated by Coinepay Pte. Ltd., with technology and ecosystem maintenance by the fintech firm CoinePay.

| Document | Description |
|---|---|
| [Energy API](energy-api.md) | Rent energy, address activation, order & balance queries, webhook callbacks with signature verification |
| [TRON Node API](node-api.md) | Query on-chain data straight from the TRON validator node operated by CoinePay (transparent proxy) |
| [openapi-energy.yaml](openapi-energy.yaml) | OpenAPI 3.0 spec of the Energy API — import into Postman / Apifox |

## Quickstart

```bash
# 1) Rent 65,000 energy for 1 hour (create your apikey at https://login.trxapi.io/)
curl "https://api.trxapi.io/api/v1/openapi/getenergy?apikey=YOUR_API_KEY&add=TR7NHqjeKQx...&value=65000&hour=1"

# 2) Poll the order until a terminal state (or configure the webhook)
curl "https://api.trxapi.io/api/v1/openapi/order?apikey=YOUR_API_KEY&orderId=10001"

# 3) Free node queries (1M requests/day at 5 req/s)
curl "https://api.trxapi.io/api/v1/openapi/node/wallet/getnowblock?apikey=YOUR_API_KEY"
```

## Conventions

- **Base URL**: `https://api.trxapi.io`
- **Auth**: `?apikey=YOUR_API_KEY` URL parameter (keep keys server-side; never leak full request URLs)
- **Responses**: business endpoints use the JSON envelope `{code, message, data}` — branch on `code` (200 = success); node endpoints return the node-native response verbatim
- **Official domains**: only `www.trxapi.io` / `login.trxapi.io` / `api.trxapi.io`
- **Security**: we never need your private key, seed phrase, or wallet password

## Contact

- Support / partnerships: admin@coinepay.cc
- Online docs: [Energy API](https://www.trxapi.io/docs/energy.html) · [Node API](https://www.trxapi.io/docs/node.html) · [Security guide](https://www.trxapi.io/security.html)
