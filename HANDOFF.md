# HANDOFF — Monitoring & Telemetry Lab

**Status (7 Jul 2026):** README created — 7-phase plan, **Phase 1 paste-ready** (Prometheus +
Grafana via Docker on WSL2). Phases 2–7 have objective/verify/checkpoint, no configs yet.
Nothing run yet. `stack/`, `configs/`, `notes/` folders not created — make them as phases build.

## Prerequisites (check before Phase 2+)

- **EVE-NG mgmt network reachable from the PC** — same unblock as the Automation track.
  Routers need mgmt IPs the Docker host can hit for SNMP/gNMI.
- Docker Desktop with WSL2 backend working.

## Next session

1. Run Phase 1 (stack up, self-scrape, Grafana data source). Log to `notes/phase1-stack-verify.md`.
2. Fill Phase 2's ► Configure blocks: SNMPv2c RO + mgmt ACL on R1–R4, snmpwalk drills from the
   host. Keep the teaching focus on reading raw OIDs before tools abstract them.
3. One phase per session; update the phase-plan table as you go.

## Gotchas

- Grafana → Prometheus URL is `http://prometheus:9090` (container name on the compose network),
  NOT `localhost` — most common Phase 1 stumble.
- Prometheus/Grafana data lives in named volumes — `docker compose down` keeps it,
  `down -v` wipes it.
- Phase 4: XRv9000 needs `grpc` config; check whether 24.3.1 wants `grpc no-tls` for lab use
  (TLS setup is out of scope until it works without).
- gnmic Prometheus-output mode (Phase 5) replaces the need for a separate telemetry DB — don't
  add InfluxDB/TIG stack complexity; stay on one stack (Prometheus).
- Phase 3.5 (Cacti): verify the current Docker image before building (`smcline06/cacti` was the
  common community one; official images may exist now). Needs MariaDB + persistent volumes.
  Weathermap is a plugin install inside Cacti, not a container. Added because work
  (Telcotech/Ezecom/Cellcard) runs Cacti — keep the teaching focus on RRD consolidation and
  polling-gap blindness, the two things that explain "weird graphs" at work.
- Ties to `Automation/ROADMAP.md` Stage 6 — the textfile-exporter tie-in belongs there too.

## Push to GitHub

Repo: `bosamart/network-telemetry-lab` (**public**, pushed early as portfolio/reading copy via
`push-monitoring-lab-to-github.ps1` in the project root — re-run it after each phase to
re-sync). Add dashboard + weathermap screenshots to the README from Phase 3 onward; that's the
portfolio value.
