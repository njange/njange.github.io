---
layout: post
title: "The bug that taught me about database concurrency."
date: 2026-07-15 18:00:00 +0300
description: >
  A lesson in database transactions, race conditions, isolation levels,
  and why SELECT FOR UPDATE exists.
image:
  path: /assets/img/posts/blog1.jpg
  alt: Database transactions and row locking
categories:
  - Databases
  - Backend
tags:
  - databases
  - concurrency
  - transactions
  - ddia
  - django
  - postgresql
  - sql
toc: true
comments: true
math: false
mermaid: true
---

One of the projects I recently worked on was a fairly ordinary clinic booking system. Patients could browse a doctor's available time slots, choose one, and confirm an appointment.

The business requirement sounded almost trivial.

> Once a slot has been booked, nobody else should be able to book it.

Like most backend engineers, I immediately translated that into code.

```python
existing = Appointment.objects.filter(
    doctor_id=doctor_id,
    slot_time=slot_time,
    cancelled=False,
).exists()

if existing:
    raise SlotAlreadyBooked()

Appointment.objects.create(...)
```

It looked perfectly reasonable.
Read the database.
If the slot doesn't exist, create it.

Done.

<!--more-->

## Everything Worked...

At least during testing.
One user.
One request.
One database transaction.
Everything behaved exactly as expected.

Then I started thinking about what would happen if two patients clicked **Book Appointment** at exactly the same time.

```
Patient A                     Patient B

Check slot
Not booked

                             Check slot
                             Not booked

Create appointment

                             Create appointment
```

Both requests succeed.
Both patients receive confirmation.

The same doctor now has two appointments for the same slot. Nothing crashed. No exceptions were thrown. Every individual line of code behaved correctly. The system still produced the wrong result. That surprised me.

## The Bug Wasn't in the Code

Around the same time I was reading Chapter 7 of *Designing Data-Intensive Applications* by Martin Kleppmann.

One idea completely changed how I think about databases. Transactions aren't only about recovering from failures. They're also about protecting your data when multiple transactions execute concurrently The booking problem wasn't really a booking problem.

It was a concurrency problem.

My application had silently assumed that once it read the database, nothing could change until it finished writing. Databases don't make that promise.

## A Small Window With Big Consequences

The problem lives in the tiny gap between these two operations.

```text
Read database

↓

Make decision

↓

Write database
```

That gap may only last a few milliseconds. That's enough. Another transaction can sneak in during that window and change the world underneath your feet. Once I started looking for this pattern, I noticed it everywhere. In

- Inventory systems
- Airline reservations
- Banking transfers
- Ticket booking
- Hotel reservations

They're all variations of the same concurrency problem.

## Isolation Suddenly Made Sense

Before this project, terms like:

- Dirty Reads
- Snapshot Isolation
- Repeatable Read
- Write Skew

felt like database trivia.

Now they had context. Isolation isn't about making SQL harder. It's about deciding what different transactions are allowed to observe while they execute. Once you understand that, the terminology becomes much less intimidating.

## Enter `SELECT FOR UPDATE`

Eventually I came across `SELECT FOR UPDATE`.

At first it looked like an ordinary query. The important difference is that it tells the database:

> "I'm about to modify this row. Don't let anyone else modify it until I'm finished."

In Django, that becomes:

```python
from django.db import transaction

with transaction.atomic():
    doctor = (
        Doctor.objects
        .select_for_update()
        .get(id=doctor_id)
    )

    # Validate availability

    Appointment.objects.create(...)
```

Now the sequence changes.

```
Transaction A
    │
    ├── acquires lock
    ├── validates
    ├── creates appointment
    └── commits

Transaction B
    │
    ├── waits
    ├── reads updated state
    └── returns "Slot already booked"
```

Instead of racing each other, the transactions become coordinated.

## The Database Should Defend Your Invariants

Another lesson I learned was that application code shouldn't be the only place where business rules live.
If your rule is:

> A doctor cannot have two appointments at the same time.

Then the database should enforce that as well.
A unique constraint on:

```text
(doctor_id, slot_time)
```

means that even if application code fails, the database refuses to create an impossible state.
Application code expresses intent. Database constraints protect truth. The two complement each other.

## Final Thoughts

Before reading DDIA, I thought transactions existed mainly so that changes could be rolled back after failures. Now I think of them differently. Transactions are agreements about how concurrent work should behave.

The biggest lesson wasn't learning `SELECT FOR UPDATE`.

It was learning to ask a different question whenever I write backend code. Instead of asking:

> Does this code work?

I now ask:

> Does this code still work when two people execute it at exactly the same time?

I've found that the second question is usually the more interesting one.

---

## Further Reading

- *Designing Data-Intensive Applications* — Martin Kleppmann (Chapter 7)
- PostgreSQL Documentation — Row-Level Locks
- Django Documentation — `select_for_update()`
