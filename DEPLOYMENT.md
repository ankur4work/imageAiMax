# ImageAi Max — deployment

Live at **https://imageaimax.onkra.online** on Coolify (`coolify.solnix.store`,
VPS `173.212.233.194`). Deployed 2026-10-08. Coolify 4.3.23.

## Coolify resources

| Resource | UUID | Notes |
|---|---|---|
| Project | `xhobjua6tpaabgvebrebhho1` | "ImageAi Max" |
| Environment | `ykaah0sgxml5d3xjk21wlfdp` | `production` |
| Server | `myfwitwdjhljv0ksumbn9vqq` | `localhost` (the Coolify host) |
| Application | `dfvfndvp91bauiwbmca9eh9w` | `imageai-max` |
| Database | `wvlrnboj4v8orermxgefnioj` | `imageaimax-postgres` (standalone PostgreSQL) |

Do **not** copy UUIDs from a sibling app (PixelPro Max, ImageBoost SEO) — they
address *that* app's container and database, and the sibling `Session` rows hold
access tokens issued to a different `client_id`.

## How it's wired

- **Source:** public GitHub repo `ankur4work/imageAiMax`, branch `main`, build
  pack **dockerfile**. No deploy key or GitHub App — the repo is public, so
  Coolify clones it anonymously. Pushing `main` does **not** auto-redeploy (no
  git webhook is configured); trigger a deploy explicitly (Coolify UI, or
  `POST /api/v1/deploy?uuid=dfvfndvp91bauiwbmca9eh9w`).
- **Port:** container listens on `3000`; health check `GET /healthz`.
- **Migrations:** the container start command is `prisma migrate deploy &&
  react-router-serve` (see `Dockerfile` → `npm run docker-start`). Because the
  health check only passes once the server is serving, a healthy container is
  proof the migration applied. The DB holds the `0001_init` baseline.
- **Database URL:** set as a runtime env var in Coolify, using the Postgres
  resource's **internal** hostname (the container UUID above) over Coolify's
  Docker network. It is not reachable from outside the host.

- **DNS:** `imageaimax.onkra.online` must have an A record pointing at
  `173.212.233.194`. Coolify issues the Let's Encrypt certificate only once that
  record resolves, so a missing record shows up as a TLS failure rather than a
  404.

## Secrets (NOT in git)

Set as **runtime** environment variables on the `imageai-max` application in
Coolify: `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET`, `SHOPIFY_APP_URL`, `SCOPES`,
`SHOPIFY_APP_HANDLE`, `DATABASE_URL`, `OPENAI_API_KEY`. The Postgres password is
stored on the database resource in Coolify. None of these live in the repo.

All seven are set with `is_buildtime = false`. Keep it that way: Coolify's
default for a new variable is build-time **and** runtime, and a build-time
variable is passed to `docker build` as an ARG, where it persists in the image
layer history. Nothing in the Dockerfile needs any of these at build time —
`prisma generate` reads only `schema.prisma`, and `react-router build` bundles
without executing the server modules.

Coolify also auto-creates a second, preview-deployment copy of each variable
(`is_preview = true`) that keeps the platform defaults. Those are inert while no
preview deployments are configured; if preview deploys are ever enabled, fix
their `is_buildtime` flags too.

## Still required before App Store launch

These are not deployable via the Coolify API and remain manual:

1. **Push Shopify app config** — `shopify.app.toml` (URLs, scopes, webhook
   subscriptions incl. the mandatory compliance webhook) only takes effect once
   pushed to Shopify: `shopify app deploy --allow-updates` with
   `SHOPIFY_APP_AUTOMATION_TOKEN` set. Until then the registered webhooks /
   redirect URLs are whatever the Dev Dashboard already has.
2. **Create the Managed Pricing plans** in the Dev Dashboard — `Free`,
   `Starter`, `Growth`, `Pro` (+ `… Annual`), names matching
   `app/plans.server.js` byte-for-byte. **The Free plan is mandatory** or a
   reviewer on a dev store hits an impassable pricing wall.
3. **Confirm `SHOPIFY_APP_HANDLE`** — set to the assumed `imageai-max`. Verify
   against a real install URL (`/store/<store>/apps/<handle>/…`) and update the
   Coolify env var if Shopify appended a suffix. A wrong handle 404s every
   pricing CTA.
4. **Rotate `SHOPIFY_API_SECRET`** — the secret for client_id
   `a12b8d4f0dedb95d87db44f6150b17f7` was shared in plaintext during setup.
   Rotate it in the Dev Dashboard and update the Coolify env var.
5. **Set `SUPPORT_EMAIL`** in Coolify — otherwise `/privacy` shows the inherited
   fallback address.
