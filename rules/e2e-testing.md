# E2E Testing with Playwright

## Setup

### 1. Install dependencies

From the app directory (e.g. `apps/nextjs`):

```bash
pnpm add -D @playwright/test
pnpm exec playwright install chromium
```

### 2. Add `playwright.config.ts`

Create at the app root (e.g. `apps/nextjs/playwright.config.ts`):

```ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./src",
  testMatch: "**/*.e2e.ts",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: "html",
  use: {
    baseURL: "http://localhost:3939",
    trace: "on-first-retry",
  },
  projects: [
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"] },
    },
  ],
  webServer: {
    command: "pnpm dev --port 3939",
    url: "http://localhost:3939",
    reuseExistingServer: !process.env.CI,
  },
});
```

Key decisions:

- **`testMatch: "**/\*.e2e.ts"`** — uses `.e2e.ts` suffix so vitest doesn't pick up e2e tests
- **`testDir: "./src"`** — tests are co-located with source files, not in a separate directory
- **Port 3939** — avoids conflicts with other local dev servers on 3000
- **`webServer`** — Playwright starts the dev server automatically and waits for it

### 3. Add scripts to `package.json`

```json
{
  "scripts": {
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"
  }
}
```

### 4. Add to `.gitignore`

```
playwright-report/
test-results/
```

## Writing Tests

### File naming and location

Tests live next to the page they test, using the `.e2e.ts` suffix:

```
app/example/
  page.tsx
  example.e2e.ts        # <-- e2e test for this page
  _components/
    contact-form.tsx
```

### Test structure

```ts
import { expect, test } from "@playwright/test";

test.describe("feature name", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/your-route");
  });

  test("description of behavior", async ({ page }) => {
    // interact and assert
  });
});
```

### Locating elements

Prefer accessible locators over CSS selectors:

```ts
// By role
page.getByRole("button", { name: "Submit" });
page.getByRole("heading", { name: /Welcome/i });

// By label (for form fields)
page.getByLabel("Email");

// By text content
page.getByText("Success message");

// By placeholder
page.getByPlaceholder("Enter your name");
```

### Common interactions

```ts
// Fill a form field
await page.getByLabel("Name").fill("Jane Doe");

// Click a button
await page.getByRole("button", { name: "Submit" }).click();

// Navigate
await page.goto("/some-page");
```

### Common assertions

```ts
// Element is visible
await expect(page.getByText("Hello")).toBeVisible();

// Input has specific value
await expect(page.getByLabel("Name")).toHaveValue("Jane Doe");

// Element is not visible
await expect(page.getByText("Error")).not.toBeVisible();
```

## Best Practices

### Use accessible locators, not test IDs

Prefer `getByRole`, `getByLabel`, and `getByText` over `data-testid` attributes.

**Why:**

- **Tests reflect real user experience.** Accessible locators find elements the way a user would — by visible text, labels, and roles. If a button's text changes from "Submit" to "Send", the test fails, which is correct — it caught a UX change.
- **Test IDs hide broken UI.** A test using `data-testid="submit-btn"` passes even if the button text is deleted or becomes nonsensical. The test gives false confidence.
- **Accessible locators double as accessibility audits.** If you can't locate an element by role or label, that's a signal the markup needs better accessibility attributes.

```ts
// Preferred — tests what the user sees
page.getByRole("button", { name: "Submit" });
page.getByLabel("Email");

// Avoid — invisible to users, hides broken UI
page.getByTestId("submit-btn");
page.getByTestId("email-input");
```

**When `data-testid` is appropriate:** Elements that genuinely lack meaningful text or roles — a canvas, a complex custom widget, or a container with no semantic meaning.

### Keep tests independent

Each test should set up its own state. Don't rely on test execution order or shared state between tests. Use `test.beforeEach` for common navigation, not for shared data setup that only some tests need.

### Test user-visible behavior, not implementation

Assert on what the user sees (text, visibility, values), not on internal state, CSS classes, or DOM structure.

```ts
// Good — asserts on user-visible outcome
await expect(page.getByText("Thanks for your message!")).toBeVisible();

// Avoid — asserts on implementation details
await expect(page.locator(".success-container")).toHaveClass("visible");
```

### Gotchas

- **Avoid `type="email"` on inputs** if you're using Zod for validation. Native browser validation runs before your JS validation and silently blocks form submission in Playwright, so validation errors never appear.
- **All assertions are auto-waiting** — Playwright retries assertions until they pass or timeout. No need for manual waits.
- **`fill()` clears the field first** — no need to clear before filling.

## Running Tests

```bash
# Headless (CI-friendly)
pnpm test:e2e

# Interactive UI mode (useful during development)
pnpm test:e2e:ui

# Run a specific test file
pnpm test:e2e -- src/app/example/example.e2e.ts
```
