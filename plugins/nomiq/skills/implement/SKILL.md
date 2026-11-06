---
name: implement
description: Add behavior (scripts) to a nomiq domain model — command inputs, validations, invariants, event-field values, effects, query conditions, process state handlers and dispatches. Use after the high-level design exists, when the user wants to implement commands/subscriptions/queries/processes or says "implement".
---

# Implement model behavior

You are filling in the behavioral scripts of an approved high-level design. You
edit the `.nomiq` DSL files in the DSL directory (settings `dsl` in
`.nomiq/settings.json`, default `dsl/`) and iterate against the platform
validator until clean. (Legacy JSON-store projects must migrate first via
`nomiq_dsl_import` — see the design skill.)

## Before anything

1. Call `nomiq_doc` for `dsl` and study §2 and §4–§7 — §2 for the lexical
   and **formatting** rules every line you add must follow, §4–§7 for the
   surface syntax you will write (inputs, errors, effects, access, processes
   with intent contracts and init/state handlers, and the infix expression
   grammar with dotted scopes `self./request./params./event./actor.`).
2. Call `nomiq_doc` for `scripting` (function catalogue per slot) and for
   `slot-contexts` (which references and
   functions are legal in each slot — this is the law; cite it when explaining
   choices). The DSL lowers 1:1 onto those rules: `self.x` is `[_/x]`,
   `request.x` is `[!/x]`, and so on.
3. Run `nomiq_model_check`. Fix structural and DSL compile errors before
   implementing behavior. The check also runs a local semantic validation
   (`semanticIssues`) — fixing those before going remote saves round-trips.
4. Confirm the API key works (`nomiq_auth_status`); if not, route the user
   through the nomiq login skill first — the loop below needs the remote validator.

## Scope

With the user, pick the target: one element, one aggregate/projection slice, or
"implement everything". For each command work in this order (each phase sees the
previous phase's outputs — see slot-contexts.md):

1. **inputs** — hydrate the state the command needs. An input is a block whose
   clauses each take their own line:

   ```nomiq
   input current {
     oneOf self.id == request.orderId
   }
   ```

2. **errors** — business validations over the request and hydrated inputs:
   `error notFound "Order not found." when current == null`
3. **invariants** — structural predicates over the request payload (only when
   they add value).
4. **event field values** — compute anything the effects will need into event
   fields: `field placedAt: Timestamp = now()`
5. **effects** — fold the event into state. An effect is a block of clause
   lines in canonical order — `when` guard, then `original`, then the selection
   (`oneOf`/`allOf`) or `identity`, then the `set` bindings:

   ```nomiq
   effect update {
     when event.total > 0                 // optional guard — must come first
     oneOf self.id == event.orderId       // which row(s); 'identity <expr>' for delete
     set status = OrderStatus::shipped
   }
   ```

   The guard goes at the top: a `when` written after the `set` lines reads as
   that *binding's* condition, and the parser rejects it to keep the two apart.
   Assignment rvalues may compute (full expression catalogue) over event
   fields — and, on subscriptions, over hydrated inputs by name — but **never
   read `self.…` state**: on a subscription hydrate the current row into an
   input; on a command compute the value on an event field. Computed values
   assigned into Term-typed fields must be wrapped in a constructor:
   `set total = Total(event.a + event.b)`. See the RULE BOX in
   slot-contexts.md.
6. **access** — only when the user has authorization requirements. Rules read
   declared `actor` fields only (`actor.roles`), so add any field a rule needs to
   the `actor` declaration first, with its `claim`. Use `containsAny` for several
   values: `contains` takes exactly one. `actor.id` is the provider's subject, so
   ask before treating it as an application id such as a customer id.

Separate those phases with a **blank line** as you write them (dsl.md §2): one
blank line between member groups, and around any member that opens a `{ … }`
block or carries a `///` run. Clause lines *inside* a block stay packed. This
is what `nomiq-fmt` emits, so writing it this way keeps your edit and the
canonical form identical — and keeps later diffs to the lines you meant.

For subscriptions: inputs → effects (foreign state comes in via a sourced
input — `input x { from ctx.Element  oneOf … }`, one clause per line). For
queries: the selection clause is **mandatory** (it types the result) and its
predicate is optional — `pageOf` alone relies on implicit key matching when
parameter names equal read-model fields. A `seqno` query takes a literal prefix
and an optional width (`seqno "ORD-" 4`) in place of a predicate and answers
with the drawn counter string; see `docs/dsl.md` §4 for how the width and the
prefix divide. For processes: intents stay contracts
(fields +
`correlate by request.<field>` + access) — never put behavior on them.
Implementing a process = writing its `init` and `state` handlers, each in
this order: inputs → errors → the optional milestone `event` (bodiless
unless you need extra *computed* fields — an intent handler's milestone
carries the intent's request fields automatically; never redeclare them) →
`transition <state> { … }` (the process fold — bare `field = expr`
assignments over event/request fields; processes have no effects) →
`dispatch <ctx.Element.Command> { when <expr>  set <field> = <expr> … }`
(guard first, then one `set` per bound request field). Branch with guarded
variants of one trigger
(`on X when …` — first match in declaration order, unguarded variant last);
time-box a state with `after <N><unit>` deadline handlers inside it.

An aggregate command may schedule **one of its own commands** with
`after <interval> <unit> { [when <expr>] [key <expr>] dispatch <Command> { set … } }`
— units are words (`seconds|minutes|hours|days`) because the interval is an
expression over the request / produced event / hydrated inputs (`self.…` is an
error: the fold has not run). Add `key` when re-arming should supersede the
pending clock. There is no firing-time guard and no cancel: if staleness must
be handled rather than refused by the target's own errors, model a process.
Writing `effect schedule` is a parse error — arming is communication, not a
fold.

## The validate-fix loop (bounded)

After each element (or small batch), run `nomiq_validate` and loop:

```
rounds = 0; history = []            # list of errFingerprints sets from prior rounds
repeat:
  result = nomiq_validate()
  errs = result.errFingerprints (sorted)
  if errs is empty  → done: report remaining wrn/inf and stop looping
  rounds += 1
  if rounds > 5                     → STALL
  if errs == last entry of history  → STALL   (no progress)
  if errs equals ANY earlier entry  → STALL   (oscillation)
  append errs to history
  fix every err, then repeat
```

- Each issue carries `file`/`line`/`col` — the DSL source position where the
  fix belongs. Go there directly. `element` (`metatype:Name`) and `path` (the
  member path, like `effects[0].data[1].rvalue`) pinpoint which slot of the
  element the issue is about.
- Treat purity **warnings** on effects as must-fix too (the generator cannot
  compile what the validator merely warns about) — but they do not gate the loop.
- Issues marked "(on the derived event; fix it on the command)" are fixed in the
  command's `event … { field … = … }` block.

**On STALL:** stop editing. Present the remaining errors grouped by file, with
what you tried for each, and ask the user to choose: (a) allow revising the
high-level design, (b) give guidance, (c) leave as is.

## Autonomy boundary

- Freely: author/repair scripts, add/adjust event fields' values, fix field
  types to existing vocabulary, add a missing term/enumeration when a script
  clearly needs it (announce it).
- **Only with explicit user approval:** adding/removing/renaming elements or
  fields of the approved design, changing an event's field set visible in the
  approved design, changing a query's selection, marking an element or slot
  `@manual`.
- Never author standalone event elements.

## Manual coding (escape hatch)

When behavior is genuinely inexpressible in scripts (random tokens, dynamic
query filters, external algorithms), hand off to hand-written Java via the
`@manual` annotation — read `docs/manual-flow.md` first. The annotation sits on
its own line, after the doc comment and directly above the declaration it marks.
Rules of thumb:

- **Prefer the smallest hole.** One granular slot beats marking the whole
  element `@manual` — the contract and the rest of the behavior stay in the
  model, and the generated `<Name>FlowBase` enforces the seam at compile time.
  Each granular form names its gap method:

  ```nomiq
  @manual(createCredentials)
  field credentials: AccessCredentials   // event-field value (no '= …')
  @manual(hydrateBox)
  input box: Box                         // hydration (explicit type, no block)
  @manual(checkFraud)
  error fraudulent "flagged"             // condition (no 'when' clause)
  @manual(routeTokens)
  effect update                          // effect body (kind optional, documentation only)
  ```

  Bare `@manual` (no argument) marks a whole `command` / `query` / `on <event>`;
  the flow class name is derived, so an argument there is an error.
- **Gap-method names share one namespace.** Every `@manual(<method>)` on an
  element becomes one abstract method on the same `<Name>FlowBase`, so the names
  must be distinct across inputs, errors, event fields and effects, and must
  avoid the generated members (`hydrate`, `handle`, `apply`, `dispatches`,
  `state`). The validator rejects collisions.
- Write the doc comment above the manual slot/element carefully — it is the
  implementation spec and becomes the generated abstract method's javadoc.
- After marking, `nomiq_model_check` reports the expected file under `manual/`
  (`missingManual`). Create it there (correct `package` — for subscriptions it
  is the *source aggregate's* package), implement, copy to the mapped `src/`
  path, and compile the module (`mvn -pl <module> compile`) until green; use
  neighboring generated flows in the same package as style reference. Re-run
  the nomiq generate skill to confirm the overlay.

When everything is clean, show the per-severity counts and suggest the nomiq
generate skill.
