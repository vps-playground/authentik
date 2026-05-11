# authentik — identity provider for vps-playground

[Authentik](https://goauthentik.io/) deployed on the shared vps-playground VPS. Acts as the **identity-aware ingress** in front of platform workloads: Traefik delegates auth decisions to Authentik via forward-auth, workloads receive a verified user identity as trusted headers.

This repo is the canonical, version-controlled definition of how Authentik runs on the VPS. The actual deployment lives in Coolify on the host described in [`vps-playground/vps-control-plane`](https://github.com/vps-playground/vps-control-plane).

## What's in here

| File | Purpose |
|---|---|
| `docker-compose.yml` | Authentik services (server + worker + postgres). Mirrors [upstream](https://goauthentik.io/docker-compose.yml) with one delta (see comment at top). |
| `.env.example` | Required environment variables. Real values live in Coolify, not in git. |
| `traefik-dynamic/authentik.yml` | Traefik forward-auth middleware definition. Reference as `authentik@file` on workload routers. |

## Deploy via Coolify

Prerequisite: a working Coolify install (see `vps-control-plane`).

1. **Generate secrets locally** (don't reuse across environments):

   ```sh
   openssl rand -base64 36 | tr -d '\n'   # → PG_PASS
   openssl rand -base64 60 | tr -d '\n'   # → AUTHENTIK_SECRET_KEY
   ```

2. **Coolify UI → New Resource → Docker Compose**:
   - Source: this repo, `main` branch
   - Compose path: `docker-compose.yml`
   - Domain: `https://auth.62.238.23.188.sslip.io` (or your registered parent if not on sslip.io)
   - Target service: `server`, port `9000`
   - Environment variables: paste from `.env.example`, fill in the generated secrets

3. **Deploy.** Wait for the Let's Encrypt cert. First boot runs migrations (~30s).

4. **Bootstrap admin.** Visit `https://auth.62.238.23.188.sslip.io/if/flow/initial-setup/` — Authentik's first-run flow lets you create the admin account.

5. **Install the forward-auth middleware:**

   ```sh
   scp traefik-dynamic/authentik.yml agr@62.238.23.188:/tmp/
   ssh agr@62.238.23.188 'sudo mv /tmp/authentik.yml /data/coolify/proxy/dynamic/ && sudo chown root:root /data/coolify/proxy/dynamic/authentik.yml'
   ```

   Traefik picks it up automatically (file watcher). Verify by checking Coolify → Servers → localhost → Proxy → Logs for `Adding middleware [authentik]`.

## Configuring forward-auth in Authentik

After deploy, in the Authentik admin UI:

1. **Applications → Providers → Create** → *Proxy Provider*:
   - Mode: **Forward auth (domain level)**
   - External host: `https://auth.62.238.23.188.sslip.io`
   - Cookie domain: `62.238.23.188.sslip.io`

2. **Applications → Applications → Create** → bind to the provider above. Slug `domain-level`.

3. **Applications → Outposts** → edit the **embedded outpost** → add the application.

That single domain-level provider protects every workload that opts in via the `authentik@file` middleware. Per-application policies (group membership, etc.) attach to *Applications* in Authentik, not at the middleware layer.

## Workload integration

A workload becomes "identity-aware" by:

1. Joining Coolify's Traefik network (default for Coolify apps).
2. Adding the middleware label on its router:
   ```
   traefik.http.routers.<name>.middlewares=authentik@file
   ```
3. Reading user identity from `X-Authentik-Username` / `X-Authentik-Email` / `X-Authentik-Groups` headers.

For path-level exemptions (e.g. `/healthz`, `/.well-known/acme-challenge/`), define a second router on the same service *without* the middleware, with a higher priority and a path match.

See `vps-control-plane/docs/identity.md` for the full design rationale, per-workload patterns, and exemption recipes.

## Updates

Bump `AUTHENTIK_TAG` in `.env` (or in Coolify env vars) and redeploy. **Always check [release notes](https://goauthentik.io/docs/releases)** for schema migrations — Authentik runs them on boot but breaking changes occasionally need manual steps.

To resync the upstream compose:

```sh
curl -fsSL https://goauthentik.io/docker-compose.yml -o /tmp/upstream.yml
diff -u /tmp/upstream.yml docker-compose.yml   # review, then re-apply the documented delta
```
