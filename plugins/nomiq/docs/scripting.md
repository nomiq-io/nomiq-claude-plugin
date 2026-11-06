# nomiq Scripting Reference (Script AST + infix shorthand)

Behavior inside nomiq elements — input conditions, error conditions, invariant
predicates, event field values, effects, query conditions, access rules — is
expressed as **Scripts**. This document defines the Script language itself: the
stored JSON form, the infix shorthand used to read and discuss scripts, the
reference (qname) grammar, the function catalogues, and the special forms.

Which references and functions are legal **in each slot** is in
`slot-contexts.md`. Element shapes (where scripts live) are in `elements.md`.

## 1. Script AST JSON — the stored form

In element files, scripts are stored as **AST JSON**, never as infix text. You
author the AST directly. A Script is a discriminated union on `type` with
exactly three node kinds:

### 1.1 `fun` — a function call

```json
{ "type": "fun", "name": "eq", "args": [ <Script>, <Script> ] }
```

- `name` — a function name from the catalogues in §4 (lowercase, kebab-case:
  `is-not-null`, `find-and-update`).
- `args` — an array of Scripts (may be empty, e.g. `now()`).

### 1.2 `lit` — a literal

```json
{ "type": "lit", "base": "string",  "value": "Berlin" }
{ "type": "lit", "base": "number",  "value": "42" }
{ "type": "lit", "base": "boolean", "value": "true" }
{ "type": "lit", "base": "null",    "value": "null" }
```

- `base` is one of `string | number | boolean | null`.
- **`value` is ALWAYS a JSON string**, even for numbers, booleans, and null.
  `base` says how to interpret it. `{"base": "number", "value": 42}` is
  malformed; write `"value": "42"`.

### 1.3 `ref` — a reference

```json
{ "type": "ref", "qname": "!/orderTotal" }
{ "type": "ref", "qname": "_/address/city" }
{ "type": "ref", "qname": "OrderStatus::pending" }
```

`qname` follows the grammar in §3.

Nothing else is a Script node. There are no statement nodes, no variables, no
lambdas — a Script is a single expression tree.

## 2. Infix shorthand — for reading and communicating

Humans (and the nomiq UI) read scripts in an infix surface syntax. Use it in
prose and comments; **the JSON AST is what goes in element files**.

- References: `[qname]` — e.g. `[_/total]`, `[!/orderId]`, `[OrderStatus::pending]`
- Calls: `name(arg, arg, …)` — e.g. `is-not-null([currentOrder])`
- Literals: `"text"` or `'text'`, `42`, `3.14`, `true`, `false`, `null`
- Grouping: `( … )`

### Operator table (operator → AST function)

| Infix | AST `name` | Kind | Precedence (1 = binds tightest) |
|---|---|---|---|
| `\|\|`, `or` | `or` | binary | 6 |
| `&&`, `and` | `and` | binary | 5 |
| `==` (AST also accepts `=`) | `eq` | binary | 4 |
| `!=` (AST also accepts `<>`) | `ne` | binary | 4 |
| `<=` | `le` | binary | 3 |
| `>=` | `ge` | binary | 3 |
| `<` | `lt` | binary | 3 |
| `>` | `gt` | binary | 3 |
| `+` | `add` | binary | 2 |
| `-` | `sub` | binary | 2 |
| `*` | `mul` | binary | 1 |
| `/` | `div` | binary | 1 |
| `%` | `mod` | binary | 1 — **avoid: `mod` is in no compiler catalogue** |
| `!`, `not` (prefix) | `not` | unary | — |
| `+` (prefix) | `pls` | unary | — **avoid: not in any catalogue** |
| `-` (prefix) | `mns` | unary | — **avoid: not in any catalogue; write `0 - x`** |

> **None of the `avoid` rows above exist in the `.nomiq` DSL.** The DSL parser
> deliberately omits `%`, unary `+`, and unary `-` (see the precedence comment in
> `core-dsl/src/expr.ts`), and it reads `=` as assignment and does not lex `<>`
> at all — so writing any of them is a compile error, not a warning. They survive
> only in the AST-level forms this document describes, where the resulting
> `mod`/`pls`/`mns` functions are still rejected by every code generator. Never
> author them.

### Side-by-side example

Infix:

```
[_/total] >= [!/amount] && [_/status] == [OrderStatus::pending]
```

Stored AST:

```json
{
  "type": "fun", "name": "and",
  "args": [
    { "type": "fun", "name": "ge",
      "args": [ { "type": "ref", "qname": "_/total" },
                { "type": "ref", "qname": "!/amount" } ] },
    { "type": "fun", "name": "eq",
      "args": [ { "type": "ref", "qname": "_/status" },
                { "type": "ref", "qname": "OrderStatus::pending" } ] }
  ]
}
```

## 3. References — the qname grammar

A `qname` is either an **enum item** (`Enum::Item`), or a `/`-separated path
whose head selects a scope:

| Form | Meaning | Example |
|---|---|---|
| `_/field` | **Own fields** of the enclosing state — the aggregate (in command inputs/effects), the projection (in subscription inputs/effects), the parent read model (in query conditions), or the request payload (in access rules and invariants). What "own" means per slot is in `slot-contexts.md`. | `_/status` |
| `^/field` | Fields of the **event** — the event produced by the command (in effects) or the triggering event (in subscriptions). | `^/orderId` |
| `!/field` | The **command request** fields (payload of the command being handled). | `!/amount` |
| `?/field` | The **query parameters** (a query element's own fields). | `?/status` |
| `@/field` | A field of the domain's **`actor`** declaration. Only in access rules. Checked against the declared fields, one segment deep. | `@/roles` |
| `name/field` | A **named value**: a hydrated input, or `elem` inside `find-and-update`. The name is the input name, **camelCase**. Bare `name` (no `/`) references the whole loaded value. | `currentOrder/status` |
| `Enum::item` | An **enumeration item** literal. Enum name PascalCase, item camelCase; both are matched case-insensitively. | `OrderStatus::pending` |
| `TypeName` | A **bare type name** (no `/`, no `::`, not a known scope or named value) — used as the first argument of `new` / `copy-from` / `mk-list-of`. | `Address` |

### Nested paths

Paths descend through valuetype/entity fields: `_/address/city`,
`^/customer/email`. Rules of the walk (matching the code generator):

- Field name matching is **camel-insensitive** (`orderId` ≡ `OrderId`).
- Descent through a **list** field stops checking (codegen chains getters into
  list elements); further segments are not validated.
- Descending into a **primitive** or **enumeration** field is an error
  (`Field [x] is a primitive and has no field [y]`).

### Resolution order

The validator (mirroring the compiler's `evalReference`) resolves a qname in
this order:

1. Contains `::` → **enum item** (unknown enum or item → error).
2. Head is `_`, `^`, `!`, or `?` → **scope prefix**; error if that scope is not
   available in the slot; otherwise the rest of the path is walked against the
   scope's field set.
3. Head is `@` → the **`actor`**; error unless the slot allows it; the path must
   be one declared actor field.
4. Head matches a **named value** (camel-cased input/`elem` name) → the
   rest of the path is walked against the value's shape. Values with unknown or
   scalar shapes (`exist`/`count`/`seqno` inputs, subscription inputs) leave
   subpaths unchecked.
5. Head is `elem` but no enclosing `find-and-update` → error.
6. Otherwise: **bare type name**, matched PascalCase against all elements.
   Unknown names are errors — the compiler would otherwise silently emit a
   nonexistent Java symbol.

Per-slot forbidden prefixes (see `slot-contexts.md`) are checked **before**
resolution: a structurally-present but forbidden scope yields the slot's
specific message, not a generic resolution error.

## 4. Function catalogue

Each compilation site accepts only a specific set of functions; anything else
either throws at codegen or silently emits broken Java. The sets (transcribed
from the code generators):

- **DOMAIN** (a.k.a. CMD_EXPR) — general expressions: error conditions, event
  field values, effect assignment rvalues (`data` AND `patch`) and conditions,
  process transition assignments (the process fold slot — processes have no
  effects), dispatch data bindings, and timer `interval` / `key` / `when`.
- **INPUT** — command input conditions (DB filter compilation).
- **QUERY** — query conditions and subscription input conditions (= INPUT +
  `filter-all-of`).
- **EFFECT-FILTER** — effect **filter** positions only: `where` in
  update/updateMany/deleteMany. (What assignment rvalues may *reference* is
  restricted separately — see the RULE BOX in `slot-contexts.md`.)
- **ACCESS** — access rules and injections.

| Function | Arity | Meaning | DOMAIN | INPUT | QUERY | EFFECT-FILTER | ACCESS |
|---|---|---|:-:|:-:|:-:|:-:|:-:|
| `eq` | 2 | equals | x | x | x | x | x |
| `ne` | 2 | not equals | x | x | x | x | x |
| `lt` | 2 | less than | x | x | x | x | |
| `gt` | 2 | greater than | x | x | x | x | |
| `le` | 2 | less or equal | x | x | x | x | |
| `ge` | 2 | greater or equal | x | x | x | x | |
| `and` | 2 | logical and | x | x | x | x | x |
| `or` | 2 | logical or | x | x | x | x | x |
| `not` | 1 | logical not | x | x | x | x | x |
| `is-null` | 1 | value is null | x | x | x | x | x |
| `is-not-null` | 1 | value is not null | x | x | x | x | x |
| `now` | 0 | current instant — **NON-DETERMINISTIC** | x | | | | x |
| `add` | 2 | addition | x | | | | |
| `sub` | 2 | subtraction | x | | | | |
| `mul` | 2 | multiplication | x | | | | |
| `div` | 2 | division | x | | | | |
| `shift-days` | 2 | `shift-days(date, n)` — date plus n days | x | | | | |
| `mk-list-of` | 1 | `mk-list-of(Type)` — new empty list | x | | | | |
| `append` | 2 | `append(list, item)` — list with item added | x | | | | |
| `concat` | ≥1 | string/list concatenation | x | | | | |
| `join` | ≥1 | join parts into a string | x | | | | |
| `unique-id` | 0 | random UUID string — **NON-DETERMINISTIC** | x | | | | |
| `random-id` | 0 | random UUID string — **NON-DETERMINISTIC** | x | | | | |
| `composite-id` | ≥1 | composite key from parts, joined with `_` | x | x | x | | |
| `hashed-uuid` | ≥1 | UUID-shaped id hashed from parts — deterministic, replay-safe | x | x | x | | |
| `set` | 2 | field setter (only inside `new`/`copy-from`/`find-and-update`, §5) | x | | | | |
| `new` | ≥1 | construct a value (§5) | x | | | | |
| `copy-from` | ≥2 | construct by copying (§5) | x | | | | |
| `find-and-update` | 3 | update a list element (§5) | x | | | | |
| `iif` | 3 | `iif(cond, then, else)` — ternary | x | | | | |
| `contains` | 2 | string contains (filter: case-insensitive regex) | | x | x | x | x |
| `match` | 2 | `match(field, regex)` | | x | x | x | |
| `in` | 2 | `in(field, list)` — membership | | x | x | x | |
| `map-not-null` | 2 | evaluate arg1 when arg0 non-null, else null | | x | x | x | |
| `filter-all-of` | ≥1 | AND-combine sub-filters | | | x | x | |

Notes:

- **NON_DETERMINISTIC_FUNCTIONS** = `now`, `random-id`, `unique-id`. They are
  in the DOMAIN catalogue (legal in event field values) but violate effect
  purity — never use them in effects; compute into an event field instead.
- Arities marked here for `contains`/`match`/`in`/`map-not-null`/
  `filter-all-of`/`now`/`unique-id`/`random-id` follow the generators; the
  shape checker does not enforce them. All other arities are enforced (an
  arity violation is an error).
- **Filter directionality convention** (not validator-checked, but required by
  the filter compilers): in INPUT/QUERY/EFFECT-FILTER comparisons the **first
  argument must be the field being filtered** (`[_/…]` or a field of the
  filtered store), the second the compared value: `eq([_/id], [!/id])`, never
  `eq([!/id], [_/id])`.

## 5. Special forms

These are DOMAIN functions with positional, non-uniform argument semantics.
The shape checker requires their first argument to be a `ref` (a type or field
name), not a literal or nested call.

### 5.1 `new([Type], value | set(…)…)` — construct a value

`args[0]` **must be a Ref whose qname is a type name** (the last path segment
is matched PascalCase against the model). Remaining args are either a single
value (for single-field types) or `set(…)` calls.

Infix: `new([Address], set([street], [^/street]), set([city], [^/city]))`

```json
{ "type": "fun", "name": "new",
  "args": [
    { "type": "ref", "qname": "Address" },
    { "type": "fun", "name": "set",
      "args": [ { "type": "ref", "qname": "street" },
                { "type": "ref", "qname": "^/street" } ] },
    { "type": "fun", "name": "set",
      "args": [ { "type": "ref", "qname": "city" },
                { "type": "ref", "qname": "^/city" } ] }
  ] }
```

`new` is also how computed values become Term values. Arithmetic and the other
raw-producing functions (`concat`, `join`, `composite-id`, `hashed-uuid`, ids,
`now`, `shift-days`) yield raw numbers/strings/instants — when such a value is
assigned into a Term-typed field in an effect `data` binding (or any entity
construction), wrap it: `new([Total], add([^/a], [^/b]))`. The validator errs
on unwrapped raw expressions in those positions (update patches are exempt).

### 5.2 `copy-from([Type], source, set(…)…)` — construct by copying

Copy `source` (an expression of the same shape) into a new `Type`, then apply
`set` overrides. At least 2 args.

Infix: `copy-from([Address], [^/address], set([city], "Berlin"))`

```json
{ "type": "fun", "name": "copy-from",
  "args": [
    { "type": "ref", "qname": "Address" },
    { "type": "ref", "qname": "^/address" },
    { "type": "fun", "name": "set",
      "args": [ { "type": "ref", "qname": "city" },
                { "type": "lit", "base": "string", "value": "Berlin" } ] }
  ] }
```

### 5.3 `set([field], value)` — field assignment inside a constructor

Exactly 2 args; `args[0]` is a Ref naming a field **of the CONSTRUCTED type**
(`new`/`copy-from`'s type, or `find-and-update`'s list element type) — **not**
a field of the slot's scope. `set([street], …)` inside `new([Address], …)`
targets `Address.street`; writing `set([_/street], …)` is wrong (the validator
only checks the last segment, but do not rely on that). `set` never appears
outside `new`, `copy-from`, or `find-and-update`.

### 5.4 `find-and-update([listField], condition, set([field], value))` — update a list element

Exactly 3 args. `args[0]` must be a Ref to a **list field visible in the
slot's scope** (e.g. `_/lines`). Inside `args[1]` (the condition) and
`args[2]` (the update), the name **`elem`** is bound to the current list
element, shaped as the list's element type.

Infix: `find-and-update([_/lines], [elem/sku] == [^/sku], set([qty], [^/qty]))`

```json
{ "type": "fun", "name": "find-and-update",
  "args": [
    { "type": "ref", "qname": "_/lines" },
    { "type": "fun", "name": "eq",
      "args": [ { "type": "ref", "qname": "elem/sku" },
                { "type": "ref", "qname": "^/sku" } ] },
    { "type": "fun", "name": "set",
      "args": [ { "type": "ref", "qname": "qty" },
                { "type": "ref", "qname": "^/qty" } ] }
  ] }
```

`[elem/…]` anywhere else is an error.

### 5.5 `iif(cond, then, else)` — conditional expression

Exactly 3 args; an ordinary expression otherwise.

```json
{ "type": "fun", "name": "iif",
  "args": [
    { "type": "fun", "name": "ge", "args": [ { "type": "ref", "qname": "!/amount" },
                                             { "type": "lit", "base": "number", "value": "100" } ] },
    { "type": "ref", "qname": "OrderTier::gold" },
    { "type": "ref", "qname": "OrderTier::standard" }
  ] }
```

## 6. Severity model

The validator distinguishes errors (codegen provably breaks) from warnings —
but you should author to **zero errors and zero warnings**.

**Always `err`:**

- Parse failures and shape violations: wrong arity on a spec'd function, a
  non-Ref where `new`/`copy-from`/`set` expects a type/field name.
- Unresolvable references: unknown fields on a known shape, unknown
  enumerations or items, unknown bare type names, a scope prefix the slot does
  not provide. This is `err` **even in uncompiled slots** (an invariant
  referencing `_/amout` errs; one referencing `!/amount` only warns).
- A function outside a **compiled** slot's catalogue.
- Structural slot problems: a `seqno` input or query whose condition is not a
  string literal, an access injection lvalue that is not `_/<requestField>`, a
  dangling `dispatch.commandId`, an empty assignment target.
- Forbidden scope prefixes marked `err` in the slot rules (most of them — see
  `slot-contexts.md`).

**`wrn` (warning):**

- Function-catalogue and forbidden-prefix violations in **uncompiled** slots:
  invariant predicates (functions there are checked against the DOMAIN
  catalogue as warnings).
- **All effect-purity violations**: `[!/…]`, `[?/…]`, `[@/…]`, or hydrated
  input names inside effects, and non-deterministic functions (`now`,
  `random-id`, `unique-id`) in effects. These compile — **but they break event
  replay. Treat every purity warning as a MUST-FIX**: move the value onto an
  event field and reference `[^/thatField]` instead.

See `slot-contexts.md` for the full per-slot matrix.
