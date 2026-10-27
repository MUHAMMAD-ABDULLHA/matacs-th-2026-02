# Write-up

## Design in one paragraph
AI is used once per proposal, to read messy PDF text and return facts with quotes. Everything after that (gates, scoring, ranking, reasons) is plain code. A person reviews every result before an award. See `diagram/process-flow.png`.

**Assumptions (no reply was sought):** sub-criteria inside Warranty and Team are weighted equally (admin-configurable); exceeding a budget or timeline ceiling makes a proposal ineligible.

**Status precedence (code, in this order):**
1. Injection, unreadable text or an invalid extraction (after one retry) -> `flagged` (security first).
2. A mandatory requirement the proposal *confirms is not met*, or a stated price or timeline above its ceiling -> `ineligible`. For B, the proposal itself says the electrical work goes to an unconfirmed subcontractor, which is positive evidence the bidder does not hold the licence.
3. A mandatory requirement, ceiling or scoring input that is *missing, deferred or ambiguous* (silent, not contradicted) -> `flagged` for a human to clarify.
4. Otherwise it is scored.

**Scoring formula (code):** Experience = installs / best installs. Price = lowest price / price. Timeline = shortest weeks / weeks. Warranty = average of (panel years / best) and (workmanship years / best). Team = 25% each for named lead, crew specified, safety plan, and monitoring months / best months. Each is 0-1, multiplied by its weight. Only eligible, unflagged proposals are scored. Ties: more installs, then lower price, else shared rank.

**Comparison set and edge values:** the best-in-batch figures come from the eligible, unflagged proposals in that run only. This means a score can change when a proposal is added, removed or a flag is resolved; that is intended, the run records exactly which proposals were compared, and the reviewer sees a "scores recalculated" notice. A zero value earns 0 points; if the best value is 0 the criterion awards 0 to all; a missing price or timeline is flagged, never scored as 0. The admin screen refuses to save criteria unless weights are non-negative and total exactly 100, and a run refuses to start against a version that does not.

## 1. AI vs. rules
**AI:** reading each proposal's text and extracting facts (numbers, yes/no, a supporting quote, whether something was deferred) into a fixed JSON shape. Wording in PDFs varies too much for simple rules.
**Code:** file checks, hashing, injection pre-scan, schema and quote validation, mandatory gates, ceilings, scoring, sorting and the written reason. These are exact comparisons and arithmetic, where an LLM is less reliable and cannot be audited. The line is: AI reports what the text says; code decides what it means. The model never sees weights or thresholds and never ranks.

## 2. Consistency and explainability
- Same file hash + same criteria version + same prompt version + same model returns the **saved extraction** (a database record, no special cache service), and scoring is a pure function, so ranking the same proposals twice gives the same result. Temperature 0 helps but is not a guarantee, which is why caching matters.
- Every score is a table: each criterion, the extracted value, the quote and page, the points awarded. The reason line is built from a template using those same numbers, so the explanation cannot disagree with the score.
- Quotes must exist word-for-word in the source text or the proposal is flagged.

## 3. Human in the loop
- The reviewer sees every status, can open the source quote, resolve a flag (for example after asking a bidder to clarify) and override any rank or status **with a mandatory reason**. Resolving a flag re-runs the rules only.
- Overrides are added to the golden test set so the prompt improves.
- The award is always a human decision; the tool only recommends an order.
- Audit log (append-only): file hashes, criteria version, prompt version, model name, raw AI output, validated extraction, every score, each override (who, when, why), every admin criteria edit (who, when, what changed), and the final decision.

## 4. Changing criteria
Criteria are saved as immutable versions; every run records the version it used. An edit creates a new version and never changes old results. The old ranking stays visible, marked "scored under v1", and the reviewer can start a new run:
- **Weights, thresholds or ceilings changed:** re-run the rules only on cached extractions. No AI cost.
- **A criterion added or its definition changed:** re-extract only that criterion.
Earlier overrides do not carry over silently; the reviewer re-confirms them under the new version. Criteria for a finished award are read-only.

## 5. Risks
| Risk | Safeguard |
|---|---|
| Wrong extraction (misread table, scanned PDF) | Tables converted to Markdown before extraction; OCR fallback or flag on empty text; schema validation; reviewer sees each quote highlighted in the source PDF; golden-set tests before changing a prompt or model |
| Made-up facts | Every field is nullable and the prompt requires `null` when a fact is missing or deferred, so the model is never pushed to guess (D's schedule). Code then checks the quote exists word-for-word in the source and that any extracted number appears in its own quote. Failures are flagged, never auto-filled |
| Deferred or missing info (D, B) | Explicit `deferred` status; code flags or fails it, never guesses |
| Prompt injection (C) | Text treated as data; regex pre-scan plus model flag; model never sees weights; scoring is code, so "rank me first" has no effect; flagged for a human |
| Bias | Vendor names and branding are masked ("Bidder") before extraction; the model extracts counts, amounts and yes/no facts, never ratings; scoring uses only admin criteria; reviewers can audit and override |
| Rank swings from a gamed price or timeline | Mandatory gate runs first (B's low price is never scored), and ceilings are enforced |
| Cost | One call per proposal, cached by hash, file and page limits, re-scoring needs no AI |
| Model drift | Pin model and prompt versions; re-run golden set on any change |
