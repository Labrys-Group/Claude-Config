# Subagents Guide

## What Subagents Are

A subagent is a specialist with its own desk and context window. It's a markdown file that encodes domain knowledge — expertise, working patterns, tool preferences — into a reusable persona that Claude can delegate to automatically.

The key difference from ad-hoc prompting: subagents persist as files in `agents/`, get their own isolated context window when invoked, and Claude routes to them based on their `description` field. You write the expertise once; every project benefits.

## When to Use Subagents vs Other Tools

| Mechanism | Location | Triggered by | Use when... |
|-----------|----------|-------------|-------------|
| **Rule** | `rules/` | Auto-loaded every session | Enforcing an always-on coding standard (e.g., "no `any` types") |
| **Command** | `commands/` | User types `/command` | Running a user-triggered workflow (e.g., `/pr`) |
| **Skill** | Skills config | Model matches trigger | A capability needs supporting files or specific tool sequences |
| **Subagent** | `agents/` | Model delegates automatically | The task needs its own context window and domain expertise |

**Rule of thumb:** if the task requires isolated context and domain knowledge that wouldn't fit in a rule or command, make it a subagent.

## Generic vs Specialist Agents

**Generic agents** (`agents/`) are role-based and reusable across any project: `frontend-developer`, `backend-developer`, `code-reviewer`, `debugger`, `documentation-engineer`, etc. They encode expertise in a technology or discipline.

**Specialist agents** (`agents/example-specialists/`) encode deep domain knowledge for a specific project or protocol. The included `lagoon-vault-analyst` and `lido-staking-vault-analyst` demonstrate this — they know contract architectures, settlement flows, and analysis checklists that would take pages to rediscover.

Create a **generic** agent when the expertise is role-based (e.g., "senior React developer"). Create a **specialist** when the knowledge is product-specific (e.g., "Lagoon ERC-7540 vault settlement flows"). Specialists should reference their generic parent for routing — see the `defi-protocol-analyst` agent for this pattern.

## Bootstrapping with `/agents`

The built-in `/agents` command has a "Generate with Claude" option that produces complete, well-structured agents in one shot. The official docs recommend this as a best practice: generate first, then customize. Running `/agents` → "Create New Agent" → "Generate with Claude" produces a full agent file — frontmatter, system prompt, expertise sections, and integration references — from a single description. It's significantly faster than writing from scratch, and the output quality is high enough to use immediately for many cases.

The key is giving it a detailed prompt. A vague request like "make a code review agent" produces generic output. Instead, front-load the prompt with the specifics that matter. A strong generation prompt covers six areas:

1. **Role and domain** — what the agent is and what project/stack it targets
2. **Core specializations** — 2-3 specific areas of expertise
3. **Workflow** — what it checks first, how it structures its work, what it hands off
4. **Technical context** — frameworks, patterns, conventions, file structure
5. **Scope boundaries** — what the agent should NOT do (prevents drift into unrelated areas)
6. **Output format** — how it should structure responses (checklist, narrative, diff annotations)

**Template:**

> Create a [role] agent for [specific project/domain]. It should specialize in [2-3 core areas]. When invoked, it should [describe the workflow — what it checks first, how it structures output, what it hands off]. It needs to know about [key technical details: frameworks, patterns, conventions, file structure]. It should NOT [scope boundaries — what to avoid, what to delegate instead]. Output should be structured as [format: checklist, categorized findings, narrative summary, etc.]. Restrict tools to [Read/Grep/Glob/Bash or whatever fits]. Use [model] model. Reference these related agents for routing: [list].

Two prompt details that improve auto-delegation: include phrases like "Use PROACTIVELY" or "MUST BE USED when..." in your description of when the agent should activate — Claude weights these trigger phrases heavily when deciding whether to route a task. Also specify what the agent should *not* handle, so it delegates cleanly rather than drifting.

**Example prompt for a real agent:**

> Create a migration-reviewer agent for our PostgreSQL database. It should specialize in reviewing Drizzle ORM migration files for correctness, backwards compatibility, and data safety. When invoked, it should check the migration SQL for destructive operations (DROP, ALTER column type, NOT NULL without default), verify the schema diff matches intent, and flag any migration that would lock large tables. It should NOT implement fixes or write migration code — it should flag issues and route to backend-developer for implementation. Output should be a categorized checklist: safety issues, compatibility warnings, and style notes. It needs to know that we use Drizzle with camelCase columns mapped to snake_case, pgTable callback style, and drizzle-zod for validators. Restrict tools to Read, Grep, Glob, Bash. Use sonnet model. Route to backend-developer for implementation questions and code-reviewer for general code quality.

**Tip:** During generation, press `e` to open the generated prompt in your editor for immediate refinement before saving.

After generation, open the file and refine — tighten the description for better routing, add domain knowledge Claude wouldn't know, and make sure the "Integration with Other Agents" section is accurate. The generated agent is a strong starting point, not the final product.


## Anatomy of a Good Agent

Every agent is a markdown file with YAML frontmatter followed by a system prompt.

**Frontmatter:**

```yaml
---
name: my-agent
description: |-
  Use this agent when analyzing X, debugging Y, or building Z.
  Examples: specific task A, specific task B.
model: inherit
color: blue
---
```

**Body structure** — four sections that appear consistently across our agents:

1. **Role statement** — One sentence: who you are and what you specialize in
2. **Core expertise** — What you know, organized by domain area
3. **Working patterns** — How you approach tasks, quality standards, methodology
4. **Integration references** — Which agents to hand off to and when

**Minimal skeleton:**

```markdown
---
name: example-agent
description: |-
  Use this agent when [specific triggers].
  Examples: [concrete task A], [concrete task B].
model: inherit
color: green
---

You are a [role] specializing in [domain]. You [key capability].

## Core Expertise

- Area 1: detail
- Area 2: detail

## Working Patterns

[How you approach tasks, what you check, quality standards]

## Integration with Other Agents

- **agent-name** — when to route there
```

## Writing Effective Descriptions

The `description` field is the most important field — it drives automatic delegation. Claude reads it to decide whether to route a task to your agent.

Include: when to use the agent, what triggers it, and concrete examples. Use action phrases like "Use this agent when..." or "Use proactively after...".

**Weak:**
```yaml
description: Frontend development expert
```

**Strong:**
```yaml
description: |-
  Use this agent when building React 19 components, implementing responsive layouts,
  working with shadcn/ui and Tailwind v4, adding interactivity, or improving accessibility.
  Examples: creating UI components, designing with CVA variants, adding dark mode support,
  implementing the controller-view-hook pattern for a feature.
```

The strong version tells Claude exactly which tasks should route here.

## Tool Selection

By default, agents inherit all available tools. Restrict tools when:

- **Read-only analysis** — a code reviewer doesn't need Write/Edit
- **Security-sensitive tasks** — limit what an agent can modify
- **Focus** — fewer tools means less scope drift

Specify allowed tools in frontmatter with the `tools` field. Omit it entirely to inherit everything (the common case for most agents).

## Model Selection

Choose the model based on task complexity:

- **`inherit`** — matches the parent conversation's model. Good default for most agents.
- **`opus`** — deep analysis, complex multi-step reasoning. Used by `defi-protocol-analyst` and specialist agents where accuracy is critical.
- **`sonnet`** — balanced cost and capability. Good default for routine development tasks.
- **`haiku`** — fast and cheap. Best for simple, well-scoped tasks.

## Cross-Agent Routing

Agents reference each other via an "Integration with Other Agents" section at the bottom of their prompt. This creates a routing network — when one agent encounters a task outside its scope, it knows where to send it.

For specialist domains, use a routing table. From `defi-protocol-analyst`:

```markdown
| Vault Type | Specialized Agent |
|------------|-------------------|
| Lagoon ERC-7540 async vaults | lagoon-vault-analyst |
| Lido V3 staking vaults | lido-staking-vault-analyst |
| Generic DeFi protocols | This agent (defi-protocol-analyst) |
```

This pattern helps Claude pick the right specialist without the user needing to know the agent names.

## Testing Your Agent

1. **Invoke explicitly:** "Use the X agent to do Y" — verify it activates
2. **Check triggers:** Does a natural task description route to it automatically?
3. **Check scope:** Does it stay within its domain or drift into unrelated areas?
4. **Iterate:** Tune the `description` and prompt body based on real usage — the description controls routing, the body controls quality
5. **Inspect:** Use the `/agents` command to list and review configured agents
