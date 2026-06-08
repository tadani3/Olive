# ONNX Model Health Dashboard — Project Plan

**Purpose:** Add a new *ONNX Health* page to the ORT Performance Dashboard that tracks the correctness (discrepancy) and speedup of ONNX models, fed by the `OnnxDiscrepancyCheck` capability in Olive (PR #2478, with the speedup metric added in PR #2502) and follow-up work.

**Author:** Tommaso Adani  **Date:** June 8, 2026  **Status:** Draft for review

---

## 1. Executive Summary

We will publish a new dashboard page that continuously tracks the **health of ONNX models** — how closely each ONNX model matches its reference PyTorch model, and how much faster it runs. It reuses the existing dashboard platform (ADO Pipeline → Cosmos DB → Azure Function App → Static Web App) and the established 9-step "add a new EP dashboard" process, so technical risk is low.

The primary lens is **accuracy/correctness**, with **speedup** as a secondary panel. Data is produced by Xavier's `OnnxDiscrepancyCheck` pass and refreshed automatically — the page polls every 60 seconds, no redeployment needed when new data arrives.

### Delivery hinges on one question: is speedup data in the dumped JSON?

The whole plan forks on whether the speedup/latency values are written to `discrepancy_check_results.json` (PR #2503 currently dumps **accuracy only** — speedup is still log-only; see §3). To avoid blocking on that, the plan is structured as two scenarios:

- **Scenario A — Accuracy-only (the safe core).** Everything that depends only on the accuracy block (MaxAE, element-count breaches, token fidelity, pass/fail). This ships regardless of speedup and has no upstream dependency beyond #2503 landing.
- **Scenario B — Accuracy + Speedup (the full vision).** Adds the speedup panel, columns, cards, and trends. **Conditional** on the speedup/latency values being added to the dumped `results` dict. If that lands, Scenario B is a thin additive layer; if it slips, we ship Scenario A and add speedup as a fast-follow with no rework.

Throughout this document, items marked **(speedup-dependent)** belong to Scenario B and are gated on that single upstream change.

---

## 2. Background

- **Source capability — foundation:** Olive PR #2478 — *"add a pass to measure the discrepancies on the test model"* (xadupre, merged). Introduces the `OnnxDiscrepancyCheck` pass: it compares exported ONNX model outputs against a reference HuggingFace/PyTorch model (max-abs error + element-count thresholds), handles dynamic input shapes, and auto-injects into the run config when `olive run --test` is used.
- **Source capability — generation comparison:** Olive PR #2487 — *"Add ONNX Runtime GenAI generation comparison in OnnxDiscrepancyCheck"* (xadupre, merged). Adds the **longest common token sequence** check: compares the reference transformers `generate()` output against ONNX Runtime GenAI generation, with `genai_model_path`, `generate_prompt`, `generate_max_new_tokens`, and `min_longest_common_tokens` config params.
- **Source capability — speedup:** Olive PR #2502 — *"Add OnnxDiscrepancyCheck speedup metric with default timing updates"* (xadupre). Adds the inference **speedup** measurement (ONNX vs PyTorch) plus `warmup_iterations` / `timing_iterations` config on top of the #2478 pass.
- **Source capability — results on disk (key enabler):** Olive PR #2503 — *"dumps OnnxDiscrepancyCheck result on disc so they can be picked up later and gathered for a dashboard"* (xadupre, **draft**). Builds a structured `results` object (`status`, `failures`, and all metrics) and writes it to `discrepancy_check_results.json` in the pass output directory (also saves the reference PyTorch model). Explicitly intended to feed a dashboard — this removes the need to scrape logs.
- **Target surface:** The existing **ORT CPU Performance Dashboard** (Azure Static Web App), which already hosts multiple pages (CPU Performance, EP Certification) using a shared platform.
- **Scope:** HuggingFace causal-LM (`ForCausalLM`) models — the model types the pass supports — aligned with the existing CPU perf model suite.

---

## 3. Metrics to Track

All metrics below are produced today by `OnnxDiscrepancyCheck`. The **Source** column notes what PR #2503's `discrepancy_check_results.json` actually contains — the accuracy block is dumped; the speedup block is **not yet** (still log-only).

### Accuracy (primary)
| Metric | Meaning | Source in #2503 JSON |
|---|---|---|
| **Max Absolute Error (MaxAE)** | Largest element-wise difference between PyTorch and ONNX logits | ✅ `max_abs_error` |
| **% elements > 0.1** | Share of output elements where abs difference exceeds 0.1 | ⚠️ derived — JSON has raw `elements_above_0_1` + `total_elements`; dashboard computes % |
| **% elements > 0.01** | Share of output elements where abs difference exceeds 0.01 | ⚠️ derived — JSON has raw `elements_above_0_01` + `total_elements` |
| **Longest Common Token Sequence** | Generation fidelity: transformers `generate()` vs ONNX Runtime GenAI `generate()` *(PR #2487)* | ⚠️ conditional — only written when `genai_model_path` is configured |
| **Pass / Fail status** | Versus configured thresholds (`max_mae`, element-count limits, min token match) | ✅ `status` + `failures` (with reasons) |

### Speedup (secondary — PR #2502)
| Metric | Meaning | Source in #2503 JSON |
|---|---|---|
| **Speedup** | PyTorch avg latency ÷ ONNX avg latency on the target device | ❌ **not in JSON** — still `logger.info` only |
| **PyTorch avg latency** | Average seconds per inference iteration (PyTorch) | ❌ **not in JSON** |
| **ONNX avg latency** | Average seconds per inference iteration (ONNX) | ❌ **not in JSON** |

Defaults: `warmup_iterations = 3`, `timing_iterations = 5` (set to `0` to skip speedup).

> **This is the fork in the plan.** The accuracy block above is dumped today (Scenario A — ships regardless). The speedup block is **not** in the JSON — PR #2503's `results` dict assembles only the accuracy fields, so speedup/latency are still `logger.info`-only. To get **Scenario B** (the speedup panel), the timing values must be added to the dumped `results` dict — a small follow-up for Xavier (whoever lands last between #2502 and #2503). Until then every speedup-tagged element in this plan stays dark, and the dashboard renders the accuracy core unchanged.

---

## 4. Dashboard Design

The page mirrors the existing CPU perf layout (tabs, status cards, CSV export, configurable regression threshold). Elements tagged **(speedup-dependent)** only render if the speedup/latency values are present in the report; the page degrades gracefully to the accuracy-only core (Scenario A) when they are absent.

**Header status cards:** Total Models · Passing · Failing (accuracy regressions) · Token-Divergent · *Avg Speedup* **(speedup-dependent)**

**Tabs:**
- **Overview** — health roll-up: pass/fail donut, count of models breaching each threshold, top MaxAE offenders; *biggest speedup wins/regressions* **(speedup-dependent)**.
- **Models** — core table, one row per model: MaxAE, %>0.1, %>0.01, longest-common-tokens, status badge; plus *PyTorch ms, ONNX ms, speedup* **(speedup-dependent)**. Sortable, color-coded, CSV export.
- **Architectures** — same metrics grouped by model family.
- **History** — time-series trends per model/architecture: MaxAE over runs; *speedup over runs* **(speedup-dependent)** for regression-over-time detection.

**Status colors:** Green = within thresholds · Red = accuracy threshold breach · *Amber = speedup regression beyond configured %* **(speedup-dependent)** · *Gray = speedup skipped/unavailable* **(speedup-dependent)**.

> In Scenario A the layout is identical minus the speedup-tagged cards/columns/series — no structural change, so moving from A to B is purely additive (new columns + cards light up once the data is present).

---

## 5. Architecture

```
ADO Pipeline  →  Cosmos DB container  →  Azure Function App (HTTP API)  →  Static Web App page
 (runs the         (onnx-health-           (reads container,                 (fetches API,
  discrepancy        reports)                serves JSON)                      renders, polls 60s)
  check, upserts
  JSON reports)
```

All resources live in the existing **AI Infra Build** subscription, reusing the `modelregressionpipeline` Cosmos account and `epcert` database.

---

## 6. Implementation Plan

**Phase 1 — Azure plumbing**
1. Create Cosmos container `onnx-health-reports` (partition key `/id`).
2. Create Function App `onnx-health-dashboard-api` (Node 22, Linux, Consumption, Canada Central). **Enable public network access** (avoids 403s).
3. Configure app settings (read-only Cosmos key) and CORS for the Static Web App origin.

**Phase 2 — Function App code**
4. Scaffold `azure-function/` and copy the reports handler from the CPU perf function app, changing only the container env-var name. Deploy with `func azure functionapp publish`; verify the API returns `[]`.

**Phase 3 — Pipeline integration**
5. Run `OnnxDiscrepancyCheck` across the LLM suite. Thanks to PR #2503, each run writes a `discrepancy_check_results.json` to the pass output directory; add a step that reads those JSON files and upserts each model's report to Cosmos (read-write key). Pattern from `dashboard-publish-template.yaml` (PR #39127). No log-scraping required.
   - **Speedup fork:** if the JSON contains the speedup/latency fields (Scenario B), upsert them as-is. If not (Scenario A), upsert the accuracy fields only — the pipeline step is written to pass through whatever keys exist, so no change is needed when speedup data later appears.

**Phase 4 — Dashboard page**
6. Add `docs/onnx-health.html` (modeled on `cpu-perf.html`, PR #39124), update `staticwebapp.config.json`, and add nav links across existing pages. Build the table/cards to render speedup columns **only when present** so the same page serves both scenarios.
7. Validate in a preview environment, then promote to production via the nightly pipeline.

> **Sequencing:** Phases 1–4 are built entirely against the accuracy core (Scenario A) and do not wait on the speedup change. Scenario B requires no extra phase — the speedup-dependent cells activate automatically once the data lands.

---

## 7. Key Risk / Open Item

**Largely resolved by PR #2503.** The original concern was that `OnnxDiscrepancyCheck` only *logged* its metrics, with no structured output for the pipeline to upsert. PR #2503 now dumps a structured `discrepancy_check_results.json` (status, failures, and all metrics) to disk specifically so a dashboard can pick it up.

**Remaining items to watch:**
- **PR #2503 is still a draft** (not merged) and currently uses `print` (failing RUFF/T201 lint), so the exact output shape may change before it lands. Confirm the final JSON schema and file location once merged.
- **Speedup metrics are not in the dump (the one real metric gap).** PR #2503's `results` dict assembles only the accuracy fields (`max_abs_error`, element counts, `status`/`failures`, and conditionally `longest_common_token_sequence`). The speedup/latency values from PR #2502 are still `logger.info`-only. **If we want the speedup columns, the timing values need to be added to the dumped `results` dict** — a small follow-up for Xavier.
- **Token-sequence metric is conditional.** `longest_common_token_sequence` is only written when `genai_model_path` is set; the CLI auto-injection path does not set it by default, so the pipeline must configure the GenAI model path for that column to populate.
- **Percentages are raw counts.** The JSON emits counts + `total_elements`, not the `%>0.1` / `%>0.01` we display — the dashboard/pipeline divides (trivial).
- We must align our **report JSON schema** with the fields #2503 emits (add run/model/build metadata in the pipeline step if the pass doesn't include it).

**Open question — report schema / identifiers (what we slice and trend by):**
The pass emits the *measurements*; the dashboard also needs descriptive "key" fields to group, filter, and chart data. Some dimensions are clear; others need a decision.
- **Assumed (slice by):** model name, architecture family (e.g. `LlamaForCausalLM`), and precision variant (fp32/fp16/int4).
- **Confirmed:** environment and execution provider (EP) — included.
- **Undecided — run / build identifiers:** do we tag each report with a timestamp + pipeline run id, and an ORT/Olive version or commit/branch? These are what enable the **History tab** (trend over time) and let us correlate a regression to a specific build. Open question: do we need per-run history and per-build correlation now, or is latest-value-per-model sufficient for v1?
- **Ownership:** does #2503's JSON already carry any of these fields, or does the pipeline step attach them before upserting to Cosmos?

---

## 8. Complexity & Effort

| Area | Complexity | Notes |
|---|---|---|
| Azure plumbing | Low | Documented 9-step process; copy-paste |
| Function App code | Low | Reuse existing handler, change one variable |
| Dashboard HTML | Low–Med | Clone existing page; new metrics/columns; render speedup cells conditionally |
| Pipeline integration | Low–Med | Read `discrepancy_check_results.json` (PR #2503) and upsert; two Cosmos keys, variable mapping |
| Structured metric output | Low | Accuracy block delivered by PR #2503 — pending merge out of draft |
| **Speedup in JSON (Scenario B)** | **Low — upstream** | **Small Olive change to add timing fields to the dumped `results` dict; gates only the speedup panel, not the page** |

**Overall: Low–Moderate.** Low technical risk, spans three repos (Olive, pipeline, dashboard). The accuracy core (Scenario A) depends only on #2503 merging. The speedup panel (Scenario B) adds one small upstream change but no extra dashboard/pipeline rework — it is additive and can land after v1.

---

## 9. Watch-Outs (from the documented process)

1. **Public network access** — new Function Apps may default to disabled, causing 403 on every browser request.
2. **CORS for preview envs** — preview SWA URLs differ from production; add the origin (or `*`) before testing.
3. **Two Cosmos keys** — pipeline uses a read-write key; Function App uses a read-only key. Don't mix them up.
4. **Pipeline variable naming** — ADO variable names must match exactly what the YAML references.
5. **Deploy with `func` CLI** — use `func azure functionapp publish`, not zip deploy or the VS Code extension.

---

## 10. Decisions Needed

1. **Confirm scope:** LLM/causal-LM models, accuracy-first with speedup secondary.
2. **Track PR #2503** (results-to-disk) to merge out of draft, and confirm the final JSON schema / output path it produces.
3. **Speedup in scope?** If yes, request that the speedup/latency values be added to #2503's dumped `results` dict (they are currently log-only). If speedup can wait, v1 ships accuracy-only and we add the column later.
4. **Decide the report identifiers:** confirm slicing by model name / architecture family / precision variant + environment + EP, and decide whether to include **run** (timestamp, pipeline run id) and **build** (ORT/Olive version or commit) identifiers — i.e. do we need per-run history and per-build correlation in v1?
5. **Confirm reuse** of the existing Cosmos account / database vs. new resources.
6. **Page name:** propose `onnx-health.html`.

---

## 11. Next Steps

- Track PR #2503 to merge and lock the `discrepancy_check_results.json` schema / output location it produces — this unblocks the accuracy core (Scenario A).
- **Decide speedup (the fork):** confirm whether the speedup/latency values will be added to the dumped `results` dict. If yes, Scenario B ships with v1; if not, v1 ships Scenario A and speedup is a fast-follow.
- Finalize report JSON schema (pass output + pipeline-added run/model/build metadata) and dashboard column set, building the speedup cells to render conditionally.
- Stand up Cosmos container + Function App in a preview environment.
- Build the page against preview data (accuracy core first), then promote to production; light up the speedup panel if/when the data lands.
