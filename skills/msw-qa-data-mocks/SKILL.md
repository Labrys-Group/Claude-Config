---
name: msw-qa-data-mocks
description: Implement an opt-in Mock Service Worker setup for browser QA/dev flows with strict production isolation. Use when adding or refactoring MSW in React/Next/Vite apps, especially when activation must be controlled by both environment variables and URL query params, with mock code kept outside production feature paths.
---

# MSW Opt-In QA Data Mocks

Implement MSW as an explicit QA layer that only starts when all gates pass. Keep mock implementation isolated in a dedicated mocks module and wire a single bootstrap call in a client-only root entrypoint.

## Workflow

1. Define activation gates first.

- Require browser context (`window` exists).
- Require non-production runtime (`NODE_ENV !== "production"`).
- Require explicit env opt-in (for example `NEXT_PUBLIC_ENABLE_MSW=true`).
- Require explicit query-param opt-in (for example `?mockApi=1`).

2. Isolate all mock code under one module tree.

- Create a folder such as `src/mocks/`.
- Keep `browser.ts`, `handlers/`, `data/`, and config/state helpers there.
- Keep production feature modules unaware of handler/data internals.

3. Start MSW via dynamic import in one place.

- Add `startMocking()` that checks gates, configures scenario params, then dynamically imports `./browser`.
- Use a module-level promise to avoid duplicate `worker.start()` calls.
- Start with `onUnhandledRequest: "bypass"` unless strict failure is explicitly requested.

4. Wire bootstrap once in a client root boundary.

- In one top-level client provider/layout effect, call `void startMocking()`.
- Do not scatter MSW startup calls across feature components.

5. Support runtime configuration through query params and env vars.

- Env var controls global availability (safe default off).
- Query params control per-session activation and scenario overrides.
- Validate query-param values before applying them.

6. Implement only needed handlers.

- Mock only endpoints required for the target QA flow.
- Keep responses deterministic where possible.
- Add scenario switches for edge-case datasets when needed.

7. Add developer ergonomics without production coupling.

- Add a dedicated QA task/script that sets the env flag.
- Document URL toggles (`mockApi`, scenario params) in README.
- Keep normal `dev` and all production paths mock-free by default.

## Non-Negotiable Guardrails

- Never start mocks in production.
- Never statically import browser worker startup into server execution paths.
- Never require mock-only imports inside business/domain modules.
- Keep a single integration point from app code to mocks (`startMocking`).
- Fail gracefully when worker assets are missing; log remediation command.

## Reference

- Read `references/msw-opt-in-pattern.md` for copy-ready snippets and a validation checklist.
