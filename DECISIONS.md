# 决策日志（DECISIONS）

> 所有沟通中做出的决策按时间顺序记录在此，避免遗忘。
> 格式：日期｜决策项｜结论｜原因/背景

---

## 2026-04-14（Phase 3a）

### D-045 Trading primitives: paper-mode-default, proxy detection, risk guard
**背景**：Phase 3a 首次让 bot 具备下单能力（可花真钱）。安全优先于功能。

**结论**：
- **`trading.paper_mode=true`** 是默认值。`PMTrader.place_order` / `cancel_*`
  在 paper mode 下**完全不调 CLOB**，仅写 DB + emit `PAPER_ORDER` event，
  订单行 `extra.paper_mode=true` 区分。切换到实盘要求：
  1. YAML 手动改 `paper_mode=false`
  2. 首个 `place_order` 必须带 `confirm_real_trade=True`（CLI: `--live` + `--confirm-real-trade`）
  3. stderr 打印 warning，不会静悄悄切换
- **Notional × 1.5 硬顶**：实盘下任何订单 `price * qty > per_market_cap * 1.5`
  直接 `raise OrderSafetyError`，不发网络请求。
- **Proxy wallet 检测（D-026 延伸）**：`PMTrader.detect_proxy_wallet()` 在
  `unlock` CLI 中自动运行，结果写 `events(type=PROXY_WALLET_DETECTED)`。
  Phase 3a 先写骨架：若 `web3.py` 未安装/RPC 未配则退化为 EOA
  (`signature_type=0, funder=<EOA>`) 并 warn，用户可在 config 手动覆盖。
  Phase 3b 将接 Polygon RPC 做真检测。
- **Relayer 限流**：所有 `cancel` / `place_order` 经过一个 25/min 的
  `TokenBucket` (`relayer_req_per_min` 可调)，避免瞬时撤重挂打爆配额。
- **Retry + CB**：Trading 方法复用 D-036 的 CLOB retry profile（10 次 ×
  0.3→30s）并走共享 `PMClient` 的断路器。
- **Risk guard** (`src/pmbot/trading/risk_guard.py`)：
  - `check_budget`：per-market cap + 总 cash-reserve 上限
  - `check_daily_drawdown`：读 `risk_state` 表
  - `check_circuit_open`：honor CB 状态
  - `on_fill`：单市场亏损比 > `risk.single_stuck_loss_ratio` 则
    `HaltLevel.PAUSE_NEW`
  - `HaltLevel ∈ {OK, PAUSE_NEW, HALT_ALL}`，持久化在新表 `risk_state`
    （每日一行，migration `a9c5d7e3b011`）。
- **Fill detection**：`FillsCollector` 轮询 authenticated `/data/orders` +
  `/data/trades`，对比 `my_orders` 状态变化；paper_mode 下空跑（心跳）
  不调 API。
- **Keystore**：`unlock_from_prompt` 用 `getpass.getpass`（只认 stdin）；
  `lock()` 把 pk 字节清零；新增 `sign_message` / `sign_typed_data` /
  `private_key_hex` 封装 eth_account。密码绝不进日志/env/file/TG。
- **CLI 新增**：`pmbot unlock` / `wallet-info` / `test-order [--paper|--live]`
  / `cancel-all`。

**新事件类型**：`ORDER_FILL`、`PARTIAL_FILL`、`CANCEL_CONFIRMED`、
`PROXY_WALLET_DETECTED`、`RISK_HALT`、`RISK_RESUMED`、`PAPER_ORDER`。

**测试**：`tests/test_keystore.py`、`tests/test_pm_trading.py`、
`tests/test_risk_guard.py`、`tests/test_fills_collector.py`、
`tests/test_cli_unlock_flow.py`、`tests/test_rate_limit.py` —
131 → 167 tests 全绿。

**Why now**：这是首次接触写操作的 phase。安全默认 + 可重试 + 可审计 + 可
紧急 kill 是把主控权留在人手里的最小必要条件；策略层接入留 Phase 3b。

---

## 2026-04-14（Phase 2.7）

### D-038 σ resilience: cache + local orderbook fallback + CB-aware fetch
**背景**：Phase 2.6 拿到了 sports tags，但 `pmbot universe-scan --with-screen`
仍然 selected=0。根因是筛选期 σ 单点依赖 `/prices-history`：CLOB 在代理抖动
时 circuit breaker 会 OPEN，于是每个市场都被 `insufficient_history` 拒绝。

**结论**：σ 计算从单源改为 4-源 pipeline（`pmbot/screening/sigma_cache.py`）：

- **A. cache**：`prices_history_snapshots` 行新鲜度 ≤ TTL（默认 12h）
- **B. orderbook_local**：`orderbooks.mid` 近 7 天 ≥ 24 个样本时，σ=stdev/mean
- **C. fresh_fetch**：调 `/prices-history`；CB OPEN 时直接跳过
- **D. default_fallback**：常量 0.025（默认），可 null 关闭

新表 `prices_history_snapshots`（迁移 `f3d8a1c2e701`）存 `(market_id, token_id, fetched_at, history_json, sigma_7d, max_jump_24h, source)`；索引 `(market_id, fetched_at DESC)` 支持按最新取。

**预热**：universe scan 收尾时 `_prewarm_targets` 挑出 long-dated 或 sports 的市场，`prewarm_sigma_cache` 顺序拉 `/prices-history`（并发=1）写入缓存；5 min 硬超时、CB OPEN 时 sleep 2s 且不算一次尝试。

**可见降级**：0.025 > 2% 长期 σ 门槛 → 市场仍然被拒，但拒绝原因从 `insufficient_history`（不可见）变成 `σ=0.025>0.02`（可见）；`screener_runs.extra.sigma_source_counts` 给出每次运行的源分布。

**配置**（`screener.*`）：`sigma_cache_ttl_hours=12.0`、`sigma_default_fallback=0.025`、`sigma_prewarm_enabled=true`、`sigma_prewarm_time_limit_s=300`。

**测试新增**：`tests/test_sigma_cache.py`（cache TTL、A/C/D 源穿透、写缓存副作用）、`tests/test_sigma_orderbook_local.py`（合成 orderbook 行 → σ 数学）、`tests/test_sigma_circuit_breaker.py`（CB OPEN → Source C 短路 → D）。

**Why now**：Phase 2.6 把数据层修通了，但筛选流水线还是 0 命中。本次不改门槛、不改 sports 语义，只把 σ 的可用性从「靠一次网络请求」提升到「缓存+本地+网络+降级」四保险。

---

## 2026-04-14（Phase 2.6）

### D-037 Gamma tag enrichment collector (unblocks Tier B sports gates)
**背景**：D-034 搭好了 `SPORTS_GATES`，但 CLOB `get_sampling_markets`（喂
`MarketsCollector`）不返回 tags 与 `gameStartTime` — 所有标的
`market.tags IS NULL` → `is_sports_market()` 永远 False → Tier B 路径从未
被触发 → 295/301 evaluated fail 于 `T<90d`。

**结论**：新增 `TagsCollector`（`src/pmbot/collectors/tags_collector.py`）
用 Gamma REST 回填 `markets.tags` 和 `markets.game_start_time`。

- **Gamma schema 验证**：`/markets?condition_ids=X` 返回的市场对象中
  `tags` 与 `events[0].tags` 均为 null；tags 实际挂在 event 上。
  →  两步解析：bulk fetch markets → 对每个 `events[0].slug` 调用
  `/events/slug/{slug}` 取 tags（逐 slug 周期内缓存）。
- **标签规范化**：存库前把 `[{"slug":"sports","label":"Sports"},...]`
  扁平成 slug 字符串列表 `["sports","nba"]`，与 `is_sports_market()`
  消费形式一致；也容忍 dict / str / None / 畸形条目。
- **节奏**：hourly（`collectors.tags.interval_sec=3600`）— 标签几乎不变，
  polling 慢即可；每周期上限 `max_markets_per_cycle=500`（让全宇宙
  5669 markets 在约 10–12 个 cycle 后完全覆盖），batch=50 per HTTP call。
- **优先级**：只挑 `tags IS NULL OR tags=[]` 的行；Gamma 返回为空的
  cid 落盘成 `tags=[]`，避免下个 cycle 重拉。
- **one-shot 回填**：新增 `pmbot tags-backfill [--limit N]` CLI，
  同一 collector 循环跑到 targets 为空为止。

**Live backfill**（2026-04-14，5669 markets，full universe）：
- 单 cycle 50 markets ≈ 10.9s → 有效吞吐 ≈ 4.6 markets/s。
- 全量预计 ~20 min；实际运行见 `/tmp/backfill2.log`。
- Schema quirk：少数 cid Gamma 不返回行，落 `tags=[]` 已处理。

**事件新增**：`TAGS_FETCHED`（每 cycle 一条，payload=`{updated, failed}`）。

**测试新增**：`tests/test_tags_collector.py`（9 用例）覆盖：
扁平化各种 tag shape、缺失 Gamma 行、batch 失败、无 target 跳过、
已标记的不重复拉、backfill 循环到空、CLI dispatch。

**Why now**：D-034 体育门槛落地但数据不通，Tier B 实际仍是死路；
先把数据层修好再看 screener 效果。

---

### D-036 Network resilience for shared-proxy Cloudflare throttling
**结论**：环境适配（而非改代理）。主要变更：

1. **Per-endpoint 重试**（`pmbot.pm.retry`）：CLOB 10 次 × 指数退避 0.3→30s；
   Gamma 3 次 × 0.5→5s。仅重试 transient 网络错误
   （`ConnectionResetError`、`httpx.ConnectError/ReadTimeout/RemoteProtocolError`、
   `PolyApiException(status_code=None)`）；`4xx` 永不重试。
2. **并发降低**：`orderbook.max_concurrent_requests` 10 → 3（pool）；
   `universe_max_concurrent_requests` 15 → 5。测得共享代理 IP 下，
   更高并发只会触发更多 Cloudflare RST。
3. **部分容忍的 universe 扫描**：15 分钟硬超时，100 个 fetch 一批
   commit，允许超时后保留已写入数据。完成率 < 70% 时 emit
   `UNIVERSE_SCAN_PARTIAL` 事件但继续 hand-off 给 screener。
4. **自适应并发**（universe scope）：每 100 fetch 为一个 batch，
   fail_ratio > 50% 则折半并发、< 10% 连续 2 batch 则翻倍（顶格到 config max）。
5. **断路器**（`pmbot.pm.circuit_breaker`）：60s 滚动窗口，≥20 样本且
   fail_ratio > 80% → open 30s；冷却后半开，单次探测成功则关闭。
6. **Pool mode 降级**：cycle 内不额外重试；同一 market 连续失败 5
   个 cycle 发 `MARKET_FETCH_STALLED` 事件 + TG 告警。screener/monitor
   使用最近一次成功快照（带 `captured_at`），不要求实时。

**实测**（2026-04-14）：在 5668 个标的的 universe 扫描中，在代理
轻载条件下稳定 11 req/s、约 16 分钟写入 ~5023 个 market 的 orderbook
数据；最后 ~2000 fetch 触碰 15 分钟墙钟硬截止，断路器在 81%
fail_ratio（191 样本）时正确 open 30s，降级路径完整生效。

**新事件类型**：`UNIVERSE_SCAN_PARTIAL`、`MARKET_FETCH_STALLED`、
`CIRCUIT_BREAKER_OPEN`、`CIRCUIT_BREAKER_CLOSE`。

**配置新增**：
- `pm_api.retry.*`（6 knob）
- `pm_api.circuit_breaker.*`（5 knob）
- `collectors.orderbook.partial_complete_ratio`、`universe_timeout_s`、
  `adaptive_batch_size`、`adaptive_halve_threshold`、
  `adaptive_double_threshold`、`pool_stall_cycles`

**测试新增**：`tests/test_retry_config.py`、`tests/test_circuit_breaker.py`、
`tests/test_adaptive_concurrency.py`。

**诊断记录**：Clash 出口 IP `103.175.98.14` 与其它 Polymarket 用户共享，
Cloudflare 对该 IP 做 40–60% 丢包式 throttle。用户明确不更换代理，
故采取环境适配策略。完整背景见 `docs/network_resilience.md`。

---

## 2026-04-14（Phase 2.5）

### D-035 Two-stage orderbook architecture
**结论**：`orderbook_collector` 拆分为三种 `scope`：

| scope | 触发 | 抓取范围 | 预算 |
|---|---|---|---|
| `universe` | 每日 `screener.daily_run_utc`（默认 UTC 00:00 = UTC+8 08:00）+ 启动时 catch-up | 全部 rewards-enabled active market，双向 top-20 档 | ~60s（直连）/ 实测 3–6min（经代理） |
| `pool` | 常驻每 `pool_interval_sec`=30s | 仅当日 `candidate_pool` 中 NEW/RETAINED | ~数秒/cycle |
| `legacy` | 可选启用 | Phase 1 行为：top-N by `reward_daily_usd` | 兼容回退 |

Runner 并行独立任务：`_universe_scanner_loop`（daily + startup catch-up + 调用 screener）+ pool collector + `_screener_loop`（兜底，若 universe scan 已跑则跳过）+ `_monitor_loop`。

**新 CLI**：`pmbot universe-scan [--with-screen]` — 手动触发一次性扫描，生产零停机试跑。

**配置新增**（`collectors.orderbook`）：`scope`、`pool_interval_sec`、`universe_scan_at_startup`、`universe_max_markets`（0=无上限）、`universe_max_concurrent_requests`（默认 15）、`progress_every`（默认 1000，日志节流）、`legacy_enabled`、`legacy_max_markets`。`screener.trigger_after_universe_scan`（默认 true）。`max_markets` 保留给 legacy scope（Phase 1 兼容）。

**为什么现在做**：全 universe ~5600 个标的一个 30s cycle 拉不完（2×5600=11200 fetches，直连 15 并发 ~60s，接近预算但实测经代理达 ~6min），日常只需对"候选池内"的标的做秒级跟踪。两阶段把"广度"（daily scan for screener）和"频度"（pool tracking for monitor）解耦，API 预算和实时性两全。

**Dress rehearsal**（2026-04-14，`docs/dress_rehearsal_2026-04-14.md`）：universe 扫描 5651 标的 ~11300 fetches 跑完、立即触发筛选、pool 采集器 30 分钟常驻循环验证无异常。

**测试新增**：
- `tests/test_orderbook_scope.py` — 三种 scope 选取目标的 DB 查询正确性
- `tests/test_runner_universe_scheduling.py` — `_parse_daily_run_utc` / `_seconds_until_next`
- `tests/test_universe_scan_cli.py` — `run_universe_scan_once` 写入 DB 的端到端

---

## 2026-04-14（继续）

### D-039 universe scan 限定 top-500 by reward
**结论**：`collectors.orderbook.universe_max_markets` 从 `0`(不限) 改为 `500`

**Why**：
- 用户提议（2026-04-14）："标的太多了，可以改成按奖池金额排序，只看前 500 个"
- 当前 5668 个 reward 标里，第 500 名 daily reward 阈值 ≈ $60/天
- 阈值之下的标长尾低价值，自身 share 拿到也只赚几毛/天，不值得占网络/算力配额
- 全量扫描受代理限制约 ~16 min；扫 500 个预计 ~2 min，screener 反应更快
- 排序逻辑 `reward_daily_usd DESC NULLS LAST` 在 collector 已实现，仅改配置即可生效

**实施**：单行配置改 `universe_max_markets: 500`，无需新代码、无需 schema 变更。

**影响**：
- universe 扫描覆盖从 5668 → 500（top-9%）
- 5168 个长尾标的不再被采集 orderbook，但元数据/tags/rewards 仍正常入库（其它 collector 不变）
- 体育标 NHL/NBA 当日比赛奖池都 ≥$1000/day，全部包含在 top-500 内

---

## 2026-04-13（首轮筛选 0 命中后 consolidated update）

### D-034 筛选 + 执行 consolidated update（基于首轮筛选 0 选取的复盘）
**结论**（12 项一次性合并）：

1. **R 门槛改日收益率**：`R_est_daily ≥ 0.3%`（≈ 109% APR）取代 `R_APR ≥ 5%`。评分分档（日）：≥0.5% = 10 / 0.4–0.5% = 8 / 0.3–0.4% = 6 / 0.2–0.3% = 4 / <0.2% = 0。公式：`R_daily = reward_fund_daily × expected_own_ratio / my_capital`（不再 /365）。
2. **`capital.max_markets` 8 → 20**（`screener.max_selected` 同步）。
3. **新增 `capital.per_market_min_usd = $5`**：分配 < $5 的标的直接 skip（不凑数；槽位空着让给下一个高分）。
4. **σ 加权单标上限**：`sigma_capital_multipliers` 三档（σ≤0.5% ×1.6、σ≤1.0% ×1.2、σ≤2.0% ×1.0）。实现：`pmbot.config.CapitalConfig.sigma_multiplier()` + `screener._allocate_capital()`。
5. **σ 评分权重 0.20 → 0.30**；T、D 权重 0.20 → 0.15。新 Score = 0.15·T + 0.15·D + 0.30·σ + 0.20·R + 0.10·价格 + 0.10·点差。
6. **σ 硬门槛保持 ≤ 2%**（长期类）；体育类 ≤ 5%。
7. **`per_market_max_ratio` 5% → 8%**（$500 × 8% = $40 基础上限）。
8. **`capital.total_usd` $300 → $500**。
9. **Bug fix — 双向挂单份数公式**：旧 `capital / 2 / price` 错误；新 `capital / $1`（双向）或 `capital / price`（单向）。PM 机制：pre-mint USDC→yes+no 或 yes-bid+no-bid 成对挂单，两种方式下每对 share 总成本均 ≈ $1。见 `src/pmbot/screening/sizing.py` 与 `tests/test_sizing.py`。
10. **Tier B sports 门槛单独一套**（`SPORTS_GATES`）：`hours_to_kickoff ≥ 5h`、`σ ≤ 5%`、`p ∈ [5, 95]`、`reward_spread ≥ ±2`，其它门槛同长期。类别识别用 `market.tags`，匹配 `sport/nfl/nba/mlb/nhl/epl/...` 前缀。统一候选池（D-032），不分 Tier 配额。体育 MVP 仍走双向默认。
11. **删除 `reward_fund_daily ≥ 5` 门槛**：被日 R 门槛覆盖，冗余。
12. **`collectors.orderbook.max_markets` 50 → 200**：首轮筛选 evaluated 50 个全部 fail，瓶颈是宇宙太窄；扩到 200 增加 zone 聚合样本。

**Why now**：用户首次跑 `pmbot screen` 返回 0 选取；Score 权重偏轻 σ，R 阈值按年率定反而过松/过紧并存，且 max_markets 50 卡死命中面。12 项合并一次性收敛以避免多次小迭代。

**附：sports 类别的 `tags` 字段**：当前 `MarketsCollector` 使用 `get_sampling_markets` 不含 tags；Phase 3 需增补 Gamma API 查询或别的 tag 来源。在 tag 未填充前，sports 标的会 fallback 到 LONG_DATED_GATES（T≥90d 会把大多数体育过滤），这是已知短板，记录在这里等后续迭代。Market 模型已新增 `tags`（JSON）与 `game_start_time`（DateTime）两列，迁移 `d0a42c034001`。

**测试新增**：
- `tests/test_allocation.py`：σ-multiplier 分配、per_market_min skip、max_markets cap
- `tests/test_sizing.py`：双向/单向 qty 公式、回归旧公式
- `tests/test_gates.py`：sports 类别、`gate_hours_to_kickoff`、`gate_sigma_sports`、`gate_price_sports`、`gate_reward_spread_sports`、`gate_r_daily`、删除 `reward_fund` 校验

---

## 2026-04-14

### D-001 AI 开发工具
**结论**：Claude Code（Anthropic CLI，本地运行）
**原因**：录音讨论后确认。本地运行，不涉及上传私钥。

### D-002 起始资金
**结论**：200–500U，单号起步
**原因**：风险可控的最小验证规模

### D-003 MVP 策略
**结论**：只做 Tier A 低风险（超远期市场），选标的也走低风险
**原因**：风险最低、规则最清晰、最容易验证

### D-004 策略风格档位
**结论**：保守档
- target_own_share = 5%
- 挂单外沿缩 2 tick
- 同时入池 ≤ 8 个标的
**原因**：用户明确偏保守

### D-005 塞单处理哲学
**结论**：预防优先
- 第一道防线：筛选标的 + 挂单位置做好，尽量不被塞
- 第二道防线：真被塞了才走自动卖出逻辑
**原因**：用户明确"前提是选标的和挂单策略做好"

### D-006 开发模式
**结论**：Claude Code 编写 + 用户指导
**原因**：用户定位为指导者，非编码者

### D-007 交付窗口
**结论**：策略完整后尽快，无硬性 deadline
**原因**：不赶进度，先把事情做对

### D-008 运行环境
**结论**：Mac mini 本地常驻
**原因**：用户已有硬件

### D-009 PM 接入方式
**结论**：先调研，Phase 0 内完成对比 + 推荐方案
**原因**：CLOB API vs 直连合约需技术评估

### D-010 挂单前方保护
**结论**：新增两条硬门槛
- 挂单价前方同档位排队资金 ≥ 500 USD
- 挂单价前方同档位排队份数 ≥ 平台最小挂单份数 × 3
**原因**：用户指出队首容易被塞单，需把"前方排队量"作为独立维度

### D-011 实时元数据同步
**结论**：架构中必须独立模块实时拉取每个标的的：
- reward_spread（每标不同）
- reward 规则（每标不同）
- 当前 mid / bid / ask
- 平台最小挂单份数
**原因**：用户明确每个标的 reward 规则不一样，不能缓存过久

### D-012 回测节奏
**结论**：AI 搭框架 → 用历史数据跑默认参数 → 基线结果同步给用户 → 共同 review → 迭代阈值
**原因**：用户要求协作判断指标是否合适

### D-013 语言与技术栈
**结论**：Python 3.11+ + asyncio
**原因**：PM 有成熟 py-clob-client SDK；AI 写 Python 效率最高；LP 决策是分钟级不需要极致性能

### D-014 数据库选型
**结论**：SQLite + WAL 模式
**原因**：
- 单进程架构，无跨进程写冲突
- WAL 模式下写吞吐 >1000 TPS 远超需求
- 读写并发，崩溃恢复可靠
- 无需额外服务
- 若未来遇到瓶颈可通过 SQLAlchemy 无痛迁移到 PostgreSQL
**用户原始疑虑**：多标的实时写入 SQLite 只能一个写会不会有问题？
**回答**：不会。本项目单进程写入，所有写请求在 bot 进程内串行化，不存在并发冲突。

**2026-04-14 二次确认（量化分析）**：
- 实际写入负载：保守档 8 标的下约 50 写/分钟 ≈ 1 写/秒；最坏扩到 30 标的 ≈ 30 写/秒
- SQLite WAL 在 Mac mini 上单进程实测 > 5000 TPS（简单 INSERT 可达 20000+）
- 使用率不到能力上限的 0.5%
- "只有一个写" 指同一时刻只能一个事务在写，事务本身是毫秒级
- 单进程 Python 的所有写请求天然通过同一 DB 连接串行，不存在并发打架
- 代码将使用 SQLAlchemy ORM 封装，未来若需切 PostgreSQL 只改连接串
- 用户确认："先这样"

**不适用 SQLite 的场景（我们均不触发）**：跨进程并发写、TPS > 5000、多机网络读写、> 1TB 数据量

### D-015 数据完整性原则
**结论**：所有执行指标与状态全部落盘数据库，决策不依赖内存
**原因**：用户明确"不能依赖记忆判断"
**落实**：见 README §6.3 的 9 张表规划

### D-016 密码管理
**结论**：
- 每次程序启动手动输入一次，驻留内存
- **绝不通过任何通讯通道传输**（包括 Telegram）
- 用户登录 Mac mini 后在本地终端输入
**原因**：用户明确"安全优先，不通过通讯通道传输"

### D-017 启动方式
**结论**：半自动
- launchd 自启 → 进程状态"已启动未解锁"，只做只读监控
- 用户登录 Mac 后手动运行解锁命令输密码 → 开始交易
**原因**：兼顾意外重启后的连续性 + 密码安全

### D-018 日志
**结论**：
- 本地 rotating file，按天切割，保留 30 天
- 路径 `./logs/pmbot.YYYY-MM-DD.log`
**原因**：用户要求实时输出方便查看历史

### D-019 异常告警
**结论**：
- 专用新 Telegram bot（不复用当前聊天 bot）
- 只推异常事件（熔断、塞单处置失败、余额异常、API 连续失败）
- 开发后期接入
**原因**：用户明确"另起一个 bot，后续开发差不多再接入"

---

## 待定项（标 ⏳ 表示回测后再定）

| 编号 | 问题 | 状态 |
|---|---|---|
| Q-A3 | mid 抖动容忍 tick 数 | ⏳ 默认 ±1 |
| Q-A4 | 塞单后冷却分钟数 | ⏳ 默认 30 |
| Q-A5 | 日回撤熔断阈值 | ⏳ 默认 −3% 暂停 / −5% 熔断 |
| Q-B1a | ~~PM 是否有公开历史 API~~ | ✅ 已定（见 D-020） |
| Q-B1b | ~~若无历史，自采 7–14 天后再回测~~ | ✅ 已定（见 D-020） |
| Q-T1 | ~~PM 是否支持 `replace`~~ | ✅ 已定（见 D-021） |
| Q-T2 | ~~reward 变更通知机制~~ | ✅ 已定（见 D-022） |

---

## 2026-04-14（续）— Phase 0 调研结论

> 详见 `docs/pm_api_research.md`

### D-020 历史数据策略
**结论**：
- mid 价历史：用 PM `GET /prices-history`（1 分钟粒度）
- 深度 / 盘口历史：**PM 无公开 API → 必须自采**
- 实施：Phase 1 数据采集启动后同时落盘盘口快照，积累 ≥7 天再做深度相关回测；先期可用纯 mid 做粗糙回测
**原因**：用户要求"先用历史数据回测"——mid 层回测可即刻开始，深度回测要等自采数据

### D-021 PM 无原子 replace
**结论**：PM 所有重挂都必然是 `cancel → post new`，**队列优先级丢失不可避免**
**影响**：
- README §5.7 "使用 replace 保留队列优先级" 假设不成立，已改写
- 策略权重进一步向 `time_in_zone`（累计在 zone 时长）倾斜
- 好消息：PM reward 按一周 10,080 次 1 分钟采样计分，是**时长累计型**而非"谁先挂谁优先"
**延伸决策**：每分钟撤重挂次数硬上限 10 次（避免触发 relayer 25 req/min 限流）

### D-022 Reward 参数获取
**结论**：
- 无 push / webhook
- 必须轮询 `GET /rewards/markets/current`
- 轮询周期：**60–120 秒**（D-023 定）
**每标返回字段**：
- `rewards_max_spread`（tick 数，即我们的 reward_spread）
- `rewards_min_size`（挂单最小份数）
- `total_daily_rate`（该标每日总 reward 池）

### D-023 Reward 轮询周期
**结论**：默认 90 秒
**原因**：
- 太频繁（30s）占用 API 配额
- 太稀疏（300s）可能错过参数调整
- 90s 为中间值，实盘跑几天后再调

### D-024 SDK 技术栈
**结论**：
- 主：`py-clob-client` v0.34.6（Feb 2026 发布，覆盖约 95% 需求）
- 补充：`web3.py`（token allowance 授权，一次性）+ `httpx`（Gamma/Data API 访问）
- **不需要**直接调 CTF Exchange 合约（MVP 阶段）

### D-025 新增风险：Adjusted Midpoint
**结论**：
- PM 用 "adjusted midpoint"（过滤 dust 订单后的中间价）来计 reward
- 这可能与我们观测的 mid 不同
- **需要实盘实验测量 dust 阈值**——加入 Phase 2 验证清单

### D-026 Proxy Wallet 签名类型
**结论**：
- PM 钱包有 `signature_type` 字段（0/1/2），错配下单会被拒（错误信息模糊）
- 交易前必须链上检测钱包是否有 Safe proxy → 动态设置 `signature_type` 和 `funder`
- 第一次启动自动检测并持久化到 DB（`config_history` 表）

---

### D-028 筛选节奏：每日筛选 + 盘中淘汰
**结论**：
- **每日筛选**每天 **UTC+8 08:00（= UTC 00:00）** 跑一次，产出当日候选池
  - 用户基于经验：PM 官方每日 reward 更新发生在此时点之前
  - 与北京时间工作节奏对齐
  - 待 Phase 1 数据收集足量后，可观察 PM reward 实际更新时点验证此假设
- 当日**不引入新标的**（避免日内频繁换标导致的撤重挂浪费）
- 盘中每 5 分钟对已选池做"健康检查"，触发退出条件即撤单移出
- 释放的资金当日仅留作现金，**不当日补仓**（次日筛选时再分配）

**退出条件**（任一触发即淘汰）：
- 24h 内出现 >5% 价格跳动
- σ 实时滑动窗口（4h）超过 3%
- 深度 D 跌出 [800, 120k] 区间（10% 缓冲）
- reward_spread 变化导致挂单位置不在 zone 内
- total_LP_capital_in_zone 涨过 80,000 USD（拥挤度恶化 60%）
- expected_own_ratio 跌破 1%
- 距到期 < 30 天
- 标的状态变 resolved/closed

**Why**：
- PM 标的池 ~5594 个，每 10 分钟全量重筛太重（D-027 调研发现）
- 用户明确"标的不是一直能做的，不合适就放弃"——退出比补仓优先
- 避免日内换标导致 cancel+post 浪费 relayer 配额（25 req/min）和 time_in_zone 累积

**How to apply**：
- Phase 2 实现：`screener` 模块按日 cron + `monitor` 模块每 5 分钟跑健康检查
- 日选完后通过 TG 频道推送结构化报告（见 README §4.6）

### D-029 TG 频道用途与归属
**结论**：
- 每日筛选报告 + 异常告警 → 推送到**专用 Telegram 频道/bot**
- 复用 D-019 计划的 "异常告警 bot"（不复用当前对话 bot）
- 推送内容分两类：
  - **每日报告**（定时）：每日筛选结果（入池/出池/未入池/账户）
  - **异常告警**（事件）：熔断、塞单处置失败、API 长时间挂、余额异常
- 频道形式 vs bot 私聊：先用 bot 私聊（简单），后期需要多人订阅再升级为 channel

**Why**：每日报告和告警都希望"即看即懂、有时间戳、可回溯"，TG 是最低成本方案

---

### D-032 Tier 体系：统一资金池 + 类别感知门槛
**结论**（2026-04-14 用户决策）：
- **不**按 Tier 切分资金比例（不再 70/30）
- 所有标的进**统一候选池**，按综合 Score（含收益率 + 风险）排序选 Top 8
- 不同类别走不同**硬门槛**集（sports 用 sports gates；其它用 long-dated gates）
- 资金分配：$300 × 90% / 实际入池数（与 Tier 无关）

**Why**：用户原话"只看收益率和风险率"——避免人为切池子带来次优分配；体育和长期标的本就竞争同一 USDC 余额，无需预留比例

**实施改动**：
- 删除"Tier B 资金占比 30%"
- 删除"Tier B 入池上限 5 个独立配额"——统一为整体 8
- 类别识别用 PM 标签：market.tags 含 sport/sports/赛事名 → sports gates；否则 long-dated gates
- 评分公式不变（§4.3），分高即入池
- 风控按总资金算（不分 Tier）

### D-033 体育标的赛前停挂时间
**结论**：**比赛开始前 120 分钟**（2 小时）停止挂单 + 撤现有挂单
**Why**：用户决策（比初稿 30 分钟更保守）；赛前 2h 起赔率波动剧增（开盘前消息、伤病更新）；为应对突发新闻预留撤单缓冲

---

### D-027 筛选维度扩展：奖池 + zone 内总资金
**结论**：在维度 4（收益率 R）下显式拆出两个独立可观测量：
- `reward_fund_daily` = PM `total_daily_rate`（该标的每日总奖励池 USD）
- `total_LP_capital_in_zone` = **仅 reward zone 内** 所有 LP 资金 = Σ(qty × price) for all orders within `[mid − rewards_max_spread, mid + rewards_max_spread]`，含 bid + ask 两侧

**新增子门槛**（与 R ≥ 5% 一同必须满足）：
- `reward_fund_daily ≥ 5 USD/day`
- `total_LP_capital_in_zone ≤ 50,000 USD`
- `expected_own_ratio = my_capital / (my_capital + total_LP_capital_in_zone) ≥ 2%`

**新派生指标**：
- 奖励密度 = `reward_fund_daily / total_LP_capital_in_zone` （越高=市场越"未被吃透"）
- R_est_APR = `reward_fund_daily × expected_own_ratio × 365 / my_capital`

**与维度 2（深度 D）的关键区别**（用户提问澄清）：
- D = 盘口前 5 档美元数 → 关注**抗冲击/反塞单**
- total_LP_capital_in_zone = zone 内全部参与方资金 → 关注**奖励稀释程度/反拥挤**
- 二者可独立变化（D 高但 zone 内人少；或 D 低但 zone 内分散许多人）→ 必须分开看

**对 Phase 1 的影响**：orderbook collector 入库时需要标记每个 level 是否在 zone 内（或在分析层算）。我会派 followup agent 在 Phase 1 收完工后增加该派生字段。

---

## 2026-04-13（Phase 2）

### D-031 Phase 2 筛选器/监控器若干实现决策

**结论汇总**（若干非平凡实现选择，非策略层决策）：

1. **启动 catchup 阈值 22h**：daemon 启动时若最近一次 `screener_runs.run_at`
   距今 > `screener.catchup_threshold_hours`（默认 22h），立即跑一次筛选并推
   TG，然后再按每日 UTC 00:00 节奏循环。避免"凌晨 00:00 重启 → 今天拿不到报
   告"的空窗。阈值定为 22h（而非 24h）是为了让昨天勉强跑过一次的情况不会被
   误当作补跑机会。

2. **σ 不足历史处理**：`get_price_history` 返回 < 30 条 1-min 样本时，
   `compute_sigma_from_points` 返回 `(None, None)`；`gate_sigma` 以
   `"insufficient_history"` 拒绝该标的；`gate_no_recent_jump` 同步拒绝
   `"missing_24h_history"`。保守语义：没有数据就不承认，宁错杀不错放。

3. **极端价格带 σ 用 peak-to-trough 替代 stdev**：当 current_mid 落在
   `(0.10, 0.90)` 之外时，7 天 stdev 在边缘被严重压缩（数学上 p 接近 1
   或 0 的伯努利过程方差本就趋近 0），直接用 stdev/mid 会低估风险。此时
   改用窗口内 max-min 幅度 / mid（即"最大跳动幅度"）作为 σ。对应 README
   §4.2 维度 3 中的"价格 >90 或 <10 改用最近 7 天最大跳动幅度"。

4. **monitor 的 4h σ 暂未实现（退而求其次）**：§4.6 退出条件 2 要求"σ 实时
   滑动窗口（最近 4h）超过 3%"。我们目前 *没有* 本地的 prices_history 表
   —— σ 是筛选时临时从 PM `/prices-history` 拉一次算出来的。每 5 分钟对
   每个池内标的再拉一次 4h 窗口会烧掉 relayer 配额。暂行方案：
   - 监控器 *不* 每周期重算 σ
   - 其它 7 个退出条件（跳动、深度、拥挤、own_ratio、到期、状态、reward_spread 收窄）
     全部生效，覆盖绝大部分风险情景
   - TODO（Phase 3）：自建 prices_history 表 + websocket 订阅，或降低到 1h 间隔

5. **depth 三段法"取两侧中较小值"**：筛选器和监控器在读取 front_depth /
   back_depth 时，用 `min(yes_side, no_side)` 而非相加，因为我们要求 *两侧都*
   能满足深度门槛（我们计划双向挂单，哪一侧先崩就先被塞）。这与 §5.5 "保护墙 + 逃生通道"
   的双向语义一致。

6. **容量预估用 per_market_max_ratio 做保守上限**：在 own_ratio 计算里，
   `my_capital` 取 `total_usd × per_market_max_ratio`（即单标最大可投），
   而非平摊额度。好处：门槛只会比实盘实际 own_ratio *更容易通过*，不会发
   生"门槛放行了但实盘资金少到进不去"的情况。

7. **TG MarkdownV2 用 fenced code block 包裹数据**：所有 slug/reason/number
   在信息块里走 ` ``` ` 围栏，块内无需逐字转义；只对头部 emoji 旁边的零碎
   标量（run date、账户行、footer）用 `escape_markdown_v2` 转。
   好处：保持密度 + 避免漏转。

8. **CLI：`pmbot collect` 默认启动 screener+monitor**，新增 `--collect-only`
   回退到 Phase 1 行为。收集 + 筛选 + 监控共用同一个 PMClient 和 session
   factory，无需多进程。

9. **TG 客户端默认启用 `trust_env=True`**：httpx 会读取 `HTTPS_PROXY` /
   `https_proxy` 环境变量。在 macOS 本地代理场景（Clash 等）下即插即用。

**Why now**：这些都是 Phase 2 实现中现实约束迫使的折中，文档化后
Phase 3（真实下单）做取舍时有据可依。

---

## 2026-04-13（Phase 1.5）

### D-030 Orderbook 存储布局：每 side 一行
**结论**：Phase 1.5 起 `orderbooks` 表采用**"每市场每周期两行"**布局，通过 `outcome` 列（`'yes'`/`'no'`）区分 YES/NO 两侧；每行还带 `token_id` 便于溯源。

**考虑过的两种方案**：
1. ❌ 单行双侧（`bids_yes`/`asks_yes`/`bids_no`/`asks_no` + 每侧聚合列各 ×2）
2. ✅ 每侧一行（`outcome` 列 + 同套聚合列）

**为什么选方案 2**：
- zone 聚合（`mid_price`、`zone_lower/upper`、`total_lp_capital_in_zone`、`front_/back_depth`）**天然每侧独立**——YES 与 NO 的 mid、reward zone 完全不同（PM 两个 token 独立建仓）
- 方案 1 会强迫每个聚合列复制为 `_yes` / `_no` 两份，schema 冗长且未来加字段要改两次
- 方案 2 保持"一个快照 = 一行"语义，Phase 3 回放时每行是完整可读的独立事实
- 筛选器需要的市场级视图可通过 `(market_id, captured_at)` join 两行即时拼出；`pmbot.db.session.latest_orderbooks_for_market()` 已封装此查询
- 市场级 `total_LP_capital_in_zone` = yes_row + no_row（每侧各自 zone 内资金相加，无双重计数）

**伴随变更**：
- 新增列：`outcome`、`token_id`、`mid_price`、`rewards_max_spread`、`tick_size`、`zone_lower`、`zone_upper`、`total_lp_capital_in_zone`、`front_depth_usd`、`back_depth_usd`、`depth_usd_top20`
- 原 `depth_usd_5` 保留向后兼容
- 采集层同时抓 YES + NO token 的 top-20 档（从 5 提升），一周期 2×50=100 次 HTTP；并发上限 5→10（实测一轮 ~5s，30s 预算内绰绰有余）
- 筛选器侧 helper：`latest_orderbook(session, market_id, outcome)`、`latest_orderbooks_for_market(session, market_id)`、`expected_own_ratio_for_market(session, market_id, my_capital_usd)`
- 迁移 `e811e703a041` 为旧 YES-only 行以 `server_default='yes'` 回填 outcome，upgrade/downgrade 均测通

**Why now**：D-027 定义了 `total_LP_capital_in_zone` 维度，需要 YES + NO 双侧 + zone 内范围筛选；Phase 1 只抓 YES top-5 不够用。D-028 每日筛选节奏要求数据层预先算好这些标量以避免把聚合带进 hot path。

---

## Phase 0 延续的待定项（Phase 1/2 验证）

| 编号 | 问题 | 状态 |
|---|---|---|
| Q-N1 | Adjusted midpoint 的 dust 阈值 | 需实盘实验 |
| Q-N2 | py-clob-client 是否有 `get_rewards_*` 包装 | Phase 1 确认 |
| Q-N3 | 单钱包最大未完成订单数 | Phase 1 确认 |
| Q-N4 | 批量撤单算 1 次还是 N 次 relayer 调用 | Phase 1 确认 |
| Q-N5 | neg-risk 市场的特殊语义 | Phase 2 调研 |
| Q-N6 | 标的到期后已挂单的处理行为 | Phase 1 确认 |
| Q-N7 | fee-tier 按钱包变化的机制 | Phase 1 确认 |

---

## 更新规则

每次有新决策（无论是 Telegram 上还是线下讨论）→ 追加到这个文件顶部（按日期倒序）。
待定项决定后移到主列表并标日期。

---

### D-041 重定位"中流动性主流"（用户决策 2026-04-14）
**结论**：放宽多个上限 + 收紧 own_ratio 到千三

| 项 | 旧 | 新 |
|---|---|---|
| `MAX_DEPTH_USD` | 100,000 | **None**（删除上限，深度高=反塞单好事） |
| `MAX_TOTAL_LP_IN_ZONE` | 50,000 | 1,000,000（升级为 sanity cap） |
| `MIN_OWN_RATIO` | 0.02 (2%) | **0.003 (千三)** ← 用户原话 |
| `MAX_SIGMA_LONG` | 0.02 (2%) | 0.05 (5%) |
| σ 资金乘数表 | 3 档 (≤0.5/1.0/2.0%) | 5 档（新增 ≤3.5%×0.7 / ≤5.0%×0.5） |

**Why**：
- 用户质疑 100k depth 上限：高深度=好事（反塞单）；"市场拥挤度"应该用 own_ratio 单独管，不该用 depth
- 用户指定目标 own_share = 千三：意味着我们接受在大市场里做小份额 LP
- 数学：own_ratio≥0.3% 隐含 zone_cap ≤ my_cap × 332；$40 my_cap → zone_cap ≤ $13k
- σ 真实数据出来后，政治长期标普遍 5-50% 波动，原 2% 完全选不出
- D-040 标签修复后，sports 类 NHL/MLB 真识别，但 depth 100k 上限把 $400k+ 的主流游戏全部挡掉

**R 公式**：自然包含 own_ratio
```
r_daily = reward_daily / (my_capital + zone_cap_in_zone)
```
（即 `reward × own_ratio / my_capital`，等价）

**Tests**：131/131 通过，新增 depth 无上限/total_lp 1M/own_ratio 千三 三个测试用例

### D-042 起始资金降至 $100（用户决策 2026-04-14）
**结论**：`capital.total_usd` 500 → **100**
**Why**：用户原话"前期计划金额100u"——更保守的小额验证起步
**数学影响**：
- 单标基础上限 = 100 × 8% = $8（之前 $40）
- 平均分配 = 100 × 90% / 20 = $4.5（< per_market_min_usd=$5 → 触发跳过）
- own_ratio ≥ 0.3% (千三) 隐含 zone_cap ≤ $8 / 0.003 - $8 ≈ **$2.66k**
- 实际入池数严重受限（只有 zone ≤ $2.6k 的小盘标能过 own_ratio）
**实测**：3 个 LoL 电竞总场数标入池（zone $2.2k-2.3k），各 $6 cap

### D-043 Tier C "news_event"（用户决策 2026-04-14）
**结论**：新增第三类 news_event，容纳短期非体育事件标（政治/科技/经济/地缘）

| 项 | Tier A 长期 | Tier B 体育 | **Tier C 新闻** |
|---|---|---|---|
| 识别 | 兜底默认 | sports tag | politics/ai/trump/iran/nato/gpt 等 tag |
| T 门槛 | ≥ 90d | 距开赛 ≥ 5h | **≥ 2d**（不做当日） |
| σ 上限 | 5% (D-041) | 5% | **10%** |
| 价格 | [1, 99] | [5, 95] | [1, 99] |
| 点差 | ≥ ±3 | ≥ ±2 | ≥ ±3 |
| 类别优先级 | 兜底 | 最高（先 sports → news → long） | 中 |

**新增 affordability 硬门槛**（适用所有类别）：
`rewards_min_size × min(mid_yes, mid_no) ≤ my_capital_usd`
防止"进了池但挂不上"的隐式不可行标。

**Why**：$100 资金下大量高奖池标是短期新闻（Trump/Iran/NATO/GPT 等），Tier A T ≥ 90d 和 Tier B sports tag 都容纳不了，需要新类别。

### D-044 Tier C 放松 own_ratio + front_queue（用户决策 2026-04-14）
**结论**：Tier C 新闻专用放松：
- `own_ratio` 0.003 → **0.0005**（千0.5，接受更小份额）
- `front_queue` $500 → **$100**（小流动性市场前方资金薄是常态）
- 其它 tier 保持不变

**Why**：$100 cap × 千三 = zone 必须 ≤ $2.66k，但新闻标的 zone 普遍 $5k-$50k，必然被卡；放松后 $100 能真正选出短期新闻标
**实测**：5669 评估 → 29 通过 → **18 入池**（全为 news_event 类别，涵盖政治/科技/经济/地缘/流行文化）

### D-046 Tier C 单边 LP + 真实 affordability（用户决策 2026-04-14）
**结论**：
1. Tier C 挂单改**单边** LP（`news_allow_one_sided=true`）——Tier A/B 仍双向
2. Affordability 公式修正：双向真实成本 = `qty × $1`（不是 `qty × min(yes,no)`）；单边成本 = `qty × cheap_side_price`
3. 单边 LP 选**便宜一侧的 token**（mid 较低那边）挂 bid

**Why**：
- $100 cap 下双向真实成本（铸造 yes 库存）超过 cap 一大截
- 用户明确"金额还是100"，必须在 $100 里跑
- 单边 LP = 只挂便宜侧 token 的 bid，风险只是无法同时赚两侧 reward（PM 双向 reward 机制下），但不是方向性下注

**实测**（paper mode 模拟 18 个候选，cheap-side 单边 LP）：
- **7 个能真实挂上**（cost $4.80-$5.00 各/cap $5）：jason-perry / sheila / anthropic / bw-industrial / gpt-5.5×2 / bryan-johnson
- 10 个 SKIP（min_size × cheap_price > $5）
- 1 个 FAIL（tick 对齐问题，需要 mid 小数精度处理）
- 实际部署资金 ~$35/$100，剩 $65 闲置

**后续**（Phase 3b 要做）：
- Trader 层实现 cheap-side 选择逻辑
- tick 对齐边缘情况（`price 0.016 not aligned to tick_size 0.01`）——round 到 tick
- Affordability 门槛增加"能否挂到 min_size"硬检查（替代当前 single-side 估算）

**测试**：167/167 通过（fixture `my_capital_usd` 15→25 适配新公式）

### D-047 Phase 3b: trader wired into runner + fixes (2026-04-13)
**结论**：strategy loop 消费 candidate_pool，dispatch place/cancel；修复 tick、proxy、auth、affordability 四处。

**新增/改动**：
1. **`trading/planner.py`**（新）— pure builder：`(Market + orderbook + cfg) → list[OrderPlan]`
   - Tier A/B 双边 yes BUY+SELL（成本 qty×$1）
   - Tier C news 单边 cheap-side BID（成本 qty×cheap_price；`mid(yes,no)` 取小者）
   - qty = floor(alloc / cost_per_share), 下限 rewards_min_size，整数股
2. **`trading/ticks.py`**（新）— `round_to_tick / validate_price`，修复 `0.016 → 0.02` bug；planner 每次 emit 前 snap，trader `_validate` 用同一校验
3. **`trading/proxy_wallet.py`**（新）— 真实 Polygon RPC 检测：
   - CREATE2 推导 Safe 1.3.0 proxy 地址（优先用 `py_clob_client.utilities`，fallback 本地 keccak）
   - `eth_getCode` → 有代码 signature_type=2 funder=proxy；无代码 type=0 funder=eoa
   - graceful degradation：web3 缺失/RPC 报错 → 默认 EOA + warn
4. **`trader.py`** 改动：
   - `_probe_proxy` 调用 proxy_wallet，并将结果回写 `cfg.trading.{signature_type,funder}`（内存）
   - `_validate` 复用 `validate_price`
   - 新增 `ensure_clob_auth()`：实例化 ClobClient（`host+private_key_hex+chain_id+signature_type+funder`）+ `set_api_creds(create_or_derive_api_creds())`；paper 模式 no-op
5. **`runner.py`** 改动：
   - 新增 `_strategy_loop` + `_dispatch_trading`：从 `asyncio.Queue[ScreenerResult]` 消费，逐 market 调 planner→trader.place_order；removed_from_yesterday→trader.cancel_all_for_market
   - `_universe_scanner_loop / _screener_loop` 完成一次 screener 后 put 到 queue
   - `_main` 构造 PMTrader（keystore=None，纯 paper）+ 启动 strategy task
6. **`gates.py` affordability 增强**：原来只查 `min_size × price ≤ cap`；现增 `max_qty (= cap/cost_per_share) ≥ min_size` 硬检查，防止"进池但挂不上"（paper 模拟显示的 10 skip）
7. **`pyproject.toml`**：新增 `web3>=6.0`
8. **`config.py`**：`PMApiConfig.polygon_rpc_url = "https://polygon-rpc.com"`

**Why 分 Queue 解耦 screener↔trader**：trader 批量 place/cancel 可能 10 秒级（relayer 25 req/min），不阻塞 universe scanner 下一轮 cadence。queue maxsize=8 防止雪崩。

**Why 单边 cheap-side**：D-046 明确 $100 资金下双向铸造成本超 cap；单边 bid 只付 qty×cheap_price，小额可行。

**测试**：167 → **203/203**（+36：ticks/planner/proxy_wallet/strategy_loop）
**Paper 验证**：strategy_loop + _dispatch_trading 在 `test_strategy_loop.py` 三个 case 下真实写入 my_orders（paper），NEW 双边 2 单、news 单边 1 单、REMOVED cancel 全部改 status=CANCELED。

**下一步（Phase 3c）**：
- 真实 keystore 解锁 CLI 命令
- 首单 `--confirm-real-trade` flow
- `get_position / get_balance_usdc` 用 web3 读 ERC-1155 / ERC-20
- Monitor → trader 的 reprice/cancel 触发（不仅仅是 pool REMOVED）
