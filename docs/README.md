# Quizzie design docs

Status: **draft, ready for full review**. Nothing is implemented yet.

In v1 a **parent** makes and manages quizzes, a **child** takes them, and the **app owner** makes the global quizzes anyone can browse. [Accounts, profiles and quiz ownership](data-and-accounts.md#accounts-profiles-and-quiz-ownership) has the details.

| Document | What it covers |
|---|---|
| [AI contract](ai-contract.md) | The data exchanged with the AI model: contract A (quiz extraction), B (answer grading) and C (answer filling), how they are enforced, and versioning |
| [App contract](app-contract.md) | The data exchanged between the web app and the API: the answer-free view, answer shapes, text limits, submission and the result view |
| [Product rules](product-rules.md) | Evaluation settings and scoring, descriptive answers, images and settings |
| [Data and accounts](data-and-accounts.md) | Roles, accounts, global quizzes and copies, personal details, database tables, quizzes and attempts |
| [Architecture](architecture.md) | Tech stack, AWS hosting, sign-in with Supabase Auth, temporary uploads and upload limits |
| [Roadmap](roadmap.md) | Future enhancements beyond v1 |

## Still open

- which sign-in methods parents get ([Signing in](architecture.md#signing-in))
- how long attempts and their answers are kept: every answer to every question in every attempt is stored, so a retention rule (e.g. archive or delete after two years) is worth deciding before launch, also as a privacy commitment for children's data ([Attempts](data-and-accounts.md#attempts))
- whether a child's answers are saved as they go, so a closed tab or a flat battery doesn't lose the attempt, or only sent at submit. Saving as they go would use the same answer shapes ([Answer shapes](app-contract.md#answer-shapes))
- which AWS service runs the API: App Runner or ECS Fargate ([Hosting](architecture.md#hosting))
- the remaining stack choices: database access library, PDF and image processing, UI component library, testing tools and infrastructure as code ([Tech stack](architecture.md#tech-stack))
