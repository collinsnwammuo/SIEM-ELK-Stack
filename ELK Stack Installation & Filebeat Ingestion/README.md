# 🔎 Project 01: ELK Stack Installation & Filebeat Ingestion

## 📋 Overview

This project covers installing Elasticsearch and Kibana on Kali via the apt/package method, getting past several real authentication and permission obstacles along the way, and configuring Filebeat to ship `auth.log` and Suricata's `eve.json` into Elasticsearch. The goal was to stand up a second SIEM stack alongside my existing Splunk setup, using the same underlying log sources, so I could eventually compare how the two platforms handle identical data.

## 🖥️ Lab Setup

- Kali Linux VM, Elasticsearch and Kibana installed via apt (package/systemd method, not the tarball)
- Elasticsearch 8.19.18, security enabled by default (`xpack.security.enabled: true`) with auto-generated TLS certs
- Filebeat installed via apt, configured to ship two log sources into Elasticsearch
- Both Elasticsearch and Kibana run as systemd services, consistent with how I run Splunk and Suricata in my other repos

## ⚙️ Installation

### Elasticsearch & Kibana

Installed via the Elastic apt repository, then started as systemd services:

```bash
sudo systemctl enable --now elasticsearch
sudo systemctl enable --now kibana
```

Elasticsearch generates its own TLS certificates and a random `elastic` superuser password on first boot. If that first-boot output isn't captured, the password can be reset any time:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

## 🐛 Problems I Hit (and How I Fixed Them)

Getting Kibana connected to Elasticsearch was not a clean, one-shot process. I hit three distinct issues in sequence, and I think documenting all three is more useful than pretending it just worked.

### 1. Wrong user for Kibana's manual configuration
Kibana's browser-based setup wizard rejected the `elastic` superuser outright: *"User 'elastic' can't be used as the Kibana system user."* Elastic deliberately separates the human superuser account from the service account Kibana uses internally. The fix was resetting the password for the dedicated `kibana_system` account instead:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
```

### 2. Browser setup wizard failing silently
Even with the correct `kibana_system` credentials, the browser wizard returned "Couldn't configure Elastic, retry or update the kibana.yml file manually." Rather than fight the wizard, I configured `/etc/kibana/kibana.yml` directly:

```yaml
server.host: "localhost"
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "<kibana_system password>"
elasticsearch.ssl.certificateAuthorities: ["/etc/elasticsearch/certs/http_ca.crt"]
```

The most important detail here: I initially left `elasticsearch.hosts` as `http://localhost:9200`, which silently failed since Elasticsearch's HTTP layer only accepts HTTPS with security enabled. Changing it to `https://` was the actual fix.

### 3. Permission denied reading the CA certificate
After fixing the scheme, Kibana crash-looped with:

```
FATAL Error: EACCES: permission denied, open '/etc/elasticsearch/certs/http_ca.crt'
```

The cert was owned `root:elasticsearch` with group-only read access, and the Kibana service runs as its own dedicated `kibana` user, which wasn't in that group. Fixed by adding the `kibana` user to the `elasticsearch` group:

```bash
sudo usermod -aG elasticsearch kibana
sudo systemctl reset-failed kibana
sudo systemctl restart kibana
```

Group membership changes require a service restart to take effect, and the restart also cleared systemd's start-limit lockout from the repeated crash loop.

## 📥 Filebeat Configuration

Installed Filebeat and configured `/etc/filebeat/filebeat.yml` to collect two sources:

```yaml
filebeat.inputs:
  - type: filestream
    id: auth-log
    enabled: true
    paths:
      - /var/log/auth.log

  - type: filestream
    id: suricata-eve
    enabled: true
    paths:
      - /var/log/suricata/eve.json
    parsers:
      - ndjson:
          keys_under_root: true
          add_error_key: true

output.elasticsearch:
  hosts: ["localhost:9200"]
  preset: balanced
  protocol: "https"
  username: "elastic"
  password: "<elastic password>"
  ssl.certificate_authorities: ["/etc/elasticsearch/certs/http_ca.crt"]

setup.kibana:
  host: "localhost:5601"
```

Filebeat runs as root by default, so it didn't hit the same certificate permission issue Kibana did.

## ✅ Validation

Tested config and connectivity before starting the service:

```bash
sudo filebeat test config
sudo filebeat test output
```

Both passed clean: `Config OK`, TLS handshake succeeded (TLSv1.3), and Filebeat confirmed talking to Elasticsearch 8.19.18.

Ran setup to load the index template, ILM policy, and dashboards:

```bash
sudo filebeat setup -e
```

This created the `filebeat-8.19.18` index template and data stream successfully.

One thing worth noting: running `filebeat setup -e` only executes Filebeat once as a one-off process. It does not start the persistent service. I initially assumed setup implied the service was running, but `systemctl status filebeat` showed `inactive (dead)` immediately afterward, and a document count check against Elasticsearch returned 0. The actual fix was simply:

```bash
sudo systemctl enable --now filebeat
```

After that, `systemctl status filebeat` showed `active (running)`, and a follow-up count check confirmed documents were flowing into the `filebeat-*` index from both `auth.log` and `eve.json`.

## 🧠 What I Learned

The biggest lesson here wasn't really about Elasticsearch or Kibana specifically, it was about the distinction between running a one-off setup command and running a persistent service. `filebeat setup -e` does real, necessary work (index templates, ILM policies, dashboards), but it's easy to mistake its successful completion for "the service is now running," which isn't the case. I also came away with a much clearer picture of how Elastic's security model separates concerns: the `elastic` superuser for administration, `kibana_system` for Kibana's internal service account, and file permissions scoped tightly per service user rather than a shared "elasticsearch does everything" account. That's a more deliberate security posture than I expected going in, and debugging through three distinct failure points (wrong account, wrong URL scheme, wrong file permissions) gave me a much better mental model of how the stack's components actually authenticate to each other.

## 🎯 SOC Relevance

Running a second SIEM stack against the same log sources as my Splunk setup gives me a genuine basis for comparison rather than a hypothetical one. Many SOC teams run ELK either as their primary stack or alongside a commercial tool like Splunk for cost reasons, so hands-on experience with Elastic's authentication model, index management, and Filebeat-based ingestion is directly transferable. The permission and configuration issues I hit here weren't contrived, they're the exact kind of first-deployment friction a junior analyst or engineer would encounter standing up a new ELK instance in a real environment, and knowing how to read the error messages (EACCES, TLS handshake failures, start-limit-hit) and trace them to root cause is a core troubleshooting skill regardless of which SIEM is in front of you.


