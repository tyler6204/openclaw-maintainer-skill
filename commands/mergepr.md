/mergepr

Input
- PR: <number|url>
  - If missing: ALWAYS ask. Never auto-detect from conversation.
  - If ambiguous: ask.

SAFETY (read before doing anything)
- The ONLY way code reaches main is `gh pr merge --squash`. No `git push` to main, ever.
- Do NOT run gateway stop commands. Do NOT kill processes. Do NOT touch port 18792.
- Do NOT run `git push` at all during merge. The only push-like operation is `gh pr merge`.

DO (merge only)
Goal: PR must end in GitHub state = MERGED (never CLOSED). Assumes /preparepr already ran.
Use gh pr merge with --squash.

After merge succeeds, this command spawns a SINGLE cleanup sub-subagent that handles all post-merge tasks serially: close superseded PRs, close related issues, and clean up the worktree. Uses `model:gpt-fast` (codex-spark, 2000 TPS) since it's just API calls.

## GitHub Identity Awareness (IMPORTANT)

Before commenting on ANY issue or PR (including the merge comment, closing superseded PRs, closing issues), you MUST check who you are and who authored the item:

```sh
gh_user=$(gh api user --jq .login)
echo "I am: $gh_user"
```

**Identity-aware commenting rules (referenced throughout this file):**
- If you are the author of the item: use self-referential language. Do NOT thank yourself.
  - Good: "Merged. Merge commit: abc123"
  - Good: "Closing this out, superseded by #1234."
  - Bad: "Thanks @myself for the contribution!"
- If you are NOT the author: be polite, thank the contributor/reporter.
  - Good: "Thanks @contributor! Merged via squash. Merge commit: abc123"
  - Good: "Resolved by #1234. Thanks for the report!"
- Use case-insensitive comparison for GitHub usernames (see Step 4).
- This applies everywhere: merge comments, closing superseded PRs, closing issues.

EXECUTION RULE (CRITICAL)
- EXECUTE THIS COMMAND. DO NOT JUST PLAN.
- After you print the TODO checklist, immediately continue and run the shell commands.
- If you delegate to a subagent, the subagent MUST run the commands and produce real outputs.

Known footguns
- Repo path is ~/Development/openclaw.
- This command must read .local/review.md, .local/related.md, and .local/prep.md in the worktree. Do not skip.
- Cleanup must remove the real worktree directory .worktrees/pr-<PR>, not a different path.
- After cleanup, .local/ artifacts are gone. This is expected.

Writing style for all output
- casual, direct
- no em dashes or en dashes, ever
- use commas or separate sentences instead

Completion criteria
- gh pr merge succeeded
- PR state is MERGED
- merge sha recorded
- post-merge cleanup completed (issues closed, duplicate PRs closed, worktree removed)

---

## Step 0: Verify gh auth

```sh
gh auth status
```
If this fails, stop and report. Do not proceed without valid GitHub auth.

## Step 1: Create a TODO checklist
Create a checklist of all merge steps. Print it. Then keep going and execute.

## Step 2: Setup Worktree

All merge work happens in an isolated worktree.

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

## Step 3: Load all local artifacts (MANDATORY)

These files should exist from earlier steps:
- .local/review.md from /reviewpr
- .local/related.md from /reviewpr (related issues/PRs to close)
- .local/prep.md from /preparepr

```sh
ls -la .local || true

if [ -f .local/review.md ]; then
  echo "Found .local/review.md"
  head -20 .local/review.md
else
  echo "Missing .local/review.md. Stop and run /reviewpr, then /preparepr."
  exit 1
fi

if [ -f .local/prep.md ]; then
  echo "Found .local/prep.md"
  cat .local/prep.md
else
  echo "Missing .local/prep.md. Stop and run /preparepr first."
  exit 1
fi

if [ -f .local/related.md ]; then
  echo "Found .local/related.md"
  cat .local/related.md
else
  echo "No .local/related.md found, will skip related item cleanup"
fi
```

If review.md says NEEDS WORK, NEEDS DISCUSSION, or NOT USEFUL, stop.

## Step 4: Identify PR meta and GitHub identity

```sh
gh pr view <PR> --json number,title,state,isDraft,author,headRefName,baseRefName,headRepository,body,headRefOid --jq '{number,title,state,isDraft,author:.author.login,head:.headRefName,headSha:.headRefOid,base:.baseRefName,headRepo:.headRepository.nameWithOwner,body}'
contrib=$(gh pr view <PR> --json author --jq .author.login)
head=$(gh pr view <PR> --json headRefName --jq .headRefName)
base=$(gh pr view <PR> --json baseRefName --jq .baseRefName)
head_repo_url=$(gh pr view <PR> --json headRepository --jq .headRepository.url)
pr_sha=$(gh pr view <PR> --json headRefOid --jq .headRefOid)

# GitHub identity check (case-insensitive comparison)
gh_user=$(gh api user --jq .login)
echo "Logged in as: $gh_user"
echo "PR author: $contrib"
if [ "$(echo "$gh_user" | tr A-Z a-z)" = "$(echo "$contrib" | tr A-Z a-z)" ]; then
  echo "SELF_AUTHORED=true (do not thank yourself in comments)"
  SELF_AUTHORED=true
else
  echo "SELF_AUTHORED=false (be polite, thank contributor)"
  SELF_AUTHORED=false
fi

if [ -z "$head" ] || [ -z "$pr_sha" ]; then
  echo "ERROR: could not determine PR head branch or sha"
  exit 1
fi
if [ "$head" = "main" ] || [ "$head" = "master" ] || [ "$base" != "main" ]; then
  echo "ERROR: unexpected head/base branch, refusing to proceed"
  echo "head=$head base=$base"
  exit 1
fi
```

## Step 5: Verify the remote PR branch includes the latest /preparepr commits (MANDATORY)

/preparepr must have pushed and verified.
We verify again here so we never merge stale code.

```sh
expected_sha=$(sed -n 's/^pushed_head_sha=//p' .local/prep.md | head -n 1)
prep_verified=$(sed -n 's/^push_verified=//p' .local/prep.md | head -n 1)

echo "expected_sha=$expected_sha"
echo "prep_verified=$prep_verified"
echo "pr_sha=$pr_sha"

if [ -z "$expected_sha" ]; then
  echo "ERROR: could not find pushed_head_sha=... in .local/prep.md"
  echo "Run /preparepr again and make sure it writes machine readable fields"
  exit 1
fi

if [ "$prep_verified" != "yes" ]; then
  echo "ERROR: .local/prep.md does not indicate push_verified=yes"
  echo "Run /preparepr again"
  exit 1
fi

if [ "$expected_sha" != "$pr_sha" ]; then
  echo "ERROR: PR head sha does not match the last prepared push"
  echo "This means the PR branch changed after prep, or prep did not actually push"
  echo "Run /preparepr again"
  exit 1
fi
```

## Step 6: Sanity checks

Stop if any of these are true:
- PR is a draft
- required checks are failing (after rebase)

```sh
# Draft?
if gh pr view <PR> --json isDraft --jq .isDraft | grep -q true; then
  echo "ERROR: PR is a draft, refusing to merge"
  exit 1
fi

# Rebase onto latest main if behind (this is the ONLY place rebase happens in the pipeline)
git fetch origin main
git fetch origin pull/<PR>/head:pr-<PR> --force
if ! git merge-base --is-ancestor origin/main pr-<PR>; then
  echo "PR is behind main, rebasing now..."
  
  # Checkout PR branch in worktree and rebase
  cd ~/Development/openclaw/.worktrees/pr-<PR>
  git checkout pr-<PR> 2>/dev/null || git checkout -b pr-<PR> pr-<PR>
  git rebase origin/main
  
  # If rebase has conflicts, stop and report
  if [ $? -ne 0 ]; then
    echo "ERROR: rebase conflicts, run /preparepr to resolve"
    git rebase --abort
    exit 1
  fi
  
  # Run full gates after rebase (this is the definitive test run)
  echo "Running full gates after rebase..."
  
  pnpm install --frozen-lockfile
  pnpm lint || { echo "ERROR: lint failed after rebase, run /preparepr"; exit 1; }
  
  # Build (background, token-optimized)
  pnpm build > .local/merge-build-output.txt 2>&1 &
  BUILD_PID=$!
  sleep 40
  while kill -0 $BUILD_PID 2>/dev/null; do sleep 10; done
  wait $BUILD_PID
  if [ $? -ne 0 ]; then
    echo "ERROR: build failed after rebase"
    cat .local/merge-build-output.txt
    exit 1
  fi
  echo "BUILD PASSED"
  
  # Full test suite (background, token-optimized)
  unset OPENCLAW_GATEWAY_TOKEN OPENCLAW_GATEWAY_PASSWORD 2>/dev/null || true
  pnpm test > .local/merge-test-output.txt 2>&1 &
  TEST_PID=$!
  sleep 60
  while kill -0 $TEST_PID 2>/dev/null; do sleep 10; done
  wait $TEST_PID
  if [ $? -ne 0 ]; then
    echo "ERROR: full test suite failed after rebase"
    cat .local/merge-test-output.txt
    exit 1
  fi
  echo "FULL TEST SUITE PASSED"
  
  # Get the PR head branch name and push
  head=$(gh pr view <PR> --json headRefName --jq .headRefName)
  head_repo=$(gh pr view <PR> --json headRepository --jq .headRepository.nameWithOwner)
  git push --force-with-lease "https://github.com/$head_repo.git" HEAD:"$head"
  
  echo "Rebased, gates passed, pushed. Waiting 15s for CI to pick up new commit..."
  sleep 15
fi

# Checks (after potential rebase)
gh pr checks <PR>
```

If required checks are failing, stop and say to run /preparepr before merge.

## Step 7: Merge PR (squash and delete branch)

This is the atomic, serial merge step. Do not parallelize.

```sh
# Check if any checks are still running
check_status=$(gh pr checks <PR> 2>&1)
if echo "$check_status" | grep -q "pending\|queued"; then
  echo "Checks still running, using --auto to queue merge"
  gh pr merge <PR> --squash --delete-branch --auto
  echo "Merge queued. Monitor with: gh pr checks <PR> --watch"
else
  gh pr merge <PR> --squash --delete-branch
fi
```

If merge fails, report the error and stop. Do not retry in a loop.

## Step 8: Get merge sha and verify state

```sh
merge_sha=$(gh pr view <PR> --json mergeCommit --jq '.mergeCommit.oid')
echo "merge_sha=$merge_sha"

pr_state=$(gh pr view <PR> --json state --jq .state)
echo "pr_state=$pr_state"

if [ "$pr_state" != "MERGED" ]; then
  echo "ERROR: PR is not in MERGED state, something went wrong"
  exit 1
fi
```

## Step 9: Comment on PR (identity-aware)

Use the identity check from Step 4. Follow the identity-aware commenting rules from the top of this file.

```sh
if [ "$SELF_AUTHORED" = "true" ]; then
  gh pr comment <PR> --body "Merged via squash.

- Merge commit: $merge_sha"
else
  gh pr comment <PR> --body "Merged via squash.

- Merge commit: $merge_sha

Thanks @$contrib!"
fi
```

## Step 10: SINGLE POST-MERGE CLEANUP SUB-SUBAGENT

Now that the PR is merged, spawn ONE cleanup sub-subagent that handles all post-merge tasks serially. This is ~5 API calls total, so a single agent is faster than spawning 3 separate agents.

**Important:** Save the data from .local/ that the cleanup sub-subagent needs BEFORE spawning it, because it will delete the worktree (and .local/ with it). Pass the data inline in the task description.

First, read .local/related.md and extract the data:
```sh
related_content=""
if [ -f .local/related.md ]; then
  related_content=$(cat .local/related.md)
fi
echo "$related_content"
```

Spawn with EXACTLY this call:
```
sessions_spawn model:gpt-fast label:"pr-<PR>-cleanup" runTimeoutSeconds:0 task:"
Post-merge cleanup for PR #<PR> in openclaw/openclaw.

Merged PR URL: <pr_url>
Merge commit: <merge_sha>
PR title: <pr_title>

GitHub identity context:
- Logged in as: <gh_user>
- PR #<PR> author: <contrib>

IDENTITY-AWARE COMMENTING RULES:
- Before commenting on each item, check who authored it.
- If <gh_user> (case-insensitive) == that item's author: use self-referential language, do NOT thank yourself.
- If <gh_user> (case-insensitive) != that item's author: be polite, thank them.
- Use: author=$(gh pr view <NUM> --json author --jq .author.login) or gh issue view <NUM> --json author --jq .author.login
- Compare with: [ \"$(echo '<gh_user>' | tr A-Z a-z)\" = \"$(echo \"$author\" | tr A-Z a-z)\" ]

Do all three tasks below IN ORDER:

---

TASK 1: Close superseded/duplicate PRs

Related data from reviewpr:
<paste the 'PRs to close after merge' section here, or 'none'>

For each PR listed:
1. Verify it is still open: gh pr view <NUM> --json state --jq .state
2. Check who authored it: gh pr view <NUM> --json author --jq .author.login
3. If open, close with identity-aware comment:
   - Self: 'Closing this out, superseded by <pr_url> (merge commit: <merge_sha>).'
   - Other: 'Superseded by <pr_url> (merge commit: <merge_sha>). Thanks for the contribution! Closing to reduce duplicate PR noise.'
4. If already closed/merged, skip it

Also do a quick search for obvious duplicates not in the list:
title='<pr_title>'
q1=$(echo \"$title\" | sed -E 's/\([^)]*\)//g' | tr -s ' ' | cut -c1-80)
gh search prs --repo openclaw/openclaw --state open --limit 10 --json number,title --search \"$q1\" || true
Close any clear duplicates found (with identity-aware comments).

---

TASK 2: Close/link related issues

Related data from reviewpr:
<paste the 'Issues to close after merge' and 'Body references' sections here, or 'none'>

For each issue listed:
1. Verify it is still open: gh issue view <NUM> --json state --jq .state
2. Check who authored it: gh issue view <NUM> --json author --jq .author.login
3. If open and clearly resolved, close with identity-aware comment:
   - Self: 'Resolved by <pr_url> (merge commit: <merge_sha>). Closing.'
   - Other: 'Resolved by <pr_url> (merge commit: <merge_sha>). Thanks for the report!'
4. If broader scope than what this PR fixed, comment but leave open:
   - Self: 'Partially addressed by <pr_url> (merge commit: <merge_sha>). Remaining scope: <describe>.'
   - Other: 'Partially addressed by <pr_url> (merge commit: <merge_sha>). Thanks for the report! Remaining scope: <describe>.'
5. If already closed, skip it

---

TASK 3: Worktree cleanup

1. Remove the worktree:
   cd ~/Development/openclaw
   git worktree remove '.worktrees/pr-<PR>' --force

2. Clean up local branches:
   git branch -D 'pr/<PR>' 2>/dev/null || true
   git branch -D pr-<PR> 2>/dev/null || true

3. Return main repo checkout to main (best effort):
   if git diff --quiet && git diff --cached --quiet; then
     git switch main 2>/dev/null || git checkout main || true
     git pull --ff-only origin main 2>/dev/null || true
   else
     echo 'NOTE: local changes present, not switching branches'
     git status -sb || true
   fi

4. Remove the prhead remote:
   git remote remove prhead 2>/dev/null || true

Report what you did for each task.
"
```

## Step 11: End turn and wait for cleanup sub-subagent

DO NOT POLL. DO NOT LOOP. DO NOT call subagents list or sessions_list.

The cleanup sub-subagent will auto-announce its results back to you as a user message when complete. Once it announces back, collect its results.

## Step 12: Output

Final output should include:
- merge_sha
- PR state (MERGED)
- What cleanup was done:
  - Which superseded PRs were closed
  - Which issues were closed or commented on
  - Worktree cleanup status
- Any cleanup items that failed or were skipped

Format:
```
PR #<PR> merged successfully.

Merge commit: <merge_sha>
Contributor: @<contrib>

Post-merge cleanup:
- Closed PRs: #X, #Y (superseded)
- Closed issues: #A, #B (resolved)
- Commented on: #C (partial fix)
- Worktree: cleaned up
```

Note: if SELF_AUTHORED was true, do not include "Thanks @contrib!" in the final output.

Rules
- Worktree only (until cleanup)
- PR must end in MERGED state
- Only cleanup after merge success
- NEVER push to main. `gh pr merge --squash` is the only path to main.
- Do NOT run `git push` at all in this command.
- Always use identity-aware commenting (case-insensitive username comparison before every comment).
