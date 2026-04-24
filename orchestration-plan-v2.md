# AI Orchestration Layer on `trading-bot-v2` — Final Plan

## Executive summary

**Goal.** Build a self-sustaining AI orchestration layer on top of the already-live `trading-bot-v2`, using a Claude Max 20x subscription as the cold-lane reasoning engine, to compound profits through the FundingPips (and later FTM + others) ladder: Step 1 → Step 2 → Master → next-size challenge.

**Situation.** Bot passed the $5k Step 1 on its own; the $5k account then busted on Step 2 due to **human manual trading during a high-impact news event** (not a code bug). EC2 was terminated, DB + logs lost (no backup existed — a separate lesson). Restarting from scratch: fresh EC2, fresh keypair, fresh repo layout, this plan.

**Approach — hybrid deploy.** GHA self-hosted runner on EC2 runs tests + builds + pushes the image. Paperclip's Deployment Agent decides *when* to deploy (no open positions, market conditions right), handles healthcheck, rollback, and operational decisions. Four P1 agents: **Deployment** (ops), **Log Reporter** (Slack digests), **Post-Trade Reviewer** (writes lessons to kb_notes), **Management** (auto-promotes ladder phases using strict data-driven criteria). Risk-10 mitigation: all bot files mounted **read-only** into Paperclip; the only write path for config / risk / compose is git → CI → deploy.yml.

**P1 code changes that gate the next FP challenge purchase.**
1. **Diff 1** — RiskManager central news gate (closes all signal-source bypasses)
2. **Diff 2** — Pre-news FLAT in position_monitor (closes open positions before NFP / FOMC / CPI)
3. **Diff 3** — Foreign-position monitor (alerts on any MT5 position without `magic=200000`)
4. **MT5 investor-password pattern** — physically prevents manual trading
5. Tests for all three, plus expanded news calendar (add ECB / BoE / PPI / Retail Sales)

**Infra.** `t3.large` (2 vCPU burstable / 8 GB) + 50 GB gp3 in `ap-south-1`, ~$64/mo on-demand. Tailscale for access (public ingress closed after setup). Nightly S3 sync of DB + logs + Paperclip PGlite. EBS `DeleteOnTermination=false`, API termination protection enabled. Never lose data again.

**Multi-account + multi-propfirm.** `config/accounts.yaml` + `config/propfirms/<name>.yaml` separate "which platform/server" from "which rulebook." Management Agent auto-promotes Step 1 → 2 → Master when strict SQL criteria pass. Fund Manager (P2) allocates payouts to next-rung challenge. P1 supports FundingPips; FTM rulebook templated for later (MT5-only platforms in P1, others deferred to P3).

**Phase timeline.** P0 root-cause post-mortem (done, interview-based). P1 = 4–6 weeks to the "safe to buy next challenge" gate. P2 = multi-active + Red Team agent (weeks 6–10). P3 = Research + Strategy Evolver agents (month 3+).

**Non-goals / YAGNI.** Hermes as a separate orchestration layer (paperclip + bot cover it). Public web UI, signal subscription business, managed accounts, airdrops, YouTube live-stream watching. MT4 / cTrader / Match-Trader / TradeLocker adapters (defer to P3 unless a prop firm offers MT5-only).

---

## Table of contents

1. [Executive summary](#executive-summary)
2. [Context](#context)
3. [What we build (and explicitly do NOT build)](#what-we-build-and-explicitly-do-not-build)
4. [Target architecture — single EC2 instance](#target-architecture--single-ec2-instance)
5. [Phase 0 — root-cause the $5k bust](#phase-0--root-cause-the-5k-bust-days-14)
6. [Phase 1 — MVP (4–6 weeks)](#phase-1--mvp-target-46-weeks)
    - [Four Paperclip agents](#11--four-paperclip-agents)
    - [Testing workstream](#12--testing-workstream-weeks-12-parallel)
    - [Infra & auto-deploy](#13--infra--auto-deploy)
    - [Verification gate (7 checks)](#14--verification-gate--all-7-must-pass-before-flipping-external-signal-path-to-live)
7. [Phase 2 — scale + safety](#phase-2--scale--safety-weeks-610)
8. [Phase 3 — self-improvement](#phase-3--self-improvement-months-3)
9. [Infrastructure sizing (EC2)](#infrastructure-sizing-ec2)
10. [Files to create / modify](#files-to-create--modify-critical-paths)
11. [Load-bearing risks](#load-bearing-risks-design-level)
12. [Verification commands](#verification-commands-runnable-after-p1)
13. [Timeline](#timeline-p0-4-days--p1-6-weeks)
14. [External research corroboration (YouTube transcripts)](#external-research-corroboration-from-transcripts)
15. [Baseline confirmed (user answers)](#baseline-confirmed-user-answers-received)
16. [Multi-account ladder](#multi-account-ladder-p1-upgrade-from-user-qa)
17. [Execution artifacts — P1 copy-paste-ready](#execution-artifacts--p1-copy-paste-ready)
    - [Artifact 1 — Multi-account ladder infrastructure](#artifact-1--multi-account-ladder-infrastructure)
    - [Artifact 2 — `scripts/bootstrap-ec2.sh`](#artifact-2--scriptsbootstrap-ec2sh)
    - [Artifact 3 — GitHub Actions workflows](#artifact-3--github-actions-workflows)
    - [Artifact 4 — Paperclip agent configs](#artifact-4--paperclip-agent-configs)
    - [Artifact 5 — `docs/post-mortem-5k.md` scaffold](#artifact-5--docspost-mortem-5kmd-scaffold-phase-0)
    - [Risk-10 mount enforcement](#risk-10-mount-enforcement-belt-and-braces)
18. [Multi-propfirm support (FundingPips + FTM)](#multi-propfirm-support-user-scope-expansion)
19. [Phase 0 outcome — post-mortem reconstruction](#phase-0-outcome--post-mortem-reconstruction)
20. [News-filter gap analysis](#news-filter-gap-analysis-p1-bug-to-fix-before-next-challenge)
21. [P1 code diffs (copy-paste-ready)](#p1-code-diffs-copy-paste-ready)
22. [Additional specs](#additional-specs-user-requested-deep-dives)
    - [Spec 1 — EC2 provisioning runbook (AWS CLI)](#spec-1--ec2-provisioning-runbook-aws-cli-copy-paste)
    - [Spec 2 — Prop-firm support-ticket templates](#spec-2--prop-firm-support-ticket-templates)
    - [Spec 3 — Paperclip foreign-position handler + Slack block](#spec-3--paperclip-deployment-agent-foreign-position-handler--slack-block)
    - [Spec 4 — `docker-compose.paperclip.yml`](#spec-4--docker-composepaperclipyml-concrete-overlay)
    - [Spec 5 — Slack Bolt Socket Mode handler](#spec-5--slack-bolt-socket-mode-handler)

---

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

## Deployment portability — MacBook + EC2 (user scope expansion)

**Goal:** the system runs identically on either a MacBook (dev, paper, backtests) or EC2 (live funded trading). One codebase, two compose profiles, no forks.

**Principle:** keep bot + paperclip containers platform-agnostic. Let the *host* environment differ via docker-compose overrides + a thin secrets-loader shim. The bot code sees env vars either way.

### File layout

```
trading-bot-v2/
├── docker-compose.yml                       # NEW — base stack (portable)
├── docker-compose.ec2.override.yml          # refactor of existing ec2.yml
├── docker-compose.macbook.override.yml      # NEW — Mac-only bits
├── docker-compose.paperclip.override.yml    # portable, works on both
├── scripts/
│   ├── run.sh                               # NEW — platform-detect launcher
│   ├── bootstrap-ec2.sh                     # existing (Spec 1)
│   └── bootstrap-mac.sh                     # NEW — Homebrew + Docker Desktop + Caffeine
└── src/config/secrets.py                    # NEW — env-var-first loader
```

### `docker-compose.yml` (base — portable)

```yaml
services:
  metatrader5:
    image: gmag11/metatrader5_vnc:latest
    restart: unless-stopped
    ports:
      - "${MT5_NOVNC_PORT:-8080}:3000"
      - "${MT5_RPYC_PORT:-8001}:8001"
    environment:
      - VNC_PASSWORD=${VNC_PASSWORD}
    volumes:
      - ${DATA_DIR}/mt5_data:/config/.wine

  trading-bot:
    build: .
    restart: unless-stopped
    depends_on: [metatrader5]
    env_file: [.env]
    environment:
      - MT5_HOST=metatrader5
      - MT5_PORT=8001
      - CONFIG_OVERLAY=${CONFIG_OVERLAY}
    volumes:
      - ./config:/app/config:ro
      - ${DATA_DIR}/bot:/app/data
      - ${LOG_DIR}/bot:/app/logs
```

### `docker-compose.ec2.override.yml`

```yaml
services:
  metatrader5:
    platform: linux/amd64
    deploy:
      resources:
        limits: { memory: 768M }
  trading-bot:
    deploy:
      resources:
        limits: { memory: 512M }
    healthcheck:
      test: ["CMD", "sh", "/app/scripts/healthcheck.sh"]
      interval: 60s
```

### `docker-compose.macbook.override.yml`

```yaml
services:
  metatrader5:
    platform: linux/amd64   # Rosetta 2 on Apple Silicon, native on Intel
    deploy:
      resources:
        limits: { memory: 1.2G }   # slack for emulation overhead
  trading-bot:
    deploy:
      resources:
        limits: { memory: 768M }
    stdin_open: true
    tty: true
```

### `scripts/run.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

case "$(uname -s)" in
  Darwin)
    export DATA_DIR="${DATA_DIR:-$HOME/trading-bot-v2/data}"
    export LOG_DIR="${LOG_DIR:-$HOME/trading-bot-v2/logs}"
    OVERRIDE=docker-compose.macbook.override.yml
    ;;
  Linux)
    export DATA_DIR="${DATA_DIR:-/home/ec2-user/trading-bot-v2/data}"
    export LOG_DIR="${LOG_DIR:-/home/ec2-user/trading-bot-v2/logs}"
    OVERRIDE=docker-compose.ec2.override.yml
    ;;
  *) echo "Unsupported: $(uname -s)"; exit 1 ;;
esac

mkdir -p "$DATA_DIR/bot" "$DATA_DIR/mt5_data" "$LOG_DIR/bot"

exec docker compose \
  -f docker-compose.yml \
  -f "$OVERRIDE" \
  -f docker-compose.paperclip.override.yml \
  "$@"
```

Usage: `./scripts/run.sh up -d` on either platform. Same command, correct stack.

### `src/config/secrets.py` — platform-agnostic secrets

```python
"""Secrets resolution — same code path on Mac (.env) and EC2 (SSM-injected env vars).

Both platforms: secrets are just env vars at runtime. What DIFFERS is how they
get INTO env:
  - Mac: `docker compose` reads .env directly
  - EC2: systemd drop-in runs `aws ssm get-parameters-by-path /trading-bot/<acct>/`
         at boot, writes to /etc/trading-bot-overlay.env, compose reads that

The bot is oblivious. This is the single point of access.
"""
import os

def get_secret(name: str, default: str | None = None) -> str:
    value = os.environ.get(name, default)
    if value is None:
        raise KeyError(
            f"Missing secret: {name}. "
            f"Set in .env (Mac) or SSM /trading-bot/<account>/{name.lower()} (EC2)."
        )
    return value
```

No AWS SDK in the bot itself — secrets injection is the host's responsibility.

### `scripts/bootstrap-mac.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Homebrew
command -v brew >/dev/null || \
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Core tools (caffeine via cask)
brew install python@3.12 node git jq awscli
brew install --cask caffeine docker

# Start Docker Desktop if not already
open -a Docker || true
echo "Docker Desktop — settings: Resources → Memory ≥ 6 GB, CPUs ≥ 4"

# Python env
python3.12 -m venv ~/.venvs/trading-bot-v2
source ~/.venvs/trading-bot-v2/bin/activate
pip install -e '.[dev]'

# Claude Code CLI (same binary as EC2)
npm install -g @anthropic-ai/claude-code

cat <<'EOF'
==> Done. Remaining manual steps:
  1. Open Caffeine from menu bar, turn it ON (☕)
  2. System Settings → Lock Screen → "Turn display off": Never
  3. Power: plugged in during trading hours
  4. claude login  (OAuth browser flow)
  5. cp .env.example .env ; fill in secrets
  6. ./scripts/run.sh up -d
EOF
```

### What's different per platform

| Concern | MacBook | EC2 |
|---|---|---|
| Secrets source | `.env` (chmod 600, gitignored) | SSM Parameter Store → systemd drop-in → env |
| Storage root | `$HOME/trading-bot-v2/data` | `/home/ec2-user/trading-bot-v2/data` (EBS) |
| Backup | Time Machine + optional manual S3 | systemd timer nightly S3 sync |
| Sleep prevention | **Caffeine** (☕ menu bar, plugged in) | N/A |
| Platform emulation | Rosetta 2 for MT5 (Apple Silicon) | Native x86 |
| Public ingress | None — Tailscale only | Tailscale only after bootstrap |
| Deploy on change | `git pull && ./scripts/run.sh restart` | GHA self-hosted runner auto-deploys |
| GHA runner | Not installed | Installed |
| Best use | Dev · tests · backtests · paper replay | Live funded trading |

### Explicit recommendation despite portability

The system **runs** on MacBook. The system **should not run live funded accounts** on MacBook — laptop sleep / reboot / network-switch mid-open-position = busted account. Caffeine prevents sleep, not security updates or network transitions or physical mobility.

**Use MacBook for:**
- Fast dev loop (edit → `pytest` → commit)
- Backtest sweeps (`python scripts/backtest_*.py` — no Docker needed)
- Paper replay (`dry_run: true`, Caffeine on)
- Full-stack integration testing before EC2 deploy
- Post-incident offline debugging

**Use EC2 exclusively for:**
- Live funded-account trading (Step 1 / Step 2 / Master)
- 24/7 paperclip agents
- Anything tied to real money or prop-firm reputation

### Plan deltas baked elsewhere

- Artifact 2 splits into `bootstrap-ec2.sh` (existing) + `bootstrap-mac.sh` (new).
- Artifact 3 (CI workflows) runs the same tests on both; only `deploy.yml` is EC2-specific.
- Spec 4 (`docker-compose.paperclip.yml`) becomes `docker-compose.paperclip.override.yml` — identical content, new name.
- Spec 6 safety-gate gains: "Live trading happens on EC2, not MacBook. Confirm `CONFIG_OVERLAY` loaded on EC2 AND MacBook bot is STOPPED (to avoid dual-bot account interference)."

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

---

## Multi-account ladder (P1 upgrade from user Q&A)

Confirmed design decisions: **Hybrid** deploy · **Auto-promotion from P1** · **AWS SSM** creds · **Manual challenge buy** (Slack alert) in P2 · **Explicit trigger** for P2 multi-active.

Each "account slot" cycles Step 1 → Step 2 → Master as three separate MT5 logins. Across sizes, the ladder repeats ($5k → $10k → $100k), with master payouts funding the next rung's challenge purchase.

**Auto-promotion safety:** criteria are strict SQL, not LLM reasoning — the Management Agent can only observe whether `balance_pct_gain >= target_pct AND trading_days_used >= trading_days_min AND violations = 0 AND no_single_day_over_40pct_profit`. If any data is missing/uncertain, promotion waits another day.

---

## Execution artifacts — P1 copy-paste-ready

### Artifact 1 · Multi-account ladder infrastructure

**`config/accounts.yaml`** (git-tracked, read-only to paperclip):

```yaml
accounts:
  - id: fp-5k-step1-v2
    size: 5000
    broker: fundingpips
    phase: step1
    overlay: fundingpips-5k-step1
    target_pct: 10
    trading_days_min: 3
    creds_ssm_prefix: /trading-bot/fp-5k-step1-v2
    next_on_pass: fp-5k-step2-v1

  - id: fp-5k-step2-v1
    size: 5000
    phase: step2
    overlay: fundingpips-5k-step2
    target_pct: 5
    trading_days_min: 3
    creds_ssm_prefix: /trading-bot/fp-5k-step2-v1
    next_on_pass: fp-5k-master-v1

  - id: fp-5k-master-v1
    size: 5000
    phase: master
    overlay: fundingpips-5k-master
    target_pct: null
    creds_ssm_prefix: /trading-bot/fp-5k-master-v1
    payouts_fund: fp-10k-step1-v1
```

**Paperclip PGlite schema (new tables):**

```sql
CREATE TABLE account_state (
  account_id        TEXT PRIMARY KEY,
  status            TEXT CHECK(status IN ('pending','active','passed','bust','paused')),
  activated_at      TIMESTAMPTZ, passed_at TIMESTAMPTZ,
  last_balance      NUMERIC, max_equity NUMERIC,
  trading_days_used INT DEFAULT 0,
  violations        JSONB DEFAULT '[]'::JSONB,
  updated_at        TIMESTAMPTZ DEFAULT NOW()
);
CREATE TABLE kb_notes (id SERIAL, trade_id INT, account_id TEXT, source_channel TEXT,
                       lesson TEXT, pattern_tags TEXT[], created_at TIMESTAMPTZ DEFAULT NOW());
CREATE TABLE source_scores (channel_id TEXT, account_phase TEXT, win_rate NUMERIC,
                            expectancy NUMERIC, sample_size INT, last_updated TIMESTAMPTZ,
                            PRIMARY KEY (channel_id, account_phase));
CREATE TABLE deploy_audit (id SERIAL, sha TEXT, triggered_by TEXT, outcome TEXT,
                           notes TEXT, ts TIMESTAMPTZ DEFAULT NOW());
CREATE TABLE ceo_journal (id SERIAL, account_id TEXT, entry TEXT, ts TIMESTAMPTZ DEFAULT NOW());
```

`account_state` is writable **only** by the Management Agent. All others read-only.

**SSM parameter layout (one-time per account):**

```bash
aws ssm put-parameter --name /trading-bot/fp-5k-step1-v2/mt5_login    --value "11836008"        --type SecureString
aws ssm put-parameter --name /trading-bot/fp-5k-step1-v2/mt5_password --value "********"        --type SecureString
aws ssm put-parameter --name /trading-bot/fp-5k-step1-v2/mt5_server   --value "FundingPips-Demo" --type String
```

IAM: EC2 instance role gets `ssm:GetParameter` + `kms:Decrypt` scoped to `arn:aws:ssm:*:*:parameter/trading-bot/*`.

---

### Artifact 2 · `scripts/bootstrap-ec2.sh`

Idempotent; one run on fresh AL2023 t3.large.

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "==> 1. OS + swap"
sudo dnf -y update
sudo dnf -y install git curl wget unzip tar gzip jq make gcc fail2ban
sudo timedatectl set-timezone UTC
if [ ! -f /swapfile ]; then
  sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile
  sudo mkswap /swapfile && sudo swapon /swapfile
  echo "/swapfile swap swap defaults 0 0" | sudo tee -a /etc/fstab
fi

echo "==> 2. Docker + Compose v2"
sudo dnf -y install docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
sudo mkdir -p /usr/libexec/docker/cli-plugins
sudo curl -sSL https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64 \
  -o /usr/libexec/docker/cli-plugins/docker-compose
sudo chmod +x /usr/libexec/docker/cli-plugins/docker-compose
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{"log-driver":"json-file","log-opts":{"max-size":"100m","max-file":"5"}}
EOF
sudo systemctl restart docker

echo "==> 3. Python 3.12 + Node.js 20"
sudo dnf -y install python3.12 python3.12-pip
sudo alternatives --install /usr/bin/python3 python3 /usr/bin/python3.12 10
[ ! -d "$HOME/.nvm" ] && curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
export NVM_DIR="$HOME/.nvm"; . "$NVM_DIR/nvm.sh"
nvm install 20 && nvm alias default 20

echo "==> 4. AWS CLI v2"
if ! command -v aws >/dev/null; then
  cd /tmp && curl -sSL https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
  unzip -q awscliv2.zip && sudo ./aws/install && rm -rf aws awscliv2.zip
fi

echo "==> 5. Claude Code CLI"
npm install -g @anthropic-ai/claude-code
# MANUAL next: scripts/claude-login-ec2.md — SSH tunnel OAuth, then `claude login`

echo "==> 6. Tailscale"
command -v tailscale >/dev/null || curl -fsSL https://tailscale.com/install.sh | sudo sh
# MANUAL: sudo tailscale up --ssh --authkey=tskey-XXX

echo "==> 7. Security"
sudo systemctl enable --now fail2ban
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl reload sshd

echo "==> 8. Nightly S3 backup (systemd timer)"
sudo tee /etc/systemd/system/tb-backup.service >/dev/null <<'EOF'
[Unit]
Description=trading-bot nightly backup
[Service]
Type=oneshot
User=ec2-user
EnvironmentFile=/etc/trading-bot.env
ExecStart=/bin/bash -c 'aws s3 sync /home/ec2-user/trading-bot-v2/data s3://${S3_BUCKET}/tb/data/ && aws s3 sync /home/ec2-user/paperclip-data s3://${S3_BUCKET}/paperclip/data/'
EOF
sudo tee /etc/systemd/system/tb-backup.timer >/dev/null <<'EOF'
[Unit]
Description=Nightly tb-backup
[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable tb-backup.timer

echo "==> 9. Clone repo"
cd ~
[ ! -d "trading-bot-v2" ] && git clone https://github.com/shivang2000/trading-bot-v2.git

cat <<'EOF'

==> DONE. Manual steps remaining:
  1. scripts/claude-login-ec2.md  (OAuth via SSH tunnel)
  2. sudo tailscale up --ssh
  3. GHA runner  (repo Settings → Actions → Runners → New self-hosted)
  4. First MT5 login via noVNC: http://<tailscale-ip>:8080
  5. aws ssm put-parameter  for each account
  6. /etc/trading-bot.env  with S3_BUCKET
EOF
```

---

### Artifact 3 · GitHub Actions workflows

**`.github/workflows/ci.yml`** — on every push/PR:

```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
jobs:
  test:
    runs-on: self-hosted
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12', cache: 'pip' }
      - run: pip install -e '.[dev]'
      - run: ruff check src tests
      - run: mypy src
      - run: pytest tests/unit -q --cov=src
      - run: pytest tests/integration -q
  build:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t trading-bot-v2:${{ github.sha }} .
      - name: Notify Deployment Agent
        run: |
          curl -X POST http://localhost:8731/events/ci-image-built \
            -H 'Content-Type: application/json' \
            -d "{\"sha\":\"${{ github.sha }}\"}"
```

**`.github/workflows/deploy.yml`** — triggered **only** by the Deployment Agent via `repository_dispatch` (never on push directly):

```yaml
name: Deploy
on:
  repository_dispatch: { types: [agent-deploy] }
jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - run: docker tag trading-bot-v2:latest trading-bot-v2:previous || true
      - run: docker tag trading-bot-v2:${{ github.event.client_payload.sha }} trading-bot-v2:latest
      - name: Compose up
        run: |
          cd /home/ec2-user/trading-bot-v2
          docker compose -f docker-compose.ec2.yml -f docker-compose.paperclip.yml up -d
      - name: Healthcheck (30s settle)
        run: |
          sleep 30
          if ! /home/ec2-user/trading-bot-v2/scripts/healthcheck.sh; then
            docker tag trading-bot-v2:previous trading-bot-v2:latest
            docker compose -f docker-compose.ec2.yml up -d trading-bot
            curl -X POST -d '{"text":"ROLLBACK: healthcheck failed"}' "${{ secrets.SLACK_WEBHOOK }}"
            exit 1
          fi
```

Split of concerns: CI runs on every push (tests + build). Deploy runs only when Deployment Agent has verified operational conditions (no open positions, MT5 healthy).

---

### Artifact 4 · Paperclip agent configs

All four read `/mnt/bot/...` (read-only mount), write only to `/paperclip/data/...`. Risk-10 enforced at the Docker mount layer.

**`paperclip/agents/deployment.yaml`**

```yaml
name: deployment
schedule: "*/5 * * * *"
model: claude-opus-4-7-1m
read_paths:
  - /mnt/bot/data/trading_bot_v2.db
  - /mnt/bot/logs/trading.log
  - /mnt/bot/config/accounts.yaml
write_paths: [/paperclip/data/deploy_audit, /paperclip/data/account_state]
tools: [bash, sqlite_read, slack_post, gh_cli, aws_ssm_read]
prompt: |
  DevOps agent.

  EVERY 5m HEARTBEAT:
    1. /mnt/bot/scripts/healthcheck.sh. On non-zero:
       - MT5 RPyC down + zero open positions → docker compose restart metatrader5
       - MT5 RPyC down + open positions → Slack alert with noVNC deep-link; NO restart
       - Bot container dead → restart
       - Paused-by-drawdown → Slack summary; no action (intended behavior)
    2. Poll http://localhost:8731/events for ci-image-built.
       If event present AND zero open positions → gh workflow run deploy.yml
         -f sha=<sha>. Else queue for next tick.

  ON activate(account_id) (invoked by Management Agent):
    1. Close open positions (docker exec + bot CLI).
    2. Wait: poll until SELECT COUNT(*) FROM trades WHERE closed_at IS NULL = 0.
    3. Fetch SSM /trading-bot/<account_id>/{mt5_login,mt5_password,mt5_server}.
    4. Write /etc/trading-bot-overlay.env (chmod 600) with new creds + overlay.
    5. docker compose stop trading-bot metatrader5 && docker compose up -d.
    6. Slack: "noVNC ready: http://<ts-ip>:8080 — manual MT5 login needed."
    7. UPDATE account_state SET status='active', activated_at=NOW() WHERE id=<new>.

  HARD RULE: never write under /mnt/bot/. Those are RO mounts.
  All code/config changes flow through git → CI → deploy.yml.
```

**`paperclip/agents/log_reporter.yaml`**

```yaml
name: log_reporter
schedule: "*/30 * * * *"
model: claude-haiku-4-5
read_paths: [/mnt/bot/data/trading_bot_v2.db, /mnt/bot/logs/trading.log]
write_paths: [/paperclip/data/log_reporter_cursor]
tools: [sqlite_read, slack_post]
prompt: |
  30-min pulse. Read trades + log tail since last cursor.
  Slack format:
    📊 *30-min — <account_id> (<phase>)*
    • Open: N (<symbols>, net $X)
    • Fills: N · Closes: N (W/L) · ΔPnL: $Y
    • Errors: <gaps or "none">
    • Account: <status>, <days_used>/<days_min>
  Update cursor. No re-reporting. Skip post on zero activity.
```

**`paperclip/agents/post_trade_reviewer.yaml`**

```yaml
name: post_trade_reviewer
schedule: "0 * * * *"
model: claude-opus-4-7-1m
read_paths:
  - /mnt/bot/data/trading_bot_v2.db
  - /mnt/bot/logs/trading.log
  - /mnt/bot/config/accounts.yaml
write_paths: [/paperclip/data/kb_notes, /paperclip/data/source_scores]
tools: [sqlite_read, pglite_write]
prompt: |
  For every CLOSED trade since last run:
    1. Pull trade + parsed_signals.raw_message + rule outcomes.
    2. Critique ≤150w: signal clarity, SMC alignment, SL/TP quality,
       PropFirmGuard margin, lessons.
    3. INSERT kb_notes {trade_id, account_id, source_channel, lesson, pattern_tags}.
    4. Upsert source_scores (channel_id, account_phase): win_rate, expectancy, sample.
  Tags: liquidity_sweep, ob_entry, range_fail, session_momentum,
        news_surprise, too_tight_sl, too_wide_sl, good_r_multiple.
  Never touches open trades.
```

**`paperclip/agents/management.yaml`**

```yaml
name: management
schedule: "30 21 * * *"
model: claude-opus-4-7-1m
read_paths:
  - /mnt/bot/data/trading_bot_v2.db
  - /mnt/bot/logs/trading.log
  - /mnt/bot/config/accounts.yaml
  - /mnt/bot/config/fundingpips-*.yaml
write_paths: [/paperclip/data/account_state, /paperclip/data/ceo_journal]
tools: [sqlite_read, pglite_write, dispatch_deployment_activate, telegram_dm]
prompt: |
  CEO Agent. Daily 21:30 UTC.

  1. Active account state: balance, peak_equity, dd_pct, days_used, violations.
  2. STRICT data-driven pass criteria (never LLM-estimated):
       step1/step2: (balance - start) / start >= target_pct
                    AND days_used >= trading_days_min
                    AND violations = 0
                    AND no_single_day_exceeds_40pct_of_total_profit
       master: never auto-"passes"; payout milestones tracked separately.
  3. ON PASS:
       a. kb_note: "Step N passed for <account>: profit=X%, days=Y."
       b. UPDATE account_state status='passed', passed_at=NOW() WHERE id=<cur>.
       c. next := accounts.yaml[cur].next_on_pass.
       d. UPDATE account_state status='active', activated_at=NOW() WHERE id=<next>.
       e. Dispatch deployment.activate(<next>).
       f. Telegram DM: "Step N passed. Promoting <cur> → <next>. Switching MT5."
  4. ON VIOLATION/BUST: set status='bust'. NO auto-promote, NO auto-buy.
     Telegram + Slack alert with post-mortem template link.
  5. Daily Telegram DM digest: account, phase, today P&L, cumulative, violations,
     days used, distance to target, next action.

  If ANY criterion data is missing/uncertain → DO NOT PROMOTE. Wait a day.
```

---

### Artifact 5 · `docs/post-mortem-5k.md` scaffold (Phase 0)

```markdown
# Post-mortem: $5k FundingPips bust

**Date:** <YYYY-MM-DD>
**Account:** fp-5k-v1
**Breach type:** <daily DD | overall DD | consistency | other>

## Timeline
<chronology: account open → first trade → peak equity → breach moment → bot halt>

## Queries

```sql
-- Trades ±24h around breach
SELECT t.id, t.opened_at, t.closed_at, t.symbol, t.action, t.volume,
       t.entry_price, t.stop_loss, t.take_profit, t.close_price, t.pnl,
       ps.parser_confidence, ps.channel_id
FROM trades t LEFT JOIN parsed_signals ps ON t.signal_id = ps.id
WHERE t.opened_at BETWEEN datetime('<bust_ts>','-24 hours')
                       AND datetime('<bust_ts>','+1 hour')
ORDER BY t.opened_at;

-- Hourly equity trajectory
SELECT datetime(opened_at,'start of hour') AS hr,
       SUM(pnl) AS h_pnl,
       SUM(SUM(pnl)) OVER (ORDER BY datetime(opened_at,'start of hour')) AS cum
FROM trades WHERE closed_at IS NOT NULL GROUP BY hr ORDER BY hr;

-- Signal source quality in breach week
SELECT ps.channel_id, COUNT(*) sigs, SUM(t.pnl) total, AVG(t.pnl) avg_pnl
FROM parsed_signals ps JOIN trades t ON t.signal_id = ps.id
WHERE ps.parsed_at >= datetime('<bust_date>','-7 days')
GROUP BY ps.channel_id ORDER BY total;
```

## Log correlation

```bash
grep -nE 'ERROR|RiskLimitExceeded|RPyC|MT5|claude|timeout|disconnect' \
  logs/trading.log | awk '$1 > "<bust_ts>-01:00" && $1 < "<bust_ts>+00:30"'
```

## Classification (tick exactly one)

- [ ] **Strategy drawdown + variance** — many small losses, PropFirmGuard didn't trigger, cadence normal. Fix: lower risk_pct, add SMC gate, revisit R:R.
- [ ] **Risk-manager bug** — a trade placed that should have been blocked. Fix: unit-test + harden `src/risk/manager.py`.
- [ ] **Execution fault** — RPyC drop / stale price / wrong SL. Fix: retry hardening + P1 Deployment Agent healthcheck.
- [ ] **Bad signal through gate** — geometrically valid but strategically wrong. Fix: raise min_confidence → 0.75, tighten source-score.
- [ ] **Config mismatch** — overlay wrong for phase. Fix: Deployment Agent asserts overlay == active account phase.

## Fix landed

- [ ] PR merged on main
- [ ] Unit test added that would have caught this
- [ ] Integration test covering the path
- [ ] Replay reproduces a no-breach outcome

## Decision

- [ ] ✅ Safe to buy next challenge
- [ ] ❌ Additional work needed: <describe>
```

---

### Risk-10 mount enforcement (belt-and-braces)

`docker-compose.paperclip.yml` mounts into the paperclip container:

```yaml
volumes:
  - ./config:/mnt/bot/config:ro
  - ./src:/mnt/bot/src:ro
  - ./docker-compose.ec2.yml:/mnt/bot/docker-compose.ec2.yml:ro
  - ./docker-compose.paperclip.yml:/mnt/bot/docker-compose.paperclip.yml:ro
  - ./.env:/mnt/bot/.env:ro
  - ./data:/mnt/bot/data:ro
  - ./logs:/mnt/bot/logs:ro
  - paperclip-data:/paperclip/data
```

Any `claude --print` write attempt outside `/paperclip/data` returns `EACCES`. The user-accepted escalator risk is bounded by challenge fee; this mount policy is zero-cost and stops the specific attack Lewis Jackson's OpenClaw demonstrated (self-modifying risk config in-place).

---

## Multi-propfirm support (user scope expansion)

User flagged potential use of multiple prop firms beyond FundingPips (e.g., FundedTraderMarkets). Researched FTM variations page — rulebook differs materially:

| Dimension | FundingPips | FundedTraderMarkets |
|---|---|---|
| Variants | 2-step only | 1-step · 2-step · instant-funding |
| Profit target | 8% step1, 5% step2 | 10% (all evaluation variants) |
| Daily DD | 5% | 4% |
| Overall DD | 10% | 6% |
| Profit split | 80% | up to 90% |
| Sizes | 5k/10k/100k | 5k/10k/25k/50k/100k/200k/300k |
| Platforms | MT5 | MT5 · cTrader · Match-Trader · TradeLocker |
| Bot policy | ✅ allowed | ⚠️ undisclosed — require written confirmation from support |

### Architecture delta: separate "which platform/server" from "which rulebook"

**`config/propfirms/<name>.yaml`** — one rulebook per firm, referenced from `accounts.yaml`.

```yaml
# config/propfirms/fundingpips.yaml
name: FundingPips
variants:
  2-step:
    phases:
      step1:  { target_pct: 8,  daily_dd_pct: 5, overall_dd_pct: 10, min_trading_days: 3 }
      step2:  { target_pct: 5,  daily_dd_pct: 5, overall_dd_pct: 10, min_trading_days: 3 }
      master: { target_pct: null, daily_dd_pct: 5, overall_dd_pct: 10 }
    profit_split: 0.80
    consistency_rule: { max_single_day_pct_of_profit: 40 }
    bot_allowed: true
    platforms: [mt5]
```

```yaml
# config/propfirms/ftm.yaml
name: FundedTraderMarkets
variants:
  1-step:
    phases:
      evaluation: { target_pct: 10, daily_dd_pct: 4, overall_dd_pct: 6, min_trading_days: 3 }
      master:     { target_pct: null, daily_dd_pct: 4, overall_dd_pct: 6 }
    profit_split: 0.90
    bot_allowed: unknown   # MUST verify with FTM support before use
    platforms: [mt5, ctrader, match-trader, tradelocker]
  2-step:
    phases:
      step1:  { target_pct: 10, daily_dd_pct: 4, overall_dd_pct: 6, min_trading_days: 3 }
      step2:  { target_pct: 10, daily_dd_pct: 4, overall_dd_pct: 6, min_trading_days: 3 }
      master: { target_pct: null, daily_dd_pct: 4, overall_dd_pct: 6 }
    profit_split: 0.90
    bot_allowed: unknown
    platforms: [mt5, ctrader, match-trader, tradelocker]
  instant-funding:
    phases:
      master: { target_pct: null, daily_dd_pct: 4, overall_dd_pct: 6 }
    profit_split: 0.80
    bot_allowed: unknown
    platforms: [mt5, ctrader, match-trader, tradelocker]
```

### Extended `config/accounts.yaml`

```yaml
accounts:
  - id: fp-5k-step1-v2
    size: 5000
    propfirm: fundingpips        # → config/propfirms/fundingpips.yaml
    variant: 2-step
    platform: mt5
    phase: step1                 # must exist in propfirm.variants[variant].phases
    mt5_server: FundingPips-Demo
    overlay: fundingpips-5k-step1
    creds_ssm_prefix: /trading-bot/fp-5k-step1-v2
    next_on_pass: fp-5k-step2-v1

  - id: ftm-25k-1step-v1         # example; only add when ready to onboard
    size: 25000
    propfirm: ftm
    variant: 1-step
    platform: mt5
    phase: evaluation
    mt5_server: FTM-Live
    overlay: ftm-25k-1step-eval
    creds_ssm_prefix: /trading-bot/ftm-25k-1step-v1
    next_on_pass: ftm-25k-1step-master-v1
```

Management Agent resolves `propfirms/<propfirm>.yaml → variants[variant].phases[phase]` at evaluation time. Same agent code handles all firms — only config changes.

### Platform scope

| Platform | P1 | Notes |
|---|---|---|
| MT5 via RPyC | ✅ | Existing bot target. All firms that offer MT5 work with no code change. |
| MT4 | ❌ P3 | Different API. Adapter needed if a firm is MT4-only. |
| cTrader | ❌ P3 | OpenAPI / FIX. Different order model. |
| Match-Trader | ❌ P3 | REST. Smaller market share. |
| TradeLocker | ❌ P3 | REST. Newer. |

**P1 rule:** only take on propfirm variants that offer **MT5**. Defer all other platforms. Keeps the bot core untouched — multi-propfirm expansion is pure configuration until you actually need a new platform.

### Hard prerequisite: bot-allowed confirmation

Before any funded purchase on a new prop firm, raise a support ticket asking:

1. Is EA / automated / algorithmic trading permitted on evaluation **and** funded accounts?
2. Any restrictions on trading frequency, hold time, or strategies (no-HFT, minimum hold time, no-copy-trading, no-martingale, no-news)?
3. If I use an MT5 EA, will funded payouts be honored?

Get answers **in writing**. Screenshot or email forward to `docs/propfirm-approvals/<name>.md` in the repo. If bot-allowed is "no" or ambiguous → skip that firm. Without this, funded profits can be revoked retroactively by the firm.

### PropFirmGuard code-side changes (land during post-mortem fix)

- Accept a `rulebook` dict at init (loaded from the propfirm YAML) instead of hardcoded 5%/10%.
- Support variable phase sequences:
  - 1-step: `evaluation → master`
  - 2-step: `step1 → step2 → master`
  - instant-funding: `master` (no eval phase)
  - 3-step (future): `step1 → step2 → step3 → master`
- Per-firm consistency rule hook (optional).
- Unit tests cover: FundingPips 2-step happy path · FTM 1-step happy path · FTM instant-funding.

### Regional restrictions to track

FTM does not offer MT5 or cTrader to US residents. Not an issue at your current region (ap-south-1 / India-based), but when scaling to collaborators or managed accounts in other regions, check firm-by-firm residency rules. Worth a small `docs/propfirm-approvals/<name>-restrictions.md` note per firm.

---

## Phase 0 outcome — post-mortem reconstruction

Database unrecoverable (EC2 terminated, no S3 backup). Post-mortem reconstructed from user interview on 2026-04-24:

**Root cause:** *user was manually trading* on the $5k Step 2 account (after Step 1 had passed) during a high-impact news event (NFP / FOMC / CPI). Not a bot bug. Manual trade(s) blew through the DD limit.

**User commitment:** no manual trading going forward.

**Context:** Phase was Step 2 — tighter than Step 1 (5% target, same 5% daily DD). Instruments configured: all 4 (XAUUSD / XAGUSD / BTCUSD / ETHUSD).

### Fix: "system-enforced single-owner" pattern

Discipline commitment is necessary but not sufficient. Humans slip, especially around news volatility. Day-1 system guards:

1. **Magic-number enforcement.** Bot already tags every order with `magic=200000` (`src/execution/executor.py`). Positions without this magic are *foreign* (human-placed or rogue).

2. **Foreign-position monitor — Deployment Agent (P1 addition).** On every 5-min heartbeat, queries MT5 positions and inspects magic numbers. Any foreign position → immediate Slack alert + Telegram DM to user. Default `auto_close_foreign: false`; user can flip to `true` for aggressive containment (agent force-closes after 60s if unacknowledged).

3. **Bot-side real-time guard — P1 code change in `src/monitoring/position_monitor.py`.** Detects foreign positions within seconds (not minutes). Fires Slack alert immediately; does NOT auto-close (too risky to race the human).

4. **`config/accounts.yaml` gains `prohibits_manual_trading: true`** (default for every prop-firm account). Deployment Agent reads this; if true, the foreign-position monitor is active.

5. **News-window flat posture.** `config/news_calendar.csv` exists in the repo per the README — **verify as P1 task** whether strategies actually consume it to go flat ±15 min around NFP / FOMC / CPI / ECB / BoE / OPEC. If parsed but not acted on, that is a P1 bug.

6. **MT5 investor-password pattern (strongest measure).** Every MT5 account has two passwords: master (full trading) and investor (read-only). Set **both** during first MT5 login via noVNC. Store master in SSM under `/trading-bot/<account>/mt5_master_password` (bot uses this); store investor under `/trading-bot/<account>/mt5_investor_password` (you use this for viewing). Manually placing a trade from the investor login is physically impossible. This single change would have prevented the $5k bust.

7. **Pre-news alert — Management Agent.** Telegram DM ~30 min before any high-impact calendar event: "⚠️ NFP in 30m. Bot flatting at 14:00 UTC. Do not open manual trades." Explicit accountability checkpoint.

### Data-durability day-1 hardening (parallel lesson from losing the DB)

Losing the DB was avoidable. Day-1 measures baked into Artifact 2 (`bootstrap-ec2.sh`):

- Nightly `aws s3 sync` of `data/` + `logs/` + paperclip PGlite → S3 (systemd timer already in Artifact 2).
- EBS volume attribute: `DeleteOnTermination=false` — instance termination does not kill the data.
- EC2 `enable-api-termination-protection=true`. Manual override required to destroy the instance.
- SQLite `.wal` checkpointed + hourly incremental to S3 (in addition to nightly full) so RPO is ≤ 1 hour.
- Every closed trade echoed to CloudWatch Logs (free-tier 5 GB/mo) as a second-line journal independent of SQLite.

### Phase 0 complete — gate for next FundingPips challenge

- ✅ Root cause identified: human discipline failure during news event
- ✅ Fix class designed: system-enforced single-owner pattern + news discipline + data durability
- ❌ **Do NOT buy the next FundingPips challenge until the following land on `main` and are deployed:**
  - `src/monitoring/position_monitor.py` foreign-position check (bot-side, real-time)
  - Paperclip Deployment Agent foreign-position monitor (external audit, 5-min)
  - `config/news_calendar.csv` verified to actually produce flat posture around high-impact events
  - MT5 investor password set up on the new account (via first noVNC login)

Discipline alone failed once; do not rely on it again.

---

## News-filter gap analysis (P1 bug to fix before next challenge)

Investigated `src/analysis/news_filter.py` and how it's wired in production.

**What exists:** `NewsEventFilter` class is well-designed. Default window: -15 min before / +30 min after. Loads `config/news_calendar.csv` (63 rows covering 2025-2026 FOMC/NFP/CPI) or falls back to hardcoded defaults (FOMC dates + first-Friday-NFP + 13th-of-month CPI). Returns `(blocked: bool, reason: str)`.

**Current wiring — where `is_blocked` is checked:**

| Signal source | Function | Gated on news? |
|---|---|---|
| M5 scalping (5 strategies) | `signal_generator._scan_scalping` line 442 | ✅ yes |
| M15 EMA pullback | `signal_generator._scan_symbol` (around line 260) | ❌ **NO** |
| M15 London breakout | same `_scan_symbol` | ❌ **NO** |
| NY range breakout | inside `_scan_symbol` / sub-helpers | ❌ **NO** |
| NY momentum | inside `_scan_symbol` / sub-helpers | ❌ **NO** |
| Telegram signals (all channels) | `src/telegram/parser.py` handlers | ❌ **NO** |
| Central `RiskManager._validate_risk_limits` | — | ❌ **NO** |
| `_publish_signal` (line 568, central strategy-signal gate) | — | ❌ **NO** |

**Estimated coverage: ~25% of signal paths.** Three-quarters bypass the filter today.

**Secondary bugs found:**
- `config.signal_parser.news_filter_enabled` (schema.py:225) appears cosmetic — the filter is instantiated and called unconditionally in `_scan_scalping`. Flipping the config off does nothing. Fix: honor the flag in the new central check.
- Filter is **preventive only** — blocks NEW entries during window. Does NOT preemptively close open positions before a window. A position opened at 13:50 with NFP at 14:30 is left exposed.

### Fix spec — P1 (lands on `main` before buying next FundingPips challenge)

**Change 1: Move news check to RiskManager (central choke point).** Every signal, from every source, gets checked in one place.

```python
# src/risk/manager.py — inside _validate_risk_limits, BEFORE other checks
def _validate_risk_limits(self, signal: Signal) -> None:
    if self._config.signal_parser.news_filter_enabled:
        blocked, reason = self._news_filter.is_blocked(signal.timestamp)
        if blocked:
            raise RiskLimitExceeded("news_window", reason, 0)
    # ... existing 5 checks unchanged
```

Keep the M5 scalping pre-check as defense-in-depth (cheap, catches at signal-generation time before the event bus).

**Change 2: Pre-news FLAT posture (new behavior).** `src/monitoring/position_monitor.py` gets a new check every 60s:

```python
async def _check_pre_news_flat(self) -> None:
    next_event, delta = self._news_filter.time_until_next_event()
    if next_event and delta < timedelta(minutes=5):
        open_positions = await self._mt5.positions_get()
        if open_positions:
            logger.warning("Pre-news FLAT: closing %d positions before %s (in %s)",
                           len(open_positions), next_event.name, delta)
            for p in open_positions:
                await self._execution.close_position(p.ticket, reason="pre_news_flat")
```

Requires adding `time_until_next_event() -> tuple[NewsEvent | None, timedelta | None]` to `NewsEventFilter`.

**Change 3: Expand `config/news_calendar.csv`.** Currently USD-only HIGH events. Add:
- ECB Rate Decision (EUR — affects XAUUSD indirectly via DXY)
- BoE Rate Decision (GBP)
- US PPI (~15th of month, 13:30 UTC)
- US Retail Sales (~15th, 13:30 UTC)
- OPEC meetings (for future crude / commodity exposure)

Write `scripts/update_news_calendar.py` — weekly cron sourcing ForexFactory or Investing.com; commits a PR with changes for user review (never auto-merges).

**Change 4: Per-propfirm news policy.** `config/propfirms/<name>.yaml` gains:

```yaml
news_policy:
  flat_before_high_impact_min: 15
  no_new_trades_after_min: 30
  enforce_on: [step1, step2, master]
  # FundingPips: per fundingpips.yaml line 39 — "News trading RESTRICTED on master (5 min window). Max risk 2% (accounts $50k+)."
```

RiskManager loads the per-account rule via `accounts.yaml → propfirm → news_policy` instead of hardcoded constants.

**Change 5: Tests (`tests/unit/test_news_filter.py`).**
- Boundary cases: timestamp = event_time - 15min (blocked), -16min (not blocked), +30min (blocked), +31min (not blocked).
- Multiple overlapping events: timestamp in overlap zone returns the first event's reason.
- Timezone-naive input handled correctly.
- CSV missing → `_load_default_calendar` fires, not crash.
- `news_filter_enabled: false` short-circuits the check.
- Integration: publish SIGNAL event 5 min pre-NFP → RiskManager rejects with `news_window` reason, no ORDER event emitted.

### Gate for buying next FundingPips challenge

Adds 5 items to the gate from Phase 0:

- [ ] Change 1 (RiskManager central news check) landed on `main`, unit+integration green
- [ ] Change 2 (pre-news FLAT in position monitor) landed, integration test green
- [ ] Change 3 (expanded news calendar) committed — at minimum ECB + BoE + PPI + Retail Sales added
- [ ] Change 4 (per-propfirm news policy schema) defined, loaded correctly for FundingPips 2-step
- [ ] Change 5 (test suite) green in CI

Only after all five are merged and tests are green on `main` should the next FundingPips $5k challenge be purchased.

---

## P1 code diffs (copy-paste-ready)

All three changes land in one PR. Existing `RiskLimitExceeded(name, current, limit)` takes mixed types (strings already used for `direction`, `strategy_position`), so news_window fits without a new exception class.

### Diff 1 — `src/risk/manager.py` (central news gate)

```python
# At top of file (new import)
from src.analysis.news_filter import NewsEventFilter

# In RiskManager.__init__ signature (new kwarg with backward-compat default)
def __init__(
    self,
    config: AppConfig,
    event_bus: EventBus,
    ...
    news_filter: NewsEventFilter | None = None,   # NEW
) -> None:
    ...
    self._news_filter = news_filter               # NEW

# In _validate_risk_limits — ADD this as check #0, before all existing ones:
def _validate_risk_limits(self, signal: Signal) -> None:
    # 0. News-window gate — central choke point for ALL signal sources
    #    (strategy, scalping, Telegram, any future source). Respects
    #    config.signal_parser.news_filter_enabled.
    if (
        self._news_filter is not None
        and self._config.signal_parser.news_filter_enabled
    ):
        signal_ts = signal.timestamp or datetime.now(timezone.utc)
        blocked, reason = self._news_filter.is_blocked(signal_ts)
        if blocked:
            raise RiskLimitExceeded("news_window", reason, "flat")

    # 1. Max open positions ... (existing)
    ...
```

### Diff 2 — `src/analysis/news_filter.py` (add `time_until_next_event`)

```python
# Add as a new method on NewsEventFilter:
def time_until_next_event(
    self, now: datetime | None = None
) -> tuple[NewsEvent | None, timedelta | None]:
    """Return (next_event, delta_until_it) or (None, None) if calendar is exhausted."""
    now = now or datetime.now(timezone.utc)
    if now.tzinfo is None:
        now = now.replace(tzinfo=timezone.utc)
    for event in self._events:   # events are kept sorted by datetime_utc
        event_time = event.datetime_utc
        if event_time.tzinfo is None:
            event_time = event_time.replace(tzinfo=timezone.utc)
        if event_time > now:
            return event, event_time - now
    return None, None
```

### Diff 3 — `src/monitoring/position_monitor.py` (pre-news FLAT + foreign-position monitor)

```python
# Imports (add)
from datetime import timedelta
from src.analysis.news_filter import NewsEventFilter
from src.core.events import ForeignPositionEvent   # NEW event type, see Diff 4

# __init__ signature — add kwargs
def __init__(
    self,
    mt5_client: AsyncMT5Client,
    event_bus: EventBus,
    tracking_db: TrackingDB,
    ...
    news_filter: NewsEventFilter | None = None,   # NEW
    pre_news_flat_minutes: int = 5,                # NEW
) -> None:
    ...
    self._news_filter = news_filter
    self._pre_news_flat_window = timedelta(minutes=pre_news_flat_minutes)
    self._pre_news_flat_handled: set[datetime] = set()
    self._foreign_alerted_tickets: set[int] = set()

# _poll_loop — add two new checks BEFORE the existing _check_positions:
async def _poll_loop(self) -> None:
    while self._running:
        await asyncio.sleep(self._poll_interval)
        try:
            await self._check_pre_news_flat()
            await self._check_foreign_positions()
            await self._check_positions()
        except Exception:
            logger.exception("Position monitor poll error (will retry next cycle)")

# New method: pre-news FLAT
async def _check_pre_news_flat(self) -> None:
    """Close bot positions ≤ window before a high-impact news event."""
    if self._news_filter is None:
        return
    next_event, delta = self._news_filter.time_until_next_event()
    if next_event is None or delta is None:
        return
    if delta > self._pre_news_flat_window:
        return
    if next_event.datetime_utc in self._pre_news_flat_handled:
        return   # already handled this event

    positions = await self._mt5.positions_get()
    bot_positions = [p for p in positions if self._is_bot_position(p)]
    if not bot_positions:
        self._pre_news_flat_handled.add(next_event.datetime_utc)
        return

    logger.warning(
        "Pre-news FLAT: closing %d bot positions before %s (in %s)",
        len(bot_positions), next_event.name, delta,
    )
    for p in bot_positions:
        close_order = Order(
            symbol=p.symbol,
            side=OrderSide.SELL if p.side == OrderSide.BUY else OrderSide.BUY,
            order_type=OrderType.MARKET,
            volume=p.volume,
            stop_loss=0.0,
            take_profit=0.0,
            magic=200000,
            comment=f"pre_news:{next_event.name[:20]}",
        )
        await self._event_bus.publish(OrderEvent(
            timestamp=datetime.now(timezone.utc),
            order=close_order,
        ))
    self._pre_news_flat_handled.add(next_event.datetime_utc)

# New method: foreign-position monitor
async def _check_foreign_positions(self) -> None:
    """Detect manually-placed (non-bot-magic) positions and publish alerts."""
    positions = await self._mt5.positions_get()
    foreign = [p for p in positions if not self._is_bot_position(p)]

    if not foreign:
        # Reset tracking — if user closed the manual trade, re-alert on any new one
        self._foreign_alerted_tickets.clear()
        return

    new_foreign = [p for p in foreign if p.ticket not in self._foreign_alerted_tickets]
    for p in new_foreign:
        logger.error(
            "FOREIGN POSITION: ticket=%s symbol=%s side=%s vol=%s entry=%s magic=%s",
            p.ticket, p.symbol, p.side.name, p.volume, p.entry_price,
            getattr(p, "magic", "?"),
        )
        await self._event_bus.publish(ForeignPositionEvent(
            timestamp=datetime.now(timezone.utc),
            position=p,
        ))
        self._foreign_alerted_tickets.add(p.ticket)
```

### Diff 4 — `src/core/events.py` (new event type)

```python
# Add to the events module:
@dataclass(frozen=True)
class ForeignPositionEvent:
    """Emitted when a non-bot-magic position is detected on MT5.

    Handled by Slack + Telegram notifiers to alert the user.
    The bot does NOT auto-close foreign positions (too risky to race
    the human); alert-only is the P1 behavior.
    """
    timestamp: datetime
    position: Position
    message: str = "Foreign position detected — not placed by bot"
```

### Diff 5 — `src/main.py` (single news_filter instance, shared)

```python
# In main.py startup, BEFORE instantiating RiskManager / SignalGenerator / PositionMonitor:
news_filter: NewsEventFilter | None = None
if config.signal_parser.news_filter_enabled:
    news_filter = NewsEventFilter(calendar_path="config/news_calendar.csv")
    logger.info("NewsEventFilter active: %d events loaded", news_filter.event_count)

# Pass into the three components
risk_manager = RiskManager(config, event_bus, ..., news_filter=news_filter)
signal_generator = SignalGenerator(config, event_bus, mt5, news_filter=news_filter)
position_monitor = PositionMonitor(
    mt5, event_bus, db, ...,
    news_filter=news_filter,
    pre_news_flat_minutes=config.signal_parser.pre_news_flat_minutes,  # NEW config key
)

# Subscribe Slack + Telegram notifiers to ForeignPositionEvent (add to event_bus setup)
event_bus.subscribe(ForeignPositionEvent, slack_notifier.on_foreign_position)
event_bus.subscribe(ForeignPositionEvent, telegram_notifier.on_foreign_position)
```

And in `SignalGenerator.__init__` (signal_generator.py:107): replace hardcoded instantiation with the passed-in instance, so ONE filter is shared:

```python
# Before:
self._news_filter = NewsEventFilter(calendar_path="config/news_calendar.csv")

# After:
self._news_filter = news_filter or NewsEventFilter(calendar_path="config/news_calendar.csv")
```

### Diff 6 — `src/config/schema.py` (new config key)

```python
# In SignalParserConfig (around line 225)
class SignalParserConfig(BaseModel):
    ...
    news_filter_enabled: bool = True
    pre_news_flat_minutes: int = 5           # NEW — Pre-news FLAT window
```

### Diff 7 — expand `config/news_calendar.csv`

Add (illustrative — user extends as firms tighten):

```csv
2025-01-30,13:15,ECB Rate Decision,HIGH,EUR
2025-01-30,13:30,US PPI,HIGH,USD
2025-02-17,13:30,US Retail Sales,HIGH,USD
2025-02-06,12:00,BoE Rate Decision,HIGH,GBP
2025-03-06,13:15,ECB Rate Decision,HIGH,EUR
2025-03-13,13:30,US PPI,HIGH,USD
2025-03-20,12:00,BoE Rate Decision,HIGH,GBP
# ... user fills in remaining 2025-2026 calendar from ForexFactory
```

Write `scripts/update_news_calendar.py` — weekly cron that fetches ForexFactory/Investing.com API, opens a PR (never auto-merge).

### Tests — `tests/unit/test_news_filter.py` (must land in same PR)

```python
import pytest
from datetime import datetime, timedelta, timezone
from src.analysis.news_filter import NewsEventFilter, NewsEvent

NFP = datetime(2026, 2, 6, 13, 30, tzinfo=timezone.utc)

@pytest.fixture
def nf_with_nfp():
    nf = NewsEventFilter(calendar_path=None)  # triggers defaults
    nf._events = [NewsEvent("Non-Farm Payrolls (NFP)", NFP)]
    return nf

def test_block_boundary_before(nf_with_nfp):
    # exactly at -15 min → blocked (inclusive)
    assert nf_with_nfp.is_blocked(NFP - timedelta(minutes=15))[0] is True
    # -16 min → not blocked
    assert nf_with_nfp.is_blocked(NFP - timedelta(minutes=16))[0] is False

def test_block_boundary_after(nf_with_nfp):
    assert nf_with_nfp.is_blocked(NFP + timedelta(minutes=30))[0] is True
    assert nf_with_nfp.is_blocked(NFP + timedelta(minutes=31))[0] is False

def test_naive_timestamp_treated_as_utc(nf_with_nfp):
    naive = (NFP - timedelta(minutes=10)).replace(tzinfo=None)
    assert nf_with_nfp.is_blocked(naive)[0] is True

def test_time_until_next_event(nf_with_nfp):
    now = NFP - timedelta(minutes=30)
    evt, delta = nf_with_nfp.time_until_next_event(now=now)
    assert evt is not None and evt.name.startswith("Non-Farm")
    assert delta == timedelta(minutes=30)

def test_no_future_events(nf_with_nfp):
    now = NFP + timedelta(days=365)
    evt, delta = nf_with_nfp.time_until_next_event(now=now)
    assert evt is None and delta is None

def test_csv_missing_falls_back(tmp_path):
    nf = NewsEventFilter(calendar_path=str(tmp_path / "missing.csv"))
    # load_calendar logs a warning but doesn't crash; events list stays empty
    assert nf.event_count == 0  # explicit path provided → no default fallback
```

Also `tests/integration/test_risk_manager_news.py`:

```python
async def test_signal_during_nfp_rejected_by_risk_manager(risk_manager_with_nfp_filter, signal_factory):
    signal = signal_factory(timestamp=NFP - timedelta(minutes=5))
    with pytest.raises(RiskLimitExceeded) as excinfo:
        risk_manager_with_nfp_filter._validate_risk_limits(signal)
    assert excinfo.value.args[0] == "news_window"
```

### PR checklist (all must be ticked before the "safe to buy next challenge" gate flips)

- [ ] Diffs 1–7 applied
- [ ] `tests/unit/test_news_filter.py` passes (6 test cases above, extend as needed)
- [ ] `tests/integration/test_risk_manager_news.py` passes
- [ ] `tests/integration/test_position_monitor_pre_news.py` — mocked MT5 with open position, event 3 min out → verifies close orders emitted (not-yet-drafted, write during implementation)
- [ ] `tests/integration/test_position_monitor_foreign.py` — mocked MT5 position with magic != 200000 → verifies `ForeignPositionEvent` published exactly once (not re-alerted on subsequent polls)
- [ ] Manual smoke test: run bot in paper-replay, simulate a news event + foreign position, verify Slack alerts fire
- [ ] `ruff check` + `mypy` green
- [ ] Merged to `main`, deployed via the Artifact 3 CI + Deployment Agent flow

---

## Additional specs (user-requested deep-dives)

### Spec 1 — EC2 provisioning runbook (AWS CLI, copy-paste)

Sequence for fresh `t3.large` in `ap-south-1`. Do NOT reuse `south-mumbai-key-pair.pem` — generate a new keypair. Assumes `aws configure` done with an IAM user that has EC2/IAM/S3 rights.

```bash
export AWS_REGION=ap-south-1
export NAME=trading-bot-v2

# 1. Fresh keypair (never reuse the one committed to git)
aws ec2 create-key-pair --region $AWS_REGION --key-name ${NAME}-key \
  --query 'KeyMaterial' --output text > ~/.ssh/${NAME}-key.pem
chmod 600 ~/.ssh/${NAME}-key.pem

# 2. Security group — SSH from your home IP only (tighten via Tailscale later)
MY_IP=$(curl -s https://checkip.amazonaws.com)
SG_ID=$(aws ec2 create-security-group --region $AWS_REGION \
  --group-name ${NAME}-sg --description "trading-bot-v2 SG" \
  --query GroupId --output text)
aws ec2 authorize-security-group-ingress --region $AWS_REGION \
  --group-id $SG_ID --protocol tcp --port 22 --cidr ${MY_IP}/32

# 3. IAM role for SSM read + S3 backup + CloudWatch Logs
aws iam create-role --role-name ${NAME}-ec2-role --assume-role-policy-document '{
  "Version":"2012-10-17",
  "Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]
}'
aws iam put-role-policy --role-name ${NAME}-ec2-role --policy-name ssm-read \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow",
      "Action":["ssm:GetParameter","ssm:GetParameters","kms:Decrypt"],
      "Resource":["arn:aws:ssm:*:*:parameter/trading-bot/*","arn:aws:kms:*:*:key/*"]}]
  }'
aws iam put-role-policy --role-name ${NAME}-ec2-role --policy-name s3-backup \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow",
      "Action":["s3:PutObject","s3:GetObject","s3:ListBucket"],
      "Resource":["arn:aws:s3:::trading-bot-v2-backups","arn:aws:s3:::trading-bot-v2-backups/*"]}]
  }'
aws iam attach-role-policy --role-name ${NAME}-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
aws iam create-instance-profile --instance-profile-name ${NAME}-ec2-profile
aws iam add-role-to-instance-profile \
  --instance-profile-name ${NAME}-ec2-profile --role-name ${NAME}-ec2-role

# 4. S3 bucket for backups (private, versioned, KMS-encrypted)
aws s3api create-bucket --bucket trading-bot-v2-backups --region $AWS_REGION \
  --create-bucket-configuration LocationConstraint=$AWS_REGION
aws s3api put-public-access-block --bucket trading-bot-v2-backups \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-versioning --bucket trading-bot-v2-backups \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket trading-bot-v2-backups \
  --server-side-encryption-configuration '{
    "Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]
  }'

# 5. Latest AL2023 x86_64 AMI
AMI=$(aws ec2 describe-images --region $AWS_REGION --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
            "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' --output text)

# 6. Launch t3.large with 50GB gp3, DeleteOnTermination=false, encrypted
INSTANCE_ID=$(aws ec2 run-instances --region $AWS_REGION \
  --image-id $AMI --instance-type t3.large --key-name ${NAME}-key \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=${NAME}-ec2-profile \
  --block-device-mappings '[{
    "DeviceName":"/dev/xvda",
    "Ebs":{"VolumeSize":50,"VolumeType":"gp3","DeleteOnTermination":false,"Encrypted":true}
  }]' \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${NAME}}]" \
  --query 'Instances[0].InstanceId' --output text)

# 7. Termination protection
aws ec2 modify-instance-attribute --region $AWS_REGION \
  --instance-id $INSTANCE_ID --disable-api-termination

# 8. Wait ready + get public IP
aws ec2 wait instance-running --instance-ids $INSTANCE_ID --region $AWS_REGION
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --region $AWS_REGION \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "SSH: ssh -i ~/.ssh/${NAME}-key.pem ec2-user@$PUBLIC_IP"

# 9. SCP bootstrap + run it
scp -i ~/.ssh/${NAME}-key.pem scripts/bootstrap-ec2.sh \
  ec2-user@$PUBLIC_IP:/tmp/
ssh -i ~/.ssh/${NAME}-key.pem ec2-user@$PUBLIC_IP \
  "chmod +x /tmp/bootstrap-ec2.sh && /tmp/bootstrap-ec2.sh"

# Manual sequence AFTER bootstrap (order matters):
#  a. Tailscale auth:  sudo tailscale up --ssh  (browser confirm)
#  b. Once on tailnet, REMOVE the public SSH rule:
#     aws ec2 revoke-security-group-ingress --region $AWS_REGION \
#       --group-id $SG_ID --protocol tcp --port 22 --cidr ${MY_IP}/32
#     From now on SSH over Tailscale:  ssh ec2-user@<tailscale-ip>
#  c. Claude Code OAuth via SSH tunnel (scripts/claude-login-ec2.md)
#  d. GHA runner (repo Settings → Actions → Runners → New self-hosted → token)
#  e. Populate SSM per account:  aws ssm put-parameter --name /trading-bot/<id>/mt5_login ...
#  f. First-time MT5 login via noVNC: http://<tailscale-ip>:8080
```

### Spec 2 — Prop-firm support-ticket templates

#### Template A — FundingPips

> **Subject:** Confirmation: automated / EA trading allowed on evaluation + funded
>
> Hi FundingPips team,
>
> I'm preparing to start a new $5k 2-step challenge using an automated trading system (MetaTrader 5 Expert Advisor) I own and operate. Before I purchase, could you please confirm the following in writing:
>
> 1. Is EA / algorithmic / automated trading permitted on (a) Step 1, (b) Step 2, (c) the funded master account?
> 2. Are there any restrictions I should be aware of — specifically:
>    - minimum hold time per trade,
>    - maximum trades per hour or per day,
>    - news-trading restrictions (FOMC / NFP / CPI windows),
>    - copy-trading / signal-provider rules,
>    - scalping / HFT-like restrictions,
>    - rules around weekend / server-downtime positions?
> 3. If I use an EA and pass Steps 1 + 2, will funded payouts be honored? Has any account ever had payouts voided specifically for EA use?
> 4. Do you support MT5 investor passwords (read-only view) in addition to the master password?
>
> Thanks — I want to be fully aligned with your rules before purchasing.
>
> Best regards,
> _\<Your name\>_
> _\<Your email on file\>_

#### Template B — FundedTraderMarkets (FTM)

> **Subject:** Automated / EA trading policy confirmation (pre-purchase)
>
> Hi FTM support,
>
> Before I purchase an account I need written confirmation on your algorithmic-trading policy. I operate a proprietary MetaTrader 5 EA that I own and run on my own infrastructure. Please confirm:
>
> 1. Is EA / automated / algorithmic trading permitted on (a) 1-Step Evaluation, (b) 2-Step Evaluation, (c) Instant Funding, (d) Simulated Funded phase?
> 2. Are there specific prohibited strategies — e.g., HFT / latency arbitrage, reverse martingale / grid, news trading around HIGH-impact events, scalping under N seconds, copy-trading between accounts, any EA-specific frequency / trade-count limits?
> 3. If I use an EA, pass the evaluation, and request a payout — will it be honored? Any prior case where EA use caused voided payouts?
> 4. Platform availability: is MT5 available for my region (India)?
> 5. Do you support MT5 investor password (read-only view)?
>
> Thanks for the clarity — I want to be fully compliant before committing.
>
> Best regards,
> _\<Your name\>_

**How to file:** send via their support portal (not just chat — paper trail matters). Screenshot/PDF the reply. Commit to `docs/propfirm-approvals/<name>.md` in the repo along with the date of the response. Re-verify annually — policies change.

### Spec 3 — Paperclip Deployment Agent foreign-position handler + Slack block

Additional prompt block to append to `paperclip/agents/deployment.yaml` (complements the bot-side real-time check from Diff 3 above):

```yaml
# Append to Deployment Agent prompt

  ON EVERY 5-MIN HEARTBEAT — FOREIGN-POSITION AUDIT:
    1. Get live MT5 positions via docker exec:
         docker exec trading-bot-v2 python -m src.mt5.client --list-positions
       Parse each: {ticket, magic, symbol, side, volume, entry_price, sl, tp}.

    2. Foreign = magic != 200000. Diff against paperclip.deploy_audit last
       foreign-tickets-seen set.

    3. On newly observed foreign ticket:
         a. Compose Slack block (format below) and post via webhook.
         b. Telegram DM user (plain text, short form).
         c. INSERT paperclip.deploy_audit {type: 'foreign_position',
            account_id, ticket, symbol, notified_at, auto_close_attempted: false}.

    4. Default action: ALERT ONLY. Do NOT auto-close (racing the human on
       MT5 is dangerous).

    5. If account_state.auto_close_foreign = true for this account AND
       no user acknowledgement after 60s:
         - docker exec trading-bot-v2 python -m src.mt5.client \
             --close-position <ticket> --magic 200000 --comment agent_auto_close_foreign
         - UPDATE deploy_audit SET auto_close_attempted = true, auto_close_at = NOW().

    6. Clear foreign tickets from the "seen" set only once MT5 confirms
       they are closed. Re-alert if a closed-then-reopened ticket appears.

    7. HARD RULE (also encoded at Docker mount layer): never touches
       /mnt/bot/config, /mnt/bot/src/risk, /mnt/bot/.env, or docker-compose*.yml.
```

#### Slack block format (exact JSON to POST)

```json
{
  "blocks": [
    { "type": "header",
      "text": {"type": "plain_text", "text": "🚨 FOREIGN POSITION DETECTED"}},
    { "type": "section",
      "fields": [
        {"type": "mrkdwn", "text": "*Account:* {{account_id}} ({{phase}})"},
        {"type": "mrkdwn", "text": "*Ticket:* {{ticket}}"},
        {"type": "mrkdwn", "text": "*Symbol:* {{symbol}}"},
        {"type": "mrkdwn", "text": "*Side:* {{side}}"},
        {"type": "mrkdwn", "text": "*Volume:* {{volume}} lots"},
        {"type": "mrkdwn", "text": "*Entry:* {{entry_price}}"},
        {"type": "mrkdwn", "text": "*Magic:* {{magic}} (expected 200000)"},
        {"type": "mrkdwn", "text": "*Detected:* {{detected_utc}}"}
      ]},
    { "type": "section",
      "text": {"type": "mrkdwn",
        "text": "Bot did *NOT* place this position. Possible causes:\n• Manual trade — see Phase 0 post-mortem; do not do this\n• Master password leaked — rotate MT5 master password in SSM immediately\n• Another EA running on this account — stop it"}},
    { "type": "actions",
      "elements": [
        {"type": "button",
         "text": {"type": "plain_text", "text": "Acknowledge (keep open)"},
         "value": "foreign_ack:{{ticket}}",
         "action_id": "foreign_ack"},
        {"type": "button", "style": "danger",
         "text": {"type": "plain_text", "text": "Close now"},
         "value": "foreign_close:{{ticket}}",
         "action_id": "foreign_close"}
      ]},
    { "type": "context",
      "elements": [
        {"type": "mrkdwn", "text": "noVNC: {{tailscale_novnc_url}}  •  Plan: docs/post-mortem-5k.md"}
      ]}
  ]
}
```

#### Telegram DM format (plain text, no buttons)

```
🚨 FOREIGN POSITION — {{account_id}} ({{phase}})
Ticket {{ticket}} | {{symbol}} {{side}} {{volume}} lots @ {{entry_price}}
Magic {{magic}} (expected 200000)

Bot did NOT place this. Investigate now.
Reply /close_{{ticket}} to close, /ack_{{ticket}} to keep open.
```

#### Slack interactive-button handler

Paperclip needs a tiny webhook endpoint (or use the existing `localhost:8731/events` from the CI integration) that:

- `POST /slack/interactions` receives Slack's `payload` (URL-decoded JSON).
- Routes `action_id == "foreign_ack"` → mark ticket as acknowledged (skip auto-close), log to `deploy_audit`.
- Routes `action_id == "foreign_close"` → dispatch `deployment.activate.close_position(ticket)` → MT5.
- Slack signing secret verified on every call (`X-Slack-Signature` + `X-Slack-Request-Timestamp`).

Keep the endpoint on localhost only; paperclip's Tailscale exposure means Slack can't reach it directly. Solution: **Slack Socket Mode** — paperclip opens a WebSocket to Slack, receives button clicks, no inbound HTTP needed. The `slack_bolt` Python SDK handles this in ~30 lines.

---

### Spec 4 — `docker-compose.paperclip.yml` (concrete overlay)

Extends the existing `docker-compose.ec2.yml`. Use:

```bash
docker compose \
  -f docker-compose.ec2.yml \
  -f docker-compose.paperclip.yml \
  up -d
```

```yaml
# docker-compose.paperclip.yml
# Paperclip orchestration layer for trading-bot-v2.
# Runs alongside MT5 + trading-bot on the same EC2 box.
#
# Prerequisites (must be done by bootstrap-ec2.sh + manual steps):
#   1. Claude Code CLI installed globally:  npm install -g @anthropic-ai/claude-code
#   2. Claude Code authenticated with Max 20x:  claude login  (via SSH OAuth tunnel)
#   3. Paperclip repo cloned to /home/ec2-user/paperclip (or use a published image)
#   4. Directories created:
#        mkdir -p /home/ec2-user/paperclip-data /home/ec2-user/paperclip-logs
#   5. .env at repo root contains SLACK_WEBHOOK, TELEGRAM_BOT_TOKEN,
#      TELEGRAM_USER_CHAT_ID, GH_TOKEN (PAT with workflow:write scope),
#      SLACK_APP_TOKEN + SLACK_BOT_TOKEN (for Socket Mode button handler).

services:
  paperclip:
    # Option A — build from cloned repo:
    build:
      context: /home/ec2-user/paperclip
      dockerfile: Dockerfile
    # Option B — published image (if available):
    # image: paperclipai/paperclip:latest
    container_name: paperclip
    restart: unless-stopped
    depends_on:
      - trading-bot
    environment:
      - NODE_ENV=production
      - PAPERCLIP_DB_PATH=/paperclip/data/paperclip.db        # PGlite file
      - CLAUDE_CODE_PATH=/usr/local/bin/claude
      - SLACK_WEBHOOK=${SLACK_WEBHOOK}
      - SLACK_APP_TOKEN=${SLACK_APP_TOKEN}                   # Socket Mode
      - SLACK_BOT_TOKEN=${SLACK_BOT_TOKEN}
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
      - TELEGRAM_USER_CHAT_ID=${TELEGRAM_USER_CHAT_ID}
      - GH_REPO=shivang2000/trading-bot-v2
      - GH_TOKEN=${GH_TOKEN}
      - AWS_REGION=ap-south-1
      - PYTHONUNBUFFERED=1

    ports:
      # UI — bind to localhost only. Reach via Tailscale (`tailscale serve 3000`) or SSH tunnel.
      - "127.0.0.1:3000:3000"
      # CI webhook endpoint (GHA ci.yml POSTs ci-image-built here)
      - "127.0.0.1:8731:8731"

    volumes:
      # --- Claude Code OAuth shared from host (EC2 is the primary Max login) ---
      - /home/ec2-user/.claude:/root/.claude:ro
      - /home/ec2-user/.config/claude-code:/root/.config/claude-code:ro
      - /usr/local/bin/claude:/usr/local/bin/claude:ro

      # --- RISK-10 MOUNTS: bot tree is READ-ONLY to paperclip ---
      - ./config:/mnt/bot/config:ro
      - ./src:/mnt/bot/src:ro
      - ./docker-compose.ec2.yml:/mnt/bot/docker-compose.ec2.yml:ro
      - ./docker-compose.paperclip.yml:/mnt/bot/docker-compose.paperclip.yml:ro
      - ./.env:/mnt/bot/.env:ro
      - ./scripts:/mnt/bot/scripts:ro
      - ./pyproject.toml:/mnt/bot/pyproject.toml:ro

      # --- Runtime data: paperclip reads bot's journal + logs (read-only) ---
      - ./data:/mnt/bot/data:ro
      - ./logs:/mnt/bot/logs:ro

      # --- Paperclip's own persistent state (read-write, bind-mounted for S3 rsync) ---
      - /home/ec2-user/paperclip-data:/paperclip/data
      - /home/ec2-user/paperclip-logs:/paperclip/logs

      # --- Docker socket: Deployment Agent uses `docker exec` + `docker compose`
      #     Security note: grants effective root on host. Acceptable because paperclip
      #     IS the DevOps agent and risk-10 is enforced at the bot-mounts layer above.
      #     Future hardening (P2): swap for an AWS Lambda or GHA dispatch endpoint. ---
      - /var/run/docker.sock:/var/run/docker.sock

    deploy:
      resources:
        limits:
          memory: 700M
          cpus: '1.0'
      restart_policy:
        condition: unless-stopped

    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:3000/health"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 120s

    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"

# No top-level volumes: block — paperclip-data and paperclip-logs are bind mounts
# straight to /home/ec2-user/... so the nightly S3 rsync timer picks them up directly.
```

#### Bind-mount rationale

Paperclip state lives in `/home/ec2-user/paperclip-data/` on the host (not an anonymous Docker volume) so the `tb-backup.service` systemd timer from Artifact 2 can `aws s3 sync` it nightly. Anonymous volumes would be harder to backup.

#### Risk-10 verification

Inside the paperclip container, any `claude --print` call that tries to write under `/mnt/bot/...` will fail with `EACCES`. To verify the mount after `up -d`:

```bash
docker exec paperclip /bin/sh -c 'touch /mnt/bot/config/foo.yaml' 2>&1
# Expected: touch: cannot touch '/mnt/bot/config/foo.yaml': Read-only file system

docker exec paperclip /bin/sh -c 'touch /paperclip/data/foo' 2>&1
# Expected: (silent success)
```

Add this as a startup smoke test in the Deployment Agent's first-heartbeat diagnostics.

#### Docker-socket alternative (P2 hardening)

The `/var/run/docker.sock` mount is the single biggest attack surface. When P2 work starts, consider replacing it with:

- **Option X1 — GHA dispatch for all operational commands.** Deployment Agent triggers a tightly-scoped `ops.yml` workflow on the self-hosted runner that does the exec. Removes docker.sock, adds ~2-5s latency per command.
- **Option X2 — A privileged sidecar container** (`docker-cli-proxy`) that wraps a minimal allowlist of commands (restart, logs, exec healthcheck.sh) and exposes them over a Unix socket to paperclip. Paperclip gets less than full docker access.

P1 accepts the tradeoff; revisit in P2 once the baseline is stable.

---

### Spec 5 — Slack Bolt Socket Mode handler

File path: `paperclip/slack_handler.py` (lives in the paperclip repo / container, not in trading-bot-v2).

Socket Mode = no public inbound HTTP endpoint needed. Paperclip opens a WebSocket to Slack; button clicks flow back over it. Keeps paperclip purely Tailscale-reachable.

#### `paperclip/slack_handler.py`

```python
"""Slack Socket Mode handler for paperclip foreign-position button clicks.

Receives button interactions from Slack via WebSocket (Socket Mode).
Dispatches acknowledged / close actions to paperclip's internal queue
for the Deployment Agent to pick up on its next heartbeat.

Environment:
  SLACK_APP_TOKEN    xapp-...   (Socket Mode, connections:write scope)
  SLACK_BOT_TOKEN    xoxb-...   (chat:write, chat:update, im:write)
  PAPERCLIP_DB_PATH  path to paperclip.db (PGlite / SQLite)
"""
from __future__ import annotations

import asyncio
import logging
import os
import sqlite3
from datetime import datetime, timezone

from slack_bolt.async_app import AsyncApp
from slack_bolt.adapter.socket_mode.async_handler import AsyncSocketModeHandler

logger = logging.getLogger(__name__)

PAPERCLIP_DB = os.environ.get("PAPERCLIP_DB_PATH", "/paperclip/data/paperclip.db")
BOT_TOKEN = os.environ["SLACK_BOT_TOKEN"]
APP_TOKEN = os.environ["SLACK_APP_TOKEN"]

app = AsyncApp(token=BOT_TOKEN)


def _record_audit(ticket: int, action: str, actor: str) -> None:
    """Append foreign-position action to paperclip.deploy_audit."""
    with sqlite3.connect(PAPERCLIP_DB) as conn:
        conn.execute(
            """INSERT INTO deploy_audit (type, ticket, action, actor, ts)
               VALUES (?, ?, ?, ?, ?)""",
            ("foreign_position", ticket, action, actor,
             datetime.now(timezone.utc).isoformat()),
        )
        conn.commit()


def _queue_agent_task(agent: str, op: str, payload: str) -> None:
    """Write a pending task row for the target agent to pick up."""
    with sqlite3.connect(PAPERCLIP_DB) as conn:
        conn.execute(
            """INSERT INTO agent_tasks (agent, op, payload, status, created_at)
               VALUES (?, ?, ?, 'pending', ?)""",
            (agent, op, payload, datetime.now(timezone.utc).isoformat()),
        )
        conn.commit()


def _parse_button_value(value: str) -> tuple[str, int]:
    """'foreign_ack:12345' -> ('foreign_ack', 12345)."""
    action, ticket = value.split(":", 1)
    return action, int(ticket)


@app.action("foreign_ack")
async def handle_ack(ack, body, client):
    """User chose to keep the foreign position open. Log only."""
    await ack()
    _, ticket = _parse_button_value(body["actions"][0]["value"])
    actor = body["user"]["name"]
    _record_audit(ticket, "ack", actor)

    await client.chat_update(
        channel=body["channel"]["id"],
        ts=body["message"]["ts"],
        text=f"✅ Acknowledged (kept open) by @{actor} — ticket {ticket}",
        blocks=[],
    )
    logger.info("Foreign position %s ACKed by %s", ticket, actor)


@app.action("foreign_close")
async def handle_close(ack, body, client):
    """User chose to close. Queue a task for the Deployment Agent."""
    await ack()
    _, ticket = _parse_button_value(body["actions"][0]["value"])
    actor = body["user"]["name"]
    _record_audit(ticket, "close_requested", actor)
    _queue_agent_task("deployment", "close_foreign_position", str(ticket))

    await client.chat_update(
        channel=body["channel"]["id"],
        ts=body["message"]["ts"],
        text=(f"🔴 Close requested by @{actor} — ticket {ticket}. "
              f"Deployment Agent will pick this up on the next heartbeat."),
        blocks=[],
    )
    logger.info("Foreign position %s CLOSE dispatched by %s", ticket, actor)


async def main() -> None:
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    )
    handler = AsyncSocketModeHandler(app, APP_TOKEN)
    logger.info("Slack Socket Mode handler starting (paperclip.slack_handler)")
    await handler.start_async()


if __name__ == "__main__":
    asyncio.run(main())
```

#### `requirements.txt` addition (paperclip container)

```
slack_bolt>=1.18
aiohttp>=3.9
```

#### Slack app setup checklist (one-time, ~5 min)

1. Visit **api.slack.com/apps → Create New App → From scratch**. Name it "paperclip-tradingbot". Pick your workspace.
2. **Socket Mode → Enable**. Generate an App-Level Token with scope `connections:write`. Save as `SLACK_APP_TOKEN` (starts with `xapp-`).
3. **OAuth & Permissions → Bot Token Scopes**, add:
   - `chat:write` (post messages)
   - `chat:write.customize` (use custom name/icon)
   - `chat:update` (edit existing messages)
   - `im:write` (direct-message the user)
   - `commands` (reserved for future slash commands)
4. **Interactivity & Shortcuts → Enable**. No Request URL needed (Socket Mode handles it).
5. **Install to Workspace**. Copy the Bot User OAuth Token (starts with `xoxb-`) into `SLACK_BOT_TOKEN` in `.env`.
6. Invite the bot to your alerts channel: `/invite @paperclip-tradingbot`.

#### Paperclip schema extension (append to `paperclip/init.sql`)

```sql
CREATE TABLE IF NOT EXISTS agent_tasks (
  id          SERIAL PRIMARY KEY,
  agent       TEXT NOT NULL,              -- 'deployment', 'management', etc.
  op          TEXT NOT NULL,              -- 'close_foreign_position', 'activate', ...
  payload     TEXT,                        -- op-specific (e.g., ticket number)
  status      TEXT NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending','running','done','failed')),
  result      TEXT,
  created_at  TIMESTAMPTZ NOT NULL,
  started_at  TIMESTAMPTZ,
  finished_at TIMESTAMPTZ
);
CREATE INDEX idx_agent_tasks_pending ON agent_tasks(agent, status)
  WHERE status = 'pending';

-- Add missing columns to deploy_audit for this flow
ALTER TABLE deploy_audit ADD COLUMN IF NOT EXISTS ticket INT;
ALTER TABLE deploy_audit ADD COLUMN IF NOT EXISTS action TEXT;
ALTER TABLE deploy_audit ADD COLUMN IF NOT EXISTS actor  TEXT;
```

#### Deployment Agent prompt — pick up queued tasks

Append to `paperclip/agents/deployment.yaml` prompt:

```yaml
  ON EVERY 5-MIN HEARTBEAT — DRAIN PENDING AGENT TASKS:
    1. SELECT id, op, payload FROM agent_tasks
       WHERE agent='deployment' AND status='pending'
       ORDER BY created_at LIMIT 10;
    2. For each row:
       UPDATE agent_tasks SET status='running', started_at=NOW() WHERE id=<id>;
       Execute the op:
         - close_foreign_position: docker exec trading-bot python -m src.mt5.client \
             --close-position <payload> --comment "slack_user_close_foreign"
         - activate: dispatch the account-switchover sequence from the main prompt
       On success: UPDATE status='done', finished_at=NOW(), result='ok'.
       On failure: UPDATE status='failed', finished_at=NOW(), result=<error>.
    3. Report outcomes to Slack (thread on the original alert message if possible).
```

#### Telegram-side mirror

For parity (if you prefer Telegram over Slack), add to `src/telegram/notifier.py` a `python-telegram-bot` command handler:

```python
# Pseudocode sketch — real impl in the bot repo during implementation
@telegram_app.command_handler("close")
async def cmd_close(update, context):
    ticket = int(context.args[0])       # /close 12345
    paperclip_queue_task("deployment", "close_foreign_position", str(ticket))
    await update.message.reply_text(f"Close queued for ticket {ticket}.")

@telegram_app.command_handler("ack")
async def cmd_ack(update, context):
    ticket = int(context.args[0])
    paperclip_record_audit(ticket, "ack", update.effective_user.username)
    await update.message.reply_text(f"Acknowledged ticket {ticket}.")
```

Run paperclip's slack_handler.py alongside the Node.js paperclip main process — in the same container, with a small supervisor (e.g., `tini` as PID 1 + a shell script that launches both, or two separate `ExecStart` lines in a systemd drop-in inside the container). Simplest: add to the container's entrypoint script:

```bash
#!/bin/sh
# /entrypoint.sh
python3 /app/paperclip/slack_handler.py &
exec node /app/paperclip/index.js
```

---

### Spec 6 — `docs/NEXT_CHALLENGE_SAFETY_GATE.md` (pre-purchase checklist)

One page. Read top-to-bottom before clicking "Buy" on any prop-firm challenge purchase. Every box must be ticked. Copy this file into the repo; sign + date at the bottom per purchase.

```markdown
# Next-challenge safety gate

**Purchased:** _(date)_  **Account ID:** _(e.g., fp-5k-step1-v2)_  **Firm / size / variant:** _(e.g., FundingPips / $5k / 2-step)_

## 1. Code — P1 diffs merged on `main`
- [ ] Diff 1 — RiskManager central news gate (`src/risk/manager.py`)
- [ ] Diff 2 — NewsEventFilter `time_until_next_event` (`src/analysis/news_filter.py`)
- [ ] Diff 3 — PositionMonitor pre-news FLAT + foreign-position monitor (`src/monitoring/position_monitor.py`)
- [ ] Diff 4 — `ForeignPositionEvent` event type (`src/core/events.py`)
- [ ] Diff 5 — main.py wiring (shared `news_filter`, subscribers for `ForeignPositionEvent`)
- [ ] Diff 6 — `pre_news_flat_minutes` config key (`src/config/schema.py`)
- [ ] Diff 7 — expanded `config/news_calendar.csv` (ECB / BoE / PPI / Retail Sales added)

## 2. Tests — green in CI
- [ ] `tests/unit/test_news_filter.py` — boundary windows, naive TS, next-event, CSV-missing
- [ ] `tests/unit/test_news_filter_enabled_flag.py` — honors `news_filter_enabled=false`
- [ ] `tests/integration/test_risk_manager_news.py` — SIGNAL 5 min pre-NFP → `news_window` rejection
- [ ] `tests/integration/test_position_monitor_pre_news.py` — mocked open position + event 3 min out → close orders emitted
- [ ] `tests/integration/test_position_monitor_foreign.py` — wrong-magic position → `ForeignPositionEvent` published exactly once (not re-published on subsequent polls)
- [ ] `ruff check` + `mypy` green on `src/`
- [ ] Coverage ≥ 60% on `src/risk`, `src/monitoring`, `src/analysis`

## 3. Paper replay
- [ ] `python scripts/backtest_unified.py --symbol XAUUSD --prop-firm --account-size <size> --phase <step1|step2> --replay data/replay/one_week.csv` runs clean
- [ ] ≥ 100 paper trades in the replay window
- [ ] Sharpe ≥ 1.0 on replay
- [ ] Max DD ≤ firm rulebook `max_drawdown_pct` (5% for FundingPips)
- [ ] Zero PropFirmGuard violations in replay logs
- [ ] News filter blocks ≥ 2 high-impact events during the replay window (sanity)

## 4. EC2 + paperclip infrastructure
- [ ] `t3.large` running in `ap-south-1` with instance profile attached
- [ ] 50 GB gp3 volume, `DeleteOnTermination=false` verified (`aws ec2 describe-volumes`)
- [ ] `--disable-api-termination` set on the instance
- [ ] Public SSH rule removed from SG (Tailscale-only access confirmed)
- [ ] `docker compose -f docker-compose.ec2.yml -f docker-compose.paperclip.yml up -d` clean
- [ ] Paperclip UI reachable over Tailscale (`http://<ts-ip>:3000`)
- [ ] `docker exec paperclip touch /mnt/bot/config/x.yaml` → `EACCES` (risk-10 mount policy verified)

## 5. SSM parameters populated (new account)
- [ ] `/trading-bot/<account_id>/mt5_login` — SecureString
- [ ] `/trading-bot/<account_id>/mt5_master_password` — SecureString
- [ ] `/trading-bot/<account_id>/mt5_investor_password` — SecureString (your personal use only)
- [ ] `/trading-bot/<account_id>/mt5_server` — String
- [ ] `aws ssm get-parameter --name /trading-bot/<id>/mt5_login --with-decryption` returns the value from EC2's instance role

## 6. MT5 investor password — physical manual-trading prevention
- [ ] First-time MT5 login on EC2 noVNC used the **master** password (bot needs this)
- [ ] MT5 terminal confirms both passwords distinct
- [ ] `.env` on EC2 contains only `MT5_PASSWORD=<master>`
- [ ] Your laptop / phone MT5 logs in with **investor** password only
- [ ] Self-test: from laptop MT5 with investor login, attempt a trade → MT5 rejects ("Invalid account") — this is the whole point

## 7. Paperclip agent 72-hour dry-run
- [ ] All 4 agents running uninterrupted for 72+ hours
- [ ] Zero crashes in `/home/ec2-user/paperclip-logs/`
- [ ] Log Reporter Slack pulses arrived at the configured cadence (skipped on weekend)
- [ ] CEO Agent's nightly digest arrived at 21:30 UTC on each of the 3 days
- [ ] Chaos drill executed: `docker compose stop metatrader5` → Deployment Agent Slack alert within 5 min → `docker compose start` → bot recovered

## 8. Claude Max quota headroom
- [ ] `logs/claude-lock.log` shows one line per `claude --print` invocation
- [ ] 48 h invocation count × per-call token estimate ≤ 70 % of Max 20x daily quota
- [ ] No `429` / rate-limit errors in the window
- [ ] Personal CC usage on laptop paused (or scheduled) to avoid overlap during live trading hours

## 9. Foreign-position monitor — end-to-end test on DEMO only
- [ ] Manually place a test trade on the MT5 DEMO account (not the funded one!) via noVNC
- [ ] Within seconds: bot-side Slack alert fires (from `position_monitor._check_foreign_positions`)
- [ ] Within 5 min: Paperclip Deployment Agent independent alert fires
- [ ] Slack "Close now" button → `docker exec ... --close-position <ticket>` → position closes on MT5
- [ ] Audit row written to `paperclip.deploy_audit`

## 10. News filter — end-to-end test
- [ ] Pick a future event from `config/news_calendar.csv` within 24 h
- [ ] Unit-test harness: fake-time to 5 min pre-event → RiskManager rejects signal with `news_window` reason
- [ ] Pre-news FLAT: mock open position, advance time to 4 min pre-event → bot emits close orders
- [ ] Production path: leave bot running through one real event, confirm no new trades fire ±15 / +30 min

## 11. Prop-firm bot-allowed confirmation on file
- [ ] Written response from the firm saved in `docs/propfirm-approvals/<firm>.md`
- [ ] Response explicitly permits EA / automated trading on this variant + phase
- [ ] Response explicitly confirms payouts honored for EA-originated profits
- [ ] Response dated within last 12 months

## 12. Backup durability
- [ ] `sudo systemctl list-timers | grep tb-backup` shows the timer active
- [ ] Latest nightly S3 sync visible: `aws s3 ls s3://trading-bot-v2-backups/tb/data/ --recursive | tail -5`
- [ ] Restore drill: download a random file from S3, diff against the live copy on EC2 — equal
- [ ] Hourly SQLite-WAL incremental to S3 confirmed (if implemented — see Data-durability section)

## 13. Final sign-off

All 12 boxes green? **Yes → proceed to purchase. No → fix the reds; do not click "Buy."**

- Completed by: _(initials)_
- Date: _(YYYY-MM-DD UTC)_
- Challenge purchase receipt: _(paste from firm's email)_
- `account_id` status flipped to `active` in `paperclip.account_state`: _(confirm after Deployment Agent activates)_
- Post-activation smoke: noVNC login successful, first heartbeat cycle clean, first Slack pulse arrived
```

This file lives in the repo. One copy per purchase, committed to `docs/next-challenge-gates/<YYYY-MM-DD>-<account_id>.md`. That creates a permanent audit trail — if a future account busts, we'll know exactly which gate failed (or wasn't exercised).
