# App contract: web app and API

The contract between the web app and the API, alongside the [AI contract](ai-contract.md). Its Zod schemas are in `packages/contract` (`src/quiz-views.ts` and `src/attempt.ts`), so the quiz screen and the marking code share one definition. Status and open items: [README](README.md).

## Answer key stays on the server

The answer key is every field that reveals or decides a correct answer: `correctOptionId`, `correctOptionIds`, `correctAnswer`, `acceptedAnswers` (including each label's and blank's), `correctPairs`, `correctOrder`, `rubric`, `modelAnswer`, `explanation`, `answerBasis`, `sourceRefs` and `evaluation`.

- The answer key is written only when the quiz is created or edited. Grading reads it and never changes it. A short answer that AI grading accepts is recorded on that attempt only; it is not added to `acceptedAnswers`.
- While a child is taking a quiz, the API sends an **answer-free view** of the quiz: `quizId`, `quizVersion`, `title`, the contexts (`id`, `title`, `text`, `image`), and per question only `id`, `type`, `prompt`, `contextId`, `image`, `options` (text and image, without which are correct), `leftItems`, `rightItems`, `items` (in display order), each label's `id` and `marker`, each blank's `id`, `wordBank`, `points` and `suggestedWordRange`. The same view is used for browsing global quizzes ([Accounts, profiles and quiz ownership](data-and-accounts.md#accounts-profiles-and-quiz-ownership)). The answer-free view has its own schema in the contract package, so a field that isn't listed there can't reach the browser by accident.
- All marking happens on the server.

## Answer shapes

Each answer carries its question's `type`, so the server checks it against the right schema (a Zod discriminated union) and rejects an answer of the wrong type:

| Type | `answer` | Rules |
|---|---|---|
| `single_choice` | `{type: "single_choice", optionId}` | One of the question's options |
| `multi_select` | `{type: "multi_select", optionIds: string[]}` | 1 or more, no repeats, all existing options. Nothing ticked is sent as `null` |
| `true_false` | `{type: "true_false", value: boolean}` | |
| `short_answer` | `{type: "short_answer", text}` | Trimmed; within the short-text limit (below) |
| `matching` | `{type: "matching", pairs: {leftId, rightId}[]}` | At most one pair per left item; a right item may be used more than once ([Question fields per type](ai-contract.md#question-fields-per-type)). Left items without a pair count as wrong |
| `ordering` | `{type: "ordering", order: itemId[]}` | Every item exactly once |
| `diagram_labelling` | `{type: "diagram_labelling", labels: {id, text}[]}` | At most one entry per label; missing labels count as wrong. With a `wordBank`, each `text` is one of its words. Each `text` is trimmed and within the short-text limit (below) |
| `fill_in_blanks` | `{type: "fill_in_blanks", blanks: {id, text}[]}` | Same rules as labels |
| `descriptive`, typed | `{type: "descriptive", kind: "text", text}` | Within the limit for the question's `length` (below). The `suggestedWordRange` is guidance only |
| `descriptive`, uploaded | `{type: "descriptive", kind: "upload", uploadIds: string[]}` | Only when uploads are allowed for this attempt ([Answering descriptive questions: type or upload](product-rules.md#answering-descriptive-questions-type-or-upload)). Files are uploaded first and get ids; limits in [Upload limits](architecture.md#upload-limits) |
| Not answered | `null` | |

- **Ordering always has an answer:** whatever order is on screen at submit is the child's answer, even if they never moved anything, and it earns a share for each item that happens to be in its correct position ([Evaluation setting and scoring](product-rules.md#evaluation-setting-and-scoring)).
- **Words picked from a word bank are sent as text,** so labels and blanks are marked the same way whether the child typed or picked.

## Text limits

Each typed answer has a maximum length, so nobody can paste thousands of words into a question that needs one word or one sentence. The limits are well above the suggested word ranges ([Question fields per type](ai-contract.md#question-fields-per-type)), so a genuine answer never comes close:

| Text | Limit | Roughly |
|---|---|---|
| Short text: a `short_answer`, each diagram label, each blank | 200 characters | 30 words |
| `descriptive`, length `short` | 2,000 characters | 300 words |
| `descriptive`, length `long` | 5,000 characters | 800 words |
| `descriptive`, length `essay` | 30,000 characters | 5,000 words |

- The server enforces the limits; an answer over its limit rejects the submission.
- The quiz screen shows a counter (e.g. "150 / 300 words") and stops typing at the limit, so the child sees the boundary instead of an error.
- The limit is the one for the question's `length` when the attempt is answered. A parent changing a question's length is an ordinary edit, so earlier attempts are unaffected.
- Uploaded answers are limited by files, size and pages instead ([Upload limits](architecture.md#upload-limits)).
- The limits are in the server config, like the upload limits, so they can be changed without a code change.

## Submission

```json
{
  "attemptId": "a_81f2",
  "quizVersion": 3,
  "answers": [
    { "questionId": "q1", "answer": { "type": "single_choice", "optionId": "b" } },
    { "questionId": "q2", "answer": null }
  ]
}
```

- Exactly one entry for every question in the quiz, with `null` for skipped ones, so nothing is silently missing.
- Every option, item, label, blank and upload id must exist in that question or attempt; otherwise the submission is rejected.

## What the child sees after submitting

What a child sees is controlled by a `resultVisibility` setting with three levels:

| Level | The child sees |
|---|---|
| `score_only` | `score` and `maxScore` |
| `score_and_results` | The above, plus per question: `number` (1, 2, 3, …) and `result` |
| `score_results_and_answers` | The above, plus per question: `prompt` (question text), the child's own answer, and the correct answer |

The setting exists at two levels:

- **Account** — in the account's settings, one value for the family. Default for a new account: `score_and_results`.
- **Quiz** — `resultVisibility` on the quiz, either one of the three levels or `null` to use the account value. A quiz-level value overrides the account one.

The parent always sees everything about an attempt; `resultVisibility` only controls what the child sees.

`result` per question:

- `correct` \| `incorrect` for single choice, true/false and short answer.
- `correct` \| `partially_correct` \| `incorrect` for multi-select, matching, ordering, diagram labelling and fill in the blanks.
- Descriptive answers: points awarded out of the question's points, e.g. `3 / 5`.
- While a grade is pending: `awaiting_grading`.

"The correct answer" per question type:

| Type | Correct answer shown |
|---|---|
| `single_choice`, `multi_select` | The correct option text(s) |
| `true_false` | True or False |
| `short_answer` | The first entry of `acceptedAnswers` |
| `matching` | The correct pairs |
| `ordering` | The correct order |
| `diagram_labelling` | Each marker with the first entry of its `acceptedAnswers` |
| `fill_in_blanks` | Each blank with the first entry of its `acceptedAnswers` |
| `descriptive` | None; the child sees their points. `modelAnswer` and `rubric` stay parent-only |

Rules:

- The **result view** has its own schema in the contract package, with the fields of each level whitelisted, so nothing outside the chosen level reaches the browser. Explanations and AI feedback (`reasoning`, `overallFeedback`) are never in it; they are for the parent.
- The setting is applied when the result is viewed, using the current effective setting. Changing it changes what children see for past attempts too.
- Changing `resultVisibility` is not a change to the quiz's questions, so it does not increment `quizVersion`.
- While any grade is pending, `score` is not shown; the view says the result is awaiting grading.
- If the quiz has changed since the attempt, only `score`, `maxScore` and `quizVersion` are shown, with "Quiz was changed after this attempt." ([Attempts](data-and-accounts.md#attempts)), whatever the setting.
