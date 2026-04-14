# Polymarket AI LP 做市机器人 — 项目文档

**版本**：v0.3（整合需求 + 筛选策略 + 执行策略 + 关键决策）
**整理时间**：2026-04-14
**状态**：需求与策略已对齐，待 Phase 0 启动

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 关键决策汇总](#2-关键决策汇总)
- [3. 需求文档](#3-需求文档)
- [4. 标的筛选策略 v1](#4-标的筛选策略-v1)
- [5. 挂单执行策略 v1](#5-挂单执行策略-v1)
- [6. 数据与持久化](#6-数据与持久化)
- [7. 安全与运维](#7-安全与运维)
- [8. 回测计划](#8-回测计划)
- [9. 实施路线](#9-实施路线)
- [10. 待定问题](#10-待定问题)

---

## 1. 项目概述

### 1.1 背景
Polymarket（PM）是一个预测市场平台，平台对 LP 挂单（做市）给予 reward 奖励。目标是构建一套 **AI 辅助的自动做市系统**，让 AI 在安全的前提下代替人工执行策略。

### 1.2 收益模型
```
日 reward = Σ_标的 [ reward_per_day(i) × own_share(i) × time_in_zone(i) ]

own_share   = 自己挂单量 / (自己挂单量 + 其它人深度)
time_in_zone = 当天挂单在奖励区间内的累计时长
```

### 1.3 目标与非目标

**目标**
- 自动化执行已验证策略，扩展至多账号可并行
- 利用 AI 从既有策略推导过程出发，探索新增益规则
- 所有运行数据可回溯，所有决策有据可查

**非目标**
- 主动方向性下注（不赌价格涨跌，只赚 reward）
- 跨平台套利
- 资金出入金自动化（入金人工管理）

---

## 2. 关键决策汇总

| 类别 | 决策 | 备注 |
|---|---|---|
| AI 开发工具 | **Claude Code** | Anthropic 官方 CLI，本地运行 |
| 起始资金 | **200–500U** | 先跑 1 个号 |
| MVP 策略 | **Tier A 低风险** | 超远期标的 |
| 风格档位 | **保守** | target_own_share=5%，外沿缩 2 tick，≤8 标的 |
| 塞单哲学 | **预防优先** | 选标的 + 挂单位置做好；自动卖出是第二道防线 |
| 开发模式 | **Claude Code 编写 + 用户指导** | |
| 语言 | **Python 3.11+** | asyncio；生态好，AI 写得快 |
| 数据库 | **SQLite (WAL 模式)** | 所有数据落盘，不依赖内存；详见 §6 |
| 运行环境 | **Mac mini 本地常驻** | |
| 启动方式 | **launchd 自启 + 手动输密码** | 程序启动但未解锁，登录 Mac 后手动输密码 |
| 密码管理 | **仅内存，不落盘** | 每次启动手动输入，不通过任何通讯通道 |
| 日志 | **本地 rotating file** | 按天切割，保留 30 天 |
| 异常告警 | **Telegram（专用新 bot）** | 只推异常事件，开发后期接入 |

---

## 3. 需求文档

### 3.1 角色
- **策略制定者**：制定筛选规则、参数阈值，审核 AI 产出的新规则
- **运营者**：监控多账号收益、处理告警
- **AI Agent**：执行挂单、调单、塞单处理

### 3.2 功能清单

#### FR-SEC 账户与密钥安全（P0）
- 每钱包独立 keystore JSON，高强度密码加密
- 密码不入库、不写盘、不存服务器，运行时内存输入用完即焚
- 签名在本进程内完成，私钥不出内存

#### FR-SEL 标的筛选（P0）
按 6 维度打分 + 硬门槛，见 §4

#### FR-EXE 挂单执行（P0）
按定价 + 份数 + 深度三重约束挂单，见 §5

#### FR-MON 收益监控（P1）
- 实时：账号 × 标的 维度展示挂单份数、占比、已到手 reward、未结算
- 历史：所有挂单 / 撤单 / 塞单事件结构化入库
- 告警：回撤超阈、塞单未平、keystore 异常、余额异常

#### FR-AI AI 学习迭代（P1）
- 知识库喂养：会议文档 + 推导过程 → 向量库
- 策略演化：基于实盘数据产出"规则调整建议"，人审核入库

### 3.3 非功能需求
- 安全：日志不打印私钥/密码，keystore 权限 0600
- 可靠：断网重连、下单幂等、进程重启可恢复挂单状态
- 可观测：所有事件结构化日志 + 数据库
- 性能：单号 20+ 标的并行，调单周期 ≤ 30s

---

## 4. 标的筛选策略 v1

### 4.1 核心思路
LP 收益公式拆成三个独立目标**同时满足**：
1. **Reward 足够高**（平台肯花钱）
2. **自身占比能做上去**（深度不能太厚）
3. **塞单成本期望低**（低波动 + 远到期 + 盘口稳）

### 4.2 六大筛选维度

#### 维度 1｜到期时间 T
| 分档 | T 范围 | 评分 |
|---|---|---|
| 超远期 | T ≥ 180 天 | 10 |
| 远期 | 60 ≤ T < 180 | 6 |
| 中期 | 14 ≤ T < 60 | 3 |
| 近期 | T < 14 | 0（过滤） |

**Tier A 硬门槛**：T ≥ 90 天

#### 维度 2｜挂单深度 D（USD）
定义：盘口前 5 档 yes+no 累计挂单美元数

| 分档 | D 范围 | 评分 |
|---|---|---|
| 黄金区 | 2k–20k | 10 |
| 次优 | 20k–100k | 6 |
| 太薄 | < 2k | 2 |
| 太厚 | > 100k | 3 |

**硬门槛**：1,000 ≤ D ≤ 100,000，且自身预期占比 ≥ 2%

#### 维度 3｜波动率 σ
定义：最近 7 天价格标准差 / 当前中间价（价格 >90 或 <10 改用最近 7 天最大跳动幅度）

| 分档 | σ 范围 | 评分 |
|---|---|---|
| 极稳 | ≤ 0.5% | 10 |
| 稳 | 0.5%–2% | 7 |
| 一般 | 2%–5% | 3 |
| 波动大 | > 5% | 0 |

**硬门槛**：σ ≤ 2%；24h 内出现过单笔 >5% 跳动 → 48h 拉黑

#### 维度 4｜挂单收益率 R（APR）

收益率受三个因子共同决定，需把 PM 提供的"奖池总量"和市场已有"参与资金量"作为独立输入：

```
变量定义（均从 PM API 实时取）：
  reward_fund_daily  = total_daily_rate            # 该标的每日总奖励池（USD）
  total_LP_capital   = Σ(qty_i × price_i)          # **仅 reward zone 内** 已有 LP 总资金（USD）
                                                   # zone = [mid − rewards_max_spread, mid + rewards_max_spread]
                                                   # 包含 bid 和 ask 两侧；区外的挂单不算（不分蛋糕）
  my_capital         = capital_per_market           # 我准备投入的资金（USD）

派生指标：
  expected_own_ratio = my_capital / (my_capital + total_LP_capital)
  R_est_APR          = reward_fund_daily × expected_own_ratio × 365 / my_capital
```

**直觉**：
- `reward_fund_daily` 越大 → 蛋糕越大
- `total_LP_capital` 越小 → 我吃得越多（拥挤度低）
- 二者比值 = "**奖励密度**"，是这个市场的内在收益率

**子门槛（与 R 一同必须满足）**：
- `reward_fund_daily ≥ 5 USD/day`（市场太小不值得碰）
- `total_LP_capital ≤ 50,000 USD`（市场太挤，自己份额≤1% 没意义）
- `expected_own_ratio ≥ 2%`（占比太低就不挂）

**最终评分仍按 R_est_APR**：

**D-034 起改用日收益率 `R_daily`**（`= reward_fund_daily × expected_own_ratio / my_capital`，不再除 365）：

| 分档 | R_daily 范围 | 评分 |
|---|---|---|
| 优 | ≥ 0.5% | 10 |
| 良 | 0.4–0.5% | 8 |
| 合格 | 0.3–0.4% | 6 |
| 勉强 | 0.2–0.3% | 4 |
| 差 | < 0.2% | 0 |

**硬门槛（D-034）**：`R_daily ≥ 0.3%`（≈ 109% APR）且上述 3 项子门槛全部满足。旧的 `reward_fund_daily ≥ 5 USD/day` 子门槛已被日 R 门槛覆盖，已删除。

**与维度 2（挂单深度 D）的区别**：
- D = 盘口前 5 档美元数（关注**价格抗冲击能力**，反塞单）
- total_LP_capital = zone 内全部参与方资金（关注**奖励稀释程度**，反"分蛋糕的人太多"）
- 一个标的可能 D 很高但 total_LP_capital 不大（深度集中在少数大单），也可能反过来——两个指标必须分开看

#### 维度 5｜价格位置
| 价格 | 模式 | 备注 |
|---|---|---|
| 10 ≤ p ≤ 90 | 单/双向均可 | 默认双向 |
| 90 < p 或 p < 10 | **必须双向** | 平台规则 |
| p ≥ 99 或 p ≤ 1 | 过滤 | 极端区流动性死 |

#### 维度 6｜奖励点差
| reward_spread | 评分 | 挂单位置（保守档） |
|---|---|---|
| ±4 | 10 | mid ± (4−2)= mid ± 2 |
| ±3 | 8 | mid ± 1 |
| ±2 | 5 | Tier A 不做 |

**Tier A 硬门槛**：reward_spread ≥ ±3。Tier B（体育，D-032/D-033/D-034）放宽到 ±2。

#### 类别门槛分流（D-034）
标的通过 `market.tags` 识别类别：

- `sports/NFL/NBA/…` 等前缀 → **SPORTS_GATES**：`hours_to_kickoff ≥ 5h`、`σ ≤ 5%`、`p ∈ [5, 95]`、`reward_spread ≥ ±2`，其余门槛同长期。
- 其它 → **LONG_DATED_GATES**：`T ≥ 90d`、`σ ≤ 2%`、`p ∈ [1, 99]`、`reward_spread ≥ ±3`。

两类标的合并进**单一候选池**按 Score 排序选 Top-N（D-032）。

### 4.3 综合打分
```
Score = 0.15·T + 0.15·D + 0.30·σ + 0.20·R + 0.10·价格位置 + 0.10·点差   # D-034 权重
```

### 4.4 硬性准入条件（必须全部满足）
- T ≥ 90 天
- 1,000 ≤ D ≤ 100,000（盘口前 5 档美元数，反塞单视角）
- σ ≤ 2%；最近 24h 无 >5% 跳动
- p ∈ [1, 99]
- reward_spread ≥ ±3
- **total_LP_capital_in_zone ≤ 50,000 USD**（市场太挤不碰）
- **expected_own_ratio ≥ 2%**（自身在 zone 内份额）
- **R_est_daily ≥ 0.3%**（D-034，取代 `R_APR ≥ 5%` 与 `reward_fund_daily ≥ 5`）
- **挂单价前方资金存量 ≥ 500 USD**（反塞单）
- **挂单价前方份数存量 ≥ 平台最小挂单份数 × 3**（反塞单）

### 4.5 分级入池（保守档）
| Score | 动作 |
|---|---|
| ≥ 8.0 | **A 级**：高仓位（最多 25U/标的） |
| 6.5–8.0 | **B 级**：常规仓位（10–15U） |
| 5.0–6.5 | **C 级**：观察池（不挂，每小时重算） |
| < 5.0 | 丢弃 |

**单号同时挂单 ≤ 8 个标的**（保守档）。

### 4.6 筛选节奏（每日 + 盘中淘汰）

**节奏**：
- **每日筛选**：每天 **UTC+8 08:00（= UTC 00:00）** 跑一次完整筛选 → 产出当日候选池
  - 与用户所在时区（北京时间）的工作节奏对齐
  - 默认时点 `screener.daily_run_utc: "00:00"`，可在 `configs/default.yaml` 调整
  - 假设：PM 官方每日 reward 更新发生在该时点之前（用户经验，待 Phase 1 数据观察确认）
- **盘中不再做新增**：当日只在已选池内执行，**不引入新标的**
- **盘中淘汰**：对已选池每 5 分钟做一次健康检查，触发任一退出条件 → **撤单 + 移出池 + 释放资金**，资金在当日仅作为现金留存（不补仓新标的）

**退出条件（任一触发即淘汰）**：
- 24h 内出现 >5% 价格跳动
- σ 实时滑动窗口（最近 4h）超过 3%
- depth D 跌出 [800, 120,000] 区间（10% 缓冲）
- reward_spread 变化导致 reward zone 收窄到我们挂单位置已不在 zone 内
- total_LP_capital_in_zone 涨过 80,000 USD（拥挤度恶化 60%）
- expected_own_ratio 跌破 1%
- 标的距到期 < 30 天
- 标的状态变为 resolved/closed

**TG 推送**（每日筛选完成后）：
推送内容固定格式：
```
📊 每日筛选 YYYY-MM-DD UTC

✅ 入池（N 个，总分配 X USD）
  1. <slug>  Score=8.4  R=15%  T=210d  zone=$12k
  2. ...

⏸️ 仍在池中（沿用昨日，无变更）
  ...

❌ 移出（盘中淘汰记录）
  - <slug>  原因：σ 突破 3%，于 14:23 撤单
  - ...

🔍 评估但未入池（前 5）
  - <slug>  Score=6.2  缺：reward_fund_daily 仅 3 USD/day
  - ...

💼 当前账户：余额 X USD，已挂 Y USD，现金 Z USD
```

---

## 5. 挂单执行策略 v1

### 5.1 三杠杆取舍（Tier A 保守档）
`time_in_zone > 塞单成本 > own_share`

### 5.2 挂单定价
**挂单价 = 奖励区外沿内缩 2 tick**（保守档）

| reward_spread | bid | ask |
|---|---|---|
| ±4 | mid − 2 | mid + 2 |
| ±3 | mid − 1 | mid + 1 |

### 5.3 份数计算（四重约束取 min）

**D-034 Bug fix**：双向挂单成本 ≈ `qty × $1`（不是 `qty × price × 2`）。PM 机制：要么先用 USDC 预铸 yes+no（每对 ≈ $1），要么走 yes-bid + no-bid 对挂（两侧成本之和 ≈ $1）。旧公式 `capital / 2 / price` 在 `price=0.25` 下会给出 2 倍错误值。

```python
# D-034 修正后
qty_by_capital = (
    capital_per_market / 1.0              # 双向：cost ≈ qty × $1
    if double_sided
    else capital_per_market / price       # 单向
)
qty_per_side = min(
    qty_by_capital,
    qty_by_depth      = other_depth_in_zone × 0.10,
    qty_by_own_ratio  = other_depth × 0.05 / (1 − 0.05),   # target=5%
    max_qty_cap       = 总资金 × per_market_max_ratio / 1.0,  # 双向统一 $1/share
)
```

实现见 `pmbot.screening.sizing.qty_by_capital`（单元测试 `tests/test_sizing.py` 覆盖双向/单向/回归旧公式）。

**硬门槛**：若 qty < platform_min_qty → 放弃该标的，不凑数（**不挂**）

### 5.4 双向 vs 单向
| 情景 | 策略 |
|---|---|
| 10 ≤ p ≤ 90 | 双向（默认） |
| p > 90 或 p < 10 | 强制双向 |
| 单向 | 仅 "爆量模式" 开启时；Tier A **禁用** |

### 5.5 深度三段法（反塞单核心）
```
[ front_depth ]  [ 自己 ]  [ back_depth ]
    保护墙                    逃生通道
```

**入场门槛**：
- front_depth ≥ 3× 自己挂单量（至少 3 层保护）
- back_depth ≥ 1× 自己挂单量（被塞后有路可逃）
- self_share ≤ 15%
- 其它人深度 ≥ 500 USD
- **挂单档位前方同价排队 ≥ 500 USD** 且 ≥ 平台最小挂单份数 × 3

**盘中监控**（每次调单循环）：
| 事件 | 动作 |
|---|---|
| front_depth 跌破 2× 自己量 | **撤单**，冷却 10 分钟 |
| back_depth 清零 | 撤单等 back 重新形成 |
| self_share 升至 >20% | 减量重挂 |
| 对向盘口剧烈倾斜（一侧 ≥70%） | 撤单 |

### 5.6 调单循环

**正常节奏**：每 5 分钟扫一次。

**伪码**：
```
for market in in_pool_markets:
    snap = fetch_orderbook_AND_metadata(market)   # 必须实时拉元数据
    if not still_in_reward_zone(my_order, snap):
        cancel_and_replace(market, snap); continue
    if depth_thresholds_violated(snap):
        cancel(market); add_cooldown(market, 10min); continue
    if optimal_price_moved(my_order, snap) >= 1 tick:
        cancel_and_replace(market, snap)
```

**事件驱动**（绕过 5 分钟节奏）：
| 事件 | 触发 |
|---|---|
| mid 60s 内移动 ≥ 1% | 立即对该标的重算 |
| 自己挂单部分成交 | 立即评估剩余 |
| reward 参数变化 | 立即全池重扫 |
| 标的到期 < 30 天 | 从池中剔除 |

### 5.7 time_in_zone 最大化技巧
- 挂单还在 zone 内 + 深度未违规 → **不动**（撤就归零排队）
- **PM 无原子 replace**（Phase 0 调研确认）：所有重挂都是 cancel + post，队列优先级必然丢失 → 更要克制频繁重挂
  - 好消息：PM reward 按一周 **10,080 次 1 分钟采样**计分，"累计在 zone 内的时长"比"队列最前"更关键
- mid 在 ±1 tick 内抖动不触发重挂（可调）
- **Relayer 限流 25 req/min**：下单/撤单都消耗配额，重挂太频繁会被限流 → 每分钟撤重挂不超 10 次为宜

### 5.8 塞单处置（第二防线）

```
on_filled(market, side, filled_qty, price):
    1. 暂停该标的新挂单
    2. fetch_orderbook_snapshot()
    3. 分析对手盘真实性:
         - 卖一/买一档位过去 60s 挂单量变化率
         - < 20% 认为真实
         - > 50% 认为诱单
    4. 真实 → 限价挂卖一/买一平仓
    5. 诱单 → 挂 mid 平仓，最多等 10 分钟
    6. 超时 → 告警人工处理（不市价扫平）
    7. 该标的进入 30 分钟冷却
```

**止损边界**：
- 单标单次塞单亏损 > capital_per_market × 30% → 强制平仓告警
- 当日塞单总亏损 > 总资金 × 3% → 熔断撤全部挂单

### 5.9 资金分配（D-034 σ 加权档）
```
总资金            = $500（D-034 从 300 上调）
保留现金          = 10%
可挂资金          = 90% → $450
入池上限          = 20 个标的（D-034 从 8 上调）
单标基础上限      = 总资金 × 8%（D-034 从 5% 上调） → $40
单标最低分配      = $5（低于则 skip 该标的，不凑数）

σ 加权单标上限（D-034）：
  σ ≤ 0.5%  → × 1.6（=$64）
  σ ≤ 1.0%  → × 1.2（=$48）
  σ ≤ 2.0%  → × 1.0（=$40，默认）

分配伪码（按 Score 降序遍历入池名单）：
  base_cap   = 500 × 0.08
  mult       = sigma_multiplier(market.σ)
  market_cap = base_cap × mult
  avg        = remaining_cash / remaining_slots
  alloc      = min(market_cap, avg)
  if alloc < per_market_min_usd: skip（不烧槽位）
  else: market.allocate(alloc); remaining_cash -= alloc
```

---

## 6. 数据与持久化

### 6.1 核心原则
> **所有执行指标与状态全部落盘到数据库，决策不依赖内存**。
> 进程重启后必须能从数据库完整恢复工作状态。

### 6.2 数据库选型：SQLite + WAL 模式

**用户疑虑**：多标的并发写入，SQLite 只能一个写会不会有问题？

**回答**：**不会有问题**，原因：
1. 本项目是 **单进程**（一个 bot 进程），所有写入经由同一进程内的 DB 连接串行化 → 天然不冲突
2. SQLite 在 WAL 模式下单机写入吞吐可达数千 TPS，远超我们需求（估计 < 50 写/秒）
3. 读可以和写并发（WAL 的优势）
4. 单文件，无服务进程，崩溃恢复依赖 journal，工业实战可靠

**为什么不选 PostgreSQL**：
- 需要装 + 起服务 + 维护
- 我们没有跨进程、跨机器、跨网络读写的场景
- 体量不需要（总数据预期 < 10GB/年）

**兜底方案**：SQLite 性能/并发遇到瓶颈时，迁移到 PostgreSQL 只需改连接串 + ORM，代码不变（用 SQLAlchemy）。

### 6.3 数据表规划（草稿）

| 表名 | 用途 | 估算写频 |
|---|---|---|
| `markets` | 标的元数据（到期、reward_spread、min_qty 等），实时同步 | 标的数 × 1/分钟 |
| `orderbooks` | 盘口快照（前 20 档 + zone 聚合） | 两阶段：daily × |U\|、30s × |pool\| |
| `scores` | 每次筛选的评分记录 | 标的数 × 1/10分钟 |
| `my_orders` | 自己挂单的全生命周期（创建/修改/撤销/成交） | 挂单事件驱动 |
| `fills` | 塞单/成交事件 | 低频 |
| `pnl_snapshots` | 账户总值快照 | 1/分钟 |
| `rewards` | 每日 reward 到账记录 | 1/日 |
| `events` | 所有关键事件（进入池、移出池、熔断触发等） | 事件驱动 |
| `config_history` | 参数变更历史（便于对比回测） | 变更时 |

### 6.3.1 两阶段 orderbook 采集（Phase 2.5, D-035）

自 Phase 2.5 起 `orderbook_collector` 支持三种 `scope`：

- **`universe`**：每天 `screener.daily_run_utc` 跑一次（启动时若距上次筛选
  ≥ `catchup_threshold_hours` 立刻补跑一次）。抓取 DB 中所有 rewards-enabled
  active market 的双向盘口 top-20 档 + zone 聚合 → 写入 `orderbooks` → 立刻
  调用筛选器 → 生成当日 `candidate_pool` → 推送 TG 日报。
  并发默认 15，总耗时实测 ~3–6 分钟（受网络制约；直连理想约 60s）。
- **`pool`**：生产常驻循环，每 `collectors.orderbook.pool_interval_sec`（默认 30s）
  只抓当日 `candidate_pool` 中 status ∈ (NEW, RETAINED) 的标的 → 为 intraday
  monitor 提供新鲜盘口数据；因只 20×2=40 次 fetch/cycle，对 PM API 压力最小。
- **`legacy`**：Phase 1 行为（按 `reward_daily_usd` 降序取 top-N 且循环拉取），
  仅保留做 fallback / 调试用，`legacy_enabled=false` 默认关闭。

Runner 层由 `_universe_scanner_loop`（含 startup catch-up + 每日调度）+
`_screener_loop`（兜底，若 universe scan 已触发筛选则跳过）+ pool collector +
`_monitor_loop` 并行独立运行。

### 6.3.2 Gamma tag 富化采集（Phase 2.6, D-037）

`tags_collector`（`src/pmbot/collectors/tags_collector.py`）独立 asyncio
任务，每 `collectors.tags.interval_sec`（默认 1 小时）跑一次。流程：
1. 选取 `markets.tags IS NULL OR tags=[]` 的行（每周期上限 500）
2. 调用 Gamma `/markets?condition_ids=...`（batch=50）拿到市场行 + 嵌套
   `events[0].slug`
3. 逐 slug 调用 `/events/slug/{slug}` 取 tag 列表（cycle 内缓存）
4. 扁平化成 slug 字符串列表回写 `markets.tags`；如市场行自带
   `gameStartTime` 一并写 `markets.game_start_time`
5. emit `TAGS_FETCHED` 事件

一次性全量回填：`pmbot tags-backfill` CLI 走同一循环直到 target 为空。
筛选器的 `is_sports_market(tags)`（`pmbot.screening.gates`）在 tag
富化后才能生效 → D-034 的 `SPORTS_GATES` 终于不再永远走不到。

### 6.4 数据完整性
- WAL 模式 + `synchronous=NORMAL`：崩溃不丢已提交数据
- 每小时自动备份 DB 文件到独立目录
- 关键表加时间戳索引

---

## 7. 安全与运维

### 7.1 密钥管理
- 钱包以 keystore JSON 形式存储，文件权限 0600
- 密码**仅在进程内存**，**每次启动手动输入**
- **绝不**通过任何通讯通道（Telegram / 邮件 / SSH）传输密码
- 日志过滤私钥、密码、keystore 内容

### 7.2 启动流程（Phase 3c, D-049 三层 live opt-in）
```
# 默认安全模式：paper mode，无密码
pmbot run --paper

# 实盘启动：三层授权缺一不可
# 1. CLI flag       --live
# 2. 终端解锁        getpass 提示输入 keystore 密码
# 3. 首单确认        --confirm-real-trade （进程生命周期首单必须带）
pmbot preflight              # 12 项绿/红 checklist，全绿才继续
pmbot run --live --confirm-real-trade
# → 提示密码 → unlock → proxy detect → L2 API creds → balance 检查 → daemon 起
```

**安全不变量**：
- `paper_mode=true` 是所有代码路径默认
- Paper 模式永不 touch 私钥 / web3 / CLOB
- SIGINT/SIGTERM 时调 `keystore.lock()`，私钥 bytearray 置 0
- 进程重启后首单 gate 重新上锁（`_live_confirmed` 仅 in-memory）
- 密码只走 `getpass`，绝不接受 env / file / TG / log

**如果 Mac 意外重启**：bot 自动起，但除非你用 `--live` 否则不下真单；`--live` 需人工输密码。

详细步骤见 `docs/going_live_checklist.md`。

### 7.3 日志
- 路径：`./logs/pmbot.YYYY-MM-DD.log`，按天切割
- 级别：INFO（正常运行）+ WARN（异常但未触发告警）+ ERROR（触发告警）
- 保留 30 天，超期自动清理

### 7.4 告警
- **专用 Telegram bot**（后期接入，不复用当前对话 bot）
- 推送事件：
  - 熔断触发（日回撤超阈）
  - 连续塞单失败（3 次以上处置失败）
  - 钱包余额异常变动
  - API 连续失败 >5 分钟
  - 启动后 30 分钟未收到解锁

### 7.5 回撤保护
| 阈值 | 动作 |
|---|---|
| 日回撤 > 3%（默认，待 A5 调） | 暂停新挂单，保持已挂 |
| 日回撤 > 5%（默认，待 A5 调） | **熔断**：撤全部挂单，推送告警等人工 |
| 单标塞单亏 > 仓位 × 30% | 强制平仓并告警 |

---

## 8. 回测计划

### 8.1 数据采集（Phase 0）
- 订阅 PM 全市场订单簿 + 成交 + reward 参数快照（每 30s 一份）
- 落盘：mid、bid/ask 前 10 档、reward_spread、reward_daily、成交流水
- 目标：先积累 7–14 天历史数据

### 8.2 回测参数扫描
```yaml
筛选阈值:
  T_min:           [30, 60, 90, 180]  # 天
  sigma_max:       [1%, 2%, 3%, 5%]
  R_min:           [3%, 5%, 8%, 10%]
  other_depth_min: [200, 500, 1000, 2000]  # USD

执行参数:
  edge_offset_ticks: [0, 1, 2]
  target_own_share:  [4%, 6%, 8%, 10%, 12%]
  front_multiplier_min: [2, 3, 5]
  front_queue_min_usd:  [100, 300, 500, 1000]

资金规模:
  capital: [200, 350, 500]  # USD
```

### 8.3 输出指标
每组参数给 5 个维度：
1. 日化收益率（中位 + P10/P90）
2. 塞单频次
3. 塞单平均亏损
4. time_in_zone 占比
5. 综合得分 = 收益 × (1 − 塞单成本率)

### 8.4 结果展示
- Top 10 参数组合对比表
- 热力图（收益 vs 塞单率，按 target_own_share × edge_offset）
- 3 套推荐方案：稳健 / 平衡 / 激进 —— 供人工审核

### 8.5 协作节奏
1. 我搭框架 + 采数据
2. 跑默认参数出基线
3. 同步数据 → 一起 review
4. 按 review 意见迭代阈值
5. 二轮回测 → 纸面 48h → 小额实盘 72h

---

## 9. 实施路线

| Phase | 交付 | 周期（估） |
|---|---|---|
| **P0 基础设施** | keystore 管理、日志、DB schema、PM API 接入调研 | 2–3 天 |
| **P1 数据采集** | 订单簿/元数据/reward 参数实时入库 | 2 天 |
| **P2 筛选引擎** | 6 维度打分 + 硬门槛；每 10 分钟扫全市场 | 3 天 |
| **P2.5 两阶段采集** | 拆分 universe scan（日）/ pool tracking（30s），`pmbot universe-scan` CLI ✅ | 0.5 天（2026-04-14 完工, D-035） |
| **P3 回测框架** | 用 P1 积累的数据回测，网格搜索参数 | 3–4 天 |
| **P4 挂单引擎** | 下单 / 调单 / 塞单处置 | 4–5 天 |
| **P5 纸面模拟** | 不真下单，跑 48h 验证逻辑 | 2 天 |
| **P6 小额实盘** | 200U 全量 72h 监控 | 3 天 |
| **P7 参数调优** | 根据实盘数据迭代阈值 | 持续 |
| **P8 告警与运维** | Telegram 告警 bot、launchd 自启、备份 | 2 天 |

（周期仅供参考，实际看 AI 编码速度 + 调研反馈）

---

## 10. 待定问题

### 10.1 策略参数待定（先用默认回测，再调整）
- **A3 mid 抖动容忍**：±1 tick（默认），待数据给出最优
- **A4 塞单后冷却时间**：30 分钟（默认），待数据
- **A5 回撤熔断阈值**：日 −3% 暂停 / −5% 熔断（默认），待数据

### 10.2 技术调研结论（Phase 0 已完成，详见 `docs/pm_api_research.md`）
- ❌ **PM 不支持** 原子 `replace` —— 所有重挂都是 cancel+post，队列重排（好消息：reward 按周 10080 次采样，时长累计型，队列位置不致命）
- ❌ **reward 无 push**：需要 REST 轮询 `GET /rewards/markets/current`，推荐 60–120s 周期
- ✅ **min_qty 可取**：rewards endpoint 每标返回 `rewards_min_size`（份数）和 `rewards_max_spread`（tick 数）
- ⚠️ **历史数据部分可用**：mid 历史有（`/prices-history`，1 分钟粒度），**盘口深度历史没有** → 深度相关回测必须自采
- ⚠️ **新风险**：PM 计分用 "adjusted midpoint"（过滤 dust 后），可能与我们观测的 mid 不同 —— 需实盘实验确认
- ⚠️ **Relayer 限速 25 req/min**：撤 + 下 都消耗配额，强化"不要无谓重挂"的规则
- ⚠️ **proxy wallet `signature_type`**：0/1/2 三种，配错下单会被拒；trade 前需链上确认钱包是否有 Safe proxy

### 10.3 SDK 选型结论
- **主用 py-clob-client v0.34.6**（Feb 2026）覆盖 ~95% 需求
- **补充**：
  - `web3.py` — 一次性 token allowance 授权
  - `httpx` — 调用 Gamma / Data API（py-clob-client 未覆盖的元数据）
- MVP 不需要直接调 CTF 合约

### 10.3 待实盘验证
- target_own_share = 5% 是否保守过头
- σ 阈值 2%（Tier A）是否合理
- 虚假挂单（诱单）识别 heuristic 的真实效果

---

## 附录 A：关键术语

| 术语 | 定义 |
|---|---|
| Tier A | 低风险超长周期标的（28 年大选、世界杯、外星人等） |
| reward_spread | 平台给的奖励点差（如 ±4 表示 mid±4 范围内挂单才有奖励） |
| reward_zone | 挂单能拿 reward 的价格区间 = [mid − spread, mid + spread] |
| own_share | 自己挂单量占该标的 reward zone 总深度的比例 |
| time_in_zone | 挂单在 reward zone 内的累计时长（不撤单就一直累加） |
| 塞单 | 自己挂单被对手单吃掉形成非预期持仓 |
| front_depth | 自己挂单价与 mid 之间的深度（保护墙） |
| back_depth | 自己挂单价外侧的深度（逃生通道） |

## 附录 B：相关资源

- 原始会议录音智能纪要：[飞书链接](https://xpkm2p9eto.feishu.cn/docx/DBESd482EouBTixCZJncBGqMnig)
- Polymarket 官方文档：https://docs.polymarket.com/
- CLOB Client (Python)：https://github.com/Polymarket/py-clob-client
