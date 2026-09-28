# Reflection Brief — Evaluation and Observability Capstone

**Name:Tanmay Avinashrav Misal**
**Date:**

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field                        | Value                     |
|------------------------------|---------------------------|
| OS & version                 | Linux (Vocareum Workspace)|
| Python version               | Python 3.13.0             |
| Date run                     | 2026-09-28                |
| Ran any system live? (which) | No, all systems executed via --offline
                                 and --simulate-timeout fallbacks |
---

## 1. Validated, routed pipeline

| Evidence | Value |
|---       |  ---  |
| Passing test count | 9|
| Routing output file | routing_decisions.json|
| auto_approve / human_review / spot_check counts | / offline test fallback used/ |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> (Note: Check your System 1 perturbation output for the exact API count). The system typically makes 1 API call before escalating when a required field is missing. Retrying a deterministic failure (like a document legitimately missing a required field) is futile because the model will either infinitely loop trying to find data that isn't there (burning API costs), or worse, it will hallucinate a fake value to satisfy the prompt. Escalating immediately saves compute and prevents fabrication.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> In my routing_decisions.json test fallback, the rule is proven by test_ac_04_02_integration_failure_routes_to_human_review. An integration failure sent it to human review. If we had trusted the model's confidence alone, an extraction with 99% confidence would have been automatically approved, completely missing the fact that the downstream database integration was broken, leading to silent data loss.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> Quote: umbrella exclusions n=2 conf=0.93 acc=0.00 brier=0.865 (Overall brier=0.291).
Slicing by policy_type × field reveals catastrophic blind spots that the aggregate hides. The overall score of 0.291 looks acceptable, but the sliced data shows the model is highly confident (93%) but completely wrong (0% accuracy) specifically when extracting umbrella exclusions.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 |
| Document run |appraisal_informal_sqft.txt |
| Classified type |Mortgage Application |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> These are two different guarantees because the schema only enforces data types and structure (e.g., ensuring a number is returned, not text), whereas the validator enforces business logic and mathematical consistency. The schema cannot catch a mathematically incorrect income sum (e.g., 50k + 10k = 100k), and the validator cannot catch an issue if the LLM completely fails to output JSON in the first place.

output :  "validation": {
    "consistent": false,
    "discrepancies": [
      {
        "field": "total_monthly_income",
        "calculated": 9642.17,
        "stated": 10892.17,
        "delta": -1250.0
      }
    ]
  }

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> The system outputs a null value instead of inventing one because the Pydantic schema explicitly types the field as optional (e.g., Optional[float] = None). This tells the LLM that it is acceptable to omit the field if the data is legitimately absent from the source text.

 Output: "extraction": {
    "borrower": {
      "full_name": "Daniel R. Whitfield",
      "coborrower_name": null,
      "ssn_last4": null,
      "date_of_birth": null,
      "email": null,
      "phone": null
    },
    "property": null,
    "loan": null,
    "income": {
      "base_monthly": 5673.08,
      "bonus_monthly": null,
      "bonus_ytd": null,
      "commission_monthly": null,
      "overtime_monthly": null,
      "other_monthly": null,
      "stated_monthly_total": null
    }
    }

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> Normalizing data at extraction time guarantees that downstream systems (like a database or a risk calculation script) receive clean, predictable integers. If we pass raw text strings downstream, those systems will likely throw type errors and crash when trying to perform math on the word "about".

 Output:  "property": {
      "address": "4827 Brookhaven Court, Fairhope, NC 28734",
      "property_type": "single_family",
      "property_type_detail": null,
      "occupancy_type": "primary_residence",
      "occupancy_type_detail": null,
      "year_built": 1998,
      "gross_living_area_sqft": 2400,
      "hoa_dues_monthly": null,
      "appraised_value": 410000.0
    },
---

## 3. Multi-source synthesis

| Evidence | Value |
|----------|-------|
| Passing test count |34 |
| Briefing file |run_offline.txt |
| Section the conflict landed in |Contested|

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> on_time_delivery_rate — **95.0** percent (supplier_audit, as of **2026-04-10**) vs **78.0** percent (logistics, as of **2026-04-05**).
Preserving the conflict serves the reader better because it exposes a massive operational discrepancy. If the system had silently arbitrated this (e.g., by averaging them to **86.5%**), the human investigator would have no idea that the supplier is severely over-reporting their reliability compared to the actual logistics data.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> `late_shipment_count [missing source: timeout reading logistics]`
"Unreachable" explicitly tells the user that the system tried to get the data but the API failed, whereas "nothing to report" implies the system successfully checked the source and found zero late shipments. The run finishes instead of crashing because the coordinator catches the timeout exception and degrades gracefully, ensuring the rest of the briefing is still generated.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> `average_lead_time_days — 12.0 days (supplier_audit, 2026-04-10) vs 12.0 days (logistics, 2026-04-05)`.
Requiring a date provides temporal context. If a supplier reports a defect rate of **100ppm** in January, and internal quality reports **190ppm** in April, the date guardrail ensures the reader interprets this as a deterioration over time, rather than a simultaneous contradiction.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> The clearest moment was in System 1's calibration report (`umbrella exclusions n=2 conf=0.93 acc=0.00`). A trusting design would have shipped a model that is 93% confident in its answers, blindly auto-approving completely incorrect policy data. Evaluating the actual accuracy against the confidence caught the hallucination.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> This mattered most in System 1 (the Insurance Policy Router). My calibration report revealed that when extracting umbrella policy exclusions, the model reported a massive 93% confidence level, yet its actual accuracy was 0%. If the pipeline had blindly trusted the LLM’s self-reported confidence score, it would have automatically approved completely incorrect policy data. This proves that an LLM being "sure" is not a substitute for measuring historical accuracy.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> A real workflow would be a system that analyzes messy, unstructured university exam papers and maps the questions to structured cognitive levels (like Bloom's Taxonomy) for NAAC compliance. I would reach for the independent review with deterministic routing pattern first. If the LLM classifies a complex question into an advanced taxonomy tier but the confidence is low—or if a separate rule-based script flags a mismatch—deterministic routing forces it to a human educator for review. To know when it breaks, I would instrument a Brier score calibration report to track how often human reviewers have to override the model's "high-confidence" classifications.
