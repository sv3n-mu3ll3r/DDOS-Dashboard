# F5 DDoS Dashboard — Installation Guide (ELK 8.x)

> Updated installation guide for **Elasticsearch / Kibana / Logstash 8.x** on **Ubuntu 22.04 LTS**.  
> The original repo uses ELK 7.x — this guide covers the current stack with security enabled by default.

---

## Prerequisites

- Ubuntu 22.04 LTS (fresh install recommended)
- Min. 4 GB RAM (8 GB recommended)
- Ports open: `5601` (Kibana), `5556–5558` (syslog UDP from BIG-IP)
- Root / sudo access

---

## Step 1 — Add Elastic Repository

```bash
# Import GPG key (new method — apt-key is deprecated)
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt-get update
```

---

## Step 2 — Install ELK Stack + syslog-ng

```bash
sudo apt-get install -y elasticsearch kibana logstash syslog-ng
```

> **Note:** ELK 8.x bundles its own JDK — no separate Java installation needed.

---

## Step 3 — Configure Elasticsearch

### `/etc/elasticsearch/elasticsearch.yml`

```yaml
cluster.name: ddos-elk
node.name: <your-hostname>
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: localhost
http.port: 9200
discovery.type: single-node
action.destructive_requires_name: true

# Security (enabled by default in 8.x)
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http.p12
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport.p12
xpack.security.transport.ssl.truststore.path: certs/transport.p12
```

### Set JVM Heap — `/etc/elasticsearch/jvm.options.d/heap.options`

```
-Xms3g
-Xmx3g
```

> Rule of thumb: 50% of available RAM, max 30 GB.

### Kernel tuning (required for Elasticsearch)

```bash
sudo sysctl -w vm.swappiness=1
sudo sysctl -w vm.max_map_count=262144

# Make persistent
echo "vm.swappiness=1" | sudo tee /etc/sysctl.d/99-elasticsearch.conf
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.d/99-elasticsearch.conf
```

### Start Elasticsearch

```bash
sudo systemctl enable --now elasticsearch
```

### Get the elastic user password

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -b
# Note: Save the output password — you will need it!
```

---

## Step 4 — Configure Kibana

### Copy CA certificate (so Kibana can verify ES)

```bash
sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/kibana/elasticsearch-ca.crt
sudo chown root:kibana /etc/kibana/elasticsearch-ca.crt
sudo chmod 640 /etc/kibana/elasticsearch-ca.crt
```

### Reset kibana_system password

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -b
# Save this password!
```

### `/etc/kibana/kibana.yml`

```yaml
server.port: 5601
server.host: '<your-server-ip>'        # IP Kibana listens on
server.name: 'ddos-elk-kibana'
elasticsearch.hosts: ['https://localhost:9200']
elasticsearch.username: 'kibana_system'
elasticsearch.password: '<kibana_system-password>'
elasticsearch.ssl.certificateAuthorities: ['/etc/kibana/elasticsearch-ca.crt']
elasticsearch.ssl.verificationMode: certificate
logging.appenders.rolling-file.type: rolling-file
logging.appenders.rolling-file.fileName: /var/log/kibana/kibana.log
logging.appenders.rolling-file.layout.type: json
logging.root.appenders: [rolling-file]
pid.file: /run/kibana/kibana.pid
```

### Start Kibana

```bash
sudo systemctl enable --now kibana
# Kibana takes ~60 seconds to fully start
```

---

## Step 5 — Create Logstash API Key

> In ELK 8.x, Logstash authenticates to Elasticsearch via API key (more secure than user/password).

```bash
# Replace <elastic-password> with the password from Step 3
curl -k -u "elastic:<elastic-password>" \
  -X POST "https://localhost:9200/_security/api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "logstash-ddos",
    "role_descriptors": {
      "logstash_writer": {
        "cluster": ["monitor", "manage_index_templates"],
        "index": [{
          "names": ["f5-*"],
          "privileges": ["write", "create_index", "create", "auto_configure"]
        }]
      }
    }
  }'
# Save the "encoded" value from the response!
```

---

## Step 6 — Configure Logstash

### Copy CA certificate for Logstash

```bash
sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/logstash/elasticsearch-ca.crt
sudo chown logstash:logstash /etc/logstash/elasticsearch-ca.crt
```

### `/etc/logstash/conf.d/ddos.conf`

```ruby
input {
  file {
    path => ["/var/log/f5ddos_all.log"]
    sincedb_path => "/var/log/logstash/sincedb"
    start_position => "end"
  }
}

filter {
  grok {
    match => {"message" => ["%{SYSLOGTIMESTAMP:syslog_timestamp}%{SPACE}%{SYSLOGHOST:eventSourceIP}%{SPACE}%{GREEDYDATA:xdata}"]}
  }
  # ... (see logstash.conf in this repo for full filter)
}

output {
  if "f5ddoskv" in [tags] {
    elasticsearch {
      hosts => ["https://localhost:9200"]
      ssl_enabled => true
      ssl_certificate_authorities => ["/etc/logstash/elasticsearch-ca.crt"]
      api_key => "<encoded-api-key>"       # from Step 5
      index => "f5-ddos-i-%{+YYYY.MM.dd}"
    }
  }
  if "f5ddos_stats" in [tags] {
    elasticsearch {
      hosts => ["https://localhost:9200"]
      ssl_enabled => true
      ssl_certificate_authorities => ["/etc/logstash/elasticsearch-ca.crt"]
      api_key => "<encoded-api-key>"       # from Step 5
      index => "f5-stats-ddos-t-%{+YYYY.MM.dd}"
    }
  }
}
```

### Logstash JVM Heap — `/etc/logstash/jvm.options.d/heap.options`

```
-Xms1g
-Xmx1g
```

### Start Logstash

```bash
sudo systemctl enable --now logstash
```

---

## Step 7 — Configure syslog-ng

### `/etc/syslog-ng/conf.d/f5ddos.conf`

```
# F5 BIG-IP DDoS syslog sources (UDP)
source s_f5ddos_stats { udp(ip(0.0.0.0) port(5556) flags(no-hostname)); };
destination d_f5ddos_stats { file("/var/log/f5ddos_stats.log" owner("root") group("root") perm(0644)); };
log { source(s_f5ddos_stats); destination(d_f5ddos_stats); };

source s_f5ddos_kv { udp(ip(0.0.0.0) port(5557) flags(no-hostname)); };
destination d_f5ddos_kv { file("/var/log/f5ddos_kv.log" owner("root") group("root") perm(0644)); };
log { source(s_f5ddos_kv); destination(d_f5ddos_kv); };

source s_f5ddos_all { udp(ip(0.0.0.0) port(5558) flags(no-hostname)); };
destination d_f5ddos_all { file("/var/log/f5ddos_all.log" owner("root") group("root") perm(0644)); };
log { source(s_f5ddos_all); destination(d_f5ddos_all); };
```

### Create log files and start syslog-ng

```bash
sudo touch /var/log/f5ddos_all.log /var/log/f5ddos_stats.log /var/log/f5ddos_kv.log
sudo chmod 644 /var/log/f5ddos_all.log /var/log/f5ddos_stats.log /var/log/f5ddos_kv.log
sudo systemctl enable --now syslog-ng
```

---

## Step 8 — Configure BIG-IP Logging

Run these commands on your F5 BIG-IP (replace `<ELK-SERVER-IP>` with your server IP):

```bash
# 1. Create logging pool
tmsh create ltm pool ddos-logging-pool members add { <ELK-SERVER-IP>:5558 }

# 2. Create high-speed log destination
tmsh create sys log-config destination remote-high-speed-log ddos-hsl \
  pool-name ddos-logging-pool protocol udp

# 3. Create formatted log destination
tmsh create sys log-config destination splunk ddos-splunk \
  forward-to ddos-hsl

# 4. Create log publisher
tmsh create sys log-config publisher ddos-publisher \
  destinations add { ddos-splunk }

# 5. Create security log profile
tmsh create security log profile ddos-log-profile \
  dos-network-publisher ddos-publisher \
  ip-intelligence { log-publisher ddos-publisher } \
  network add { local-dos { publisher ddos-publisher } }

# 6. Apply to global network DoS
tmsh modify security dos device-config dos-device-config log-publisher ddos-publisher

# 7. Save config
tmsh save sys config
```

---

## Step 9 — Import Kibana Dashboard

1. Open Kibana: `http://<server-ip>:5601`
2. Login with user `elastic` and the password from Step 3
3. Go to **Stack Management → Saved Objects → Import**
4. Upload `export.ndjson` from this repository
5. Done — navigate to **Dashboard** to see the F5 DDoS Dashboard

---

## Step 10 — Log Rotation & Maintenance

### Logrotate for f5ddos logs

```bash
sudo tee /etc/logrotate.d/f5ddos << 'EOF'
/var/log/f5ddos_all.log
/var/log/f5ddos_stats.log
/var/log/f5ddos_kv.log
{
    daily
    size 1G
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    rotate 1
}
EOF
```

### Automated index cleanup (crontab)

```bash
# Check every 30 minutes if log exceeds 1 GB
(crontab -l; echo "*/30 * * * * /usr/sbin/logrotate /etc/logrotate.d/f5ddos") | crontab -

# Delete old ES indices daily at 2am
(crontab -l; echo "0 2 * * * /usr/local/bin/es-index-cleanup.sh") | crontab -
```

See `es-index-cleanup.sh` in this repository for the cleanup script.

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Kibana `EACCES` on cert | Wrong file permissions | Copy cert to `/etc/kibana/` with kibana ownership |
| ES startup timeout | `TimeoutStartSec` too short | Add override: `TimeoutStartSec=300` |
| Logstash not indexing | API key wrong or SSL error | Check `/var/log/logstash/logstash-plain.log` |
| syslog-ng not receiving | UDP port blocked | Check `sudo ss -ulnp \| grep 555` |

---

## Key Differences: ELK 7.x → 8.x

| Topic | ELK 7.x | ELK 8.x |
|---|---|---|
| Security | Disabled by default | **Enabled by default** |
| Java | Requires separate install | **Bundled** |
| GPG key import | `apt-key add` (deprecated) | `gpg --dearmor` |
| Kibana setup | Manual config | Enrollment token or manual config |
| Logstash auth | user/password | **API key** (recommended) |
| HTTP | Plain HTTP | **HTTPS with TLS** |
| Logstash SSL option | `ssl => true` | `ssl_enabled => true` |
