# Authentik blueprints

Reusable Authentik blueprints for the vps-playground platform.

[Authentik blueprints](https://docs.goauthentik.io/docs/customize/blueprints/) are YAML files that declare Authentik objects (Providers, Applications, policy bindings, groups, …) idempotently. Applying a blueprint creates the objects or updates them to match.

## When to use a blueprint vs the UI

| Situation | Best path |
|---|---|
| First-time platform setup (Brand, embedded outpost wiring) | UI — these are one-off and the UI's first-run flow guides you |
| Adding a new protected workload (Provider + Application + Bindings) | **Blueprint** — declarative, diff-able, reproducible per workload |
| Editing existing config while debugging | UI — fast iteration, then back-port to blueprint |

## Files

| File | Purpose |
|---|---|
| `workload-template.yaml` | Per-workload Provider + Application + optional Group binding. Copy, edit placeholders, apply. |

## Applying a blueprint

Three options, pick what fits.

### Option A — Authentik UI (one-off)

1. Edit a copy of `workload-template.yaml` locally, replacing the `<<<…>>>` placeholders.
2. Authentik UI → **System → Blueprints → Create** → paste the YAML → set state to **Apply**.
3. Authentik applies it. Verify under **Applications → Applications** that the new entry exists.

### Option B — Mounted in the worker container (auto-applied on boot)

Authentik's worker scans `/blueprints/` for `*.yaml` and applies matching ones on every boot. To use this:

1. Add a bind-mount in the workload's deployment that mounts your blueprint files into `/blueprints/<some-folder>/` on the worker container.
2. Restart the worker (or the whole Authentik compose).

We don't currently use this path — the worker compose in this repo only mounts `authentik-data:/data`. Worth wiring up if blueprint count grows.

### Option C — API push (CI-friendly)

Authentik exposes a blueprint API; a CI job could `POST` blueprints to it. Out of scope for now; revisit when we have ≥5 protected workloads and want zero-UI workload onboarding.

## Template placeholders

`workload-template.yaml` uses `<<<NAME>>>` markers for substitution. Search-replace the following before applying:

| Placeholder | Example | Notes |
|---|---|---|
| `<<<WORKLOAD_SLUG>>>` | `myapp` | Lowercase, hyphenated. Used for the Application slug and as the Provider's name. |
| `<<<WORKLOAD_NAME>>>` | `My App` | Human-readable, shown in the Authentik dashboard. |
| `<<<WORKLOAD_HOST>>>` | `myapp.3eee17bc.nip.io` | Hostname only (no scheme). Used in `external_host` (the template adds `https://`). |
| `<<<REQUIRED_GROUP>>>` | `admins` | Only needed if the workload is admin-gated. Otherwise delete the entire `policybinding` entity from the YAML. |

## Outpost binding

The template **does not** bind the new Application to the embedded outpost — that's a one-line UI click per workload (Outposts → embedded outpost → Edit → Applications tab → check the new app → Save), and doing it via blueprint would mean re-listing every existing Application in the outpost's providers list (Authentik blueprints replace, not append, for list-typed fields).

When platform automation matures and we want fully scripted onboarding, the right move is to model the outpost's full Application list as a single owned blueprint. Until then: one UI click per workload.

## Validation

Authentik validates blueprints on apply and reports parse/schema errors in **System → Blueprints**. If a blueprint fails, fix and retry — failed blueprints don't partially apply.

Schema reference: <https://goauthentik.io/blueprints/schema.json> (loadable into editor for YAML completion).
