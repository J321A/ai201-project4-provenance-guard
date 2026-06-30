# Provenance Guard

A backend service any creative-sharing platform can plug into to classify
submitted text as **human-** or **AI-written**, score confidence honestly,
surface a plain-language transparency label, and let creators **appeal** a
verdict. The goal is not to police creativity — it's to protect attribution and
give readers honest context, while acknowledging that perfect AI detection is an
unsolved problem.

> Full design rationale and the architecture diagram live in
> [planning.md](planning.md).

---

## Quick start

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

echo "GROQ_API_KEY=your_key_here" > .env   # already gitignored

python app.py            # serves on http://localhost:5050
```

> macOS note: port 5000 is often taken by AirPlay Receiver. If so, run
> `python -c "import app; app.app.run(port=5050)"` and use `:5050` below.

If `GROQ_API_KEY` is missing or the API errors, the system **degrades
gracefully** to the stylometry signal alone and flags `"degraded": true` in the
response rather than failing.

---

## API

| Method | Route | Body | Returns |
|---|---|---|---|
| `POST` | `/submit` | `{ "text": "...", "creator_id": "..." }` | `content_id`, `attribution`, `confidence`, `label`, per-signal `signals` |
| `POST` | `/appeal` | `{ "content_id": "...", "creator_reasoning": "..." }` | confirmation + `status: under_review` |
| `GET`  | `/log` | `?limit=N` | recent structured audit entries |

```bash
curl -s -X POST http://localhost:5050/submit \
  -H "Content-Type: application/json" \
  -d '{"text": "The sun dipped below the horizon...", "creator_id": "test-user-1"}' \
  | python -m json.tool
```

---

## Detection pipeline (two independent signals)

The system uses **two genuinely distinct signals** — one semantic, one
structural — so the combination is more informative than either alone.

### Signal 1 — Groq LLM semantic classifier (`llama-3.3-70b-versatile`)
- **Captures:** holistic semantic + stylistic coherence — does the text carry a
  human voice (specific lived detail, irregular emphasis) or read as
  machine-smooth (hedged, evenly balanced, generically fluent)?
- **Why it works:** AI prose trends toward even, "on-the-one-hand" structure;
  human writing carries idiosyncrasy a single statistic can't see.
- **Output:** `p_ai ∈ [0,1]`, parsed from a strict JSON response (`temperature=0`).
- **Blind spot:** fooled by heavily-edited AI or very polished human prose;
  non-deterministic; depends on an external API.

### Signal 2 — Stylometric heuristics (pure Python, no external libs)
Blends three measurable structural cues into one `p_ai`:
- **Sentence-length burstiness** (coefficient of variation) — humans vary a lot,
  AI is uniform.
- **Type-token ratio** — lexical diversity; very low diversity reads uniform.
- **Casual markers per 100 words** — contractions, dashes, ellipses, lowercase
  "i"; abundant in casual human writing, scrubbed from formal AI output.
- **Blind spot:** unstable on short text; **formal human writing** (academic,
  legal) is uniform and clean, so it skews AI-like — the canonical false
  positive this project warns about.

**Combination:** `p_ai = 0.60 · llm + 0.40 · stylometry`. The LLM is weighted
higher (it sees meaning); stylometry is the independent structural check.

---

## Confidence scoring & uncertainty

We decided **what a score should mean to a user first**, then implemented toward
it. `confidence` is always reported as the **probability of the leaning class**
(`max(p_ai, 1−p_ai)`), so an uncertain result honestly shows a *low* number
(~50–65%) instead of a falsely reassuring one.

**Asymmetric thresholds** (the heart of the design). Because labeling a human's
work as AI is the costlier error on a writing platform, the human band is wider
than the AI band — we require *stronger* evidence to assert "AI":

| Combined `p_ai` | Verdict |
|---|---|
| `≥ 0.65` | `likely_ai` |
| `0.40 – 0.65` | `uncertain` |
| `≤ 0.40` | `likely_human` |

A confident (non-uncertain) **label** additionally requires `confidence ≥ 0.65`,
so a `0.51` result and a `0.95` result produce meaningfully different labels —
not a binary flip at 0.5.

### How we tested that scores are meaningful
We ran four deliberately chosen inputs (the project's calibration set) and
confirmed the scores track intuition and span the full range:

| Input | `llm` | `style` | combined `p_ai` | Verdict | Confidence |
|---|---|---|---|---|---|
| Clearly AI (paradigm-shift essay) | 0.80 | 0.45 | **0.66** | likely_ai | 0.66 |
| Clearly human (ramen rant) | 0.20 | 0.00 | **0.12** | likely_human | 0.88 |
| Borderline formal human (monetary policy) | 0.80 | 0.56 | **0.70** | likely_ai* | 0.70 |
| Borderline edited-AI (remote work) | 0.70 | 0.45 | **0.60** | uncertain | 0.60 |

The clear-AI and clear-human cases sit at opposite ends; the two borderline
cases land near the boundary, exactly where they should. (*The formal-human case
landing as `likely_ai` is a known false positive — see Limitations — and is
precisely why the appeal path exists.)

**Two example submissions with noticeably different confidence:**
- *High confidence:* the ramen rant → `likely_human`, **confidence 0.88**.
- *Lower confidence:* the remote-work paragraph → `uncertain`, **confidence 0.60**.

---

## Transparency label — three variants

The label shown to a reader changes with the verdict **and** the confidence.
`{pct}` is confidence as a whole-number percent.

| Variant | Shown when | Exact text |
|---|---|---|
| **High-confidence AI** | `likely_ai` & confidence ≥ 65% | `🤖 Likely AI-generated — Our analysis suggests this text was probably created with AI assistance (confidence: {pct}%). This is an automated estimate, not a certainty. The creator can appeal this label.` |
| **High-confidence human** | `likely_human` & confidence ≥ 65% | `✍️ Likely human-written — Our analysis found no strong signs of AI generation in this text (confidence: {pct}%). This is an automated estimate, not a guarantee of authorship.` |
| **Uncertain** | any other case | `❓ Uncertain origin — We couldn't confidently determine whether this text was written by a human or AI (confidence: {pct}%). We're showing this honestly rather than guessing. No attribution claim is being made.` |

Design choices: every label hedges ("estimate, not a certainty"), the AI label
explicitly advertises the appeal path, and the uncertain label refuses to make a
claim at all — honoring the false-positive asymmetry in the UX, not just the
math.

---

## Appeals workflow

Any creator who disagrees calls `POST /appeal` with the `content_id` (returned at
submission) and free-text `creator_reasoning`. The system:
1. flips the content's `status` from `classified` → `under_review`,
2. appends an `appeal` entry to the audit log that **preserves the original
   decision** (verdict, confidence, both signal scores) alongside the reasoning,
3. returns a confirmation including the original decision.

No automated re-classification — a human reviewer reads the original
`classification` entry and the `appeal` entry side by side via `GET /log`.

```bash
curl -s -X POST http://localhost:5050/appeal \
  -H "Content-Type: application/json" \
  -d '{"content_id": "PASTE-ID", "creator_reasoning": "I wrote this myself; I am a non-native speaker so my style reads formal."}'
```
```json
{
  "content_id": "7c1ada15-...",
  "status": "under_review",
  "message": "Appeal received. This content is now under human review.",
  "original_decision": { "attribution": "likely_ai", "confidence": 0.76 }
}
```

---

## Rate limiting

Implemented with Flask-Limiter (in-memory store) on `/submit`:

```python
@limiter.limit("10 per minute;100 per day")
```

**Chosen limits & reasoning.** A real creator submits their own work
occasionally — a handful of drafts in a sitting, rarely more than a few dozen a
day. **10/minute** comfortably covers a human iterating on revisions while
stopping a script from hammering the (latency-bound) Groq signal. **100/day**
caps sustained abuse — a flood attacker, or someone probing thousands of
variants to reverse-engineer the classifier — without ever constraining a
genuine writer. The two limits are layered: per-minute blunts bursts, per-day
blunts slow grinding.

**Verified** with 12 rapid requests (limit 10/min):
```
200
200
200
200
200
429   <- limit reached
429
429
429
429
429
429
```

---

## Audit log

Every classification and every appeal is written to a structured **SQLite** log
(`provenance.db`), surfaced as JSON via `GET /log`. Each entry records:
timestamp, content id, creator id, event type, attribution, combined confidence,
combined `p_ai`, **both individual signal scores**, signals used, status, and
appeal reasoning (when applicable).

Live sample — three classifications plus one appeal (note `id 4` is the appeal
of `id 2`: status flipped to `under_review`, reasoning captured, original
verdict preserved):

```json
{
  "entries": [
    {
      "id": 4, "event": "appeal", "content_id": "e385532d-...",
      "creator_id": "sam-writes", "timestamp": "2026-06-30T02:12:27.847Z",
      "attribution": "likely_ai", "confidence": 0.693, "p_ai": 0.693,
      "llm_score": 0.8, "style_score": 0.533, "status": "under_review",
      "appeal_reasoning": "This is my own essay drafted over two weeks; I just write in a formal register."
    },
    {
      "id": 3, "event": "classification", "content_id": "2ff79e85-...",
      "creator_id": "alex-essays", "attribution": "uncertain",
      "confidence": 0.617, "p_ai": 0.617, "llm_score": 0.7, "style_score": 0.492,
      "signals_used": ["groq_llm", "stylometry"], "status": "classified"
    },
    {
      "id": 2, "event": "classification", "content_id": "e385532d-...",
      "creator_id": "sam-writes", "attribution": "likely_ai",
      "confidence": 0.693, "p_ai": 0.693, "llm_score": 0.8, "style_score": 0.533,
      "signals_used": ["groq_llm", "stylometry"], "status": "classified"
    },
    {
      "id": 1, "event": "classification", "content_id": "567cd4d5-...",
      "creator_id": "maya-blogs", "attribution": "likely_human",
      "confidence": 0.88, "p_ai": 0.12, "llm_score": 0.2, "style_score": 0.0,
      "signals_used": ["groq_llm", "stylometry"], "status": "classified"
    }
  ]
}
```

---

## Known limitations

- **Formal human writing is the system's worst case.** Academic, legal, or
  economics prose has low sentence-length variance and clean punctuation, so the
  stylometry signal scores it AI-like — and a confident LLM can compound it. In
  testing, a genuine human paragraph on monetary policy landed at `likely_ai`
  (p_ai 0.70). This is a direct consequence of *what the stylometry signal
  measures* (uniformity), not a data-volume problem, and it's the exact reason
  the appeal path is a first-class feature.
- **Minimalist / repetitive poetry** crushes the type-token ratio and looks
  uniform, so it over-flags as AI even when the repetition is a deliberate human
  craft choice.
- **Very short submissions** don't give stylometry stable statistics; the signal
  falls back to a neutral 0.5 and the result widens toward `uncertain`.

If deploying for real, we'd calibrate thresholds against a labeled corpus, add a
length-aware confidence penalty, and route low-confidence AI verdicts to human
review *before* showing any label.

---

## Spec reflection

- **Where the spec helped:** writing the three label variants *verbatim in
  planning.md before coding* forced the asymmetry decision early — the uncertain
  label "makes no claim" because the spec made us name what 0.5 means to a user
  first. The label function became a near-mechanical translation of that table.
- **Where the implementation diverged:** planning.md originally defined the
  uncertain confidence as `1 − 2·|p_ai − 0.5|`, which produced a *high* number
  near the boundary (e.g. 0.80) — misleading on an "uncertain" label. During
  testing we changed confidence to uniformly mean "probability of the leaning
  class" (`max(p_ai, 1−p_ai)`), so uncertain cases now honestly report ~60%.
  The behavior is better than the spec; the spec's *goal* (honest uncertainty)
  drove the change.

## AI usage

- **Flask skeleton + Groq signal (M3):** Directed an AI tool, with the
  detection-signals section and diagram, to scaffold the `/submit` route and the
  LLM classifier. It produced a working skeleton; we overrode the LLM prompt to
  force strict JSON (`response_format`) and added the graceful-degradation path
  (return `None` → fall back to stylometry) which the draft lacked.
- **Confidence scoring (M4):** Asked for the combine + threshold logic against
  the uncertainty section. The draft used a symmetric 0.5 split; we corrected it
  to the asymmetric `0.65 / 0.40` bands and replaced its boundary-peaking
  confidence formula with `max(p_ai, 1−p_ai)` after testing showed the original
  reported misleadingly high confidence on uncertain cases.
