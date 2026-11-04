# HANDOFF — Truth Orchestrator (`truth`)

> **Scope of this document.** Everything below is taken from `truth/src/worker.js` — the
> deployed script, pulled down with
> `npx create-cloudflare@latest truth --existing-script truth` (the built bundle, hence the
> `__defProp` / `__name` prelude). That fetch did **not** reconstruct `wrangler.jsonc`
> bindings; see § Bindings for what the config still needs. The doc is written so the next
> person does not have to re-derive the contract from the bundle.
>
> **No behavior changes were made alongside this doc.** It is documentation only. The
> bugs in [Landmines](#landmines-read-before-you-touch-anything) were first *documented, not
> fixed*; `1` and `4` have since been fixed and **deployed to production on 2026-11-03** (version
> `a82aa770-5234-4b88-bc3b-d79cd240e5f1`). The items still listed as open are still open.

Live deployment: `https://truth.ai-n.workers.dev`
Verified against production on 2026-11-03 (§ [Production checks](#production-checks)).

---

## What this worker is

One Worker, one handler, nine routes. It answers a question in three layers, in order:

1. **MASTER** — curated truth (`master:<id>` in `MASTER_TRUTHS`). Answers with no model involved.
2. **PROVISIONAL** — a candidate (`provisional:<id>` in `PROV_TRUTH`). Answers, but flagged
   unverified. **In production this layer has nothing to read:** `Memoria` holds zero
   `provisional:` and zero `truth:provisional:` keys (§ [What the keys actually are](#what-the-keys-actually-are-same-api-same-day)).
2b. **MEMORY** — `truth:<id>` in that same namespace: the 1,150 main-memory rows. Read by
   `getMemory()` and reported at the `provisional` layer with `source: "memory_truth"`.
3. **PROXY candidate** — no truth at all yet, so ask `env.PROXY` for one. Whatever comes back is
   *written as* a provisional (24 h TTL) and returned.

A total miss is **HTTP 200 with `status: "miss"`**, not a 404. Clients must branch on `status`,
not on the status code.

The header comment states the three behaviors that must survive any refactor:

```
// 1) getProvisional checks both "provisional:<id>" (new) and "truth:provisional:<id>" (legacy).
// 2) /api/truth/provisional/update is create-or-update:
//    - If provisional exists: update it.
//    - If not: create it as a provisional.
// 3) Proxy candidate flow uses strict JSON-array instruction.
```

---

## Bindings

| Binding | Type | Used for |
|---|---|---|
| `MASTER_TRUTHS` | KV | `master truths` — `904eff3af4c14f06964e4f651cce9573`; the **same namespace** as `mind`'s `MASTER` |
| `PROV_TRUTH` | KV | `Memoria` — `4a9bfbffb4d440cba86badc508e2045a`; the **same namespace** as `mind`'s `MEM` |
| `QC` | Durable Object → `QCLogger` | in-memory event log (capped 1,000), mirrored to R2 |
| `TBACKED` | R2 | daily JSONL audit: `qc-events/YYYY-MM-DD.jsonl` |
| `PROXY` | Service binding | `/candidate` — model call for candidates |
| `JUSTICE` | Service binding, optional | `/notify/provisional`, `/notify/delete` fire-and-forget |

### Topology (verified against the Cloudflare API, 2026-11-03)

Account `ai-nspired` (`771ee2cc20f09248129114b7535b2cc9`). Two KV namespaces are shared across
Workers, and both of them are the ones that matter here:

| Namespace | Title | Bound as |
|---|---|---|
| `904eff3af4c14f06964e4f651cce9573` | `master truths` | `MASTER` in `mind`, `MASTER_TRUTHS` in `truth`, also `TRUTHS` (`cods`), `MASTER_TRUTHS` (`ai-proxy`, `orproxy`), `MASTER_TRUTHS` (`ingenium-veritas`) |
| `4a9bfbffb4d440cba86badc508e2045a` | `Memoria` | `MEM` in `mind`, `PROV_TRUTH` in `truth`, also `memy` (`justice`, `neo-te`, `truthengine`, `ingenium-veritas`), `MEMORIA` (`ai-proxy`), `KV` (`cods`, `cards`, `ara`) |

So **MEM and PROV_TRUTH are one store.** It is the user's main memory — the layer that exists
*before* the curated truth — not a pile of throwaway candidates. There is no separate candidate
store, and `PROV_TRUTH` is a misleading name for it.

Counted with a cursor walk, `Memoria` holds **1,192 keys**, not the `954` that
`/debug/kv-status` reports: both the Worker's `list()` and a single REST page return a short
first page of 954 and stop. 238 keys are invisible to anything that does not follow
`result_info.cursor`.

### What the keys actually are (same API, same day)

| Namespace | Prefix | Keys |
|---|---|---|
| `Memoria` — 1,192 | `truth:` | 1,150 |
| | `candidate:` | 13 |
| | `ans:` | 6 |
| | `budget:` | 4 |
| | `deck:` | 2 |
| | `prefs:` / `approved:` | 1 / 1 |
| | loose (hash keys, test keys) | ~15 |
| `master truths` — 65 | `review:` | 33 |
| | `master:` | 18 |
| | `truth:` (`truth:t_<hex>`) | 13 |
| | `truths:index` | 1 |

The consequences are the important part:

- **`provisional:` = 0 and `truth:provisional:` = 0.** `getProvisional`'s two lookups cannot hit
  anything, so layer 2 never fires. The dual-key logic was written for a key shape this store does
  not use — memory is stored as `truth:<slug>`, which neither lookup matches.
- **`master:` = 18 of 65.** `getMaster` only reads `master:<id>`, so the other 47 keys in
  `master truths` (33 `review:`, 13 `truth:t_<hex>`, `truths:index`) are unreadable by this Worker.
- **Layer 3 never fires either**, because `env.PROXY` is unbound (§ below).
- Net effect: **`/api/truth/resolve` can only ever answer from those 18 rows.** The 1,150 memory
  entries are unreachable through this API — not because they are low quality, but because no
  reader queries their prefix.

Note also that the earlier MEM prefix list (`settings:`, `tts:`, `upload:`, `decision:`, `review:`)
 does not match this store: `truth:settings:` is 0, and `review:` lives in `master truths`, not here.

Expirations, from the same listings: in `Memoria`, **23** of the first 954 keys carry an
`expiration` and the rest are permanent — the memory is durable, not a decaying backlog. In
`master truths`, **33** keys carry an `expiration`, which equals the `review:` count exactly: the
review queue is the TTL'd part, while the 18 `master:` rows persist. Neither namespace exposes key
`metadata` (0 in both samples), so any filtering has to read the value, not the key.

**The console is the `mind` Worker, not this one.** `mind` carries `ADMIN_KEY` (the gate), `ASSETS`
(the console page), `MASTER` → `master truths`, `MEM` → `Memoria`, and a `TRUTH` service binding to
this Worker. That is why `/admin` on this hostname is a `404` and why `503 admin_key_unset` comes
from somewhere else.

`JUSTICE` and `QC` are both optional at runtime: notifications are `.catch(() => {})`'d and
`logEvent` warns and returns if `env.QC` is missing. **A resolve never fails because logging
or notification is down.** Keep it that way.

### What the reconstructed config is missing

`npx create-cloudflare@latest truth --existing-script truth` restored `MASTER_TRUTHS`,
`PROV_TRUTH`, `QC` (class `QCLogger`), `TBACKED` (bucket `ai-nspired`) and `JUSTICE` into
`truth/wrangler.jsonc`, at the dashboard's `compatibility_date` of `2025-11-10`.

Two gaps to check before relying on it:

- **`PROXY` is absent — confirmed, not inferred.** The deployed `truth` settings list only
  `JUSTICE`, `MASTER_TRUTHS`, `PROV_TRUTH`, `QC` and `TBACKED`. The code guards with
  `if (!env.PROXY) throw new Error("PROXY service not bound")` and `handleResolve` catches that
  and returns a `miss`, so **layer 3 can never produce a candidate in production**: every cache
  miss is a miss. Nothing is generating new memories through this Worker.
- **The `QCLogger` export is incomplete.** `wrangler deploy --dry-run` succeeds — 31.74 KiB,
  `QC`/`MASTER_TRUTHS`/`PROV_TRUTH`/`TBACKED`/`JUSTICE` all resolve — but it warns that
  `durable_objects` names `QCLogger` with no live `exports` entry, and suggests
  `"exports": { "QCLogger": { "type": "durable-object", "storage": "sqlite" } }`. Do not paste
  that blindly: the `storage` value picks a backend for a Durable Object that already exists in
  production, so check how the live object is backed before changing it. There is also no
  `migrations` block, which matters if the class is ever recreated.

---

## API contract

All responses are JSON with `Access-Control-Allow-Origin: *`. `OPTIONS` returns `204`.
Anything unmatched returns `404 {"error":"Not found"}`. Any thrown error returns
`500 {"error": <message>}`.

| Method | Path | Body / query | Success | Failure |
|---|---|---|---|---|
| POST | `/api/truth/resolve` | `{ input }` | `200` canonical, `layer: master \| provisional`, or `200` `status: "miss"` | `400 input required` |
| POST | `/api/truth/master/create` | `{ input, answer, editor?, meta? }` | `200` canonical, `layer: master` | `400` |
| POST | `/api/truth/master/update` | `{ input, answer, editor?, meta?, expectedVersion? }` | `200` canonical | `404 Not found`, `409 VERSION_CONFLICT` |
| POST | `/api/truth/master/delete` | `{ input }` | `200 { success: true, id }` | `400` |
| POST | `/api/truth/provisional/update` | `{ id \| input, answer?, meta? }` | `200` canonical, `layer: provisional` | `404 Not found` |
| POST | `/api/truth/provisional/delete` | `{ id \| input }` | `200 { success: true, id }` | `400` |
| GET | `/api/truth/provisional/query` | `?q=<string>&limit=20` | `200 { success, results: [] }` | `400 query too long` (>100 chars) |
| POST | `/api/truth/promote` | `{ id \| input, editor?, confidence?, literalVsMythos?, model?, eitlVerifier? }` | `200` canonical, `layer: master` | `404 No provisional truth to promote` |
| GET | `/debug/kv-status` | — | `200 { masterCount, provisionalCount }` | `500` |
| OPTIONS | any | — | `204`, no body | — |

Two gotchas that are not obvious from the table:

- **`master/update` requires `answer`.** `if (!body.input || !body.answer)` → `400`. You cannot
  edit just the metadata; you must resend the full answer.
- **`provisional/update` does not.** Omitting `answer` creates nothing and returns `404`, but
  updates an existing row while leaving its answer alone.

### The canonical response

Every truth-bearing route returns this shape, built by `canonicalResponse()`. `meta` and
`audit` are always present; absent fields are `null`, never missing.

```js
function canonicalResponse({
  status = "ok", layer = "none", id, query = null, answer = null,
  source = "none", verified = false, provisional = false,
  eitl = null, meta = {}, audit = null
}) {
  return {
    status, layer, id, query, answer, source, verified, provisional, eitl,
    meta: {
      literalVsMythos: meta.literalVsMythos ?? null,
      model: meta.model ?? null,
      confidence: meta.confidence ?? null,
      citations: meta.citations ?? [],
      created: meta.created ?? null,
      updated: meta.updated ?? null,
      qc_passed: meta.qc_passed ?? null,
      spec_version: meta.spec_version ?? SPEC_VERSION,
      reason: meta.reason ?? null
    },
    audit: audit ?? null
  };
}
```

`SPEC_VERSION` is `"v1"`. A miss is the same envelope with `status: "miss"`, `layer: "none"`,
`source: "none"`, and `meta.reason` set — see `missResponse()`.

### Stored record shapes

`setMaster()` writes exactly this (there is no `version` on create; `updateMaster` adds one):

```js
const record = {
  id, query, answer,
  created: meta?.created ? Number(meta.created) : now,
  updated: now,
  editor: editor || "justice",
  source: "justice",
  verified: true,
  literalVsMythos: meta?.literalVsMythos || "literal",
  model: meta?.model || "justice-approved",
  confidence: clampMasterConfidence(meta?.confidence),
  eitlVerifier: meta?.eitlVerifier || editor || "justice",
  metadata: meta?.metadata || {}
};
```

`setProvisional()` writes the same skeleton plus `citations`, `provisional: true`,
`verified: false`, `qc_passed`, and always with `{ expirationTtl: 24 * 3600 }`.

Note `metadata.citations` is where citations live for master rows, while provisional rows keep
`citations` at the top level. `handleResolve` reads master citations from
`master.metadata.citations` and provisional citations from `prov.citations`. Do not "unify" these
without migrating both stores.

### Ids

Every route normalizes its identifier with `normalizeId()`. It is the primary key **and** the
join key between the two stores, so it is effectively part of the contract:

```js
  q = q.replace(/\s+/g, "-").substring(0, 100);
  if (q.length > 50) {
    const enc = new TextEncoder().encode(input);
    let hash = 0;
    for (const b of enc) { hash = ((hash << 5) - hash + b) | 0; }
    const short = (hash >>> 0).toString(16).substring(0, 8);
    q = q.substring(0, 80) + "-" + short;
  }
  return q || "unknown-query";
```

Before that it lowercases, strips non-word characters, collapses whitespace, and applies stem
rules (`running`→`run`, `programming`→`program`, …). **Ids are lossy in both directions:** the
stemming merges distinct questions, and the hash is 8 hex chars, so collisions are possible.
A collision means one question is answered with another's truth. Curation should be aware that
`id` is not a faithful record of `query`.

---

## Verbatim snippets worth keeping

### Dual-key read (legacy compatibility)

New code writes `provisional:<id>`. Rows written before the rename live at
`truth:provisional:<id>`. `getProvisional` is the only place that knows this:

```js
async function getProvisional(env, inputOrId) {
  const id = normalizeId(inputOrId);
  if (!env.PROV_TRUTH) return null;

  const newKey = KV_PREFIX_PROV + id;          // "provisional:<id>"
  const legacyKey = `truth:provisional:${id}`; // legacy

  let rawProv = await env.PROV_TRUTH.get(newKey);
  let key = newKey;
  if (!rawProv) {
    rawProv = await env.PROV_TRUTH.get(legacyKey);
    key = legacyKey;
  }
  if (!rawProv) return null;
  ...
}
```

`key` is assigned but never used — the delete paths re-derive the new-style key instead, which is
the cause of the first landmine below.

### Create-or-update

`handleProvisionalUpdate` delegates to this; the create branch is the interesting half:

```js
  const raw = await env.PROV_TRUTH.get(key);

  if (!raw) {
    // If no existing record: create, but only if answer is provided
    if (!answer) {
      const e = new Error("NOT_FOUND");
      e.code = "NOT_FOUND";
      throw e;
    }
    const record = { /* …, source: "justice_manual", model: meta?.model || "manual" */ };
    await env.PROV_TRUTH.put(key, JSON.stringify(record), {
      expirationTtl: 24 * 3600
    });
    return record;
  }
```

### Confidence clamps

Two different ranges, deliberately:

```js
function clampMasterConfidence(c) {
  const n = Number(c);
  if (!Number.isFinite(n)) return 0.97;
  const clamped = Math.max(0.9, Math.min(1, n));   // 0.9 – 1.0
  return Number(clamped.toFixed(4));
}
function clampProvisionalConfidence(c) {
  const n = Number(c);
  if (!Number.isFinite(n)) return 0.5;
  const clamped = Math.min(0.8, Math.max(0, n));   // 0.0 – 0.8
  return Number(clamped.toFixed(4));
}
```

A master can never report below `0.9` and a provisional can never report above `0.8`, so the
numbers are not comparable across layers.

### Promote

```js
  const prov = await getProvisional(env, idOrInput);
  if (!prov || !prov.answer) {
    return new Response(JSON.stringify({ error: "No provisional truth to promote" }),
      { status: 404, headers: CORS_HEADERS });
  }
  const record = await setMaster(env, {
    query: prov.query || idOrInput,
    answer: prov.answer,
    editor: body.editor || "justice",
    meta: { /* …, metadata: { ...(prov.metadata || {}), citations: prov.citations || [], promotedFrom: prov.id } */ }
  });
  await deleteProvisional(env, prov.id);
```

`promotedFrom` records provenance. Note there is **no overwrite check** — see landmine 2.

### The 404

```js
      return new Response(JSON.stringify({ error: "Not found" }), {
        status: 404,
        headers: CORS_HEADERS
      });
```

This is the exact body production returns for `/`, `/admin`, `/docs` — see below.

---

## Changes in this branch

Three edits to `truth/src/worker.js`, **live in production** as of 2026-11-03 (version
`a82aa770-5234-4b88-bc3b-d79cd240e5f1`). The pre-deploy script is backed up in
`/tmp/truth-live-1793742986.js` and Cloudflare retains the previous version for rollback.

1. **The resolver reads memory rows.** New `getMemory()` looks up `truth:<id>` in the same
   namespace and is called between the provisional and proxy layers. It is **read-only** — no
   write or delete path can touch a memory row. The response keeps the existing contract rather
   than adding a new `layer` value: `layer: "provisional"`, `verified: false`, `provisional:
   true`, with `source: "memory_truth"` to distinguish it. Confidence goes through
   `clampProvisionalConfidence`, so a memory row stored at `0.9` reports `0.8` — deliberate, to
   stay inside the layer's existing bound.
2. **`master/update` TDZ fixed** (landmine 1): `existing.updated` now uses `|| 0`, matching the
   `created` line above it, which removes the only read of `now` before its declaration.
3. **`/debug/kv-status` walks the cursor** (landmine 4), via a new `countKeys()` for both
   namespaces.

One string changed: the miss reason is now
`no_master_memory_or_provisional_and_no_valid_candidate`. If a client matches the old value
exactly, update it.

Verified with a local harness (stub KV, real handler, no network, no deploy): memory hit, memory
flagged unverified, master still outranking memory, miss still `200` with `status: "miss"`,
`master/update` returning `200`, and a 1,192-key store counting correctly.

Confirmed in production after the deploy: `/debug/kv-status` returns `1,192`, and a `resolve` for a
key that exists only as a memory row changed from `status: "miss"` to `status: "ok"` with
`source: "memory_truth"`. `master/update` was **not** exercised against a curated row — only
against a stub — so a first real edit should be done on a throwaway id.

---

## Landmines (read before you touch anything)

Status: **1 and 4 are fixed and live.** Everything else below still describes the running Worker.

1. **`/api/truth/master/update` threw on every existing key — fixed and live.** In
   `updateMaster`, `now` was used before it is declared:

   ```js
     existing.created = Number(existing.created) || 0;
     existing.updated = Number(existing.updated) || now;   // ← `now` is in TDZ here
     ...
     const now = Date.now();                               // ← declared after
   ```

   `const now` is block-scoped, so this was a `ReferenceError: Cannot access 'now' before
   initialization` that the router's `catch` turned into a `500` for any row that exists. `404`
   still worked — it is thrown before this line when the key is absent. The fix is one token:
   `Number(existing.updated) || 0`. With the TDZ gone, `409 VERSION_CONFLICT` becomes reachable as
   designed, but it still only fires when the stored row already carries a `version` field, which
   only this function writes. Test `master/update` against a throwaway id, never a curated one —
   a successful call overwrites the row.

2. **`promote` silently overwrites an existing master truth.** `setMaster` is an unconditional
   `put`. If a provisional is promoted for an id that already has a curated truth, the curated
   answer is replaced and the old value is unrecoverable. Any "never overwrite, offer
   `overwrite`/`alternatives[]`" behavior lives in the console, *not here*. Do not assume this
   route is safe to call blindly from a new UI.

3. **Legacy rows survive `delete` and `promote`.** `deleteProvisional` deletes
   `provisional:<id>` only:

   ```js
   async function deleteProvisional(env, idOrQuery) {
     const id = normalizeId(idOrQuery);
     const key = KV_PREFIX_PROV + id;
     await env.PROV_TRUTH.delete(key);
     return { id };
   }
   ```

   For a row that still lives at `truth:provisional:<id>`, deleting is a **no-op**, and
   `promote` will `delete` the wrong key after writing the master. The legacy row keeps
   resolving at the provisional layer forever. A one-time sweep of `truth:provisional:*` is the
   real fix.

4. **`/debug/kv-status` undercounted below the cap — fixed and live.** `list()` returns one short
   page and no cursor, so it reported **954** where a cursor walk finds **1,192**. `countKeys()`
   now follows the cursor, and production reports `1,192`. The route is still public and
   unauthenticated.

5. **`queryProvisionals` costs one KV op per hit.** It lists keys, then `get`s each one. With
   `limit=20` that is 1 list + up to 20 reads, and `limit` is passed straight through from the
   query string with no ceiling. Matching is a **prefix match on the normalized id**, not a
   substring search — `q=a` returns everything whose id starts with `a`.

6. **Nothing on this Worker is authenticated.** There is no key check anywhere. `ACCESS-Control-
   Allow-Origin: *` means any origin in a browser can `create`, `update`, `delete` or `promote`
   curated truths. The console's `ADMIN_KEY` gate does not protect these routes — they are a
   different hostname and a different Worker. Do not describe this API as protected.

7. **Provisional TTL resets on every write.** `expirationTtl: 24 * 3600` is set on create *and*
   update, so a candidate that keeps being touched never expires, and one that is never promoted
   disappears silently after a day. Promotion is the only persistence path.

8. **`expectedVersion` mostly does nothing.** The `409` fires only when the stored row already has
   a `version` field, which only `updateMaster` ever writes — and `updateMaster` is the broken
   path in landmine 1. In practice first edits never conflict.

---

## What must not change

- **The three-layer order: master → provisional → proxy.** A provisional must never outrank a
  master, and a proxy candidate must never be returned with `verified: true`.
- **Miss is `200`, not `404`.** Changing it would break every existing client's error handling.
- **Dual-key provisional reads.** Dropping the legacy lookup orphans rows still in production.
- **`provisional/update` staying create-or-update.** Justice's edit flow depends on it.
- **The strict JSON-array proxy instruction.** `requestCandidateFromProxy` rejects anything that
  is not a non-empty array and throws `NO_CANDIDATE`; the resolve path converts every such
  failure into a `miss` rather than an error. Do not let a proxy schema change surface as a `500`.
- **`JUSTICE` / `QC` failures never break a resolve.**
- **Confidence clamps** (0.9–1.0 master, 0.0–0.8 provisional).

---

## Production checks

Copy-paste, in order. All are read-only except where noted.

```bash
BASE=https://truth.ai-n.workers.dev

# Deployment is alive and is this Worker (identical 404 envelope)
curl -s "$BASE/"                        # {"error":"Not found"}

# Contract: empty input is rejected without touching a model
curl -s -X POST "$BASE/api/truth/resolve" \
  -H 'Content-Type: application/json' -d '{"input":""}'       # {"error":"input required"}

# CORS preflight
curl -s -o /dev/null -w '%{http_code}\n' -X OPTIONS "$BASE/api/truth/resolve"   # 204

# Store sizes (see landmine 4 — 1000 is a cap, not a total)
curl -s "$BASE/debug/kv-status"

# Candidate listing
curl -s "$BASE/api/truth/provisional/query?limit=5"
```

Observed on 2026-11-03:

```
POST /api/truth/resolve {"input":""}        → 400 {"error":"input required"}
GET  /api/truth/provisional/query?limit=1   → 200 {"success":true,"results":[]}
GET  /debug/kv-status                       → 200 {"masterCount":65,"provisionalCount":1192}
                                               (reported 954 before the 2026-11-03 deploy)
POST /api/truth/resolve {"input":"hello"}   → 200 ok, layer "provisional",
                                               source "memory_truth", confidence 0.8
                                               (was a miss before the deploy)
OPTIONS /api/truth/resolve                  → 204
GET  / , /admin , /docs , /admin.html       → 404 {"error":"Not found"}
```

That last line is **expected**: `/admin` belongs to the console Worker, not this one. A `503
admin_key_unset` from `/admin` is the gate working on *that* deployment; a `404` from this
hostname is not a bug.

---

## Where curation goes next

**The promotion design is in [`CURATION.md`](CURATION.md)** — answer clustering, ballot integrity,
the evidence gate that keeps popularity from ratifying anything, what confidence can honestly mean,
contested answers with alternatives, and the phase plan. This section is only the state of the data
that design has to work with.

The numbers are the whole story, and they are not what the earlier notes assumed:

- **18** `master:` rows are addressable. The other 47 keys in `master truths` are `review:`,
  `truth:t_<hex>` and `truths:index`, and no route reads them.
- **1,150** memory rows sit under `truth:<slug>` in `Memoria`, and `resolve` reads that prefix
  as of the 2026-11-03 deploy.
- **0** `provisional:` rows exist, so layer 2 never fires; `PROXY` is unbound, so layer 3 never
  fires. The 24 h TTL is not decaying a backlog — nothing is being written.

**The reader half of that is now done** — `getMemory()` serves `truth:<slug>` at the provisional
layer. What is still missing is the other direction: nothing turns a memory row into a curated
`master:` row except a hand-written `master/create` call. That is a collection gap now, not a
reader gap, and it is the thing to build next.

In rough priority order:

1. **Master edits are unblocked** (landmine 1 fixed). Test against a throwaway id under
   `wrangler dev`, then deploy — until it is deployed, curation still has only read access.
2. **Decide the TTL policy — but note it is not touching `Memoria` today.** `expirationTtl` is
   applied only by `setProvisional` / `updateProvisional`, and there are no `provisional:` rows.
   The 1,150 `truth:*` rows carry whatever expiry the Worker that wrote them chose; check
   `expiration` on those keys before assuming they are durable.
3. **Add a per-kind filter over the real key space.** Paging through 1,150 `truth:*` rows 500 at
   a time is what makes curation stall. Do not assume these rows carry `source`, `confidence` or
   `citations` — they were written by other Workers, not by this Worker's `setProvisional`, so
   inspect one before designing a filter around fields that may not exist.
4. **Sweep the legacy keys.** Landmine 3 means an unknown number of rows cannot be deleted or
   cleaned up through the API. Enumerate `truth:provisional:*`, rewrite or drop, once.
5. **Fix the count.** `min(total, 1000)` will lie exactly when the backlog is worst.
6. **Then the console** — prefix-filtered loading, counts before load, per-kind filtering, and the
   500-row render cap, as already listed in the earlier handoff for that Worker.
7. **Guard `promote`.** Before any UI exposes it to a human, add a conflict check or return
   `alternatives[]`; today it is a silent overwrite.

---

## First day

1. Read the three behaviors in the header comment, then § Changes in this branch and § Landmines.
   Landmines 1 and 4 are fixed and live; the rest still describe the running Worker.
2. Run the five `curl`s in § Production checks and confirm you get the same numbers.
3. Confirm which Worker serves the console (`/admin` → `503 admin_key_unset`) and which serves
   this API (`404`). They are different hostnames; do not hand out the wrong one.
4. `truth/src/worker.js` is the whole Worker — there is no build step in what was handed over, and no
   test suite for this Worker. Any `npm run check` / `npm test` / `npm run build` from earlier
   notes belongs to the console project, not here.
5. Before changing anything: the only safe mutation to test with is
   `provisional/update` on a throwaway id. `master/create` and `promote` write to the curated
   store immediately.
