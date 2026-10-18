# PDF Generation Microservice — Requirements Analysis

> Source: *Eng-Technical Screening — Round 1 (Take-Home) - TranZact.pdf*  
> Role of this document: senior-backend review of the stated production engineering requirement.  
> **Not a solution design.** No architecture is proposed here.

**Legend**

| Tag | Meaning |
|-----|---------|
| **[FACT]** | Stated explicitly in the assignment |
| **[IMPLIED]** | Strongly suggested by the assignment’s wording, but not fully specified |
| **[GAP]** | Ambiguous, incomplete, or missing; must not be silently resolved |

---

## 1. Concise Problem Summary

**[FACT]** Build a **PDF generation microservice** for an **ERP application** that handles **both individual and bulk document generation**.

**[FACT]** The service receives **HTTP requests with JSON payloads** containing data for **purchase orders** and **invoices** from the **primary service**.

**[FACT]** PDFs must be generated with **Puppeteer** from **HTML templates**. Users download results as a **single file** or a **zip file for bulk**. The design must integrate with an existing **monolithic Django** ERP on a **single EC2 instance** **without requiring the monolith to be re-architected**, while addressing **peak concurrency**, **large-table OOM crashes**, **unreliable bulk downloads**, **snapshot consistency during bulk**, **tamper evidence**, and a **monthly cost cap**.

**[FACT]** Submission is an asynchronous take-home (48 hours): architecture diagram, `APPROACH.md` (template-mandated), public AI chat links, private GitHub repo with collaborators `sun337` and `rss-holmes`.

---

## 2. Functional Requirements

| ID | Requirement | Source tag |
|----|-------------|------------|
| F1 | Accept **HTTP requests with JSON payloads** containing data for **purchase orders** and **invoices** from the **primary service**. | **[FACT]** |
| F2 | Support **single document generation**: **one PDF at a time**. | **[FACT]** |
| F3 | Support **bulk generation**: **multiple documents (up to 100 PDFs)** in **one request**. | **[FACT]** |
| F4 | Generate PDFs using **Puppeteer** based on **HTML templates**. | **[FACT]** |
| F5 | Allow users to **download** generated PDFs as a **single file** (single) or a **zip file** (bulk). | **[FACT]** |
| F6 | Handle **concurrent requests** efficiently. | **[FACT]** |
| F7 | For **bulk requests**, provide **progress updates** to users. | **[FACT]** |
| F8 | **Generated PDFs may need to be archived** for **compliance** purposes. | **[FACT]** — soft “may need”; archival is called out but not fully specified |
| F9 | Integrate with the existing primary ERP setup **without requiring the monolith to be re-architected**. | **[FACT]** |
| F10 | Handle **variable document complexity**, including dynamic tables with **5 to 500+ line items**, and handle Puppeteer **OOM / mid-render crash** cases for **500+ line items** **gracefully**. | **[FACT]** |
| F11 | Account for **unreliable client connectivity**, specifically that **bulk downloads of 50+ PDFs frequently fail midway**. | **[FACT]** |
| F12 | Handle **data consistency for bulk operations** so rendered PDFs do not reflect **stale vs mid-flight ERP updates** in the sense illustrated by the GST invoice scenario (data updated between trigger and later PDF renders). | **[FACT]** scenario; exact consistency model **[GAP]** |
| F13 | Make generated PDFs **tamper-evident** so compliance can **prove a PDF was not modified after generation**, **without overcomplicating the architecture**. | **[FACT]** |
| F14 | Expose / support flows such that **single downloads start within 5 seconds** and **bulk downloads provide status updates** and **complete within reasonable time**. | **[FACT]** (framed as “Users expect”) |

**Out of scope for “functional product API” but required as deliverables [FACT]**

- High-level **architecture diagram** (services, data flow for single and bulk, storage/queue/caching, external integrations).
- **`APPROACH.md`** filled per the provided template (Initial Understanding, Assumptions & Clarifying Questions, Capacity Planning & Math, Design Decisions ≥3, AI Usage Log, Weaknesses & Future Improvements, One Thing Not Mentioned, Cost Estimation).
- **Public links** to AI chat conversations.
- Private GitHub repo; add **`sun337`** and **`rss-holmes`** as collaborators; reply to email with repo link.

---

## 3. Non-Functional Requirements

| ID | Category | Requirement | Source tag |
|----|----------|-------------|------------|
| N1 | Performance / latency | **Single downloads to start within 5 seconds**. | **[FACT]** |
| N2 | Performance / latency | **Bulk downloads** to provide **status updates** and **complete within reasonable time**. | **[FACT]** — “reasonable time” undefined **[GAP]** |
| N3 | Throughput / load | Expected load: **~1000 single requests/min** during **peak times**, and **5–10 bulk requests/min**. | **[FACT]** |
| N4 | Resource profile | Each PDF generation takes **2–3 seconds** and consumes **~400MB memory temporarily**. | **[FACT]** |
| N5 | Payload / artifact size | Individual PDFs are typically **100KB–1MB** in size. | **[FACT]** |
| N6 | Concurrency | Service should handle **concurrent requests efficiently**. | **[FACT]** |
| N7 | Reliability / client UX | Design must account for **unreliable internet** and **midway-failed bulk downloads (50+ PDFs)**. | **[FACT]** |
| N8 | Reliability / render failures | Design must handle **Puppeteer crash mid-render with OOM** for **500+ line items** **gracefully**. | **[FACT]** |
| N9 | Consistency | Design must address **data consistency for bulk operations** (GST invoice stale-data incident: update between trigger and **~87th PDF**, **roughly 3 minutes** later). | **[FACT]** |
| N10 | Security / integrity | PDFs must be **tamper-evident**; prove **not modified after generation**; avoid overcomplication. | **[FACT]** |
| N11 | Compliance / retention | PDFs **may need to be archived** for compliance. | **[FACT]** — retention period, immutability vs archive, legal hold **[GAP]** |
| N12 | Cost | Total **monthly infrastructure cost** for **this service** must stay **under ₹12,500 (~$150 USD)** for a customer generating approximately **30,000 PDFs/month**. | **[FACT]** |
| N13 | Integration / coupling | Must integrate with existing monolith **without re-architecting** it. | **[FACT]** |
| N14 | Technology constraint | PDF generation via **Puppeteer** + **HTML templates** (technology choice is mandated, not optional). | **[FACT]** |
| N15 | Operability (evaluation) | Capacity math, failure modes, infrastructure costs, consistency reasoning expected in evaluation. | **[FACT]** (Evaluation Criteria) |
| N16 | Scale thinking (submission) | Weaknesses section asks what would change for **10x the load** — not a current NFR of the system, but a required reflection. | **[FACT]** (template) |

---

## 4. Explicit Constraints (Infrastructure, Performance, Reliability, Consistency, Security, Budget)

### 4.1 Infrastructure **[FACT]**

| Constraint | Stated value / rule |
|------------|---------------------|
| Primary ERP | **Monolithic Django** application |
| Compute | **Single EC2 instance**: **r6i.large**, **2 vCPUs**, **16GB RAM** |
| Database | **PostgreSQL (RDS db.r6i.large)** |
| Cache | **Redis** for caching |
| Integration rule | PDF service must integrate **without requiring the monolith to be re-architected** |
| Generation stack | **Puppeteer** + **HTML templates** |
| Request protocol / payload | **HTTP** + **JSON** from the **primary service** |

**[GAP]** Whether the PDF microservice must run *on* the same EC2, or may be separate infrastructure paid from the ₹12,500 cap, is not stated.  
**[GAP]** Whether Redis/PostgreSQL usage by the PDF service must stay on the *existing* Redis/RDS (shared) or may add new instances under the budget is not stated.  
**[GAP]** Cloud provider beyond the EC2/RDS naming (AWS-shaped) is implied but region, AZ, networking, and VPC assumptions are not stated.

### 4.2 Performance **[FACT]**

| Constraint | Stated value |
|------------|--------------|
| Peak single rate | **~1000 single requests/min** |
| Peak bulk rate | **5–10 bulk requests/min** |
| Bulk size cap | **up to 100 PDFs** per bulk request |
| Per-PDF latency (generation) | **2–3 seconds** |
| Per-PDF temporary memory | **~400MB** |
| PDF size | **100KB–1MB** typical |
| Single download start SLA | **within 5 seconds** |
| Bulk completion SLA | **“reasonable time”** only |

### 4.3 Reliability **[FACT]**

| Constraint | Stated value / rule |
|------------|---------------------|
| Large docs | **5 to 500+ line items**; **500+** frequently causes **Puppeteer OOM / mid-render crash** — must handle **gracefully** |
| Client network | SMB customers; **unreliable connections**; **bulk downloads of 50+ PDFs frequently fail midway** — design must **account for this** |
| Concurrency | Must handle concurrent requests **efficiently** |
| Bulk UX | **Progress updates** required |

### 4.4 Consistency **[FACT]**

| Constraint | Stated scenario |
|------------|-----------------|
| Incident | GST invoice PDF showed **stale data** because **ERP data was updated** between **bulk generation trigger** and render of the **87th PDF** (**roughly 3 minutes** later) |
| Design obligation | Explain **how the design handles data consistency for bulk operations** |

**[GAP]** Assignment does not define whether the required property is **point-in-time snapshot at request**, **read-your-writes**, **strong consistency with ERP**, or **eventual** with version stamps. The incident narrative implies users expected renders to reflect a stable view as of trigger (or at least not a silently mixed mid-bulk view) — but the exact contract is not named.

### 4.5 Security / Integrity **[FACT]**

| Constraint | Stated rule |
|------------|-------------|
| Tamper evidence | Generated PDFs must be **tamper-evident** |
| Proof goal | Prove a PDF was **not modified after generation** |
| Complexity bound | Handle **without overcomplicating the architecture** |

**[GAP]** No mention of authn/authz, encryption at rest/in transit, PII, tenant isolation, secrets, or audit logs beyond tamper evidence and compliance archival.

### 4.6 Budget **[FACT]**

| Constraint | Stated value |
|------------|--------------|
| Monthly cap | **Under ₹12,500 (~$150 USD)** |
| Scope of cost | **Total monthly infrastructure cost for this service** |
| Demand reference | Customer generating approximately **30,000 PDFs/month** |

**[GAP]** Whether ₹12,500 excludes the *existing* monolith EC2/RDS/Redis (i.e., incremental cost of the PDF service only) vs includes shared usage uplift is not explicit; wording says “for this service,” which suggests the PDF service’s infra, but shared-DB/Redis marginal cost is undefined.

---

## 5. Requirement → Engineering Problem → Evidence Mapping

| Requirement (preserve assignment terms) | Engineering problem it creates | Evidence the design must provide |
|-----------------------------------------|--------------------------------|----------------------------------|
| **~1000 single requests/min** peak + **2–3 s** generation + **~400MB** temp memory | Sustained concurrency vs memory: naive parallel Puppeteer blows RAM; need queueing, worker caps, backpressure | Capacity math: peak concurrency, max parallel browsers/pages, queue depth, wait time vs **5 s start** |
| **5–10 bulk requests/min**, each **up to 100 PDFs** | Bulk fan-out can dominate worker pool and starve singles; zip assembly and progress tracking under load | Isolation or prioritization strategy; bulk throughput math; progress mechanism under peak |
| **Single downloads start within 5 seconds** | End-to-end path (accept → generate or enqueue → first-byte/download availability) bounded; at peak, queue delay may violate SLA | Latency budget breakdown; what “start” means (generation begin vs download URL ready) **[GAP]**; proof under ~1000/min |
| Bulk **progress updates** + complete in **reasonable time** | Long-running jobs need observable state; “reasonable” must be justified | Progress API/channel; estimated completion math for 100×(2–3s) under worker limits |
| Puppeteer + **~400MB** temporary + monolith host **16GB RAM** / **2 vCPUs** | Co-tenancy risk if on same box; even if separate, memory sets hard parallelism ceiling | Placement decision + memory headroom math; CPU contention with Chrome |
| **500+ line items** → Puppeteer **OOM / crash mid-render** | Partial failure, retries, chunking, fallback render path, or degrade without silent corruption | Graceful failure mode: detection, retry policy, user-visible error, optional mitigation for large tables |
| Bulk zip download **50+ PDFs** fail **midway** on unreliable links | Non-resumable single-shot zip download is insufficient UX/reliability | Resume, retry, multipart, per-PDF re-download, or durable URLs — mapped to failure mode |
| GST bulk consistency: update before **87th** PDF (**~3 min**) | Live reads during bulk → mixed/stale documents across one bulk job | Explicit consistency mechanism (snapshot, freeze payload, version pins, etc.) and when data is captured |
| **Tamper-evident** PDFs, not overcomplicated | Integrity after generation without heavy PKI theater | Concrete scheme (hash, signature, store-side seal) + verification procedure for compliance |
| Archive **may need** for compliance | Retention storage cost vs **₹12,500** cap; durability | What is archived, where, how long **[GAP]**; cost line items |
| Integrate w/ Django monolith **without re-architecture** | API contract, sync vs async handoff, no forced split of ERP domain | Clear integration boundary; what monolith changes are *not* required |
| Existing **PostgreSQL RDS** + **Redis** | Temptation to overload shared DB/cache; consistency and cost coupling | What state lives where; impact on existing Redis/RDS |
| Budget **< ₹12,500 (~$150)** for **~30,000 PDFs/month** | Unit economics: compute-minutes, storage, egress, queue; Puppeteer is expensive per PDF | Line-by-line monthly cost; PDFs/month → instance-hours; sensitivity to peak |
| Downloads: **single file** or **zip for bulk** | Object storage, zip packaging timing (incremental vs end), memory for zip | Storage & packaging design; failure if zip built only at end and client drops |
| Evaluation: capacity math, failure modes, costs, consistency | Submission quality bar ≠ feature list | `APPROACH.md` sections with calculations and tradeoffs |

---

## 6. Six Most Important Scenario-Specific Risks (and Questions They Raise)

### Risk 1 — Peak memory vs throughput (`~1000/min`, `~400MB`, `2–3s`)

At face value, unconstrained parallelism is physically impossible on realistic RAM; queueing may break the **5 second** single-download-start expectation.

**Questions raised**

- What does “downloads to **start** within 5 seconds” mean operationally (HTTP 202 + URL, first byte of PDF, or generation kickoff)?
- What maximum concurrent Puppeteer executions are affordable under the **₹12,500** cap?
- Are peak **~1000 single/min** and **5–10 bulk/min** simultaneous, and do bulks share the same worker pool?

### Risk 2 — Large-table OOM (`500+ line items`)

Frequent mid-render crashes create poison messages, incomplete PDFs, and retry amplification that worsens memory pressure.

**Questions raised**

- Is a failed 500+ PDF a hard user error, a degraded/split PDF, or a non-Puppeteer fallback? (Assignment says handle **gracefully**, not which graceful mode.)
- Should line-item count be validated before enqueue?
- Do retries of OOM jobs get isolated so they cannot starve the peak single-request path?

### Risk 3 — Unreliable bulk download (`50+ PDFs` fail midway)

If the only artifact is one large zip streamed once, SMB clients will repeatedly regenerate or re-download, multiplying cost toward the **30,000 PDFs/month** / budget envelope.

**Questions raised**

- Must the zip be resumable, or is per-PDF download + client-side assemble acceptable?
- How long do generated PDFs / zips remain available for retry?
- Does a failed download require regenerating PDFs or only re-fetching stored bytes?

### Risk 4 — Bulk snapshot consistency (GST invoice, **87th** PDF, **~3 minutes**)

Without an explicit data-capture moment, bulk jobs can emit a mixed set of documents that is worse than uniformly “old” or uniformly “new.”

**Questions raised**

- Is the JSON payload from the primary service already a full snapshot, or does the PDF service re-read ERP/DB per document?
- Is consistency required **across the bulk** (all 100 from one snapshot) or **per document** as of its render time?
- Who is the source of truth during the bulk window: request payload, Redis, or PostgreSQL?

### Risk 5 — Tamper evidence vs operational simplicity

Overbuilt signing infrastructure can blow budget and complexity; underbuilt hashing may not satisfy “prove not modified.”

**Questions raised**

- Who verifies tamper evidence (compliance team tooling, ERP UI, external auditor)?
- Is tamper evidence required for **every** PDF or only archived/compliance documents?
- Must proof hold if storage is compromised, or only detect client-side/file modification?

### Risk 6 — Budget ceiling (`₹12,500` / `~$150`) under Puppeteer economics

**30,000 PDFs/month** × **2–3 s** × Chromium memory footprint competes with storage, egress (zips), and peak capacity headroom for **~1000/min**.

**Questions raised**

- Is the cap incremental cost of the PDF service only, or fully loaded including shared RDS/Redis growth?
- Is **30,000/month** average compatible with **~1000/min** peaks (peak-to-average ratio), or are those different customer profiles?
- What egress assumption applies when bulk zips are re-downloaded after mid-transfer failure?

---

## 7. Ambiguous or Missing Requirements

### Ambiguous (stated but underspecified) **[GAP]**

| Topic | What is unclear |
|-------|-----------------|
| “Downloads to **start** within 5 seconds” | Start of download vs job accepted vs PDF ready |
| Bulk “**reasonable time**” | No numeric SLO |
| “**May need** to be archived” | Optional vs required; retention duration; who accesses archives |
| “Handle **gracefully**” (OOM) | User error vs retry vs alternate renderer vs chunked PDF |
| “**Tamper-evident**” / “prove not modified” | Cryptographic strength, verifier, key custody |
| “Data **consistency** for bulk” | Snapshot isolation vs per-doc read vs payload-only |
| “Without **overcomplicating**” | Subjective complexity budget |
| “Integrate… **without… re-architected**” | Allowed additive endpoints/webhooks vs forbidden changes |
| Cost “for **this service**” | Shared infra attribution |
| “**Efficiently**” (concurrency) | No quantitative efficiency metric |
| Progress **updates** | Polling vs push (SSE/WebSocket); granularity |
| HTML **templates** | Who owns templates; versioning; multi-tenant branding |
| AuthN/Z for download URLs | Not mentioned |
| Idempotency / retries of HTTP generate APIs | Not mentioned |

### Missing (not stated; do not invent answers) **[GAP]**

- Authentication, authorization, multi-tenant isolation between ERP customers  
- Encryption in transit/at rest; secrets management  
- PII / GST invoice data handling and logging redaction  
- Explicit availability SLO (uptime), RPO/RTO  
- Retention/deletion policy and legal hold for archives  
- Observability requirements (metrics, tracing, alerting) — implied by “production” evaluation themes but not specified  
- Exact API schema for JSON payloads  
- Ordering guarantees within bulk  
- Partial bulk success semantics (99 succeed, 1 OOM)  
- Whether PDF service may call back into Django/DB or must be payload-only  
- Geographic region, CDN, and egress pricing assumptions  
- Whether **~1000/min** is sustained for how long (burst length)  
- Non-Puppeteer fallback allowed or forbidden (Puppeteer is mandated for generation; fallback on OOM unclear)  
- Test/QA environment cost under the same ₹12,500 cap  
- Support for languages/fonts (GST invoices often need Indic fonts) — not mentioned  
- Rate limiting / abuse protection  

---

## 8. Acceptance Criteria for Evaluating a Proposed Design

Use these as review checks. Each criterion maps to assignment language.

1. **Dual mode**: Design supports **single** (one PDF) and **bulk** (**up to 100 PDFs** / request), with bulk results downloadable as a **zip**.  
2. **Tech mandate**: Generation path is **Puppeteer** + **HTML templates**, fed by **HTTP JSON** from the **primary service**.  
3. **Peak load reasoning**: Capacity math addresses **~1000 single/min** and **5–10 bulk/min** given **2–3 s** and **~400MB** temporary memory per PDF.  
4. **Single latency**: Argument shows how **single downloads start within 5 seconds** under peak (including queueing), with an explicit definition of “start.”  
5. **Bulk UX**: Bulk jobs expose **progress updates**; completion time is justified as **reasonable** with numbers.  
6. **Large documents**: Explicit graceful behavior for **500+ line items** / Puppeteer **OOM mid-render**.  
7. **Unreliable download**: Explicit mechanism addressing **midway failure** of **bulk downloads of 50+ PDFs**.  
8. **Consistency**: Explicit answer to the **GST / 87th PDF / ~3 minutes** stale-data scenario for **bulk operations**.  
9. **Tamper evidence**: Mechanism to **prove a PDF was not modified after generation**, with complexity kept modest.  
10. **Compliance archival**: Position on archiving generated PDFs (even if “may need”) and cost impact.  
11. **Integration**: Clear fit with **Django monolith** on **EC2 r6i.large (2 vCPU, 16GB)**, **RDS db.r6i.large**, **Redis**, **without re-architecting** the monolith.  
12. **Budget**: Line-by-line monthly cost **under ₹12,500 (~$150)** for approximately **30,000 PDFs/month**.  
13. **Diagram completeness**: Shows components, single & bulk data flows, **storage, queue, and caching**, and external integrations.  
14. **APPROACH.md completeness**: Template sections filled, including **capacity math**, **≥3 design decisions** with alternatives/tradeoffs, **AI usage links**, weaknesses/**10x load**, and “one thing not mentioned.”  
15. **Honesty bar**: Assumptions and unresolved **[GAP]** items are listed rather than silently filled.

---

## Facts vs Assumptions (Working Discipline)

### Facts to preserve in any later design (do not “round away”)

- Peak: **~1000** single/min; **5–10** bulk/min  
- Bulk cap: **100** PDFs  
- Generate: **2–3 s**, **~400MB** temp  
- PDF size: **100KB–1MB**  
- Single start: **5 seconds**  
- Line items: **5 to 500+**; OOM on **500+**  
- Download fragility: **50+** PDF bulks fail midway  
- Consistency anecdote: **87th** PDF, **~3 minutes**  
- Budget: **₹12,500 (~$150)** / month; **~30,000** PDFs/month  
- Infra: Django monolith, **EC2 r6i.large (2 vCPU, 16GB)**, **RDS db.r6i.large**, **Redis**  
- Stack: **Puppeteer**, **HTML templates**, HTTP JSON  

### Assumptions **not** made in this document

- No chosen architecture (queue product, object storage, separate EC2/ECS/Lambda, etc.).  
- No resolution of “start within 5 seconds,” “reasonable time,” or consistency model.  
- No invented auth, retention, or availability SLOs.  
- No assumption that the PDF workers run on the monolith host — only that integration must not re-architect it.

---

## Five Decisions That Need Quantitative Analysis Before an Architecture Can Be Chosen

1. **Parallelism ceiling vs peak arrival rate**  
   Convert **~1000 single requests/min** (+ bulk fan-out from **5–10 × up to 100**) into required completions/sec; given **2–3 s** hold time and **~400MB** each, compute max concurrent render slots and resulting **queue wait** vs the **5 second** single-download-start target.

2. **Worker placement and memory budget under ₹12,500**  
   Compare cost and headroom of sharing the **r6i.large (16GB)** monolith host versus separate render capacity; quantify how many simultaneous **~400MB** Puppeteer jobs fit after OS/Django/PostgreSQL-client/Redis overhead, and what instance mix still stays **under ₹12,500/month**.

3. **Peak-to-average capacity sizing**  
   Reconcile **~30,000 PDFs/month** (~0.7 PDF/min average if uniform) with peak **~1000/min**: required burst duration, queue backlog, and whether the budget buys peak capacity, average capacity, or relies on deferring bulk.

4. **Bulk job wall-clock and progress under worker limits**  
   For **100** PDFs at **2–3 s** each, under the concurrent-slot limit from (1), compute bulk completion time, interference with single-request SLA, and whether zip packaging / storage I/O fits **“reasonable time”** and unreliable-download retries without regenerating.

5. **Storage + egress + archival cost at 30k PDFs/month (and retry amplification)**  
   At **100KB–1MB** per PDF, model monthly storage for working set + compliance archive (**retention [GAP]**), zip sizes for **50–100** PDF bulks, and re-download/egress multipliers from **midway download failures** — show the residual budget left for compute after storage/egress.

---

*End of analysis. Next step (when requested): capacity spreadsheet + architecture options scored against Section 8 acceptance criteria — still without committing to a final solution until the five quantitative decisions above are estimated.*
