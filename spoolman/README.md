# Spoolman (Home Assistant Add-on)

Runs [Spoolman](https://github.com/Donkie/Spoolman) — filament inventory, weight, and cost tracking — as a Home Assistant add-on.

- Wraps the official `ghcr.io/donkie/spoolman` multi-arch image (no build).
- SQLite DB persisted to the add-on's `/addon_config` (included in HA backups).
- **Ingress**: open from the HA sidebar.
- **Direct LAN access** (for Moonraker `[spoolman]`): port `7912` on the HA host, i.e. `http://<HA-IP>:7912`.

## Moonraker integration (optional)
On the printer's `moonraker.conf`:
```
[spoolman]
server: http://<HA-IP>:7912
```
NOTE: requires a Moonraker version with the `[spoolman]` component (the Creality Sonic Pad's stock Moonraker is too old; use the UI standalone instead).
