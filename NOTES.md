# How Convex Works

Notes from [How Convex Works](https://stack.convex.dev/how-convex-works) by Sujay Jayakar, mapped to the codebase.

---

## Overview

A Convex deployment is a database that executes application-defined functions as transactions, coupled with consistency guarantees via its sync protocol. 

<img src="https://cdn.sanity.io/images/ts10onj4/production/2ae339f39a62935266bd1518844dbdb32f5667f3-1252x832.svg" alt="High-level architecture" width="600">

A Convex deployment has three main components: the sync worker, the function runner, and the database.
The sync worker manages WebSocket sessions and tracks each client's active query set.
The function runner executes user-defined functions inside V8 isolates and caches their results.
The database owns the schema, tables, indexes, the committer, the transaction log, and subscriptions.

<img src="https://cdn.sanity.io/images/ts10onj4/production/080ed291f78449353715fd412f161633ac237078-1412x1627.svg" alt="Deployment internals" width="600">

| Component | Crate | Key files |
|-----------|-------|-----------|
| Sync Worker | `crates/sync/` | `worker.rs`, `state.rs` |
| Function Runner | `crates/function_runner/` | `server.rs`, `in_memory_indexes.rs` |
| Function Cache | `crates/application/` | `cache/mod.rs` |
| V8 / Isolate Runtime | `crates/isolate/` | `isolate_worker.rs`, `isolate2/runner.rs` |
| Database | `crates/database/` | `database.rs`, `lib.rs` |
| Committer | `crates/database/` | `committer.rs` |
| Transaction Log | `crates/database/` | `write_log.rs`, `snapshot_manager.rs` |
| Subscriptions | `crates/database/` | `subscription.rs` |
| Transactions | `crates/database/` | `transaction.rs`, `reads.rs`, `writes.rs` |
| Indexes | `crates/database/` | `transaction_index.rs`, `index_workers/` |
| Local Backend | `crates/local_backend/` | `lib.rs` |
| Application | `crates/application/` | `lib.rs`, `application_function_runner/mod.rs` |

---

## Functions

There are three function types: queries (read-only), mutations (read-write), and actions (side effects allowed).
Queries and mutations run as transactions within the database.

## Transaction Log and Index

An append-only data structure that stores all versions of documents within the database. Every version of a document contains a monotonically increasing timestamp within the log that’s like a version number.

The transaction log contains all tables’ documents mixed together in timestamp order, where all tables share the same sequence of timestamps.

Each timestamp `t` defines a snapshot of the database that includes all revisions up to `t`.

Indexes are built on top of the log, mapping each `_id` to its latest value. They are implementated using standard multiversion concurrency control (MVCC) techniques so the index can be queried at any past timestamp.
The system does not store many copies of each value -- see [CMU's Advanced DB Systems](https://www.cs.cmu.edu/~15721-f25/schedule.html) for more info.

<img src="https://cdn.sanity.io/images/ts10onj4/production/731ffc87aab77be756fe915d6fc3fe6f5ecbf45b-1124x980.svg" alt="Index mapping" width="600">

---

## Transactions and Optimistic Concurrency Control

All transactions are serializable, which means that their behavior is exactly the same as if they executed one at a time. It is implemented using optimistic concurrency control, which assumes that conflicts between txs are rare, record what each tx reads and writes, and check for conflicts at the end of the tx's execution. 

Transactions have three main ingredients: a begin timestamp [1], their read set, and their write set.

This timestamp chooses a snapshot of the database for all reads during the transaction’s execution.

After querying the index, we record the index range we scanned in the transaction’s read set. The read set precisely records all of the data that a transaction queried.

Instead of writing to the Tx log or indexes immediately, the transaction accumulates updates in its write set, which contains a map of each ID to the new value proposed by the transaction.

[1] Convex’s timestamps are [Hybrid Logical Clocks](https://cse.buffalo.edu/tech-reports/2014-04.pdf) of nanoseconds since the Unix epoch in a 64-bit integer.

The `Transaction` struct lives in `crates/database/src/transaction.rs`.
Read set tracking is in `crates/database/src/reads.rs`.
Write set accumulation is in `crates/database/src/writes.rs`.

## Commit Protocol

The committer in our system is the sole writer to the transaction log, and it receives finalized transactions, decides if they’re safe to commit, and then appends their write sets to the transaction log.

The committer starts by first assigning a commit timestamp to the transaction that’s larger than all previously committed transactions.

We can check whether it’s serializable to commit our transaction at timestamp 19 by answering the question, “Would our transaction have the exact same outcome if it executed at timestamp 19 instead of timestamp 16?”.

To do this, we can check whether any of the writes between the begin timestamp and commit timestamp overlap with our transaction’s read set.

If, however, we found a concurrent write that overlapped with our transaction’s read set, we have to abort the transaction. This error signals that the transaction conflicted with a concurrent write and needs to be retried. The function runner will then retry addCart at a new begin timestamp past the conflict write.

Our commit protocol is similar in design to FoundationDB’s and Aria’s. 

<img src="https://cdn.sanity.io/images/ts10onj4/production/b2c6c8c76eba8c59fe7cd381cf1e97e24debe0dd-1124x1172.svg" alt="Conflict detection" width="600">

The committer implementation is in `crates/database/src/committer.rs`.
The database-level commit entry point is in `crates/database/src/database.rs`.

### Detailed Commit Flow

Transactions do not write to the transaction log immediately.
Instead, they collect reads and writes locally, then submit the finalized transaction to the committer for validation.

During execution, `Transaction` accumulates reads into `TransactionReadSet` and writes into `Writes` (an `OrdSet<Update>`).
On completion, the transaction is finalized into a `FinalTransaction` containing both sets.
`Committer::start_commit()` receives the `FinalTransaction` and calls `validate_commit()`.

The validation step assigns a commit timestamp and calls `commit_has_conflict()`.
This method checks the read set against two sources: already-published writes in `LogWriter` and staged-but-not-yet-published writes in `PendingWrites`.
Checking both sources prevents two concurrent transactions from both passing validation against published state while conflicting with each other.

The conflict check window is `(begin_timestamp, commit_ts]`.
For each write in that window, `ReadSet::writes_overlap_docs()` checks whether the written document's index keys fall within any interval the transaction read.

If no conflict is found, the commit is staged in `PendingWrites`, then `write_to_persistence()` durably writes it, and finally `publish_commit()` pops it from pending, appends to the write log, and updates the snapshot manager.
If a conflict is detected, an OCC error is returned and the function runner retries at a new begin timestamp.

<img src="commit-flow.png" alt="Detailed commit flow sequence diagram" width="600">

| Component | File | Key struct / method |
|-----------|------|---------------------|
| Write accumulation | `writes.rs` | `Writes::update()` |
| Read tracking | `reads.rs` | `ReadSet`, `TransactionReadSet` |
| Transaction | `transaction.rs` | `Transaction` -> `FinalTransaction` |
| Commit validation | `committer.rs` | `Committer::validate_commit()` |
| Conflict detection | `reads.rs` | `ReadSet::writes_overlap_docs()` |
| Pending staging | `write_log.rs` | `PendingWrites::push_back()`, `is_stale()` |
| Persistence write | `committer.rs` | `write_to_persistence()` |
| Publish | `committer.rs` | `publish_commit()` |

All files under `crates/database/src/`.

## Subscriptions

Read sets also power realtime updates.
After running a query, the system keeps its read set in the client's WebSocket session within the sync worker.
When new entries appear in the transaction log, the same overlap-detection algorithm determines whether the query result might have changed.

The subscription manager aggregates all client sessions, walks the transaction log once, and efficiently identifies which subscriptions are invalidated.

<img src="https://cdn.sanity.io/images/ts10onj4/production/06ad6e04496479c2c5cd7362c27f606e04f0d51e-1268x1300.svg" alt="Subscription manager" width="600">

The subscription manager lives in `crates/database/src/subscription.rs`.
Local backend subscription handling is in `crates/local_backend/src/subs/mod.rs`.
The sync worker that manages WebSocket sessions and query sets is in `crates/sync/src/worker.rs`.

## Function Cache

Serving a cached result from memory is much faster than spinning up a V8 isolate.
Convex automatically caches queries, and the cache is always fully consistent.
It uses the same overlap-detection algorithm as the subscription manager to determine whether a cached result's read set is still valid at a given timestamp.

The cache implementation is in `crates/application/src/cache/mod.rs`.

---

## Not Covered Here

Actions, auth, end-to-end type-safety, file storage, virtual system tables, scheduling, crons, import/export, text search and vector search indexes, pagination, and more.
