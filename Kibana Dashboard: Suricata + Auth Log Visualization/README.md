# 📊 Project 03: Kibana Dashboard - Suricata + Auth Log Visualization

## 📋 Overview

This project builds a Kibana dashboard covering the same detection surface as my Splunk Project 05 dashboard, using the field names confirmed in Project 02 (`event_type`, `alert.signature`, `alert.signature_id`). The goal was to get hands-on with Kibana's Lens visualization editor and see how it compares to building the equivalent panels in Splunk.

Three of the four planned panels made it into this dashboard cleanly. The fourth, a dedicated SSH brute force detection table, hit a configuration issue I wasn't able to resolve in this session and is left out of this build rather than shipped in a broken state. That panel is documented below as a known gap, and I'll pick it back up separately.

## 🖥️ Lab Setup

- Kibana Lens, data view `filebeat-*`
- Time range set to Last 7 days to capture all test windows from earlier Suricata projects
- Same underlying data confirmed in Project 02: Suricata `eve.json` events tagged with `log_type: "suricata_eve"`, auth.log events tagged with `log.file.path: "/var/log/auth.log"`

## 🧩 Dashboard Panels

### 📈 Alert Volume Over Time
Vertical bar chart, `Count of records` on the vertical axis, `@timestamp` (3-hour interval) on the horizontal axis, broken down by `alert.signature`. Filtered to `event_type : "alert"`.

This panel clearly shows two distinct spike clusters around the 7th and 9th-10th of July, corresponding to my Suricata scan testing sessions (Projects 02 and 04 in the suricata-ids-lab repo). The dominant series by far is the custom NULL scan rule (green), consistent with what I already observed in my Splunk dashboard, since that rule fires per-packet rather than on a threshold.

### 🏆 Top Alert Signatures
Horizontal bar chart, top 10 values of `alert.signature` by count. Filtered to `event_type : "alert"`.

Results:
| Signature | Approx. Count |
|---|---|
| CUSTOM Nmap NULL Scan Detected | ~4,000 |
| POSSBL PORT SCAN (second variant) | ~1,700 |
| POSSBL PORT SCAN (third variant) | ~1,300 |
| POSSBL SCAN SHELL M-SPLOIT TCP | small |
| Remaining signatures | small, long tail |

This matches the same ranking pattern I found in the Splunk version of this dashboard: the custom NULL scan rule dominates raw event count, with the ET/open SYN, XMAS, and ACK scan signatures trailing well behind it. Seeing the identical ranking in two independent SIEMs, built from the same source data, is a good sanity check that neither pipeline is silently dropping or duplicating events.

### 🗂️ Scan Detection Coverage
Built as a Markdown panel rather than a live query, since Kibana doesn't have a direct equivalent to Splunk's `makeresults`/`append` pattern for hardcoding reference rows into a queryable table.

| Scan Type | Detected | Signature |
|---|---|---|
| SYN | Yes | 3400001/3400002 |
| NULL | Yes (custom rule) | 1000004 |
| FIN | No | none |
| XMAS | Yes | 3400005 |
| ACK | Yes | 3400004 |
| Connect | No | none |

This carries the same Project 04 findings from my Suricata repo forward into this dashboard. I did catch and fix a copy-paste error here mid-build, where the NULL row briefly showed `1000001` (my SSH brute force rule's SID) instead of `1000004` (the actual NULL scan rule). Worth noting since it's a good example of how easy it is to introduce a small but meaningful error when manually transcribing reference data instead of pulling it from a live query.

### ❌ SSH Brute Force Detection (Not Completed This Session)
The fourth panel was meant to filter on `alert.signature_id : 1000001` and display the single confirmed SSH brute force alert from Suricata Project 03 as a simple event table. In practice, the panel's "Count of records" column consistently rendered as empty (`-`) across every row, even after removing and re-adding the metric and confirming the filter syntax matched what worked elsewhere in this same dashboard.

Rather than include a panel showing incorrect or missing data, I'm leaving this out of the current dashboard build. The underlying data is confirmed present and queryable, since the same alert was successfully pulled in Discover back in Project 02, so this looks like a Lens table-configuration issue specific to this panel rather than a data problem.

## 🧠 What I Learned

Building three of four panels successfully, and hitting a real wall on the fourth, was a useful contrast to how smoothly the equivalent Splunk dashboard came together. Splunk's SPL made it trivial to pull a single-event table with `table _time, src_ip, dest_ip, dest_port, alert.signature`, but Kibana's Lens editor separates "metrics" and "breakdowns" in a way that isn't always intuitive when you just want a flat list of matching raw events rather than an aggregation. I also confirmed something valuable across both the Alert Volume and Top Alert Signatures panels: identical rankings and proportions showed up in both SIEMs from the same source data, which is a meaningful cross-validation that my Filebeat ingestion pipeline isn't introducing any silent data loss or duplication compared to my Splunk pipeline.

## 🎯 SOC Relevance

Knowing when to stop and document a blocker honestly, rather than force a broken panel into a dashboard just to say it's "done," reflects real analyst judgment. A dashboard panel silently showing `-` for every row is arguably worse than no panel at all, since it can be mistaken for "zero alerts" by anyone glancing at it, when the reality is a configuration problem. This project also reinforced the value of cross-validating detection data across two independent SIEM pipelines fed by the same source, which is a legitimate practice in real SOC environments when trustworthiness of a new ingestion pipeline needs to be established before it's relied on operationally.




