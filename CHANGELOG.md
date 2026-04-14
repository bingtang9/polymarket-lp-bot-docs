# Changelog

All notable decisions and feature shifts are logged in `DECISIONS.md`; this
file is the high-level timeline.

## [1.0.0] — 2026-04-14

First release: the bot can be started in paper mode today and flipped to
live mode once the user approves USDC.e allowance on the PM proxy wallet.

### Added

- **Phase 0** — Python 3.11 + SQLAlchemy + Alembic scaffold, v3 keystore stub, 9 core tables.
- **Phase 1 + 1.5** — markets / rewards / orderbook collectors; zone-aware aggregates (yes + no sides, top-20 levels, `total_lp_capital_in_zone`, front/back depth).
- **Phase 2 + 2.5 + 2.6 + 2.7** — daily screener with 11 hard gates + 6-dim scoring; universe-scan / pool / legacy collector scopes; Tags enrichment via Gamma; σ multi-source pipeline (cache → local orderbook → fresh fetch → fallback).
- **Phase 3a** — Keystore unlock + PMTrader (place / cancel / batch) with paper mode default, tenacity retry, circuit breaker, 25 req/min token bucket; FillsCollector; RiskGuard.
- **Phase 3b** — Runner strategy loop wiring; `OrderPlan` planner with cheap-side selection; tick-alignment helpers; real Polygon-RPC proxy-wallet detection; ClobClient L2 auth scaffolding.
- **Phase 3c** — `pmbot run --live` unlock pipeline, `pmbot preflight` 12-point checklist, monitor-evictions → trader.cancel wiring, real USDC.e + POL balance reads via web3, `--confirm-real-trade` first-order gate.
- **D-043 + D-044** — Tier C news_event category + loosened own_ratio/front_queue for $100 starting capital.
- **D-046** — Tier C single-sided LP; affordability gate aware of double-sided `qty × $1` cost.
- **D-047** — Runner ↔ trader queue wiring; cheap-side selection; tick-alignment; real RPC proxy detection; affordability max_qty check.
- **D-054** — Polymarket Proxy detection via `PolymarketProxyFactoryV1.5` (0xaacFeEa0…).computeProxyAddress; user's proxy 0x06B579… persisted.
- Comprehensive tests: 218 unit tests, 13/14 QA scenarios pass, formal code review with 2 CRIT / 2 HIGH / 5 MED / 7 LOW findings documented + all CRITs fixed.

### Fixed

- **D-048** — Trader validates price against `market.tick_size`, not the global `min_tick_size`. Fixes rejection of 0.001-tick markets (gpt-5.5 series, italy-warship).
- **D-055** — Risk guard `authorize()` now runs in both paper and live modes; paper mode will respect `HALT_ALL` / `PAUSE_NEW` drawdown halts.
- **Review C-01** — `key=self._ks.private_key_hex` missing call parens in ClobClient auth wiring.
- **Review C-02** — `--confirm-real-trade` CLI flag now propagates into the runner-driven strategy loop.

### Changed

- **D-030** — Orderbook storage: one row per (market, outcome, cycle).
- **D-034** — R threshold switched from annual APR to daily (≥0.3%); σ scoring weight bumped 0.20 → 0.30; σ-tiered capital multipliers; per-market min USD allocation.
- **D-041** — Depth upper cap removed; σ long-dated 2% → 5%; own_ratio threshold 千三 (0.003); total-LP upper cap raised; news-tier multipliers added.
- **D-042** — Starting capital $500 → $100.
- **D-050** — Intraday eviction TG alerts disabled (daily report still summarises).
- **D-052** — Polygon RPC default switched to publicnode (default polygon-rpc.com SSL-EOFs through shared proxy).
- **D-053** — min USDC.e preflight threshold 5.0 → 4.0 to match user's actual $4.9 balance.
- **D-054** — signature_type set to 2 and funder to 0x06B579… (user's PM Proxy).

### Known limitations / pre-live blockers (user action required)

1. **USDC.e → CTF Exchange allowance = $0.** Must be approved via polymarket.com UI before live orders can fill.
2. **Polygon-RPC proxy detection fallback** — upstream `py_clob_client.utilities.derive_proxy_address` is used when available; the inline fallback uses a placeholder init-code hash (Review H-01). Safe users with the upstream path get correct detection; fallback misclassifies to EOA (conservative — just won't trade, doesn't leak money).
3. **Single-sided LP reward model unconfirmed.** PM typically rewards double-sided LP. Tier C cheap-side BID may earn 0 rewards until placed and measured. Fix post-launch based on 1 live test order.
4. **Daily relayer budget tracking** — currently enforced per-minute via TokenBucket; daily accumulation not tracked yet. Pending Phase 3d polish.

## Unreleased

Next: first live `pmbot test-order --live --confirm-real-trade` once allowance is in, then 7-day paper-to-live observation window.
