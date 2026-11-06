# nomiq Slot Contexts — what each script slot may reference

Every script-bearing slot is validated against the exact scope its compiler
provides. This file transcribes that rule table. Script syntax, the qname
grammar, function-catalogue contents, and special forms are in `scripting.md`;
element shapes are in `elements.md` (§5 command phases, §7.1 effects).

Catalogue names below refer to `scripting.md` §4: **DOMAIN**, **INPUT**,
**QUERY**, **EFFECT-FILTER**, **ACCESS**.

> **Slot names here are the stored AST's, not the DSL's.** `where`, `identity`
> and `condition` name the slots a script lands in; the DSL surface that fills
> them is the selection clause
> (`oneOf`/`allOf`/`pageOf`/`existOf`/`numberOf`/`seqno`), `identity <expr>`,
> and `when <expr>`. `where` is **not** a DSL keyword in these positions —
> writing it is a parse error. See `dsl.md` §4.

> ## RULE BOX — effect assignment rvalues: compute freely, never read state
>
> Assignment rvalues (`insert`/`replace`/`upsert` `data` and
> `update`/`updateMany` `patch`) accept the **full DOMAIN expression
> catalogue** — arithmetic, `iif`, constructors — over:
>
> - event fields — `[^/field]`
> - **subscriptions only:** hydrated inputs by name — `[updatingBill/total]`
> - enum items and literals
>
> Three hard bans (err):
>
> - **`[_/…]` (current state) never appears in an rvalue** — not even as
>   `find-and-update`'s target. Prior state enters the computation explicitly:
>   on a subscription, hydrate the current row into an **input** and reference
>   it by name (`total = add([updatingBill/total], mul([^/price],[^/quantity]))`);
>   on a command, compute the value **on an event field** and reference `[^/…]`.
> - Non-deterministic functions (`now`, `random-id`, `unique-id`) — compute
>   them in event-field values. `hashed-uuid` is NOT one of these: it derives a
>   UUID-shaped id from its parts, so a replay yields the same id and it is
>   legal in an effect.
> - **Raw-producing functions in `data` bindings must construct the field's
>   Term explicitly**: `new([Total], add([^/a],[^/b]))`, never bare
>   `add([^/a],[^/b])` — entity builders take Term values (arithmetic, `concat`,
>   `join`, `composite-id`, `hashed-uuid`, ids, `now`, `shift-days` all produce
>   raws).
>   **Patches are exempt**: `total = add(…)` is fine in `update`/`updateMany`
>   `patch` (the update DSL accepts raw values).
>
> Command effects additionally must not reference command inputs (`[input/…]`)
> in any position: aggregates replay from the event stream without re-running
> the command, so inputs don't exist at replay time. Subscription inputs DO
> re-hydrate during a projection rebuild, which is why they are legal there.
>
> Filter positions (`condition`, `where`, `identity`) are unrestricted in this
> sense — they legitimately read current state to select the target.

## 1. Command slots

Execution order: `ACCESS → HYDRATE (inputs) → VALIDATE (errors) → EVENT →
EFFECT`. Each slot sees only what exists at its phase.

### 1.1 `inputs[i].condition` — hydrate filters

| | |
|---|---|
| Catalogue | **INPUT** (compiled → violations err) |
| `_/…` | the **enclosing AGGREGATE's** fields (the store being filtered) |
| `!/…` | the command request fields |
| `name/…` | **EARLIER inputs only** (declared before this one); `one`/`many`/`page` inputs are shaped like the aggregate, `exist`/`count`/`seqno` are opaque scalars (subpaths unchecked) |
| Forbidden (err) | `?/…` — "query parameters are not available in command input conditions"; `^/…` — "the event does not exist yet"; `@/…` — "principal is only available in access rules" |

**Special rule — `seqno`:** an input with `"selection": "seqno"` takes a
**literal string** sequence prefix as its condition, not an expression:
`"condition": { "type": "lit", "base": "string", "value": "VAC-" }`. Anything
else is an err ("takes a literal sequence name, not an expression"). The prefix
is emitted verbatim and owns its separator — `"VAC-"` draws `VAC-000042`,
`"VAC"` draws `VAC000042`. It may also carry `"padding": <1..18>` — the digits
the drawn counter is zero-padded to (default 6), sizing the counter alone and
not the prefix. `padding` on any other selection is an err.

### 1.2 `errors[i].condition` — validations (command rejected when TRUE)

| | |
|---|---|
| Catalogue | **DOMAIN** (compiled → err). Note: `contains`/`match`/`in` are **NOT** in DOMAIN — they are filter-only. |
| `!/…` | the command request fields |
| `name/…` | **ALL inputs** (by camelCase name) |
| Forbidden (err) | `_/…` — "aggregate state is not available in error conditions; **hydrate an input and reference it by name**"; `^/…` — event does not exist yet; `?/…`; `@/…` |

There is no direct state access in errors: load state via an input first, then
test `[inputName/field]` or `is-not-null([inputName])`.

### 1.3 `invariants[i].predicate` — structural predicates (uncompiled)

| | |
|---|---|
| Catalogue | none (uncompiled → function issues are **wrn**, checked against DOMAIN) |
| `_/…` | the **command's own request fields** (for a command invariant) |
| Forbidden (wrn) | `!/…`, `^/…`, `?/…`, `@/…` — "Invariants are structural predicates over the element's own fields; […] is outside that scope." |

Unknown fields under `_/…` are still **err** even though the slot is
uncompiled. Command invariants are structural rules over the request payload,
not aggregate state.

### 1.4 `event.fields[i].value` — event field mappings

| | |
|---|---|
| Catalogue | **DOMAIN** (compiled → err) |
| `!/…` | the command request fields |
| `name/…` | ALL inputs |
| Allowed | `now()`, `random-id()`, `unique-id()` — this is exactly where non-deterministic values belong |
| Forbidden (err) | `_/…` — "not available when mapping event fields; hydrate an input and reference it by name"; `^/…` — "cannot reference the event being built"; `?/…`; `@/…` |

### 1.5 `effects[i]` — the fold `state' = apply(state, event)`

Scopes and purity:

| | |
|---|---|
| `_/…` | the aggregate's fields (current state) — **filter positions and lvalues only; err in assignment rvalues** (see RULE BOX) |
| `^/…` | the produced event's fields (= `command.fields ++ command.event.fields`) |
| Purity forbidden (**wrn — treat as MUST-FIX**) | `!/…`, `?/…`, `@/…`, and **every hydrated input name** — "effect handlers must be pure functions of (state, event) so events can be replayed. Map the value through an event field instead." Non-deterministic functions (`now`, `random-id`, `unique-id`) likewise warn. |

Which catalogue applies depends on the position:

| Position | Catalogue |
|---|---|
| `condition` (any effect type) | DOMAIN |
| `insert`/`replace`/`upsert`: `original`, `data` rvalues | DOMAIN (no `[_/…]`; computed values need `new([Term], …)` — RULE BOX) |
| `update`/`updateMany`: `patch` rvalues | DOMAIN (no `[_/…]` — RULE BOX; raw computed values OK) |
| `update`/`updateMany`/`deleteMany`: `where` | EFFECT-FILTER (predicates only) |
| `remove`: `identity` | DOMAIN |

(`updateMany`/`deleteMany` are what projection subscriptions use; they are
structurally legal on a command too, but validation rejects them there pending
generator support, so command effects remain instance ops in practice. Policies
carry no effects at all — a process's fold is the structural transition, §3.8.)

Lvalue rules:

- `data` lvalues name a **single field** of the aggregate (`{"type":"ref","qname":"total"}`;
  a leading `_/` is tolerated). Unknown fields err.
- `patch` lvalues may be **multi-segment paths** into the aggregate
  (`address/city`); nested record rvalues validate against the **terminal**
  segment's type.

### 1.7 `dispatches[i]` — outbound bindings (commands, policies)

Dispatches exist on **commands** (`command.dispatches`) and **policies**
(§3.9) — never on projection subscriptions, and never on intents (an intent
is a contract; its handlers dispatch). The target kind is contract-checked:
a policy may target a **command, intent, or push port**; a plain command may
target **push ports only** (its own integration adapters — a peer decision
needs a process). `pull` ports are reads and never dispatch targets;
dangling targets → err.

| | |
|---|---|
| Catalogue | **DOMAIN** for `data` rvalues and `condition` |
| `data` rvalues | compute over the request (`!/…`), the **produced event's fields** (`^/…`), and **hydrated inputs** by name; `_/…` err (the instance may not exist yet) |
| `data` lvalues | the **TARGET's request fields** (single segment), not this element's |

Dispatches are communication, not effects: reliably delivered, suppressed on
replay (`_docs/semantics.md` §3) — which is why they are a separate array,
not an effect variant.

### 1.7a `timers[i]` — deferred self-dispatch (commands)

Timers exist on **plain commands** (`command.timers`) — never on intents,
subscriptions, or policies (a process waits with a deadline, §3.9). The target
must be a command of the **same aggregate**: peers, intents, and ports all err.

| | |
|---|---|
| Catalogue | **DOMAIN** for `interval`, `key`, `condition`, and the nested dispatch |
| `interval` | integer-valued; same scope as dispatch data — `!/…`, `^/…`, hydrated inputs; `_/…` **err** (the fold has not run when the timer is armed) |
| `key` | string-valued, same scope; the timer's durable identity — re-arming supersedes |
| `condition` | gates **arming**, evaluated at decision time — not a firing-time guard |
| `timers[i].dispatch` | an ordinary dispatch node (§1.7), self-targeted |

The scope is the tell: an interval that reads `!/…` is legal here and banned
in every effect, because effects re-run at replay time and timers never do.

### 1.6 `access.rule` + `access.injections` — authorization

| | |
|---|---|
| Catalogue | **ACCESS** (compiled → err): `eq ne and or not is-null is-not-null now contains contains-any` only; `contains` takes exactly one value, `containsAny` several |
| `_/…` | the command's **request fields** |
| `@/…` | fields of the `actor` declaration |
| Forbidden (err) | `^/…` — "only [_/…] and [@/…] are"; `!/…` — "use [_/…] for request fields"; `?/…` — "use [_/…] for request fields" |

Injections overwrite request fields from the actor. Each injection:

- `lvalue` must be exactly `[_/<requestField>]` — two segments, `_` head
  (else err: "Injection targets must be [_/<request field>]").
- `rvalue` must be an **expression**, not a record; validated with the same
  ACCESS rule (typically `{"type":"ref","qname":"@/sub"}`).
- optional `condition` — same ACCESS rule.

## 2. Subscription slots

### 2.1 `inputs[i].condition`

| | |
|---|---|
| Catalogue | **INPUT + QUERY** union (compiled → err; includes `filter-all-of`) |
| `_/…` | the **enclosing PROJECTION's** fields — or, for a **sourced input** (`sourceId` set), the fields of the source read model being filtered |
| `^/…` | the **trigger event's** fields |
| `name/…` | earlier inputs — but subscription inputs hydrate from arbitrary read models, so their **shape is unknown: subpaths are unchecked** |
| Forbidden (err) | `!/…` — "request fields are not available in subscriptions: there is no command"; `?/…`; `@/…` |

The `seqno` literal rule (§1.1) applies here too. Sourced inputs are
advisory snapshot reads of another element's read model — never locked, no
consistency guarantee; an invariant that depends on the data belongs on the
aggregate that owns it.

### 2.2 `effects[i]`

Same structure as command effects (§1.5) with two differences: the many-ops
(`updateMany`/`deleteMany`) are legal here, and **assignment rvalues may
reference the subscription's hydrated inputs by name** — they re-hydrate
deterministically when the projection is rebuilt from the event stream.

| | |
|---|---|
| `_/…` | the **projection's** fields — filter positions and lvalues only; **err in assignment rvalues** (hydrate the current row into an input instead) |
| `^/…` | the **trigger event's** fields |
| `name/…` (rvalues) | inputs (shapes unknown → subpaths unchecked) |
| Purity forbidden (wrn, MUST-FIX) | `!/…`, `?/…`, `@/…` |

The canonical accumulator — "bill total += price × quantity":

```
input:  updatingBill (one)  condition: eq([_/id], [^/sessionId])
effect: update
        where: eq([_/id], [^/sessionId])
        patch: total = add([updatingBill/total], mul([^/price], [^/quantity]))
```

## 3. Process slots — intent contracts and policy handlers

A policy is one handler of the process machine. Two structural facts drive
every rule below:

- **`scope` decides `[_/…]`**: state/any handlers act on a hydrated instance
  — `_/…` is the process's fields (the entry-state snapshot); init handlers
  create — the instance does not exist yet, so `_/…` is err everywhere.
- **The trigger decides `[^/…]` vs `[!/…]`**: intent handlers see the
  request as `!/…` (there is no observed fact); fact and deadline handlers
  see the observed/arming fact as `^/…` (there is no request). Never both.

For **deadline** handlers `^/…` is the **intersection** of all arming
milestones' fields — only fields common to every transition into the state
are referencable.

### 3.1 Intent `correlate` (on the contract)

| | |
|---|---|
| Catalogue | **DOMAIN** (compiled → err) |
| `!/…` | the intent's request fields — the ONLY scope |
| Forbidden (err) | `_/…` — "the key is computed from the request before the instance is loaded"; `^/…`; `?/…`; `@/…` |

Required iff the intent has any state-scoped handler. An intent carries
nothing else scriptable: it is a contract — `fields` + `correlate` +
`access` (§1.6 applies to the access rule).

### 3.2 Event-trigger `correlate` (on the handler)

| | |
|---|---|
| Catalogue | **DOMAIN** (compiled → err) |
| `^/…` | the observed event's fields — the ONLY scope |
| Forbidden (err) | `_/…` — the key is computed from the event before the instance loads; `!/…`; `?/…`; `@/…` |

### 3.3 `guard` — handler selection

| | |
|---|---|
| Catalogue | **DOMAIN** (compiled → err; purity: no `now`/`random-id`/`unique-id`) |
| `_/…` | the process's fields (entry state) — **state/any only; err in init** |
| `^/…` or `!/…` | the trigger |
| Forbidden (err) | **input names — NEVER**: selection must be replay-deterministic, guards never see hydrated inputs; `?/…`; `@/…` |

Same-trigger variants are tried in declaration order, first match fires; the
unguarded variant is the else and must be last.

### 3.4 `inputs[i].condition`

As subscription inputs (§2.1): `_/…` is the process's fields — or the
`sourceId` store being filtered — the trigger is `^/…` (or `!/…` in intent
handlers), plus earlier inputs by name. Advisory snapshot reads; the `seqno`
literal rule applies.

### 3.5 `errors[i].condition`

| | |
|---|---|
| Catalogue | **DOMAIN** (compiled → err) |
| `_/…` | the process's fields — **state/any only; err in init** ("the instance does not exist yet") |
| `^/…` or `!/…` | the trigger |
| `name/…` | ALL inputs |
| Forbidden (err) | `?/…`; `@/…` |

TRUE rejects the trigger with a correlated rejection fact (guarantee 5). A
request with **no handler at all** at its lifecycle position never reaches
errors — that is the implicit lifecycle rejection (`errors: ["lifecycle"]`).

### 3.6 `invariants[i].predicate`

As §1.3, with `_/…` = the **trigger's fields** for init handlers (no
instance yet) and the **process's fields** otherwise.

### 3.7 Milestone `event.fields[i].value`

| | |
|---|---|
| Catalogue | **DOMAIN** (compiled → err) |
| `^/…` or `!/…` | the trigger |
| `name/…` | ALL inputs |
| Forbidden (err) | `_/…` — "hydrate an input and reference it by name" (no self here, even in state handlers); `?/…`; `@/…` |

Intent-handler milestones carry the intent's request fields automatically;
`event.fields` declare only the extras.

### 3.8 `transition.data` — the fold

| | |
|---|---|
| Catalogue | **DOMAIN** (compiled → err; purity: no non-deterministic functions) |
| Rvalues | the trigger (`^/…` or `!/…`) + the handler's **own milestone fields** (`^/…`) + `_/…` **entry state, state/any only** (counters: `reminders = add([_/reminders], 1)`) — **never inputs** (advisory reads are not replayable) |
| Lvalues | the **process's fields** (single segment); `state` is **not bindable** — the engine writes it from `to` |
| Init handlers | `_/…` err (no instance); `data` **must bind `id`** — it keys the created instance |

The write is keyed — insert for init, patch for state/any — so there is no
`where` slot on a process, ever.

### 3.9 `dispatches[i]` — data + condition

| | |
|---|---|
| Catalogue | **DOMAIN** for `data` rvalues and `condition` |
| Rvalues / condition | the trigger (`^/…` or `!/…`) + hydrated inputs by name + `_/…` **state/any only** (the "only act if still pending" guard is the point); err in init ("the instance may not exist yet") |
| `data` lvalues | the **TARGET's request fields** (single segment) |
| Targets | commands, intents, push ports — a process may send peer decisions |

## 4. Query slots

### 4.1 `condition`

| | |
|---|---|
| Catalogue | **QUERY** (compiled → err) |
| `_/…` | the **parent read model's** fields (projection or aggregate the query lives under) |
| `?/…` | the query's own parameter fields (`query.fields`) |
| Forbidden (err) | `!/…` — "use [?/…] for query parameters"; `^/…`; `@/…` |

Typical: `eq([_/orderId], [?/orderId])` — parent field left, parameter right.

The `seqno` literal rule (§1.1) applies here too: a query with
`"selection": "seqno"` carries the prefix as its condition and answers with the
drawn counter string, so nothing in the table above is in scope for it.

### 4.2 `invariants[i].predicate`

As §1.3, with `_/…` = the query's **parameter fields**.

### 4.3 `access`

As §1.6, with `_/…` = the query's **parameter fields**.

## 5. Aggregate / entity / process `invariants[i].predicate`

As §1.3, with `_/…` = the element's **own fields**. Uncompiled: foreign
prefixes warn, unknown own fields err. (A process field named `state` does
not exist — the machine-state column is implicit and reserved.)

## 6. Push / pull `access`

As §1.6, with `_/…` = the push/pull element's **own fields**.

## 7. Cheat table

| Slot | `_/` = | `^/` | `!/` | `?/` | `@/` | named values | Catalogue |
|---|---|:-:|:-:|:-:|:-:|---|---|
| command input condition | aggregate | — | yes | — | — | earlier inputs | INPUT |
| command error condition | — | — | yes | — | — | all inputs | DOMAIN |
| command invariant | request | — | — | — | — | — | (wrn/DOMAIN) |
| event field value | — | — | yes | — | — | all inputs | DOMAIN |
| command effect (rvalues) | **err in rvalue** | produced event | purity! | purity! | purity! | purity! | DOMAIN |
| command effect (where/filters) | aggregate | produced event | purity! | purity! | purity! | purity! | EFFECT-FILTER |
| command/push/pull/query access | request/params | — | — | — | yes | — | ACCESS |
| subscription input condition | projection (or `sourceId` store) | trigger event | — | — | — | earlier inputs | INPUT+QUERY |
| subscription effect (rvalues) | **err in rvalue** | trigger event | purity! | purity! | purity! | **inputs OK** | DOMAIN |
| subscription effect (where/filters) | projection | trigger event | purity! | purity! | purity! | — | EFFECT-FILTER |
| command dispatch data/condition | **err** | produced event | yes | — | — | inputs | DOMAIN |
| command timer interval/key/condition | **err** | produced event | yes | — | — | inputs | DOMAIN |
| intent `correlate` | **err** | — | yes | — | — | — | DOMAIN |
| policy event `correlate` | **err** | observed event | — | — | — | — | DOMAIN |
| policy guard | entry state × | trigger fact | intent request | — | — | **NEVER** | DOMAIN |
| policy input condition | process (or `sourceId` store) | trigger fact | intent request | — | — | earlier inputs | INPUT+QUERY |
| policy error condition | entry state × | trigger fact | intent request | — | — | all inputs | DOMAIN |
| policy milestone field value | **err** | trigger fact | intent request | — | — | all inputs | DOMAIN |
| transition data rvalues | entry state × | trigger + milestone | intent request | — | — | **NEVER** | DOMAIN |
| policy dispatch data/condition | entry state × | trigger fact | intent request | — | — | inputs | DOMAIN |
| query condition | parent read model | — | — | yes | — | — | QUERY |
| aggregate/entity/process invariant | own fields | — | — | — | — | — | (wrn/DOMAIN) |

"purity!" = structurally present but referencing it is a purity violation
(warning that MUST be fixed by routing the value through an event field).
"entry state ×" = the process's fields for **state/any** handlers, err in
**init** (no instance yet). Policy rows: `^/…` and `!/…` are mutually
exclusive — the handler has either an observed/arming fact or a request.

## 8. Authoring order

Author a command's behavior in execution order — each phase may only reference
the previous phases' outputs:

1. **Inputs** — they see only the aggregate store, the request, and earlier
   inputs; nothing else exists yet.
2. **Errors** — they see the request and the now-hydrated inputs (state is
   reachable ONLY through inputs).
3. **Event field values** — they see the request and inputs, and are the ONLY
   place for `now()` / `random-id()` / `unique-id()` and computed derivations
   destined for state.
4. **Effects** — they see only `(state, event)`; everything an effect needs
   must already be on the event (or in current state, for filters).

If while writing an effect you need a request value, an input value, or a
computed value: **stop, add an event field with the right `value` script, then
bind `[^/thatField]` in the effect.** Same discipline for subscriptions:
inputs → effects, with effects again restricted to (projection state,
trigger event, hydrated inputs).

A process handler runs the same discipline in its order: guard → inputs →
errors → milestone → transition → dispatches. The two extra rules: the
guard comes FIRST and therefore sees no inputs (ever), and the transition
may additionally read the entry state (`[_/…]`, state/any handlers only) —
but never inputs.
