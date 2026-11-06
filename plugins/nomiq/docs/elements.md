# nomiq Element Reference (AST nodes)

Elements are the **backend's internal representation** of a nomiq domain model,
derived from the `.nomiq` DSL sources by the compiler. The DSL is the only
source of truth — elements are never authored or stored in a project. This
reference exists to explain the underlying semantics the DSL compiles to
(useful when interpreting validator messages that mention element concepts)
and for migrating legacy `.nomiq/model` JSON stores via `nomiq_dsl_import`.

Scripts (`condition`, `predicate`, `value`, `rule`, effect expressions) are covered
in `scripting.md`; which references and functions are legal in each script slot is
covered in `slot-contexts.md`.

## 1. Common ground

Every element extends the same base:

```json
{
  "id": "V1StGXR8IZ5j",
  "pid": "000000000000",
  "metatype": "aggregate",
  "name": "Order",
  "desc": "## Purpose\n...\n\n## Need\n..."
}
```

| Property | Rule |
|---|---|
| `id` | 12-character id, derived deterministically by the compiler from the element's metatype and path. Ids never appear in DSL sources and are never authored; renaming an element re-keys it. |
| `pid` | The parent element's id — this forms the model tree. |
| `metatype` | One of: `domain context aggregate projection process command query entity valuetype term subscription policy enumeration gateway integration push pull` (`event` exists but is never authored — see §7.2). |
| `name` | **PascalCase** for all elements (`PlaceOrder`, `OrderId`). Member names inside elements (fields, inputs, errors, invariants) are **camelCase**, and so are enumeration items. |
| `desc` | Markdown. Convention: `## Purpose` (what it is) and `## Need` (why it exists) sections. |

Two reserved ids:

- `000000000000` (**ROOT_ID**) — the domain root's id. Exactly one domain element, with exactly this id.
- `------------` (**NONE_ID**) — the domain root's `pid` (it has no parent).

### Tree-shape rules

```
domain (root, id = 000000000000)
├── term | valuetype | enumeration     ← vocabulary, always directly under root
├── entity                             ← reusable component-with-identity, under root
└── context                            ← bounded context
    ├── aggregate
    │   ├── command
    │   └── query
    ├── projection
    │   ├── subscription
    │   └── query
    └── process
        ├── command (kind: "intent")
        └── policy
```

Anything else is invalid: no aggregate directly under the root, no command under a
context, no vocabulary inside a context.

## 2. Types and fields

A field's type is a reference to another element (or a primitive):

```json
{ "functor": "id",   "elementId": "V1StGXR8IZ5j" }   // single value
{ "functor": "list", "elementId": "V1StGXR8IZ5j" }   // ordered collection
```

Primitive `elementId`s are the literal strings `#string`, `#integer`, `#float`,
`#boolean`, `#date`, `#money`.

> **Rule: primitives are never used directly as field types.** Always wrap the
> primitive in a term that names the concept: an `email` field is typed by an
> `EmailAddress` term wrapping `#string`, not by `#string` itself. Primitives
> appear only inside `term.primitive`.

A field:

```json
{
  "name": "orderTotal",
  "desc": "",
  "type": { "functor": "id", "elementId": "<Price term id>" },
  "value": { "type": "ref", "qname": "!/orderTotal" }
}
```

`value` is a Script and is only meaningful on **event fields** (the mapping that
computes the event field when the command fires) — leave it off everywhere else.

These metatypes carry a `fields` array: `aggregate command event projection
process query valuetype entity push pull integration`. What `fields` *means*
differs:

| Metatype | `fields` are… |
|---|---|
| aggregate, projection, process, entity, valuetype | the record shape (state / read model / value) |
| command | the request payload |
| query, push, pull | input parameters (a pull's answer is its separate `result`) |

## 3. Vocabulary elements

### 3.1 `term` — a named primitive with constraints

```json
{
  "id": "…", "pid": "000000000000", "metatype": "term",
  "name": "OrderId", "desc": "",
  "primitive": "string",
  "constraints": [ { "type": "max-length", "value": 40 } ]
}
```

`primitive`: `string | integer | float | date | boolean | money`.

All 16 constraint kinds (apply only those matching the primitive):

| Constraint JSON | Applies to |
|---|---|
| `{ "type": "max-length", "value": n }` (positive int) | string |
| `{ "type": "min-length", "value": n }` (non-negative int) | string |
| `{ "type": "pattern", "regex": "…" }` | string |
| `{ "type": "format", "value": "email\|uuid\|uri\|hostname\|ipv4\|ipv6\|date-time" }` | string |
| `{ "type": "starts-with", "value": "…" }` | string |
| `{ "type": "ends-with", "value": "…" }` | string |
| `{ "type": "contains", "value": "…" }` | string |
| `{ "type": "greater-than", "value": n }` | integer, float, money |
| `{ "type": "greater-than-or-equal", "value": n }` | integer, float, money |
| `{ "type": "less-than", "value": n }` | integer, float, money |
| `{ "type": "less-than-or-equal", "value": n }` | integer, float, money |
| `{ "type": "multiple-of", "value": n }` (positive) | integer, float |
| `{ "type": "precision", "value": n }` (non-negative int) | float, money |
| `{ "type": "before", "date": "YYYY-MM-DD" }` | date |
| `{ "type": "after", "date": "YYYY-MM-DD" }` | date |
| `{ "type": "within-days", "days": n }` (positive int) | date |

### 3.2 `enumeration` — a closed set of named values

```json
{
  "id": "…", "pid": "000000000000", "metatype": "enumeration",
  "name": "OrderStatus", "desc": "",
  "items": [
    { "name": "pending", "desc": "" },
    { "name": "shipped", "desc": "" }
  ]
}
```

Items are camelCase. Scripts reference items as `[OrderStatus::pending]`.

### 3.3 `valuetype` — an immutable record without identity

```json
{
  "id": "…", "pid": "000000000000", "metatype": "valuetype",
  "name": "Address", "desc": "",
  "fields": [
    { "name": "street", "desc": "", "type": { "functor": "id", "elementId": "<StreetName term>" } },
    { "name": "city",   "desc": "", "type": { "functor": "id", "elementId": "<CityName term>" } }
  ]
}
```

### 3.4 `entity` — a component with identity, embedded in aggregates

Same shape as valuetype plus `"invariants": []`. Use for aggregate-internal
collections whose items must be addressed individually (e.g. `OrderLine` inside
`Order`, referenced as `{ "functor": "list", "elementId": "<OrderLine entity>" }`).

## 4. Structural elements

### 4.1 `domain` — the root (exactly one)

```json
{
  "id": "000000000000",
  "pid": "------------",
  "metatype": "domain",
  "name": "Shop",
  "desc": ""
}
```

`id` and `pid` are fixed by schema — any other value is rejected.

### 4.2 `context` — a bounded context

```json
{ "id": "…", "pid": "000000000000", "metatype": "context", "name": "Ordering", "desc": "" }
```

No extra properties. Contains aggregates and projections.

### 4.3 `aggregate` — write-side state, event-sourced

```json
{
  "id": "…", "pid": "<context id>", "metatype": "aggregate",
  "name": "Order", "desc": "",
  "fields": [
    { "name": "id",     "desc": "", "type": { "functor": "id", "elementId": "<OrderId term>" } },
    { "name": "status", "desc": "", "type": { "functor": "id", "elementId": "<OrderStatus enum>" } },
    { "name": "lines",  "desc": "", "type": { "functor": "list", "elementId": "<OrderLine entity>" } }
  ],
  "invariants": []
}
```

Rules: every aggregate has an **identity field named exactly `id`**, typed by a
`{Name}Id` term (`Order` → `id: OrderId`) — the code generators key the document
store on this field and crash without it. State is only ever changed by effects
folding events (§7.1).
`invariants` are structural predicates over own fields:

```json
{ "name": "linesNotEmpty", "desc": "", "predicate": { "type": "fun", "name": "gt", "args": [ … ] } }
```

### 4.4 `projection` — read-side view

Same shape as aggregate minus invariants: `fields` describe one row/document of the
read model, and — like aggregates — the row identity field must be **named exactly
`id`** (it maps to the document store's `_id`). **Every projection needs at least
one subscription** — otherwise nothing ever writes to it.

### 4.5 `process` — a long-lived, correlation-keyed state machine

```json
{
  "id": "…", "pid": "<context id>", "metatype": "process",
  "name": "BoxBooking", "desc": "",
  "fields": [
    { "name": "id", "desc": "", "type": { "functor": "id", "elementId": "<ReservationId term>" } }
  ],
  "invariants": [],
  "states": [ { "name": "started", "desc": "" } ],
  "finals": [
    { "name": "reserved", "desc": "" },
    { "name": "failed",   "desc": "" },
    { "name": "expired",  "desc": "" }
  ]
}
```

`fields` are the per-instance state; the mandatory `id` field is the
**correlation key**. The machine dimension:

- `states` — the declared `state` blocks; `finals` — absorbing terminal
  states (bodiless: no handlers, no deadlines). The state space is exactly
  `states ∪ finals`; every transition target must be in it. Both arrays
  preserve declaration order.
- **Implicit `state` column** — every process row carries a machine-state
  column named `state`. It is **NOT in `fields`**: users neither declare nor
  script it (a user field named `state` is an error; `self.state` is
  reserved). The engine writes it from each transition's `to`.
- **Derived state enum** — each process derives a root-level `enumeration`
  named `<Process>State` with `id = stateEnumId(process.id)` (case-swap
  involution, beside `mirroredId` = reverse and `rejectionId` = reverse +
  swap), items = states ∪ finals. Never authored, never printed —
  referencable everywhere (readers fold milestone facts into it).

Children are intent contracts (commands with `"kind": "intent"`, §5) and
policies — the machine's handlers (§6.3). The order of a process's policy
children is **semantically load-bearing** (first-match, §6.3). A process is
the only element that dispatches decisions to peers;
delivery/replay/deadline/finality semantics are specified in
`_docs/semantics.md`.

## 5. `command` — the write-side entry point

```json
{
  "id": "…", "pid": "<aggregate id>", "metatype": "command",
  "name": "PlaceOrder", "desc": "",
  "fields": [
    { "name": "orderId", "desc": "", "type": { "functor": "id", "elementId": "<OrderId term>" } },
    { "name": "lines",   "desc": "", "type": { "functor": "list", "elementId": "<OrderLine entity>" } }
  ],
  "event": {
    "name": "OrderPlaced",
    "fields": [
      { "name": "placedAt", "desc": "",
        "type": { "functor": "id", "elementId": "<Timestamp term>" },
        "value": { "type": "fun", "name": "now", "args": [] } }
    ]
  },
  "invariants": [],
  "inputs": [],
  "errors": [],
  "effects": [],
  "access": { "rule": { "type": "fun", "name": "…", "args": [] }, "injections": [] }
}
```

All of `invariants`, `inputs`, `errors`, `effects` are **required arrays** (use
`[]`). `event` and `access` are optional. `isManualFlow: true` marks a
contract-only element whose flow class is hand-written under the project's
`manual/` tree (DSL: bare `@manual` above the declaration — see
`docs/manual-flow.md`); `manualFlowCode` is deprecated and ignored.

Granular manual slots are marked per member instead, by `@manual(<method>)` in
the DSL: `input.isManual` (+ `input.type`, the explicit result type),
`error.isManualCondition`, `field.isManualValue` on an event field, and the
`manual` effect variant. Each carries an optional `manualName` — the gap-method
name from the annotation, falling back to the member's own name. All of an
element's gap names share one namespace on the generated flow class, so they
must be distinct and must not shadow its generated members.

Two optional properties distinguish process intents:

- `kind` — `'command' | 'intent'`. Absent/`'command'`: a decision against one
  aggregate instance, the ack is final (fact committed or rejected).
  `'intent'`: a decision on a process (`pid` = the process id) — the ack means
  accepted-for-pursuit; outcomes arrive later as facts.
- `correlate` — intents only, optional Script: computes the target instance
  key from the request (`correlate by request.<field>`). Required iff the
  intent has any state-scoped handler — an outside request carries no
  causation chain to find the instance.

**Intents are contracts**: `fields` + `correlate` + `access` and nothing
else. `event`, `inputs`, `errors`, `effects`, and `dispatches` are **invalid
on an intent** ("an intent is a contract — handling lives in the process's
init/state handlers", §6.3). Routing is by lifecycle position: no instance →
the init handlers (create); an instance → its current state's handlers; no
handler at that position → the implicit lifecycle rejection, a rejection
fact with `errors: ["lifecycle"]`. The same intent handled in init and in a
state is start-or-continue.

Plain commands may carry `dispatches` too, but only **push-port signals**
(§6.4) — a peer decision needs a process.

- `fields` — the request payload. **Must include the aggregate identity field.**
- `event.name` — past participle of the command (`PlaceOrder` → `OrderPlaced`,
  `ShipOrder` → `OrderShipped`).
- `event.fields` — *extra* fields the event carries beyond the request. Each needs
  a `value` script computing it. The derived event's full field list is
  `command.fields ++ event.fields` (request fields are copied in automatically).
- `inputs` — data hydrated from state before validation:

  ```json
  { "name": "currentOrder", "desc": "", "selection": "one",
    "condition": { "type": "fun", "name": "eq", "args": [ … ] } }
  ```

  `selection`: `one | many | page | exist | count | seqno`. A `seqno` input's
  condition is a **literal string prefix**, not an expression, and its optional
  `padding` (1..18, default 6) is how many digits the drawn counter is zero-padded
  to — `{"selection": "seqno", "condition": {…"ORD-"}, "padding": 8}` hydrates
  `ORD-00000042`. The prefix is emitted **verbatim** and is not counted by
  `padding`, so it owns its separator: `"ORD"` draws `ORD000042`. `padding` on
  any other selection is an err. On a command, inputs read the **enclosing
  aggregate's own store only** — an optional `sourceId` (a **sourced input**,
  reading another element's read model) is **rejected on commands**: an
  aggregate is a consistency boundary and must decide from its own state plus
  the command payload, never from an unlocked cross-boundary snapshot. Sourced
  reads stay available where they are coordination rather than decision —
  projection subscriptions and process policies — and are advisory snapshots
  there: never locked, no consistency guarantee; invariants that depend on the
  data belong on the aggregate that owns it.
- `errors` — validations; the command is rejected when `condition` is TRUE:

  ```json
  { "name": "orderAlreadyExists", "desc": "", "message": "Order already exists.",
    "condition": { "type": "fun", "name": "is-not-null",
                   "args": [ { "type": "ref", "qname": "currentOrder" } ] } }
  ```
- `effects` — how the produced event folds into aggregate state (§7.1).
  In practice **instance ops** (`insert | replace | update | remove | upsert`):
  the schema now admits `updateMany`/`deleteMany` here too, but validation
  rejects them with a "generator support is pending" error until the
  command-flow templates emit multi-row writes. Fold many rows in a projection
  instead.
- `access` — authorization: `rule` is a boolean script over the `actor`'s fields;
  `injections` overwrite request fields from the actor (each lvalue must be
  `[_/<requestField>]`).

### Execution phases (why the slot rules differ)

```
ACCESS → HYDRATE (inputs) → VALIDATE (errors) → EVENT (field values) → EFFECT (fold)
```

Each phase sees different data — that is exactly what `slot-contexts.md` encodes.

## 6. `query`, `subscription`, `policy`

### 6.1 `query` — read-side (or aggregate-side) lookup

```json
{
  "id": "…", "pid": "<projection or aggregate id>", "metatype": "query",
  "name": "ListOrders", "desc": "",
  "selection": "page",
  "fields": [
    { "name": "status", "desc": "", "type": { "functor": "id", "elementId": "<OrderStatus enum>" } }
  ],
  "invariants": [],
  "condition": { "type": "fun", "name": "eq",
    "args": [ { "type": "ref", "qname": "_/status" }, { "type": "ref", "qname": "?/status" } ] }
}
```

- `selection`: `one | many | page | exist | count | seqno`. A `seqno` query takes
  the same literal prefix and optional `padding` as a `seqno` input.
- `fields` are the query **parameters** (not the result shape — results come from
  the parent's fields).
- `condition` optional; without it the query filters by matching parameter names
  against the parent's key fields.

### 6.2 `subscription` — projection's reaction to an event

```json
{
  "id": "…", "pid": "<projection id>", "metatype": "subscription",
  "name": "OnOrderPlaced", "desc": "",
  "triggerId": "<mirroredId of the PlaceOrder command id>",
  "inputs": [],
  "effects": []
}
```

- `triggerId` — the id of the **derived event**, which is the command's id
  **reversed** (`mirroredId`): command `V1StGXR8IZ5j` → event `j5ZI8RXGtS1V`.
- `inputs` — rows loaded before applying effects: from the projection's own
  store, or — with `sourceId` — a snapshot read of another element's read
  model (§5).
- `effects` — how the event folds into the projection (§7.1). Projections own
  no invariants, so the many-ops (`updateMany`/`deleteMany`) are legal here.

Subscriptions never communicate: cross-service writes are process dispatches,
foreign reads are sourced inputs.

### 6.3 `policy` — one handler of a process machine

A policy is "when \<trigger> [while in \<scope>] [if \<guard>], then …" —
the compiled form of an `on`/`after` handler inside the process's
`init`/`state` blocks (or top-level, for any-state).

```json
{
  "id": "…", "pid": "<process id>", "metatype": "policy",
  "name": "OnReservationCreated", "desc": "",
  "scope": { "kind": "state", "state": "started" },
  "trigger": { "type": "event", "eventId": "<mirroredId of a command/intent/policy id>" },
  "inputs": [],
  "errors": [],
  "invariants": [],
  "transition": { "to": "reserved", "data": [] },
  "dispatches": []
}
```

- `scope` — where the handler lives, a discriminated union:

  ```json
  { "kind": "init" }                       // creates the instance (keyed insert, no self)
  { "kind": "state", "state": "started" }  // requires an instance in that state (full self)
  { "kind": "any" }                        // every non-final state; fact triggers only (v1)
  ```

  `init` is a pseudostate: not in the state enum, never a transition target,
  no deadlines; init handlers take intent triggers only (v1).
- `trigger` — a discriminated union:

  ```json
  { "type": "event",    "eventId": "…", "correlate": <script, optional> }
  { "type": "deadline", "eventIds": [ "…" ], "duration": { "value": 15, "unit": "m" } }
  { "type": "intent",   "intentId": "<sibling intent id>" }
  ```

  An event trigger fires when the fact is observed; correlation follows the
  causation chain unless `correlate` computes the key from the event (only
  needed for join events). A deadline trigger fires when `duration` (`unit`:
  `s | m | h | d`) elapses in the enclosing state; `eventIds` are the
  milestone facts of **every transition into that state** — each arms it, a
  later one supersedes the pending clock, and firing re-checks the instance
  is still in the state. An intent trigger fires when the sibling contract
  (§5) is requested while the instance is at the handler's lifecycle
  position.
- `guard` — optional handler-selection guard: same-trigger variants are
  tried in **declaration order**, first match fires; the unguarded variant
  is the else and must be last. Guards read the trigger + self (state scope
  only) — never inputs (replay determinism). Facts with no matching handler
  are ignored; intents are rejected.
- `inputs` — as on subscriptions (sourced reads allowed); advisory snapshots.
- `errors` / `invariants` — handler-owned, as on commands: a TRUE error
  condition rejects the trigger with a correlated rejection fact.
- `event` — optional milestone, same shape as a command's `event`: records
  the reaction as a milestone fact of the process. Its derived event id is
  `mirroredId(policy.id)`. **Intent-handler milestones carry the intent's
  request fields + the declared fields** (`eventOfPolicy(policy,
  intentFields)` — mirrors `eventOf` for commands); fact/deadline handlers
  carry declared fields only.
- `transition` — the fold (there are **no effects on policies**):

  ```json
  { "to": "reserved", "data": [ …bindings… ] }
  ```

  `to` must be a declared state or final. Stored structurally, never
  lowered to an effect: the write is **keyed** by the resolved instance
  (insert for init handlers — `data` must bind `id`; patch otherwise), so
  no `where` filter exists anywhere. The implicit `state` column is set by
  the engine from `to` and is not bindable in `data`.
- `dispatches` — decisions sent to other elements (§6.4). Required array.

**First-match order invariant**: a process's policy children array is in
declaration order, and that order is semantically load-bearing — exactly one
handler fires per delivery, selected by first match. Every producer and
store must preserve the array order.

### 6.4 Dispatches — decisions sent to other elements

Carried by policies (`policy.dispatches` — the process is the sender) and by
plain commands (`command.dispatches` — **push-port signals only**, the
service's own outbound adapters). Intents never dispatch — they are
contracts (§5).

```json
{ "desc": "", "condition": <script, optional>,
  "commandId": "<target command / intent / push id>",
  "data": [ { "lvalue": { "type": "ref", "qname": "reservationId" },
              "rvalue": { "type": "ref", "qname": "!/reservationId" } } ] }
```

`data` bindings target the **target's request fields**. A policy may target
commands, intents, and push ports; a plain command only its own push ports —
a peer decision needs a process. Unlike transitions, dispatches cross a
consistency boundary: the runtime delivers them reliably and suppresses them
when facts are replayed (`_docs/semantics.md` §3).

### 6.5 Timers — a command's deferred self-dispatch

Carried by plain commands (`command.timers`). Each arms one durable deferred
dispatch when the decision commits; on expiry the target — always a command of
the **same aggregate** — re-enters through the normal intake. Intents have no
timers (a contract does nothing); process waits are deadlines (§6.3).

```json
{ "desc": "",
  "interval": { "type": "ref", "qname": "!/graceMinutes" },
  "unit": "s" | "m" | "h" | "d",
  "key":       <script, optional — durable identity; re-arming supersedes>,
  "condition": <script, optional — gates ARMING, at decision time>,
  "dispatch":  <Dispatch, exactly one, targeting a sibling command> }
```

Unlike a deadline's `duration` (a literal `{value, unit}`), `interval` is an
**expression** in the dispatch scope — request, produced event, hydrated
inputs; `[_/…]` is an error because the fold has not run yet. That scope is
exactly why timers are dispatch-family: effects replay and may not read the
request, and arming must not replay at all.

No firing-time guard and no cancellation exist: a fired timer delivers, and the
target command's errors judge it, producing a `<Command>Rejected` fact. `key`
is the handle a future `cancel` would name.

## 7. Effects and events

### 7.1 Effects — the fold `state' = apply(state, event)`

Effects are pure: they may read event fields (`[^/…]`) and current state
(`[_/…]`) — never request fields, inputs, principal, or non-deterministic
functions (`now`, `random-id`, `unique-id`). Compute such values into an event
field first, then use the event field.

`Record` = array of bindings: `{ "condition"?, "lvalue": Ref, "rvalue": Script | Record | null }`.
`lvalue` names the target field; nested records set composite (valuetype/entity) fields.

The 8 variants (`desc` optional, defaults to `""`; `condition` optional on all):

```json
{ "type": "insert",     "data": [ …bindings… ] }
{ "type": "replace",    "data": [ …bindings… ] }
{ "type": "upsert",     "data": [ …bindings… ] }
{ "type": "update",     "where": <script>, "patch": [ …bindings… ] }
{ "type": "updateMany", "where": <script>, "patch": [ …bindings… ] }
{ "type": "remove",     "identity": <script> }
{ "type": "deleteMany", "where": <script> }
{ "type": "manual",     "name": "routeTokens", "kind": "update" }
```

`manual` is a hand-written effect: `name` is the abstract gap method on the
generated flow base, and the optional `kind` records the declared effect kind
for documentation only. Processes carry no effects at all — a policy's fold is
its structural `transition` (§6.3).

The write ops split by emitter:

- **Instance ops** (`insert replace update remove upsert`) — what aggregate
  commands actually use: a consistency boundary writes its own instance.
- **Many-ops** (`updateMany`/`deleteMany`) — projection subscriptions, which
  legitimately fold many rows from one fact. They are structurally legal on a
  command too, but validation rejects them there pending generator support.

Example — insert a projection row from an event:

```json
{
  "type": "insert",
  "data": [
    { "lvalue": { "type": "ref", "qname": "id" },
      "rvalue": { "type": "ref", "qname": "^/orderId" } },
    { "lvalue": { "type": "ref", "qname": "status" },
      "rvalue": { "type": "ref", "qname": "OrderStatus::pending" } }
  ]
}
```

`insert/replace/upsert` write a whole row; `update/updateMany` patch rows matching
`where`; `remove` deletes by identity; `deleteMany` by filter. Effects never
communicate — communication is a **dispatch** (§6.4): decisions to peer
commands/intents may only be sent by processes (policies); push ports (the
service's own outbound adapters) may be signalled by commands and policies.

### 7.2 Events are derived — never author them

There is an `event` metatype in the element schema, but clients never write event
elements. The server derives each command's event at validation/generation time:

- `event.id = mirroredId(command.id)` (the command id, reversed)
- `event.fields = command.fields ++ command.event.fields`
- `event.name = command.event.name`

A policy's optional milestone derives the same way from the policy's id:
`mirroredId(policy.id)`. Its fields are `policy.event.fields` — prefixed,
for **intent handlers**, by the triggering intent's request fields
(`eventOfPolicy(policy, intentFields)`). A process's derived state enum
(§4.5) is likewise never authored.

Consequences: an authored `event` element is an error; `subscription.triggerId`
and policy `trigger.eventId` always equal the reversed id of some command,
intent, or policy in the model.

## The `actor`

`{ fields: ActorField[] }`, exactly one per domain: the authenticated caller. Each
field adds `claim` (the token claim's path) and optional `isOptional`. Generating
the gateway needs it; a domain without one gets a validator warning.

## 8. Rarely authored: `gateway`, `integration`, `push`, `pull`

- `gateway` — `{ commands: Command[], queries: Query[] }` inline (not children); an
  external API facade.
- `integration` — a fieldable holding an external system's ports.
- `push` / `pull` — fieldables with optional `access`; outbound/inbound integration
  endpoints. Their `fields` are request parameters. A `pull` also carries
  `result: { functor: "id" | "list", fields: Field[] }` — the record it reads
  back, required (an absent one errs) and never shared with the enclosing
  integration's record. A `push` has no result: a port signal is
  fire-and-forget. Result fields are a record shape, so they carry no `value`.

Only author these when the user explicitly needs external integrations.

## 9. Common pitfalls

1. **Primitive as a field type** — `"elementId": "#string"` in a field. Wrap it in a term.
2. **Missing required arrays** — `invariants`, `inputs`, `errors`, `effects` on a
   command must be present even when empty.
3. **Authoring an event element** — events are derived; extra fields belong
   in `command.event.fields`.
4. **Command without the aggregate identity field** — the request cannot be routed.
   And **aggregate/projection/process identity fields not named `id`** — `orderId`
   as the aggregate's own field crashes the generator; the field is `id`, the
   *term* is `OrderId`.
5. **Wrong `pid` nesting** — e.g. a term under a context, a command under a
   projection. Check the tree in §1.
6. **Wrong casing** — element names PascalCase; member names and enum items
   camelCase.
7. **`triggerId` pointing at the command id** — it must be the *mirrored* (reversed)
   id.
8. **Projection with no subscription** — nothing will ever populate it.
9. **Non-deterministic values in effects** — compute `now()` etc. into an event
   field, then bind `[^/thatField]`.
10. **A process field named `state`** — the machine-state column is implicit
    and reserved; declaring it (or reading `self.state`) is an error.
11. **Reordering a process's policies** — handler selection is first-match in
    declaration order; the array order is meaning, not formatting.
12. **Handling on an intent** — `event`/`inputs`/`errors`/`effects`/`dispatches`
    on a `kind: "intent"` command err; an intent is a contract, handling lives
    in the process's init/state handlers.
13. **Redeclaring intent request fields in a milestone event** — an intent
    handler's milestone carries the request fields automatically
    (`eventOfPolicy` concatenates them with the declared fields, so a
    redeclaration yields a duplicated field). Declare extras only; usually
    the milestone is bodiless.
