# Manual coding — hand-written flow classes

Some behavior is inexpressible in nomiq scripts (random token generation,
dynamic query filters, external algorithms). The `@manual` annotation hands
exactly that behavior off to hand-written Java while everything around it —
DTOs, the event contract, routing, access rules, the engine — stays generated.

The annotation sits **on its own line**, after the doc comment and directly
above the declaration it marks. It has two shapes: bare `@manual` for a whole
element (the flow class name is derived, so an argument is an error), and
`@manual(<method>)` for a granular slot, where the argument **names the gap
method** and is required.

**The principle: contracts stay in the DSL, decisions may leave.** Fields,
event shapes, error names, access rules are never manual (other services
verify against them); values, conditions, reads, and effect bodies each get an
escape sized exactly to them.

## The ladder

### Granular slots (prefer these)

Only the marked slot is hand-written. The generator emits an **abstract
`<Name>FlowBase`** whose orchestration methods are `final`, with one
`protected abstract` method per slot; you write `<Name>Flow extends
<Name>FlowBase` and override the gaps. The DSL doc comment above each slot
becomes the abstract method's javadoc — write it as the implementation spec.

The annotation argument — not the slot's own name — is the method name.

| DSL | Abstract method in the base |
|---|---|
| `@manual(createCredentials)` ⏎ `field credentials: AccessCredentials` (event field; no `= …`) | `protected abstract AccessCredentials createCredentials(C command, State state)` |
| `@manual(hydrateBox)` ⏎ `input box: Box` (explicit type — no block, no selection to derive it from) | `protected abstract CompletionStage<Box> hydrateBox(C command, ReadableDatabase rdb)` — subscriptions: `(E event, ReadableDatabase rdb, Messaging messaging)`. The input joins the `State` record and is referenceable from other scripts (`box.id`, `box == null`) like any input. |
| `@manual(checkFraud)` ⏎ `error fraudulent "msg"` (no `when` clause) | `protected abstract boolean checkFraud(C command, State state)` — name/message stay public contract |
| `@manual(routeTokens)` ⏎ `effect update` (head-only; the kind is optional documentation) | `protected abstract Effect routeTokens(E event)` for commands, `(E event, State state)` for subscriptions — composed at its declared position among the generated effects |

Only **event** fields can be `@manual` — a plain field has no computed value to
hand off. `@manual` is illegal inside a `gateway` (gateways generate no flow
class) and on process handlers and intents (an intent is already a pure
contract; handler behavior lives in `init`/`state`).

**Gap-method names share one namespace.** Every slot on an element becomes one
abstract method on the same `<Name>FlowBase`, so the names must be distinct
across all four families and must avoid the generated members — `hydrate`,
`handle`, `apply`, `dispatches`, `state` for commands; `hydrate`, `apply`,
`state` for subscriptions. Two slots claiming one name would emit two methods
differing only in return type, so the validator rejects it.

Drift is compile-checked in both directions: adding a slot creates a new
abstract method (subclass fails to compile until implemented); removing or
renaming one makes the `@Override` fail.

### Whole element

Bare `@manual` above a `command X` / `on <event>` / `query Q { … }` — the entire
flow class is yours. The declaration is **contract-only**: fields, the `event`
with *valueless* fields, optional condition-less `error` entries (client-visible
API surface), and `access`. A query keeps its selection clause in the body (it
types the result); a manual `on <event>` may omit its body entirely. Your class
implements the generated interface directly:

- `CommandFlow<C, State, E>` — nested `State` record of your own design;
  `hydrate(C, ReadableDatabase)` reads only via the database; `handle(C, State)`
  is pure and rejects by `throw new Rejection(errors)` (error name strings);
  `apply(E)` returns an `Effect`; `dispatches(C, E, State)` optional.
- `SubscriptionFlow<E, State>` — `hydrate(E, ReadableDatabase, Messaging)`,
  `apply(E, State)`. **Must be deterministic — it re-runs on event replay.**
- `QueryFlow<Q, D, R>` — `fetch(Q, ReadableDatabase)`, optional `transform`.

The class needs the exact conventional name (`<Command>Flow`,
`<TriggerEvent>Flow`, `<Query>Flow`), a **no-arg constructor**, and the right
package — the engine instantiates it by convention.

## Where the code lives

Hand-written sources go in the project's `manual/` tree (settings `manual`,
default `manual/`), shaped by **domain coordinates**, not Java packages:

```
manual/<context>/<Holder>/<Element>Flow.java     (helper classes alongside)
```

| Element | Manual file | Copied to (src/-relative) |
|---|---|---|
| command `Create` on aggregate `Reservation` in ctx `core` | `manual/core/Reservation/CreateFlow.java` | `core-reservation-app/src/main/java/<dmn>/core/reservation/CreateFlow.java` |
| granular slots on the same command | same file (extends `CreateFlowBase`) | same destination |
| subscription `on core.Reservation.TokenRedeemed` in projection `BoxDirectory` | `manual/core/BoxDirectory/TokenRedeemedFlow.java` | `core-boxdirectory-app/src/main/java/<dmn>/core/reservation/TokenRedeemedFlow.java` |
| helper `TokenGenerator.java` next to a flow | same folder | the holder's own package directory |

Note the subscription gotcha: the class is named after the **trigger event**
and its `package` statement is the **source aggregate's** package — but the
file is filed under the *declaring projection*, where you'd look for it. The
plugin owns the translation.

`src/` stays a fully disposable derived artifact: `/nomiq:generate` overlays
`manual/` files onto their destinations after extraction and tracks the copies
in the `.generated` ledger (they are overwritten on every generate — the
`manual/` file is the source of truth). Commit `dsl/` and `manual/` to git;
`src/` can always be regenerated.

## The workflow loop

1. **Mark** the element with bare `@manual`, or the slot with
   `@manual(<method>)`, and give it a precise doc comment.
2. **Check** — `nomiq_model_check` reports `missingManual` with the expected
   path; `strayManual` flags files matching no manual element.
3. **Implement** under `manual/`: correct `package`, conventional class name,
   implement the interface or extend the generated `FlowBase`. Use the
   neighboring generated flows in the same package as style reference.
4. **Compile-iterate**: copy the file to its mapped `src/` destination and run
   `mvn -pl <module> compile`; fix until green. (Re-running generate also
   refreshes the copy.)
5. **Generate** — the overlay copies and ledger-tracks the files
   (`copiedCount` in the result).

Guards on generate:

- `manual-conflict` — generation now produces a path the overlay maps: the
  element is no longer manual; delete the manual file or restore the `@manual`
  annotation.
- `manual-divergence` — the derived `src/` copy was edited in place: port the
  edits back to `manual/` (see sync-back) or revert them.

### Sync-back (IDE-assisted edits)

The copy under `src/` is the one with full IDE support. If a human edits it
there, `nomiq_manual_sync` adopts the changes back into `manual/` — only for
files mapped to an element currently marked manual, and only after showing the
user the diff and getting explicit approval. Never call it automatically.

## Rules of thumb

- Prefer the smallest hole: one `@manual(<method>)` event field beats a
  whole-manual command.
- Nondeterminism (random, now) is fine in command `handle` — the result is
  captured in the event. It is **forbidden** in projection effect code, which
  re-runs on replay.
- Manual `hydrate`/inputs read only via `ReadableDatabase` — no external I/O;
  `handle` stays pure.
- Renaming a command/event renames the expected file — `strayManual` will
  point at the orphan; move it yourself (it is your code).
