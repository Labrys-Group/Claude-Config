# Code Style

## Naming Conventions

- **Components/Classes**: PascalCase (`UserProfile`, `DataManager`)
- **Component/Class Files**: Match export (`UserProfile.tsx`, `DataManager.ts`)
- **Other Files**: Hyphenated lowercase (`api-client.ts`, `use-auth.hook.ts`, `home-page.md`)
- **Functions/Methods**: camelCase, verb-first (`getUserData`, `handleClick`)
- **Hooks**: camelCase with `use` prefix (`useAuth`, `useFeature`)
- **Private fields**: camelCase with `private` keyword
- **Constants**: SCREAMING_SNAKE_CASE (`API_URL`, `MAX_ITEMS`)
- **Types/Interfaces**: PascalCase (`Props`, `Config`, `UserData`)

## Simplicity First

- No premature abstraction
- Three similar lines beats wrong abstraction
- Clear code over clever code

## Explicit Over Implicit

- Pass dependencies explicitly
- Named constants over magic numbers
- Clear names over comments

## Consistency

- Match existing patterns
- TypeScript strict mode
- Use formatter (Prettier)
- Meaningful variable names
