# Productivity, ROI and Hypervelocity Dashboard

Pipeline to measure the impact of **GitHub Copilot** when the code **stays on Bitbucket**
(no migration) and work is tracked in **Jira**. Collect → Elasticsearch → Grafana.

## Why a custom pipeline?

The native GitHub dashboard (impact dashboard) and PR metrics (`merged`, `time-to-merge`)
**only exist for GitHub repositories**. With code on Bitbucket, Copilot runs in the IDE
(completions, chat, agent mode) and the "output" (PRs, cycle time) must be assembled from
Bitbucket + Jira. This project does exactly that, keeps history indefinitely (the GitHub
dashboard only keeps 28 days) and compares against a **pre-rollout baseline**.

## Stack

| Component | Detail |
|---|---|
| Elasticsearch | 8.13 (docker-compose), templates + ILM (400 day retention) |
| Grafana | 11.1, auto-provisioned 5 dashboards |
| Collectors | Python 3.10+ (`requests` + `elasticsearch`), one collector per ES datasource |

## Collectors (one Python collector per Elasticsearch datasource)

`src/copilot_hv/collectors/` — each datasource the dashboards query has one
collector with a `collect()` for the real API and is also populated by the
dummy collector:

| Grafana datasource | Collector module | Real source | Index (prefix) |
|---|---|---|---|
| `es_finance` | `finance_expense.py` | `data/finance_products.csv` (+ optional `data/finance_actuals.csv`) | `copilot_hv.finance.expense` |
| `es_adoption` | `copilot_adoption.py` | GitHub Copilot usage metrics | `copilot_hv.copilot.adoption` |
| `es_adoption_team` | `copilot_adoption_team.py` | " (rolled up by team) | `copilot_hv.copilot.adoption_team` |
| `es_aggregate` | `copilot_aggregate.py` | " (org/day totals) | `copilot_hv.copilot.aggregate` |
| `es_pr` | `bitbucket_pr.py` | Bitbucket PRs (+ diffstat) | `copilot_hv.bitbucket.pr` |
| `es_review` | `bitbucket_review.py` | Bitbucket PR activities | `copilot_hv.bitbucket.review` |
| `es_commit` | `bitbucket_commit.py` | Bitbucket PR commits | `copilot_hv.bitbucket.commit` |
| `es_jira` | `jira_issue.py` | Jira Cloud search | `copilot_hv.jira.issue` |
| `es_payroll` | `payroll.py` | `data/payroll.csv` | `copilot_hv.payroll` |

**Dummy collector** (`dummy.py`) is the only "collector" with no real API: it
seeds *every* datasource above with synthetic scenarios so the dashboards work
before any credentials or API access exist (`seed-demo` / `seed-scenarios`).

Real collectors stamp `scenario: real` (**Production**) by default; switch to
`COPILOT_HV_SCENARIO=good|bad` in `.env` only to test filtering.

## Setup

1. Dependencies:
   ```bash
   pip install -e .
   ```

2. Infrastructure:
   ```bash
   docker compose up -d          # Elasticsearch + Grafana
   python -m copilot_hv.cli setup
   ```

3. Configuration (copy and fill in `.env`):
   ```bash
   Copy-Item .env.example .env
   ```
- `GH_TOKEN`: fine-grained token (scope **View Enterprise Copilot Metrics**) or classic PAT
      `read:enterprise` / `manage_billing:copilot`.
    - `GH_ORGS`: comma separated organizations to collect.
    - `BITBUCKET_TOKEN`: app password / OAuth token (read-only).
    - `BITBUCKET_WORKSPACE` and optional `BITBUCKET_REPOS` (slug list; otherwise discovered).
    - `JIRA_URL/JIRA_EMAIL/JIRA_TOKEN/JIRA_PROJECT_KEYS`: Jira Cloud API token + project keys.
    - `PAYROLL_CSV` / `FINANCE_PRODUCTS_CSV`: local files for the payroll and finance
      collectors (`data/payroll.csv` and `data/finance_products.csv` are pre-filled).
    - Enable the **"Copilot usage metrics" = Enabled** policy at the enterprise/org.

4. Identity map (first time):
   ```bash
   python -m copilot_hv.cli init-identity   # creates data/identity/identity.csv
   # fill in: email, github_login, bitbucket_uuid, bitbucket_nickname, jira_account_id, jira_display_name
   python -m copilot_hv.cli import-identity
   ```
   > `bitbucket_uuid` is in the profile URL (`https://bitbucket.org/{workspace}/{user}/` →
   > `{uuid-...}`). `jira_account_id` comes from the Jira user API.

## Rollout runbook (recommended order)

### Phase 1 — Pre-Copilot baseline (DO BEFORE granting licenses)
Captures the Bitbucket + Jira history that becomes the "before" for comparisons.
```bash
python -m copilot_hv.cli baseline --days-back 180 [--dry-run]
```
> Without this there is no counterfactual. Record the **rollout date** (e.g., as a panel/annotation
> in Grafana) so before/after comparisons are possible. `--dry-run` previews without indexing.

### Phase 2 — Continuous collection (schedule daily: cron / Task Scheduler / GitHub Actions)
```bash
python -m copilot_hv.cli collect --copilot --bitbucket --jira --payroll --finance
```
- Flags select groups of collectors; `--all` runs every datasource. Run with no
  group flags to get all of `--copilot --bitbucket --jira`.
- `--days-back` overrides the backfill window (default from `BASELINE_DAYS`).
- `--dry-run` prints what *would* be collected without indexing anything.
- Missing credentials (e.g. `gh_token`) make a collector print the variable to
  set in `.env` and exit — exactly what the dummy collector avoids.

### Phase 3 — Dashboards (http://localhost:3000 · admin/copilot)

Every dashboard has a **Scenario** filter (`All`, `Good (high ROI)`, `Bad (low ROI)`, `Production (real)`).

| Dashboard | Question it answers |
|---|---|
| `01 — Executive / ROI` | Budget vs actual AI investment, top spend products, LLM cost by provider, merged PRs, time-to-merge, ROI |
| `02 — Adoption & Engagement` | Active users, acceptance, LoC, interactions, per-dev usage |
| `03 — Productivity & Velocity` | Throughput, story points/sprint, cycle time |
| `04 — Quality & Stability` | Rework, declines, review approval, PR size |
| `05 — Hypervelocity (Agents)` | Agent-first devs, agent sessions/edits, phase progression |

### Phase 4 — Preview without real data
```bash
python -m copilot_hv.cli seed-demo --purge
```
Generates ~6 months of realistic data (25 devs, 5 teams, post-rollout productivity ramp,
daily Copilot usage, Bitbucket PRs/reviews/commits, Jira issues, payroll and AI platform
expenses) to validate the dashboards before integration. `seed-demo --scenario bad` seeds
the underperforming storyline instead.

### Synthetic scenarios for ROI storytelling
```bash
python -m copilot_hv.cli seed-scenarios --purge
```
Seeds **two** full datasets (the dummy collector) tagged with `scenario: good` (high ROI:
high adoption, ~30% acceptance, +35% PR lift, low rework, growing agents, spend at/under
budget) and `scenario: bad` (cost without payoff: partial licensing, ~22% active, ~10%
acceptance, no throughput lift, slow weak agents, rework climbing, Products + LLM spend
overrunning). Use the **Scenario** filter in each dashboard to compare the storylines.
> The Scenario filter reads the same `scenario` field real collectors stamp, so the exact
> same panels work with synthetic or production data.

## Metrics and calibration benchmarks

- **Adoption (inputs)**: active users, acceptance rate (target 26–30%), accepted LoC,
  chat/agent interactions, feature engagement, agent sessions.
- **Productivity (outputs)**: merged PRs per dev/month, time-to-merge, cycle time, story points,
  deployment frequency, % AI-assisted work.
- **Quality (guardrails)**: post-review rework (PRCR proxy), declines/superseded, review
  approval, average PR size (detects "slicing").
- **ROI**: cost/dev/month (license + AI credits), cost as % of investments, cost per PR,
  `ROI/hours = (time saved × $_per_hour) − Copilot cost`. The finance datasource budgets come
  from `data/finance_products.csv`; add `data/finance_actuals.csv` (`product,day,amount_usd,tokens_m`)
  to overlay real spend and get utilization %.
- **Hypervelocity**: phase progression *code-first → agent-first → multi-agent*.

Benchmarks to calibrate targets:
- ~**+40%** PRs/dev in high-intensity weeks (GitHub×arXiv, dose-response, 2026).
- Phase 1→2 **+78%** PRs; 1→3 **+151%** (GitHub impact dashboard).
- **+5.4%** productivity (GitHub × BlueOptima, ECI).

## Known limitations

- "AI-assisted PR" attribution is **per user** (developer with a license), not per PR created
  by Copilot — the Copilot *cloud* agent requires GitHub repositories.
- Agents/CLI on Bitbucket: IDE/CLI telemetry still counts in Copilot adoption metrics.
- The Copilot `repos-1-day` report depends on GitHub repos and is not used here.