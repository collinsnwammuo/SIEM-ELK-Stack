# 🔍 Project 02: Kibana Discover & Index Patterns

## 📋 Overview

With Elasticsearch, Kibana, and Filebeat all confirmed working from Project 01, this project covers creating a proper Data View in Kibana and exploring both ingested log sources (`auth.log` and Suricata's `eve.json`) in Discover. The goal was to get comfortable with KQL as a query language and directly compare how Kibana structures and exposes fields against how I already know Splunk handles the same two log sources.

## 🖥️ Lab Setup

- Elasticsearch, Kibana, and Filebeat all running as systemd services (confirmed via `systemctl status` before starting)
- Data View created: `filebeat-*`, timestamp field auto-detected as `@timestamp`
- Same two log sources from Project 01: `/var/log/auth.log` and `/var/log/suricata/eve.json`

## 🗂️ Creating the Data View

Kibana → Stack Management → Data Views → Create data view, matching the index pattern `filebeat-*`. Kibana correctly auto-detected `@timestamp` as the time field, so no manual configuration was needed there.

## 🔎 Exploring auth.log

Query used in Discover:
```
log.file.path : "/var/log/auth.log"
```

This worked immediately and returned 42 matching documents. Each document exposed `log.file.path` cleanly as `/var/log/auth.log`, along with the standard Filebeat/ECS metadata fields (`agent.*`, `host.*`, `ecs.version`). This confirms Filebeat's filestream input is tagging every event with its exact source file path, which made this query trivial to write correctly on the first try.

## 🐛 Exploring Suricata eve.json — The Query That Didn't Work

My first attempt used the same pattern as the auth.log query:
```
log.file.path : "/var/log/suricata/eve.json"
```

This returned zero results, even though the field list in the sidebar clearly showed Suricata-specific fields (`alert.signature_id`, `flow.bytes_toclient`, `dns.rrtype`, `fileinfo.filename`) were present somewhere in the index. That told me ingestion had worked, the query itself just wasn't matching anything.

Rather than guess at the exact path string, I cleared the query and inspected a raw document's fields directly in the sidebar and expanded rows. That's how I found the actual field Filebeat was tagging these events with wasn't `log.file.path` at all in a way that matched cleanly, it was a custom field:

```
log_type: "suricata_eve"
```

This came from the `fields:` block I'd set in my `filebeat.yml` input configuration back in Project 01, which I'd honestly forgotten was there. Once I queried on the correct field:

```
log_type: "suricata_eve"
```

This returned 24,747 documents. That volume checks out. It reflects every scan and traffic-flow event logged during all of my Suricata testing across Projects 01-04 of the suricata-ids-lab repo, not just alert-type events, since Suricata's `eve.json` also logs `flow` and `stats` event types alongside `alert` events.

## ✅ Filtering to a Specific Custom Rule

Once the correct base field was confirmed, I tested a more targeted query matching my custom NULL scan detection rule from the Suricata repo:

```
event_type : "alert" and alert.signature_id : 1000004
```

This returned exactly 2,000 matching documents, all showing `alert.signature: CUSTOM Nmap NULL Scan Detected`, `alert.signature_id: 1,000,004`, `alert.category: Attempted Information Leak`, and consistent `dest_ip: 192.168.56.101` across every hit. This lines up directly with the same rule and the same NULL scan test traffic documented in my Suricata Project 02 and Project 04 READMEs, confirming the exact same underlying detection event is queryable in both Splunk and Kibana, just through different field names and query syntax.

## 🧠 What I Learned

The biggest lesson here was that my assumption from the auth.log query, that `log.file.path` would consistently work as a filter across every log source, was wrong, and the reason wasn't a Kibana quirk, it was my own Filebeat configuration. I'd added a custom `log_type` field specifically for the Suricata input back in Project 01 and then forgot about it when writing this query. This is a good reminder that when a query fails silently (zero results, no error), the fastest path to an answer is inspecting a raw document's actual field values directly, rather than guessing variations of the query syntax. I also got a much clearer sense of how KQL differs from SPL in practice: the `field : "value"` syntax is more compact than Splunk's `sourcetype=x field=y`, but it also means field name accuracy matters even more since there's no equivalent to Splunk's more forgiving keyword search across all fields by default.

## 🎯 SOC Relevance

Being able to quickly pivot from "my query returned nothing" to "let me check what's actually in this document" is a core investigative skill for any SIEM, not just Kibana. This also reinforces something worth remembering in a real SOC: custom field mappings set during log shipper configuration (Filebeat, in this case) are easy to forget about weeks or months later, and undocumented custom fields are a common source of "the data's in the SIEM but nobody can find it" problems. Documenting exactly which field names route to which log source, as I've done here, is a small but genuinely useful piece of institutional knowledge.

## 🗺️ MITRE ATT&CK Mapping

| Technique | ID |
|---|---|
| Network Service Discovery (NULL scan detection) | T1046 |

