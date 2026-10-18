# Capacity & Cost Calculations — Chosen Design

> Architecture: Hybrid singles/bulk + snapshot payloads; Django + Postgres + Redis; **one isolated Puppeteer worker**; object storage (`design.md`).  
> Budget cap **[FACT]:** **₹12,500 / month (~$150 USD)** for this service at ~**30,000 PDFs/month**.  
> Pricing checked against published AWS regional rates and RBI/FBIL FX (sources linked). **Re-verify on [AWS Pricing Calculator](https://calculator.aws/) before committing.**

---

## 0. Legend

| Tag | Meaning |
|-----|---------|
| **[FACT]** | From the TranZact assignment |
| **[PRICE]** | From a cited price source (may change) |
| **[ASSUME]** | Explicit design/scenario assumption |
| **[UNPROVEN]** | Cannot be asserted without load testing / production measurement |

---

## 1. Global inputs

### 1.1 Assignment inputs **[FACT]**

| Symbol | Quantity | Value | Unit |
|--------|----------|-------|------|
| \(\lambda_{\mathrm{peak,single}}\) | Peak single requests | \(\approx 1000\) | 1/min |
| \(\lambda_{\mathrm{peak,bulk}}\) | Peak bulk requests | \(5\)–\(10\) | 1/min |
| \(K_{\max}\) | Max PDFs per bulk | \(100\) | PDFs |
| \(W\) | Render time | \(2\)–\(3\) | s/PDF |
| \(M_{\mathrm{assign}}\) | Stated temp memory per generation | \(\approx 400\) | MB |
| \(s\) | Typical PDF size | \(100\) KB – \(1\) MB | /PDF |
| \(V\) | Cost-scenario volume | \(\approx 30{,}000\) | PDFs/month |
| \(B_{\mathrm{cap}}\) | Infra budget | \(12{,}500\) | ₹/month |
| | Assignment’s USD gloss | \(\sim 150\) | USD/month |

### 1.2 Design concurrency **[ASSUME]** (from `design.md`)

| Symbol | Value | Notes |
|--------|-------|-------|
| \(c_{\max}\) | \(2\) or \(3\) | Hard in-flight Puppeteer cap on one worker |
| Worker shape (costed) | **t3.large** (2 vCPU, **8 GiB**) | Fits \(c=2\) baseline RAM; see §6 |
| Alt. cheaper | **t3.medium** (4 GiB) | Only if load test proves \(c=1\) safe |
| Region for quotes | **ap-south-1 (Mumbai)** | Natural for ₹ billing; assignment did not mandate region |

### 1.3 FX rates

| Rate | Value | Source |
|------|-------|--------|
| Assignment-implied | \(12{,}500 / 150 = \mathbf{83.333}\) ₹/USD | Assignment “₹12,500 (~$150)” |
| RBI/FBIL reference | \(\mathbf{95.0194}\) ₹/USD | [RBI](https://www.rbi.org.in/) / FBIL as of **16 Oct 2026**, 13:00 |

\[
B_{\mathrm{cap,USD}}^{\mathrm{assign}} = 150\ \mathrm{USD}
\qquad
B_{\mathrm{cap,USD}}^{\mathrm{RBI}} = 12{,}500 / 95.0194 \approx \mathbf{131.55}\ \mathrm{USD}
\]

**Use both columns in the cost table.** Taxes (e.g. GST in India) are **excluded** unless noted — AWS list prices are typically pre-tax.

---

## 2. Throughput

### 2.1 Formulas

Service rate of one concurrency slot:

\[
\mu_{\mathrm{slot}}(W) = \frac{1}{W}\ \left[\frac{\mathrm{PDF}}{\mathrm{s}}\right]
= \frac{60}{W}\ \left[\frac{\mathrm{PDF}}{\mathrm{min}}\right]
\]

Fleet (here: one worker) with \(c\) parallel slots, **ideal** efficiency:

\[
\mu(c,W) = \frac{c}{W}\ \left[\frac{\mathrm{PDF}}{\mathrm{s}}\right]
= c\cdot\frac{60}{W}\ \left[\frac{\mathrm{PDF}}{\mathrm{min}}\right]
\]

**[UNPROVEN]** Real \(\mu\) may be **lower** than \(c/W\) due to CPU contention, browser startup, and GC — must measure.

### 2.2 Sustained throughput at design \(c\)

| \(c_{\max}\) | \(W=2\) s | \(W=2.5\) s **[ASSUME midpoint]** | \(W=3\) s |
|--------------|-----------|-------------------------------------|-----------|
| 2 | \(60\) PDF/min | \(48\) PDF/min | \(40\) PDF/min |
| 3 | \(90\) PDF/min | \(72\) PDF/min | \(60\) PDF/min |

### 2.3 Peak demand vs design \(\mu\)

\[
\lambda_{\mathrm{peak,single,s}} = \frac{1000}{60} = \frac{50}{3} \approx 16.667\ \mathrm{s}^{-1}
= 1000\ \mathrm{min}^{-1}
\]

| Comparison | Peak singles | Design \(\mu\) at \(c=2,\ W=2.5\) | Gap |
|------------|--------------|-------------------------------------|-----|
| Rate | \(1000\) /min | \(48\) /min | Peak is \(\approx 20.8\times\) design \(\mu\) |

**Conclusion:** With \(c=2\)–\(3\), the design **cannot** sustain continuous \(1000\) singles/min. It **can** sustain the **monthly average** (next subsection) and short bursts via queue + admission (`design.md`).

### 2.4 Monthly average throughput **[ASSUME]** 30-day month, 24×7 uniform

\[
T_{\mathrm{month}} = 30\times 24\times 60 = 43{,}200\ \mathrm{min}
\]

\[
\lambda_{\mathrm{avg}} = \frac{30{,}000}{43{,}200} \approx 0.6944\ \mathrm{PDF/min}
\approx 0.01157\ \mathrm{PDF/s}
\]

At \(c=2,\ W=3\) s: \(\mu = 40\) PDF/min \(\gg 0.694\) PDF/min → **average volume is easy**.

**Do not treat** \(\lambda_{\mathrm{avg}}\) and \(\lambda_{\mathrm{peak}}\) as one distribution without proof (`CAPACITY_MATH_AUDIT.md`).

### 2.5 Bulk add-on (stress, max fill) **[ASSUME]** every bulk has 100 PDFs

\[
\lambda_{\mathrm{bulk\ docs}} = (5\ldots 10)\times 100 = 500\ldots 1000\ \mathrm{PDF/min}
\]

Only meaningful if bulk peak **overlaps** singles peak — **not stated as FACT**.

---

## 3. Concurrent renders (Little’s Law)

### 3.1 Formula

For a system that **sustains** arrival rate \(\lambda\) (1/s) with mean service time \(W\) (s):

\[
L = \lambda \cdot W
\quad\text{(mean in-service renders)}
\]

Configured cap must satisfy \(c_{\max} \ge L\) to sustain without growing backlog; if \(c_{\max} < L\), backlog grows (§5).

### 3.2 Concurrent renders to **sustain** peak singles

| \(W\) (s) | \(\lambda = 1000/60\) (1/s) | \(L = \lambda W\) |
|-----------|----------------------------|-------------------|
| 2 | \(16.667\) | \(\mathbf{33.33}\) |
| 2.5 | \(16.667\) | \(\mathbf{41.67}\) |
| 3 | \(16.667\) | \(\mathbf{50.00}\) |

### 3.3 Concurrent renders under **design cap**

At steady state with \(c_{\max}=2\) or \(3\), in-service count \(\le c_{\max}\) (not \(L_{\mathrm{peak}}\)).

| Mode | Mean in-service | Comment |
|------|-----------------|--------|
| Design steady | \(\le 2\) or \(\le 3\) | By semaphore |
| Required for sustained 1000/min | \(33\)–\(50\) | **Not provisioned** under budget design |

---

## 4. Memory

### 4.1 Baseline formula (assignment-literal)

\[
\mathrm{Mem}_{\mathrm{baseline}} = c \times 400\ \mathrm{MB}
\]

| \(c\) | Baseline |
|-------|----------|
| 1 | \(400\) MB |
| 2 | \(800\) MB |
| 3 | \(1200\) MB |
| 33 (peak \(W=2\)) | \(13{,}200\) MB ≈ 12.9–13.3 GB depending on MB/MiB |
| 50 (peak \(W=3\)) | \(20{,}000\) MB ≈ 19.5–20 GB |

### 4.2 Why baseline ≠ proof of safe \(c\)

Real RSS includes browser base, OS, Node, PDF buffers, and **500+** row spikes.

\[
M_{\mathrm{worker}}(c) = M_{\mathrm{os}} + M_{\mathrm{base}} + c\cdot M_{\mathrm{page}}
\quad\text{(shared-browser model — illustrative)}
\]

**[UNPROVEN]** All of \(M_{\mathrm{os}}, M_{\mathrm{base}}, M_{\mathrm{page}}\), p99 RSS, and whether memory returns after each job.

### 4.3 Fit on t3.large (8 GiB) **[ASSUME]** headroom

\[
R_{\mathrm{usable}} \approx 8192\times(1-H) - M_{\mathrm{os}}
\]

Example **[ASSUME]** \(H=0.25\), \(M_{\mathrm{os}}=512\) MB → \(R_{\mathrm{usable}} = 6144-512 = 5632\) MB.

If each job ≈ 400 MB additive: \(c_{\mathrm{safe}} = \lfloor 5632/400\rfloor = 14\) **on paper** — but Chromium often needs more headroom and CPU. **Design still uses \(c=2\)–\(3\)** until measured.

**[UNPROVEN]** That \(c=2\) or \(c=3\) is stable on t3.large with real templates.

**t3.medium (4 GiB):** baseline \(c=2\) → 800 MB + overhead may be tight; treat \(c=1\) as default until tested.

---

## 5. Queue capacity

### 5.1 Backlog growth when \(\lambda > \mu\)

\[
\mu = \frac{c_{\max}}{W}\ \left[\mathrm{s}^{-1}\right]
\qquad
\frac{dB}{dt} = \lambda - \mu
\quad (\lambda > \mu,\ \text{slots saturated})
\]

\[
B(t) = B(0) + (\lambda - \mu)\,t
\quad [B]=\text{jobs},\ [t]=\mathrm{s}
\]

**Example — peak singles vs design \(c=2,\ W=2.5\) s**

\[
\lambda = 16.667\ \mathrm{s}^{-1},\quad
\mu = 2/2.5 = 0.8\ \mathrm{s}^{-1},\quad
\lambda-\mu = 15.867\ \mathrm{s}^{-1}
\]

After \(t = 60\) s of sustained peak from empty:

\[
B(60) = 15.867\times 60 \approx \mathbf{952}\ \text{jobs}
\]

Waiting time for a **new** arrival at that moment (fluid):

\[
T_{\mathrm{wait,new}} \approx B/\mu \approx 952/0.8 \approx \mathbf{1190}\ \mathrm{s}\ (\sim 20\ \mathrm{min})
\]

Oldest waiting job’s age ≈ elapsed saturation time (\(\sim 60\) s here), **not** \(B/\mu\) (see audit).

### 5.2 Admission / max queue depth for a wait SLO

\[
B_{\max} \le \mu \times T_{\mathrm{wait,max}}
\]

| \(c\) | \(W\) | \(\mu\) (1/s) | \(T_{\mathrm{wait,max}}=5\) s | \(B_{\max}\) |
|-------|-------|---------------|------------------------------|--------------|
| 2 | 2.5 | 0.8 | 5 | \(\mathbf{4}\) jobs |
| 2 | 2.5 | 0.8 | 30 | \(\mathbf{24}\) jobs |
| 3 | 2.5 | 1.2 | 5 | \(\mathbf{6}\) jobs |

**Interpretation:** To keep single-lane queue wait ≤ 5 s at \(c=2\), allow only a handful of waiting jobs — then **reject/defer** (429/503) or fall back to async poll messaging. **Increasing \(B_{\max}\) does not increase \(\mu\).**

### 5.3 Postgres queue storage (order of magnitude)

**[ASSUME]** average snapshot payload \(P = 50\) KB/job (invoice JSON) — **[UNPROVEN]** real size.

\[
\mathrm{QueueBytes} \approx B \times P
\]

| \(B\) | At \(P=50\) KB |
|-------|----------------|
| 24 | \(\approx 1.2\) MB |
| 952 | \(\approx 47.6\) MB |
| 10{,}000 | \(\approx 500\) MB |

Payload storage is cheap vs Chromium; **limit \(B\) for latency/UX**, not disk.

### 5.4 Monthly work vs queue

At \(V=30{,}000\)/month and \(\mu_{\min}=40\) PDF/min, a constantly busy worker finishes:

\[
40\times 43{,}200 = 1{,}728{,}000\ \mathrm{PDF/month\ capacity}
\gg 30{,}000
\]

Queue is for **peaks**, not for average backlog of a month of work.

---

## 6. Monthly storage

### 6.1 New bytes per month (decimal, \(1\ \mathrm{KB}=10^3\ \mathrm{B}\), \(1\ \mathrm{MB}=10^6\ \mathrm{B}\))

\[
S_{\mathrm{month}}(s) = 30{,}000 \times s
\]

| Size/PDF **[FACT range / sensitivity]** | \(S_{\mathrm{month}}\) |
|-----------------------------------------|------------------------|
| 100 KB | \(3.0\) GB |
| 500 KB **[ASSUME interior]** | \(15.0\) GB |
| 1 MB | \(30.0\) GB |

Binary GiB (if \(100\ \mathrm{KiB}\), etc.): ≈ **2.86 / 14.31 / 29.30 GiB** (`CAPACITY_MATH_AUDIT.md`).

### 6.2 Hot vs archive

\[
\mathrm{Hot} \approx S_{\mathrm{month}} \times \frac{TTL_{\mathrm{hot\ days}}}{30}
\]

| Case **[ASSUME]** | Hot store |
|-------------------|-----------|
| 500 KB PDFs, TTL 7 d | \(15 \times 7/30 = 3.5\) GB |
| 500 KB, TTL 1 d | \(0.5\) GB |
| 1 MB, TTL 7 d | \(7.0\) GB |

\[
\mathrm{Archive}(Y) \approx f_{\mathrm{archive}} \times S_{\mathrm{month}} \times 12 \times Y
\]

| **[ASSUME]** \(f=1\), 500 KB, 1 year | \(15\times 12 = 180\) GB |
| **[ASSUME]** \(f=1\), 1 MB, 7 years | \(30\times 12\times 7 = 2520\) GB |

Indefinite archive of all PDFs **can** threaten the budget; hot TTL alone at 30k/month does not.

### 6.3 ZIP size (one bulk)

\[
S_{\mathrm{zip}} \approx \alpha \times K \times s
\quad \alpha\approx 1\ \text{for PDFs **[ASSUME]**}
\]

| \(K\) | \(s\) | ZIP ≈ |
|-------|-------|-------|
| 100 | 500 KB | 50 MB |
| 100 | 1 MB | 100 MB |

---

## 7. Verified-ish cloud pricing (ap-south-1)

> Official pages are region pickers / calculators; third-party aggregators mirror offer files. Always confirm in [AWS Pricing Calculator](https://calculator.aws/) and [EC2 On-Demand](https://aws.amazon.com/ec2/pricing/on-demand/), [S3 Pricing](https://aws.amazon.com/s3/pricing/).

### 7.1 Compute — Linux On-Demand **[PRICE]**

| Instance | Spec | $/hour (ap-south-1) | $/month @ 730 h | Source class |
|----------|------|---------------------|-----------------|--------------|
| **t3.medium** | 2 vCPU, 4 GiB | **0.0448** | **32.70** | Aggregators mirroring AWS (DoiT / aws-pricing style) |
| **t3.large** | 2 vCPU, 8 GiB | **0.0896** | **65.41** | Same |
| **r6i.large** (monolith type; reference) | 2 vCPU, 16 GiB | **0.1300** | **94.90** | Widely quoted OD Mumbai; [EC2 pricing](https://aws.amazon.com/ec2/pricing/on-demand/) |

Month hours **[ASSUME]:** \(730\) h ≈ \(365.25/12\times 24\).

### 7.2 Object storage — S3 Standard **[PRICE]**

| Item | ap-south-1 rate | Source |
|------|-----------------|--------|
| S3 Standard storage (first 50 TB) | **$0.025 / GB-month** | Aggregated from AWS offer file / [S3 pricing](https://aws.amazon.com/s3/pricing/) region tables |
| PUT/COPY/POST/LIST | ~$0.005 / 1000 requests | Same class of sources |
| GET | ~$0.0004 / 1000 requests | Same |

AWS documents that S3 storage billing uses **binary GB (GiB)** in FAQs — for small volumes the $ difference vs decimal is minor; we still show GB from §6.

### 7.3 Data transfer out to internet **[PRICE]**

| Item | Rate | Notes |
|------|------|-------|
| First **100 GB/month** out to internet | **$0** | AWS free tier aggregate ([EC2/S3 pricing pages](https://aws.amazon.com/ec2/pricing/on-demand/)) |
| Next (typical first paid tier, ap-south-1) | **~$0.1093 / GB** | Aggregator quote for Mumbai DTO; confirm in calculator |
| Same-region EC2 ↔ S3 | **$0** | [S3 FAQ](https://aws.amazon.com/s3/faqs/) |

### 7.4 Existing ERP stack (not re-bought)

| Resource **[FACT]** | Treatment in **this service** budget |
|---------------------|--------------------------------------|
| EC2 r6i.large monolith | **$0 incremental** (already exists) |
| RDS db.r6i.large | **$0 incremental** for instance; tiny marginal for job rows |
| Redis | **$0 incremental** if existing cache reused |

RDS PostgreSQL db.r6i.large Single-AZ Mumbai is on the order of **~$0.285/h ≈ $208/mo** **[PRICE]** (aggregator) — cited only to show **why** the PDF service must not require a second DB of that size.

---

## 8. Estimated monthly infrastructure cost (PDF service incremental)

### 8.1 Scenario definitions **[ASSUME]**

| ID | Compute | Storage | Egress |
|----|---------|---------|--------|
| **S1 Lean** | 1× **t3.large** On-Demand 730 h | Hot **3.5 GB** (15 GB/mo × 7 d TTL) | ≤100 GB free → **$0** DTO if free tier unused |
| **S2 Lean + paid egress** | Same | Same | **15 GB** DTO paid (free tier already consumed) |
| **S3 Stress egress** | Same | Same | **60 GB** DTO (e.g. 30 GB volume × retry factor 2) |
| **S4 Wrong size** | 1× **r6i.large** always-on for “peak RAM” | 3.5 GB hot | 15 GB paid DTO |

**Not invented into totals:** EBS, Elastic IP, CloudWatch, NAT, GST — confirm in Pricing Calculator (see `APPROACH.md` §8).

Request charges (30k PUT + 30k GET):  

\[
30{,}000/1000 \times (0.005 + 0.0004) \approx \$0.16
\]

negligible vs compute.

### 8.2 Dollar math

| Line | Formula | S1 | S2 | S3 | S4 |
|------|---------|----|----|----|-----|
| EC2 | $/h × 730 | \(0.0896×730=\mathbf{65.41}\) | 65.41 | 65.41 | \(0.13×730=\mathbf{94.90}\) |
| S3 storage | GB × 0.025 | \(3.5×0.025=\mathbf{0.09}\) | 0.09 | 0.09 | 0.09 |
| S3 requests | ~ | **0.16** | 0.16 | 0.16 | 0.16 |
| DTO | GB × 0.1093 | **0** | \(15×0.1093=\mathbf{1.64}\) | \(60×0.1093=\mathbf{6.56}\) | 1.64 |
| **Total USD** | | **≈ 65.7** | **≈ 67.3** | **≈ 72.2** | **≈ 96.8** |

### 8.3 Rupee conversion

| Scenario | USD | ₹ @ 83.333 (assignment) | ₹ @ 95.0194 (RBI 16 Oct 2026) | vs ₹12,500 |
|----------|-----|-------------------------|-------------------------------|------------|
| S1 Lean | 65.7 | **≈ 5,475** | **≈ 6,243** | Under **quoted** lines; add-ons TBD |
| S2 + egress | 67.3 | **≈ 5,608** | **≈ 6,395** | Under **quoted** lines; add-ons TBD |
| S3 retry egress | 72.2 | **≈ 6,017** | **≈ 6,860** | Under **quoted** lines; add-ons TBD |
| S4 r6i worker | 96.8 | **≈ 8,067** | **≈ 9,198** | Tight once EBS/tax/IPv4 added |

**Headroom** (RBI FX, S1): \(12{,}500 - 6{,}243 \approx ₹6{,}257\) for EBS, public IPv4, CloudWatch, GST, spikes, second AZ experiments.

### 8.4 What would **break** the ₹12,500 cap

| Mistake | Approx impact |
|---------|----------------|
| Always-on fleet sized to \(c=40\) (peak Little’s Law) | \(20×\) t3.large ≈ **$1,300+/mo** ≫ budget |
| Archive **all** PDFs 7 years @ 1 MB | 2520 GB × $0.025 ≈ **$63/mo** storage alone — OK alone, but + compute + egress + GST tightens |
| Heavy internet egress (hundreds of GB retries) | 200 GB × $0.1093 ≈ **$22** extra |
| Second full RDS / ElastiCache for PDF only | Hundreds USD — **avoid** |
| Counting **existing** monolith+RDS into “this service” | Already **>$300/mo** — would “fail” budget by attribution, not by PDF design |

### 8.5 Budget verdict for the chosen design

**S1–S3 quoted lines are under ₹12,500** at both FX rates, **if**:

- One small/medium worker (not peak-sized fleet),  
- Hot TTL (not unbounded hot retention),  
- Existing Postgres/Redis reused,  
- Egress kept near monthly PDF volume (resumable downloads help).

**[UNPROVEN]** Final invoice including GST, EBS, Elastic IP, CloudWatch, and support — confirm in Pricing Calculator before treating the design as budget-closed.

---

## 9. What cannot be proven without load testing

| Claim | Status |
|-------|--------|
| Effective \(\mu = c/W\) on t3.large | **[UNPROVEN]** — measure completions/min |
| Safe \(c_{\max}=2\) or \(3\) without OOM | **[UNPROVEN]** — RSS/p99 with real templates |
| Memory exactly 400 MB (vs base+page) | **[UNPROVEN]** — assignment typical only |
| p95 time-to-first-byte ≤ 5 s when idle | **[UNPROVEN]** — includes persist + URL mint |
| Behaviour under true 1000/min for duration \(D\) | **[UNPROVEN]** — \(D\) unknown; expect queue + 429 |
| ZIP compression \(\alpha\) | **[UNPROVEN]** |
| Payload size \(P\) for queue disk | **[UNPROVEN]** |
| Retry multiplier \(r\) on SMB links | **[UNPROVEN]** — drives egress |
| CPU credit exhaustion on T3 Unlimited | **[UNPROVEN]** — sustained Chrome may need non-burstable or Unlimited spend |

---

## 10. Interview summary (numbers to remember)

1. **Peak:** \(1000/\mathrm{min} = 16.67/\mathrm{s}\) → need \(L \approx 33\)–\(50\) concurrent at 2–3 s.  
2. **Design:** \(c=2\)–\(3\) → \(\mu \approx 40\)–\(90\) PDF/min — enough for **30k/month avg (~0.7/min)**, not continuous peak.  
3. **Memory baseline:** \(c\times 400\) MB; peak sustain would need **tens of GB** temporary — why we don’t size for literal peak under ₹12,500.  
4. **Storage:** 30k × (0.1–1) MB = **3–30 GB/month** new; hot 7-day ≈ **0.5–7 GB** resident.  
5. **Queue:** At peak vs \(c=2\), backlog grows ~**16 jobs/s**; \(B_{\max}\approx 4\) for 5 s wait — then admit-control.  
6. **Cost:** ~**$66–72/mo** lean worker + S3 in Mumbai ≈ **₹5.5k–7k** → **inside ₹12,500** with headroom; peak-fleet does not.

---

## 11. Source links (re-check before submission)

| Topic | URL |
|-------|-----|
| EC2 On-Demand | https://aws.amazon.com/ec2/pricing/on-demand/ |
| S3 Pricing | https://aws.amazon.com/s3/pricing/ |
| S3 FAQs (billing GB, transfer) | https://aws.amazon.com/s3/faqs/ |
| RDS PostgreSQL pricing | https://aws.amazon.com/rds/postgresql/pricing/ |
| AWS Pricing Calculator | https://calculator.aws/ |
| RBI / FBIL FX | https://www.rbi.org.in/ |
| Design | `design.md` |
| Prior math / audit | `CAPACITY_MATH.md`, `CAPACITY_MATH_AUDIT.md` |

---

*End. These figures support the chosen lean hybrid design under ₹12,500; they do **not** prove continuous 1000/min × 5 s R3 without a much larger (over-budget) fleet or measured burst + admission behaviour.*
