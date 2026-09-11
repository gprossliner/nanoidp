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
  `version` field, no `appVersion`. Committed as `0.0.0`, a placeholder,
  not a real release version, see the closing note at the end of this
  plan: the real version is set at publish time from the git tag, via
  `helm package --version`, since Helm requires the field to be a valid,
  non-empty SemVer string and there's no way to omit or defer it.
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
  values: `replicas: 1`, `Recreate`, no `INGRESS_*` vars, tag defaulting
  to `.Chart.Version`; `ingress.host` set without `ingress.tls`:
  `INGRESS_URL` is `http://...`; with `ingress.tls.secretName` set:
  `https://...`; `service.type`/
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
- [x] `charts/nanoidp/README.md` covering, per the design contract above:
  single-replica limitation and why; versioning convention (`version` ==
  nanoidp release) and the chart-only-patch escape hatch; the read-only
  config mount's actual failure mode (a surfaced OS error, not a crash);
  ephemeral signing keys on restart and `keys.existingSecret` as a future
  option; `INGRESS_HOST`/`INGRESS_URL` usage plus
  `issuer_from_request`/`issuer_from_proxy_headers`/`issuer_allowlist` for
  TLS-terminates-upstream topologies; Pod Security Standard `baseline`
  (image runs as root); the security warning that nanoidp is a test IdP
  and an Ingress belongs on an internal network or behind an auth proxy.
  Also covers installation and `configFiles`/`existingSecret` usage,
  since those didn't have a home elsewhere yet.

### 6. Getting-started docs
- [x] `book/src/getting-started/install.md`: new `## Helm` section, same
  style as the existing PyPI/Docker/From-source sections, the `helm
  install` command against `oci://ghcr.io/cdelmonte-zg/charts/nanoidp`,
  then a pointer straight to `charts/nanoidp/README.md` for the working
  example and the full value/limitation list, no duplicated `values.yaml`
  snippet or restated limitations here, that's the README's job, this
  page only mentions the install path exists.

### 7. Example values.yaml
- [x] Folded into task 5's `charts/nanoidp/README.md`, as a "Complete
  example" section rather than a separate `values-example.yaml` file:
  normal password login by default (`login.mode: persona`/`auto_login:
  true` shown only as commented-out lines with a one-line warning),
  `oauth.issuer: ${INGRESS_URL}`, one real registered OAuth client, and
  its `client_secret` sourced via `env`/`secretKeyRef` against a Secret
  the user creates themselves (`kubectl create secret generic ...`)
  rather than inlined, with a note that the same Secret can be imported
  into the client application's own OIDC configuration. Verified via
  `helm lint`/`helm template` against a copy of the exact example.

### 8. CI
- [x] `.github/workflows/helm.yml`, matching this repo's existing
  conventions (`tests.yml`, `docker.yml`, `publish.yml`): trigger on
  `pull_request: branches: [main, "stack/**"]` and `push: branches:
  [main]`, no path filter (none of the existing workflows scope by
  changed-path either, so a helm-only PR gets the same CI visibility as
  any other). `runs-on: ubuntu-latest`. Worth knowing: per the
  [runner-images Ubuntu 24.04 software list](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md),
  `ubuntu-latest` already ships Helm (3.21.4) and `yq` (4.53.6)
  preinstalled, `kubeconform` does not. We still install Helm explicitly
  via `azure/setup-helm` rather than relying on the runner's version,
  so the exact version is pinned and doesn't drift if GitHub bumps the
  runner image; `kubeconform` is installed as a pinned static Linux
  binary (curled directly from its GitHub release, no cluster and no
  package-manager bootstrap needed), matching the version verified
  locally (v0.7.0). The actual checks live in
  `charts/nanoidp/ci/check.sh`, runnable identically locally (once
  `kubeconform`/`yq` are on `PATH`, e.g. `brew install kubeconform yq`)
  or from CI, so there's one source of truth instead of duplicated
  inline shell:
  - `helm lint`/`helm template` against default values and three fixture
    files under `charts/nanoidp/ci/` (a `ci/` directory of values
    overrides is also the convention Helm's own `chart-testing`/`ct` tool
    looks for, so this stays adoptable by that tool later without
    renaming anything): `values-minimal.yaml` (just enough `configFiles`
    to boot), `values-full.yaml` (the README's complete example: Ingress,
    env-sourced client secret, a real OAuth client),
    `values-existing-secret.yaml` (`configFiles.existingSecret` path).
  - Every rendered manifest set piped through `kubeconform -strict`
    against the real Kubernetes v1.30.0 OpenAPI schema, no cluster
    involved, catches structural mistakes (wrong types, missing required
    fields, invalid `apiVersion`/`kind`) that `values.schema.json` can't,
    since that only validates the *input* values, not the *output*
    manifests. Deliberately no `kind`/real-cluster dry-run: that would add
    real CI time and complexity this small a chart doesn't need.
  - `values.schema.json` re-confirmed enforced (`ingress.create: "yes"`
    as a string must fail).
  - A handful of chart-specific behavioral assertions via `yq`, formalizing
    what was checked by hand during implementation rather than adding a
    `helm-unittest` plugin dependency for v1: single replica, `Recreate`
    strategy, `INGRESS_HOST`/`INGRESS_URL` injection and scheme (both
    `http`, no `ingress.tls`, and `https`, `ingress.tls.secretName` set),
    `existingSecret` skipping the generated `Secret` and being referenced
    on the volume.
- [x] Note for the maintainer: there is a [`setup-kubeconform`
  marketplace action](https://github.com/marketplace/actions/setup-kubeconform).
  Not used here, it's a small, not particularly popular third-party
  action for a one-line `curl` we already control directly and can pin
  precisely; mentioned in case the maintainer prefers it instead.
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
no influence over its shape, so it stays out of this plan entirely.

`Chart.yaml`'s committed `version: 0.0.0` is a placeholder, not the real
version. Helm requires `version` to be set to a valid, non-empty SemVer 2
string (there is no way to omit it or defer it, filed upstream as
[helm/helm#32143](https://github.com/helm/helm/issues/32143)), but the
real chart version must come from the git tag at publish time, not a
value committed to the repo, otherwise every release needs a
`Chart.yaml` edit commit, and the version is either set before the tag
exists or has to be back-filled after, neither of which is tag-friendly.
`0.0.0` is a clearly-a-placeholder value that will never collide with a
real release.

A rough sketch, offered only as an optional starting point, not committed
as an actual file, mirroring `docker.yml`'s tag-triggered publish as closely
as sensible: same `push: tags: 'v*'` trigger, same source of truth
(`github.ref_name`, the pushed tag). No `docker/metadata-action`-style
`type=raw,value=latest` equivalent is needed, an OCI Helm chart registry
has no "latest" tag convention/consumer the way `docker pull image:latest`
does, so only the tag-derived version matters. `github.ref_name` includes
the tag's leading `v` (e.g. `v3.0.0`), which is invalid SemVer for
`--version`, so it's stripped once with a `sed` into a job output before
either publish step uses it:

```yaml
name: Publish Helm Chart

on:
  push:
    tags:
      - 'v*'

jobs:
  publish-chart:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v6

      - name: Set up Helm
        uses: azure/setup-helm@v4

      - name: Chart version from tag
        id: version
        run: echo "version=${GITHUB_REF_NAME#v}" >> "$GITHUB_OUTPUT"

      - name: Package chart
        run: |
          helm package charts/nanoidp \
            --version "${{ steps.version.outputs.version }}" \
            --destination .helm-dist

      - name: Log in to GHCR
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | helm registry login ghcr.io \
            --username ${{ github.actor }} --password-stdin

      - name: Push chart
        run: |
          helm push .helm-dist/nanoidp-*.tgz oci://ghcr.io/cdelmonte-zg/charts
```

## Review round 1 (maintainer, verified live against a kind cluster, k8s 1.36)

Tested for real, not just templated: default values, each of the three
`ci/` fixtures, a self-created `existingSecret`, and one with a
management secret, six installs total, plus namespaces enforcing
`baseline` and `restricted`. Overall assessment: contract from #327
respected point by point, README honest about its limitations,
`ci/check.sh` asserts behavior rather than just rendering successfully.
Three blocking findings, five smaller ones, everything else confirmed
working as documented.

### Blocking

1. **`helm upgrade` with a changed `configFiles` is a silent no-op.** The
   Secret updates, the kubelet refreshes the projected file on disk, but
   nanoidp reads its config once at boot and nothing rolls the pod, so
   the running process and the file on disk disagree until a manual
   `kubectl rollout restart`. Confirmed live: `users.yaml` on disk gained
   `alice` after a `helm upgrade`, `/api/users` kept returning only
   `admin` until the pod was restarted by hand.
2. **The "Complete example" publishes an unauthenticated admin surface.**
   nanoidp's `/api/*` and UI are unauthenticated by default; the example
   renders a real Ingress with none of the two available gates
   (`NANOIDP_MANAGEMENT_SECRET`, `session.require_ui_login`) turned on.
   Confirmed live: `POST /api/users/admin/token` and `POST
   /api/keys/rotate` both returned `200` through the Ingress, no
   credentials. With both gates set: those return `401`, `/api/health`
   and `/.well-known/openid-configuration` stay `200`, so probes/discovery
   are unaffected.
3. **`helm install ./charts/nanoidp` from a clone cannot start.**
   `image.tag` defaults to `.Chart.Version`, the committed `0.0.0`
   placeholder, so a straight `helm install` from a checkout fails with
   an opaque `ImagePullBackOff` (`ghcr.io/...nanoidp:0.0.0` not found).
   The placeholder itself is agreed and not being revisited, just its
   silence.

### Smaller

4. **No `ingress.className`.** Worked in the maintainer's kind cluster
   only because its bundled ingress-nginx runs with
   `--watch-ingress-without-class=true`; ingress-nginx's own chart
   defaults that to `false`, so on a normal cluster this Ingress is
   silently ignored without the deprecated class annotation added by
   hand.
5. **No `securityContext`.** The `baseline`/`restricted` claim is
   correct, verified live (`baseline`: `1/1 Running`; `restricted`:
   `FailedCreate` on `allowPrivilegeEscalation`, capabilities,
   `runAsNonRoot`, `seccompProfile`). Three of those four are
   image-independent and can be set today
   (`allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]`,
   `seccompProfile.type: RuntimeDefault`, confirmed live by patching the
   running Deployment), narrowing `restricted` down to just
   `runAsNonRoot`, the already-agreed image-side change.
6. **Default values render a Secret with two empty config files.**
   `configFiles.users`/`settings` default to `""`, and that empty mount
   shadows the image's own bundled demo config rather than falling back
   to it. A bare install starts, passes its probes, and has zero users
   and zero OAuth clients. Maintainer's three options, any acceptable:
   require non-empty in `values.schema.json`, default to
   `ci/values-minimal.yaml`'s content, or just document the shadowing.
7. **Dead code**: `nanoidp.labels`' `{{- if .Chart.AppVersion }}` branch
   can never fire, there deliberately is no `appVersion`.
8. **No `NOTES.txt`**, `helm install` prints nothing about how to reach
   what it just installed.

### Verified working, no action needed

- `${INGRESS_URL}` expansion end to end: boot log shows `Issuer:
  http://idp.example.com` with `values-full.yaml`, and a full
  authorization code flow through the Ingress returns an ID token with
  matching `iss`/`aud`/`sub`.
- Read-only config mount fails exactly as documented, visibly (a UI
  error alert with the real `Errno 30` message), not silently.
- Signing keys rotate on restart, `kid` changes across a pod delete.
- `configFiles.existingSecret` mounts the external Secret and renders no
  Secret of its own.
- `ingress.create: false` by default; the empty-`ingress.host` `fail()`
  fires as designed.
- Probes hit `/api/health`, stay ungated even with a management secret
  set, `startupProbe` covers first-boot key generation.
- `ci/check.sh` passes locally against Helm 3.19 and kubeconform 0.7.0.

### Maintainer's own notes, nothing for us to do

- The release checklist is simpler than the plan assumed: since
  `image.tag` falls back to `.Chart.Version` and `helm package --version`
  sets that from the tag, a release needs no file edit at all, the
  maintainer will write the checklist that way.
- The publish workflow sketch is a good starting point, the maintainer
  will take it from there.
- Confirmed: drop `docs/plans/helm-chart.md` before merge.

## Round 2: implementation plan for the above

Compacted from the maintainer's 7 numbered findings into 5 stages,
grouped by file overlap and risk profile rather than 1:1 with the
findings list.

### Stage A: Config rollout + cleanup (findings 1, 7)
- [x] `templates/deployment.yaml`: add a `checksum/config` annotation to
  the pod template, computed from the rendered `config-secret.yaml`
  content (the standard Helm idiom:
  `{{ include (print $.Template.BasePath "/config-secret.yaml") . | sha256sum }}`),
  so a `configFiles.users`/`settings` change forces a new pod. Only when
  `configFiles.existingSecret` is unset, the chart cannot see content it
  does not render, so the checksum stays absent (not a fixed/fake value)
  when `existingSecret` is set. `strategy: Recreate` already makes this a
  short gap rather than an overlap, matching the chart's own single-pod
  design.
- [x] README: one line stating that with `configFiles.existingSecret`,
  rolling the pod on a config change is the user's own responsibility
  (e.g. their own checksum annotation, or `kubectl rollout restart`).
- [x] `templates/_helpers.tpl`: delete the dead
  `{{- if .Chart.AppVersion }}...{{- end }}` block in `nanoidp.labels`,
  it can never fire, there deliberately is no `appVersion`. Unrelated to
  the checksum fix, bundled here as a trivial, zero-risk cleanup rather
  than its own commit.
- [x] Tests: `helm template` with two different `configFiles.users`
  values renders two different `checksum/config` annotations; with
  `configFiles.existingSecret` set, no `checksum/config` annotation
  renders at all; rendered labels otherwise unchanged. Added as permanent
  assertions in `ci/check.sh`, full suite passes.

### Stage B: Authenticate the "Complete example" (finding 2)
- [x] README's "Complete example": add `NANOIDP_MANAGEMENT_SECRET` via
  `env`/`secretKeyRef` (same `kubectl create secret` step already there,
  one more key) and `session.require_ui_login: true` in the
  `configFiles.settings` block. Note the verified behavior: mutating
  endpoints return `401`, `/api/health` and
  `/.well-known/openid-configuration` stay `200`.
- [x] `charts/nanoidp/ci/values-full.yaml`: mirror the same two additions,
  so the CI fixture and the README's complete example stay in sync (the
  existing invariant between them).
- [x] No template/chart code changes in this stage, docs and CI fixture
  content only.
- [x] Tests: re-run `charts/nanoidp/ci/check.sh`, add a rendered-output
  assertion that `NANOIDP_MANAGEMENT_SECRET` is present in the
  Deployment's `env` for `values-full.yaml`.

### Stage C: Hardening knobs, `ingress.className` + `securityContext` (findings 4, 5)
- [x] `values.yaml`/`values.schema.json`: add `ingress.className: ""`.
- [x] `templates/ingress.yaml`: render `spec.ingressClassName` only when
  non-empty.
- [x] README: mention `ingress.className`, since ingress-nginx's own
  chart defaults `watchIngressWithoutClass: false`.
- [x] `values.yaml`/`values.schema.json`: add a `securityContext` value
  (container-level: `allowPrivilegeEscalation`, `capabilities`,
  `seccompProfile` are container-scoped fields), defaulting to the three
  fields verified live: `allowPrivilegeEscalation: false`,
  `capabilities.drop: ["ALL"]`, `seccompProfile.type: RuntimeDefault`.
  Exposed as a real value, not hardcoded, so a future non-root image can
  add `runAsNonRoot`/`runAsUser` without a template change.
- [x] README's Pod Security Standard section: note this narrows the
  `restricted` gap down to just `runAsNonRoot`, the already-agreed
  image-side change, not something this chart can close on its own.
- [x] Tests: `helm template` with and without `ingress.className` set;
  default values render the three `securityContext` fields; full
  `ci/check.sh` (kubeconform + assertions) still passes. Real
  `baseline`/`restricted` admission behavior was already verified live by
  the maintainer, not re-provable by `kubeconform`/`yq` alone.

### Stage D: `NOTES.txt` (findings 3, 8)
- [x] `charts/nanoidp/templates/NOTES.txt`: print the resolved image
  (`repository:tag`, tag defaulting to `.Chart.Version`); when the
  effective tag is exactly `0.0.0`, print an explicit warning that this
  is a publish-time placeholder and installing from a git checkout needs
  `--set image.tag=<release>`. No `fail()`, that would force
  `ci/check.sh` to always pass `--set image.tag=...`, more than this
  needs. Also print how to reach the instance: the Ingress URL when
  `ingress.create` is true; otherwise a plain statement that Ingress
  isn't enabled and nanoidp isn't reachable from outside the cluster, no
  `kubectl port-forward` hint, that only actually works for OIDC/SAML
  testing when the browser and the OAuth client are both on the same
  machine as whoever runs the command, narrow enough to not be worth
  suggesting as the general answer. Folds both findings into the same
  piece of work.
- [x] Tests: `helm template` does not render `NOTES.txt` at all, only
  `helm install`/`upgrade` (or `--dry-run=client`) do, discovered while
  testing; used `helm install ci-check <chart> --dry-run=client` instead.
  Default values show the placeholder warning; `--set
  image.tag=v3.0.0` removes it; `ingress.create: true` prints the Ingress
  URL, the default prints the "Ingress is not enabled" message. Added as
  permanent assertions in `ci/check.sh`.

### Stage E: Default `configFiles.users`/`settings`, stop shipping silent emptiness (finding 6)
- [x] Decision: make them required and non-empty via
  `values.schema.json`, conditioned on `configFiles.existingSecret` being
  unset (JSON Schema `if`/`then`, draft-07, which the declared
  `$schema` already targets), consistent with this chart's existing
  "fail loud instead of silently misbehaving" pattern (the Ingress
  empty-host `fail()`, the issuer validator). Verified empirically that
  Helm's schema validator does enforce `if`/`then`: with the conditional
  added, a bare `helm lint` now fails with
  `at '/configFiles/users': minLength: got 0, want 1` (and the same for
  `settings`), while all three `ci/` fixtures still pass unaffected
  (`values-minimal.yaml`/`values-full.yaml` already set both,
  `values-existing-secret.yaml`'s `existingSecret` makes the `if`
  condition false). No fallback needed.
- [x] A bare `helm lint`/`helm template` with no values now fails by
  design; updated `ci/check.sh` accordingly: dropped the old "default
  values" check entirely (redundant with `values-minimal.yaml`'s own
  check in the fixture loop), added an explicit assertion that empty
  `configFiles` is rejected (same shape as the existing
  `ingress.create: "yes"` rejection check), and rebased every downstream
  "default values" behavioral check (`securityContext`, `ingress.className`,
  `NOTES.txt`) onto `values-minimal.yaml` instead of bare defaults, since
  those no longer render at all.
- [x] README: states plainly that `configFiles` is mandatory (unless
  `existingSecret` is set), no further rationale, that has no value for
  the user. `values.yaml`'s own `configFiles` comment keeps the
  rationale (shadows the image's bundled demo config), that one's for
  whoever reads the chart source, not the end user installing it.

### Re-verification
- [x] Full `charts/nanoidp/ci/check.sh` re-run locally after all of the
  above, passes clean.
- [x] The two findings most dependent on real cluster behavior, the
  config-change rollout (stage A) and `securityContext` under
  `restricted` admission (stage C), self-verified live against a kind
  cluster (k8s 1.35.0), using the real published
  `ghcr.io/cdelmonte-zg/nanoidp:latest` image directly, no custom build
  needed, this PR makes no changes under `src/`:
  - Config rollout: installed with `ci/values-minimal.yaml`, pod
    `...-677bd86956-mjzpf`, `/api/users` showed 1 user (`admin`).
    `helm upgrade` with `configFiles.users` adding `alice`, no manual
    `kubectl rollout restart`: pod became `...-c5f74794d-5nt4c` (a new
    ReplicaSet, not just a restart), `/api/users` immediately showed both
    `admin` and `alice`.
  - `securityContext` under `restricted`: installed into a namespace
    labeled `pod-security.kubernetes.io/enforce=restricted`. The only
    admission violation reported was `runAsNonRoot != true`;
    `allowPrivilegeEscalation`, `capabilities`, `seccompProfile` no
    longer appear, confirming the fix narrows `restricted` down to
    exactly the one already-agreed image-side gap, as claimed.
