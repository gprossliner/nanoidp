# Helm Chart - Implementation Plan

Tracking doc for [issue #327](https://github.com/cdelmonte-zg/nanoidp/issues/327)
(provide a Helm chart for Kubernetes deployment). This file is a working plan,
not published documentation, remove or fold relevant bits into `charts/nanoidp/README.md`
once the feature ships, per the `docs/plans/auto-login.md` precedent (#318).

## Design contract (from maintainer, issue #327, two rounds of comments)

1. Single replica only. No `replicaCount` value in `values.yaml` at all,
   hardcode 1 in the Deployment, `strategy: Recreate` so a rollout never runs
   two pods at once. Reason: authorization codes, refresh-token families,
   revocations and the audit log are in-process state with no coordination
   between instances (VISION.md, "Production persistence and distributed
   state"). README states more than one replica is not supported.
2. Versioning: a single `version` field in `Chart.yaml`, no `appVersion`.
   Chart `version` equals the nanoidp release it ships, so `helm ls` answers
   "which nanoidp is this" on its own. Escape hatch, and the reason a second
   field isn't needed: a chart-only fix can still patch-bump `version` while
   leaving the default `image.tag` in `values.yaml` pinned to the last app
   release, both documented in the chart README as a stated convention with
   a stated exception. Consequence outside this PR: the maintainer's release
   checklist gains a bump of both `Chart.yaml` `version` and the default
   `image.tag` in `values.yaml`, since the tag is no longer derived from
   `.Chart.AppVersion`.
3. Configuration as whole files, not structured values. Top-level key
   `configFiles` (not `config`, to leave room for a later structured/
   generated mode without a naming clash), with `configFiles.users` and
   `configFiles.settings` as multiline strings written verbatim into a
   generated `Secret` (`Secret`, not `ConfigMap`, `settings.yaml` carries
   client secrets). `configFiles.existingSecret`, when set, skips the
   generated Secret and mounts the named one instead (same
   `users.yaml`/`settings.yaml` keys expected inside it).
   Constraint on the generated files: `config_version` is read before
   placeholder expansion (`config.py`), so it must be a literal integer,
   never `${VAR}`, and both files must declare the same value or the load
   stops with an explicit error. Simplest correct choice: omit
   `config_version` from both generated files.
4. Read-only config mount: acknowledged as a real limitation, not fixed in
   this PR. A save attempt from the UI or MCP against the mounted Secret
   surfaces whatever OS error the write hits (see `_cross_process_lock`,
   `config_writer.py`, no dedicated detection or UI adaptation exists
   today). Same limitation `docker-compose.yml`'s `:ro` config mount already
   has. README documents the failure mode so a chart user recognizes it
   instead of filing it as a nanoidp bug.
5. Signing keys: no PVC for `keys_dir` in this PR. The RSA keypair is
   regenerated on every pod restart without persistent storage
   (`CryptoService._ensure_keys()`, `services/crypto.py`), invalidating every
   previously issued token and the SAML cert. Acceptable for a dev IdP,
   documented in the README. `keys.existingSecret` (mount a pre-generated
   keypair) is the later shape, out of scope here.
6. Issuer behind the Ingress: `configFiles.settings` is written verbatim, so
   the chart cannot fill `oauth.issuer` itself. The chart auto-injects two
   env vars whenever `ingress.host` is set: `INGRESS_HOST` (bare hostname)
   and `INGRESS_URL` (scheme derived from whether `ingress.tls` is set, the
   same field that already decides whether the chart renders a `tls:` block
   on the Ingress, not a guess). The example settings blob uses
   `oauth.issuer: ${INGRESS_URL}`, relying on nanoidp's own
   `serialization.expand_env_vars` placeholder expansion (independent of
   Helm). Verified safe: an unset variable expands to an empty string, and
   the issuer validator (`models.py`) rejects anything not starting with
   `http://`/`https://`, so a missing `INGRESS_URL` is a loud boot failure,
   not a silently broken issuer. README also documents
   `oauth.issuer_from_request` + `oauth.issuer_from_proxy_headers` +
   `oauth.issuer_allowlist` as the supported alternative for topologies
   where TLS terminates upstream of the Ingress.
7. Pod Security Standard: the image has no `USER` in the Dockerfile and runs
   as root today. Chart targets the `baseline` Pod Security Standard, not
   `restricted`, and documents this in the README. No `securityContext`
   hardening in this PR that would break the container as-is (e.g. no
   `runAsNonRoot: true`). Making the image non-root is a separate nanoidp-
   side change (Dockerfile `USER`, `/app/keys` and `/app/config` ownership),
   explicitly out of scope here, the maintainer owns that decision.
8. Probes: both `/health` and `/api/health` return `{"status": "ok"}`
   ungated by `management_secret`/`require_ui_login` under every
   configuration, either works for liveness/readiness. Prefer a
   `startupProbe` over a generous `initialDelaySeconds` on the liveness
   probe, since first boot does RSA key generation.
9. Example `values.yaml` must not ship an active security hole: the
   original draft combined `login.mode: persona` with `auto_login: true`
   behind a real Ingress hostname, meaning anyone reaching the URL is
   auto-logged-in as an ADMIN user with no credentials at all (the same
   concern #170 already applies to every example under `examples/`, which
   pin the issuer to loopback and leave `auto_login` commented out). The
   shipped example uses normal password login; `mode: persona` and
   `auto_login: true` appear only as a commented-out opt-in with a one-line
   warning beside them. Chart README carries the same warning: nanoidp is a
   test IdP, an Ingress belongs on an internal cluster or behind an auth
   proxy, not on the public internet.
10. Chart default is `ingress.create: false` with an empty `host`, so a bare
    `helm install` with no values never renders an Ingress for a hostname
    the installer doesn't own. The example `values.yaml` can show it
    enabled.
11. Scope for v1, nothing beyond it: Deployment (single replica), Service,
    optional Ingress, `image`/`tag`/`pullPolicy`, `resources`,
    labels/annotations, liveness/readiness on `/api/health`, `env`/`envFrom`
    (for `NANOIDP_MANAGEMENT_SECRET` and `${VAR}` placeholders), the config
    Secret with `existingSecret`. No autoscaling, PVC, PodDisruptionBudget,
    NetworkPolicy or per-field configuration.
12. Chart lives under `charts/nanoidp`. CI runs `helm lint` plus
    `helm template` against two or three example `values.yaml` files on
    every PR. OCI push to `ghcr.io` happens on a separate trigger (release
    tag or workflow_dispatch), not on every PR, maintainer owns that
    workflow. No changes under `src/` in this PR.
13. **Not requested by the maintainer, proposed addition (`#327-addition`,
    call out in the PR description for confirmation, not blocking):** ship
    a `values.schema.json` alongside `values.yaml`. Helm validates the
    coalesced values against it on `helm install`/`upgrade`/`template`/
    `lint`, client-side, before any template renders, so it is real
    fail-fast validation, not just documentation metadata. Scope of what it
    can check is limited by design: `configFiles.users`/`configFiles.settings`
    are opaque multiline strings, the schema can only require them to be
    non-empty strings, it cannot validate the YAML content inside them.
    Everywhere else in `values.yaml` (`ingress.*` shape, `image.*`,
    `resources`, `env`, `configFiles.existingSecret` as a string, etc.) is
    fully covered.

## Breakdown into commits (single feature branch / PR)

### 1. Chart scaffold
- [x] `charts/nanoidp/Chart.yaml`: `apiVersion: v2`, `name: nanoidp`, single
  `version` field, set to match the current nanoidp release version in
  our sample (e.g. `3.0.0`), no `appVersion`.
- [x] `charts/nanoidp/.helmignore`, `charts/nanoidp/templates/_helpers.tpl`
  (standard `nanoidp.fullname`, `nanoidp.labels`, `nanoidp.selectorLabels`
  helpers, following Helm chart best practices). Also added
  `nanoidp.mergedLabels`/`nanoidp.mergedAnnotations`, merging chart-managed
  labels, `commonLabels`/`commonAnnotations`, and a resource's own
  `labels`/`annotations` (later wins on key collision), verified at
  runtime against a throwaway template before being removed.
- [x] `charts/nanoidp/values.yaml` skeleton: `image.repository`/`tag`/
  `pullPolicy`, `resources`, `podLabels`/`podAnnotations` (kept flat at
  root, matching `resources`/`env`/`envFrom`, they describe the one
  always-present pod template, not a separate optional resource, so they
  follow the near-universal Helm convention rather than nesting under a
  `pod:` key), `commonLabels`/`commonAnnotations` (applied to every
  resource), `service` block (`type: ClusterIP`, `labels`, `annotations`),
  `ingress` block (`create: false`, `host: ""`, `labels: {}`,
  `annotations: {}`, `tls: {}`), `env: []`/`envFrom: []`, `configFiles`
  block (`existingSecret: ""`, `users: ""`, `settings: ""`).
- [x] `charts/nanoidp/values.schema.json`: types and required fields for
  every key above (`ingress.create` boolean, `service.type` enum,
  `image.repository`/`tag` strings, `configFiles.users`/`settings`/
  `existingSecret` strings, no validation of the YAML inside the
  multiline string fields, that's out of schema's reach by design). Keep
  in sync with `values.yaml` by hand for v1, no codegen tool introduced
  in this PR.
- [x] Tests: `helm lint` (already planned under CI) exercises schema
  validation implicitly; add one `helm template` case with a deliberately
  wrong type (e.g. `ingress.create: "yes"` as a string) asserting Helm
  rejects it, so the schema's presence is itself pinned by a test.

### 2. Deployment + Service
- [x] `templates/deployment.yaml`: single replica hardcoded (no
  `replicaCount` value read anywhere), `strategy.type: Recreate`,
  `image`/`tag`/`pullPolicy` from values (tag defaults to `.Chart.Version`,
  not `.Chart.AppVersion`, since there is no `appVersion`), `env`/`envFrom`
  passthrough, `resources`, pod labels/annotations, `startupProbe` +
  `livenessProbe`/`readinessProbe` on `/api/health`. Deployment's own
  metadata uses `nanoidp.mergedLabels`/`nanoidp.mergedAnnotations` (with
  no per-resource `extra`, just chart-managed + `commonLabels`/
  `commonAnnotations`); the pod template keeps `podLabels`/
  `podAnnotations` as-is (selector labels never merged with user input).
  No `securityContext` hardening that assumes non-root. Probe timing:
  conservative defaults, not tuned against measured startup, nanoidp has
  no known history of a slow boot, e.g. `startupProbe` with
  `periodSeconds: 2`, `failureThreshold: 15` (up to ~30s to become ready
  before liveness takes over), `livenessProbe`/`readinessProbe` with
  `periodSeconds: 10`. Subject to the usual review round.
- [x] Auto-inject `INGRESS_HOST`/`INGRESS_URL` env vars into the container
  when `.Values.ingress.host` is set; `INGRESS_URL` scheme is `https` when
  `.Values.ingress.tls` is non-empty, `http` otherwise.
- [x] `templates/service.yaml`: `service.type` (default `ClusterIP`) on
  the app's port, metadata via `nanoidp.mergedLabels`/
  `nanoidp.mergedAnnotations` with `.Values.service.labels`/
  `.Values.service.annotations` as the resource-specific `extra`.
- [x] Tests: verified via `helm lint`/`helm template` runs (default
  values: `replicas: 1`, `Recreate`, no `INGRESS_*` vars, tag `3.0.0`;
  `ingress.host` set without `ingress.tls`: `INGRESS_URL` is `http://...`;
  with `ingress.tls.secretName` set: `https://...`; `service.type`/
  `labels`/`annotations` overrides render correctly). No `helm unittest`/
  snapshot harness introduced yet, formal CI-integrated assertions land
  with task 8.

### 3. Config Secret
- [x] `templates/config-secret.yaml`: renders `users.yaml`/`settings.yaml`
  from `configFiles.users`/`configFiles.settings` into a generated
  `Secret`, mounted read-only at `/app/config` (matching the Dockerfile's
  `NANOIDP_CONFIG_DIR` default, no chart-side env override needed);
  skipped entirely when `configFiles.existingSecret` is set, in which
  case that Secret name is mounted instead. Both paths use the same
  `nanoidp.configSecretName` helper for the name, so the generated
  Secret's own name and the volume's `secretName` can never drift apart.
- [x] Confirm neither generated file's example content sets
  `config_version` (per the design contract's constraint): the
  `values.yaml` defaults are empty strings, no example content ships yet,
  that lands with task 7.
- [x] Tests: verified via `helm template`, with `configFiles.users`/
  `settings` set the Secret renders with both keys and the Deployment
  mounts its generated name read-only at `/app/config`; with
  `configFiles.existingSecret` set, no `Secret` object renders at all and
  the volume's `secretName` is the existing one instead.

### 4. Ingress
- [x] `templates/ingress.yaml`: rendered only when `ingress.create: true`,
  using `ingress.host`, `ingress.tls` (a single `{secretName: ...}` object,
  built into the standard k8s `spec.tls` list with `ingress.host` as its
  one entry), and metadata via `nanoidp.mergedLabels`/
  `nanoidp.mergedAnnotations` with `.Values.ingress.labels`/
  `.Values.ingress.annotations` as the extra. Added a guard beyond what
  the plan asked for: `ingress.create: true` with an empty `ingress.host`
  now fails the render with an explicit message, rather than producing an
  Ingress rule with no host (which Kubernetes treats as "match every
  host", a footgun for a single-tenant test IdP that's meant to be
  reachable at exactly one hostname).
- [x] Tests: verified via `helm template`, default renders zero `Ingress`
  objects; `ingress.create: true` with `ingress.host` set renders the
  expected rule with no `tls:` block; adding `ingress.tls.secretName`
  adds the `tls:` block; `ingress.labels`/`ingress.annotations` merge
  correctly; `ingress.create: true` with no `ingress.host` fails the
  render with the guard's message instead of producing an empty host.

### 5. Chart README
- [ ] `charts/nanoidp/README.md` covering, per the design contract above:
  single-replica limitation and why; versioning convention (`version` ==
  nanoidp release) and the chart-only-patch escape hatch; the read-only
  config mount's actual failure mode (a surfaced OS error, not a crash);
  ephemeral signing keys on restart and `keys.existingSecret` as a future
  option; `INGRESS_HOST`/`INGRESS_URL` usage plus
  `issuer_from_request`/`issuer_from_proxy_headers`/`issuer_allowlist` for
  TLS-terminates-upstream topologies; Pod Security Standard `baseline`
  (image runs as root); the security warning that nanoidp is a test IdP
  and an Ingress belongs on an internal network or behind an auth proxy.

### 6. Getting-started docs
- [ ] `book/src/getting-started/install.md`: new `## Helm` section, same
  style as the existing PyPI/Docker/From-source sections, an `helm install`
  example against `oci://ghcr.io/cdelmonte-zg/charts/nanoidp` with a
  minimal `values.yaml` (image tag left at its default, a `configFiles`
  block, `ingress` left disabled), a pointer to `charts/nanoidp/README.md`
  for the full option list and limitations (single replica, ephemeral
  keys, Pod Security Standard).

### 7. Example values.yaml
- [ ] `charts/nanoidp/values-example.yaml` (or embedded in the README):
  normal password login by default; `login.mode: persona` and
  `auto_login: true` present only as commented-out lines with a one-line
  warning; `oauth.issuer: ${INGRESS_URL}`; a client secret sourced via
  `env`/`secretKeyRef` rather than inlined, to avoid modeling a
  committed plaintext secret.

### 8. CI
- [ ] New workflow `.github/workflows/helm.yml`, matching this repo's
  existing conventions (`tests.yml`, `docker.yml`, `publish.yml`): trigger
  on `pull_request: branches: [main, "stack/**"]` and
  `push: branches: [main]`, no path filter (none of the existing workflows
  scope by changed-path either, so a helm-only PR gets the same CI
  visibility as any other). `runs-on: ubuntu-latest`, install Helm via
  `azure/setup-helm` (or equivalent), then `helm lint charts/nanoidp` and
  `helm template charts/nanoidp` against two or three example values
  files.
- [x] Not added as an actual file in this PR, since the maintainer said
  he'd own this side of the release checklist ("I will handle that
  side"). He runs that workflow, so we have no influence over its shape,
  left entirely out of this plan.

- [x] OCI registry path: resolved as `oci://ghcr.io/cdelmonte-zg/charts/nanoidp`,
  the maintainer's suggested layout from the first round of comments on
  #327 ("the more common layout... but I do not mind either"), preferred
  over the originally proposed `ghcr.io/cdelmonte-zg/nanoidp-chart`.


## Closing note: OCI publish workflow

Not part of this PR, the maintainer runs that workflow himself and we have
no influence over its shape, so it stays out of this plan entirely. What
we do provide is the example: `Chart.yaml`'s `version` set to match the
current nanoidp release (`3.0.0`), demonstrating the versioning convention
agreed in the design contract above, chart version equals the nanoidp
release it ships, for whichever release automation the maintainer builds
around it.
