---
title: The locks that taught me about deadlocks and isolation levels.
description: A follow-up on SELECT FOR UPDATE — what happens once locking works, and the new problems it quietly introduces.
date: 2026-07-22 18:00:00 +0300
categories: [Backend, Databases]
tags: [postgresql, concurrency, transactions, deadlocks, isolation-levels]
image:
  path: /assets/img/posts/blog2.jpg
  alt: Deadlocks, lock contention, and transaction isolation
---

## Where the last article left me

In [the last post](https://njange.github.io/posts/The-Bug-That-Taught-Me-Database-Concurrency/) I walked through a booking bug: two patients could confirm the same slot because my code read the slot, saw it was free, and wrote with a gap in between where a second request could slip through. `SELECT FOR UPDATE` fixed it by locking the row, so the second transaction had to wait its turn.

I was pleased with myself for about a day. Then an engineer, Mr Bala, asked a quation that i couldn't stop thinking about:

> `select_for_update` will lock the tables a query is touching. How do you ensure deadlocks don't become an issue and hurt application performance? You could also look into how isolation levels address concurrency problems like the dirty and repeatable reads you mentioned.

That is the whole second half of the story. A lock solves a race, but a lock is also a shared resource that other transactions now have to queue behind and queues have costs. This post is my answer, in two parts: deadlocks and performance, then isolation levels.

## First, a small but important correction

It's worth being precise about what `SELECT FOR UPDATE` locks, because the rest of the article depends on it. On the databases most of us reach for PostgreSQL, and MySQL's InnoDB engine, it takes **row-level** locks, not table locks. It locks the specific rows the query returns, not the whole table.

That's mostly good news: two people booking two *different* slots never block each other. But there's a catch that matters later the database can only lock rows efficiently if it can *find* them efficiently. If you run `SELECT ... FOR UPDATE` on a column with no index, the engine may scan and lock far more than you intended (in InnoDB's default mode it can even take gap locks on ranges it walked through). So the first rule of keeping locks cheap is boring: **index the columns you filter and lock on.**

With that settled, on to question.

## What a deadlock actually is

A deadlock isn't a lock that got stuck. It's two transactions politely waiting for each other, forever. Each holds something the other needs.

Picture two requests, each updating a booking *and* a per-doctor counter, but touching them in opposite orders:

```sql
-- Transaction A
BEGIN;
SELECT * FROM slots WHERE id = 1 FOR UPDATE;      -- A locks slot 1
-- ... some work ...
SELECT * FROM doctors WHERE id = 7 FOR UPDATE;     -- A now wants doctor 7

-- Transaction B (running at the same time)
BEGIN;
SELECT * FROM doctors WHERE id = 7 FOR UPDATE;     -- B locks doctor 7
-- ... some work ...
SELECT * FROM slots WHERE id = 1 FOR UPDATE;       -- B now wants slot 1
```

A holds slot 1 and wants doctor 7. B holds doctor 7 and wants slot 1. Neither can proceed. This is a **circular wait**, and it's the textbook shape of every deadlock.

The good news is you rarely have to detect this yourself. Postgres and InnoDB both run a deadlock detector, notice the cycle, and kill one of the transactions the "victim" rolling it back so the other can continue. You'll see it surface as an error:

- PostgreSQL: `deadlock detected` (SQLSTATE `40P01`)
- MySQL/InnoDB: `Deadlock found when trying to get lock` (error `1213`)

So the question isn't really "how do I stop the database from deadlocking" it's "how do I make deadlocks rare, and handle them gracefully when they still happen."

## Why my booking code probably wasn't deadlocking (yet)

Here's the honest nuance a lot of articles skip. My original fix locked *one* row a single slot. A transaction that only ever grabs one lock cannot form a circular wait with itself. Deadlocks need **at least two locks held in an inconsistent order** across concurrent transactions.

That's reassuring, but it's also a warning: the risk appears the moment my "book a slot" logic grows. The day I add "and decrement the doctor's remaining-appointments counter" and "and insert an audit row," I've created three locks per request and if two code paths acquire them in different orders, I've built a deadlock generator without noticing. So prevention is mostly about discipline before the problem exists.

## How I keep deadlocks rare

**1. Lock in a consistent order.** This is the single most effective fix. If every transaction acquires locks in the same order, a cycle is impossible. The cheap trick is to sort:

```sql
SELECT * FROM slots
WHERE id IN (1, 2, 3)
ORDER BY id
FOR UPDATE;
```

Locking rows in `id` order everywhere means two transactions can never hold each other's next lock.

**2. Keep transactions short.** A lock is held until you `COMMIT`. So the critical section the time between acquiring the lock and committing should be as small as possible. In particular, **never do slow or external work while holding a lock**: no HTTP calls to a payment provider, no sending the confirmation email, no `sleep`. Do the read-lock-write against the database, commit, *then* fire the side effects.

**3. Lock the smallest set of rows you can.** Narrow your `WHERE` clause, and (as above) index it so the engine locks exactly the rows you meant and no more.

**4. Fail fast or step aside when you can.** Two clauses are worth knowing:

- `FOR UPDATE NOWAIT` — if the row is already locked, don't queue; raise an error immediately. Good when waiting is pointless and you'd rather tell the user "try again."
- `FOR UPDATE SKIP LOCKED` — skip rows that are currently locked and move on. This is the classic pattern for **job queues** and "grab the next available item," where any free row will do. It's *not* what you want for "book *this specific* slot" there you actually need that one row — but it's a superpower for worker pools pulling from a shared table.

**5. Expect deadlocks and retry.** Under real concurrency you can't eliminate them entirely, so treat them as a normal, recoverable condition. Because the victim transaction was rolled back completely, it's safe to run the *whole* transaction again:

```python
from django.db import transaction, OperationalError

for attempt in range(3):
    try:
        with transaction.atomic():
            slot = Slot.objects.select_for_update().get(id=slot_id)
            # ... booking logic ...
        break  # success
    except OperationalError:
        if attempt == 2:
            raise
        continue  # deadlock victim — retry the whole block
```

Retry the entire block, not just the last statement, add a small backoff, and cap the attempts. This is also *why* rule 2 matters: if you'd sent an email inside that block, a retry would send it twice.

## The cost that isn't a deadlock

Deadlocks are the dramatic failure. The quieter one is **contention** and it hurts performance even when nothing ever deadlocks.

Locks serialize access to a row. If a hundred people try to book against the same hot row at once, `SELECT FOR UPDATE` turns that into a queue: one at a time, everyone else waiting. Correctness is preserved, but throughput on that row collapses. The mitigation is the same instinct as before — keep the critical section tiny so each holder releases the lock quickly but it's also worth knowing you don't always have to reach for a pessimistic lock at all:

- **Optimistic locking.** Add a `version` column. Read the row (no lock), do your work, then `UPDATE ... WHERE id = ? AND version = ?`. If zero rows were affected, someone changed it first you retry. No lock is held between read and write, so there's nothing to contend on. This shines when conflicts are *rare*; if they're common, all the retries make it slower than just locking.

- **Let a constraint do the work.** For my actual requirement "don't double-book" the cleanest fix might not be a lock at all. A unique constraint (say on `(doctor_id, start_time)` in the appointments table) means the *second* concurrent insert simply fails with a unique-violation error, which I catch and turn into "sorry, that slot was just taken." The database enforces the invariant for me, with no explicit locking and no deadlock surface.

The theme across all of this: locking is a scalpel, not a hammer. The goal is the *smallest correct amount* of it.

## The other lens: isolation levels

Bala's second suggestion was to look at isolation levels, and it turns out my original bug is best understood through them. Every transaction runs at an isolation level, and that level defines which concurrency anomalies are allowed to happen. There are three classic anomalies:

- **Dirty read** — you read a row another transaction has written but not yet committed. If it rolls back, you acted on data that never existed.
- **Non-repeatable read** — you read a row twice in one transaction and get different values, because another transaction committed a change in between.
- **Phantom read** — you run the same query twice and get a different *set* of rows, because another transaction inserted or deleted rows matching your condition.

The four standard levels are defined by which of these they forbid:

| Isolation level    | Dirty read | Non-repeatable read | Phantom read |
| ------------------ | ---------- | ------------------- | ------------ |
| Read Uncommitted   | Possible   | Possible            | Possible     |
| Read Committed     | Prevented  | Possible            | Possible     |
| Repeatable Read    | Prevented  | Prevented           | Possible*    |
| Serializable       | Prevented  | Prevented           | Prevented    |

A couple of real-world footnotes, because the defaults surprise people:

- **PostgreSQL defaults to Read Committed.** It also never allows dirty reads at all asking for Read Uncommitted just gives you Read Committed. And its Repeatable Read uses snapshot isolation that, in practice, *also* prevents phantoms which is why I starred that cell.
- **MySQL/InnoDB defaults to Repeatable Read.**

So my booking bug was, precisely, a **Read Committed** anomaly. Between my "is this slot free?" read and my "book it" write, another committed transaction changed the answer under me. That reframes the fix in a useful way — there are actually two ways to solve it:

- **Pessimistically**, with `SELECT FOR UPDATE`: lock the row so no one else can touch it until I commit. That's what I did.
- **Optimistically**, by running the transaction at **Serializable** isolation: the database watches for exactly this kind of interference and, if it detects it, aborts one transaction with a serialization failure which I catch and retry (the same retry pattern as deadlocks).

Same bug with two philosophies. Lock and make others wait, or let everyone run and abort-and-retry the ones that conflicted. Which is better depends on how often your transactions actually collide.

## So, the direct answer 

How do I keep deadlocks from hurting the app? In one breath: acquire locks in a consistent order, hold them for as short a time as possible, lock only well-indexed rows, use `NOWAIT`/`SKIP LOCKED` where the semantics fit, and treat the occasional deadlock as a normal retryable event rather than a crash. And before reaching for a lock at all, ask whether a unique constraint, an optimistic version check, or a stricter isolation level expresses what I actually mean more cleanly.

The first article taught me *why* `SELECT FOR UPDATE` exists. This one taught me that turning it on is where the interesting engineering starts, not where it ends.

Thanks to Mr. Bala for the question that became this post.
