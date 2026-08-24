[English](energy-api.md) | **简体中文**

# 能量 API

> 在线版：https://www.trxapi.io/docs/energy.html · 接口版本 v1

TRXAPI 能量 API 是一组 REST 接口，用于把 TRON 能量委托集成到你的系统：交易所提币通道、支付商户代付、批量转账工具等。核心流程：**下单（getenergy）→ 轮询或等待回调（order / webhook）→ 对账（balance）**。每个订单返回链上交易哈希 `txid`，可在任意区块浏览器独立核验。

- 计费：从**预付余额**按单扣费（客户中心充值）；订单最终失败自动退款
- 能量规格：`65000` / `130000`；时长：`1` / `24` 小时

## 通用约定

- 接口域名 `https://api.trxapi.io`，鉴权 `?apikey=YOUR_API_KEY`
- 统一响应信封（**以 `code` 判断结果，不要依赖 message 文案**）：

```jsonc
{
  "code": 200,          // 200 = 成功，其余为错误码
  "message": "SUCCESS",
  "data": { /* 各接口负载 */ },
  "traceId": "7c2c1d6f..."
}
```

- TRON 地址为 Base58 格式（`T` 开头 34 位）；金额字段为字符串，单位 TRX
- 业务接口均为 `GET` + query string；URL 含 `&`，shell 调用务必给整个 URL 加引号

## 接口总览

| 接口 | 方法 | 说明 |
|---|---|---|
| `/api/v1/openapi/getenergy` | GET | 下单租能量（65000/130000，1h/24h） |
| `/api/v1/openapi/activation` | GET | 激活从未使用过的 TRON 地址 |
| `/api/v1/openapi/order` | GET | 按 orderId 查询订单状态与 txid |
| `/api/v1/openapi/balance` | GET | 查询商户预付余额 |
| `/api/v1/openapi/node/…` | GET/POST | 节点直连查询，免费（见[节点 API](node-api.zh-CN.md)） |
| Webhook 回调 | POST（服务器 → 你） | 订单终态主动通知，客户中心配置 |

## 下单租能量

`GET /api/v1/openapi/getenergy`

| 参数 | 必填 | 说明 |
|---|---|---|
| `apikey` | 是 | 商户密钥，客户中心创建 |
| `add` | 是 | 接收能量的 TRON 地址 |
| `value` | 是 | 能量数量，仅 65000 或 130000 |
| `hour` | 是 | 时长（小时），仅 1 或 24 |
| `traceId` | 否 | 链路追踪 ID（≤36 位），原样回显便于对账 |

响应 `data`：`orderId`（订单号）、`amount`（本单扣费 TRX）、`balance`（扣费后余额）、`txid`（链上委托哈希）。

```bash
curl "https://api.trxapi.io/api/v1/openapi/getenergy?apikey=YOUR_API_KEY&add=TR7NHqjeKQx...&value=65000&hour=1"
```

> ⚠️ 接收地址必须已激活（有过至少一笔链上交易），否则请先调用地址激活接口。

## 地址激活

`GET /api/v1/openapi/activation` — 参数 `apikey`、`add`。返回 `add` / `txid` / `amount`。已激活地址返回 `702001`（不扣费，可安全重复调用）。

## 查询订单

`GET /api/v1/openapi/order` — 参数 `apikey`、`orderId`。状态：`processing`（处理中）/ `completed`（已完成）/ `failed`（已失败，自动退款）。建议每 2–3 秒轮询直至终态，或改用 Webhook。

## 查询余额

`GET /api/v1/openapi/balance` — 参数 `apikey`。返回 `balance`（预付余额，TRX 计价）、`currency`。该余额是你在 TRXAPI 的服务账户余额，与外部钱包资产无关。

## 订单回调（Webhook）

订单进入终态时，服务器向你在客户中心配置的回调地址 POST 通知。回调为**账户级**：所有 API Key 共用同一地址与签名密钥。

**Body**：`event`（order.completed / order.failed）、`orderId`、`status`、`txid`（失败为空）、`timestamp`（Unix 秒）。

**请求头**：`X-Timestamp`（发送时间戳）、`X-Signature`（`sha256=<hex(HMAC-SHA256)>`）。

**验签**（务必用原始请求体 rawBody）：

```text
sig = "sha256=" + hex(hmac_sha256(secret, ts + "." + rawBody))
```

```python
# Python
import hmac, hashlib

def verify(secret: str, ts: str, raw_body: bytes, header_sig: str) -> bool:
    mac = hmac.new(secret.encode(), f"{ts}.".encode() + raw_body, hashlib.sha256)
    return hmac.compare_digest("sha256=" + mac.hexdigest(), header_sig)
```

```js
// Node.js（用 raw body，不要用重新序列化的 JSON）
const crypto = require('crypto')
function verify(secret, ts, rawBody, headerSig) {
  const expected = 'sha256=' + crypto.createHmac('sha256', secret).update(ts + '.' + rawBody).digest('hex')
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(headerSig))
}
```

**重试**：HTTP 2xx 视为成功；失败按 1m / 5m / 30m / 2h / 6h 退避重试，最多 6 次。请校验时间戳偏差（如 5 分钟内）防重放，并按 orderId 幂等处理。

## 错误码

| code | 说明 |
|---|---|
| **200** | 成功，结果见 data |
| 500 | 参数错误 / 下单失败 / 服务暂不可用，稍后重试 |
| 501 | apikey 不存在或已禁用 |
| 502 | 预付余额不足，请充值 |
| 429 | 超出限额（仅节点直连 API） |
| 702001 | 地址已激活（不扣费） |
| 100002 | 委托通道繁忙，稍后重试（不扣费，已扣自动退回） |

## 最佳实践

- **traceId 对账**：用你系统的唯一业务号做 traceId，响应原样回显
- **谨防重复下单**：GET 请求超时后盲目重发可能重复扣费——以业务单号加锁，超时先查单再重下
- **重试克制**：100002 等数秒再试；500 先查参数
- **下单前校验地址**：格式校验 + 新地址先走激活接口
- **余额告警**：定时查余额，低于阈值提醒充值，避免高峰期 502 中断
- **密钥安全**：apikey 只放服务端、环境变量注入；Webhook 必须验签 + 幂等

## 代码示例（下单 → 轮询）

```python
import time, requests

BASE = "https://api.trxapi.io/api/v1/openapi"
KEY  = "YOUR_API_KEY"

def rent(address, value=65000, hour=1, trace=None):
    params = {"apikey": KEY, "add": address, "value": value, "hour": hour}
    if trace: params["traceId"] = trace
    r = requests.get(f"{BASE}/getenergy", params=params, timeout=15).json()
    assert r["code"] == 200, r
    return r["data"]["orderId"]

def wait_done(order_id, timeout=60):
    deadline = time.time() + timeout
    while time.time() < deadline:
        r = requests.get(f"{BASE}/order", params={"apikey": KEY, "orderId": order_id}, timeout=15).json()
        if r["code"] == 200 and r["data"]["status"] in ("completed", "failed"):
            return r["data"]
        time.sleep(2)
    raise TimeoutError(order_id)

order_id = rent("TR7NHqjeKQx...", trace="withdraw-8801")
print(wait_done(order_id))
```

机器可读定义见 [openapi-energy.yaml](openapi-energy.yaml)。技术支持：admin@coinepay.cc
