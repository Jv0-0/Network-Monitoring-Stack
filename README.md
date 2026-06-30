# 🛡️ Network Monitoring Stack with Multi-Protocol Routing Topology

> Full-stack network monitoring infrastructure integrating Zabbix, Grafana, n8n, and AI — connected to a multi-protocol Cisco topology via Tailscale VPN.

![Stack](https://img.shields.io/badge/Stack-Zabbix%20%7C%20Grafana%20%7C%20n8n%20%7C%20Docker-blue)
![Protocols](https://img.shields.io/badge/Protocols-RIP%20%7C%20EIGRP%20%7C%20OSPF-orange)
![Platform](https://img.shields.io/badge/Platform-WSL2%20%2B%20PNetLab-green)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen)

---

## 📌 Overview

This project implements a production-grade **network monitoring solution** built entirely on free and open-source tools. It combines a Dockerized observability stack running on WSL2 with a simulated multi-protocol Cisco network in PNetLab, connected securely through a **Tailscale VPN mesh**.

The goal was to replicate real-world NOC/SOC monitoring workflows: automated host discovery, SNMP polling, threshold-based alerting, and AI-assisted network analysis — all from a personal lab environment.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     WSL2 (Ubuntu)                       │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐  ┌───────────┐   │
│  │  Zabbix  │  │ Grafana  │  │  n8n  │  │Open WebUI │   │
│  │ + PgSQL  │  │          │  │       │  │(Ollama AI)│   │
│  └────┬─────┘  └────┬─────┘  └───┬───┘  └─────┬─────┘   │
│       └─────────────┴────────────┴────────────┘         │
│                  Docker Compose                         │
│                       │                                 │
│               ┌───────┴────────┐                        │
│               │  Tailscale VPN │                        │
│               └───────┬────────┘                        │
└───────────────────────┼─────────────────────────────────┘
                        │ VPN Tunnel
┌───────────────────────┼────────────────────────────────────┐
│              PNetLab Topology                              │
│                                                            │
│   [Paipe]──EIGRP──[Manga]──OSPF──[Boyaca]                  │
│      │                 |              │                    │
│    EIGRP               |             OSPF                  │
│      └──[Medellin]───[Manizales]      └───[FrrLone]        │
│                                                            │
│ 18 Cisco, Frrouting routers/switches monitored/linux sv/pc │
└────────────────────────────────────────────────────────────┘
```

---

## 🧰 Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Emulation** | PNetLab | Cisco, Frrouting multi-protocol topology |
| **Containerization** | Docker Compose + WSL2 | Stack orchestration |
| **Monitoring** | Zabbix 6.x + PostgreSQL | SNMP polling, host management, alerting |
| **Visualization** | Grafana | Custom dashboards, time-series graphs |
| **Automation** | n8n | Workflow automation, alert routing |
| **AI Interface** | Open WebUI + Ollama | Natural language network analysis |
| **Connectivity** | Tailscale VPN | Secure tunnel between WSL2 and PNetLab |
| **Reverse Proxy** | nginx | Internal routing between services |
| **Alert Delivery** | Telegram Bot + ngrok | Real-time incident notifications |

---

## 🌐 Network Topology

The PNetLab topology simulates a multi-site enterprise WAN using **three routing protocols simultaneously**:

| Domain | Protocol | Routers |
|---|---|---|
| Core backbone | EIGRP | Manga, Bogota, etc ISP nodes |
| Branch offices | OSPF | Boyaca, Frrlone, and downstream switches |

**18 hosts monitored** via SNMP v2c, including routers, Linux Servers / PC, and edge devices.

---

## 🔒 Access Control Lists (ACLs)

Four routers in the topology implement ACLs for traffic filtering and management-plane protection. Each one was designed for a different purpose, and not all of them are actively enforced — this section documents exactly what's configured and what's actually applied, which is just as important as the configuration itself.

### Rayo

Rayo connects to a downstream **Layer 2 switch** that handles VLAN segmentation (see [VLANs](#-vlans) below), and runs three separate ACLs:

```cisco
ip access-list standard TELNET
 permit 192.168.66.50

ip access-list extended SERV_IN
 deny   icmp host 192.168.10.17 any echo
 permit ip any any

ip access-list standard 10
 permit 192.168.10.0 0.0.0.255
 permit 192.168.20.0 0.0.0.255
 permit 192.168.30.0 0.0.0.255
 permit 192.168.40.0 0.0.0.255
 permit 192.168.50.0 0.0.0.255
```

| ACL | Type | Status | Purpose |
|---|---|---|---|
| `TELNET` | Standard | Defined, not applied to any `access-class` | Intended to restrict Telnet/VTY access to a single management host (`192.168.66.50`) |
| `SERV_IN` | Extended | **Applied** (`ip access-group SERV_IN in`) | Blocks ICMP echo requests from a specific monitoring host while permitting all other traffic |
| `10` | Standard | Defined, **not referenced** by any `distribute-list`, `route-map`, or `nat` statement | Covers the exact subnet ranges (`192.168.10–50.0/24`) handled by the VLANs on the downstream switch — likely staged for future route filtering or NAT, but currently inactive |

### Calima

Calima enforces the most granular policy in the topology, combining management restriction with inter-site traffic control between its local LAN (`192.168.66.0/24`) and a remote network (`172.16.200.0/24`):

```cisco
ip access-list standard TELNET
 permit 192.168.66.50

ip access-list extended LAN_IN
 permit tcp host 192.168.66.50 host 172.16.200.1 eq telnet
 deny   tcp host 192.168.66.50 host 192.168.10.17 eq www
 deny   tcp host 192.168.66.50 host 192.168.10.17 eq 443
 permit tcp 172.16.200.0 0.0.0.255 192.168.66.0 0.0.0.255 established
 permit icmp 192.168.66.0 0.0.0.255 172.16.200.0 0.0.0.255 echo-reply
 deny   icmp host 192.168.66.51 host 192.168.10.17
 permit tcp 192.168.66.0 0.0.0.255 172.16.200.0 0.0.0.255 established
 deny   ip 192.168.66.0 0.0.0.255 172.16.200.0 0.0.0.255
 permit ip any any
```

| ACL | Type | Status | Purpose |
|---|---|---|---|
| `TELNET` | Standard | Defined, not applied | Same management-host pattern as Rayo |
| `LAN_IN` | Extended | **Applied** (`ip access-group LAN_IN in`) | Allows the management host Telnet access to a specific remote host; blocks HTTP/HTTPS from that host toward a monitored server; permits only *established* TCP sessions and ICMP echo-replies between the two sites; explicitly denies new inbound IP traffic from the remote subnet before the final permit-all |

### Paipe

Paipe only defines the same baseline Telnet restriction, with no extended filtering applied:

```cisco
ip access-list standard TELNET
 permit 192.168.66.50
```

| ACL | Type | Status | Purpose |
|---|---|---|---|
| `TELNET` | Standard | Defined, not applied | Consistent management-access pattern across the topology, not yet enforced here |

### Manga (Internet Edge Node)

Manga is the designated internet-facing router in the topology, but its only ACL is a broad standard list covering internal, CGNAT, and Docker bridge ranges:

```cisco
ip access-list standard 1
 permit 10.0.0.0 0.0.255.255
 permit 192.168.0.0 0.0.255.255
 permit 172.16.0.0 0.0.255.255
 permit 100.0.0.0 0.255.255.255
 permit 172.16.0.0 0.15.255.255
 permit 172.19.0.0 0.0.255.255
```

| ACL | Type | Status | Purpose |
|---|---|---|---|
| `1` | Standard | Defined, **not referenced** by `ip nat`, `distribute-list`, or `route-map` | Covers the lab's internal ranges plus Tailscale CGNAT (`100.64.0.0/10`) and the Docker bridge subnet (`172.19.0.0/16`) — staged for a NAT overload rule (`ip nat inside source list 1 interface X overload`), but no `ip nat` configuration currently exists on this router. **Internet egress in this lab is handled outside of Cisco IOS**, via the WSL2/Docker host and Tailscale, not through router-level PAT. |

> 📌 **Why document inactive ACLs at all?** Two ACLs in this topology (Rayo's `10` and Manga's `1`) are configured but not applied to any process. Rather than omitting them, they're documented here to show the distinction between *defining* an ACL and *enforcing* it — a common real-world gap that shows up in security audits.

---

## 🧩 VLANs

VLAN segmentation is handled by a dedicated **Cisco Layer 2 switch** downstream of Rayo, which explains why Rayo's ACL `10` (above) mirrors these exact subnet ranges.

### VLAN Database

| VLAN ID | Name | Status | Access Ports |
|---|---|---|---|
| 1 | default | active | Et0/3, Et1/0, Et1/3, Et2/0, Et2/1, Et3/0–3, Et4/0–3, Et5/0–3, Et6/0–3 |
| 10 | VLAN0010 | active | Et0/0, Et0/1, Et0/2 |
| 20 | VLAN0020 | active | Et2/2, Et2/3 |
| 30 | VLAN0030 | active | Et1/1, Et1/2 |

### Access Port Configuration

Confirmed per-interface `switchport` config on the L2 switch:

```cisco
interface Ethernet0/0
 switchport access vlan 10
 switchport mode access
interface Ethernet0/1
 switchport access vlan 10
 switchport mode access
interface Ethernet0/2
 switchport access vlan 10
 switchport mode access
interface Ethernet1/1
 switchport access vlan 30
 switchport mode access
interface Ethernet1/2
 switchport access vlan 30
 switchport mode access
interface Ethernet2/2
 switchport access vlan 20
 switchport mode access
```

### Trunk Links

The switch uplinks to the rest of the topology over four 802.1Q trunk ports:

| Trunk Port | Encapsulation | Status | Allowed VLAN | Active in Management Domain |
|---|---|---|---|---|
| Et7/0 | 802.1Q | trunking | 30 | ✅ Yes |
| Et7/1 | 802.1Q | trunking | 10 | ✅ Yes |
| Et7/2 | 802.1Q | trunking | 40 | ⚠️ No — VLAN 40 not yet created in VLAN database |
| Et7/3 | 802.1Q | trunking | 20 | ✅ Yes |

> 📌 **Note on VLAN 40:** It's allowed on the `Et7/2` trunk but doesn't exist yet in `show vlan brief` — meaning the trunk is pre-provisioned for a VLAN that hasn't been created on the switch. Documented as-is rather than silently fixed, consistent with the approach taken for the inactive ACLs above.

---

## 📊 Monitoring Features

- ✅ **SNMP v2c polling** of all 10 Cisco, Frrouting devices (interfaces, CPU, memory, uptime)
- ✅ **Custom Grafana dashboards** — interface traffic, latency trends, host availability
- ✅ **Threshold-based alerting** with Zabbix triggers
- ✅ **Telegram bot notifications** via n8n workflows and ngrok tunneling
- ✅ **AI-assisted analysis** through Open WebUI for natural language network queries
- ✅ **NAT access-list tuning** to expose Docker/Tailscale traffic correctly on edge routers

---

## 🚀 Getting Started

### Prerequisites

- WSL2 with Ubuntu 22.04+
- Docker + Docker Compose v2
- Tailscale account (free tier)
- PNetLab CE (local vm or cloud)

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/network-monitoring-stack.git
cd network-monitoring-stack
```

### 2. Configure environment variables

```bash
cp .env.example .env
# Edit .env with your Zabbix DB credentials, Tailscale auth key, Telegram bot token
```

### 3. Start the stack

```bash
docker compose up -d
```

### 4. Access the services

| Service | URL | Default Credentials |
|---|---|---|
| Zabbix Web | `http://localhost:8080` | Admin / zabbix |
| Grafana | `http://localhost:3000` | admin / admin |
| n8n | `http://localhost:5678` | — |
| Open WebUI | `http://localhost:8501` | — |

### 5. Import PNetLab topology

Import the `.unl` topology file from `/pnetlab/topology/` into your PNetLab instance and configure SNMP community string `public` on all Cisco devices.

---

## ⚠️ Key Technical Challenges Solved

**1. Docker/Tailscale traffic not visible in Zabbix**
Zabbix was not seeing traffic from the Docker subnet or Tailscale IPs because the Bogota and Manga routers had NAT access-lists that excluded those ranges in the simulated network topology. Fixed by adding explicit `permit` entries for `100.64.0.0/10` (Tailscale CGNAT) and the Docker bridge subnet. (Later I delete the RIP routing protocol from Bogota an other routers, and change it for EIGRP, and left only the Manga router with the NAT access)

**2. SNMP reachability across routing domains**
Inter-domain SNMP polling required verifying route redistribution at protocol boundaries. Resolved by confirming EIGRP ↔ OSPF redistribution metrics and adding static host routes where needed.

**3. n8n Telegram alerts through ngrok**
Telegram webhooks require a public HTTPS endpoint. Solved by running ngrok as a sidecar service and dynamically updating the webhook URL in n8n on each restart.

---

## 📁 Repository Structure

```
network-monitoring-stack/
├── README.md
├── .gitignore
├── Docker-Stack/
│   ├── docker composes/
│   │   ├── docker-compose.yml      # Full stack definition
│   │   ├── docker-compose-01.yml   # Alternate/iterative version
│   │   ├── env.example             # Environment variable template
│   │   └── nginx.conf              # Reverse proxy config
│   └── function Proves/
│       ├── Docker-Comp.png         # Screenshot: stack running
│       └── Stack-Demostration.mp4  # Video demo of the stack in action
├── Zabbix/
│   ├── zbx_export_hosts.xml    # Exported host configuration
│   ├── All-On-Topol-Zabbix.png # Screenshot: full topology monitored (all hosts up)
│   └── Half-On-Topol-Zabbix.png# Screenshot: partial topology monitored
├── Graffana/
│   └── dashboards/
│       └── network_overview.json  # Pre-built dashboard
├── N8N/
│   ├── function Images/        # Screenshots of the workflow in action
│   └── workflows/
│       └── Automatization-n8n.json # Alert routing / AI workflow (JSON export)
├── Tailscale/
│   ├── Conf1-Talescale.png     # Tailscale setup walkthrough
│   ├── PNETLAB-Subnets.png     # Advertised subnet routes (PNetLab side)
│   └── ZNG-Subnets.png         # Advertised subnet routes (ZNG/WSL2 side)
├── pnetlab/
│   ├── acls/                   # Per-router ACL configs (Rayo, Calima, Paipe, Manga)
│   ├── Topology/                # PNetLab .unl topology file
│   ├── vlans/                   # L2 switch VLAN & trunk configs
│   ├── Manga-PNETLAB-ipconf.png # Screenshot: Manga IP configuration
│   └── PNETLA-ip-route.png      # Screenshot: routing table
└── docs/
    ├── Architecture_All_On.png  # Architecture diagram (full topology online)
    └── Architecture_Half_On.png # Architecture diagram (partial topology online)
    └── setup-guide-es.md # Setup guide in Spanish
```

---

## 🎯 Skills Demonstrated

`Docker` `Docker Compose` `Zabbix` `Grafana` `n8n` `SNMP v2c` `Tailscale VPN`  
`Cisco IOS` `EIGRP` `OSPF` `NAT / ACL` `VLAN Segmentation` `802.1Q Trunking` `TCP/IP` `Linux (WSL2)`  
`PostgreSQL` `nginx` `Network Monitoring` `Incident Alerting` `AI Integration`

---

## 📚 Academic Context

This project was developed as the final deliverable for the **Communications Networks 2** course at **Fundación Universitaria Konrad Lorenz** (Bogotá, Colombia), 6th semester — Systems Engineering.

It goes beyond the academic requirements by incorporating a production-grade monitoring stack, automated alerting pipelines, and AI-assisted analysis, replicating workflows used in real NOC/SOC environments.

---

## 👤 Author

**Joseph Vargas**  
Systems Engineering Student — Fundación Universitaria Konrad Lorenz   
🔗 [LinkedIn](https://www.linkedin.com/in/jvo007/) | 🐙 [GitHub](https://github.com/Jv0-0)

---

## 📄 License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
