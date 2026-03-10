---
description: Generate PR description and create/update pull request
disable-model-invocation: true
argument-hint: "[base-branch]"
---

You are tasked with generating a comprehensive pull request description and either creating a new PR or updating an existing one.

**Command accepts an optional parameter**: `/pr [base-branch]`
- If `base-branch` is provided, use it as the target branch
- If not provided, auto-detect the base branch

Follow these steps:

## 1. Gather Branch Information

Run these git commands in parallel to understand the current state:

```bash
# Get current branch name
git rev-parse --abbrev-ref HEAD

# Get the default branch (usually main or master)
git remote show origin | grep 'HEAD branch' | cut -d' ' -f5

# Get git status
git status
```

## 2. Determine the Base Branch

**If a base branch was provided as a parameter, use it immediately — skip the detection below and go to step 3.**

**Otherwise, auto-detect in this order (stop at the first that works):**

1. Check for an existing PR on this branch:
```bash
gh pr view --json baseRefName -q '.baseRefName' 2>/dev/null
```

2. Get the remote HEAD branch (the repo's default):
```bash
git remote show origin 2>/dev/null | grep 'HEAD branch' | cut -d' ' -f5
```

3. Check the remote tracking branch for the current branch:
```bash
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null | cut -d'/' -f2
```

4. Fall back to `main` if all else fails — but note this in your output so the user can correct it if wrong.

The detected base branch will be used for generating diffs and creating the PR.

## 2.5. Check if Branch is Pushed

Before proceeding, check whether the current branch is tracked by a remote:

```bash
git branch -vv | grep "$(git rev-parse --abbrev-ref HEAD)"
```

If the output shows no remote tracking branch (no `[origin/...]` in the result):
- Tell the user: "Branch `<name>` is not pushed to remote. You'll need to push it before a PR can be created."
- Ask: "Push now, or continue to generate the description only?"
- If they say push: do NOT push — ask them to run `git push -u origin <branch>` themselves (never push autonomously)
- If they say continue: generate the description and skip steps 6–7

## 3. Check for PR Template

Check for the repository's PR template:

```bash
# Check for PR template in common locations
if [ -f .github/PULL_REQUEST_TEMPLATE.md ]; then
  cat .github/PULL_REQUEST_TEMPLATE.md
elif [ -f .github/pull_request_template.md ]; then
  cat .github/pull_request_template.md
elif [ -f docs/PULL_REQUEST_TEMPLATE.md ]; then
  cat docs/PULL_REQUEST_TEMPLATE.md
elif [ -f PULL_REQUEST_TEMPLATE.md ]; then
  cat PULL_REQUEST_TEMPLATE.md
else
  echo "No PR template found"
fi
```

## 4. Analyze the Changes

Get comprehensive information about all changes since the branch point:

```bash
# Get all commits in this branch (not in base branch)
git log <base-branch>..HEAD --oneline

# Get the full diff between base branch and current branch
git diff <base-branch>...HEAD

# Get list of changed files with stats
git diff <base-branch>...HEAD --stat
```

## 5. Generate PR Title and Description

### Title

Generate a title that:
- Is under 72 characters
- Leads with the most significant change (not a file list)
- Uses a conventional commit prefix if the commit history does (`feat:`, `fix:`, `refactor:`, `chore:`, etc.)
- Names the *impact*, not just the mechanism — e.g., "Add path-scoped rules to reduce context waste" rather than "Update rules/react.md frontmatter"

### Description

**If a PR template was found in step 3:**
- Use the template structure as the base
- Fill in each section with information drawn from the actual diff — not from commit messages alone
- Replace HTML comments with real content; keep all sections even if brief

**If no PR template was found:**
- Generate a description with these sections: Summary, Changes, Technical Details, Testing, Breaking Changes

**In both cases:**
- Ground the description in the diff from step 4 — reference specific files, specific values, specific behavior changes
- Explain the *why* (motivation, problem being solved) in the Summary, not just the *what*
- Be thorough but concise
- Include checkboxes for testing items

## 6. Check for Existing PR

Check if a PR already exists for this branch:

```bash
gh pr view --json number,title,body 2>&1
```

## 7. Create or Update PR

**Do NOT ask for confirmation about PR content or title — proceed automatically.**

**If PR exists:**
- Update the PR description with the generated content using:
```bash
gh pr edit --body "$(cat <<'EOF'
[generated description]
EOF
)"
```
- Display the PR URL to the user

**If no PR exists:**
- Create the PR immediately using the base branch determined in step 2:
```bash
gh pr create --base <base-branch> --title "[generated title]" --body "$(cat <<'EOF'
[generated description]
EOF
)"
```
- Display the created PR URL to the user

## Important Notes

- **PR content**: Auto-create or update without asking for confirmation on content or title.
- **git push**: NEVER push autonomously. Step 2.5 handles this check proactively.
- If there are a large number of changes, summarize by file/module rather than listing every single change
- Ensure the PR description is well-formatted and professional
