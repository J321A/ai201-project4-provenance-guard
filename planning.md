# Provenance Guard — Planning / Spec

A backend service that any creative-sharing platform can plug into to classify
submitted text as human- or AI-written, score confidence honestly, surface a
plain-language transparency label, and let creators appeal a verdict.

---

## Architecture narrative

A piece of text enters through `POST /submit` with a `text` and `creator_id`.
The text is run through **two independent detection signals** (an LLM semantic
classifier and a structural stylometric analyzer). Each signal returns a
probability that the text is AI-generated (`p_ai` in 0–1). The **confidence
scorer** combines the two signals into a single `p_ai`, picks an attribution
verdict (`likely_ai` / `uncertain` / `likely_human`), and converts that into a
calibrated confidence number. The **label generator** maps the verdict +
confidence to one of three human-readable transparency labels. Everything —
content id, both raw signal scores, combined score, verdict, label — is written
to a structured **SQLite audit log** and returned to the caller.

A creator who disagrees calls `POST /appeal` with the `content_id` and their
reasoning. The system flips the content's status to `under_review`, appends an
appeal row to the audit log alongside the original decision, and confirms
receipt. `GET /log` exposes recent audit entries for transparency/grading.

## Architecture

```
                          POST /submit { text, creator_id }
                                       |
                                       v
                          +------------------------+
                          |   Submission handler    |
                          |   (rate-limited)        |
                          +------------------------+
                  raw text  |                     | raw text
                            v                     v
              +----------------------+   +--------------------------+
              | Signal 1: Groq LLM   |   | Signal 2: Stylometry     |
              | semantic classifier  |   | structural heuristics    |
              | -> p_ai (0-1)        |   | -> p_ai (0-1)            |
              +----------------------+   +--------------------------+
                            |                     |
                p_ai_llm    +----------+----------+   p_ai_style
                                       v
                          +------------------------+
                          | Confidence scorer       |
                          | combine -> p_ai         |
                          | -> verdict + confidence |
                          +------------------------+
                                       | verdict, confidence
                                       v
                          +------------------------+
                          | Label generator         |
                          | -> transparency label   |
                          +------------------------+
                                       |
                 content_id, scores,   v
                 verdict, label   +------------------------+
                 +--------------->|  SQLite audit log       |
                 |                +------------------------+
                 v                          |
        JSON response  <---------------------+
        { content_id, attribution, confidence, label, signals }


   APPEAL FLOW:
   POST /appeal { content_id, creator_reasoning }
        |  content_id, reasoning
        v
   +------------------------+   status: classified -> under_review
   | Appeal handler         |---------------------------------> SQLite
   +------------------------+   append appeal entry (links original decision)
        |
        v
   JSON { status: "under_review", confirmation }
```

---

## 1. Detection signals

We use **two genuinely independent signals** — one semantic, one structural.

### Signal 1 — Groq LLM semantic classifier
- **Measures:** holistic semantic + stylistic coherence. Does the text read as
  human (idiosyncratic voice, lived detail, irregular reasoning) or as
  machine-smooth (hedged, balanced, generically fluent)?
- **Why it differs:** AI prose tends toward even, hedged, "on-the-one-hand"
  structure and generic transitions; human writing carries voice, specific
  detail, and uneven emphasis. An LLM judge captures this gestalt better than
  any single statistic.
- **Output:** `p_ai` ∈ [0,1] (probability the text is AI-generated), parsed
  from a strict JSON response.
- **Blind spot:** can be fooled by heavily-edited AI text or by highly polished
  human writing; also non-deterministic and dependent on an external API.

### Signal 2 — Stylometric heuristics (pure Python)
Computes three structural metrics and blends them:
- **Sentence-length variance (burstiness):** humans vary sentence length a lot;
  AI is uniform. Low variance → more AI-like.
- **Type-token ratio (lexical diversity):** measured against expected diversity;
  AI text is often mid-range and repetitive in function words.
- **Punctuation/contraction density:** casual human writing uses contractions,
  dashes, ellipses, lowercasing; formal AI output is cleaner.
- **Output:** `p_ai` ∈ [0,1].
- **Blind spot:** short texts give unstable statistics; formal human writing
  (academic, legal) looks "uniform" and can be mis-scored as AI — the canonical
  false-positive risk this project warns about.

**Combination:** `p_ai = 0.60 * p_ai_llm + 0.40 * p_ai_style`. The LLM is
weighted higher because it captures semantics; stylometry is the independent
structural check. If the LLM signal is unavailable (no key / API error), we fall
back to stylometry alone and flag the result as `degraded`.

## 2. Uncertainty representation

- `p_ai` is the combined probability the text is AI-generated.
- **Verdict thresholds (asymmetric, biased against false-positive AI calls):**
  - `p_ai >= 0.65` → **likely_ai**
  - `p_ai <= 0.40` → **likely_human**
  - otherwise        → **uncertain**
  The "human" band is wider than the "AI" band on purpose: labeling a human's
  work as AI is the costlier error on a writing platform, so we require stronger
  evidence to assert AI.
- **Confidence** = probability of the *asserted* class:
  - likely_ai → `confidence = p_ai`
  - likely_human → `confidence = 1 - p_ai`
  - uncertain → `confidence = 1 - 2*|p_ai - 0.5|` (peaks low near the boundary).
- A `0.6` combined score means "leans AI but well within the uncertain band —
  not enough to assert." Only scores past `0.65`/below `0.40` produce a
  confident verdict.

## 3. Transparency label variants

| Variant | When | Exact text |
|---|---|---|
| **High-confidence AI** | verdict `likely_ai` & confidence ≥ 0.65 | `"🤖 Likely AI-generated — Our analysis suggests this text was probably created with AI assistance (confidence: {pct}%). This is an automated estimate, not a certainty. The creator can appeal this label."` |
| **High-confidence human** | verdict `likely_human` & confidence ≥ 0.65 | `"✍️ Likely human-written — Our analysis found no strong signs of AI generation in this text (confidence: {pct}%). This is an automated estimate, not a guarantee of authorship."` |
| **Uncertain** | any other case | `"❓ Uncertain origin — We couldn't confidently determine whether this text was written by a human or AI (confidence: {pct}%). We're showing this honestly rather than guessing. No attribution claim is being made."` |

`{pct}` is the confidence rendered as a whole-number percentage.

## 4. Appeals workflow

- **Who:** the creator of a submission (any caller with the `content_id`).
- **Provides:** `content_id` + `creator_reasoning` (free text).
- **System does:** sets the content's `status` to `under_review`; writes an
  appeal entry to the audit log that references the original decision
  (`event="appeal"`, original verdict/confidence preserved, `appeal_reasoning`
  stored); returns confirmation.
- **Reviewer view:** `GET /log` shows the original `classified` entry plus the
  `appeal` entry with `status="under_review"` and the reasoning — a human
  reviewer can read both side by side. No automated re-classification.

## 5. Anticipated edge cases

1. **Formal human writing** (academic abstract, legal/economics prose) — low
   sentence-length variance and clean punctuation make stylometry score it
   AI-like. Mitigated by the asymmetric threshold + LLM weight, but it will
   still skew toward `uncertain`. This is exactly why appeals exist.
2. **Repetitive / minimalist poetry** — heavy repetition and tiny vocabulary
   crush the type-token ratio and look uniform, so stylometry over-flags it as
   AI even though it's a deliberate human style.
3. **Very short submissions** (a tweet-length line) — stylometric statistics are
   unstable on little text; the system widens toward `uncertain`.

---

## AI Tool Plan

- **M3 (endpoint + signal 1):** Provide the Detection-signals section + the
  architecture diagram. Ask for the Flask skeleton with `POST /submit` and the
  Groq classifier function returning `p_ai`. Verify by calling the function
  directly on sample inputs and checking the returned JSON shape before wiring.
- **M4 (signal 2 + scoring):** Provide Detection-signals + Uncertainty sections
  + diagram. Ask for the stylometry function and the combine/threshold scorer.
  Verify the implemented thresholds match this spec (0.65 / 0.40) and that
  clearly-AI vs clearly-human inputs produce clearly different scores.
- **M5 (production layer):** Provide Label-variants + Appeals sections + diagram.
  Ask for the label generator and `POST /appeal`. Verify all three label
  variants are reachable and that an appeal flips status to `under_review` and
  logs alongside the original decision.
