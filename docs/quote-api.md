# 报价接口详解

`POST https://api.sera.cx/api/v1/swap/quote`

**公开接口。不需要 API key，不需要签名，不需要钱包里有钱。**

一个请求就能问到跨币种的价。下面把字段、响应结构和调用方式逐个写清楚，
可以对照 [`sera-cx/sera-mcp`](https://github.com/sera-cx/sera-mcp) 的
`src/sera/types.ts` 里的 `interface SwapQuoteRequest` 一起看。

## 请求体

七个字段，**全部必填**。

| 字段 | 类型 | 说明 |
|---|---|---|
| `from_token` | string | 卖出币种的合约地址 |
| `to_token` | string | 买入币种的合约地址 |
| `from_amount` | string | **最小单位**，不是小数。100 USDC 写 `"100000000"` |
| `owner_address` | string | 持有地址。问价用不到真地址 |
| `recipient` | string | 收款地址。可以跟上面不同，付款给第三方就是这么表达的 |
| `expiration` | number | Unix **秒**，未来时间 |
| `gas_mode` | string | `"receive_less"` 或 `"pay_more"` |

### 三个要注意的地方

**1. 用地址，不是符号。** `"from_token": "USDC"` 不行，要 `0x...` 合约地址，
从 `/tokens` 拿。

**2. 用最小单位，不是小数。** `"from_amount": "100"` 是 0.0001 USDC，
不是 100 USDC。先查 `decimals` 再乘。

**3. 七个字段都要给。** 只发 `from_token` / `to_token` / `amount` 是最自然的写法，
但 `owner_address`、`recipient`、`expiration`、`gas_mode` 也都要有值，
哪怕问价时用不上它们。

### `gas_mode` 决定成本落在哪一边

不是决定成本多少，是决定谁承担：

- `receive_less` —— 从收到的那一头扣。付款方付的正好是 `from_amount`
- `pay_more` —— 收款金额保持不动，付款方多付一点来覆盖成本

做收款、开发票这类场景要的是 `pay_more`：账单上的金额必须原样到账，
成本得落在付款方身上。两种都是正常用法，按业务意图选就行。

## 响应

```json
{
  "uuid": "dac5aff8-...",
  "route_params": {
    "inputToken": "0xa0b8...",
    "outputToken": "0x3fc9...",
    "maxInputAmount": "100000000",
    "minOutputAmount": "398038805",
    "deadline": 1787496059
  },
  "fee_breakdown": {
    "gas_cost_usd": "1.00",
    "gas_cost_from_token": "1.00"
  },
  "expires_at": 1787495642,
  "route_metadata": { "leg_count": 1 },
  "permit": { ... }
}
```

### `minOutputAmount` 是唯一的数量字段

响应里没有「预计得到多少」这种字段。`minOutputAmount` 是最小保证量，
也是你能拿到的唯一数字。

所以描述的时候要准确：说「报价给出的下限是 X」，
不要说「你会拿到 X」。两者不一样，做市商会挑这个毛病。

### `uuid` 把报价和执行绑在一起

`POST /swap` 要的是 `uuid` 加上对 `route_params` 的签名。
拿 A 报价的参数配 B 报价的 `uuid` 提交，是这套设计专门要防的事。

### `expires_at` 很短

报价撑不过一个慢吞吞的人工确认步骤。过期了就重新问一次，不要留着旧的用。

### `gas_cost_usd` 是按笔收的

实测在各个金额上都是 `"1.00"`：**固定费用，和金额无关**。

跟按比例抽成是两种成本结构 —— 金额越大，这一笔固定费用摊得越薄。
算成本的时候把它当固定项，不要当百分比。

## 批量问价

```
POST /swap/quote/batch
```

请求体是 `{ "quotes": [ ...同样的对象数组... ] }`。
要扫很多个币种或很多档金额的话用这个，比循环快很多。

## 相关只读接口

| 接口 | 需要凭证 | 用途 |
|---|---|---|
| `GET /tokens` | 否 | 币种列表与 decimals |
| `GET /markets` | 否 | 交易对列表 |
| `GET /health` | 否 | 服务状态 |
| `GET /system/time` | 否 | 链上时间，对 `expiration` 有用 |
| `GET /fx/rate` | 否 | 参考中间价 |
| `GET /balances` | **是** | 余额 |
| `GET /orders` | **是** | 订单 |
| `GET /fills` | **是** | 成交记录 |

需要凭证的三个只要 API key，**不需要私钥**。只读。

问价这一步本身也是只读的：返回的是一个没签名的意向，什么都不会动。
真要成交是 `POST /swap`，需要签名，那是另一个决定。

---
*实测于 2026 年 8 月 25 日。*
