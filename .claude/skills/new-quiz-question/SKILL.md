---
name: new-quiz-question
description: Add a batch of multiple-choice questions to a topic's quiz bank (src/<topic>/quiz.json) for the /quiz page, without generating a new prose question-set file. Use when the user types /new-quiz-question <topic> [subtopic-or-tag].
---

# /new-quiz-question

Appends multiple-choice questions to `src/<topic>/quiz.json`, the data
source for `/quiz/` ([quiz.html](../../../quiz.html)). Use this to grow the
quiz bank directly (e.g. more questions on an existing subtopic, or a batch
tagged for a theme that cuts across multiple prose files). To pair a *new*
prose interview-question file with matching quiz questions, use
`/new-question` instead — it does both automatically.

## Args

`args` is `<topic> [subtopic-or-tag]`, e.g. `java`, `java concurrency`.

- `<topic>` (required): must match an existing `src/<topic>/` folder.
- `[subtopic-or-tag]` (optional): the tag/theme to focus this batch on. If
  omitted, infer a good focus from what's already covered in
  `src/<topic>/quiz.json` and the prose files in `src/<topic>/` — pick
  something under-covered rather than duplicating an existing batch.

## Steps

1. Read `src/README.md`'s "Quiz question bank" section and
   `src/QUIZ_TEMPLATE.json` for the exact schema and rules.
2. Read the existing `src/<topic>/quiz.json` (create it as
   `{"topic": "<topic>", "questions": []}` if missing) to see existing
   `id`s and tags, so new items don't collide or duplicate.
3. Optionally skim `src/<topic>/*.md` for relevant subtopics to draw
   questions from.
4. Draft ~10 new multiple-choice questions on the chosen focus. For each:
   - Use WebSearch/WebFetch to verify the correct answer and explanation
     against official documentation before finalizing it. Do not fabricate
     URLs.
   - Write plausible, non-trivial distractors — wrong choices a candidate
     who half-knows the material could plausibly pick.
   - `explanation` covers why the correct choice(s) are right and why each
     wrong choice is wrong.
   - Tag appropriately (the subtopic/theme plus finer-grained tags like
     `performance`, `internals`, `trade-offs`).
   - Give it a unique `id` (`<topic>-<focus-slug>-NNN`).
5. Append the new items to `src/<topic>/quiz.json` (never remove or
   overwrite existing entries).
6. Report back: file touched, how many questions added, and the tags used.
