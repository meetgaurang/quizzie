# Architecture

What Quizzie is built with, where it runs, how sign-in works, and the limits on uploads. Status and open items: [README](README.md).

## Tech stack

| Part | Choice |
|---|---|
| Language | TypeScript throughout |
| Web app | TanStack Start in SPA mode |
| API | Node.js with Fastify |
| Shared contract | `packages/contract`: Zod schemas for the [AI contract](ai-contract.md) and the [app contract](app-contract.md). JSON Schema files are generated from them and committed, and CI fails if they drift |
| Repository layout | A pnpm workspace: `apps/web`, `apps/api`, `packages/contract` |
| AI | The Claude API through the official Anthropic TypeScript SDK, with the model `claude-opus-5-5` and structured outputs |
| Sign-in | Supabase Auth; the API verifies tokens with `jose` ([Signing in](#signing-in)) |
| Database | PostgreSQL on AWS RDS ([Hosting](#hosting)) |
| Files, jobs and hosting | AWS: S3, SQS, CloudFront, and App Runner or ECS Fargate for the API ([Hosting](#hosting)) |

Still to choose:

- **Database access library**: Drizzle is the suggestion, so the TypeScript types match the tables
- **PDF rendering and image processing**: rendering PDF pages and cropping images, resizing, and converting HEIC to JPEG
- **UI component library** for the web app
- **Testing tools**
- **Infrastructure as code** for the AWS resources: AWS CDK or Terraform
- **App Runner or ECS Fargate** for the API

## Hosting

Supabase is used for sign-in only ([Signing in](#signing-in)). Everything else runs in Quizzie's AWS account, in the Sydney region (`ap-southeast-2`), so all quiz, attempt and child data stays in Australia:

| Part | Where |
|---|---|
| Sign-in (parents' emails and passwords) | Supabase Auth, in a Supabase project in the Sydney region, on a paid plan for production because free projects pause when idle |
| Web app (the SPA's static files) | AWS: S3 and CloudFront |
| API (Fastify) | AWS: containers on App Runner or ECS Fargate (to be chosen) |
| AI jobs (extraction, answer filling, grading) | AWS: an SQS queue and a worker. The API answers straight away and the web app checks back, because an AI call can take longer than a web request should wait |
| Database | AWS: RDS for PostgreSQL, starting on one `db.t4g.micro` instance in a single availability zone. A reserved instance and a standby in a second zone can be added once usage is steady |
| Images and uploaded files | AWS: S3, in private buckets. The API hands the browser short-lived links only to files the signed-in user may see. A lifecycle rule deletes leftover temporary uploads after one day, as a safety net for [Uploaded files are temporary](#uploaded-files-are-temporary) |

The database is in a private network next to the API and is not reachable from the internet. The API reaches the database and S3 through AWS permission roles, not stored passwords or keys.

## Signing in

**Signing in** uses Supabase Auth:

- The web app signs the parent in with Supabase and sends the access token (a JWT) with every API request.
- The API verifies the token itself against Supabase's published signing keys (the project uses asymmetric JWT signing keys), then finds the account from the token's user id.
- **The account is created on first sign-in.** Supabase and Quizzie's database are separate, so when the API sees a valid token for a user it has no account for, it creates the `accounts` row and asks the parent for their details. The database can't enforce that `authUserId` matches a real Supabase user; the API does.
- **Deleting an account** removes the Supabase user and all of the account's Quizzie data (profiles, quizzes, attempts, answers, files) in one step run by the API.
- Supabase stores the parent's email and password; Quizzie never handles passwords. Child profiles are Quizzie data only, so Supabase holds no information about children.
- The app owner's role is a flag on the app owner's account, checked by the API.
- All quiz data goes through the Quizzie API; the browser talks to Supabase only to sign in, so the server stays in control of what reaches the browser ([Answer key stays on the server](app-contract.md#answer-key-stays-on-the-server)).
- Which sign-in methods parents get in v1 (email and password, an emailed link, Google) is not decided yet.

## Uploaded files are temporary

Source files and answer key files are kept only until the AI call that uses them has finished and, for source files, the images have been cropped. Then they are deleted, and only their row in `quiz_uploads` remains: filename, kind (`source` or `answer_key`), type, size and pages. The parent never needs to upload them again: images are replaced by uploading a new image ([Images in questions, options and contexts](product-rules.md#images-in-questions-options-and-contexts)), and answer filling and grading work from the quiz JSON ([Question fields every type has](ai-contract.md#question-fields-every-type-has)). If extraction fails, the files are deleted too, and the parent uploads them again to retry.

## Upload limits

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

Replacement images ([Images in questions, options and contexts](product-rules.md#images-in-questions-options-and-contexts)): 5 MB per image, before HEIC conversion and resizing.

Questions per quiz: hard limit of **100**. The app-wide `maxQuestionsPerQuiz` setting ([Settings summary](product-rules.md#settings-summary)) can be set to any value up to this, and the server rejects anything higher. Parents who need more questions create another quiz. The limit is chosen so that a full extraction fits in one AI response (at most 128K output tokens, including thinking); this is to be confirmed by measuring real extractions.

Child answer uploads, per descriptive question:

| Limit | Value |
|---|---|
| Files | 5 |
| Size | 20 MB per file |
| PDF pages | 5 in total |

- The file, size and page limits are checked as soon as the upload arrives, so the user gets a clear message straight away.
- The token limit is checked after upload and before the extraction call. If it is exceeded, no AI call is made and the parent is asked to upload less.
- Images are resized on the server to the resolution the AI actually uses before they are sent. The exact target size is to be confirmed against the API docs when building.
