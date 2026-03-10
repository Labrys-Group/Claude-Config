# MSW Opt-In Pattern

## 1) Bootstrap Gate + Dynamic Import

```ts
// src/mocks/startMocking.ts
let startPromise: Promise<void> | null = null;

export const shouldStartMocking = () => {
  if (typeof window === "undefined") return false;
  if (process.env.NODE_ENV === "production") return false;
  if (process.env.NEXT_PUBLIC_ENABLE_MSW !== "true") return false;

  const params = new URLSearchParams(window.location.search);
  return params.get("mockApi") === "1";
};

export const startMocking = async () => {
  if (!shouldStartMocking()) return;
  if (startPromise) return startPromise;

  startPromise = (async () => {
    const params = new URLSearchParams(window.location.search);
    setMockScenario(params.get("mockScenario"));

    try {
      const { worker } = await import("./browser");
      await worker.start({ onUnhandledRequest: "bypass" });
    } catch (error) {
      console.error(
        "[MSW] Failed to start. Run `yarn msw init public/ --save`.",
        error,
      );
      startPromise = null;
    }
  })();

  return startPromise;
};
```

## 2) Worker Setup

```ts
// src/mocks/browser.ts
import { setupWorker } from "msw/browser";

import { handlers } from "./handlers";

export const worker = setupWorker(...handlers);
```

## 3) Root Integration (Single Call Site)

```tsx
// src/providers/Providers.tsx (or root client layout)
useEffect(() => {
  void startMocking();
}, []);
```

## 4) Query-Param Configuration with Validation

```ts
// src/mocks/config.ts
export type MockScenario = "default" | "edge-cases";

let currentScenario: MockScenario = "default";

export const setMockScenario = (scenario: string | null) => {
  currentScenario = scenario === "edge-cases" ? "edge-cases" : "default";
};

export const getMockScenario = () => currentScenario;
```

## 5) Minimal Handler Strategy

- Start with only target endpoints for the QA flow.
- Keep handler list centralized (`handlers/index.ts`).
- Prefer deterministic fixture factories over ad-hoc random data.

## 6) Suggested File Layout

```text
src/mocks/
  browser.ts
  startMocking.ts
  config.ts
  handlers/
    index.ts
    featureA.ts
  data/
    featureAFactory.ts
public/
  mockServiceWorker.js
```

## 7) Environment + URL Contract

- Env gate example: `NEXT_PUBLIC_ENABLE_MSW=true`
- URL gate example: `?mockApi=1`
- Optional scenario: `?mockApi=1&mockScenario=edge-cases`

## 8) Validation Checklist

- `dev` without env flag: worker does not start.
- QA task with env flag but no `mockApi=1`: worker does not start.
- QA task with env flag and `mockApi=1`: worker starts and intercepts targeted endpoints.
- Production build/run: worker never starts.
- Unhandled requests pass through when using `onUnhandledRequest: "bypass"`.

## 9) Common Mistakes to Reject

- Importing mock handlers from feature code.
- Enabling mocks by env var alone (no per-session URL opt-in).
- Omitting production guard.
- Starting worker from multiple components.
- Mocking broad API surface before confirming QA scope.

## 10) Validated Override Example (Address Param)

```ts
const ETH_ADDRESS_REGEX = /^0x[a-fA-F0-9]{40}$/;
let mockWithdrawalAddress: string | null = null;

export const setMockWithdrawalAddress = (address: string | null) => {
  mockWithdrawalAddress =
    address && ETH_ADDRESS_REGEX.test(address) ? address : null;
};
```

Use this for high-impact override params so invalid URLs fall back safely.
