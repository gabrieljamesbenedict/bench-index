# BenchIndex

Simple web app for comparing publicly available AI model benchmarks.
No single upstream API aggregates everything — BenchIndex polls several free
sources, normalizes model identities, and serves one clean leaderboard.

- **App:** BenchIndex · **Repo:** `bench-index`
- **Plan:** [`plans/project-plan.md`](plans/project-plan.md) (full design: sources, pipeline, hosting)

## Architecture (short version)

- Static-first: React + TS + Vite frontend reads committed JSON snapshots in `data/`.
- `pipeline/` (one module per source) fetches → normalizes → joins → emits `data/latest.json`.
- GitHub Actions cron runs the sync daily; commits only when data changed.
- Optional upgrade: Cloudflare Worker + KV serves hourly-fresh snapshots (`/sync`, `/snapshot`) with zero commits. See plan §7–§8.

## Data sources

LMArena (anchor) · Artificial Analysis · SWE-bench · OpenRouter (metadata) ·
Hugging Face Hub · BenchLM · LiveBench · Epoch AI. All free tiers. Details in plan §3.

## Quickstart (once scaffolded)

```bash
npm install
npm run sync     # fetch sources → data/*.json (needs AA_API_KEY in .env for AA source; skips cleanly without it)
npm run dev      # run web UI
```

## Repo layout (target)

```text
bench-index/
├─ apps/web/     # Vite + React UI
├─ pipeline/     # sync scripts + model-aliases.json
├─ worker/       # Cloudflare Worker (/sync, /snapshot) + cron
├─ data/         # generated snapshots (committed seed/fallback)
├─ plans/        # this plan
└─ .github/workflows/sync.yml
```

## Notes

- GitHub disables scheduled workflows after ~60 days with no repo activity — re-run manually if quiet that long.
- `AA_API_KEY` stays in `.env` / repo secrets, never in client code or git.
