# Product rules

How quizzes behave for parents and children: grading, what children see, descriptive answers, images and settings. Status and open items: [README](README.md).

## Evaluation setting and scoring

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
- **Descriptive default setting** (`descriptiveEvaluationDefault`) exists at two levels, like `resultVisibility` ([What the child sees after submitting](app-contract.md#what-the-child-sees-after-submitting)):
  - **Account** — in the account's settings. Default for a new account: `ai_auto`.
  - **Quiz** — on the quiz, one of the three descriptive values or `null` to use the account value. A quiz-level value overrides the account one. It can be chosen at upload, before the AI draft arrives.
  - The default is **copied** into a descriptive question's `evaluation` when the question is created (AI draft or added by the parent). After that the question keeps its own value, so changing a default never changes existing questions or the grading of attempts already submitted.
  - Changing the quiz-level default affects only descriptive questions created after the change. To change an existing descriptive question, the parent edits that question.
- **The AI extraction response does not include `evaluation`.** How to grade is the parent's decision. The app fills in the defaults by question type when a draft arrives, and the parent changes them in review.

## Answering descriptive questions: type or upload

For descriptive questions, a child either **types** the answer or **uploads** it as one or more photos or PDFs of a handwritten or printed page. Handwritten and printed uploads are handled the same way.

Whether uploading is allowed is the `allowAnswerUpload` setting (true/false):

- **Account** — in the account's settings. Default for a new account: `false` (uploads not allowed).
- **Quiz** — on the quiz, `true`, `false`, or `null` to use the account value. A quiz-level value overrides the account one.
- When uploading is allowed, the child picks type or upload per question. When it isn't, the child types.
- Changing `allowAnswerUpload` is not a change to the quiz's questions, so it does not increment `quizVersion`.
- The effective value is recorded on the attempt when it starts and applies for that whole attempt, so a child who has already uploaded isn't caught out by a change mid-attempt.

File formats (source files, answer key files, replacement images and child answer uploads): JPEG, PNG, GIF, WebP and PDF are sent to the AI as they are. HEIC/HEIF photos (the iPhone default) are converted to JPEG on the server first. Any other format is rejected with a message listing the accepted ones.

Uploaded answers:

- Several files per answer are allowed, for answers longer than one page (limits in [Upload limits](architecture.md#upload-limits)).
- The files are stored with the attempt, and deleted if the attempt is discarded because the quiz changed ([Attempts](data-and-accounts.md#attempts)).
- With AI grading, the grading response includes a `transcription` ([Grading response for a descriptive answer](ai-contract.md#grading-response-for-a-descriptive-answer)). An unreadable upload sets `needsReview` instead of getting a low mark.
- With `manual` grading, the parent grades from the files; no AI call is made.
- `suggestedWordRange` is only a guide for uploads; it can't be checked before submitting.
- In the result view ([What the child sees after submitting](app-contract.md#what-the-child-sees-after-submitting)), "the child's own answer" for an upload is the uploaded files.

## Images in questions, options and contexts

The model picks the crop, the server cuts it out once, and the parent can replace or remove it.

- **Cropping:** straight after extraction, while the uploaded files still exist ([Uploaded files are temporary](architecture.md#uploaded-files-are-temporary)), the server renders the PDF page (or takes the uploaded photo, after HEIC conversion), cuts out `box`, and saves the result in S3, with a `quiz_images` row and a random file name that reveals nothing about the answer.
- **Stored quiz:** a question, option or context stores the image (`imageId`) and its `description`, not the box.
- **Review:** the parent checks each image. The review screen reminds them to check that an image doesn't include an answer (e.g. a ticked box on a worksheet with an answer key). For each image the parent can:
  - **Replace** it by uploading an image file they cropped themselves (a screenshot, a photo or an edited picture). The same formats as other uploads, with HEIC converted to JPEG; resized by the server and saved with a new random name. The `description` is kept, and the parent can edit it.
  - **Remove** it.
- Parents can't **add** an image where there isn't one in v1 ([Roadmap](roadmap.md)). A `diagram_labelling` question must keep its image, so its image can be replaced but not removed.
- Any change to an image is a change to the question, so it increments `quizVersion` and follows the usual edit rules ([Quizzes](data-and-accounts.md#quizzes)).
- **Children** see question, option and context images in the answer-free view, and in the result view where the question text is shown (`score_results_and_answers`).

## Settings summary

| Setting | Set by | Default | Quiz-level override | Applied |
|---|---|---|---|---|
| `resultVisibility` | Each account | `score_and_results` | Yes | Whenever results are viewed |
| `descriptiveEvaluationDefault` | Each account | `ai_auto` | Yes | Copied into each new descriptive question |
| `allowAnswerUpload` | Each account | `false` | Yes | Fixed for an attempt when it starts |
| `maxQuestionsPerQuiz` | The app owner, app-wide | 75 | **No** | Checked at extraction and whenever a quiz is saved. Can't exceed the hard limit of 100 ([Upload limits](architecture.md#upload-limits)) |

The three account settings start at the defaults above when an account is created, and the parent can change them. Global quizzes have no account; their quiz-level values are copied with them ([Accounts, profiles and quiz ownership](data-and-accounts.md#accounts-profiles-and-quiz-ownership)), and a `null` value means the copying account's setting.

None of these settings increments `quizVersion`.

Lowering `maxQuestionsPerQuiz` below a quiz's current number of questions doesn't delete anything; the quiz just can't gain more questions until it is back under the limit.
