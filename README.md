
Polymarket 统一 TypeScript SDK - 预测市场交易、套利检测、聪明钱分析和完整市场数据。

## 核心功能

- 🔄 **实时套利检测**: WebSocket 监控 + 自动执行
- 📊 **聪明钱分析**: 追踪顶级交易者及其策略
- 💱 **交易集成**: 支持 GTC/GTD/FOK/FAK 订单类型
- 🔐 **链上操作**: Split、Merge、Redeem CTF 代币
- 🌉 **跨链桥接**: 从 Ethereum、Solana、Bitcoin 充值
- 💰 **DEX 交换**: 使用 QuickSwap V3 在 Polygon 上兑换代币
- 📈 **市场分析**: K 线、信号、成交量分析

## 安装

```bash
pnpm add @catalyst-team/poly-sdk
```

## 框架使用

```bash
pnpm tsx main.ts
```

## 快速开始

```typescript
import { PolymarketSDK } from '@catalyst-team/poly-sdk';

const sdk = new PolymarketSDK();

// 通过 slug 或 condition ID 获取市场
const market = await sdk.getMarket('will-trump-win-2024');
console.log(market.tokens.yes.price); // 0.65

// 获取处理后的订单簿（包含分析数据）
const orderbook = await sdk.getOrderbook(market.conditionId);
console.log(orderbook.summary.longArbProfit); // 套利机会

// 检测套利
const arb = await sdk.detectArbitrage(market.conditionId);
if (arb) {
  console.log(`${arb.type} 套利: ${arb.profit * 100}% 利润`);
}
```

## 架构

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                             PolymarketSDK                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│  第三层: 服务层                                                                │
│  ┌─────────────┐ ┌─────────────┐ ┌───────────────┐ ┌─────────────────────────┐│
│  │钱包服务     │ │市场服务     │ │实时服务       │ │   授权服务              ││
│  │ - 用户画像  │ │ - K 线     │ │- 订阅管理     │ │   - ERC20 授权          ││
│  │ - 卖出检测  │ │ - 信号     │ │- 价格缓存     │ │   - ERC1155 授权        ││
│  └─────────────┘ └─────────────┘ └───────────────┘ └─────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ 套利服务: 实时套利检测、自动再平衡、智能清仓                                │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ 交换服务: Polygon 上的 DEX 交换 (QuickSwap V3, USDC/USDC.e 转换)         │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────────────────────┤
│  第二层: API 客户端                                                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌────────────────────┐  │
│  │ Data API │ │Gamma API │ │ CLOB API │ │ WebSocket │ │   跨链桥            │  │
│  │持仓数据  │ │ 市场数据 │ │ 订单簿   │ │ 实时价格  │ │   跨链充值          │  │
│  │交易记录  │ │ 事件数据 │ │ 交易     │ │ 价格推送  │ │   多链支持          │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────┘ └────────────────────┘  │
│  ┌──────────────────────────────────────┐ ┌────────────────────────────────┐  │
│  │ 交易客户端: 订单执行                  │ │ CTF 客户端: 链上操作           │  │
│  │ GTC/GTD/FOK/FAK, 奖励, 余额查询      │ │ Split / Merge / Redeem 代币    │  │
│  └──────────────────────────────────────┘ └────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────────────────────┤
│  第一层: 基础设施                                                             │
│  ┌────────────┐  ┌─────────┐  ┌──────────┐  ┌────────────┐ ┌──────────────┐   │
│  │速率限制器  │  │  缓存   │  │  错误    │  │   类型     │ │ 价格工具     │   │
│  │按 API 限制 │  │TTL 机制 │  │ 重试机制 │  │ 统一定义   │ │ 套利检测     │   │
│  └────────────┘  └─────────┘  └──────────┘  └────────────┘ └──────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 核心概念

### 理解 Polymarket 的镜像订单簿

⚠️ **关键：Polymarket 订单簿有一个容易被忽略的镜像特性**

```
买 YES @ P = 卖 NO @ (1-P)
```

这意味着**同一订单会出现在两个订单簿中**。例如，一个 "卖 NO @ 0.50" 订单会同时作为 "买 YES @ 0.50" 出现在 YES 订单簿中。

**常见错误：**
```typescript
// ❌ 错误: 简单相加会重复计算镜像订单
const askSum = YES.ask + NO.ask;  // ≈ 1.998-1.999，而非 ≈ 1.0
const bidSum = YES.bid + NO.bid;  // ≈ 0.001-0.002，而非 ≈ 1.0
```

**正确做法：使用有效价格**
```typescript
import { getEffectivePrices, checkArbitrage } from '@catalyst-team/poly-sdk';

// 计算考虑镜像后的最优价格
const effective = getEffectivePrices(yesAsk, yesBid, noAsk, noBid);

// effective.effectiveBuyYes = min(YES.ask, 1 - NO.bid)
// effective.effectiveBuyNo = min(NO.ask, 1 - YES.bid)
// effective.effectiveSellYes = max(YES.bid, 1 - NO.ask)
// effective.effectiveSellNo = max(NO.bid, 1 - YES.ask)

// 使用有效价格检测套利
const arb = checkArbitrage(yesAsk, noAsk, yesBid, noBid);
if (arb) {
  console.log(`${arb.type} 套利: ${(arb.profit * 100).toFixed(2)}% 利润`);
  console.log(arb.description);
}
```

详细文档见: [docs/01-polymarket-orderbook-arbitrage.md](docs/01-polymarket-orderbook-arbitrage.md)

### ArbitrageService - 自动化交易

实时套利检测和执行，支持市场扫描、自动再平衡和智能清仓。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  核心功能                                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  • scanMarkets()     - 扫描市场找套利机会                                      │
│  • start(market)     - 启动实时监控 + 自动执行                                 │
│  • clearPositions()  - 智能清仓 (活跃市场卖出, 已结算市场 redeem)                │
├─────────────────────────────────────────────────────────────────────────────┤
│  自动再平衡器                                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  套利需要 USDC + YES/NO 代币，再平衡器自动维持最佳组合：                         │
│  • USDC < 20%  → 自动 Merge (YES+NO → USDC)                                 │
│  • USDC > 80%  → 自动 Split (USDC → YES+NO)                                 │
│  • 冷却机制：操作间隔 30 秒，检测间隔 10 秒                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  部分成交保护                                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  套利需要同时买入 YES 和 NO，但订单可能部分成交：                                 │
│  • sizeSafetyFactor=0.8 → 仅使用 80% 的订单簿深度                             │
│  • autoFixImbalance=true → 如果只成交一侧，自动卖出多余的代币                   │
│  • imbalanceThreshold=5 → YES-NO 差额超过 $5 时触发修复                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

**完整工作流：**

```typescript
import { ArbitrageService } from '@catalyst-team/poly-sdk';

const arbService = new ArbitrageService({
  privateKey: process.env.POLY_PRIVKEY,
  profitThreshold: 0.005,  // 最小 0.5% 利润
  minTradeSize: 5,         // 最小 $5
  maxTradeSize: 100,       // 最大 $100
  autoExecute: true,       // 自动执行套利机会

  // 再平衡配置
  enableRebalancer: true,  // 启用自动再平衡
  minUsdcRatio: 0.2,       // 最小 20% USDC (低于则 Split)
  maxUsdcRatio: 0.8,       // 最大 80% USDC (高于则 Merge)
  targetUsdcRatio: 0.5,    // 再平衡目标 50%
  imbalanceThreshold: 5,   // 修复前的最大 YES-NO 差额
  rebalanceInterval: 10000, // 每 10 秒检查一次
  rebalanceCooldown: 30000, // 操作间隔最小 30 秒

  // 执行安全（防止部分成交导致 YES ≠ NO）
  sizeSafetyFactor: 0.8,   // 使用 80% 的订单簿深度
  autoFixImbalance: true,  // 如果一侧失败自动卖出多余部分
});

// 监听事件
arbService.on('opportunity', (opp) => {
  console.log(`${opp.type.toUpperCase()} 套利: ${opp.profitPercent.toFixed(2)}%`);
});

arbService.on('execution', (result) => {
  if (result.success) {
    console.log(`✅ 已执行: $${result.profit.toFixed(2)} 利润`);
  }
});

// ========== 步骤 1: 扫描市场 ==========
const results = await arbService.scanMarkets({ minVolume24h: 5000 }, 0.005);
console.log(`找到 ${results.filter(r => r.arbType !== 'none').length} 个机会`);

// 或者一键扫描 + 启动最佳市场
const best = await arbService.findAndStart(0.005);
if (!best) {
  console.log('未找到套利机会');
  process.exit(0);
}
console.log(`🎯 已启动: ${best.market.name} (+${best.profitPercent.toFixed(2)}%)`);

// ========== 步骤 2: 运行套利 ==========
// 服务现在自动监控并执行套利...
await new Promise(resolve => setTimeout(resolve, 60 * 60 * 1000)); // 1 小时

// ========== 步骤 3: 停止并清仓 ==========
await arbService.stop();
console.log('统计数据:', arbService.getStats());

// 智能清仓: 活跃市场 merge+sell, 已结算市场 redeem
const clearResult = await arbService.clearPositions(best.market, true);
console.log(`✅ 已回收: $${clearResult.totalUsdcRecovered.toFixed(2)}`);
```

## API 客户端

### TradingClient - 订单执行

```typescript
import { TradingClient, RateLimiter } from '@catalyst-team/poly-sdk';

const tradingClient = new TradingClient(new RateLimiter(), {
  privateKey: process.env.POLYMARKET_PRIVATE_KEY!,
});

await tradingClient.initialize();

// GTC 限价单（保持有效直到成交或取消）
const order = await tradingClient.createOrder({
  tokenId: yesTokenId,
  side: 'BUY',
  price: 0.45,
  size: 10,
  orderType: 'GTC',
});

// FOK 市价单（完全成交或取消）
const marketOrder = await tradingClient.createMarketOrder({
  tokenId: yesTokenId,
  side: 'BUY',
  amount: 10, // $10 USDC
  orderType: 'FOK',
});

// 订单管理
const openOrders = await tradingClient.getOpenOrders();
await tradingClient.cancelOrder(orderId);
```

### CTFClient - 链上代币操作

CTF (Conditional Token Framework) 客户端支持 Polymarket 条件代币的链上操作。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  CTF 操作快速参考                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  操作       │ 功能                │ 典型用例                                   │
├────────────┼─────────────────────┼────────────────────────────────────────────┤
│  Split     │ USDC → YES + NO     │ 做市：创建代币库存                          │
│  Merge     │ YES + NO → USDC     │ 套利：买入双边后合并                        │
│  Redeem    │ 获胜代币 → USDC     │ 结算：领取获胜代币                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

```typescript
import { CTFClient } from '@catalyst-team/poly-sdk';

const ctf = new CTFClient({
  privateKey: process.env.POLYMARKET_PRIVATE_KEY!,
  rpcUrl: 'https://polygon-rpc.com',
});

// Split: USDC → YES + NO 代币
const splitResult = await ctf.split(conditionId, '100');
console.log(`创建了 ${splitResult.yesTokens} YES + ${splitResult.noTokens} NO`);

// Merge: YES + NO → USDC
const tokenIds = {
  yesTokenId: market.tokens[0].tokenId,
  noTokenId: market.tokens[1].tokenId,
};
const mergeResult = await ctf.mergeByTokenIds(conditionId, tokenIds, '100');
console.log(`收到 ${mergeResult.usdcReceived} USDC`);

// Redeem: 获胜代币 → USDC
const redeemResult = await ctf.redeemByTokenIds(conditionId, tokenIds);
console.log(`兑换了 ${redeemResult.tokensRedeemed} 个代币`);
```

⚠️ **重要：对 Polymarket 市场使用 `mergeByTokenIds()` 和 `redeemByTokenIds()`**

Polymarket 使用的自定义 token ID 与标准 CTF position ID 不同。始终使用 `*ByTokenIds` 方法配合 CLOB API 返回的 token ID。

### SwapService - Polygon 上的 DEX 交换

使用 QuickSwap V3 在 Polygon 上交换代币。对于 CTF 操作，转换为 USDC.e 是必需的。

⚠️ **Polymarket CTF 的 USDC vs USDC.e**

| 代币 | 地址 | Polymarket CTF |
|------|------|---------------|
| USDC.e | `0x2791...` | ✅ **必需** |
| USDC (原生) | `0x3c49...` | ❌ 不接受 |

```typescript
import { SwapService, POLYGON_TOKENS } from '@catalyst-team/poly-sdk';

const swapService = new SwapService(signer);

// 将原生 USDC 交换为 USDC.e 用于 CTF 操作
const swapResult = await swapService.swap('USDC', 'USDC_E', '100');
console.log(`已交换: ${swapResult.amountOut} USDC.e`);

// 将 MATIC 交换为 USDC.e
const maticSwap = await swapService.swap('MATIC', 'USDC_E', '50');

// 交换前获取报价
const quote = await swapService.getQuote('WETH', 'USDC_E', '0.1');
console.log(`预期输出: ${quote.estimatedAmountOut} USDC.e`);
```

### WalletService - 聪明钱分析

```typescript
// 获取顶级交易者
const traders = await sdk.wallets.getTopTraders(10);

// 获取钱包画像（含聪明分数）
const profile = await sdk.wallets.getWalletProfile('0x...');
console.log(profile.smartScore); // 0-100

// 检测卖出活动（用于跟单策略）
const sellResult = await sdk.wallets.detectSellActivity(
  '0x...',
  conditionId,
  Date.now() - 24 * 60 * 60 * 1000
);
if (sellResult.isSelling) {
  console.log(`已卖出 ${sellResult.percentageSold}%`);
}
```

### MarketService - K 线和信号

```typescript
// 获取 K 线蜡烛图
const klines = await sdk.markets.getKLines(conditionId, '1h', { limit: 100 });

// 获取双 K 线（YES + NO）含价差分析
const dual = await sdk.markets.getDualKLines(conditionId, '1h');

// 历史价差（来自成交收盘价）- 用于回测
for (const point of dual.spreadAnalysis) {
  console.log(`${point.timestamp}: 总和=${point.priceSum}, 价差=${point.priceSpread}`);
  if (point.arbOpportunity) {
    console.log(`  历史 ${point.arbOpportunity} 信号`);
  }
}

// 实时价差（来自订单簿）- 用于实盘交易
if (dual.realtimeSpread) {
  const rt = dual.realtimeSpread;
  if (rt.arbOpportunity) {
    console.log(`🎯 ${rt.arbOpportunity} 套利: ${rt.arbProfitPercent.toFixed(2)}%`);
  }
}
```

#### 价差分析 - 两种方法

```
┌─────────────────────────────────────────────────────────────────────────┐
│  spreadAnalysis (历史分析)        │  realtimeSpread (实时交易)         │
├─────────────────────────────────────────────────────────────────────────┤
│  数据源: 成交收盘价                │  数据源: 订单簿 bid/ask             │
│  YES_close + NO_close             │  使用有效价格（考虑镜像）            │
├─────────────────────────────────────────────────────────────────────────┤
│  ✅ 可构建历史图表                 │  ❌ 无历史数据*                      │
│  ✅ Polymarket 保留成交历史        │  ❌ Polymarket 不保留快照           │
│  ✅ 适合回测                       │  ✅ 适合实盘交易                     │
│  ⚠️ 套利信号仅供参考               │  ✅ 套利利润计算准确                 │
└─────────────────────────────────────────────────────────────────────────┘

* 要构建历史实时价差，必须自己存储订单簿快照
  参考: apps/api/src/services/spread-sampler.ts
```

### RealtimeService - WebSocket 订阅

⚠️ **重要：订单簿自动排序**

Polymarket CLOB API 返回的订单簿顺序与标准预期相反：
- **bids**: 升序（最低价在前 = 最差价）
- **asks**: 降序（最高价在前 = 最差价）

我们的 SDK **自动规范化**订单簿数据：
- **bids**: 降序（最高价在前 = 最佳买价）
- **asks**: 升序（最低价在前 = 最佳卖价）

这意味着你可以安全地使用 `bids[0]` 和 `asks[0]` 获取最优价格：

```typescript
const book = await sdk.clobApi.getOrderbook(conditionId);
const bestBid = book.bids[0]?.price;  // ✅ 最高买价（最佳）
const bestAsk = book.asks[0]?.price;  // ✅ 最低卖价（最佳）

// WebSocket 更新同样自动排序
wsManager.on('bookUpdate', (update) => {
  const bestBid = update.bids[0]?.price;  // ✅ 已排序
  const bestAsk = update.asks[0]?.price;  // ✅ 已排序
});
```

### ChainMonitorClient - 链上交易监控

监控 Polygon 链上的 CTF 交易事件，用于鲸鱼发现。

```typescript
import { ChainMonitorClient } from '@catalyst-team/poly-sdk';

const monitor = new ChainMonitorClient({
  infuraApiKey: 'your-infura-key',
  wsEnabled: true, // 使用 WebSocket (推荐)
});

await monitor.connect();

// 监听所有交易
monitor.on('transfer', (event) => {
  console.log(`${event.from} → ${event.to}: ${event.amount} tokens`);
});

// 或使用异步迭代
for await (const event of monitor.subscribeAllTransfers()) {
  console.log(event);
}
```

### 鲸鱼发现服务 (Whale Discovery)

从链上交易中自动发现潜在的跟单目标。

```
┌─────────────────────────────────────────────────────────────────┐
│  阶段 1: 链上预过滤（无 API 调用）                                │
│  • 过滤小额交易 (< $50)                                         │
│  • 跳过已分析地址 (24h 缓存)                                     │
│  • 排除合约和官方地址                                            │
├─────────────────────────────────────────────────────────────────┤
│  阶段 2: 批量分析（每分钟 10 个地址）                              │
│  • 获取钱包画像                                                  │
│  • 评估胜率、盈利、交易量                                         │
│  • 符合条件则标记为鲸鱼                                           │
└─────────────────────────────────────────────────────────────────┘
```

**配置文件 (config.json):**

```json
{
  "INFURA_API_KEY": "your-infura-api-key",
  "API_BASE_URL": "http://localhost:3000"
}
```

**启动方式:**

```bash
# 1. 启动 API 后端
cd api_src && pnpm install && pnpm dev

# 4. 或使用 Web 前端
cd web_front_src && pnpm install && pnpm dev
# 访问 http://localhost:5173/whale
```

## 示例

| 示例 | 说明 | 命令 |
|------|------|------|
| [基础用法](examples/01-basic-usage.ts) | 获取市场、订单簿、检测套利 | `pnpm example:basic` |
| [聪明钱](examples/02-smart-money.ts) | 顶级交易者、钱包画像、聪明分数 | `pnpm example:smart-money` |
| [市场分析](examples/03-market-analysis.ts) | 市场信号、成交量分析 | `pnpm example:market-analysis` |
| [K 线聚合](examples/04-kline-aggregation.ts) | 从成交记录构建 OHLCV 蜡烛图 | `pnpm example:kline` |
| [跟单策略](examples/05-follow-wallet-strategy.ts) | 追踪聪明钱持仓、检测退出 | `pnpm example:follow-wallet` |
| [服务演示](examples/06-services-demo.ts) | 所有 SDK 服务实战 | `pnpm example:services` |
| [实时 WebSocket](examples/07-realtime-websocket.ts) | 实时价格推送、订单簿更新 | `pnpm example:realtime` |
| [交易订单](examples/08-trading-orders.ts) | GTC、GTD、FOK、FAK 订单类型 | `pnpm example:trading` |
| [奖励追踪](examples/09-rewards-tracking.ts) | 做市激励、收益 | `pnpm example:rewards` |
| [CTF 操作](examples/10-ctf-operations.ts) | Split、merge、redeem 代币 | `pnpm example:ctf` |
| [实时套利扫描](examples/11-live-arbitrage-scan.ts) | 扫描真实市场寻找机会 | `pnpm example:live-arb` |

## 错误处理

```typescript
import { PolymarketError, ErrorCode, withRetry } from '@catalyst-team/poly-sdk';

try {
  const market = await sdk.getMarket('invalid-slug');
} catch (error) {
  if (error instanceof PolymarketError) {
    if (error.code === ErrorCode.MARKET_NOT_FOUND) {
      console.log('市场未找到');
    } else if (error.code === ErrorCode.RATE_LIMITED) {
      console.log('速率限制，稍后重试');
    }
  }
}

// 自动重试（指数退避）
const result = await withRetry(() => sdk.getMarket(slug), {
  maxRetries: 3,
  baseDelay: 1000,
});
```

## 速率限制

内置按 API 类型的速率限制：
- Data API: 10 请求/秒
- Gamma API: 10 请求/秒
- CLOB API: 5 请求/秒

```typescript
import { RateLimiter, ApiType } from '@catalyst-team/poly-sdk';

// 自定义速率限制器
const limiter = new RateLimiter({
  [ApiType.DATA]: { maxConcurrent: 5, minTime: 200 },
  [ApiType.GAMMA]: { maxConcurrent: 5, minTime: 200 },
  [ApiType.CLOB]: { maxConcurrent: 2, minTime: 500 },
});
```

## 多模块项目架构

除了核心 SDK，项目还包含三个独立的应用模块：

```
FKPolySDK/
├── src/              # SDK 核心源码
├── api_src/          # API 后端服务 (Fastify)
├── console_src/      # 控制台客户端 (Commander.js)
└── web_front_src/    # Web 前端 (React + Vite)
```

### API 后端 (api_src/)

基于 Fastify 的 REST API 服务，提供市场数据、套利扫描、钱包分析接口。

```bash
cd api_src
pnpm install
pnpm dev    # 启动服务: http://localhost:3000
```

| 端点 | 描述 |
|------|------|
| `GET /api/markets/trending` | 获取热门市场 |
| `GET /api/arbitrage/scan` | 扫描套利机会 |
| `GET /api/wallets/leaderboard` | 获取排行榜 |
| `ws://localhost:3000/ws/market/:id` | 实时数据推送 |

API 文档: http://localhost:3000/docs

### 控制台 (console_src/)

基于 Commander.js 的命令行工具，用于快速测试和监控。

```bash
cd console_src
pnpm install
pnpm tsx src/index.ts markets list      # 列出热门市场
pnpm tsx src/index.ts arb scan          # 扫描套利
pnpm tsx src/index.ts wallet leaderboard  # 交易员排行
pnpm tsx src/index.ts monitor scan      # 持续监控套利
```

### Web 前端 (web_front_src/)

基于 React + Vite + Ant Design 的 Web 仪表盘。

```bash
cd web_front_src
pnpm install
pnpm dev    # 启动: http://localhost:5173
```

> **注意**: 前端需要 API 后端运行在 `localhost:3000`

| 页面 | 功能 |
|------|------|
| `/dashboard` | 仪表盘概览 |
| `/markets` | 市场列表 |
| `/arbitrage` | 套利扫描 |
| `/wallets` | 钱包排行 |

## 文档

- [订单簿与套利指南](docs/01-polymarket-orderbook-arbitrage.md) - 理解镜像订单

## 依赖

- `@nevuamarkets/poly-websockets` - WebSocket 客户端
- `bottleneck` - 速率限制
- `ethers` - 区块链交互

## 许可

MIT

