# Contract Watchman Review Checklist

Use this reference when reviewing or changing smart contract deployment workflows.

## Deployment Script Standard

Require one canonical deployment entrypoint per deployment type, not one copied script per environment.

Acceptable:

- One `Deploy.s.sol`, `deploy.ts`, or equivalent script that reads an explicit `DEPLOY_ENV`, `CHAIN_ID`, or network name.
- Shared deployment logic with small per-contract modules when the difference is contract behavior, not environment config.
- Generated deployment artifacts under `broadcast/`, `deployments/`, `out/`, or similar framework output directories.

Flag:

- `deploy-prod.ts`, `deploy-staging.ts`, `deploy-dev.ts` copies where only addresses, owner, RPC, salts, or constructor args differ.
- Mainnet-only branches that bypass validation.
- Environment selection inferred from RPC URL string matching.
- Deployment scripts that silently default to production values.

## Environment Configuration Standard

Use environment files and secrets, not checked-in production JSON, for deploy-time parameters.

Preferred files and variables:

```text
.env.development
.env.production
DEPLOY_ENV=development|production
CHAIN_ID=...
RPC_URL=...
DEPLOYER_PRIVATE_KEY=...
OWNER_ADDRESS=...
TREASURY_ADDRESS=...
CREATE2_SALT=...
```

Framework patterns:

- Foundry: use `vm.envString`, `vm.envAddress`, `vm.envUint`, or a typed config loader that wraps them.
- Hardhat/TypeScript: use `dotenv` or the repo's config loader, then validate with a schema before deploying.
- CI: source secrets from protected environment variables and keep `.env.production` uncommitted unless it contains placeholders only.

Flag:

- `prod.json`, `production.json`, `mainnet.json`, or similar files used as deploy-time authority for addresses, salts, owner/admin addresses, private keys, or RPC URLs.
- Checked-in private keys, mnemonics, API keys, RPC URLs with embedded credentials, multisig signer secrets, or admin credentials.
- Missing `.env.example` or missing validation for required production variables.
- Defaults that let production deploy without `DEPLOY_ENV=production` and explicit chain checks.

## Human Sign-Off Standard

Every live-network deployment or upgrade must have a human approval gate.

Acceptable controls:

- GitHub Actions protected environment with required reviewers for production/mainnet.
- Manual workflow dispatch plus required reviewer approval before deployment steps run.
- Multisig transaction proposal where execution requires human signer confirmations.
- Release checklist or PR approval that names the deployed contract, network, bytecode/artifact, constructor args, owner/admin, and expected address.

Flag:

- Deploy-on-merge to production/mainnet with no protected environment or reviewer gate.
- CI deploys that can run from unreviewed branches or forks.
- Direct deployer private key usage in CI without a manual approval boundary.
- Upgrade scripts that can execute implementation upgrades without explicit reviewer approval.
- Missing evidence that a human reviewed deployment params.

## Hard-Coded Address Detection

Run targeted searches before deciding a hard-coded value is safe:

```bash
rg -n "0x[a-fA-F0-9]{40}" .
rg -n "0x[a-fA-F0-9]{64}" .
rg -n "(PRIVATE_KEY|MNEMONIC|SEED|RPC_URL|ALCHEMY|INFURA|ETHERSCAN|OWNER|ADMIN|TREASURY|MULTISIG)" .
rg -n "(chainId|chain_id|CHAIN_ID|mainnet|sepolia|polygon|arbitrum|optimism|base)" .
rg -n "(prod\\.json|production\\.json|mainnet\\.json|deploy-prod|deployProd|DeployProd)" .
```

Classify each hard-coded address:

- `Allowed protocol constant`: canonical token, factory, router, precompile, system contract, or library address. Require chain scoping and a comment/name that explains it.
- `Generated artifact`: framework output from prior deployment. Do not edit by hand; verify it is ignored or intentionally committed.
- `Test fixture`: acceptable inside tests, mocks, fixtures, local chain setup, or deterministic fork tests.
- `Registry entry`: acceptable if chain-scoped, typed, validated, and reviewed.
- `Unsafe deploy parameter`: owner/admin/treasury/deployer/implementation/proxy/salt/fee recipient or any contract address used by live deployment logic. Move to env/config.

Address checks:

- Confirm the chain ID and network match the address.
- Verify EIP-55 checksum where the tooling supports it.
- Check whether the address is an EOA, multisig, proxy, implementation, token, or protocol contract as expected.
- For upgradeable contracts, verify proxy admin/owner and implementation addresses are not confused.
- Require tests or preflight validation to fail when production addresses are missing or malformed.

## "Other Silly Things" To Flag

Flag these during contract deployment reviews:

- `tx.origin` authorization.
- Public upgrade, pause, mint, sweep, or owner-only methods missing access control.
- `onlyOwner` owner initialized from an unchecked env var.
- Production deploys using test accounts, Anvil defaults, mnemonic defaults, or `PRIVATE_KEY || default`.
- Scripts that broadcast before validating chain ID and environment.
- Missing `--verify`/verification step where source verification is required by the release process.
- CREATE2 salts reused accidentally across environments without an intentional address plan.
- Proxy deployment where implementation initialization can be front-run or called twice.
- Constructor or initializer args not recorded in the deployment artifact/release notes.
- Silent fallback to zero address, deployer address, or `msg.sender` for owner/admin/treasury.
- Committed deployment artifacts that conflict with current source or ABI.

## Greptile Custom Review Instructions

Add concise repo-level custom instructions like this:

```markdown
When reviewing smart contract deployment changes:

- Flag one-script-per-environment deployment patterns. Prefer a single canonical deployment script that reads explicit environment variables such as DEPLOY_ENV, CHAIN_ID, RPC_URL, OWNER_ADDRESS, TREASURY_ADDRESS, and CREATE2_SALT.
- Flag deploy-time config loaded from prod.json, production.json, mainnet.json, or copied environment JSON files. Prefer .env.production, .env.development, CI secrets, and typed validation.
- Require every production/mainnet deployment or upgrade path to have a human approval gate, such as protected GitHub environments with required reviewers, manual workflow approval, multisig confirmation, or documented reviewer sign-off.
- Flag hard-coded Ethereum addresses and long hex constants in live deployment logic unless they are clearly named, chain-scoped protocol constants, generated artifacts, or test fixtures.
- For each hard-coded address, ask whether the chain ID, checksum, role, and source of truth are clear. Owner/admin/treasury/deployer/proxy/implementation addresses should come from validated env/config.
- Flag direct private key or mnemonic defaults, deploy-on-merge production workflows, broadcasts before chain/environment validation, zero-address owner/admin fallbacks, and upgrade scripts without explicit approval.
```

Optional stricter Greptile wording:

```markdown
Treat smart contract deployment diffs as release-critical. If a deployment can reach a live network without human review, or if production parameters are hard-coded or loaded from checked-in prod JSON, mark it as blocking.
```
