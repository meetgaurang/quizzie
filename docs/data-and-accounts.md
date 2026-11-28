# Data and accounts

App data: roles, accounts, quizzes and attempts, and the database tables. This is not part of the [AI contract](ai-contract.md). Status and open items: [README](README.md).

## Tables

| Table | Holds |
|---|---|
| `accounts` | One row per parent account: `authUserId` (the Supabase Auth user id), the parent's details, `isAppOwner`, the account settings |
| `child_profiles` | One row per child profile |
| `quizzes` | One row per quiz. `contexts` and `questions` are `jsonb` columns, checked by the contract's Zod schemas |
| `quiz_assignments` | Which children each quiz is given to |
| `quiz_uploads` | A record of each uploaded file (the files themselves are temporary, [Uploaded files are temporary](architecture.md#uploaded-files-are-temporary)) |
| `quiz_images` | One row per image; the image file itself is in S3 |
| `ai_responses` | The AI's original extraction and answer-filling responses, unchanged, in a `jsonb` column |
| `attempts` | One row per attempt |
| `attempt_answers` | One row per question in an attempt, including skipped ones ([Attempts](#attempts)) |
| `answer_uploads` | One row per file a child uploads for a descriptive answer |
| `app_settings` | App-wide values set by the app owner, such as `maxQuestionsPerQuiz` |

- Document-shaped data (questions, contexts, answers, AI responses) is stored in `jsonb` columns. Everything Quizzie filters or joins on (owner, status, version, grading status) is an ordinary column.
- Image and upload files are not stored in the database; their rows hold their S3 location.
- All access goes through the Quizzie API ([Signing in](architecture.md#signing-in)).

## Accounts, profiles and quiz ownership

Who's who in v1:

| Role | Who | What they can do |
|---|---|---|
| **App owner** | Whoever runs Quizzie | Makes global quizzes the same way a parent makes their own quizzes: upload, review, fill answers, publish. Sets the app-wide limits |
| **Parent** | The account holder, and the account's only login | Creates quizzes from their own uploads, copies global quizzes into the account, edits, fills answers, grades, chooses which children get each quiz, and sees every result. Doesn't take quizzes |
| **Child** | A profile inside the parent's account, with no login, email or password | Takes the quizzes given to them, and sees their own results as allowed by `resultVisibility` ([What the child sees after submitting](app-contract.md#what-the-child-sees-after-submitting)) |
| **Visitor** | Anyone, signed in or not | Browses global quizzes |

Wherever this document says the parent does something to a quiz, the app owner does the same to a global quiz. Teachers, schools and classes come later ([Roadmap](roadmap.md)) and will take the parent and child roles.

**Account:** `accountId`, `authUserId` (the parent's Supabase Auth user), the parent's details, `profiles[]` and the account settings ([Settings summary](product-rules.md#settings-summary)).

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

A child profile can later be linked to its own login ([Roadmap](roadmap.md)); the data model doesn't change, because the login points to the existing `profileId`.

**Switching profiles** needs no PIN or password in v1. **Known limitation:** a child using the device can switch to the parent view and see answers, rubrics and other children's results. A parent PIN is listed for later ([Roadmap](roadmap.md)).

**Quiz ownership.** Every quiz has an `owner`: `{kind: "app"}` for a global quiz, or `{kind: "account", accountId}`.

| | Global quiz | Account quiz |
|---|---|---|
| Made by | The app owner only, so the app owner controls what copyrighted material is published | The parent, from their own uploads, or copied from a global quiz |
| Who can see it | Anyone, as the answer-free view ([Answer key stays on the server](app-contract.md#answer-key-stays-on-the-server)): the questions, never the answers | Only that account |
| Attempts | None; it can't be submitted | By the children it is given to |

- A parent's uploads and quizzes are private to their account and never become global.
- The API checks every request for an account quiz, its attempts or its results against the signed-in account.

**Browsing global quizzes.** Only published global quizzes are listed. Anyone can browse them without signing in, in the answer-free view, with no way to submit. Each has an **Assign to my account** button; a visitor who isn't signed in is asked to sign in or create an account first.

**Assign to my account** makes a copy:

- The copy is a new quiz with a new `quizId`, owned by the account, at `quizVersion` 1 and `published`, because a global quiz can only be published once every question has an answer.
- **Copied:** the title, the contexts, the questions with all their answer fields, `answerBasis`, `evaluation` and `points`, the images (as new files with new random names), and the quiz-level settings.
- **Not copied:** the AI responses (`ai_responses`), and the `sourceRefs` of questions, contexts and answers. They refer to the app owner's uploads, which the parent never had, so they stay with the global quiz. There are no source files to copy, because uploaded files are deleted after extraction ([Uploaded files are temporary](architecture.md#uploaded-files-are-temporary)).
- The copy records `copiedFrom: {quizId, quizVersion}`.
- The copy is fully editable, like any account quiz. Editing it never changes the global quiz, and later edits to the global quiz never reach copies. Unpublishing or deleting a global quiz doesn't affect existing copies either.
- Assigning the same global quiz again makes another, separate copy. The browse page shows when a quiz is already in the account.

**Giving a quiz to children.** An account quiz has `assignedProfileIds[]`, the children it's given to; it starts empty, including for a copy.

- A child sees only published quizzes given to them.
- A child can take a quiz as many times as they like. Each attempt is kept with its own score.
- Taking a child off a quiz stops new attempts; their earlier attempts and results are kept.
- Changing who a quiz is given to doesn't change its questions, so it doesn't increment `quizVersion`.

## Quizzes

There is one copy of each quiz. It carries a `quizVersion` number, and every saved change to the quiz increments it. Earlier versions are not kept.

Changes to a quiz can't be saved while any of its attempts has a pending grade: an AI grading call still running, an `ai_suggest` grade waiting for the parent, a `manual` descriptive answer not yet graded, or an AI grade with `needsReview`. The editor tells the parent how many grades are pending and links to them.

A quiz (a row in `quizzes`):

| Field | Notes |
|---|---|
| `quizId`, `title`, `createdAt`, `updatedAt` | |
| `owner` | `{kind: "app"}` (global) or `{kind: "account", accountId}` ([Accounts, profiles and quiz ownership](#accounts-profiles-and-quiz-ownership)) |
| `copiedFrom` | `{quizId, quizVersion}` of the global quiz it was copied from, or `null` |
| `assignedProfileIds` | The children the quiz is given to. Account quizzes only |
| `quizVersion` | Starts at 1; +1 on every saved change |
| `status` | `draft` → `published`. Publishing is blocked while any question has `answerBasis: "none"` |
| `contexts[]` | The shared contexts as they are now ([Question fields every type has](ai-contract.md#question-fields-every-type-has)), each with its image |
| `questions[]` | The questions as they are now, each with `fromAiQuestionId`, `contextId`, `answerBasis` (`answer_key` \| `ai_solved` \| `manual_entry` \| `none`, [Applying the answers](ai-contract.md#applying-the-answers)), `editedAfterDraft: boolean`, `evaluation`, `points` |

Points per question are set by the app, not the AI, when the question is created:

| Question type | Default `points` |
|---|---|
| `single_choice`, `multi_select`, `true_false`, `short_answer`, `matching`, `ordering`, `diagram_labelling`, `fill_in_blanks` | 1 |
| `descriptive` | The rubric total (sum of the criteria's `maxPoints`). `null` until the question has a rubric |

The parent can change points in review or when editing. For a descriptive question, `points` always equals its rubric total: to make it worth more, the parent changes the criteria's `maxPoints`, so grades never need scaling.

The responses in `ai_responses` are the AI's output, not copies of the quiz. Each row also records the model, the token counts, when the call ran and whether it succeeded. They are written once and let us see what the AI produced versus what the parent changed, and later measure extraction quality.

## Attempts

| Field | Notes |
|---|---|
| `attemptId`, `quizId` | |
| `profileId` | The child who made the attempt. Only a child the quiz is given to can start one ([Accounts, profiles and quiz ownership](#accounts-profiles-and-quiz-ownership)) |
| `quizVersion` | The quiz's version when the attempt started. It must still match at submit (see below) |
| `startedAt`, `submittedAt` | |
| `answers[]` | Per question, including skipped ones: `questionId`, the child's `answer` (see [Answer shapes](app-contract.md#answer-shapes)), how it was graded (`automatic`, `accepted_answer`, `ai` or `parent`), `gradingStatus` (`graded` \| `pending`), `result`, `pointsAwarded`, `maxPoints` at the time of grading, and any AI grading response (contract B) |
| `score`, `maxScore` | Calculated by the server from the answers |

The shape of each answer, the submission format and the text limits are in the app contract ([Answer shapes](app-contract.md#answer-shapes)).

- If the quiz changed while the attempt was in progress (its `quizVersion` no longer matches), the submission is rejected with "Quiz was changed, please start again" and the in-progress attempt is discarded.
- The mark of every attempt is shown against its `quizVersion`.
- Submitted attempts are **not** re-graded when the quiz changes. Points are stored on the attempt, so its score stays the same whatever happens to the quiz later.
- An attempt's answers can only be viewed while its `quizVersion` matches the quiz's current `quizVersion`. Otherwise its details are not shown; the UI shows its mark and version with the message "Quiz was changed after this attempt."
