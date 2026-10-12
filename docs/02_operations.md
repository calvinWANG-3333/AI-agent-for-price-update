<sub>[README](../README.md) &nbsp;·&nbsp; [01 SETUP](01_setup.md) &nbsp;·&nbsp; **02 OPERATIONS** &nbsp;·&nbsp; [03 SPECIFICATION](03_specification.md)</sub>

# 02 OPERATIONS

How to prepare the input, read the output, and what to expect on a real run.

## INPUT PREPARATION GUIDE (`To_be_update.xlsx`)

Always refer to Ruihang's sample input file **Sample_To_be_update.xlsx**. The agent expects a hand-prepared Excel file named **exactly** `To_be_update.xlsx` in the project root. The quality of the run **depends entirely on the quality of this file.**

### 1. Source of the input file
Start from the standard `evolution` sheet produced by the legacy Price Check report (`price_check_tool_all_brands_-_VXII.py`). The agent consumes the **same column schema**: `Url`, `Name`, `Brand`, `Market`, `currency_s`, `Original Price_validated`, `status`, `Remarks L1`, `Remarks L2 (details)`. **Do not rename, reorder, or delete these columns** — the writer locates `Remarks L1` and `Remarks L2 (details)` by header name and will refuse to start if either is missing. Manually create a new excel file called `To_be_update.xlsx` and copy the header from report genrated by `price_check_tool_all_brands_-_VXII.py` then copy the rows you want agent to check and paste it to `To_be_update.xlsx` following the rules below in Mandatory pre-filter chapter. When copy and paste rows from price check report, due to the fact that some columns are hidden and pasting may unhide the columns, make sure what you paste align with the header or you will waste the API usage.

### 2. Mandatory pre-filter (manual, before saving the file)

> [!CAUTION]
> **The agent is not a replacement for the human review filter.** Skipping the steps below will either waste API budget on rows that don't need verification, or worse, write the wrong Remark to rows the agent was never validated against.

Apply these filters in the order listed:

| # | Action | Reason |
| --- | --- | --- |
| 1 | **Keep only `status == "Missing in crawl"` OR `Remarks L1 == "Refresh data"`** | These are the only two row types the reconciliation logic in `reconcile.py` knows how to handle: **refresh data(data outside of delivery range) and pure missing in crawl data(with status of missing in crawl and no compensating pair in new addition)** . Any other row will be skipped. |
| 2 | **Drop every row where `Brand == "Loewe"`** | Loewe requires region switching that the headless browser cannot perform. Loewe rows are 100% human-owned. |
| 3 | **Drop `Audemars Piguet` (AP) rows** | AP data comes directly from the brand feed, not from a re-crawl, so it must not be flagged as `Refresh data` in the first place. |
| 4 | **Drop `Hermès` rows where data originates from in-store collection** | Per the onboarding guide §3.2.3, Hermès store data must not be modified. |
| 5 | **Drop rows already reviewed by a human in this delivery cycle** | For **missing in crawl data** If `Remarks L1` or `Remarks L2 (details)` already contains a non-blank manual note, or for **refresh data** if `Remarks L2 (details)` already contains info, the agent will overwrite it. Filter these out. In another word: For **missing in crawl data**, keep `Remarks L1` or `Remarks L2 (details)` blank, for **refresh data**, keep `Remarks L2 (details)` empty and `Remarks L1` remained as refresh data |
| 6 | **Save as `To_be_update.xlsx` in the project root** | The filename is enforced by `run_demo.py`. |

The runner performs a **pre-flight scan** at startup and prints a warning if any rows from `EXCLUDED_BRANDS` slipped through, but it will not auto-remove them — the operator is expected to know.

### 3. Sanity-check checklist before launching

Run through this mentally before typing `python run_demo.py`:

- [ ] File is named exactly `To_be_update.xlsx` (case-sensitive on macOS/Linux).
- [ ] File sits in the same directory you will launch the script from.
- [ ] Loewe rows: 0.
- [ ] AP rows: 0.
- [ ] Every remaining row has a non-blank `Url`, `Original Price_validated`, and `currency_s`.
- [ ] You have a sense of the row count — a 500-row file will take roughly 1.5–3 hours wall-clock at `MAX_CONCURRENT = 3`.

## OUTPUT INTERPRETATION GUIDE (`To_be_update_filled.xlsx`)

The runner **never modifies the input file in place**. It writes a new file named `<input_stem>_filled.xlsx` next to the input — for the canonical `To_be_update.xlsx`, this is `To_be_update_filled.xlsx`.

### 1. What gets written

Only two columns are touched: `Remarks L1` and `Remarks L2 (details)`. Everything else in every row is copied from the source verbatim, including original formatting, column widths, filters, and any pre-existing cell fills.

### 2. Cell color legend

The writer applies a fill color to the two updated cells based on the reconciliation decision label:

| Color | Hex | Meaning | What you should do |
| --- | --- | --- | --- |
| Light green | `#C6EFCE` | `AUTO_*` decision — agent saw a high-confidence price that matched, or a same-price refresh, with no anomalies. | Spot-check ~5%. If clean, accept the whole batch. |
| Light yellow | `#FFEB9C` | `LOW_CONFIDENCE_NEEDS_REVIEW` — the agent extracted something, but its self-rated confidence was `medium` or `low`, so the row was escalated. The original `Remarks L1` is preserved and `Remarks L2 = "manual"`. | Open the URL yourself and confirm the price. The agent's structured log will already tell you why it wasn't sure. |
| Light red | `#F4CCCC` | `AUTO_PRODUCT_GONE` / `AUTO_CONFIRMED_MISSING` — the URL redirected to a homepage / category / search page, meaning the product is no longer on sale. | Sanity-check by opening the URL in a real browser. If it really is dead, accept; otherwise treat as manual. |

### 3. Crash-safety guarantee

`run_demo.py` flushes partial progress to `To_be_update_filled.xlsx` every `CHECKPOINT_EVERY` (default: 10) completed rows. **If the run is interrupted** (network drop, API rate limit, browser hang, accidental Ctrl-C), every checkpointed row is already on disk and you only lose the in-flight `MAX_CONCURRENT` rows. To resume after a partial run, simply launch the script again — but be aware that the current implementation **does not skip already-completed rows**; it will re-process them. For now, the recommended pattern is: let the full run complete, accept that the few in-flight rows at crash time are wasted API calls, and live with it.

## KNOWN OPERATIONAL ISSUES (READ BEFORE FIRST RUN)

These are not bugs — they are real-world behaviors a colleague will encounter on the first run and may misinterpret as a broken script.

### 1. JavaScript-rendered prices (Dior, Gucci, others)

The agent runs in **DOM mode** (`use_vision=False`). It reads the page's HTML structure directly, not a rendered screenshot. This is 3–4× cheaper and 2× faster than vision mode, but it has one real cost:

> Some luxury sites — **Dior is the worst offender** — inject the price into the DOM via client-side JavaScript with a noticeable delay. On a slow connection, the agent can finish reading the page **before** the price node has been injected. From the agent's perspective: it loaded the URL, the product name was there, but the price field was empty. It correctly reports `confidence = medium` with evidence like `"page seems to have rendered only partially"`, and `reconcile.py` escalates the row to manual review.

This is **expected behavior**, not a failure. Mitigations already in place:
- `minimum_wait_page_load_time=3.0` in `agent_1.py` forces a 3-second floor on page-load wait.
- The confidence rubric explicitly distinguishes "price found in a fully-rendered page" from "price found but page may have only partially rendered."

If you see a Dior-heavy batch with a manual escalation rate above ~40%, the network was likely slow during the run. Re-running those specific rows usually resolves them.

### 2. The terminal log is interleaved and unreadable

With `MAX_CONCURRENT = 3`, three independent agents print to the same stdout. You will see this kind of output:

```
======================================================================
[1/89]  Excel row 14  |  Dior / FRA
URL: https://www.dior.com/...
======================================================================
Row type: refresh_data

→ Agent (Claude) running...

======================================================================
[2/89]  Excel row 22  |  Chanel / USA
URL: https://www.chanel.com/...
======================================================================
Row type: refresh_data
  price_found = 2000.0  confidence = high  redirected = False   ← from [1/89]
→ Agent (Claude) running...

→ Decision: AUTO_REFRESH_SAME_PRICE  (62.3s)    ← from [1/89]
```

**Do not try to read the log linearly.** The final summary table at the end is the authoritative view; the row-by-row log is best-effort. If you must trace a single row, grep for its Excel row index (`Excel row 22`).

### 3. Multiple Chromium windows pop up on your screen

`browser-use` runs Chromium in **headed** (visible) mode by default. With `MAX_CONCURRENT = 3`, three Chromium windows will appear, layered on top of each other, scrolling autonomously. **Do not click them, do not close them, do not switch focus** — Playwright is driving them and any human interaction can derail the agent. Minimize them if they bother you; closing one kills the corresponding task.

### 4. Cookie banners, region selectors, and anti-bot interstitials

Luxury sites frequently display one of these before the product page renders, especially when the request originates from a region different from the URL's market (e.g., a French laptop opening a `/en_us/` URL):

- "Accept cookies" banner — agent prompt instructs it to click through.
- "You are visiting from X, continue to Y?" — agent prompt instructs it to stay on the original URL.
- Cloudflare / Akamai / PerimeterX block screens — agent returns `evidence = "BLOCKED: ..."` and the row is escalated.

Markets behind a geo-block (e.g., a `ko_kr` URL accessed from outside Korea) will sometimes never render and the agent will time out. These rows are the agent's natural failure mode and need either a VPN-equipped human or a re-run from a machine in the right region.

### 5. Anthropic per-minute rate limits

Anthropic enforces a per-minute token budget that depends on your account tier:
- **Tier 1** (default, no top-up): ~50K input + ~10K output tokens/minute.
- **Tier 2** (after ~$40 in spend): roughly 4× higher.

Each agent call uses ~3–5K input + ~200 output tokens. `MAX_CONCURRENT = 3` consumes ~15K input/min — well within Tier 1. **`MAX_CONCURRENT = 5` consumes ~25K input/min and is still safe on Tier 1, but risky on a freshly created account.** If you see HTTP 429 errors in the log, lower `MAX_CONCURRENT` and re-run.

### 6. The same URL can give different confidence levels on different runs

The agent is not deterministic. The same URL, run twice, can produce `confidence = high` once and `confidence = medium` the other time — typically because the page rendered differently between the two attempts (different network speed, different cookie state, different Cloudflare challenge outcome). **Do not treat a single low-confidence row as proof that the agent is wrong about that product.** Re-run before reporting an issue.

## TROUBLESHOOTING & FAILURE MODES

<details>
<summary><b>1. Playwright throws <code>Error: Only Archer/Mac-Arm64 is supported</code> during install</b></summary>
<br>
<b>Root Cause:</b> Architecture Mismatch. Your physical machine is an Apple Silicon Mac, but either your active Terminal application or the Python interpreter instance is locked under Intel (x86_64) translation.<br>
<b>Resolution:</b> Re-run <b>macOS Setup -> Step 1 through Step 3</b> to clear structural pollution and force-rebuild your runtime layer using the explicit <code>macos-aarch64</code> parameter.
</details>

<details>
<summary><b>2. Terminal shell stubbornly prefixes <code>(base)</code> even after explicit activation</b></summary>
<br>
<b>Root Cause:</b> Anaconda / Miniconda global shell hooks take visual priority in prompt layouts.<br>
<b>Resolution:</b> This is purely cosmetic. Run <code>which python</code> (macOS) or <code>where.exe python</code> (Windows). If the generated string points directly into your local repository's <code>.venv</code> path, your current execution thread is correctly sandboxed. You can safely ignore the Conda prefix visual bug.
</details>

<details>
<summary><b>3. Execution fails immediately with <code>No such file or directory (os error 2)</code> targeting <code>playwright</code></b></summary>
<br>
<b>Root Cause:</b> Missing path symmetry inside the <code>uv</code> runner ecosystem.<br>
<b>Resolution:</b> Ensure <code>playwright</code> is explicitly declared as a top-level requirement inside your <code>requirements.txt</code> file, and re-execute <code>uv pip install -r requirements.txt</code> to generate the required binary symlinks in the local environment's <code>bin/</code> path.
</details>

<details>
<summary><b>4. The script appears to hang for 30+ seconds after launch with no output</b></summary>
<br>
<b>Root Cause:</b> Playwright is downloading the Chromium binary on first use, or browser-use is performing a one-time initialization handshake with the Anthropic API.<br>
<b>Resolution:</b> Wait. First-launch overhead is normal. If nothing prints after 60 seconds, kill the process and re-run <code>uv run playwright install chromium</code> to confirm the browser binary is locally available.
</details>

<details>
<summary><b>5. The run finishes but <code>To_be_update_filled.xlsx</code> has a high manual-escalation rate (&gt;30%)</b></summary>
<br>
<b>Root Cause:</b> Almost always a JavaScript rendering issue (see Known Operational Issues §1). A slow network or a CPU-pressured machine causes the agent to read the DOM before client-side scripts have finished injecting prices. Less commonly, the brand mix in your input is heavy on Dior/Gucci, both of which have render delays even on good networks.<br>
<b>Resolution:</b> (a) Re-run the full file when the network is quieter. (b) Lower <code>MAX_CONCURRENT</code> from 3 to 2 to give each browser more CPU headroom. (c) Accept that ~20–30% manual review is the steady-state design target — the dual-agent pipeline was retired and the remaining single agent escalates conservatively by design.
</details>

<details>
<summary><b>6. Several rows fail with <code>❌ Row N failed with exception: ...</code></b></summary>
<br>
<b>Root Cause:</b> Per-row exceptions (Playwright timeouts, Anthropic transient 5xx, browser process crashes) are caught at the row boundary so one bad row doesn't abort the whole run. The row is logged and skipped — its <code>Remarks L1/L2</code> are left untouched in the output.<br>
<b>Resolution:</b> Grep the log for <code>❌ Row</code> to extract the affected Excel row indices. Either review those rows manually, or copy them into a small <code>To_be_update.xlsx</code> and re-run just that subset.
</details>

<details>
<summary><b>7. HTTP 429 or "rate_limit_error" appears in the log</b></summary>
<br>
<b>Root Cause:</b> Anthropic per-minute token budget exhausted. This is the single most common cause of mid-run failures when <code>MAX_CONCURRENT</code> is set too aggressively for a Tier 1 account.<br>
<b>Resolution:</b> Edit <code>run_demo.py</code> and lower <code>MAX_CONCURRENT</code> to 2 (or 1 in the worst case). Re-run. The crash-safe checkpointing means rows already completed before the rate-limit hit are preserved in the output file from the previous run — open it and confirm before restarting.
</details>

<details>
<summary><b>8. Output Excel exists but is locked / cannot be opened</b></summary>
<br>
<b>Root Cause:</b> The most common cause is that you have <code>To_be_update_filled.xlsx</code> open in Excel from a previous run, and the runner has flushed a checkpoint while Excel held a file lock. On Windows the runner will raise <code>PermissionError</code>; on macOS the write may silently fail.<br>
<b>Resolution:</b> Close the output file in Excel before running. As a hygiene practice, never open the output file while the run is still in progress — wait for the <code>✅ Output:</code> line, or at minimum close it again before the next checkpoint window.
</details>
