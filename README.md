# Wazuh Threat Mitigation

Automated detection and response pipeline built on **Wazuh SIEM** and **Suricata IDS**,
running in Docker. Network events from Suricata are classified, correlated by custom
Wazuh rules, and high-confidence threats (for example cryptomining traffic) are
**blocked automatically** with Wazuh Active Response.

## Architecture

```text
 Network traffic
       │
       ▼
 Suricata 7 (IDS) ──► eve.json
       │
       ▼
 eve filter (Python) ── classifies events (e.g. threat_type=cryptomining), drops noise
       │
       ▼
 Wazuh Manager 4.14 ── custom decoders + 20 custom rules (MITRE ATT&CK mapped)
       │                         │
       │                         ▼
       │              Active Response: firewall-drop
       │              (block source IP for 600 s)
       ▼
 Wazuh Indexer ──► Wazuh Dashboard (https://localhost)
```

## Detection Rules

Custom rules live in [`single-node/config/wazuh_cluster/local_rules.xml`](single-node/config/wazuh_cluster/local_rules.xml).

| Category | Examples |
| --- | --- |
| Cryptomining | Miner activity (MITRE T1496), repeated activity from the same source within 5 minutes |
| Web application | Web-application attack patterns |
| Identity & access | Authentication and access threats |
| Malware / C2 | Command-and-control and malware traffic, auto-response candidates |
| Dependency risk | Vulnerable Next.js / NestJS / Node.js packages, dependency manifest changes |
| Noise reduction | Suppression of low-value informational events to reduce alert fatigue |

## Automated Response

Rule `100100` (cryptomining) triggers a custom `firewall-drop` active response that
blocks the offending source IP with `iptables` for **10 minutes**. The wrapper script
[`firewall-drop-custom`](single-node/config/wazuh_cluster/firewall-drop-custom) makes
sure `iptables` is present inside the manager container before running the action.

## Regression Testing

[`scripts/defense_regression.sh`](single-node/scripts/defense_regression.sh) verifies the
whole chain after every change:

1. `docker compose config` is valid
2. All core containers (manager, indexer, dashboard, Suricata, eve filter) are running
3. The manager can resolve and reach the indexer on port 9200
4. The Suricata filter is producing output
5. A test alert is ingested by Wazuh
6. Active response fires and an `iptables` DROP rule is created

## Quick Start

```bash
cd single-node

# 1. Required kernel setting for the indexer (Linux host)
sudo sysctl -w vm.max_map_count=262144

# 2. Generate TLS certificates
docker compose -f generate-indexer-certs.yml run --rm generator

# 3. Configure secrets (never commit real values)
cp .env.example .env

# 4. Start the stack and run the regression test
docker compose up -d
./scripts/defense_regression.sh
```

Dashboard: `https://localhost` (port 443 → 5601)

## Tech Stack

Wazuh 4.14.5 (manager, indexer, dashboard) · Suricata 7 · Python · Docker Compose · iptables · MITRE ATT&CK

## What I Learned

- Filtering and classifying IDS events before they reach the SIEM greatly reduces noise.
- Active response should only fire on high-confidence rules, with a timeout, to avoid blocking legitimate users.
- A regression script turns a fragile security lab into something that can be changed safely.
