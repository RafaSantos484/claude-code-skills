---
name: pr-review
description: Perform a rigorous, evidence-based code review directly against a live git repository the agent can read, run, and search — no exported dump files needed. Use whenever the user asks to review a pull request, review "my changes"/"this branch"/"this diff", check whether code is ready to merge, wants a second opinion before opening a PR, or asks for an architecture/SOLID/best-practices assessment of real on-disk code. Works with a PR number or URL (via the gh CLI), a branch-vs-base comparison, or uncommitted working-tree changes. Trigger on phrases like "review this PR", "review my changes", "is this ready to merge", "check this diff against our conventions" — even without the word "PR".
allowed-tools: Bash(git status *), Bash(git diff *), Bash(git log *), Bash(git show *), Bash(git branch *), Bash(git fetch *), Bash(git remote *), Bash(gh pr view *), Bash(gh pr diff *), Bash(gh pr checks *), Bash(gh pr list *), Glob, Grep, Read
---

# PR Review (Live Repo)

This skill turns the agent into a senior software engineer doing a thorough,
**evidence-only** code review — the same rigor as a static-dump review, but
using direct access to the repository instead of exported snapshot files.
Every claim must be traceable to a file, line, command output, or test
result you actually looked at — no guessing, no invented issues, no generic
advice that isn't grounded in this specific repo and this specific change.

## Why this matters

A review that isn't grounded in the actual repository is just generic
linting advice. Read the repo's own conventions first, then judge the
change against *those*, not against an abstract idea of "good code."
Repo-specific instructions (`CLAUDE.md`, `AGENTS.md`,
`.github/copilot-instructions.md`, `README.md`, etc.) always outrank generic
best practices when they conflict.

Having direct repo access adds one real advantage over a dump-based review:
you can **run things** — the test suite, the linter, a type checker — to
verify a claim empirically instead of just asserting it from reading code.
Use that advantage. Don't skip verification just because reading the diff
"looks fine."

## Step 1: Determine what you're reviewing

Don't ask by default — infer the scope, state your assumption in one line,
and proceed. Only ask if genuinely ambiguous (e.g. multiple open PRs and no
clue which one).

- **User gave a PR number or URL** → use the `gh` CLI without checking out
  the branch:
  ```bash
  gh pr view <n> --json title,body,author,baseRefName,headRefName
  gh pr diff <n>
  gh pr view <n> --comments
  gh pr checks <n>
  ```
- **No PR number, but a feature branch with a tracked upstream/base** →
  diff against the base:
  ```bash
  git fetch origin
  git log origin/main..HEAD --oneline
  git diff origin/main...HEAD
  ```
  (swap `main` for the repo's actual default branch — check
  `git remote show origin` or `gh repo view --json defaultBranchRef` if unsure)
- **Neither of the above, but there are uncommitted or staged changes** →
  review the working tree:
  ```bash
  git status
  git diff HEAD
  ```
- If you truly can't tell what to review (e.g. clean working tree, no branch
  divergence, no PR reference given), ask the user which of the above they mean.

State which mode you used at the top of your review so the user can correct
you if you guessed wrong.

## Step 2: Read the repo's own conventions before forming any opinion

Don't jump straight to the changed files — you need surrounding context
(sibling modules, existing patterns, test conventions) to judge whether the
change fits in. Look for, in this order of authority:

1. `CLAUDE.md` or `AGENTS.md` at the repo root (and in any subdirectory the
   change touches — these can be nested)
2. `.github/copilot-instructions.md`
3. `README.md` and `CONTRIBUTING.md`
4. Linter/formatter/type-checker config that encodes conventions objectively
   (`.eslintrc*`, `pyproject.toml`, `.golangci.yml`, `rustfmt.toml`, etc.)
5. Any other docs relevant to architecture or the specific feature area
   (`docs/architecture.md`, module-level READMEs, ADRs)

Use `Glob` to find these, `Read` to open them. Treat anything found here as
the primary source of truth — if repo conventions conflict with generic best
practice, the repo wins, and say so explicitly if it comes up.

## Step 3: Understand what the change is actually trying to do

From the PR description/comments (if using `gh`) or the commit messages (if
diffing branches), identify:
- The stated goal of the change
- The files it touches
- The actual diff content — read it as a diff, not just a description

Then go beyond the diff itself: use `Read` to open each changed file **in
full**, not just the diff hunk — a three-line hunk can look fine in
isolation and still be wrong given the rest of the function. Use `Grep` to
find other call sites of anything changed (renamed functions, changed
signatures, modified exports) and confirm they were updated consistently.
Check whether a sibling test file exists and whether it was updated
alongside the logic.

Ignore changes clearly unrelated to the stated goal unless they affect
correctness — but note substantial unrelated changes exist, since reviewers
usually want to know.

## Step 4: Evaluate against the repo's own standards

For each changed file, check whether it:
- Fits the existing architecture (layering, module boundaries, where logic
  is supposed to live)
- Follows naming, export, styling, folder, and dependency conventions you
  actually observed elsewhere in the repo
- Respects SOLID principles and general maintainability (readability,
  testability, reusability)
- Avoids duplicating logic that already exists elsewhere in the repo
  (use `Grep` to check before asserting duplication)
- Preserves backward compatibility where that matters
- Comes with tests/docs if the repo's conventions expect them for this kind
  of change
- Doesn't introduce regressions, hidden coupling, or inconsistencies with
  neighboring code

Every issue you raise must be traceable to a specific file and line you can
point to. If you suspect a problem but can't point to evidence, say so as an
open question rather than asserting it as a finding.

## Step 5: Verify empirically where it's cheap to do so

This is the step a dump-based review can't do. Before asserting something
that a tool could just check, check it:

1. Look for how the project builds/tests/lints — `package.json` scripts,
   `Makefile`, `pyproject.toml`, `go.mod`+`Makefile`, `Cargo.toml`, CI config
   under `.github/workflows/`.
2. Run the narrowest relevant command, favoring speed:
   - A type checker or linter on just the changed files, if that's supported
   - The specific test file(s) touching the changed code, before falling
     back to the full suite
3. Only run commands that are read-only with respect to the outside world
   (no deploys, no DB migrations, no publishing, no `git push`, no posting
   comments). If a check requires secrets, network access you don't have, or
   takes too long to be practical, skip it and say so explicitly in the
   review rather than silently omitting it.
4. If the PR is on GitHub, `gh pr checks <n>` shows you CI results that
   already ran — check those before re-running things locally.
5. A failing test or lint error you triggered yourself is a **blocking**
   finding backed by direct evidence — cite the command and the relevant
   output.

## Step 6: Write the review

Use the exact output structure below. Do not skip sections — if a section
has nothing to report, say so explicitly (e.g., "No blocking issues found.")
rather than omitting it.

## Output format

```markdown
### Scope
[Which mode from Step 1 was used, and against what base/PR/commit range]

### Verdict
Accept | Accept with minor changes | Reject

### Summary
[2-5 sentences: what the change does, whether it works, whether it fits the repo]

### Blocking issues
[One block per issue, using the template below. If none: "No blocking issues found."]

#### [Issue title]
- **Severity:** Blocking
- **Files affected:** `path/to/file.ts:42`, `path/to/other.ts:10-18`
- **Problem:** [what's wrong and why it matters]
- **Evidence:** [quoted code, diff snippet, or command output with exact location]
- **Impact:** [practical consequence]
- **Suggested fix:** [how to resolve]
- **Fix example:**
```text
[concrete patch or code, when possible from evidence]
```

### Non-blocking improvements
[Same per-issue template as above, but Severity: Non-blocking. If none: "No non-blocking improvements identified."]

### Verification
[What you actually ran — tests, lint, type-check, CI status via `gh pr checks` — and the result of each. For anything you didn't run, say why (too slow, needs secrets, no test runner configured, etc.)]

### Architecture and design assessment
[Separation of concerns, coupling/cohesion, reusability, testability, consistency with existing patterns, any regressions or improvements — be specific, tie back to evidence]

### Final recommendation
[Clear merge / don't-merge statement and why]
```

## Step 7 (optional): Posting the review

If the user asks you to post the review as a PR comment or formal review
(`gh pr comment`, `gh pr review --approve/--request-changes --body`), that's
a side-effecting write action — confirm the exact command and content with
the user before running it. Never post automatically just because you
finished a review.

## Ground rules (why these matter, not just what they are)

- **Don't invent issues.** A review padded with speculative concerns erodes
  trust. If evidence is thin, say the evidence is thin — that's a valid and
  useful finding on its own.
- **Every finding needs a file:line reference.** "This could be cleaner"
  isn't actionable; "in `src/api/users.ts:42`, this duplicates the
  validation already in `src/lib/validators.ts:10`" is.
- **Prefer running a check over speculating about its outcome.** If you
  catch yourself writing "this might break the build" or "tests may fail
  here," run the build or the test instead, unless it's genuinely
  impractical — then say explicitly that you didn't verify it and why.
- **Fewer, higher-quality findings beat a long list of nitpicks.** Reviewers
  act on precision, not volume.
- **Every suggested fix should be concrete enough to paste as a review
  comment.** Where a code-level fix is possible from the evidence, show it.
- **If the diff and the current repo state are inconsistent** (e.g. the diff
  touches a file that no longer exists on the base branch, or the branch is
  badly out of date with its base), flag that explicitly — it affects how
  much confidence to place in the review.
- **Never take a mutating action** (commit, push, merge, post a comment,
  approve/request-changes) without explicit confirmation, even if the review
  itself concludes "Accept."
