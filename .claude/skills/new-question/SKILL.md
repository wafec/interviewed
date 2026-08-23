---
name: new-question
description: Generate a new 20-question interview prep set for a topic (java, api, sql, cloud, ...) following the src/README.md contract. Use when the user types /new-question <topic> [subtopic].
---

# /new-question

Generates a new interview question set file under `src/<topic>/`.

## Args

`args` is `<topic> [subtopic]`, e.g. `java`, `java concurrency`, `sql query-performance`.

- `<topic>` (required): the target subfolder under `src/` (e.g. `java`, `api`, `sql`, `cloud`).
  If the folder doesn't exist yet, create it (this repo may grow new topics over time).
- `[subtopic]` (optional): a short focus area. If not given, infer a good next
  subtopic yourself based on what's already covered (see step 2) — pick one
  that isn't covered yet and is high-value/trending for real interviews. Do
  not stop to ask the user unless the topic itself is ambiguous.

## Steps

1. **Read the contract.** Read `src/README.md` in full — it defines the
   mandatory coverage areas (fundamentals, internals, performance, SE theory,
   real-world problems, pitfalls, advanced features, trade-offs) — and read
   `src/TEMPLATE.md`, the canonical per-question structure (front matter,
   Question / Good answer / optional Code example / Follow-up question /
   Glossary / Mental model / References). Every file you generate must match
   `src/TEMPLATE.md`'s shape exactly.

2. **Survey existing files.** List `src/<topic>/*.md`. Read their titles/H1
   and subtopic slugs (don't need to read every question in full, just enough
   to know what's already been covered) so you:
   - Pick the next sequence number (`NN`, zero-padded to 2 digits, continuing
     from the highest existing number in that folder, starting at `01`).
   - Avoid heavily duplicating a subtopic already covered, unless the user
     explicitly asked for that subtopic again (e.g. going deeper).

3. **Pick the subtopic** (if not given as an arg): choose something specific,
   not a repeat of an existing file, and skew toward what real interviews
   are asking right now — e.g. performance diagnosis methodology, concurrency
   internals, resilience/failure modes, trade-off/comparison questions. Keep
   it narrow enough that 20 questions can go deep rather than staying generic.

4. **Draft the 20 questions** covering every mandatory category from
   `src/README.md` (fundamentals, internals, performance diagnosis, SE theory
   + practice, real-world problems, pitfalls, advanced features, trade-offs).
   Do not skip the performance-diagnosis angle (tools, methodology, how to
   detect + validate a fix) or the internals angle (locks/semaphores/mutexes/
   memory model/execution plans/etc., whichever apply to this topic).

5. **Verify each answer online before writing it down — this is mandatory,
   not optional.** For every question, use WebSearch/WebFetch to find the
   **official documentation** backing the claims in the "Good answer"
   (language spec, vendor docs, RFC, official framework docs). Do this
   *before* finalizing the answer text, not after — if what you find
   contradicts or refines your draft answer, fix the answer to match the
   authoritative source. Do not fabricate or guess at a plausible-looking
   URL: only include a link you actually retrieved and confirmed supports
   the claim. If truly no official source exists for a niche claim, note
   that explicitly rather than linking something unrelated.

6. **Add code examples where they help.** For questions where a short code
   snippet would make the answer clearer (e.g. a locking pattern, a query
   plan, an API request/response shape), include a focused `Code example`
   block. Skip it for purely conceptual/trade-off questions where code adds
   no clarity.

7. **Write the file** at `src/<topic>/NN-<topic>-interview-<subtopic>.md`
   with:
   - Jekyll front matter as the first lines of the file (required for the
     file to render on GitHub Pages and show up in the topic's index page):
     ```yaml
     ---
     layout: default
     title: "<Human-readable title>"
     ---
     ```
   - An H1 title.
   - A one-paragraph intro naming the subtopic and what it covers.
   - Exactly 20 questions, each following the mandatory structure from
     `src/README.md` (`### Q<N>. ...` with Question / Good answer / optional
     Code example / Follow-up question / Glossary / Mental model /
     References subsections).
   - Every question ends with a **References** section containing at least
     one verified official-documentation link.
   - Write real, accurate, senior-level answers — not placeholders. This is
     study material the user will rely on, so answers must be technically
     correct and complete enough to actually learn from.

8. **Report back** with the file path and a one-line summary of the subtopic
   and coverage, so the user can review it.

## Notes

- Never overwrite an existing file. If the computed `NN-<topic>-interview-<subtopic>.md`
  already exists (e.g. same subtopic requested twice), append the next number
  in sequence instead of clobbering it.
- Keep the tone practical and interview-realistic — answers should read like
  what a strong senior candidate would actually say, not textbook prose.
