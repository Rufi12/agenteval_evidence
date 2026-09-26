# Reflection Brief — Evaluation and Observability Capstone

**Name:** Anlet Rufitta Arul Das
**Date:** 2026-09-26

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Debian GNU/Linux 12 (bookworm), kernel 6.6.97+ (x86_64), Udacity course workspace (see `environment.txt`) |
| Python version | Python 3.13.0 (see `environment.txt`) |
| Date run | 2026-09-26 |
| Ran any system live? (which) | No. The live policy pipeline's model was not enabled on the course API account, so the Instructions' offline fallback (`test_us04_routing.py`) was used. All other runs are offline by design. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped (live-API tests), from `system1-policy-pipeline/tests.txt` |
| Routing output file | `system1-policy-pipeline/routing_decisions.json` (from the offline fallback run `routing-test-run.txt`, 9 passed) |
| auto_approve / human_review / spot_check counts | 1 / 1 / 0 |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> Escalation record (`system1-policy-pipeline/perturbation-run.txt`): `RetryFutileEscalation(policy_id='POL-2025-009', field='deductible', category='missing_source', detected_pattern='deductible_absent', reason="Field 'deductible' returned null — the source document does not contain this information. Retry is futile; escalate to human review.")` with `API calls   : 1 (up to 4 were allowed)`.
> The system made **1** API call. Retrying cannot create information that is not in the document. It either returns the same null again or pushes the model to invent a value, and it wastes calls and delays the human who has to resolve it. My comparison case shows the cost: a negative premium was retried and used `API calls   : 4` before escalating (`retries_exhausted__negative_premium`).

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> Record `POL-B` (home) in `routing_decisions.json`: `"decision": "human_review"`, `"reason": "fields_below_threshold=['premium_amount']"`, with `"premium_amount": 0.5` in its `confidence_summary`. The signal was **confidence** (0.5 is below the 0.90 threshold). `reviewer_disagreements` and `integration_failures` are both empty.
> Confidence caught this record, but it is not enough on its own. In `routing-test-run.txt`, `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review PASSED`: every field was at 0.99 confidence, but the reviewer disagreed on `premium_amount`, so the record went to `human_review`. Trusting the model's confidence alone would have auto-approved that record.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> From `system1-policy-pipeline/calibration-report.txt`: `umbrella  exclusions      n=2 conf=0.93 acc=0.00 brier=0.865` and `OVERALL brier=0.291`.
> The overall figure looks moderate, but umbrella/exclusions is wrong every time while claiming 93% confidence. The auto and home cells are at `acc=1.00`, and they average the bad cell away. Slicing by `policy_type × field` shows exactly which document type and field is miscalibrated, so that cell can be routed to humans instead of trusted.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed, from `system2-mortgage-extraction/tests.txt` |
| Document run | `appraisal_informal_sqft.txt` and `income_missing_bonus.txt` (`extract-run.txt`), `income_sum_mismatch.txt` (`discrepancy-run.txt`) |
| Classified type | `type=appraisal` (appraisal_informal_sqft), `type=income_verification` (both income documents) |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> From `system2-mortgage-extraction/discrepancy-run.txt`: `"consistent": false` with `"field": "total_monthly_income", "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0`.
> Tool use with the JSON schema guarantees the **shape**: every field has the right name and type (a number or null). The validator guarantees the **math**: the line items must add up to the stated total. 10892.17 is a perfectly valid number, so the schema accepts it. Only the arithmetic check shows it is $1,250 more than the line items (5416.67 + 1250.0 + 2140.0 + 385.5 + 450.0 = 9642.17).
> What the schema cannot catch: well-typed numbers that don't add up (this case). What the validator cannot catch: a wrong value that has no total to check it against, for example a misread `appraised_value` or `gross_living_area_sqft`, because the validator only checks `total_monthly_income`.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> From `system2-mortgage-extraction/extract-run.txt` (income_missing_bonus.txt, `type=income_verification`): `"bonus_monthly": null` (and `"bonus_ytd": null`), with `"consistent": true`.
> The pay stub has no bonus line, so there is nothing to extract. An invented number would look exactly like a real one and would flow into the income calculation and the loan decision. `null` tells the reader "not in the document". The schema allows this: `"bonus_monthly": {"type": ["number", "null"]}` in `mortgage_extractor/schema.py` is a nullable union type, and the prompt rule says "Return null for any field not explicitly stated in the document. Do not infer, default, or fabricate." Without the `null` option, the forced tool call would have to put a number there.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> Source line in `fixtures/documents/appraisal_informal_sqft.txt`: `Gross Living Area:   approximately 2,400 sq ft (above-grade finished)`. Extracted value in `system2-mortgage-extraction/extract-run.txt` (`type=appraisal`): `"gross_living_area_sqft": 2400`.
> Normalizing once, at extraction time, gives every downstream user a clean integer that can be compared, summed and validated, and the schema (`{"type": ["integer", "null"]}`) can enforce the type. If the raw text went downstream, every consumer would need its own parser, and those parsers drift and break. The extractor is also the step that sees the full document, so it can interpret informal wording correctly.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed, from `system3-supply-chain/tests.txt` |
| Briefing file | `system3-supply-chain/briefing.md` (from the run in `investigation-run.txt`) |
| Section the conflict landed in | Contested (`on_time_delivery_rate`, marked ⚠️ ESCALATE) |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> From `system3-supply-chain/briefing.md`, section **Contested**: `on_time_delivery_rate` has `95.0 percent — supplier_audit (as of 2026-04-10)` and `78.0 percent — logistics (as of 2026-04-05)`, with `escalation: high-impact metric is contested across sources`.
> The 95% is the supplier's own audit claim, and the 78% comes from the shipment log (the same source reports `late_shipment_count` of 11.0). An averaged number like 86% would be a figure that no source actually reported, and it would hide the fact that the supplier's self-reported number is 17 points higher than the measured one. With both values kept, a buyer can see the gap, see who said what and when, and act on it, for example by asking the supplier for evidence before trusting the 95%.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> From `system3-supply-chain/timeout-run.txt`: `> Sources unavailable: logistics unavailable (timeout)` and, under Incomplete, `### late_shipment_count  _[missing source: timeout reading logistics]_`, ending with `exit code: 0`.
> "Unreachable" and "nothing to report" are labelled differently. `late_shipment_count` says `timeout reading logistics` (the source failed, so the data may exist but could not be read), while `production_capacity_utilization` says `no source reported this metric` (every source was read and none had it). The run still finishes because the coordinator uses local recovery: the failed reader returns a result with its failure recorded, the coordinator carries on with the other sources (audit, quality, news), and the gap is annotated in the briefing instead of aborting. Also, with logistics gone, `on_time_delivery_rate` shows only `95.0 percent — supplier_audit` and Contested reads `_none_`, so the "Sources unavailable" line is what warns the reader that a conflict may be hidden.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> From `system3-supply-chain/briefing.md`: `defect_rate_ppm` has `180.0 ppm — supplier_audit (as of 2026-04-10)` and `190.0 ppm — internal_quality (as of 2026-04-08)`. The news claim `A 72-hour dockworker strike at the Port of Long Beach beginning March 16, 2026 ... (industry_news, 2026-03-17)` is also dated.
> Because every claim carries its date, the reader can see these are separate measurements taken at different times by different sources, so a small difference reads as normal change between reports rather than one source being wrong. The dated port strike (March 2026) also explains late deliveries in part of the quarter without contradicting the quarter-level lead-time figures. Without dates, older and newer facts would look like two sources disagreeing about the same moment.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> System 2, `system2-mortgage-extraction/discrepancy-run.txt`: the extraction was valid JSON and matched the document, but the validator reported `"calculated": 9642.17, "stated": 10892.17, "delta": -1250.0`. A design that trusted the model's structured output would have passed a monthly income that is $1,250 too high straight into a mortgage decision. Checking the arithmetic on the output, instead of trusting that well-formed means correct, is what caught it.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> System 1 is where this mattered most. In `system1-policy-pipeline/calibration-report.txt`, the model was 93% confident on umbrella exclusions and wrong every time (`umbrella  exclusions      n=2 conf=0.93 acc=0.00`), while `OVERALL brier=0.291` looked moderate. In `routing-test-run.txt`, `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review PASSED` shows a record at 0.99 confidence still going to a human because the independent reviewer disagreed. If routing trusted confidence alone, both of these would have been auto-approved.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> Workflow: pulling fields (vendor, invoice number, line items, tax, total) from scanned supplier invoices into an accounts-payable system. I would reach for **validated retry with escalation** first: a schema with nullable fields so missing values come back as null, an arithmetic check (line items + tax = total), a retry for fixable format errors, and immediate escalation to a person when a required field is missing from the invoice. To know when it breaks, I would instrument the escalation rate and retry count per vendor, the validator discrepancy rate, and accuracy on a spot-checked sample sliced by vendor × field, alerting when any slice's accuracy falls below its average confidence (the same kind of hidden cell as umbrella/exclusions in System 1).
