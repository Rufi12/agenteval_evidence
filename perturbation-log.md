# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** For policy `POL-2025-009` (`data/policies/POL-2025-009.txt`), I set the required field `deductible` to `null` in the recorded extraction response (all other fields unchanged) and replayed it through the real `extract_with_retry` using the project's `RecordedClient` (no API key). The starter test `test_ac_01_04_missing_source_halts_immediately` covers the same case.
- **Command I ran:** `.venv/bin/pytest tests/test_us01_retry.py -v` (13 passed, 1 skipped), then the experiment block with `.venv/bin/python -`. Output: `system1-policy-pipeline/perturbation-run.txt`.
- **What I predicted:** The validator flags `deductible` as `missing_source`, and the system escalates to a human after exactly 1 API call: no retry, no invented value.
- **What actually happened (paste the key output line):** `result type : RetryFutileEscalation` / `API calls   : 1 (up to 4 were allowed)` / `detected_pattern='deductible_absent'`
- **How this differs from the unperturbed run:** The unperturbed run of the same document returns `PolicyExtraction` with `deductible=500.0`, `retry_count=0` and 1 API call. The perturbed run returns an escalation instead of an extraction. For comparison, a retryable format error (negative premium) used all 4 calls before escalating (`retries_exhausted__negative_premium`), while the missing field stopped at 1.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** I took the extraction the system produced for `fixtures/documents/income_sum_mismatch.txt` (replay mode) and edited only `stated_monthly_total`, to 9642.17 (equal to the line items), 9642.67 (+$0.50) and 9644.17 (+$2.00). I re-ran the real `validate()` after each change. Replay mode only has recorded responses for the original documents, so I applied the edit to the extracted record instead of the text file.
- **Command I ran:** the experiment block with `.venv/bin/python -`. Output: `system2-mortgage-extraction/perturbation-run.txt`.
- **What I predicted:** 9642.17 and 9642.67 pass (within the $1 tolerance), and 9644.17 is flagged with a delta of -2.0.
- **What actually happened (paste the key output line):** `PERTURBED A ... {"consistent":true,"discrepancies":[]}` / `PERTURBED B ... {"consistent":true,"discrepancies":[]}` / `PERTURBED C ... {"consistent":false,"discrepancies":[{"field":"total_monthly_income","calculated":9642.17,"stated":9644.17,"delta":-2.0}]}`
- **How this differs from the unperturbed run:** The unperturbed run (total as printed, 10892.17) is flagged with `"delta":-1250.0` (`discrepancy-run.txt`). Setting the total to the line-item sum clears the flag. The validator ignores a $0.50 rounding gap but still catches a $2.00 error, so it checks the arithmetic, not whether the document "looks right".

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** A configuration change, with no file edited: I ran the investigation with `--simulate-timeout`, which makes the logistics source (`data/meridian/logistics.csv`) fail with a timeout while it is being read.
- **Command I ran:** `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout`. Output: `system3-supply-chain/timeout-run.txt`.
- **What I predicted:** The run still completes (exit code 0), logistics is reported as unavailable, and the metrics only logistics provides move to Incomplete instead of the run crashing.
- **What actually happened (paste the key output line):** `> Sources unavailable: logistics unavailable (timeout)` / `### late_shipment_count  _[missing source: timeout reading logistics]_` / `exit code: 0`
- **How this differs from the unperturbed run:** In the unperturbed run (`investigation-run.txt`), logistics supplies `late_shipment_count` (11.0 shipments) and the 78.0 percent on-time figure, so `on_time_delivery_rate` is **Contested** (95.0 vs 78.0). With logistics down, `late_shipment_count` moves to **Incomplete**, `average_lead_time_days` drops from "corroborated across 2 sources" to "single source only", and `on_time_delivery_rate` shows only the audit's 95.0 percent, with Contested reading `_none_`. The conflict disappears along with the source, which is why the "Sources unavailable" line at the top of the briefing matters.
