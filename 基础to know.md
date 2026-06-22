# 基础 to know

这份笔记用于补齐量化研究里最容易被忽略但最关键的底层知识，重点放在高频数据、订单簿、逐笔数据和多因子模型训练框架上。

核心判断：

```text
中低频量化主要研究“哪些资产未来更值得持有”；
高频量化主要研究“下一小段时间价格、成交和盘口会怎样变化，以及如何低成本执行”。
```

---

## 1. 高频策略先看市场微观结构

高频不是把日频策略跑得更快，而是研究交易机制本身。

最基础对象：

- `Bid`：买一价，当前市场愿意买入的最高价格。
- `Ask`：卖一价，当前市场愿意卖出的最低价格。
- `Mid = (Bid + Ask) / 2`：中间价，常用于短周期收益标签。
- `Spread = Ask - Bid`：买卖价差，是高频交易的基础成本。
- `Depth`：各档盘口挂单量，代表可立即交易的流动性。
- `Queue`：同一价格档位上订单排队，决定限价单成交优先级。
- `Tick size`：最小价格变动单位，会影响价差、排队和做市收益。

高频研究的核心不是预测一天后的涨跌，而是回答：

```text
未来几笔成交或未来几秒，mid-price 是否会移动？
我的限价单能否成交？
成交后是否会被逆向选择？
执行这一笔交易的真实成本是多少？
```

在 A 股股票上还要注意制度边界：普通股票通常是 T+1，买入后当日不能卖出；部分 ETF、债券、可转债、期货、期权等品种可能存在回转交易或更适合日内策略。做高频策略时必须先确认品种、交易所规则、涨跌停、集合竞价、最小申报单位、撤单限制和费用结构。

下文统一使用这些记号：

```text
P^b_{t,k}, Q^b_{t,k}: t 时刻第 k 档买价和买量
P^a_{t,k}, Q^a_{t,k}: t 时刻第 k 档卖价和卖量
Bid_t = P^b_{t,1}, Ask_t = P^a_{t,1}
Mid_t = (Bid_t + Ask_t) / 2
Spread_t = Ask_t - Bid_t
tick: 最小价格变动单位
event index n: 第 n 个订单簿/成交/撤单事件
H: 预测 horizon，可以是秒数、bar 数或事件数
```

高频公式里最容易混淆的是“时钟时间”和“事件时间”：`t + 5s` 是固定秒数，`n + 50 events` 是固定事件数。高活跃品种在 5 秒内可能有几百个事件，低活跃品种可能没有任何事件，所以高频特征和标签要明确采样口径。

---

## 2. 限价订单簿数据

限价订单簿（Limit Order Book, LOB）记录不同价格档位上的买卖挂单。

常见 Level-2 快照字段：

| 字段 | 含义 | 高频用途 |
| --- | --- | --- |
| `bid_price_1...K` | 买盘第 1 到 K 档价格 | 估计可成交价格、支撑区间 |
| `ask_price_1...K` | 卖盘第 1 到 K 档价格 | 估计卖压、冲击成本 |
| `bid_size_1...K` | 买盘各档挂单量 | 计算深度和买盘支撑 |
| `ask_size_1...K` | 卖盘各档挂单量 | 计算卖盘压力 |
| 时间戳 | 交易所或行情源时间 | 对齐逐笔成交和委托 |
| 序号 | 行情事件顺序 | 处理乱序、丢包和重复 |

常见订单簿因子：

```text
Order Book Imbalance:
OBI_t = (BidSize_1 - AskSize_1) / (BidSize_1 + AskSize_1)

Multi-level Imbalance:
OBI_t^K = (sum_k w_k BidSize_k - sum_k w_k AskSize_k)
          / (sum_k w_k BidSize_k + sum_k w_k AskSize_k)

Microprice:
MicroPrice_t = (Ask_t * BidSize_1 + Bid_t * AskSize_1)
               / (BidSize_1 + AskSize_1)
```

`MicroPrice` 的直觉是：如果买一挂单很厚、卖一挂单很薄，价格更容易向上移动，所以微价格会偏向 `Ask`。

更细的典型变量：

| 变量 | 公式 | 解释 |
| --- | --- | --- |
| 相对价差 | `RelSpread_t = (Ask_t - Bid_t) / Mid_t` | 不同价格水平的股票可以比较。 |
| tick 价差 | `TickSpread_t = (Ask_t - Bid_t) / tick` | 判断价差是一跳、两跳还是更宽。 |
| K 档买方深度 | `Depth^b_t(K) = sum_{k=1}^K Q^b_{t,k}` | 买方可见流动性。 |
| K 档卖方深度 | `Depth^a_t(K) = sum_{k=1}^K Q^a_{t,k}` | 卖方可见流动性。 |
| 加权深度 | `WDepth^b_t = sum_k w_k Q^b_{t,k}` | 近档权重通常更高，如 `w_k = exp(-rho * (k-1))`。 |
| 深度不平衡 | `(Depth^b_t(K) - Depth^a_t(K)) / (Depth^b_t(K) + Depth^a_t(K))` | 多档版买卖压力。 |
| 微价格偏离 | `(MicroPrice_t - Mid_t) / (Spread_t / 2)` | 标准化到半价差口径，范围通常接近 `[-1, 1]`。 |
| 档位斜率 | `(P^a_{t,K} - Ask_t) / sum_{k=1}^K Q^a_{t,k}` | 盘口越薄、价格越快变差，斜率越大。 |
| 盘口凸性 | `Depth(K2) / Depth(K1), K2 > K1` | 判断流动性是集中在近档还是远档。 |

多档价格和数量经常要先做归一化，否则模型会把价格水平、股本规模和流动性混在一起：

```text
price_gap^b_{t,k} = (Mid_t - P^b_{t,k}) / tick
price_gap^a_{t,k} = (P^a_{t,k} - Mid_t) / tick
norm_size^b_{t,k} = Q^b_{t,k} / median_volume_i
norm_size^a_{t,k} = Q^a_{t,k} / median_volume_i
```

更贴近前沿研究的盘口状态变量：

```text
Queue depletion:
Depl^b_t(H) = Consumed^b_t(H) / Q^b_{t,1}
Depl^a_t(H) = Consumed^a_t(H) / Q^a_{t,1}

Depth refill / resiliency:
Refill^b_t(H) = (Depth^b_{t+H}(K) - Depth^b_{t,after shock}(K)) / H

Book pressure:
Pressure_t = z(OBI_t^K) + z(MicroPrice_t - Mid_t) - z(RelSpread_t)
```

其中 `Consumed^b` 包括打到 bid 的主动卖出和买一撤单，`Consumed^a` 包括打到 ask 的主动买入和卖一撤单。`Refill` 衡量盘口被吃掉后补回来的速度，常用于判断流动性弹性。

更前沿的订单簿特征：

- 盘口斜率：价格档位越远，深度如何变化。
- 深度恢复速度：大单吃掉盘口后，流动性多久补回来。
- 价差状态：一档价差、两档价差、锁盘/交叉盘异常。
- 队列耗尽概率：买一或卖一挂单是否快被吃完。
- 盘口弹性：冲击后 mid-price 是否快速回归。
- 事件时间特征：不用固定秒数，而按订单事件序列建模。

---

## 3. 逐笔委托

逐笔委托记录订单进入市场、撤单和修改的事件。它比盘口快照更接近市场真实机制。

常见事件：

- 新增限价买单。
- 新增限价卖单。
- 撤销买单。
- 撤销卖单。
- 修改价格或数量。
- 市价单或可立即成交的主动单。

逐笔委托可以构造的关键变量：

| 变量 | 含义 |
| --- | --- |
| 新增委托强度 | 单位时间新增多少买/卖订单 |
| 撤单强度 | 单位时间撤单数量和金额 |
| 委托不平衡 | 新增买单和新增卖单的差异 |
| 撤单不平衡 | 买盘撤单和卖盘撤单的差异 |
| Order-to-trade ratio | 委托数量相对成交数量 |
| Queue position | 自己订单在队列中的大致位置 |
| Queue depletion | 前方排队订单被成交或撤掉的速度 |

逐笔委托常用强度变量：

```text
NewBuyIntensity_t(W)  = count(new buy orders in W) / length(W)
NewSellIntensity_t(W) = count(new sell orders in W) / length(W)
CancelBuyIntensity_t(W)  = count(cancel buy orders in W) / length(W)
CancelSellIntensity_t(W) = count(cancel sell orders in W) / length(W)

NewBuyValue_t(W)  = sum(price * size of new buy orders in W)
NewSellValue_t(W) = sum(price * size of new sell orders in W)
```

委托不平衡可以按订单笔数、股数或金额算，面试里要主动说明口径：

```text
NOI_t = (N_new_buy - N_new_sell) / (N_new_buy + N_new_sell)
VOI_t = (V_new_buy - V_new_sell) / (V_new_buy + V_new_sell)
AOI_t = (Amount_new_buy - Amount_new_sell)
        / (Amount_new_buy + Amount_new_sell)
```

撤单不平衡要注意方向。卖盘撤单减少卖压，通常偏多；买盘撤单减少买盘支撑，通常偏空。因此可以定义：

```text
CancelImb_t = (CancelAskVolume_t - CancelBidVolume_t)
              / (CancelAskVolume_t + CancelBidVolume_t)
```

`CancelImb_t > 0` 表示卖盘撤得更多，短期上行压力可能更强；`CancelImb_t < 0` 表示买盘撤得更多，短期下行风险更高。

订单活跃度和噪声交易可用 `Order-to-trade ratio` 粗略衡量：

```text
OTR_t = (NewOrders_t + CancelOrders_t + ModifyOrders_t) / Trades_t
```

`OTR` 很高说明盘口更新频繁但真实成交少，模型更要关注撤单、延迟和队列优先级，而不是只看成交。

队列位置的近似更新：

```text
Q_front(t0) = displayed_size_ahead_of_my_order + hidden_size_estimate

Q_front(t + dt)
= Q_front(t)
  - executions_at_my_price_before_me
  - cancellations_ahead_of_me
  + priority_loss_adjustment

Filled when Q_front(t) <= 0
```

如果没有完整 order id，只能估计 `cancellations_ahead_of_me`，常见做法是按队列比例分配撤单，或者用保守假设把撤单先分配到自己后面，避免高估成交概率。

高频里撤单不是噪声。撤单经常反映流动性提供者对风险的重新评估：卖盘突然撤掉，可能意味着卖方不愿继续提供流动性，价格上行概率上升。

---

## 4. 逐笔成交

逐笔成交记录真实成交事件，包括价格、数量、时间、成交方向等。

核心不是简单看成交量，而是识别主动买入和主动卖出：

```text
主动买入：买方打到 ask，通常表示 aggressive buy。
主动卖出：卖方打到 bid，通常表示 aggressive sell。
```

常见成交因子：

- Signed Volume：带方向的成交量。
- Trade Imbalance：主动买入量和主动卖出量的差。
- VWAP 偏离：成交均价相对 mid-price 或 VWAP 的偏离。
- 大单占比：大额成交是否集中出现。
- 成交簇：短时间内同方向成交是否连续出现。
- 成交后价格冲击：成交发生后 mid-price 是否继续沿同方向移动。

成交方向的常见推断规则：

```text
如果 trade_price >= Ask_{t-}: epsilon = +1  主动买入
如果 trade_price <= Bid_{t-}: epsilon = -1  主动卖出
否则使用 tick rule:
epsilon_t = sign(trade_price_t - trade_price_{t-1})
若价格不变，则沿用上一次非零方向
```

其中 `epsilon_t` 是成交方向，`+1` 表示主动买，`-1` 表示主动卖。基于它可以构造：

```text
SignedVolume_t(W) = sum_{j in W} epsilon_j * volume_j

TradeImbalance_t(W)
= (BuyVolume_t(W) - SellVolume_t(W))
  / (BuyVolume_t(W) + SellVolume_t(W))

VWAP_t(W) = sum_{j in W} price_j * volume_j / sum_{j in W} volume_j

VWAPDeviation_t(W) = (VWAP_t(W) - Mid_{start(W)}) / Mid_{start(W)}
```

成交后的价格冲击可以写成：

```text
Impact_j(H) = epsilon_j * (Mid_{j+H} - Mid_j) / Mid_j
```

如果 `Impact_j(H) > 0`，说明主动买入后价格继续涨、主动卖出后价格继续跌，订单流有延续性；如果很快反转，则更像流动性需求或噪声成交。

订单流毒性可以用响应函数或 Kyle lambda 做近似：

```text
Response(H) = E[epsilon_j * (Mid_{j+H} - Mid_j)]

DeltaMid_t = lambda_Kyle * SignedVolume_t + noise_t
```

`lambda_Kyle` 越大，表示同样的主动成交量会造成更大的价格变化，流动性越脆弱。实务中还会看大单连续扫单、同侧撤单增加、成交后 mid-price 不回落等毒性特征。

逐笔成交的难点：

- 成交方向有时需要推断，行情源未必直接给 aggressor side。
- 同一大订单可能被拆成多笔成交，不能简单当作独立样本。
- 成交本身是条件事件：能成交通常意味着对手方愿意交易，包含逆向选择信息。

---

## 5. 高频因子框架

高频因子一般不是传统的价值、质量、成长，而是围绕微观结构。

### 5.1 盘口压力因子

目标：判断下一跳价格更可能向上还是向下。

典型变量：

- 买卖盘口不平衡。
- 多档深度变化。
- 微价格相对中间价偏离。
- 买一/卖一队列消耗速度。
- 价差扩大或收窄状态。

常用公式：

```text
Pressure_OBI_t = OBI_t^K

MicroSkew_t = (MicroPrice_t - Mid_t) / (Spread_t / 2)

DepthPressure_t
= (sum_k w_k Q^b_{t,k} - sum_k w_k Q^a_{t,k})
  / (sum_k w_k Q^b_{t,k} + sum_k w_k Q^a_{t,k})

SpreadState_t = Spread_t / median(Spread over recent W)

QueuePressure_t
= Depl^a_t(H) - Depl^b_t(H)
```

方向解释：

```text
MicroSkew_t > 0      -> 微价格靠近 ask，价格上跳概率更高
DepthPressure_t > 0  -> 买方深度更厚，短期支撑更强
QueuePressure_t > 0  -> 卖一更快被消耗，上行压力更强
SpreadState_t >> 1   -> 流动性变差，预测信号要扣更高成本
```

### 5.2 订单流因子

目标：判断主动资金方向和短期价格冲击。

典型变量：

- 主动买入成交量。
- 主动卖出成交量。
- 订单流不平衡 OFI。
- 大单连续性。
- 撤单方向不平衡。

OFI 的核心思想是把盘口价格变化、挂单增加、成交消耗和撤单消耗统一成“净买压/净卖压”。它通常比单纯成交量更接近短期价格压力。

Cont-Kukanov-Stoikov 的 best-level OFI 常写成：

```text
e_n =
  1{P^b_n >= P^b_{n-1}} * Q^b_n
- 1{P^b_n <= P^b_{n-1}} * Q^b_{n-1}
- 1{P^a_n <= P^a_{n-1}} * Q^a_n
+ 1{P^a_n >= P^a_{n-1}} * Q^a_{n-1}

OFI_t(W) = sum_{n in W} e_n
```

直觉：

```text
买价上移或买一数量增加 -> 买压增加
买价下移或买一数量消失 -> 买压减少
卖价下移或卖一数量增加 -> 卖压增加
卖价上移或卖一数量消失 -> 卖压减少
```

多档版本 MLOFI 把每一档的订单流变化都纳入：

```text
MLOFI_t^K = [OFI_t^{level 1}, OFI_t^{level 2}, ..., OFI_t^{level K}]

WeightedOFI_t = sum_{k=1}^K w_k * OFI_t^{level k}
```

常用预测基线：

```text
DeltaMid_t(H) = beta_0 + beta_1 * OFI_t(W) / Depth_t(K) + epsilon_t
P(DeltaMid_t(H) > 0) = sigmoid(beta_0 + beta' x_t)
```

这里用 `Depth_t(K)` 标准化，是因为同样 10 万股 OFI 对深度很薄和深度很厚的股票意义不同。

### 5.3 流动性与成本因子

目标：判断能不能交易、交易会花多少钱。

典型变量：

- bid-ask spread。
- 有效价差和实现价差。
- 盘口深度。
- 成交量曲线。
- 参与率。
- 冲击成本。
- 盘口恢复速度。

核心公式：

```text
EffectiveSpread = 2 * d * (P_exec - Mid_decision)

RealizedSpread(H) = 2 * d * (P_exec - Mid_{exec+H})

PriceImpact(H) = 2 * d * (Mid_{exec+H} - Mid_decision)

ImplementationShortfall = d * (P_exec_avg - P_decision) * Q

ParticipationRate = abs(trade_value) / market_volume_or_ADV
```

其中 `d = +1` 表示买入，`d = -1` 表示卖出。`EffectiveSpread` 衡量成交时跨了多少价差，`RealizedSpread` 衡量一段时间后价差收益还剩多少，`PriceImpact` 衡量成交后价格是否沿交易方向继续走。

常见冲击成本经验式：

```text
ImpactCost = k * sigma * (abs(trade_value) / ADV)^alpha * abs(trade_value)
```

`alpha` 通常小于 1，表示冲击随交易规模非线性上升。高频里还要把 `ADV` 换成更短周期的可交易量或盘口深度。

### 5.4 波动与风险因子

目标：控制短周期风险和异常行情。

典型变量：

- realized volatility。
- price jump。
- order book volatility。
- spread volatility。
- intraday drawdown。
- halt/limit-up/limit-down risk。

常用公式：

```text
RealizedVol_t(W) = sqrt(sum_{j in W} r_j^2)

SpreadVol_t(W) = std(Spread_j / Mid_j, j in W)

OBIVol_t(W) = std(OBI_j, j in W)

JumpScore_t = abs(r_t) / RealizedVol_t(W)

IntradayDrawdown_t = Mid_t / max_{u <= t} Mid_u - 1
```

高频波动不是只看价格，也要看盘口状态是否变得不稳定。例如 `SpreadVol` 突然升高，说明交易成本和成交不确定性都在上升，即使方向模型仍然有信号，也可能应该降仓或暂停。

### 5.5 跨品种因子

目标：利用相关市场之间的信息领先关系。

例子：

- 股指期货和指数 ETF。
- 期权隐含波动率和现货。
- 可转债和正股。
- ETF 一篮子成分股和 ETF 本身。
- 商品期货期限结构和相关股票。

跨品种变量通常写成 lead-lag 或 basis：

```text
LeadLag:
r^A_{t+H} = alpha + beta_1 * r^B_t + beta_2 * OFI^B_t + epsilon_t

CrossImpact:
DeltaMid^A_t(H) = sum_j beta_j * OFI^j_t(W) + epsilon_t

ETF basis:
Basis_t = ETFPrice_t - sum_i w_i * ComponentPrice_{t,i}

Futures basis:
Basis_t = Futures_t - Spot_t
AnnualizedBasis = (Futures_t / Spot_t - 1) * 365 / days_to_maturity
```

这类因子最重要的是时间对齐：必须确认 `B` 市场的信息在 `A` 市场下单前已经可见，且扣除延迟、撮合和交易成本后仍有空间。

---

## 6. 高频标签怎么构造

高频标签的构造比日频更敏感。常见标签包括：

### 6.1 下一跳方向

```text
y_t = sign(Mid_{t+1 event} - Mid_t)
```

适合订单簿方向预测，但类别极不平衡时要小心。

### 6.2 固定时间收益

```text
y_t = (Mid_{t + 5s} / Mid_t) - 1
```

问题是高活跃和低活跃股票在 5 秒内事件数量差别很大。

### 6.3 固定事件收益

```text
y_t = (Mid_{t + 50 events} / Mid_t) - 1
```

更接近市场事件流，但不同股票之间时间跨度不可比。

### 6.4 Triple Barrier 标签

设定上边界、下边界和最长持有时间：

```text
先触及止盈 -> 正样本
先触及止损 -> 负样本
到期未触及 -> 中性样本
```

它比简单未来收益更接近真实交易，因为真实策略会考虑止盈、止损和持有时间。

### 6.5 成交/排队标签

用于做市和执行：

- 限价单是否成交。
- 多久成交。
- 成交后是否发生不利价格变动。
- 成交收益能否覆盖逆向选择损失。

更完整的标签写法：

```text
Return label:
y_t(H) = log(Mid_{t+H} / Mid_t)

Direction label with neutral zone:
y_t =
  +1, if y_t(H) > c_t + theta
   0, if abs(y_t(H)) <= c_t + theta
  -1, if y_t(H) < -(c_t + theta)
```

这里 `c_t` 是预计成本，至少应包含半价差、手续费和滑点。加入中性区的原因是：很多微小涨跌无法覆盖交易成本，强行分成涨跌会让模型学到不可交易噪声。

Triple Barrier 的数学形式：

```text
upper_t = Mid_t * (1 + a * sigma_t)
lower_t = Mid_t * (1 - b * sigma_t)
timeout = t + H

y_t =
  +1, if upper_t is touched before lower_t and before timeout
  -1, if lower_t is touched before upper_t and before timeout
   0, otherwise
```

其中 `sigma_t` 可用近期 realized volatility。用波动率缩放边界，比固定百分比边界更适合不同波动状态。

执行和做市更常用成交标签：

```text
FillLabel_t(H) = 1{T_fill <= H}

TimeToFill_t = min{tau: Q_front(t + tau) <= 0}

AdverseSelection_t(H)
= - d_passive * (Mid_{fill+H} - P_fill)

NetPassivePnL_t(H)
= spread_saved_or_rebate - fee - AdverseSelection_t(H)
```

`d_passive = +1` 表示被动买入，`d_passive = -1` 表示被动卖出。`AdverseSelection_t(H) > 0` 表示成交后价格朝自己不利方向走。

Meta-labeling 的常见形式：

```text
Primary model:  side_t in {-1, +1}
Meta label:     z_t = 1{side_t * future_return_t - expected_cost_t > 0}
Final action:   trade only when z_hat_t is high enough
```

它把“方向对不对”和“这笔是否值得交易”分开，适合低信噪比和成本敏感的策略。

---

## 7. 高频建模方法

### 7.1 线性和广义线性基线

高频研究必须先有强基线：

- OFI 回归预测短期 mid-price change。
- Logistic 回归预测下一跳方向。
- Hazard model 预测队列成交时间。
- Kalman filter / state space model 跟踪短期状态。

典型基线公式：

```text
Linear OFI regression:
DeltaMid_t(H) = beta_0 + beta_1 * OFI_t(W) / Depth_t(K) + epsilon_t

Logistic next-tick model:
P(y_t = +1 | x_t) = sigmoid(beta_0 + beta' x_t)

Hazard / survival model:
h(t | x_t) = h_0(t) * exp(beta' x_t)
S(t | x_t) = exp(- integral_0^t h(u | x_t) du)
P(T_fill <= H | x_t) = 1 - S(H | x_t)

State space:
latent_fair_price_t = latent_fair_price_{t-1} + eta_t
observed_mid_t = latent_fair_price_t + noise_t
```

这些基线的价值是可解释、可校准、能快速暴露数据泄漏。复杂模型至少要在相同标签、相同切分、相同成本假设下打败它们。

不要一开始就上深度学习。高频数据噪声极大，如果一个复杂模型不能明显打败 OFI、OBI、microprice 这些基线，通常不值得上线。

### 7.2 Hawkes 过程

Hawkes 过程适合建模“事件会激发后续事件”的市场现象：

```text
一次主动买入，可能提高随后主动买入或价格上跳的强度。
一次撤单，可能触发同侧撤单簇。
```

它适合研究：

- 订单到达强度。
- 撤单强度。
- 成交簇。
- 价格跳变。
- 市场冲击衰减。

单变量 Hawkes 过程：

```text
lambda(t) = mu + sum_{t_i < t} alpha * exp(-beta * (t - t_i))
```

其中：

- `lambda(t)`：事件在 `t` 时刻发生的条件强度。
- `mu`：外生基础强度。
- `alpha`：一次事件对未来事件强度的激发幅度。
- `beta`：激发效应的衰减速度。

多变量 Hawkes 可以同时建模主动买、主动卖、买盘撤单、卖盘撤单、价格上跳、价格下跳：

```text
lambda_i(t)
= mu_i + sum_j sum_{t_n^j < t} alpha_{ij} * exp(-beta_{ij} * (t - t_n^j))
```

`alpha_{ij}` 表示第 `j` 类事件对第 `i` 类事件的激发强度。例如主动买入后，未来主动买入和价格上跳的强度可能上升；卖一撤单后，价格上跳强度也可能上升。为了过程稳定，常要求 branching ratio 小于 1，否则事件强度会爆炸。

### 7.3 Queue-reactive / Markov 模型

这类模型直接把订单簿队列状态作为状态变量，研究：

- 买一/卖一队列如何变化。
- 队列耗尽后价格如何跳动。
- 不同盘口状态下订单到达率如何变化。

它比普通时间序列模型更贴近限价订单簿机制。

简化状态可以写成：

```text
S_t = (Q^b_{t,1}, Q^a_{t,1}, Spread_t, regime_t)
```

每类事件在不同状态下有不同强度：

```text
lambda_new_bid(S_t), lambda_cancel_bid(S_t), lambda_market_sell(S_t)
lambda_new_ask(S_t), lambda_cancel_ask(S_t), lambda_market_buy(S_t)
```

买一队列变化：

```text
dQ^b_t = new_bid_t - cancel_bid_t - market_sell_t
```

卖一队列变化：

```text
dQ^a_t = new_ask_t - cancel_ask_t - market_buy_t
```

价格跳动通常由队列耗尽触发：

```text
tau_bid = first time Q^b_{t,1} <= 0
tau_ask = first time Q^a_{t,1} <= 0

P(next move up | S_t) = P(tau_ask < tau_bid | S_t)
P(next move down | S_t) = P(tau_bid < tau_ask | S_t)
```

这类模型的好处是直接对应“哪一侧队列先被吃完”。缺点是需要较高质量的逐笔委托和成交数据，否则队列状态会估错。

### 7.4 DeepLOB 与 LOB Transformer

DeepLOB 类模型把多档盘口看成二维结构：

```text
价格档位 x 特征通道 x 时间
```

常见结构：

- CNN 提取局部盘口结构。
- LSTM/GRU 提取时间依赖。
- Attention/Transformer 处理长距离事件依赖。
- TCN 处理高频序列卷积。

优点是能自动学习多档盘口之间的非线性关系；缺点是容易受到数据供应商、交易制度、采样方式和时间泄漏影响。

典型输入张量：

```text
X_t in R^{T x K x F}

T: 回看事件数或时间步数
K: 盘口档位数
F: 每档特征，如 price_gap、size、OFI、cancel、trade 等
```

DeepLOB 常见流程：

```text
LOB tensor
-> CNN / Inception block 提取档位局部结构
-> LSTM / GRU / TCN 提取时间动态
-> softmax 输出 {-1, 0, +1} 方向概率
```

分类损失：

```text
L_CE = - sum_i sum_c y_{i,c} * log(p_{i,c})
```

如果类别不平衡，使用加权交叉熵：

```text
L_WCE = - sum_i sum_c w_c * y_{i,c} * log(p_{i,c})
```

Transformer 的注意力核心：

```text
Attention(Q, K, V) = softmax(QK' / sqrt(d)) V
```

LOB Transformer 更适合长事件序列、跨资产联动和 regime 变化，但必须严格做因果 mask，确保第 `t` 个预测不能看到 `t` 之后的订单簿状态。

### 7.5 强化学习

强化学习更适合执行和做市，而不是直接“预测涨跌”。

典型任务：

- 最优拆单：什么时候交易多少。
- 做市报价：bid/ask 报多远、库存如何控制。
- 限价单和市价单选择。
- 执行速度和价格风险权衡。

关键状态：

- 盘口状态。
- 剩余订单量。
- 剩余时间。
- 当前库存。
- 波动率。
- 成交概率。
- 逆向选择风险。

关键奖励：

```text
PnL - 手续费 - 冲击成本 - 库存风险惩罚 - 未完成惩罚
```

执行问题可以写成 MDP：

```text
state_t  = (LOB_t, remaining_qty_t, remaining_time_t, inventory_t, alpha_t, vol_t)
action_t = (market_qty_t, limit_qty_t, quote_offset_t, cancel_or_keep_t)
reward_t = - execution_cost_t - lambda_risk * inventory_t^2 - penalty_unfinished_t
```

Almgren-Chriss 是最优执行强基线：

```text
x_t: t 时刻剩余未成交量
v_t = x_t - x_{t+1}: t 时刻交易量

min_v E[sum_t eta * v_t^2 + gamma * x_t * v_t]
      + lambda * Var(cost)

subject to x_0 = Q, x_T = 0
```

直觉是：`eta * v_t^2` 代表临时冲击，`gamma * x_t * v_t` 代表永久冲击，`lambda * Var(cost)` 代表执行过程中价格波动风险。

做市问题常用 Avellaneda-Stoikov 作为基线。库存为 `q_t`，中间价为 `Mid_t`，风险厌恶为 `gamma`，到期时间为 `T`：

```text
ReservationPrice_t = Mid_t - q_t * gamma * sigma^2 * (T - t)

OptimalSpread_t
= gamma * sigma^2 * (T - t)
  + 2 / gamma * log(1 + gamma / k)
```

库存多头过高时，`ReservationPrice` 下降，模型会更积极卖出、更保守买入；库存空头过高时反过来。真实做市还要加入短期 alpha、成交概率、手续费返佣、tick size 和队列优先级。

### 7.6 生成式与自监督方法

更前沿的方向是先让模型学习市场事件流的表示，再用于下游任务：

- masked event modeling。
- contrastive learning。
- sequence-to-sequence order flow reconstruction。
- synthetic LOB simulation。
- agent-based market simulator。

典型目标函数：

```text
Masked event modeling:
L = - sum_t log P(event_t | visible context)

Sequence reconstruction:
L = sum_t ||LOB_t - LOB_hat_t||^2

Contrastive learning:
L_InfoNCE = - log exp(sim(z_i, z_i+) / tau)
             / sum_j exp(sim(z_i, z_j) / tau)
```

前沿方向的重点不是把模型做大，而是让表示更贴近交易动作：

- survival-based fill probability：直接预测限价单在 `H` 内成交的概率和时间分布。
- cost-aware representation：让模型同时编码方向、价差、深度、冲击和未成交风险。
- cross-asset order flow：把期货、ETF、成分股、期权等订单流放入统一表示。
- generative LOB：生成可回放订单簿，用于压力测试和仿真，但必须检验价差、深度、成交簇、冲击和自相关等 stylized facts。
- agent-based simulator：模拟不同交易者交互，用于测试策略在拥挤、冲击和制度变化下的表现。

这类方法的价值不在于“自动赚钱”，而在于提升数据表示、压力测试和仿真能力。

---

## 8. 多因子模型训练框架

一个成熟的多因子训练框架至少包括这些层。

### 8.1 数据层

必须保证：

- point-in-time，可交易时刻只用当时已知信息。
- 去除幸存者偏差。
- 处理复权、停牌、涨跌停、除权除息。
- 高频数据要处理乱序、重复、丢包、时间戳对齐。
- 统一股票池、行业、市值、可交易约束。

### 8.2 特征层

中低频特征：

- 价值、质量、成长、动量、反转、波动、流动性、情绪。

高频特征：

- OBI、OFI、microprice、spread、depth、queue depletion、trade imbalance、cancel imbalance。

跨频特征：

- 日频 alpha 决定“交易方向和目标仓位”。
- 高频 alpha 决定“何时执行、用什么订单类型、是否拆单”。

### 8.3 标签层

标签要和交易目标一致：

- 选股模型：未来超额收益、行业中性收益、残差收益、排名。
- 高频方向模型：下一跳 mid-price、固定事件收益。
- 执行模型：implementation shortfall、成交概率、逆向选择损失。
- 做市模型：spread capture after adverse selection。

### 8.4 样本切分

不能随机切分。推荐：

- walk-forward。
- rolling training。
- purged K-fold。
- embargo，避免标签窗口重叠导致信息泄漏。
- 按市场状态做分层检验。

高频数据还要避免同一订单流片段同时进入训练和测试，否则模型会记住局部事件。

### 8.5 模型层

从强基线到复杂模型逐层推进：

```text
单因子/线性模型
-> Ridge/Lasso/Elastic Net
-> GBDT/LightGBM/XGBoost
-> 排序模型 LambdaRank/ListNet
-> 深度时序模型 CNN/LSTM/TCN/Transformer
-> 成本感知/强化学习执行模型
```

### 8.6 目标函数

量化不要只优化 MSE。

更贴近交易的目标：

- RankIC 最大化。
- 分组收益单调性。
- top-k precision。
- 加权 MSE，按流动性或风险加权。
- pairwise/listwise ranking loss。
- 交易成本调整后的收益。
- 风险调整收益。
- 执行短缺最小化。

### 8.7 评估层

研究指标：

- IC、RankIC、ICIR。
- 分组收益。
- 多空收益。
- 滚动表现。
- 样本外稳定性。

组合指标：

- 年化收益。
- Sharpe。
- IR。
- 最大回撤。
- 换手。
- 行业/风格暴露。
- 容量。

高频指标：

- hit ratio。
- fill ratio。
- adverse selection after fill。
- realized spread。
- implementation shortfall。
- queue position loss。
- latency sensitivity。

### 8.8 上线层

上线前必须有：

- paper trading。
- shadow trading。
- 实盘和回测差异归因。
- 模型漂移监控。
- 成本漂移监控。
- 数据延迟和缺失监控。
- 风控熔断。
- 日志可回放。

### 8.9 训练框架里的常用公式

**Point-in-time 约束**

```text
x_{t,i} must be measurable with F_t
y_{t,i} uses information after t

valid sample:
x_{t,i} -> decision at t -> trade after t -> y_{t,i}

invalid sample:
x_{t,i} contains price, report, constituent, or label information unavailable at t
```

其中 `F_t` 表示 `t` 时刻已经可知的信息集合。量化里很多“神奇高收益”来自 `x_t` 偷看了 `F_{t+1}`。

**特征处理**

```text
Winsorize:
x_i = min(max(x_i, q_low), q_high)

Z-score:
z_i = (x_i - mean_cross_section(x)) / std_cross_section(x)

Rank transform:
rank_z_i = rank(x_i) / N

Neutralization:
x = B * gamma + epsilon
x_neutral = epsilon
```

`B` 通常包含行业哑变量、市值、Beta、波动率等不希望模型无意暴露的风险来源。高频特征还要按 `Spread`、`Depth`、`ADV` 或日内时段做标准化。

**样本权重**

```text
sample_weight_i
= liquidity_weight_i * freshness_weight_i * label_confidence_i

liquidity_weight_i = min(1, ADV_i / threshold_ADV)
freshness_weight_i = exp(-age_i / half_life)
```

低流动性、标签噪声大、时间太久远的样本可以降低权重，而不是完全等权训练。

**Purged K-fold 和 embargo**

如果样本 `i` 的标签使用区间是：

```text
label_interval_i = [t_i, t_i + H_i]
```

验证集为 `V` 时，训练集中要删除：

```text
purge samples where label_interval_train intersects label_interval_validation
```

并在验证集之后留出缓冲：

```text
embargo_length >= max(H_i) or a conservative leakage horizon
```

这样避免训练样本和验证标签窗口重叠，尤其适合 overlapping return、Triple Barrier 和高频事件标签。

**目标函数**

普通回归：

```text
L_MSE = mean((y_i - y_hat_i)^2)
```

加权回归：

```text
L_WMSE = sum_i w_i * (y_i - y_hat_i)^2 / sum_i w_i
```

pairwise ranking：

```text
L_pair = sum_{i,j} log(1 + exp(-s_ij * (score_i - score_j)))
s_ij = sign(y_i - y_j)
```

成本感知标签：

```text
y_net = future_return - expected_cost - risk_penalty
```

组合感知目标：

```text
maximize E[R_net] - lambda * Var(R_net) - eta * Turnover

R_net,t = w_t' r_{t+1} - Cost(w_t, w_{t-1})
```

执行模型目标：

```text
min E[ImplementationShortfall] + lambda * Var(ImplementationShortfall)
```

**评估指标**

```text
IC_t = Corr(score_{t,*}, y_{t,*})
RankIC_t = Corr(rank(score_{t,*}), rank(y_{t,*}))
ICIR = mean(IC_t) / std(IC_t)

LongShortReturn_t = mean(r_top_group) - mean(r_bottom_group)
Turnover_t = sum_i abs(w_{t,i} - w_{t-1,i}) / 2
Sharpe = mean(R_net) / std(R_net) * sqrt(annualization)
IR = mean(R_p - R_b) / std(R_p - R_b) * sqrt(annualization)
```

高频执行评估：

```text
HitRatio = count(correct direction) / count(predictions)
FillRatio = filled_qty / submitted_qty
CancelRatio = canceled_qty / submitted_qty
RealizedSpread(H) = 2 * d * (P_exec - Mid_{exec+H})
IS = d * (P_exec_avg - P_decision) * Q

LatencyDecay(delta)
= performance_with_latency_delta / performance_with_zero_latency - 1
```

预测指标和交易指标要同时看。一个方向模型 `HitRatio` 高，但如果只在价差很宽、成交很差的时候发信号，净收益可能仍为负。

**漂移和治理**

特征漂移可以用 PSI：

```text
PSI = sum_b (actual_pct_b - expected_pct_b)
            * ln(actual_pct_b / expected_pct_b)
```

模型漂移可以看：

```text
PredictionDrift = PSI(score_live, score_train)
ICDecay = rolling_mean(IC_live) - mean(IC_backtest)
CostDrift = realized_cost - predicted_cost
ExposureDrift = B' * w_live - target_exposure
```

上线阈值要提前定义，例如：

```text
if data_delay > threshold: pause trading
if realized_cost > cost_budget: reduce participation
if rolling_IC < lower_bound: switch to fallback model
if drawdown > risk_limit: trigger kill switch
```

---

## 9. 高频研究检查清单

每做一个高频信号，至少问这些问题：

- 信号是否只使用下单前可见数据？
- 标签窗口是否和特征窗口重叠泄漏？
- 预测收益是否超过半价差、手续费、滑点和冲击？
- 是否考虑了限价单不成交和成交后的逆向选择？
- 回测是否按事件顺序撮合，而不是直接用理想价格？
- 策略容量是多少？
- 延迟增加 1ms、10ms、100ms 后还有效吗？
- 换一个交易日、股票池、市场状态是否仍有效？
- A 股制度下是否允许对应的日内交易路径？
- 极端行情、涨跌停、集合竞价和停牌怎么处理？

---

## 10. 调研依据

这份笔记参考了以下研究脉络，后续做项目时可以继续深入：

- Avellaneda and Stoikov, `High-frequency trading in a limit order book`
- Cont, Kukanov and Stoikov, `The Price Impact of Order Book Events`
- Cont and de Larrard, `Price dynamics in a Markovian limit order market`
- Xu, Gould and Howison, `Multi-Level Order-Flow Imbalance in a Limit Order Book`
- Bacry, Mastromatteo and Muzy, `Hawkes Processes in Finance`
- Huang, Lehalle and Rosenbaum, `Simulating and analyzing order book data: The queue-reactive model`
- Cartea, Jaimungal and Penalva, `Algorithmic and High-Frequency Trading`
- Zhang, Zohren and Roberts, `DeepLOB: Deep Convolutional Neural Networks for Limit Order Books`
- Almgren and Chriss, `Optimal Execution of Portfolio Transactions`
- Lopez de Prado, `Advances in Financial Machine Learning`
- Bailey, Borwein, Lopez de Prado and Zhu, `The Probability of Backtest Overfitting`
- Gu, Kelly and Xiu, `Empirical Asset Pricing via Machine Learning`
