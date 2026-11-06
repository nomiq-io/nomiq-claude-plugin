---
name: generate
description: Validate the nomiq model and generate microservice source code via the platform. Use when the user wants to generate code, build the services, or export the model to Java.
---

# Generate microservice source

One call on the happy path: `nomiq_generate` compiles and validates the DSL
locally, then generates via the platform and unpacks the zip into the
generated-sources directory. Do NOT read docs or run other tools first.

## Steps

1. Call `nomiq_generate` — no arguments in the default case. Output directory,
   package, language and identity provider come from `.nomiq/settings.json`;
   pass overrides only if the user explicitly asked for them. Generation needs
   `identity` in the settings and an `actor` declaration in the model.
2. **Success** → report the outcome and stop: output directory, file count,
   per-module counts (`modules`), how many existing files were overwritten
   (`overwrittenCount`), how many stale files from the previous generation
   were removed (`deletedCount` — generation tracks its output in a
   `.generated` ledger and prunes files the new model no longer produces;
   files it did not write are never touched), and how many hand-written
   manual files were overlaid into src/ (`copiedCount`). If the result carries
   `missingManual`, relay it — manual-marked elements are waiting for their
   implementation under `manual/` (call `nomiq_doc` for `manual-flow`);
   `strayManual` names manual
   files matching no manual element (stale after a rename or unmarking).
3. **`dsl-errors` / `local-errors`** → the model needs fixing before code can
   be generated. List the issues compactly (`file:line — message`), then ask
   the user whether to fix them now. Only if they say yes: run the
   validate-fix loop exactly as specified by the nomiq implement skill (call
   `nomiq_doc` for `slot-contexts` at that point, not before), then re-run
   step 1.

## Failure handling

- `invalid-settings` → `.nomiq/settings.json` is not valid JSON. Show the error and
  fix only that syntax; never rewrite the file, because it holds other settings.
- `missing-identity` / `invalid-identity` → ask the user which identity provider
  they use (`keycloak`, or `oauth` for any other provider issuing JWT access
  tokens), set `identity` in `.nomiq/settings.json`, then retry.
- `missing-actor` → the model has no `actor`. Ask which token claims the system
  relies on, call `nomiq_doc` for `dsl` and follow its "The `actor`" section, add
  the declaration and a term for each field to the root DSL file, then retry.
- `401`/`no-api-key` → route the user through the nomiq login skill, then retry.
- `422 compile-failed` → the server could not compile sources the local check
  accepted; report it as a platform/plugin version mismatch and show the message.
- `422 build-failed` → the model validates but the generator rejected it.
  Report the failure message (it names the project/module) and offer to fix as
  in step 3 — the fix loop treats it as one more issue.
- `manual-conflict` → generation now produces a path a `manual/` file also
  maps to: the element is no longer marked manual. Ask the user whether to
  delete the manual file or restore the `@manual` annotation; nothing was
  written.
- `manual-divergence` → a derived copy under src/ was edited in place. Show
  the user the diff against the `manual/` source and ask: adopt the src/ edits
  into `manual/` (call `nomiq_manual_sync` only after they approve) or revert
  them, then re-run. Nothing was written.

Never edit the generated sources to "fix" a build problem — fix the model and
regenerate. (Hand-written manual flows are the exception: their home is the
`manual/` tree, never the copies under src/.)

## Known platform limitations (verify before blaming the model)

- **`money` terms**: the Java templates currently emit a `double`→`BigDecimal`
  mismatch in the vocabulary `Json.java` — the generated code does not compile.
  Until fixed server-side, suggest `float` with a `precision` constraint.
- **Custom `package` setting**: a dotted package (e.g. `io.acme.shop`) produces
  inconsistent package qualifiers across modules (imports still reference the
  domain-name-derived package) — the generated code does not compile. Until
  fixed, omit `package` so everything derives from the domain name.
