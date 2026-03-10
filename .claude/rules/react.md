# React Standards

## Controller-View Pattern

Every feature follows a controller-view-hook pattern:

```
src/feature/
  feature.tsx            # Orchestration
  feature.hook.ts        # Business logic & state
  feature.hook.spec.ts   # Hook tests
  feature.view.tsx       # Pure presentation
  feature.view.spec.tsx  # View tests
  feature.md             # Documentation (optional)
```

### Controller (feature.tsx)

Thin orchestration layer - compose hook + view only:

```tsx
export const HomeController = () => {
  const props = useHome();
  return <HomeView {...props} />;
};
```

### Hook (feature.hook.ts)

All business logic, API calls, external state, side effects:

```tsx
export const useHome = () => {
  const { data } = api.home.get.useQuery();
  const [state, setState] = useState();

  const handleAction = () => {
    // business logic
  };

  return { data, state, handleAction };
};

// Export inferred type for View
export type UseHomeProps = ReturnType<typeof useHome>;
```

### View (feature.view.tsx)

**Pure components only** - no state, no context, same props, same output.

**Why?** Testing (mock props → assert output), debuggability (no hidden state), and refactorability (swap hooks without breaking UI) are simple when view and business logic are separated.

### Valid Local State

Local state is allowed for **ephemeral UI concerns only** - things that control how content displays, not what displays.

**Valid:** Modal/dropdown open state, accordion expanded, form inputs (pre-submit), tab index, animations
**Invalid:** Data fetching (`useQuery`), filtering/sorting, business calculations, auth state

### Examples

```tsx
// BAD - Business logic in View
export const ProductListView = ({ products }: UseProductListProps) => {
  const [search, setSearch] = useState("");
  const filtered = products.filter(p => p.name.includes(search)); // BAD
  const { data } = api.categories.list.useQuery(); // BAD
  return <div>{filtered.map(...)}</div>;
};

// GOOD - Business logic in Hook
export const useProductList = () => {
  const [search, setSearch] = useState("");
  const products = api.products.list.useQuery();
  const filtered = products.filter(p => p.name.includes(search));
  return { products: filtered, search, setSearch };
};

export const ProductListView = ({ products, search, setSearch }: UseProductListProps) => {
  const [isFilterOpen, setIsFilterOpen] = useState(false); // OK - UI state only
  return <div><input value={search} onChange={e => setSearch(e.target.value)} /></div>;
};
```

**Rule of thumb:** If it affects **what** data is shown → Hook. If it affects **how** it's shown → View.

If your view contains a lot of valid local state, consider moving it to a separate hook for readability.

## Folder Structure (Next.js)

Co-locate page components with routes:

```
app/
  home/
    page.tsx                    # Route entry point
    _components/
      home.controller.tsx       # Orchestration
      home.hook.ts              # Single hook when page isn't complex
      home.hook.spec.ts         # Specs co-located
      home.view.tsx
      home-table/               # Complex component gets subdirectory
        home-table.tsx
        home-table.hook.ts
        home-table.view.tsx
    _hooks/                     # Multiple hooks for complex pages
    _lib/                       # Page-specific helpers
```

**Rules:**
- `_components/` for page-specific components (not reusable across pages)
- Shared/reusable components go in root `components/` directory
- `_` prefix keeps folders private to Next.js routing
- Spec files co-located next to implementation files

## Folder Structure (Vanilla React)

Same pattern under `features/`:

```
src/
  features/
    home/
      home.tsx                  # Controller
      home.hook.ts
      home.view.tsx
      components/               # Feature-specific components
      hooks/                    # Multiple hooks for complex features
      lib/                      # Feature-specific helpers
  components/                   # Shared/reusable components
  hooks/                        # Shared hooks
  lib/                          # Shared utilities
```

## Component Guidelines

- Functional components only
- Hooks for logic reuse
- Early returns for conditionals
- Destructure props in signature
- Use composition over prop threading
- Context for truly global state only (theme, auth)
