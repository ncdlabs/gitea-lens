# Deploy Gitea Lens — reference

## Cluster facts

| Item | Value |
|------|--------|
| Namespace / release | `gitea-lens` |
| Chart | `deploy/helm/gitea-lens` |
| Values | `deploy/helm/gitea-lens/values-k3s-home.yaml` |
| Image | `git.ncdlabs.com/ncdlabs/gitea-lens:<tag>` |
| Pull secret | `gitea-registry` |
| Lens URL | `https://lens.ncdlabs.com` |
| Gitea URL | `https://git.ncdlabs.com` |
| Runtime Containerfile | `deploy/docker/Containerfile.runtime` |

Live tag and gotchas: repo root `PROJECT_SHARED_STATE.md`.

## Build context layout

`Containerfile.runtime` does `COPY lens /usr/local/bin/lens`. Build context should be a directory that contains the linux/amd64 binary named `lens` (e.g. `.build-image/lens`).

## Smoke — extended

```bash
# Ready replicas
kubectl -n gitea-lens get deploy gitea-lens -o wide

# Endpoints
kubectl -n gitea-lens get ingress,certificate

# Optional: port-forward if ingress DNS is broken from operator machine
kubectl -n gitea-lens port-forward svc/gitea-lens 18090:8090
# then curl http://127.0.0.1:18090/api/v1/system/status
```

`GET /api/v1/system/status` may require auth depending on config; treat **5xx** and **crash loops** as fail. Connection refused / 502 after rollout timeout = fail.

## Helm history tips

```bash
helm history gitea-lens -n gitea-lens
helm get values gitea-lens -n gitea-lens
helm get manifest gitea-lens -n gitea-lens | head
```

## Companion UI markers

See `integrations/gitea/README.md`:

- `<!-- BEGIN GITEA-LENS -->` / `<!-- END GITEA-LENS -->`
- `<!-- BEGIN GITEA-LENS-TABS -->` / `<!-- END GITEA-LENS-TABS -->`

Custom path on k3s3 hostPath: typically `/var/lib/gitea/custom` (templates under `templates/custom/`).

## What optional B does / does not do

**Does:** write/update marker-bounded snippets in `extra_links.tmpl` and `extra_tabs.tmpl`; restart Gitea so templates reload.

**Does not:** upgrade Gitea; change OAuth, webhooks, Actions, or `app.ini`; remove admin content outside markers.
