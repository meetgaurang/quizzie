# Quizzie AI contract (v1)

Status, open items and the other design docs: [README](README.md).

## What the contract is

The contract is the agreed shape of the data that moves between the AI model and the web app. There are three AI contracts:

| Contract | Direction | Used when |
|---|---|---|
| **A. Quiz extraction** | Uploaded files in → draft quiz out | A parent, or the app owner for a global quiz, uploads PDFs and images |
| **B. Answer grading** | Question + child's answer in → grade out | A child submits a descriptive answer, or a short answer, label or blank that doesn't match the accepted answers |
| **C. Answer filling** | Questions without answers (+ answer key files) in → answers out | After extraction, the parent uploads an answer key or asks the AI to work out missing answers |

All three contracts are defined once, as Zod schemas in a shared package, and every side uses that one definition:

```
packages/contract
  src/quiz-extraction.ts     Zod schemas + TypeScript types for contract A
  src/grading.ts             Zod schemas + TypeScript types for contract B
  src/answer-filling.ts      Zod schemas + TypeScript types for contract C
  schemas/quiz-extraction.v1.json   JSON Schema generated from the Zod schema, committed
  schemas/grading.v1.json
  schemas/answer-filling.v1.json
  src/quiz-views.ts          answer-free view and result view ([Answer key stays on the server](app-contract.md#answer-key-stays-on-the-server), [What the child sees after submitting](app-contract.md#what-the-child-sees-after-submitting)): API → web app
  src/attempt.ts             answer shapes and submission ([Answer shapes](app-contract.md#answer-shapes)): web app → API
```

The last two are not AI contracts. They are the contract between the web app and the API, kept in the same package so the quiz screen and the marking code share one definition.

- **The API** sends the generated JSON Schema to Claude as the required output format (structured outputs), so the model's reply has to match the shape.
- **The API** then parses the reply with the Zod schema. That also runs the rules JSON Schema can't express ([How the contract is enforced](#how-the-contract-is-enforced)).
- **The SPA** imports the TypeScript types to render quizzes and the review screen.
- **The committed JSON Schema files** are the language-neutral copy of the contract. CI fails if they drift from the Zod source.

The AI's output is **not** the stored quiz. The stored quiz ([Quizzes](data-and-accounts.md#quizzes)) wraps the AI output and adds things the AI never produces: cropped images, parent edits, publish state. That keeps the contract about the AI only.

## Contract A: quiz extraction

### Contract A request

The files themselves go to Claude as document and image content blocks. The rest of the request becomes the instructions.

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `sources` | `Source[]`, 1 or more | One entry per uploaded file, in upload order |
| `sources[].sourceId` | string | Server-assigned, e.g. `"s1"`. The AI uses it in its references |
| `sources[].kind` | `"pdf"` \| `"image"` | |
| `sources[].filename` | string | Original name, only shown to the model for context |
| `sources[].pageCount` | integer \| null | PDFs only |
| `options.maxQuestions` | integer | The app-wide `maxQuestionsPerQuiz` setting ([Settings summary](product-rules.md#settings-summary)). The result must not contain more questions than this |

v1 is **extraction only**: the model takes questions that already exist in the sources and does not write new ones. Questions are extracted in source order. If the sources contain more questions than `maxQuestions`, the model extracts the first `maxQuestions` and says so in that source's `notes`, e.g. "Extracted 75 of 240 questions; upload the rest as a separate quiz". Generating new questions is a future enhancement ([Roadmap](roadmap.md)).

### Contract A response

Top level:

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `status` | `"ok"` \| `"insufficient_content"` | |
| `reason` | string \| null | Required when `status` is `insufficient_content`, e.g. "The image is too blurry to read" |
| `title` | string | Suggested quiz title |
| `sourceAssessments` | `SourceAssessment[]` | Exactly one per source |
| `contexts` | `Context[]` | Shared material that questions depend on ([Question fields every type has](#question-fields-every-type-has)). Empty if there is none |
| `questions` | `Question[]` | At least 1 when `ok`, empty when `insufficient_content` |

`SourceAssessment` — what the model decided about each file:

| Field | Type | Notes |
|---|---|---|
| `sourceId` | string | |
| `contentKind` | `"questions"` \| `"no_questions"` \| `"unusable"` | `questions`: has questions to extract. `no_questions`: readable, but contains no questions (e.g. plain notes). `unusable`: can't be read, or not in English. If no source is `questions`, the result is `insufficient_content` |
| `hasAnswerKey` | boolean | Whether the source contains answers to its own questions |
| `handwriting` | `"none"` \| `"partial"` \| `"mostly"` | How much of the source is handwritten. The review screen warns the parent to check questions from handwritten sources closely |
| `notes` | string \| null | e.g. "Page 3 is cut off" |

Handwritten source files are supported on a best-effort basis: the model reads what it can, describes unreadable parts in `notes`, and returns `insufficient_content` if nothing usable is left.

**Language:** v1 supports English only. Quizzes are always written in English. A source that is not in English is assessed as `unusable` with a note saying so; if no usable source is left, the result is `insufficient_content` with a reason such as "The document is not in English". An English source with occasional words in another language is fine. The app's own interface is also English.

### Question fields every type has

| Field | Type | Notes |
|---|---|---|
| `id` | string | `"q1"`, `"q2"`, … unique within the result |
| `type` | `QuestionType` | `single_choice` \| `multi_select` \| `true_false` \| `short_answer` \| `descriptive` \| `matching` \| `ordering` \| `diagram_labelling` \| `fill_in_blanks` |
| `sourceNumber` | string \| null | The question's number as printed, e.g. `"5"` or `"3(b)"`. Answer keys refer to questions by it ([Contract C: answer filling](#contract-c-answer-filling)) |
| `answerBasis` | `"answer_key"` \| `"none"` | Whether the sources give this question's answer (table below) |
| `prompt` | string | The question text, as written in the source |
| `contextId` | string \| null | The shared context the question depends on, if any (below) |
| `sourceRefs` | `SourceRef[]`, 1 or more | Where the question, and its answer if given, appear in the sources, so the parent can find it in their own file during review |
| `sourceRefs[].sourceId` | string | Must match a request source |
| `sourceRefs[].page` | integer \| null | 1-based page for PDFs, `null` for images |
| `image` | `ImageRef` \| null | The question's image (a diagram, figure or picture), if it has one ([Question fields per type](#question-fields-per-type)) |
| `explanation` | string \| null | Why the answer is correct, only if the source gives one. Parent-only ([What the child sees after submitting](app-contract.md#what-the-child-sees-after-submitting)) |

`answerBasis` values:

| `answerBasis` | Meaning | Review badge |
|---|---|---|
| `answer_key` | The sources give the answer (an answer key, or a marking scheme or model answer for descriptive questions). All of the question's answer fields are filled | From answer key |
| `none` | The sources have the question but no answer. All of the question's answer fields are `null` | No answer yet |

**Extraction never works out answers.** Copying an answer from the sources is cheap; solving a question takes reasoning, which costs more, takes longer and can be wrong. A question without an answer in the sources comes back with `answerBasis: "none"`, and the parent then chooses how to fill it in ([Contract C: answer filling](#contract-c-answer-filling)).

`Context` — shared material that one or more questions can't be answered without, such as a reading passage, a table or graph used by several questions ("Use Figure 2 for Q3–Q5"), or section instructions ("Give all answers to 2 decimal places"):

| Field | Type | Notes |
|---|---|---|
| `id` | string | `"c1"`, `"c2"`, … unique within the result |
| `title` | string \| null | As printed, e.g. "Passage A" or "Figure 2" |
| `text` | string \| null | The passage or instructions, as written in the source |
| `image` | `ImageRef` \| null | The table, graph, map or picture, cropped like other images |
| `sourceRefs` | `SourceRef[]`, 1 or more | Where it appears in the sources |

- A context has `text`, `image`, or both.
- Every context is used by at least one question, through the question's `contextId`.
- The child sees a question's context with the question ([Answer key stays on the server](app-contract.md#answer-key-stays-on-the-server)), so the quiz JSON holds everything needed to answer it. That is also why answer filling and grading need only the JSON, never the source files ([Contract B: answer grading](#contract-b-answer-grading) and [Contract C: answer filling](#contract-c-answer-filling)).
- Material only one question uses stays in that question's `prompt` and `image`.

### Question fields per type

| `type` | Extra fields |
|---|---|
| `single_choice` | `options: {id, text, image}[]` (2–8), `correctOptionId` |
| `multi_select` | `options: {id, text, image}[]` (2–8), `correctOptionIds` (1 or more) |
| `true_false` | `correctAnswer: boolean` |
| `short_answer` | `acceptedAnswers: string[]` (1 or more, main answer first, then accepted variants) |
| `matching` | `leftItems: {id, text, image}[]` (2–10), `rightItems: {id, text, image}[]` (2–12), `correctPairs: {leftId, rightId}[]` (exactly one per left item) |
| `ordering` | `items: {id, text, image}[]` (3–10) in the order the source shows them, `correctOrder: id[]` |
| `diagram_labelling` | `image` required (the diagram), `labels: {id, marker, acceptedAnswers}[]` (1–15), `wordBank: string[] \| null` |
| `fill_in_blanks` | `blanks: {id, acceptedAnswers}[]` (1–10), `wordBank: string[] \| null`; each blank appears in `prompt` as `[[id]]` |
| `descriptive` | `length: "short" \| "long" \| "essay"`, `rubric: {criterion, description, maxPoints}[]` (1 or more), `modelAnswer: string \| null`, `suggestedWordRange: {min, max} \| null` |

**Answer fields** are `correctOptionId`, `correctOptionIds`, `correctAnswer`, `acceptedAnswers`, `correctPairs`, `correctOrder`, each label's and blank's `acceptedAnswers`, `rubric` and `modelAnswer`. They are all `null` when `answerBasis` is `none`. Everything else (options, items, labels, blanks, markers, word banks) is part of the question and is always filled.

Each option has `text: string | null` and `image: ImageRef | null`, and at least one of the two. An option can be text, a picture, or both.

`ImageRef` — where an image is in the source. The model can see images but can't return image files, so it says where the image is and the server crops it ([Images in questions, options and contexts](product-rules.md#images-in-questions-options-and-contexts)):

| Field | Type | Notes |
|---|---|---|
| `sourceId` | string | Must match a request source |
| `page` | integer \| null | 1-based page for PDFs, `null` for images |
| `box` | `{x, y, width, height}` | Fractions of the page or image, from 0 to 1, measured from the top-left corner. Includes the image's labels and caption |
| `description` | string | A short description of what the image shows. Used as alt text, and helps the parent check the image |

**Matching** — the child pairs each left item with a right item, e.g. organelles with their functions. `rightItems` may contain extra items that match nothing. A right item may be the answer for more than one left item if the source allows it.

**Ordering** — the child puts the items in order, e.g. the stages of the water cycle. `items` keeps the order the source printed them in; that is the order the child first sees. If the source happens to print them in the correct order, the server shuffles them for display.

**Diagram labelling** — the child names the marked parts of a diagram, e.g. parts A, B and C of a leaf. `marker` is the mark as it appears on the diagram (`"A"`, `"1"`). Each label is checked like a short answer against its `acceptedAnswers`. If the source gives a list of words to choose from, it goes in `wordBank` and the child picks from it instead of typing.

**Fill in the blanks** — every fill-in-the-blank question uses this type, with one blank or several. Each blank appears in `prompt` as `[[id]]`, e.g. `"Plants take in [[b1]] and give out [[b2]]."`, and is checked like a short answer against its own `acceptedAnswers`. A word list from the source ("use words from the box") goes in `wordBank`.

Any other format the contract has no type for is skipped and listed in the source's `notes`, e.g. "Skipped 1 crossword on page 7".

`short_answer` is for answers that can be checked by matching: a word, phrase or number. Anything that has to be judged on meaning is `descriptive`:

| `length` | Typical answer | Example | Guidance given to the AI for `suggestedWordRange` |
|---|---|---|---|
| `short` | 1–3 sentences | "Define photosynthesis." | about 20–60 words |
| `long` | A paragraph or two | "Describe the two stages of photosynthesis." | about 80–200 words |
| `essay` | Several paragraphs of argued writing | "Discuss why photosynthesis matters to life on Earth." | about 300 words or more |

`suggestedWordRange` comes from the source when it states one (e.g. "answer in about 50 words"); otherwise the model uses the guidance above. If the source gives a marking scheme, the rubric follows it. If it gives only a model answer, the model turns that answer's key points into a rubric. If it gives neither, `rubric` and `modelAnswer` are `null` and `answerBasis` is `none`.

All three lengths are graded the same way, against the rubric. For `short` and `long` the rubric is usually a list of key points ("mentions light energy – 1 mark"); for `essay` it also covers structure and argument. Every descriptive question has a rubric before the quiz can be published, even when the parent plans to grade by hand, so that AI grading is always possible. The word ranges above are starting guidance and can be tuned.

### Contract A example response

```json
{
  "contractVersion": "1",
  "status": "ok",
  "reason": null,
  "title": "Photosynthesis — Chapter 4",
  "sourceAssessments": [
    { "sourceId": "s1", "contentKind": "questions", "hasAnswerKey": true, "handwriting": "none", "notes": null },
    { "sourceId": "s2", "contentKind": "questions", "hasAnswerKey": false, "handwriting": "none", "notes": "Photo is slightly tilted but readable" }
  ],
  "contexts": [],
  "questions": [
    {
      "id": "q1",
      "type": "single_choice",
      "sourceNumber": "1",
      "answerBasis": "answer_key",
      "prompt": "Which organelle carries out photosynthesis?",
      "contextId": null,
      "sourceRefs": [
        { "sourceId": "s1", "page": 6 },
        { "sourceId": "s1", "page": 9 }
      ],
      "image": null,
      "options": [
        { "id": "a", "text": "Mitochondrion", "image": null },
        { "id": "b", "text": "Chloroplast", "image": null },
        { "id": "c", "text": "Nucleus", "image": null }
      ],
      "correctOptionId": "b",
      "explanation": null
    },
    {
      "id": "q2",
      "type": "descriptive",
      "length": "long",
      "sourceNumber": "5",
      "answerBasis": "none",
      "prompt": "Explain why plants need both light and carbon dioxide to make glucose.",
      "contextId": null,
      "sourceRefs": [
        { "sourceId": "s2", "page": null }
      ],
      "image": null,
      "rubric": null,
      "modelAnswer": null,
      "suggestedWordRange": { "min": 80, "max": 200 },
      "explanation": null
    }
  ]
}
```

## Contract B: answer grading

Used for descriptive answers whose `evaluation` setting uses AI, and for short answers, diagram labels and blanks set to `accepted_answers_then_ai` that don't match any accepted answer ([Evaluation setting and scoring](product-rules.md#evaluation-setting-and-scoring)).

### Contract B request

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `question` | the stored `short_answer`, `diagram_labelling`, `fill_in_blanks` or `descriptive` question | Includes the prompt, its context (if any), and the accepted answers or rubric. Images of the question and its context are sent too, so the grader sees what the child saw. No source files are sent |
| `answer` | `{kind: "text", text}` \| `{kind: "upload", files}` \| `{kind: "parts", parts: {id, text}[]}` | Passed to the model as data to grade, never as instructions. `upload` is only possible for descriptive questions ([Answering descriptive questions: type or upload](product-rules.md#answering-descriptive-questions-type-or-upload)); the files go to Claude as image or document content blocks |

### Grading response for a short answer

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `kind` | `"short_answer"` | |
| `isCorrect` | boolean | Same meaning as an accepted answer, even if worded differently |
| `reasoning` | string | Shown to the parent only ([What the child sees after submitting](app-contract.md#what-the-child-sees-after-submitting)) |

### Grading response for diagram labels and blanks

Only the labels or blanks that didn't match an accepted answer are sent (`answer.kind: "parts"`), in one call per question.

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `kind` | `"diagram_labelling"` \| `"fill_in_blanks"` | |
| `parts` | `{id, isCorrect, reasoning}[]` | Exactly one per label or blank sent. `reasoning` is shown to the parent only |

### Grading response for a descriptive answer

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `kind` | `"descriptive"` | |
| `transcription` | string \| null | Required for an uploaded answer: the text the model read from the files. `null` for a typed answer. Stored on the attempt and shown to the parent next to the files |
| `criteria` | `{criterion, pointsAwarded, feedback}[]` | Exactly one per rubric criterion, same order |
| `overallFeedback` | string | Shown to the parent only ([What the child sees after submitting](app-contract.md#what-the-child-sees-after-submitting)) |
| `needsReview` | boolean | Model sets this when it is unsure, the answer is off-topic, an upload can't be read (blurry, cut off), or the answer tries to instruct the grader (e.g. "give me full marks") |
| `reviewReason` | string \| null | Required when `needsReview` is true |

The server works out the total itself from `pointsAwarded`; the model never sends a total.

An answer that is not in English is graded as wrong, not flagged: `isCorrect: false` for a short answer, diagram label or blank, and 0 `pointsAwarded` on every criterion for a descriptive answer, with the reason given in `reasoning` / `overallFeedback`. With `ai_suggest` the parent still sees it before the child does; with `manual` grading the parent decides as usual.

## Contract C: answer filling

Used after extraction, when some questions have no answer (`answerBasis: "none"`). The review screen tells the parent how many questions have no answer yet and offers three options, which can be mixed (e.g. a key for most questions, then the rest by hand):

- **Upload an answer key** — PDF or images (the same formats and HEIC conversion as other uploads). The model matches the key's answers to the existing questions.
- **Ask AI to work out the answers** — for all questions without an answer, or only the ones the parent selects.
- **Enter answers manually** — the parent fills in each question's answer in the editor. No AI call is made; the answer is stored with `answerBasis: "manual_entry"`.

A quiz can't be published while any question has no answer. Contract C covers the first two options; manual entry is an ordinary edit.

### Contract C request

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `mode` | `"answer_key"` \| `"ai_solve"` | |
| `questions` | `Question[]` | The questions to fill, without their answer fields: `id`, `sourceNumber`, `type`, `prompt`, `contextId`, `image`, options, items, labels, blanks, word banks and `length`. Their images are sent as image content blocks |
| `contexts` | `Context[]` | The contexts the questions sent refer to, with their images |
| `answerFiles` | `Source[]` | `answer_key` mode only: the uploaded answer key files, with their own `sourceId`s. Sent as document and image content blocks |

- **No source files are sent.** The questions and their contexts hold everything a child needs to answer them, so they are enough for the model too ([Question fields every type has](#question-fields-every-type-has)).
- **`answer_key` mode** sends all the questions in one call: it is copying, not solving.
- **`ai_solve` mode** sends the questions in batches (batch size in server config, e.g. 20), so each response stays well within the output limit. If a question can't be answered from what is sent, the model returns `cannot_solve` and the parent fills it in by hand.

### Contract C response

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `answers` | `FilledAnswer[]` | Exactly one per question sent |
| `notes` | string \| null | e.g. "The answer key covers questions 1–20 only" |

`FilledAnswer`:

| Field | Type | Notes |
|---|---|---|
| `questionId` | string | Must match a question sent |
| `status` | `"filled"` \| `"not_found"` \| `"cannot_solve"` | `not_found`: `answer_key` mode, the key has no answer for this question. `cannot_solve`: `ai_solve` mode, the model can't answer reliably (e.g. information is missing) |
| `answer fields` | as in [Question fields per type](#question-fields-per-type) for the question's type | Filled when `status` is `filled`, otherwise `null`. For descriptive questions this includes a `rubric` (from the key's marking scheme, or written by the model) and `modelAnswer` |
| `explanation` | string \| null | From the key if it gives one; in `ai_solve` mode, written by the model |
| `sourceRefs` | `SourceRef[]` | `answer_key` mode: where the answer is in the answer key. `ai_solve` mode: empty |
| `reason` | string \| null | Required when `status` is not `filled` |

### Applying the answers

The stored question's `answerBasis` records where its answer came from: `answer_key` (from the original sources or an uploaded key), `ai_solved`, `manual_entry` (typed by hand) or `none`. Review badges: "From answer key", "AI answer", "Entered manually", "No answer yet".

- **An answer key overrides AI answers.** If the parent asked the AI first and then uploads a key, the key's answers replace the AI's, and the parent sees each difference, e.g. "Q4: answer key says B; AI answered C".
- **Answers the parent typed are never overwritten.** Where the key disagrees, the parent sees both and picks one.
- **AI never overwrites an answer from a key or the parent;** it is only offered for questions with no answer.
- **Descriptive questions:** the filled rubric sets the question's `points` (rubric total, [Quizzes](data-and-accounts.md#quizzes)).
- **Answer key files** count toward the quiz's upload limits ([Upload limits](architecture.md#upload-limits)). Like source files, they are deleted once the call has finished ([Uploaded files are temporary](architecture.md#uploaded-files-are-temporary)).
- **On a published quiz,** filling or replacing answers is an ordinary edit: it increments `quizVersion` and follows the usual edit rules ([Quizzes](data-and-accounts.md#quizzes)).

## How the contract is enforced

Three layers, in order:

1. **Structured outputs.** Claude is given the JSON Schema, so the reply has the right fields and types.
2. **Zod parsing on the server**, including rules JSON Schema can't express:
   - `status: ok` has at least one question; `insufficient_content` has none and has a `reason`
   - question ids are unique; option ids are unique within a question
   - `correctOptionId` / `correctOptionIds` refer to existing options
   - context ids are unique; every context has `text`, `image`, or both; every `contextId` refers to an existing context; every context is used by at least one question
   - every `sourceRefs[].sourceId` and `image.sourceId` exists in the request; pages are within `pageCount`
   - every `box` lies within 0–1 and has a non-zero width and height
   - every option and item has `text`, `image`, or both
   - `matching`: ids are unique; every `correctPairs` entry refers to existing items; exactly one pair per left item
   - `ordering`: `correctOrder` contains every item id exactly once
   - `diagram_labelling`: has an `image`; markers are unique; every label has at least one accepted answer; with a `wordBank`, every accepted answer is in it
   - `fill_in_blanks`: every blank id appears exactly once in `prompt` as `[[id]]` and the prompt has no other `[[…]]`; every blank has at least one accepted answer; with a `wordBank`, every accepted answer is in it
   - the number of questions is at most `options.maxQuestions`
   - `answerBasis: "answer_key"` has every answer field for its type filled; `answerBasis: "none"` has every answer field `null`
   - answer filling: exactly one `FilledAnswer` per question sent; `filled` answers pass the same per-type checks as above; `not_found` and `cannot_solve` have a `reason`
   - grading: one `criteria` entry per rubric criterion, and `0 ≤ pointsAwarded ≤ maxPoints`

   A reply that fails parsing is retried once with the errors given back to the model; if it fails again, the job fails with a clear error.
3. **Parent review** before publishing. This is the main protection against the model's mistakes, such as a misread question, an invented option or a wrong answer. The review screen shows:
   - the answer badges ([Applying the answers](#applying-the-answers))
   - a warning on questions from handwritten sources ([Contract A response](#contract-a-response))
   - a **numbering gap warning**: if `sourceNumber`s skip a number (e.g. 1–6 then 8), "Question 7 may be missing"
   - each question's `sourceRefs`, so the parent can find it in their own file

There is no automatic check that the model's text matches the source files in v1. It would only work for PDFs with a text layer, not for photos, and it can't catch a wrong answer or a missed question. Whether one is needed later will be judged from how often parents correct extracted questions (`editedAfterDraft`, [Quizzes](data-and-accounts.md#quizzes)) compared with the AI's original response.

## Versioning

- `contractVersion` is a fixed value in every request and response.
- Any change that could break an existing reader (removing or renaming a field, narrowing allowed values) means a new version, `"2"`, with its own schema file. The app keeps a reader for every version it has stored.
- Adding an optional field with a safe default can stay in the same version.
- `contractVersion` is the version of the **contract's shape**. It is unrelated to `quizVersion` ([Quizzes](data-and-accounts.md#quizzes)), which counts edits to one quiz.
