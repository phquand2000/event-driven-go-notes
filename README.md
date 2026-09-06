# Event-Driven Go Services — Field Notes

Patterns from running a multi-service Go platform in production: ~20 services,
one Postgres per service, a single NATS JetStream event stream, proto as the
source of truth.

These are notes on the decisions that turned out to matter, not a framework and
not a tutorial. No business logic here.

---

## Proto is the source of truth, and generated code is committed

Every RPC surface and every event lives in `proto/`, managed with Buf. Generated Go
is committed to the repository rather than produced at build time.

Committing generated code is unfashionable. It earns its keep:

- A reviewer sees the wire-level effect of a schema change in the same diff as the change
- CI does not need a protoc toolchain to build
- A breaking change is a visible diff, not a runtime surprise

Buf's breaking-change detector runs against the committed output, so schema drift fails
the build rather than production.

## Connect over plain gRPC

[Connect](https://connectrpc.com/) speaks gRPC and plain HTTP/1.1 from one handler. In
practice this means a service is reachable with `curl` during an incident, and browser
clients need no proxy layer. The cost is a smaller ecosystem than gRPC proper. For a
platform with browser frontends as first-class consumers, this trade favours Connect.

## One logical database per service

No shared tables across contexts. Cross-context reads go through RPC or through a
projection built from events.

The discipline this enforces is worth more than the convenience it costs: a service
cannot quietly grow a dependency on another service's schema, so bounded contexts stay
bounded under deadline pressure — which is the only time it actually matters.

Migration policy differs by environment on purpose: `AutoMigrate` in development so
iteration is fast, explicit per-service migrations in production so changes are
reviewable and reversible.

## The transactional outbox

**The single most important pattern here.**

A service that writes to its database and then publishes an event has two failure modes:
crash after commit (event lost, state changed) or publish before commit (event describes
state that never existed). Both corrupt downstream consumers, and both are rare enough to
survive testing and appear in production.

The outbox removes the choice. The state change and the outbox row are written in **one
transaction**:

```
BEGIN
  UPDATE order SET status = 'confirmed' WHERE id = ...
  INSERT INTO outbox (event_type, payload, created_at) VALUES (...)
COMMIT
```

A separate dispatcher polls the outbox and publishes to the broker. If it crashes, it
retries; if it double-publishes, consumers deduplicate. The database transaction is the
only atomic boundary in the system, so everything that must be atomic is put inside it.

Consequence to design for from day one: **delivery is at-least-once, so every consumer
must be idempotent.** Retrofitting idempotency onto consumers written under an
exactly-once assumption is a large and unpleasant migration.

## One stream, many consumers

All events land on a single JetStream stream, proto-encoded as CloudEvents, with
per-consumer filtering by subject.

The practical benefit is a single ordered log to replay when debugging, and a single
place to attach an audit consumer that persists everything. A per-service stream
topology gives finer retention control but makes cross-context causality much harder to
reconstruct — and reconstructing causality is most of what incident response is.

## Session state in the KV store, not the database

Sessions live in NATS KV rather than Postgres. Session reads happen on nearly every
request; they are ephemeral, TTL-bound, and carry no relational structure. Putting them
in KV keeps that load off the databases that hold the data actually worth protecting.

## BFF per frontend, not one shared API

Each frontend app gets its own service surface shaped for that app, rather than every
client sharing one general-purpose API.

This duplicates some code. It also means a change for the admin console cannot break the
warehouse client, and each surface stays small enough to reason about. For a platform
with several distinct actor types, that isolation has consistently been worth the
duplication.

---

## What I would tell someone starting this

1. Adopt the outbox before you need it. Adding it later means auditing every publish site.
2. Make consumers idempotent from the first one. There is no cheap retrofit.
3. Commit generated code. The review quality gain is immediate.
4. Do not share a database "just for now". That decision is effectively permanent.
5. Keep one replayable log. Debugging distributed systems is mostly archaeology.
