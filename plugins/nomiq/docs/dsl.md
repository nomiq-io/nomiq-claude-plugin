# The nomiq DSL

A textual authoring surface for nomiq domain models — the **only source of
truth**. `.nomiq` sources compile to the elements described in
`plugin-nomiq/docs/elements.md` (the backend's internal representation) — the
DSL adds **zero new semantics**: every construct lowers to existing
`fun`/`lit`/`ref` script nodes and element JSON, so validators, slot rules
(`slot-contexts.md`), and code generation are untouched.

```
compile(sources) → { elements: Model.Element[], issues: Diagnostic[] }
print(elements)  → { files: PrintedFile[], issues: Diagnostic[] }
```

`print` is the inverse: it renders elements back into canonical `.nomiq`
sources (one root file for domain + vocabulary, one file per context) and
doubles as the formatter — `compile(print(compile(src)))` is the identity on
elements.

## 1. Modularity — Java-package-style contexts

All `.nomiq` files passed to `compile()` form **one compilation unit**. There
are no imports: root vocabulary is one flat namespace, and references resolve
globally in a second pass — declaration order and file boundaries never matter,
and cyclic references between contexts (event choreography) are fine.

A file starts with at most one header:

- **no header** — declarations belong to the domain root: `domain <Name>`
  (exactly once per unit) and vocabulary (`term`, `enum`, `valuetype`,
  `entity`, `actor`, `integration`, `gateway`, `push`, `pull`).
- **`context <path>`** — like a Java `package` statement: brace-less,
  lowercase dotted segments (`context shop.ordering`), scoping every
  declaration in the file (`aggregate`, `projection`, `process`,
  `integration`). Files
  sharing the same path **merge** into one context. `shop` and
  `shop.ordering` are unrelated flat contexts — dots are naming convention,
  not containment.

**Partial declarations**: a holder re-opened across files of one context
(same kind and name) merges its members — splitting a large holder across
files is a supported layout:

```nomiq
// shop.ordering/Order.nomiq           // any other file of the context
context shop.ordering                  context shop.ordering

aggregate Order {                      aggregate Order {
  field id: OrderId                      command PlaceOrder { … }
  field total: Price                   }
}
```

Duplicate same-kind leaf members (two `command PlaceOrder` parts, two
subscriptions on the same event, …) are errors. An aggregate and a
projection may share a name — their ids differ by metatype. The canonical
layouts the printer emits: `'split'` is one **directory per context** with
one whole-holder file per aggregate/projection/process/integration
(`domain.nomiq` at the root, then `<ctx>/<Holder>.nomiq` with the holder's
members inside); the default `'compact'` keeps one file per context. When
two holders in one context share a name, their files are qualified with the
metatype (`core/Wallet.aggregate.nomiq`, `core/Wallet.projection.nomiq`) so
neither
overwrites the other — file names are only a printer convention; compile
ignores them.

An `integration` (root-level or inside a context) may nest `push` and `pull`
ports:

```nomiq
integration LockerBridge {
  push OpenBox { field lockerId: LockerId }

  pull BoxStatus {
    field lockerId: LockerId

    result {
      field doorOpen: Flag
      field reportedAt: DateTime
    }
  }
}
```

A port's own `field`s are its request parameters. A `pull` additionally states
the record it reads back in a mandatory `result { … }` block — `result list
{ … }` when it reads back many. The answer belongs to the port, not to the
integration: one external system's resources have unrelated shapes.

`push` ports are valid `dispatch` targets alongside commands and intents
(`dispatch core.LockerBridge.OpenBox { … }` — §6). Because a port is the
service's own outbound adapter (no consistency boundary behind it), a plain
aggregate command may also carry `dispatch <port> { … }` clauses — the
declarative form of "this decision signals the device". Peer commands remain
process-only targets; `pull` ports are reads and never dispatch targets, and a
`push` never declares a `result` — it signals and reads nothing back.

```nomiq
// shop.nomiq                      // ordering.nomiq
domain Shop                        context shop.ordering

term OrderId: string               aggregate Order { … }
```

## 2. Lexical rules

- **Names**: element/type names PascalCase (`Order`, `PlaceOrder`); members
  camelCase (`orderId`). Context segments lowercase.
- **Members are newline-terminated** — no semicolons. A line continues when it
  is incomplete: inside `( … )`, after a trailing operator / `,` / `=` / `:` /
  word like `when if where from on set and or` or a selection keyword, or
  before a leading `when / then / else / from / and / or / {` or binary
  operator. (`set` and the selection keywords never continue a line from the
  left — they start clause lines inside blocks.)
- **Annotations**: `@manual` / `@manual(<method>)` on its own line marks the
  following declaration as hand-written (§10). `@manual` is the only
  annotation; any other `@name` is a compile error.
- **Comments**: `// line`, `/* block */`. **Doc comments** `/// …` attach to
  the following declaration/member as its `desc` (markdown preserved, so the
  `## Purpose` / `## Need` convention is written directly). A `///` run sits
  **directly above** what it documents — the blank line that sets it off goes
  above the comment, never between the comment and its member.
- **Formatting** — what `nomiq-fmt` emits (§11), so write it this way by hand
  and the formatter will leave it alone. Two-space indent, wrapped at 100
  columns, and **one blank line**:
  - between member **groups** — fields | event | inputs | errors | invariants
    | effects | dispatches | timers | access, in that order;
  - around any member carrying a `///` run or a `@manual` annotation;
  - around any member that opens a multi-line `{ … }` block.

  Consecutive plain one-liners of the same kind stay packed (two `field` lines,
  two `error` lines). **Clause** lines inside an effect / input / access /
  dispatch body (`when`, `original`, `oneOf`, `set`, `rule`, `inject`) are not
  members and stay packed. Never a blank line straight after `{` or before `}`.
- **Literals**: `"text"` (or `'text'`), `42`, `3.14`, `true`, `false`, `null`.
  Bare ISO dates (`2026-12-31`) are legal **only** in term constraints — the
  script AST has no date or money literal.
- **Backticks** force an identifier: an input literally named `event` is
  referenced as `` `event` ``.

## 3. Vocabulary (root-level)

```nomiq
term OrderId: string { max-length 40 }
term Price: money {
  >= 0                    // symbolic forms: > >= < <=
  precision 2
}
term Email: string { format email }
term DeliveryDate: date { after 2026-01-01  within-days 90 }
// all 16 constraints: max-length min-length pattern format starts-with
// ends-with contains > >= < <= multiple-of precision before after within-days

enum OrderStatus {
  /// Placed, awaiting shipment.
  item pending

  item shipped
}

valuetype Address {
  field street: StreetName
  field city: CityName
}

entity OrderLine {
  field sku: Sku
  field qty: Quantity

  invariant qtyPositive: self.qty > 0
}
```

Field types are always PascalCase element names — primitives exist only behind
terms. List types: `field lines: list<OrderLine>`.

## 4. Aggregates, commands, queries

```nomiq
context shop.ordering

aggregate Order {
  field id: OrderId              // 'id' field is mandatory
  field status: OrderStatus
  field total: Price

  invariant totalNonNegative: self.total >= 0

  command PlaceOrder {
    field orderId: OrderId       // request payload
    field total: Price

    event OrderPlaced {          // extra event fields, usually with a value
      field placedAt: Timestamp = now()   // value optional when hydrated via access injections
    }

    input existing {
      oneOf self.id == request.orderId
    }
    // selections: oneOf allOf pageOf existOf numberOf seqno
    // seqno takes a prefix string instead: input no { seqno "ORD-" }
    // the prefix is emitted verbatim — it owns its separator, if any
    // an optional width follows it — 'seqno "ORD-" 8' draws ORD-00000042; the
    // width sizes the counter alone (not the prefix), and defaults to 6
    // a command's inputs read THIS aggregate's own store only — a 'from' clause
    // (a sourced input) is rejected here, because an aggregate must decide from
    // its own state and the command. Sourced reads belong on projection
    // subscriptions and process policies; see §"Subscriptions" below.

    error orderAlreadyExists "Order already exists." when existing != null

    effect insert {
      set id = event.orderId
      set status = OrderStatus::pending
      set total = event.total
    }

    access {
      rule actor.id != null
      inject customerId = actor.id           // lowers to [_/customerId]
      // conditional injects: first matching 'when' wins, in declaration
      // order; when none matches the request must supply the field
      inject tier = "GOLD" when actor.roles contains "vip"
    }
  }

  query GetOrder {
    field orderId: OrderId       // query parameters

    oneOf self.id == params.orderId
  }
}
```

Queries and inputs share one **selection vocabulary**: `oneOf` (zero-or-one
row), `allOf` (every matching row), `pageOf` (a page), `existOf` (does one
exist), `numberOf` (a count), plus `seqno "PFX" [<digits>]` (a sequence draw).
A query's selection clause is required — it types the result — and its
predicate is optional (`allOf` alone reads everything).

`seqno` is the odd one: it takes a **literal** prefix, not a predicate, and
draws the next counter under it as a string. The prefix is emitted **verbatim**
— it is the whole prefix, separator included: `seqno "ORD-"` yields `ORD-000042`
and `seqno "ORD"` yields `ORD000042`.
The optional trailing number is the counter's **width in digits** (1..18,
default 6): `seqno "ORD-" 8` yields `ORD-00000042`. It pads the counter alone —
the prefix is not counted, so the width survives a change of prefix — and it is
a minimum, not a cap: once the counter outgrows the width it prints in full, and
the series stops sorting lexicographically. Size it for the volume you expect.

The prefix is also the counter's **identity** — it keys the stored counter
document. Changing it starts a **new counter at 1**, so adding a separator to a
prefix already in production is not a rename: `"ORD"` and `"ORD-"` are two
independent series.

Effects use a **subset**: only `oneOf` and `allOf` select rows to write (plus
`identity` for a single-row delete). `pageOf`/`existOf`/`numberOf`/`seqno` read,
so they have no meaning in a write, and an effect selection always needs a
predicate — unlike a query's, it may not stand alone.

Inputs and queries put **one clause per line**: their bodies end each clause at
the newline, so `input row { from X  oneOf … }` is a parse error. Effect and
dispatch blocks parse their clauses as a list and do tolerate a one-liner.

### Effects

An effect body is an **ordered clause list**: `when` (should it run) →
`original` (record ops) → selection / `identity` (which rows) → `set` lines
(what changes).

```nomiq
effect insert {                    // also: replace, upsert — record ops
  set id = event.orderId
  set status = OrderStatus::pending
}

effect replace {
  original current                 // optional original ref (a name or event.foo)
  set status = OrderStatus::shipped
}

effect update {
  when event.status == OrderStatus::active   // optional execution condition
  oneOf self.id == event.orderId             // the one matching row
  set status = OrderStatus::shipped
}

effect update {
  allOf self.customerId == event.id          // every matching row
  set status = OrderStatus::stale
}

effect delete {
  identity event.orderId                     // single-row delete by key
}

effect delete {
  allOf self.expired == true
}
```

- `when` gates whether the effect runs at all; `oneOf`/`allOf` select the
  rows it touches; `identity` names the single row a delete removes.
  `delete { oneOf … }` is reserved — the backend has no selection-based
  single delete yet; use `identity`.
- Many-ops (`allOf`) are grammatical in aggregate commands too — the
  historical "an aggregate instance writes only itself" restriction is
  lifted at the DSL level; platform validation still flags them until the
  command-flow generators support multi-row writes. Projections fold
  freely. Process handlers never use `effect` at all — their folds are
  `transition`s (§6).
- The guard comes first. A `when` written on its own line *after* the `set`
  lines would otherwise be swallowed as that last binding's condition — two
  different meanings that look identical — so the parser rejects it and asks
  you to move it up. A per-binding guard goes on the same line as its `set`.
- `delete` takes no `set` clauses: it removes rows, it does not write them.
- Bindings: `set field = <expr>` — plus `set field = { nested }` for
  composite fields (nested bindings are bare `field = <expr>`),
  `set field = null` for explicit clears, `set field = expr when cond` for
  conditional bindings, and dotted patch lvalues (`set address.city = …`)
  in update patches.

### Timers — `after <interval> <unit>`

A command may arm a **timer**: after the interval elapses, one of the
aggregate's **own** commands is delivered back to it.

```nomiq
command PlaceHold {
  field holdId: HoldId
  field graceMinutes: GraceMinutes

  event HoldPlaced

  effect insert {
    set id = event.holdId
    set status = HoldStatus::pending
  }

  after request.graceMinutes minutes {
    when request.graceMinutes > 0     // gates ARMING, at decision time
    key event.holdId                  // durable identity — re-arming supersedes
    dispatch ExpireHold {
      set holdId = event.holdId
    }
  }
}
```

- Units are **words**: `seconds | minutes | hours | days`. The `15m` suffix
  form belongs to the process deadline (§6), whose duration is a literal; a
  timer's interval is an **expression**, so the unit cannot ride on it.
- The interval, `key`, and `when` read the request (`request.…`), the produced
  event (`event.…`), and hydrated inputs — the dispatch scope. `self.…` is an
  error: the fold has not run when the timer is armed.
- **This is why a timer is a `dispatch` and not an `effect`.** Effects re-run
  on replay and may not read the request; arming does neither. Writing
  `effect schedule` is a parse error that says so.
- The body takes exactly `when`, `key`, and one `dispatch`. No `effect` (folds
  stay in the command's effects), no `transition` or `event` (those are process
  handlers).
- The target must be a command of the **same aggregate**. Reaching anything
  else — a peer aggregate, an intent, a port — needs a process.
- `key` is optional. With one, arming again replaces the pending clock;
  without one, every arming fires independently.
- There is **no firing-time guard and no cancel verb**. A fired timer always
  delivers, and the target command's own `error`s decide whether it still
  applies — producing a `<Command>Rejected` fact nobody observes. When
  staleness must be *handled*, model a process: `state` + `after` gives you
  guards, branches, and absorbing terminal states.


## The `actor`

`actor` is a root declaration, exactly one per domain. It defines the
authenticated caller and maps each field from a claim of the verified token:

```
term ActorId: string
term Role: string

/// The authenticated caller.
actor {
  field id: ActorId       claim "sub"
  field roles: list<Role> claim "realm_access.roles"
  field phone: Phone      claim "phone_number"  optional
}
```

- `claim` takes a string, read exactly as written; dots address nested claims.
- Fields are required unless marked `optional`. A missing required claim, or a
  claim of the wrong type, rejects the token.
- `id` is mandatory. It is the provider's subject, not an application id:
  bind it to one explicitly, as in `inject customerId = actor.id`.
- Field types are `string`, `integer`, `float` or `boolean`, terms over them,
  or lists of those.
- Rules read declared fields only, one segment deep: `actor.roles`, never
  `actor.realm_access.roles`.
- A comparison on an absent optional is false for both `==` and `!=`, so rules
  fail closed; test for absence with `actor.phone == null`. Injecting an absent
  optional writes null over the request field.

## 5. Projections and subscriptions

```nomiq
context shop.tracking

projection OrderList {
  field id: OrderId
  field status: OrderStatus

  on shop.ordering.Order.OrderPlaced {
    input row {
      oneOf self.id == event.orderId
    }

    input customer {
      from shop.crm.Customers
      oneOf self.id == event.customerId
    }

    effect insert { … }
  }

  query ListOrders {
    field status: OrderStatus

    pageOf self.status == params.status
  }
}
```

`on` names an **event** by path; the compiler finds the command whose event
matches and emits `triggerId = mirroredId(commandId)`. Paths in `on` /
`dispatch` / `from` split at the first PascalCase segment (lowercase = context
path, PascalCase = elements) and may be relative: `on OrderPlaced` works when
unique in the enclosing aggregate → context → model; ambiguity is a compile
error listing the candidates.

Every command and intent also implicitly defines a **rejection fact** named
`<Command>Rejected` (`triggerId = rejectionId(commandId)`) — no declaration
needed. A refused decision appends it to the same correlation chain, carrying
the refused request's fields, so failure handling is ordinary reaction logic:
`on core.Reservation.CreateRejected { transition failed }` in a process
state (see `_docs/semantics.md` guarantee 5).

Subscriptions are **anonymous**: `on <event> { … }` is the whole header.
The element name derives from the resolved trigger — `On` + the event name,
prefixed with the source holder unless the event already carries it
(`Order.OrderPlaced → OnOrderPlaced`, `Reservation.Created →
OnReservationCreated`). Two subscriptions on the same event in one
projection are an error: merge the blocks.

A subscription's body is `input`s and `effect`s. An input with a `from
<ctx.Element>` clause is a **sourced input**: an advisory snapshot read of another
element's read model (aggregate, projection, or process) — never locked, no
consistency guarantee; an invariant that depends on the data belongs on the
aggregate that owns it (`_docs/semantics.md` §4). Without `from`, an input
reads the enclosing element's own store, as before. Subscriptions never
communicate — cross-service writes are process dispatches (§6).

## 6. Processes — state machines, intents, dispatch

A `process` is the third context holder beside `aggregate` and `projection`:
a long-lived, correlation-keyed **state machine**. Like an aggregate it owns
per-instance state — its fields, with a mandatory `field id: <Term>`; unlike
an aggregate its triggers can also be observed facts, and it is the **only**
element that dispatches decisions to peers (any command may signal its own
integration push ports — §3). Members: `field`, `invariant`, `intent`,
`init { … }`, `state <name> { … }`, `final a b c`, and top-level `on`
handlers (any-state, facts only).

```nomiq
context core

process BoxBooking {
  field id: ReservationId

  intent BookTheBox {
    field reservationId: ReservationId
    field lockerId: LockerId
  }

  init {
    on BookTheBox {
      event BookingStarted

      transition started {
        id = request.reservationId
      }

      dispatch core.Reservation.Create {
        set reservationId = request.reservationId
        set lockerId = request.lockerId
      }
    }
  }

  state started {
    on core.Reservation.Created {
      transition reserved
    }

    on core.Reservation.CreateRejected {
      transition failed
    }

    after 15m {
      event BookingExpired

      transition expired

      dispatch core.Reservation.Cancel {
        set reservationId = self.id
      }
    }
  }

  final reserved failed expired
}
```

- **`intent <Name> [correlate by <expr>] { … }`** — a **contract only**:
  request fields, optional `correlate by request.<field>` (required iff any
  state handler handles the intent — causation covers everything else), and
  `access`. No behavior members — handling lives in init/state handlers.
  Vocabulary follows the acknowledgement: a `command`'s ack is final; an
  `intent`'s ack means accepted-for-pursuit — outcomes arrive later as
  facts. `command` inside a process and `intent` inside an aggregate are
  both compile errors.
- **Handlers** — `on <ctx.Element.Event | Intent> [correlate by <expr>]
  [when <guard>] { … }`, and, inside states only, `after <N><unit>
  [when <guard>] { … }` (units `s|m|h|d`). In `init` a bare `on <Name>`
  names a sibling intent; in states it resolves intent-first, and a name
  shadowing a reachable event demands a qualified path. Names always derive
  (`OnBookTheBox`, `OnReservationCreated` — stutter-deduped like
  subscriptions — `After15M`). Body: an optional `event <Name> { … }` (the
  handler's **milestone** fact), `input`s, `error`s (runtime effect on
  intent handlers only — a fact cannot be refused), `invariant`s, at most
  one `transition`, `dispatch`es. `effect` is gone: folds are transitions.
  An intent handler's milestone automatically carries the intent's request
  fields — the braces declare only *extra* computed fields (never redeclare
  a request field; usually the milestone is bodiless: `event BookingStarted`).
  Fact/deadline milestones carry declared fields only.
- **`transition <state> [{ field = expr … }]`** — enter a state and assign
  fields; targets must be declared states or finals. The fold is keyed by
  correlation — never a `where` script — and is a function of the entry
  state and the fact: `init` transitions insert (and **must bind `id`**);
  state transitions patch, and their rvalues may read `self.…` (counters).
  Init handlers have no `self` scope at all.
- **`dispatch <ctx.Element.Command> [{ when <expr>  set <field> = <expr> … }]`**
  — sends a decision (a command, an intent, or an integration `push` port);
  `when` gates the send, `set` clauses bind the target's request fields
  (same clause vocabulary as effects). Communication, not effects: reliably
  delivered, suppressed on replay.

Semantics (delivery, ordering, and deadline details live in
`_docs/semantics.md` — the DSL never mentions transport):

- `init` is not a state: its handlers react to **intents only**, create the
  instance, and must transition. Handling one intent in `init` **and** in a
  state is start-or-continue — the engine routes on instance existence.
- An intent nobody handles (no instance and no init handler, or the current
  state lacks a handler — final states in particular) is **rejected
  implicitly** with a lifecycle rejection fact.
- A fact selects **at most one** handler — the current state's plus the
  any-state `on` handlers; guarded variants of one trigger try first-match
  in declaration order, the unguarded variant is the else and must be last.
  Unmatched facts are ignored. Selection guards read the trigger
  (event/request) and `self` — never sourced inputs.
- `final` states are bodiless and **absorbing**: entering one disarms the
  instance's deadlines and ends its correlation.
- `after` is a durable deadline armed by the **milestone events of every
  transition into its state** — each inbound transition must declare an
  `event` (compile error otherwise). A self-transition with a milestone
  re-arms; re-arm supersedes the prior timer. Firing re-checks the instance
  is still in the state.
- `process XXX` derives a root-level enum **`XXXState`** (items = states ∪
  finals, declaration order). It is never printed but referencable
  everywhere (`BoxBookingState::reserved`); a user root declaration named
  `XXXState` is a collision error.
- The machine state is an implicit `state` column: a user field named
  `state` is an error, and `self.state` is not readable in scripts — branch
  with guarded handler variants instead.

## 7. Expressions

Infix, lowering to the stored script AST (`plugin-nomiq/docs/scripting.md` §2):

| Precedence (tight → loose) | Operators | AST |
|---|---|---|
| 1 | `* /` | `mul div` |
| 2 | `+ -` | `add sub` |
| 3 | `< > <= >=` | `lt gt le ge` |
| 4 | `== !=` and word ops `contains matches in` | `eq ne contains match in` |
| 5 | `&& / and` | `and` |
| 6 | `\|\| / or` | `or` |
| prefix | `! / not` | `not` |

Deliberately absent: `%`, unary `+`/`-` (their AST functions compile nowhere —
write `0 - x`). `x == null` / `x != null` sugar to `is-null(x)` /
`is-not-null(x)`.

### References — dotted scopes

| DSL | Lowers to | Scope |
|---|---|---|
| `self.total` | `_/total` | own state (per slot) |
| `request.orderId` | `!/orderId` | command request |
| `params.status` | `?/status` | query parameters |
| `event.placedAt` | `^/placedAt` | trigger/produced event |
| `actor.roles` | `@/roles` | fields of the `actor` (access only) |
| `elem.sku` | `elem/sku` | list element in `update … where` |
| `currentOrder.status` | `currentOrder/status` | named value (hydrated input) |
| `OrderStatus::pending` | `OrderStatus::pending` | enumeration item |

Which scopes are *legal* where remains `slot-contexts.md`'s contract — the
compiler is scope-syntax-uniform and leaves slot legality to the validator.

### Calls and special forms

Function calls use camelCase, mapped to the kebab-case catalogue:
`isNotNull(x)` → `is-not-null`, `shiftDays(d, 5)` → `shift-days`,
`mapNotNull(…)` → `map-not-null`, `containsAny(xs, "a", "b")` →
`contains-any` (the variadic any-of; infix `contains` stays binary),
`now()`, `iif(c, t, e)`, …. A **PascalCase** callee is a value constructor:
`Price(x)` is the constructor call, `price(x)` a catalogue function.

| Surface form | Lowers to |
|---|---|
| `new Order { id = event.orderId }` | `new(Order, set(id, ^/orderId))` |
| `Price(event.a + event.b)` | `new(Price, add(^/a, ^/b))` — value constructor, no `new` |
| `Period(request.from, request.to)` | `new(Period, !/from, !/to)` — multi-arg constructor |
| `new Address from event.address { city = "X" }` | `copy-from(Address, ^/address, set(city, "X"))` |
| `new list<OrderLine>` | `mk-list-of(OrderLine)` |
| `update self.lines where elem.sku == event.sku set qty = event.qty` | `find-and-update(_/lines, …, set(qty, ^/qty))` |
| `if c then a else b` | `iif(c, a, b)` |

`new` stays on the record form (`new T { … }`), the copy form (`new T from
… { … }`), and `new list<T>`. The value-call form drops it — the PascalCase
casing is the marker — though `new Price(x)` still parses; the printer
canonicalizes it to `Price(x)`. A value constructor needs at least one
argument: `Price()` is an error.

Constructor bindings target fields of the **constructed** type — `self.` is
structurally impossible inside `new { … }`, preventing the classic
`set([_/street], …)` mistake. `set`/`new`/`copy-from`/`find-and-update`/
`mk-list-of` cannot be called directly; the error message points at the
surface form.

## 8. Identity — deterministic ids

- id = first 12 chars of `base64url(SHA-256("metatype:Path/To/Element"))` —
  base64url's alphabet is exactly the nanoid alphabet of `Model.newId()`.
  Examples: `command:shop.ordering/Order/PlaceOrder`, `term:OrderId`,
  `intent:core/BoxBooking/BookTheBox`. Process handlers key on their block:
  `policy:<ctx>/<Process>/<init|any|state>/<Name>`
  (`policy:core/BoxBooking/started/OnReservationCreated`); guarded variants
  of one trigger append a `#k` ordinal in declaration order
  (`…/OnAGone#2`). The domain name is excluded from paths (renaming the
  domain re-keys nothing).
- Event ids are `mirroredId(commandId)` (string reversal), matching
  `core-model`; a handler's milestone event id is `mirroredId(policyId)`;
  `subscription.triggerId` and handler triggers are computed the same way.
  The derived state enum's id is `stateEnumId(processId)` — a case-swap
  involution beside `mirroredId`/`rejectionId`.
- Same source ⇒ byte-identical elements — compilation is reproducible, so
  every consumer derives the same elements without a lockfile.
- **Renames are re-keys** (a rename is delete + create). Collisions (hash or
  reserved ids) are compile errors, never silently perturbed.
- **The DSL is the only source of truth.** Element ids never appear in
  source — they are derived data, recomputed on every compile. Importing a
  pre-DSL model (random nanoids) re-keys every element to its deterministic
  id; that is by design, since nothing outside the generated AST depends on
  the ids (generated code is a function of names and structure).

## 9. Structural rules (matching `plugin-nomiq` store checks)

- Exactly one `domain` per unit; events are **never authored** (only the
  `event` block inside a command or process handler); aggregates,
  projections, and processes must declare a field named exactly `id`;
  unknown types, contexts, and targets are compile errors with file:line
  diagnostics.

## 10. Manual coding

Behavior the DSL cannot express is handed off to hand-written flow code with
the **`@manual` annotation** — one shape for every granularity (full
workflow: `docs/manual-flow.md`). The annotation sits on its own line
directly above the declaration it marks (doc comment first, then the
annotation, then the declaration).

- **Whole element** — bare `@manual` (the flow class name is derived);
  contract-only body (fields, `event` with *valueless* fields, optional
  condition-less `error` entries, `access`;
  inputs/invariants/effects/dispatches are validation errors):

  ```nomiq
  @manual
  command Create { … }            // hand-written {Command}Flow

  @manual
  on core.Order.Created           // hand-written {Event}Flow; body optional

  @manual
  query FindOrders {              // hand-written {Query}Flow
    field …

    allOf                         // the selection stays — it types the result
  }
  ```

- **Granular slots** — `@manual(<method>)` names the gap method; only the
  marked slot is hand-written, everything else stays declarative, and the
  generator emits an abstract `<Name>FlowBase` with one `protected abstract`
  method per slot (the hand-written `<Name>Flow extends <Name>FlowBase`
  fills the gaps). Which slot is manual follows from the declaration kind:

  ```nomiq
  @manual(createCredentials)
  field credentials: AccessCredentials    // event-field value (no '= …')

  @manual(hydrateBox)
  input box: Box                          // hydration (explicit type, no block)

  @manual(checkFraud)
  error fraudulent "flagged"              // condition (no 'when' clause)

  @manual(routeTokenState)
  effect update                           // effect body (kind is documentation, optional)
  ```

  The doc comment above a manual slot doubles as the implementation spec and
  becomes the generated abstract method's javadoc.

  Every slot on one element becomes a method on the same `<Name>FlowBase`, so
  the names must be distinct across inputs, errors, event fields and effects,
  and must avoid the generated members (`hydrate`, `handle`, `apply`,
  `dispatches`, `state`; subscriptions have no `handle`/`dispatches`).
  Validation rejects a collision — the Java would not compile.

  Prefer a granular slot over whole-manual whenever only one
  value/condition/read is inexpressible — the public contract and all other
  behavior stay in the model.

Hand-written classes live in the project's `manual/` tree
(`manual/<context>/<Holder>/<Element>Flow.java`); `/nomiq:generate` overlays
them into `src/` and guards against drift.
`@manual` is the only annotation — any other `@name` is a compile error (the
syntactic category is open for future markers). An element named `manual`
still parses via backticks.

Reserved (not implemented yet): manual query predicates; manual process
handlers (`@manual` on `on <Intent>` / `after <duration>` — intents
themselves are never manual, they are already pure contracts); transition
fold-value slots. Legacy `manualFlowCode` is deprecated: the printer warns
and drops it.

## 10a. Out of scope / open questions

- **`original`** on insert/replace/upsert: the `original <ref>` clause is
  provisional pending platform semantics.
- **`delete { oneOf … }`**: reserved until the backend supports a
  selection-based single delete.
- **Many-ops in commands**: accepted by the compiler; platform validation
  flags them until the command-flow generators handle multi-row writes.
- **List literals** for `in`: deferred until the AST supports them.
- **Bare primitive field types** (`#string` without a term): printable with a
  warning, but not compilable — introduce a term.

## 11. Tooling

- **`nomiq-fmt`** — canonical formatter, the `bin` of `@nomiq/core-dsl`:

  ```
  nomiq-fmt [dir] [--layout auto|compact|split] [--check] [--force]
  ```

  `dir` defaults to `dsl/` when present, falling back to `.nomiq/src` for
  older projects. fmt = print ∘ compile, so it
  is byte-idempotent, applies the blank-line house style of §2 (a source's own
  blank lines are discarded and re-derived), and **canonicalizes the file
  layout** (files may be
  renamed, merged, or split; `auto` keeps whichever layout the existing file
  names match best, falling back to `compact`). Regular `//` and block
  comments do not survive the element round-trip — only `///` doc comments
  do, as `desc` — so fmt refuses to rewrite files that would lose comments
  unless `--force` is passed. `--check` writes nothing and exits non-zero
  when formatting would change anything (CI mode). Sources that do not
  compile cleanly are never rewritten. The same logic is available as
  `fmt(sources, options)` from the package.

- **VS Code highlighting** — `editors/vscode-nomiq/` at the repo root: a
  declarative extension with a TextMate grammar (scope `source.nomiq`) and
  language configuration (comment toggling, auto-indent). See its README for
  install; the grammar also works in any TM-based highlighter (Shiki, …).
  An LSP (completions per slot) is deferred.
