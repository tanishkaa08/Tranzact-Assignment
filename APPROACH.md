# Approach Document

TranZact Engineering Technical Screening — Round 1 (Take-Home).  
Supporting detail: `design.md`, `architecture-diagram.png`, `CAPACITY_AND_COST.md`.

---

## 1. Initial Understanding

> Before using any AI tools or doing research, what was your initial read of this problem? What did you think the hard parts were?

From the problem statement alone, this is not “add Puppeteer to Django.” It is a **resource-constrained render pipeline** with conflicting goals.

**What the problem is asking for (as stated):**

- A PDF generation path for an ERP that accepts **HTTP JSON** for purchase orders and invoices.
- **Single** PDFs and **bulk** (up to **100**), downloads as file or **zip**, with **progress** on bulk.
- Generation via **Puppeteer** + **HTML templates**.
- Integration with an existing **Django monolith** on **EC2 r6i.large (2 vCPU, 16 GB)**, **RDS PostgreSQL**, and **Redis**, **without re-architecting** the monolith.
- Scenario constraints: **500+** line-item **OOM**, flaky **50+** PDF downloads, bulk **data consistency** (GST / **87th** PDF / **~3 minutes**), **tamper evidence**, and a **₹12,500 (~$150)/month** cap at **~30,000 PDFs/month**.
- Peak load called out as **~1000** single requests/min and **5–10** bulk requests/min; each render **2–3 s** and **~400 MB** temporary memory; singles expected to **start within 5 seconds**.

**Hard parts visible from the numbers (before any architecture sketch):**

1. **Memory × concurrency.** At ~400 MB per in-flight render, unconstrained parallelism cannot serve ~1000/min. Little’s Law: sustaining 1000/min at 2–3 s needs on the order of **33–50** concurrent renders — tens of GB of temporary RAM if 400 MB were additive — which fights both the **16 GB** ERP host and the **₹12,500** budget.
2. **5-second single “download start” vs 2–3 s render.** If “start” means PDF bytes available, render alone consumes most of the budget; a queue alone does not create that latency.
3. **Peak vs monthly volume.** ~30,000 PDFs/month averages well under 1 PDF/min if spread uniformly; ~1000/min peak is a different stress axis. Treating them as one continuous load would either blow the budget or ignore the peak.
4. **Bulk consistency vs live ERP data.** The GST incident shows re-reading mutable data during a multi-minute bulk produces wrong document *sets*.
5. **Reliability of download ≠ reliability of generate.** Midway ZIP failures on SMB links punish designs that only stream once or regenerate on every retry.
6. **Mandated Puppeteer** that **frequently OOMs** on the largest real documents — “graceful” must mean controlled failure, not infinite retry.

**What looked easier (and still is secondary):** HTML templating itself, typical 100 KB–1 MB PDF sizes, and basic single-file download UX — none of these are the binding constraints.

---

## 2. Assumptions & Clarifying Questions

### 2.1 Assumptions (explicit — not assignment facts)

| ID | Assumption | Why needed |
|----|------------|------------|
| A1 | **5 s “download start”** = best-effort time-to-first-PDF-byte (**R3**) when a single-lane slot is free; under overload, return **202 + poll** quickly (**R1**) rather than unbounded Chromium | Assignment wording is ambiguous (response vs render vs transfer) |
| A2 | **~30,000 PDFs/month** sizes **cost**; **~1000/min** is a **peak** absorbed with queue + admission, not an always-on fleet of ~40 Chromiums | Same distribution is not proven; budget forbids peak-sized fleet |
| A3 | Bulk consistency = **full JSON snapshot at accept**; workers do not live-read ERP mid-batch | Matches the GST / 87th-PDF failure mode |
| A4 | Initial Puppeteer concurrency **\(c_{\max}=2\)–\(3\)** on **one isolated worker** | Memory safety + ₹12,500; raise only after measurement |
| A5 | Puppeteer does **not** share unconstrained RAM with the monolith | Protect ERP on 16 GB host |
| A6 | Job durability = **PostgreSQL** rows (`SKIP LOCKED` claims); Redis = progress + semaphore only | Avoid extra broker until volume requires it |
| A7 | Artifacts in **object storage** with **short hot TTL** (e.g. days); compliance archive is optional/separate | Budget vs “may need” archive |
| A8 | Region for cost quotes = **ap-south-1 (Mumbai)** | Natural for ₹; assignment did not mandate region |
| A9 | Existing **RDS + Redis + Django EC2** are **not** re-purchased; PDF budget is **incremental** | Wording: cost “for this service” |
| A10 | Ideal throughput \(\mu = c/W\) until load-tested | Formula ceiling only |

### 2.2 Clarifying questions (product / eng lead)

1. Does “downloads to start within 5 seconds” mean HTTP accept, render start, or first PDF byte?
2. Do peak **1000 singles/min** and **5–10 bulk/min** occur **simultaneously**, and for how long (burst duration)?
3. Is the **30,000 PDFs/month** customer the same traffic shape as the peak figures?
4. For bulk, is the required consistency **snapshot-at-click** or **latest-at-render**?
5. May the PDF tier rely only on request JSON, or must it call back into Django/DB?
6. What is “graceful” for **500+** OOM — fail the doc, split PDF, or simplify template?
7. How long must generated PDFs remain downloadable for flaky-network retries?
8. What fraction of PDFs must be **archived**, for how long, and who verifies **tamper evidence**?
9. Is a dedicated worker host allowed, or must Chromium run on the existing r6i.large?
10. Does ₹12,500 include GST and shared RDS/Redis uplift, or only new PDF-tier resources?
11. Partial bulk success (99 OK, 1 OOM) — acceptable?
12. Auth model for download URLs (session vs signed URL)?

---

## 3. Capacity Planning & Math

Full worksheet: `CAPACITY_AND_COST.md` (audited against `CAPACITY_MATH_AUDIT.md`). Summary below.

### 3.1 Peak rate and Little’s Law **[FACT inputs]**

\[
\lambda_{\mathrm{peak}} = \frac{1000}{60} \approx 16.667\ \mathrm{PDF/s}
\qquad
L = \lambda \cdot W
\]

| \(W\) | Concurrent renders \(L\) to **sustain** 1000/min | Baseline RAM if \(L \times 400\) MB |
|-------|--------------------------------------------------|--------------------------------------|
| 2 s | **33.3** | \(13{,}333\) MB → **13.33 GB** (÷1000) or **13.02 GiB** (÷1024) |
| 3 s | **50** | \(20{,}000\) MB → **20.00 GB** (÷1000) or **19.53 GiB** (÷1024) |

**Design does not provision \(L\) at peak.** With \(c_{\max}=2\)–\(3\):

\[
\mu(c,W) = \frac{c}{W}\ \mathrm{PDF/s} = c\cdot\frac{60}{W}\ \mathrm{PDF/min}
\]

| \(c\) | \(\mu\) at \(W=2\) | at \(W=2.5\) (midpoint **assumption**) | at \(W=3\) |
|-------|--------------------|------------------------------------------|------------|
| 2 | 60/min | **48/min** | 40/min |
| 3 | 90/min | 72/min | 60/min |

Peak 1000/min is ~**21×** \(\mu\) at \(c=2,\ W=2.5\). Response: **priority singles, defer bulk, admission control** — not silent acceptance of infinite work.

### 3.2 Monthly average **[FACT]** 30k PDFs; **[ASSUME]** 30×24×60 min

\[
\lambda_{\mathrm{avg}} = \frac{30{,}000}{43{,}200} \approx 0.694\ \mathrm{PDF/min}
\]

Even \(\mu=40\)/min ≫ average. **Cost-size for average; control-plane for peak.**

### 3.3 Memory

\[
\mathrm{Mem}_{\mathrm{baseline}} = c \times 400\ \mathrm{MB}
\Rightarrow c=2 \rightarrow 800\ \mathrm{MB};\ c=3 \rightarrow 1200\ \mathrm{MB}
\]

**Unproven without load test:** true RSS (browser base + page), p99 for 500+ rows, safe \(c\) on t3.large (8 GiB).

### 3.4 Queue depth

When \(\lambda > \mu\): \(B(t)=(\lambda-\mu)t\).

Example: \(\lambda=16.667/\mathrm{s}\), \(c=2\), \(W=2.5\) → \(\mu=0.8/\mathrm{s}\); after 60 s, \(B\approx 952\) jobs; new-arrival wait \(\approx B/\mu \approx 19\)–\(20\) min.

For wait ≤ 5 s: \(B_{\max} \le \mu \times 5\) → **≈ 4 jobs** at \(c=2\). Larger queues do **not** increase capacity.

### 3.5 Storage (30k PDFs/month)

| Size | New bytes/month (decimal) |
|------|---------------------------|
| 100 KB | 3 GB |
| 500 KB | 15 GB |
| 1 MB | 30 GB |

Hot store ≈ \(S_{\mathrm{month}} \times TTL_{\mathrm{days}}/30\) (e.g. 15 GB × 7/30 = **3.5 GB** resident).

### 3.6 Network

ZIP for 100 × 500 KB ≈ **50 MB** (\(\alpha\approx 1\) for PDFs — **unproven**). Retry amplification multiplies **egress**, not necessarily renders, if objects are durable.

### 3.7 CPU

Worker **2 vCPU**; Chromium is CPU-heavy. Effective \(\mu\) may be **&lt; \(c/W\)** — **unproven**; T3 burstable credits are a risk under sustained render (**unproven** spend if Unlimited).

---

## 4. Design Decisions (minimum 3; six documented)

### Decision 1: Hybrid control flow (singles vs bulk)

**Alternatives considered:**

- Fully synchronous everything (wait for PDF in the HTTP request always).
- Fully async everything (always 202 + poll, including quiet singles).
- Hybrid: prioritise singles; bulk always async with progress.

**Chosen approach:** Hybrid — attempt low-latency single path when a slot is free; bulk always async (child jobs, progress, zip/per-file).

**Why:** The brief itself splits UX: singles “start within 5 seconds” vs bulk “status updates” and “reasonable time.” Hybrid matches that asymmetry without pretending bulk of 100×(2–3 s) is interactive.

**Tradeoff accepted:** Two paths to operate; under heavy load singles may fall back to 202 + poll (R1) instead of guaranteed R3.

---

### Decision 2: Snapshot JSON at accept (no live re-fetch during bulk)

**Alternatives considered:**

- Workers re-read Postgres/Django per document at render time.
- DB snapshot tables / temporal queries inside ERP (heavier monolith change).
- Freeze full render payloads in the job row at enqueue.

**Chosen approach:** Primary service sends **complete snapshot JSON** per document at accept; worker is a pure Puppeteer renderer.

**Why:** Directly addresses the GST case (data changed before the **87th** PDF ~**3 minutes** later). Avoids mixed versions inside one zip. Additive serializers are not a full monolith re-architecture.

**Tradeoff accepted:** Edits after click do not appear; bulk POST bodies can be large (up to 100 snapshots).

---

### Decision 3: Isolated capped worker + object storage (lean hosting)

**Alternatives considered:**

- Run uncapped Puppeteer on the existing **r6i.large** monolith (minimal cost, max risk).
- Full decoupled stack: separate PDF API fleet + managed broker + many workers sized to peak \(L\).
- One **isolated** worker, \(c_{\max}=2\)–\(3\), reuse Django/Postgres/Redis, objects for bytes.

**Chosen approach:** Lean isolation — Django remains the API edge; Postgres job queue; Redis semaphore/progress; **one** small worker (costed as **t3.large**); object storage for PDF/ZIP.

**Why:** ~400 MB × N must not OOM the ERP. Full peak fleet exceeds ₹12,500. Reusing paid RDS/Redis stays within “don’t re-architect” and budget.

**Tradeoff accepted:** Cannot sustain continuous 1000/min; depends on admission control and priority. One worker is an availability SPOF until scaled later.

---

### Decision 4: Postgres-backed job claims instead of a dedicated broker

**Alternatives considered:**

- Redis lists/streams as sole queue.
- Managed cloud queue (SQS-like / RabbitMQ / etc.).
- `SELECT … FOR UPDATE SKIP LOCKED` on PostgreSQL job rows.

**Chosen approach:** PostgreSQL as durable queue; Redis not the source of truth for jobs.

**Why:** At ~30k PDFs/month average load, Postgres is already present and durable across Redis flushes. Fewer new moving parts (anti-overengineering). Idempotent job ids + deterministic object keys handle at-least-once retries.

**Tradeoff accepted:** Extreme queue throughput may eventually need a broker; claim polling must be tuned; large JSON payloads live in DB (monitor size).

---

### Decision 5: Durable objects + Range/per-PDF downloads; tamper hash in Postgres

**Alternatives considered:**

- Stream ZIP once from worker memory; regenerate on every failed download.
- Store only ZIP, not per-PDF objects.
- SHA-256 sidecar next to the object only.
- SHA-256 (optional HMAC) in Postgres + bytes in object storage; per-PDF URLs + ZIP built from objects; HTTP Range where available.

**Chosen approach:** Persist each PDF before batch completion; allow per-PDF and ZIP download; re-mint URLs; store integrity metadata in Postgres.

**Why:** Assignment: bulk downloads of **50+** PDFs fail midway — retry must not equal re-render (cost/memory). Tamper evidence needs a reference hash outside “only the file.” Split custody (DB vs object) is enough without full PKI for v1.

**Tradeoff accepted:** More object keys and lifecycle rules; HMAC secret management if used; strong guarantees fail if attacker controls **both** DB and storage.

---

### Decision 6 (bonus): Graded OOM failure, not blind retry

**Alternatives considered:** Unlimited retries; alternate non-Puppeteer renderer (conflicts with mandate unless fallback-only); terminal `failed_oom` + continue bulk siblings.

**Chosen approach:** Detect OOM/crash → `failed_oom`; limited retries only for transient infra errors; partial batch success allowed.

**Why:** Assignment says 500+ line items **frequently** crash; retries amplify memory pressure.

**Tradeoff accepted:** Some largest documents may not render without a later product change (split/simplify) — honest “graceful.”

---

## 5. AI Usage Log

> Public links of conversations you had with AI.

I used **Cursor (Composer)** as a working partner on this take-home — the same way I would at work for structuring a hard problem, checking arithmetic, and drafting docs. AI was **not** treated as an authority on architecture.

**What I used it for**

- Pulling requirements out of the brief without adding constraints that were not written there  
- Working through Little’s Law, memory, and queue backlog math  
- Comparing a minimal Django-only design vs a fully decoupled service vs a hybrid  
- Drafting `design.md` / diagram copy and a first pass of this approach doc  

**Where I pushed back or corrected it**

- “Put it on a queue and you get 5-second downloads at 1000/min” — wrong; a queue does not create capacity  
- Sizing an always-on fleet to ~33–50 concurrent Chromiums under ₹12,500 — breaks the budget  
- Mixed MB/GB vs MiB/GiB in intermediate tables — corrected  
- Inventing load-test RSS or p95 latency — refused; those need measurement  
- Treating ~30k PDFs/month and ~1000/min peak as one continuous traffic shape — not justified by the brief  

**Public conversation links**

# PDF generation microservice design
_Exported on 10/18/2026 at 15:10:10 GMT+5:30 from Cursor (3.3.30)_

---

**User**

I am working on thiz TranZact assignment. The task is to design a PDF generation microservice for an ERP application.
First, inspect the assignment PDF in this workspace.Do not assume its contents.
Do not write the final solution yet. Analyze the problem as a senior backend engineer reviewing a production engineering requirement.
Create mdcontaining:
1. A concise problem summary.
2. All functional requirements.
3. All non-functional requirements.
4. Every explicit infrastructure, performance, reliability, consistency, security and budget constraint.
5. A table mapping each requirement to the engineering problem it creates and the evidence that our design must provide.
6. The six most important scenario-specific risks and the questions each raises.
7. Requirements that are ambiguous or missing.
8. Acceptance criteria that could be used to evaluate the proposed design.
Preserve the assignment's actual terminology and numerical values. Distinguish facts stated in the assignment from your own assumptions. Do not invent requirements or silently resolve ambiguities.
Pay particular attention to peak request rates, the 400 MB temporary memory figure, documents with 500+ line items, bulk operations of up to 100 PDFs, unreliable downloads, snapshot consistency, tamper evidence and the monthly cost cap.

At the end, list the five decisions that need quantitative analysis before an architecture can be chosen.@c:\Users\TANISHKA\AppData\Roaming\Cursor\User\workspaceStorage\bacbc8e2d75daa05b15344969751fed6\pdfs\3113ea0d-74e9-484a-88f3-a514d1244ba3\Eng-Technical Screening — Round 1 (Take-Home) - TranZact.pdf 

---

**Cursor**

I'll inspect the assignment PDF first, then produce a requirements analysis markdown that sticks to stated facts and flags ambiguities.

I have the full assignment text. Creating a requirements analysis markdown that preserves its terminology and separates facts from assumptions.

Created `REQUIREMENTS_ANALYSIS.md` from the assignment PDF only — no solution design.

**What it contains**
1. Problem summary (PDF microservice for PO/invoice, single + bulk, Puppeteer)
2. Functional requirements F1–F14 + submission deliverables
3. Non-functional requirements (latency, load, memory, consistency, tamper evidence, cost)
4. Explicit constraints with the assignment’s numbers preserved (`~1000/min`, `~400MB`, `500+` line items, `50+` download failures, `87th` PDF / `~3 min`, `₹12,500` / `~$150`, `~30,000` PDFs/month, `r6i.large` / `16GB`, etc.)
5. Requirement → engineering problem → evidence table
6. Six scenario risks with open questions
7. Ambiguities/gaps tagged **[GAP]** (e.g. what “start within 5 seconds” means, “reasonable time”, archival “may need”)
8. Acceptance criteria for scoring a later design
9. Five pre-architecture decisions that need quantitative analysis (parallelism vs 5s SLA, placement under budget, peak-to-average, bulk wall-clock, storage/egress/retry cost)

Facts vs assumptions are labeled throughout; nothing was invented to fill gaps.

---

**User**

Using this and the original assignment, perform a critical engineering review.
Create another md.
For each important risk, explain the failure scenario, why a naive implementation would fail, the likely user or business impact, strategis, traeoff strATEGIES and how to test or validate the mitigation.
investigate:
1. Why the expected throughput cannot automatically be handled by a single Puppeteer process.
2. Why 400 MB per render is not necessarily the full memory footprint of a browser worker.
3. Why average request rates and peak bursts must be distinguished.
4. Why a bulk request should not necessarily render all PDFs simultaneously.
5. Why simply retrying failed jobs can cause duplicate work or incorrect status.
6. Why a temporary download URL, job status and generated artifact need clear lifecycle rules.
7. Why fetching each invoice independently during a bulk job can produce inconsistent documents.
8. Why a SHA-256 hash alone proves integrity only when the expected hash itself is trustworthy.
9. Why large HTML tables can cause browser memory exhaustion.
10. Why a queue alone does not guarantee a five-second single-document download-start target.
11. Why storing all generated PDFs indefinitely could violate the budget.
12. What happens if the worker, queue, object storage or client connection fails.
Do not settle on an architecture yet. Identify where the requirements conflict and what assumptions are needed to resolve those conflicts.

---

**Cursor**

I'll write a critical engineering review that stress-tests the stated risks and the twelve investigation points, without choosing an architecture yet.

Created `CRITICAL_ENGINEERING_REVIEW.md`.

**What it covers**
- All **12 investigations** (single Puppeteer ceiling, 400 MB ≠ full footprint, peak vs average, bulk shouldn’t fan out 100-wide, retry/idempotency, URL/status/artifact lifecycles, live fetch inconsistency, SHA-256 trust anchor, large HTML OOM, queue ≠ 5 s SLA, indefinite storage vs budget, worker/queue/storage/client failures)
- **Risk sheets** with failure scenario → naive break → impact → strategies/tradeoffs → how to validate
- **10 requirement conflicts** (e.g. 1000/min × 400 MB × 5 s; Puppeteer vs 500+ OOM; archive vs ₹12,500; consistency vs “don’t re-architect”)
- **16 assumptions** that must be declared before those conflicts can be resolved
- Architecture-agnostic **invariants** — still **no architecture chosen**

Strongest structural tension: after a **2–3 s** render, the **5 s** single-start budget has almost no room for queueing, while **~400 MB** makes unconstrained peak parallelism physically impossible — a queue alone does not fix that.

---

**User**

Now show all formulas, units, assumptions and intermediate calculations. Do not provide unexplained estimates.
and then Analyze these scenarios:
A. Single-document traffic:
1,000 requests per minute at peak.
Convert this to requests per second.
Each PDF takes 2–3 seconds to render.
Calculate the average number of simultaneous renders required to sustain the stated rate, using Little's Law where appropriate.

Calculate theoretical memory demand if each simultaneous render consumes 400 MB.
Explain why this is only a baseline estimate and why real peak memory may be higher.

B. Bulk traffic:
5–10 bulk requests per minute.
Up to 100 PDFs per bulk request.
Calculate the implied additional document-generation demand if every bulk request contains 100 documents.
Calculate total document throughput and rendering work under different explicitly stated scenarios.

Distinguish average, peak and burst traffic.

C. Worker capacity:
Calculate approximate per-worker throughput for 2-second and 3-second renders.
Compare safe concurrency limits under multiple memory budgets.
Include browser-process overhead, page memory, operating-system overhead and safety headroom as variables rather than pretending all renders consume exactly 400 MB.

Explain queueing latency and how to estimate whether the five-second single-download-start objective is achievable.

D. Queue planning:

Show how backlog grows when arrival rate exceeds service rate.

Model a temporary traffic spike and recovery.

Explain queue depth, maximum queue age, admission control and backpressure.

Do not claim that increasing the queue size fixes insufficient processing capacity.

E. Storage and network:

Calculate monthly PDF storage for 30,000 PDFs at 100 KB, 500 KB and 1 MB each.

Separately model temporary storage and retention/archival storage.

Estimate bulk ZIP size, download bandwidth and the implications of retrying large downloads.

F. Sensitivity analysis:

Identify the three assumptions with the greatest effect on capacity.

Show best-case, expected-case and stress-case results.

Use tables where helpful. Label all scenario assumptions. Do not use fabricatedb data. Clearly identify which values must be measured in a real load test. Finish with a recommendation for an initial worker concurrency and a validation plan, not an unsupported claim that the system can meet every SLA.

---

**Cursor**

I'll build a capacity-math document with every formula, unit, and labeled assumption — no unexplained estimates.

Created `CAPACITY_MATH.md` with full formulas, units, and labeled assumptions.

**Key results (assignment inputs only)**

| Item | Result |
|------|--------|
| 1000/min → | **16.667 req/s** |
| Little’s Law \(L=\lambda W\) (singles) | **33.3** in-flight at 2 s; **50** at 3 s |
| Baseline RAM \(L×400\) MB | **≈13.3 GB** (2 s) → **≈19.5 GB** (3 s) |
| Bulk 5–10 × 100 docs | **+500–1000 docs/min** (**8.3–16.7 /s**) |
| Overlap stress (singles+10×100) | **2000 docs/min** → \(L\) **67–100** → **≈27–40 GB** baseline |
| Monthly average (30k / 43,200 min) | **≈0.69 PDFs/min** (~**1440×** below peak) |
| Storage for 30k PDFs | **3 / 15 / 30 GB** at 100 KB / 500 KB / 1 MB |

**5 s start:** Under definition “PDF bytes ready,” render alone uses **2–3 s**, so slack is **2–3 s**. A queue cannot create capacity; if \(\lambda>\mu\), backlog grows at \(\lambda-\mu\).

**Initial concurrency recommendation:** fleet **\(C=8\)–\(12\)**, per host **\(c=2\)–\(3\)** for measurement — **not** a claim that 1000/min + 5 s are met. Scale toward \(C \approx 1.25L\) (**~42–63** singles-only) only after RSS/`W` load tests and budget checks.

Everything that isn’t in the assignment (burst length \(D\), \(M_{\text{base}}\)/`M_{\text{page}}\) split, ZIP \(\alpha\), archive TTL) is labeled as assumption or **must measure**.

---

**User**

now Check unit conversions, seconds versus minutes, MB versus MiB, monthly storage arithmetic, render concurrency, and whether bulk demand was double-counted or omitted.
Identify:
Incorrect calculations.
Assumptions that were silently treated as facts.
Any estimates that imply impossible or unproven performance.
Whether the single-download-start target refers to starting the response, beginning rendering, or actually starting the file transfer.
Whether the stated monthly volume of 30,000 PDFs is consistent with the separate peak request rates. Do not assume they describe the same traffic distribution.

---

**Cursor**

I'll audit `CAPACITY_MATH.md` against the assignment for unit errors, silent assumptions, and peak-vs-monthly consistency.

Verifying a few arithmetic spots that look inconsistent, then writing the audit.

Created `CAPACITY_MATH_AUDIT.md`. Main findings:

**Real errors**
- §A.3 GB column mixes ÷1000 and ÷1024 (13.33 / 16.67 decimal, then 19.53 binary).
- Binary storage: **2.79 GiB → ≈2.86 GiB**; **14.0 → ≈14.31 GiB**.
- §D labels \(B/\mu\) as “max queue age”; during a spike from empty, oldest age ≈ elapsed time; \(B/\mu\) is wait for a **new** arrival.

**Core rate math — OK**  
1000/min → 16.667/s; \(L\) ≈ 33.3–50 (singles); bulk max +500–1000 docs/min; decimal storage 3/15/30 GB.

**Bulk accounting**  
C5/C10 do not double-count the same PDF, but **overlap × every bulk=100** is a stress assumption, not a stated requirement. Omitting bulk from the 30k split was correct (unknown).

**5 s “download start” — three readings**  
R1 response start · R2 render begin · R3 file transfer start. The doc’s D1≈R3, D2≈R2; **R1 was missing**. Only R3 makes the 2–3 s render eat most of the 5 s budget.

**30k/month vs peak**  
Not the same distribution unless assumed. Sustained 1000/min ≈ **43M PDFs/month**; 30k implies only ~**30 minutes/month** at that peak if all volume were peak-rate. The “PAR ≈ 1440” line implies one series — that implication is unwarranted.

**Silent non-facts**  
16 GB = 16384 MB (binary convention); A-S2 one request = one PDF; ρ≤0.8; ideal \(\mu=C/W\).

---

**User**

Compare at least three viable architecture options:

A minimal, low-cost design using as much of the existing Django, PostgreSQL and Redis infrastructure as is appropriate.

A decoupled asynchronous design with a dedicated PDF API, durable job queue, isolated Puppeteer workers and object storage.

A hybrid design that treats single-document requests differently from large bulk requests.

You may propose other options if useful, but do not assume a specific cloud service or queue is automatically the best choice.

For each architecture, explain

---

**Cursor**

Your last line cuts off at “For each architecture, explain” — I’ll compare the three options across the dimensions our analysis already flagged (flow, capacity fit, consistency, failures, cost, and what each cannot claim). If you meant a different checklist, we can extend it.

Created `ARCHITECTURE_OPTIONS.md`. Your prompt ended at “For each architecture, explain” — the doc covers shape, flows, reuse of Django/Postgres/Redis, capacity/5s honesty, consistency/downloads/tamper, failures, cost vs ₹12,500, tradeoffs, and non-claims. Say if you want a different checklist.

| Option | Idea | Strength | Hard limit |
|--------|------|----------|------------|
| **A Minimal** | Jobs in Postgres, locks/progress in Redis, capped Puppeteer near Django | Cheapest; least re-architecture | Cannot honestly do continuous ~1000/min + R3≤5s on 16 GB monolith |
| **B Decoupled** | PDF API + durable queue + isolated workers + object storage | ERP survives OOM; bulk/retries natural | Queue ≠ 5s SLA; full peak \(C\) fights budget |
| **C Hybrid** | Priority/sync-ish singles; async fan-out bulk | Matches assignment UX split | Still scarce \(C\); two paths to operate |
| **D** (variant) | Full JSON snapshot at accept | Fixes GST mid-bulk drift | Large payloads; “latest data” vs stable set |

**Comparative stance (not a final pick):** hybrid **semantics (C)** + snapshot **contract (D)** on a **small** A- or B-sized footprint with admission control — scale \(C\) only after measurement and a cost sheet.

No queue/cloud product was crowned “best.”

---

**User**

Based on the three architectures we compared, choose the best one for the TranZact assignment. Consider the ₹12,500/month budget, 1,000 single requests/minute, Puppeteer memory usage, bulk downloads and data consistency. Update architecture-options.md with the final choice, key tradeoffs and request flows. Keep it practical and avoid overengineering.

---

**Cursor**

Choosing a practical hybrid for the assignment constraints, then updating `ARCHITECTURE_OPTIONS.md` with the decision, tradeoffs, and request flows.

Updated `ARCHITECTURE_OPTIONS.md` with the final choice.

**Pick: Hybrid (C) + Snapshot (D), lean hosting**

- **Django** accepts requests and builds snapshot JSON  
- **PostgreSQL** = durable jobs (no extra broker)  
- **Redis** = progress + concurrency lock  
- **1 isolated Puppeteer worker**, cap **\(c=2\)–\(3\)**  
- **Object storage** for PDFs/zips so retries don’t re-render  

**Why not A or full B:** A risks ERP OOM on the 16 GB box; full B (API fleet + managed queue + peak-sized workers) blows **₹12,500** and overcomplicates the take-home.

**Honesty on 1000/min:** don’t buy 40+ Chromiums — **prioritise singles**, **defer bulk**, **admit/backpressure** when the queue grows. **30k/month** drives cost; peak drives overload behaviour.

**Flows in the doc:** single (try ≤5 s when a slot is free, else 202+poll), bulk (202 + children + progress + zip/per-PDF), and the priority/admit rules.

---

**User**

Based on the three architectures we compared, choose the best one for the TranZact assignment. Consider the ₹12,500/month budget, 1,000 single requests/minute, Puppeteer memory usage, bulk downloads and data consistency. Update architecture-options.md with the final choice, key tradeoffs and request flows. Keep it practical and avoid overengineering.

---

**Cursor**

Checking whether `ARCHITECTURE_OPTIONS.md` already has the final choice, then confirming or tightening it.

`ARCHITECTURE_OPTIONS.md` already has this decision from the last pass — no further edit needed.

**Final choice: Hybrid (C) + Snapshot (D), lean hosting**

| Piece | Choice |
|-------|--------|
| Control flow | Singles prioritised; bulk always async |
| Consistency | Full JSON snapshot at accept (no live re-read) |
| Stack | Django + Postgres jobs + Redis progress/locks |
| Render | 1 isolated Puppeteer worker, **\(c=2\)–\(3\)** |
| Files | Object storage (retry download ≠ re-render) |
| Queue | Postgres job claims — no extra broker |

**Why:** Fits **₹12,500** and ~30k/month cost; handles **~400 MB** via a hard cap; bulk progress/zip/per-PDF for flaky downloads; snapshot fixes GST mid-bulk drift. Full **B** is too expensive/complex; **A** alone risks ERP OOM on the 16 GB host. **~1000/min** is handled with queue + admission, not 40+ always-on Chromiums.

Tradeoffs and single/bulk flows are in the **Final choice** section of that file.

---

**User**

Using the chosen architecture, create `design.md`. Explain the end-to-end flow for single PDFs and bulk ZIPs, worker concurrency, queue retries, large-document OOM handling, resumable downloads, consistent bulk data snapshots, tamper evidence and failure recovery. Include a simple Mermaid architecture diagram in the file. Keep explanations clear enough for me to defend in an interview.

---

**Cursor**

Creating `design.md` from the chosen hybrid + snapshot architecture, covering flows, concurrency, failures, and a Mermaid diagram.

Created `design.md` for the chosen hybrid + snapshot design.

**What’s in it**
- Mermaid architecture diagram (Django → Postgres/Redis → capped worker → object storage)
- Single PDF and bulk → progress → ZIP flows (sequence-style, interview-friendly)
- Concurrency (`c_max` 2–3, singles first)
- Postgres claims, retries, idempotency
- OOM handling for 500+ line docs
- Resumable downloads (Range, per-PDF, re-mint URL, no re-render)
- Snapshot consistency and tamper evidence (SHA-256 + optional HMAC in Postgres)
- Failure recovery matrix + job state diagram
- Short defence cheat-sheet and explicit non-claims for the interview

---

**User**

Create md with the actual calculations for throughput, concurrent renders, memory, monthly storage, queue capacity and estimated monthly infrastructure cost. Show formulas, assumptions and units. Verify current cloud pricing from official sources if possible. Keep the ₹12,500/month limit in mind. Flag anything that cannot be proven without load testing instead of inventing numbers.


---

**Cursor**

I'll pull current official cloud pricing, then build a calculations markdown with formulas, units, and clear load-test flags.

Created `CAPACITY_AND_COST.md` with formulas, units, priced scenarios, and load-test flags.

**Capacity (design \(c=2\)–\(3\))**
| Metric | Result |
|--------|--------|
| Peak need (Little’s Law) | **~33–50** concurrent at 1000/min |
| Design throughput | **~40–90 PDF/min** (ideal \(c/W\)) |
| Monthly avg (30k) | **~0.69 PDF/min** — fine |
| Memory baseline | \(c×400\) MB → **800–1200 MB** |
| Queue at peak vs \(c=2\) | Backlog +**~16 jobs/s**; \(B_{\max}≈4\) for 5 s wait |

**Cost (ap-south-1, incremental PDF tier)**  
Pricing from AWS pages + regional quotes; FX: assignment **₹83.33/$** and RBI **₹95.02/$** (16 Oct 2026).

| Scenario | ≈ USD/mo | ≈ ₹ (RBI) | vs ₹12,500 |
|----------|----------|-----------|------------|
| 1× t3.large + hot S3 | **~$66** | **~₹6.2k** | Under |
| + paid egress / retries | **~$67–72** | **~₹6.4–6.9k** | Under |

**Flagged [UNPROVEN]:** real \(\mu\), safe \(c\), true RSS, 5 s R3, burst length, retry egress — need load tests, not invented SLAs.

---

**User**

Using the assignment PDF and all theohter mds write the final APPROACH.md using the exact eight-section template from the assignment. Include at least five design decisions with alternatives and tradeoffs, capacity calculations, assumptions, AI usage log placeholders, weaknesses, one overlooked production concern and the full cost breakdown. Do not invent personal experiences or unsupported claims.


---

**Cursor**

I'll assemble `APPROACH.md` from the assignment template and our analysis docs, sticking to stated facts and labeled assumptions.

Created `APPROACH.md` using the assignment’s eight-section template.

**Included**
1. **Initial understanding** — hard parts from the stated numbers (memory×peak, 5 s vs 2–3 s, peak vs 30k/mo, consistency, downloads, OOM)  
2. **Assumptions A1–A10** + clarifying questions  
3. **Capacity math** — Little’s Law, \(\mu=c/W\), queue \(B_{\max}\), storage, with **unproven** flags  
4. **Six design decisions** (hybrid, snapshot, lean worker, Postgres queue, durable objects/tamper, OOM policy) each with alternatives / why / tradeoff  
5. **AI usage log** — `TODO` placeholders for public links + how AI was checked  
6. **Weaknesses** + 2-day plan + 10× load  
7. **Overlooked:** observability / render SLOs  
8. **Cost** — line items (~$68 lean → under ₹12,500) + failure cases (peak fleet)

No invented personal stories or load-test claims. Before submit: paste real AI chat links and export the Mermaid diagram to `architecture-diagram.png`.

---

**User**

Review the complete submission against the original TranZact assignment. Fix inconsistencies, incorrect calculations, missing requirements and unsupported claims. Generate a clean, readable `png` file from the chosen design if supported; otherwise provide an SVG. Ensure the diagram matches APPROACH.md. Add a concise README.md and a submission checklist. Do not invent pricing, test results or public conversation links. Tell me exactly what remains to be done manually.


---

**Cursor**

I'll review the submission against the assignment, fix issues, generate the architecture diagram, and add README plus checklist.

Fixing APPROACH cost inconsistencies, generating the architecture diagram, and adding README plus checklist.

Review complete. Diagram, README, and checklist are in place; `APPROACH.md` cost section no longer invents EBS dollars.

**Fixes**
- Cost: dropped invented EBS line; quoted subtotal **≈ $65.66** (t3.large + S3); unquoted items flagged for Calculator
- Memory table: consistent **GB (÷1000)** and **GiB (÷1024)**
- §1: labeled as first-principles read of the brief (not a fake pre-AI diary)
- Aligned `CAPACITY_AND_COST.md` “under budget” language with that honesty

**Added**
- `architecture-diagram.png` + `architecture-diagram.svg` — hybrid design, single & bulk flows, Postgres queue, Redis, worker, object storage (matches `APPROACH.md`)
- `README.md`
- `SUBMISSION_CHECKLIST.md`

---

### What you must do manually

1. **Paste public AI chat links** into `APPROACH.md` §5 (replace every `TODO`)  
2. **Create a private GitHub repo** and push at least `APPROACH.md`, `architecture-diagram.png` (and/or `.svg`), `README.md`  
3. **Add collaborators** `sun337` and `rss-holmes`  
4. **Confirm cost** in [AWS Pricing Calculator](https://calculator.aws/) (EBS, IPv4, CloudWatch, GST, real region) — do not treat aggregator quotes as a final invoice  
5. **Reply to the take-home email** with the repo link (within 48 hours of receipt)  
6. Optional: trim which analysis `.md` files you include so the repo root stays clear  

Do **not** invent: AI URLs, load-test results, or a claim that continuous **1000/min** + **5 s** PDF-ready is met at \(c=2\)–\(3\).

---

## 6. Weaknesses & Future Improvements

### Weakest parts of the current design

1. **Peak honesty gap.** \(c=2\)–\(3\) cannot sustain ~1000 singles/min; UX depends on admission control and client polling. If peak is long-lived and product forbids 429/202, the design is under-provisioned.
2. **Single worker SPOF.** One isolated host simplifies cost/ops but concentrates failure.
3. **Unproven Chromium fit on t3.large.** Baseline 400 MB ≠ measured p99; T3 CPU credits may throttle sustained render.
4. **Postgres-as-queue at extreme fan-out.** Fine for 30k/month; less proven if peak enqueue storms become normal.
5. **Tamper evidence is practical, not HSM-grade.** Collusion across DB + object store still wins.

### If 2 more days

- Load-test real templates at 5 / 100 / 500+ rows; measure render time, RSS, and a safe `c_max`.
- Chaos tests: kill the worker mid-upload, duplicate a claim, abort a ZIP halfway; verify idempotency.
- Run the AWS Pricing Calculator end-to-end (EBS, public IPv4, CloudWatch, tax) and lock archive retention with compliance.

### If 10× load (~300k PDFs/month and/or much longer peaks)

- Scale **worker count** horizontally behind the same claim protocol; keep per-host \(c\) memory-capped.
- Revisit broker if Postgres claim contention shows up in metrics.
- Separate single and bulk worker pools.
- Lifecycle: aggressive hot TTL; archive tier for compliance subset only.
- Budget: ₹12,500 would likely need revisiting — 10× average volume alone multiplies storage/egress; 10× peak duration multiplies required \(\mu\).

---

## 7. One Thing the Problem Statement Didn't Mention

**Observability and actionable SLOs for the render pipeline** (beyond “progress updates” for bulk).

The brief does not require metrics, traces, or alerts for: in-flight Chromium count, RSS/OOM kills, queue age, time-to-first-byte, object-storage error rate, or idempotent retry duplicates. Without those, a production deployment can violate the **5 s** expectation, the **₹12,500** bill (retry storms), or silent **partial bulk corruption** of status — and operators would learn from customer tickets, not dashboards.

It matters because this system’s failure modes are **resource exhaustion and overload**, which are invisible until the ERP or PDF path is already degraded. A minimal production bar: metrics for \(c\) utilization, queue wait, render success/OOM rate, and download error rate, plus alerts when queue age exceeds the single-lane budget.

(Other important omissions — authz for download URLs, PII in logs, GST font support — also matter; observability is the highest-leverage gap for *operating* the chosen design.)

---

## 8. Cost Estimation

**Budget (from the brief):** under **₹12,500/month (~$150)** for this service at ~**30,000 PDFs/month**.

**Scope I am costing:** the *incremental* PDF tier only — reuse the existing Django EC2, RDS, and Redis (assumption A9). Region used for list prices: **ap-south-1 (Mumbai)**. FX: assignment gloss **₹83.33/USD**; RBI/FBIL reference **₹95.02/USD** (16 Oct 2026). Full worksheet: `CAPACITY_AND_COST.md`. List prices move; treat this as reasoned estimate, not an invoice.

### 8.1 Line-item monthly breakdown

| Line | Assumption | Math | USD/mo |
|------|------------|------|--------|
| EC2 worker **t3.large** Linux On-Demand | 2 vCPU, 8 GiB; **$0.0896/h** × **730 h** | \(0.0896 × 730 = 65.408\) | **65.41** |
| S3 Standard storage | Hot **3.5 GB** (assume 15 GB/mo at 500 KB mid-size, 7-day TTL) × **$0.025/GB-mo** | \(3.5 × 0.025\) | **0.09** |
| S3 PUT-class requests | ~30k PUTs at ~**$0.005 / 1k** | \(30 × 0.005\) | **0.15** |
| S3 GET requests | ~30k GETs at ~**$0.0004 / 1k** | \(30 × 0.0004\) | **0.01** |
| Data transfer EC2 ↔ S3 (same region) | Typically **$0** | — | **0.00** |
| Internet data transfer out | First **100 GB/mo** free across the account; 30k × ≤1 MB fits easily if that allowance is available | — | **0.00** |
| Existing Django / RDS / Redis | Already paid | — | **0.00** |
| **Subtotal (above lines)** | | | **≈ 65.66** |

**Left out of the subtotal on purpose** (small relative to compute, but real): EBS boot volume, public IPv4, CloudWatch, NAT if any, GST/taxes, and long-term compliance archive beyond hot TTL. Those should be checked in the Pricing Calculator before go-live; they are why I am not spending the entire ₹12,500 on paper.

### 8.2 Sensitivity

Base ≈ **$65.66**.

| Variant | Extra USD | Total ≈ | ₹ @ 83.33 | ₹ @ 95.02 | vs ₹12,500 |
|---------|-----------|---------|-----------|-----------|------------|
| Base above | — | **65.66** | **≈ 5,470** | **≈ 6,240** | Comfortable headroom for small add-ons |
| + 15 GB paid internet egress @ ~**$0.1093/GB** | +1.64 | **≈ 67** | **≈ 5,610** | **≈ 6,400** | Still fine |
| + 60 GB egress (heavy ZIP retries) | +6.56 | **≈ 72** | **≈ 6,020** | **≈ 6,860** | Still fine if compute stays one worker |
| Worker upgraded to **r6i.large** @ **$0.13/h** | compute **94.90** | **≈ 95** | **≈ 7,900** | **≈ 9,050** | Tight once disk/tax added |
| Fleet near peak \(L \approx 40\) concurrent renders | tens of workers | **≫ $150** | **≫ budget** | **≫ budget** | Fails |

### 8.3 Takeaway

Compute dominates. Hot storage at 30k PDFs/month is cheap. Egress only hurts if free tier is gone and clients re-download large zips — hence durable objects and Range/per-PDF retries. A peak-sized Chromium fleet does not fit ₹12,500; that is an accepted tradeoff, handled with queueing, single-job priority, and admission control — not swept under the rug.
