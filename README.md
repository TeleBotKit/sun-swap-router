# sun-swap-router

运行在全球边缘网络的 TRON 链上兑换（SunSwap）路由服务：一套 HTTP 接口同时构造 **V2 / V4 两种协议**的 swap 交易、查询兑换预估，（可选）转发广播。

只需传入钱包地址、币种和金额，即可拿到**签名后就能上链的交易**——无需自建节点、无需维护 RPC、无需把私钥交给任何人。

SunSwap,TRON,兑换,swap,USDT,TRX,API,路由器,DeFi,TRC20

> 开通服务 / 获取接入凭证（`X-API-KEY`），请联系：[https://t.me/secp256k0](https://t.me/secp256k0)

---

## 目录

- [为什么选择 sun-swap-router](#为什么选择-sun-swap-router)
- [应用场景](#应用场景)
- [工作原理](#工作原理)
- [快速开始](#快速开始)
- [鉴权](#鉴权)
- [单笔限额](#单笔限额)
- [API 参考](#api-参考)
- [阶梯手续费](#阶梯手续费)
- [错误码与排障](#错误码与排障)
- [联系我们](#联系我们)

---

## 为什么选择 sun-swap-router

| 优势 | 说明 |
| ---- | ---- |
| **两种协议并存** | V2（`SunswapV2Router02`）与 V4（Universal Router）共用一套接口，按需选择，无需分别对接 |
| **不碰私钥** | 服务只负责编码交易并返回**可签名交易**，签名、保管私钥永远在您自己的环境里完成 |
| **全球网络** | 服务运行在全球边缘网络，响应快、可用性高；无需自建节点或维护 RPC |
| **接入简单** | 一个 POST 接口传 `ownerAddress`/`tokenIn`/`tokenOut`/`amountIn`，即返回签名后可上链的交易 |
| **智能路由** | 自动寻找最优兑换路径；V2 查询失败自动降级兜底，报价稳定 |
| **自动补资源** | 提交前可自动补齐能量和带宽（`X-Auto-Energy`），避免账户资源不足时白烧 TRX |
| **收费透明** | 按当月累计兑换量阶梯计费，整单切换；每笔广播都有完整流水记录，便于核对与对账 |

## 应用场景

- **钱包 / DApp 内置兑换**：输入金额实时显示预估（`/quote`），一键构造（`/build`）、签名、广播。
- **Telegram 机器人 / 应用内兑换**：在机器人或应用内直接完成 USDT ↔ TRX 兑换，余额自动结算。
- **交易所 / OTC 出入金**：统一的 TRX/USDT 兑换入口，业务侧只对接 HTTP。
- **批量 / 自动兑换服务**：程序化构造大量兑换交易，V4 自动补能量和带宽，减少白烧 TRX。

## 工作原理

```text
您的系统                          sun-swap-router（全球边缘网络）                    TRON 链
   │                                    │                                          │
   │  1. POST /v2|v4/build ───────────> │  构造 swap 交易，返回可签名交易         │
   │  2. 返回未签名 transaction          │  （不碰私钥、不广播）                    │
   │  3. 本地签名（私钥只在本机）        │                                          │
   │  4. POST /v4/broadcast ───────────> │  鉴权 →（可选）补足能量/带宽 → 提交     │
   │                                    │  5. 提交到 TRON 节点 ──────────────────> │
   │                                    │  6. 交易生效                            │
```

## 快速开始

Base URL：**开通服务后由我们告知**——为保障服务安全，接口地址不在公开文档中展示；完成开通后随 `X-API-KEY` 一起提供。

下面是一个完整的 V4 兑换流程示例（Node.js）：构造 → 本地签名 → 提交上链。

```js
const BASE_URL = "<开通服务后由我们提供的接口地址>";
const API_KEY = "<管理员分配给你的 X-API-KEY>";

// 1. 构造 V4 swap 交易（返回未签名 transaction + feeEstimate + quote）
const buildRes = await fetch(`${BASE_URL}/v4/build`, {
  method: "POST",
  headers: { "Content-Type": "application/json", "X-API-KEY": API_KEY },
  body: JSON.stringify({
    ownerAddress: "TYourTronAddressBase58...",
    tokenIn: "USDT",
    tokenOut: "TRX",
    amountIn: "2000000", // 2 USDT（6 位小数）
  }),
});
const built = await buildRes.json();

// 2. 用您的私钥对 built.transaction 签名（例如 TronWeb trx.sign()）
//    —— 私钥始终只出现在您的环境里
const signedTx = await tronWeb.trx.sign(built.transaction);

// 3. 提交广播：回传 transaction + feeEstimate + quote，带 X-API-KEY
const broadcastRes = await fetch(`${BASE_URL}/v4/broadcast`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-API-KEY": API_KEY,
  },
  body: JSON.stringify({
    transaction: signedTx,
    feeEstimate: built.feeEstimate,
    quote: built.quote,
  }),
});
console.log(await broadcastRes.json());
// => { "ok": true, "result": true, "txid": "..." }
```

> 也可以只用 `/quote` 查价、`/build` 构造交易，然后自行签名广播。

## 鉴权

所有接口默认均需鉴权：每个请求都必须携带 `X-API-KEY`（管理员手动分配，并与 Telegram 用户绑定），缺失或无效一律返回 HTTP 401。

手续费按该用户当月累计兑换量走[阶梯费率](#阶梯手续费)计收。

## 单笔限额

`tokenIn` 为 **TRX 或 USDT** 时，单笔兑换金额必须落在以下闭区间内（其它 TRC20 不受限制，可自由构造/查询）：

| tokenIn | 下限 | 上限 |
| ---- | ---- | ---- |
| TRX | 10 TRX | 10,000 TRX |
| USDT | 5 USDT | 5,000 USDT |

- 单位是该代币最小单位（两者都是 6 位小数）。
- 同一套限额在 `/v2` `/v4` 的 build/quote 与 `/v4/broadcast` 上口径一致；超限直接返回 `400`。
- 限额是经营策略，以服务端配置为准，可能调整。

## API 参考

### Base URL

**开通服务后由我们告知**。为保障服务安全，接口地址不在公开文档中展示；您完成开通后，我们会随 `X-API-KEY` 一起提供正式的调用地址，本文档中所有接口均基于该地址。

### 接口一览

| 接口 | 方法 | 说明 |
| ---- | ---- | ---- |
| `/v2/build` | GET/POST | 构造 SunSwap V2 swap 交易 |
| `/v2/approve` | GET/POST | 构造 V2 前置授权（TRC20 approve → Router） |
| `/v2/quote` | GET/POST | 查询 V2 兑换预估 |
| `/v2/broadcast` | POST | 广播已签名交易 |
| `/v4/build` | GET/POST | 构造 SunSwap V4（Universal Router）swap 交易 |
| `/v4/approve` | GET/POST | 构造 V4 两步 Permit2 授权交易 |
| `/v4/quote` | GET/POST | 查询 V4 兑换预估 |
| `/v4/broadcast` | POST | 将已签名交易提交到全球 TRON 节点使其生效，提交前进行智能带宽/能量补足 |

GET 用 query string，POST 用 JSON body，两种方式参数完全一样。

### 构造交易：GET/POST /v2/build · /v4/build

公共参数：

| 字段 | 必填 | 说明 | 示例 |
| ---- | ---- | ---- | ---- |
| `ownerAddress` | 是 | 发起交易的钱包地址（`T...` base58） | `TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t` |
| `tokenIn` | 是 | 输入代币：`TRX`/`USDT`（大写）或 TRC20 合约地址 | `USDT` |
| `tokenOut` | 是 | 输出代币：同上 | `TRX` |
| `amountIn` | 是 | 输入数量（最小单位整数），TRX/USDT 需在[单笔限额](#单笔限额)内 | `2000000` |
| `amountOutMin` | 否 | 最小可接受输出（滑点保护）；不传则自动按报价折算 | `1000000` |
| `slippageBps` | 否 | 滑点，单位基点（`100` = 1%），仅在 `amountOutMin` 未传时生效 | `500` |
| `recipient` | 否 | 收款地址，默认等于 `ownerAddress` | `T...` |
| `deadline` / `ttlSec` | 否 | 交易过期时间（二选一），默认约 300 秒 | `600` |
| `feeLimit` | 否 | 覆盖默认 `fee_limit`（sun） | `250000000` |
| `autoPath` | 仅 V2 | 传 `false` 关闭智能路由、强制直连两跳；V4 无此参数（永远查智能路由） | `false` |

响应包含：

- `transaction` —— **未签名交易**原文（`raw_data` / `raw_data_hex` / `txID`），签名后即可广播；
- `quote` —— 报价与自动折算的 `amountOut` / `amountOutMin`；
- `feeEstimate` —— 可选返回：预计消耗的能量/带宽与烧 TRX 金额。

V2 与 V4 的关键差异：

| 差异点 | V2 | V4 |
| ------ | ---- | ---- |
| 授权模型 | 标准 TRC20 `approve(Router, amount)` | Permit2 两步授权（见 `/v4/approve`） |
| 路径解析 | 智能路由失败可降级直连两跳 | 必须智能路由，无降级 |
| 原生 TRX | 内部包装成 WTRX | 池子可直接配对原生 TRX |
| 报价核实 | 链上二次核实 | 路由方估算 |

### 授权：GET/POST /v2/approve · /v4/approve

- **V2**：非原生 TRC20 第一次 swap 前，先 `approve` 给 V2 Router（`spender` 固定为内置地址，不接受指定）。参数：`ownerAddress` + `token`（`USDT` 或合约地址）+ `amount`（`0` 撤销 / `MAX` 无限授权 / 具体额度）。
- **V4**：走 Permit2，需要**两笔且按顺序执行**的授权——先 `approve` 给 Permit2，链上确认后再让 Permit2 授权给 Universal Router。`/v4/approve` 一次返回 `steps` 数组（长度固定 2，顺序即执行顺序）；`amount` 上限是 `uint160`，`expiration` 支持 `MAX`。

### 查询预估：GET/POST /v2/quote · /v4/quote

纯查询接口，**不需要 `ownerAddress`**：传 `tokenIn` / `tokenOut` / `amountIn`（可加 `slippageBps`），返回 `amountOut` / `amountOutMin` / `path`，适合前端输入框实时显示预估。不构造任何交易、不产生广播。

### 广播：POST /v2/broadcast · POST /v4/broadcast

| | `/v2/broadcast` | `/v4/broadcast` |
| ---- | ---- | ---- |
| 请求体 | 已签名交易（`raw_data` + `signature`） | 已签名 `transaction` + `feeEstimate` + `quote`（`/v4/build` 响应原样回传） |
| 附加头 | - | `X-Auto-Energy`：提交前自动补齐能量和带宽 |

**处理流程（`/v4/broadcast`）**：校验 `X-API-KEY` → 校验请求体 →（可选）智能补足带宽/能量 → 将交易提交到全球 TRON 节点使其生效。

## 阶梯手续费

费率不是固定值，按下单人**当月累计兑换量**（USDT 计价）从阶梯表里选一档。当前档位（以服务端配置为准）：

| 当月累计兑换量 | 本单费率 |
| ---- | ---- |
| 0 起 | 3% |
| 500 USDT 起 | 2% |
| 1,000 USDT 起 | 1.2% |
| 2,000 USDT 起 | 0.8% |
| 3,000 USDT 起 | 0.5% |

要点：

- **整档切换**，不是累进分段——跨过阈值后整单按新费率收。
- 只统计真正上链成功的单；被拒/广播失败的单不计入。
- TRX 输入的单按下单时汇率折算成 USDT 后计入。
- 按 **UTC+8 每月 1 日 00:00** 归零。
- **档位次日生效**（兑换量按日更新）——当天跨过阈值，次日才享受新档；滞后方向保守（先按更贵的档收，不会倒挂）。
- 响应里会带本单实际适用的 `feePercentage` 与 `monthlyVolumeUsdtMicro`（不含本单），可直接用于展示"还差多少降档"。

## 错误码与排障

### HTTP 状态码

| 状态码 | 含义 |
| ---- | ---- |
| 200 | 成功，按各接口返回响应体 |
| 400 | 参数错误/超限/补能量失败/烧 TRX 超限等，响应体含 `error` 与 `message` |
| 401 | `X-API-KEY` 缺失或无效 |
| 404 | 接口路径不存在 |
| 500 | 服务端内部异常 |

### 常见问题（FAQ）

**广播时报 `TRANSACTION_EXPIRATION_ERROR`？**
build 返回的未签名交易有效期很短（约 60 秒）。build → 签名 → 广播间隔太久会过期——不是 bug，是 TRON 协议本身的行为。重新 build 一次拿新鲜交易即可，建议整条流程写成自动化脚本。

**`/v4/broadcast` 为什么要求回传 `feeEstimate`/`quote`？**
`feeEstimate` 用于评估这笔交易需要的能量/带宽，`quote` 用于校验兑换参数。为保障交易正确提交，接口会对请求体做二次校验（单笔限额、代币类型等），超限/不支持直接拒绝，在提交之前拦截。

**`energy_topup_failed` 怎么办？**
只在带 `X-Auto-Energy` 时可能出现，此时交易**没有被广播**。按 `reason` 分支：`purchase_failed`（最常见：账户余额不足，或尚未开通补资源服务——联系管理员）；`exceeds_limit`（请求体里的 `feeEstimate` 被改过）；`not_delivered`（买到了但没等到账，钱已花、能量已归你，重新签一笔交易再发即可）。

**自动补资源（`X-Auto-Energy`）是什么？**
提交前会自动补齐能量和带宽，避免账户资源不足导致交易变慢或额外消耗。能量补不上**中止提交**；带宽补不上**照常提交**，只在 `bandwidthTopup.failReason` 里如实报告。

**支持哪些代币？**
构造与查询（build/quote）支持任意 TRC20 + TRX；提交上链（`/v4/broadcast`）目前只支持 **TRX/USDT**。

**私钥安全吗？**
安全。服务不接触、不保存任何私钥，只返回未签名交易；签名永远在您自己的环境完成。build 接口也不接受 `router`/`path`/`spender` 等字段——交易只能导向内置的官方 Router，防钓鱼授权。

## 联系我们

开通服务、获取 `X-API-KEY`、调整限额/费率、定制功能，请联系：

**Telegram：[https://t.me/secp256k0](https://t.me/secp256k0)**
