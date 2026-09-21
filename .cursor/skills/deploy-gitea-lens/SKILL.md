---
name: deploy-gitea-lens
description: >-
  Builds, publishes, and Helm-deploys the current gitea-lens to the ncdlabs
  k3s-home cluster (lens.ncdlabs.com / registry git.ncdlabs.com), runs a smoke
  test, and always prepares a safety rollback. Optional Gitea companion
  install-ui for template links. Use when the user asks to deploy, ship, roll
  out, or update gitea-lens on k3s-home, lens.ncdlabs.com, or git.ncdlabs.com
  companion integration — never for local-only builds unless they also ask to
  deploy.
---

# Deploy Gitea Lens (k3s-home)

Ship the **current workspace** build of Gitea Lens to production-adjacent
k3s-home. **Gitea (`https://git.ncdlabs.com`) is a production system and
critical to operations** — treat every step as reversible.

## Hard rules

1. **Only run when the user explicitly asks to deploy/ship/update Lens** on this cluster.
2. **Capture rollback targets before any cluster or Gitea change.** Do not proceed without them.
3. **Default path = Lens only** (image + Helm). Do **not** touch Gitea templates or restart the `gitea` Deployment unless the user **explicitly** opts into companion UI (`install-ui`).
4. **If smoke fails → roll back immediately**, then report. Do not leave a broken release.
5. Prefer **Podman** (not Docker). Use `podman` / `podman compose`.
6. Never force-push, never delete PVCs/secrets, never change Gitea `app.ini` / OAuth / webhooks as part of this skill.
7. Read `PROJECT_SHARED_STATE.md` first for live image tag, gotchas, and URLs.

## Scope

| Mode | What changes | When |
|------|----------------|------|
| **Default (A)** | Build → push `git.ncdlabs.com/ncdlabs/gitea-lens:<tag>` → Helm upgrade `gitea-lens` | Always for deploy |
| **Optional (B)** | `lens install-ui` marker blocks in Gitea custom templates + Gitea rollout restart | **Only** if user asks (companion links/tabs in Gitea UI) |

Optional B is for companion chrome (nav/tab links to Lens). It is **not** required to ship Lens features.

## Checklist (copy and update)

```
Deploy gitea-lens:
- [ ] Read PROJECT_SHARED_STATE.md
- [ ] Confirm mode: A only / A+B (user must opt into B)
- [ ] Record rollback: helm revision + image tag (+ template backup if B)
- [ ] Build frontend + linux/amd64 binary + runtime image
- [ ] Push image; bump values-k3s-home.yaml tag
- [ ] Helm upgrade
- [ ] Wait for rollout
- [ ] Smoke test Lens
- [ ] If B: install-ui + Gitea restart + smoke Gitea
- [ ] On any failure: execute rollback
- [ ] Report result + rollback command used or still available
```

## 0. Preconditions

- Workspace: gitea-lens repo root.
- Tools: `kubectl` (k3s-home context), `helm`, `podman`, Go, Node.
- Cluster: namespace `gitea-lens`; chart `deploy/helm/gitea-lens`; values `deploy/helm/gitea-lens/values-k3s-home.yaml`.
- Registry: `git.ncdlabs.com/ncdlabs/gitea-lens` (pull secret `gitea-registry` already in ns).
- Public URL: `https://lens.ncdlabs.com`.
- If `go mod` fails on `/Users/lou/go`, set `GOPATH`/`GOMODCACHE` under `~/Library/Caches` (see shared state).

Confirm kube context is the intended home cluster before mutating anything.

## 1. Record rollback (mandatory, first)

```bash
NS=gitea-lens
REL=gitea-lens

helm history "$REL" -n "$NS" --max 5
PREV_REV=$(helm history "$REL" -n "$NS" --max 2 -o json | jq -r '.[1].revision // empty')
CUR_TAG=$(kubectl -n "$NS" get deploy "$REL" -o jsonpath='{.spec.template.spec.containers[0].image}')
echo "ROLLBACK_HELM_REV=${PREV_REV:-<none>}"
echo "ROLLBACK_IMAGE=${CUR_TAG}"
```

Also note the current `image.tag` in `deploy/helm/gitea-lens/values-k3s-home.yaml`.

If **B** is opted in, **before** template edits:

```bash
# On the Gitea data host / via a debug pod that sees the hostPath custom dir —
# backup marker files (paths from PROJECT_SHARED_STATE / integrations/gitea/README.md):
#   .../custom/templates/custom/extra_links.tmpl
#   .../custom/templates/custom/extra_tabs.tmpl
# Copy to a timestamped backup under /tmp or operator machine; keep until smoke passes.
```

Store these values in the chat report so rollback does not depend on memory.

## 2. Choose new image tag

Bump semver patch (or use the version the user specifies). Update:

`deploy/helm/gitea-lens/values-k3s-home.yaml` → `image.tag`

`pullPolicy` is `IfNotPresent` — **always use a new tag** for each ship (do not reuse a tag with different bits).

## 3. Build (cross-compile + runtime image)

Do **not** rely on full multi-stage amd64 `Dockerfile` under QEMU (known SIGSEGV). Use:

```bash
make frontend
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags='-s -w' -o .build-image/lens ./cmd/lens
podman build --platform linux/amd64 -f deploy/docker/Containerfile.runtime -t "git.ncdlabs.com/ncdlabs/gitea-lens:${TAG}" .build-image
```

(`Containerfile.runtime` expects `COPY lens` from the build context dir that contains the binary.)

Optional: `go test ./...` before shipping if the user wants a gate; do not skip if they asked for tests.

## 4. Push + Helm upgrade

```bash
podman push "git.ncdlabs.com/ncdlabs/gitea-lens:${TAG}"
helm upgrade --install gitea-lens deploy/helm/gitea-lens \
  -n gitea-lens \
  -f deploy/helm/gitea-lens/values-k3s-home.yaml
kubectl -n gitea-lens rollout status deploy/gitea-lens --timeout=180s
```

## 5. Smoke test (Lens) — required

Fail closed; on failure go to **Rollback**.

```bash
# TLS + reachability
curl -fsS -o /dev/null -w "%{http_code}\n" https://lens.ncdlabs.com/

# API system status (may be 401 without cookie — still proves routing if not 5xx/000)
curl -fsS -o /tmp/lens-status.json -w "%{http_code}\n" https://lens.ncdlabs.com/api/v1/system/status || true

# Pod health
kubectl -n gitea-lens get pods -l app.kubernetes.io/name=gitea-lens
kubectl -n gitea-lens logs deploy/gitea-lens --tail=80
```

Pass criteria:

- Ingress returns **200** (or expected login HTML) for `/` — not connection error / 502/503 sustained.
- Deployment pods **Ready**.
- Logs show no crash loop / panic on startup.
- If authenticated smoke is available in-session, hit `/api/v1/auth/me` or summary; otherwise unauthenticated checks above are enough.

Details: [reference.md](reference.md).

## 6. Optional B — Gitea companion templates

**Skip unless the user explicitly requested it.**

1. Confirm backups from step 1.
2. Build a local `bin/lens` if needed (`make frontend && make build-go`).
3. Run against the live Gitea **custom** root (hostPath on k3s3 — see shared state):

```bash
./bin/lens install-ui --custom-path /var/lib/gitea/custom --lens-url https://lens.ncdlabs.com
kubectl -n gitea rollout restart deployment/gitea
kubectl -n gitea rollout status deployment/gitea --timeout=180s
```

4. Smoke Gitea: `https://git.ncdlabs.com/` loads; login page or user session OK; no prolonged 5xx.

Rollback B (templates only):

```bash
./bin/lens uninstall-ui --custom-path /var/lib/gitea/custom
# or restore backed-up tmpl files
kubectl -n gitea rollout restart deployment/gitea
```

Prefer `uninstall-ui` when markers are intact; restore file backups if markers were corrupted.

## 7. Rollback (mandatory capability)

### Lens (A)

Prefer Helm revision when available:

```bash
helm rollback gitea-lens "${ROLLBACK_HELM_REV}" -n gitea-lens
kubectl -n gitea-lens rollout status deploy/gitea-lens --timeout=180s
```

If history is insufficient, re-set previous tag in values and `helm upgrade --install` with the prior image tag (cluster must still be able to pull it; tags are immutable per ship).

Re-run Lens smoke after rollback.

### Companion UI (B)

Only if B was applied: `uninstall-ui` or restore backups → restart Gitea → smoke Gitea.

**Never** “fix forward” on a failed smoke without user approval. Default is roll back.

## 8. Report

Tell the user:

- New image tag and Helm revision
- Smoke results (pass/fail)
- Whether B ran
- Exact rollback command still valid (or that rollback already ran)
- Reminder to update `PROJECT_SHARED_STATE.md` image tag if the deploy succeeded

## Anti-patterns

- Reusing an existing image tag with new bits
- Full-Dockerfile amd64 build under Lima/QEMU for this environment
- Restarting Gitea “just in case” without B
- Deploying without recorded `ROLLBACK_*` values
- Leaving a failed rollout without attempting rollback
