---
description: Generate unit testing plan for a file
---

You are tasked with generating a concise, execution-ordered unit testing plan for a code file.

**Command accepts a required parameter**: `/testing-plan <file-path>[#L<line-number>]`
- `file-path`: Path to the file to generate tests for
- `#L<line-number>` (optional): Specific line number to focus on

**Examples:**
- `/testing-plan src/utils/payment.ts` - Generate plan for entire file
- `/testing-plan contracts/Token.sol#L42` - Generate plan focused on function at line 42

## 1. Parse and Validate Parameters

**Parse the parameter:**
- Split on `#L` to extract file path and optional line number
- Extract line number if present (e.g., `#L42` → `42`)

**Validate:**
- Check if file exists and is readable
- If line number provided, verify it's within file bounds
- If validation fails, show clear error and exit

## 2. Detect Testing Framework

Identify the testing framework from project context:

```bash
# Check for Hardhat (smart contracts)
if [ -f "hardhat.config.ts" ] || [ -f "hardhat.config.js" ] || ls *.sol 2>/dev/null | grep -q .; then
  echo "hardhat"
# Check for Vitest
elif [ -f "vitest.config.ts" ] || [ -f "vitest.config.js" ] || grep -q '"vitest"' package.json 2>/dev/null; then
  echo "vitest"
# Check for Jest
elif [ -f "jest.config.ts" ] || [ -f "jest.config.js" ] || grep -q '"jest"' package.json 2>/dev/null; then
  echo "jest"
else
  echo "unknown"
fi
```

Store the detected framework for later use.

## 3. Analyze Existing Test Files

Find and analyze existing test files to understand patterns:

**Find test files:**
- Look for `*.test.ts`, `*.spec.ts`, `*.test.sol` files in the same directory as target file
- If none found, check parent directory
- Read 2-3 most relevant test files (prioritize files testing similar functionality)

**Extract reusable patterns:**
- Mock implementations (e.g., `MockAPIClient`, contract mocks)
- Common fixtures and setup functions (`beforeEach`, `describe` blocks, deployment functions)
- Testing utilities and helpers
- Contract deployment patterns (for Hardhat)
- Signer/account setup patterns

**Note these for prerequisites section:**
- Existing mocks/fixtures that can be reused
- Common setup patterns to follow
- Only suggest creating new infrastructure when existing patterns don't cover the need

## 4. Analyze Target Code

**Read the target file** and understand its structure.

**If line number was provided:**
1. Identify the function/method at that line
2. Analyze:
   - Function size and complexity
   - Dependencies on other functions in the file
   - Whether it's a public/external entry point vs internal helper
3. **Decide scope** based on context:
   - **Test only that function** if it's self-contained or a clear entry point
   - **Test entire file** if functions are tightly coupled
   - **Test logical group** if the function is part of a cohesive unit with nearby functions

Inform the user of your scope decision:
- "Planning tests for `transferTokens` function and its dependencies"
- "Planning tests for entire file (functions are tightly coupled)"

**If no line number:**
- Plan tests for the entire file

## 5. Check for Existing Testing Plan

**Check if a testing plan already exists:**

```bash
# Generate expected plan filename
PLAN_FILE="${FILE_PATH%.*}-testing-plan.md"

# Check if it exists
if [ -f "$PLAN_FILE" ]; then
  echo "exists"
else
  echo "new"
fi
```

**If plan exists:**
- Ask user: "Found existing plan at `filename-testing-plan.md`. Overwrite? (Yes/No)"
- Use the AskUserQuestion tool for this
- If user says No: Exit gracefully with message "Keeping existing plan"
- If user says Yes: Proceed with generation

## 6. Generate Testing Plan

Create a structured markdown file with this format:

```markdown
# Testing Plan: [FileName or FunctionName]

**File:** `path/to/file.ts`
**Framework:** [Detected Framework]
[**Focus:** `functionName` at line X] (only include if line number was provided)

## Prerequisites

- [ ] Setup item 1 (e.g., "Create mock for ExternalAPI")
- [ ] Setup item 2 (e.g., "Reuse `deployTestContract` from Contract.test.sol")

Notes:
- Reference existing test utilities found in step 3
- Note any special setup requirements

## Test Case 1: [Descriptive test name]

**What:** High-level description of what this test validates

**How:**
1. Step-by-step approach
2. Setup requirements
3. Expected outcome

**Progress:**
- [ ] Test written
- [ ] Test approved

## Test Case 2: [Next test in execution order]
...
```

**Test Planning Principles:**

**Conciseness (avoid redundant tests):**
- No duplicate coverage - don't test the same code path twice
- Combine related scenarios - group related validations
- Skip obvious cases - don't test framework features or library behavior
- Focus on meaningful assertions - business logic, state changes, critical error paths

**Execution-Order Testing:**
Order ALL tests by execution flow:
1. Early validation/guard clauses first
2. Main logic steps in order they execute
3. Edge cases in context of execution flow
4. Integration/end-to-end tests last

Example for this function:
```typescript
function processPayment(amount: number) {
  if (amount <= 0) throw new Error("Invalid amount");
  const fee = calculateFee(amount);
  const total = amount + fee;
  return executeTransaction(total);
}
```

Test order:
1. Rejects invalid amount (first guard)
2. Calculates fee correctly (next step)
3. Executes transaction with correct total (final step)
4. End-to-end successful payment flow (integration)

**User Feedback:**
Show progress as you work:
- "Analyzing `MyContract.sol`..."
- "Found existing test patterns in `OtherContract.test.sol`"
- "Planning tests for `transferTokens` function and its dependencies"

## 7. Write Plan File

**Generate filename:**
- Input: `packages/contracts/Token.sol`
- Output: `packages/contracts/Token-testing-plan.md`
- Pattern: `${filename_without_extension}-testing-plan.md` in same directory

**Write the plan** to the generated filename.

## 8. Offer to Start Implementation

After writing the plan:

1. Show summary: "Testing plan created at `path/to/file-testing-plan.md`"
2. Display high-level overview of what was planned:
   - Number of prerequisites
   - Number of test cases
   - Brief mention of what's being tested
3. Ask: "Ready to implement the first test?"
   - Use the AskUserQuestion tool
   - If Yes: Read the testing plan, mark first test as in-progress, and begin implementing it
   - If No: Exit with "You can start implementing tests by asking me to work through the testing plan"

## Important Notes

- **Be concise** - Avoid redundant tests, focus on meaningful coverage
- **Follow execution order** - Tests mirror the code's execution flow for readability
- **Reuse existing patterns** - Reference mocks/fixtures from existing tests when possible
- **Adaptive scope** - When line number provided, intelligently decide what to test based on coupling
- **Clear communication** - Keep user informed of what you're analyzing and deciding
- **Respect existing plans** - Always ask before overwriting
