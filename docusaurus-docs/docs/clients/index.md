---
title: Client Libraries
description: Dgraph client libraries in various programming languages.
---

Dgraph client libraries allow you to run DQL transactions, queries and mutations in various programming languages.

If you are interested in clients for GraphQL endpoint, please refer to [GraphQL clients](/graphql/graphql-clients) section.


Go, python, Java, C# and JavaScript clients are using **[gRPC](http://www.grpc.io/):** protocol and [Protocol
  Buffers](https://developers.google.com/protocol-buffers) (the proto file
used by Dgraph is located at
[api.proto](https://github.com/dgraph-io/dgo/blob/master/protos/api.proto)).

A JavaScript client  using **HTTP:** is also available.


It's possible to interface with Dgraph directly via gRPC or HTTP. However, if a
client library exists for your language, that will be an easier option.

:::tip
For multi-node setups, predicates are assigned to the group that first sees that
predicate. Dgraph also automatically moves predicate data to different groups in
order to balance predicate distribution. This occurs automatically every 10
minutes. It's possible for clients to aid this process by communicating with all
Dgraph instances. For the Go client, this means passing in one
`*grpc.ClientConn` per Dgraph instance, or routing traffic through a load balancer.
Mutations will be made in a round robin
fashion, resulting in a semi-random initial predicate distribution.
:::


### Transactions

Dgraph clients perform mutations and queries using transactions. A
transaction bounds a sequence of queries and mutations that are committed by
Dgraph as a single unit: that is, on commit, either all the changes are accepted
by Dgraph or none are.

A transaction always sees the database state at the moment it began, plus any
changes it makes --- changes from concurrent transactions aren't visible.

On commit, Dgraph will abort a transaction, rather than committing changes, when
a conflicting, concurrently running transaction has already been committed.  Two
transactions conflict when both transactions:

- write values to the same scalar predicate of the same node (e.g both
  attempting to set a particular node's `address` predicate); or
- write to a singular `uid` predicate of the same node (changes to `[uid]` predicates can be concurrently written); or
- write a value that conflicts on an index for a predicate with `@upsert` set in the schema (see [upserts](/dql/upserts)).

When a transaction is aborted, all its changes are discarded.  Transactions can be manually aborted.

#### Abort reasons

A write-write conflict is not the only reason a commit can abort. When Dgraph
aborts a commit it now reports a *category* describing why, so clients can decide
how to react (for example, retry immediately versus back off). The category is
carried as a `"<code>: <detail>"` prefix on the gRPC `ABORTED` status message
(and on the HTTP error message), where `<code>` is one of:

- `conflict` — a write-write conflict with another concurrent transaction, as
  described above. Retrying the transaction typically succeeds.
- `stale-startts` — the transaction's start timestamp predates the current Zero
  leader's lease (for example, after a leader change). Retrying with a fresh
  transaction succeeds.
- `predicate-move` — a predicate the transaction wrote is being moved between
  groups, so commits on it are temporarily blocked. Retrying once the move
  completes succeeds.

For example, a conflict abort carries the message
`conflict: Transaction has been aborted. Please retry`.

:::note
Older Dgraph servers do not emit a category prefix. Clients that parse the
reason degrade gracefully and report it as `UNKNOWN` in that case, so existing
error handling keeps working unchanged.
:::

The official [Java](/clients/java) and [Python](/clients/python) clients parse
this category into a typed enum; see those pages for the API.


### In this section
