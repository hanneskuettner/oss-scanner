# Directus

Directus is a backend that wraps any SQL database (PostgreSQL, MySQL/MariaDB,
SQLite, MS SQL, Oracle, CockroachDB) with REST, GraphQL and WebSocket APIs, and
a Vue admin app on top of them. It is self-hosted and is routinely exposed to
the public internet, often with the "Public" role granted read access to some
collections. This enrollment covers the pnpm monorepo in directus/directus:
`api/` (the Node.js server), `app/` (the admin app), `sdk/`, and `packages/*`.

Our published advisories (https://github.com/directus/directus/security/advisories)
are the best guide to what we consider a vulnerability and how we score it.

## Where untrusted input enters

- **Anonymous clients**: every REST/GraphQL route and the WebSocket endpoint,
  evaluated with the Public role's permissions. Login, password reset, public
  registration, invites, SSO callbacks (`api/src/auth/drivers`: local, OAuth2,
  OpenID, LDAP, SAML), shares, `/assets` (including image transformations),
  `/server/*`, `/utils/*`, and webhook- or manually-triggered flows.
- **Authenticated users** with limited roles/policies: query parameters
  (`filter`, `fields`, `deep`, `sort`, `search`, `aggregate`, `groupBy`,
  `alias`, `export`), item payloads (including nested relational writes),
  GraphQL documents, WebSocket messages and subscriptions, file uploads
  (multipart and TUS), file import from URL, imports/exports, content versions,
  revisions, comments, shares, presets, and the AI chat.
- **Content stored by one user and rendered for another** in the admin app:
  field values, display/interface/layout options, markdown, WYSIWYG HTML,
  comments, project settings (name, color, logo), uploaded SVG/HTML files.

## What matters most

These areas have produced most of our past advisories and deserve the deepest
review:

1. **Data access control** (`api/src/permissions`, `api/src/services`,
   `api/src/utils/sanitize-query.ts`, `api/src/database/run-ast`). A
   role/policy must only read and write the collections, fields and items its
   rules allow. Past bugs: concealed or unreadable fields recovered through
   `aggregate`, `search`, `alias`, `_in` filters, revisions and exports;
   overlapping policies widening field write access; side effects in service
   overrides running before the permission check; deleted fields keeping
   permissions; existence of collections leaking.
2. **WebSocket and GraphQL subscriptions** (`api/src/websocket`): auth state
   after a failed or expired auth message, subscription permission checks,
   Origin validation on upgrade, malformed messages crashing the process.
3. **Authentication and sessions**: JWT/refresh/session handling, static
   tokens, suspended users keeping access, TFA, password reset and
   registration (where the email goes, timing, collation-insensitive matching
   on MySQL), SSO account linking and session caching, MCP OAuth
   (`api/src/services/mcp-oauth`), open redirects in OAuth2/OpenID/SAML/TFA
   flows.
4. **Privilege escalation**: a non-admin gaining admin access, changing its own
   or another user's role/policies, or abusing shares (reassigning
   `user_created`/role) or presets to act as someone else.
5. **SSRF** through file import from URL (`api/src/utils/import-file-url.ts`),
   the AI chat's file downloads, the flow "request" operation and anything else
   that fetches a URL. We have repeatedly fixed bypasses of
   `IMPORT_IP_DENY_LIST` (`0.0.0.0`, loopback forms, IPv4-mapped IPv6,
   redirects); new bypasses are high severity.
6. **Files and storage**: TUS upload authorization and path validation,
   overwriting another user's file, path traversal in storage drivers or in the
   mail template engine, `filename_disk` handling, inline serving of active
   content.
7. **Caching**: responses that depend on the caller's permissions cached under a
   key that does not include them (`api/src/utils/get-cache-key.ts`).
8. **Flows**: manual and webhook triggers running without the intended
   permission checks, or returning data the caller cannot read.
9. **SQL injection** through the query DSL or schema operations (field types,
   geometry, defaults) for any supported vendor.
10. **Sandbox escapes** for sandboxed extensions and the flow "exec" (run
    script) operation (`isolated-vm`; `api/src/extensions/lib/sandbox`,
    `api/src/operations/exec`).
11. **Denial of service**: GraphQL alias/field amplification and unbounded
    documents (HTTP and WebSocket), expensive unauthenticated utilities
    (hashing, health checks), image transformations with unvalidated
    dimensions, prototype pollution in query parsing, malformed input that
    crashes the process or soft-locks the instance.
12. **Secrets in logs and responses**: tokens or credentials written to logs
    (query-string tokens, flow logs, event hooks), and sensitive
    `directus_settings` fields (for example AI provider headers) readable by
    non-admins.
13. **Stored and DOM XSS in the admin app** (`app/`), including through
    project settings, comments, layout options and generated assets.

Less important: the `create-directus-extension`/`create-directus-project` CLIs,
`packages/release-notes-generator`, and test utilities. Cloud storage drivers
are in scope for path and permission handling; their SDKs are not.

## How to exercise it

The image has every package built at `/src` and a bootstrapped SQLite instance.
Run `directus-start` to serve the API and app on http://127.0.0.1:8055 (admin:
`admin@example.com` / `oss-scanner-admin`). Create roles, policies, collections,
shares and flows through the API as admin, then reproduce as a restricted user
or anonymously. Unit tests: `pnpm --filter @directus/api test` and
`pnpm --filter @directus/app test` (vitest). The `tests/e2e` suite needs Docker
databases and does not run offline. External services (SMTP, S3, SSO providers,
AI providers, PostgreSQL/PostGIS, MySQL) are not available; mock them, or reason
from the code for vendor-specific paths.

## How we rate severity

We score with CVSS 3.1, as in our published advisories. As a guide:

- **Critical**: unauthenticated RCE, admin takeover, or unrestricted read/write
  of data (for example, a Public WebSocket client gaining full item access; an
  unauthenticated file upload/overwrite).
- **High**: permission bypasses exposing or modifying data a role must not
  access, including concealed fields; SSRF that bypasses the deny list; TUS or
  file authorization bypasses; permission-dependent responses served from a
  shared cache; cross-site WebSocket hijacking; SQL injection, even when it
  needs admin rights; sandbox escapes; unauthenticated denial of service
  (crash, soft-lock, or amplification); user enumeration that reveals
  accounts reliably.
- **Medium**: field-level information leaks with limited impact; stored XSS;
  open redirects; secrets written to logs; flows triggerable without intended
  permissions; authenticated denial of service; timing-based enumeration;
  version disclosure; admin-only issues that reach outside Directus's own data,
  such as reading host files.
- **Low**: issues that need an unusual configuration or user interaction from
  an admin with limited impact, such as DOM XSS through admin-only options.

Admins are trusted to manage Directus's own data, schema, flows and extensions,
so that alone is not a vulnerability. An admin crossing a boundary Directus
should still enforce is in scope: SQL injection, reading host files, escaping
the extension/flow sandbox, or recovering password hashes and other redacted
values.

## Anything to leave alone

- Vulnerabilities in dependencies with no demonstrated reachable path in
  Directus.
- Missing rate limiting when `RATE_LIMITER_ENABLED` is off (the default),
  unless one request causes disproportionate work (amplification).
- Behaviour that only occurs with clearly insecure environment settings, such as
  a weak `SECRET` or a wildcard `CORS_ORIGIN` with credentials.
- Running non-sandboxed extensions, which are trusted code by design.
- Licensing/entitlement checks (`api/src/license`); bypassing them is not a
  security issue for us.

Please include a minimal reproduction against the bundled instance (HTTP or
WebSocket requests with the role/permission setup) and, where possible, a patch
with a unit test in the style of the neighbouring `*.test.ts`.
