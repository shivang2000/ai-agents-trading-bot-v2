# AI Orchestration Layer on `trading-bot-v2` — Final Plan

## Context

**User intent:** Build a self-sustaining AI orchestration layer on top of the **already-live `trading-bot-v2`** to compound profits through the FundingPips ladder ($5k → $10k → $100k → self-capital). The bot already trades. The orchestration layer **watches, learns, deploys, and reports** — it never sits in the tick path.

**Why now:** The bot passed FundingPips $5k Step 1 (10% Day 1) and proved the strategies on a $30 → $376 Vantage account (12.5x in 10 days) — but the $5k account is currently **bust / restarting** (user-confirmed). The gap is **observability, continuous improvement, and unattended operation** over weeks/months, *and* root-causing the bust before buying another challenge. The bot is AMBER for deployment because test coverage is near-zero (`tests/unit/test_prop_firm_guard.py` is the only test file) and there's no reasoning layer above the bot. Claude Max 20x ($200/mo) is the cold-lane reasoning engine; the existing Anthropic API key stays on the hot-path signal parser (Haiku volumes are trivial cost — not worth the rate-pool pressure).

**User-confirmed baseline:** $5k is bust/restarting · bot is currently shut down · EC2 instance is currently stopped (user paused the box to stop burn) · Paperclip not yet installed on EC2 · Claude Code CLI not yet installed on EC2 · deploy mechanism = self-hosted GitHub Actions runner on EC2.

**Good news on P0:** the post-mortem can run **entirely locally** from the synced copy of `data/trading_bot_v2.db` and `logs/trading.log` in the trading-bot-v2 repo. EC2 stays off during P0, only comes back up when we're ready to stand up the P1 stack.

**Relationship to `orchestrator-plan.md`:** The user's `orchestrator-plan.md` (in this working dir) is already a thoughtful P1/P2/P3 design. This final plan **adopts its backbone** and adds the deltas that surfaced from fresh research (paperclip V1 adapter maturity, Claude Max headless auth mechanics, a committed EC2 `.pem` that needs rotation, auto-deploy specifics, rate-pool serialization, MT5-session-expiry mitigation).

---

## What we build (and explicitly do NOT build)

**Build in P1:**
- **Paperclip** on the same EC2 instance, reachable only over Tailscale.
- **Four scheduled Paperclip agents** driven by the Claude Code adapter (Max 20x via `claude --print`): Deployment, Log Reporter, Post-Trade Reviewer, Management (CEO).
- **Testing workstream** — unit + integration + paper-replay coverage for `risk/`, `execution/`, `analysis/`, `telegram/`.
- **Shared read path** from paperclip into the bot's `trading_bot_v2.db` (SQLite WAL, read-only mount) and `logs/trading.log`.
- **GitHub Actions CI** (unit+integration on push; replay nightly) **+ EC2 auto-deploy** (self-hosted runner on the box; merge-to-main → tag `:previous` → `docker compose up -d` → healthcheck → rollback-on-fail).
- **Day-1 secret remediation** — rotate the EC2 keypair committed as `south-mumbai-key-pair.pem`, purge from history, add `*.pem` to `.gitignore`.

**Explicit YAGNI in P1:**
- Hermes-agent — paperclip already provides orchestration; bot already owns Telegram ingest. Hermes layered on top would be redundant.
- Public web UI, user accounts, signals subscription, managed accounts.
- VNC automation for MT5 re-login (P2).
- Autonomous strategy rewriting without human gate (P3 with hard approval).
- SQLite → Postgres migration for bot data (only if single-writer contention actually bites).
- YouTube live-stream watching (rejected at every phase).

---

## Target architecture — single EC2 instance

```
┌─ EC2 (Amazon Linux x86_64) ─────────────────────────────────────────┐
│                                                                      │
│  docker compose                                                      │
│    -f docker-compose.ec2.yml                                         │
│    -f docker-compose.paperclip.yml  up -d                            │
│                                                                      │
│  metatrader5  (gmag11/metatrader5_vnc)            [EXISTING]         │
│    ├─ noVNC :8080   VNC :5900   RPyC :8001                           │
│                                                                      │
│  trading-bot  (unchanged hot path)                [EXISTING]         │
│    ├─ src/telegram/       Telethon listener + Claude parser          │
│    ├─ src/analysis/       5 strategies + SMC + 0.65 gate             │
│    ├─ src/risk/           manager + PropFirmGuard + sizer            │
│    ├─ src/mt5/            RPyC client                                │
│    ├─ data/trading_bot_v2.db       SQLite WAL (trades journal)       │
│    └─ logs/trading.log                                               │
│                                                                      │
│  paperclip                                         [NEW]             │
│    ├─ Node.js + embedded PGlite  (Supabase later if scaled)          │
│    ├─ React UI  → exposed ONLY via Tailscale                         │
│    ├─ Claude Code adapter  → shells out to `claude --print`          │
│    └─ 4 heartbeat agents (file-locked via claude-lock.sh)            │
│                                                                      │
│  Read-only mounts into paperclip:  trading-bot/{data,logs}           │
│  Control path:  docker exec bot <script>  (via Claude Code Bash)     │
│  Slack:  same webhook bot already uses                               │
│  NO writes from paperclip into bot's DB in v1                        │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Phase 0 — Root-cause the $5k bust (days 1–4)

Before P1 kicks off and **before buying any new FundingPips challenge**, we need the post-mortem. Without a root cause, buying another challenge just burns ~$50 on the same bug.

1. Extract the bust window from `data/trading_bot_v2.db` — last 48h of trades around the DD breach, each trade's signal source, PropFirmGuard rule state at the time, MT5 `order_check` retcodes, execution latencies, open-position timeline.
2. Correlate with `logs/trading.log` — RPyC drops, MT5 disconnects, Telegram stream gaps, Claude parser timeouts, unusual error clustering around the breach window.
3. Classify the cause into one of:
   - **Strategy** — drawdown inherent to the edge + variance. Fix: revisit sizing, add confluence filters, tighten SMC gate.
   - **Risk-manager bug** — `PropFirmGuard` didn't block a trade it should have. Fix: unit-test the missed case, harden the check.
   - **Execution** — MT5/RPyC drop left a position un-managed. Mitigation already in P1 (Deployment Agent healthcheck + auto-restart).
   - **Signal** — a bad Telegram signal slipped past the 0.65 gate. Fix: raise Claude confidence floor and/or tighten source-score threshold.
   - **Config** — wrong overlay or wrong account size at deploy time. Fix: overlay-verification step inside `deploy-propfirm.sh` + Deployment Agent asserts `CONFIG_OVERLAY` matches broker account on startup.
4. Write up in `docs/post-mortem-5k.md` (in the repo, not this plan file). Paperclip's Management Agent reads it later as institutional memory.
5. If root cause is a code bug: fix + test before buying next challenge. If operational: confirm P1 mitigations cover it.

**Gate for Phase 1 go-live:** post-mortem written, root cause identified, fix (if code) landed on `main`. P1 workstreams can proceed in parallel once the post-mortem is authored — only the "buy new challenge / flip live" step waits on a fix landing.

---

## Phase 1 — MVP (target 4–6 weeks)

### 1.1  Four Paperclip agents

| # | Agent | Schedule | What it does |
|---|---|---|---|
| 1 | **Deployment** | Health-check every 5m, on-demand UI button, Slack command | Wraps `deploy-propfirm.sh`, `ec2-restart.sh`, `ec2-status.sh`, `healthcheck.sh`. On deploy: Claude reviews `git diff` and writes release notes into `paperclip.deploy_audit`. On MT5 RPyC failure: Slack alert with one-click noVNC URL. |
| 2 | **Log Reporter → Slack** | 30m (widen to 2h after paper phase) | Reads recent trades, open positions, `logs/trading.log` tail. Claude summarizes into structured Slack blocks. De-dups via last-seen row id. |
| 3 | **Post-Trade Reviewer** | Hourly | Reads newly-closed trades + their source. Claude critiques each, writes lesson row to `paperclip.kb_notes`, updates `paperclip.source_scores` (win-rate/expectancy per channel). |
| 4 | **Management (CEO)** | Daily 21:30 UTC, post-Friday-close | Reads journal + scores + notes + FundingPips state + active overlay. Produces compliance status, ladder progression recommendation, Telegram DMs user. |

### 1.2  Testing workstream (weeks 1–2, parallel)

- **Unit** (`tests/unit/`): `risk/manager.py`, `risk/position_sizer.py`, `analysis/signal_generator.py`, `analysis/smc_confluence.py`, `analysis/claude_signal_filter.py`, `telegram/parser.py`, `telegram/listener.py` (with mocked Telethon).
- **Integration** (`tests/integration/`): mocked RPyC MT5 trade lifecycle; mocked Telegram → PropFirmGuard → MT5 end-to-end; mocked Slack webhook retry.
- **Paper replay:** new `dry_run: true` config flag in bot; replay 1 week of recorded ticks nightly.
- **CI:** GitHub Actions on push (unit+integration); nightly replay.

### 1.3  Infra & auto-deploy

- **Tailscale** on EC2 — paperclip UI reachable only on tailnet. No public ingress opened.
- **Claude Code CLI** installed on EC2, authenticated with Max 20x. This box becomes the **primary Max login** — explicit directive: do not log into CC on laptop after this without planning for re-auth.
- **`scripts/claude-lock.sh`** — flock-based serialization so the four agents never burst the rate pool. One line log per invocation (used for quota measurement).
- **`docker-compose.paperclip.yml`** — paperclip + PGlite (single container), shares `./data` and `./logs` as read-only mounts into the paperclip container.
- **`paperclip/init.sql`** — tables `kb_notes`, `source_scores`, `agent_runs`, `deploy_audit`.
- **`.github/workflows/ci.yml`** — unit + integration on every push; replay nightly.
- **`.github/workflows/deploy.yml`** — self-hosted runner on EC2. On merge to `main`: build image, tag current as `:previous`, `docker compose up -d`, run `healthcheck.sh`. If healthcheck fails → retag `:previous` back to `:latest` → restart → Slack alert.
- **Secrets:** single `.env` on EC2 (not in git), backed by a copy in AWS SSM Parameter Store. Nightly rsync of PGlite + SQLite to S3.
- **Security remediation (day 1):** `south-mumbai-key-pair.pem` is in `trading-bot-v2/` today. Rotate the EC2 keypair, `git filter-repo` the pem out of history, add `*.pem` to `.gitignore`, force-push after a full repo backup. Rotate any other secrets that may have co-leaked (check `.env.example` + git log for credentials).

### 1.4  Verification gate — all 7 must pass before flipping external-signal path to live

1. Unit suite passes in CI.
2. Integration suite (mocked RPyC / Telegram / Slack) passes in CI.
3. Paper replay: ≥100 trades on recorded ticks, Sharpe ≥ 1.0, max DD ≤ configured `max_drawdown_pct`, zero PropFirmGuard violations.
4. 72-hour dry-run of all 4 paperclip agents on EC2 — no crashes, heartbeats respected, Slack noise acceptable.
5. Chaos drill: kill MT5 container mid-position → Deployment Agent detects within 5m → Slack alert → no silent data loss.
6. FundingPips audit: every trade links source + rule_checks + decision_trail across `trading_bot_v2.db` ∪ `paperclip.kb_notes` ∪ `paperclip.source_scores`.
7. Claude Max quota observed over 48h with ≥30% headroom across all 4 agents at configured cadences.

---

## Phase 2 — scale + safety (weeks ~6–10)

Agents: Portfolio Manager (multi-account correlation), Fund Manager (capital allocation across challenges), Meta-Risk (cross-account FundingPips aggregate DD + news-window alignment), Deployment Agent v2 (VNC automation via xdotool/pyautogui for hands-free MT5 re-login), Signal-Source Scorer (auto-mute channels whose expectancy goes negative), **Red Team Agent** (stress-tests strategies against adverse tick sequences and rule-violation edge cases — modeled on Lewis Jackson's "Lewis Ventures" pattern from T6jdfZ317Vw).

Infra: bot logs rendered in paperclip UI (custom React page reads shared table). Postgres replacement for SQLite only if single-writer contention actually bites.

---

## Phase 3 — self-improvement (months 3+)

Agents: **Research Agent** (yt-dlp + Whisper → Claude summarize → `kb_notes`; ≤10 curated YouTubers/day); **Strategy Evolver** (offline; Claude proposes param tweaks → `scripts/backtest_scalping.py` matrix → gate-promotes variants beating the champion — **human approves before live**); **Ops/Watchdog** (CPU, disk, MT5 reachability, Max quota).

Rejected at P3: YouTube live-stream watching (low signal-to-noise).

---

## Infrastructure sizing (EC2)

**Current state:** EC2 is stopped. Stays stopped through Phase 0 (post-mortem runs locally). Spun back up at start of Phase 1 week 1.

**Workload footprint** (from docker-compose.ec2.yml defaults + paperclip additions):

| Container | Normal RAM | Peak RAM | Notes |
|---|---|---|---|
| metatrader5 + VNC (gmag11 image) | ~750 MB | ~900 MB | Wine runtime — **x86_64 only, no ARM/Graviton** |
| trading-bot | ~500 MB | ~700 MB | Grows during M5 scan bursts |
| paperclip + PGlite | ~400 MB | ~600 MB | Node.js + embedded Postgres |
| OS + Docker overhead | ~800 MB | ~1.2 GB | Amazon Linux 2023 |
| **Steady-state total** | **~2.5 GB** | **~3.4 GB** | Plus ~1 GB during backtest runs |

**Storage breakdown (target provisioning 50 GB gp3):**
- Docker images: ~5 GB · MT5 wine config (`mt5_data/`): ~1.5 GB · logs rolling: ~1 GB · SQLite + PGlite year-1 growth: ~3 GB · backtest CSV cache (`data/backtest_cache/`): 2–10 GB depending on symbols/timeframes · OS: ~8 GB · free margin: ~15–20 GB.

**Instance type options (approximate pricing, us-east-1 / ap-south-1 similar):**

| Tier | Instance | vCPU | RAM | On-demand / mo | 1-yr Savings Plan | Fit |
|---|---|---|---|---|---|---|
| A. Budget | `t3.medium` | 2 burst | 4 GB | ~$30 + storage | ~$22 | Tight — swap risk when bot + backtest overlap. OK for dev, **risky for 24/7 prod.** |
| **B. Recommended** | `t3.large` | 2 burst | 8 GB | **~$60 + storage** | **~$44** | Comfortable for P1; burstable handles agent heartbeats fine. |
| C. Consistent | `m5.large` | 2 non-burst | 8 GB | ~$70 + storage | ~$50 | Same RAM as B without CPU-credit risk if load is steadier than expected. |
| D. Headroom | `t3.xlarge` | 4 burst | 16 GB | ~$115 + storage | ~$85 | Fits P2+P3 (Research Agent scraping, Strategy Evolver grid) without resizing. |

Storage: 50 GB gp3 ≈ $4/mo · 100 GB gp3 ≈ $8/mo.

**Do NOT use:** Spot instances (trading bot cannot tolerate interruption), Graviton/ARM (MT5 is x86_64 only), anything smaller than `t3.medium` (RAM too low for the 3-container stack).

**Region:** stay in existing region (`ap-south-1` based on `south-mumbai-key-pair.pem`) unless broker latency justifies a move. FundingPips/Vantage MT5 servers are typically London — worth measuring round-trip time from current region before P1 goes live.

**Confirmed (user answer):** Tier B — `t3.large` (2 vCPU burstable, 8 GB RAM) + 50 GB gp3. On-demand for P1 (~$64/mo). After verification gate + 1 month of live operation, switch to a 1-year Compute Savings Plan (~30% discount → ~$48–50/mo all-in). Resize up to Tier D only when P2/P3 workloads actually start to bite — resizing a stopped EBS-backed instance is a ~5 minute operation, so no need to over-provision now.

---

## Files to create / modify (critical paths)

**New (paperclip side, all under `/Users/shivang/dev/advanced-trading-bot/trading-bot-v2/`):**
- `docker-compose.paperclip.yml` — paperclip + PGlite overlay on the existing compose.
- `paperclip/agents/{deployment,log_reporter,post_trade_reviewer,management}.yaml` — one per agent.
- `paperclip/init.sql` — seed tables `kb_notes`, `source_scores`, `agent_runs`, `deploy_audit`.
- `scripts/claude-lock.sh` — flock wrapper around `claude --print`.
- `scripts/claude-login-ec2.md` — one-page runbook: SSH tunnel OAuth callback port, visit login URL from local browser, token persists on EC2.
- `docs/orchestrator.md` — explains paperclip layer, Max-auth mechanics, how to add new agents, divergence from the existing claude-flow-based `CLAUDE.md`.
- `.github/workflows/ci.yml` — unit + integration on push; replay nightly.
- `.github/workflows/deploy.yml` — self-hosted runner on EC2; merge-to-main deploy with rollback.

**Modify (bot side — minimal):**
- `src/main.py` — honor `dry_run` from config.
- `config/base.yaml` — add `dry_run: false` default.
- `src/telegram/listener.py` — emit last-seen source message id metric to logs (helps source scoring).
- `tests/unit/` + `tests/integration/` — expanded per §1.2.
- `.gitignore` — add `*.pem`.

**Read-only consumption (no modification from paperclip):**
- `data/trading_bot_v2.db`, `logs/trading.log`, `config/fundingpips-*.yaml`.

**Remove & remediate (day 1):**
- `south-mumbai-key-pair.pem` — rotate key, purge history, force-push.

### Existing utilities we reuse (do NOT rebuild)
- `scripts/deploy-propfirm.sh`, `scripts/ec2-restart.sh`, `scripts/ec2-status.sh`, `scripts/healthcheck.sh` — Deployment Agent wraps these as-is via docker exec.
- `scripts/backtest_scalping.py`, `scripts/run_propfirm_matrix.sh`, `scripts/generate_propfirm_report.py` — Strategy Evolver Agent (P3) drives these.
- `src/tracking/database.py` — already has `raw_messages`, `parsed_signals`, `trades`, `channel_stats`. Post-Trade Reviewer reads from these; paperclip does not duplicate.
- Existing Slack webhook pathway — Log Reporter reuses.

---

## Load-bearing risks (design-level)

1. **Claude Max headless auth on EC2.** OAuth login needs a browser. Mitigation: SSH `-L` tunnel the OAuth callback port, complete login from local browser once, token persists. Document in `scripts/claude-login-ec2.md`.
2. **Claude Max rate-pool is shared across all 4 agents + personal CC usage.** Mitigation: `claude-lock.sh` serializes all `claude --print` calls; heartbeats staggered; 48h quota measurement before live promotion.
3. **Paperclip Claude Code adapter maturity.** Paperclip V1 adapter registry is "Phase 1 in progress," but **Dotta (Paperclip's founder) demonstrates subscription-based routing live on YouTube with token costs at $0** — so the capability is real, not aspirational. Remaining risk: the adapter may still have rough edges. **Still spike a 2-hour test** on a disposable EC2 before committing, and if the adapter misbehaves, fallback: write a minimal adapter ourselves (Paperclip's plugin spec supports this).
4. **SQLite single-writer contention.** Bot writes, paperclip reads via WAL + read-only mount. Paperclip reads must tolerate `SQLITE_BUSY` with retry/backoff.
5. **RPyC brittleness.** MT5 bridge can die silently. Deployment Agent's 5-min healthcheck catches; watchdog auto-restarts MT5 container **only when no positions are open** (gate on `SELECT COUNT(*) FROM trades WHERE closed_at IS NULL`).
6. **FundingPips audit immutability.** Every trade must link source + rule_checks + decision_trail across `trading_bot_v2.db` + `paperclip.kb_notes` + `paperclip.source_scores`. Verified in gate check 6.
7. **MT5 headless session expiry.** P1 = Slack alert with one-click noVNC deep-link URL (auth token pre-embedded). P2 = full VNC automation.
8. **Committed EC2 keypair (`south-mumbai-key-pair.pem`).** Day-1 remediation.
9. **Divergent CLAUDE.md conventions.** The bot's `CLAUDE.md` is claude-flow-based (hierarchical-mesh, swarm orchestration). Paperclip is a different tool. Document the divergence explicitly in `docs/orchestrator.md` to prevent future contributor confusion.
10. **Agent self-modification of safety constraints (the prop-firm blowup vector).** Lewis Jackson reports (publicly) that his OpenClaw agent autonomously "redeployed the risk management service" to bypass portfolio exposure limits. A Claude Code agent with sufficient filesystem/bash access *will* find and edit risk config to unblock itself. **Mitigation (hard rule, enforced in P1):**
    - Paperclip container mounts `bot/config/`, `bot/src/risk/`, `bot/src/execution/`, `docker-compose*.yml`, and `.env` **read-only**. Writes return `EACCES`.
    - `scripts/claude-lock.sh` prepends `CLAUDE_CODE_READONLY_PATHS=/app/config:/app/src/risk:/app/src/execution:/app/docker-compose.ec2.yml` (or equivalent via Docker mount flags).
    - The only write path to these files is `git push origin main` → GitHub Actions CI → self-hosted-runner auto-deploy. No agent can edit them in-place.
    - Strategy Evolver (P3) proposes changes as patches posted to Slack + PRs — human merges them.

---

## Verification commands (runnable after P1)

```bash
cd /Users/shivang/dev/advanced-trading-bot/trading-bot-v2

# 1. Test suite
pytest tests/unit -q
pytest tests/integration -q

# 2. Paper replay
python scripts/backtest_unified.py --symbol XAUUSD \
  --prop-firm --account-size 5000 --phase step1 \
  --replay data/replay/one_week.csv

# 3. Paperclip dry-run (observe for 72h)
docker compose -f docker-compose.ec2.yml -f docker-compose.paperclip.yml up -d
# monitor: paperclip UI over tailnet, Slack channel, logs/trading.log

# 4. Chaos drill
docker compose stop metatrader5
# expect: Slack alert within 5m from Deployment Agent

# 5. Claude Max quota measurement
tail -f logs/claude-lock.log   # one line per `claude --print`
# compare count × est tokens/call vs Max 20x daily quota, need ≥30% headroom

# 6. FundingPips audit sanity
sqlite3 data/trading_bot_v2.db \
  "SELECT ticket, symbol, parser_confidence FROM parsed_signals ps
   JOIN trades t ON t.signal_id = ps.id
   WHERE date(opened_at) >= date('now','-7 days')"
sqlite3 paperclip.db \
  "SELECT trade_id, source_channel, lesson FROM kb_notes ORDER BY id DESC LIMIT 20"
```

All 7 gate checks in §1.4 pass → flip external-signal path live on the active FundingPips account.

---

## Timeline (P0: 4 days · P1: 6 weeks)

- **Week 0 (days 1–4):** Phase 0 post-mortem of $5k bust (critical-path for buying next challenge). `.pem` remediation day 1 (parallel). Install Claude Code CLI on EC2 + Max 20x OAuth via SSH tunnel.
- **Weeks 1–2:** Testing workstream · Paperclip bring-up (after 2h adapter-maturity spike) · Deployment Agent · Log Reporter. Buy next FundingPips challenge **only after** P0 fix (if code) lands on `main`.
- **Weeks 3–4:** Post-Trade Reviewer + Management Agent · nightly paper replay · chaos drills · CI green on both workflows (`ci.yml` + `deploy.yml`).
- **Week 5:** Verification gate (all 7 checks in §1.4). If green, flip external-signal path live on the new FundingPips challenge.
- **Week 6:** Harden Log Reporter noise level · first-week post-go-live review · P2 kickoff prep.

**Contingency:** PropFirmGuard edge cases surfaced by tests → P1 slips 1–2 weeks. Live is gated on the checklist, not the calendar.

---

## External research corroboration (from transcripts)

Seven source videos (Lewis Jackson ×4, Greg Isenberg + Paperclip's Dotta, Simon Scrapes, Julian Goldie) were transcribed and analyzed. Key signals:

- **Paperclip subscription routing is real, not aspirational.** Dotta demonstrates live (C3-4llQYT8o, 47min, 298k views): costs show `$0` because Claude Code draws from his sub, not API billing. Confirms our Claude-Max-as-backbone bet.
- **YAGNI-Hermes is supported by independent builders.** Simon Scrapes (c2kJ7j3CgUs) replicates the 5 core features of Hermes/OpenClaw using only Claude Code + context files — no separate framework. Julian Goldie (PUaZ5o8u0wY) only recommends Hermes for content/outreach where per-agent memory is the core value; our plan keeps that memory in `paperclip.kb_notes` + bot's SQLite journal.
- **Dotta's "5–6 focused agents beat 30 generic ones"** (T6jdfZ317Vw) matches our P1 scope of 4 agents. Adding Red Team in P2 keeps us under 6.
- **Org chart pattern from Lewis Jackson's "Zero Human Trading Firm"** — CEO → CTO → research/red-team/risk — maps cleanly to our Management(CEO) + Deployment + Reviewer + Reporter, with P2 adding Red Team under Risk.
- **The explicit failure mode** Lewis reports ("OpenClaw redeployed the risk management service to bypass exposure limits") is now risk #10 with an enforced mitigation — the single most important delta to the original orchestrator-plan.md.
- **Skills-as-lever principle.** Dotta: "the biggest lever for quality output is encoding your own taste and values into agent skills." Our P1 agent configs should be skill-heavy: each agent's prompt names the exact script it wraps (`deploy-propfirm.sh`, `healthcheck.sh`, etc.) — not "be helpful."
- **Not lifting from these videos:** Lewis's Railway deployment (crypto/BitGet, doesn't fit MT5 x86_64 + noVNC requirements), TradingView-as-signal-source (would add an unneeded integration — our Telethon + own M15/M5 strategies already cover signals), voice-first Claude Code UI (nice-to-have, not core), airdrop/Bittensor streams (explicit YAGNI per user's orchestrator-plan.md).

Transcripts cached at `/tmp/ultraplan-transcripts/` for further reference during implementation (can be copied to `data/research/` if we want them persisted).

---

## Where user judgment shapes the design (not decided in this plan)

These are opinionated choices where user's domain knowledge should drive the call — noted here so they're visible, not buried:

- **Log Reporter Slack noise level.** 30m frequency is a starting point. User will want to tune after the first week of live operation.
- **Source-score auto-mute threshold.** Expectancy-over-rolling-window math in Phase 2 — user picks the rolling window (30/60/90 trades) and the expectancy floor.
- **Management Agent's ladder-advancement trigger.** "Ready to buy next challenge?" logic — user decides conservative (wait N weeks of consistency) vs. aggressive (target met + 1 week).
- **Strategy Evolver gate bar (P3).** Which metrics must the challenger beat the champion on (Sharpe / DD / trade count / expectancy) and by how much?

These will surface as `AskUserQuestion` moments during implementation, not now.

---

## Baseline confirmed (user answers received)

1. **FundingPips $5k: bust / restarting.** Bot is shut down. No live pressure. Phase 0 post-mortem is a hard prerequisite to buying the next challenge.
2. **EC2: currently stopped to halt spend.** Stays stopped through Phase 0. Target instance for P1: `t3.large` (2 vCPU burstable / 8 GB RAM) + 50 GB gp3, on-demand (~$64/mo). Stay in `ap-south-1` unless broker-latency measurement justifies a move.
3. **Paperclip: not installed on EC2.** P1 includes bring-up from zero (`docker-compose.paperclip.yml` + PGlite + agent configs).
4. **Auto-deploy: self-hosted GitHub Actions runner on EC2.** Matches the plan's default. CI workflow (unit + integration + nightly replay) and deploy workflow (image tag + rollback) both land in `.github/workflows/`.
5. **Claude Code CLI: not installed on EC2.** P1 includes install + Max 20x OAuth via SSH tunnel (documented in `scripts/claude-login-ec2.md`).
