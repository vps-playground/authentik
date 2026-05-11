# Authentik blueprints

Reusable Authentik blueprints for the vps-playground platform.

[Authentik blueprints](https://docs.goauthentik.io/docs/customize/blueprints/) are YAML files that declare Authentik objects (Providers, Applications, policy bindings, groups, …) idempotently. Applying a blueprint creates the objects or updates them to match.

## How application works here

This directory is **bind-mounted into the Authentik worker at `/blueprints/`** (see `docker-compose.yml`). The worker scans it on boot and on file-content change, creates a `BlueprintInstance` for each `*.yaml`, and applies it.

That means **the workflow for onboarding a new workload is:**

1. Copy `workload-template.yaml` → `<workload>.yaml` in this directory.
2. Replace the `<<<…>>>` placeholders.
3. Set `metadata.labels.blueprints.goauthentik.io/instantiate: "true"`.
4. Commit, push, redeploy Authentik in Coolify.
5. Authentik's worker discovers and applies. Status in the UI under **System → Blueprints**.

That's it. No UI clicks per workload (except the one-time outpost binding — see below).

## Files

| File | Purpose |
|---|---|
| `workload-template.yaml` | Per-workload Provider + Application + optional Group binding. Carries `instantiate: "false"` so the placeholder body isn't discovered as an applyable blueprint. Copy to a sibling file and flip the flag to `true`. |
| `solarscout.yaml` | Live config for the solarscout workload. First consumer of the auto-discovery path. |

## Why not paste in the UI?

The Authentik UI's **System → Blueprints → Create** form only accepts a *path* (filesystem) or an OCI reference — there's no working inline-YAML field as of 2026.x. POSTing a `BlueprintInstance` via the API with the `content` field also doesn't apply (Authentik treats `content` as a fetched-blueprint cache, not as a source). File-mounted discovery is the only path that actually works end-to-end for inline-style YAML.

## Template placeholders

`workload-template.yaml` uses `<<<NAME>>>` markers for substitution. Search-replace the following before applying:

| Placeholder | Example | Notes |
|---|---|---|
| `<<<WORKLOAD_SLUG>>>` | `myapp` | Lowercase, hyphenated. Used for the Application slug and as the Provider's name. |
| `<<<WORKLOAD_NAME>>>` | `My App` | Human-readable, shown in the Authentik dashboard. |
| `<<<WORKLOAD_HOST>>>` | `myapp.3eee17bc.nip.io` | Hostname only (no scheme). Used in `external_host` (the template adds `https://`). |
| `<<<REQUIRED_GROUP>>>` | `admins` | Only needed if the workload is admin-gated. Otherwise delete the entire `policybinding` entity from the YAML. |

Don't forget to flip `instantiate: "false"` → `"true"` so the worker actually applies it.

## Prerequisites at apply time

Every `!Find` reference must resolve, or the blueprint silently rolls back and the UI shows "Successful" with no `managed_models`. Common prereqs:

- **Groups referenced by `policybinding` entities must exist** (Directory → Groups). Create them in the UI before pushing the workload blueprint.
- **Flow slugs** (`default-authentication-flow`, `default-provider-authorization-implicit-consent`) exist out of the box; only relevant if your platform install renamed them.

## Outpost binding

The template **does not** bind the new Application to the embedded outpost — that's a one-line UI click per workload (Outposts → embedded outpost → Edit → Applications tab → check the new app → Save), and doing it via blueprint would mean re-listing every existing Application in the outpost's providers list (Authentik blueprints replace, not append, for list-typed fields).

When platform automation matures and we want fully scripted onboarding, the right move is to model the outpost's full Application list as a single owned blueprint. Until then: one UI click per workload.

## Validation

Authentik validates blueprints on apply and reports parse/schema errors in **System → Blueprints**. If a blueprint fails, fix and retry — failed blueprints don't partially apply.

Schema reference: <https://goauthentik.io/blueprints/schema.json> (loadable into editor for YAML completion).
