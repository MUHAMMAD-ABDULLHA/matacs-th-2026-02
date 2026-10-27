# MATACS-TH-2026-02: AI Proposal Triage Design

- **Name:** Muhammad Abdullah
- **Reference:** MATACS-TH-2026-02
- **Time actually spent:** About 1 hour 15 minutes

## How I used AI tools
I used Claude to understand the problem first, then to challenge my own design with questions about edge cases and system design, such as how to save extractions so the same proposal is not re-read (using a file hash), what the hash does and does not prove, and what to do when criteria change after scoring. I then had Claude help draft the files, and I checked every number and rule against the brief myself.

**Where I corrected or disagreed with the AI:**
- The first prompt sent criteria weights to the model and hard-coded criterion names in the output schema. That breaks "criteria are never hard-coded" and gives injected text a target, so the model now only learns what to find and weights stay in code.
- The first ranking had placeholder scores. I replaced them with scores calculated from the formula in WRITEUP.md.
- A review said A's score should be 97.5. I recalculated and kept 96.9: the review had A's warranty score wrong.
- A review said B should be flagged, not ineligible. I checked the brief: the criterion is a licence held by the bidder and stated in the proposal, and B states neither, so B stays ineligible and a reviewer can reverse it with a reason.
- I asked for extra safeguards found by comparison: a clearer hallucination check (nullable fields, quote and number checks), masking vendor names, table conversion, quote highlighting, logging admin edits, and a clear order for flagged, ineligible and ranked.

## Files
- [WRITEUP.md](WRITEUP.md): answers to the five questions, plus the scoring formula
- [PROMPTS.md](PROMPTS.md): extraction prompt, exactly as sent
- [ranking.json](ranking.json): expected ranking of proposals A-E
- [diagram/process-flow.png](diagram/process-flow.png): flow from upload to ranked output