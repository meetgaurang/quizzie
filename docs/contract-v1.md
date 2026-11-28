# Quizzie AI contract — v1 (draft for review)

Status: **draft, ready for full review**. Nothing is implemented yet. Still open:

- which sign-in methods parents get (section 7.1)
- how long attempts and their answers are kept: every answer to every question in every attempt is stored, so a retention rule (e.g. archive or delete after two years) is worth deciding before launch, also as a privacy commitment for children's data (section 7.3)
- whether a child's answers are saved as they go, so a closed tab or a flat battery doesn't lose the attempt, or only sent at submit. Saving as they go would use the same answer shapes (section 7.3)
- which AWS service runs the API: App Runner or ECS Fargate (section 7)

Future enhancements beyond v1 are listed at the end.

In v1 a **parent** makes and manages quizzes, a **child** takes them, and the **app owner** makes the global quizzes anyone can browse. Section 7.1 has the details.

## 1. What the contract is

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
  src/quiz-views.ts          answer-free view and result view (sections 7.4, 7.5): API → web app
  src/attempt.ts             answer shapes and submission (section 7.3): web app → API
```

The last two are not AI contracts. They are the contract between the web app and the API, kept in the same package so the quiz screen and the marking code share one definition.

- **The API** sends the generated JSON Schema to Claude as the required output format (structured outputs), so the model's reply has to match the shape.
- **The API** then parses the reply with the Zod schema. That also runs the rules JSON Schema can't express (section 6).
- **The SPA** imports the TypeScript types to render quizzes and the review screen.
- **The committed JSON Schema files** are the language-neutral copy of the contract. CI fails if they drift from the Zod source.

The AI's output is **not** the stored quiz. The stored quiz (section 7) wraps the AI output and adds things the AI never produces: cropped images, parent edits, publish state. That keeps the contract about the AI only.

## 2. Contract A — quiz extraction

### 2.1 Request (API → AI step)

The files themselves go to Claude as document and image content blocks. The rest of the request becomes the instructions.

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `sources` | `Source[]`, 1 or more | One entry per uploaded file, in upload order |
| `sources[].sourceId` | string | Server-assigned, e.g. `"s1"`. The AI uses it in its references |
| `sources[].kind` | `"pdf"` \| `"image"` | |
| `sources[].filename` | string | Original name, only shown to the model for context |
| `sources[].pageCount` | integer \| null | PDFs only |
| `options.maxQuestions` | integer | The app-wide `maxQuestionsPerQuiz` setting (section 7.8). The result must not contain more questions than this |

v1 is **extraction only**: the model takes questions that already exist in the sources and does not write new ones. Questions are extracted in source order. If the sources contain more questions than `maxQuestions`, the model extracts the first `maxQuestions` and says so in that source's `notes`, e.g. "Extracted 75 of 240 questions; upload the rest as a separate quiz". Generating new questions is a future enhancement (section 9).

### 2.2 Response (AI → API)

Top level:

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `status` | `"ok"` \| `"insufficient_content"` | |
| `reason` | string \| null | Required when `status` is `insufficient_content`, e.g. "The image is too blurry to read" |
| `title` | string | Suggested quiz title |
| `sourceAssessments` | `SourceAssessment[]` | Exactly one per source |
| `contexts` | `Context[]` | Shared material that questions depend on (section 2.3). Empty if there is none |
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

### 2.3 Question — fields every type has

| Field | Type | Notes |
|---|---|---|
| `id` | string | `"q1"`, `"q2"`, … unique within the result |
| `type` | `QuestionType` | `single_choice` \| `multi_select` \| `true_false` \| `short_answer` \| `descriptive` \| `matching` \| `ordering` \| `diagram_labelling` \| `fill_in_blanks` |
| `sourceNumber` | string \| null | The question's number as printed, e.g. `"5"` or `"3(b)"`. Answer keys refer to questions by it (section 4) |
| `answerBasis` | `"answer_key"` \| `"none"` | Whether the sources give this question's answer (table below) |
| `prompt` | string | The question text, as written in the source |
| `contextId` | string \| null | The shared context the question depends on, if any (below) |
| `sourceRefs` | `SourceRef[]`, 1 or more | Where the question, and its answer if given, appear in the sources, so the parent can find it in their own file during review |
| `sourceRefs[].sourceId` | string | Must match a request source |
| `sourceRefs[].page` | integer \| null | 1-based page for PDFs, `null` for images |
| `image` | `ImageRef` \| null | The question's image (a diagram, figure or picture), if it has one (section 2.4) |
| `explanation` | string \| null | Why the answer is correct, only if the source gives one. Parent-only (section 7.5) |

`answerBasis` values:

| `answerBasis` | Meaning | Review badge |
|---|---|---|
| `answer_key` | The sources give the answer (an answer key, or a marking scheme or model answer for descriptive questions). All of the question's answer fields are filled | From answer key |
| `none` | The sources have the question but no answer. All of the question's answer fields are `null` | No answer yet |

**Extraction never works out answers.** Copying an answer from the sources is cheap; solving a question takes reasoning, which costs more, takes longer and can be wrong. A question without an answer in the sources comes back with `answerBasis: "none"`, and the parent then chooses how to fill it in (section 4).

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
- The child sees a question's context with the question (section 7.4), so the quiz JSON holds everything needed to answer it. That is also why answer filling and grading need only the JSON, never the source files (sections 3 and 4).
- Material only one question uses stays in that question's `prompt` and `image`.

### 2.4 Question — fields per type

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

`ImageRef` — where an image is in the source. The model can see images but can't return image files, so it says where the image is and the server crops it (section 7.7):

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

### 2.5 Example response

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

## 3. Contract B — answer grading

Used for descriptive answers whose `evaluation` setting uses AI, and for short answers, diagram labels and blanks set to `accepted_answers_then_ai` that don't match any accepted answer (section 5).

### 3.1 Request (API → AI step)

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `question` | the stored `short_answer`, `diagram_labelling`, `fill_in_blanks` or `descriptive` question | Includes the prompt, its context (if any), and the accepted answers or rubric. Images of the question and its context are sent too, so the grader sees what the child saw. No source files are sent |
| `answer` | `{kind: "text", text}` \| `{kind: "upload", files}` \| `{kind: "parts", parts: {id, text}[]}` | Passed to the model as data to grade, never as instructions. `upload` is only possible for descriptive questions (section 7.6); the files go to Claude as image or document content blocks |

### 3.2 Response for a short answer

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `kind` | `"short_answer"` | |
| `isCorrect` | boolean | Same meaning as an accepted answer, even if worded differently |
| `reasoning` | string | Shown to the parent only (section 7.5) |

### 3.3 Response for diagram labels and blanks

Only the labels or blanks that didn't match an accepted answer are sent (`answer.kind: "parts"`), in one call per question.

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `kind` | `"diagram_labelling"` \| `"fill_in_blanks"` | |
| `parts` | `{id, isCorrect, reasoning}[]` | Exactly one per label or blank sent. `reasoning` is shown to the parent only |

### 3.4 Response for a descriptive answer

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `kind` | `"descriptive"` | |
| `transcription` | string \| null | Required for an uploaded answer: the text the model read from the files. `null` for a typed answer. Stored on the attempt and shown to the parent next to the files |
| `criteria` | `{criterion, pointsAwarded, feedback}[]` | Exactly one per rubric criterion, same order |
| `overallFeedback` | string | Shown to the parent only (section 7.5) |
| `needsReview` | boolean | Model sets this when it is unsure, the answer is off-topic, an upload can't be read (blurry, cut off), or the answer tries to instruct the grader (e.g. "give me full marks") |
| `reviewReason` | string \| null | Required when `needsReview` is true |

The server works out the total itself from `pointsAwarded`; the model never sends a total.

An answer that is not in English is graded as wrong, not flagged: `isCorrect: false` for a short answer, diagram label or blank, and 0 `pointsAwarded` on every criterion for a descriptive answer, with the reason given in `reasoning` / `overallFeedback`. With `ai_suggest` the parent still sees it before the child does; with `manual` grading the parent decides as usual.

## 4. Contract C — answer filling

Used after extraction, when some questions have no answer (`answerBasis: "none"`). The review screen tells the parent how many questions have no answer yet and offers three options, which can be mixed (e.g. a key for most questions, then the rest by hand):

- **Upload an answer key** — PDF or images (the same formats and HEIC conversion as other uploads). The model matches the key's answers to the existing questions.
- **Ask AI to work out the answers** — for all questions without an answer, or only the ones the parent selects.
- **Enter answers manually** — the parent fills in each question's answer in the editor. No AI call is made; the answer is stored with `answerBasis: "manual_entry"`.

A quiz can't be published while any question has no answer. Contract C covers the first two options; manual entry is an ordinary edit.

### 4.1 Request (API → AI step)

| Field | Type | Notes |
|---|---|---|
| `contractVersion` | `"1"` | |
| `mode` | `"answer_key"` \| `"ai_solve"` | |
| `questions` | `Question[]` | The questions to fill, without their answer fields: `id`, `sourceNumber`, `type`, `prompt`, `contextId`, `image`, options, items, labels, blanks, word banks and `length`. Their images are sent as image content blocks |
| `contexts` | `Context[]` | The contexts the questions sent refer to, with their images |
| `answerFiles` | `Source[]` | `answer_key` mode only: the uploaded answer key files, with their own `sourceId`s. Sent as document and image content blocks |

- **No source files are sent.** The questions and their contexts hold everything a child needs to answer them, so they are enough for the model too (section 2.3).
- **`answer_key` mode** sends all the questions in one call: it is copying, not solving.
- **`ai_solve` mode** sends the questions in batches (batch size in server config, e.g. 20), so each response stays well within the output limit. If a question can't be answered from what is sent, the model returns `cannot_solve` and the parent fills it in by hand.

### 4.2 Response (AI → API)

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
| `answer fields` | as in section 2.4 for the question's type | Filled when `status` is `filled`, otherwise `null`. For descriptive questions this includes a `rubric` (from the key's marking scheme, or written by the model) and `modelAnswer` |
| `explanation` | string \| null | From the key if it gives one; in `ai_solve` mode, written by the model |
| `sourceRefs` | `SourceRef[]` | `answer_key` mode: where the answer is in the answer key. `ai_solve` mode: empty |
| `reason` | string \| null | Required when `status` is not `filled` |

### 4.3 Applying the answers

The stored question's `answerBasis` records where its answer came from: `answer_key` (from the original sources or an uploaded key), `ai_solved`, `manual_entry` (typed by hand) or `none`. Review badges: "From answer key", "AI answer", "Entered manually", "No answer yet".

- **An answer key overrides AI answers.** If the parent asked the AI first and then uploads a key, the key's answers replace the AI's, and the parent sees each difference, e.g. "Q4: answer key says B; AI answered C".
- **Answers the parent typed are never overwritten.** Where the key disagrees, the parent sees both and picks one.
- **AI never overwrites an answer from a key or the parent;** it is only offered for questions with no answer.
- **Descriptive questions:** the filled rubric sets the question's `points` (rubric total, section 7.2).
- **Answer key files** count toward the quiz's upload limits (section 7.9). Like source files, they are deleted once the call has finished (section 7.2).
- **On a published quiz,** filling or replacing answers is an ordinary edit: it increments `quizVersion` and follows the usual edit rules (section 7.2).

## 5. Evaluation setting per question (set by the parent)

Each question has an `evaluation` setting, chosen when the quiz is created or edited. The question type limits which values are allowed:

| Question type | Allowed `evaluation` values | Default |
|---|---|---|
| `single_choice`, `multi_select`, `true_false`, `matching`, `ordering` | `automatic` (fixed) | `automatic` |
| `short_answer`, `diagram_labelling`, `fill_in_blanks` | `accepted_answers_then_ai`, `accepted_answers_only` | `accepted_answers_then_ai` |
| `descriptive` | `ai_auto`, `ai_suggest`, `manual` | From the descriptive default setting (below) |

What each value does:

| Value | Flow |
|---|---|
| `automatic` | Compared with the correct option(s), true/false answer, pairs or order. Multi-select, matching and ordering can earn part of the points (below) |
| `accepted_answers_only` | Matched against `acceptedAnswers` (per label or blank, for diagram labelling and fill in the blanks), ignoring case, surrounding spaces and repeated spaces. With a `wordBank`, the child picks a word, so it's an exact match |
| `accepted_answers_then_ai` | Same match first; if there's no match, contract B decides |
| `ai_auto` | AI grade goes straight to the child, unless `needsReview` is true, in which case it waits for the parent |
| `ai_suggest` | AI grade is a suggestion; the parent accepts or changes it before the child sees it |
| `manual` | No AI call; the parent grades with the rubric |

Scoring, as a share of the question's `points`:

| Type | Points awarded |
|---|---|
| `single_choice`, `true_false`, `short_answer` | Full points if correct, otherwise 0 |
| `multi_select` | Full points if exactly the correct options are picked; half points if at least one correct option is picked but not exactly the correct set; 0 if no correct option is picked (including nothing picked) |
| `matching` | A share per correct pair: points × correct pairs ÷ left items |
| `ordering` | A share per item in its correct position: points × items in place ÷ items |
| `diagram_labelling` | A share per correct label: points × correct labels ÷ labels |
| `fill_in_blanks` | A share per correct blank: points × correct blanks ÷ blanks |

For example, a matching question worth 2 points with 3 of 4 pairs right earns 1.5 points. Unanswered pairs, positions, labels and blanks count as wrong. Points awarded are rounded to two decimal places.

Picking every option of a multi-select question scores 0, unless every option is correct. Otherwise picking everything would always earn half points.

- **Whether a question needs AI evaluation** follows from its `evaluation` value (`accepted_answers_then_ai`, `ai_auto`, `ai_suggest`). There is no separate flag to keep in sync.
- **Whether a quiz needs AI evaluation** is calculated: true if any of its questions does. It is not stored.
- **Descriptive default setting** (`descriptiveEvaluationDefault`) exists at two levels, like `resultVisibility` (section 7.5):
  - **Account** — in the account's settings. Default for a new account: `ai_auto`.
  - **Quiz** — on the quiz, one of the three descriptive values or `null` to use the account value. A quiz-level value overrides the account one. It can be chosen at upload, before the AI draft arrives.
  - The default is **copied** into a descriptive question's `evaluation` when the question is created (AI draft or added by the parent). After that the question keeps its own value, so changing a default never changes existing questions or the grading of attempts already submitted.
  - Changing the quiz-level default affects only descriptive questions created after the change. To change an existing descriptive question, the parent edits that question.
- **The AI extraction response does not include `evaluation`.** How to grade is the parent's decision. The app fills in the defaults by question type when a draft arrives, and the parent changes them in review.

## 6. How the contract is enforced

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
   - the answer badges (section 4.3)
   - a warning on questions from handwritten sources (section 2.2)
   - a **numbering gap warning**: if `sourceNumber`s skip a number (e.g. 1–6 then 8), "Question 7 may be missing"
   - each question's `sourceRefs`, so the parent can find it in their own file

There is no automatic check that the model's text matches the source files in v1. It would only work for PDFs with a text layer, not for photos, and it can't catch a wrong answer or a missed question. Whether one is needed later will be judged from how often parents correct extracted questions (`editedAfterDraft`, section 7.2) compared with the AI's original response.

## 7. Accounts, quizzes and attempts (app data, not part of the AI contract)

Shown here only to make the boundary with the AI contract clear.

**Hosting.** Supabase is used for sign-in only (section 7.1). Everything else runs in Quizzie's AWS account, in the Sydney region (`ap-southeast-2`), so all quiz, attempt and child data stays in Australia:

| Part | Where |
|---|---|
| Sign-in (parents' emails and passwords) | Supabase Auth, in a Supabase project in the Sydney region, on a paid plan for production because free projects pause when idle |
| Web app (the SPA's static files) | AWS: S3 and CloudFront |
| API (Fastify) | AWS: containers on App Runner or ECS Fargate (to be chosen) |
| AI jobs (extraction, answer filling, grading) | AWS: an SQS queue and a worker. The API answers straight away and the web app checks back, because an AI call can take longer than a web request should wait |
| Database | AWS: RDS for PostgreSQL, starting on one `db.t4g.micro` instance in a single availability zone. A reserved instance and a standby in a second zone can be added once usage is steady |
| Images and uploaded files | AWS: S3, in private buckets. The API hands the browser short-lived links only to files the signed-in user may see. A lifecycle rule deletes leftover temporary uploads after one day, as a safety net for section 7.2 |

The database is in a private network next to the API and is not reachable from the internet. The API reaches the database and S3 through AWS permission roles, not stored passwords or keys.

**Tables:**

| Table | Holds |
|---|---|
| `accounts` | One row per parent account: `authUserId` (the Supabase Auth user id), the parent's details, `isAppOwner`, the account settings |
| `child_profiles` | One row per child profile |
| `quizzes` | One row per quiz. `contexts` and `questions` are `jsonb` columns, checked by the contract's Zod schemas |
| `quiz_assignments` | Which children each quiz is given to |
| `quiz_uploads` | A record of each uploaded file (the files themselves are temporary, section 7.2) |
| `quiz_images` | One row per image; the image file itself is in S3 |
| `ai_responses` | The AI's original extraction and answer-filling responses, unchanged, in a `jsonb` column |
| `attempts` | One row per attempt |
| `attempt_answers` | One row per question in an attempt, including skipped ones (section 7.3) |
| `answer_uploads` | One row per file a child uploads for a descriptive answer |
| `app_settings` | App-wide values set by the app owner, such as `maxQuestionsPerQuiz` |

- Document-shaped data (questions, contexts, answers, AI responses) is stored in `jsonb` columns. Everything Quizzie filters or joins on (owner, status, version, grading status) is an ordinary column.
- Image and upload files are not stored in the database; their rows hold their S3 location.
- All access goes through the Quizzie API (section 7.1).

### 7.1 Accounts, profiles and quiz ownership

Who's who in v1:

| Role | Who | What they can do |
|---|---|---|
| **App owner** | Whoever runs Quizzie | Makes global quizzes the same way a parent makes their own quizzes: upload, review, fill answers, publish. Sets the app-wide limits |
| **Parent** | The account holder, and the account's only login | Creates quizzes from their own uploads, copies global quizzes into the account, edits, fills answers, grades, chooses which children get each quiz, and sees every result. Doesn't take quizzes |
| **Child** | A profile inside the parent's account, with no login, email or password | Takes the quizzes given to them, and sees their own results as allowed by `resultVisibility` (section 7.5) |
| **Visitor** | Anyone, signed in or not | Browses global quizzes |

Wherever this document says the parent does something to a quiz, the app owner does the same to a global quiz. Teachers, schools and classes come later (section 9) and will take the parent and child roles.

**Account:** `accountId`, `authUserId` (the parent's Supabase Auth user), the parent's details, `profiles[]` and the account settings (section 7.8).

Quizzie collects only the personal details it uses:

| Record | Field | Required | Used for |
|---|---|---|---|
| Parent | `firstName` | Yes | Greetings in the app and in emails |
| Parent | `lastName` | No | |
| Parent | `city` | No | |
| Parent | `country` | Yes | Which privacy rules apply; later prices, tax and curriculum |
| Parent | `timeZone` | Yes, detected by the browser | Showing attempt dates and times |
| Child profile | `profileId`, `name` | Yes | A first name or nickname, so the parent can tell profiles apart |
| Child profile | `yearLevel` | No | Later, suggesting global quizzes |

- The parent's email and password are held by Supabase only. The details above are Quizzie data.
- Not collected: street address, phone number, and a child's last name, date of birth, school or photo. If paid plans arrive, the payment provider collects the billing address.

**Signing in** uses Supabase Auth:

- The web app signs the parent in with Supabase and sends the access token (a JWT) with every API request.
- The API verifies the token itself against Supabase's published signing keys (the project uses asymmetric JWT signing keys), then finds the account from the token's user id.
- **The account is created on first sign-in.** Supabase and Quizzie's database are separate, so when the API sees a valid token for a user it has no account for, it creates the `accounts` row and asks the parent for their details. The database can't enforce that `authUserId` matches a real Supabase user; the API does.
- **Deleting an account** removes the Supabase user and all of the account's Quizzie data (profiles, quizzes, attempts, answers, files) in one step run by the API.
- Supabase stores the parent's email and password; Quizzie never handles passwords. Child profiles are Quizzie data only, so Supabase holds no information about children.
- The app owner's role is a flag on the app owner's account, checked by the API.
- All quiz data goes through the Quizzie API; the browser talks to Supabase only to sign in, so the server stays in control of what reaches the browser (section 7.4).
- Which sign-in methods parents get in v1 (email and password, an emailed link, Google) is not decided yet.

A child profile can later be linked to its own login (section 9); the data model doesn't change, because the login points to the existing `profileId`.

**Switching profiles** needs no PIN or password in v1. **Known limitation:** a child using the device can switch to the parent view and see answers, rubrics and other children's results. A parent PIN is listed for later (section 9).

**Quiz ownership.** Every quiz has an `owner`: `{kind: "app"}` for a global quiz, or `{kind: "account", accountId}`.

| | Global quiz | Account quiz |
|---|---|---|
| Made by | The app owner only, so the app owner controls what copyrighted material is published | The parent, from their own uploads, or copied from a global quiz |
| Who can see it | Anyone, as the answer-free view (section 7.4): the questions, never the answers | Only that account |
| Attempts | None; it can't be submitted | By the children it is given to |

- A parent's uploads and quizzes are private to their account and never become global.
- The API checks every request for an account quiz, its attempts or its results against the signed-in account.

**Browsing global quizzes.** Only published global quizzes are listed. Anyone can browse them without signing in, in the answer-free view, with no way to submit. Each has an **Assign to my account** button; a visitor who isn't signed in is asked to sign in or create an account first.

**Assign to my account** makes a copy:

- The copy is a new quiz with a new `quizId`, owned by the account, at `quizVersion` 1 and `published`, because a global quiz can only be published once every question has an answer.
- **Copied:** the title, the contexts, the questions with all their answer fields, `answerBasis`, `evaluation` and `points`, the images (as new files with new random names), and the quiz-level settings.
- **Not copied:** the AI responses (`ai_responses`), and the `sourceRefs` of questions, contexts and answers. They refer to the app owner's uploads, which the parent never had, so they stay with the global quiz. There are no source files to copy, because uploaded files are deleted after extraction (section 7.2).
- The copy records `copiedFrom: {quizId, quizVersion}`.
- The copy is fully editable, like any account quiz. Editing it never changes the global quiz, and later edits to the global quiz never reach copies. Unpublishing or deleting a global quiz doesn't affect existing copies either.
- Assigning the same global quiz again makes another, separate copy. The browse page shows when a quiz is already in the account.

**Giving a quiz to children.** An account quiz has `assignedProfileIds[]`, the children it's given to; it starts empty, including for a copy.

- A child sees only published quizzes given to them.
- A child can take a quiz as many times as they like. Each attempt is kept with its own score.
- Taking a child off a quiz stops new attempts; their earlier attempts and results are kept.
- Changing who a quiz is given to doesn't change its questions, so it doesn't increment `quizVersion`.

### 7.2 Quiz

There is one copy of each quiz. It carries a `quizVersion` number, and every saved change to the quiz increments it. Earlier versions are not kept.

Changes to a quiz can't be saved while any of its attempts has a pending grade: an AI grading call still running, an `ai_suggest` grade waiting for the parent, a `manual` descriptive answer not yet graded, or an AI grade with `needsReview`. The editor tells the parent how many grades are pending and links to them.

**Uploaded files are temporary.** Source files and answer key files are kept only until the AI call that uses them has finished and, for source files, the images have been cropped. Then they are deleted, and only their row in `quiz_uploads` remains: filename, kind (`source` or `answer_key`), type, size and pages. The parent never needs to upload them again: images are replaced by uploading a new image (section 7.7), and answer filling and grading work from the quiz JSON (section 2.3). If extraction fails, the files are deleted too, and the parent uploads them again to retry.

A quiz (a row in `quizzes`):

| Field | Notes |
|---|---|
| `quizId`, `title`, `createdAt`, `updatedAt` | |
| `owner` | `{kind: "app"}` (global) or `{kind: "account", accountId}` (section 7.1) |
| `copiedFrom` | `{quizId, quizVersion}` of the global quiz it was copied from, or `null` |
| `assignedProfileIds` | The children the quiz is given to. Account quizzes only |
| `quizVersion` | Starts at 1; +1 on every saved change |
| `status` | `draft` → `published`. Publishing is blocked while any question has `answerBasis: "none"` |
| `contexts[]` | The shared contexts as they are now (section 2.3), each with its image |
| `questions[]` | The questions as they are now, each with `fromAiQuestionId`, `contextId`, `answerBasis` (`answer_key` \| `ai_solved` \| `manual_entry` \| `none`, section 4.3), `editedAfterDraft: boolean`, `evaluation`, `points` |

Points per question are set by the app, not the AI, when the question is created:

| Question type | Default `points` |
|---|---|
| `single_choice`, `multi_select`, `true_false`, `short_answer`, `matching`, `ordering`, `diagram_labelling`, `fill_in_blanks` | 1 |
| `descriptive` | The rubric total (sum of the criteria's `maxPoints`). `null` until the question has a rubric |

The parent can change points in review or when editing. For a descriptive question, `points` always equals its rubric total: to make it worth more, the parent changes the criteria's `maxPoints`, so grades never need scaling.

The responses in `ai_responses` are the AI's output, not copies of the quiz. Each row also records the model, the token counts, when the call ran and whether it succeeded. They are written once and let us see what the AI produced versus what the parent changed, and later measure extraction quality.

### 7.3 Attempts

| Field | Notes |
|---|---|
| `attemptId`, `quizId` | |
| `profileId` | The child who made the attempt. Only a child the quiz is given to can start one (section 7.1) |
| `quizVersion` | The quiz's version when the attempt started. It must still match at submit (see below) |
| `startedAt`, `submittedAt` | |
| `answers[]` | Per question, including skipped ones: `questionId`, the child's `answer` (shapes below), how it was graded (`automatic`, `accepted_answer`, `ai` or `parent`), `gradingStatus` (`graded` \| `pending`), `result`, `pointsAwarded`, `maxPoints` at the time of grading, and any AI grading response (contract B) |
| `score`, `maxScore` | Calculated by the server from the answers |

**Answer shapes.** Each answer carries its question's `type`, so the server checks it against the right schema (a Zod discriminated union) and rejects an answer of the wrong type:

| Type | `answer` | Rules |
|---|---|---|
| `single_choice` | `{type: "single_choice", optionId}` | One of the question's options |
| `multi_select` | `{type: "multi_select", optionIds: string[]}` | 1 or more, no repeats, all existing options. Nothing ticked is sent as `null` |
| `true_false` | `{type: "true_false", value: boolean}` | |
| `short_answer` | `{type: "short_answer", text}` | Trimmed; within the short-text limit (below) |
| `matching` | `{type: "matching", pairs: {leftId, rightId}[]}` | At most one pair per left item; a right item may be used more than once (section 2.4). Left items without a pair count as wrong |
| `ordering` | `{type: "ordering", order: itemId[]}` | Every item exactly once |
| `diagram_labelling` | `{type: "diagram_labelling", labels: {id, text}[]}` | At most one entry per label; missing labels count as wrong. With a `wordBank`, each `text` is one of its words. Each `text` is trimmed and within the short-text limit (below) |
| `fill_in_blanks` | `{type: "fill_in_blanks", blanks: {id, text}[]}` | Same rules as labels |
| `descriptive`, typed | `{type: "descriptive", kind: "text", text}` | Within the limit for the question's `length` (below). The `suggestedWordRange` is guidance only |
| `descriptive`, uploaded | `{type: "descriptive", kind: "upload", uploadIds: string[]}` | Only when uploads are allowed for this attempt (section 7.6). Files are uploaded first and get ids; limits in section 7.9 |
| Not answered | `null` | |

- **Ordering always has an answer:** whatever order is on screen at submit is the child's answer, even if they never moved anything, and it earns a share for each item that happens to be in its correct position (section 5).
- **Words picked from a word bank are sent as text,** so labels and blanks are marked the same way whether the child typed or picked.

**Text limits.** Each typed answer has a maximum length, so nobody can paste thousands of words into a question that needs one word or one sentence. The limits are well above the suggested word ranges (section 2.4), so a genuine answer never comes close:

| Text | Limit | Roughly |
|---|---|---|
| Short text: a `short_answer`, each diagram label, each blank | 200 characters | 30 words |
| `descriptive`, length `short` | 2,000 characters | 300 words |
| `descriptive`, length `long` | 5,000 characters | 800 words |
| `descriptive`, length `essay` | 30,000 characters | 5,000 words |

- The server enforces the limits; an answer over its limit rejects the submission.
- The quiz screen shows a counter (e.g. "150 / 300 words") and stops typing at the limit, so the child sees the boundary instead of an error.
- The limit is the one for the question's `length` when the attempt is answered. A parent changing a question's length is an ordinary edit, so earlier attempts are unaffected.
- Uploaded answers are limited by files, size and pages instead (section 7.9).
- The limits are in the server config, like the upload limits, so they can be changed without a code change.

**Submission:**

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

- If the quiz changed while the attempt was in progress (its `quizVersion` no longer matches), the submission is rejected with "Quiz was changed, please start again" and the in-progress attempt is discarded.
- The mark of every attempt is shown against its `quizVersion`.
- Submitted attempts are **not** re-graded when the quiz changes. Points are stored on the attempt, so its score stays the same whatever happens to the quiz later.
- An attempt's answers can only be viewed while its `quizVersion` matches the quiz's current `quizVersion`. Otherwise its details are not shown; the UI shows its mark and version with the message "Quiz was changed after this attempt."

### 7.4 Answer key stays on the server

The answer key is every field that reveals or decides a correct answer: `correctOptionId`, `correctOptionIds`, `correctAnswer`, `acceptedAnswers` (including each label's and blank's), `correctPairs`, `correctOrder`, `rubric`, `modelAnswer`, `explanation`, `answerBasis`, `sourceRefs` and `evaluation`.

- The answer key is written only when the quiz is created or edited. Grading reads it and never changes it. A short answer that AI grading accepts is recorded on that attempt only; it is not added to `acceptedAnswers`.
- While a child is taking a quiz, the API sends an **answer-free view** of the quiz: `quizId`, `quizVersion`, `title`, the contexts (`id`, `title`, `text`, `image`), and per question only `id`, `type`, `prompt`, `contextId`, `image`, `options` (text and image, without which are correct), `leftItems`, `rightItems`, `items` (in display order), each label's `id` and `marker`, each blank's `id`, `wordBank`, `points` and `suggestedWordRange`. The same view is used for browsing global quizzes (section 7.1). The answer-free view has its own schema in the contract package, so a field that isn't listed there can't reach the browser by accident.
- All marking happens on the server.

### 7.5 What the child sees after submitting

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
- If the quiz has changed since the attempt, only `score`, `maxScore` and `quizVersion` are shown, with "Quiz was changed after this attempt." (section 7.3), whatever the setting.

### 7.6 Answering descriptive questions: type or upload

For descriptive questions, a child either **types** the answer or **uploads** it as one or more photos or PDFs of a handwritten or printed page. Handwritten and printed uploads are handled the same way.

Whether uploading is allowed is the `allowAnswerUpload` setting (true/false):

- **Account** — in the account's settings. Default for a new account: `false` (uploads not allowed).
- **Quiz** — on the quiz, `true`, `false`, or `null` to use the account value. A quiz-level value overrides the account one.
- When uploading is allowed, the child picks type or upload per question. When it isn't, the child types.
- Changing `allowAnswerUpload` is not a change to the quiz's questions, so it does not increment `quizVersion`.
- The effective value is recorded on the attempt when it starts and applies for that whole attempt, so a child who has already uploaded isn't caught out by a change mid-attempt.

File formats (source files, answer key files, replacement images and child answer uploads): JPEG, PNG, GIF, WebP and PDF are sent to the AI as they are. HEIC/HEIF photos (the iPhone default) are converted to JPEG on the server first. Any other format is rejected with a message listing the accepted ones.

Uploaded answers:

- Several files per answer are allowed, for answers longer than one page (limits in section 7.9).
- The files are stored with the attempt, and deleted if the attempt is discarded because the quiz changed (section 7.3).
- With AI grading, the grading response includes a `transcription` (section 3.4). An unreadable upload sets `needsReview` instead of getting a low mark.
- With `manual` grading, the parent grades from the files; no AI call is made.
- `suggestedWordRange` is only a guide for uploads; it can't be checked before submitting.
- In the result view (section 7.5), "the child's own answer" for an upload is the uploaded files.

### 7.7 Images in questions, options and contexts

The model picks the crop, the server cuts it out once, and the parent can replace or remove it.

- **Cropping:** straight after extraction, while the uploaded files still exist (section 7.2), the server renders the PDF page (or takes the uploaded photo, after HEIC conversion), cuts out `box`, and saves the result in S3, with a `quiz_images` row and a random file name that reveals nothing about the answer.
- **Stored quiz:** a question, option or context stores the image (`imageId`) and its `description`, not the box.
- **Review:** the parent checks each image. The review screen reminds them to check that an image doesn't include an answer (e.g. a ticked box on a worksheet with an answer key). For each image the parent can:
  - **Replace** it by uploading an image file they cropped themselves (a screenshot, a photo or an edited picture). The same formats as other uploads, with HEIC converted to JPEG; resized by the server and saved with a new random name. The `description` is kept, and the parent can edit it.
  - **Remove** it.
- Parents can't **add** an image where there isn't one in v1 (section 9). A `diagram_labelling` question must keep its image, so its image can be replaced but not removed.
- Any change to an image is a change to the question, so it increments `quizVersion` and follows the usual edit rules (section 7.2).
- **Children** see question, option and context images in the answer-free view, and in the result view where the question text is shown (`score_results_and_answers`).

### 7.8 Settings summary

| Setting | Set by | Default | Quiz-level override | Applied |
|---|---|---|---|---|
| `resultVisibility` | Each account | `score_and_results` | Yes | Whenever results are viewed |
| `descriptiveEvaluationDefault` | Each account | `ai_auto` | Yes | Copied into each new descriptive question |
| `allowAnswerUpload` | Each account | `false` | Yes | Fixed for an attempt when it starts |
| `maxQuestionsPerQuiz` | The app owner, app-wide | 75 | **No** | Checked at extraction and whenever a quiz is saved. Can't exceed the hard limit of 100 (section 7.9) |

The three account settings start at the defaults above when an account is created, and the parent can change them. Global quizzes have no account; their quiz-level values are copied with them (section 7.1), and a `null` value means the copying account's setting.

None of these settings increments `quizVersion`.

Lowering `maxQuestionsPerQuiz` below a quiz's current number of questions doesn't delete anything; the quiz just can't gain more questions until it is back under the limit.

### 7.9 Upload limits

These are operator limits in the server config, not parent settings, so they can be changed without a code change. They are starting points, to be adjusted once real usage (the token counts each AI response reports) shows whether they are too strict or too loose.

Source files, per quiz:

| Limit | Value |
|---|---|
| Files | 10 |
| PDF size | 50 MB per file |
| Image size | 20 MB per file, before HEIC conversion and resizing |
| Total size | 100 MB (the API accepts at most 128 MB per request) |
| Pages | 100 in total; each image counts as 1 page |
| Input tokens | 500,000, counted with the free token-counting endpoint before extraction |

Replacement images (section 7.7): 5 MB per image, before HEIC conversion and resizing.

Questions per quiz: hard limit of **100**. The app-wide `maxQuestionsPerQuiz` setting (section 7.8) can be set to any value up to this, and the server rejects anything higher. Parents who need more questions create another quiz. The limit is chosen so that a full extraction fits in one AI response (at most 128K output tokens, including thinking); this is to be confirmed by measuring real extractions.

Child answer uploads, per descriptive question:

| Limit | Value |
|---|---|
| Files | 5 |
| Size | 20 MB per file |
| PDF pages | 5 in total |

- The file, size and page limits are checked as soon as the upload arrives, so the user gets a clear message straight away.
- The token limit is checked after upload and before the extraction call. If it is exceeded, no AI call is made and the parent is asked to upload less.
- Images are resized on the server to the resolution the AI actually uses before they are sent. The exact target size is to be confirmed against the API docs when building.

## 8. Versioning

- `contractVersion` is a fixed value in every request and response.
- Any change that could break an existing reader (removing or renaming a field, narrowing allowed values) means a new version, `"2"`, with its own schema file. The app keeps a reader for every version it has stored.
- Adding an optional field with a safe default can stay in the same version.
- `contractVersion` is the version of the **contract's shape**. It is unrelated to `quizVersion` (section 7.2), which counts edits to one quiz.

## 9. Future enhancements (not in v1)

1. **Generating new questions** — v1 only extracts questions that already exist in the uploaded files. A later version could also write new questions from study material (notes, textbook chapters). The design discussed so far:
   - **Auto mode:** the model decides per source whether to extract, generate or both, so a chapter followed by end-of-chapter questions gets its existing questions extracted and new ones added.
   - **Contract additions:** each question gets an `origin` (`extracted` \| `generated`), and `answerBasis` gains `source_text`: a generated question's answer must be backed by a quoted passage from the source. The request gains generation options: how many questions to generate (never more than what's left under `maxQuestionsPerQuiz`) and their difficulty. `contentKind` would distinguish study material from plain unusable sources.
   - **Review:** generated questions get a "Generated" badge. Whether the question is fair and the answer right stays the parent's judgment in review.
   - **Images:** generated questions could reuse diagrams from the source, but the model can't draw new images.
   These are additions, so they can come as a new contract version without breaking quizzes created in v1.
2. **Regenerating a single question during review** — the parent edits or deletes the question by hand in v1. It can be added later as an extra request shape in contract A without changing the existing one.
3. **Adding images** — adding a picture to a question or option that has none. In v1 a parent can only replace or remove the images the model picked (section 7.7).
4. **Teachers, schools and classes** — new account types beyond parents and children. Teachers and students would take the parent and child roles (section 7.1), with classes in place of child profiles.
5. **Parent PIN** — a PIN to leave a child profile or open the parent view, so a child can't see answers or other children's results (section 7.1).
6. **Child sign-in** — older children get their own login and use Quizzie on their own device:
   - The parent decides which children get a login, and creates and owns it.
   - The login is linked to the child's existing profile (`profileId`), so their quizzes, attempts and results are already there.
   - A child login can only see the quizzes given to that child, take them in the answer-free view, and see their own results as allowed by `resultVisibility`. The API enforces this from the token's role, so a child can never reach the parent view.
   - Younger children who share the parent's device keep using profile switching.
   - How a child signs in without an email address is to be decided. Options:
     - **Username and password set by the parent** (preferred so far): no email needed; the parent resets it. Supabase password sign-in needs an email or phone, so Quizzie would give Supabase an internal address that never receives email.
     - **The child's own email, invited by the parent:** normal password resets, but only for children who have an email address.
     - **Sign-in code or QR code from the parent's device:** easiest for young children, but more custom work.
   - Before this ships, check the children's privacy rules that apply, such as COPPA in the US and Australia's Children's Online Privacy Code under the Privacy Act.
