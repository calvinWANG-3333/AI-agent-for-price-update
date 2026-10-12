<sub>[README](../README.md) &nbsp;·&nbsp; [01 SETUP](01_setup.md) &nbsp;·&nbsp; [02 OPERATIONS](02_operations.md) &nbsp;·&nbsp; **03 SPECIFICATION**</sub>

# 03 SPECIFICATION

The exact rules encoded in each component. If the code and this page disagree, the code wins and this page should be fixed.

## PIPELINE FLOW & RECONCILIATION LOGIC

For colleagues who will operate or extend the runner, the high-level data flow is:

```
   To_be_update.xlsx
          │
          ▼
   ┌─────────────────────────────────────────┐
   │  run_demo.py  (orchestrator)            │
   │  - eligibility filter                   │
   │  - asyncio.Semaphore (MAX_CONCURRENT)   │
   │  - checkpointing every 10 rows          │
   └─────────────────────────────────────────┘
          │
          ▼  for each eligible row, in parallel
   ┌─────────────────────────────────────────┐
   │  agent_1.py  (Claude Sonnet 4.5)        │
   │  - browser-use + Chromium, DOM mode     │
   │  - structured Pydantic output           │
   │    {price_found, url_redirected,        │
   │     confidence ∈ {high, medium, low},   │
   │     evidence}                           │
   └─────────────────────────────────────────┘
          │
          ▼
   ┌─────────────────────────────────────────┐
   │  reconcile.py  (pure rule engine)       │
   │  Priority cascade (first match wins):   │
   │    P1. redirected + null price          │
   │        → AUTO_PRODUCT_GONE              │
   │    P2. evidence starts with "BLOCKED:"  │
   │        → LOW_CONFIDENCE_NEEDS_REVIEW    │
   │    P3. confidence=high AND price set    │
   │        → AUTO_REFRESH_* / AUTO_MISSING_*│
   │    P4. anything else                    │
   │        → LOW_CONFIDENCE_NEEDS_REVIEW    │
   └─────────────────────────────────────────┘
          │
          ▼
   ┌─────────────────────────────────────────┐
   │  excel_writer.py                        │
   │  - copies input → output                │
   │  - patches Remarks L1 / L2 only         │
   │  - color-codes by decision label        │
   └─────────────────────────────────────────┘
          │
          ▼
   To_be_update_filled.xlsx
```

**Why the agent escalates so aggressively.** The previous dual-agent architecture (Claude + GPT-4o cross-verification) was retired to halve LLM cost. With only one agent left, the `confidence` field is the **sole gate** between auto-accept and human review. The rubric is intentionally strict: anything that isn't a textbook "one product, one price, page fully rendered" answer drops to `medium`, which means manual review. Expect a real-world AUTO rate around 70–80% on a clean input, not 100%.

## COMPONENT SPECIFICATIONS

This section documents the **exact behavioral rules** encoded in each of the three core components. It is intended for colleagues who need to understand *why* the agent made a particular call on a particular row, or who will maintain or extend the pipeline. The rules below are extracted directly from the source code — if the code and this document ever disagree, the code is authoritative and this section should be updated.

### 1. Agent 1 — Verification Agent (`agent_1.py`)

#### 1.1 Purpose
A single Claude-driven browser agent that opens a product URL, detects whether the product is still on sale, reads the current price from the DOM, and returns a structured JSON result with a self-assessed confidence score. **This is the only LLM call per row** — the previous dual-agent (Claude + GPT-4o) cross-verification architecture has been retired to halve LLM cost.

#### 1.2 Structured Output Schema (`PriceVerificationResult`)
Every agent invocation must produce a JSON object matching this Pydantic schema. Fields the agent fails to populate are coerced to `null` (or `"low"` for confidence) by the fallback handler.

| Field | Type | Meaning |
| --- | --- | --- |
| `product_available` | `"yes"` / `"no"` / `"uncertain"` | Is the product still on sale at the original URL? `"yes"` requires both a recognizable product name AND a visible price in the DOM. |
| `url_redirected` | `bool` | Did the browser end up on a different page than the one we opened? See §1.3 for the strict definition. |
| `final_url` | `str` or `null` | The URL after the page finished loading. |
| `price_found` | `float` or `null` | Current price as a plain number (`€2,000.00` → `2000.0`). **Must be `null` if the agent is not certain.** |
| `currency_found` | 3-letter ISO code or `null` | EUR, USD, GBP, JPY, CNY, etc. Must be `null` if `price_found` is `null`. |
| `evidence` | `str` | What the agent actually observed. When the page was blocked, this field **must start with the literal prefix `"BLOCKED:"`** — `reconcile.py` keys off this exact prefix in §2.2. |
| `confidence` | `"high"` / `"medium"` / `"low"` | Self-assessed certainty. The reconciliation engine relies entirely on this field to decide auto-accept vs manual escalation (§2.2). |

#### 1.3 URL Redirect — Strict Definition
> [!IMPORTANT]
> **Not all URL changes are redirects.** Many luxury sites rewrite URLs to SEO-friendly slugs *while remaining on the same product page* (e.g., `/products/S5573CRIW_M928` → `/products/sac-dior-book-tote-mini-S5573CRIW_M928`). This is **NOT** a redirect.
>
> `url_redirected = true` is set **only** when the final URL is a homepage, a category listing, or a search results page — i.e., the original product has been removed from sale and the site now points elsewhere.

#### 1.4 Confidence Rubric (Verbatim from Prompt)

| Level | Required Conditions |
| --- | --- |
| **`high`** | **ALL FOUR** must hold: (1) exactly one main product price clearly visible in DOM; (2) product name on page closely matches the requested name; (3) page is a proper, fully-rendered product page (not blocked, not partially loaded); (4) no ambiguity from sale/regular price or installment plans being mistaken for the main price. |
| **`medium`** | A price was found, but **at least one** of: multiple prices visible (sale vs regular, multiple variants), product name only partially matches, page seems to have rendered only partially, or the agent had to use reasoning to pick among candidate prices. |
| **`low`** | **Any** of: price not visible, page blocked, redirect detected, product name does not match, page failed to load, or the agent is guessing. |

> [!CAUTION]
> **Tie-breaker rule:** When the agent is between two confidence levels, the prompt instructs it to **always choose the lower one**. False `high` causes silent data corruption (the wrong price is written to Excel without human review); false `medium` only costs a few minutes of manual review. The asymmetry of cost is the entire justification for the strict rubric.

#### 1.5 Hard-Coded Behavioral Rules (Encoded in the Prompt)

##### a) Early-Termination Triggers
The agent is instructed to **stop immediately, without retrying or navigating elsewhere**, if it observes any of the following, and return a `BLOCKED:` evidence string with `confidence = "low"`:

- `"Access Denied"` / `"Accès refusé"` / `"Zugriff verweigert"` (multilingual variants)
- HTTP 403 Forbidden
- Cloudflare challenge pages (`"Checking your browser"`, `"Just a moment..."`)
- CAPTCHA, hCaptcha, reCAPTCHA
- HTTP 429 / `"Too many requests"`
- PerimeterX / DataDome / Akamai block screens
- Network errors: `ERR_TUNNEL_CONNECTION_FAILED`, `ERR_CONNECTION_REFUSED`, DNS errors
- A page that is entirely a blocking message with no product information

##### b) Pre-Content Interstitial Handling
Luxury sites frequently show one of these before the product page renders — the agent has a **strict 2-step budget** to dismiss them, after which it proceeds regardless:

| Interstitial | Action |
| --- | --- |
| Cookie consent banner | Click `Accept` / `Accept all` / `OK` |
| Region / country selector | Pick the country matching the URL market (`fr_fr` → France, `en_us` → US, `gb` → UK) |
| Newsletter signup popup | Click the `X` close button or `No thanks` |
| Age gate or terms confirmation | Confirm to proceed |

##### c) Anti-Hallucination Rules
Five non-negotiable rules baked into the prompt:

1. **Never fabricate prices.** If the price is not clearly visible in the DOM, `price_found` MUST be `null`.
2. **The reference price is context, not the answer.** The expected price is passed to the agent for grounding, but it must read the actual price from the page, not echo the reference.
3. **Redirects mean "product is gone", not "try harder".** If a redirect is detected, do not browse, do not search, do not navigate. Report and stop.
4. **Evidence must be observation, not expectation.** Quote DOM content where possible.
5. **Be conservative with confidence.** Because no second agent cross-checks the result, a false `high` will be written to Excel without review. When in doubt, downgrade.

#### 1.6 Runtime Configuration

| Parameter | Value | Rationale |
| --- | --- | --- |
| Model | `claude-sonnet-4-5` | Current production model. |
| `use_vision` | `False` | DOM mode (read raw HTML structure). 3–4× cheaper and 2× faster than vision mode; tradeoff documented in [operations, known issue 1](02_operations.md). |
| `minimum_wait_page_load_time` | `3.0` seconds | Forces a 3-second floor on page-load waits to mitigate JS-injected price delays. |
| `max_steps` | `8` | The agent gets at most 8 reasoning steps; beyond that the run is terminated and the fallback (§1.7) kicks in. |
| `initial_actions` | Force `navigate` | Defensive — we explicitly enqueue the navigation as the first action instead of trusting the LLM to navigate on its own. |

#### 1.7 Failure Fallback
If the agent exhausts `max_steps = 8` without producing a parseable `final_result`, the code synthesizes a safe placeholder:

```python
{
    "product_available": "uncertain",
    "url_redirected": False,
    "final_url": None,
    "price_found": None,
    "currency_found": None,
    "evidence": "Agent failed to produce a final result.",
    "confidence": "low",
}
```

This placeholder is routed by `reconcile.py` to the P4 manual-review bucket (§2.2), so the row is **never silently dropped** — it is always either written or flagged for human review.

### 2. Reconciliation Engine (`reconcile.py`)

#### 2.1 Purpose
A pure rule engine (no LLM, fully deterministic) that consumes the agent's structured output and decides:
- Which two values to write into the `Remarks L1` and `Remarks L2 (details)` columns
- Which color the cells should be filled with (via the `decision` label, consumed by `excel_writer.py` in §3)

#### 2.2 Tunable Constants

| Constant | Value | Meaning |
| --- | --- | --- |
| `PRICE_TOLERANCE_PCT` | `1.0` | A `price_found` is considered to **match** `Original Price_validated` if their absolute difference is ≤ **1%** of the validated price. For a €2,000 product, this is a €20 window. |
| `MANUAL_TAG` | `"manual"` | The literal string written to `Remarks L2 (details)` whenever a row is escalated for human review. |

#### 2.3 Decision Priority Cascade
The engine evaluates four priority levels in strict top-down order. **The first match wins**; subsequent rules are not evaluated.

| Priority | Trigger Condition | Outcome Family |
| --- | --- | --- |
| **P1** | `url_redirected == true` AND `price_found is None` | Product gone — auto-mark removed/confirmed-missing |
| **P2** | `evidence` (stripped, uppercased) starts with `"BLOCKED"` | Page was blocked — escalate to manual |
| **P3** | `confidence == "high"` AND `price_found is not None` | Trust the agent — auto-write the price |
| **P4** | Everything else (medium/low confidence, or high with `null` price, etc.) | Escalate to manual |

#### 2.4 Complete Decision Matrix
The combination of P1–P4 and the two possible `row_type` values yields exactly **eight** output patterns:

| # | Trigger | `row_type` | Remarks L1 written | Remarks L2 written | `decision` label |
| --- | --- | --- | --- | --- | --- |
| 1 | redirect + `null` price | `refresh_data` | `Missing-remove` | *(blank)* | `AUTO_PRODUCT_GONE` |
| 2 | redirect + `null` price | `missing_in_crawl` | `OK, missing` | *(blank)* | `AUTO_CONFIRMED_MISSING` |
| 3 | `BLOCKED:` in evidence | any | **original L1 preserved** | `manual` | `LOW_CONFIDENCE_BLOCKED` |
| 4 | high + price matches validated | `refresh_data` | `Refresh data` | `Same price` | `AUTO_REFRESH_SAME_PRICE` |
| 5 | high + price matches validated | `missing_in_crawl` | `Not OK, available` | `Same price` | `AUTO_MISSING_BUT_AVAILABLE_SAME_PRICE` |
| 6 | high + price differs from validated | `refresh_data` | `Refresh data` | *(new price as number)* | `AUTO_REFRESH_NEW_PRICE` |
| 7 | high + price differs from validated | `missing_in_crawl` | `Not OK, available` | *(new price as number)* | `AUTO_MISSING_BUT_AVAILABLE_NEW_PRICE` |
| 8 | anything else | any | **original L1 preserved** | `manual` | `LOW_CONFIDENCE_NEEDS_REVIEW` |

#### 2.5 Business Semantics: `refresh_data` vs `missing_in_crawl`
The two row types represent different business situations and trigger different L1 labels:

**`refresh_data` rows** —— These are existing records that the team has flagged for a price re-check.

- Agent says product is gone → `Missing-remove` (downstream should delete this SKU from inventory)
- Agent confirms price unchanged → `Refresh data` + `Same price` (no action needed)
- Agent finds a new price → `Refresh data` + the new price number (price update applied)

**`missing_in_crawl` rows** —— These are SKUs the daily crawler failed to retrieve. They might be genuinely off-sale, OR the crawler might have missed them.

- Agent confirms product is gone → `OK, missing` (the crawler was correct; product is indeed off-sale)
- Agent finds the product still on sale → `Not OK, available` (the crawler missed it; data needs to be restored manually)

**Why "preserve original L1" matters (rows 3 and 8).** When the agent cannot produce a confident answer, the engine **does not overwrite** whatever `Remarks L1` value already existed in the input file. It only stamps `manual` into `Remarks L2`. This prevents the agent from destroying human-curated annotations during ambiguous cases.

### 3. Excel Writer (`excel_writer.py`)

#### 3.1 Purpose
Apply the reconciliation engine's decisions to the input Excel — without ever modifying the original file — and color-code the affected cells for fast visual triage.

#### 3.2 Workflow
1. **Copy.** `shutil.copyfile(input_path, output_path)` — the original input file is never opened in write mode. The output is always a fresh copy.
2. **Open the copy.** `openpyxl.load_workbook(output_path)`.
3. **Locate columns by header name.** Walk row 1 looking for the literal strings `"Remarks L1"` and `"Remarks L2 (details)"`. **Column positions are not hard-coded** — if the schema reorders columns, the writer still works as long as the header names are intact. If either header is missing, the writer raises `ValueError` and aborts.
4. **Apply each update.** For each `(row_index, remark_l1, remark_l2, decision)` tuple, write the two cell values and apply a fill color derived from the `decision` label (see §3.3).
5. **Save.** `wb.save(output_path)`.

#### 3.3 Decision-to-Color Mapping
The mapping is defined by `_decision_to_color()` as a **priority-ordered `if` chain**. The order matters — `AUTO_PRODUCT_GONE` and `AUTO_CONFIRMED_MISSING` are checked **before** the generic `AUTO_` prefix, otherwise they would incorrectly resolve to green.

| Match Order | Decision Prefix | Fill Color | Hex |
| --- | --- | --- | --- |
| 1st | `AUTO_PRODUCT_GONE` OR `AUTO_CONFIRMED_MISSING` | Light red | `#F4CCCC` |
| 2nd | Any other `AUTO_*` | Light green | `#C6EFCE` |
| 3rd | Any `LOW_CONFIDENCE*` | Light yellow | `#FFEB9C` |
| Default | (no match) | *(no fill)* | — |

#### 3.4 What is Preserved vs Modified

| Aspect of the file | Behavior |
| --- | --- |
| Original input file | **Never touched.** All work is done on the copy. |
| Columns other than `Remarks L1` / `Remarks L2 (details)` | Copied verbatim from the input — values, formatting, column widths, autofilters, frozen panes, pre-existing fill colors. |
| Row order | Preserved — the writer locates rows by 1-based `row_index` passed in from `run_demo.py`, never by re-sorting. |
| Cells in the two Remarks columns for **eligible** rows | Overwritten with the engine's decision; filled with the color from §3.3. |
| Cells in the two Remarks columns for **ineligible** rows | Untouched — they keep whatever the human reviewer wrote (or stay blank). |

---
