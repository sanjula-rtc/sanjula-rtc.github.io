---
layout: post
title: "Mongoose ObjectId Gotcha: Why aggregate() Matched Nothing"
date: 2026-10-08
categories: mongodb
tags: [mongoose, mongodb, nestjs, objectid, aggregate]
---

A NestJS query worked fine with `find()`, but the same filter inside `aggregate()` returned **an empty array**. No error, no warning, just nothing. The cause was one line in the schema.

## The rule

> **Never use `Types.ObjectId` as the `type` in a Mongoose `@Prop`.**
> Use `mongoose.Schema.Types.ObjectId`.

```ts
import mongoose, { Types } from 'mongoose';

// ❌ Wrong: in our case the field was stored as a plain string
@Prop({ type: Types.ObjectId, ref: 'User' })
userId: Types.ObjectId;

// ✅ Right
@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'User' })
userId: Types.ObjectId;
```

Notice that the **TypeScript type** (`userId: Types.ObjectId`) is fine and should stay. The problem is only the **runtime `type` option** inside `@Prop`.

## Two things with the same name

Mongoose has two similar-looking `ObjectId`s, and they do different jobs:

```
 mongoose.Types.ObjectId          →  the VALUE
                                     a real id you create:  new Types.ObjectId()

 mongoose.Schema.Types.ObjectId   →  the SCHEMA TYPE
                                     a description that says "this field holds an ObjectId"
```

`@Prop({ type })` expects a **schema type**, a description of the field. Passing the value class `Types.ObjectId` doesn't give Mongoose the description it needs. In our case it didn't error out. The field ended up stored as a **string**.

## Why nothing complained: `find()` hides the bug

Here's the sneaky part. The two query methods treat your data differently:

```
 find({ userId: "507f1f77..." })          aggregate([{ $match: { userId: ... } }])
          │                                         │
          ▼                                         ▼
 Mongoose CASTS the value                 Mongoose does NOT cast
 using the schema                         pipeline stages at all
          │                                         │
          ▼                                         ▼
 sends it as the schema says              sends exactly what you wrote
```

MongoDB compares **type and value**. A string `"507f…"` and an `ObjectId("507f…")` look the same when printed but are **different values**, so they never match.

```
 Stored in DB (the bug):    userId: "507f1f77bcf86cd799439011"        ← string

 Your aggregate match:      { $match: { userId: new Types.ObjectId(id) } }
                                                   ↓
                            ObjectId("507f1f77bcf86cd799439011")      ← ObjectId

                            string ≠ ObjectId   →   0 documents
```

With `find()`, Mongoose casts using the (wrong) schema, so both sides become strings and everything matches. That's why the bug survived until the first `aggregate()`.

## How to spot it

Don't trust what the app prints. Look at what's **really stored**, in `mongosh`:

```js
db.orders.findOne()
// { userId: '507f1f77bcf86cd799439011' }          ← quotes = string  ❌
// { userId: ObjectId('507f1f77bcf86cd799439011') } ← ObjectId        ✅

// Or count by BSON type:
db.orders.countDocuments({ userId: { $type: 'string' } })
db.orders.countDocuments({ userId: { $type: 'objectId' } })
```

A quick test for an empty aggregate: remove stages one by one, starting from the `$match`, until documents appear.

## How to fix it

**1. Fix the schema** (as shown above) so new documents store a real ObjectId.

**2. Fix existing data.** Documents already saved as strings stay strings. Convert them:

```js
db.orders.updateMany(
  { userId: { $type: 'string' } },
  [{ $set: { userId: { $toObjectId: '$userId' } } }]
);
```

Back up first, and try it on a small collection before running it on production data.

## Rules for aggregate() in general

Even with a correct schema, `aggregate()` is raw. You must do the casting yourself:

```ts
// ❌ id is a string from the request, and the DB holds an ObjectId
this.model.aggregate([{ $match: { userId: id } }]);

// ✅ convert it first
this.model.aggregate([{ $match: { userId: new Types.ObjectId(id) } }]);
```

The same applies to `$lookup`: both sides must be the **same type**, or you get empty joins.

## The rule of thumb

> `Types.ObjectId` is for **creating values**. `Schema.Types.ObjectId` is for **declaring fields**.
> If `find()` works but `aggregate()` returns nothing, check whether the id is stored as a string.
