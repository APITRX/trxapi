[English](node-api.md) | **简体中文**

# 波场节点 API

> 在线版：https://www.trxapi.io/docs/node.html · 接口版本 v1

直连我们的 TRON 节点查询链上数据：区块、账户、交易、资源、合约。这是一个**透明代理**——请求原样转发，响应格式与 java-tron 标准 HTTP API 完全一致，按标准接口编写的代码只需替换域名并附上 apikey。

- **免费开放**：注册客户每天百万次请求、每秒 5 次限流，由金融科技机构 CoinePay 赞助
- **两类端点**：`/node/wallet/{method}`（fullnode，最新数据）与 `/node/walletsolidity/{method}`（solidity，已固化不可逆数据）
- **GET / POST 均支持**：JSON body 原样转发

## 关于节点

为你服务的并非普通自建节点：本节点由金融科技机构 CoinePay 运营，是 TRON 链上注册的**验证器（见证人）节点**，已进入全网前 127 名的超级代表合伙人行列。节点地址 `TCoinepWiWHaeyB7tCnebrmuB9k1WeJrhu`，可在 [Tronscan](https://tronscan.org/#/address/TCoinepWiWHaeyB7tCnebrmuB9k1WeJrhu) 查验。CoinePay 平台自身钱包的入账核验同样运行在本节点上——稳定性与安全性经受真实资金业务 7×24 考验。

> 🤝 **生态共建**：本节点由 CoinePay 赞助、面向生态免费开放。如果它对你有帮助——请善用免费额度、遵守限流规则，不要恶意刷量占用资源，把带宽留给真正需要的开发者；已质押 TRX、手中有选票的用户，欢迎在 [Tronscan](https://tronscan.org/#/sr/votes) 为我们的见证人节点 `TCoinepWiWHaeyB7tCnebrmuB9k1WeJrhu`（coinepay）投上一票。链上激励将持续投入免费额度与生态维护。

## 快速开始

```bash
# 把标准节点接口路径中的域名替换为本服务，并附加 apikey：
curl "https://api.trxapi.io/api/v1/openapi/node/wallet/getnowblock?apikey=YOUR_API_KEY"

# POST（JSON 原样转发）：solidity 端查账户
curl -X POST "https://api.trxapi.io/api/v1/openapi/node/walletsolidity/getaccount?apikey=YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"address":"TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t","visible":true}'
```

`{method}` 为 java-tron 标准 HTTP API 接口名（路径最后一段）。完整清单以 [TRON 官方开发者文档](https://developers.tron.network/)为准。

## fullnode 还是 solidity？

- 查**最新状态**（余额、资源、最新区块）→ `wallet`（fullnode）
- 判断**交易是否已不可逆**（入账确认、对账）→ `walletsolidity`（solidity）
- **资金相关场景务必以 solidity 数据为准**

## 常用方法参考

### 区块

| method | HTTP | 说明 |
|---|---|---|
| `getnowblock` | GET/POST | 最新区块 |
| `getblockbynum` | POST | 按高度取块（`{"num": 12345}`） |
| `getblockbyid` | POST | 按区块哈希取块 |
| `getblockbylatestnum` | POST | 最近 N 个区块 |

### 账户与资源

| method | HTTP | 说明 |
|---|---|---|
| `getaccount` | POST | 账户信息与 TRX 余额（`{"address":"T...","visible":true}`） |
| `getaccountresource` | POST | 能量/带宽余量——监控能量到账常用 |
| `getaccountnet` | POST | 带宽使用情况 |
| `getdelegatedresourcev2` | POST | 两地址间资源委托关系（Stake 2.0） |
| `validateaddress` | POST | 校验地址格式 |

### 交易

| method | HTTP | 说明 |
|---|---|---|
| `gettransactionbyid` | POST | 按哈希查交易（`{"value":"txid"}`） |
| `gettransactioninfobyid` | POST | 执行结果、所在区块、能量消耗——核验订单 txid 常用 |
| `gettransactioninfobyblocknum` | POST | 整块交易执行信息 |
| `broadcasttransaction` | POST | 广播本地已签名交易 |
| `broadcasthex` | POST | 按 hex 原文广播 |
| `createtransaction` | POST | 构造未签名 TRX 转账（本地签名后广播） |

### 合约与链参数

| method | HTTP | 说明 |
|---|---|---|
| `triggerconstantcontract` | POST | 只读调合约（免费）——查 TRC20 余额常用 |
| `triggersmartcontract` | POST | 构造合约调用交易 |
| `getcontract` | POST | 合约信息与 ABI |
| `getchainparameters` | GET/POST | 链参数（含能量单价） |
| `getenergyprices` | GET/POST | 历史能量单价 |
| `listwitnesses` | GET/POST | 全部见证人（可看到我们的节点） |
| `getnodeinfo` | GET/POST | 节点运行状态 |

## 常见用例

1. **充值入账确认**：solidity 端 `gettransactioninfobyid` 查到即不可逆；TRC20 需同时确认 `receipt.result` 为 SUCCESS。批量对账用 `getblockbynum` + `gettransactioninfobyblocknum` 按块遍历
2. **能量到账监控**（配合能量 API）：`getaccountresource` 看 `EnergyLimit` / `EnergyUsed`；订单 txid 走 `gettransactioninfobyid` 留存凭证
3. **TRC20 余额查询**：`triggerconstantcontract` 只读调 `balanceOf(address)`；转账明细从 `gettransactioninfobyid` 的 `log`（Transfer 事件）解析
4. **自托管钱包/代付流水线**：`createtransaction` 或 `triggersmartcontract` 构造 → 本地签名 → `broadcasttransaction`。私钥不离开你的服务器
5. **成本测算**：`getchainparameters` / `getenergyprices` 拿实时能量单价

## 限流与错误处理

- **成功**：返回节点原生响应，原样转发、零改写，无额外信封
- **429**：超出每秒频率或当日总量——指数退避（1s → 2s → 4s），检查失控轮询；TRON 约 3 秒一个块，轮询快于 3 秒无意义
- **501**：apikey 缺失、无效或被禁用
- 非法方法/参数的节点报错原样返回

> 多数 SDK（如 TronWeb）会在 fullHost 后自行拼接路径、无法携带 query 鉴权参数——建议直接调用 HTTP 端点，或在你的服务端加一层转发注入 apikey：

```js
// Node.js 18+ 轻量封装
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

提额申请与技术支持：admin@coinepay.cc
