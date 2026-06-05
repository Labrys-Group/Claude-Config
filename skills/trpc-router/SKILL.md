---
name: trpc-router
description: "Scaffold a complete tRPC v11 router for a named entity in this monorepo. Use when creating a new tRPC router, adding API endpoint procedures, or scaffolding CRUD routes. Trigger on phrases like 'create a router for X', 'scaffold X routes', 'I need CRUD for X', 'add X endpoints', or any mention of 'router', 'route', 'CRUD', 'procedure', or 'endpoint' alongside an entity name. Do NOT trigger on edits to existing routers, general API design discussions, config changes, or architecture debates."
---

# tRPC Router Scaffolder

Scaffold a complete new tRPC router for a named entity. Invoked via `/trpc-router <entity> [options]` or automatically when the user wants a new API route or endpoint for a named entity.

## Step -1 — Help flag and interactive mode

**Handle these before anything else.**

### --help

If `--help` or `-h` appears in the arguments, output the following and **stop** — do not scaffold:

```
Usage: /trpc-router <entity> [options]

Scaffolds a complete tRPC v11 router for a named entity.

Arguments:
  entity                 The entity name (any casing — e.g. post, ProductVariant, order-item)

Options:
  --procedures <value>   Which procedures to generate (default: all)
                           all       → all, byId, create, update, delete
                           query     → all, byId
                           mutation  → create, update, delete
                           Or pick individually: byId, create, update, delete

  --auth <mode>          Auth mode for procedures (default: mixed)
                           mixed     → queries use publicProcedure, mutations use protectedProcedure
                           public    → all procedures use publicProcedure
                           protected → all procedures use protectedProcedure

  --with-schema          Also append a Drizzle table + Zod schemas to packages/db/src/schema.ts
                         (You'll be asked for column definitions before anything is written)

  --no-tests             Skip generating the test file

  --guided               Step through all options interactively before scaffolding

  --help, -h             Show this help text

Examples:
  /trpc-router post
  /trpc-router comment --procedures query --auth public
  /trpc-router orderItem --procedures mutation --with-schema --no-tests
  /trpc-router user --auth protected --guided
```

### Interactive guided mode

Trigger guided mode when **either** of these is true:
- No entity name was provided (e.g. bare `/trpc-router` or "scaffold a router")
- `--guided` is in the arguments

**Guided flow:**

1. **Entity name** *(skip if already provided)*
   Ask: `What entity would you like to scaffold a router for?`
   Wait for a response.

2. **Procedures**
   Ask:
   ```
   Which procedures should I generate? (default: all)
     all       → all, byId, create, update, delete
     query     → all, byId only
     mutation  → create, update, delete only
     Or list specific ones: byId, create, delete
   ```
   Accept a blank reply or "default" to mean `all`.

3. **Auth mode**
   Ask:
   ```
   Auth mode? (default: mixed)
     mixed     → queries public, mutations protected
     public    → all procedures public
     protected → all procedures protected
   ```
   Accept a blank reply or "default" to mean `mixed`.

4. **Schema**
   Ask: `Generate a Drizzle table + Zod schema in schema.ts? (y/n, default: n)`
   Accept y/yes to set `--with-schema`.

5. **Tests**
   Ask: `Generate a test file? (y/n, default: y)`
   Accept n/no to set `--no-tests`.

6. **Confirm** — summarise the resolved options before scaffolding:
   ```
   Ready to scaffold:
     Entity:     <entity>
     Procedures: <procedures>
     Auth:       <auth>
     Schema:     <yes/no>
     Tests:      <yes/no>

   Proceed? (y/n)
   ```
   If the user says no or asks to change something, go back to the relevant question.

Once confirmed, continue to Step 0 with the collected options.

---

## Step 0 — Pre-flight reads

Before generating any files, read these four files (always):

- `packages/api/src/root.ts` — router registration pattern, existing routers, collision check
- `packages/api/src/test-helpers.ts` — `makeTestCaller` signature and context shape
- `packages/api/src/trpc.ts` — available procedure types and context fields
- `packages/api/src/index.ts` — package exports (check if the new router needs type re-exports)

If `--with-schema` was requested, also read:

- `packages/db/src/schema.ts` — find the exact insertion point, verify imports are not duplicated

Do **not** read any files under `packages/api/src/router/` — they are template placeholders and may not exist.

## Step 1 — Parse and validate the entity name

Accept any casing and normalise:

| Input | Normalised |
|-------|-----------|
| `ProductVariant` | `productVariant` |
| `product-variant` | `productVariant` |
| `comment` | `comment` |

Derive these identifiers from the normalised name:

- **Router key** (in `root.ts`): camelCase — e.g. `productVariant`
- **Router file**: kebab-case — e.g. `product-variant.ts`
- **Router export**: camelCase + `Router` — e.g. `productVariantRouter`
- **Table/class name**: PascalCase — e.g. `ProductVariant`
- **Zod schema names**: `Create<Entity>Schema`, `Update<Entity>Schema` — e.g. `CreateProductVariantSchema`, `UpdateProductVariantSchema`

Check `root.ts` for a collision — if a router with the same key already exists, warn the user and abort. Confirm the result is a valid TypeScript identifier (alphanumeric, starts with a letter).

## Step 2 — Resolve options

| Option | Values | Default |
|--------|--------|---------|
| `--procedures` | `all`, `query`, `mutation`, or comma-separated list of: `all`, `byId`, `create`, `update`, `delete` | `all` (all five procedures) |
| `--auth` | `public`, `protected`, `mixed` | `mixed` |
| `--with-schema` | flag | off |
| `--no-tests` | flag | off |

**Procedure shortcuts:**
- `all` → `all, byId, create, update, delete`
- `query` → `all, byId`
- `mutation` → `create, update, delete`

When a subset is specified, omit unspecified procedures entirely — no stubs or comments for them.

**Auth modes:**
- `public` — all procedures use `publicProcedure`
- `protected` — all procedures use `protectedProcedure`
- `mixed` — queries (`all`, `byId`) use `publicProcedure`; mutations (`create`, `update`, `delete`) use `protectedProcedure`

## Step 3 — Execution order

This order is mandatory — the router imports from the schema, so the schema must exist first.

1. **If `--with-schema`**: append table + Zod schema to `packages/db/src/schema.ts`
2. Create `packages/api/src/router/<entity>.ts`
3. Register the router in `packages/api/src/root.ts`
4. **Unless `--no-tests`**: create `packages/api/src/router/<entity>.test.ts`

## Step 4 — Generate files using the templates

Read the relevant template files from the `templates/` folder adjacent to this skill. Each template is a markdown file — extract only the code block (the content inside the ` ```ts ``` ` fence), then substitute the entity placeholders before writing:

- **Router** → read `templates/router.md`, write to `packages/api/src/router/<entity>.ts`
- **Tests** → read `templates/test.md`, write to `packages/api/src/router/<entity>.test.ts`
- **Schema** (only with `--with-schema`) → read `templates/schema.md`, append to `packages/db/src/schema.ts`

Placeholder substitution:

| Placeholder | Replace with |
|-------------|-------------|
| `<Entity>` | PascalCase entity name |
| `<entity>` | camelCase entity name |
| `<entity-kebab>` | kebab-case entity name (file names only) |

**After substitution, apply these adjustments:**

- Remove any imports that aren't used (e.g. omit `protectedProcedure` if `--auth public`; omit `Create<Entity>Schema` if `create` is not included)
- Remove procedures not in the requested set — omit entirely, no stubs
- Adjust `publicProcedure` / `protectedProcedure` per `--auth` mode
- For `--with-schema`: ask the user for entity-specific column definitions before writing — do not generate columns without input

**Confirm `root.ts` was updated** before presenting the checklist.

---

## Post-scaffolding checklist

Present this after all files are generated:

- [ ] Schema appended to `packages/db/src/schema.ts` *(only if `--with-schema`)*
- [ ] Router created at `packages/api/src/router/<entity>.ts`
- [ ] Router registered in `packages/api/src/root.ts`
- [ ] Test file created at `packages/api/src/router/<entity>.test.ts` *(unless `--no-tests`)*
- [ ] Run `pnpm typecheck`
- [ ] Run `pnpm db:push` *(only if `--with-schema`)*
- [ ] Run `pnpm test`

---

## Constraints

- Use Australian/British English in all user-facing text
- Zod imports are **always** from `"zod/v4"` — never from `"zod"`
- TypeScript strict mode — no `any` and no `eslint-disable` comments in generated files; use `makeAuthenticatedCaller()` from `test-helpers` for mutation tests (protected procedures), use `makeTestCaller()` for query tests (public procedures)
- Import ordering: `import type` first, then value imports — match the router template exactly
- When a procedure subset is given, omit the rest entirely — no stubs or placeholder comments
