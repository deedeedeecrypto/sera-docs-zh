# 报价接口详解

`POST https://api.sera.cx/api/v1/swap/quote`

**公开接口。不需要 API key，不需要签名，不需要钱包里有钱。**

这一篇存在的唯一原因：**这个请求体的格式没有出现在公开文档里。**
它只写在 [`sera-cx/sera-mcp`](https://github.com/sera-cx/sera-mcp) 的
`src/sera/types.ts` 里，`interface SwapQuoteRequest`。

## 请求体

七个字段，**全部必填**。少一个就是 422。

| 字段 | 类型 | 说明 |
|---|---|---|
| `from_token` | string | 卖出币种的合约地址 |
| `to_token` | string | 买入币种的合约地址 |
| `from_amount` | string | **最小单位**，不是小数。100 USDC 写 `"100000000"` |
| `owner_address` | string | 持有地址。问价用不到真地址 |
| `recipient` | string | 收款地址。同上 |
| `expiration` | number | Unix **秒**，未来时间 |
| `gas_mode` | string | `"receive_less"` 或 `"pay_more"` |

### 三个最容易踩的坑

**1. 用符号代替地址。** `"from_token": "USDC"` 不行，必须是 `0x...` 合约地址。
从 `/tokens` 拿。

**2. 用小数代替最小单位。** `"from_amount": "100"` 会被当成 0.0001 USDC，
不是 100 USDC。先查 `decimals` 再乘。

**3. 漏字段。** 只发 `from_token` / `to_token` / `amount` 是最自然的写法，
也是一定会 422 的写法。`owner_address`、`recipient`、`expiration`、`gas_mode`
一个都不能少，哪怕问价根本用不上它们。

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

响应里**没有**「预计得到多少」这种字段。`minOutputAmount` 是最小保证量，
也是你能拿到的唯一数字。

所以描述的时候要准确：说「报价给出的下限是 X」，
不要说「你会拿到 X」。两者不一样，做市商会挑这个毛病。

### `gas_cost_usd` 是固定的

实测在所有金额上都是 `"1.00"`。它是**按笔收的固定费用，和金额无关**。

这一点在算成本的时候很关键：同一条链、同一分钟、同一个方向，
换的金额不同，实际汇率就不同。金额越小，这一美元占的比例越大。

## 实测数据（2026-08-25）

USDC → MYRT，同一分钟，只改金额：

| 投入 | `minOutputAmount` | 实际汇率 |
|---|---|---|
| 1 USDC | `0` | 0 |
| 2 USDC | 3.402636 MYRT | 1.70 |
| 10 USDC | 35.617832 MYRT | 3.56 |
| 100 USDC | 398.038805 MYRT | 3.98 |
| 1000 USDC | 4022.248526 MYRT | 4.02 |

1 USDC 那一档返回 `0`，因为固定的一美元 gas 正好等于全部本金。
这不是报错，是一条正常返回的报价，下限是零。

**结论不是「小额不能用」，而是「小额要先把固定成本算进去」。**
跨境汇款本来也不是按 1 美元一笔在走的。

## 批量问价

```
POST /swap/quote/batch
```

请求体是 `{ "quotes": [ ...同样的对象数组... ] }`。
要扫很多个币种的话用这个，比循环快很多。

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

---
*实测于 2026 年 8 月 25 日。*
