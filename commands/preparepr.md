/preparepr

Input
- PR: <number|url>
  - If missing: ALWAYS ask. Never auto-detect from conversation.
  - If ambiguous: ask.

SAFETY (read before doing anything)
- NEVER push to `main` or `origin/main`. All pushes go to the PR head branch only.
- NEVER run `git push` without specifying the remote and branch explicitly. No bare `git push`.
- Do NOT run gateway stop commands. Do NOT kill processes. Do NOT touch port 18792. The gateway is what is running you.

DO (prep only)
Goal: make the PR branch clean on top of main, fix issues from /reviewpr, run gates, commit fixes, and push back to the PR head branch.
Do NOT merge to main during this command.

This command uses PARALLELISM and a LATE REBASE strategy:
- Phase 1 (Parallel Scan): spawn 2 sub-subagents to scan for lint issues and identify fixes needed. All read-only.
- Phase 2 (Serial Fix): apply fixes following the recommended fix order, commit. Must be serial because it modifies files.
- Phase 3 (Gates): run lint, build, test sequentially (token-optimized for build and test). Tests run HERE, after fixes are applied.
- Phase 4 (Late Rebase + Push): rebase onto latest main LAST (right before push), then push and verify. This avoids repeated rebase cycles when main moves fast.

CRITICAL: Rebase happens ONCE at the end, not at the beginning. This prevents the expensive pattern of: rebase → fix → gates → main moved → rebase again → gates again → main moved again...

**Model tiers for sub-subagents:**
- Lint scan: `model:gpt-fast` (codex-spark, 2000 TPS) - just running commands and collecting output
- Fix identification (what to change and how): `model:gpt thinking:xhigh` (codex) - needs code reasoning
- All serial phases run in the parent agent which is already `model:gpt thinking:xhigh`

Note: tests are NOT run during the parallel scan phase. Running tests before fixes are applied produces misleading results. Tests run in Phase 3 (Gates) after all fixes are committed.

---

## Token-Optimized Command Execution (IMPORTANT)

When running build and test commands, do NOT watch output in real time. Run them in the background and only read output if they fail. This saves significant tokens.

**Pattern for tests (~60s average):**
```sh
pnpm test > .local/test-output.txt 2>&1 &
TEST_PID=$!
sleep 60
while kill -0 $TEST_PID 2>/dev/null; do sleep 10; done
wait $TEST_PID
TEST_EXIT=$?
if [ $TEST_EXIT -ne 0 ]; then
  echo "TESTS FAILED (exit $TEST_EXIT)"
  cat .local/test-output.txt
else
  echo "TESTS PASSED"
fi
```

**Pattern for build (~30-40s average):**
```sh
pnpm build > .local/build-output.txt 2>&1 &
BUILD_PID=$!
sleep 40
while kill -0 $BUILD_PID 2>/dev/null; do sleep 10; done
wait $BUILD_PID
BUILD_EXIT=$?
if [ $BUILD_EXIT -ne 0 ]; then
  echo "BUILD FAILED (exit $BUILD_EXIT)"
  cat .local/build-output.txt
else
  echo "BUILD PASSED"
fi
```

**Pattern for lint (fast, run inline):**
```sh
pnpm lint
```

Key rules:
- NEVER cat/read build or test output on success. Waste of tokens.
- ONLY read output when exit code is non-zero.
- The sleep durations are tuned to typical runtimes. The `while` loop handles variance.
- Lint is fast enough to run inline, no need for background.

---

EXECUTION RULE (CRITICAL)
- EXECUTE THIS COMMAND. DO NOT JUST PLAN.
- After you print the TODO checklist, immediately continue and run the shell commands.
- The previous failure mode was: printed a checklist and stopped. Do not do that.
- If you delegate to a subagent, the subagent MUST run the commands and produce real outputs.

Known footguns
- Repo path is ~/Development/openclaw. If you cd into ~/openclaw you will get "not a git repository".
- Do not run `git clean -fdx`, it would delete .local/ artifacts.
- Do not run `git add -A` or `git add .` blindly. Always stage specific files you changed.
- OPENCLAW_GATEWAY_TOKEN or OPENCLAW_GATEWAY_PASSWORD in env will cause test failures. Unset them before running tests.

Env preflight (run before any gates):
```sh
unset OPENCLAW_GATEWAY_TOKEN OPENCLAW_GATEWAY_PASSWORD 2>/dev/null || true
```

Process polling rules (for background build/test):
- Use the sleep + poll pattern shown above. Do NOT poll with 1s timeouts in tight loops.
- After the initial sleep, poll every 10s minimum.
- Do NOT use process poll with timeout:1000 repeatedly. That burns tokens reading "still running".
- If using exec with background:true, use yieldMs:60000 for tests and yieldMs:40000 for build.

Writing style for all output
- casual, direct
- no em dashes or en dashes, ever
- use commas or separate sentences instead

Completion criteria
- You fixed all BLOCKER and IMPORTANT items from .local/review.md.
- You ran gates and they passed.
- You committed any prep changes.
- You pushed the updated HEAD back to the PR head branch.
- You VERIFIED the push by comparing local HEAD sha vs the remote branch sha and gh PR head sha.
- You saved a prep summary to .local/prep.md.
- Final output is exactly: PR is ready for /mergepr

**CRITICAL**
- The push step is NOT optional.
- Do not write "PR is ready for /mergepr" unless push verification succeeded.

---

## Step 0: Verify gh auth

```sh
gh auth status
```
If this fails, stop and report. Do not proceed without valid GitHub auth.

## Step 1: Create a TODO checklist
Create a checklist of all prep steps. Print it. Then keep going and execute.

## Step 2: Setup Worktree

All prep work happens in an isolated worktree.

```sh
cd ~/Development/openclaw
# Sanity: confirm you are in the repo
git rev-parse --show-toplevel

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

mkdir -p .local

# Sanity, from here on, ALL commands run inside the worktree
pwd
```

From here on, ALL commands run inside the worktree directory.

## Step 3: Load Review Findings and Related Items (MANDATORY)

```sh
if [ -f .local/review.md ]; then
  echo "Found review findings from /reviewpr"
else
  echo "Missing .local/review.md. Run /reviewpr first and save findings."
  exit 1
fi

# Read the review
cat .local/review.md

# Read related issues/PRs (written by /reviewpr)
if [ -f .local/related.md ]; then
  echo "Found related items from /reviewpr"
  cat .local/related.md
else
  echo "No .local/related.md found, will skip related item tracking"
fi
```

Use .local/review.md to drive what you fix, especially the Concerns section.

## Step 4: Identify PR meta

```sh
gh pr view <PR> --json number,title,author,headRefName,baseRefName,headRepository,body --jq '{number,title,author:.author.login,head:.headRefName,base:.baseRefName,headRepo:.headRepository.nameWithOwner,body}'
contrib=$(gh pr view <PR> --json author --jq .author.login)
head=$(gh pr view <PR> --json headRefName --jq .headRefName)
base=$(gh pr view <PR> --json baseRefName --jq .baseRefName)
head_repo_url=$(gh pr view <PR> --json headRepository --jq .headRepository.url)

# Safety, never target main branches
if [ -z "$head" ]; then
  echo "ERROR: could not determine PR headRefName"
  exit 1
fi
if [ "$head" = "main" ] || [ "$head" = "master" ] || [ "$base" != "main" ]; then
  echo "ERROR: unexpected head/base branch, refusing to proceed"
  echo "head=$head"
  echo "base=$base"
  exit 1
fi
```

## Step 5: Configure remote and fetch PR head

```sh
# Ensure remote for PR head exists
prhead_url="$head_repo_url.git"
git remote add prhead "$prhead_url" 2>/dev/null || git remote set-url prhead "$prhead_url"
(git remote -v | rg '^prhead\s') || true

# Update origin main
git fetch origin main

# Fetch the PR head branch from the PR head repo
git fetch prhead "$head"

# Also keep a local snapshot via the upstream pull ref
git fetch origin pull/<PR>/head:pr-<PR> --force
```

## Step 6: Checkout PR head (NO REBASE)

```sh
# Move worktree to the PR head commit. Do NOT rebase onto main at all.
# /mergepr handles the final rebase right before squash merge.
git reset --hard "prhead/$head"
```

## Step 7: Install dependencies and PARALLEL SCAN PHASE

The worktree now has the PR code as-is. Install dependencies ONCE before spawning scan sub-subagents to prevent race conditions from parallel installs in the same worktree.

```sh
cd ~/Development/openclaw/.worktrees/pr-<PR>
pnpm install --frozen-lockfile
```

Now spawn 2 sub-subagents to scan for issues in parallel. Both are READ-ONLY scanners. They inspect the current state and write findings to separate files. The parent agent then reads all findings and applies fixes serially.

**Model assignment:**
- Lint Scanner: `model:gpt-fast` - just running pnpm lint and collecting output
- Fix Identifier: `model:gpt thinking:xhigh` - needs code reasoning to plan fixes

Spawn both at the same time. Do NOT wait for one before spawning the next.

### Sub-subagent: Lint Scanner

Spawn with EXACTLY this call:
```
sessions_spawn model:gpt-fast label:"pr-<PR>-lint-scan" runTimeoutSeconds:0 task:"
Scan for lint and formatting issues in PR #<PR>.

Worktree: ~/Development/openclaw/.worktrees/pr-<PR>
You are READ-ONLY for source files. Only write to .local/scan-lint.md.
Dependencies are already installed. Do NOT run pnpm install.

Run these commands and capture ALL output:
cd ~/Development/openclaw/.worktrees/pr-<PR>
pnpm lint 2>&1 | tee .local/lint-raw.txt
pnpm format:check 2>&1 | tee .local/format-raw.txt || true

Write .local/scan-lint.md with:
- LINT_STATUS: pass/fail
- LINT_ERRORS: full list of errors with file:line references
- FORMAT_STATUS: pass/fail
- FORMAT_ERRORS: files that need formatting
"
```

### Sub-subagent: Fix Identifier

Spawn with EXACTLY this call:
```
sessions_spawn model:gpt thinking:xhigh label:"pr-<PR>-fix-identifier" runTimeoutSeconds:0 task:"
Analyze the review findings and current code state to plan fixes for PR #<PR>.

Worktree: ~/Development/openclaw/.worktrees/pr-<PR>
You are READ-ONLY for source files. Only write to .local/scan-fixes.md.
Dependencies are already installed. Do NOT run pnpm install.

Read the review:
cd ~/Development/openclaw/.worktrees/pr-<PR>
cat .local/review.md

For each BLOCKER and IMPORTANT item in the review's Concerns section (section E):
1. Find the exact file and line(s) involved
2. Read the surrounding code for context
3. Determine the specific fix needed
4. Note any files that would need to change

Also check for:
- Local-only files that should not be committed (testing checklists, notes, TODOs, temp files, debug outputs)
  git diff --name-only origin/main..HEAD | grep -iE '(checklist|testing|todo|notes|temp|debug|scratch|plan)' || echo 'No suspicious files'
  git diff --name-only origin/main..HEAD | grep -v '/' | grep -vE '^(README|CHANGELOG|CONTRIBUTING|LICENSE|AGENTS|CLAUDE|Dockerfile|package|pnpm|tsconfig|vitest|\.)' || echo 'No suspicious root files'
- Test quality issues:
  - Unit tests using real timers instead of fake timers (vi.useFakeTimers)
  - Tests using sleep(), setTimeout(), delay patterns, or polling with real time
  - Tests reimplementing the function they test
  - Test isolation issues (reading/writing real user dirs)
  - Missing test coverage for new functionality
- Changelog needs (from review section H)
- Docs needs (from review section G)

Write .local/scan-fixes.md with:
- BLOCKERS: list of BLOCKER items with exact file:line and proposed fix (code sketch)
- IMPORTANTS: list of IMPORTANT items with exact file:line and proposed fix
- LOCAL_FILES: files to move to .local/
- TEST_QUALITY_FIXES: test improvements needed with file:line
- CHANGELOG_NEEDED: yes/no, what to add
- DOCS_NEEDED: yes/no, what to update
- FIX_ORDER: recommended order to apply fixes (dependencies between fixes, e.g. 'fix X before Y because Y imports from X')
"
```

## Step 8: End turn and wait for scan sub-subagents

DO NOT POLL. DO NOT LOOP. DO NOT call subagents list or sessions_list.

Sub-subagents will auto-announce their results back to you as user messages when they complete. Once both have announced back, read their output:

```sh
cd ~/Development/openclaw/.worktrees/pr-<PR>
echo "=== Lint Scan ==="
cat .local/scan-lint.md
echo ""
echo "=== Fix Plan ==="
cat .local/scan-fixes.md
```

## Step 9: SERIAL FIX PHASE (parent agent, do not delegate)

Now apply fixes serially. You (the parent agent) do this yourself, do not spawn sub-subagents for file modifications.

**Follow the FIX_ORDER from .local/scan-fixes.md** when applying fixes. The Fix Identifier analyzed dependencies between fixes and recommended an order. If FIX_ORDER is present, follow it. If not, use your judgment (blockers first, then importants).

Use the combined scan results to:

1) Fix local-only files first (if any):
   - Create `.local/` directory if needed: `mkdir -p .local`
   - Move offending files: `git rm <file> && git checkout HEAD -- <file> && mv <file> .local/`
   - Add `.local/` to `.gitignore` if not already: `grep -q '^.local/$' .gitignore || echo '.local/' >> .gitignore && git add .gitignore`

2) Fix all BLOCKER and IMPORTANT items from the review:
   - Fix them YOURSELF. Do not request changes from the author.
   - If tests are broken, fix them. If code has bugs, fix them. If formatting is wrong, fix it.
   - We handle everything in preparepr, contributors should not have to do more work.
   - NITs optional.
   - Keep scope tight, no drive-by refactors.

3) Test quality fixes (apply when relevant):
   - Convert unit tests using real timers (sleep, setTimeout waits) to fake timers (vi.useFakeTimers / vi.advanceTimersByTime)
   - Replace polling patterns with deterministic assertions where possible
   - If tests reimplement the function they test, refactor to call the actual function
   - Factor out heavily duplicated test setup into shared helpers
   - Ensure test isolation: no reading/writing real user dirs, mock paths before import

4) Fix lint and format issues:
   ```sh
   pnpm format:fix
   ```

5) Update docs (if PR has user-facing changes):
   Scan the diff for user-facing changes:
   - New tool parameters or options
   - Changed behavior or defaults
   - New config options
   - Removed features or deprecated functionality
   - New CLI commands or flags

   If user-facing changes found:
   ```sh
   # Find the docs folder
   ls -d docs/ doc/ documentation/ 2>/dev/null || echo "No docs folder found"

   # Search for docs related to the changed area
   rg -l "<feature_keyword>" docs/ README.md 2>/dev/null || true
   ```
   Update relevant docs to reflect the changes. Keep updates focused and accurate.

   If NO user-facing changes (purely internal refactor, test fixes, internal plumbing), skip this step entirely.

6) Update CHANGELOG.md (if PR has user-facing changes):
   This step is MANDATORY for user-facing PRs. Skip only for purely internal changes.

   ```sh
   # Get the latest release version and date
   gh release list --limit 1 --exclude-drafts --exclude-pre-releases

   # Get current date
   current_date=$(date +%Y-%m-%d)
   echo "current_date=$current_date"

   # Read the existing changelog
   if [ -f CHANGELOG.md ]; then
     head -50 CHANGELOG.md
   else
     echo "No CHANGELOG.md found, skip changelog update"
   fi
   ```

   Rules for changelog updates:
   - Check if the latest release is already tagged. If so, add changes under an "## Unreleased" section at the top of the changelog (below any header).
   - If there's already an "## Unreleased" section, append your entry to the appropriate subsection within it.
   - NEVER put new entries inside an already-released version's section.
   - Use the existing format/style of the changelog (heading levels, bullet style, categories).
   - Entry should be concise: one line summarizing what the PR does, with PR number reference.
   - Common categories: Added, Changed, Fixed, Removed, Deprecated.
   - Example entry: `- Fixed edge case in tool parameter validation (#1234)`
   - If the changelog uses a different format, match it.

Keep a running log of everything you fix for .local/prep.md.

## Step 10: Commit fixes

Stage only the specific files you changed (never `git add -A` or `git add .`):
```sh
git add <file1> <file2> ...
```

Preferred commit tool:
```sh
committer "fix: <summary> (#<PR>) (thanks @$contrib)" <changed files>
```

If `committer` is not found, fall back to:
```sh
git commit -m "fix: <summary> (#<PR>) (thanks @$contrib)"
```

## Step 11: Run focused gates (BEFORE pushing) - TOKEN OPTIMIZED

Gates run on the committed state. This is what will be pushed.
Prep runs FOCUSED tests (only files related to changes), not the full suite. /mergepr runs the full suite after rebase.

```sh
pnpm install

# 1. Format fix (inline, fast)
pnpm format:fix

# 2. Lint (inline, fast)
pnpm lint

# 3. Build (background, token-optimized)
pnpm build > .local/build-output.txt 2>&1 &
BUILD_PID=$!
sleep 40
while kill -0 $BUILD_PID 2>/dev/null; do sleep 10; done
wait $BUILD_PID
BUILD_EXIT=$?
if [ $BUILD_EXIT -ne 0 ]; then
  echo "BUILD FAILED (exit $BUILD_EXIT)"
  cat .local/build-output.txt
else
  echo "BUILD PASSED"
fi

# 4. FOCUSED tests only (files related to PR changes)
# Find test files related to changed source files
changed_files=$(git diff --name-only origin/main..HEAD -- '*.ts' '*.tsx' | grep -v '.test.' | grep -v '.e2e.' || true)
related_tests=""
for f in $changed_files; do
  # Look for corresponding test file
  base="${f%.ts}"
  base="${base%.tsx}"
  for pattern in "${base}.test.ts" "${base}.test.tsx" "${base}.node.test.ts" "${base}.e2e.test.ts"; do
    if [ -f "$pattern" ]; then
      related_tests="$related_tests $pattern"
    fi
  done
done

# Also include any test files that were directly changed in the PR
changed_tests=$(git diff --name-only origin/main..HEAD -- '*.test.ts' '*.test.tsx' || true)
related_tests="$related_tests $changed_tests"

# Deduplicate
related_tests=$(echo "$related_tests" | tr ' ' '\n' | sort -u | tr '\n' ' ')

if [ -n "$related_tests" ]; then
  echo "Running focused tests: $related_tests"
  pnpm vitest run $related_tests > .local/test-output.txt 2>&1 &
  TEST_PID=$!
  sleep 30
  while kill -0 $TEST_PID 2>/dev/null; do sleep 10; done
  wait $TEST_PID
  TEST_EXIT=$?
  if [ $TEST_EXIT -ne 0 ]; then
    echo "FOCUSED TESTS FAILED (exit $TEST_EXIT)"
    cat .local/test-output.txt
  else
    echo "FOCUSED TESTS PASSED"
  fi
else
  echo "No related test files found, skipping focused tests. Full suite runs in /mergepr."
fi
```

All must pass. If something fails, fix, commit the fix, and rerun.
MAX 3 ATTEMPTS. If gates still fail after 3 fix-and-rerun cycles, stop and report the failures. Do not loop indefinitely.

**Remember: only read build/test output if the command FAILED.** Don't burn tokens on success output.
**Note:** Full test suite runs in /mergepr after final rebase. Prep only validates the changed code.

## Step 12: PUSH AND VERIFY (MANDATORY, DO NOT SKIP)

Do NOT rebase here. Preparepr's job is to fix code and verify gates pass. The /mergepr step handles the final rebase onto latest main right before squash merge. This avoids wasted rebase cycles when main moves fast.

You must push the updated HEAD to the PR head branch and verify it landed.

```sh
# Safety, never push to main
if [ "$head" = "main" ] || [ "$head" = "master" ]; then
  echo "ERROR: head branch is main/master. This is wrong. Stopping."
  exit 1
fi

local_sha=$(git rev-parse HEAD)
echo "local_sha=$local_sha"
echo "Pushing to: prhead $head"

# Force with lease is required after rebase
git push --force-with-lease prhead HEAD:"$head"

# Verify remote branch sha matches local sha
remote_sha=$(git ls-remote prhead "refs/heads/$head" | awk '{print $1}' | head -n 1)
pr_sha=$(gh pr view <PR> --json headRefOid --jq .headRefOid)

echo "remote_sha=$remote_sha"
echo "pr_sha=$pr_sha"

if [ -z "$remote_sha" ]; then
  echo "ERROR: could not read remote sha for prhead/$head"
  exit 1
fi

if [ "$remote_sha" != "$local_sha" ]; then
  echo "ERROR: push verification failed, remote branch sha does not match local HEAD"
  exit 1
fi

if [ "$pr_sha" != "$local_sha" ]; then
  echo "ERROR: push verification failed, gh PR head sha does not match local HEAD"
  echo "This usually means the push did not land, or gh is pointing at a different head repo"
  exit 1
fi

echo "push_verified=yes"
```

**STOP** if verification fails. Do not proceed, do not claim success.

## Step 13: Check main distance (informational only)

This is informational. Do NOT rebase or loop. /mergepr handles the final rebase.

```sh
git fetch origin main
behind_count=$(git rev-list --count HEAD..origin/main 2>/dev/null || echo "?")
echo "PR is $behind_count commits behind main (mergepr will rebase before merge)"
```

## Step 14: Update review.md verdict (MANDATORY)

After all gates pass and push is verified, update .local/review.md so /mergepr knows the blockers were resolved.

Replace the first line that starts with `**NEEDS WORK**` or `**NOT USEFUL**` or `**NEEDS DISCUSSION**` with `**APPROVED (post-prep)** - All blockers resolved during /preparepr. Gates passing, push verified.`

If review.md already says `**APPROVED**` or `**GOOD TO MERGE**`, leave it as is.

Use your file edit tool to make this change. Then verify:
```sh
head -5 .local/review.md
```

## Step 15: Write prep summary artifacts (MANDATORY)

Write a prep summary to .local/prep.md.

It MUST include these machine-readable lines near the top so /mergepr can verify:
- prep_head_sha=<sha>
- push_branch=<branch>
- pushed_head_sha=<sha>
- push_verified=yes

Template:

```md
prep_head_sha=<REPLACE_WITH_LOCAL_SHA>
push_branch=<REPLACE_WITH_HEAD_BRANCH>
pushed_head_sha=<REPLACE_WITH_LOCAL_SHA>
push_verified=yes

PR: #<PR>
Contributor: @<contrib>
Pushed to: <head_repo_url> (<head>)

Changes made
- ...

Changelog
- updated: yes/no
- entry: <what was added>

Docs
- updated: yes/no
- files: <which docs were updated, or "n/a">

Gates
- pnpm lint: PASS
- pnpm build: PASS
- pnpm test: PASS

Rebase status
- up to date with main: yes
```

EXECUTE THIS, DO NOT JUST SAY YOU DID IT:
- Use your file write tool to create or overwrite .local/prep.md.
- Then verify it exists and is non-empty.

```sh
git rev-parse HEAD
ls -la .local/prep.md
wc -l .local/prep.md
```

## Step 16: Return the MAIN repo checkout to main (best effort)

```sh
cd ~/Development/openclaw

if git diff --quiet && git diff --cached --quiet; then
  git switch main 2>/dev/null || git checkout main || true
else
  echo "NOTE: ~/Development/openclaw has local changes, not switching branches"
  git status -sb || true
fi
```

## Step 17: Output

Include a diff stat summary:
```sh
cd ~/Development/openclaw
final_sha=$(git -C "$WORKTREE_DIR" rev-parse HEAD)
git -C "$WORKTREE_DIR" diff --stat origin/main..$final_sha
git -C "$WORKTREE_DIR" diff --shortstat origin/main..$final_sha
```

Report the total: X files changed, Y insertions(+), Z deletions(-)

- If gates passed AND push_verified=yes, print exactly:
  PR is ready for /mergepr
- Otherwise, list remaining failures and stop.

Rules
- Worktree only
- Do not delete the worktree on success, /mergepr may reuse it
- Do not run `gh pr merge`
- NEVER push to main. Only push to the PR head branch.
- All gates must pass before pushing. No pushing broken code.
