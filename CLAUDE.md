# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

This is Labrys' global Claude Code configuration. It is cloned to `~/.claude` and provides coding standards, slash commands, and specialized agents that apply across all projects. It is not a software project — there is nothing to build, lint, or test.

## Structure

- `rules/` — Coding standards auto-loaded by Claude Code into every session
- `skills/` — Slash commands (`/pr`, `/check`, `/testing-plan`) in the skills format
- `agents/` — Specialized subagents for domain-specific tasks
- `docs/` — Internal guides ([Subagents Guide](docs/subagents-guide.md), [Monorepos Guide](docs/monorepos-guide.md))

## Rules (auto-loaded)

These define the coding conventions enforced across all projects:

- **react.md** — Controller-view-hook pattern: every feature splits into `feature.tsx` (thin controller), `feature.hook.ts` (all business logic), `feature.view.tsx` (pure presentation). Views may only hold ephemeral UI state (modals, dropdowns). Rule of thumb: "what" data → hook, "how" displayed → view.
- **typescript.md** — `any` is forbidden; use `unknown`, generics, union types, or type guards instead. Document the rare exception.
- **style.md** — PascalCase components/classes, camelCase functions, SCREAMING_SNAKE_CASE constants, hyphenated-lowercase file names (except component files which match their export). Simplicity over cleverness, explicit over implicit.
- **documentation.md** — README must stay in sync with code; update immediately on architectural changes.

## Skills (Slash Commands)

Skills live in `skills/<name>/SKILL.md` and are invoked as `/name`. All three are user-only (`disable-model-invocation: true`) — Claude will not auto-trigger them.

- `/pr [base-branch]` — Analyzes diff, generates PR description, creates or updates PR via `gh`. Never pushes without permission.
- `/check` — Discovers CI workflows in `.github/workflows/`, runs locally-runnable checks in parallel, auto-fixes format/lint issues, reports remaining problems.
- `/testing-plan <file-path>[#L<line>]` — Generates an execution-ordered unit testing plan as a markdown file co-located with the source.

## Agents

Two tiers of agents exist:

**Generic agents** (`agents/`): Reusable across any project — frontend-developer, backend-developer, fullstack-developer, nextjs-expert, typescript-pro, code-reviewer, debugger, documentation-engineer, qa-expert. Each encodes domain expertise and references the rules above.

**Specialist agents** (`agents/example-specialists/`): Project-specific examples showing how to create deep-domain agents. The included examples (lagoon-vault-analyst, lido-staking-vault-analyst) demonstrate the pattern — they encode protocol-specific contract architectures, flows, and analysis checklists. The generic `defi-protocol-analyst` agent routes to these specialists when appropriate.

### Agent anatomy

Each agent is a markdown file with YAML frontmatter (`name`, `description`, `model`, `color`) followed by the agent's system prompt including expertise, patterns, tool usage guidance, and cross-references to related agents.

## Editing Guidelines

When modifying this repo:
- Rules should be concise and opinionated — state what to do, not every possible alternative
- Skills should be self-contained instructions that work without user interaction where possible
- Agents should encode domain knowledge that would take multiple files to discover, not generic advice
- Keep specialist agents in `agents/example-specialists/` to separate them from the generic set

<!-- CODEGRAPH_START -->
## CodeGraph

CodeGraph builds a semantic knowledge graph of codebases for faster, smarter code exploration.

### If `.codegraph/` exists in the project

**Use codegraph tools for faster exploration.** These tools provide instant lookups via the code graph instead of scanning files:

| Tool | Use For |
|------|---------|
| `codegraph_search` | Find symbols by name (functions, classes, types) |
| `codegraph_context` | Get relevant code context for a task |
| `codegraph_callers` | Find what calls a function |
| `codegraph_callees` | Find what a function calls |
| `codegraph_impact` | See what's affected by changing a symbol |
| `codegraph_node` | Get details + source code for a symbol |

**When spawning Explore agents in a codegraph-enabled project:**

Tell the Explore agent to use codegraph tools for faster exploration.

**For quick lookups in the main session:**
- Use `codegraph_search` instead of grep for finding symbols
- Use `codegraph_callers`/`codegraph_callees` to trace code flow
- Use `codegraph_impact` before making changes to see what's affected

### If `.codegraph/` does NOT exist

At the start of a session, ask the user if they'd like to initialize CodeGraph:

"I notice this project doesn't have CodeGraph initialized. Would you like me to run `codegraph init -i` to build a code knowledge graph?"
<!-- CODEGRAPH_END -->
# graphify
- **graphify** (`~/.claude/skills/graphify/SKILL.md`) - any input to knowledge graph. Trigger: `/graphify`
When the user types `/graphify`, invoke the Skill tool with `skill: "graphify"` before doing anything else.
