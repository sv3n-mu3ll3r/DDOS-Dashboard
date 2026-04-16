# F5 DDoS Dashboard

Kibana dashboards and ELK stack configuration for visualizing F5 Networks DDoS events and DoS vector statistics.

<img src="images/image1.png">
<img src="images/image2.png">

---

## Quick Start

➡️ **[INSTALL.md](INSTALL.md)** — Complete step-by-step installation guide for **ELK 8.x on Ubuntu 22.04**

> The original README contained ELK **7.x** instructions. This repo has been updated for **ELK 8.x** with security enabled by default. Key differences are documented in [INSTALL.md](INSTALL.md).

---

## Repository Contents

| File | Description |
|---|---|
| `export.ndjson` | Kibana dashboard, visualizations and saved objects — import via Kibana UI |
| `logstash.conf` | Logstash pipeline config to parse F5 BIG-IP syslog messages |
| `template_1` / `template_1.json` | Elasticsearch index template for DDoS events (`f5-ddos-i-*`) |
| `template-stats` | Elasticsearch index template for DoS vector statistics (`f5-stats-ddos-t-*`) |
| `node1-pipline` | Elasticsearch ingest pipeline — **must be loaded before Logstash starts** |
| `syslog-ng.conf` | syslog-ng configuration for receiving F5 syslog (UDP 5556/5557/5558) |
| `INSTALL.md` | Complete installation guide for ELK 8.x |

---

## Architecture

```
F5 BIG-IP
  │
  │  UDP Syslog (port 5558)
  ▼
syslog-ng  →  /var/log/f5ddos_all.log
                     │
                     ▼
                  Logstash  →  Elasticsearch  →  Kibana
```

**Syslog ports on ELK server:**
- `5556/UDP` — DoS vector statistics
- `5557/UDP` — DDoS key-value events
- `5558/UDP` — Combined (main input for Logstash)

---

## BIG-IP Logging Configuration

Replace `<ELK-SERVER-IP>` with your ELK server's IP address:

```bash
# 1. Logging pool
tmsh create ltm pool ddos-logging-pool members add { <ELK-SERVER-IP>:5558 }

# 2. High-speed log destination
tmsh create sys log-config destination remote-high-speed-log ddos-hsl \
  pool-name ddos-logging-pool protocol udp

# 3. Splunk-format destination
tmsh create sys log-config destination splunk ddos-splunk \
  forward-to ddos-hsl

# 4. Log publisher
tmsh create sys log-config publisher ddos-publisher \
  destinations add { ddos-splunk }

# 5. Security log profile
tmsh create security log profile ddos-log-profile \
  dos-network-publisher ddos-publisher \
  ip-intelligence { log-publisher ddos-publisher } \
  network add { local-dos { publisher ddos-publisher } }

# 6. Apply to global network DoS
tmsh modify security log profile global-network \
  dos-network-publisher ddos-publisher \
  ip-intelligence { log-publisher ddos-publisher }

tmsh modify security dos device-config dos-device-config \
  log-publisher ddos-publisher

# 7. Save
tmsh save sys config
```

---

## DoS Statistics Collection (BIG-IP crontab)

Add to BIG-IP crontab — replace `<ELK-SERVER-IP>` and `<PORT>`:

```bash
* * * * * nb_of_tmms=$(tmsh show sys tmm-info | grep Sys::TMM | wc -l); \
tmctl -c dos_stat -s context_name,vector_name,attack_detected,stats_rate,drops_rate, \
int_drops_rate,ba_stats_rate,ba_drops_rate,bd_stats_rate,bd_drops_rate,detection, \
mitigation_low,mitigation_high,detection_ba,mitigation_ba_low,mitigation_ba_high, \
detection_bd,mitigation_bd_low,mitigation_bd_high | grep -v "context_name" | \
sed '/^$/d' | sed "s/$/,$nb_of_tmms/g" | logger -n <ELK-SERVER-IP> --udp --port 5558
```

> **Better approach:** Use an F5 external monitor instead of crontab:  
> https://support.f5.com/csp/article/K71282813

---

## ELK 8.x — Key Differences from 7.x

| | ELK 7.x | ELK 8.x |
|---|---|---|
| Security | Manual opt-in | **Enabled by default** |
| TLS | Optional | **Mandatory** |
| Java | Separate install | **Bundled** |
| GPG key import | `apt-key add` (deprecated) | `gpg --dearmor` |
| GeoIP filter | No `target` needed | **`target => "geoip"` required** |
| ECS compatibility | v1 | v8 (disable for this setup: `pipeline.ecs_compatibility: disabled`) |
| Ingest pipeline | Optional | **Required — load before Logstash** |

---

## License

MIT — see [LICENSE](LICENSE)
