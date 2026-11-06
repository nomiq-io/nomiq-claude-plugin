# nomiq Design Method

How to run a conversational domain-design session that produces a **high-level nomiq domain model**. This document covers structure and vocabulary only. Inputs, errors, effects, invariants, conditions, and event-field value expressions belong to the implement pass (see `scripting.md` and `slot-contexts.md`).

## 1. Conceptual model

nomiq generates event-sourced CQRS microservices from a domain model. The runtime flow is:

```
Command → Aggregate → Event → Projection → Query
```

- **Aggregates** are the write side: consistency boundaries that receive commands, validate them, and emit events. `Command → Validate → Event → Effect`.
- **Events** are immutable facts, the source of truth. Every command emits exactly one event on success. Events are *derived* from commands — you name them inside the command.
- **Projections** are the read side: denormalized read models that subscribe to events to stay current, and expose queries. `Event → Update View → Query`.
- **Queries** are read operations, hosted on aggregates or projections.
- **Processes** are long-lived workflow coordinators (sagas): correlation-keyed **state machines** that accept intents and react to events and deadlines through handlers living in `init` and `state` blocks, dispatching decisions to other elements. `Intent/Event → Handler → Transition + Dispatch`. The only element that communicates across services.
- **Vocabulary** (terms, valuetypes, entities, enumerations) is the ubiquitous language of the domain, shared flat under the domain root and referenced by name everywhere a field needs a type.

## 2. Design method — how to run the conversation

### Vocabulary first

Identify the domain's ubiquitous language before proposing structures. Ask what the business calls things; capture identities, money-like quantities, closed status sets, and composite values as vocabulary items. Propose vocabulary *before or alongside* the structures that use it, so every field points at an existing type.

### Interview, then propose incrementally

- Be a critical DDD advisor. Question vague requirements; surface trade-offs. Don't agree just to be agreeable.
- Ask about the business need: Who acts on the system? What decisions change state? What do users need to *see*? The answers map to commands, aggregates, and projections respectively.
- Propose in small, reviewable increments — one bounded context or one aggregate-plus-projection slice per turn. Many turns will have no model changes at all; pure conversation is valuable.
- When the user asks to extend something that already exists, add fields to it and present only the change.
- Never rename, retype, or delete existing fields as part of routine proposals; only add. Restructure only when the user explicitly asks for it.

### Think in bounded contexts

Group aggregates and projections into bounded contexts named after business sub-domains (Ordering, Fulfillment, Banking). A context is a language boundary: the same word may mean different things in different contexts — that's a signal to split.

### Present first, persist after approval

During the design session:

- **Present every proposal as readable markdown** — a tree of the proposed structure plus field tables (format in §8). The user reviews and refines in conversation.
- **Do NOT write any model files** while designing. Files are written only after the user approves the design and you move to the write-out step.
- **Keep the design pass free of scripting.** It produces names, shapes, descriptions, and wiring (command→event, projection→subscription). Inputs, errors, effects, invariants, conditions and value expressions belong to the implement pass.

## 3. Structural rules (strict)

### Tree shape

```
domain (root)
├── context            — bounded context (one or more)
│   ├── aggregate
│   │   ├── command    — leaf
│   │   └── query      — leaf
│   ├── projection
│   │   ├── query      — leaf
│   │   └── subscription — leaf
│   └── process
│       ├── intent     — leaf (contract)
│       └── state machine: init block, state blocks, final list
├── term               ┐
├── valuetype          │ vocabulary: FLAT under the
├── entity             │ domain root
└── enumeration        ┘
```

- A context contains only aggregates, projections, and processes.
- An aggregate contains only commands and queries.
- A projection contains only queries and subscriptions.
- A process contains intent contracts plus its state machine: one `init` block, `state` blocks, and a `final` list of terminal states. Handlers (`on …` / `after …`) live inside init/state blocks.
- Commands, queries, subscriptions, intents, and handlers are leaves — no children.
- Vocabulary elements have no parent; they live flat under the domain root.

### Element kinds

| Kind | Role | Notes |
|---|---|---|
| context | Bounded context, top-level grouping | Business sub-domain |
| aggregate | Consistency boundary owning commands and queries | Write side |
| projection | Read model exposing queries | Read side; fed by subscriptions |
| process | Long-lived workflow coordinator (saga), a state machine | The only element that sends decisions (dispatches) to others |
| command | State-changing decision on an aggregate | Emits exactly one event; ack is final (committed or rejected) |
| query | Read operation on an aggregate or projection | Fields are its parameters |
| subscription | A projection's reaction to one event | Names the event it reacts to |
| intent | A business request on a process — a **contract** only (fields + correlation) | Ack means accepted-for-pursuit; outcomes arrive later as facts; all handling lives in init/state blocks |
| handler | A reaction inside `init` or a `state`: "whenever X, then Y" | Triggered by an intent, an event, or a state-scoped deadline (`after` N s/m/h/d) |
| actor | The authenticated caller, one per domain, mapped from token claims | Holds only what authorization and injection need |
| term | A primitive wrapped with semantic meaning | primitive: string, integer, float, date, boolean, money |
| enumeration | Closed set of named values | Items are camelCase |
| valuetype | Immutable record, no identity | Address, DateRange |
| entity | Record with identity, used as a component inside aggregates/valuetypes | OrderLine inside Order |

### Never design at this level

- Inputs, errors, effects, invariants, dispatch bindings — implement-pass concerns.
- Handler bodies — the `on …` / `after …` handlers inside init/state blocks, with their transitions and dispatches, are implement-pass concerns. At design time init/state blocks stay empty; you design the state *names* and the `final` list.
- Events as standalone elements — they are derived from commands; reference them by name only.
- Integrations, push, pull.

### Naming conventions

**PascalCase** for all element names; **camelCase** for all field names.

| Kind | Form | Examples |
|---|---|---|
| Context | noun phrase, business sub-domain | Ordering, Fulfillment, Banking |
| Aggregate | noun | Order, Cart, Account |
| Projection | noun | OrderHistory, OutstandingBalances |
| Process | noun for the workflow | BoxBooking, OrderFulfillment |
| **Command** | **imperative verb**, verb-first | Withdraw, OpenAccount, PlaceOrder |
| **Event** | **past participle** of the command | Withdrawn, AccountOpened, OrderPlaced |
| **Query** | **noun phrase** describing what's returned | TransactionList, AccountSummary, PendingOrders |
| Subscription | On + event name | OnOrderPlaced, OnOrderShipped |
| Intent | imperative verb, like commands | BookTheBox, FulfillOrder |
| Handler | derived from the trigger | OnReservationCreated, OnBookTheBox, After15M |
| State | camelCase, few and meaningful | started, reserved, expired |
| Term | noun, often ending in Id for identities | OrderId, EmailAddress, AccountNumber |
| Enumeration | noun for the closed set | OrderStatus, PaymentMethod |
| Valuetype | noun for the value record | Address, Money, GpsLocation |
| Entity | noun for the component | OrderLine, AddressBook |
| Field | camelCase | id, customerId, orderTotal |

Never collide with existing element names in the model.

### Command → event name derivation

Every command's event name is the past participle of the command:

| Command | Event |
|---|---|
| Withdraw | Withdrawn |
| Deposit | Deposited |
| OpenAccount | AccountOpened |
| CloseAccount | AccountClosed |
| PlaceOrder | OrderPlaced |
| ShipOrder | OrderShipped |
| CancelReservation | ReservationCancelled |
| RegisterUser | UserRegistered |
| ApprovePayment | PaymentApproved |
| RejectInvoice | InvoiceRejected |

Pattern:

- Single-word verb (`Withdraw`) → past-participle the verb (`Withdrawn`).
- Verb-noun compound (`PlaceOrder`) → rearrange to **noun + past participle** (`OrderPlaced`).
- Use irregular past participles correctly (write → written, give → given, take → taken).

The event automatically inherits all of the command's fields. List extra event-only fields only for server-generated values (assigned ids, timestamps).

### When to model a process

Reach for a process when the requirement is a workflow that spans consistency boundaries:

- **Multi-aggregate workflows**: a business outcome that needs decisions on several aggregates (book a box = create a reservation + open the locker + start billing).
- **Fan-out**: one decision that must reach many instances — a process accepts the intent and dispatches per instance; its ack means "accepted".
- **Timeouts that branch**: "if X hasn't happened within N minutes, then Y — unless Z" — a deadline handler (`after 15m`) inside the state it time-boxes. A timeout that only ever does *one unconditional thing to the aggregate itself* is a **timer** (see below).
- **Cross-service failure handling**: rejections are facts, so a downstream refusal is just another event a state handler reacts to — compensation is ordinary handler logic.

A process **is a state machine** — design it as one:

- **Name the states** — few and meaningful, each a phase the workflow can rest in (started, reserved, expired). The derived `{Process}State` enumeration (states + finals) is referencable everywhere, so the names are part of the ubiquitous language.
- **One intent per business request.** An intent is a contract — fields plus which request field it correlates by; how it is handled belongs to init/state blocks at implement time. An intent handled in no block for the instance's current state is rejected implicitly (lifecycle).
- **Terminal outcomes go in `final`** — the absorbing end states (`final reserved failed expired`). Once there, the instance is done.
- **Deadlines live in the state they time-box** — "no confirmation within 15 minutes" is an `after 15m` in `started`.

### Timer or process?

An aggregate can schedule **one of its own commands** with `after <interval> <unit> { dispatch <Command> { … } }`. The interval is computed (`after request.graceMinutes minutes`), which a process deadline cannot express — its duration is a literal.

Choose a **timer** when the wait is self-contained: one aggregate, one unconditional follow-up, staleness that can simply be refused ("re-check this box in N minutes", "expire this hold", "retry the charge once"). Give it a `key` when re-arming should replace the pending clock rather than add a second one.

Choose a **process** the moment the wait needs anything a timer deliberately lacks: a guard at firing time, a branch between outcomes, a terminal state that absorbs late facts, or a second element to talk to. A timer has no firing guard and no cancel — it delivers and the command's own errors judge it — so if "did it still apply?" is a business question rather than an error, you want a state machine.

Cross-service invariants belong to an aggregate — wanting one signals a missing aggregate (e.g. a reservation aggregate for uniqueness). Every process needs an `id` correlation-key field; intents follow command naming. Handlers may name a **milestone event** (BookingStarted, BookingExpired) marking the workflow's lifecycle; handler names are derived from the trigger (OnBookTheBox, OnReservationCreated, After15M). The same mirroring rule as commands applies: an intent handler's milestone automatically inherits the intent's request fields; declare extras only for computed values, and prefer a bodiless `event BookingStarted`.

### Identity rules

- **Every aggregate, projection, and process needs an identity field**: a field named `id` whose type is a term named `{Name}Id` (aggregate Order → `id: OrderId`; for a process it is the correlation key). Propose the `{Name}Id` term alongside the aggregate.
- **Every command carries its aggregate's identity.** A command is a self-contained message: it says *which* instance it targets. For a command under aggregate `X`, include a field `xId: XId` in addition to the command's other fields. This applies to creation commands too: the caller supplies a freshly minted id.
  - Correct: `AssignCourier` on Order → fields `orderId: OrderId`, `courierId: CourierId`.
  - Wrong: `AssignCourier` with only `courierId` — cannot tell which Order to assign to.

### The actor

- **Every domain declares exactly one `actor`**, with a required `id` field mapped from the `sub` claim. Generation fails without it.
- **Map a claim only when a rule or injection needs it.** A required field rejects any token that lacks the claim, so mark a field `optional` unless every caller has it.
- **`id` is the provider's subject**, unique within one issuer. Map it to a customer or account id explicitly.
- Name fields in domain language (`roles`, `partnerId`). The `claim` string holds the provider's name, so changing provider changes only those strings.

### Projections must subscribe

Subscriptions are how a projection stays current. **Every projection must have at least one subscription**, one per event it consumes. When you propose a projection, propose its subscriptions in the same breath, and make sure each referenced event exists (i.e., some command in the design emits it). How subscriptions update the read model is implement-pass work.

## 4. Field-type coverage rule (mandatory)

**Primitives live inside terms.** Primitives (`string`, `integer`, `float`, `date`, `boolean`, `money`) appear in exactly one place: as the wrapped primitive of a term.

Every field's type must reference an existing vocabulary element by name — a term, enumeration, valuetype, or entity that is already in the model or that you propose in the same design increment. Before finalizing any proposal, scan every field and verify its type resolves; if it doesn't, add the missing vocabulary item.

Prefer domain-specific terms drawn from the business vocabulary:

| Field | Wrong | Right |
|---|---|---|
| `total` | `money` | `OrderTotal` or `Money` (term wrapping money) |
| `createdAt` | `date` | `Timestamp` (term wrapping date) |
| `quantity` | `integer` | `Quantity` or `Number` (term wrapping integer) |
| `name` | `string` | `ProductName` or `Text` (term wrapping string) |
| `status` | `string` | `OrderStatus` (enumeration) |

When no domain-specific meaning is natural, fall back to this canonical generic set (propose each the first time it's needed, reuse by name thereafter):

| Term | Primitive | Term | Primitive |
|---|---|---|---|
| Text | string | Flag | boolean |
| LongText | string | Date | date |
| Number | integer | Timestamp | date |
| Decimal | float | Email | string |
| Money | money | PhoneNumber | string |
| | | Url | string |

A field may be a list (`isList`) when it collects multiple values (e.g., `lines: OrderLine[]`).

## 5. The `desc` convention

Every element gets a `desc` in markdown with exactly two sections:

```markdown
## Purpose
<1-2 sentences: what this element is and what it does>

## Need
<1-2 sentences: which business or user need it fulfills>
```

This rationale is what a reviewer reads to judge the element, and what the implement pass reads to fill in behavior. Write it for both audiences. Field descriptions are optional and brief (one line).

## 6. Completeness checklist

A high-level design is ready to hand to the implement pass when:

- [ ] **Vocabulary is complete** — every field's type resolves to a term/enumeration/valuetype/entity.
- [ ] **The actor** is declared, with `id` and any claims authorization will need.
- [ ] **Bounded contexts** are named and every aggregate/projection lives in one.
- [ ] **Every aggregate** has an `id: {Name}Id` identity field (with the term) plus its key state fields.
- [ ] **Every command** carries the aggregate identity field plus its request fields, and names its event (past participle) with any event-only fields.
- [ ] **Every projection** has read-model fields (including its identity) and at least one declared subscription, each referencing an event some command emits.
- [ ] **Every process** has its `id` correlation-key field, at least one intent contract (fields + the request field it correlates by), named states, and a `final` list of terminal outcomes; which facts and deadlines drive each transition is captured in the descs — handler bodies are implement-pass work.
- [ ] **Every query** states what it returns (noun-phrase name + desc) and lists its parameter fields.
- [ ] **Every element** has a `## Purpose` / `## Need` desc.
- [ ] **Design only** — inputs, errors, effects, invariants, and conditions wait for the implement pass.

## 7. Worked mini-example

A minimal library-lending domain, shown in the markdown proposal format the design session should emit.

---

### Proposed design: Library Lending

```
Lending (context)
├── Loan (aggregate)
│   ├── CheckOutBook (command) → emits BookCheckedOut
│   └── ReturnBook (command)   → emits BookReturned
└── MemberLoans (projection)
    ├── OnBookCheckedOut (subscription ← BookCheckedOut)
    ├── OnBookReturned (subscription ← BookReturned)
    └── LoansByMember (query)

Actor
└── id (claim: sub)

Vocabulary
├── ActorId (term: string)
├── LoanId (term: string)
├── MemberId (term: string)
├── BookId (term: string)
├── Timestamp (term: date)
└── LoanStatus (enumeration: Active, Returned, Overdue)
```

#### Aggregate: Loan

> **Purpose** — A single lending of one book to one member, from checkout to return.
> **Need** — The library must track who holds which book and enforce return rules.

| Field | Type | Notes |
|---|---|---|
| id | LoanId | identity |
| memberId | MemberId | the borrowing member |
| bookId | BookId | the borrowed book |
| status | LoanStatus | Active / Returned / Overdue |

#### Command: CheckOutBook → event `BookCheckedOut`

| Field | Type | Notes |
|---|---|---|
| loanId | LoanId | aggregate identity, minted by caller |
| memberId | MemberId | |
| bookId | BookId | |

Event-only fields on `BookCheckedOut`:

| Field | Type | Notes |
|---|---|---|
| checkedOutAt | Timestamp | server-generated |

#### Command: ReturnBook → event `BookReturned`

| Field | Type | Notes |
|---|---|---|
| loanId | LoanId | aggregate identity |

Event-only fields on `BookReturned`: `returnedAt: Timestamp`.

#### Projection: MemberLoans

> **Purpose** — Per-member view of current and past loans.
> **Need** — Members and librarians need to see what a member has borrowed.

| Field | Type | Notes |
|---|---|---|
| id | LoanId | identity |
| memberId | MemberId | lookup key |
| bookId | BookId | |
| status | LoanStatus | |
| checkedOutAt | Timestamp | |

Subscriptions: `OnBookCheckedOut` (← BookCheckedOut), `OnBookReturned` (← BookReturned).

Query `LoansByMember` — returns a member's loans. Parameters:

| Field | Type |
|---|---|
| memberId | MemberId |

#### Actor

| Field | Type | Claim | Notes |
|---|---|---|---|
| id | ActorId | sub | required — the provider's subject |

#### Vocabulary

| Item | Kind | Definition |
|---|---|---|
| ActorId | term | string — the caller's subject at the identity provider |
| LoanId | term | string — identity of a Loan |
| MemberId | term | string — identity of a library member |
| BookId | term | string — identity of a book copy |
| Timestamp | term | date — point in time |
| LoanStatus | enumeration | Active, Returned, Overdue |

---

Present proposals in exactly this shape: tree first, then per-element field tables, then the vocabulary table. Ask for feedback before writing anything to disk.
