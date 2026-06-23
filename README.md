# Provenance Guard

A backend system for classifying whether submitted creative writing is
human-authored or AI-generated. Built for Milestone 4 of AI201.

---

## Architecture Overview

A creator submits text via POST /submit. The rate limiter checks their
request count first — if over limit, returns 429. If allowed, two
independent signals analyze the text:

- Signal 1 (Groq LLM) sends the text to llama-3.3-70b-versatile and
  returns a 0–1 score for how AI-like the text is semantically.
- Signal 2 (Stylometric heuristics) computes three structural metrics
  in pure Python and returns a 0–1 score.

Both scores combine into a weighted confidence score. That score maps
to one of three transparency labels. The result writes to the audit
log. The JSON response returns content_id, attribution, confidence,
and label.

Appeal flow: a creator submits POST /appeal with their content_id and
reasoning. The system updates status to under_review in the audit log
and returns confirmation. No automated re-classification occurs.

---

## Detection Signals

### Signal 1 — Groq LLM (llama-3.3-70b-versatile)
- What it measures: semantic coherence, phrasing patterns, and
  stylistic uniformity holistically
- Why chosen: LLMs can assess whether writing "reads" as AI in ways
  pure statistics cannot — it captures meaning, not just structure
- Output: float 0–1 (0 = human-like, 1 = AI-like)
- What it misses: stylistically unusual human writing (experimental
  poetry, stream-of-consciousness) may score falsely high

### Signal 2 — Stylometric Heuristics (pure Python)
- What it measures: three structural properties:
  1. Sentence length variance — AI text is more uniform
  2. Type-token ratio — vocabulary diversity; AI reuses patterns
  3. Punctuation density — humans use more casual punctuation
- Why chosen: completely independent from Signal 1 (structural vs
  semantic) — disagreement between signals is itself informative
- Output: float 0–1 (average of 3 normalized metrics)
- What it misses: formal human writing (academic papers, legal text)
  scores as AI-like on these metrics

---

## Confidence Scoring

Combined score formula:
  confidence = (0.6 × llm_score) + (0.4 × stylo_score)

Groq is weighted higher (0.6) because semantic judgment is more
reliable than structural heuristics alone. The 0.4 weight on
stylometrics still meaningfully shifts borderline cases.

Thresholds:
- score > 0.65 → likely_ai
- score 0.35–0.65 → uncertain
- score < 0.35 → likely_human

The "likely_ai" threshold is deliberately set at 0.65 (not 0.5) to
bias toward uncertainty rather than false positives. A false positive
harms a human creator more than a false negative harms a platform.

Validation: tested on 4 inputs spanning the range:
- Clearly AI text: llm=0.8, stylo=0.56, combined=0.704 (likely_ai)
- Clearly human text: llm=0.2, stylo=0.359, combined=0.264 (likely_human)
- Formal human prose: lands in uncertain range as expected
- Lightly edited AI: lands in uncertain range as expected

---

## Transparency Label Variants

### High-confidence AI (score > 0.65)
"Our system detected patterns strongly associated with AI-generated
writing (confidence: {score}%). This does not prevent posting — it
gives readers context. If you wrote this yourself, you can appeal."

### Uncertain (score 0.35–0.65)
"Our system is not confident about the origin of this content
(confidence: {score}%). Detection is imperfect. If this label seems
wrong, you can submit an appeal."

### High-confidence human (score < 0.35)
"This content shows strong signals of human authorship (confidence:
{100 - score}%). No action needed."

---

## API Endpoints

### POST /submit
Accepts: { "text": "...", "creator_id": "..." }
Returns: { "content_id", "attribution", "confidence", "label" }

### POST /appeal
Accepts: { "content_id": "...", "creator_reasoning": "..." }
Returns: { "status": "received", "content_id", "message" }

### GET /log
Returns: { "entries": [ ...all audit log entries... ] }

---

## Rate Limiting

Limits applied to POST /submit:
- 10 requests per minute
- 100 requests per day

Reasoning: a real writer submitting their own work would submit at
most a few pieces per session — 10/minute is generous for legitimate
use. 100/day prevents scripted flooding while allowing heavy users.
These limits only apply to /submit because that is the expensive
endpoint (calls Groq API). /appeal and /log are not rate limited.

---

## Known Limitations

1. Formal human prose — academic papers, legal writing, and business
   reports written by humans use uniform sentence structure and formal
   vocabulary. Stylometric Signal 2 will score these as AI-like
   because it measures uniformity, not intent. These cases should land
   in "uncertain" but may occasionally tip into "likely_ai."

2. Heavily edited AI output — if a human takes AI-generated text and
   rewrites it substantially, the LLM signal may drop while the
   stylometric signal stays elevated. The combined score lands in
   "uncertain," which is the honest answer for genuinely ambiguous
   content.

---

## Spec Reflection

One way the spec helped: writing out the three label variants in
planning.md before touching any code forced a real UX decision — what
does "confidence: 62%" actually mean to a non-technical reader? That
thinking shaped the label wording directly.

One way implementation diverged: the spec suggested SQLite for the
audit log, but a JSON file was simpler to implement and sufficient for
this scale. In a production system with concurrent writes, SQLite or
a real database would be necessary.

---

## AI Usage

1. Prompted Claude to generate the Flask app skeleton and
   classify_with_groq() function, providing the Detection Signals
   section and architecture diagram as context. The generated prompt
   for Groq asked for a 0–10 integer — revised it to ask for a float
   directly to avoid a division step and reduce parsing errors.

2. Prompted Claude to generate the stylometric_score() function.
   The initial output used a single sentence length metric. Revised
   it to include all three metrics (variance, TTR, punctuation
   density) as specified in planning.md, and adjusted the
   normalization ranges after testing on sample inputs.

---

## Setup

1. Clone the repo
2. Create a virtual environment: python -m venv .venv
3. Activate: source .venv/bin/activate (Mac/Linux)
               .venv\Scripts\activate (Windows)
4. Install: pip install -r requirements.txt
5. Create .env with: GROQ_API_KEY=your_key_here
6. Run: python app.py