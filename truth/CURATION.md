# Curation plan — choosing a master truth without trusting popularity

**Status:** design, nothing implemented. Companion to [`HANDOFF.md`](HANDOFF.md).
**Scope:** how a candidate becomes a ratified `master:` truth, how confidence is scored, and what
happens when two answers are close and nobody can settle it.

The central rule, stated once and applied everywhere below:

> **Votes decide what is worth looking at. Evidence decides what gets served as truth.**
> A majority can nominate an answer for ratification. A majority cannot ratify one. If the only
> thing behind a candidate is agreement, it stays a candidate forever — no matter how many people
> agreed.

---

## 1. What the code does today (the starting conditions)

Read `src/worker.js` before designing around it. These are constraints, not opinions.

| Fact | Where | Why it matters |
|---|---|---|
| Only `candidates[0]` is ever used | `handleResolve`, proxy flow | `PROXY` returns an *array*; everything past the first answer is discarded. There is no multi-candidate state, so "many votes" has nothing to vote on yet. |
| Provisional rows expire in 24 h | `setProvisional`, `updateProvisional` (`expirationTtl: 24*3600`) | Any ballot placed here is gone tomorrow. Votes need durable storage; provisional rows stay a render cache. |
| `promote` deletes the provisional it promoted | `handlePromote` | After promotion the losing candidates leave no trace. No audit trail of what was rejected. |
| Every curated answer is `verified: true` at ≥ 0.9 | `clampMasterConfidence` (`Math.max(0.9, …)`), `canonicalResponse` | The contract **cannot express "we aren't sure"**. A 51/49 split would be served as a 0.97 certainty. This is the single biggest blocker to honest confidence scores. |
| `promote` overwrites unconditionally | `setMaster` → bare `put` | A second promotion silently clobbers the first. `updateMaster` already implements `expectedVersion` + 409 `VERSION_CONFLICT`; `promote` just doesn't use it. |
| No route has authentication | `truthHandler.fetch` | Any ballot endpoint is openly writable by anyone on the internet until a gate exists. Vote weighting must assume unauthenticated input. |
| `review:` keys exist but nothing here writes them | 33 TTL'd keys in `master truths` | An orphan prefix. Something else expects to consume review items. Decide the owner before adding a second review channel. |
| `normalizeId` stems and truncates | `normalizeId` | `run`/`running`/`runs` collapse to one id, and >50 chars becomes `80 chars + 8-hex hash`. Master is keyed by that id, so two genuinely different questions can share one answer slot. A self-populating KB will hit this. |
| Memory rows carry a `confidence` from an unknown model | 1,150 `truth:<id>` rows | Treating that number as a prior imports someone else's self-assessment. It is a hint for ordering, never evidence. |
| R2 audit appends rewrite the whole day file | `QCLogger` (`old + line`, then `put`) | Quadratic per day-file. Fine at today's volume; a vote log would multiply it. Not the place to put ballots. |

---

## 2. Separate the four numbers people call "confidence"

One score cannot answer "is this right?" and "do people agree?". Keep them apart, and only ever
publish a *derived* confidence so the API stays simple.

| Quantity | Meaning | Source | Can votes move it? |
|---|---|---|---|
| **Support** | weighted agreement mass for one answer | ballots | yes, by definition |
| **Share** | support ÷ total support = the vote split | ballots | yes |
| **Evidence** | grounding behind the answer | citations checked, verifier signature | **no** |
| **Freshness** | how long since the substantive claim was confirmed | timestamps | no (decays) |
| *Confidence* (published) | calibrated statement of trust | function of the four above | yes, but capped by evidence |

The published `confidence` must be monotone in evidence and **capped by it**. No amount of support
lifts an answer above the ceiling its evidence allows:

```
ceiling(evidence) = { 0: 0.55, 1: 0.70, 2: 0.90, 3: 0.99 }
```

So an unsourced answer that 5,000 people endorsed publishes at 0.55 — visibly provisional — while a
single verifier-signed answer publishes at 0.99. That one line is the mechanical expression of
"popularity doesn't make it right".

**Prerequisite:** `clampMasterConfidence` must stop flooring at 0.9, and `canonicalResponse` needs a
`status` field (`canonical` | `contested` | `provisional` | `deprecated`). Until then, every master
truth is advertised as near-certain and this design cannot be expressed at all.

---

## 3. Answers, not rows — cluster before you count

Free text fragments votes. "Use `wrangler deploy`", "run wrangler deploy", and "`npx wrangler
deploy`" are one answer with three spellings; counted naively each gets a third of the support and
nothing reaches quorum while the question is effectively settled.

Cluster candidate answers before tallying:

1. Normalize: trim, collapse whitespace, strip trailing punctuation, lowercase for comparison only
   (never for storage — preserve the original text as the answer).
2. Near-duplicate test: token-set Jaccard ≥ 0.85, **or** embedding cosine ≥ 0.92 if embeddings are
   available. Merge into a cluster; keep the highest-evidence member as the cluster's representative
   text, breaking ties by earliest `created`.
3. Only clusters are voted on and scored. Displaying a cluster shows its members' paraphrases as
   aliases, so a voter recognises their own wording.

Without this step, "a lot of votes" measures vocabulary, not truth.

---

## 4. Ballot integrity — the part that decides whether any of this survives

An open endpoint with unlimited anonymous votes is not a knowledge base, it is a brigading target.
Assume that from day one:

- **Identity.** One ballot per voter per question id per epoch (default 30 days). Re-voting replaces
  the previous ballot (last write wins, versioned) so people can change their minds without
  inflating counts. Until real auth exists, the voter identity is a best-effort key (hashed
  user id if present, otherwise a coarse fingerprint) and is explicitly *not* trusted.
- **Trust tiers, capped.** Weights: anonymous 0.1, member 1, experienced 2, verifier 4, editor 8.
  **No single voter may hold more than 20% of quorum**, so one enthusiastic editor cannot ratify
  alone.
- **Anonymous votes nominate, never ratify.** Their support counts toward ranking and toward
  surfacing a candidate; their weight counts as zero toward the quorum gate.
- **Two ballot kinds.** `endorse` (I think this is right) and `dispute` (I think this is wrong):
  dispute with evidence −1.0, dispute without evidence −0.25. Silent negation is barely worth
  anything; a cited correction is worth more than a vote.
- **Decay.** Half-life 180 days from ballot time. Votes are not eternal; a wrong answer's
  accumulated agreement should fade rather than fossilise.
- **Rate limits and burst detection.** Per identity and per question id. Flag when one id's ballots
  arrive in a tight burst or from a single network range — flag, don't silently discard, so the
  audit shows why a tally looks odd.

---

## 5. The decision function

For each question id and answer cluster *c*:

```
support(c)  = Σ  weight(voter) × kind(vote) × 0.5^(age_days / 180)
share(c)    = support(c) / Σ support(all clusters)
quorum      = 10.0 weighted support, at least 3 distinct non-anonymous voters
margin      = (support(top) − support(runner_up)) / Σ support(all)
H_norm      = −Σ share·ln(share) / ln(k)        # k = clusters considered
```

Then:

| Condition | Outcome |
|---|---|
| `evidence(top) < 2` | **never ratifies.** Stays a candidate. |
| `quorum` unmet | **nominated**, not ratified. Surfaced for a human. |
| `margin < 0.15` or `H_norm > 0.85` | **contested** — close enough that serving one answer with confident framing is a lie. |
| `evidence ≥ 2` and `quorum` met and `margin ≥ 0.15` | eligible for ratification. |
| eligible **and** a verifier signature (`evidence == 3`) | auto-ratify. |
| eligible, `evidence == 2`, no signature | ratify **only with human confirmation** (phase 3+; see §7). |

Starting published confidence for a ratified answer, floored by the evidence ceiling:

```
confidence = min( ceiling(evidence),
                  0.60 + 0.30 × min(1, support / quorum) + 0.10 × (evidence == 3) )
```

Every threshold above is a **starting value chosen to be conservative**, not a finding. Tune them
against the revert rate (§8), not against how they feel.

---

## 6. When answers are close: give people the option

Contested is a first-class outcome, not a failure. When `margin < 0.15` or entropy is high:

```json
{
  "status": "contested",
  "layer": "provisional",
  "answer": "<best-supported, still served>",
  "verified": false,
  "provisional": true,
  "source": "contested_candidates",
  "meta": {
    "confidence": 0.55,
    "status": "contested",
    "alternatives": [
      { "answer": "…", "share": 0.46, "evidence": 2 },
      { "answer": "…", "share": 0.41, "evidence": 1 }
    ],
    "dissent": { "clusters": 3, "h_norm": 0.91 }
  }
}
```

Rules for this:

- **Additive only.** New fields (`status`, `alternatives`, `dissent`) — do not add a `layer` enum
  value. Clients that ignore unknown fields keep working, exactly as they did when memory hits
  arrived as `source: "memory_truth"` on the existing provisional layer.
- **Serve the leader, labelled.** A plausible answer plus visible disagreement beats a miss. The
  label is not decoration — `verified: false` and `confidence ≤ 0.55` must hold for every contested
  response.
- **Never flip silently.** A master can be demoted to contested only after it has been contested
  for 7 uninterrupted days, and demotion writes an audit event with the ballot snapshot that caused
  it.
- **Dissent is preserved, not overwritten.** Losing clusters stay addressable at
  `contested:<id>/<cluster_id>` so a later reader can see what was rejected and why. Today
  `promote` deletes that history.

---

## 7. Lifecycle and who may move it

```
  candidate ──(second source / citation)──▶ corroborated
      │                                          │
      │                                   (votes accumulate)
      ▼                                          ▼
  discarded ◀──(no support, 30d)──────────  nominated
                                                 │
                              evidence ≥2 + quorum + margin
                                                 ▼
                                     contested ◀──▶ ratified (master:)
                                            │
                                     (revert / newer evidence)
                                            ▼
                                        deprecated
```

| Transition | Who | Reversible | Guard |
|---|---|---|---|
| candidate → corroborated | automatic (evidence appears) | yes | evidence recomputed, never cached forever |
| → nominated | automatic (quorum) | yes | notification only, no write to `master:` |
| → ratified | verifier signature (auto); otherwise human | yes, via revert | must pass through `updateMaster` with `expectedVersion` → 409 on concurrent edit |
| ratified → contested | automatic (7-day contest) or human | yes | audit event with ballot snapshot |
| ratified → deprecated | human only | yes (soft delete) | row retained with `deprecated: true`, not `delete` |

Non-negotiable mechanical rules:

- **Use the existing conflict guard.** `promote` must call `updateMaster`'s `expectedVersion` path,
  not `setMaster`'s blind `put`. The 409 exists and is currently unused.
- **Every transition writes one audit event** (`logEvent` already fans out to the `QCLogger` DO and
  R2 JSONL). A curation decision that cannot be reconstructed afterwards is not a decision.
- **Soft delete.** `deprecated: true` keeps the row readable for the next reader; hard deletes lose
  the evidence that an answer was ever believed.

---

## 8. Sustainability: what to measure, and what tells you it's failing

A self-populating KB dies in three ways: it fills with confident wrong answers, it drowns reviewers
in noise, or nobody votes because voting does nothing visible. Watch these:

| Metric | Healthy | Alarm |
|---|---|---|
| Revert rate within 30 days of ratification | < 2% | > 5% → evidence gate is too loose |
| Contested fraction of served answers | 5–20% | 0% → thresholds too loose (false certainty); > 30% → candidates are bad, not the voters |
| Median time to ratify | hours to days | weeks → review is the bottleneck, reduce human steps |
| Vote concentration (top voter's share of quorum) | < 20% | > 40% → one actor is steering topics |
| Ballot coverage (questions with ≥ 1 ballot) | rising | flat → votes invisible to voters, or no prompt to vote |
| Answered-from-curated share | rising | flat → self-population isn't working, it's just storage |

Also track **cost per ratified truth** (KV writes + DO writes + any verifier/LLM calls) and
**queue depth** for review. If the queue only grows, the promotion criteria are too strict for the
volume, and the honest answer is to raise the evidence bar *and* lower the volume of candidates, not
to auto-ratify more.

**Vote tally storage:** put ballots in a Durable Object keyed by question id
(`idFromName(normalizeId(id))`) so writes for one question serialise and counters are atomic, with
D1 as the escape hatch once you want ad-hoc queries across ballots (that is a real migration, not a
config change — don't do it in phase 1). Keep provisional rows as the 24 h render cache. Never tally
in R2 JSONL: that day-file rewrite is quadratic.

---

## 9. Phases

Each phase ships alone and leaves the system correct-but-limited if you stop there.

**Phase 0 — make honesty expressible (prerequisite, no new endpoints)**
Lower/remove the 0.9 floor in `clampMasterConfidence`; add `status` to `canonicalResponse`; keep
`candidates[1..]` instead of discarding them; decide who owns the `review:` prefix; document that
unauthenticated ballots are ranked-only. *Without this, scores cannot mean anything.*

**Phase 1 — collect ballots on real candidates**
Durable ballot store; `POST /api/truth/candidate/vote`; answer clustering (§3); tally exposed at
`GET /api/truth/candidate/:id/tally`. No promotion changes. Votes become visible, nothing moves.

**Phase 2 — surface disagreement**
`contested` responses with `alternatives` and `dissent` (§6). Still zero automatic promotion. This is
the phase that answers "give people the option" without giving anyone the power to overwrite truth.

**Phase 3 — ratify, with an evidence gate**
Quorum + margin + evidence gate; `promote` moved onto `updateMaster`'s version guard; audit events;
human confirmation required for `evidence == 2`. Revert path first, promotion second.

**Phase 4 — reduce the human step, carefully**
Auto-ratify only where a verifier signature exists (`evidence == 3`). Reputation earned from
observed outcomes (reverts, upheld disputes), never from volume. Still no auto-ratification of
`evidence == 2` — that is the line popularity must not cross.

---

## 10. Not doing

- No learned ranker. A model that decides which answer is true just relocates the problem, and you
  cannot audit it when it is wrong.
- No per-vote payments, karma economy, or vote farming incentives.
- No auto-demotion without a human confirm in phases 1–3.
- No new `layer` enum values; new capabilities arrive as additive fields or new routes.
- No ballot storage inside provisional rows (24 h TTL) or inside the R2 day-file audit.

---

## 11. Decisions I need from you

1. **Do anonymous votes count at all?** *Recommended:* yes for ranking, weight 0.1, never toward
   quorum. Otherwise a public site has no votes; otherwise a public site has brigading.
2. **Serve contested answers to end users?** *Recommended:* yes, labelled, confidence ≤ 0.55, with
   alternatives. A flagged plausible answer beats a miss.
3. **Who ratifies?** *Recommended:* verifier signature for factual claims; quorum-only for
   how-to/procedural answers, where "right" means "what works" and is cheaper to check.
4. **Do the 1,150 memory rows get to vote?** *Recommended:* no. They seed candidates and ordering
   only — they are already `confidence`-tagged by an unknown model, and importing that as agreement
   would let a model's self-assessment masquerade as a consensus.
5. **`review:` prefix ownership** — is `justice` or the `mind` console writing those 33 TTL'd keys,
   and does it become the review queue, or is it orphaned? Decide before Phase 3 adds a second one.

Threshold values in §5 are deliberately conservative and expected to move once real ballot data
exists. Wrong numbers are recoverable; a wrong evidence gate is not.
