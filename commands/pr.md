---
description: Generate PR description and create/update pull request
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

**If a base branch was provided as a parameter to /pr, use that.**

**Otherwise, auto-detect using this logic:**

1. Try to infer from existing PR or branch configuration:
```bash
# Check if there's already a PR and get its base branch
gh pr view --json baseRefName -q '.baseRefName' 2>/dev/null || echo ""
```

2. If no existing PR, use the repository's default branch (from step 1):
   - This is typically `main`, `master`, or `development`
   - This is the most reliable fallback since feature branches are usually created from the default branch

The base branch will be used for generating diffs and creating the PR.

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

## 5. Generate PR Description

**If a PR template was found in step 3:**
- Use the template structure as the base
- Fill in each section of the template with relevant information from the commits and diff
- Replace HTML comments with actual content
- Keep all template sections, even if some are brief

**If no PR template was found:**
- Generate a description with these sections: Summary, Changes, Technical Details, Testing, Breaking Changes

**In both cases:**
- Analyze the full diff, not just commit messages
- Be thorough but concise
- Use proper markdown formatting
- Include checkboxes for testing items

## 6. Check for Existing PR

Check if a PR already exists for this branch:

```bash
gh pr view --json number,title,body 2>&1
```

## 7. Create or Update PR

**IMPORTANT: Do NOT ask for user confirmation. Automatically proceed with creating or updating the PR.**

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
- Create the PR immediately using the base branch determined in step 2 (no confirmation needed):
```bash
gh pr create --base <base-branch> --title "[generated title]" --body "$(cat <<'EOF'
[generated description]
EOF
)"
```
- Display the created PR URL to the user

## Important Notes

- NEVER run `git push` without explicit user permission
- If the current branch is not pushed to remote, inform the user and ask if they want to push it first
- Be thorough in analyzing the diff - look at all changes, not just commit messages
- If there are a large number of changes, summarize by file/module rather than listing every single change
- Ensure the PR description is well-formatted and professional
