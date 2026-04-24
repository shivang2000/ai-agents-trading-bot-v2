# Plan — AI Orchestration Layer on Top of `trading-bot-v2`

## Context
The user is building an AI-driven passive-income system via **funded-account forex
trading**, progressing through the FundingPips ladder (5k → 10k → 100k → 200k →
own capital). The existing `trading-bot-v2` (Python + Docker + EC2) is already
**live-trading** — $5k Step 1 passed, Vantage $30→$376 account demonstrated,
5 strategies, PropFirmGuard, Telethon-based Telegram listener, Claude-Haiku
signal parser, MT5 via RPyC, SQLite journal, Slack notifications wired, 36-case
backtest matrix committed. The bot runs but is **under-tested, unobserved, and
unimproved over time.**

This plan adds a **Paperclip-driven orchestration layer on the same EC2 box**
that stacks on top of the bot without touching its hot path, layered across
three phases. **Claude Max 20x ($200/mo) is the LLM backbone** for the cold
lane; Anthropic API remains the hot-path signal-parser because it's already
there and trivially cheap at Haiku volumes. Window is roughly 4–6 weeks for P1,
driven by the user's ~2-month Max subscription budget certainty.

## What we are (and are NOT) building
We are building a **cold-lane orchestrator on top of the existing bot**:
- NOT re-building the bot. It stays unchanged in v1 except for small reads and test coverage.
- NOT building a telegram-ingestor. `src/telegram/` already does this via Telethon.
- NOT replacing PropFirmGuard. It stays authoritative for per-trade risk.
- NOT an LLM-in-the-loop trading system. Claude Max never sits in the tick path.

We ARE building:
- A **Paperclip** deployment on the same EC2 instance, with a React UI reachable over Tailscale.
- **Four scheduled agents** (Phase 1) that each invoke **Claude Code CLI** for reasoning (uses Max sub, not API).
- A **testing workstream** bringing the bot from ~1 test file to meaningful unit + integration + paper-replay coverage.
- A **shared read path** from paperclip into the bot's SQLite journal and log file.

## Target architecture — single EC2 instance
```
┌─ EC2 instance ─────────────────────────────────────────────────┐
│                                                                │
│  Docker compose stack:                                         │
│                                                                │
│    metatrader5 (gmag11/metatrader5_vnc)                        │
│      ├─ MT5 GUI via noVNC at :8080                             │
│      ├─ VNC at :5900                                           │
│      └─ RPyC at :8001                                          │
│                                                                │
│    trading-bot (existing, unchanged core)                      │
│      ├─ src/telegram/  — Telethon listener + parser            │
│      ├─ src/analysis/  — 5 strategies + SMC + Claude 0.65 gate │
│      ├─ src/risk/      — manager + PropFirmGuard + sizer       │
│      ├─ src/mt5/       — RPyC client                           │
│      ├─ data/trading_bot_v2.db — SQLite (trades journal)       │
│      └─ logs/trading.log                                       │
│                                                                │
│    paperclip (NEW — Node.js + embedded Postgres)               │
│      ├─ React UI → exposed via Tailscale (no public ingress)   │
│      ├─ Claude Code adapter → uses Max 20x via `claude --print`│
│      └─ Scheduled agents (heartbeats)                          │
│                                                                │
│    Shared volumes:                                             │
│      - bot's data/  → mounted read-only into paperclip agents  │
│      - bot's logs/  → mounted read-only into paperclip agents  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

Bot ↔ paperclip interfaces:
- **Read path:** paperclip agents read the bot's SQLite DB (read-only mount, SQLite WAL mode) and `logs/trading.log`.
- **Control path:** paperclip agents invoke bot scripts via docker exec (`deploy-propfirm.sh`, `ec2-restart.sh`, `healthcheck.sh`, `backtest_scalping.py`) through Claude Code's bash tool.
- **Notification path:** agents post to the same Slack webhook the bot already uses.
- **No direct writes from paperclip into the bot's DB in v1** — preserves bot's authority over its own state. (v2 may add a `pending_external_signals` table the bot opts into polling.)

## Phase 1 — MVP (target: 4–6 weeks)
### Agents (four)
1. **Deployment Agent**
   - Wraps existing scripts: `deploy-propfirm.sh`, `ec2-restart.sh`, `ec2-status.sh`, `healthcheck.sh`.
   - Triggers: UI button, health-check heartbeat every 5 min, Slack command.
   - Actions: switch config overlay (5k/10k/100k), `docker compose up/down`, run health check; on MT5 VNC session failure, post a Slack alert with a one-click noVNC URL for manual re-login (full VNC automation deferred to P2).
   - LLM use: on each deploy, Claude Code reviews `git diff` vs. last deploy, generates release notes into the paperclip audit log.

2. **Log Reporter Agent → Slack**
   - Heartbeat: every 30 min initially; widen to 2 h after paper-trade phase.
   - Reads: `trading_bot_v2.db` (recent trades, open positions), `logs/trading.log` tail.
   - Action: Claude Code summarizes into a structured Slack block (open positions, fills, errors, PnL delta) and posts via existing webhook.
   - De-dup: tracks last-seen row id so it never re-reports.

3. **Post-Trade Reviewer Agent**
   - Heartbeat: hourly.
   - Reads: newly-closed trades since last run, plus the signal source that produced each trade (telegram channel / own-strategy id).
   - Action: Claude Code critiques each trade, writes a lesson row into `paperclip.kb_notes` keyed by trade_id + source; updates `paperclip.source_scores` with win-rate/expectancy per Telegram channel.

4. **Management (CEO) Agent**
   - Heartbeat: daily at 21:30 UTC (after Friday close).
   - Reads: trades journal + source_scores + kb_notes + FundingPips rule state + current overlay.
   - Action: produces a daily summary — FundingPips compliance status, ladder progression ("ready to buy next challenge?"), proposed config changes, Telegram DM the user.

### Testing workstream (runs weeks 1–2 in parallel with agent build)
- Unit tests in `tests/unit/`:
  - `src/risk/manager.py` — directional exposure + position cap + daily loss + DD tiers
  - `src/risk/position_sizer.py` — leverage-aware sizing edge cases
  - `src/analysis/signal_generator.py` — dual-loop ordering, session filter
  - `src/analysis/smc_confluence.py` — OB/FVG/sweep scoring
  - `src/analysis/claude_signal_filter.py` — confidence gate pass/fail paths
  - `src/telegram/parser.py` — signal format variations from the ≤5 configured channels
  - `src/telegram/listener.py` — with mocked Telethon client
- Integration tests in `tests/integration/`:
  - Mocked RPyC MT5 stub — trade lifecycle (open → modify → close)
  - Mocked Telegram channel — end-to-end signal → PropFirmGuard → MT5 call
  - Mocked Slack webhook — notification dispatch + retry
- Replay:
  - Add a paper-trade mode to the bot (config flag `dry_run: true`) and replay 1 week of historical ticks through it.
- CI: GitHub Actions on push to `main` runs unit + integration; replay runs nightly.

### Gate promotion (before flipping external-signal path to live)
- ≥ 100 paper trades on replayed data
- Sharpe ≥ 1.0 on those trades
- Max drawdown ≤ bot's configured `max_drawdown_pct`
- Zero `PropFirmGuard` rule violations in the paper run

### Infra work in P1
- Provision Tailscale on EC2; expose paperclip UI only via Tailscale address.
- Install Claude Code CLI on EC2, authenticate with Max 20x account (**this box becomes the primary Claude Max login**; avoid logging in elsewhere).
- Create `paperclip` Postgres (embedded, local to the box).
- Add `docker-compose.paperclip.yml` overlay or extend the EC2 compose.
- Central secret store on EC2 (`.env` extended — no secrets in git).
- Backup: nightly rsync of `paperclip` Postgres and bot's SQLite to S3.

## Phase 2 — scale + safety (weeks ~6–10)
Add agents:
- **Portfolio Manager Agent** — multi-account awareness, correlation control, exposure rebalancing suggestions.
- **Fund Manager Agent** — capital allocation across challenges; decides when to fund the next rung (5k payout → 10k challenge, etc.).
- **Meta-Risk Agent** — cross-account FundingPips rule check (e.g. aggregate DD across accounts, news-window alignment).
- **Deployment Agent v2** — adds VNC automation via xdotool/pyautogui for hands-free MT5 re-login.
- **Signal-Source Scorer Agent** — auto-mutes sources whose expectancy goes negative over rolling window.

Add infra:
- Bot logs rendered in paperclip UI (custom React page reads a shared table).
- Postgres replacement for SQLite if the single-writer bottleneck bites (deferred until we see it).

## Phase 3 — self-improvement (months ~3+)
Add agents:
- **Research Agent** — yt-dlp + auto-captions / Whisper → Claude summarize → `kb_notes`; daily ingest of ≤ 10 curated YouTubers.
- **Strategy Evolver Agent** — offline: Claude proposes param tweaks; runs `scripts/backtest_scalping.py` matrix; gate-promotes variants that beat the champion on Sharpe / DD / trade-count bar.
- **Ops / Watchdog Agent** — system-health heartbeats (CPU, disk, MT5 reachability, Claude Max quota), alerts on Slack.

Explicitly rejected even at P3:
- YouTube live-stream watching (low signal-to-noise; tiered stack required; do not build).

## Load-bearing risks (design-level)
1. **Claude Max headless auth on EC2.** Claude Code OAuth needs a one-time login on the box. Secondary logins elsewhere can invalidate the session. Treat EC2 as the *primary* Max login; avoid hopping between devices on this account.
2. **Max rate pool is shared.** All paperclip agents + any personal Claude Code usage share the 5-hour reset pool. Sequence heartbeats; add a file-lock wrapper around every `claude --print` call.
3. **SQLite single-writer.** Bot writes; paperclip reads. Use SQLite WAL mode and read-only mount. If the bot ever locks, paperclip reads must tolerate busy/retry.
4. **RPyC brittleness.** MT5 bridge dies silently. Deployment Agent's health-check heartbeat catches this; watchdog auto-restarts MT5 container (carefully — not during open positions).
5. **Audit trail for FundingPips scaling reviews.** The journal must be immutable-append, link each trade to its signal source, agent decision chain, and rule-check outcome. `paperclip.kb_notes` + `source_scores` + the bot's `trading_bot_v2.db` together form the record.
6. **Headless MT5 session expiry.** MT5 occasionally forces re-login. P1 = Slack alert + manual noVNC click. P2 = VNC automation.
7. **Claude-flow conventions in the repo.** The bot's `CLAUDE.md` is claude-flow-based (hierarchical-mesh topology, swarm orchestration). We're not using claude-flow in the orchestration layer — paperclip stands alone. We should document this divergence in a new `docs/orchestrator.md` to avoid confusion later.

## Critical files (to create / modify)
**New (paperclip side):**
- `docker-compose.paperclip.yml` or extension of existing compose
- Paperclip agent configs (one per agent) — Claude Code prompts, schedules, reads/writes
- `paperclip-postgres/init.sql` — tables: `kb_notes`, `source_scores`, `agent_runs`, `deploy_audit`
- `docs/orchestrator.md` — why paperclip; how Max auth works on this box; how to add new agents
- Tailscale config on EC2
- GitHub Actions CI workflow under `.github/workflows/ci.yml`

**Modify (bot side — minimal changes in v1):**
- `src/main.py` — add optional `dry_run` flag (paper-trade mode) driven by config
- `config/base.yaml` — add `dry_run: false` default
- `src/telegram/listener.py` — add a "last-seen source-side message id" metric emitted to logs (helps source scoring)
- `tests/unit/` + `tests/integration/` — expanded coverage per testing workstream
- `.github/workflows/ci.yml` — new

**Read-only consumption (no modification):**
- `data/trading_bot_v2.db` (SQLite, read-only mount into paperclip)
- `logs/trading.log` (read-only mount into paperclip)
- `config/fundingpips-*.yaml` (read by Deployment + Management agents to know current overlay)

## Verification (gate before live hybrid operation)
1. All new unit tests pass in CI.
2. Integration tests with mocked RPyC / Telegram / Slack pass in CI.
3. Replay harness runs 1 week of recorded ticks through paper mode with zero PropFirmGuard violations and ≥ 100 paper trades.
4. Paperclip agents survive a 72-hour dry-run on EC2 (no crashes, heartbeats respected, Slack clean).
5. Chaos drill: kill MT5 container mid-position — Deployment Agent detects, posts Slack alert, no silent data loss.
6. FundingPips audit log reviewed: every trade has `source`, `rule_checks`, `decision_trail` linked.
7. Claude Max quota usage measured over 48 hours — headroom ≥ 30% before going live.

Only after all seven pass do we flip the `approved_signals` consumer to live and let external-signal trades execute on the funded account.

## Out of scope for v1 (explicit YAGNI)
- Airdrops, preptop, non-forex income streams
- Selling / licensing; signals subscription; managed accounts
- Multi-tenant paperclip; user accounts; subscription billing
- Public web UI (Tailscale only)
- Hermes in v1 (paperclip covers orchestration; bot owns Telegram ingest)
- Autonomous self-rewriting of production code
- YouTube live-stream watching (rejected at all phases)
- VNC automation in v1 (P2)
- Migration from SQLite to Postgres for bot data (revisit if locking bites)

## Timeline
- **Weeks 1–2:** Testing workstream + paperclip bring-up + Deployment Agent + Log Reporter.
- **Weeks 3–4:** Post-Trade Reviewer + Management Agent. Paper-trade replay running nightly. Chaos drills.
- **Week 5:** Verification gate. If green, flip external-signal path live on $5k account.
- **Week 6:** Monitor first week of live hybrid; harden Log Reporter + Slack noise level; prepare P2 kick-off.

Contingency: if the testing workstream uncovers PropFirmGuard edge cases that require fixes in the bot, P1 slips by 1–2 weeks — acceptable because live is gated on the verification checklist.

## Next steps (post-plan-approval, outside plan mode)
1. Create a GitHub branch `orchestrator-p1` on `trading-bot-v2`.
2. Stand up Tailscale on EC2.
3. `docker-compose.paperclip.yml` — paperclip + its Postgres, sharing volumes.
4. Install + authenticate Claude Code on EC2 with the user's Max account.
5. Implement agents and testing workstream in parallel using the `feature-dev` / `executor` pipeline.
6. Each agent ships with tests; CI is green before merge.
7. Convert this plan into a working spec (`docs/superpowers/specs/...`) inside the bot repo once plan mode exits.
