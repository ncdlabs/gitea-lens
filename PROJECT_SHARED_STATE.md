# PROJECT_SHARED_STATE

**Last updated:** 2026-09-22

## Architecture

- **Product:** Gitea Lens — self-hosted CI/CD and PR operations console for Gitea.
- **Shape:** Go single-binary (`cmd/lens`) + embedded React/Vite SPA (`internal/server/ui/dist`); SQLite default; Gitea via API + webhooks.
- **Boundaries:** `internal/forge` (+ `gitea`) · `internal/store` · `internal/api` · `internal/sync` · `internal/webhooks` · `internal/auth` / `internal/authz` · `internal/attention` · `internal/workflows` · `internal/realtime` (SSE) · `internal/metrics` · `internal/ratelimit` · `internal/settings` · `internal/uiinstall`.
- **Auth:** Gitea OAuth Authorization Code + PKCE (`/api/v1/auth/login` → `/api/v1/auth/callback`) plus optional bootstrap password. ACL table `user_repository_access` enforced for non-bootstrap users; periodic ACL refresh (`auth.acl_refresh_interval`, default 6h) for users with decryptable OAuth tokens. Sync does not auto-grant ACL.
- **CSRF:** Double-submit cookie `lens_csrf` (non-HttpOnly) + header `X-CSRF-Token` on state-changing `/api/v1` writes (bootstrap login, logout, sync-repos, settings PUT). Issued via `/api/v1/ui-config` and rotated on session create / `/auth/me`. Exempt: OAuth GET callback, webhooks, health, metrics, GETs.
- **Attention:** Discrete PRD §10 rules with severities `critical` / `warning` / `waiting` only; legacy `open_pull_request` fingerprints resolved on evaluate. Periodic sweep (~10m). `attention.long_running_after` (default 2h). Runner-unavailable rule is a no-op until forge exposes signals.
- **Settings:** `GET/PUT /api/v1/settings` — editable runtime overrides (instance name, sync history days, long-running threshold, retention windows, **`server_external_url`**) plus **DB-backed Gitea integration** (URL, service token, webhook HMAC, private-network/unsigned flags, OAuth client id/secret) and `setup_completed`, all in `app_settings` (single row). File/env remain defaults until UI/DB wins on save; cluster Secret/env still optional default for k3s. Secrets are write-only (`*_configured` booleans on GET; empty secret on PUT = leave unchanged; `clear_*` flags clear). PUT is bootstrap-admin + CSRF only; applies live to sync/auth/webhooks/attention/retention (external URL live-applies to OAuth redirect + webhook delivery). **Setup wizard** at `/setup` for bootstrap admins when `setup_completed` is false (gated in `App.tsx`); steps: Connect → Validate → Finish. Connect collects Gitea URL/token + Lens public URL (`POST /api/v1/setup/check-gitea-url` on blur) and continues; Validate **auto-starts** `POST /api/v1/setup/test-connection` (connectivity/permissions, system hooks + OAuth apps as warn-if-missing) and returns webhook + OAuth app previews. After checks: webhook confirm modal (Create Webhook / I'll Add It in Gitea), then OAuth confirm modal (**Create OAuth App** via `POST /user/applications/oauth2`, **I'll Configure It** with step-by-step Gitea instructions + paste client id/secret, or **Skip OAuth** for bootstrap-only), then Finish. `POST /api/v1/setup/create-webhook`; `POST /api/v1/setup/create-oauth`; `POST /api/v1/setup/complete`. First integration persist is via create-webhook (URL/token/secret); OAuth credentials via create-oauth when chosen. Unsigned webhook toggle and secret/OAuth edits also under Settings. UI: tabbed `/settings` (**Preferences** / **Integration** / **Status**, hash deep-links) with dirty Cancel/Save and unsaved tab dots. **Local skip:** `dev.allow_skip_setup` / `LENS_ALLOW_SKIP_SETUP` (defaulted true by `scripts/dev-start.sh`) exposes **Skip Setup** in the wizard top bar via `ui-config.allow_skip_setup`, and when set also returns `ui-config.dev_bootstrap_password` so the login screen prefills the bootstrap password; never enable on k3s/production.

## Decisions

- Follow `docs/prd-spec.md` + `docs/implementation-plan.md`.
- Module path: `github.com/ncdlabs/gitea-lens`.
- Stack: chi, goose (embedded SQL in `migrations/`), hand-written store SQL (sqlc deferred), Vite/React/TanStack Query.
- **UI:** flat minimal ops-console shell (no glow backgrounds / heavy card shadows); IBM Plex; themes system / light / dark / gruvbox / terminal via `data-theme` tokens in `web/src/styles/app.css`. **Branding:** official theme-matched logos/marks in `web/src/assets/branding/` (`Brand` component — logo lockup expanded/login/setup, mark when rail collapsed); favicon + apple-touch from night mark in `web/public/`. Shell-refresh light/dark tokens must not bind to `:root` (that stomps gruvbox/terminal on `<html>`). Gruvbox = morhetz dark medium (`#282828` / `#ebdbb2` / accent `#fe8019`); Terminal = phosphor CRT (`#0a0e0a` / `#b8f0b8` / accent `#39ff14`) with ANSI/ASCII graphics (mono glyphs, `[####----]` bars, ASCII charts/DAG instead of SVG/ReactFlow); both declared after light/dark overrides. OAuth users: `/api/v1/auth/me` may include mapped `theme` from Gitea `GET /user/settings` (only `gitea-light`→light, `gitea-dark`→dark, `gitea-auto`→system); custom/unknown Gitea themes fail closed (no sync). Manual ThemePicker sets `lens-theme-manual` and stops syncing. Requires stored OAuth token (`LENS_ENCRYPTION_KEY`). **Shell account menu:** avatar + login live at the bottom of the left rail; click opens an upward flyout with Themes (`ThemePicker`), Settings, and Log Out (Settings removed from primary nav). List pages share a table/card view toggle (`lens-view-mode` in localStorage) plus a filter flyout (`ListFilter` / `ListControls`) to the left of the toggle — icon expands left into a search field whose icon becomes Apply. Server `q` on repositories, PRs, workflow runs, and attention. **Forms:** text/password/url/number input names use `placeholder` + `aria-label` (no external `<label>`), except when placeholder is not feasible (e.g. retention day grid with always-filled side-by-side numbers; checkbox text; non-input captions like Redirect URI). **Dashboard** (`/`) is the home route; summary metrics are time-scoped via `GET /api/v1/summary?days=` (allowlist 0/1/7/30/90, default **0** / Now; `0` = current snapshot with no lookback); range control persists last choice in `lens-dashboard-range-days`. Repositories count is inventory (not ranged); open PRs / attention / failed / running respect the window (`Now` = current open/running state; failed counts only attention-linked failures). **Dashboard trends/breakdowns** use `GET /api/v1/stats?days=` (same allowlist; `0` returns current-state breakdowns only — no day series), ACL-scoped; charts are lightweight SVG/CSS (`StackedAreaChart`, `BarList`, `StatCallout`) — no chart library. Day-series JSON field is `day` (not `date`).
- **PR CI state:** populated from Gitea combined commit status (`/commits/{sha}/status`) on sync, with fallback from indexed workflow runs; live updates on `workflow_run` webhooks; shown as pass/fail/pending badges on the Pull Requests tab.
- **Pipelines UI:** `/pipelines` groups runs by action (`repo_full` + `workflow_path`, fallback name); expand a group to list individual runs, then open `/pipelines/:id` for run detail.
- **Active Actions flyout:** shell control near Sync/account; badge = in-flight count; panel lists `queued`/`waiting`/`running` runs from `GET /api/v1/workflow-runs/active` with job progress (completed/total jobs) and step progress when `steps_json` is present. Open state persists in `sessionStorage` (`lens-actions-flyout-open`) across SPA navigations and reloads; closes via toggle, Escape, or Pop Out (not outside click — that was dismissing on nav). **Pop Out** opens `/actions-popout` in a named browser window. `workflow_job` webhooks persist steps and publish SSE (`workflow_job`) like `workflow_run`.
- No Redis; SSE not WebSockets; Gitea-only forge; compose filename `compose.yaml`.
- Logs fetched on demand; Prometheus `/metrics` requires auth (same session cookie as API) and exposes the 11 PRD §41 series (labels: status/event only — never repo names).
- **Webhook HMAC:** Fail closed when `gitea.url` is set and secret empty unless `gitea.allow_unsigned_webhooks` / `LENS_WEBHOOK_ALLOW_UNSIGNED=true`. Secrets via `LENS_WEBHOOK_SECRET` / `LENS_WEBHOOK_SECRET_FILE` (and longer aliases). Unreadable `*_FILE` paths fail config load (no silent clear).
- **Integrity:** Upserts COALESCE nil timestamps; PR rejects older `updated_at`; runs accept greater `run_attempt` or same attempt with non-regressing status; open-PR sync closes numbers absent from open list; soft-delete only rows with `last_synced_at < syncStart`; webhook `processing` reaper (~5m).
- **Proxy prefix:** Strip only `PathPrefix()` from `external_url`; do not trust client `X-Forwarded-Prefix`.
- **Encryption:** `LENS_ENCRYPTION_KEY` (min 16 chars → SHA-256 AES key) required to persist integration secrets and OAuth tokens in DB; seal fail-closed without key; decrypt fail-closed when key set.
- **Rate limits:** In-process per-IP limits on bootstrap login, OAuth login start, and webhook POST. Forwarded client IPs honored only when peer is in `server.trusted_proxies` / `LENS_SERVER_TRUSTED_PROXIES` (chi RealIP not used).
- **OAuth redirect:** only same-app relative paths (`auth.SafeRedirectPath`); absolute/`//` URLs dropped. Login `bootstrap` is reserved (OAuth cannot inherit bootstrap-admin). Users upserted by `(instance_id, gitea_user_id)`.
- **SSE:** `/api/v1/events` filters by `authz.CanAccessRepo` (bootstrap admins see all). Event types include `workflow_run`, `workflow_job`, `pull_request`.
- **OAuth tokens:** refresh_token grant used when access token expiry is within 2m; ACL refresh uses refreshed token when available.
- **Installer:** `scripts/install.sh` (interactive or `--config` + `--non-interactive`); writes gitignored `.env` + `config.yaml`; Compose default, `--method binary` optional.
- **k3s-home deploy:** namespace `gitea-lens`, Helm chart `deploy/helm/gitea-lens`, values `values-k3s-home.yaml`.
- **Image:** `git.ncdlabs.com/ncdlabs/gitea-lens:0.1.14` (linux/amd64; built via host cross-compile + `deploy/docker/Containerfile.runtime` because QEMU `go build` SIGSEGVs). Tag lives in `deploy/helm/gitea-lens/values-k3s-home.yaml` (`pullPolicy: IfNotPresent` — bump tag on each ship). Cluster Secret `gitea-lens/gitea-lens` must include `LENS_WEBHOOK_SECRET` (required at startup when `LENS_GITEA_URL` is set).
- **URL:** `https://lens.ncdlabs.com` (Traefik + cert-manager `letsencrypt-cloudflare-production`; Tailscale private-ingress VIP `100.125.125.244`).
- **Gitea:** `https://git.ncdlabs.com` (1.25.5, hostNetwork on k3s3). System webhook id `1` → `https://lens.ncdlabs.com/api/webhooks/gitea`. OAuth app name `Gitea Lens` (user apps id `4`), redirect `https://lens.ncdlabs.com/api/v1/auth/callback`.

## References

- Docs index: `docs/README.md` (architecture, features, API, auth, deploy, ops)
- Wiki clone (sibling): `/Users/lou/git/gitea-lens.wiki` — push after first GitHub wiki page exists
- Spec: `docs/prd-spec.md`
- Plan: `docs/implementation-plan.md`
- Config example: `config.example.yaml`
- Installer: `scripts/install.sh`, `install.example.yaml` → `./scripts/install.sh` or `make install`
- Run local: `npm run start` / `restart` (prints App+API URLs, bootstrap user/password, opens browser; API `:8090` + Vite `:5173`; default password `lens-local` if unset; login prefills that password via `dev_bootstrap_password`; prefers `config.yaml`); `npm run stop`
- Binary-shaped local: `make frontend && make build-go` then `./bin/lens serve`
- Compose: `podman compose -f compose.yaml up --build`
- Tests: `go test ./...`
- Cluster: `helm upgrade --install gitea-lens deploy/helm/gitea-lens -n gitea-lens -f deploy/helm/gitea-lens/values-k3s-home.yaml`
- Deploy skill: `.cursor/skills/deploy-gitea-lens/` (build/push/Helm + smoke + mandatory rollback; optional Gitea `install-ui` only when explicitly requested)
- Secrets (live, not in git): `gitea-lens/gitea-lens` (token, bootstrap, oauth, webhook secret, encryption key), `gitea-lens/gitea-registry`

## Known gotchas

- Default `GOPATH` symlink `/Users/lou/go` may point at an unavailable volume; use `GOPATH`/`GOMODCACHE` under `~/Library/Caches` if `go mod` fails with `mkdir /Users/lou/go`.
- Embed requires `internal/server/ui/dist` (populated by `make frontend` from `web/dist`).
- Private/lab Gitea URLs need `LENS_GITEA_ALLOW_PRIVATE_NETWORK=true` (SSRF guard fails closed on DNS errors; dial pins resolved IPs; HTTP(S)_PROXY ignored for forge client).
- Empty repo sync does **not** soft-delete the catalog (zero-result reconcile is a no-op for deletes).
- Local compose leaves `LENS_AUTH_BOOTSTRAP_PASSWORD` empty by default (bootstrap login disabled until set).
- Saving integration secrets via Settings/Setup requires `LENS_ENCRYPTION_KEY`; env/file secrets still work as in-memory defaults without DB cipher writes.
- `LENS_ALLOW_SKIP_SETUP` is rejected when `server.external_url` is non-local (blocks `dev_bootstrap_password` exposure on public URLs).
- Behind Traefik/Ingress, set `LENS_SERVER_TRUSTED_PROXIES` to the proxy pod CIDR(s) so auth rate limits key on the real client IP.
- Gitea Actions run/job JSON shapes vary; client accepts wrapped or flat arrays and degrades on 404.
- **Gitea 1.25 Actions `path`:** often `ci.yaml@refs/heads/main` (not a repo file path). Lens normalizes to the filename and tries `.gitea/workflows/` then `.github/workflows/` when fetching YAML for the Workflow Graph.
- Subpath deploys must set `server.external_url` with the correct path; client-forwarded prefix is ignored.
- **Image build:** full multi-stage Dockerfile amd64 under Lima/QEMU crashes during `go build`; use cross-compile + `Containerfile.runtime`.
- **Registry pull:** namespace needs `gitea-registry` dockerconfig for `git.ncdlabs.com` (portal copy was stale; recreate with a token that can pull `ncdlabs/gitea-lens`).
- **DNS:** Private hosts live in Pi-hole v6 `dns.hosts` inside `/etc/pihole/pihole.toml` (not `custom.list`). Entry: `100.125.125.244 lens.ncdlabs.com`. Also Cloudflare DNS-only A → `100.125.125.244` (for Chrome Secure DNS / 1.1.1.1). Gitea host `k3s3` has `/etc/hosts` pin for webhooks. After adding hosts, restart `pihole-FTL` and flush Mac DNS (`sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder`).
- Connected Gitea is **1.25.5** (plan target family ~1.26); Actions APIs present with capability detection.

## Policies

- Do not remove documented product/debug features without explicit approval.
- No ncdLabs-hosted service dependencies.
- Never trust client-side repository filtering for authorization.
- Ask before major scope expansion (write ops, multi-forge, Redis, WebSockets).

## Known constraints

- Target Gitea API family ~1.26; capability JSON stored on instance; Actions features degrade when APIs missing.
- V1 read-first; rerun/cancel deferred. Intentional V1 service-token log fetch remains as-is.
- Postgres: insert paths use `RETURNING id` (bootstrap user, webhook events); broader Postgres production readiness still incomplete vs SQLite.
- React Flow DAG optional polish; accessible list fallback ships.
- User-editable attention severity UI deferred.

## Environment notes

- Default listen `:8090`; data dir `/data` in containers.
- Local containers: Podman / `podman compose` (nerdctl).
- Webhook endpoint: `POST /api/webhooks/gitea`.
- install-ui markers: `<!-- BEGIN GITEA-LENS -->` / `<!-- BEGIN GITEA-LENS-TABS -->` under `/var/lib/gitea/custom/templates/custom/` on k3s3 (hostPath for `gitea` Deployment). Links use `target="_blank"`; Gitea must be restarted after template changes (`kubectl -n gitea rollout restart deployment/gitea`).
- Bootstrap password and OAuth client secret live only in cluster Secret `gitea-lens/gitea-lens` (and `/tmp/gitea-lens-deploy/` on the operator machine — purge when done).
- Initial sync (2026-09-20): 96 repos, 296 open PRs, 3809 runs, 6680 jobs.
