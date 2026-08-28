# 快速开始

目标：三分钟内问到第一个跨币种报价。**不需要 API key，不需要钱包，不需要私钥。**

## 一、先看有哪些币

```bash
curl -s https://api.sera.cx/api/v1/tokens
```

返回 40 个稳定币，覆盖 22 种法币。每一条长这样：

```json
{
  "address": "0x3fc98a885e99420d0ce43bcb81bf21a4e3f45e5f",
  "symbol": "MYRT",
  "decimals": 6,
  "currency": "MYR",
  "min_trade_amount": "0.100000"
}
```

**`decimals` 一定要记下来。** 后面所有金额都用最小单位表示，不是小数。

## 二、看有哪些交易对

```bash
curl -s https://api.sera.cx/api/v1/markets
```

注意返回的 `markets` 数组很长（实测 780 条），里面**大部分和 USDC 无关**。
要找 USDC 的对，得同时看 `base_symbol` 和 `quote_symbol` 两边：

- `X/USDC`（USDC 在报价端）：实测 27 条
- `USDC/X`（USDC 在基础端）：实测 12 条

两个数字都对，只是数的东西不一样。**引用的时候一定要说清楚数的是哪一种**，
不然 27 和 39 看起来像矛盾，其实不是。

## 三、问价

这一步就是大家会卡住的地方。完整格式看[下一篇](quote-api.md)，先给能直接跑的：

```bash
curl -s -X POST https://api.sera.cx/api/v1/swap/quote \
  -H 'Content-Type: application/json' \
  -d '{
    "from_token": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
    "to_token":   "0x3fc98a885e99420d0ce43bcb81bf21a4e3f45e5f",
    "from_amount": "100000000",
    "owner_address": "0x000000000000000000000000000000000000dead",
    "recipient":     "0x000000000000000000000000000000000000dead",
    "expiration": 1790000000,
    "gas_mode": "receive_less"
  }'
```

这是拿 100 USDC 问马币（MYRT）的价。`from_amount` 是 `100000000`，
因为 USDC 是 6 位小数，100 × 10⁶。

`owner_address` 和 `recipient` 填了个黑洞地址。**问价是只读的，不签名、不动钱**，
所以随便填一个合法地址就行。真要成交才需要签。

`expiration` 是 Unix 秒，填一个未来时间。过期了会被拒。

## 四、看懂返回

```json
{
  "route_params": { "minOutputAmount": "398038805", ... },
  "fee_breakdown": { "gas_cost_usd": "1.00" },
  "expires_at": 1787495642
}
```

`minOutputAmount` 也是最小单位。MYRT 是 6 位小数，所以 `398038805` = **398.038805 MYRT**。

换算成汇率：398.038805 ÷ 100 = **约 3.98 MYR/USD**。

## 下一步

- 完整字段说明 → [报价接口详解](quote-api.md)
- 报错了 → [常见错误](troubleshooting.md)

---
*本文数据实测于 2026 年 8 月 25 日。链上流动性会变，数字仅供参照，命令随时可以自己重跑。*
