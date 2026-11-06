---
name: login
description: Sign in to the nomiq platform — set up or refresh the API key used for validation and code generation. Use when the user wants to log in, authenticate, fix an expired/revoked key, or when another nomiq tool reports a missing or rejected API key.
---

# Sign in to the nomiq platform

The plugin authenticates with an API key (`nmq_…`). The key lives in
`~/.nomiq/credentials.json` — outside the repository — or in the `NOMIQ_API_KEY`
environment variable (which always wins; used for CI/headless runs).

## Steps

1. Call `nomiq_auth_status`.
   - If it reports a working key (`user`, `plan` present), tell the user who they
     are signed in as and stop, unless they want to switch keys.
   - If the key comes from `env`, say so: replacing it means changing the
     environment variable.
2. If there is no key or the platform rejects it: give the user the portal keys
   page URL from the status/hint output (e.g. `<portalUrl>/keys`), and ask them to
   sign in there, mint a key, and paste it here.
3. Call `nomiq_auth_set` with the pasted key. The tool verifies the key against
   the platform **before** saving, and reports the account email and plan.
   - On failure, relay the error (`unauthorized` → wrong/revoked key;
     `no-subscription` → account has no active plan) and let the user retry.
4. Confirm success with the account email. **Never echo the full key back and
   never write it into any file inside the repository.**
