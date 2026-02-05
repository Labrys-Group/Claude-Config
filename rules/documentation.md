# Documentation

## README Maintenance

**Rule**: README must stay in sync with code. Update immediately when making architectural changes.

**Update README when:**
- Adding/removing major systems or features
- Changing architectural patterns
- Modifying setup/installation steps
- Updating dependencies with breaking changes
- Changing API contracts or data flow

**README Structure:**
```markdown
# Project Name
Brief description

## Architecture
High-level system design (link to ARCHITECTURE.md for details)

## Setup
Installation and running instructions

## Key Concepts
Core patterns and conventions used

## Project Structure
Directory layout and purpose
```

**Anti-patterns:**
- "Will update README later" - Update it now
- Out-of-sync diagrams/examples - Verify before committing
- Documenting implementation details - Focus on concepts
