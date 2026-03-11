# Mongoose Best Practices

## Method Selection Guide

| Need                          | Method              |
|-------------------------------|---------------------|
| Delete, don't need the doc    | `deleteOne` / `deleteMany` |
| Delete, need the doc back     | `findOneAndDelete`  |
| Update, don't need the doc    | `updateOne` / `updateMany` |
| Update, need the updated doc  | `findOneAndUpdate`  |
| Read, must exist              | `findById().orFail()` / `findOne().orFail()` |
| Read, might not exist         | `findById()` / `findOne()` + null check |

## Avoid `findOneAnd*` for Destructive Operations

**Rule**: Use `deleteOne`/`updateOne` instead of `findOneAndDelete`/`findOneAndUpdate` unless you need the returned document. Validate success by checking the returned operation result (e.g., `deletedCount === 1` or `matchedCount === 1`).

**Why**: `findOneAndDelete` throws internally when no document exists.

```typescript
// BAD - Throws if document doesn't exist, kills transaction
await UserOnboardingModel.findOneAndDelete({ user: profile }, { session }).orFail();

// GOOD - Succeeds regardless of whether document exists
await UserOnboardingModel.deleteOne({ user: profile }, { session });

// ACCEPTABLE - When you need the doc back, wrap in try/catch
try {
  const doc = await UserOnboardingModel.findOneAndDelete({ user: profile }, { session }).orFail(new Error("Failed to Delete Onboarding Document"));
} catch (error) {
  console.log("Error", error);
}
```

## Use `.orFail()` Only When Absence Is an Error

**Rule**: Only chain `.orFail()` when a missing document means the operation cannot continue.

**Why**: Using `.orFail()` on optional documents aborts transactions and throws where "not found" is a valid state.

```typescript
// BAD - Rewards may not exist yet, kills the whole transaction
await UserRewardsModel.findOneAndDelete({ userId: profile }, { session }).orFail();

// GOOD - User must exist to proceed, .orFail() is appropriate
const user = await UserProfileModel.findById(profileId).orFail();

// GOOD - Silently skips if no rewards document exists
await UserRewardsModel.deleteOne({ userId: profile }, { session });
```

## Always Pass Session to `.save()` and `.create()` in Transactions

**Rule**: Every `.save()` and `.create()` call inside a transaction must receive the session option.

**Why**: Without the session, the write executes outside the transaction and won't roll back on abort.

```typescript
// BAD - save() runs outside the transaction
const doc = new UserProfileModel({ name: "Ben" });
await doc.save();

// GOOD - save() is part of the transaction
const doc = new UserProfileModel({ name: "Ben" });
await doc.save({ session });
```

## Use Array Syntax for `Model.create()` with Sessions

**Rule**: Pass an array of documents as the first argument when calling `Model.create()` with a session.

**Why**: `Model.create(doc, { session })` treats the options object as a second document; the session is ignored.

```typescript
// BAD - Session is ignored, second arg is treated as another document
await UserProfileModel.create({ name: "Ben" }, { session });

// GOOD - Array syntax allows the second arg to be options
await UserProfileModel.create([{ name: "Ben" }], { session });

// ALSO GOOD - insertMany is often better for bulk inserts (single round-trip)
await UserProfileModel.insertMany([{ name: "Ben" }], { session });
```

## Use Dot Notation for Nested `$set` Updates

**Rule**: Use dot notation (`"address.city"`) instead of object notation (`{ address: { city } }`) when updating nested fields with `$set`.

**Why**: Object notation replaces the entire nested object, wiping out sibling fields you didn't include.

```typescript
// BAD - Overwrites the entire address object, deletes address.street
await UserProfileModel.updateOne(
  { _id: userId },
  { $set: { address: { city: "Toronto" } } }
);

// GOOD - Only updates address.city, leaves address.street intact
await UserProfileModel.updateOne(
  { _id: userId },
  { $set: { "address.city": "Toronto" } }
);
```

## Null-Check Populated References

**Rule**: Always null-check populated fields before accessing properties on them.

**Why**: Populated references return `null` if the referenced document was deleted; accessing properties on `null` throws a runtime error.

```typescript
// BAD - Crashes if the referenced user was deleted
const order = await OrderModel.findById(orderId).populate("user").orFail();
console.log(order.user.name);

// GOOD - Handles deleted references gracefully
const order = await OrderModel.findById(orderId).populate("user").orFail();
if (!order.user) {
  throw new Error("Referenced user no longer exists");
}
console.log(order.user.name);
```

## `.lean()` Returns Plain Objects — No Methods or Virtuals

**Rule**: Do not call instance methods, virtuals, or `.save()` on documents returned by `.lean()`.

**Why**: `.lean()` returns raw JS objects for performance; they have no mongoose document prototype, so method calls throw.

```typescript
// BAD - .lean() strips all methods, .save() is undefined
const user = await UserProfileModel.findById(userId).lean();
user.name = "Updated";
await user.save();

// GOOD - Omit .lean() when you need document methods
const user = await UserProfileModel.findById(userId).orFail();
user.name = "Updated";
await user.save();

// GOOD - Use .lean() for read-only queries where you only need data
const user = await UserProfileModel.findById(userId).lean();
return res.json(user);
```

## `pre('save')` Does Not Fire on `updateOne`/`findOneAndUpdate`

**Rule**: Do not rely on `pre('save')` middleware to run during update operations.

**Why**: `updateOne`, `updateMany`, and `findOneAndUpdate` bypass `save` middleware entirely; use `pre('findOneAndUpdate')` or move logic to the application layer.

```typescript
// BAD - This middleware never fires on updateOne
schema.pre("save", function () {
  this.updatedAt = new Date();
});
await UserProfileModel.updateOne({ _id: userId }, { $set: { name: "Ben" } });

// GOOD - Use the correct middleware hook for updates
schema.pre("findOneAndUpdate", function () {
  this.set({ updatedAt: new Date() }); // `this` is the Query, not the doc
});

// GOOD - Or use timestamps: true in the schema for automatic updatedAt
const schema = new Schema({ name: String }, { timestamps: true });
```

## Schema Defaults Don't Apply on `updateOne`/Upsert

**Rule**: Do not expect schema `default` values to populate during `updateOne` or `upsert` operations.

**Why**: Schema defaults only apply when constructing new mongoose documents via `new Model()` or `Model.create()`; raw update operations bypass the schema layer.

```typescript
// BAD - Default role:"user" won't be set on upsert
const schema = new Schema({ name: String, role: { type: String, default: "user" } });
await UserModel.updateOne({ email }, { $set: { name: "Ben" } }, { upsert: true });

// GOOD - Explicitly set defaults in $setOnInsert for upserts
await UserModel.updateOne(
  { email },
  { $set: { name: "Ben" }, $setOnInsert: { role: "user" } },
  { upsert: true }
);

// GOOD - timestamps: true also handles createdAt/updatedAt on upserts in recent Mongoose versions
const schema = new Schema({ name: String, role: { type: String, default: "user" } }, { timestamps: true });
```

## Mixed Type Requires `markModified()` Before `save()`

**Rule**: Call `doc.markModified(path)` before `save()` when mutating a `Schema.Types.Mixed` or untyped object field.

**Why**: Mongoose cannot detect changes inside Mixed fields; without `markModified()`, `save()` silently skips the update.

```typescript
// BAD - Change is not detected, save() does nothing
const user = await UserProfileModel.findById(userId).orFail();
user.metadata.preferences.theme = "dark";
await user.save();

// GOOD - Explicitly mark the path as modified
const user = await UserProfileModel.findById(userId).orFail();
user.metadata.preferences.theme = "dark";
user.markModified("metadata");
await user.save();
```

## Empty `$in` Array Matches Nothing

**Rule**: Guard against passing an empty array to `$in` — it returns zero results, not all results.

**Why**: `{ field: { $in: [] } }` is a valid query that matches no documents, which silently returns empty results when you may expect all documents.

```typescript
// BAD - Returns nothing when userIds is empty, looks like a bug
const users = await UserProfileModel.find({ _id: { $in: userIds } });

// GOOD - Short-circuit when the array is empty
if (userIds.length === 0) {
  return [];
}
const users = await UserProfileModel.find({ _id: { $in: userIds } });
```

## Disable `bufferCommands` in Serverless Environments

**Rule**: Set `bufferCommands: false` globally or per-schema when running in serverless (Lambda, Vercel, etc.).

**Why**: Mongoose buffers operations until a connection is established; in serverless cold starts, this can cause requests to hang until timeout instead of failing fast.

```typescript
// BAD - Operations hang if connection isn't ready during cold start
mongoose.connect(uri);

// GOOD - Disable globally for fast failure
mongoose.set("bufferCommands", false);
mongoose.connect(uri);

// GOOD - Or disable per-schema
const schema = new Schema({ name: String }, { bufferCommands: false });

// ALSO GOOD - Lower serverSelectionTimeoutMS on the connection for faster failure
mongoose.connect(uri, { serverSelectionTimeoutMS: 5000 });
```

## Check `deletedCount`/`modifiedCount` in Bulk Operations

**Rule**: Always verify `deletedCount` or `modifiedCount` after `deleteMany`/`updateMany`, especially inside transactions.

**Why**: These operations succeed with zero matches without throwing. Inside transactions, this means your "cleanup" step may silently do nothing while the rest of the transaction proceeds, leaving stale data behind.

```typescript
// BAD - Assumes all related docs were cleaned up
await session.withTransaction(async () => {
  await UserProfileModel.deleteOne({ _id: userId }, { session });
  await UserSessionModel.deleteMany({ userId }, { session }); // might delete 0 docs silently
});

// GOOD - Verify the operation did what you expected
await session.withTransaction(async () => {
  await UserProfileModel.deleteOne({ _id: userId }, { session });
  const { deletedCount } = await UserSessionModel.deleteMany({ userId }, { session });
  console.log(`Cleaned up ${deletedCount} sessions for user ${userId}`);
});

// GOOD - Throw if a specific count was expected
const { modifiedCount } = await OrderModel.updateMany(
  { status: "pending", createdAt: { $lt: cutoffDate } },
  { $set: { status: "expired" } },
  { session }
);
if (modifiedCount === 0) {
  console.warn("No pending orders found to expire");
}
```