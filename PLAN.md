# Milestone 1 — the plan

Written before any of `/submit`, `/appeal`, `stylometry.py` or `scoring.py`
exists. Everything later has to keep the contract in part 4. The thresholds
in part 3 are a starting position that Milestone 4 calibration can move, and
if it does, the change gets written down.

---

## 1. The path a piece of text takes

From the moment someone sends text to the moment they read a label.

1. **The request arrives at `POST /submit`** (`app.py::submit`).
   Flask parses the body into a dict. *Adds:* structure. Or, if the body isn't
   JSON or isn't an object, it adds nothing, and the route has to notice.

2. **The rate limiter decides whether this caller may ask at all** (unit 8,
   `app.py::rate_limit_key`). It counts per `creator_id`, falling back to the
   caller's address. *Adds:* a yes, or a `429` plus a `rejected` line in the
   audit log, so a flood leaves a trace.

3. **The route validates the input and gives it an identity.** It checks that
   `text` and `creator_id` exist, are strings and are within the length limits
   in part 4, then mints a `content_id` (uuid4). *Adds:* the guarantee that
   everything downstream gets real text, and the id the writer will need if
   they appeal. A refusal is a `400` and is logged as `rejected`.

4. **Signal one reads the text** (`detector.model_signal`). gpt2 computes the
   perplexity of the **first 400 tokens (about 300 words)**, and a logistic
   curve centred on perplexity 45 squashes it onto 0–1.
   *Adds:* `model_score`, where higher means more predictable, which this
   signal treats as more AI-like.

5. **Signal two measures the text's shape** (`stylometry.style_signal`). It
   looks at sentence-length variation, vocabulary variety and punctuation
   variety, each scaled to 0–1 and combined. *Adds:* `style_score`, where
   higher means more even and uniform, which this signal treats as more
   AI-like.

6. **The scoring stage turns two numbers into one and then into words.**
   `scoring.combine_signals` produces the combined score, and
   `scoring.score_to_label` maps it to a `guess` (`ai` / `unsure` / `human`)
   and the sentence a reader sees. *Adds:* the decision, which is where the
   false-positive policy in part 3 actually takes effect.

7. **The decision is written down, then answered** (`audit.log_decision`, then
   the JSON response). All nine log fields are recorded with
   `status: "decided"` before the response goes out. *Adds:* a permanent,
   append-only record that `/appeal` can find later. The writer gets the label,
   both signal scores and their `content_id`.

---

## 2. What the two signals measure

### Signal one: the model signal (`detector.py`, given)

**What it measures.** Predictability: how surprised gpt2 is, word by word, by
the text. That is reported as perplexity and mapped so that low perplexity
gives a high score.

**Why it might differ between human and AI writing.** A language model writes
by repeatedly choosing a likely next word, so its output is text a model finds
easy to predict. People pick odd words, change direction mid-thought and name
specific things, and each of those surprises the model.

**What it can't see.** **Who wrote the text, as distinct from how ordinary the
text is.** Predictability is a property of the words, not of the author.
Several kinds of human writing score as machine-like:

- plain writing in a second language, where the writer sticks to words they
  are sure of
- a younger writer's work
- anything written simply on purpose

Ornate, rare-word prose goes the other way, so AI text asked to "use unusual
vocabulary" reads as human. Two smaller blind spots follow from how it is
built:

- **It reads only the first ~300 words**, because of truncation at 400 tokens.
  Anything after that is invisible. Human writing pasted after an AI opening
  scores as AI, and the reverse also holds.
- **It reads text through gpt2's 2019 view of English.** It has no idea what a
  newer model's "typical" output looks like.

### Signal two: the style signal (`stylometry.py`, mine to build)

**What it measures.** The shape of the text, without reading it. It combines
three measures, each scaled to 0–1 and pointing the same way (higher means
more AI-like):

| Measure | Computed as | Direction |
|---|---|---|
| Sentence-length spread | stdev ÷ mean of words per sentence | low spread → AI-like, so flipped |
| Vocabulary variety | type-token ratio averaged over 50-word windows, so length doesn't drive it | low variety → AI-like, so flipped |
| Punctuation variety | semicolons, dashes, colons, parentheses, `?` and `!` per 100 words (full stops and commas excluded) | low → AI-like, so flipped |

`style_signal` is the mean of the three.

**Why it might differ between human and AI writing.** "Most likely next word,
on average" produces even, mid-length sentences, a safe middle band of
vocabulary and tidy punctuation. Human writing is lumpier: a three-word
sentence next to a forty-word one, an aside in parentheses, a question.

**What it can't see.** **Meaning.** It cannot read. Anything that changes the
shape without changing the substance moves it:

- typos, which add new "words" and raise vocabulary variety
- one long sentence chopped into short ones, or a few one-word sentences
  dropped in, which raises the spread
- a pasted quotation, a list, or a dash added here and there

All of these are cheap to do on purpose. It also has nothing to measure on
very short text: one sentence has no spread at all.

**The blind spot the two share, which matters most.** Plain, even, careful
human writing scores AI-like on **both** signals: common words for signal one,
even sentences and little punctuation for signal two. The signals are least
independent exactly where a false positive would hurt most. Agreement between
them is therefore weaker evidence than it looks, and the combining rule must
not treat two agreeing signals as twice the proof.

---

## 3. Tracing the false positive

**The person.** Amaka is writing in English as a second language. She submits
a 250-word personal essay that she wrote herself. She used short, complete
sentences and words she is confident about, because that's how she avoids
mistakes, and she used almost no punctuation beyond full stops and commas.

**Through the service:**

| Step | What happens to her essay |
|---|---|
| Route, limiter, validation | Passes. 250 words is inside the limits. She gets a `content_id`. |
| Signal one | Common words in predictable order give low perplexity. **Measured: perplexity 13.7, `model_score` 0.8705.** That came from a 100-word plain, first-person paragraph in this style, run through `python detector.py "..."`. The starter's own AI-written sample scores 0.8864, so signal one barely tells them apart. |
| Signal two | Sentences 9–14 words long give low spread. Vocabulary is narrow and there are no semicolons, dashes or questions. **Projected `style_score` ≈ 0.65** (not built yet). |
| Combine (placeholder 50/50 average) | (0.87 + 0.65) ÷ 2 = **0.76**. |
| Label (placeholder thresholds 0.35 / 0.65) | 0.76 > 0.65, so **high-confidence AI**. |

**What the label says to her.** With the starter as shipped, it says
"PLACEHOLDER". With any honest high-confidence AI wording, it says something
like *"We think this was probably written by AI."* To her, that is an
accusation about work she wrote, and the response gives her no reason she
could argue with: only two numbers she has no way to interpret.

**What she can do about it.** She can `POST /appeal` with her `content_id` and
her reasoning. Her item moves to `under_review` and the original decision
stays in the log. That is everything she can do. Nothing re-scores her essay,
no reviewer is notified, and nothing tells her when or whether a person will
look. And she can only appeal at all if she kept the `content_id` and knows
that appealing is possible.

**What this decides.** The placeholder settings accuse her. Three changes
follow, and they are the starting position for Milestones 4 and 5:

1. **The AI label needs both signals to agree strongly, not just a high
   average.** It requires `model_score ≥ 0.70` **and** `style_score ≥ 0.70`.
   If either falls short, the combined score is capped just below the AI
   threshold, so the result is "unsure" at worst. Because of the shared blind
   spot, this is a minimum and not a guarantee. With her measured 0.87 on
   signal one, she is protected **only** if signal two stays below 0.70, so
   signal two's calibration on plain human writing is what actually decides
   her label.
2. **The AI threshold rises from 0.65 to about 0.80.** That widens the unsure
   band on the AI side and costs more AI text getting through. A wrong
   accusation costs her far more than a missed detection costs the site.
3. **Every response that isn't "human" tells the reader they can appeal and
   carries the `content_id` they would need.** The label sentence says it in
   plain words (see part 4).

Milestone 4's calibration has to include a plainly-written human paragraph.
If that paragraph lands in the AI band, these numbers move before anything
else does.

---

## 4. The two endpoints

Conventions for both:
- JSON in and JSON out. Requests need `Content-Type: application/json`.
- Every score is a float from 0 to 1, rounded to 4 decimal places, and
  **higher always means more likely AI.**
- Every error has the same shape: `{"error": "<code>", "message": "<sentence a
  person can read>"}`. The codes are fixed strings; the messages may change.
- Unknown request fields are ignored.
- Every refused request is logged with `audit.log_rejection`.

### `POST /submit`

**Request**

| Field | Type | Required | Rules |
|---|---|---|---|
| `text` | string | yes | After trimming whitespace: at least **25 words** and at most **20,000 characters**. Signal one reads only the first ~300 words. |
| `creator_id` | string | yes | After trimming: 1–100 characters. Not authenticated; it is a name, not a login. |

**Response `200`**

| Field | Type | Meaning |
|---|---|---|
| `content_id` | string | uuid4. Needed for `/appeal`. Unique per submission, even for identical text. |
| `guess` | string | Exactly one of `"ai"`, `"unsure"`, `"human"`. |
| `confidence` | float 0–1 | **The combined score**, the same number written to the log as `combined_score`. Higher means more likely AI. It is *not* "how sure we are of `guess`": a `"human"` guess comes with a *low* confidence. |
| `label` | string | The full sentence a reader sees. Its exact wording is fixed per `guess` and listed in the README's Label Variants table. The `"unsure"` and `"ai"` sentences say how to ask for a review. |
| `model_score` | float 0–1 | Signal one. |
| `style_score` | float 0–1 | Signal two. |
| `status` | string | Always `"decided"` on a fresh submission. |

```json
{
  "content_id": "3f1c9a2e-7b4d-4e8a-9c61-0d2f5b8e1a47",
  "guess": "unsure",
  "confidence": 0.5812,
  "label": "We couldn't tell whether a person or AI wrote this. ...",
  "model_score": 0.7104,
  "style_score": 0.4520,
  "status": "decided"
}
```

**Errors**

| Status | `error` | When |
|---|---|---|
| 400 | `invalid_json` | The body isn't JSON, or is JSON but not an object (a list or a bare string, for example). |
| 400 | `missing_text` | `text` is absent or isn't a string. |
| 400 | `text_too_short` | Fewer than 25 words after trimming, including empty or whitespace-only text. |
| 400 | `text_too_long` | More than 20,000 characters. |
| 400 | `missing_creator_id` | `creator_id` is absent, isn't a string, or is blank. |
| 429 | `rate_limited` | Over the limit (unit 8). Also carries `limit`, e.g. `"10 per 1 minute"`. |
| 503 | `detector_unavailable` | The model couldn't load. Nothing is decided and nothing is logged as a decision. |

The route never returns `500`. Any input that would make a signal raise is
caught by the checks above first.

### `POST /appeal`

**Request**

| Field | Type | Required | Rules |
|---|---|---|---|
| `content_id` | string | yes | Must match a submission already in the audit log (a `decided` entry). |
| `reasoning` | string | yes | After trimming: 1–2,000 characters. The writer's own words. |

There is no `creator_id` field. The appeal is recorded under the `creator_id`
of the original submission, read from the log. Because nothing is
authenticated, anyone who knows a `content_id` can appeal it. That is
acceptable for this project and noted as a known gap.

**Response `200`**

| Field | Type | Meaning |
|---|---|---|
| `content_id` | string | Echoed back. |
| `status` | string | Always `"under_review"`. |
| `message` | string | A plain sentence confirming that a person will look. It promises no outcome and no timeline. |
| `appealed_at` | string | ISO 8601 UTC timestamp of the appeal entry. |

Appealing the same `content_id` again is allowed and returns `200` again.
Every appeal is appended to the log, because a writer adding more reasoning
shouldn't be refused. Nothing is re-scored, and the original decision entry is
never changed.

**Errors**

| Status | `error` | When |
|---|---|---|
| 400 | `invalid_json` | Same as `/submit`. |
| 400 | `missing_content_id` | Absent, not a string, or blank. |
| 400 | `missing_reasoning` | Absent, not a string, or blank. |
| 400 | `reasoning_too_long` | More than 2,000 characters. |
| 404 | `content_not_found` | No `decided` entry has this `content_id`. This is a loud error, not a silent `200`. |

### Could someone build against this without asking me?

Questions I asked of the contract, and where each is answered:

- *Which way do scores point, and is `confidence` the score or the
  certainty?* The conventions say higher means AI everywhere, and the
  `confidence` row says it is the combined score.
- *What happens with empty, short, huge or non-JSON input?* An exact error
  code for each, all `400`, never `500`.
- *Is one sentence enough text?* No. Under 25 words is `text_too_short`.
- *What counts as a valid `content_id` for an appeal?* One with a `decided`
  entry in the log. Otherwise `404`.
- *Can a writer appeal twice?* Yes, with `200` each time.
- *What does a client get if the model isn't downloaded?* `503
  detector_unavailable`.
- **Still open:** the exact label sentences. Their shape is fixed (three of
  them, one per `guess`, with the non-human ones mentioning review). The
  wording is Milestone 5's job, and the README table becomes the source of
  truth once it's written.
