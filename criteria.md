# Acceptance criteria — The Unofficial Guide

Five criteria that say what "working" means for this system, written in unit 1
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"Retrieval works"* is an opinion. *"For at
least 4 of my 5 test questions, the top results include a chunk containing the
answer"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter or looser one. A reason that says something about your corpus or your
pipeline earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## 1. Retrieved chunks contain the answer

For at least 4 of my 5 test questions, the retrieved chunks include one that
contains the answer.

**Why this target:** My hardest question is "How many exams are there for CS
210?" — the document never states the count "three"; it says "two midterms
and a final," so a correct answer requires combining a phrase into a number
rather than matching one. My other four questions (when study abroad
applications open, Halden Hall's hours, the campus job hour cap, credits
needed to graduate) are each a single literal fact stated in one sentence of
one document, so I expect those four to retrieve cleanly and the exam-count
question to be the one likely miss.

---

## 2. Every answer names a source

Every answer the system produces names at least one source document.

**Why this target:** Every document in `campus_life` is short, self-contained
prose with a clear filename, and `generate.py`'s system prompt explicitly
instructs the model to name the filename it used and refuse rather than
guess when nothing relevant came back. Since the relevance gate only lets a
question through once retrieval has already found something close enough,
there's always at least one real source available to name by the time
generation runs — I'd only expect to miss this if the model ignores the
instruction outright, which is worth catching, not assuming away. That's why
all 5 and not 4.

---

## 3. The relevance gate stops out-of-corpus questions

When I ask a question my documents clearly don't cover, the relevance gate
stops it and the system returns "I don't have enough information about that" —
in at least 4 of 5 tries.

<!-- The five questions are the ones in `OUT_OF_SCOPE` at the bottom of
     `questions.py`, and `run_eval.py` puts them through the gate and writes
     what happened into your run log. Swap them for your own if you'd rather —
     just keep five of them, or the "4 of 5" above has nothing to be 4 of. -->

**Why this target:** My `OUT_OF_SCOPE` questions (the capital of Mongolia,
changing diesel oil, the 1994 World Cup, ibuprofen dosage, a Rust for-loop)
share essentially no vocabulary with a US campus-life corpus about dining
halls, dorms, courses and admin policy, so I expect their embeddings to land
far from anything in my index — a clean gap above the 0.6 default, not a
close call. I haven't measured the actual distances yet — that's Milestone
4 — so I'm setting 4 of 5 rather than 5 of 5 as a hedge against one embedding
turning out closer than I predict, and I'll revise this note once I have the
real numbers.

---

## 4. Every document lands in exactly one chunk

For all 88 documents in `campus_life`, the whole document produces exactly
one chunk — no document gets cut across a chunk boundary.

**Why this target:** I checked this directly: my documents run 178–549
characters, well under my 800-character chunk size, and `chunker.py`'s own
fallback splitter already produces 88 documents → 88 chunks with nothing
split. Each post is a self-contained answer to one question under its
filename (e.g. `thread_parking.txt`), so cutting inside one would produce a
fragment that only makes sense with the piece removed. For this corpus, the
right chunk size isn't a fixed character count — it's "big enough to hold
the longest document" — so the target is 88 of 88, not 4 of 5: anything less
means my chunk size stopped being big enough for at least one post.

---

## 5. Refusal precision — the source named is the one that's actually right

When the system answers a question (doesn't refuse), the source it names is
the specific document that actually contains the fact stated — not merely a
different document from the same topic cluster. For the 3 of my 5 test
questions that fall inside a multi-document cluster (CS 210's exam count, out
of its main/exams/workload trio; Halden Hall's hours, out of its
main/follow-up pair; the campus job hour cap, out of its two related jobs
documents), at least 2 of 3 name the document that actually contains the
specific fact used.

**Why this target:** `campus_life` isn't one document per topic — 8 courses
each have 3 near-duplicate pages (main, exams, workload), and dining halls
and dorms have paired main/follow-up documents that repeat some facts but
not others (Halden Hall's follow-up repeats its 7:00pm closing time but never
mentions its 7:30am opening time). A model can satisfy criterion 2 — "names a
source" — while citing the wrong document in a pair like that, and that
failure is invisible unless I specifically check the cited file against the
fact stated. I'm checking only the 3 of my 5 questions that actually sit in a
cluster, since the other 2 (study abroad, graduation requirements) have no
sibling document to confuse retrieval with. I set 2 of 3 rather than 3 of 3
because the CS 210 exam count is duplicated verbatim in two documents, so
either citation is technically correct — that case may not be a fair test of
the failure mode I'm actually trying to catch.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 2 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 1. Retrieved chunks contain the answer

         For at least 4 of my 5 test questions, the retrieved chunks include
         one that contains the answer.

         **Why this target:** ...

         > **Revised in unit 2:** For at least 4 of 5 questions, the top three
         > results contain the answer.
         >
         > **Why revised:** I couldn't judge "the chunks include one that
         > contains the answer" the same way twice — I scored two questions
         > differently on Monday than on Wednesday. The new version is
         > something I can actually check.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said 4 of 5 but got 2 of 5, so 2 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.

     The whole reason the originals stay visible is so someone can see what you
     said before you knew the answer.
     ───────────────────────────────────────────────────────────────────────── -->
