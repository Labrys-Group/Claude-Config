# Test Template

Use to generate `packages/api/src/router/<entity-kebab>.test.ts`.

Substitute all placeholders:
- `<Entity>` → PascalCase (e.g. `ProductVariant`)
- `<entity>` → camelCase (e.g. `productVariant`)

Fill `/* required fields */` with the actual column values needed to satisfy the schema's non-nullable constraints. Use valid RFC 4122 UUIDs for all UUID fields — never bare strings like `"test-uuid"` or zero-padded strings like `"00000000-0000-4000-8000-000000000001"` (the 3rd group must start with `[1-8]` for version, 4th group with `[89ab]` for variant). Use the pattern `"00000000-0000-4000-8000-00000000000X"` for readable test fixtures.

Omit tests for procedures that were not included in `--procedures`.

```ts
import { beforeEach, describe, expect, it, vi } from "vitest";

import { eq } from "@acme/db";
import { db } from "@acme/db/client";
import { <Entity> } from "@acme/db/schema";

import { makeAuthenticatedCaller, makeTestCaller } from "../test-helpers";

vi.mock("@acme/db/client", async () => {
  const { createMockDb } = await import("@acme/db/mocks");
  return { db: await createMockDb() };
});

describe("<entity>Router", () => {
  beforeEach(async () => {
    // If <Entity> has FK dependencies, delete child tables first, then parents.
    // See like.test.ts for an example (deletes Like then Post).
    await db.delete(<Entity>);
  });

  it("should return all <entity>s", async () => {
    const caller = makeTestCaller();
    const result = await caller.<entity>.all();
    expect(result).toEqual([]);
  });

  it("should fetch <entity> by id — found", async () => {
    const record = { id: "00000000-0000-4000-8000-000000000001", /* required fields */ };
    await db.insert(<Entity>).values(record);

    const caller = makeTestCaller();
    const result = await caller.<entity>.byId({ id: record.id });
    expect(result?.id).toBe(record.id);
  });

  it("should fetch <entity> by id — not found", async () => {
    const caller = makeTestCaller();
    const result = await caller.<entity>.byId({ id: "00000000-0000-4000-8000-000000000099" });
    expect(result).toBeUndefined();
  });

  it("should create a <entity>", async () => {
    const caller = makeAuthenticatedCaller();

    await caller.<entity>.create({ /* required fields */ });

    const records = await db.select().from(<Entity>);
    expect(records).toHaveLength(1);
    expect(records[0]?./* field */).toBe(/* expected value */);
  });

  it("should update a <entity>", async () => {
    const record = { id: "00000000-0000-4000-8000-000000000001", /* required fields */ };
    await db.insert(<Entity>).values(record);

    const caller = makeAuthenticatedCaller();
    await caller.<entity>.update({ id: record.id, /* fields to update */ });

    const [updated] = await db.select().from(<Entity>).where(eq(<Entity>.id, record.id));
    expect(updated?./* updated field */).toBe(/* expected value */);
  });

  it("should delete a <entity>", async () => {
    const record = { id: "00000000-0000-4000-8000-000000000001", /* required fields */ };
    await db.insert(<Entity>).values(record);

    const caller = makeAuthenticatedCaller();
    await caller.<entity>.delete(record.id);

    const records = await db.select().from(<Entity>);
    expect(records).toHaveLength(0);
  });
});
```
