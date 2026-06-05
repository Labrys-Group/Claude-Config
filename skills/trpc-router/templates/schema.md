# Schema Template

Append to `packages/db/src/schema.ts` — do **not** create a new file.

The following imports are already present in `schema.ts` — do **not** duplicate them:
- `import { sql } from "drizzle-orm"`
- `import { pgTable } from "drizzle-orm/pg-core"`
- `import { createInsertSchema } from "drizzle-zod"`
- `import { z } from "zod/v4"`

Ask the user for entity-specific column definitions (name, type, nullable, validation rules) before writing — do not generate columns without input.

Substitute all placeholders:
- `<Entity>` → PascalCase (e.g. `ProductVariant`)
- `<entity>` → camelCase (e.g. `productVariant`)

```ts
export const <Entity> = pgTable("<entity>", (t) => ({
  id: t.uuid().notNull().primaryKey().defaultRandom(),
  // entity-specific columns here
  createdAt: t.timestamp().defaultNow().notNull(),
  updatedAt: t
    .timestamp({ mode: "date", withTimezone: true })
    .$onUpdateFn(() => new Date()),
}));

export const Create<Entity>Schema = createInsertSchema(<Entity>, {
  // add per-field Zod overrides here if needed
}).omit({
  id: true,
  createdAt: true,
  updatedAt: true,
});

export const Update<Entity>Schema = createInsertSchema(<Entity>)
  .omit({ createdAt: true, updatedAt: true })
  .partial()
  .required({ id: true });
```
