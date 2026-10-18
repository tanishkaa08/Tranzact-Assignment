# TranZact Take-Home — PDF Generation Microservice

Architecture and approach for TranZact Engineering Technical Screening — Round 1.

## Design in brief

**Hybrid** handling (singles prioritised, bulk async) plus **snapshot JSON at accept**, on a lean stack:

- **Django** (existing monolith) — API edge, snapshots, status  
- **PostgreSQL** — durable jobs / queue (`SKIP LOCKED`)  
- **Redis** — progress + concurrency semaphore  
- **Isolated Puppeteer worker** — `c_max` 2–3  
- **Object storage** — PDFs/ZIPs, short hot TTL, re-download without re-render  

Peak **~1000 singles/min** is absorbed with **queue + admission control**, not an always-on fleet sized to ~33–50 concurrent Chromiums (that would break the **₹12,500/month** cap).

## What’s in this repo

| File | What it is |
|------|------------|
| [`APPROACH.md`](APPROACH.md) | Required approach document (assignment template) |
| [`architecture-diagram.png`](architecture-diagram.png) | Architecture diagram |
| [`architecture-diagram.svg`](architecture-diagram.svg) | Same diagram (vector) |
| [`design.md`](design.md) | End-to-end design notes |
| [`CAPACITY_AND_COST.md`](CAPACITY_AND_COST.md) | Throughput, storage, and cost math |
| [`ARCHITECTURE_OPTIONS.md`](ARCHITECTURE_OPTIONS.md) | Options considered and final choice |

Other analysis notes in the repo (`REQUIREMENTS_ANALYSIS.md`, `CRITICAL_ENGINEERING_REVIEW.md`, capacity worksheets) document the working trail.

## Constraints from the brief

Puppeteer + HTML templates; single and bulk (≤100); zip + progress; ~1000 single/min and 5–10 bulk/min peak; 2–3 s and ~400 MB per PDF; Django on EC2 r6i.large with RDS + Redis; handle 500+ line OOM, flaky bulk downloads, GST bulk consistency, tamper evidence; stay under **₹12,500 (~$150)/month** at ~**30,000 PDFs/month**.
