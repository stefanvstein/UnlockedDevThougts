---
title: "Isolation Levels and Write Skew"
date: 2016-10-08
slug: "isolation-and-skew"
tags: ["Database"]
---

A few notes on isolation levels in relational databases—and a tricky little issue called *write skew*.

---

## Isolation Levels in Brief

Most relational databases default to the **Read Committed** isolation level. This prevents dirty reads, but does *not* protect you from more subtle anomalies such as **read skew** or **write skew**.

### Read Skew Example

Imagine one user moves data from location X to location Y, ensuring a value (say, 100 units) is transferred. If another user reads both locations during this operation, they might observe the same value in *both* places temporarily. That's **read skew**: the illusion of duplicated data due to timing.

![Figure 1: Read skew scenario](/images/read-skew.png)

To prevent this, you can increase the isolation level.

---

## Serializable vs Snapshot Isolation

- **Serializable isolation** means operations behave *as if* executed sequentially. This is the strongest level and guarantees full consistency—but it's expensive.

- **Snapshot isolation** is cheaper. Each transaction gets a consistent view of the data, and conflicting writes cause failure.

However, snapshot isolation doesn't protect against **write skew**.

### Write Skew Example

Imagine two users reading shared data with a rule that at least one of two fields must remain true. Each transaction reads, validates that the rule holds, and writes a change. But if both users update the same fields simultaneously, the invariant might break—because each acted on an outdated view.

![Figure 2: Write skew anomaly](/images/write-skew.png)

---

## Reality in Practice

Some databases, such as **Oracle**, don't support true serializable isolation. Their "serializable" mode is really snapshot isolation under the hood.

### Workarounds

- **Single Writer Principle:** Achieve serializability by allowing only one writer process.
- **Explicit Locks:** Lock all rows or objects you read during the transaction to avoid skew.

These methods ensure correctness—though at the cost of concurrency.

---

Isolation levels aren't just theoretical—they shape the behavior of concurrent applications. Choosing the right level (and knowing its limits) is crucial for correctness.

