# Architecture Options Comparison — PDF Generation Microservice

> Builds on `REQUIREMENTS_ANALYSIS.md`, `CRITICAL_ENGINEERING_REVIEW.md`, `CAPACITY_MATH.md`, and `CAPACITY_MATH_AUDIT.md`.  
> Options A–C are compared below; **§ Final Choice** records the selected design for the TranZact assignment.  
> No cloud queue/object-store brand is treated as mandatory.

**For each option this document covers:** shape, flows, reuse of Django/Postgres/Redis, capacity honesty, consistency/downloads/tamper, failures, cost vs **₹12,500**, and tradeoffs.

---

## Shared constraints every option must respect **[FACT]**

| Constraint | Value / rule |
|------------|----------------|
| Generation | **Puppeteer** + **HTML templates** |
| API shape | **HTTP** + **JSON** from primary service (PO / invoice data) |
| Modes | Single PDF; bulk **up to 100**; zip for bulk; progress on bulk |
| Peak (stated) | **~1000** single/min; **5–10** bulk/min |
| Render profile | **2–3 s**, **~400 MB** temporary |
| UX | Single downloads **start within 5 s** (meaning **ambiguous** — R1/R2/R3 in audit); bulk “reasonable time” |
| Hosted ERP today | Django monolith on **EC2 r6i.large (2 vCPU, 16 GB)**; **RDS db.r6i.large**; **Redis** cache |
| Integration | **Without re-architecting** the monolith |
| Hard problems | **500+** line OOM; flaky **50+** PDF downloads; bulk **consistency** (~87th PDF / ~3 min); **tamper evidence**; budget **₹12,500** |

**Capacity reminder (derived, not a vendor claim):** sustaining **1000** singles/min needs on average **~33–50** in-flight renders at 2–3 s (`CAPACITY_MATH.md`). At **~400 MB** additive baseline that is **~13–20 GiB** class temporary RAM for singles alone — in tension with a **16 GB** shared monolith and a **$150**/mo cap if peak is taken literally and continuously.

**Traffic reminder:** **~30,000 PDFs/month** and peak rates are **not** proven to be the same distribution (`CAPACITY_MATH_AUDIT.md`). Options must say which load they optimise for.

---

## Option map

| ID | Name | One-line idea |
|----|------|----------------|
| **A** | Minimal / shared-infra | Extend Django + Postgres + Redis; Puppeteer on or beside the same small footprint; lowest new spend |
| **B** | Decoupled async | Dedicated PDF API, durable queue, isolated workers, object storage |
| **C** | Hybrid by request class | Fast path for singles; async fan-out for large bulk |
| **D** *(extra)* | Payload-snapshot renderer | Same as B/C mechanically, but **require full JSON snapshots** so the PDF tier never live-reads ERP mid-bulk |

Option **D** is a **data-contract variant**, not a fourth hosting topology; it can apply to A–C.

---

# Option A — Minimal, low-cost (maximise existing Django / PostgreSQL / Redis)

## A.1 Shape

```
[ERP Users] → [Django monolith (EC2 r6i.large)]
                    ├─ PostgreSQL (job rows, hash metadata, optional snapshot JSON)
                    ├─ Redis (progress counters, short-lived locks, optional hot cache)
                    ├─ Local or small attached/object disk for PDF/ZIP bytes
                    └─ Puppeteer process(es) with a hard concurrency cap (c = 1..few)
```

**Appropriate reuse**

| Existing piece | Role in Option A |
|----------------|------------------|
| Django | Accept generate/download HTTP; auth session already exists; enqueue work; serve status |
| PostgreSQL | Job/batch tables; idempotency keys; tamper hash / signature metadata; optional embedded payload |
| Redis | Progress (`done/total`); rate limits; distributed lock so only N Puppeteer jobs run |
| EC2 | Run capped workers as management commands / supervisord — **only if** memory isolation is enforced |

**Not appropriate:** unbounded Chromium on the same 16 GB box as Django; using Redis as the only durable queue for multi-minute bulk without Postgres backup; infinite disk retention on the instance.

## A.2 Flows

**Single**

1. Primary/UI → Django `POST` with JSON (or ids — see consistency risk).  
2. Insert job row `queued`; Redis incr in-flight if under cap, else stay queued.  
3. Worker renders with Puppeteer → write file → store path + hash → `complete`.  
4. Client polls or redirects to download.

**Bulk**

1. Create `batch` + up to 100 child jobs in Postgres (not 100 Chromiums).  
2. Workers pull children under **global** concurrency semaphore.  
3. Redis/Postgres progress; when all terminal, build zip (or zip lazily).  
4. Download zip or per-file; retry download **without** re-render if bytes kept.

## A.3 How it addresses key problems (pattern-level)

| Concern | Approach in A |
|---------|----------------|
| Memory / OOM | Global `c_max` (start **1–3**); large-doc fail-fast; poison child status |
| 5 s start | Only plausible if queue empty and definition is **R1** or lucky **R3** with \(W≈2\) s; **cannot** honestly claim R3 at 1000/min on one 16 GB host |
| Consistency | Prefer **payload stored on job row at enqueue**; avoid live re-fetch per child |
| Flaky download | Persist files; re-mint download; optional per-PDF endpoints |
| Tamper evidence | Hash (and optional HMAC) in **Postgres**, bytes on disk — split custody vs editable sidecar only |
| Budget | Minimal new infra — best fit to **₹12,500** *if* peak is rare or shed |

## A.4 Cost posture

- **Incremental** cost can stay near zero–low (disk, maybe slightly larger instance).  
- **Risk cost:** one OOM / CPU storm **takes down ERP** — business cost outside the ₹ line item.  
- Fits **~30,000/month** average work; does **not** buy continuous **~1000/min**.

## A.5 Tradeoffs

| Pros | Cons |
|------|------|
| Least new moving parts; uses known Django/Postgres/Redis | Peak concurrency physics vs 16 GB |
| Fastest to ship for evaluation / low average load | Weak isolation; deploy coupled to monolith release train |
| Natural place for auth and “no re-architecture” | ZIP + Chromium I/O contend with ERP latency |
| Cheap under budget cap | Scaling = vertical or eventually forced toward B |

## A.6 What A cannot claim

- Sustained **~1000** singles/min with **R3 ≤ 5 s** on the stated monolith host.  
- Safe co-tenancy without cgroup/process limits (must be engineered, not assumed).  
- That Redis-alone durability equals a dedicated queue under crash.

## A.7 When A is appropriate

- Cost scenario (**~30k/month**) is the real driver and peak is bursty with **admission control**.  
- Team must ship with almost no new infra spend.  
- Product accepts **queueing / 503 / “try later”** at peak rather than silent unlimited accept.

---

# Option B — Decoupled asynchronous (dedicated PDF API + durable queue + isolated workers + object storage)

## B.1 Shape

```
[Django ERP] --HTTP JSON--> [PDF API service]
                               ├─ durable job queue (implementation TBD: DB queue, Redis streams, RabbitMQ, SQS-like, etc.)
                               ├─ metadata store (can be existing Postgres or separate)
                               ├─ [Puppeteer worker pool]  (isolated hosts/containers; hard c per host)
                               └─ [Object storage]  PDF/ZIP bytes + lifecycle rules
                                      ↑
[Users] download via signed/temporary URLs (or ERP proxy)
```

**Queue / storage:** choose explicitly after cost and ops skill — **not** “managed product X is mandatory.”

## B.2 Flows

**Single**

1. ERP posts snapshot JSON to PDF API → `202` + `job_id` (good fit for **R1 ≤ 5 s**).  
2. Message durable in queue; worker renders; PUT object; metadata `complete` + hash.  
3. Client polls status; GET download URL when ready (**R3** may be 2–3 s + wait).

**Bulk**

1. API creates batch; enqueues **N child messages** (N≤100) or one message expanded by a scheduler.  
2. Workers process children with **fleet-wide** concurrency limit.  
3. Progress via API (Redis/cache optional); zip object built when policy says (end or incremental).  
4. Downloads hit object storage; retries re-use objects.

## B.3 Existing infra use

| Piece | Role |
|-------|------|
| Django | Thin client: gather data, call PDF API, show progress UI — **no Chromium** |
| PostgreSQL | Optional shared metadata; or left for ERP only |
| Redis | Optional progress/cache; **not** the sole durability story if using true durable queue |

Monolith is **not** re-architected into microservices; it gains a dependency (acceptable “integration” if scoped as HTTP client).

## B.4 Key problems

| Concern | Approach in B |
|---------|----------------|
| Memory | Workers sized/capped independently of ERP 16 GB |
| Peak | Scale worker count toward \(C \approx L\) **only if budget allows**; else queue + backpressure to ERP |
| Consistency | API stores request payload / batch snapshot id with job — workers are pure renderers |
| Flaky net | Object storage + TTL + re-mintable URLs; per-object fetch beats one-shot zip |
| Tamper | Hash/signature in metadata DB; objects in store with restricted write IAM / WORM-ish policy if needed |
| OOM | Child fails terminal after classified crash; no infinite retry |

## B.5 Cost posture

- **Highest** structural spend (always-on or burst workers + storage + queue).  
- Aligns with **isolation** and peak *headroom*, fights **₹12,500** if you provision for continuous 1000/min (\(C~40+\), multi-GB RAM).  
- Viable under budget if sized for **average + short bursts** and **shed/queue** beyond that — same honesty requirement as A, better failure isolation.

## B.6 Tradeoffs

| Pros | Cons |
|------|------|
| ERP survives PDF OOMs | More components to operate |
| Clear service boundary for Puppeteer mandate | Latency of pure async hurts **R3** unless sync fast-path added (→ hybrid) |
| Natural place for lifecycle, idempotency, scale-out | Cost and complexity can violate “don’t overcomplicate” / budget |
| Progress and bulk fan-out are straightforward | Duplicate “source of truth” if metadata diverges from ERP |

## B.7 What B cannot claim

- That “we added a queue” meets **5 s R3** at peak (`CRITICAL_ENGINEERING_REVIEW` §10).  
- That isolation automatically fits **₹12,500** at literal peak concurrency.  
- Exactly-once effects without idempotent job keys.

## B.8 When B is appropriate

- Peak behaviour and ERP uptime matter more than minimal parts count.  
- Bulk and retries are first-class.  
- Team can operate queue + object storage within budget **at chosen C**, not fantasy C=125.

---

# Option C — Hybrid (singles ≠ large bulk)

## C.1 Shape

Combines A/B ideas by **request class**:

```
                    ┌─ Single / small job ─► [low-latency path]
[Django or PDF API] ┤                         (sync or priority queue, reserved slots)
                    └─ Bulk (e.g. >K docs or always) ─► [async fan-out path]
                                                      durable children, progress, zip, long TTL
```

**Policy knobs (must be explicit assumptions):**

| Knob | Example (not FACT) |
|------|---------------------|
| Threshold \(K\) | Bulk HTTP always async; or \(N\ge 10\) → async |
| Reserved slots | e.g. keep \(c_{\mathrm{single}}\) reserved; bulk uses leftover |
| Single path | Inline wait up to \(T_{\mathrm{sync}}<5\) s then fallback async |
| Bulk path | Never blocks ERP worker threads on 100× render |

Hosting can be **shared** (A-like) or **dedicated workers** (B-like); hybrid refers to **control flow**, not a mandatory third cluster.

## C.2 Flows

**Single (fast path)**

1. Enqueue with **high priority** or attempt sync render if `in_flight < c_reserved`.  
2. If finished within budget → return download (**R3** possible when load low).  
3. If not → return `202` + poll (**R1** success, R3 deferred) — **must document**.

**Bulk (slow path)**

1. Accept batch immediately; persist payloads; enqueue children at **low priority**.  
2. Progress updates; partial results downloadable; zip when complete or on demand.  
3. Never schedule 100 parallel renders.

## C.3 Key problems

| Concern | Hybrid stance |
|---------|----------------|
| 5 s | Optimises **probability** that **interactive** singles meet R3; bulk explicitly excluded from 5 s |
| Peak | Admission: reject/defer bulk first; protect singles |
| Consistency | Same snapshot rule on both paths |
| Flaky download | Focus durability investment on bulk artifacts |
| Cost | Can keep small worker pool if bulk is elastic/overnight |

## C.4 Cost posture

- Better **value** than B-at-full-peak: pay for reserved single slots + modest bulk drain.  
- Still cannot conjure 33–50 concurrent Chromiums from ₹12,500 without measurement.  
- Matches the assignment’s UX split (“singles start in 5 s” vs bulk “status + reasonable time”).

## C.5 Tradeoffs

| Pros | Cons |
|------|------|
| Matches stated UX asymmetry | Two paths = more bugs (priority inversion, dual code) |
| Protects interactive users from bulk storms | Unfairness if “single” API abused to fake bulk |
| Works atop A or B infrastructure | Sync path can still wedge the API if \(c_{\mathrm{reserved}}\) too high |
| Clearer SLO storytelling in APPROACH.md | Need careful definition of R1 vs R3 on fallback |

## C.6 What C cannot claim

- That hybrid **removes** Little’s Law; it **allocates** scarce \(C\).  
- Singles always ≤5 s R3 when \(\lambda\) exceeds reserved \(\mu\).  
- Simplicity equal to Option A.

## C.7 When C is appropriate

- Product agrees singles are latency-critical and bulk is throughput/progress-critical.  
- Peak mix includes both classes (whether or not they overlap is still an assumption).  
- Strong candidate for the **submission design narrative** even if hosting stays minimal.

---

# Option D *(optional variant)* — Snapshot / payload-first contract

Applies on top of A, B, or C.

**Rule:** At accept time, the primary service sends **all fields needed to render** (or a frozen snapshot id resolved once). Workers **must not** `SELECT` live invoice rows while rendering child 87 three minutes later.

| Pros | Cons |
|------|------|
| Directly answers GST stale/mixed bulk incident | Large bulk POST bodies (100 × JSON) |
| PDF tier stays a pure renderer — less ERP coupling | ERP must assemble snapshots (small additive work, not full re-architecture if done via existing serializers) |
| Tamper hash covers exact bytes of agreed input version | Stale-by-policy: user edits after click won’t appear — **must be the accepted consistency model** |

---

## Side-by-side comparison

| Dimension | A Minimal | B Decoupled | C Hybrid |
|-----------|-----------|-------------|----------|
| New components | Few | Many | Medium (policy + maybe B’s parts) |
| ERP isolation from OOM | Weak unless cgroup | Strong | Strong if workers isolated; weak if sync on monolith |
| Fit to ₹12,500 at **30k/mo avg** | Best | Tight | Good |
| Fit to literal **1000/min** sustained | Poor | Possible only with spend/C | Same physics; better **triage** |
| Single **R3 ≤ 5 s** | Only if idle / small W | Needs priority + enough C | **By design** attempts this |
| Bulk progress / zip / retry | DIY on Django | Natural | Natural on bulk path |
| Consistency story | Easy if payload-on-row | Easy if payload-on-job | Same + D |
| Ops complexity | Low | High | Medium–high |
| “Don’t re-architect monolith” | Best alignment | OK as HTTP client | OK |
| Overcomplicating risk | Low | High | Medium |

---

## Failure matrix (all options)

| Failure | A | B | C |
|---------|---|---|---|
| Worker / Chromium crash | Can starve Django if co-hosted; job row must go `failed`/`retry` | Queue redelivery + idempotent object key | Same as host model; protect single reserved slots from poison bulk |
| Queue loss | If Redis-only: **job loss** — prefer Postgres jobs | Durable queue + replay | Bulk on durable path; singles may be sync-only loss window |
| Object/disk full | Downloads break; need TTL | Lifecycle policies | Hot TTL on bulk zips |
| Client drop mid-zip | Re-download if persisted; else user re-clicks generate (**bad**) | Re-mint URL | Emphasise per-PDF + zip rebuild from objects |
| Overload \(\lambda>\mu\) | Must admit-control | Same | **Shed bulk first** |

---

## Honest capacity / SLA posture by option

| Option | Reasonable claim | Unreasonable claim |
|--------|------------------|--------------------|
| A | Cheap correct behaviour for **average** 30k/mo with cap \(c\ll L_{\mathrm{peak}}\) and visible backoff | Full peak + R3 5 s on one r6i.large |
| B | Isolated rendering; durable bulk; scale \(C\) until money/RAM end | “Async queue ⇒ 5 s downloads at 1000/min” |
| C | Interactive singles preferred; bulk progressive | Singles immune to peak math |
| D | Bulk documents share one accepted snapshot | “Always latest ERP data” and “stable bulk” at once |

---

---

# Final choice (TranZact assignment)

## Decision

**Chosen architecture: Option C (Hybrid) + Option D (snapshot payloads), on a lean footprint that reuses Django / PostgreSQL / Redis and adds only what Puppeteer isolation and flaky downloads require.**

| Layer | Choice | Why |
|-------|--------|-----|
| Request handling | **Hybrid (C)** | Assignment already splits UX: singles ≈5 s vs bulk progress + zip |
| Data contract | **Snapshot JSON at accept (D)** | Fixes GST mid-bulk drift without live re-reads |
| API edge | **Django** (thin endpoints / client to workers) | No re-architecture; no second public API fleet |
| Job durability | **PostgreSQL job/batch tables** | Already paid for; survives Redis flush; simple idempotency |
| Progress / locks | **Redis** | Cheap counters + concurrency semaphore |
| Render | **1 small isolated worker** (separate process/host/container), Puppeteer, **hard cap \(c=2\)–\(3\)** | Keeps ~400 MB × N off the monolith’s 16 GB; stays inside budget |
| Bytes | **Object storage** (or equivalent durable blob store) + short hot TTL | Mid-download retry without regenerate; zip rebuild from objects |
| Queue product | **Postgres-backed claims** (SELECT … FOR UPDATE SKIP LOCKED or equivalent) | Avoids a dedicated broker until volume proves need — anti-overengineering |

**Rejected as the primary pick**

| Option | Why not primary |
|--------|-----------------|
| **A alone** | Co-locating uncapped (or loosely capped) Puppeteer on the **r6i.large** risks ERP OOM; weak story for bulk retry and isolation |
| **B full** | Dedicated PDF API + durable broker + multi-worker fleet sized toward continuous **~1000/min** overruns **₹12,500** and overcomplicates a take-home |
| Pure sync everything | Bulk of 100 × 2–3 s cannot meet interactive expectations; blocks workers |

**Working assumptions for this choice** (declared, not assignment facts):

1. **5 s “download start”** for singles = **best-effort time-to-first-PDF-byte when the single lane is free (R3)**; under overload return **202 + poll** quickly (**R1**) rather than melting the host.  
2. **~30,000 PDFs/month** drives **cost sizing**; **~1000/min** is a **peak** the system absorbs with **queue + admission**, not with always-on \(C\approx 40\)–\(50\) Chromiums.  
3. Bulk consistency = **all children render from payloads frozen at batch accept**.  
4. Puppeteer **does not** share unconstrained RAM with Django.

---

## Why this wins on the stated criteria

| Criterion | How the choice fits |
|-----------|---------------------|
| **₹12,500/month** | No always-on large worker fleet; reuse RDS/Redis; one small worker + object storage + lifecycle beats full Option B |
| **~1000 single/min** | Honest: cannot sustain with \(c=2\)–\(3\) (\(L\) needs ~33–50). Hybrid **prioritises singles**, **sheds/defer bulk**, **backpressures** when `queue_age` or depth exceeds limit — practical for assignment + budget |
| **~400 MB Puppeteer** | Hard concurrency cap; isolated worker; large-table jobs fail graded (no retry storm) |
| **Bulk downloads** | Per-PDF objects durable; zip built from objects; re-download ≠ re-render; progress in Redis/Postgres |
| **Data consistency** | Snapshot payloads (D) — child #87 cannot see a newer ERP row than child #1 |
| **Avoid overengineering** | No extra microservice mesh, no mandatory cloud queue brand, no multi-region, no sync+async dual clusters |

---

## Target shape (practical)

```
[Users / ERP UI]
       │
       ▼
[Django monolith]  ←── auth, assemble snapshot JSON, enqueue, status UI
       │
       ├── PostgreSQL: batches, jobs, payload JSON, status, content hash
       ├── Redis: progress, global in-flight semaphore, short-lived locks
       │
       ▼
[Worker (isolated)]  ←── claim jobs (single lane priority > bulk children)
       │                 Puppeteer HTML→PDF, c ≤ 2..3
       ▼
[Object storage]     ←── pdf/{job_id}.pdf ; zip/{batch_id}.zip (optional, TTL)
       │
       ▼
[Download]  signed/temporary URL or Django-proxied stream (re-mintable)
```

**Concurrency policy (initial):**

| Lane | Cap | Behaviour |
|------|-----|-----------|
| Single | Prefer up to \(c_{\mathrm{single}}\) of the global \(c\) | Try render path aimed at ≤5 s when slots free |
| Bulk child | Uses remaining slots only | Always async; progress; never `Promise.all` 100 browsers |
| Global | \(c_{\mathrm{max}} = 2\)–\(3\) until measured | Scale only after RSS/`W` + cost check |

---

## Request flows

### Single document

```
1. Client/ERP → Django POST /pdf/single
   Body: full render snapshot JSON (D) + idempotency key
2. Django inserts job (status=queued|rendering), stores payload + optional input hash
3. If single-lane slot free:
     Worker claims → Puppeteer (2–3 s) → PUT object → status=complete
     + store SHA-256 (and optional HMAC) in Postgres
     → respond with download URL when ready  (target: R3 within ~5 s if idle)
   Else:
     Return 202 + job_id immediately (R1)
     Client polls GET /pdf/jobs/{id} until complete → download URL
4. Download: GET bytes from object storage (or proxy). Retry download does not re-render.
5. Overload: if predicted wait > threshold → 503/429 with Retry-After (admission control)
```

### Bulk (up to 100)

```
1. Client/ERP → Django POST /pdf/bulk
   Body: array of up to 100 snapshot JSON docs (same freeze time) + batch idempotency key
2. Django creates batch + N child jobs (payload per child), status=queued
   Return 202 + batch_id immediately
3. Workers claim children only when single-lane demand allows (bulk shed-first under pressure)
   Each child: render → PUT pdf → hash in Postgres → progress++ in Redis/Postgres
4. Client polls GET /pdf/batches/{id} → {done, total, failed[], ready_pdf_urls[]}
5. When all terminal (or on demand): build zip from existing objects → PUT zip
   (Prefer also exposing per-PDF URLs so a 50+ file zip need not be the only path)
6. Failed child (e.g. 500+ line OOM): status=failed_oom, no blind infinite retry;
   batch can be partial_success per explicit policy
7. Mid-download failure: re-fetch zip or per-PDF URLs; regenerate only if hot TTL expired
```

### Priority under load (hybrid essence)

```
on_claim:
  prefer job_type=single AND age < 5s budget
  else claim oldest bulk child
on_admit:
  if queue_wait_estimate > limit: reject new bulk first, then singles with backoff
```

---

## Key tradeoffs accepted

| We accept | In exchange for |
|-----------|-----------------|
| Not provisioning \(C\approx 40\) Chromiums for continuous 1000/min | Staying under **₹12,500** and not melting the ERP host |
| Singles may fall back to **202 + poll** at true peak | Host stability + predictable memory |
| Bulk is slower when singles are busy | Interactive path stays usable longer |
| Snapshot may omit edits made *after* click | Stable GST set across the batch (~3 min window) |
| Postgres-as-queue is enough at 30k/mo | Skipping broker ops cost/complexity |
| One isolated worker to start | Simpler ops; scale-out later is a measured gate |
| Hash (+ optional HMAC) in Postgres, not a full PKI/KMS design | Tamper evidence without overcomplicating |
| Short hot TTL on objects | Budget; archive only what compliance marks (when defined) |

---

## What we explicitly do **not** claim

- Continuous **~1000/min** with **R3 ≤ 5 s** for every single on day one.  
- That a queue alone creates capacity.  
- That co-locating uncapped Puppeteer on the monolith is safe.  
- Perfect success for every **500+** line PDF without a graded failure mode.

---

## vs A / B in one line each

- **Better than A:** Chromium isolated; object durability for flaky SMB downloads; clear single vs bulk lanes.  
- **Leaner than B:** No mandatory separate PDF API fleet or managed broker; Django + Postgres + Redis do accept/status/progress.  
- **Still hybrid:** Singles prioritised; bulk async with progress and zip/per-file download.

---

## Next implementation artefacts (when needed)

1. Architecture diagram matching the “Target shape” above.  
2. Cost line items under ₹12,500 for: small worker, object storage + egress, negligible Redis/RDS marginal.  
3. Load-test plan from `CAPACITY_MATH.md` before raising \(c_{\mathrm{max}}\).

---

## Assumptions checklist (locked for this choice)

| # | Locked decision for submission design |
|---|----------------------------------------|
| 1 | 5 s = best-effort **R3** when single lane free; else fast **R1** |
| 2 | Peak 1000/min handled by **queue + admit**, not full parallel fleet |
| 3 | 30k/month informs **cost**; peak informs **backpressure** design |
| 4 | Consistency = **snapshot-at-accept (D)** |
| 5 | Initial \(c_{\mathrm{max}}=2\)–\(3\) on isolated worker |
| 6 | Puppeteer **off** the unconstrained monolith RAM pool |
| 7 | Hot artifact TTL (e.g. days, not years); archive optional/separate |
| 8 | Queue = **Postgres job claims**; Redis for progress/semaphore only |

---

*End of document. Chosen design: **Hybrid (C) + Snapshot (D)**, lean hosted — practical under budget, honest about peak math, strong on bulk consistency and download retry.*
