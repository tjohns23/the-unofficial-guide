# The Unofficial Guide

Terell — `campus_life` corpus.

> **This file is your submission.** Fill it in as you go — most sections get
> written during the milestone that produces them, not at the end.
>
> How the starter works, and every command you'll need, is in `RUNNING.md`.
> Leave that file alone.
>
> **Paste everything as text.** No screenshots, no video. A typed table gets
> full credit; a picture of the same table gets none.
>
> Delete these instruction blocks as you replace them. The `<!-- -->` comments
> are notes to you and don't show up when the page renders — you can leave them
> or remove them.

---

# Unit 1

## What This Does

This is a retrieval-augmented question-answering system for `campus_life`, a
corpus of 88 short, student-written posts about life at a university —
dining halls, dorms, course workload and exams, and the administrative rules
nobody explains properly (add/drop deadlines, study abroad, graduation
requirements, campus jobs). Ask it a specific, factual question about any of
that — "how many exams does CS 210 have?", "when do study abroad applications
open?", "how many hours a week can I work a campus job?" — and it retrieves
the post(s) that actually answer it, checks whether anything relevant enough
came back at all, and has a model write a short answer that names the
specific file it came from. Ask it something outside that world — car
repair, world capitals, a programming language — and it says so instead of
guessing.

## Chunking Strategy

**Chunk size:** No fixed size — one whole document is one chunk.
**Overlap:** None (there's never a second piece of the same document to overlap with).

I started by reading what the starter's own fixed-size chunker did to `campus_life`:
`python app.py index` reported 88 documents → 88 chunks, because every document
(178–549 characters) is already well under the 800-character window, so nothing
ever got split. That's not a coincidence worth ignoring — every post in this corpus
is a short, self-contained answer to one question, named for that question in its
own filename (`thread_parking.txt`, `admin_add_drop_deadline.txt`). There's no
paragraph inside one of these worth pulling apart from the rest.

I considered one alternative: grouping documents by the first word of their
filename (`course_*`, `admin_*`, `dining_*`, `housing_*`) into bigger topic
chunks. I checked what that would actually produce before writing any code, and
it's worse, not just riskier — `course_*` alone would combine 27 documents
(all 9 courses' main/exam/workload pages) into a single 7,731-character chunk,
mixing CS 210 with Econ 101 with Physics 130. `admin_*` does the same to 16
unrelated policy topics. That's the "chunk covers four topics at once and matches
every question a little" failure, except with a dozen topics instead of four —
splitting those combined blobs back down to size would just reintroduce the
mid-sentence cutting problem I was trying to avoid in the first place.

So instead of relying on the accident that 800 > 549, I replaced
`split_documents` with `chunker.py::document_split`, which doesn't window by
character count at all — it emits exactly one `Chunk` per `Document`, with no
overlap parameter because there's nothing adjacent within a document to stitch
back together. Chunk boundaries are document boundaries, on purpose, because for
this corpus a document *is* the right unit of retrieval.

## Sample Chunks

<!-- Five chunks, pasted as text. Label each one and name the file it came from
     AND the function that produced it — the grader checks your code against
     what you claim here.

     `python app.py chunks -n 5` prints all three for you. Copy them straight
     across.

     Milestone 3. -->
     

**Chunk 1** — source: `admin_add_drop_deadline.txt#0` — produced by: `chunker.py::document_split`

```
On the add/drop deadline

You can add a course through the end of the second week. Dropping is a longer window — through the end of week six — but a drop after week two shows as a W on your transcript. Nothing anywhere on the registrar's site says this plainly, and students find out from each other.
```

**Chunk 2** — source: `course_biol_160.txt#0` — produced by: `chunker.py::document_split`

```
BIOL 160 Cell Biology

I lived here my sophomore year. Format is lecture three times a week with a weekly lab. Assessment: four unit tests and a cumulative final. Not curved.

Expect 9 to 11 hours a week, the heaviest first-year course by reputation.

The one piece of advice: the unit tests come fast, roughly every three weeks; falling behind once is very hard to recover from.
```

**Chunk 3** — source: `course_hist_118_workload.txt#0` — produced by: `chunker.py::document_split`

```
Workload for HIST 118 Modern World History

People keep asking so: a lot of reading, about 120 pages a week, but no problem sets. That's real time, not optimistic time.

It's front-loaded — the first month is heavier than the rest, partly because you're learning the format.
```

**Chunk 4** — source: `dining_pellew_dining_hall_followup.txt#0` — produced by: `chunker.py::document_split`

```
Re: Pellew Dining Hall

Adding to what people have said about Pellew Dining Hall. The wait figure of 12 to 18 minutes at peak matches what I've seen. If you're trying to eat between classes, go before 11:45 and it's a different building entirely.

Also worth saying: the furthest hall from anywhere, next to the athletics centre. Nobody tells you this at orientation.
```

**Chunk 5** — source: `housing_innisfree_hall.txt#0` — produced by: `chunker.py::document_split`

```
Innisfree Hall — what it's actually like

Transferred in last year, so take this with a grain of salt. Built 1991, renovated 2022. Rooms are doubles arranged as pairs sharing one bathroom between two rooms.

The good: the shared-bathroom-between-two-rooms arrangement is the best compromise on campus.

The bad: no air conditioning, which matters for the first three weeks of September.

Laundry costs $1.75 wash, $1.75 dry, app-based. On noise: moderate; the building is L-shaped and the short wing is much quieter.
```

## Sample Answer

<!-- One complete question and answer, pasted as text, with the source line
     visible. Milestone 4. -->

**Question:** How many exams are there for cs210?

**Answer:**

```
There are three exams for CS 210: two midterms and a final.

Sources: `course_cs_210_exams.txt` and `course_cs_210.txt`
```

This one's worth noting: `top_k=5` also pulled back `course_cs_340_exams.txt` (Databases —
a different course with "one midterm and a final") at distance 0.415, closer than CS 210's
own main page at 0.485 — the two courses' exam pages share almost identical boilerplate
phrasing ("assessment... midterm(s) and a final"), which confuses the embedding even though
it never confused the model. The answer above still named only the correct two CS 210
sources and never touched CS 340's numbers. See criterion 5 in `criteria.md` — this is
exactly the risk that criterion is watching for.

**My relevance cutoff:** 0.6 (kept the starter default — see reasoning below)

<!-- The number you set in config.py, and how you got there.

     You ran five questions your corpus covers and the five in OUT_OF_SCOPE
     that it clearly doesn't, and wrote down the best distance for each. What
     did those two groups look like? Where was the gap? Put the actual numbers
     here — the table below wants all ten rows.

     Milestone 4. -->

| Question | In corpus? | Best distance |
|---|---|---|
| When do study abroad applications open? | Yes | 0.234 |
| How many credits do I need to graduate? | Yes | 0.303 |
| How many exams are there for cs210? | Yes | 0.338 |
| When is Halden Hall open? | Yes | 0.357 |
| How many hours a week can I work a job while in school? | Yes | 0.498 |
| What is the capital of Mongolia? | No | 0.825 |
| Who won the 1994 World Cup? | No | 0.886 |
| What is the recommended dosage of ibuprofen for a headache? | No | 0.844 |
| How do I change the oil in a diesel engine? | No | 0.934 |
| How do I write a for loop in Rust? | No | 0.896 |

The two groups don't overlap at all: every in-corpus question landed between 0.234 and
0.498, every out-of-corpus question landed between 0.825 and 0.934 — a clean 0.327-wide
gap with nothing in it. The starter's default of 0.6 sits comfortably inside that gap
(0.102 above my worst in-corpus case, 0.225 below my best out-of-corpus case), so I kept
it rather than moving it. The one question that came closest to the line, job hours per
week (0.498), was pulled toward the cutoff by several `course_*_workload.txt` chunks that
share "X hours a week" phrasing with the jobs question — a real near-miss, but still well
clear of the gate.

## How I Used AI

<!-- Two specific moments. For each: what you asked for, what came back, and
     what you changed about it.

     "I asked Claude to write the chunking function from my notes. It ignored
     the overlap, so I added that myself" is the level of detail we're after.
     "I used AI to help me code" is not.

     Milestone 5. -->

**1.** I asked Claude to explain what already existed in the starter before I
touched anything, since it's a lot of files for a first read. It walked
through the five pipeline stages (ingest → chunker → store → gate →
generate), what each file was responsible for, and what each of the four
corpora looked like. It also caught something I hadn't noticed myself:
`.env.example` was tracked in git but missing from my actual working tree,
which would have broken `RUNNING.md`'s own setup instructions (`copy
.env.example .env`) for anyone else cloning the repo. I didn't change
anything about its explanation — it just meant I started Milestone 1 knowing
what each file did instead of guessing from filenames.

**2.** After I wrote my own `document_split` function and asked Claude to
check it, it didn't just read the code — it actually ran `python chunker.py`
and got a `NameError` immediately, because I'd never initialized the
`chunks` list before appending to it. Running it also surfaced a second bug
I'd have missed by eye: my `index` counter incremented once per document
across the *whole corpus* instead of resetting to 0 for each document's own
chunk, which would have made every chunk's `source#index` label wrong even
though the function wouldn't have crashed. After I applied the fix it fed,
it re-ran the chunker and compared the printed summary (88 chunks, shortest
178, longest 549) against the raw document length stats to confirm every
document turned into exactly one chunk with nothing dropped or cut — that
diagnostic is what actually told me chunking was working, not just that it
ran without an error.

<!-- ── Stretch features ─────────────────────────────────────────────────────
     Doing one? Say so here BEFORE you start. A feature this README never
     claims earns nothing.
     ───────────────────────────────────────────────────────────────────────── -->

---

# Unit 2

<!-- These sections get ADDED to what's already above. Don't delete or rewrite
     unit 1 — the point is that someone can see what you said before you knew
     how it went. -->

## Run Log — Before

<!-- Your five criteria, three runs each. `python run_eval.py --label before`
     runs the questions, puts the OUT_OF_SCOPE ones through the gate, and
     writes it all into results/ for you. Targets come from criteria.md; the
     verdict column is your call.

     Criterion 3 is measured in one deterministic pass rather than three, so
     the same number goes in all three run columns. That's correct, not lazy.

     Milestone 1. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 |  |  |  |  |
| 2. Every answer names a source | 5 of 5 |  |  |  |  |
| 3. Gate stops out-of-corpus questions | 4 of 5 |  |  |  |  |
| 4. | | | | | |
| 5. | | | | | |

<!-- Underneath, paste the REAL output for each criterion from one of your
     runs — the actual text your system produced, not a description of it.
     Name the file and function that produced it. -->

## Verdicts

<!-- MET or MISSED for each of the five, against the target you wrote last
     unit — not a new one. Plus a sentence on how you decided. That sentence
     matters most where it was close.

     If your target said 4 of 5 and your runs came out 4, 3, 4, that's a MISS.
     The target has to hold, not show up occasionally.

     Milestone 2. -->

| # | Criterion | Verdict | How I decided |
|---|---|---|---|
| 1 |  |  |  |
| 2 |  |  |  |
| 3 |  |  |  |
| 4 |  |  |  |
| 5 |  |  |  |

## Diagnoses

<!-- For each miss: which stage caused it, and how. The stage alone isn't
     enough — you need the mechanism.

     Not a diagnosis: "Question 3 didn't work."
     A diagnosis:     "Question 3 asks about laundry costs. The answer is in
                       one sentence that got split across two chunks, so
                       neither chunk on its own contains it."

     The five stages: loading → chunking → embedding → retrieval → generation.

     Look for a pattern. If three misses all ask about numbers, that's one
     problem, not three.

     Missed nothing? Say so, then say honestly whether your targets were set
     low, and which one you'd tighten and to what.

     Milestone 3. -->

## The Improvement

**What I changed:**

**Why I picked it:**

<!-- Connect it to a specific diagnosis above in one sentence. If you can't,
     you picked a fix because it sounded impressive. -->

### Run Log — After

<!-- Same format, same five criteria, three runs each.
     `python run_eval.py --label after` -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 |  |  |  |  |
| 2. Every answer names a source | 5 of 5 |  |  |  |  |
| 3. Gate stops out-of-corpus questions | 4 of 5 |  |  |  |  |
| 4. | | | | | |
| 5. | | | | | |

**Did it help?**

<!-- Say plainly whether it did, and how you know. If it made things worse,
     say that — a change that backfired, honestly reported, earns full credit
     and is more interesting than one that worked. What matters is that you can
     tell.

     Milestone 4. -->

## What's Still Broken

<!-- For each criterion still missed after your fix: what you'd do about it,
     and why you stopped where you did.

     "I ran out of time" is fine if it's true. Pretending nothing is left is
     not.

     Milestone 5. -->

## What I'd Do Differently

<!-- Knowing what you know now — which of your five criteria would you write
     differently, and why?

     Milestone 5. -->
