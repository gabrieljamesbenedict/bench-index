# bench-index — Project Plan

> Simple web app for comparing publicly available AI model benchmarks.
> App name: **BenchIndex** · Repo/folder name: **bench-index**
> Status: planning complete, pre-scaffold. `ModelBench` was rejected (taken by MLCommons).

## 1. Goal

A bookmarkable reference tool that answers: "which model is best for *my* use case?"
It aggregates public benchmark scores, plus pricing / speed / context metadata,
normalizes model identities across sources, and presents sortable tables and
quality-vs-cost charts. No single upstream API does this today — the app's value
is the normalization layer.

## 2. Naming (decided)

- App: **BenchIndex** — short, professional, "index of benchmarks".
- Repo/folder: `bench-index`.
- Rejected: `ModelBench` (taken by MLCommons safety-benchmark project),
  `BenchArena`, `ModelLens`, `StackRank`, `ModelArena` (crowded / less precise).

## 3. Data sources (researched, free)

There is **no single unified API**. The app polls several sources and joins them.
CORS blocks most of these from browser JS — hence the server-side sync pipeline.

| Source | Access | Data | Limits / cost | Notes |
|---|---|---|---|---|
| LMArena (anchor) | HF dataset `lmarena-ai/leaderboard-dataset` via datasets-server API | Elo, CI, rank, votes; subsets text/vision/webdev/search/t2i; `latest` + `full` (history) splits | Free, no key, ~weekly/daily refresh | Attribution expected |
| Artificial Analysis | `GET /data/llms/models`, `x-api-key` header | Intelligence Index, per-eval scores, tok/s, latency, pricing | Free tier ~100 req/day, key required; no redistribution on free tier (fine for personal use, check before public launch) | Only source with speed/cost; 1 req/day is plenty |
| SWE-bench | `swe-bench.github.io` repo `data/leaderboards.json` (raw GitHub URL) | Coding-agent % resolved per (model + scaffold) | Free, no key | Scores belong to model+scaffold pairs — keep a `variant` field |
| OpenRouter | `GET openrouter.ai/api/v1/models`, no auth | Pricing, context window, modality — metadata backbone | Free, no key | No scores, metadata only |
| Hugging Face Hub | Hub API + per-benchmark `.eval_results` leaderboards (48+ official) | Community eval scores (GPQA, MMLU-Pro, …) | Free | Messy names, fuzzy matching needed |
| BenchLM | Public API/MCP | Aggregated rankings, 600+ benchmarks | Free ~250 reads/mo | Aggregator with own judgment calls |
| LiveBench | Site insights JSON / GitHub | Contamination-free scores, 7 categories, cost/task | Free | No stable documented API (scrape-ish) |
| Epoch AI | Public leaderboard data | FrontierMath / hard-math scores | Free | Small model coverage |

Key pipeline consequences: caching/periodic sync is mandatory (rate limits +
freshness); model-name normalization is the real engineering work, not the UI.

## 4. Tech stack (decided: static-first)

This app is ~80% data pipeline, ~20% UI. Optimize for a free, reliable pipeline.

- **Frontend:** React + TypeScript + Vite, Tailwind CSS.
  - TanStack Table (sortable/filterable grids), Recharts (comparisons,
    quality-vs-cost scatter), cmdk (⌘K model search).
- **Data layer:** pre-fetched JSON snapshots. No backend, no DB, no CORS issues.
  - Sync script in TypeScript (`tsx`) pulls sources, normalizes, writes
    versioned JSON to `data/` committed to the repo.
  - GitHub Actions cron (daily) runs sync, commits on change, triggers deploy.
- **Deploy:** Cloudflare Pages (static frontend, free, unlimited bandwidth) or
  GitHub Pages / Vercel. Recommended: **all-Cloudflare** (see §7).
- **Stretch:** one serverless function for on-demand refresh + proxying the
  AA key (must not ship in client-side code for a public repo).
- **Alt (rejected for now):** Next.js + Postgres (Neon/Supabase) + cron routes.
  More power (accounts, per-request filtering) but inherits backend maintenance.
  Only if user accounts are planned.
- **Why not Python:** wrangling is light (JSON/CSV parse, name-map, join);
  one language (TS) end-to-end. HF datasets API serves JSON directly.

### Repo layout (target)

```text
bench-index/
├─ apps/web/            # Vite + React UI
├─ pipeline/            # sync scripts, one file per source
│  ├─ sources/lmarena.ts
│  ├─ sources/artificial-analysis.ts
│  ├─ sources/swebench.ts
│  ├─ sources/openrouter.ts
│  ├─ model-aliases.json
│  └─ sync.ts
├─ worker/              # Cloudflare Worker: /sync + /snapshot + cron (Always Current)
├─ data/                # generated JSON snapshots (committed seed/fallback)
├─ plans/project-plan.md
└─ .github/workflows/sync.yml
```

## 5. Data pipeline (detail)

### 5.1 Big picture

```text
SOURCES (fetch) → STAGE 1 normalize → STAGE 2 join & de-dupe → STAGE 3 emit snapshots
       ↑                                                                     ↓
 GitHub Actions cron / Worker cron                                    data/*.json committed
                                                                             ↓
                                                                   static site consumes
```

Principles:

1. Pipeline is a **pure function**: raw in → snapshot JSON out. Same inputs →
   byte-identical output. Debuggable, diffable in git.
2. **Sources fail independently.** One dead API never blocks the other four.
3. **Every number carries provenance** (source + fetchedAt) so the UI can cite it.

### 5.2 Stage 0 — Fetch (one module per source)

Common interface: `fetch(): Promise<RawData>`, called in a loop with
per-source try/catch, retries with backoff, conditional requests
(`ETag` / `If-Modified-Since`), disk/KV cache for dev re-runs.

- **lmarena.ts** — anchor. HF datasets-server API, subsets (text, vision,
  webdev, …) × splits (`latest` = current board, `full` = history for
  Elo-over-time charts). Row: model_name, organization, license, score,
  score_ci_lower/upper, observation_count, rank, category,
  leaderboard_publish_date. No key.
- **artificial-analysis.ts** — `GET /data/llms/models` + key from env.
  Intelligence Index, eval scores, tok/s, latency, pricing (powers cost
  scatter). 1 req/day. Missing key → clean skip, snapshot just lacks AA fields.
- **swebench.ts** — raw `leaderboards.json`, parse `Verified` board
  (model, agent/scaffold, % resolved, date).
- **openrouter.ts** — `GET /api/v1/models`, no auth. Canonical pricing,
  context, modality, release date. Joined onto every model card.
- Later: `hf-benchmarks.ts`, `epoch.ts` — adding a source = one new file +
  one registration in `sync.ts`.

### 5.3 Stage 1 — Normalize (the hard part)

Every source maps to one row shape:

```ts
type MetricRow = {
  modelId: string;       // canonical, e.g. "anthropic/claude-sonnet-4.5"
  benchmarkId: string;   // e.g. "lmarena.text", "swebench.verified", "aa.intelligence"
  score: number;
  scoreNorm: number;     // 0–1 for cross-benchmark charts
  scoreDisplay: string;  // "68.2%" | "1421 Elo"
  ci?: [number, number];
  variant?: string;      // scaffold, e.g. "live-SWE-agent"
  sampleSize?: number;   // votes / tasks — trust indicator
  source: "lmarena" | "artificial-analysis" | ...;
  fetchedAt: string;     // ISO
};
```

**Model identity resolution.** Same model arrives as `Claude Sonnet 4.5`
(LMArena), `Claude 4.5 Sonnet (Reasoning)` (AA), `Claude 4.5 Sonnet`
(SWE-bench), `anthropic/claude-sonnet-4.5` (OpenRouter). Solution:
committed `pipeline/model-aliases.json` (raw string → canonical ID) +
canonical registry `data/models.json`, with matching strategy:

1. Exact alias hit (~90% after curation).
2. Normalization fallback: lowercase, strip org/date/suffixes → fuzzy match.
3. Unmatched → quarantine bucket (`data/unmatched.json`). **Never guess
   silently.** New models get one alias line; next sync they join. Log
   similarity suggestions to make curation copy-paste.

### 5.4 Stage 2 — Join & de-dupe

Deterministic merge on canonical `modelId`:

- One metric per (model, benchmark, source); explicit preference when a source
  publishes duplicates (e.g. LMArena raw vs style-controlled), recorded in
  the benchmark definition.
- Newest wins within a source (`leaderboard_publish_date` / `fetchedAt`);
  retain history in `history/` for trend lines.
- **Never average across sources.** Same-named scores from different eval runs
  stay separate rows; UI shows both and flags disagreement (signal, not noise).

### 5.5 Stage 3 — Emit snapshots

Plain JSON in `data/`, committed:

```text
data/
├─ models.json       # canonical registry (id, name, org, license, pricing, context)
├─ benchmarks.json   # definitions (id, display name, source, scale, description, URL)
├─ latest.json       # ALL current rows — the single file the web app fetches
├─ history/          # optional {model}-{benchmark}.json series from `full` splits
├─ unmatched.json    # quarantine for review
└─ meta.json         # per-source status: last success, row counts, errors
```

- `latest.json` is the **contract**: UI imports only it (+ models/benchmarks).
- **Git history is the database**: every sync commit diffs scores; `git log -p`
  shows when anything moved. Zero-infrastructure time series.

### 5.6 Orchestration — sync.ts + GitHub Actions

`pipeline/sync.ts`: for each source → fetch → normalize → collect rows+errors
→ merge → emit. Exit 0 on partial failure (partial data > no deploy); record
failures in `meta.json`. Site footer reads meta ("LMArena as of Oct 14 —
refresh pending"); site never goes blank (renders last committed snapshot).

`.github/workflows/sync.yml`: cron daily ~06:00 UTC → checkout → `npm ci` →
`npm run sync` (AA key from repo secrets) → commit `data/` **iff changed**
→ push → deploy workflow rebuilds. Staleness ≤24h, zero manual work.

**No empty commits:** `git diff --quiet data/` gates the commit. Per-metric
`fetchedAt` only updates when value/sample changes, so timestamps don't force
daily diffs. "Last sync" lives in `meta.json`, rewritten only on content change.

**Failure visibility:** "no commit" = healthy quiet day OR dead pipeline.
Distinguish via `meta.json` errors (surfaced in UI footer) + GitHub's native
failed-run emails. Also note: GitHub disables scheduled workflows after ~60
days of zero repo activity — re-run manually (`workflow_dispatch`) or touch
the repo if quiet that long.

### 5.7 Why static beats a backend (here)

No CORS fights, no server/DB bills, static bundle. Rate limits vanish
(AA: 1 call/day, not 1/visitor). Bad sync = `git revert`. Trade-off: ≤24h
staleness — invisible for a benchmark browser. Escape hatch: one serverless
function for "refresh now" running the same pipeline.

## 6. Sync cadence

|              | Daily | Hourly |
|---|---|---|
| Runs/mo      | ~30   | ~720   |
| Commits/mo   | ~8–15 (change-triggered) | same volume, less batched |
| Actions cost | trivial | trivial (public = unlimited; private ~1–2 min/run) |
| AA usage     | 1/day | 24/day (limit ~100/day — fits) |
| Freshness    | ≤24h  | ≤1h    |

Realistic hourly volume is **1–2 commits/day on busy weeks, zero on quiet
days** — benchmark data doesn't change 24×/day. Costs of hourly: chattier
`git log` (mitigate: commit per source), and "synced N min ago" must come
from the run layer (Actions/deploy timestamp), not committed data. If the goal
is freshness-as-a-feature rather than fresher commits, see §7 (fetch hourly,
serve from cache, commit nothing).

## 7. Always Current (optional upgrade)

Decouple freshness from git: freshness lives in a cache, repo stays boring.

```text
Scheduler (Worker cron / Actions curl / cron-job.org)
  → POST /sync (serverless, Bearer SYNC_TOKEN)
  → runs pipeline in memory → writes KV: snapshot:latest + snapshot:<ver> + sync:status
Static site: renders bundled seed instantly → fetch GET /snapshot (Cache-Control: 300)
  → swaps fresher rows → footer "live · synced 12 min ago"
```

- **Writer `/sync`:** same pipeline modules, disk cache → KV-backed cache,
  idempotent, conditional requests. Rolling versioned keys (~4) for rollback.
- **Store:** edge KV, one key is the contract (`snapshot:latest`). ~300 models
  × ~10 rows ≈ <1 MB. Cold KV → reader falls back to bundled seed snapshot.
- **Reader `/snapshot`:** 5 lines — read KV, return JSON. CDN absorbs traffic
  (~12 KV reads/hour even at 10k visitors/hour).
- **Frontend:** stale-while-revalidate — instant seed paint, then swap on fetch;
  optional re-fetch on interval/focus.
- **Failures:** dead source → last-good + `stale:true` badge; dead sync → KV
  untouched, `sync:status.lastError` recorded (UI can show "failing since
  15:00"); dead KV → seed snapshot; dead scheduler → `syncedAt` visibly freezes
  (product itself is the monitor); new model → in review queue within an hour.
- **Unchanged from static plan:** sources, normalization, aliases, join,
  schema, UI (~90% shared). Changed: sink (KV), `/snapshot` endpoint,
  sync-token, ~20-line frontend fetch. `data/*.json` becomes seed-only.
- **Trade-offs:** runtime dependency (function + KV, mitigated by seed
  fallback); metering moves to $0-with-eyes-open (free tiers cover this many
  times over); 3 new things to configure (function, KV, token).

## 8. Hosting (decided direction: all-Cloudflare)

One Worker exposes both routes + owns the cron:

```text
Worker: bench-index-api (Cloudflare free: 100k req/day)
├─ POST /sync      → pipeline → KV        [cron-triggered]
├─ GET  /snapshot  → KV → JSON            [site-triggered]
└─ triggers: cron "7 * * * *"            [native, reliable, hourly OK]
```

Static frontend on **Cloudflare Pages** (free, unlimited bandwidth) + Worker +
KV namespace = one free account, ~$0/mo, no card to start, no egress fees.

**How GitHub cron calls the function** (if used instead of / alongside native
cron): scheduled job spins an ephemeral Ubuntu VM that `curl`s the public
endpoint — the function just sees an HTTPS POST. Auth via
`Authorization: Bearer ${{ secrets.SYNC_TOKEN }}`; `--fail-with-body` so HTTP
errors fail the run visibly. GitHub cron is best-effort (5–30 min delays,
occasional skips; UTC; default-branch only; disabled after ~60 idle days) —
fine for hourly freshness, or swap in cron-job.org / UptimeRobot as a
"curl-as-a-service" timer. Endpoint never changes, only the pinger does.

**Free KV options (workload ≈ 720 writes + ~5k reads/mo, ≤4 MB):**

| Host | Free tier | Fit |
|---|---|---|
| Cloudflare KV + Workers | 1k writes/day, 100k reads/day, 1 GB; unlimited cron schedules; no card | Best fit — one account for all three pieces |
| Upstash Redis | 10k cmds/day, 256 MB, HTTP API, no card | Great if frontend lands on Vercel/Netlify (host-agnostic) |
| Vercel Edge Config | Huge reads, tight writes | Marginal — wrong tool for recurring writes |
| Supabase Postgres | Free project, ~500 MB | Works (one table) but a DB, not KV — overkill |
| Deno Deploy KV | Generous, KV included, native cron | Solid alt if you like Deno |
| Cloudflare R2 | 10 GB, free egress | Works as object store; KV is the more natural primitive |
| Self-host (Pi/PC + tunnel) | Truly free | Your uptime is the SLA — not recommended |

Anti-recommendation: GitHub-as-KV (hourly commits to a branch) — defeats the
purpose. Coherent stacks: **all-Cloudflare** (recommended) or
Vercel + Upstash + Actions-as-timer.

## 9. Build order (next)

1. Scaffold repo layout (§4) + seed `data/` + Actions daily sync file.
2. Implement LMArena source → normalize → `latest.json` → minimal table UI
   (real data in browser day one).
3. Add OpenRouter metadata join → model cards complete.
4. Add AA (key-gated skip) + SWE-bench (variant field) sources.
5. Polish UI (filters, scatter, ⌘K, provenance footer, stale badges).
6. Always-Current upgrade: Worker + KV + cron; `wrangler deploy`. Interactive
   steps for owner: Cloudflare account + `wrangler login` (cannot be automated).
