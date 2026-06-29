# Provenance Guard

A backend API for detecting AI-generated content on creative platforms. Built for AI201 Project 4.

---

## Architecture Overview

A submitted piece of text takes the following path through the system:

1. `POST /submit` receives the text and a `creator_id`
2. The text is sent to the **Groq LLM signal** — the model scores it 0.0–1.0 for AI likelihood based on semantic and stylistic coherence
3. The same text is run through the **stylometric heuristics signal** — pure Python computes sentence length variance, type-token ratio, and punctuation density into a single score
4. Both scores are combined via weighted average into a single **confidence score**
5. The confidence score is mapped to one of three **transparency label** variants
6. The full result is written to the **audit log**
7. A JSON response is returned to the caller containing the `content_id`, attribution, confidence, both signal scores, and label text

For appeals:
1. `POST /appeal` receives a `content_id` and `creator_reasoning`
2. The system looks up the original classification and updates its status to `under_review`
3. An appeal entry is appended to the audit log with the creator's reasoning
4. A confirmation response is returned

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
**What it measures:** Semantic and stylistic coherence as assessed holistically by `llama-3.3-70b-versatile`. The model is prompted to evaluate whether the text reads as human-written or AI-generated based on phrasing patterns, idea development, and stylistic consistency. Returns a float 0.0–1.0.

**Why I chose it:** LLM-based classification captures meaning and tone in a way that pure statistics cannot. It can detect formulaic transitions, balanced hedging, and overly polished structure that are hallmarks of AI output.

**What it misses:** It can be fooled by deliberately casual AI output or very formal human writing. It is also a black box — there is no way to inspect why it scored what it did.

### Signal 2 — Stylometric heuristics
**What it measures:** Three statistical structural properties:
- **Sentence length variance** — standard deviation of sentence lengths in words. AI text is more uniform; human text is burstier.
- **Type-token ratio (TTR)** — unique words ÷ total words. AI text repeats vocabulary in a balanced way; human text is more variable.
- **Punctuation density** — irregular punctuation marks ÷ total characters. Humans use dashes, ellipses, and parentheses more than AI.

Each metric is normalized to 0.0–1.0 and averaged into a single `style_score`.

**Why I chose it:** Stylometrics are structurally independent from the LLM signal — one is semantic, one is statistical. Combining genuinely different signals produces more reliable results than two versions of the same approach.

**What it misses:** Formal human writing (academic essays, legal documents) scores as AI-like because it is deliberately uniform. Short texts under 5 sentences produce unreliable variance estimates.

---

## Confidence Scoring

### Formula

confidence = (0.6 × llm_score) + (0.4 × style_score)

The LLM signal carries more weight (0.6) because semantic meaning is harder to fake than structural uniformity. Stylometrics carry 0.4 — useful signal but more easily triggered by formal human writers.

### Thresholds
| Score range | Label category |
|---|---|
| > 0.70 | High-confidence AI |
| 0.45 – 0.70 | Uncertain |
| < 0.45 | High-confidence human |

The lower bound is 0.45 rather than 0.50 because a false positive (labeling a human as AI) is worse than a false negative on a writing platform. The wider uncertain band gives borderline cases a fair path to appeal.

### Example submissions with actual scores

**High-confidence uncertain (AI-leaning):**
Text: "Artificial intelligence represents a transformative paradigm shift in modern

society. It is important to note that while the benefits of AI are numerous, it is

equally essential to consider the ethical implications..."

llm_score: 0.80

style_score: 0.5242

confidence: 0.6897

label: ◐ Analysis inconclusive

**High-confidence human:**
Text: "ok so i finally tried that new ramen place downtown and honestly? underwhelming.

the broth was fine but they put WAY too much sodium in it..."

llm_score: 0.20

style_score: 0.485

confidence: 0.314

label: ✓ Appears human-written

---

## Transparency Label

Three variants returned by the system based on confidence score.

### High-confidence AI (confidence > 0.70)
```
⚠ Likely AI-generated
Our analysis found strong indicators that this content may have been generated by an AI tool.
This label reflects automated analysis and may not be accurate. If you created this work yourself,
you can submit an appeal and a human reviewer will take a closer look.
```

### Uncertain (confidence 0.45–0.70)
```
◐ Analysis inconclusive
Our analysis found mixed signals — this content has some characteristics of AI-generated text
and some characteristics of human writing. No strong determination was made.
If you feel this assessment is inaccurate, you can submit an appeal.
```

### High-confidence human (confidence < 0.45)
```
✓ Appears human-written
Our analysis found no strong indicators of AI generation in this content.
This label reflects automated analysis only. Attribution labels are never fully definitive.
```
---

## Rate Limiting

**Limits applied to `POST /submit`:** 10 requests per minute, 100 requests per day.

**Reasoning:**
- 10 per minute reflects realistic creator behavior — a writer submitting their own work would never need more than a few submissions per minute
- 100 per day prevents scripted flooding while allowing a platform with moderate traffic to operate normally
- An adversary trying to probe the system or flood it with submissions would hit the per-minute cap almost immediately

**Evidence — rate limit test output (12 rapid requests):**
200

200

200

200

200

200

200

200

200

200

429

429

Requests 11 and 12 return `429 Too Many Requests` as expected.

---

## Known Limitations

**Formal human writing scores as AI-like on stylometrics.** A human writing an academic essay, legal brief, or grant proposal produces text that is deliberately uniform in sentence length and vocabulary — the exact properties stylometrics flag as AI-generated. In testing, a two-sentence passage about monetary policy scored `style_score: 0.53` despite being clearly human-written. The LLM signal partially compensates, but combined confidence can still land in the uncertain band for this content type. The appeal pathway is the mitigation — but it puts the burden on the creator to contest a label that was never well-founded.

**Short texts are unreliable.** Sentence length variance requires multiple sentences to be meaningful. A single-sentence submission produces a variance of zero, which the system interprets as AI-like. Any submission under 3–4 sentences should be treated with extra skepticism regardless of score.

---

## Spec Reflection

**One way the spec helped:** Writing out the three label variants in planning.md before touching any code meant the `generate_label()` function had a concrete spec to implement against. The exact strings were already decided, it was easy to understand what the function should return at each threshold.

**One way implementation diverged from the spec:** I kept the almost the same log structure and the same structure for weighing the signals. I have adjusted the score a little differnt, I made the uncertainty cut out at .70 which can be considered low and the the model might get run into more erros labeling human work for AI.

---

## AI Usage

**Instance 1 — Stylometric signal function**
I directed Claude to generate a `compute_stylometric_score(text)` function that measures 
sentence length variance, type-token ratio, and punctuation density, returning a single 
float 0.0–1.0. Claude produced the function with all three metrics and normalization logic. 
I reviewed the normalization approach for punctuation density — the multiplier of 20 felt 
arbitrary, but after testing it produced reasonable separation between human and AI text 
so I kept it.

**Instance 2 — Appeal endpoint**
I directed Claude to generate a `POST /appeal` endpoint that accepts a `content_id` and 
`creator_reasoning`, updates the original entry's status to `under_review`, appends a 
new appeal entry to the audit log, and returns a confirmation response. Claude produced 
the full endpoint. I removed a duplicate `GET /log` route that Claude included alongside 
it since I already had one, which was causing a Flask startup error.
---

## API Reference

### `POST /submit`
Accepts a piece of text for attribution analysis.

**Request body:**
```json
{
  "text": "string",
  "creator_id": "string"
}
```

**Response:**
```json
{
  "content_id": "uuid",
  "attribution": "likely_ai | likely_human",
  "confidence": 0.0,
  "llm_score": 0.0,
  "style_score": 0.0,
  "label": "string"
}
```

### `POST /appeal`
Contest a classification.

**Request body:**
```json
{
  "content_id": "uuid",
  "creator_reasoning": "string"
}
```

**Response:**
```json
{
  "message": "Appeal received. Your submission is now under review.",
  "content_id": "uuid",
  "status": "under_review"
}
```

### `GET /log`
Returns all audit log entries.

**Response:**
```json
{
  "entries": []
}
```

---

## Audit Log Sample

Run `GET /log` or open `audit_log.json` to see structured entries. Each classification entry contains:

```json
{
  "content_id": "uuid",
  "creator_id": "string",
  "timestamp": "ISO8601",
  "attribution": "likely_ai | likely_human",
  "confidence": 0.0,
  "llm_score": 0.0,
  "style_score": 0.0,
  "label": "string",
  "status": "classified | under_review",
  "event_type": "classification"
}
```

Appeal entries additionally contain:
```json
{
  "event_type": "appeal",
  "original_attribution": "string",
  "original_confidence": 0.0,
  "appeal_reasoning": "string"
}
```

## Setup

```bash
git clone <your-repo-url>
cd ai201-project4-provenance-guard
python -m venv .venv
source .venv/bin/activate  # Mac/Linux
# or .venv\Scripts\activate  # Windows
pip install -r requirements.txt
```

Create a `.env` file: