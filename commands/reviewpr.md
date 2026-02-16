/reviewpr

Input
- PR: <number|url>
  - If missing: ALWAYS ask. Never auto-detect from conversation.
  - If ambiguous: ask.

DO (review only)
Goal: produce a thorough review and a clear recommendation (READY for /preparepr vs NEEDS WORK vs CLOSE). Do NOT merge, do NOT push, do NOT make code changes that you intend to keep.

This command uses PARALLELISM. After worktree setup and PR meta collection, you spawn 3 sub-subagents that run concurrently. Then you combine their results into the final review.

SAFETY (read before doing anything)
- NEVER push to `main` or `origin/main`. Not during review, not ever.
- Do NOT stop or kill the gateway. Do not run gateway stop commands, do not kill processes on port 18792.
- Do NOT run `git push` at all during review. This is a read-only operation.

EXECUTION RULE (CRITICAL)
- EXECUTE THIS COMMAND. DO NOT JUST PLAN.
- After you print the TODO checklist, immediately continue and run the shell commands.
- If you delegate to a subagent, the subagent MUST run the commands and produce real outputs, not a plan.

Known failure modes (read this)
- If you see "fatal: not a git repository", you are in the wrong directory. The repo is at ~/Development/openclaw, not ~/openclaw.
- Do not stop after printing the checklist. That is not completion.

Writing style for all output
- casual, direct
- no em dashes or en dashes, ever
- use commas or separate sentences instead

Completion criteria
- You ran the commands in the worktree and inspected the PR, you are not guessing.
- You produced the structured review A through K.
- You saved the full review to .local/review.md inside the worktree.
- You saved related issues/PRs to .local/related.md inside the worktree.

---

## Step 0: Verify gh auth

```sh
gh auth status
```
If this fails, stop and report. Do not proceed without valid GitHub auth.

## Step 1: Create a TODO checklist
Create a checklist of all review steps. Print it. Then keep going and execute.

## Step 2: Setup Worktree

All review work happens in an isolated worktree.

Important: keep the main repo checkout clean and on main.
This helps Cursor and VS Code show sane branch and sync state.
This is best effort, do not destroy local changes.

```sh
cd ~/Development/openclaw
# Sanity: confirm you are in the repo
git rev-parse --show-toplevel

# Best effort, keep the main repo checkout on main
if git diff --quiet && git diff --cached --quiet; then
  git switch main 2>/dev/null || git checkout main || true
else
  echo "NOTE: ~/Development/openclaw has local changes, leaving its branch alone"
  git status -sb || true
fi

PR=<PR>
WORKTREE_DIR=".worktrees/pr-$PR"
WORKTREE_BRANCH="pr/$PR"

git fetch origin main

# Reuse existing worktree if it exists, otherwise create new
if [ -d "$WORKTREE_DIR" ]; then
  cd "$WORKTREE_DIR"
  git checkout "$WORKTREE_BRANCH" 2>/dev/null || git checkout -b "$WORKTREE_BRANCH"
  git fetch origin main
  git reset --hard origin/main
else
  git worktree add "$WORKTREE_DIR" -b "$WORKTREE_BRANCH" origin/main
  cd "$WORKTREE_DIR"
fi

# Local scratch space that persists across /reviewpr -> /preparepr -> /mergepr
mkdir -p .local

# Sanity, from here on, ALL commands run inside the worktree
pwd
```

The worktree starts on origin/main intentionally so you can check for existing implementations before looking at the PR code.

## Step 3: Collect PR meta (needed by all sub-subagents)

Gather all PR metadata upfront. This data gets passed to the parallel sub-subagents.

```sh
gh pr view <PR> --json number,title,state,isDraft,author,baseRefName,headRefName,headRepository,url,body,labels,assignees,reviewRequests,files,additions,deletions --jq '{number,title,url,state,isDraft,author:.author.login,base:.baseRefName,head:.headRefName,headRepo:.headRepository.nameWithOwner,additions,deletions,files:.files|length,body}'
```

Save the PR meta to a temp file for sub-subagents:
```sh
gh pr view <PR> --json number,title,state,isDraft,author,baseRefName,headRefName,headRepository,url,body,labels,assignees,files,additions,deletions > .local/pr-meta.json
```

Also fetch the diff and save it so sub-subagents can read it without re-fetching:
```sh
gh pr diff <PR> > .local/pr-diff.txt

# Also fetch PR head to a local ref for deeper analysis
git fetch origin pull/<PR>/head:pr-<PR> --force

# Find merge base for rebase awareness
merge_base=$(git merge-base origin/main pr-<PR>)
echo "$merge_base" > .local/merge-base.txt
echo "merge_base=$merge_base"
```

Fetch PR comments and review comments:
```sh
# PR conversation comments (includes bot reports from Greptile, Codex, etc.)
gh pr view <PR> --comments --json comments --jq '.comments[] | "[\(.author.login) \(.createdAt)]\n\(.body)\n---"' > .local/pr-comments.txt 2>/dev/null || true

# Code review comments
gh api repos/openclaw/openclaw/pulls/<PR>/reviews --jq '.[] | "[\(.user.login) \(.state) \(.submitted_at)]\n\(.body)\n---"' > .local/pr-reviews.txt 2>/dev/null || true
```

Claim the PR:
```sh
gh_user=$(gh api user --jq .login)
gh pr edit <PR> --add-assignee "$gh_user"
```

## Step 4: SPAWN 3 PARALLEL SUB-SUBAGENTS

This is the core parallelism step. All three sub-subagents are READ-ONLY and safe to run concurrently. They all read from the same worktree and .local/ files but write to separate output files.

**Model assignment:**
- Sub-subagent A (Code Analysis): `model:gpt thinking:xhigh` - needs deep code reasoning
- Sub-subagent B (CI & Related Scan): `model:gpt-fast` - just API calls and string matching, 2000 TPS
- Sub-subagent C (Test Coverage & Docs): `model:gpt thinking:xhigh` - needs judgment about test quality

Spawn all three at the same time using `sessions_spawn`. Do NOT wait for one to finish before spawning the next.

### Sub-subagent A: Code Analysis

Spawn with EXACTLY this call:
```
sessions_spawn model:gpt thinking:xhigh label:"pr-<PR>-code-analysis" runTimeoutSeconds:0 task:"
Read the PR diff and analyze code quality for PR #<PR> in the openclaw repo.

Worktree: ~/Development/openclaw/.worktrees/pr-<PR>
You are READ-ONLY. Do not push, commit, or modify any files except writing your output to .local/review-a.md.

Input files (already exist in worktree .local/):
- .local/pr-meta.json (PR metadata)
- .local/pr-diff.txt (full diff)
- .local/merge-base.txt (merge base sha)
- .local/pr-comments.txt (conversation comments)
- .local/pr-reviews.txt (review comments)

Your job:

1) Read the PR description from pr-meta.json. Summarize goal, scope, missing context.

2) Read PR comments and review comments from .local/pr-comments.txt and .local/pr-reviews.txt.
   Note findings from automated reviews (Greptile confidence, Codex suggestions). Validate real issues, note false positives.

3) Check if the feature already exists in main (before looking at PR code).
   You are on origin/main in the worktree. Search using keywords from PR title, changed file paths, function names.
   cd ~/Development/openclaw/.worktrees/pr-<PR>
   rg -n '<keyword>' -S src packages apps ui || true
   git log --oneline --all --grep='<keyword>' | head -20
   If it already exists, flag as BLOCKER.

4) Read the diff thoroughly from .local/pr-diff.txt.
   Also examine with git diff for context:
   git diff --stat origin/main..pr-<PR>
   git diff origin/main..pr-<PR>

5) REBASE AWARENESS (critical):
   Read .local/merge-base.txt. If code exists on current main but NOT at the merge base, the PR author did not remove it, they never had it. Do NOT flag this as a regression.

6) Validate the change is needed and valuable.
   Would you want this merged if you maintained the codebase long term? Is it the right solution or a quick fix? Does it add cruft?
   Default to closing. Most PRs should not be merged.
   Explicitly recommend CLOSE if: AI slop, duplicate, bandaid, wrong approach, unnecessary complexity, or out of scope.

   INTENT CHECK (critical):
   - Does the PR body reference a specific open issue (fixes #X, closes #X)?
   - Is there an open issue or feature request that motivated this work?
   - If NO linked issue and NO clear demand from the community: apply much higher scrutiny.
     Ask: did anyone actually ask for this? Is this solving a real problem or is it speculative?
   - External contributors with no linked issue and no prior discussion = strong close signal.
   - PRs that add new API surface (endpoints, RPC methods, config options) without a linked issue are almost always close candidates.
   
   Set VALUE_VERDICT to one of: MERGE / NEEDS_WORK / CLOSE with a brief reason.
   Include INTENT_SCORE: high (linked issue, community demand) / medium (reasonable but unasked for) / low (no issue, speculative, no one asked)

7) Evaluate implementation quality: correctness, design, performance, ergonomics.

8) Security review: auth, input validation, secrets, dependencies, tool safety, privacy.
   OpenClaw subagents run with full disk access including git, gh, and shell.

9) Check for local-only files in the diff that should not be committed (testing checklists, notes, TODOs, temp files, debug outputs).

Write your output to .local/review-a.md with these sections:
- EXISTING_IN_MAIN: yes/no and details
- DESCRIPTION_SUMMARY: what the PR does
- COMMENTS_SUMMARY: notable automated/human review findings
- CODE_QUALITY: detailed analysis
- SECURITY: findings
- VALUE_JUDGMENT: needed/valuable/cruft assessment
- LOCAL_FILES: any files that should be in .local/ instead
- VALUE_VERDICT: MERGE / NEEDS_WORK / CLOSE with reason
- INTENT_SCORE: high / medium / low with explanation (linked issue? community demand? who asked for this?)
- CONCERNS: numbered list, each marked BLOCKER/IMPORTANT/NIT with file reference and proposed fix
"
```

### Sub-subagent B: CI & Related Scan

Spawn with EXACTLY this call:
```
sessions_spawn model:gpt-fast label:"pr-<PR>-ci-scan" runTimeoutSeconds:0 task:"
Check CI status and scan for related issues/PRs for PR #<PR> in the openclaw repo.

Worktree: ~/Development/openclaw/.worktrees/pr-<PR>
You are READ-ONLY. Do not push, commit, or modify any files except writing your output to .local/review-b.md and .local/related.md.

Input files (already exist in worktree .local/):
- .local/pr-meta.json (PR metadata)

Your job:

1) Check CI/check status:
   cd ~/Development/openclaw/.worktrees/pr-<PR>
   gh pr checks <PR>
   Note which checks passed, failed, or are pending.

2) Find related issues that this PR might fix:
   pr_title=$(gh pr view <PR> --json title --jq .title)
   gh issue list --state open --limit 20 --json number,title --jq '.[] | \"\(.number)\t\(.title)\"' | rg -i '<keyword>' || true
   gh pr view <PR> --json body --jq .body | rg -io '(fix(es)?|close[sd]?|resolve[sd]?)\s*#\d+' || true

3) Find related/duplicate open PRs:
   gh pr list --state open --limit 30 --json number,title --jq '.[] | \"\(.number)\t\(.title)\"' | rg -i '<keyword>' || true

4) Search more broadly for related items:
   title=$(gh pr view <PR> --json title --jq .title)
   q1=$(echo \"$title\" | sed -E 's/\([^)]*\)//g' | tr -s ' ' | cut -c1-80)
   gh search issues --repo openclaw/openclaw --state open --limit 20 --json number,title --search \"$q1\" || true
   gh search prs --repo openclaw/openclaw --state open --limit 20 --json number,title --search \"$q1\" || true

5) Write your output to .local/review-b.md with sections:
   - CI_STATUS: pass/fail/pending for each check
   - RELATED_ISSUES: list of issue numbers with titles and why they are related
   - RELATED_PRS: list of PR numbers with titles and whether they are duplicates or complementary
   - BODY_REFS: explicit closes/fixes references from PR body

6) ALSO write .local/related.md (THIS IS CRITICAL, used by /preparepr and /mergepr):
   Format:
   # Related Issues and PRs for PR #<PR>
   # Generated by /reviewpr sub-subagent B

   ## Issues to close after merge
   - #<num>: <title> (reason: fixes/resolves/duplicate)

   ## PRs to close after merge (superseded/duplicate)
   - #<num>: <title> (reason: duplicate/superseded)

   ## PRs to comment on (related but not duplicate)
   - #<num>: <title> (reason: overlapping/complementary)

   ## Body references
   - fixes #<num>
   - closes #<num>

   If no related items found, still create the file with empty sections.
"
```

### Sub-subagent C: Test Coverage & Docs

Spawn with EXACTLY this call:
```
sessions_spawn model:gpt thinking:xhigh label:"pr-<PR>-test-coverage" runTimeoutSeconds:0 task:"
Check test coverage gaps, test quality, docs, and changelog for PR #<PR> in the openclaw repo.

Worktree: ~/Development/openclaw/.worktrees/pr-<PR>
You are READ-ONLY. Do not push, commit, or modify any files except writing your output to .local/review-c.md.

Input files (already exist in worktree .local/):
- .local/pr-meta.json (PR metadata)
- .local/pr-diff.txt (full diff)

Your job:

1) Analyze test coverage:
   cd ~/Development/openclaw/.worktrees/pr-<PR>
   - Read the diff from .local/pr-diff.txt
   - Identify all new or changed functions/modules
   - Check if corresponding test files exist
   - Check if new code paths have test coverage
   - Look at existing tests for the changed areas
   - FLAG AS IMPORTANT: any new functionality that has zero test coverage

2) Test quality analysis (FLAG THESE EXPLICITLY):
   - sleep / setTimeout / delays: Flag any test using sleep(), setTimeout(), new Promise(resolve => setTimeout(...)), await delay(), or any other real-time delay pattern. Tests should use fake timers (vi.useFakeTimers() / vi.advanceTimersByTime()) instead of real sleeps. Flag as IMPORTANT.
   - Polling patterns: Flag tests that use setInterval, retry loops with delays, or waitFor with real time polling. These make tests slow and flaky. Flag as IMPORTANT.
   - Reimplementation: Look for tests that reimplement the function they are testing inside the test body. The test should call the actual function, not a copy. Flag as IMPORTANT.
   - Duplicated test code: Look for heavily duplicated test setup that could use shared helpers. Flag as NIT.
   - Slow tests: Any single test or test file taking multiple seconds in unit test context is a red flag. Flag as IMPORTANT.
   - Missing coverage for new code: If the PR adds new functions, classes, or significant code paths with no corresponding tests, flag as IMPORTANT with a specific note about what needs testing.

3) Test isolation check:
   - Tests must NEVER read from or write to real user directories (~/.openclaw/, ~/.cache/, etc.)
   - Check for imports of registry/store modules that resolve paths at import time
   - If code uses module-level constants for paths, tests must mock them

4) Docs check:
   - Check if the PR touches code with related documentation (README, docs/, inline API docs, config examples)
   - If docs exist for the changed area and PR does not update them, flag as IMPORTANT
   - If PR adds a new feature/config with no docs, flag as IMPORTANT
   - If purely internal with no user-facing impact, note 'not applicable'

5) Changelog check:
   - Check if CHANGELOG.md exists: ls CHANGELOG.md 2>/dev/null
   - If the project has a CHANGELOG.md and this PR is user-facing, flag missing entry as IMPORTANT

6) Identify what minimal regression test would look like (1-2 sentences)

Write your output to .local/review-c.md with sections:
- TEST_COVERAGE: what is covered, what is missing, file-by-file. Explicitly note any new functions/modules with zero coverage.
- TEST_QUALITY: issues found. For each issue, include the exact pattern found, file path and line number, severity (IMPORTANT or NIT), and suggested fix.
- TEST_ISOLATION: any isolation concerns
- DOCS_STATUS: up to date / missing / not applicable
- CHANGELOG: needs entry (yes/no), what category (added/changed/fixed/removed)
- SUGGESTED_TESTS: minimal regression test description
- CONCERNS: numbered list, each marked BLOCKER/IMPORTANT/NIT
"
```

## Step 5: End turn and wait for sub-subagents

DO NOT POLL. DO NOT LOOP. DO NOT call subagents list or sessions_list.

Sub-subagents will auto-announce their results back to you as user messages when they complete. Just end your turn and wait. When all three have announced back, read their output files:

```sh
cd ~/Development/openclaw/.worktrees/pr-<PR>
echo "=== Sub-subagent A: Code Analysis ==="
cat .local/review-a.md
echo ""
echo "=== Sub-subagent B: CI & Related ==="
cat .local/review-b.md
echo ""
echo "=== Sub-subagent C: Test Coverage ==="
cat .local/review-c.md
echo ""
echo "=== Related Issues/PRs ==="
cat .local/related.md
```

## Step 6: Combine results into final review

Synthesize all three sub-subagent outputs into one structured review. Use your judgment to:
- Deduplicate concerns flagged by multiple sub-subagents
- Promote or demote severity based on combined context
- Resolve contradictions (e.g., sub-A says code is fine but sub-C found missing tests)
- Ensure test quality issues (sleep/setTimeout/polling) are prominently surfaced

Key question: Can /preparepr fix the issues ourselves? Almost always yes. We fix everything in preparepr, contributors should not need to do more work. Only flag something as "needs author" if it requires domain knowledge we cannot infer from the codebase.

## Step 7: Save artifacts (MANDATORY)

Write the full structured review to .local/review.md.
Verify .local/related.md exists (written by sub-subagent B).

EXECUTE THIS, DO NOT JUST SAY YOU DID IT:
- Use your file write tool to create or overwrite .local/review.md
- Verify both files exist and are non-empty

```sh
ls -la .local/review.md .local/related.md
wc -l .local/review.md .local/related.md
```

## Step 8: Output (structured)

Produce a review with these sections (must match what you saved to .local/review.md):

A) TL;DR recommendation
- One of: READY FOR /preparepr | NEEDS WORK | NEEDS DISCUSSION | RECOMMEND CLOSE
- RECOMMEND CLOSE when any of these apply:
  - Low quality, AI slop, or clearly auto-generated with no thought
  - Bandaid fix that adds tech debt without solving the root cause
  - Wrong approach or fundamentally flawed design
  - Duplicates an existing PR or feature already on main
  - Out of scope, not aligned with project direction
  - Adds unnecessary complexity for minimal value
  - No tests, no docs, no description, and trivial change
  - No linked issue, no community demand, and adds new API surface (strong close signal)
  - External contributor with no prior discussion or issue reference
- Be willing to recommend closing. Protecting codebase quality > merging PRs.
- Default bias: close. Most external PRs should not be merged. Only recommend merge if it genuinely improves the codebase.
- Include INTENT score (high/medium/low) in your recommendation.
- If recommending close, explain why briefly. Do NOT actually close the PR; recommend it and wait for owner confirmation.
- 1 to 3 sentences.

B) What changed

C) What's good

D) Security findings

E) Concerns / questions (actionable)
- Numbered list
- Mark each item as BLOCKER, IMPORTANT, or NIT
- For each, point to file or area and propose a concrete fix

F) Tests
- Coverage status
- Quality issues (explicitly call out any sleep/setTimeout/polling/delay patterns found)
- Missing test coverage for new functionality
- Isolation concerns

G) Docs status
- Are related docs up to date, missing, or not applicable?

H) Changelog
- Does CHANGELOG.md need an entry? What category (added, changed, fixed, removed)?

I) Follow ups (optional)

J) Suggested PR comment (optional)

K) Related issues/PRs to close after merge
- List issue numbers that this PR fixes (from "fixes #X" in body or search)
- List duplicate PR numbers that should be closed once this merges
- Format: "Close #1234 (duplicate)" or "Close #5678 (fixes this issue)"
- Reference .local/related.md for the full list

Guardrails
- Worktree only
- Do not delete the worktree after review
- Review only, do not merge, do not push
