---
layout: default
title: Contributor Guide
---

# Interview Prep — Source

This `src/` folder is the single source of truth for all interview question sets.
Every generated file, in every topic, must follow the conventions below so the
content stays consistent and comparable over time.

## Structure

```
src/
  README.md          <- this file (the contract every new file must follow)
  java/
    01-java-interview-<subtopic>.md
    02-java-interview-<subtopic>.md
  api/
    01-api-interview-<subtopic>.md
  sql/
    01-sql-interview-<subtopic>.md
  cloud/
    01-cloud-interview-<subtopic>.md
```

- One subfolder per topic (add new topics the same way: `src/<topic>/`).
- Files are numbered sequentially per topic, zero-padded to 2 digits:
  `NN-<topic>-interview-<subtopic>.md`.
- Each file contains **20 questions**.
- `<subtopic>` is a short kebab-case slug describing the focus of that set
  (e.g. `concurrency`, `jvm-memory-model`, `query-performance`, `iam-and-networking`).

## Mandatory coverage per file

A set of 20 questions is not just "20 random questions" — every file must, across
its questions, touch all of these angles for the topic/subtopic it covers:

1. **Fundamentals** — core concepts a working practitioner must know cold.
2. **Internals** — what actually happens under the hood (locks, semaphores,
   mutexes, memory models, execution plans, garbage collection, connection
   pooling, network hops, caching layers, etc.). "Explain it like you've read
   the source / the RFC."
3. **Performance** — this is trending hard in interviews right now, so it is
   never optional. Cover: how do you *detect* a performance problem, which
   *tools/commands* you reach for (profilers, `EXPLAIN ANALYZE`, flame graphs,
   `jstack`/`jcmd`, APM traces, browser devtools, etc.), your *methodology* to
   isolate root cause, and how you validate the fix actually worked.
4. **Software engineering theory mixed with tech-specific practice** — connect
   the concrete technology to the underlying CS/SE principle (e.g. a SQL index
   question should also probe understanding of B-trees / time complexity; a
   Java concurrency question should also probe the general theory of
   race conditions and memory visibility).
5. **Real situations & problems the technology solves** — "why does this
   feature/pattern exist, what pain predates it, when would you reach for it
   in production."
6. **Challenges / pitfalls / anti-patterns** — common mistakes, footguns,
   things that look right but aren't.
7. **Advanced / lesser-known features** — the stuff that separates a senior
   from a mid-level candidate.
8. **Trade-offs & comparisons** — vs. alternatives, when NOT to use it.

Not every question needs to map to exactly one category above — a strong
question often blends several — but a full 20-question file should leave no
category untouched.

## Every file needs Jekyll front matter

GitHub Pages renders this repo through Jekyll from the repo root, and Jekyll
only converts a Markdown file into an HTML page if it starts with front
matter. So every file under `src/` (including this one) must start with:

```yaml
---
layout: default
title: "<Human-readable title for this file>"
---
```

Without this block the file stays a plain, unstyled Markdown file (still
readable on GitHub, but not part of the Pages site navigation).

Each topic folder has an `index.md` that automatically lists every file in
that folder (via a Liquid loop over `site.pages`) — so once a new file has
front matter, it shows up in the site nav with no extra wiring.

The site uses the `jekyll-theme-hacker` theme with a custom `_layouts/default.html`
that adds a breadcrumb (Home » Topic » Page) above every page's content —
this is automatic for any page with front matter, no per-file work needed.

## Mandatory structure per question

The canonical, always-up-to-date template lives at
[`src/TEMPLATE.md`](TEMPLATE.md) (raw file — it's excluded from the Pages
build since it's scaffolding, not content). Copy its shape exactly for every
question. It is reproduced here for reference:

```markdown
### Q<N>. <Question title/text>

**Question:**
<the interview question itself, verbatim as it would be asked>

**Good answer:**
<a strong, complete answer — accurate, concise, structured>

**Code example:** (only when a code example genuinely makes the answer
clearer — not mandatory for every question, e.g. a pure trade-offs/theory
question may not need one)
```<language>
<short, focused snippet illustrating the answer — not a full app>
```

**Follow-up question:**
<a natural follow-up an interviewer would ask to probe deeper>

**Follow-up good answer:**
<a strong, complete answer to the follow-up question — same bar as the main
Good answer, claims backed by References below>

**Glossary:**
- **Term** — definition
- **Term** — definition

**Mental model:**
<what this question is actually testing — the underlying skill, judgment,
or experience the interviewer is trying to surface. This is the "why" behind
the question, not a restatement of the answer.>

**References:**
- [Official doc title](https://...)
```

### References are mandatory

Every question's **Good answer** and **Follow-up good answer** must be backed
by a **References** section
linking to **official documentation** (language spec, vendor docs, RFC,
official framework docs — not blog posts, not Stack Overflow, not
third-party tutorials, unless no official source exists for that exact claim).

This exists for two reasons:
1. **Fact-checking at generation time.** Whoever (or whatever) is writing the
   answer must actually look the claim up online before writing it, rather
   than generating from memory. If the answer can't be backed by an official
   source, it needs to be re-verified before the file is considered done.
2. **Fact-checking at study time.** So the reader (me) can click through and
   verify/deepen understanding straight from the primary source.

At least one reference link per question, more if multiple distinct claims
in the answer need backing.

## Quiz question bank

`/quiz/` ([quiz.html](../quiz.html)) is a self-scoring multiple-choice quiz
that runs entirely client-side (no backend, nothing persisted) against a
per-topic question bank at `src/<topic>/quiz.json`. Schema and an example
live in [`src/QUIZ_TEMPLATE.json`](QUIZ_TEMPLATE.json) (excluded from the
Pages build, same as `TEMPLATE.md`).

**Every time `/new-question` generates a new prose file, it must also
contribute matching entries to `src/<topic>/quiz.json`** — create the file
if it doesn't exist yet (`{"topic": "<topic>", "questions": []}`), otherwise
append to it (never overwrite existing entries; keep `id`s unique). Aim for
roughly 10 MCQ items per prose file, drawn from the same subtopic, each
tagged with:
- the file's subtopic slug (so the file and its quiz stay linked), and
- one or more finer-grained tags (e.g. `performance`, `internals`,
  `trade-offs`) so quizzes can also be composed across files.

Each MCQ's `explanation` should cover why the correct choice(s) are right
*and* why the wrong choices are plausible-but-wrong, and its `reference`
should be a verified official-documentation link (same bar as the prose
References section).

### Linking a question-set page to its quiz

Every generated `src/<topic>/NN-<topic>-interview-<subtopic>.md` file must
end with a link that jumps straight into an auto-started quiz scoped to that
file's subtopic:

```markdown
---

**Test your knowledge:** [Take a quiz on this topic]({{ "/quiz/?topics=<topic>&tags=<subtopic>&autostart=1" | relative_url }})
```

`quiz.html` reads `topics` (comma-separated topic slugs), `tags`
(comma-separated tags, ANY match), `count` (default 10), and `autostart=1`
(skip the setup screen and jump straight into the sampled quiz) from the URL
query string.

## Adding a new file

Use the `/new-question <topic>` skill (e.g. `/new-question java`,
`/new-question sql`). It will:

1. Read this README for the contract above.
2. Look at existing files in `src/<topic>/` to pick the next number and avoid
   repeating subtopics already covered.
3. Ask (or infer, if given) the subtopic to focus on.
4. Generate a new `NN-<topic>-interview-<subtopic>.md` file with 20 questions
   following the mandatory coverage and structure above.

## Topics

- `java/` — Core Java, JVM internals, concurrency, collections, streams, GC.
- `api/` — REST/HTTP API design, versioning, auth, idempotency, rate limiting.
- `sql/` — Query design, indexing, execution plans, transactions, isolation levels.
- `cloud/` — Cloud infra, networking, IAM, scaling, resilience, cost/perf trade-offs.
- `dotnet/` — C#/.NET runtime (CLR), ASP.NET Core, async/await, GC, EF Core, performance diagnosis.
- `react/` — React internals (fiber/reconciliation), hooks, rendering performance, state management, SSR/hydration.
