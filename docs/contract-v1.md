# Quizzie AI contract — v1 (draft for review)

Status: **draft, ready for full review**. Nothing is implemented yet. Still open: the database (section 7) and how parents sign in (section 7.1). Future enhancements beyond v1 are listed at the end.

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
```

- **The API** sends the generated JSON Schema to Claude as the required output format (structured outputs), so the model's reply has to match the shape.
- **The API** then parses the reply with the Zod schema. That also runs the rules JSON Schema can't express (section 6).
- **The SPA** imports the TypeScript types to render quizzes and the review screen.
- **The committed JSON Schema files** are the language-neutral copy of the contract. CI fails if they drift from the Zod source.

The AI's output is **not** the stored quiz. The stored quiz (section 7) wraps the AI output and adds things the AI never produces: server checks, parent edits, publish state. That keeps the contract about the AI only.

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
| `questions` | `Question[]` | At least 1 when `ok`, empty when `insufficient_content` |

`SourceAssessment` — what the model decided about each file:

| Field | Type | Notes |
|---|---|---|
| `sourceId` | string | |
| `contentKind` | `"questions"` \| `"no_questions"` \| `"unusable"` | `questions`: has questions to extract. `no_questions`: readable, but contains no questions (e.g. plain notes). `unusable`: can't be read, or not in English. If no source is `questions`, the result is `insufficient_content` |
| `hasAnswerKey` | boolean | Whether the source contains answers to its own questions |
| `handwriting` | `"none"` \| `"partial"` \| `"mostly"` | How much of the source is handwritten. The review screen warns the parent to check questions from handwritten sources closely |
| `notes` | string \| null | e.g. "Page 3 is cut off" |

Handwritten source files are supported on a best-effort basis: the model reads what it can, describes unreadable parts in `notes`, and returns `insufficient_content` if nothing usable is left. Handwriting is usually photographed, so its quotes are `not_checkable` (section 6).

**Language:** v1 supports English only. Quizzes are always written in English. A source that is not in English is assessed as `unusable` with a note saying so; if no usable source is left, the result is `insufficient_content` with a reason such as "The document is not in English". An English source with occasional words in another language is fine. The app's own interface is also English.

### 2.3 Question — fields every type has

| Field | Type | Notes |
|---|---|---|
| `id` | string | `"q1"`, `"q2"`, … unique within the result |
| `type` | `QuestionType` | `single_choice` \| `multi_select` \| `true_false` \| `short_answer` \| `descriptive` \| `matching` \| `ordering` \| `diagram_labelling` \| `fill_in_blanks` |
| `sourceNumber` | string \| null | The question's number as printed, e.g. `"5"` or `"3(b)"`. Answer keys refer to questions by it (section 4) |
| `answerBasis` | `"answer_key"` \| `"none"` | Whether the sources give this question's answer (table below) |
| `prompt` | string | The question text, as written in the source |
| `sourceRefs` | `SourceRef[]`, 1 or more | Where the question, and its answer if given, appear in the sources |
| `sourceRefs[].sourceId` | string | Must match a request source |
| `sourceRefs[].page` | integer \| null | 1-based page for PDFs, `null` for images |
| `sourceRefs[].quote` | string | Short verbatim passage from the source, checked by the server (section 6) |
| `image` | `ImageRef` \| null | The question's image (a diagram, figure or picture), if it has one (section 2.4) |
| `explanation` | string \| null | Why the answer is correct, only if the source gives one. Parent-only (section 7.5) |

`answerBasis` values:

| `answerBasis` | Meaning | Review badge |
|---|---|---|
| `answer_key` | The sources give the answer (an answer key, or a marking scheme or model answer for descriptive questions). All of the question's answer fields are filled | From answer key |
| `none` | The sources have the question but no answer. All of the question's answer fields are `null` | No answer yet |

**Extraction never works out answers.** Copying an answer from the sources is cheap; solving a question takes reasoning, which costs more, takes longer and can be wrong. A question without an answer in the sources comes back with `answerBasis: "none"`, and the parent then chooses how to fill it in (section 4).

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
| `description` | string | A short description of what the image shows. Used as alt text, and helps the parent check the crop |

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
  "questions": [
    {
      "id": "q1",
      "type": "single_choice",
      "sourceNumber": "1",
      "answerBasis": "answer_key",
      "prompt": "Which organelle carries out photosynthesis?",
      "sourceRefs": [
        { "sourceId": "s1", "page": 6, "quote": "1. Which organelle carries out photosynthesis?" },
        { "sourceId": "s1", "page": 9, "quote": "1. (b) chloroplast" }
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
      "sourceRefs": [
        { "sourceId": "s2", "page": null, "quote": "Q5. Explain why plants need both light and carbon dioxide to make glucose." }
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
| `question` | the stored `short_answer`, `diagram_labelling`, `fill_in_blanks` or `descriptive` question | Includes prompt, accepted answers or rubric, and source quotes. If the question has an image, the image is sent too, so the grader sees what the child saw |
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
| `questions` | `Question[]` | The questions to fill, without their answer fields: `id`, `sourceNumber`, `type`, `prompt`, `image`, options, items, labels, blanks, word banks and `length` |
| `answerFiles` | `Source[]` | `answer_key` mode only: the uploaded answer key files, with their own `sourceId`s. Sent as document and image content blocks |
| `sources` | `Source[]` | The quiz's original source files. Sent in `ai_solve` mode, because a question may depend on a passage or figure elsewhere in the source. Not sent in `answer_key` mode |

- **`answer_key` mode** sends all the questions in one call: it is copying, not solving.
- **`ai_solve` mode** sends the questions in batches (batch size in server config, e.g. 20), so each response stays well within the output limit. The source files are the same in every batch, so they are sent with prompt caching, which makes the repeats much cheaper than the first send.

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
| `sourceRefs` | `SourceRef[]` | `answer_key` mode: where the answer is in the answer key, checked like other quotes. `ai_solve` mode: empty |
| `reason` | string \| null | Required when `status` is not `filled` |

### 4.3 Applying the answers

The stored question's `answerBasis` records where its answer came from: `answer_key` (from the original sources or an uploaded key), `ai_solved`, `manual_entry` (typed by hand) or `none`. Review badges: "From answer key", "AI answer", "Entered manually", "No answer yet".

- **An answer key overrides AI answers.** If the parent asked the AI first and then uploads a key, the key's answers replace the AI's, and the parent sees each difference, e.g. "Q4: answer key says B; AI answered C".
- **Answers the parent typed are never overwritten.** Where the key disagrees, the parent sees both and picks one.
- **AI never overwrites an answer from a key or the parent;** it is only offered for questions with no answer.
- **Descriptive questions:** the filled rubric sets the question's `points` (rubric total, section 7.2).
- **Answer key files** count toward the quiz's upload limits (section 7.9) and are kept with its source files.
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
| `automatic` | Compared with the correct option(s), true/false answer, pairs or order. Multi-select, matching and ordering can earn partial credit (below) |
| `accepted_answers_only` | Matched against `acceptedAnswers` (per label or blank, for diagram labelling and fill in the blanks), ignoring case, surrounding spaces and repeated spaces. With a `wordBank`, the child picks a word, so it's an exact match |
| `accepted_answers_then_ai` | Same match first; if there's no match, contract B decides |
| `ai_auto` | AI grade goes straight to the child, unless `needsReview` is true, in which case it waits for the parent |
| `ai_suggest` | AI grade is a suggestion; the parent accepts or changes it before the child sees it |
| `manual` | No AI call; the parent grades with the rubric |

Scoring, as a share of the question's `points`:

| Type | Full points | Half points | 0 |
|---|---|---|---|
| `single_choice`, `true_false`, `short_answer` | Correct | — | Anything else |
| `multi_select` | Exactly the correct options picked | At least one correct option picked, but not exactly the correct set | No correct option picked (including nothing picked) |
| `matching` | Every pair correct | At least half the pairs correct, but not all | Fewer than half the pairs correct |
| `ordering` | Every item in its correct position | At least half the items in their correct position, but not all | Fewer than half in their correct position |
| `diagram_labelling` | Every label correct | At least half the labels correct, but not all | Fewer than half the labels correct |
| `fill_in_blanks` | Every blank correct | At least half the blanks correct, but not all | Fewer than half the blanks correct |

Unanswered pairs, positions, labels and blanks count as wrong. "At least half" means correct × 2 ≥ total, so 2 of 4 or 2 of 3 is half points, and 1 of 3 is 0. The threshold is there so that guessing doesn't earn half points: a random arrangement usually gets one or two items right by chance.

Picking every option of a multi-select question scores 0, unless every option is correct. Otherwise picking everything would always earn half points.

- **Whether a question needs AI evaluation** follows from its `evaluation` value (`accepted_answers_then_ai`, `ai_auto`, `ai_suggest`). There is no separate flag to keep in sync.
- **Whether a quiz needs AI evaluation** is calculated: true if any of its questions does. It is not stored.
- **Descriptive default setting** (`descriptiveEvaluationDefault`) exists at two levels, like `resultVisibility` (section 7.5):
  - **Account** — in the account's settings. Default for a new account: `ai_auto`.
  - **Quiz** — on `quiz.json`, one of the three descriptive values or `null` to use the account value. A quiz-level value overrides the account one. It can be chosen at upload, before the AI draft arrives.
  - The default is **copied** into a descriptive question's `evaluation` when the question is created (AI draft or added by the parent). After that the question keeps its own value, so changing a default never changes existing questions or the grading of attempts already submitted.
  - Changing the quiz-level default affects only descriptive questions created after the change. To change an existing descriptive question, the parent edits that question.
- **The AI extraction response does not include `evaluation`.** How to grade is the parent's decision. The app fills in the defaults by question type when a draft arrives, and the parent changes them in review.

## 6. How the contract is enforced

Four layers, in order:

1. **Structured outputs.** Claude is given the JSON Schema, so the reply has the right fields and types.
2. **Zod parsing on the server**, including rules JSON Schema can't express:
   - `status: ok` has at least one question; `insufficient_content` has none and has a `reason`
   - question ids are unique; option ids are unique within a question
   - `correctOptionId` / `correctOptionIds` refer to existing options
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
3. **Quote check.** For PDFs with a text layer, the server confirms each `quote` appears in the stated page (normalising whitespace and hyphenation). Each question gets `verification: "verified" | "mismatch" | "not_checkable"`. Images and scanned PDFs are `not_checkable` in v1.
4. **Parent review** before publishing, with the answer badges (section 4.3) and the `verification` result shown on each question.

## 7. Accounts, quizzes and attempts (app data, not part of the AI contract)

Shown here only to make the boundary with the AI contract clear.

Where this data is stored is **not decided yet** (files, MongoDB or PostgreSQL). The file paths below show what is stored, not how it will be stored.

### 7.1 Accounts, profiles and quiz ownership

Who's who in v1:

| Role | Who | What they can do |
|---|---|---|
| **App owner** | Whoever runs Quizzie | Makes global quizzes the same way a parent makes their own quizzes: upload, review, fill answers, publish. Sets the app-wide limits |
| **Parent** | The account holder, and the account's only login | Creates quizzes from their own uploads, copies global quizzes into the account, edits, fills answers, grades, chooses which children get each quiz, and sees every result. Doesn't take quizzes |
| **Child** | A profile inside the parent's account, with no login, email or password | Takes the quizzes given to them, and sees their own results as allowed by `resultVisibility` (section 7.5) |
| **Visitor** | Anyone, signed in or not | Browses global quizzes |

Wherever this document says the parent does something to a quiz, the app owner does the same to a global quiz. Teachers, schools and classes come later (section 9) and will take the parent and child roles.

**Account:** `accountId`, the parent's sign-in details, `profiles[]` (`profileId`, `name`) and the account settings (section 7.8). How parents sign in (email and password, Google, an emailed link) is not decided yet.

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
- **Copied:** the title, the questions with all their answer fields, `answerBasis`, `verification`, `evaluation` and `points`, the cropped images (as new files with new random names), and the quiz-level settings.
- **Not copied:** the source files, `extraction.json` and the answer-filling responses. They stay with the global quiz, so the original copyrighted documents never leave it. Without the source, a copy's image crops can't be adjusted, only removed.
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

```
data/quizzes/<quizId>/quiz.json         the quiz (one copy)
data/quizzes/<quizId>/extraction.json   the AI's original contract A response, unchanged
data/quizzes/<quizId>/answer-filling/   each contract C response, unchanged
data/quizzes/<quizId>/sources/          uploaded source files and answer key files
data/quizzes/<quizId>/images/           cropped question and option images (section 7.7)
```

`quiz.json`:

| Field | Notes |
|---|---|
| `quizId`, `title`, `createdAt`, `updatedAt` | |
| `owner` | `{kind: "app"}` (global) or `{kind: "account", accountId}` (section 7.1) |
| `copiedFrom` | `{quizId, quizVersion}` of the global quiz it was copied from, or `null` |
| `assignedProfileIds` | The children the quiz is given to. Account quizzes only |
| `quizVersion` | Starts at 1; +1 on every saved change |
| `status` | `draft` → `published`. Publishing is blocked while any question has `answerBasis: "none"` |
| `questions[]` | The questions as they are now, each with `fromAiQuestionId`, `answerBasis` (`answer_key` \| `ai_solved` \| `manual_entry` \| `none`, section 4.3), `verification`, `editedAfterDraft: boolean`, `evaluation`, `points` |

Points per question are set by the app, not the AI, when the question is created:

| Question type | Default `points` |
|---|---|
| `single_choice`, `multi_select`, `true_false`, `short_answer`, `matching`, `ordering`, `diagram_labelling`, `fill_in_blanks` | 1 |
| `descriptive` | The rubric total (sum of the criteria's `maxPoints`). `null` until the question has a rubric |

The parent can change points in review or when editing. For a descriptive question, `points` always equals its rubric total: to make it worth more, the parent changes the criteria's `maxPoints`, so grades never need scaling.

`extraction.json` and the `answer-filling/` responses are the AI's output, not copies of the quiz. They are written once and let us see what the AI produced versus what the parent changed, and later measure extraction quality.

### 7.3 Attempts

| Field | Notes |
|---|---|
| `attemptId`, `quizId` | |
| `profileId` | The child who made the attempt. Only a child the quiz is given to can start one (section 7.1) |
| `quizVersion` | The quiz's version when the attempt started. It must still match at submit (see below) |
| `startedAt`, `submittedAt` | |
| `answers[]` | Per question: `questionId`, the child's answer, how it was graded, `pointsAwarded`, `maxPoints` at the time of grading, and any AI grading response (contract B) |
| `score`, `maxScore` | Calculated by the server from the answers |

- If the quiz changed while the attempt was in progress (its `quizVersion` no longer matches), the submission is rejected with "Quiz was changed, please start again" and the in-progress attempt is discarded.
- The mark of every attempt is shown against its `quizVersion`.
- Submitted attempts are **not** re-graded when the quiz changes. Points are stored on the attempt, so its score stays the same whatever happens to the quiz later.
- An attempt's answers can only be viewed while its `quizVersion` matches the quiz's current `quizVersion`. Otherwise its details are not shown; the UI shows its mark and version with the message "Quiz was changed after this attempt."

### 7.4 Answer key stays on the server

The answer key is every field that reveals or decides a correct answer: `correctOptionId`, `correctOptionIds`, `correctAnswer`, `acceptedAnswers` (including each label's and blank's), `correctPairs`, `correctOrder`, `rubric`, `modelAnswer`, `explanation`, `answerBasis`, `sourceRefs`, `verification` and `evaluation`.

- The answer key is written only when the quiz is created or edited. Grading reads it and never changes it. A short answer that AI grading accepts is recorded on that attempt only; it is not added to `acceptedAnswers`.
- While a child is taking a quiz, the API sends an **answer-free view** of the quiz: `quizId`, `quizVersion`, `title`, and per question only `id`, `type`, `prompt`, `image`, `options` (text and image, without which are correct), `leftItems`, `rightItems`, `items` (in display order), each label's `id` and `marker`, each blank's `id`, `wordBank`, `points` and `suggestedWordRange`. The same view is used for browsing global quizzes (section 7.1). The answer-free view has its own schema in the contract package, so a field that isn't listed there can't reach the browser by accident.
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
- **Quiz** — `resultVisibility` on `quiz.json`, either one of the three levels or `null` to use the account value. A quiz-level value overrides the account one.

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
- **Quiz** — on `quiz.json`, `true`, `false`, or `null` to use the account value. A quiz-level value overrides the account one.
- When uploading is allowed, the child picks type or upload per question. When it isn't, the child types.
- Changing `allowAnswerUpload` is not a change to the quiz's questions, so it does not increment `quizVersion`.
- The effective value is recorded on the attempt when it starts and applies for that whole attempt, so a child who has already uploaded isn't caught out by a change mid-attempt.

File formats (parent source files and child answer uploads): JPEG, PNG, GIF, WebP and PDF are sent to the AI as they are. HEIC/HEIF photos (the iPhone default) are converted to JPEG on the server first. Any other format is rejected with a message listing the accepted ones.

Uploaded answers:

- Several files per answer are allowed, for answers longer than one page (limits in section 7.9).
- The files are stored with the attempt, and deleted if the attempt is discarded because the quiz changed (section 7.3).
- With AI grading, the grading response includes a `transcription` (section 3.4). An unreadable upload sets `needsReview` instead of getting a low mark.
- With `manual` grading, the parent grades from the files; no AI call is made.
- `suggestedWordRange` is only a guide for uploads; it can't be checked before submitting.
- In the result view (section 7.5), "the child's own answer" for an upload is the uploaded files.

### 7.7 Images in questions and options

The model picks the crop; the server cuts it out; the parent can adjust it.

- **Cropping:** the server renders the PDF page (or takes the uploaded photo, after HEIC conversion), cuts out `box`, and saves the result under `data/quizzes/<quizId>/images/` with a random file name that reveals nothing about the answer.
- **Stored quiz:** a question or option stores the cropped image (`imageId`) and its `description`, not the box.
- **Source files are kept** with the quiz (`data/quizzes/<quizId>/sources/`) so a crop can be adjusted later. A copy of a global quiz has no source files, so its crops can only be removed (section 7.1).
- **Review:** each crop is shown next to its page. The parent can move or resize the box, or remove the image. The review screen reminds the parent to check that a crop doesn't include an answer (e.g. a ticked box on a worksheet with an answer key).
- Parents can't upload their own images in v1; they can only adjust or remove the crops the model picked (see section 9).
- Any change to an image is a change to the question, so it increments `quizVersion` and follows the usual edit rules (section 7.2).
- **Children** see question and option images in the answer-free view, and in the result view where the question text is shown (`score_results_and_answers`).

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
   - **Review:** generated questions get a "Generated" badge, and the quote check confirms the supporting passage exists. Whether the question is fair and the answer right stays the parent's judgment in review.
   - **Images:** generated questions could reuse diagrams from the source, but the model can't draw new images.
   These are additions, so they can come as a new contract version without breaking quizzes created in v1.
2. **Regenerating a single question during review** — the parent edits or deletes the question by hand in v1. It can be added later as an extra request shape in contract A without changing the existing one.
3. **Parents uploading their own images** — adding a picture to a question or option while editing. In v1 a parent can only adjust or remove the crops the model picked.
4. **Teachers, schools and classes** — new account types beyond parents and children. Teachers and students would take the parent and child roles (section 7.1), with classes in place of child profiles.
5. **Parent PIN** — a PIN to leave a child profile or open the parent view, so a child can't see answers or other children's results (section 7.1).
