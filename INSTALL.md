# F5 DDoS Dashboard — Installation Guide (ELK 8.x)

> Getestete Installationsanleitung für **ELK 8.x** auf **Ubuntu 22.04 LTS**.  
> Basiert auf dem originalen [elk-f5ddos/DDOS-Dashboard](https://github.com/elk-f5ddos/DDOS-Dashboard) — aktualisiert für ELK 8.x mit Security, TLS und allen bekannten Fixes.

---

## Voraussetzungen

| | |
|---|---|
| **OS** | Ubuntu 22.04 LTS (frische Installation empfohlen) |
| **RAM** | Min. 4 GB (8 GB empfohlen) |
| **Disk** | Min. 50 GB |
| **Ports** | `5601` (Kibana), `5556–5558/UDP` (syslog vom BIG-IP) |
| **Zugang** | root / sudo |

---

## Schritt 1 — Elastic Repository hinzufügen

```bash
# GPG Key (neue Methode — apt-key ist deprecated)
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Repository hinzufügen
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/8.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt-get update
```

---

## Schritt 2 — ELK Stack + syslog-ng installieren

```bash
sudo apt-get install -y elasticsearch kibana logstash syslog-ng
```

> **Hinweis:** ELK 8.x bringt ein eigenes JDK mit — kein separates Java nötig.

---

## Schritt 3 — System-Tuning (Kernel)

```bash
# Sofort setzen
sudo sysctl -w vm.swappiness=1
sudo sysctl -w vm.max_map_count=262144

# Dauerhaft
sudo tee /etc/sysctl.d/99-elasticsearch.conf << 'EOF'
vm.swappiness=1
vm.max_map_count=262144
EOF
```

---

## Schritt 4 — Elasticsearch konfigurieren

### `/etc/elasticsearch/elasticsearch.yml`

```yaml
cluster.name: ddos-elk
node.name: <hostname>              # z.B. ip-10-1-1-5
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: localhost
http.port: 9200
discovery.type: single-node
action.destructive_requires_name: true
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http.p12
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport.p12
xpack.security.transport.ssl.truststore.path: certs/transport.p12
```

### JVM Heap — `/etc/elasticsearch/jvm.options.d/heap.options`

```
-Xms3g
-Xmx3g
```

> Faustregel: 50% des verfügbaren RAM, max. 30 GB.

### Elasticsearch starten + Passwort setzen

```bash
sudo systemctl enable --now elasticsearch

# Passwort für elastic User generieren (AUSGABE SPEICHERN!)
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -b
```

---

## Schritt 5 — Kibana konfigurieren

### CA-Zertifikat kopieren

```bash
# Wichtig: In Kibana-eigenes Verzeichnis kopieren (Berechtigungsproblem!)
sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/kibana/elasticsearch-ca.crt
sudo chown root:kibana /etc/kibana/elasticsearch-ca.crt
sudo chmod 640 /etc/kibana/elasticsearch-ca.crt
```

### Passwort für kibana_system setzen

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -b
# AUSGABE SPEICHERN!
```

### `/etc/kibana/kibana.yml`

```yaml
server.port: 5601
server.host: '<server-ip>'           # IP auf der Kibana lauscht
server.name: 'ddos-elk-kibana'
elasticsearch.hosts: ['https://localhost:9200']
elasticsearch.username: 'kibana_system'
elasticsearch.password: '<kibana_system-passwort>'
elasticsearch.ssl.certificateAuthorities: ['/etc/kibana/elasticsearch-ca.crt']
elasticsearch.ssl.verificationMode: certificate
logging.appenders.rolling-file.type: rolling-file
logging.appenders.rolling-file.fileName: /var/log/kibana/kibana.log
logging.appenders.rolling-file.layout.type: json
logging.root.appenders: [rolling-file]
pid.file: /run/kibana/kibana.pid
```

### Kibana starten

```bash
sudo systemctl enable --now kibana
# Kibana braucht ~60 Sekunden bis es vollständig gestartet ist
```

---

## Schritt 6 — Index-Templates und Ingest-Pipeline laden

> **Wichtig:** Diese müssen VOR dem ersten Logstash-Start geladen werden, sonst sind die Field-Mappings falsch und das Dashboard funktioniert nicht.

```bash
# Templates aus diesem Repo herunterladen
curl -s -o /tmp/template_1_body.json \
  https://raw.githubusercontent.com/sv3n-mu3ll3r/DDOS-Dashboard/main/template_1.json
# Erste Zeile (PUT ...) entfernen
tail -n +2 /tmp/template_1_body.json > /tmp/t1.json

curl -s -o /tmp/template_stats_raw.json \
  https://raw.githubusercontent.com/sv3n-mu3ll3r/DDOS-Dashboard/main/template-stats
tail -n +2 /tmp/template_stats_raw.json > /tmp/ts.json

curl -s -o /tmp/node1_raw.json \
  https://raw.githubusercontent.com/sv3n-mu3ll3r/DDOS-Dashboard/main/node1-pipline
tail -n +2 /tmp/node1_raw.json > /tmp/n1.json

# In Elasticsearch laden
ES_PASS="<elastic-passwort>"

curl -k -u "elastic:${ES_PASS}" \
  -X PUT 'https://localhost:9200/_index_template/template_1' \
  -H 'Content-Type: application/json' -d @/tmp/t1.json

curl -k -u "elastic:${ES_PASS}" \
  -X PUT 'https://localhost:9200/_index_template/template-stats' \
  -H 'Content-Type: application/json' -d @/tmp/ts.json

curl -k -u "elastic:${ES_PASS}" \
  -X PUT 'https://localhost:9200/_ingest/pipeline/node1-pipeline' \
  -H 'Content-Type: application/json' -d @/tmp/n1.json
```

Erwartete Antwort jeweils: `{"acknowledged":true}`

---

## Schritt 7 — Logstash konfigurieren

### ECS-Kompatibilität deaktivieren — `/etc/logstash/logstash.yml`

```bash
echo 'pipeline.ecs_compatibility: disabled' | sudo tee -a /etc/logstash/logstash.yml
```

> **Wichtig für ELK 8.x:** Ohne diese Einstellung schickt Logstash `host` als Objekt `{name: hostname}`, was mit dem Index-Template (erwartet `text`) kollidiert und alle Dokumente ablehnt.

### JVM Heap — `/etc/logstash/jvm.options.d/heap.options`

```bash
sudo mkdir -p /etc/logstash/jvm.options.d
sudo tee /etc/logstash/jvm.options.d/heap.options << 'EOF'
-Xms1g
-Xmx1g
EOF
```

### CA-Zertifikat für Logstash

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

  if "root:" not in [xdata] {
    mutate { add_tag => ["f5ddoskv"] }
    kv { source => "xdata" field_split => "," value_split => "=" }
    if "N/A" in [dest_ip] { mutate { update => {"dest_ip" => "0.0.0.0"} } }

    if "23003173" in [errdefs_msgno] {
      mutate {
        add_field => {"dos_attack_name" => "%{sig_name}"}
        add_field => {"dos_src" => "%{bdos_family}"}
        add_field => {"action" => "%{errdefs_msg_name}"}
      }
    }
    if "Allow" in [action] and "23003142" not in [errdefs_msgno] {
      mutate { add_field => {"context_type" => "%{context_name}"} }
      mutate { add_field => {"SuspiciousPackets" => "%{dos_packets_received}"} }
      if "Device" not in [context_name] { mutate { update => {"context_type" => "Virtual Server"} } }
    }
    if "Drop" in [action] and "23003142" not in [errdefs_msgno] {
      mutate { add_field => {"context_type" => "%{context_name}"} }
      if "Device" not in [context_name] { mutate { update => {"context_type" => "Virtual Server"} } }
    }
    if "23003142" in [errdefs_msgno] {
      mutate { add_field => {"dos_attack_name" => "IPI:%{ip_intelligence_policy_name}(%{ip_intelligence_threat_name})"} }
      mutate { add_field => {"dos_src" => "%{errdefs_msg_name}"} }
      if "/Common/" in [dos_attack_name] { mutate { gsub => ["dos_attack_name", "/Common/", ""] } }
      mutate { add_field => {"IPIallowpckts" => "0"} }
      mutate { add_field => {"IPIdroppedpckts" => "0"} }
      if "Accept" in [action] {
        mutate {
          update => {"IPIallowpckts" => "1"} update => {"IPIdroppedpckts" => "0"}
          add_field => {"dos_packets_received" => "1"} add_field => {"dos_packets_dropped" => "0"}
        }
      }
      if "Drop" in [action] {
        mutate {
          update => {"IPIallowpckts" => "0"} update => {"IPIdroppedpckts" => "1"}
          add_field => {"dos_packets_received" => "1"} add_field => {"dos_packets_dropped" => "1"}
        }
      }
    }
    date { match => ["date_time", "MMM dd yyyy HH:mm:ss"] target => "date_time" }
    mutate { split => {"metric_bins" => "\s"} }
    if [source_ip] and [source_ip] !~ "(^127\.0\.0\.1)|(^10\.)|(^172\.1[6-9]\.)|(^172\.2[0-9]\.)|(^172\.3[0-1]\.)|(^192\.168\.)|(^169\.254\.)" {
      # ELK 8.x: target-Feld ist Pflicht!
      geoip { source => "source_ip" target => "geoip" }
    }
    if "L4 BDOS Dynamic" in [action] { mutate { gsub => ["action", "L4 BDOS Dynamic", ""] } }
    if "dos-common" in [dos_attack_name] { mutate { gsub => ["dos_attack_name", "/Common/dos-common/", ""] } }
    if "BDOS Signature-Based" in [dos_src] { mutate { add_field => {"status" => "RE-USED"} } }
  }

  if "root:" in [xdata] {
    mutate { add_tag => ["f5ddos_stats"] }
    mutate { gsub => ["xdata", ":", ","] }
    csv {
      source => "xdata"
      separator => ","
      columns => [
        "user_process","context_name","dos_attack_name","attack_detected",
        "stats_rate","drops_rate","int_drops_rate","ba_stats_rate","ba_drops_rate",
        "bd_stats_rate","bd_drops_rate","detection","mitigation_low","mitigation_high",
        "detection_ba","mitigation_ba_low","mitigation_ba_high",
        "detection_bd","mitigation_bd_low","mitigation_bd_high","tmm_count"
      ]
    }
    ruby { code => 'event.set("overall_drop_rate", event.get("drops_rate").to_i + event.get("int_drops_rate").to_i + event.get("ba_drops_rate").to_i + event.get("bd_drops_rate").to_i)' }
    ruby { code => 'event.set("total_drop_rate", event.get("drops_rate").to_i + event.get("int_drops_rate").to_i)' }
    ruby { code => 'event.set("detection_all_tmm", event.get("detection").to_i * event.get("tmm_count").to_i)' }
    ruby { code => 'sum = event.get("mitigation_low").to_i + event.get("mitigation_high").to_i; avg = sum / 2; event.set("mitigation_curr_all_tmm", avg * event.get("tmm_count").to_i)' }
    ruby { code => 'sum = event.get("mitigation_ba_low").to_i + event.get("mitigation_ba_high").to_i; avg = sum / 2; event.set("mitigation_ba_all_tmm", avg * event.get("tmm_count").to_i)' }
    ruby { code => 'sum = event.get("mitigation_bd_low").to_i + event.get("mitigation_bd_high").to_i; avg = sum / 2; event.set("mitigation_bd_all_tmm", avg * event.get("tmm_count").to_i)' }
    if "dos-common" in [dos_attack_name] { mutate { gsub => ["dos_attack_name", "/Common/dos-common/", ""] } }
    if "Device" in [context_name] {
      ruby { code => 'event.set("incoming_packet_rate", event.get("stats_rate").to_i + event.get("ba_drops_rate").to_i + event.get("bd_drops_rate").to_i)' }
    } else {
      ruby { code => 'event.set("incoming_packet_rate", event.get("drops_rate").to_i + event.get("int_drops_rate").to_i + event.get("ba_drops_rate").to_i + event.get("bd_drops_rate").to_i)' }
    }
    if "TCP half open" in [dos_attack_name] {
      mutate {
        update => {"incoming_packet_rate" => "0"}
        update => {"overall_drop_rate" => "0"}
        update => {"total_drop_rate" => "0"}
      }
    }
  }
}

output {
  if "f5ddoskv" in [tags] {
    elasticsearch {
      hosts => ["https://localhost:9200"]
      ssl_enabled => true
      ssl_certificate_authorities => ["/etc/logstash/elasticsearch-ca.crt"]
      user => "elastic"
      password => "<elastic-passwort>"
      manage_template => false
      index => "f5-ddos-i-%{+YYYY.MM.dd}"
    }
  }
  if "f5ddos_stats" in [tags] {
    elasticsearch {
      hosts => ["https://localhost:9200"]
      ssl_enabled => true
      ssl_certificate_authorities => ["/etc/logstash/elasticsearch-ca.crt"]
      user => "elastic"
      password => "<elastic-passwort>"
      manage_template => false
      index => "f5-stats-ddos-t-%{+YYYY.MM.dd}"
    }
  }
}
```

### Logstash starten

```bash
sudo systemctl enable --now logstash
```

---

## Schritt 8 — syslog-ng konfigurieren

### `/etc/syslog-ng/conf.d/f5ddos.conf` (neu erstellen)

```
# F5 BIG-IP DDoS Syslog Quellen (UDP)
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

### Log-Dateien anlegen + syslog-ng starten

```bash
sudo touch /var/log/f5ddos_all.log /var/log/f5ddos_stats.log /var/log/f5ddos_kv.log
sudo chmod 644 /var/log/f5ddos_all.log /var/log/f5ddos_stats.log /var/log/f5ddos_kv.log
sudo systemctl enable --now syslog-ng
```

### syslog-ng daemon.log Rotation auf täglich setzen

```bash
# Den zweiten Block in /etc/logrotate.d/syslog-ng von weekly auf daily ändern
sudo sed -i '/\/var\/log\/daemon\.log/{
  :loop
  N
  /}/!b loop
  s/weekly/daily/
  s/rotate 4/rotate 7/
}' /etc/logrotate.d/syslog-ng
```

---

## Schritt 9 — Log-Rotation einrichten

### `/etc/logrotate.d/f5ddos`

```
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
    dateext
    dateformat -%Y%m%d
    extension .log
    create 0644 root root
}
```

### `/etc/logrotate.d/kibana`

```
/var/log/kibana/kibana.log
{
    daily
    size 100M
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    rotate 7
}
```

### `/etc/logrotate.d/logstash`

```
/var/log/logstash/logstash-plain.log
/var/log/logstash/logstash-deprecation.log
{
    daily
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    rotate 7
}
```

### Systemd Timer mit Persistent=true (holt verpasste Rotationen nach)

```bash
sudo tee /etc/systemd/system/logrotate.timer << 'EOF'
[Unit]
Description=Daily rotation of log files

[Timer]
OnCalendar=daily
AccuracySec=1h
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo tee /etc/systemd/system/logrotate.service << 'EOF'
[Unit]
Description=Rotate log files

[Service]
Type=oneshot
ExecStart=/usr/sbin/logrotate /etc/logrotate.conf
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now logrotate.timer
```

---

## Schritt 10 — Crontab einrichten

```bash
# f5ddos_all.log: alle 30 Minuten auf Größe prüfen
(sudo crontab -l 2>/dev/null; echo "*/30 * * * * /usr/sbin/logrotate /etc/logrotate.d/f5ddos") | sudo crontab -

# Beim Booten sofort prüfen (wichtig wenn Server nachts aus)
(sudo crontab -l 2>/dev/null; echo "@reboot /usr/sbin/logrotate /etc/logrotate.d/f5ddos") | sudo crontab -

# Elasticsearch Indices täglich um 02:00 aufräumen
(sudo crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/es-index-cleanup.sh") | sudo crontab -
```

---

## Schritt 11 — ES Index Cleanup Script

```bash
sudo tee /usr/local/bin/es-index-cleanup.sh << 'SCRIPT'
#!/bin/bash
RETENTION_DAYS=1
ES_HOST="https://localhost:9200"
ES_USER="elastic"
ES_PASS="<elastic-passwort>"
CACERT="/etc/logstash/elasticsearch-ca.crt"
LOGFILE="/var/log/es-index-cleanup.log"
INDEX_PATTERNS=("f5-ddos-i" "f5-stats-ddos-t")

log() { echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOGFILE"; }
log "=== Cleanup gestartet ==="

if ! curl -s -k --max-time 10 "$ES_HOST" > /dev/null 2>&1; then
  log "FEHLER: Elasticsearch nicht erreichbar"; exit 1
fi

CUTOFF_DATE=$(date -d "-${RETENTION_DAYS} days" '+%Y.%m.%d')
ALL_INDICES=$(curl -s -k -u "${ES_USER}:${ES_PASS}" "$ES_HOST/_cat/indices?h=index" 2>/dev/null)
DELETED=0; ERRORS=0

for PATTERN in "${INDEX_PATTERNS[@]}"; do
  for INDEX in $(echo "$ALL_INDICES" | grep "^${PATTERN}-" | sort); do
    INDEX_DATE=$(echo "$INDEX" | grep -oP '\d{4}\.\d{2}\.\d{2}$')
    [ -z "$INDEX_DATE" ] && continue
    if [[ "$INDEX_DATE" < "$CUTOFF_DATE" ]]; then
      RESP=$(curl -s -k -o /dev/null -w '%{http_code}' \
        -u "${ES_USER}:${ES_PASS}" -X DELETE "$ES_HOST/$INDEX")
      if [ "$RESP" = "200" ]; then
        log "GELÖSCHT: $INDEX"; ((DELETED++))
      else
        log "FEHLER: $INDEX (HTTP $RESP)"; ((ERRORS++))
      fi
    fi
  done
done
log "Ergebnis: ${DELETED} gelöscht, ${ERRORS} Fehler"
SCRIPT

sudo chmod +x /usr/local/bin/es-index-cleanup.sh
```

---

## Schritt 12 — systemd Journal begrenzen

```bash
sudo tee -a /etc/systemd/journald.conf << 'EOF'
SystemMaxUse=500M
SystemKeepFree=1G
MaxFileSec=1week
EOF
sudo systemctl restart systemd-journald
```

---

## Schritt 13 — Kibana Dashboard importieren

```bash
# export.ndjson aus diesem Repo importieren
curl -X POST \
  -u "elastic:<elastic-passwort>" \
  "http://<server-ip>:5601/api/saved_objects/_import?overwrite=true" \
  -H "kbn-xsrf: true" \
  --form file=@export.ndjson
```

Oder in Kibana: **Stack Management → Saved Objects → Import → export.ndjson**

---

## Schritt 14 — BIG-IP Logging konfigurieren

Auf dem F5 BIG-IP (tmsh), `<ELK-IP>` durch die Server-IP ersetzen:

```bash
# 1. Logging Pool
tmsh create ltm pool ddos-logging-pool members add { <ELK-IP>:5558 }

# 2. High-Speed Log Destination
tmsh create sys log-config destination remote-high-speed-log ddos-hsl \
  pool-name ddos-logging-pool protocol udp

# 3. Splunk-Format Destination
tmsh create sys log-config destination splunk ddos-splunk \
  forward-to ddos-hsl

# 4. Log Publisher
tmsh create sys log-config publisher ddos-publisher \
  destinations add { ddos-splunk }

# 5. Security Log Profile
tmsh create security log profile ddos-log-profile \
  dos-network-publisher ddos-publisher \
  ip-intelligence { log-publisher ddos-publisher } \
  network add { local-dos { publisher ddos-publisher } }

# 6. Auf Global Network DoS anwenden
tmsh modify security dos device-config dos-device-config \
  log-publisher ddos-publisher

# 7. Konfiguration speichern
tmsh save sys config
```

---

## Bekannte Probleme bei ELK 8.x (Troubleshooting)

| Problem | Ursache | Fix |
|---|---|---|
| Kibana `EACCES` auf CA-Zertifikat | Falsche Berechtigungen | Zertifikat nach `/etc/kibana/` kopieren mit `chown root:kibana` |
| Logstash `GeoIP requires target` | ECS v8 erzwingt explizites `target` | `geoip { source => "source_ip" target => "geoip" }` |
| Logstash `host field type mismatch` | ECS v8 sendet `host` als Objekt | `pipeline.ecs_compatibility: disabled` in `logstash.yml` |
| Logstash `node1-pipeline does not exist` | Ingest Pipeline nicht geladen | Pipeline VOR erstem Logstash-Start über ES API laden (Schritt 6) |
| Logstash `Mutex relocking` | Template-Management-Konflikt | `manage_template => false` im Logstash Output |
| API Key Authentication schlägt fehl | Key-Format inkompatibel | Auf user/password umstellen im Output-Block |
| Dashboard zeigt keine Daten | Falsches Field-Mapping | Index-Template vor Logstash laden, dann Index löschen und neu befüllen |

---

## Unterschiede ELK 7.x → ELK 8.x (Zusammenfassung)

| | ELK 7.x | **ELK 8.x** |
|---|---|---|
| Security | Manuell aktivieren | **Standard — immer aktiv** |
| TLS | Optional | **Pflicht** |
| Java | Separat installieren | **Im Paket enthalten** |
| GPG Key | `apt-key add` (veraltet) | `gpg --dearmor` |
| Logstash Auth | user/password | user/password (API Key hat Probleme) |
| Logstash SSL Option | `ssl => true` | **`ssl_enabled => true`** |
| GeoIP Filter | Kein target nötig | **`target => "geoip"` Pflicht** |
| ECS Kompatibilität | v1 | v8 (muss für alte Templates deaktiviert werden) |
| Ingest Pipeline | Optional | **Pflicht (node1-pipeline muss vor Logstash laden)** |

---

## DoS Statistics Collection

### Option A — Crontab (immer verfügbar)

Auf dem BIG-IP Crontab eintragen — `<ELK-SERVER-IP>` ersetzen:

```bash
* * * * * nb_of_tmms=$(tmsh show sys tmm-info | grep Sys::TMM | wc -l); \
tmctl -c dos_stat -s context_name,vector_name,attack_detected,stats_rate,drops_rate, \
int_drops_rate,ba_stats_rate,ba_drops_rate,bd_stats_rate,bd_drops_rate,detection, \
mitigation_low,mitigation_high,detection_ba,mitigation_ba_low,mitigation_ba_high, \
detection_bd,mitigation_bd_low,mitigation_bd_high | grep -v "context_name" | \
sed '/^$/d' | sed "s/$/,$nb_of_tmms/g" | logger -n <ELK-SERVER-IP> --udp --port 5558
```

### Option B — EAV External Monitor (empfohlen, erfordert Lizenz)

> **Voraussetzung:** Erfordert eine BIG-IP Lizenz die EAV (External Active Verification) unterstützt.  
> In UDF Lab-Instanzen ist EAV typischerweise **nicht** lizenziert → Crontab (Option A) verwenden.

Referenz: https://my.f5.com/manage/s/article/K71282813

```bash
# 1. Monitor Script erstellen
cat > /config/monitors/dos_stats_collector << 'SCRIPT'
#!/bin/bash
PIDFILE="/var/run/dos_stats_collector.${NODE_IP}..${NODE_PORT}.pid"
if [ -f "$PIDFILE" ]; then
    OLD_PID=$(cat "$PIDFILE" 2>/dev/null)
    [ -n "$OLD_PID" ] && kill -9 "$OLD_PID" 2>/dev/null
    rm -f "$PIDFILE"
fi
echo $$ > "$PIDFILE"

NB_TMMS=$(tmsh show sys tmm-info | grep 'Sys::TMM' | wc -l)
tmctl -c dos_stat -s context_name,vector_name,attack_detected,stats_rate,drops_rate,\
int_drops_rate,ba_stats_rate,ba_drops_rate,bd_stats_rate,bd_drops_rate,detection,\
mitigation_low,mitigation_high,detection_ba,mitigation_ba_low,mitigation_ba_high,\
detection_bd,mitigation_bd_low,mitigation_bd_high \
  | grep -v 'context_name' | sed '/^$/d' | sed "s/$/$,${NB_TMMS}/g" \
  | logger -n "${NODE_IP}" --udp --port "${NODE_PORT}"

rm -f "$PIDFILE"
echo 'up'
exit 0
SCRIPT
chmod +x /config/monitors/dos_stats_collector

# 2. Script in TMOS registrieren
tmsh create sys file external-monitor dos_stats_collector \
  source-path file:/config/monitors/dos_stats_collector

# 3. External Monitor erstellen (alle 30 Sekunden)
tmsh create ltm monitor external dos-stats-monitor \
  interval 30 timeout 25 run dos_stats_collector

# 4. Node für ELK Server erstellen
tmsh create ltm node elk-server address <ELK-SERVER-IP>

# 5. Pool mit Monitor erstellen
tmsh create ltm pool dos-stats-pool \
  monitor dos-stats-monitor \
  members add { <ELK-SERVER-IP>:5558 }

# 6. Konfiguration speichern
tmsh save sys config

# 7. Crontab-Eintrag entfernen
crontab -e   # tmctl-Zeile löschen
```


---

## Schritt 15 (Optional) — Grafana als alternatives Dashboard

> Grafana bietet bessere Zeitreihen-Visualisierung und mächtiges Alerting als Kibana.  
> Empfohlen als **Monitoring-Dashboard**, Kibana bleibt für Log-Analyse.

### Installation

```bash
# Repository hinzufügen
wget -q -O - https://apt.grafana.com/gpg.key | \
  sudo gpg --dearmor -o /usr/share/keyrings/grafana.gpg
echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana
sudo systemctl enable --now grafana-server
```

### Passwort setzen (beim ersten Login als admin/admin)

```bash
curl -s -X PUT http://admin:admin@<server-ip>:3000/api/user/password \
  -H 'Content-Type: application/json' \
  -d '{"oldPassword":"admin","newPassword":"<neues-passwort>","confirmNew":"<neues-passwort>"}'
```

### Elasticsearch Datenquellen hinzufügen

```bash
for INDEX in 'F5-DDoS-Events:f5-ddos-i-*' 'F5-DDoS-Stats:f5-stats-ddos-t-*'; do
  NAME=$(echo $INDEX | cut -d: -f1)
  IDX=$(echo $INDEX | cut -d: -f2)
  curl -s -X POST http://admin:<passwort>@<server-ip>:3000/api/datasources \
    -H 'Content-Type: application/json' \
    -d "{
      \"name\": \"$NAME\",
      \"type\": \"elasticsearch\",
      \"url\": \"https://localhost:9200\",
      \"access\": \"proxy\",
      \"basicAuth\": true,
      \"basicAuthUser\": \"elastic\",
      \"secureJsonData\": {\"basicAuthPassword\": \"<elastic-passwort>\"},
      \"jsonData\": {
        \"index\": \"$IDX\",
        \"timeField\": \"@timestamp\",
        \"esVersion\": \"8.0.0\",
        \"tlsSkipVerify\": true
      }
    }"
done
```

### Dashboards erstellen

Die fertigen Dashboard-Definitionen als JSON via Grafana API importieren — siehe `grafana_dashboards/` Verzeichnis (falls vorhanden) oder manuell in der Grafana UI erstellen.

**Empfohlene Panels für DDoS Overview:**
- Stat: Total Events / Drop Events / Allow Events / Unique Vectors / Unique Source IPs
- Time Series: Drop vs Allow Events Timeline
- Bar Chart: Top 10 Attack Vectors
- Table: Top Source IPs
- Pie Charts: Action-Verteilung, Context Type

**Empfohlene Panels für Vector Statistics:**
- Stat: Avg Drop Rate / Avg Incoming Rate / Detection Threshold
- Time Series: Overall Drop Rate / BA+BD Drop Rates / Incoming vs Drop
- Time Series: Detection vs Mitigation Threshold
- Table: Top Vectors by Drop Rate

### Logrotate

```
/var/log/grafana/grafana.log {
    daily
    size 100M
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    rotate 7
}
```

### Grafana vs Kibana — Wann was nutzen?

| | Grafana | Kibana |
|---|---|---|
| DDoS Rate-Monitoring | ✅ Ideal | Gut |
| Alerting | ✅ Mächtig | Eingeschränkt |
| Log-Exploration | Eingeschränkt | ✅ Ideal |
| Dashboard-Sharing | ✅ Einfach | Komplex |
| **Empfehlung** | Live-Monitoring | Event-Analyse |
