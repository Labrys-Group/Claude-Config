# Claude Code in Monorepos

## How Memory Loading Works

Claude Code loads instructions from CLAUDE.md files using two mechanisms: **ancestor loading** walks upward from your working directory at startup and loads every CLAUDE.md it finds between you and the repo root, **descendant loading** walks downward and lazy-loads CLAUDE.md files only when you touch files in their directory.

Ancestors always load. Descendants lazy-load. Siblings never load. This is the fundamental rule. If you're in `packages/web/` and edit a file in `packages/api/`, the API package's CLAUDE.md lazy-loads at that point — but `packages/shared/CLAUDE.md` never loads unless you touch files there too.

Five memory types exist, listed by precedence (highest first):

1. **Managed policy** — enterprise-managed settings (admin-controlled)
2. **Project memory** — `./CLAUDE.md` or `./.claude/CLAUDE.md`
3. **Project rules** — `.claude/rules/*.md` files
4. **User memory** — `~/.claude/CLAUDE.md` (applies to all your projects)
5. **Project local** — `./CLAUDE.local.md` (gitignored automatically)

Higher precedence types load first and take priority when instructions conflict. Note that `.claude/CLAUDE.md` is an alternative location to root `CLAUDE.md` — useful for keeping monorepo roots clean when they already have many config files.

## Where You Launch Matters

| Launch from | Loads at startup | Loads lazily |
|---|---|---|
| Repo root | Root CLAUDE.md, `.claude/rules/` | Package CLAUDE.md files (when accessed) |
| `packages/frontend/` | Root + `packages/frontend/CLAUDE.md`, `.claude/rules/` | Other package CLAUDE.md files |

**Launch from root** for cross-cutting work that spans multiple packages — refactoring shared types, updating CI, coordinating changes across boundaries. **Launch from a package directory** for focused work — you get both root and package instructions loaded at startup with no lazy-load delay, and Claude's context is primed for that package's conventions from the first message.

The `--add-dir` flag (or the `additionalDirectories` permission setting) gives a session access to directories outside your working directory without changing where you launched from. This is useful when you need to reference a sibling package's code while staying focused in one package.

## Monorepo Structure Example

```
monorepo/
├── CLAUDE.md                        # Repo-wide: build system, PR conventions
├── CLAUDE.local.md                  # Personal preferences (gitignored)
├── .claude/
│   ├── rules/
│   │   ├── testing.md               # No paths: applies everywhere
│   │   ├── react-patterns.md        # paths: ["packages/web/**"]
│   │   └── api-conventions.md       # paths: ["packages/api/**"]
│   ├── settings.json                # Team permissions, env vars
│   └── settings.local.json          # Personal overrides (gitignored)
├── packages/
│   ├── web/
│   │   └── CLAUDE.md                # Next.js patterns, component rules
│   ├── api/
│   │   └── CLAUDE.md                # Express conventions, DB patterns
│   └── shared-lib/
│       └── CLAUDE.md                # Export conventions, package boundaries
```

The layering principle: root CLAUDE.md holds universal standards (build commands, PR process, language conventions). Package CLAUDE.md files hold context that only matters when working in that package — framework choices, local test commands, architecture decisions specific to that service. `.claude/rules/` holds standards that can be scoped to specific file paths via glob patterns, which is particularly powerful in monorepos where different packages use different frameworks or patterns.

## What Goes Where

| Content | Location | Why |
|---|---|---|
| Build system, PR process | Root CLAUDE.md | Always relevant |
| React component patterns | `.claude/rules/react.md` with `paths: ["packages/web/**"]` | Only loads for frontend files |
| API endpoint conventions | `packages/api/CLAUDE.md` | Only loads in API context |
| TypeScript strictness | `.claude/rules/typescript.md` (no paths) | Applies everywhere |
| Personal preferences | `CLAUDE.local.md` or `~/.claude/CLAUDE.md` | Not committed to version control |

**Rule of thumb:** if a standard applies to specific file paths across the repo, use a path-conditional rule in `.claude/rules/`. If it applies to everything inside a single package, use that package's CLAUDE.md. If it applies everywhere regardless of context, put it in the root. Personal conventions that shouldn't be committed go in `CLAUDE.local.md` or your global `~/.claude/CLAUDE.md`.

## Imports for Shared Instructions

Any CLAUDE.md file can import another file with `@path/to/file` syntax:

```markdown
# packages/web/CLAUDE.md

@../../shared-standards.md
@../../.claude/rules/component-patterns.md

## Web-Specific Patterns
- Use Next.js App Router for all new pages
```

Paths resolve relative to the importing file's location, not the working directory. Imports chain up to 5 hops deep. Imports inside markdown code spans and code blocks are not evaluated — so references to npm packages like `@anthropic-ai/sdk` in your instructions are safe and won't trigger file resolution.

The first time Claude encounters an import in a project, it shows an approval dialog. After approval, imports load silently on subsequent sessions.

This is the primary tool for avoiding duplication in monorepos. Instead of copying your TypeScript conventions into every package's CLAUDE.md, write them once in a shared file and import from each package that needs them.

**For git worktrees:** use `@~/.claude/my-project-instructions.md` so personal preferences follow across worktrees without duplication. Home-directory imports (`@~/...`) resolve the same way regardless of which worktree you're in.

## Path-Conditional Rules

Files in `.claude/rules/` auto-load as project rules at the same priority as `.claude/CLAUDE.md`. This is the most powerful monorepo feature — it lets you write one rule file that only activates when Claude is working on matching files. Add `paths:` YAML frontmatter to make a rule conditional:

```markdown
---
paths:
  - "packages/web/**"
  - "packages/mobile/**"
---

# React Patterns
- Use the controller-view-hook pattern for all features
- No business logic in view components
```

Without `paths:` frontmatter, a rule is unconditional — it applies to every file in the project.

Glob patterns follow standard syntax: `**/*.ts`, `src/**/*`, `*.md`, and brace expansion like `src/**/*.{ts,tsx}`. Subdirectories are supported — `.claude/rules/frontend/react.md` works. Symlinks are also supported: `ln -s ~/shared-claude-rules .claude/rules/shared` lets you share rules across repos.

User-level rules in `~/.claude/rules/` load before project rules, but project rules take priority when they conflict. This means a team can enforce conventions via committed rules that override any individual developer's personal preferences.

## Settings Hierarchy

Five levels of settings, highest precedence first:

1. **Enterprise managed** — `managed-settings.json`
2. **Command line args** — flags passed at launch
3. **Local project** — `.claude/settings.local.json` (gitignored)
4. **Shared project** — `.claude/settings.json` (committed)
5. **User** — `~/.claude/settings.json` (personal, all projects)

Key monorepo settings:

```json
{
  "permissions": {
    "deny": ["Bash(rm -rf *)", "Bash(cat .env*)"],
    "allow": ["Bash(npm run test)", "Bash(npm run build)"]
  }
}
```

Use shared project settings (`.claude/settings.json`) for team-wide permissions — deny destructive commands, allow standard build/test commands. Use local settings for personal overrides like editor preferences or additional allowed commands. The `additionalDirectories` permission controls whether Claude can access directories added via `--add-dir`, enabling cross-package file access when needed.

## Keeping Context Lean

Every CLAUDE.md file contributes to baseline context that persists across the entire session. In a monorepo with many packages, unconditional instructions from every package can add up quickly. Bloated instructions waste tokens on every message and reduce the space available for actual code context.

Strategies for keeping context lean:

- **Use path-conditional rules** so frontend instructions don't load during backend work and vice versa
- **Use imports** instead of duplicating instructions across packages — one source of truth, many consumers
- **Keep each CLAUDE.md focused:** build/test commands, architecture decisions, conventions that Claude needs to follow. Don't include reference documentation, tutorials, or explanations of why a convention exists — just state the convention
- **Use `/init`** to bootstrap a CLAUDE.md for a new package; **use `/memory`** to edit instructions mid-session without leaving Claude
- **Prefer `.claude/rules/` with path conditions** over package-level CLAUDE.md files when the instructions are about file patterns rather than package identity

**Rule of thumb:** if you're copy-pasting instructions into multiple CLAUDE.md files, extract to a shared file and import it.

## Quick Reference

| Feature | Location / Syntax | Notes |
|---|---|---|
| Root instructions | `CLAUDE.md` or `.claude/CLAUDE.md` | Loads for every session |
| Package instructions | `packages/x/CLAUDE.md` | Lazy-loads when files accessed |
| Conditional rules | `.claude/rules/x.md` with `paths:` | Glob patterns, only for matching files |
| Unconditional rules | `.claude/rules/x.md` (no frontmatter) | Always active |
| Shared imports | `@path/to/file.md` in any CLAUDE.md | Max 5 hops, relative to importing file |
| Personal preferences | `CLAUDE.local.md` | Gitignored automatically |
| User rules | `~/.claude/rules/` | Personal, all projects, lower priority |
| Project settings | `.claude/settings.json` | Committed, team-wide |
| Local settings | `.claude/settings.local.json` | Gitignored, personal |
| Bootstrap | `/init` | Generates starter CLAUDE.md |
| Edit memory | `/memory` | Opens in system editor |

---

**Sources:**
- [Claude Code — Manage Claude's Memory](https://code.claude.com/docs/en/memory)
- [Claude Code — Settings](https://code.claude.com/docs/en/settings)
- [shanraisshan/claude-code-best-practice — Monorepo Guide](https://github.com/shanraisshan/claude-code-best-practice/blob/main/reports/claude-md-for-larger-mono-repos.md)
