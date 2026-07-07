# Monitoring & Telemetry Lab — SNMP → gNMI → Prometheus/Grafana

Track 3 of the [Learning Roadmap](../../LEARNING-ROADMAP.md) as its own lab. The diamond gets
watched: first the old way (SNMP polling), then the new way (gNMI streaming telemetry), landing
in the same Prometheus + Grafana stack Telcotech runs. By the end you're building lab versions
of the dashboards you watch at work — and you know what's behind every panel.

**Platform:** XRv9000 ~24.3.1 diamond (shared topology, see [CLAUDE.md](../../CLAUDE.md)) +
a Docker host for the monitoring stack. **Prerequisite:** EVE-NG mgmt network reachable from
your PC — the same unblock the Automation track needs. One cable, two tracks.

**Monitoring stack placement:** Docker on WSL2 (Docker Desktop) is the recommended start —
zero extra EVE-NG resources. The Ubuntu-VM-inside-EVE-NG variant (per the Linux lab design)
is a later swap; the configs are identical either way.

**Build status:** Phase 1 paste-ready. Phases 2–7 planned — fill one per session.

---

## Phase plan

| # | Phase | Status |
|---|-------|--------|
| 1 | Monitoring stack up (Prometheus + Grafana in Docker) | ✅ paste-ready below |
| 2 | SNMP baseline on IOS-XR | 🟡 planned |
| 3 | snmp_exporter → first real dashboard | 🟡 planned |
| 3.5 | Cacti — the legacy stack at work (Telcotech/Ezecom/Cellcard run it) | 🟡 planned |
| 4 | gNMI on IOS-XR + gnmic exploration | 🟡 planned |
| 5 | Streaming telemetry pipeline + core dashboards | 🟡 planned |
| 6 | Alerting that matters (Alertmanager + break-tests) | 🟡 planned |
| 7 | Capstone: SNMP vs telemetry + automation tie-in | 🟡 planned |

---

## Phase 1 — Monitoring stack up

**Goal:** Prometheus and Grafana running in Docker, Prometheus scraping itself, Grafana showing
Prometheus as a data source. No routers involved yet — prove the stack before pointing it at
the network.

**Why this order:** every later phase lands data in this stack. Debugging "no data in Grafana"
is miserable when the exporter, the network, *and* the stack are all suspects — so make the
stack the one thing you already trust.

**► Configure** — create this tree on the Docker host (WSL2):

```
monitoring/
├── docker-compose.yml
└── prometheus/
    └── prometheus.yml
```

`docker-compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports: ["9090:9090"]
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prom_data:/prometheus
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports: ["3000:3000"]
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=lab123    # lab only — never a real password
    volumes:
      - graf_data:/var/lib/grafana
    restart: unless-stopped

volumes:
  prom_data:
  graf_data:
```

`prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s        # how often Prometheus pulls each target

scrape_configs:
  - job_name: prometheus      # Prometheus watching itself — first target
    static_configs:
      - targets: ["localhost:9090"]
```

Bring it up:

```bash
cd monitoring && docker compose up -d
```

**Verify:**

```bash
docker compose ps                                  # both containers Up
curl -s localhost:9090/-/ready                     # "Prometheus Server is Ready."
curl -s "localhost:9090/api/v1/query?query=up"     # value 1 for the prometheus job
```

Then in a browser: `localhost:9090/targets` (target UP), `localhost:3000` (admin/lab123) →
Connections → Add data source → Prometheus → URL `http://prometheus:9090` → Save & Test.
Build one throwaway panel graphing `prometheus_http_requests_total` — your first time series.

**Can I explain it?** Pull vs push monitoring — which is Prometheus, and why does that design
scale? What exactly is a time series (metric name + labels + samples)? Why does Grafana reach
Prometheus at `http://prometheus:9090` and not `localhost` (hint: Docker networking)?

---

## Phase 2 — SNMP baseline on IOS-XR *(planned)*

**Objective:** SNMPv2c read-only on R1–R4 (`snmp-server community lab RO` + mgmt ACL), then
walk it from the Docker host with `snmpwalk`: find hostname (sysName), interface names
(ifDescr), counters (ifHCInOctets). Learn to read an OID before any tool hides it from you.
**Verify:** `snmpwalk -v2c -c lab <R1> ifDescr`, watch a counter move during a ping flood.
**Checkpoint:** why HC (64-bit) counters — what breaks with 32-bit at 10 Gbps? What does the
mgmt ACL protect against?

## Phase 3 — snmp_exporter → first real dashboard *(planned)*

**Objective:** add `snmp_exporter` to the compose file, scrape all four routers (if_mib module),
Grafana dashboard: per-interface traffic, errors, discards, oper-status.
**Verify:** `up{job="snmp"}` == 1 for all routers; shut a link, watch oper-status drop on the
panel.
**Checkpoint:** the exporter pattern — why does Prometheus never speak SNMP itself?

## Phase 3.5 — Cacti: the legacy stack at work *(planned)*

**Objective:** run **Cacti** in Docker beside Prometheus (community image, e.g. `smcline06/cacti`
+ MariaDB — verify current image before building), add R1–R4 via the same SNMP community as
Phase 2, build interface graphs, then install the **Weathermap plugin** and draw the diamond —
a mini version of the NOC wall map at work. Finish by graphing the *same* interface in Cacti and
Grafana simultaneously and explaining every difference you see.
**Why:** Telcotech/Ezecom/Cellcard run Cacti today. Debugging a wrong graph at work means
knowing RRDtool underneath: polling interval vs resolution, **consolidation** (why last month's
graph smooths away spikes — averages destroy microbursts), counter wraps showing as absurd
spikes.
**Verify:** 5-min polling populating graphs for all four routers; weathermap link colors change
under a ping flood; force a counter wrap discussion with a 32-bit vs 64-bit (HC) data source.
**Checkpoint:** where did yesterday's 1-minute detail go after a week (RRD consolidation)? Why
can Cacti never show a 10-second spike that Grafana/gNMI can? Which failure does *polling
gap* hide?

## Phase 4 — gNMI on IOS-XR + gnmic *(planned)*

**Objective:** `grpc` + gNMI on the routers; from the host use **gnmic** to `capabilities`,
`get`, then `subscribe` (sample-interval 10s) to interface counters — watch data *stream* in,
no polling. First contact with YANG paths (`openconfig-interfaces`).
**Verify:** a gnmic subscription printing counter updates live.
**Checkpoint:** poll vs subscribe — what does streaming fix at 5000-device scale (hint: what
did Phase 2's snmpwalk cost per poll)?

## Phase 5 — Streaming telemetry pipeline + core dashboards *(planned)*

**Objective:** gnmic's built-in prometheus output (gnmic subscribes to routers, exposes
metrics; Prometheus scrapes gnmic). Dashboards that mirror ops reality: BGP session state,
IS-IS adjacencies, interface errors, CPU/memory. This is the lab version of your Telcotech
Grafana wall.
**Verify:** break a BGP peer, watch the panel go red inside one sample interval.
**Checkpoint:** trace one metric end-to-end: YANG leaf → gnmic → Prometheus metric+labels →
PromQL query → panel.

## Phase 6 — Alerting that matters *(planned)*

**Objective:** Alertmanager + rules for the few things worth waking for: BGP peer down > 1m,
interface error-rate rising (`rate()`), route count anomaly, device unreachable. Then
**break-test each alert** — an alert you've never seen fire is a rumor, not an alert.
**Verify:** every rule fired once on purpose; false-positive check after link flap.
**Checkpoint:** why alert on `rate(errors)` not `errors`? What makes an alert *actionable*
(the 2am test: what would you actually do)?

## Phase 7 — Capstone: comparison + automation tie-in *(planned)*

**Objective:** write the SNMP vs streaming-telemetry trade-off table from your own evidence
(latency, load, coverage, config effort). Then close the loop with the Automation track's
Stage 6: expose `check_bgp.py`-style results as Prometheus metrics (textfile exporter), so
scripted checks and telemetry land on one dashboard. Stretch: syslog → Grafana Loki.
**Checkpoint:** when is SNMP still the right answer in 2026? What belongs in a check script
vs a telemetry subscription?

---

## Files

- `stack/` — docker-compose + prometheus/gnmic/alertmanager configs (created as phases build)
- `configs/` — router deltas per phase (`! Phase N:` markers)
- `notes/` — verification logs per phase (paste real output, including PromQL used)
- [`HANDOFF.md`](HANDOFF.md) — session resume point
