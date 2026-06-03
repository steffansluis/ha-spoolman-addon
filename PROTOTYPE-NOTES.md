# Prototype validation (podman, 2026-06-03)

Validated the add-on config by running the exact image+env+volume that HA Supervisor would.

## Confirmed working
- Image `ghcr.io/donkie/spoolman` is multi-arch (amd64, arm64, arm/v7) — covers any HAOS host.
- Web UI serves on container :8000; API `/api/v1/health`, `/api/v1/info` respond 200.
- SQLite `spoolman.db` persists to the mapped data dir; external filament DB synced (6957 filaments).

## Key finding (the gotcha)
- The image entrypoint runs as **root**, chowns the data dir to uid 1000 (`app`), then drops to `app` via gosu.
- It MUST start as root-in-container. Local rootless-podman tests with `--userns=keep-id` FAILED
  ("error: failed switching to app: operation not permitted" / "cannot lock /etc/group").
- **HA Supervisor runs add-on containers as root by default → this works in HA.** `init: false`
  lets the image's own entrypoint handle the user drop. Do NOT set a custom user.
- Data dir must be writable: HA's `addon_config:rw` map satisfies this.

## Result
Add-on config is sound for HAOS. Remaining (untested locally, needs real HA): ingress base-path
behavior — Spoolman generally works behind HA ingress; if the UI assets 404 behind ingress,
add `SPOOLMAN_BASE_PATH` via the ingress entry path. Direct port 7912 access works regardless
(use that for Moonraker [spoolman] integration).
