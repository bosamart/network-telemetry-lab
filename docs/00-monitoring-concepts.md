# Monitoring Concepts — Start Here (Beginner Friendly)

Read this before Phase 1. No prior monitoring knowledge assumed. Every tool in this lab exists
to answer one of three questions about a network:

1. **Is it up?** (device, link, protocol session)
2. **How much is it carrying?** (bandwidth, errors, CPU)
3. **Who/what is the traffic?** (which customers, which applications)

Different tools answer different questions — that's why NOCs run several at once.

---

## The big idea: metrics are just numbers over time

A router knows thousands of numbers about itself: bytes received on each interface, CPU %,
BGP session state, temperature. Monitoring is simply: **collect some of those numbers
repeatedly, store them with timestamps, draw them, and alarm when they look wrong.**

A stored number-with-timestamp-and-labels is called a **time series**:

```
interface_in_octets{device="R1", interface="Gi0/0/0/0"}  =  8,213,450,112   @ 10:15:00
interface_in_octets{device="R1", interface="Gi0/0/0/0"}  =  8,214,890,340   @ 10:15:15
```

Everything else in this lab — SNMP, Cacti, NetFlow, gNMI, Prometheus, Grafana — is just
different machinery for producing, moving, storing, or drawing time series.

---

## Pull vs push — the one design split that explains every tool

- **Pull (polling):** the monitoring server asks each device "what's your counter now?" every N
  seconds. Simple, but the server does all the work, and nothing between polls is ever seen.
  → SNMP polling, Cacti, Prometheus scraping.
- **Push (streaming/export):** the device sends data out on its own — either on a timer or when
  something happens. Scales better, catches short events.
  → NetFlow export, gNMI streaming telemetry, syslog.

When a graph "missed" something at work, the first question is: was this a pull system, and did
the event fall between two polls?

---

## Counters vs gauges — why graphs need math

- A **gauge** is a value that goes up and down: CPU %, temperature, active sessions. Graph it
  directly.
- A **counter** only ever increases: total bytes received since boot. Nobody cares about the
  total — you care about the **rate**: (this poll − last poll) ÷ seconds = bits per second.
  Every traffic graph you've ever seen is this subtraction.

Two counter gotchas that cause "impossible" graphs at work:
- **Counter wrap:** a 32-bit counter maxes out at ~4.29 billion, then restarts at 0. At 10 Gbps
  that happens every ~3.4 seconds — the math sees a "negative" jump and draws a spike. Fix:
  64-bit ("HC", high-capacity) counters. This is why Phase 2 insists on `ifHCInOctets`.
- **Device reboot:** counters reset to 0 → same fake spike.

---

## The tools in this lab, in one paragraph each

**SNMP** (1988, still everywhere): a question-answer protocol. Every fact on the device has a
numeric address called an **OID** (e.g. `1.3.6.1.2.1.2.2.1.10` = interface input bytes). A
**MIB** is the dictionary that maps OIDs to names. A **community string** is a shared password
(v2c — plaintext, hence the ACL; v3 adds real auth). Tools "walk" a subtree of OIDs to
discover what's there.

**Cacti** (the legacy stack at your work): a web app that polls SNMP every 5 minutes and stores
results in **RRD** (round-robin database) files. RRD has a fixed size forever because it
**consolidates**: it keeps 5-min detail for a day or two, then averages it into 30-min points,
then 2-hour points. That's why last month's graph is smooth — the detail was *averaged away*,
not lost in transmission. Averages destroy spikes: a 30-second microburst that filled a link
never shows on a weekly Cacti graph.

**NetFlow** (who/what): the router groups packets into **flows** (same src/dst IP, ports,
protocol) and exports records: "10.1.1.5 → 8.8.8.8, UDP/443, 1.2 MB". A **collector** (nfdump)
stores them; a UI (NfSen at work) shows top talkers. Because tracking every packet is expensive,
routers usually **sample** (e.g. 1 in 1000 packets) and scale up the numbers — fine for "who is
filling the link", dangerous for billing small flows.

**gNMI / streaming telemetry** (the modern replacement for SNMP polling): the device *pushes*
chosen counters every N seconds over gRPC. What to send is named by **YANG paths** — a
standardized tree of every configurable/readable thing (e.g.
`interfaces/interface/state/counters/in-octets`). Faster, structured, scales to thousands of
devices.

**Prometheus** (the database): pulls ("scrapes") metrics from HTTP endpoints every 15s, stores
time series, and answers questions in **PromQL** — e.g.
`rate(interface_in_octets[5m]) * 8` = bits per second over the last 5 minutes. Prometheus never
speaks SNMP or gNMI itself; small adapters called **exporters** translate
(snmp_exporter, gnmic).

**Grafana** (the glass): draws PromQL results as dashboards. Grafana stores nothing — it's a
window onto Prometheus (and at work, onto other sources too).

**Alertmanager** (the pager): PromQL expressions checked continuously; true for N minutes →
notify. The craft is not alerting on everything — it's alerting only on things a human must act
on ("BGP peer down 5 min" yes; "CPU touched 80% once" no).

---

## How they fit together (this lab's plumbing)

```
Routers (R1–R4)
 ├─ SNMP  ──► snmp_exporter ──► Prometheus ──► Grafana
 ├─ SNMP  ──► Cacti (own poller + RRD + web UI, separate stack)
 ├─ NetFlow ─► nfcapd/nfdump (own collector, separate stack)
 └─ gNMI  ──► gnmic ─────────► Prometheus ──► Grafana
                                    └────────► Alertmanager ──► you
```

Cacti and NetFlow are deliberately separate stacks — that mirrors real NOCs, where the modern
(Grafana) and legacy (Cacti/NfSen) systems run side by side and disagree slightly. Phase 3.5's
closing exercise (same interface graphed in both) teaches you *why* they disagree.

---

## Mini-glossary

| Term | Meaning |
|------|---------|
| OID | Numeric address of one fact in SNMP (`1.3.6.1.2.1.1.5` = hostname) |
| MIB | Dictionary translating OIDs to human names |
| Community string | SNMP v1/v2c shared password (plaintext — always ACL it) |
| RRD | Fixed-size database that averages old data into coarser points |
| Consolidation | That averaging — why old graphs lose spikes |
| Flow | Packets sharing src/dst IP + ports + protocol, summarized as one record |
| Sampling | Inspecting 1-in-N packets and scaling up — cheap but approximate |
| YANG | Standardized tree naming everything on a device (gNMI reads it) |
| Scrape | One Prometheus pull of an HTTP metrics endpoint |
| Exporter | Adapter that translates a protocol (SNMP…) into Prometheus format |
| Time series | One metric + label set, stored over time |
| PromQL | Prometheus query language (`rate()`, `up == 0`) |
| Cardinality | Number of distinct label combinations — high cardinality kills TSDBs |

---

**Next:** [README Phase 1](../README.md) — bring up Prometheus + Grafana and graph your first
time series. Come back to this doc whenever a phase term feels foggy; it's meant to be re-read.
