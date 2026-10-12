# Directus

Directus is a backend that wraps any SQL database (PostgreSQL, MySQL/MariaDB,
SQLite, MS SQL, Oracle, CockroachDB) with REST, GraphQL and WebSocket APIs, and
a Vue admin app on top of them. It is self-hosted and is routinely exposed to
the public internet, often with the "Public" role granted read access to some
collections. This enrollment covers the pnpm monorepo in directus/directus:
`api/` (the Node.js server), `app/` (the admin app), `sdk/`, and `packages/*`.

## Where untrusted input enters

- **Anonymous clients**: every API route, evaluated with the Public role's
  permissions. Login, password reset, user registration/invites, SSO callbacks
  (`api/src/auth/drivers`: local, OAuth2, OpenID, LDAP, SAML), shares
  (`/shares`), asset delivery (`/assets`), `/server/*`, and webhook-triggered
  flows.
- **Authenticated users** with limited roles/policies: query parameters
  (`filter`, `fields`, `deep`, `sort`, `search`, `aggregate`, `groupBy`,
  `alias`), item payloads (including nested relational writes), GraphQL
  queries, WebSocket subscriptions, file uploads (multipart and TUS), imports
  and exports, and content versions.
- **Content stored by one user and rendered for another** in the admin app:
  field values, display/interface options, markdown, WYSIWYG HTML, SVG and
  other uploaded files, comments, notifications.

## What matters most

1. **Access control** (`api/src/permissions`, `api/src/services`,
   `api/src/utils/sanitize-query.ts`, `api/src/database/run-ast`). The core
   promise is that a role/policy only reads and writes the collections,
   fields and items its permission rules allow. Bypasses through relational
   traversal (`fields=a.b.c`, `deep`, M2A), filters or aggregates that leak
   values of fields the user cannot read, GraphQL vs REST discrepancies,
   permission-rule dynamic variables (`$CURRENT_USER`, `$CURRENT_ROLE`, ...),
   batch/nested writes, versions and revisions are all in scope.
2. **Authentication and sessions**: JWT/refresh/session handling, static
   tokens, TFA, password reset, invites, SSO account linking, share passwords,
   MCP OAuth (`api/src/services/mcp-oauth`).
3. **Privilege escalation**: a non-admin gaining admin access, changing its own
   role/policies, or reaching admin-only endpoints (settings, schema,
   extensions, flows, server/metrics).
4. **SQL injection** through the query DSL into Knex for any supported
   database vendor.
5. **Server-side request forgery** in file import from URL
   (`api/src/utils/import-file-url.ts`), the flow "request" operation, and
   anything gated by `IMPORT_IP_DENY_LIST` / `is-url-allowed`.
6. **File handling**: path traversal in storage drivers and asset
   transformations (sharp), unsafe content types served inline, upload size
   and type limits.
7. **Sandbox escapes** for sandboxed extensions and the flow "exec" (run
   script) operation, which run in `isolated-vm`
   (`api/src/extensions/lib/sandbox`, `api/src/operations/exec`).
8. **Stored XSS in the admin app** (`app/`) from content a lower-privileged user
   can write and an admin will view.

Less important: the `create-directus-extension`/`create-directus-project` CLIs,
`packages/release-notes-generator`, and test utilities. Storage drivers for
third-party clouds are in scope for path handling but their SDKs are not.

## How to exercise it

The image has every package built at `/src` and a bootstrapped SQLite instance.
Run `directus-start` to serve the API and app on http://127.0.0.1:8055 (admin:
`admin@example.com` / `oss-scanner-admin`). Create roles, policies and
collections through the API as admin, then reproduce as a restricted user or
anonymously. Unit tests: `pnpm --filter @directus/api test` and
`pnpm --filter @directus/app test` (vitest). The `tests/e2e` suite needs Docker
databases and does not run offline. External services (SMTP, S3, SSO providers,
AI providers) are not available; mock them.

## How we rate severity

- **Critical**: unauthenticated RCE; unauthenticated admin takeover or
  authentication bypass; unauthenticated read/write of arbitrary data or SQL
  injection.
- **High**: authenticated (non-admin) privilege escalation to admin, RCE or
  sandbox escape; permission bypass exposing or modifying data a role must not
  access; SQLi reachable by a non-admin; SSRF that reaches internal services
  despite the deny list; stored XSS a non-admin can plant that executes for an
  admin.
- **Medium**: field-level information leaks through filters/aggregates/errors;
  account enumeration with practical impact; CSRF; open redirects in auth flows;
  DoS reachable without authentication.
- **Low**: issues that require a non-default insecure configuration, DoS that
  needs authentication, missing hardening without a demonstrated impact.

Admin users are fully trusted: anything that needs admin rights (including
installing extensions, writing flows, or editing the schema) is out of scope
unless it escalates beyond the Directus process's intended capabilities, for
example a sandbox escape.

## Anything to leave alone

- Vulnerabilities in dependencies with no demonstrated reachable path in
  Directus.
- Missing rate limiting when `RATE_LIMITER_ENABLED` is off (the default).
- Behaviour that only occurs with clearly insecure environment settings, such as
  a weak `SECRET` or `CORS_ORIGIN=true` with credentials.
- Licensing/entitlement checks (`api/src/license`); bypassing them is not a
  security issue for us.

Please include a minimal reproduction against the bundled instance (HTTP
requests with the role/permission setup) and, where possible, a patch with a
unit test in the style of the neighbouring `*.test.ts`.
