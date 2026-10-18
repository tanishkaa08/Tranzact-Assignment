# Capacity Planning Math — PDF Generation Microservice

> Companion to `REQUIREMENTS_ANALYSIS.md` and `CRITICAL_ENGINEERING_REVIEW.md`.  
> Source numerical inputs are **[FACT]** from the TranZact assignment unless marked otherwise.  
> **No architecture is chosen.** This document only computes implications.

---

## Notation and Units

| Symbol | Meaning | Unit |
|--------|---------|------|
| \(\lambda\) | Arrival rate of **documents to render** (or requests, when stated) | 1/s or 1/min |
| \(W\) or \(T\) | Mean render (service) time | s |
| \(L\) | Mean number of **in-service** simultaneous renders (Little’s Law) | dimensionless (count) |
| \(C\) | Configured max concurrent renders on a worker or fleet | count |
| \(\mu_{\text{slot}}\) | Service rate of **one** concurrency slot | PDFs/s |
| \(\mu\) | Aggregate service rate of the fleet | PDFs/s |
| \(N\) | Number of workers | count |
| \(c_i\) | Concurrency (in-flight renders) on worker \(i\) | count |
| \(M_{\text{assign}}\) | Assignment “temporary” memory per generation | MB |
| \(M_{\text{base}}\) | Browser/worker base RSS before page content | MB |
| \(M_{\text{page}}\) | Incremental memory for one page/render above base sharing model | MB |
| \(M_{\text{os}}\) | OS / non-browser overhead attributed to the host slice | MB |
| \(H\) | Safety headroom fraction (unused RAM reserved) | 0–1 |
| \(R_{\text{usable}}\) | RAM available for renders after overhead and headroom | MB |
| \(B(t)\) | Queue backlog (waiting jobs) at time \(t\) | count |
| \(A\) | Age of oldest waiting job (max queue age) | s |

**Little’s Law (stable system, mean values):**

\[
L = \lambda \cdot W
\]

Applies to the **in-service** (rendering) population when \(\lambda\) is the **throughput that is actually sustained** and \(W\) is mean time in service. It does **not** by itself include queue wait.

**Time conversion:**

\[
\lambda_{\text{per second}} = \frac{\lambda_{\text{per minute}}}{60}
\]

---

## Global inputs from the assignment **[FACT]**

| ID | Input | Value | Unit |
|----|-------|-------|------|
| F1 | Peak single-document request rate | \(\approx 1000\) | requests/min |
| F2 | Peak bulk request rate | \(5\)–\(10\) | bulk requests/min |
| F3 | Max PDFs per bulk request | \(100\) | PDFs/bulk |
| F4 | Render duration | \(2\)–\(3\) | s/PDF |
| F5 | Stated temporary memory per generation | \(\approx 400\) | MB |
| F6 | Typical PDF size | \(100\) KB – \(1\) MB | per PDF |
| F7 | Monthly volume (cost scenario) | \(\approx 30{,}000\) | PDFs/month |
| F8 | Single download-start expectation | \(5\) | s |
| F9 | Monolith host RAM (context only) | \(16\) | GB \(= 16{,}384\) MB |

**Values that are variables / must be measured** (not fabricated as facts): \(M_{\text{base}}\), \(M_{\text{page}}\) distribution, \(M_{\text{os}}\), peak **burst duration**, real \(W\) distribution, ZIP compression ratio, egress retry multiplier, actual concurrent bulk+single mix.

---

# A. Single-document traffic

## A.1 Convert peak rate to requests per second

**Assumption A-S1:** “\(\approx 1000\) single requests/min” is treated as \(\lambda_{\text{single}} = 1000\ \text{min}^{-1}\) for arithmetic (the “≈” is preserved as assignment wording; we do not invent a different peak).

\[
\lambda_{\text{single,s}} = \frac{1000}{60} = 16.\overline{6}\ \text{s}^{-1} = \frac{50}{3}\ \text{requests/s}
\]

| Quantity | Formula | Result | Unit |
|----------|---------|--------|------|
| Peak single rate | given | \(1000\) | 1/min |
| Peak single rate | \(1000/60\) | \(16.667\) (exact \(50/3\)) | 1/s |

## A.2 Simultaneous renders via Little’s Law

**Assumption A-S2:** Every single request requires exactly one PDF render.  
**Assumption A-S3:** System is in steady state at this arrival rate (service rate ≥ arrival rate; otherwise \(L\) in service still caps at \(C\), and backlog grows — see §D).  
**Assumption A-S4:** Mean render time \(W \in \{2, 2.5, 3\}\) s; \(2.5\) is the midpoint of the stated range (derived), not a new FACT.

\[
L_{\text{single}}(W) = \lambda_{\text{single,s}} \cdot W = \frac{50}{3} \cdot W
\]

| \(W\) (s) | Calculation | \(L_{\text{single}}\) (simultaneous renders) |
|-----------|-------------|-----------------------------------------------|
| \(2\) | \((50/3)\times 2 = 100/3\) | \(33.333\) |
| \(2.5\) | \((50/3)\times 2.5 = 125/3\) | \(41.667\) |
| \(3\) | \((50/3)\times 3 = 50\) | \(50.000\) |

**Interpretation:** To **sustain** \(1000\) singles/min with mean render \(W\), you need on average **\(L\)** renders in flight. Configured concurrency \(C\) must satisfy \(C \ge L\) (plus margin for variance); if \(C < L\), you cannot sustain the rate and a queue grows.

## A.3 Theoretical memory demand at 400 MB per simultaneous render

**Assumption A-S5 (baseline only):** Each in-flight render accounts for exactly \(M_{\text{assign}} = 400\) MB, additive, with zero shared base and zero OS overhead.

\[
\text{Mem}_{\text{baseline}}(W) = L_{\text{single}}(W) \times 400\ \text{MB}
\]

| \(W\) (s) | \(L\) | Calculation | Baseline RAM |
|-----------|-------|-------------|--------------|
| \(2\) | \(33.333\) | \(33.333 \times 400 = 13{,}333.3\) | \(13.33\) GB |
| \(2.5\) | \(41.667\) | \(41.667 \times 400 = 16{,}666.7\) | \(16.67\) GB |
| \(3\) | \(50\) | \(50 \times 400 = 20{,}000\) | \(19.53\) GB |

Compare to monolith **16 GB** **[FACT]**: even the **baseline** for singles-only at \(W=2.5\) s already equals/exceeds the entire ERP host RAM; at \(W=3\) s baseline exceeds it by \(\approx 3.5\) GB before Django/OS/Redis clients.

## A.4 Why this is only a baseline (real peak memory may be higher)

The \(400\) MB figure is an assignment **typical temporary** cost, not a proven p99. Real peak RAM can exceed baseline because:

1. **Base Chromium/Node RSS** may exist *in addition* to per-page cost (see §C variables).  
2. **\(500+\) line-item** documents are stated to OOM — implying some renders consume **more than** \(400\) MB.  
3. **Additivity:** \(N\) concurrent pages may share one browser (sub-additive) *or* isolate processes (super-additive vs naive sharing assumptions).  
4. **Transient copies:** HTML string, DOM, PDF buffer, and upload buffer can overlap in time.  
5. **Variance:** Little’s Law uses means; instantaneous in-flight count can be \(> L\) (e.g. Poisson arrivals). For M/D/∞ intuition, variance still produces spikes; with finite \(C\), spikes become queueing instead of RAM — unless you over-admit.

**Must measure in load test:** RSS/PSS time series at concurrency \(C\), for templates with \(5\), \(100\), \(500+\) rows; p50/p95/p99 memory per render; whether memory returns to baseline after each job.

---

# B. Bulk traffic

## B.1 Implied document-generation demand if every bulk has 100 documents

**Assumption B-S1 (stress for bulk size):** Every bulk request contains exactly \(100\) PDFs (the assignment maximum).  
**Assumption B-S2:** Rate bounds \(5\) and \(10\) bulk/min are evaluated separately.

Document arrival rate from bulk:

\[
\lambda_{\text{bulk docs}} = (\text{bulk requests/min}) \times 100
\]

| Bulk requests/min | Docs/min | Docs/s \(= /60\) |
|-------------------|----------|------------------|
| \(5\) | \(5 \times 100 = 500\) | \(500/60 = 8.333\ \text{s}^{-1}\) |
| \(10\) | \(10 \times 100 = 1000\) | \(1000/60 = 16.667\ \text{s}^{-1}\) |

## B.2 Simultaneous renders for bulk-only (Little’s Law)

\[
L_{\text{bulk}}(W) = \lambda_{\text{bulk docs,s}} \cdot W
\]

| Scenario | \(\lambda\) (1/s) | \(W=2\) s → \(L\) | \(W=3\) s → \(L\) |
|----------|-------------------|-------------------|-------------------|
| 5×100 | \(8.333\) | \(16.667\) | \(25.000\) |
| 10×100 | \(16.667\) | \(33.333\) | \(50.000\) |

Baseline memory \(L \times 400\) MB:

| Scenario | \(W=2\) | \(W=3\) |
|----------|---------|---------|
| 5×100 | \(16.667\times400 = 6{,}667\) MB (\(6.51\) GB) | \(10{,}000\) MB (\(9.77\) GB) |
| 10×100 | \(13{,}333\) MB (\(13.02\) GB) | \(20{,}000\) MB (\(19.53\) GB) |

## B.3 Combined single + bulk document throughput (explicit scenarios)

**Assumption B-S3:** Single peak and bulk peak can overlap (worst case) **or** not — both labeled.

Define:

- \(\lambda_{\text{single docs}} = 1000\ \text{min}^{-1}\) (A)  
- \(\lambda_{\text{bulk docs}} \in \{500, 1000\}\ \text{min}^{-1}\) under B-S1  

| Scenario ID | Description | Docs/min | Docs/s | Notes |
|-------------|-------------|----------|--------|-------|
| S0 | Singles only | \(1000\) | \(16.667\) | Peak singles |
| B5 | Bulk only @ 5×100 | \(500\) | \(8.333\) | |
| B10 | Bulk only @ 10×100 | \(1000\) | \(16.667\) | |
| C5 | Singles + bulk 5×100 | \(1500\) | \(25.000\) | **Overlap assumed** |
| C10 | Singles + bulk 10×100 | \(2000\) | \(33.333\) | **Overlap assumed** |
| C5-mid | Same as C5, \(W=2.5\) | \(1500\) | \(25.000\) | |

Little’s Law in-service requirement \(L = \lambda_s \cdot W\):

| Scenario | Docs/s | \(L\) at \(W=2\) | \(L\) at \(W=3\) | Baseline MB at \(W=2\) (\(L\times400\)) | Baseline MB at \(W=3\) |
|----------|--------|------------------|------------------|----------------------------------------|------------------------|
| S0 | \(16.667\) | \(33.3\) | \(50.0\) | \(13{,}333\) | \(20{,}000\) |
| B5 | \(8.333\) | \(16.7\) | \(25.0\) | \(6{,}667\) | \(10{,}000\) |
| B10 | \(16.667\) | \(33.3\) | \(50.0\) | \(13{,}333\) | \(20{,}000\) |
| C5 | \(25.000\) | \(50.0\) | \(75.0\) | \(20{,}000\) | \(30{,}000\) |
| C10 | \(33.333\) | \(66.7\) | \(100.0\) | \(26{,}667\) | \(40{,}000\) |

## B.4 Rendering work (CPU-time), not just concurrency

Render-seconds demanded per minute:

\[
\text{Render-seconds/min} = (\text{docs/min}) \times W
\]

| Scenario | Docs/min | Work at \(W=2\) (render·s / min) | Work at \(W=3\) |
|----------|----------|----------------------------------|-----------------|
| S0 | \(1000\) | \(2000\) | \(3000\) |
| C5 | \(1500\) | \(3000\) | \(4500\) |
| C10 | \(2000\) | \(4000\) | \(6000\) |

One concurrency slot provides at most \(60\) render-seconds per minute. Therefore minimum slots:

\[
C_{\min} = \frac{(\text{docs/min})\times W}{60} = L
\]

(same as Little’s Law — consistency check).

## B.5 Average vs peak vs burst

| Traffic class | How derived | Order of magnitude |
|---------------|-------------|--------------------|
| **Average (monthly)** | \(30{,}000\) PDFs / month **[FACT]** | See below |
| **Peak (stated)** | \(1000\) singles/min; \(5\)–\(10\) bulks/min **[FACT]** | §A–B |
| **Burst** | Peak sustained for duration \(D\) | \(D\) **must be assumed or measured** — not in assignment |

### Average rate from monthly volume

**Assumption B-S4:** \(30{,}000\) PDFs uniformly spread over \(30\) days × \(24\) h (illustrative uniform average; real business days are peakier).

\[
\begin{align*}
\text{Days} &= 30\ \text{(assumption for month length)} \\
\text{Minutes/month} &= 30 \times 24 \times 60 = 43{,}200\ \text{min} \\
\lambda_{\text{avg}} &= \frac{30{,}000}{43{,}200} \approx 0.6944\ \text{PDFs/min} \\
&= \frac{0.6944}{60} \approx 0.01157\ \text{PDFs/s}
\end{align*}
\]

Peak-to-average ratio (singles peak vs monthly average), documents/min:

\[
\frac{1000}{0.6944} \approx 1440
\]

**Meaning:** Capacity sized only to average (\(\approx 0.7\)/min) under-serves peak by ~**three orders of magnitude**. Capacity sized to continuous peak wastefully idles relative to \(30\)k/month billable work — budget tension.

**Burst:** Let peak last \(D\) minutes. Documents arriving in the burst (singles only):

\[
N_{\text{burst,singles}} = 1000 \times D
\]

If service rate \(\mu < 1000\)/min, backlog after burst ≈ \((1000 - \mu_{\text{per min}}) \times D\) (see §D).  
**\(D\) is not stated** → label as **MUST MEASURE / MUST ASSUME** before claiming SLA.

### Bulk “average” under monthly cap (optional decomposition)

The assignment does **not** split \(30\)k into single vs bulk. Any split is an **extra assumption**. Example **labeled hypothetical only** (not FACT): if \(20\%\) of PDFs are bulk children → \(6{,}000\) bulk PDFs/month → not used as a requirement.

---

# C. Worker capacity

## C.1 Per-slot and per-worker throughput

One concurrency **slot** (one in-flight render at a time on that slot):

\[
\mu_{\text{slot}}(W) = \frac{1}{W}\ \text{PDFs/s} = \frac{60}{W}\ \text{PDFs/min}
\]

| \(W\) (s) | PDFs/s | PDFs/min |
|-----------|--------|----------|
| \(2\) | \(0.500\) | \(30\) |
| \(3\) | \(0.333\) | \(20\) |

**Assumption C-S1:** A “worker” runs \(c\) parallel slots (pages/browsers), ideal efficiency \(100\%\) (no lock contention). Then:

\[
\mu_{\text{worker}}(W,c) = c \cdot \mu_{\text{slot}}(W)
\]

| \(c\) | PDFs/min at \(W=2\) | PDFs/min at \(W=3\) |
|-------|---------------------|---------------------|
| \(1\) | \(30\) | \(20\) |
| \(2\) | \(60\) | \(40\) |
| \(3\) | \(90\) | \(60\) |
| \(4\) | \(120\) | \(80\) |
| \(5\) | \(150\) | \(100\) |

To sustain \(\lambda = 1000\) PDFs/min with mean \(W\):

\[
C_{\text{fleet}} \ge L = \lambda_{\min} \times \frac{W}{60} \times 60\ \text{wait} = \lambda_{\text{per s}}\times W
\]

Same \(L\) as §A: need **\(\approx 33\)–\(50\)** in-flight singles-only; **\(\approx 50\)–\(100\)** for overlap scenarios C5/C10.

Fleet of \(N\) workers each at concurrency \(c\):

\[
C_{\text{fleet}} = N \times c, \quad \mu_{\text{fleet,min}} = N \times c \times \frac{60}{W}
\]

**Efficiency caveat (must measure):** Chromium CPU contention on small vCPU counts can make effective \(\mu < c/W\).

## C.2 Memory model with variables (not “exactly 400 MB”)

Two explicit models; both are **formulas**, not claimed truths.

### Model M1 — Isolated browser per render (conservative additivity)

\[
M_{\text{job}} = M_{\text{base}} + M_{\text{page}}
\]

\[
M_{\text{worker}}(c) = M_{\text{os}} + c \cdot (M_{\text{base}} + M_{\text{page}})
\]

### Model M2 — Shared browser, \(c\) pages (shared base)

\[
M_{\text{worker}}(c) = M_{\text{os}} + M_{\text{base}} + c \cdot M_{\text{page}}
\]

Assignment baseline embeds roughly \(M_{\text{base}} + M_{\text{page}} \approx 400\) MB **typical**, but does not split them.

**Usable RAM:**

\[
R_{\text{usable}} = R_{\text{total}} \times (1 - H) - M_{\text{os}}
\]

**Safe concurrency:**

\[
\begin{align*}
c_{\text{safe,M1}} &= \left\lfloor \frac{R_{\text{usable}}}{M_{\text{base}} + M_{\text{page}}} \right\rfloor \\
c_{\text{safe,M2}} &= \left\lfloor \frac{R_{\text{usable}} - M_{\text{base}}}{M_{\text{page}}} \right\rfloor
\end{align*}
\]

### Parameter tables — **symbolic / sensitivity**, not fabricated “production” measurements

We **do not invent** measured \(M_{\text{base}}\). We show how \(c_{\text{safe}}\) depends on inputs. For sensitivity, use **explicit placeholder sets** labeled **HYPOTHESIS FOR SENSITIVITY ONLY** (to be replaced by load-test measurements):

| Parameter set | \(M_{\text{os}}\) | \(M_{\text{base}}\) | \(M_{\text{page}}\) | Implied typical job (M1) | Label |
|---------------|--------------------|----------------------|----------------------|---------------------------|-------|
| P0 | \(0\) | \(0\) | \(400\) | \(400\) | Degenerate “assignment-literal additive” |
| P1 | \(512\) | \(200\) | \(200\) | \(400\) | Split 400 into base+page **hypothesis** |
| P2 | \(512\) | \(300\) | \(250\) | \(550\) | Stress: heavier than 400 **hypothesis** |
| P3 | \(512\) | \(200\) | \(400\) | \(600\) | Stress: large table page **hypothesis** |

Headroom \(H \in \{0.15, 0.25\}\) (ops choice, not FACT).

### Example numerical walk-through (labeled hypotheses)

**Host budget example:** \(R_{\text{total}} = 8192\) MB (8 GB worker) — **illustration of a worker size**, not an architecture decision.  
**Assumption C-S2:** \(H = 0.25\) → reserve 25%.

\[
R_{\text{usable}} = 8192 \times 0.75 - M_{\text{os}}
\]

| Param set | \(M_{\text{os}}\) | \(R_{\text{usable}}\) | \(c_{\text{safe}}\) M1 | \(c_{\text{safe}}\) M2 |
|-----------|--------------------|------------------------|------------------------|------------------------|
| P0 | \(0\) | \(6144\) | \(\lfloor 6144/400\rfloor = 15\) | N/A (P0 has no base split) |
| P1 | \(512\) | \(5632\) | \(\lfloor 5632/400\rfloor = 14\) | \(\lfloor(5632-200)/200\rfloor = 26\) |
| P2 | \(512\) | \(5632\) | \(\lfloor 5632/550\rfloor = 10\) | \(\lfloor(5632-300)/250\rfloor = 21\) |
| P3 | \(512\) | \(5632\) | \(\lfloor 5632/600\rfloor = 9\) | \(\lfloor(5632-200)/400\rfloor = 13\) |

**16 GB host shared with monolith (danger context):**  
\(R_{\text{total}} = 16384\) MB. If Django+Postgres client+OS already use unknown \(M_{\text{erp}}\) (**MUST MEASURE**), usable for PDF may be small.

Example **if** (hypothesis) \(M_{\text{erp}} = 8\) GB, \(H=0.25\) on remaining:

\[
R_{\text{pdf}} = (16384 - 8192) \times 0.75 = 6144\ \text{MB}
\]

Then under P0: \(c_{\text{safe}} = \lfloor 6144/400\rfloor = 15\) — **far below** \(L \approx 33\)–\(50\) needed for singles peak → **cannot** meet sustained \(1000\)/min on leftover monolith RAM alone under these hypotheses.

## C.3 Queueing latency and the 5-second start objective

### Latency budget decomposition

Let:

- \(T_{\text{accept}}\) = time to accept HTTP and enqueue  
- \(T_{\text{wait}}\) = queue wait  
- \(T_{\text{render}} = W\) ∈ [2, 3] s **[FACT]**  
- \(T_{\text{persist}}\) = write artifact + metadata  
- \(T_{\text{url}}\) = issue download capability  

**Assumption C-S3 (definition fork):** “Downloads to **start** within 5 seconds” is ambiguous. Two evaluable definitions:

| Def | Meaning | Inequality for success |
|-----|---------|------------------------|
| D1 | First byte of PDF available to client by 5 s | \(T_{\text{accept}}+T_{\text{wait}}+T_{\text{render}}+T_{\text{persist}}+T_{\text{url}} \le 5\) |
| D2 | Generation **started** (left queue) by 5 s | \(T_{\text{accept}}+T_{\text{wait}} \le 5\) (render may finish later) |

Under **D1**, even with \(T_{\text{wait}}=0\) and negligible I/O:

\[
T_{\text{render}} \in [2,3] \implies \text{slack for everything else} \in [2,3]\ \text{s}
\]

If \(W=3\) s, slack \(= 2\) s total for accept+wait+persist+url.  
If \(W=2\) s, slack \(= 3\) s.

**Therefore under D1:** any non-trivial queue wait at peak makes the objective **fragile**; at \(W=3\) it is nearly binding even with empty queue.

Under **D2**, render time does not consume the 5 s budget; 5 s only limits admission delay. That is a **product assumption**, not stated.

### Estimating wait time (simple finite-capacity approximation)

**Assumption C-S4:** Fleet service rate \(\mu = C / W\) PDFs/s when busy (deterministic render, always utilized when backlog > 0).

Utilization:

\[
\rho = \frac{\lambda}{\mu}
\]

If \(\rho \ge 1\), mean wait → grows without bound (no steady-state 5 s).  
If \(\rho < 1\), a rough **bound** for wait under deterministic service and fairly smooth arrivals is small, but **Poisson** arrivals with deterministic service (M/D/C) still have positive wait. Exact M/D/C formulas need measured variance — **do not fake a percentile**.

**Necessary condition (hard):**

\[
\mu > \lambda \quad \text{i.e.} \quad C > \lambda \cdot W = L
\]

Plus margin for variance. Example: want \(\rho \le 0.8\):

\[
C \ge \frac{\lambda W}{0.8} = 1.25\, L
\]

| Scenario | \(L\) at \(W=2\) | \(C\) for \(\rho\le0.8\) | \(L\) at \(W=3\) | \(C\) for \(\rho\le0.8\) |
|----------|------------------|--------------------------|------------------|--------------------------|
| S0 singles | \(33.3\) | \(\ge 41.7\) → **42** | \(50\) | \(\ge 62.5\) → **63** |
| C5 overlap | \(50\) | \(\ge 63\) | \(75\) | \(\ge 94\) |
| C10 overlap | \(66.7\) | \(\ge 84\) | \(100\) | \(\ge 125\) |

**Achievability statement (honest):**

- Under **D1** + S0 + \(W=3\) s: even \(T_{\text{wait}}=0\) leaves \(2\) s slack; **claiming** p100 compliance without measured \(T_{\text{persist}}\) is unsupported.  
- Under **D1** + S0 + \(W=2\) s + \(C \ge 42\) and low variance: **plausible** only if I/O stays ≪ 1 s — **MUST MEASURE**.  
- Under **D1** + C10 + \(W=3\): need \(C \ge 125\) in-flight and still fight I/O — memory baseline \(125 \times 400\) MB ≈ **50 GB** temporary under P0 — **in tension with ₹12,500** (cost analysis separate).  
- Queue **depth limit** does not create \(\mu\); it only caps \(T_{\text{wait}}\) by **rejecting** work.

---

# D. Queue planning

## D.1 Backlog growth when arrival exceeds service

Let \(\lambda\) = arrival rate (jobs/s), \(\mu\) = max completions/s = \(C/W\).

While \(\lambda > \mu\) and concurrency saturated:

\[
\frac{dB}{dt} = \lambda - \mu
\]

\[
B(t) = B(0) + (\lambda - \mu)\, t
\]

**Units:** \(B\) in jobs; \(t\) in s; \(\lambda,\mu\) in 1/s.

**Example (singles peak, undersized fleet):**  
\(\lambda = 16.667\ \text{s}^{-1}\), \(W=3\) s, \(C=20\) slots → \(\mu = 20/3 = 6.667\ \text{s}^{-1}\).

\[
\lambda - \mu = 10\ \text{s}^{-1}
\]

After \(t = 60\) s:

\[
B(60) = 0 + 10 \times 60 = 600\ \text{jobs}
\]

Max queue age if FIFO and still saturated:

\[
A \approx \frac{B}{\mu} = \frac{600}{6.667} \approx 90\ \text{s}
\]

(when the 600th waiting job finally enters service, roughly — more precisely age of oldest ≈ time to drain past it).

During saturation, mean wait for a new arrival ≈ \(B/\mu\) growing with \(t\).

## D.2 Temporary spike and recovery

**Assumption D-S1:** Spike of duration \(D\) seconds at rate \(\lambda_{\text{peak}}\), then return to \(\lambda_{\text{normal}} < \mu\).

Backlog at end of spike (from empty):

\[
B(D) = \max\bigl(0,\ (\lambda_{\text{peak}} - \mu)\, D\bigr)
\]

Drain time after spike (while \(\lambda_{\text{normal}} < \mu\)):

\[
T_{\text{drain}} = \frac{B(D)}{\mu - \lambda_{\text{normal}}}
\]

**Worked example (all inputs explicit):**

| Parameter | Value | Unit | Source |
|-----------|-------|------|--------|
| \(\lambda_{\text{peak}}\) | \(16.667\) | 1/s | 1000/min |
| \(W\) | \(2.5\) | s | Midpoint assumption |
| \(C\) | \(30\) | slots | **Illustrative undersize** (\(L=41.7\) needed) |
| \(\mu = C/W\) | \(30/2.5 = 12\) | 1/s | Derived |
| \(D\) | \(120\) | s | **Assumption** (2 min burst) |
| \(\lambda_{\text{normal}}\) | \(0.01157\) | 1/s | Monthly avg §B.5 |

\[
B(120) = (16.667 - 12) \times 120 = 4.667 \times 120 = 560\ \text{jobs}
\]

\[
T_{\text{drain}} = \frac{560}{12 - 0.01157} \approx \frac{560}{11.988} \approx 46.7\ \text{s}
\]

Oldest-job age at spike end ≈ \(B/\mu = 560/12 \approx 46.7\) s ≫ 5 s → **D2 violated during/after spike** for FIFO.

**If** \(C = 45\) at \(W=2.5\) → \(\mu = 18\ \text{s}^{-1} > 16.667\):

\[
B(D)=0\ \text{(no growth while }\mu>\lambda\text{)}
\]

Spike creates **no backlog** in the fluid model (stochastic spikes may still queue briefly — measure).

## D.3 Queue depth, max age, admission control, backpressure

| Concept | Definition | Formula / rule |
|---------|------------|----------------|
| Queue depth \(B\) | Waiting jobs not yet rendering | Observable counter |
| Max queue age \(A\) | Age of oldest waiting job | Aim \(A \le A_{\max}\) |
| Target for D2 | \(A_{\max} \le 5\) s | Requires \(\mu\) and admission such that wait ≤ 5 s |
| Admission control | Refuse/ defer new work when \(B\) or \(A\) exceeds limit | e.g. reject if \(B > B_{\max}\) or predicted wait \(B/\mu > 5\) |
| Backpressure | Signal primary ERP to slow sends | Reduces \(\lambda\) toward \(\mu\) |

**Predicted wait (fluid):** \(T_{\text{wait}} \approx B / \mu\) when busy.

Set \(B_{\max}\) so that:

\[
\frac{B_{\max}}{\mu} \le T_{\text{wait,max}}
\]

Example: \(\mu = 18\ \text{s}^{-1}\), \(T_{\text{wait,max}} = 2\) s (leave room under D1):

\[
B_{\max} \le 18 \times 2 = 36\ \text{jobs}
\]

### Increasing queue size does **not** fix insufficient capacity

If \(\lambda > \mu\) persistently:

- Larger \(B_{\max}\) only **delays** failure: more jobs wait longer (\(A\) increases).  
- It **increases** memory for job payloads/metadata and worsens UX.  
- Stabilization requires **increase \(\mu\)** (more \(C\) or lower \(W\)) and/or **decrease \(\lambda\)** (admission control / backpressure) and/or **shed bulk** priority.

---

# E. Storage and network

## E.1 Monthly PDF storage for 30,000 PDFs

**Assumption E-S1:** \(30{,}000\) PDFs stored once each (no dedup, no retries stored as extras).  
**Assumption E-S2:** Sizes \(100\) KB, \(500\) KB, \(1\) MB are evaluation points spanning the **[FACT]** typical range \(100\) KB–\(1\) MB (`500 KB` is interior point for sensitivity, not a FACT mode).

Unit conversions:

\[
\begin{align*}
100\ \text{KB} &= 100 \times 1024 = 102{,}400\ \text{B} \quad \text{(binary KiB)} \\
&\quad\text{or } 100 \times 1000 = 100{,}000\ \text{B (decimal)} 
\end{align*}
\]

**Assumption E-S3:** Use **decimal MB** for cost-style arithmetic: \(1\) MB \(= 10^6\) B; \(1\) KB \(= 10^3\) B (label clearly). Many cloud bills use GB-month decimal.

| Size/PDF | Bytes/PDF (decimal) | × 30,000 | Total bytes | Total GB (\(10^9\)) |
|----------|---------------------|----------|-------------|---------------------|
| \(100\) KB | \(10^5\) | \(3.0 \times 10^9\) | \(3.0 \times 10^9\) | \(3.0\) GB |
| \(500\) KB | \(5.0 \times 10^5\) | \(1.5 \times 10^{10}\) | \(1.5 \times 10^{10}\) | \(15.0\) GB |
| \(1\) MB | \(10^6\) | \(3.0 \times 10^{10}\) | \(3.0 \times 10^{10}\) | \(30.0\) GB |

Binary (GiB) alternate: divide by \(2^{30}\):

| Size | Total GiB |
|------|-----------|
| 100 KiB × 30k | \(30{,}000 \times 100 \times 1024 / 2^{30} \approx 2.79\) GiB |
| 500 KiB × 30k | \(\approx 14.0\) GiB |
| 1 MiB × 30k | \(30{,}000 / 1024 \approx 29.3\) GiB |

## E.2 Temporary vs retention/archival storage

**Assignment:** downloads needed; archive **may** be needed; budget cap exists. Retention days **not stated**.

Define:

- \(S_{\text{month}}(s)\) = monthly new bytes at size \(s\) (table above)  
- \(TTL_{\text{hot}}\) = days temporary download artifacts retained (**ASSUME** to compute)  
- \(TTL_{\text{archive}}\) = days/years compliance retention (**ASSUME**)  
- \(f_{\text{archive}}\) = fraction of PDFs archived ∈ [0,1] (**ASSUME**)

**Steady-state hot storage** (approximate, constant rate):

\[
\text{Hot} \approx S_{\text{month}}(s) \times \frac{TTL_{\text{hot}}}{30}
\]

**Archive store** after \(Y\) years (no deletion):

\[
\text{Archive}(Y) \approx f_{\text{archive}} \times S_{\text{month}}(s) \times 12 \times Y
\]

**Worked examples with explicit assumptions (not FACTS):**

| Case | \(s\) | \(TTL_{\text{hot}}\) | \(f_{\text{archive}}\) | \(Y\) | Hot ≈ | Archive ≈ |
|------|-------|----------------------|-------------------------|-------|--------|-----------|
| T1 | 500 KB → 15 GB/mo | 7 days | 0 | — | \(15 \times 7/30 = 3.5\) GB | 0 |
| T2 | 500 KB | 1 day | 0 | — | \(15 \times 1/30 = 0.5\) GB | 0 |
| T3 | 500 KB | 7 days | 1.0 | 1 y | 3.5 GB hot | \(15 \times 12 = 180\) GB |
| T4 | 1 MB → 30 GB/mo | 7 days | 1.0 | 7 y | 7 GB hot | \(30 \times 12 \times 7 = 2520\) GB |

**Conclusion from formulas:** Indefinite \(f_{\text{archive}}=1\) at 1 MB dominates any ₹12,500-class budget discussion; hot TTL alone at 30k/month is modest if days-not-years.

**Cost ₹/GB-month:** **MUST LOOK UP** current cloud pricing for the chosen region — do not fabricate a bill here.

## E.3 Bulk ZIP size, bandwidth, retries

**Assumption E-S4:** ZIP size ≈ sum of PDF sizes × \(\alpha\) compression factor. For already-compressed PDFs, \(\alpha \approx 1.0\) (little gain). Use \(\alpha = 1.0\) as baseline; measure actual \(\alpha\).

For \(K\) PDFs in one bulk (\(K \le 100\)):

\[
S_{\text{zip}} \approx \alpha \times K \times s
\]

| \(K\) | \(s\) | \(\alpha\) | Approx ZIP size |
|-------|-------|------------|-----------------|
| 50 | 500 KB | 1.0 | \(25{,}000\) KB = \(25\) MB |
| 100 | 500 KB | 1.0 | \(50\) MB |
| 100 | 1 MB | 1.0 | \(100\) MB |
| 100 | 100 KB | 1.0 | \(10\) MB |

**Download time** at client bandwidth \(b\) (MB/s):

\[
T_{\text{dl}} = \frac{S_{\text{zip}}}{b}
\]

Assignment states SMB links are unreliable and **50+** PDF bulk downloads fail midway — \(b\) and loss rate **MUST MEASURE** in field; do not invent ISP speeds.

**Retry implications:**

If failure occurs at fraction \(p\) through the transfer and client restarts from byte 0:

- Expected bytes transferred per successful download ≈ \(S_{\text{zip}} \times (1 + \text{extra})\)  
- If average \(r\) full restarts before success: egress ≈ \((1+r) S_{\text{zip}}\)

**Assumption E-S5 (sensitivity only):** \(r \in \{0, 1, 2\}\).

| ZIP | \(r=0\) egress | \(r=1\) | \(r=2\) |
|-----|----------------|---------|---------|
| 50 MB | 50 MB | 100 MB | 150 MB |
| 100 MB | 100 MB | 200 MB | 300 MB |

Monthly egress if all 30k PDFs downloaded once as singles at size \(s\), plus bulk retry amplification on fraction \(g\) of volume — requires product mix assumptions. Minimal statement:

\[
\text{Egress} \ge (1+r_{\text{eff}}) \times \text{bytes downloaded}
\]

Resumable download or per-PDF fetch reduces \(r_{\text{eff}}\) — design implication, not calculated as a chosen architecture.

**Regenerate-vs-re-fetch:** If mid-download failure triggers **new render**, render cost amplifies by \((1+r)\) on CPU/memory, not only egress — worse under §A/C.

---

# F. Sensitivity analysis

## F.1 Three assumptions with greatest effect on capacity

| Rank | Assumption | Why dominant | Mechanism |
|------|------------|--------------|-----------|
| 1 | **Sustainable fleet concurrency \(C\)** (memory-safe slots) | Sets \(\mu = C/W\); determines whether \(\rho<1\) at peak | One slot moves only \(60/W\) PDFs/min |
| 2 | **Mean render time \(W\)** (2 vs 3 s) and tail | \(L=\lambda W\); 50% more in-flight from 2→3 s; eats 5 s budget under D1 | Linear in Little’s Law; linear in work |
| 3 | **Overlap of single peak with bulk peak + bulk fill factor** | Switches demand from \(1000\)/min to \(1500\)–\(2000\)/min when bulk=100 | Directly scales \(\lambda\) |

Close fourth: **definition D1 vs D2** of “start within 5 s” (changes feasibility without changing \(\mu\)).  
Close fifth: **\(M_{\text{page}}\) for 500+ rows** (reduces \(c_{\text{safe}}\)).

## F.2 Best-case / expected-case / stress-case

All cases use **declared** assumptions.

### Case definitions

| | Best-case | Expected-case | Stress-case |
|--|-----------|---------------|-------------|
| Singles \(\lambda\) | 1000/min | 1000/min | 1000/min |
| Bulk | 0 overlap | 5/min × **50** PDFs (**extra assume** mid fill) | 10/min × **100** |
| Docs/min | 1000 | \(1000+250=1250\) | 2000 |
| \(W\) | 2 s | 2.5 s | 3 s |
| Memory model | P0, 400 MB additive | P1 hyp. ~400–550 effective | P3 / 500+ spikes |
| “Start” def | D2 (queue wait only) | D1 with tight I/O | D1 strict |
| \(\rho\) target | 0.9 | 0.8 | 0.7 |

**Expected-case bulk fill:** Assignment does not say average bulk size; **50** is midpoint of 1..100 for sensitivity only — labeled **ASSUMED**.

### Results table

| Metric | Best | Expected | Stress |
|--------|------|----------|--------|
| \(\lambda\) (docs/s) | \(16.667\) | \(1250/60=20.833\) | \(33.333\) |
| \(L=\lambda W\) | \(33.3\) | \(52.1\) | \(100.0\) |
| \(C\) for \(\rho\) target | \(33.3/0.9\approx37\) | \(52.1/0.8\approx66\) | \(100/0.7\approx143\) |
| Baseline RAM \(L\times400\) MB | \(13.3\) GB | \(20.8\) GB | \(40.0\) GB |
| Baseline RAM at \(C\times400\) | \(14.8\) GB | \(26.4\) GB | \(57.2\) GB |
| Slack under D1 if wait=0 | \(5-2=3\) s | \(5-2.5=2.5\) s | \(5-3=2\) s |
| 5 s D1 feasible without measure? | **Possibly** if I/O≪1s & \(C\ge37\) | **Uncertain** — needs measure | **Unlikely** at stated overlap without large \(C\) + tiny I/O |
| 5 s D2 feasible | If \(B/\mu \le 5\) | Same with larger \(C\) | Needs \(\mu>\lambda\) first |

**Per-slot throughput reminder:** at stress \(W=3\), each slot = 20 PDFs/min → \(143\) slots ⇒ \(\mu \approx 2860\) PDFs/min > 2000 demand.

## F.3 What must be measured in a real load test

| Measurement | Why |
|-------------|-----|
| Render time histogram for real templates (p50/p95/p99), by line-item count | Replaces flat 2–3 s |
| RSS/PSS vs concurrency; \(M_{\text{base}}\), \(M_{\text{page}}\) | Replaces P0–P3 hypotheses |
| OOM rate at 500+ rows; peak RSS before kill | Graceful-handling design input |
| \(T_{\text{persist}}+T_{\text{url}}\) | D1 budget |
| CPU saturation vs \(c\) on chosen vCPU shape | Effective \(\mu < C/W\)? |
| Arrival process / burst duration \(D\) from production-like ERP traffic | Queue drain design |
| ZIP \(\alpha\); client bandwidth; mid-download failure rate | Egress & retry |
| Object storage latency/errors | Complete-vs-bytes invariant |

---

# Recommendation: initial worker concurrency + validation plan

## Initial concurrency (engineering starting point — not an SLA claim)

**Recommendation:** Start with **fleet-wide in-flight cap \(C = 8\) to \(12\)**, and **per-host \(c \le 2\) to \(3\)**, until measurements replace hypotheses.

### Why this range (derived reasoning, not “the system meets 1000/min”)

1. Under **P0** additive 400 MB: \(c=3\) ⇒ \(\approx 1.2\) GB render-side before OS/base — a cautious probe band on a small VM.  
2. \(C_{\text{fleet}}=8\)–\(12\) at \(W=2.5\) s yields  
   \(\mu = C/W \approx 8/2.5\)–\(12/2.5 = 3.2\)–\(4.8\) PDFs/s ≈ **192–288 PDFs/min** — enough to exercise queueing, progress, OOM, and idempotency **without** pretending to absorb \(1000\)/min.  
3. Assignment peak \(L \approx 33\)–\(50\) (singles) **exceeds** this initial \(C\) → **admission control / backpressure must be tested from day one**; do **not** claim 5 s D1 at full peak until \(C \ge \sim 1.25 L\) **and** memory measurements allow it.  
4. Raising \(C\) toward \(42\)–\(63\) (singles, \(\rho\le0.8\)) is a **scale-up gate** gated on §F.3 measurements and the ₹12,500 cost model — not an initial setting.

**Explicit non-claim:** This document does **not** assert that any concurrency meets **1000/min** and **5 s** and **₹12,500** simultaneously; §A–C show those goals **conflict** under literal additive 400 MB math.

## Validation plan

| Stage | Action | Pass criteria (examples) |
|-------|--------|---------------------------|
| 1. Microbench | 1 worker, \(c=1..4\), templates 5 / 100 / 500+ rows | Record \(W\), peak RSS; no host OOM at chosen \(c\) |
| 2. Cap confirm | Fix \(c_{\text{safe}}\) from measured RSS + \(H=0.25\) | \(c_{\text{safe}}\) stable over 30 min soak |
| 3. Throughput | Increase job rate until \(\rho\to0.8\) at fixed \(C\) | Measured \(\mu \approx C/\mathbb{E}[W]\); note CPU throttle gap |
| 4. Latency budget | Instrument \(T_{\text{wait}}, W, T_{\text{persist}}\) | Under load where \(\mu>\lambda\), check D1 or D2 per declared definition |
| 5. Overload | Drive \(\lambda > \mu\) | Backlog grows at \(\lambda-\mu\); admission rejects; **no** unbounded memory from queue payloads |
| 6. Spike/recovery | Burst \(D\) (try 60s, 180s) | Measure \(B(D)\), \(T_{\text{drain}}\); compare to §D formulas |
| 7. Bulk | Bulks of 50 and 100 with child concurrency \(1..c\) | Parent progress correct; singles priority policy verified if implemented |
| 8. Failure | Kill worker mid-job; duplicate delivery; abort ZIP at 50% | Idempotent artifact; re-download without re-render; status monotonic |
| 9. Scale gate | Only if memory/cost allow, raise \(C\) toward \(1.25 L\) for target scenario (S0 or C5) | Re-check ₹ budget; only then discuss peak SLA |

---

## Summary formulas cheat-sheet

\[
\begin{align*}
\lambda_s &= \lambda_{\min}/60 \\
L &= \lambda_s \cdot W \\
\mu &= C / W \\
\rho &= \lambda_s / \mu \\
B(t) &= B(0) + (\lambda_s - \mu)t \quad (\lambda_s>\mu) \\
\text{Mem}_{\text{baseline}} &= L \times 400\ \text{MB} \\
S_{\text{month}} &= 30000 \times s \\
S_{\text{zip}} &\approx \alpha K s
\end{align*}
\]

---

*End of capacity math. Next (when requested): map these numbers onto cost under ₹12,500 and compare architecture options — still gated on measured \(M_{\text{base}}, M_{\text{page}}, W\), and declared assumptions D1/D2, burst \(D\), archive TTL.*
