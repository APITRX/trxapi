[English](README.md) | **简体中文**

# TRXAPI 开发者文档

TRON 能量委托与节点 API 服务 · [www.trxapi.io](https://www.trxapi.io/) · 由 Coinepay Pte. Ltd. 运营，金融科技机构 CoinePay 提供技术服务与生态维护。

| 文档 | 说明 |
|---|---|
| [能量 API](energy-api.zh-CN.md) | 下单租能量、地址激活、订单查询、余额查询、Webhook 回调与验签 |
| [波场节点 API](node-api.zh-CN.md) | 直连 CoinePay 运营的 TRON 验证器节点查询链上数据（透明代理） |
| [openapi-energy.yaml](openapi-energy.yaml) | 能量 API 的 OpenAPI 3.0 描述，可导入 Postman / Apifox |

## 快速开始

```bash
# 1) 下单租 65000 能量 1 小时（apikey 在客户中心 https://login.trxapi.io/ 创建）
curl "https://api.trxapi.io/api/v1/openapi/getenergy?apikey=YOUR_API_KEY&add=TR7NHqjeKQx...&value=65000&hour=1"

# 2) 轮询订单直至终态（或配置 Webhook）
curl "https://api.trxapi.io/api/v1/openapi/order?apikey=YOUR_API_KEY&orderId=10001"

# 3) 免费节点查询（每天百万次 · 每秒 5 次）
curl "https://api.trxapi.io/api/v1/openapi/node/wallet/getnowblock?apikey=YOUR_API_KEY"
```

## 基本约定

- **接口域名**：`https://api.trxapi.io`
- **鉴权**：URL 参数 `?apikey=YOUR_API_KEY`（密钥只放服务端，勿泄露完整请求 URL）
- **响应**：业务接口统一 JSON 信封 `{code, message, data}`，以 `code` 判断结果（200 = 成功）；节点接口原样返回节点原生响应
- **官方域名**：仅 `www.trxapi.io` / `login.trxapi.io` / `api.trxapi.io` 三个
- **安全**：我们永远不需要你的私钥、助记词或钱包密码

## 联系

- 技术支持 / 商务：admin@coinepay.cc
- 在线文档：[能量 API](https://www.trxapi.io/docs/energy.html) · [节点 API](https://www.trxapi.io/docs/node.html) · [安全说明](https://www.trxapi.io/security.html)
