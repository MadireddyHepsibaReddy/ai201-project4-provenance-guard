# Provenance Guard — Planning

## Architecture Narrative

A creator submits text + creator_id via POST /submit. The rate limiter
checks their request count first — if over limit, it returns 429. If
allowed, Signal 1 (Groq LLM) sends the text to llama-3.3-70b-versatile
and gets back a 0–1 score for how AI-like the text is. Signal 2
(stylometric heuristics) computes sentence length variance, type-token
ratio, and punctuation density in pure Python — also returning 0–1.
Both scores combine into a weighted confidence score (0.6 × llm_score
+ 0.4 × stylo_score). That score maps to one of three transparency
labels. The result plus both signal scores write to the audit log. The
JSON response returns content_id, attribution, confidence, and label.

## Detection Signals

### Signal 1 — Groq LLM (llama-3.3-70b-versatile)
- What it measures: semantic coherence, phrasing patterns, stylistic
  uniformity holistically
- Output: float 0–1 (0 = human-like, 1 = AI-like)
- Why: LLMs can assess whether writing "reads" as AI in ways pure
  statistics cannot
- Blind spot: stylistically unusual human writing (e.g. experimental
  poetry) may score high

### Signal 2 — Stylometric Heuristics (pure Python)
- What it measures: three structural properties:
    1. Sentence length variance (low variance = AI-like)
    2. Type-token ratio / vocabulary diversity (low TTR = AI-like)
    3. Punctuation density (low casual punctuation = AI-like)
- Output: float 0–1 (average of 3 normalized metrics)
- Why: AI text is statistically more uniform than human writing
- Blind spot: formal human writing (academic papers, legal text) looks
  AI-like on these metrics

### Combining signals
combined_score = (0.6 × llm_score) + (0.4 × stylo_score)

Groq weighted higher because semantic judgment is more reliable than
structural heuristics alone. Asymmetry intentional: we bias toward
uncertainty rather than false positives.

## Uncertainty Representation

Score thresholds:
- score > 0.65  →  attribution: "likely_ai"
- score 0.35–0.65  →  attribution: "uncertain"
- score < 0.35  →  attribution: "likely_human"

A score of 0.60 means "we lean toward AI but are not confident" — the
label must reflect that honestly. A score of 0.95 means strong signal
from both detectors. These are not the same and the label text must
differ meaningfully between them.

False positive bias: we deliberately set the "likely_ai" threshold at
0.65 (not 0.5) so borderline cases fall into "uncertain" rather than
being flagged as AI. A false positive harms a human creator; a false
negative is less damaging.

## Transparency Label Variants

### High-confidence AI (score > 0.65)
"Our system detected patterns strongly associated with AI-generated
writing (confidence: {score}%). This does not prevent posting — it
gives readers context. If you wrote this yourself, you can appeal
below."

### Uncertain (score 0.35–0.65)
"Our system is not confident about the origin of this content
(confidence: {score}%). Detection is imperfect. If this label seems
wrong, the author can submit an appeal."

### High-confidence human (score < 0.35)
"This content shows strong signals of human authorship
(confidence: {100 - score}%). No action needed."

## Appeals Workflow

- Who can appeal: any creator, using the content_id from their
  submission response
- What they provide: content_id + creator_reasoning (free text)
- What the system does:
    1. Looks up the original audit log entry by content_id
    2. Updates status from "classified" to "under_review"
    3. Appends appeal_reasoning and appeal_timestamp to the log entry
    4. Returns confirmation JSON
- What a human reviewer sees: the full log entry including original
  scores, both signal values, the label that was shown, and the
  creator's reasoning
- Automated re-classification: NOT implemented

## Anticipated Edge Cases

1. Non-native English speaker writing formal prose — their sentence
   structure may be unusually uniform, pushing stylo_score high even
   though the text is human-written. System should land in "uncertain"
   and the label should invite appeal.

2. Lightly edited AI output — a human takes AI-generated text and
   edits it heavily. The LLM signal may drop but stylo_score stays
   high. Combined score lands mid-range. This is genuinely ambiguous
   and "uncertain" is the correct honest answer.

## Architecture Diagram

Submission flow:
POST /submit → Rate limiter → Signal 1 (Groq) → Signal 2 (Stylo)
    → Confidence scoring → Transparency label → Audit log → Response

Appeal flow:
POST /appeal → Lookup by content_id → Status: under_review
    → Log appeal + reasoning → Confirmation response

Utility:
GET /log → Return last N audit entries as JSON

## AI Tool Plan

### M3 — Submission endpoint + Signal 1
- Spec sections to provide: Detection Signals + Architecture Diagram
- Ask AI to generate:
    1. Flask app skeleton with POST /submit stub and GET /log stub
    2. classify_with_groq(text) function returning float 0–1
- How I will verify: call classify_with_groq() directly on 2 test
  inputs before wiring into the endpoint; check the float comes back
  in range

### M4 — Signal 2 + confidence scoring
- Spec sections to provide: Detection Signals + Uncertainty
  Representation + Architecture Diagram
- Ask AI to generate:
    1. stylometric_score(text) function with 3 metrics
    2. combine_scores(llm, stylo) returning weighted float
- How I will verify: test on 4 inputs (clearly AI, clearly human,
  two borderline); print both signal scores separately; check scores
  vary meaningfully

### M5 — Production layer
- Spec sections to provide: Transparency Label Variants + Appeals
  Workflow + Architecture Diagram
- Ask AI to generate:
    1. get_label(score) returning correct label text per threshold
    2. POST /appeal endpoint with status update + log append
- How I will verify: submit inputs at 3 different score ranges to
  confirm all 3 label variants are reachable; submit an appeal and
  check GET /log shows under_review status