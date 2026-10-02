# 开放 API

StableMesh **开放 API** 让你从自己的服务器上发行和管理 Visa / Mastercard 虚拟卡。这是一套服务器到服务器的 REST API，用你**在仪表盘里自助创建的 API Key** 认证。

**API 和仪表盘是同一个账户。** 通过 API 开的卡会出现在仪表盘里，仪表盘里开的卡 API 也能看到；余额、卡状态、交易记录两边完全一致，不需要单独充值或对账。

***

## 基础地址

| 环境     | API 地址                                     | 仪表盘                                            |
| -------- | -------------------------------------------- | ------------------------------------------------- |
| 生产环境 | `https://api.stablemesh.io`                  | `https://app.stablemesh.io`                       |
| 沙盒     | `https://api-server-dev-1440.up.railway.app` | `https://stablemesh-dashboard-dev.up.railway.app` |

沙盒和生产完全独立：账户、API Key、余额都是分开的。沙盒卡不能真实消费，可以用 `sandbox/deposit` 和 `sandbox/spend` 模拟充值和消费。

***

## 三分钟上手

1. 登录仪表盘 → **设置 → API 密钥 → 创建密钥**，填名称，可选填 **IP 白名单**，用支付密码、邮箱或 Telegram 验证码确认。
2. **立即复制 Key**（生产以 `smk_live_` 开头，沙盒以 `smk_test_` 开头）——只显示这一次。
3. 每个请求带上请求头：

```bash
curl -X POST https://api.stablemesh.io/v1/account/balance \
  -H "X-API-KEY: smk_live_…" \
  -H "Content-Type: application/json" \
  -d '{}'
```

***

## 运作方式

* **一个资金账户，USD 计价。** 往你的充值地址链上转 **USDT**（部分网络也收 **USDC**），按 1:1 记入 USD。
* **卡从资金账户扣款。** 开卡、给卡充值时从资金账户扣，扣除手续费后记入卡上。
* **卡上的钱可以提回资金账户。**
* **卡的操作在发卡方是异步的。** 开卡（通常一分钟内激活）、冻结 / 解冻（通常几秒）、提现（通常几分钟）、注销（余额约 **30 天**后退回）都是先受理、后完成，用 `card/detail` 轮询或订阅 webhook 跟进。
* **冻结状态的卡不能充值、也不能提现**，需要先解冻。
* 所有接口都是 `POST` + JSON，返回统一格式 `{ "code": 0, "message": "ok", "data": { … } }`，金额一律是字符串。
* 会动钱的接口建议带 `Idempotency-Key` 请求头，重试时不会重复开卡或重复扣款。

***

## 接口一览

| 分类     | 接口                                                   | 用途                     |
| -------- | ------------------------------------------------------ | ------------------------ |
| 账户     | `account/balance` · `limits` · `transactions`          | 资金账户余额、开卡额度、账户全部流水（与 dashboard「交易」页一致） |
| 充值     | `deposit/networks` · `deposit/address`                 | 可充值的网络、充值地址   |
| 卡片     | `card/products`                                        | 可开的卡产品及费率       |
| 卡片     | `card/create` · `list` · `detail` · `balance`          | 开卡、查卡               |
| 卡片     | `card/deposit` · `withdraw` · `withdraw/status`        | 给卡充值、从卡提回资金账户 |
| 卡片     | `card/freeze` · `unfreeze` · `cancel` · `update`       | 冻结、解冻、注销、改别名 |
| 交易     | `card/transactions`                                    | 卡的消费、退款、手续费记录 |
| Webhook  | `webhook/get` · `set` · `rotateSecret` · `delete`      | 事件推送                 |
| 沙盒     | `sandbox/deposit` · `sandbox/spend`                    | 模拟充值、模拟消费（仅沙盒） |

所有路径都在 `/v1` 下，例如 `https://api.stablemesh.io/v1/card/create`。

***

## API 不提供的

* **卡号、CVV、有效期、持卡人姓名、账单地址** 不会通过 API 返回。请在仪表盘 **卡片 → 选择卡 → 敏感信息** 中查看（需要卡密码）。
* **从资金账户提币到外部链上地址** 只能在仪表盘操作。

{% hint style="info" %}
完整的字段说明、请求 / 响应示例和错误码以英文版为准，请参阅英文文档 **Open API** 章节：Integration Guide、Authentication、Funding & Deposits、Cards、Transactions、Webhooks、Idempotency、Errors。
{% endhint %}
