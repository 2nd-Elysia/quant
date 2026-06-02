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

### 5.2 订单流因子

目标：判断主动资金方向和短期价格冲击。

典型变量：

- 主动买入成交量。
- 主动卖出成交量。
- 订单流不平衡 OFI。
- 大单连续性。
- 撤单方向不平衡。

OFI 的核心思想是把盘口价格变化、挂单增加、成交消耗和撤单消耗统一成“净买压/净卖压”。它通常比单纯成交量更接近短期价格压力。

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

### 5.4 波动与风险因子

目标：控制短周期风险和异常行情。

典型变量：

- realized volatility。
- price jump。
- order book volatility。
- spread volatility。
- intraday drawdown。
- halt/limit-up/limit-down risk。

### 5.5 跨品种因子

目标：利用相关市场之间的信息领先关系。

例子：

- 股指期货和指数 ETF。
- 期权隐含波动率和现货。
- 可转债和正股。
- ETF 一篮子成分股和 ETF 本身。
- 商品期货期限结构和相关股票。

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

---

## 7. 高频建模方法

### 7.1 线性和广义线性基线

高频研究必须先有强基线：

- OFI 回归预测短期 mid-price change。
- Logistic 回归预测下一跳方向。
- Hazard model 预测队列成交时间。
- Kalman filter / state space model 跟踪短期状态。

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

### 7.3 Queue-reactive / Markov 模型

这类模型直接把订单簿队列状态作为状态变量，研究：

- 买一/卖一队列如何变化。
- 队列耗尽后价格如何跳动。
- 不同盘口状态下订单到达率如何变化。

它比普通时间序列模型更贴近限价订单簿机制。

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

### 7.6 生成式与自监督方法

更前沿的方向是先让模型学习市场事件流的表示，再用于下游任务：

- masked event modeling。
- contrastive learning。
- sequence-to-sequence order flow reconstruction。
- synthetic LOB simulation。
- agent-based market simulator。

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
- Bacry, Mastromatteo and Muzy, `Hawkes Processes in Finance`
- Huang, Lehalle and Rosenbaum, `Simulating and analyzing order book data: The queue-reactive model`
- Zhang, Zohren and Roberts, `DeepLOB: Deep Convolutional Neural Networks for Limit Order Books`
- Almgren and Chriss, `Optimal Execution of Portfolio Transactions`
- Bailey, Borwein, Lopez de Prado and Zhu, `The Probability of Backtest Overfitting`
- Gu, Kelly and Xiu, `Empirical Asset Pricing via Machine Learning`
