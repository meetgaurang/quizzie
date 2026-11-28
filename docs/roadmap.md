# Roadmap: future enhancements (not in v1)

1. **Generating new questions** — v1 only extracts questions that already exist in the uploaded files. A later version could also write new questions from study material (notes, textbook chapters). The design discussed so far:
   - **Auto mode:** the model decides per source whether to extract, generate or both, so a chapter followed by end-of-chapter questions gets its existing questions extracted and new ones added.
   - **Contract additions:** each question gets an `origin` (`extracted` \| `generated`), and `answerBasis` gains `source_text`: a generated question's answer must be backed by a quoted passage from the source. The request gains generation options: how many questions to generate (never more than what's left under `maxQuestionsPerQuiz`) and their difficulty. `contentKind` would distinguish study material from plain unusable sources.
   - **Review:** generated questions get a "Generated" badge. Whether the question is fair and the answer right stays the parent's judgment in review.
   - **Images:** generated questions could reuse diagrams from the source, but the model can't draw new images.
   These are additions, so they can come as a new contract version without breaking quizzes created in v1.
2. **Regenerating a single question during review** — the parent edits or deletes the question by hand in v1. It can be added later as an extra request shape in contract A without changing the existing one.
3. **Adding images** — adding a picture to a question or option that has none. In v1 a parent can only replace or remove the images the model picked ([Images in questions, options and contexts](product-rules.md#images-in-questions-options-and-contexts)).
4. **Teachers, schools and classes** — new account types beyond parents and children. Teachers and students would take the parent and child roles ([Accounts, profiles and quiz ownership](data-and-accounts.md#accounts-profiles-and-quiz-ownership)), with classes in place of child profiles.
5. **Parent PIN** — a PIN to leave a child profile or open the parent view, so a child can't see answers or other children's results ([Accounts, profiles and quiz ownership](data-and-accounts.md#accounts-profiles-and-quiz-ownership)).
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
