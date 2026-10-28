# MATACS-TH-2026-02: AI Proposal Triage Design

- **Name:** [YOUR FULL NAME]
- **Assessment reference:** MATACS-TH-2026-02
- **Time actually spent:** [FILL IN: actual time, e.g. "about 75 minutes"]

## Files

| File | What it is |
|---|---|
| [WRITEUP.md](./WRITEUP.md) | Answers to the five questions (under 1,000 words) |
| [PROMPTS.md](./PROMPTS.md) | The exact extraction and evidence-assist prompts, schemas, and the checks done in code |
| [ranking.json](./ranking.json) | Expected ranking of proposals A to E |
| [diagram/process-flow.png](./diagram/process-flow.png) | Flow diagram from upload to the human award decision |

**Suggested reading order:** the diagram, then WRITEUP.md, then ranking.json with the scoring section below, then PROMPTS.md.

## Design in brief

The principle is **AI reads, code decides, humans approve.**

1. An admin saves criteria as an immutable, versioned configuration. Nothing is hard-coded.
2. A reviewer uploads PDFs; code validates, hashes and extracts the text with page numbers.
3. **AI (Prompt A)** turns each proposal into structured JSON facts with quotes. Proposal text is treated as untrusted data.
4. Code validates the JSON, checks every quote exists in the source, and scans for injection attempts. A reviewer confirms or corrects the facts.
5. **Code** checks the mandatory criteria first, separately from the score. A fail is *ineligible*; unknown or conflicting is *flagged*. Neither is scored or ranked.
6. **Code** scores eligible proposals with fixed formulas, ranks them, and produces a report. AI (Prompt B) only helps explain evidence and gaps.
7. A human reviews, overrides if needed (with a reason), and makes the final award. An audit log records each step.

## Scoring method (defaults the admin can change in the configuration)

Each weighted criterion is scored 0 to 100, then `total = sum(weight × score)`, rounded once to one decimal.

| Criterion (weight) | Score 0 to 100 |
|---|---|
| Experience (30%) | `min(100, 100 × qualifying installs ÷ 12)` (installs of 100 kW or more in the last 5 years, as stated) |
| Price (25%) | Linear from the ceiling (0) to 80% of the ceiling (100): `100 × (260,000 − price) ÷ 52,000`. Absolute, so a score does not depend on the other bidders |
| Timeline (15%) | Linear from 16 weeks (0) to 8 weeks (100): `100 × (16 − weeks) ÷ 8` |
| Warranty (15%) | Average of panel `min(100, 100 × years ÷ 25)` and workmanship `min(100, 100 × years ÷ 15)` |
| Team and approach (15%) | 25 points each for: named project lead, crew described, safety plan, monitoring plan. Not evidenced scores 0 for that item, with a warning |

**Missing, ambiguous or conflicting data:** unknown price, schedule or install count means *flagged* (no score), because the score would be a guess. Unknown optional team items score 0 with a warning, because they are not evidenced. Persuasive wording is never evidence.
**Ties:** higher total, then higher experience score, then lower price, then shorter schedule, then proposal letter, with a "tie" warning for the reviewer.
**Versioning:** every run stores the criteria version, and only runs with the same version can be compared.

### Sample results

| Proposal | Status | Score | Main reason |
|---|---|---:|---|
| A | ranked 1 | 75.6 | Passes mandatory; strongest experience, price and schedule among eligible bidders |
| E | ranked 2 | 62.5 | Passes mandatory; best warranty, full team and approach; less experience and a slower schedule |
| C | ranked 3 | 53.9 | Passes mandatory; no safety or monitoring plan evidenced; embedded "rank me first" text ignored and flagged as a security warning |
| D | flagged | null | No stated completion time, so the 16-week limit cannot be confirmed |
| B | ineligible | null | Bidder does not state it holds the licence; relies on a subcontractor still to be confirmed |

Note on D: its result matters. I calculated outside the ranking that if D confirmed a schedule of 14 weeks or less, it would outrank E. That is why it goes to a human with a clarification request instead of being dropped.

### Assumptions (also in WRITEUP.md)

- The budget and 16-week limits are hard limits.
- The licence must be held by the bidder; "subcontractor to be confirmed" fails.
- Stated install counts are used as given. Some proposals say "since 2020/2021", which may include work older than 5 years. I count them as stated and add a "verify the window" warning. This does not change the order here.
- The system reads claims; it does not check them against ESA or WSIB registers. A reviewer verifies before award.
- Scoring constants (12 installs, 80% price band, 8-week full-marks schedule, 25/15-year warranty caps) are my reasonable defaults. They are configuration values, not code.

## How I used AI tools

I used Claude to help draft the structure, prompts and flow diagram, and then checked the scoring numbers with a separate calculation. **[VERIFY AND EDIT: keep only if it matches what you really did.]** For example, the AI's first draft [treated Proposal B's "licensed subcontractor to be confirmed" as a pass with a warning because of its price and schedule; I changed it to ineligible because the admin criterion says the licence must be held by the bidder and stated in the proposal, and a pending subcontractor is not that].

> **Note to self (delete before publishing):** the bracketed disagreement above is a suggested example based on the assessment, not a record of what you did. Replace it with a real correction you made, or keep it only if it genuinely reflects your review.

Nothing here used real client or personal data. No code or API keys are in this repository.
