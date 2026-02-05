---
description: Run CI checks locally
---

You are tasked with running CI checks locally by analyzing GitHub Actions workflow files.

## 1. Find Workflow Files

Check for workflow files:

```bash
find .github/workflows -type f \( -name "*.yml" -o -name "*.yaml" \) 2>/dev/null || echo "No workflow files found"
```

If no files found, display: "No CI workflow files found in `.github/workflows/`" and exit.

## 2. Analyze Workflows

Read all workflow files and identify locally-runnable jobs. Look for common CI steps:

- **Format checks**: Look for `prettier`, `format`, `fmt` commands
- **Lint checks**: Look for `eslint`, `lint`, `clippy` commands
- **Type checks**: Look for `typecheck`, `tsc`, `mypy` commands
- **Tests**: Look for `test`, `jest`, `vitest`, `pytest`, `cargo test` commands
- **Build**: Look for `build`, `compile` commands

**Skip these (not locally runnable):**
- Deployment steps (deploy, publish, release)
- Cloud-specific actions (AWS, GCP, Azure operations)
- GitHub-specific actions (creating releases, commenting on PRs)
- Docker builds/pushes to registries
- Steps that require secrets/credentials not available locally

## 3. Extract Commands

For each runnable job, extract the actual shell commands. Common patterns:

- `run:` steps in GitHub Actions
- `pnpm <command>` / `npm run <command>` / `yarn <command>`
- Direct tool invocations like `prettier --check`, `eslint .`, etc.

Normalize commands to work locally (remove CI-specific flags if needed).

## 4. Run Checks in Parallel

Execute all identified checks in parallel using the Bash tool with multiple parallel calls.

For example, if you find these commands:
- `pnpm format`
- `pnpm lint`
- `pnpm typecheck`
- `pnpm test`

Run them all at once in a single response with multiple Bash tool calls.

## 5. Handle Failures

Based on the type of failure, take appropriate action:

### Format Failures
If format check fails (e.g., `pnpm format` or `prettier --check`):
- Automatically run the fix command: `pnpm format:fix` or `pnpm format -- --write`
- No need to report anything unless the fix itself fails

### Lint Failures
If linting fails (e.g., `pnpm lint`, `eslint`):
- First try auto-fix: `pnpm lint:fix` or `eslint --fix`
- If auto-fix doesn't resolve everything, read the files with remaining errors
- Fix ONLY obvious issues:
  - Unused imports
  - Unused variables (if clearly safe to remove)
  - Missing semicolons
  - Simple rule violations (prefer-const, no-console in production, etc.)
  - Do NOT change any logic
  - Do NOT refactor code structure
- Add any non-obvious issues to the issues list

### Type Check Failures
If type checking fails (e.g., `pnpm typecheck`, `tsc`):
- Read files with type errors
- Fix ONLY obvious issues:
  - Missing type imports
  - Obvious type annotations
  - Simple null/undefined checks where intent is clear
  - Do NOT add `any` types
  - Do NOT change function signatures that might affect consumers
- Add any non-obvious issues to the issues list

### Test Failures
If tests fail:
- Read the test output and failing test files
- Fix ONLY obvious issues:
  - Import errors in tests
  - Simple assertion fixes where expected value is clearly wrong in the test
  - Do NOT modify application logic to make tests pass
  - Do NOT disable or skip tests
- Add any non-obvious test failures to the issues list

### Build Failures
If build fails:
- Read the build errors
- Fix ONLY obvious issues:
  - Missing imports
  - Simple syntax errors
  - Do NOT restructure code
  - Do NOT change build configuration without user approval
- Add any non-obvious issues to the issues list

## 6. Report Results

After all checks complete and fixes are attempted, provide a summary:

```markdown
## CI Check Results

### Passing Checks
- Format
- Lint
- Typecheck

### Fixed Issues
- **Format**: Auto-fixed 3 files with formatting issues
- **Lint**: Fixed 2 unused imports in `src/utils/helper.ts`

### Issues Requiring Review
1. **Type Error** in `src/services/api.ts:45`
   - `Property 'username' does not exist on type 'User'`
   - Needs investigation - unclear if this is a missing field or incorrect usage

2. **Test Failure**: `user.test.ts` - "should create user with valid email"
   - Test expects email validation but implementation allows any string
   - Requires decision: update test or add validation?
```

## Important Notes

- **Be conservative with fixes**: When in doubt, add to the issues list rather than making changes
- **Never change business logic**: Only fix mechanical/syntactic issues
- **Run commands in parallel** when possible to save time
- **If a workflow uses custom scripts**, read them to understand what they do before running
- **Don't run commands that might be destructive** (database migrations, deployments, etc.)
