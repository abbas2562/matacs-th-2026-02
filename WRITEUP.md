# Write-up: AI Proposal Triage Design (MATACS-TH-2026-02)

**Assumptions (my decisions, not given in the brief)**
- The CAD 260,000 ceiling and 16-week limit are hard project limits: a price over the ceiling or a stated schedule over 16 weeks is *ineligible*; a missing schedule is *flagged*.
- The licence check passes only if the **bidder itself** holds the licence and says so. A subcontractor "to be confirmed" does not pass.
- Missing information is *unknown*, never assumed true. Flagged and ineligible proposals get `rank: null` and `score: null`.
- The system reads claims; it does not verify them with ESA or WSIB. A reviewer does that before award.

## 1. AI vs. rules

| Step | Who does it | Why |
|---|---|---|
| File checks, hashing | Code | Exact, cheap, no judgement |
| PDF text extraction | Code (OCR only if scanned) | Deterministic and keeps page numbers |
| Extracting facts into JSON (Prompt A) | **AI** | Proposals are free text in many styles; reading them is what AI is good at |
| Evidence mapping and gap notes (Prompt B) | **AI, advisory** | Helps the reviewer see what supports each criterion; produces no scores |
| Checking AI output (schema, quotes exist in source) | Code | An AI answer is not trusted until verified |
| Mandatory eligibility | Code | Legal and fairness impact; must be exact |
| Weighted scoring, ranking, report | Code | Arithmetic must be repeatable and explainable |
| Award decision | Human | Accountability |

The line is: **AI reads, code decides.** AI turns messy text into facts with quotes. Every decision that affects who wins is a fixed rule.

## 2. Consistency and explainability

- Scores come from written formulas (see README) applied to confirmed facts and a **pinned criteria version**. The calculation is a pure function: same facts + same version = same score and rank, every time. Ties use a fixed tie-break order.
- AI extraction is the only variable part. I limit it with temperature 0, a pinned model and prompt version, a strict JSON schema, and a code check that every quote exists in the source. Extracted facts are **saved and human-confirmed**, so a re-rank reuses stored facts rather than calling AI again. Re-extracting the PDF could differ slightly; that is why facts are stored.
- A reviewer sees, for each proposal: the fact, its quote and page, the criterion score, weight and points, then the total. Ineligible or flagged proposals show the exact rule that triggered. Warnings (missing data, security) appear beside the result.

## 3. Human in the loop

Review points: (1) admin approves criteria before use; (2) reviewer confirms or corrects extracted facts beside source quotes; (3) reviewer resolves flagged proposals, e.g. asks the bidder for clarification; (4) reviewer must acknowledge security warnings; (5) reviewer may override a result with a mandatory written reason; (6) a human makes the final award.

Overrides never silently edit AI output. They create a new fact version (who, when, why), and the system recomputes. The calculated rank stays visible next to any manual change.

**Logged (append-only):** file hash and upload ID; extracted facts with quotes and pages; model, prompt and schema versions; criteria version; validation failures and flags; score breakdowns; every reviewer edit with before/after, name, timestamp and reason; clarification requests; the final decision and rationale. I would not copy full PDFs into the log (hash plus storage reference is enough), and API keys are never logged.

## 4. Changing criteria

- Criteria versions are **immutable**. Editing creates v(N+1); weights must total 100% to save.
- Existing runs stay linked to their original version and remain viewable and reproducible.
- Using the new version means a **new evaluation run** on the stored confirmed facts, since scoring is code. AI is called again only if the new version needs a fact not yet extracted.
- The UI shows the version on every result and only ranks proposals scored under the same version. It shows a banner ("newer criteria available, re-score?") with a weight diff, and never merges mixed-version results into one list.
- Once an award is recorded, that run is locked.

## 5. Risks and safeguards

| Risk | Safeguard |
|---|---|
| **Wrong PDF extraction** (tables, scans) | Page-referenced text, quality check, flag on failure; reviewer sees quotes beside facts |
| **Made-up facts** | Schema forces `null` for absent values; code checks each quote appears verbatim and numbers match; unsupported items are flagged |
| **Prompt injection** (one sample asks to be ranked first) | Proposal text is wrapped as data and never mixed with instructions; AI has no tool or score authority; code scans for injection phrases, logs a security warning, and ignores the text. Even if the model obeyed, scores and eligibility are computed by code, so there is nothing to hijack |
| **Wrong eligibility decision** | Rules need status `stated` plus a quote; `inferred` or `unknown` never passes; unknown means flagged, not failed or passed |
| **Biased or inconsistent scoring** | Fixed formulas, no AI scoring; same rules for all bidders; reviewer sees all breakdowns |
| **Ambiguous evidence** (e.g. "since 2021" against a 5-year window) | Counted as stated, with a warning for reviewer verification |
| **Confidential data disclosure** | Send only proposal text and criteria to the AI; no keys or personal data in logs; access limited to reviewers; use a provider under a no-training agreement |
| **Cost, latency, outage** | Cache by file hash; one extraction call per proposal; one retry; a failed proposal is flagged and the others continue |
| **Criteria changes** | Versioning as in Q4 |
| **Human overreliance** | The rank is labelled as triage support; reviewers must acknowledge warnings; the UI shows evidence before the rank; overrides are easy and logged |
