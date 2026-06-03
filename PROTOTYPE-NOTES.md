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

## UPDATE: real HA install revealed the permission fix (2026-06-03)
First real install failed: `PermissionError: [Errno 13] Permission denied: '/addon_config'`.
Cause: HA creates the `addon_config` map dir owned by ROOT, but the upstream entrypoint drops
Spoolman to the `app` user (uid 1000), which then can't write the data dir. The entrypoint does
NOT chown the data dir.
FIX: set `PUID: "0"` / `PGID: "0"` in config.yaml environment → 'app' becomes root → can write
/addon_config. Verified by reproducing the exact condition (root-owned volume dir) with podman:
"Startup complete", health {"status":"healthy"}, spoolman.db written. THIS is the working config.

## UPDATE 2: ingress blank UI -> switched to direct port (2026-06-03)
Real install: backend healthy but UI BLANK under ingress. Cause: Spoolman's base_path is set
ONCE from SPOOLMAN_BASE_PATH at startup (env.py:463) and does NOT read HA ingress's dynamic
per-session path (X-Ingress-Path). So /assets/* load from the wrong path -> blank. Upstream limit.
FIX: `ingress: false` + `webui: http://[HOST]:[PORT:8000]` -> "Open Web UI" button opens the
direct port 7912 where assets resolve correctly (verified: full HTML + /assets + /config.js served
on http://192.168.178.38:7912). Same port serves Moonraker [spoolman] too. Trade-off: opens in a
new tab instead of embedded in HA sidebar — minor.
