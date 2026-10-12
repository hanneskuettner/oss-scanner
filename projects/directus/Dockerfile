# Directus for the OSS Scanner: the whole pnpm monorepo (API, app, SDK and shared packages) built from source with
# development dependencies kept, plus a local SQLite instance that starts with no network (`directus-start`).
# node:22-trixie carries the toolchain native modules need (gcc, make, python3, git, curl), and the glibc (2.38+)
# the prebuilt sqlite3 binary needs, which bookworm's 2.36 lacks.
FROM node:22-trixie
ENV DEBIAN_FRONTEND=noninteractive \
    CI=true \
    HUSKY=0

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# pnpm at the version package.json pins (packageManager).
RUN corepack enable && corepack prepare --activate

# Node's default heap is too small for the app bundle and the API's type build.
ENV NODE_OPTIONS=--max-old-space-size=8192

# Same steps as CI (.github/actions/prepare): install, then build every package. The e2e suite needs Docker
# databases and is left out, as in the project's own Dockerfile.
RUN pnpm install --frozen-lockfile
RUN npm_config_workspace_concurrency=2 pnpm --recursive --filter '!e2e' run build

# Local-only settings for a runnable instance; no production credentials are involved. They live in a config file
# rather than ENV so they do not leak into the unit tests, which assume Directus's defaults.
COPY <<'CONFIG' /src/.local/directus.env
DB_CLIENT=sqlite3
DB_FILENAME=/src/.local/database.sqlite
STORAGE_LOCATIONS=local
STORAGE_LOCAL_ROOT=/src/.local/uploads
EXTENSIONS_PATH=/src/.local/extensions
SECRET=oss-scanner-local-only-not-a-production-secret
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=oss-scanner-admin
PUBLIC_URL=http://127.0.0.1:8055
TELEMETRY=false
CONFIG
RUN mkdir -p /src/.local/uploads /src/.local/extensions \
    && CONFIG_PATH=/src/.local/directus.env node /src/directus/cli.js bootstrap

COPY --chmod=755 <<'SCRIPT' /usr/local/bin/directus-start
#!/bin/bash
# Starts Directus on http://127.0.0.1:8055 against the bundled SQLite database (/src/.local).
# Log in as admin@example.com / oss-scanner-admin. Settings are in /src/.local/directus.env; extra environment
# (LOG_LEVEL=debug, ...) passes through.
export CONFIG_PATH="${CONFIG_PATH:-/src/.local/directus.env}"
exec node /src/directus/cli.js start "$@"
SCRIPT

# Smoke test: boot the instance once. A failure is reported in the build log and does not abort the build.
RUN (directus-start > /tmp/directus-smoke.log 2>&1 & \
      for _ in $(seq 1 120); do curl -fsS http://127.0.0.1:8055/server/ping > /dev/null 2>&1 && break; sleep 1; done; \
      if curl -fsS http://127.0.0.1:8055/server/ping; then echo " <- directus is up"; \
      else echo "WARNING: directus did not come up"; tail -n 80 /tmp/directus-smoke.log; fi; \
      pkill -f 'directus/cli.js start'; sleep 2) || echo "WARNING: smoke test failed"

# Unit tests (vitest) across the workspace. Failures are logged, not fatal.
RUN pnpm --recursive --filter '!e2e' test || echo "WARNING: unit tests reported failures"
