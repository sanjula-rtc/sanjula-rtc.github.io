---
layout: post
title: "MySQL SKIP LOCKED: A Job Queue Without the Waiting"
date: 2026-10-08
categories: mysql
tags: [mysql, innodb, locking, queue, concurrency]
---

You have a table of jobs and several workers. Each worker should take **one job nobody else has**. Sounds easy, until two workers grab the same row, or all of them stand in line behind one locked row. `SKIP LOCKED` solves both.

## The problem

The naive approach:

```sql
SELECT id FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1;
-- then, in app code:
UPDATE jobs SET status = 'running' WHERE id = ?;
```

Two workers run the SELECT at the same moment and **both get job #1**. The job runs twice.

The usual fix is `FOR UPDATE`, which locks the row:

```sql
START TRANSACTION;
SELECT id FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1 FOR UPDATE;
```

Now worker B can't take job #1, but it also **waits** for worker A to finish, then often finds nothing left. Your workers end up taking turns instead of working in parallel.

## Picture: waiting vs skipping

```
 Plain FOR UPDATE                       FOR UPDATE SKIP LOCKED

 jobs:  [1] [2] [3] [4]                 jobs:  [1] [2] [3] [4]

 Worker A  locks [1]  ✔                 Worker A  locks [1]  ✔
 Worker B  wants [1]  ⏳ waits...        Worker B  [1] is locked → skips → locks [2] ✔
 Worker C  wants [1]  ⏳ waits...        Worker C  [1],[2] locked → skips → locks [3] ✔

 Everyone queues behind one row.        Everyone works on a different row.
```

## The solution

Available in **MySQL 8.0 and later**, with the InnoDB engine:

```sql
START TRANSACTION;

SELECT id, payload
FROM jobs
WHERE status = 'pending'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- ...do the work, or mark the job as claimed...

UPDATE jobs SET status = 'done' WHERE id = ?;
COMMIT;
```

`SKIP LOCKED` means: *"if a row is locked by someone else, pretend it isn't there and move on."* Each worker gets a different row, and nobody waits.

Its sibling is `NOWAIT`, which doesn't skip: it returns an error immediately if a row is locked.

```
 FOR UPDATE              → wait for the lock (default, up to innodb_lock_wait_timeout)
 FOR UPDATE NOWAIT       → fail right away if locked
 FOR UPDATE SKIP LOCKED  → ignore locked rows and carry on
```

## Claim, then work: two patterns

**Pattern 1: hold the transaction while working.** Simple, and crash-safe. If a worker dies, its connection closes, MySQL rolls back, the lock is released, and another worker picks the job up again. The downside is a long-lived open transaction, which is bad for long jobs.

**Pattern 2: claim quickly, work outside.** Mark the row as taken in a short transaction, commit, then do the slow work:

```sql
START TRANSACTION;

SELECT id FROM jobs
WHERE status = 'pending'
ORDER BY id LIMIT 1
FOR UPDATE SKIP LOCKED;

UPDATE jobs SET status = 'running', locked_at = NOW() WHERE id = ?;
COMMIT;
-- now run the job with no lock held
```

The cost: if a worker crashes after committing, the job is stuck at `running`. You need a cleanup rule, such as resetting rows with an old `locked_at` back to `pending`.

## Gotchas

1. **Add an index** that matches your `WHERE` and `ORDER BY`, for example on `(status, id)`. Without one, MySQL scans and locks far more rows than you expect, and workers collide again.
2. **Keep transactions short.** Locks last until `COMMIT` or `ROLLBACK`.
3. **It's not a consistent view.** Skipped rows are invisible to that query, so the results are **incomplete by design**. Use it for "give me any available work", never for reports or totals. The MySQL manual warns about this too.
4. **InnoDB only.** Other engines don't have row locks.
5. **Don't rely on strict ordering.** If job #1 is locked, a worker takes #2 first. Jobs can finish out of order.
6. **Isolation level matters.** Under the default `REPEATABLE READ`, gap locks can block other inserts and cause deadlocks. If you see unexpected waiting, check your isolation level (`READ COMMITTED` avoids gap locks) and your query plan with `EXPLAIN`.

## A real story: Shopify and the last item in stock

This is how I first learned about `SKIP LOCKED`. In May 2026, Shopify's engineering team wrote about [replacing Redis with MySQL for inventory reservations](https://shopify.engineering/scaling-inventory-reservations).

**The problem.** When you start paying, Shopify must *hold* the item so two buyers can't both get the last one (overselling), without wrongly showing it as sold out (underselling). During Black Friday 2025 the platform peaked at about $5.1 million in sales per minute.

**The old way: Redis.** Each item had a counter. Reserve meant `DECR`, release meant `INCR`. It was fast, but the reservation lived in Redis while the inventory ledger lived in MySQL. Finishing a purchase meant updating two systems that couldn't share one atomic transaction.

**The new way: MySQL.** Putting both in MySQL allowed one real transaction. But a single row with a `quantity` column is a hotspot, because every buyer of a popular item fights for the same row. So they changed the shape of the data:

```
 Before: one row per item              After: one row per sellable unit

 ┌──────────┬──────────┐               ┌─────────┐ ┌─────────┐ ┌─────────┐
 │ item 42  │ qty = 10 │               │ unit 1  │ │ unit 2  │ │ unit 3  │ ...
 └──────────┴──────────┘               └─────────┘ └─────────┘ └─────────┘
 everyone locks this one row           each buyer takes different unit rows
```

Reserving 3 units means moving 3 rows, and `SKIP LOCKED` lets each buyer grab whichever rows nobody else is holding. It's the same job-queue idea as above, with "units" instead of "jobs".

**The details worth stealing:**

- **A bounded pool.** Each item and location keeps at most **1,000** available rows, and a refill step tops it up from the ledger. The number came from observed peak rates, balancing burst capacity against slow `SKIP LOCKED` scans.
- **`READ COMMITTED` instead of the default `REPEATABLE READ`.** The default's gap locks blocked the refill inserts and caused deadlocks. This is the isolation-level warning from the gotchas list in real life.
- **A consistent lock order** (always touch tables in the same order) removed circular-wait deadlocks.
- **A composite primary key** that includes the filter columns cut row locks per reservation from two to one. This is the "index your filter" gotcha again.

**The surprise.** The real bottleneck wasn't the reservation queries at all. It was how long *other* checkout code held database connections. Fixing that mattered more than tuning the locking. Writer CPU stayed under 50% during big flash sales.

The lesson I took from it: `SKIP LOCKED` isn't only for job queues. It works whenever many people compete for "any one of these interchangeable things", including units of stock, tickets, and seats. This summary comes from Shopify's post, so read the original for the full design.

## Trying it yourself

Open two MySQL sessions side by side:

```sql
-- Session 1
START TRANSACTION;
SELECT id FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED;
-- returns id 1, and leave the transaction OPEN

-- Session 2
START TRANSACTION;
SELECT id FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED;
-- returns id 2 immediately. Remove SKIP LOCKED and it hangs.
```

## The rule of thumb

> **Many workers, one table of jobs? `FOR UPDATE SKIP LOCKED`.**
> Each worker takes the next free row instead of waiting for the locked one. Index the filter, keep transactions short, and never use it for anything that must see *all* the rows.
