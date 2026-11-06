---
name: visualize
description: Show the current nomiq domain model as a tree with a health summary. Use when the user wants to see, inspect, or review the model structure.
---

# Visualize the model

Read-only: never modify any model source. `nomiq_model_check` is local,
offline, and writes nothing — it compiles the DSL sources in memory.

1. Call `nomiq_model_check` (pass `projectDir` if the user is working outside the
   current directory). If it reports the DSL directory is missing, tell the
   user to run the nomiq init skill. If it reports DSL compile errors, show them
   their file/line and stop — there is no tree until the sources compile.
2. Print the returned `tree` verbatim, e.g.:

   ```
   domain Shop
   ├─ term OrderId (string)
   ├─ enumeration OrderStatus
   └─ context Ordering
      ├─ aggregate Order
      │  ├─ command PlaceOrder → event OrderPlaced
      │  └─ query GetOrder (one)
      ├─ projection OrderList
      │  ├─ subscription OnOrderPlaced
      │  └─ query ListOrders (page)
      └─ process Fulfillment
         ├─ intent FulfillOrder
         ├─ init
         │  └─ handler OnFulfillOrder → event FulfillmentStarted
         ├─ state started
         │  ├─ handler OnOrderShipped
         │  └─ handler After2D
         └─ final shipped expired
   ```

3. Summarize the returned `issues` grouped by file (say "model is structurally
   clean" when empty). Additionally point out domain-level smells the tool does
   not flag: projections without subscriptions, commands missing their
   aggregate's identity field, fields typed directly by primitives (`#string`
   etc.).
4. Summarize counts (contexts, aggregates, commands, projections, processes,
   queries) and, if behavior is missing (commands with empty `effects`),
   mention the nomiq implement skill as the next step.

Fallback: if the `nomiq_model_check` tool is unavailable, read the raw
`.nomiq` files in the DSL directory (settings `dsl` in `.nomiq/settings.json`,
default `dsl/`) and build the tree from the declarations directly.
