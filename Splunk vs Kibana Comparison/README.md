# ⚖️ Project 04: Splunk vs Kibana Comparison

## 📋 Overview

This project is a direct comparison between Splunk and the Elastic Stack, based on genuinely building the same detection dashboard twice against identical source data: the same Suricata `eve.json` alerts and the same `auth.log` SSH activity, ingested through two completely separate pipelines (Splunk's custom sourcetype ingestion vs Filebeat into Elasticsearch). Rather than a feature-list comparison pulled from documentation, this is written from what actually happened while building both, including the parts that went smoothly and the parts that didn't.

## 🖥️ What Was Compared

- **Splunk**: SIEM-Splunk repo, Projects 01-14, SPL-based dashboards
- **Kibana/ELK**: SIEM-ELK-Stack repo, Projects 01-03, KQL-based dashboards with Lens
- Same underlying data in both: Suricata scan and alert traffic, SSH brute force test, Linux auth logs

## 🔤 Query Language: SPL vs KQL

Splunk's SPL felt more expressive once I understood the pipe-based syntax, particularly for building tables from scratch. Pulling a clean single-event table in Splunk was as simple as:

```spl
index=soc_lab sourcetype=suricata "1000001"
| table _time, src_ip, dest_ip, dest_port, alert.signature
```

The equivalent task in Kibana's Discover was just as simple as a raw search:
```
alert.signature_id : 1000001
```

But when I tried to carry that same simplicity into a Lens dashboard panel, it became noticeably harder. Lens separates every visualization into "metrics" and "breakdowns" rather than letting me just declare a flat table of fields the way SPL's `table` command does. My SSH Brute Force Detection panel in Project 03 never got resolved this session because of exactly this friction, the "Count of records" column kept rendering empty no matter how I reconfigured the metric, and I don't yet have a clean explanation why. SPL's model, where you build a result set step by step and then just `table` the columns you want, mapped much more directly onto what I was trying to do than Lens's aggregation-first approach did.

**Verdict**: SPL is more directly expressive for ad hoc, non-aggregated tables. KQL/Discover is comparably easy for simple searches, but Lens's visualization model adds friction for anything that isn't a straightforward count or aggregation.

## 🐛 Setup and Ingestion Experience

This is where the two platforms diverged the most, and not in Splunk's favor from a raw effort standpoint, but the divergence taught me more.

Getting Splunk ingesting Suricata and auth log data required real work too (custom sourcetypes, `rex` field extraction for the `linux_secure` src field, threshold tuning on my custom detection rules), but once configured, it stayed configured. I never had to debug an authentication or certificate issue in Splunk the way I did in Elastic.

Standing up ELK, by contrast, involved genuine troubleshooting across three separate failure points before Kibana would even load correctly:
1. Using the `elastic` superuser instead of the dedicated `kibana_system` service account
2. An `http://` vs `https://` scheme mismatch in `kibana.yml` that failed silently
3. A certificate permission error (`EACCES`) that caused Kibana to crash-loop until I added the `kibana` user to the `elasticsearch` group

None of these were Elastic being poorly designed, they reflect a more deliberate security model with separated service accounts and locked-down file permissions by default. But it meant my time-to-first-working-dashboard was considerably longer with ELK than it was with Splunk, where the apt install and initial dashboard setup was comparatively frictionless.

**Verdict**: Splunk had a smoother initial setup experience. ELK's security model is arguably more sound by default (separated service accounts, TLS everywhere out of the box), but that comes at the cost of a steeper initial configuration curve.

## 📊 Dashboard Building

Both platforms produced visually comparable dashboards covering the same detection surface: alert volume over time, top signatures, and a scan coverage reference table. The numbers matched closely between them, which was itself a valuable outcome, since it cross-validated that neither ingestion pipeline was silently dropping or duplicating events.

Where they differed:
- **Splunk** let me hardcode a reference table (the scan coverage table) directly in SPL using `makeresults` and `append`, keeping it as a live, queryable panel
- **Kibana** has no direct equivalent, so I built the same reference table as a static Markdown panel instead. It displays the same information but isn't a queryable visualization, just formatted text

**Verdict**: Splunk's dashboard tooling has more flexibility for mixing live queries with hardcoded reference data in the same visualization type. Kibana's Markdown panel is a reasonable workaround but is a genuinely different kind of object, not a query result.

## 🔍 Field Extraction and Data Structure

Splunk required manual field extraction work for less structured logs, most notably `rex "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"` to pull the source IP out of `linux_secure` events, since Splunk doesn't automatically parse arbitrary syslog formats into structured fields.

Kibana, via Filebeat's `ndjson` parser with `keys_under_root: true`, exposed Suricata's nested JSON fields (`alert.signature_id`, `flow.bytes_toclient`, etc.) as first-class queryable fields with zero manual extraction work. For structured JSON sources specifically, this was a clear advantage for Kibana.

However, the same ease of structured-field mapping is what caused my own confusion in Project 02: I'd set a custom `log_type` field in my Filebeat config and then forgot about it, leading to a query that silently returned zero results against the correct data. That wasn't Kibana's fault exactly, it was closer to a Splunk-style sourcetype-tagging decision that I made and then didn't document for myself.

**Verdict**: Kibana/Filebeat handles structured JSON sources (like Suricata's eve.json) with less manual work than Splunk. Splunk requires more manual extraction for semi-structured text logs, but that manual process also makes the extraction logic more visible and self-documenting at the point of use, which arguably made my own mistake in Kibana less likely to happen in Splunk.

## 🧠 What I Learned

The clearest lesson from this whole comparison is that "easier to set up" and "easier to operate long-term" aren't the same thing, and they don't always favor the same tool. Splunk got me to a working dashboard faster, but Kibana's default security posture (separated service accounts, mandatory TLS, locked-down file permissions) is a more defensible design for a production environment, even though it cost me real setup time. Similarly, Kibana's automatic JSON field mapping saved me manual extraction work but introduced a different kind of failure mode, a forgotten custom field silently breaking a query, that Splunk's more manual approach would have made harder to overlook. Neither platform came out as simply "better." They made different tradeoffs, and understanding those tradeoffs concretely, rather than abstractly, is the actual value of having built the same thing twice.




