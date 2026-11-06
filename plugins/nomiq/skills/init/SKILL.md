---
name: init
description: Initialize a nomiq project in the current repository — creates .nomiq/ with the domain root and project settings. Use when the user wants to start a new nomiq domain model.
---

# Initialize a nomiq project

Set up a nomiq project: the model is authored in `.nomiq` DSL files — the
**only source of truth** — and project settings live in `.nomiq/settings.json`.

## Steps

1. If `.nomiq/settings.json` already exists, stop and tell the user the project
   is already initialized (point them to the nomiq design skill to evolve the
   model).
2. Ask the user (one question, sensible defaults):
   - **Domain name** — PascalCase, e.g. `Shop` (required).
   - **Language** — only `java` is supported today (default: `java`).
   - **Package** — e.g. `io.acme.shop` (default: derive from the domain name).
   - **DSL directory** — where the `.nomiq` sources live (default: `dsl/`).
   - **Source directory** — where generated code is unpacked (default: `src/`).
3. Write `.nomiq/settings.json`:

   ```json
   {
     "language": "java",
     "identity": "keycloak",
     "package": "io.acme.shop",
     "dsl": "dsl",
     "src": "src"
   }
   ```

   `identity` is required to generate: `keycloak`, or `oauth` for any other
   provider issuing JWT access tokens. Ask the user which one they use.
   `dsl` and `src` may be omitted when the user keeps the defaults. Other
   optional keys (`techStack`, `apiUrl`, `portalUrl`) exist for advanced
   setups — do not write them unprompted.
4. Create the DSL directory and write `<dsl>/domain.nomiq` — the root file
   holding the domain declaration and the authenticated caller (vocabulary
   will join it later):

   ```nomiq
   domain Shop

   term ActorId: string

   /// The authenticated caller.
   actor {
     field id: ActorId claim "sub"
   }
   ```

5. Run `nomiq_model_check` — local and offline; it compiles the DSL sources
   and confirms the project is well-formed.
6. Tell the user:
   - Commit the DSL directory and `.nomiq/settings.json` — together they are
     the whole model. Whether the generated source directory is committed is
     team preference. API credentials live elsewhere, in `~/.nomiq/`, and
     never end up in the repo.
   - Next step: the nomiq design skill, to design the domain model.
