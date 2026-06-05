---
name: contract-watchman
description: Review smart contract development and deployment workflows for deployment hygiene, environment handling, human release sign-off, hard-coded addresses, and Greptile/code-review guardrails. Use when working on Solidity, Foundry, Hardhat, deployment scripts, contract release pipelines, environment config, address registries, .env files, Greptile custom instructions, or smart contract PR/release reviews.
---

# Contract Watchman

## Review Workflow

Use this skill as a release-readiness and code-review lens for smart contract work. Bias toward concrete findings with file and line references, then propose small fixes that fit the repository's framework.

1. Map the deployment flow before editing.
   - Identify the contract framework: Foundry, Hardhat, Forge scripts, Ignition, custom TypeScript scripts, CI, multisig, or deployment service.
   - Find deployment entrypoints with `rg "deploy|script|broadcast|ignition|CREATE2|upgrade|proxy|verify|signer|privateKey|PRIVATE_KEY"`.
   - Find config sources with `rg "prod\\.json|production|development|\\.env|process\\.env|vm\\.env|dotenv|config"`.

2. Enforce one deployment script with environment-driven parameters.
   - Prefer one deployment entrypoint that branches on an explicit environment name, chain ID, or env-loaded config.
   - Flag separate `deploy-prod`, `deploy-staging`, `deploy-mainnet`, or copied deployment scripts when the differences are only addresses, RPC URLs, salts, owners, or constructor params.
   - Require parameters to come from `.env.production`, `.env.development`, CI secrets, or the repository's established env loader.
   - Flag `prod.json`, `mainnet.json`, checked-in secret-bearing config, and hand-maintained environment JSON files unless they are generated artifacts or non-sensitive chain metadata.

3. Require human sign-off for contract deployments.
   - Verify deployment workflows have an explicit human approval gate before live-network deployment or upgrade execution.
   - Accept examples: GitHub protected environments, manual workflow dispatch with required reviewers, multisig confirmation, release checklist approval, or deployment PR sign-off from a human reviewer.
   - Flag autonomous deploy-on-merge flows to production/mainnet, direct private-key deploys from CI without review gates, and missing reviewer evidence.

4. Detect and check hard-coded addresses and constants.
   - Search for Ethereum-style addresses, long hex constants, private keys, mnemonic usage, chain IDs, RPC URLs, and owner/admin addresses.
   - Classify each finding: allowed named protocol constant, generated artifact, test fixture, address registry entry, or unsafe hard-coded deployment parameter.
   - Require unsafe constants to move into typed config/env or a clearly named registry with chain scoping and validation.
   - Cross-check addresses against known network, chain ID, checksum casing, intended role, and comments/tests that explain why the address is stable.

5. Add or update automated guardrails.
   - Prefer repository-native tests/checks over prose-only guidance.
   - Add static checks only when they are maintainable: focused regex tests, lint rules, CI grep checks, or config validation tests.
   - For Greptile, add concise custom review instructions that tell it what to flag and what to ignore; use `references/review-checklist.md` for copy-ready content.

## Output Style

When reviewing, lead with findings ordered by deployment risk. Include:

- `Finding`: risk and impact.
- `Evidence`: file and line references.
- `Fix`: the smallest code or workflow change.
- `Verification`: command, review gate, or checklist item that proves the fix.

If making changes, preserve the repo's framework and deployment conventions. Do not introduce a new deployment tool unless the current flow cannot support the required controls.

## Reference

Read `references/review-checklist.md` when you need concrete search patterns, hard-coded address heuristics, environment examples, sign-off requirements, or Greptile custom instruction text.
