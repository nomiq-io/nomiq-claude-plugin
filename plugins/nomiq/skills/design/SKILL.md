---
name: design
description: Design or evolve a nomiq domain model conversationally — terms, aggregates, commands, events, projections, queries. Produces high-level structure only (no scripting). Use when the user describes their domain or business needs, wants to add features to the model, or says "design".
---

# Design the domain model

You are running a conversational DDD design session. The outcome is a set of
`.nomiq` DSL files in the DSL directory (settings `dsl` in
`.nomiq/settings.json`, default `dsl/`) — but **only after the user approves
the design**.

## Before anything

1. Call `nomiq_doc` for `design-method` (the method and structural rules) and
   for `dsl` (the DSL syntax you will author). Call it for `elements` when you
   need the underlying element semantics.
2. If `.nomiq/settings.json` does not exist, run the `init` flow first — follow
   the nomiq init skill.
3. Read every existing `.nomiq` file in the DSL directory — you are evolving
   that model.

## Phase A — propose (no files, no scripts)

1. Interview the user about their domain and business needs. Ask focused
   questions and build the picture incrementally.
2. Propose a **high-level design only**, following design-method.md: vocabulary
   (terms, enumerations, valuetypes), contexts, aggregates with fields, commands
   with request fields and event name/fields, projections with read-model fields,
   subscriptions (which command's event feeds them), queries with selection and
   parameter fields, processes with their `id` field, intent contracts (fields +
   which request field they correlate by), named states, and `final` terminal
   outcomes. Include the `actor`: ask who calls the system and which token claims
   carry their roles or tenant, and map only what authorization will need.
3. Present the proposal as readable markdown — a tree plus field tables, in the
   format shown at the end of design-method.md. **Do not write any files. Do not
   author any scripts** (inputs, errors, invariants, effects, conditions belong to
   the implement skill).
4. Iterate on feedback. Repeat until the user **explicitly approves**
   ("yes", "looks good", "go ahead"). Silence or a follow-up question is not
   approval.

## Phase B — materialize (after approval only)

1. Author the DSL files in the DSL directory, one file per bounded context
   plus the root file:

   ```
   dsl/
     domain.nomiq          # domain Shop + actor + vocabulary (terms, enums, valuetypes, entities)
     shop.ordering.nomiq   # context shop.ordering — aggregates + their commands/queries
     shop.tracking.nomiq   # context shop.tracking — projections + subscriptions/queries
     shop.fulfillment.nomiq # context shop.fulfillment — processes + their intents/states
   ```

   The canonical split layout is one directory per context with one
   whole-holder file each: `dsl/domain.nomiq` (root), then
   `dsl/shop.ordering/Order.nomiq` (the aggregate with its commands and
   queries), `dsl/shop.tracking/OrderList.nomiq`, … Files repeating a context
   header merge,
   and a holder re-opened across files (same kind + name) merges its members
   (partial declarations), so splitting a huge holder further stays legal.

   Follow `docs/dsl.md` exactly: keyword-block declarations, `context <path>`
   headers (lowercase dotted), `field name: Type`, `command X { … event Y }`,
   `on ctx.Agg.EventName { … }` (anonymous subscription),
   `query Q { field … <selection> }` — a query's selection clause
   (`oneOf`/`allOf`/`pageOf`/`existOf`/`numberOf`/`seqno`) is **mandatory** even
   at design time, since it types the result; the predicate after it is what
   waits for implement, except `seqno`, which takes a literal prefix and an
   optional width (`seqno "ORD-" 4`) and so is already complete,
   `process P { intent X correlate by request.<field> { field … }
   init { } state started { } final done }` — intent contracts (`correlate by`
   sits on the header, before the block) plus empty `init`/`state` blocks and
   the `final` list; handlers come at implement time.
   Doc comments `///` for descriptions — the run sits directly above what it
   documents, with a **blank line above the comment**. Follow the formatting rules in `docs/dsl.md` §2 as you write: one
   blank line between member groups, around any documented member, and around
   any member that opens a `{ … }` block; plain one-liners of the same kind
   stay packed. That separation keeps a documented model readable, and
   it is exactly what `nomiq-fmt` emits.
   Leave behavior to the implement skill: inputs, errors, effects, handlers,
   transitions, dispatch bindings and selection predicates. Trivially obvious
   event blocks with no extra fields belong here.

   **Ids are derived**: the compiler derives element ids from names, and the DSL
   is the only source of truth. Renaming an element re-keys it, which is normal;
   tell the user before renaming anything in an approved design.

2. When editing existing files, change only what the approved design changed.
3. Run `nomiq_model_check` — it compiles all `.nomiq` sources in the DSL
   directory and reports DSL diagnostics with file/line plus structural
   checks. Fix every reported error.
4. Show the user the final model tree (from `nomiq_model_check`) and suggest the
   next step: the nomiq implement skill, to add behavior.
