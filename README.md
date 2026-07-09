# 🔎 SIEM-ELK-Stack

A hands-on SOC analyst lab documenting my build-out of the Elastic Stack (Elasticsearch, Kibana, Filebeat) as a second SIEM platform, run alongside my existing Splunk-based lab. This repo captures the full process: installation, real authentication and permission troubleshooting, log ingestion, and dashboard-building, using the same underlying log sources (Linux auth logs, Suricata IDS alerts) I already work with in my Splunk and Suricata repos.

The goal isn't just "install ELK and call it done." It's to build genuine, comparable experience across two different SIEM platforms using identical data, so I can speak to the real tradeoffs between them rather than just knowing one tool.

## 🖥️ Lab Environment

- **Kali Linux VM** — Elasticsearch, Kibana, and Filebeat all installed via apt and run as systemd services
- **Elasticsearch 8.19.18** — security enabled by default, TLS on the HTTP layer, separate `elastic` superuser and `kibana_system` service accounts
- **Kibana** — web UI at `localhost:5601`
- **Filebeat** — ships two log sources: `/var/log/auth.log` and `/var/log/suricata/eve.json`
- Same VM environment documented across my [Wireshark-projects](https://github.com/collinsnwammuo/Wireshark-projects), [SIEM-Splunk](https://github.com/collinsnwammuo/SIEM-Splunk), and [suricata-ids-lab](https://github.com/collinsnwammuo/suricata-ids-lab) repos, so detection events referenced here can be cross-checked against those repos directly

## 📁 Projects

| # | Project | Status |
|---|---|---|
| 01 | [ELK Stack Installation & Filebeat Ingestion](./project-01-elk-installation/) | ✅ Complete |
| 02 | [Kibana Discover & Index Patterns](./project-02-discover-index-patterns/) | ✅ Complete |
| 03 | Kibana Dashboard: Suricata + Auth Log Visualization | 🔜 Planned |
| 04 | Splunk vs Kibana Comparison | 🔜 Planned |

### 🛠️ Project 01: ELK Stack Installation & Filebeat Ingestion
Full install of Elasticsearch and Kibana, working through three real failure points along the way: using the wrong service account for Kibana's setup wizard, an HTTP/HTTPS scheme mismatch, and a certificate permission error that caused Kibana to crash-loop. Also covers configuring and validating Filebeat as the log shipper for both `auth.log` and Suricata's `eve.json`.

### 🔍 Project 02: Kibana Discover & Index Patterns
Creating the `filebeat-*` Data View and exploring both ingested log sources in Discover. Includes a real debugging story: a query that should have worked (`log.file.path`) returned zero results against the Suricata data because of a custom `log_type` field I'd set in my own Filebeat config and forgotten about. Also confirms cross-platform detection parity by pulling the same custom NULL scan rule alert (SID 1000004) that's already documented in my Suricata repo.

### 📊 Project 03: Kibana Dashboard (Planned)
Building a Kibana dashboard covering the same detection surface as my existing Splunk dashboard (Suricata Project 05): alert volume over time, top signatures, and SSH brute force detection, using the field names confirmed in Project 02.

### ⚖️ Project 04: Splunk vs Kibana Comparison (Planned)
A direct writeup comparing both SIEMs against identical source data: SPL vs KQL, field extraction behavior, dashboard-building experience, and which platform I'd reach for in different SOC scenarios.

## ✍️ Writing Style

Every project README in this repo follows the same format as my other lab repos:
- First-person voice, written as genuine analyst documentation rather than a tutorial
- No em dashes, double hyphens used instead where needed
- Every project includes What I Learned, SOC Relevance, and MITRE ATT&CK Mapping sections
- Real errors and troubleshooting are documented as they happened, not smoothed over. If something failed on the first attempt, that's in the README along with how I diagnosed and fixed it

## 🔗 Related Repos

- [Wireshark-projects](https://github.com/collinsnwammuo/Wireshark-projects) — packet-level traffic analysis and malware PCAP investigation
- [SIEM-Splunk](https://github.com/collinsnwammuo/SIEM-Splunk) — Splunk-based log ingestion, SPL, dashboards, and live attack detection
- [suricata-ids-lab](https://github.com/collinsnwammuo/suricata-ids-lab) — custom Suricata rule development and live detection testing, the same detection events referenced throughout this repo
