# nomiq for Claude Code

Design event-sourced domain models conversationally, then generate Java
microservices from them. The model lives in your repository as `.nomiq` files,
and the nomiq platform turns it into source code.

## Install

```bash
claude plugin install nomiq --marketplace nomiq-io/nomiq-claude-plugin
```

Or from inside a session:

```
/plugin install nomiq --marketplace nomiq-io/nomiq-claude-plugin
```

The plugin needs Node.js 20 or later. To update it later, run
`claude plugin update nomiq@nomiq`.

## Get started

Use the skills with the `/nomiq:` prefix, or just describe what you want.

| Skill | Purpose |
| --- | --- |
| `/nomiq:init` | Start a nomiq project in the current repository |
| `/nomiq:login` | Set up the API key used for validation and generation |
| `/nomiq:design` | Design or evolve the domain model |
| `/nomiq:implement` | Add behavior: inputs, validations, effects, process handlers |
| `/nomiq:visualize` | Show the model tree with a health summary |
| `/nomiq:generate` | Validate the model and generate microservice source |

Use the prefix: bare `/design` is a Claude Code built-in.

## What the plugin sends and stores

The plugin runs a local MCP server and contacts one service,
`https://platform.nomiq.io`.

- **Sent:** your `.nomiq` DSL sources and generation settings, when you validate
  against the platform or generate. The platform does not store your sources:
  it builds in a temporary folder, returns a zip and deletes the folder.
- **Sent:** your nomiq API key, with each of those requests.
- **Not sent:** generated code, hand-written `manual/` sources, or any other file
  in your repository.
- **Stored locally:** the API key, in `~/.nomiq/credentials.json`. The
  `NOMIQ_API_KEY` environment variable takes precedence when set.
- **Written locally:** generated sources, into your project's source directory.
  Generation tracks what it wrote and only removes files it created.

Validation while you work runs locally, with no network call.

## Generated gateways

Generated gateways verify tokens from one identity provider. Set it in
`.nomiq/settings.json`, using `keycloak`, or `oauth` for any other provider
issuing JWT access tokens:

```json
{ "identity": "keycloak" }
```

Declare the caller as an `actor` in your model, then configure the gateway through
the environment. It refuses to start without the required settings, and
`GET /ready` answers 503 until signing keys are loaded, and again once cached
keys pass `GATEWAY_OIDC_MAX_KEY_AGE`.

| Variable | Provider | Purpose |
| --- | --- | --- |
| `GATEWAY_OIDC_ISSUER` | Both | Required: the exact issuer URL |
| `GATEWAY_OIDC_AUDIENCES` | Both | Required: API audiences, comma-separated |
| `GATEWAY_OIDC_ACCESS_TOKEN_RULE` | `oauth` | Required: `claim-typ:<value>`, `header-typ:<value>` or `none` |
| `GATEWAY_OIDC_CLIENTS` | Both | Allowed `azp` values, comma-separated |
| `GATEWAY_OIDC_JWKS_URI` | Both | JWKS URL, used instead of discovery |
| `GATEWAY_JWT_KEY_FILE` | `oauth` | PEM public key, for development or providers with no JWKS |

Keycloak access tokens carry only `aud: account` by default, so add an audience
mapper for your API.
