# PROJECT_SHARED_STATE

**Last updated:** 2026-09-20

## Architecture

- **Product:** Gitea Lens — self-hosted CI/CD and PR operations console for Gitea (“Native experience. External architecture.”).
- **Shape:** Go single-binary backend + embedded React/TypeScript SPA; SQLite default, PostgreSQL optional; Gitea via API + webhooks only (no fork).
- **Boundaries:** `internal/forge` (Gitea-only impl in V1) · `internal/store` (SQL) · `internal/api` (REST `/api/v1`) · `internal/sync` + `internal/webhooks` · `internal/auth` / `internal/authz` · `internal/attention` · `internal/workflows` · SSE in `internal/realtime`.
- **Authz:** Sync service credential ≠ user session; all user-visible queries scoped by `user_repository_access` server-side (no client-side filtering as security).

## Decisions

- Follow [`docs/prd-spec.md`](docs/prd-spec.md) as product/tech source of truth; build from [`docs/implementation-plan.md`](docs/implementation-plan.md).
- Adopt PRD ADR-001…015; plan adds ADR-016…028 (chi, goose, sqlc, Vite/TanStack/React Flow, DB sessions, AES-GCM secrets, ACL table, in-process webhook workers, SSE-first, module path `github.com/ncdlabs/gitea-lens`).
- **Sequencing refinement:** Authentication/authorization (M4) before core UI (M5), even though PRD §59 lists OAuth later — so UI never ships without server-side repo filtering.
- No Redis, no WebSockets for V1 unless SSE proven insufficient; no Forgejo/GitHub/GitLab impl yet; no per-repo Lens config; telemetry off.
- Logs fetched on demand; not persisted by default.
- Compose/Helm use `.yaml` preference (`compose.yaml`).

## References

- Spec: `docs/prd-spec.md`
- Implementation blueprint: `docs/implementation-plan.md`
- Target tree: PRD §49 / plan §22
- First build slice: plan §24

## Known gotchas

- Repository was greenfield at planning (only PRD present; no application code/commits).
- Gitea Actions run/job APIs and webhooks are relatively new — declare minimum version after capability spike (plan Q1).
- Subpath reverse-proxy (`/lens`) is first-class and easy to get wrong (assets, OAuth redirects, cookies).
- Gitea UI installer must use marker blocks; never silently overwrite admin `extra_*.tmpl` content.
- SQLite needs WAL + short transactions under webhook + reconcile writers.

## Policies

- Do not remove documented product/debug features without explicit approval.
- Do not introduce ncdLabs-hosted service dependencies.
- Do not trust client-side repository filtering for authorization.
- Ask before major scope expansion (write ops, multi-forge, Redis, WebSockets).

## Known constraints

- Target Gitea API family ~1.26 (docs cite 1.26.4); capability-detect and degrade.
- V1 read-first; rerun/cancel deferred.
- Performance design targets: ~1k repos, 10k open PRs, 1M retained runs (engineering targets, not SLAs).

## Environment notes

- Default listen `:8090`; data dir `/data` in containers.
- Image target: `ghcr.io/ncdlabs/gitea-lens`.
- Local containers: Podman / `podman compose` preferred per project conventions.
