# Router Template

Use to generate `packages/api/src/router/<entity-kebab>.ts`.

Substitute all placeholders:
- `<Entity>` → PascalCase (e.g. `ProductVariant`)
- `<entity>` → camelCase (e.g. `productVariant`)

After substitution, remove any imports that are unused given the selected `--procedures` and `--auth` options (e.g. omit `protectedProcedure` if `--auth public`; omit `Create<Entity>Schema` if `create` is excluded).

```ts
import type { TRPCRouterRecord } from "@trpc/server";
import { z } from "zod/v4";

import { desc, eq } from "@acme/db"; // desc: only if 'all' is included; eq: always
import { Create<Entity>Schema, Update<Entity>Schema, <Entity> } from "@acme/db/schema"; // Omit Create<Entity>Schema if 'create' excluded; omit Update<Entity>Schema if 'update' excluded

import { protectedProcedure, publicProcedure } from "../trpc";

export const <entity>Router = {
  all: publicProcedure.query(({ ctx }) => {
    return ctx.db.query.<Entity>.findMany({
      orderBy: desc(<Entity>.id),
      limit: 10,
    });
  }),

  byId: publicProcedure
    .input(z.object({ id: z.string() }))
    .query(({ ctx, input }) => {
      return ctx.db.query.<Entity>.findFirst({
        where: eq(<Entity>.id, input.id),
      });
    }),

  create: protectedProcedure
    .input(Create<Entity>Schema)
    .mutation(({ ctx, input }) => {
      return ctx.db.insert(<Entity>).values(input);
    }),

  update: protectedProcedure
    .input(Update<Entity>Schema)
    .mutation(({ ctx, input }) => {
      const { id, ...data } = input;
      return ctx.db.update(<Entity>).set(data).where(eq(<Entity>.id, id)).returning();
    }),

  delete: protectedProcedure.input(z.string()).mutation(({ ctx, input }) => {
    return ctx.db.delete(<Entity>).where(eq(<Entity>.id, input));
  }),
} satisfies TRPCRouterRecord;
```
