# Provenance Guard — Planning Document

## Architecture

### Submission flow narrative
A piece of text enters via `POST /submit` with a `creator_id`. It passes through two independent detection signals in sequence: first the Groq LLM signal (semantic/holistic assessment), then the stylometric heuristics signal (structural/statistical properties). Both signals produce a score between 0.0 and 1.0. A confidence scoring function combines them via weighted average into a single score. That score is mapped to one of three transparency label variants. The full result — timestamp, content_id, creator_id, both signal scores, combined confidence, label text, and status — is written to the audit log. The endpoint returns a JSON response containing the content_id, attribution result, confidence score, and label text.

### Appeal flow narrative
A creator submits `POST /appeal` with their `content_id` and a written `creator_reasoning`. The system looks up the original classification, updates the content's status to `"under_review"`, and appends an appeal entry to the audit log containing the original decision plus the creator's reasoning. A confirmation response is returned. No automated re-classification occurs; a human reviewer handles it.

### Diagram
SUBMISSION FLOW

───────────────

POST /submit ──[raw text]──► Groq LLM signal ──[llm_score]──►┐

├──► Confidence scoring ──[0.0–1.0]──► Transparency label ──► Audit log

──[raw text]──► Stylometric signal ──[style_score]──►┘

│

◄──────────[JSON: content_id, attribution, confidence, label]──────────────────────────┘
APPEAL FLOW

───────────

POST /appeal ──[content_id, reasoning]──► Status update → "under_review" ──[appeal entry]──► Audit log

◄──────────────────────[confirmation: appeal received]──────────────────────────────────────


---

## Detection Signals

### Signal 1 — Groq LLM classification
**What it measures:** Semantic and stylistic coherence as assessed holistically by a large language model. The model is prompted to evaluate whether the text reads as human-written or AI-generated based on phrasing patterns, idea development, and stylistic consistency.

**Output format:** A float between 0.0 and 1.0, where 1.0 = confident AI-generated, 0.0 = confident human-written. Extracted from a structured JSON response in the Groq API call.

**Why it differs between human and AI writing:** LLM-generated text tends toward polished coherence, balanced hedging, and formulaic transitions. Human writing varies more unpredictably in structure and tone.

**Blind spot:** It can be fooled by deliberately casual AI output or by very formal human writing (academic papers, legal documents). It is also a black box — we cannot inspect why it scored what it did.

---

### Signal 2 — Stylometric heuristics
**What it measures:** Statistical structural properties of the text. Specifically:
- **Sentence length variance:** Standard deviation of sentence lengths in words. AI text tends to be more uniform; human text is burstier.
- **Type-token ratio (TTR):** Unique words ÷ total words. AI text often repeats vocabulary in a balanced, uniform way; human text is more variable.
- **Punctuation density:** Punctuation marks ÷ total characters. Humans use dashes, ellipses, and irregular punctuation more than AI.

**Output format:** Each metric is normalized to a 0.0–1.0 subscale (higher = more AI-like). The three subscales are averaged into a single `style_score` float.

**Why it differs between human and AI writing:** AI models are trained to produce smooth, readable output — which results in statistically uniform sentence structures and vocabulary patterns. Humans are messier and more variable by nature.

**Blind spot:** Formal human writing (academic essays, grant proposals) will score as AI-like on stylometrics because it is deliberately uniform. Short texts (under 5 sentences) will produce unreliable variance estimates.

---

## Uncertainty Representation

### Confidence scoring formula
confidence = (0.6 × llm_score) + (0.4 × style_score)
The LLM signal carries more weight (0.6) because it captures semantic meaning, which is harder to fake than structural uniformity. Stylometrics carry 0.4 — useful, but more easily gamed by formal human writers.

### What each score range means

| Score range | Meaning | Label category |
|---|---|---|
| > 0.70 | Strong agreement between both signals that text is AI-generated | High-confidence AI |
| 0.45 – 0.70 | Signals disagree or both are uncertain | Uncertain |
| < 0.45 | Strong agreement that text is human-written | High-confidence human |

### Why 0.45 not 0.50 as the lower threshold
A false positive (labeling a human as AI) is worse than a false negative on a writing platform. Setting the lower bound at 0.45 rather than 0.50 means the system requires stronger evidence before clearing content as "likely human" — giving uncertain cases a wider safety band that routes them to the Uncertain label instead of a clean human verdict.

### What 0.6 means to this system
A score of 0.6 means the signals lean toward AI-generated but without strong agreement. The system will label this "Uncertain — our analysis was inconclusive." This is intentional: 0.6 is not actionable enough to call definitively AI, and a creator in this band who appeals should find the system credible and fair.

---

## Transparency Label Variants

Three label variants, mapped to confidence score ranges. These are the exact strings the system returns.

### High-confidence AI (confidence > 0.70)
⚠ Likely AI-generated

Our analysis found strong indicators that this content may have been generated by an AI tool.

This label reflects automated analysis and may not be accurate. If you created this work yourself,

you can submit an appeal and a human reviewer will take a closer look.

### Uncertain (confidence 0.45–0.70)
◐ Analysis inconclusive

Our analysis found mixed signals — this content has some characteristics of AI-generated text

and some characteristics of human writing. No strong determination was made.

If you feel this assessment is inaccurate, you can submit an appeal.

### High-confidence human (confidence < 0.45)
✓ Appears human-written

Our analysis found no strong indicators of AI generation in this content.

This label reflects automated analysis only. Attribution labels are never fully definitive.

---

## Anticipated Edge Cases

### Edge case 1 — Poet using intentional repetition and simple vocabulary
A human poet writing in a minimalist or incantatory style (short sentences, repeated words, low vocabulary diversity) will score high on stylometric AI-likelihood because those properties — low TTR, low sentence length variance — match AI patterns statistically. The LLM signal may or may not catch the stylistic intentionality. This is a known false positive risk. Mitigation: the Uncertain band (0.45–0.70) acts as a buffer, and the appeal pathway is always available.

### Edge case 2 — Heavily edited AI output
A creator who generates text with AI and then substantially rewrites it produces a hybrid artifact. The stylometric signal may detect residual uniformity; the LLM signal may or may not be fooled by the edits. Expected result: a mid-range confidence score (0.50–0.65) landing in the Uncertain band. The system will not label this definitively AI — which is arguably the correct call, since the final product reflects significant human creative input.

---

## AI Tool Plan

### Milestone 3 — Submission endpoint + Signal 1
**Sections to provide:** Detection Signals (Signal 1 only), Architecture diagram  
**What to ask for:** Flask app skeleton with `POST /submit` route stub; a `classify_with_groq(text)` function that calls the Groq API and returns a float 0.0–1.0; a `write_log_entry(entry_dict)` helper that appends JSON to a log file  
**Verification:** Call `classify_with_groq()` directly with the four test inputs from the spec. Confirm output is a float, not a string or dict. Confirm the Flask route returns a JSON body with `content_id`, `attribution`, `confidence` (placeholder), and `label` (placeholder).

### Milestone 4 — Signal 2 + confidence scoring
**Sections to provide:** Detection Signals (Signal 2), Uncertainty Representation, Architecture diagram  
**What to ask for:** A `compute_stylometric_score(text)` function that returns a float 0.0–1.0 from sentence length variance, TTR, and punctuation density; a `combine_scores(llm_score, style_score)` function implementing the weighted formula  
**Verification:** Run all four test inputs (clearly AI, clearly human, two borderline). Confirm clearly AI > 0.70 and clearly human < 0.45. Confirm borderline cases fall in 0.45–0.70. Print both individual signal scores for any case that doesn't match intuition.

### Milestone 5 — Production layer
**Sections to provide:** Transparency Label Variants, Appeals Workflow, Architecture diagram  
**What to ask for:** A `generate_label(confidence)` function that returns the correct label text string based on thresholds; a `POST /appeal` endpoint; Flask-Limiter setup on `/submit`  
**Verification:** Call `generate_label()` with 0.30, 0.58, and 0.85 — confirm all three variant texts appear. Submit an appeal via curl, then query `GET /log` and confirm `status: "under_review"` and `appeal_reasoning` are present. Run the 12-request rate-limit test and confirm 429s appear after request 10.