# truth — Truth Orchestrator Worker

A single Cloudflare Worker that answers a query in layers: a curated **master** truth, then a
**provisional** candidate, then a **memory** row, then (if a model is reachable) a generated
candidate. A total miss is `200` with `status: "miss"` — not a `404`.

Live: <https://truth.ai-n.workers.dev>

`src/worker.js` is the whole Worker — the deployed bundle, pulled down from Cloudflare with
`npx create-cloudflare@latest truth --existing-script truth`. There is no build step.

## Bindings

| Binding | Resource |
|---|---|
| `MASTER_TRUTHS` | KV `master truths` (`904eff3af4c14f06964e4f651cce9573`) |
| `PROV_TRUTH` | KV `Memoria` (`4a9bfbffb4d440cba86badc508e2045a`) — the main memory store |
| `QC` | Durable Object `QCLogger` |
| `TBACKED` | R2 `ai-nspired` (audit JSONL) |
| `JUSTICE` | Service binding → `justice` |

`PROXY` is referenced by the code but **not bound**, so candidate generation currently cannot
run. This Worker holds no secrets and has **no authentication on any route** — do not expose new
write paths without adding a gate.

## Routes

`POST /api/truth/resolve`, `/master/create`, `/master/update`, `/master/delete`,
`/provisional/update`, `/provisional/delete`, `/promote`; `GET /api/truth/provisional/query`,
`/debug/kv-status`.

Full contract, verified store topology, and known landmines: **[HANDOFF.md](HANDOFF.md)**.

## Develop and deploy

```bash
npm install
npx wrangler dev          # local
npx wrangler deploy       # publish
npx wrangler deploy --dry-run   # build check, no publish
```

Before connecting Cloudflare Workers Builds to this repository, read the two config gaps noted in
HANDOFF.md § "What the reconstructed config is missing" — the incomplete `QCLogger` export and the
absent `PROXY` binding.

## Local checks

`node --check` does not apply to an ES-module bundle, so verification is done by importing the
handler with a stub KV and exercising routes (no network, no deploy).
