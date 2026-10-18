# Audit of `CAPACITY_MATH.md`

> Purpose: verify unit conversions, concurrency math, bulk accounting, and whether assumptions were smuggled in as facts.  
> Scope: critique of our capacity document against the TranZact assignment. **Still not an architecture.**

**Verdict in one line:** Core Little’s Law rates (1000/min → 16.667/s; \(L \in \{33.3, 50\}\) at 2–3 s) are correct; several **GB conversions are inconsistent**, one **GiB storage figure is wrong**, **queue “age” is mis-labeled**, and **peak vs 30,000/month must not be treated as one traffic distribution**.

---

## 1. Unit conversions

### 1.1 Seconds ↔ minutes — OK

| Check | Result |
|-------|--------|
| \(1000/60 = 50/3 \approx 16.667\ \mathrm{s}^{-1}\) | Correct |
| Bulk \(500/60 \approx 8.333\ \mathrm{s}^{-1}\), \(1000/60 \approx 16.667\ \mathrm{s}^{-1}\) | Correct |
| \(\mu_{\mathrm{slot}} = 60/W\) PDFs/min | Correct |
| \(30 \times 24 \times 60 = 43{,}200\) min/month | Correct **given** 30-day month assumption |

No silent minute/second swaps found in the rate formulas.

### 1.2 MB vs MiB / GB vs GiB — **errors and inconsistency**

**Assignment wording:** “~400MB”, “16GB RAM”, “100KB–1MB”. It does not define decimal vs binary.

**What `CAPACITY_MATH.md` did:**

| Location | What was written | Actual arithmetic | Issue |
|----------|------------------|-------------------|-------|
| Global F9 | \(16\ \mathrm{GB} = 16{,}384\ \mathrm{MB}\) | \(16 \times 1024\) | Treats RAM GB as **GiB** and MB as **MiB** without labeling |
| §A.3 | \(13{,}333.3 \rightarrow 13.33\ \mathrm{GB}\) | \(13333.3/1000\) | **Decimal** GB |
| §A.3 | \(16{,}666.7 \rightarrow 16.67\ \mathrm{GB}\) | \(16666.7/1000\) | **Decimal** GB |
| §A.3 | \(20{,}000 \rightarrow 19.53\ \mathrm{GB}\) | \(20000/1024\) | **Binary** GiB — **inconsistent with the two rows above** |
| §B.2 | \(6{,}667\ \mathrm{MB} \rightarrow 6.51\ \mathrm{GB}\) | \(6667/1024\) | Binary |
| §B.2 | \(13{,}333 \rightarrow 13.02\ \mathrm{GB}\) | \(13333/1024\) | Binary |
| §E.1 storage | Explicit decimal GB (\(10^9\)) and separate GiB table | Intentional split | OK **if** readers only use the matching table |

**Incorrect / inconsistent calculations (render memory → “GB”):**

Correct **pairs** if \(400\) and products are kept in the same unit as F9’s \(16{,}384\):

| \(W\) | \(L \times 400\) “MB” | ÷1000 (decimal GB) | ÷1024 (GiB) |
|-------|----------------------|--------------------|-------------|
| 2 s | \(13{,}333.3\) | **13.33** | **13.02** |
| 2.5 s | \(16{,}666.7\) | **16.67** | **16.28** |
| 3 s | \(20{,}000\) | **20.00** | **19.53** |

`CAPACITY_MATH.md` §A.3 mixed columns: decimal for 2 s and 2.5 s, binary for 3 s. **That is an error.**

**Impact on the “exceeds 16 GB host” claim:**

- Binary compare at \(W=2.5\): \(16.28\ \mathrm{GiB} > 16\ \mathrm{GiB}\) → still exceeds (narrowly).  
- Decimal compare at \(W=2.5\): \(16.67\ \mathrm{GB}\) vs “16 GB” depends on whether host RAM is 16 GiB — usually yes for EC2.  
- Conclusion “exceeds host” **directionally survives**, but the **stated 16.67 vs 19.53 figures must not be trusted as a consistent series**.

### 1.3 Monthly storage arithmetic

**Decimal table (§E.1)** — correct under E-S3 (\(1\ \mathrm{KB}=10^3\ \mathrm{B}\), \(1\ \mathrm{MB}=10^6\ \mathrm{B}\)):

| Size | ×30,000 | GB (\(10^9\)) |
|------|---------|---------------|
| 100 KB | \(3.0\times10^9\) B | **3.0** OK |
| 500 KB | \(1.5\times10^{10}\) B | **15.0** OK |
| 1 MB | \(3.0\times10^{10}\) B | **30.0** OK |

**Binary GiB table — one clear error:**

| Claim in doc | Correct value | Status |
|--------------|---------------|--------|
| 100 KiB × 30k ≈ **2.79 GiB** | \(30{,}000 \times 100 \times 1024 / 2^{30} = \mathbf{2.861\ldots\ GiB}\) | **Incorrect** |
| 500 KiB × 30k ≈ **14.0 GiB** | \(\mathbf{14.305\ldots\ GiB}\) | **Understated** (rounding too aggressive) |
| 1 MiB × 30k ≈ **29.3 GiB** | \(30{,}000/1024 = 29.296875\) | OK |

**Hot-storage formula** \(\mathrm{Hot} \approx S_{\mathrm{month}} \times TTL_{\mathrm{hot}}/30\): arithmetically fine **only** under the same 30-day month assumption; not an assignment fact.

**ZIP sizes** (α=1): \(100 \times 500\ \mathrm{KB} = 50{,}000\ \mathrm{KB}\). Calling that “50 MB” is fine if KB→MB means ÷1000; if KiB→MiB, \(50000/1024 \approx 48.8\ \mathrm{MiB}\). Minor labeling imprecision, not a capacity-breaker.

---

## 2. Render concurrency math — OK with caveats

| Formula | Check |
|---------|-------|
| \(L = \lambda_s W\) with \(\lambda_s=1000/60\) | **Correct** → \(L=33.333\) (2 s), \(50\) (3 s) |
| \(\mathrm{Mem_{baseline}} = L \times 400\) | Correct **as a baseline identity**, not as measured RSS |
| \(\mu = C/W\) (jobs/s) | Correct for ideal parallel slots |
| \(C \ge 1.25 L\) for \(\rho\le 0.8\) | Algebra OK; **0.8 is an ops assumption**, not FACT |
| Slot work: \((\mathrm{docs/min})\times W / 60 = L\) | Correct consistency check |

**Garbled line (not wrong answer, bad writing)** in §C.1:

> \(C_{\mathrm{fleet}} \ge L = \lambda_{\min} \times (W/60) \times 60\ \mathrm{wait} = \lambda_{\mathrm{per\ s}}\times W\)

The stray word “wait” is nonsense. The intended identity is \(L=\lambda_s W\). Readers should ignore the middle fragment.

**Unproven performance implied by ideal \(\mu=C/W\):** assumes no CPU contention, no browser startup, 100% parallel efficiency (C-S1). That can **overstate** achievable throughput on 2 vCPU-class hardware — flagged in the doc, but any table that lists “PDFs/min at \(c=5\)” still **looks** like promised capacity. Those rows are **ceilings under C-S1**, not demonstrated rates.

---

## 3. Bulk demand — double-counted or omitted?

### 3.1 What was done

| Scenario | Docs/min | Accounting |
|----------|----------|------------|
| S0 | 1000 | Singles only |
| B5 / B10 | 500 / 1000 | Bulk only, **assuming 100 PDFs every bulk** (B-S1) |
| C5 / C10 | 1500 / 2000 | Singles **plus** bulk docs |

### 3.2 Double-counting?

**Not double-counting the same PDF** in C5/C10: a single-document request and a bulk child are different generation jobs **if** both traffic classes run at once.

**What *was* overstated as “the” peak:**

1. **B-S1** (every bulk = 100 PDFs) is the **maximum**, not the expected fill. Using it inside C5/C10 without always repeating “max fill” inflates demand.  
2. **B-S3 overlap** (singles peak ∧ bulk peak simultaneously) is **not stated** in the assignment. The assignment lists both rates under “Expected load” but does **not** say they coincide. Treating C10 as required capacity would be a **silent escalation**.  
3. Bulk was **not omitted** from stress scenarios; it **was omitted** from the monthly average decomposition (correctly — assignment gives no split).

### 3.3 Requests vs documents (silent unit shift)

| Assignment phrase | Unit |
|-------------------|------|
| “~1000 **single requests**/min” | requests/min |
| “5–10 **bulk requests**/min” | bulk requests/min |
| “~30,000 **PDFs**/month” | documents/month |

`CAPACITY_MATH.md` **Assumption A-S2:** one single request → one PDF. That is **plausible but not explicit**. If some “single requests” are status/download polls, peak **render** λ would be lower. Conversely, bulk was correctly expanded requests → documents via ×100 under B-S1.

**Finding:** Bulk math is coherent when scenarios are labeled; the risk is **presenting C5/C10 as fact-required peak** rather than **optional overlap × max-fill stress**.

---

## 4. Incorrect calculations (summary list)

| ID | Location | Error | Correction |
|----|----------|-------|------------|
| E1 | §A.3 GB column | Mixed ÷1000 and ÷1024 | Pick one convention; publish both columns |
| E2 | §E.1 binary table | 2.79 GiB for 100 KiB×30k | **≈ 2.86 GiB** |
| E3 | §E.1 binary table | ≈14.0 GiB for 500 KiB×30k | **≈ 14.31 GiB** |
| E4 | §D.1 / §D.2 | Calls \(B/\mu\) “max queue age” \(A\) | In a fluid spike from empty, **oldest wait ≈ spike elapsed time**; \(B/\mu\) is **waiting time for a new arrival** (or remaining busy delay), not the same as oldest age at time \(t\) |
| E5 | §C.1 prose | Broken formula with “wait” | Cosmetic; fix wording |

**Example for E4:** §D.1 after 60 s with \(B=600\), \(\mu=6.667\ \mathrm{s}^{-1}\):

- Doc: \(A \approx 90\ \mathrm{s}\).  
- Age of oldest waiting job ≈ **60 s** (been waiting since backlog began).  
- A job arriving at \(t=60\) expects wait ≈ **90 s**.  

Same issue in §D.2: after a 120 s spike, oldest age ≈ **120 s**, not \(B/\mu \approx 46.7\ \mathrm{s}\). The 46.7 s figure is still useful — just **misnamed**.

**Not errors:** Little’s Law table; backlog growth \(B(t)=(\lambda-\mu)t\); drain \(T_{\mathrm{drain}}=B/(\mu-\lambda_{\mathrm{normal}})\); decimal storage 3/15/30 GB; 1440 peak-to-average **ratio arithmetic** (interpretation challenged in §7).

---

## 5. Assumptions silently treated as facts

Labeled assumptions that are fine when labeled; problems arise when later prose treats them as given.

| Item | Was it labeled? | Silent-fact risk |
|------|-----------------|------------------|
| Exact \(1000\) not “≈1000” | A-S1 yes | Low |
| \(W=2.5\) midpoint | A-S4 yes | Medium — reused in §D.2 / recommendation without always repeating “midpoint” |
| One single request = one render | A-S2 yes | Medium — unit bridge to PDFs/month |
| Steady state at peak (service keeps up) | A-S3 yes | High if reader thinks \(L\) is “what you will observe” even when under-provisioned (doc does warn) |
| Bulk always 100 PDFs | B-S1 yes | High if C5/C10 cited as “required” |
| Single peak overlaps bulk peak | B-S3 yes in table | High in sensitivity “stress-case” branding |
| 30-day × 24 h uniform month | B-S4 yes | Medium — calendar months differ; business hours peakier |
| \(\rho\le 0.8\) / \(1.25L\) | Example only | Medium — looks like a requirement |
| D1 = first byte by 5 s | C-S3 “definition fork” | **High** — only one of several readings (see §6) |
| D2 = render started by 5 s | C-S3 | Same |
| P1–P3 memory splits (200/200 etc.) | Hypothesis table | Low if unread as measurements |
| 8 GB worker, \(H=0.25\), \(M_{\mathrm{erp}}=8\) GB | Explicit illustration | Medium if copied into cost/architecture as need |
| ZIP \(\alpha=1.0\) | E-S4 | Low |
| Expected-case bulk size 50 | F.2 labeled | Medium |
| F9 \(16\ \mathrm{GB}=16384\ \mathrm{MB}\) | Presented under **[FACT]** inputs | **Yes — silent:** binary expansion is convention, not assignment text |
| Comparing \(L\times 400\) to entire monolith 16 GB as if PDF service owns that RAM | Rhetorical compare | **Silent framing:** assignment does not require co-locating workers on that host |

---

## 6. What “single downloads to start within 5 seconds” can mean

Assignment text **[FACT]:** “Single downloads to start within 5 seconds.”

It does **not** define “start.” At least **three** distinct SLIs exist:

| ID | Interpretation | Clock stops when… | Includes \(W=2\)–\(3\) s render? |
|----|----------------|--------------------|----------------------------------|
| **R1 — Response start** | HTTP response to the generate/download request begins (e.g. \(202\) + job id, or headers) | First response byte / status committed | **No** |
| **R2 — Render begin** | Job leaves the queue and Puppeteer starts | Worker claims job | **No** (only queue wait) |
| **R3 — Transfer start** | Client can begin receiving PDF bytes (file ready or streaming starts) | First PDF byte (or redirect to object) | **Yes** (unless streaming while rendering — not stated) |

**Mapping to `CAPACITY_MATH.md`:**

| Doc definition | Matches |
|----------------|---------|
| D1 “first byte of PDF available” | **R3** (transfer / artifact ready) |
| D2 “generation started (left queue)” | **R2** |
| *(missing)* | **R1** response start |

**Critical implication:** Under **R3/D1**, render duration alone consumes **2–3 s** of the **5 s** budget; feasibility is tight even at zero queue. Under **R1**, the 5 s target is mostly API latency and is **much easier**, but may disappoint users who hear “download.” Under **R2/D2**, 5 s is a **queue-wait** SLO.

**Audit finding:** The capacity doc correctly called ambiguity out, but then leaned on D1/D2 only. **R1 was omitted.** No calculation may treat R3 as FACT. Any claim that “5 s conflicts with 2–3 s render” is valid **only under R3** (or streaming-not-allowed).

---

## 7. Is 30,000 PDFs/month consistent with peak request rates?

**Do not assume they describe the same traffic distribution.** The assignment presents them in different roles:

| Figure | Role in assignment |
|--------|---------------------|
| ~1000 single/min and 5–10 bulk/min | “Expected load … during **peak times**” |
| ~30,000 PDFs/month | **Budget / cost** scenario (“customer generating approximately…”) |

### 7.1 Arithmetic tension if forced onto one distribution

Uniform average from 30k/month (30-day assumption):

\[
\lambda_{\mathrm{avg}} = 30{,}000 / 43{,}200 \approx 0.694\ \mathrm{PDFs/min}
\]

If **singles peak** \(1000\ \mathrm{PDFs/min}\) were sustained for a full month:

\[
1000 \times 43{,}200 = 43{,}200{,}000\ \mathrm{PDFs/month}
\]

≈ **1,440×** the stated monthly volume.

Minutes of **pure singles-peak** needed to produce 30k PDFs (if all volume were at that rate, zero elsewhere):

\[
30{,}000 / 1000 = 30\ \mathrm{minutes/month}
\]

If bulk max (10×100 = 1000 docs/min) ran alone at peak: same order — **~30 minutes/month** of such peak exhausts 30k.

If C10 overlap (2000 docs/min) were the peak shape: **15 minutes/month** of that peak exhausts 30k.

### 7.2 Consistent readings (none proven; all possible)

| Hypothesis | Consistency |
|------------|-------------|
| **H1:** 30k/month customer ≠ customer who hits 1000/min peak | Compatible — different scenarios in one brief |
| **H2:** Same customer; peak lasts only tens of minutes per month | Compatible with volume; still must size for peak **or** intentionally degrade |
| **H3:** “1000/min” is a design/target load for the service generation, 30k is a cost reference | Compatible; peak-to-average ratio is **not** an empirical PAR |
| **H4:** Same distribution, continuous peak | **Incompatible** with 30k/month |

**Finding:** `CAPACITY_MATH.md` correctly noted orders-of-magnitude gap and budget tension, but the phrase “peak-to-average ratio ≈ 1440” **implies** one series. Safer statement: **“If these were the same distribution under a uniform average, the ratio would be ~1440; the assignment does not establish that they are.”**

Using \(\lambda_{\mathrm{normal}} = 0.01157\ \mathrm{s}^{-1}\) from 30k/month inside the §D.2 spike recovery example **silently adopts H2-style linkage** (post-spike return to monthly average). That is a **scenario assumption**, not a measured diurnal pattern.

---

## 8. Estimates that imply impossible or unproven performance

| Estimate / statement | Problem |
|----------------------|---------|
| Ideal PDFs/min for \(c=5\) (150 at \(W=2\)) | Unproven without CPU/RSS tests; easy to read as capacity |
| “Possibly” D1 OK if I/O ≪ 1 s and \(C\ge 37\) | Soft claim; **unproven**; depends on R3 + measurements |
| \(C \approx 125\) for C10@\(W=3\), \(\rho\le0.8\) | Math OK; **memory baseline ~50 GB** under P0 makes “just provision it” **cost/RAM-implausible** under ₹12,500 without further proof — doc notes tension, good |
| \(\mu = C/W\) during bulk+single peak | Ignores ZIP build, storage, Django, Chromium inefficiency |
| P2/P3 \(c_{\mathrm{safe}}\) on 8 GB | Fine as sensitivity; **not** measured |
| Initial \(C=8\)–\(12\) recommendation | OK as probe; must **not** be read as meeting 1000/min (doc forbids that — keep it that way) |
| Treating \(L\times 400\) as mandatory RAM bill | Baseline only; real RSS may be higher **or** sub-additive with shared browser — either direction **unproven** |

**Nothing in the doc correctly claims** full peak + 5 s R3 + ₹12,500 simultaneously. Good. The hazard is **tables of ideal throughput** without the word “ceiling.”

---

## 9. Corrected reference snippets (for future edits)

**§A.3 memory — consistent columns:**

| \(W\) | \(L\) | \(L\times 400\) MiB-units | Decimal GB (/1000) | GiB (/1024) |
|-------|-------|---------------------------|--------------------|-------------|
| 2 | 33.333 | 13,333 | 13.33 | 13.02 |
| 2.5 | 41.667 | 16,667 | 16.67 | 16.28 |
| 3 | 50 | 20,000 | 20.00 | 19.53 |

**§E.1 binary storage:**

| Size | Total GiB |
|------|-----------|
| 100 KiB × 30k | **≈ 2.86** |
| 500 KiB × 30k | **≈ 14.31** |
| 1 MiB × 30k | **≈ 29.30** |

**Queue naming:**

- \(A_{\mathrm{oldest}}(t) \approx t\) during continuous saturation from empty (fluid).  
- \(T_{\mathrm{wait,new}}(t) \approx B(t)/\mu\).

---

## 10. Audit checklist (pass/fail)

| Check | Result |
|-------|--------|
| s ↔ min conversions for stated peaks | **Pass** |
| Little’s Law singles concurrency | **Pass** |
| Bulk max-fill expansion 5–10 × 100 | **Pass** (as stress) |
| Bulk omitted from monthly split | **Pass** (unknown → omitted) |
| Bulk double-counted with singles | **Pass** if overlap labeled; **Fail** if treated as required FACT |
| MB/MiB consistency in §A.3 | **Fail** |
| GiB storage 100 KiB row | **Fail** (2.79 → 2.86) |
| Queue age vs \(B/\mu\) | **Fail** (mislabel) |
| 5 s definition pinned to one meaning | **Fail as FACT**; **Pass as ambiguity discussion** (add R1) |
| 30k/month same distribution as peak | **Must not assume** — doc partially implies via PAR and §D.2 |
| Silent binary 16 GB = 16384 MB as FACT | **Fail** (convention, not assignment text) |
| Unproven ideal parallel efficiency | **Caution** |

---

## 11. What to change in `CAPACITY_MATH.md` when editing next

1. Fix §A.3 GB column consistency; label MiB/GiB vs MB/GB.  
2. Fix §E.1 GiB values (2.86 / 14.31).  
3. Rename queue “age” vs “wait for new arrival.”  
4. Add **R1/R2/R3** for the 5 s objective; stop letting D1 stand as the default FACT.  
5. Demote C5/C10 and PAR≈1440 to **“if overlap / if same distribution.”**  
6. Mark F9 binary expansion as **convention**, not assignment quote.  
7. Prefix ideal \(c\)-throughput tables with **“theoretical ceiling under C-S1.”**

---

*End of audit. Numbers that remain trustworthy for later design work: \(\lambda_s=1000/60\); \(L\in[33.3,50]\) for singles-only; bulk max add-on \(500\)–\(1000\) docs/min; decimal monthly storage \(3/15/30\) GB; backlog fluid model \(dB/dt=\lambda-\mu\).*
