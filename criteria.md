# Acceptance criteria — Provenance Guard

Five criteria that say what "working" means for this service, written in unit 7
**before** you built it.

**All five are yours.** None are given.

An acceptance criterion names a number or something plainly observable. *"The
detector is accurate"* is an opinion — and perfect AI detection isn't possible
anyway, so it's an opinion about something nobody can deliver. *"Of 10 pieces I
wrote myself, at most 1 is labelled high-confidence AI"* is a criterion.

Under each, write a sentence or two on **why that target**. *"I allow at most 1
false positive in 10 because on a writing site a wrong accusation costs more
than a missed detection"* is a real answer.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## Pick numbers you can defend

**Cover at least three of these five areas.** They're prompts, not a form.

| Area | A question it could answer |
|---|---|
| False positives | How often may a human's writing get called AI? |
| Score spread | How far apart should clearly-AI and clearly-human text score? |
| Label coverage | Is every one of your three labels actually reachable? |
| The appeal path | Does an appeal always change status and always get recorded? |
| Bad input | What happens when `text` is missing, empty, or enormous? |

Two things worth knowing before you pick numbers:

- **The expensive mistake is one-sided.** Missing some AI text costs the site a
  little. Accusing a real writer costs that person a lot. A criterion that
  treats both errors as equally bad is describing a system nobody would ship.
- **Unit 8 tests across three trials, and the target has to hold across all
  three.** Your first signal runs a model, so identical input can score
  differently twice. A target of "at most 1 in 10" against trials of 2, 1, 2 is
  a **miss**.

---
## 1. My own writing is almost never called AI

Of **10 pieces I wrote myself**, at most **1** is labelled high-confidence AI,
in **each** of the three trials. **3** of the 10 are deliberately plain: short
sentences and common words, the style in the false-positive trace in
[PLAN.md](PLAN.md). **None** of those 3 may be the one.

**Why this target:** On a writing site, a wrong accusation costs the writer
far more than a missed detection costs the site, so this is the strictest
target of the five. The plain pieces are named separately because signal one
already scores a plain human paragraph at 0.87, almost the same as AI text at
0.89. If the system fails anyone, it will fail them, and an average over all
10 could hide that.

---

## 2. Clearly-AI and clearly-human text score far apart

Across **5 texts I'm confident are AI-generated** and **5 I'm confident are
human-written**, the mean combined score of the AI set is at least **0.40
higher** than the mean of the human set, in **each** of the three trials.

**Why this target:** My unsure band runs from 0.35 to about 0.80, a width of
0.45. If the clear cases sit less than 0.40 apart, both sets mostly land in
"unsure" and the service says nothing useful even about easy text. Requiring
the gap in every trial also catches a gap that exists only because one model
run happened to be lucky.

---

## 3. All three labels are reachable, every time

Of **9 coverage inputs** (3 clearly AI, 3 clearly human, 3 that I judge
borderline), each of the three labels (high-confidence human, unsure,
high-confidence AI) is produced by **at least 1 input in every trial**.
Separately, at least **7 of the 9** inputs get the **same label in all three
trials**.

**Why this target:** A label no score can reach is a gap in the ranges, and
the stub combining rule already makes that easy to do by accident (see
`config.py`). The stability half matters to a writer: if submitting the same
essay twice can flip it between "unsure" and "AI", the label is noise and not
a judgment.

---

## 4. Every appeal is recorded, and nothing is lost or overwritten

For **5 real submissions** that I appeal: **5 of 5** appeals return `200`
with `status: "under_review"`, and each adds **exactly one** `appeal` entry to
the audit log. Each submission's original `decided` entry stays **unchanged**
(same scores, same label). An appeal against a **made-up `content_id`**
returns `404 content_not_found` and adds **no** entry. All of this must hold
in **each** of the three trials.

**Why this target:** The appeal is the only thing between a wrong accusation
and nothing at all, so "usually works" isn't good enough. The bar is 5 of 5.
The made-up id is there because the starter warns that a typo here fails
silently, and a writer whose appeal silently vanished would never find out.
`run_eval.py` sends only `/submit`, so I test this by hand with `curl` and
paste the log entries.

---

## 5. Bad input is refused clearly and never crashes the service

Of **6 malformed requests**:
- empty `text`
- no `text` field
- `text: null`
- a one-word `text`
- about 50,000 words of `text`
- a blank `creator_id`

**6 of 6** return a `400` with the specific error code from the contract in
[PLAN.md](PLAN.md), **0** return a `500`, and **6 of 6** leave a `rejected`
entry in the audit log. This must hold in **each** of the three trials. After
the run, `POST /ping` still answers.

**Why this target:** A crash on malformed input is one of the most common real
failures, and unit 8's attack set will try exactly these. Zero `500`s is the
only acceptable number, because no threshold makes a crash acceptable. The
log requirement exists because a refused request that leaves no trace looks
exactly like a quiet afternoon.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 8 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath, like this:

         ## 2. Uncertain cases land in the unsure band

         The system handles uncertain cases well.

         **Why this target:** ...

         > **Revised in unit 8:** Text I judged borderline scores between
         > 0.4 and 0.6 in at least 4 of 5 cases.
         >
         > **Why revised:** "well" gave me nothing to check. I couldn't
         > reach a verdict from it at all.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "At most 1 of 10 of my own pieces flagged as AI" → "at most 3
            of 10", because 1 of 10 turned out to be hard.

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
