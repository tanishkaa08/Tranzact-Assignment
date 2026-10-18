# PDF Generation Microservice — Critical Engineering Review

> Companion to `REQUIREMENTS_ANALYSIS.md` and the TranZact Round-1 assignment PDF.  
> **Purpose:** stress-test the requirement surface as a senior backend engineer.  
> **Not an architecture.** No queue product, hosting shape, or final component graph is chosen here.

**Discipline**

- **[FACT]** = stated in the assignment  
- **[DERIVED]** = engineering consequence of stated facts (math / failure physics), not a product requirement  
- **[ASSUMPTION NEEDED]** = must be declared explicitly before a design can resolve a conflict  
- **[CONFLICT]** = two or more stated goals that cannot all be maximized without tradeoffs

---

## 0. Executive Tension (Read This First)

The assignment simultaneously asks for:

| Goal A | Goal B | Why they pull apart |
|--------|--------|---------------------|
| Peak **~1000 single requests/min** | **~400MB** temp memory per PDF + **2–3 s** render | Memory × concurrency caps parallel work far below unconstrained peak |
| Single download **start within 5 seconds** | Same peak + shared workers with bulk (**5–10 × up to 100**) | Queueing required for memory safety delays singles |
| Puppeteer mandated | **500+ line items** frequently **OOM** | The mandated tool is known-unstable on the worst documents |
| Bulk zip download | Unreliable links; **50+** PDFs fail midway | Large atomic downloads fight flaky networks |
| Live ERP data (primary service + DB) | Bulk consistency over **~3 minutes** (87th PDF anecdote) | Re-reading mutable state during a long job yields mixed documents |
| Archive for compliance | Monthly cost **under ₹12,500 (~$150)** at **~30,000 PDFs/month** | Indefinite retention of **100KB–1MB** objects burns the budget |
| Tamper evidence | “Without overcomplicating” | Cryptographic trust needs a trustworthy reference; naive hashing is incomplete |
| Integrate without re-architecting monolith on **r6i.large (16GB)** | Heavy Chromium workers | Co-locating workers risks taking down the ERP; splitting costs money |

None of these conflicts means the problem is unsolvable. They mean **every viable design must pick losers** and document the **[ASSUMPTION NEEDED]** that makes the pick acceptable.

---

## 1. Investigations (1–12)

### 1. Why expected throughput cannot automatically be handled by a single Puppeteer process

**[FACT]** Peak ≈ **1000 single requests/min** ⇒ ≈ **16.7 requests/s**. Each generation takes **2–3 s**.

**[DERIVED]** Even ignoring memory: one in-process render that is busy for 2.5 s average can finish only ≈ **0.4 PDFs/s** (≈ **24/min**). Peak demand is roughly **40×** that rate. Puppeteer drives Chromium; a “single Puppeteer process” typically means one Node controller with limited useful parallelism via pages/browsers, each still paying CPU and RAM.

**Failure scenario:** All peak traffic hits one process; event loop blocks on long renders; Chromium tabs pile up; process OOMs or becomes unresponsive.

**Why naive fails:** “HTTP handler → `page.pdf()` inline” assumes request rate ≤ 1 / render_time.

**Impact:** Timeouts, 5xx, ERP users cannot download POs/invoices at month-end peak.

**Strategies / tradeoffs**

| Strategy | Upside | Tradeoff |
|----------|--------|----------|
| Horizontal/vertical worker pool with hard concurrency cap | Matches throughput physics | Cost; need orchestration |
| Queue + limited in-flight renders | Protects memory | Adds latency (threatens 5 s) |
| Multiple pages in one browser | Less cold-start than N browsers | Shared-process blast radius; still RAM-bound |
| Reject / shed load above capacity | Protects core ERP | Bad UX unless backlog + ETA |

**Validate:** Load test at 1000/min synthetic; measure completions/s, p95 time-to-ready, OOM rate; prove single-process ceiling with a controlled A/B (1 worker vs N).

---

### 2. Why 400 MB per render is not necessarily the full memory footprint of a browser worker

**[FACT]** Each PDF generation “consumes **~400MB memory temporarily**.”

**[DERIVED]** That figure is a *generation* cost, not a complete process budget. Additional footprints typically include:

- Chromium base process + GPU/utility processes (often tens–hundreds of MB before any page)
- Node/Puppeteer controller RSS
- HTML DOM for **500+** row tables (can spike *above* the “typical” 400 MB)
- PDF buffer / stream copies in userland
- Zip assembly buffers for bulk
- Fragmentation: RSS may not return to baseline immediately after “temporary” use
- Concurrent pages sharing one browser: not strictly `N × 400MB`, but also not `400MB` total

**Failure scenario:** Capacity plan assumes `floor(RAM / 400MB)` concurrent jobs; real host OOMs earlier; kernel kills Chromium mid-render (looks like the assignment’s crash case).

**Why naive fails:** Treating 400 MB as both average and peak, and as the only consumer on the box.

**Impact:** Underestimated worker count → missed SLAs; overestimated concurrency → cascading OOMs.

**Strategies / tradeoffs**

| Strategy | Upside | Tradeoff |
|----------|--------|----------|
| Measure RSS/PSS under prod-like templates (5 vs 500+ rows) | Real ceiling | Time; template-dependent |
| Size for p95/p99 memory, not mean | Stability | Fewer slots → more queueing |
| Isolate one browser per job | Failure containment | Higher base overhead |
| Browser reuse with page recycle | Amortize base cost | Leak risk; noisy neighbor |

**Validate:** `cgroup`/container memory limits; prometheus RSS during render; force 500+ line fixtures; confirm limit kills are detected as graceful failures.

---

### 3. Why average request rates and peak bursts must be distinguished

**[FACT]** Peak: **~1000 single/min**, **5–10 bulk/min**. Also: approximately **30,000 PDFs/month** for the cost scenario.

**[DERIVED]** Uniform average for 30k/month ≈ **30,000 / (30 × 24 × 60) ≈ 0.7 PDFs/min** — three orders of magnitude below peak single rate. Peak and monthly volume describe different stress axes: **latency/concurrency** vs **cost/storage**.

**Failure scenario:** Design sized for monthly average (tiny VM) collapses at peak; or design sized for continuous 1000/min forever blows **₹12,500**.

**Why naive fails:** “30k/month is small, so a single small instance is fine” or “we need 1000 concurrent Chromiums always.”

**Impact:** Either chronic peak outages or chronic budget overrun.

**Strategies / tradeoffs**

| Strategy | Upside | Tradeoff |
|----------|--------|----------|
| Size compute for peak *burst length* (unknown — **[ASSUMPTION NEEDED]**) + queue for overflow | Meets 5 s for most of burst | Need burst-duration assumption |
| Soften peak with async + “start” = accepted+URL when ready only if queue short | Honest UX | May miss literal 5 s under sustained peak |
| Priority: singles over bulk during peak | Protects interactive path | Bulk “reasonable time” suffers |
| Cost model on average + peak headroom as % of budget | Budget realism | May require declaring 5 s best-effort at extreme peak **[CONFLICT]** |

**Validate:** Traffic shape sensitivity: 1-minute vs 15-minute peak plateaus; queue depth vs 5 s start; cost of peak capacity idle 95% of month.

---

### 4. Why a bulk request should not necessarily render all PDFs simultaneously

**[FACT]** Bulk: **up to 100 PDFs**/request; **5–10** bulk/min; each render **~400MB**, **2–3 s**.

**[DERIVED]** 100 simultaneous renders ⇒ on the order of **~40 GB** temporary if 400 MB were additive — absurd vs **16GB** monolith host and vs **$150**/mo. Even 10 parallel bulk jobs × 100 would be catastrophic. Bulk should be a *fan-out of queued work*, not a *fork-bomb*.

**Failure scenario:** One bulk request spawns 100 Chromiums; host OOMs; all singles fail; possibly Django on same host dies.

**Why naive fails:** `Promise.all(docs.map(render))` inside the bulk HTTP handler.

**Impact:** One customer’s bulk export takes down PDF service (or ERP). Progress jumps 0→crash.

**Strategies / tradeoffs**

| Strategy | Upside | Tradeoff |
|----------|--------|----------|
| Bulk = parent job + child jobs with global concurrency semaphore | Stable memory | Longer wall-clock |
| Cap bulk child parallelism (e.g. 1–3) | Predictable | “Reasonable time” needs definition |
| Fair-share across tenants/jobs | Noisy-neighbor control | Complexity |
| Pre-zip only as children complete / zip at end | Memory vs time trade | Zip-at-end hurts flaky downloads |

**Validate:** Submit 10 concurrent bulks of 100 under a concurrency cap; assert max RSS; assert singles still make progress.

---

### 5. Why simply retrying failed jobs can cause duplicate work or incorrect status

**[FACT]** OOM mid-render is frequent for large docs; downloads fail midway; concurrent load is high — failures and retries are normal.

**[DERIVED]** Retries without idempotency keys / generation tokens produce: double PDFs, double billing of CPU toward budget, zip containing duplicates, status flapping (`failed` → `running` → `success` with different bytes), or compliance archive with multiple “the” invoice.

**Failure scenario:** Worker crashes after PDF uploaded but before ack; queue redelivers; second render; client saw `failed` then later `ready` with different hash; tamper/compliance metadata mismatched.

**Why naive fails:** “If error, retry 3 times” with no job identity, no “exactly-once effect,” no status state machine.

**Impact:** Wrong GST invoice version archived; user downloads mixed set; cost amplification under OOM storms.

**Strategies / tradeoffs**

| Strategy | Upside | Tradeoff |
|----------|--------|----------|
| Idempotent job ID (bulk_id, doc_id, payload_hash) | Safe retries | Must define identity |
| At-least-once queue + compare-and-swap status transitions | Correct observability | More state logic |
| Separate “render attempt” from “artifact committed” | Clear semantics | Extra storage metadata |
| Limited retries + poison queue for OOM-class errors | Stops retry storms | Need user-visible permanent failure |

**Validate:** Kill worker after upload before ack; assert one artifact; assert status monotonicity; assert progress counters do not double-count.

---

### 6. Why temporary download URL, job status, and generated artifact need clear lifecycle rules

**[FACT]** Users download singles/zips; bulk needs progress; unreliable mid-download; archive *may* be needed; budget capped.

**[DERIVED]** Three clocks exist and diverge if undefined: (1) job status TTL, (2) object/blob TTL, (3) signed URL expiry. Mid-download failure requires artifacts to outlive the first URL. Compliance archive may require a *different* lifecycle than “temporary download.”

**Failure scenario:** Status says `complete` but object already deleted; or URL expired while zip still uploading; or unlimited objects retained → budget breach; or status `running` forever after worker death.

**Why naive fails:** “Store file, return link” with no expiry, no GC, no distinction between working-set and compliance archive.

**Impact:** Broken retries for flaky SMB links; storage cost creep; support burden; false compliance confidence.

**Strategies / tradeoffs** *(patterns only — not a chosen design)*

| Concern | Needs a rule for | Tension |
|---------|------------------|---------|
| Job status | Terminal states, retention for UI/history | Long history vs Redis/DB cost |
| Artifact bytes | Hot TTL vs archive storage class | Retry window vs ₹12,500 |
| Download URL | Expiry, re-mint without regenerate | Security vs flaky-net UX |
| Bulk zip | Ephemeral packaging vs durable per-PDF objects | Zip rebuild cost vs storage |

**[ASSUMPTION NEEDED]** Retry window (e.g. hours vs days); whether regenerate-on-miss is allowed; archive retention.

**Validate:** Expire URL, re-issue, download succeeds without new render; GC deletes hot objects after TTL; archived objects excluded from GC; orphan detection after worker crash.

---

### 7. Why fetching each invoice independently during a bulk job can produce inconsistent documents

**[FACT]** Customer report: GST invoice PDF showed **stale data** because ERP data was **updated between trigger and the 87th PDF** (**~3 minutes** later). Design must handle **data consistency for bulk operations**.

**[DERIVED]** If each child render does `GET invoice N` (or SQL read) at render time Tₙ, then invoice 1..86 may reflect version V₀ and 87..100 version V₁ — or worse, partial field updates. “Stale” in the ticket may mean “not what I just edited” *or* “not consistent with sibling PDFs”; both hurt trust in GST paperwork.

**Failure scenario:** Accountant bulk-exports 100 invoices while ops edits tax fields; zip contains mixed tax regimes; auditor rejects set.

**Why naive fails:** Bulk API only passes IDs; workers hydrate live from PostgreSQL/Django per document.

**Impact:** Compliance and customer trust failure; silent, hard-to-reproduce.

**Strategies / tradeoffs**

| Strategy | Upside | Tradeoff |
|----------|--------|----------|
| Primary service sends **full JSON snapshot** per doc in the bulk request | Clear point-in-time; PDF service is pure renderer | Large payloads (100 × invoice JSON) |
| Snapshot/freeze rows at bulk enqueue (version or temp snapshot tables) | Consistent set | Touches ERP data plane — re-architecture risk **[CONFLICT]** with “don’t re-architect monolith” |
| Record `as_of` timestamp + read historical rows | Auditable | Requires history (may not exist) |
| Accept read-anytime + stamp `data_version` on each PDF | Honest | Does not prevent mixed bulk |

**[ASSUMPTION NEEDED]** Consistency contract: *all-or-nothing snapshot at trigger* vs *per-document latest at render* vs *payload-only*.

**Validate:** Mutation injected at child #50 during bulk; assert all PDFs share same snapshot marker **or** assert documented per-doc versions — matching the chosen contract.

---

### 8. Why a SHA-256 hash alone proves integrity only when the expected hash itself is trustworthy

**[FACT]** PDFs must be **tamper-evident**; prove **not modified after generation**; without overcomplicating.

**[DERIVED]** `sha256(pdf)` detects change *relative to a reference hash*. If the attacker who alters the PDF can also alter the stored hash (same DB/bucket/policy), verification always passes. Hash-without-separate-trust-anchor = checksum for accidental corruption, not strong tamper evidence.

**Failure scenario:** Object overwritten in storage; sidecar `.sha256` overwritten to match; compliance “verifies” successfully.

**Why naive fails:** “We store SHA-256 next to the file” as the entire answer.

**Impact:** False assurance to compliance team; failed audits.

**Strategies / tradeoffs**

| Strategy | Upside | Tradeoff |
|----------|--------|----------|
| Hash stored in **WORM / append-only / separate custody** from object bytes | Stronger evidence | Slightly more moving parts |
| Sign hash with key only compliance/ops hold (HMAC or asymmetric) | Proves origin + intactness | Key management |
| Monolith DB row with hash + object in bucket; different IAM | Practical split | Not perfect if both compromised |
| Publish hash to customer at generation time (email/UI) | User-held reference | UX; loss of email |

**Validate:** Modify PDF bytes without updating reference → verify fails; modify both in same trust domain → document residual risk honestly in APPROACH.md.

---

### 9. Why large HTML tables can cause browser memory exhaustion

**[FACT]** Dynamic tables **5 to 500+** line items; **500+** frequently causes Puppeteer **OOM mid-render**.

**[DERIVED]** Chromium builds a DOM + layout + often raster/skia state. A table with hundreds of rows, cells, borders, and repeated styles is not “500 small strings” — it is thousands of layout objects. `page.pdf()` may clone/layout the full document. Memory scales worse than O(rows) in practice (styles, fonts, images). The assignment’s **~400MB typical** is not a cap for pathological docs.

**Failure scenario:** Worker RSS climbs during layout; cgroup OOM killer; partial PDF or no artifact; job stuck if not detected.

**Why naive fails:** One HTML template for all sizes; no preflight on row count; retry loops on the same HTML.

**Impact:** Permanent failure for largest POs; retry storms; capacity collapse.

**Strategies / tradeoffs**

| Strategy | Upside | Tradeoff |
|----------|--------|----------|
| Preflight row-count thresholds → fail fast / split PDF / simplify template | Predictability | Product decisions needed |
| Stream/chunk tables across pages with lighter DOM | May reduce peak RAM | Template complexity; still Puppeteer |
| Isolate large jobs on restricted concurrency | Protect pool | Slower large docs |
| Detect OOM vs logic errors; no blind retry | Stability | Need crash classification |

**Validate:** Fixtures at 5, 100, 500, 1000 rows; measure peak RSS; confirm graceful status `failed_oom` without worker death loops.

---

### 10. Why a queue alone does not guarantee a five-second single-document download-start target

**[FACT]** Users expect **single downloads to start within 5 seconds**.

**[DERIVED]** A queue provides *absorption* and *decoupling*, not *latency*. Time-to-start ≈ enqueue delay + wait_in_queue + render_time(2–3s) + persist + URL issue. If wait_in_queue ≫ 0 under peak, **5 s is impossible** even with a perfect queue. Also: if “start” means first byte of PDF, then **2–3 s render alone** consumes most of the budget — only **2–3 s** left for queueing and I/O.

**Failure scenario:** Peak backlog 30 s deep; every single waits; product claims “we queued it” while missing the expectation.

**Why naive fails:** “Put jobs in Redis/SQS → architecture is scalable” conflates throughput shaping with latency SLO.

**Impact:** Interactive single-doc UX fails exactly when business is busiest.

**Strategies / tradeoffs**

| Strategy | Upside | Tradeoff |
|----------|--------|----------|
| Define “start” precisely (see conflicts) | Evaluable | May need product buy-in |
| Reserve capacity / priority lane for singles | Protects 5 s | Bulk slower |
| Cap admit rate; return busy with ETA | Honesty | Not silent success |
| Sync path only when queue depth = 0 | Fast happy path | Two code paths |

**[ASSUMPTION NEEDED]** Meaning of “downloads to start”; whether 5 s holds at full stated peak or at “typical peak after admission control.”

**Validate:** Queue-depth SLI; p50/p95 time-to-ready under 1000/min; show when 5 s is mathematically infeasible given 2–3 s render.

---

### 11. Why storing all generated PDFs indefinitely could violate the budget

**[FACT]** PDFs **100KB–1MB**; **~30,000/month**; cost **under ₹12,500 (~$150)**; archive **may** be needed; flaky downloads encourage keeping artifacts for retry.

**[DERIVED]** Order-of-magnitude storage: 30k × 0.1–1 MB ≈ **3–30 GB/month** *new*. Indefinite retention → linear growth (year 1: tens–hundreds of GB) plus egress when re-downloading zips after mid-transfer failure. Object storage is cheap per GB but not free next to a **$150** all-in cap that must also pay for Puppeteer compute. Compliance “keep forever” without cold tiering competes with render capacity money.

**Failure scenario:** Hot bucket grows unbounded; bill exceeds cap; emergency deletes break compliance.

**Why naive fails:** S3/minio without lifecycle; treating archive = same store as CDN downloads.

**Impact:** Budget breach or sudden purge risk.

**Strategies / tradeoffs**

| Strategy | Upside | Tradeoff |
|----------|--------|----------|
| Short hot TTL + optional compliance archive subset | Cost control | **[ASSUMPTION NEEDED]** what must be archived |
| Store per-PDF durable; zip ephemeral | Retry-friendly | More objects |
| Cold/infrequent access for archive | Cheaper retention | Retrieval latency |
| Deduplicate by content hash | Savings | Complexity |

**Validate:** 12-month storage projection in cost sheet; lifecycle simulation; egress under 2× re-download factor.

---

### 12. What happens if the worker, queue, object storage, or client connection fails

Failure domains (pattern-level; no vendor lock-in assumed):

| Dependency fails | What users see | Naive system behavior | Required engineering concern |
|------------------|----------------|----------------------|------------------------------|
| **Worker** crash mid-render | Job hangs or fails | Message lost or infinite retry; orphan temp files | Heartbeats, visibility timeout, idempotent commit, poison handling |
| **Worker** crash after upload, before status update | Download missing though file exists / duplicate on retry | Split brain status vs artifact | Single commit record tying object key + hash + status |
| **Queue** unavailable | Cannot accept generate requests | Sync fallback overload or total outage | Backpressure to primary service; do not fork-bomb monolith |
| **Queue** delivers twice | Duplicate PDFs/status flicker | Double cost; wrong progress % | Idempotency (see §5) |
| **Object storage** unavailable | Generate “succeeds” in memory but no download | Lost work; retries amplify | Persist-before-complete; retry with same job id |
| **Object storage** slow/partial | Corrupt/truncated PDF | User shares bad GST PDF | Content-length/hash verify before `complete` |
| **Client connection** drops mid zip | “Download failed” | User hits Generate again → new bulk → cost ×N | Durable artifacts + resumable or re-mintable download **without** regenerate |
| **Primary Django / RDS** blip during live hydrate | Partial bulk inconsistency or fail | Mixed PDFs | Snapshot/payload strategy (see §7) |

**Validate:** Chaos matrix — kill worker, block storage, drop client at 50% zip, duplicate queue message; assert invariants: at most one canonical artifact per job id; status never `complete` without verified bytes; client can retry download without new render.

---

## 2. Important Risks — Structured Review

For each risk: failure scenario → naive failure → impact → strategies/tradeoffs → validation.

### Risk A — Memory × peak concurrency collapse

- **Failure:** Unbounded parallel Puppeteer under **~1000/min** + bulk fan-out.  
- **Naive:** One process, unlimited pages, co-located on **16GB** ERP host.  
- **Impact:** OOM; ERP downtime if co-located; missed **5 s**.  
- **Strategies:** Hard concurrency; queue; isolate workers; priority lanes.  
- **Validate:** RSS caps; load test; failure injection at limit.

### Risk B — 5-second single start vs queueing physics

- **Failure:** Backlog wait + **2–3 s** render > 5 s.  
- **Naive:** “Async queue = fast UX.”  
- **Impact:** Peak-hour interactive failure.  
- **Strategies:** Define “start”; reserve capacity; admission control; sync fast-path.  
- **Validate:** Latency budget waterfall; infeasibility proof under stated peak.

### Risk C — Bulk simultaneous render / zip

- **Failure:** 100-way parallel render; huge zip stream once.  
- **Naive:** `Promise.all` + single non-resumable zip.  
- **Impact:** Host death; flaky SMB cannot finish download.  
- **Strategies:** Child job concurrency limits; durable per-PDF; resumable or re-fetchable downloads.  
- **Validate:** Multi-bulk soak; pause client mid-download and resume/re-fetch.

### Risk D — Large-table OOM

- **Failure:** Mid-render crash on **500+** rows.  
- **Naive:** Blind retries of same HTML.  
- **Impact:** Capacity burn; permanent customer pain on biggest POs.  
- **Strategies:** Preflight; isolate; classify poison; template simplification options.  
- **Validate:** Row-count fixtures; assert no retry storm.

### Risk E — Bulk data consistency (GST / 87th / ~3 min)

- **Failure:** Live re-fetch mid-bulk → mixed documents.  
- **Naive:** Bulk of IDs only.  
- **Impact:** Tax/compliance documents disagree inside one zip.  
- **Strategies:** Snapshot payload / freeze / version pins — pick one contract.  
- **Validate:** Concurrent mutation test at child #87.

### Risk F — Tamper evidence without trust anchor

- **Failure:** Editable hash beside editable object.  
- **Naive:** SHA-256 sidecar only.  
- **Impact:** False compliance.  
- **Strategies:** Separate custody or signature; honest threat model.  
- **Validate:** Tamper both sides vs one side.

### Risk G — Budget vs archive vs peak compute

- **Failure:** Keep all PDFs forever + pay for peak Chromium.  
- **Naive:** Infinite S3 + always-on peak fleet.  
- **Impact:** Breach **₹12,500** or under-provision peak.  
- **Strategies:** TTL tiers; archive subset; burst assumptions.  
- **Validate:** Line-item cost model with storage growth + egress retries.

### Risk H — Idempotency / status correctness under retries

- **Failure:** Duplicate work; wrong progress; ambiguous “the” PDF.  
- **Naive:** Status=`failed` then requeue without identity.  
- **Impact:** Support chaos; compliance ambiguity.  
- **Strategies:** Job identity; CAS status; attempt vs artifact.  
- **Validate:** Redelivery tests.

### Risk I — Lifecycle of URL / status / bytes

- **Failure:** Status/object/URL disagree; GC too aggressive or too weak.  
- **Naive:** Immortal links, no GC, or GC without retry window.  
- **Impact:** Broken flaky-net UX or budget blowout.  
- **Strategies:** Explicit TTLs per artifact class.  
- **Validate:** Time-travel / TTL tests.

---

## 3. Where Requirements Conflict

| ID | Conflict | Why it is real | What must give (categories, not choices) |
|----|----------|----------------|------------------------------------------|
| C1 | **~1000/min peak** × **~400MB** × **5 s start** | Little slack after **2–3 s** render; memory caps parallelism | Peak latency guarantee **or** unconstrained concurrency **or** cheap tiny hosts |
| C2 | **Bulk up to 100** + **5–10 bulk/min** vs single **5 s** | Shared workers: bulk children consume slots | Bulk speed **or** single latency **or** more spend |
| C3 | **Puppeteer mandate** vs **500+ OOM** | Tool class known to fail on required docs | Perfect success rate **or** graceful hard failure / product limits |
| C4 | **Zip bulk download** vs **midway fail 50+** | Large single transfer on bad networks | One-shot zip UX **or** retry/resume durability (may add complexity/cost) |
| C5 | **Live ERP accuracy** vs **bulk consistency over minutes** | Mutable source during long jobs | Freshness **or** snapshot stability |
| C6 | **Archive may need** vs **₹12,500** and **100KB–1MB** objects | Retention is recurring cost | Forever hot storage **or** budget headroom for compute |
| C7 | **Tamper evidence** vs **don’t overcomplicate** | Real evidence needs trust separation | Strong threat model **or** lightweight checksum with stated limits |
| C8 | **Don’t re-architect monolith** vs snapshot-in-DB approaches | Strong consistency may need ERP changes | Payload-only purity **or** minimal ERP cooperation **or** weaker consistency |
| C9 | **Co-locate vs budget** | Separate workers cost money; co-locate risks **16GB** ERP | Isolation safety **or** minimal monthly bill |
| C10 | **30k/month average** vs **1000/min peak** for cost narrative | Sizing target ambiguous | Cost optimized for average **or** for peak (state which) |

---

## 4. Assumptions Needed to Resolve Conflicts

These are **not** facts. A later design must **declare** them (or get product answers) before claiming the conflicts are resolved.

| # | Assumption needed | Resolves toward |
|---|-------------------|-----------------|
| A1 | Definition of “**downloads to start** within 5 seconds” (accepted job vs bytes available vs first byte) | C1, Risk B |
| A2 | Whether **5 s** is required at full **~1000/min** sustained, or after admission control / for p95 | C1, C10 |
| A3 | Peak **burst duration** (seconds/minutes) and whether bulk runs concurrently at peak | C1, C2, C10 |
| A4 | Bulk consistency contract: snapshot-at-enqueue vs payload-embedded vs live-read | C5, C8, §7 |
| A5 | Whether PDF service may **only** use request JSON (no per-doc DB fetch) | C5, C8 |
| A6 | Allowed monolith touchpoints (new endpoint vs webhook vs none) under “no re-architecture” | C8 |
| A7 | Meaning of “graceful” for OOM: fail job vs split document vs simplified template | C3, §9 |
| A8 | Download retry window and whether **regenerate** is forbidden when bytes exist | C4, §6, §12 |
| A9 | Zip requirement: must be single zip vs zip-of-available + per-file fallback acceptable | C4 |
| A10 | Compliance archive scope (all PDFs vs financial finals only) and retention period | C6, §11 |
| A11 | Tamper-evidence threat model (accidental edit vs malicious storage admin) | C7, §8 |
| A12 | Cost scope: incremental PDF-service only vs including shared RDS/Redis uplift | C9, C10 |
| A13 | Worker placement: forbidden on monolith host vs allowed with strict cgroup limits | C9 |
| A14 | Priority policy: singles preempt bulk children during peak | C2 |
| A15 | Idempotency key format and who generates it (ERP vs PDF service) | §5, Risk H |
| A16 | “Reasonable time” numeric target for bulk of 100 | C2 |

Until A1–A16 are stated, any architecture is **premature optimization of an underspecified objective function**.

---

## 5. Invariants a Correct Design Must Preserve (Architecture-Agnostic)

Regardless of later component choices, reviewers should demand these invariants:

1. **Global render concurrency** is bounded by measured memory, not by request concurrency.  
2. **Bulk** is modeled as many units of work, not one mega-render.  
3. **Status** is a state machine with monotonic terminal semantics; retries do not create ambiguous duplicates.  
4. **Complete ⇒** durable artifact exists and matches integrity metadata.  
5. **Download retry** does not imply **re-render** (unless artifact TTL expired — and that rule is explicit).  
6. **Bulk consistency** follows a declared contract; mixed live reads are not an accident.  
7. **Tamper evidence** names the trust anchor, not only the hash algorithm.  
8. **Lifecycle** distinguishes hot download artifacts vs compliance archive.  
9. **Cost model** separates average volume (**~30k/mo**) from peak capacity (**~1000/min**).  
10. **Failure** of worker/queue/storage/client has an explicit behavior in the chaos matrix (§12).

---

## 6. What This Review Intentionally Does *Not* Decide

- Message bus vs DB-as-queue vs HTTP-only  
- S3 vs disk vs Redis for bytes  
- ECS/EC2/Lambda/Bare process model  
- Sync Django call vs async fire-and-forget  
- Exact signing scheme (HMAC vs X.509 vs WORM hash log)  
- Whether to co-locate with the **r6i.large** monolith  

Those decisions require the **quantitative analyses** listed in `REQUIREMENTS_ANALYSIS.md` (parallelism ceiling, placement under ₹12,500, peak-to-average, bulk wall-clock, storage/egress) **plus** the assumptions in §4 above.

---

*End of critical review. Next disciplined step (when requested): declare assumptions A1–A16 as a working product contract, then run the five quantitative analyses — still comparing architecture options against conflicts C1–C10 rather than committing to one.*
