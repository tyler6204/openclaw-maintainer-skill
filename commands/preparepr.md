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

This command is FULLY SERIAL in one agent.
No scan fanout, no sub-subagents.

Execution order:
1) Install deps once
2) Scan lint and review findings inline
3) Apply fixes inline
4) Rebase onto latest main
5) Update changelog and docs if needed
6) Run gates (token-optimized build and test)
7) Push and verify

Why this is serial:
- prep scan fanout adds orchestration overhead for small time savings
- fewer moving parts means fewer announce chains breaking
- fresh parallel context is better spent in /reviewpr where readers are independent

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
- no em dash or en dash, ever
- use commas or separate sentences instead

Completion criteria
- You fixed all BLOCKER and IMPORTANT items from .local/review.md.
- You ran gates and they passed.
- You committed prep changes.
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

## Step 6: Checkout PR head

```sh
# Move worktree to the PR head commit
git reset --hard "prhead/$head"
```

## Step 7: Install dependencies once

```sh
cd ~/Development/openclaw/.worktrees/pr-<PR>
pnpm install --frozen-lockfile
```

## Step 8: SERIAL SCAN PHASE (no delegation)

Run scans inline and write scan artifacts for traceability.

```sh
cd ~/Development/openclaw/.worktrees/pr-<PR>

# 1) Lint + format checks
pnpm lint 2>&1 | tee .local/lint-raw.txt
pnpm format:check 2>&1 | tee .local/format-raw.txt || true

# 2) Create lint summary
{
  echo "LINT_STATUS=$(if rg -q 'error|✖|failed' .local/lint-raw.txt; then echo fail; else echo pass; fi)"
  echo "FORMAT_STATUS=$(if rg -q 'error|✖|failed|would be changed' .local/format-raw.txt; then echo fail; else echo pass; fi)"
  echo
  echo "## LINT_RAW"
  cat .local/lint-raw.txt
  echo
  echo "## FORMAT_RAW"
  cat .local/format-raw.txt
} > .local/scan-lint.md

# 3) Build a serial fix plan from review + repo state
{
  echo "# Serial fix plan for PR #<PR>"
  echo
  echo "## REVIEW_CONCERNS"
  rg -n '^E\)' -n .local/review.md || true
  rg -n 'BLOCKER|IMPORTANT|NIT' .local/review.md || true
  echo
  echo "## LOCAL_FILES"
  git diff --name-only origin/main..HEAD | grep -iE '(checklist|testing|todo|notes|temp|debug|scratch|plan)' || echo 'No suspicious files'
  git diff --name-only origin/main..HEAD | grep -v '/' | grep -vE '^(README|CHANGELOG|CONTRIBUTING|LICENSE|AGENTS|CLAUDE|Dockerfile|package|pnpm|tsconfig|vitest|\.)' || echo 'No suspicious root files'
  echo
  echo "## TEST_QUALITY_HINTS"
  rg -n 'sleep\(|setTimeout\(|setInterval\(|await delay\(|new Promise\(.*setTimeout|waitFor\(' --glob '**/*test*.ts' --glob '**/*test*.tsx' . || true
  echo
  echo "## FIX_ORDER"
  echo "1. local file cleanup"
  echo "2. BLOCKER fixes"
  echo "3. IMPORTANT fixes"
  echo "4. test quality fixes"
  echo "5. format fixes"
} > .local/scan-fixes.md

cat .local/scan-lint.md
cat .local/scan-fixes.md
```

## Step 9: SERIAL FIX PHASE (parent agent only)

Now apply fixes serially. Do not delegate file modifications.

Use `.local/review.md`, `.local/scan-lint.md`, and `.local/scan-fixes.md` to drive the work.

1) Fix local-only files first (if any):
   - Create `.local/` directory if needed: `mkdir -p .local`
   - Move offending files out of git-tracked paths and into `.local/`
   - Add `.local/` to `.gitignore` if not already: `grep -q '^.local/$' .gitignore || echo '.local/' >> .gitignore`

2) Fix all BLOCKER and IMPORTANT items from review:
   - Fix them yourself, do not request changes from the author.
   - If tests are broken, fix them.
   - Keep scope tight, no drive-by refactors.

3) Test quality fixes (apply when relevant):
   - Convert real timer tests to fake timers (vi.useFakeTimers / vi.advanceTimersByTime)
   - Replace polling patterns with deterministic assertions where possible
   - Remove tests that reimplement the function under test
   - Ensure test isolation, no reading or writing real user dirs

4) Apply formatting fixes:
```sh
pnpm format:fix
```

Keep a running log of all edits for `.local/prep.md`.

## Step 10: Rebase onto latest main (once)

Rebase after fixes, before docs/changelog and gates.

```sh
git fetch origin main
git rebase origin/main
```

If conflicts occur, resolve carefully. If not confident, stop and report instead of forcing bad resolutions.

## Step 11: Docs and changelog awareness

Update docs/changelog only when user-facing behavior changed.

Docs checks:
```sh
# Find docs directories if present
ls -d docs/ doc/ documentation/ 2>/dev/null || echo "No docs folder found"

# Search for docs related to changed area
rg -l "<feature_keyword>" docs/ README.md 2>/dev/null || true
```

Changelog checks:
```sh
# Get latest release metadata
gh release list --limit 1 --exclude-drafts --exclude-pre-releases

# Read changelog if present
if [ -f CHANGELOG.md ]; then
  head -50 CHANGELOG.md
else
  echo "No CHANGELOG.md found, skip changelog update"
fi
```

Rules:
- Add user-facing entries under `## Unreleased`.
- Never place new entries inside already-released sections.
- Match existing style.
- Keep entries concise with PR reference.

## Step 12: Commit fixes

Stage only files you intentionally changed, never `git add -A` or `git add .`.

```sh
git add <file1> <file2> ...
```

Preferred commit tool:
```sh
committer "fix: <summary> (#<PR>) (thanks @$contrib)" <changed files>
```

Fallback:
```sh
git commit -m "fix: <summary> (#<PR>) (thanks @$contrib)"
```

## Step 13: Run focused gates (BEFORE pushing), token optimized

Gates run on committed code after rebase.
Prep runs focused tests for changed areas, full suite remains in /mergepr.

```sh
unset OPENCLAW_GATEWAY_TOKEN OPENCLAW_GATEWAY_PASSWORD 2>/dev/null || true
pnpm install

# 1) Format fix (inline)
pnpm format:fix

# 2) Lint (inline)
pnpm lint

# 3) Build (background)
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

# 4) Focused tests (background)
changed_files=$(git diff --name-only origin/main..HEAD -- '*.ts' '*.tsx' | grep -v '.test.' | grep -v '.e2e.' || true)
related_tests=""
for f in $changed_files; do
  base="${f%.ts}"
  base="${base%.tsx}"
  for pattern in "${base}.test.ts" "${base}.test.tsx" "${base}.node.test.ts" "${base}.e2e.test.ts"; do
    if [ -f "$pattern" ]; then
      related_tests="$related_tests $pattern"
    fi
  done
done

changed_tests=$(git diff --name-only origin/main..HEAD -- '*.test.ts' '*.test.tsx' || true)
related_tests="$related_tests $changed_tests"
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

All gates must pass.
If gates fail, fix, commit, and rerun.
MAX 3 attempts, then stop and report failures.

## Step 14: Push and verify (MANDATORY)

Push the updated HEAD to PR head branch and verify it landed.

```sh
# Safety, never push to main
if [ "$head" = "main" ] || [ "$head" = "master" ]; then
  echo "ERROR: head branch is main/master. This is wrong. Stopping."
  exit 1
fi

local_sha=$(git rev-parse HEAD)
echo "local_sha=$local_sha"
echo "Pushing to: prhead $head"

git push --force-with-lease prhead HEAD:"$head"

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

## Step 15: Check main distance (informational)

```sh
git fetch origin main
behind_count=$(git rev-list --count HEAD..origin/main 2>/dev/null || echo "?")
echo "PR is $behind_count commits behind main"
```

## Step 16: Update review.md verdict (MANDATORY)

After gates pass and push verification succeeds, update `.local/review.md` so /mergepr knows blockers are resolved.

Replace the first line that starts with `**NEEDS WORK**` or `**NOT USEFUL**` or `**NEEDS DISCUSSION**` with:

`**APPROVED (post-prep)** - All blockers resolved during /preparepr. Gates passing, push verified.`

If review already says `**APPROVED**` or `**GOOD TO MERGE**`, leave it.

Verify:
```sh
head -5 .local/review.md
```

## Step 17: Write prep summary artifacts (MANDATORY)

Write `.local/prep.md`.

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
- up to date with main: yes/no
```

EXECUTE THIS, DO NOT JUST SAY YOU DID IT:
- Use your file write tool to create or overwrite .local/prep.md.
- Then verify it exists and is non-empty.

```sh
git rev-parse HEAD
ls -la .local/prep.md
wc -l .local/prep.md
```

## Step 18: Return main repo checkout to main (best effort)

```sh
cd ~/Development/openclaw

if git diff --quiet && git diff --cached --quiet; then
  git switch main 2>/dev/null || git checkout main || true
else
  echo "NOTE: ~/Development/openclaw has local changes, not switching branches"
  git status -sb || true
fi
```

## Step 19: Output

Include diff stat summary:
```sh
cd ~/Development/openclaw
final_sha=$(git -C "$WORKTREE_DIR" rev-parse HEAD)
git -C "$WORKTREE_DIR" diff --stat origin/main..$final_sha
git -C "$WORKTREE_DIR" diff --shortstat origin/main..$final_sha
```

Report total: X files changed, Y insertions(+), Z deletions(-)

- If gates passed AND push_verified=yes, print exactly:
  PR is ready for /mergepr
- Otherwise, list remaining failures and stop.

Rules
- Worktree only
- Do not delete worktree on success, /mergepr may reuse it
- Do not run `gh pr merge`
- NEVER push to main, only push to PR head branch
- All gates must pass before pushing
