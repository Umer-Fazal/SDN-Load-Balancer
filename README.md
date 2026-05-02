<div align="center">

# 🌐 SDN Load Balancer

### A Software-Defined Networking based dynamic load balancer built with POX and Mininet

[![Python](https://img.shields.io/badge/Python-3.8%2B-blue?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![POX](https://img.shields.io/badge/POX-0.7.0%20gar-purple?style=flat-square)](https://noxrepo.github.io/pox-doc/html/)
[![Mininet](https://img.shields.io/badge/Mininet-2.3.1-green?style=flat-square)](http://mininet.org)
[![OpenFlow](https://img.shields.io/badge/OpenFlow-1.0-orange?style=flat-square)](https://opennetworking.org)
[![License](https://img.shields.io/badge/License-Apache%202.0-red?style=flat-square)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-yellow?style=flat-square&logo=linux&logoColor=white)](https://kernel.org)

<br/>

> *Replacing expensive hardware load balancers with intelligent, programmable SDN-based traffic distribution*

<br/>

[Features](#-features) • [Architecture](#-architecture) • [Requirements](#-requirements) • [Installation](#-installation) • [Usage](#-usage) • [Algorithms](#-algorithms) • [Demo](#-demo) • [Authors](#-authors)

</div>

---

## 📖 Overview

This project implements a **fully functional Layer-4 load balancer** using the SDN paradigm. A centralized **POX controller** intercepts TCP connections directed at a **Virtual IP (VIP)** and intelligently redistributes them across multiple backend HTTP servers — all within a **Mininet-emulated** network environment.

Traditional hardware load balancers cost tens of thousands of dollars and are inflexible. This project achieves the same result in **pure Python**, running on any Linux machine.

```
Client → VIP (10.0.0.100) → [POX Controller decides] → server1 / server2 / server3 / server4
```

---

## ✨ Features

| Feature | Description |
|---|---|
| ⚖️ **Round Robin** | Distributes requests evenly across servers in circular order |
| 🧠 **Least Connection** | Routes to the server with fewest active connections |
| 💓 **Health Monitoring** | ARP probes detect dead servers every 5 seconds and auto-remove them |
| 🔁 **Session Persistence** | All packets of a TCP connection always reach the same server |
| ⚡ **Flow Rule Installation** | OpenFlow rules installed in switch — controller not needed per-packet |
| 🌐 **DNS Resolution** | Built-in DNS server resolves domain names to the VIP address |
| 🖥️ **HTTP Simulation** | Backend servers run real HTTP services (Python http.server) |
| 🗺️ **Mininet Topology** | Fully emulated network: 1 switch, 4 servers, 9 clients |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENTS (pc1 – pc9)                      │
│                  IPs: 10.0.0.11 – 10.0.0.19                │
└──────────────────────┬──────────────────────────────────────┘
                       │  TCP requests to VIP (10.0.0.100)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│               OpenFlow Switch  (s1 — OVS)                   │
│     First packet → PacketIn    │    Subsequent → Direct     │
└──────────────┬──────────────────────────────────────────────┘
               │ PacketIn                      ▲ Flow Rules
               ▼                               │
┌─────────────────────────────────────────────────────────────┐
│                    POX CONTROLLER                           │
│   ┌─────────────────┐   ┌──────────────────────────────┐   │
│   │  _do_probe()    │   │      _pick_server()          │   │
│   │  ARP every 5s  │   │  Round Robin | Least Conn    │   │
│   └─────────────────┘   └──────────────────────────────┘   │
│   ┌─────────────────┐   ┌──────────────────────────────┐   │
│   │ _handle_Packet  │   │       DNS Server             │   │
│   │ IP rewriting    │   │  UDP :53 → resolves to VIP  │   │
│   └─────────────────┘   └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
               │ Rewritten packets
               ▼
┌──────────────────────────────────────────────────────────────┐
│                    BACKEND SERVERS                           │
│  server1 (10.0.0.101)  │  server2 (10.0.0.102)             │
│  server3 (10.0.0.103)  │  server4 (10.0.0.104)             │
│              All running: python3 -m http.server 80          │
└──────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Client** sends a TCP connection request to VIP `10.0.0.100`
2. **Switch** receives the packet — it's a new flow, so it sends it to the controller (`PacketIn`)
3. **POX Controller** picks a backend server using the selected algorithm
4. **Controller** rewrites destination IP/MAC to the chosen server and installs **bidirectional OpenFlow rules** in the switch
5. **Switch** forwards all future packets of this connection directly (no controller needed)
6. **Server** responds — controller rewrites source IP back to VIP before delivery to client
7. **Client** receives the response as if it came from `10.0.0.100` — completely transparent

---

## 📋 Requirements

### System
- **OS:** Linux (Ubuntu 20.04+, Debian, or Kali Linux recommended)
- **RAM:** 4 GB minimum (8 GB recommended)
- **Python:** 3.8+

### Software

```bash
# Core dependencies
sudo apt-get install -y mininet openvswitch-switch python3 git curl
```

| Package | Version | Purpose |
|---|---|---|
| Mininet | 2.3.1+ | Network emulation framework |
| Open vSwitch | 2.13+ | Virtual OpenFlow switch |
| POX | 0.7.0 (gar) | SDN controller |
| Python 3 | 3.8+ | Controller scripting language |

---

## 🚀 Installation

### Step 1 — Install Mininet

```bash
sudo apt-get update
sudo apt-get install -y mininet

# Verify installation
sudo mn --version
```

### Step 2 — Clone POX Controller

```bash
cd ~
git clone https://github.com/noxrepo/pox.git
```

### Step 3 — Clone This Repository

```bash
git clone https://github.com/yourusername/sdn-load-balancer.git
cd sdn-load-balancer
```

### Step 4 — Copy Files into POX

```bash
cp l2.py ~/pox/ext/
cp l2_learning.py ~/pox/ext/
```

### Step 5 — Verify Open vSwitch is Running

```bash
sudo service openvswitch-switch start
sudo ovs-vsctl show
```

---

## ▶️ Usage

You need **two terminal windows** open simultaneously.

### Terminal 1 — Start the POX Controller

```bash
cd ~/pox

# Run with Round Robin
sudo python3 pox.py l2_learning l2 \
  --ip=10.0.0.100 \
  --servers=10.0.0.101,10.0.0.102,10.0.0.103,10.0.0.104 \
  --algo=round-robin

# OR run with Least Connection
sudo python3 pox.py l2_learning l2 \
  --ip=10.0.0.100 \
  --servers=10.0.0.101,10.0.0.102,10.0.0.103,10.0.0.104 \
  --algo=least-connection
```

**Expected output:**
```
POX 0.7.0 (gar) / Copyright 2011-2020 James McCauley, et al.
INFO:iplb:Selected load balancing algorithm: round-robin
INFO:core:POX 0.7.0 (gar) is up.
INFO:openflow.of_01:[00-00-00-00-00-01 2] connected
INFO:iplb:IP Load Balancer Ready.
DNS Server listening on 127.0.0.1:53
INFO:iplb:Server 10.0.0.101 up
INFO:iplb:Server 10.0.0.102 up
INFO:iplb:Server 10.0.0.103 up
INFO:iplb:Server 10.0.0.104 up
```

### Terminal 2 — Start the Mininet Topology

```bash
sudo python3 topology.py
```

**Expected output:**
```
Network setup complete. Host IPs:
server1 has MAC address xx:xx:xx:xx:xx:xx ...
Web servers started.
mininet>
```

---

## 🎮 Demo

Once both terminals are running, use the `mininet>` prompt to test the system:

### Test Network Connectivity
```
mininet> pingall
** Results: 0% dropped (156/156 received)
```

### Test Load Balancing (watch Terminal 1 for server selection logs)
```
mininet> pc1 curl http://10.0.0.100
mininet> pc2 curl http://10.0.0.100
mininet> pc3 curl http://10.0.0.100
mininet> pc4 curl http://10.0.0.100
```

### Test Server Health Monitoring
```
# Kill a server
mininet> server1 kill %python3

# After ~5 seconds, Terminal 1 will show:
# WARN:iplb: Server 10.0.0.101 down
# Traffic is now automatically redirected to remaining servers
```

### Send Multiple Concurrent Requests
```
mininet> pc1 curl http://10.0.0.100 & pc2 curl http://10.0.0.100 & pc3 curl http://10.0.0.100 &
```

---

## ⚙️ Algorithms

### Round Robin
```
Request 1 → server1  (index: 0 → 1)
Request 2 → server2  (index: 1 → 2)
Request 3 → server3  (index: 2 → 3)
Request 4 → server4  (index: 3 → 0)
Request 5 → server1  (wraps back!)
```
> Best for: Equal, fast, homogeneous requests

### Least Connection
```
State:    {server1: 3, server2: 1, server3: 5, server4: 2}
Decision: → server2  (fewest active connections)
Updated:  {server1: 3, server2: 2, server3: 5, server4: 2}
```
> Best for: Mixed, real-world traffic where some requests take longer

| | Round Robin | Least Connection |
|---|:---:|:---:|
| Tracks active connections | ❌ | ✅ |
| Fair for equal workloads | ✅ | ✅ |
| Fair for unequal workloads | ❌ | ✅ |
| Implementation complexity | Low | Medium |

---

## 📁 Project Structure

```
sdn-load-balancer/
│
├── l2.py               # Main load balancer — POX controller application
│                       # Contains: iplb class, DNSServer, MemoryEntry
│
├── l2_learning.py      # L2 learning switch — MAC address table management
│
├── topology.py         # Mininet network topology
│                       # 1 switch + 4 servers + 9 clients
│
└── README.md           # This file
```

### Key Components in `l2.py`

| Component | Role |
|---|---|
| `iplb` | Main load balancer class — core of the entire system |
| `_handle_PacketIn()` | Processes every packet: ARP handling, IP rewriting, flow installation |
| `_pick_server()` | Implements Round Robin and Least Connection algorithms |
| `_do_probe()` | Sends ARP probes every 5s to monitor server health |
| `_do_expire()` | Cleans up expired flows and updates connection counts |
| `MemoryEntry` | Stores active flow → server mappings for session persistence |
| `DNSServer` | Background UDP thread resolving domain names to VIP |

---

## 🔧 Configuration

| Parameter | Flag | Default | Description |
|---|---|---|---|
| Virtual IP | `--ip` | required | The VIP address clients connect to |
| Backend Servers | `--servers` | required | Comma-separated list of server IPs |
| Algorithm | `--algo` | required | `round-robin` or `least-connection` |
| Switch DPID | `--dpid` | first connected | Specific switch to load balance on |

**Flow Timeouts** (configurable in `l2.py`):

```python
FLOW_IDLE_TIMEOUT = 10      # seconds — flow removed after 10s of inactivity
FLOW_MEMORY_TIMEOUT = 60*5  # seconds — controller memory entry expires after 5 min
```

---

## ⚠️ Known Issues & Notes

- **Python 3.13 Warning:** POX officially supports Python 3.6–3.9. A deprecation warning appears on Python 3.13 (Kali 2026) but does **not** affect functionality.
- **Kali Linux:** Mininet's install script does not support Kali natively — install via `sudo apt-get install mininet` instead.
- **DNS Port 53:** Requires `sudo` as port 53 is a privileged port.
- **OVS Must Be Running:** Always start Open vSwitch before launching the topology.

---

## 📚 References

1. McKeown, N., et al. *"OpenFlow: Enabling Innovation in Campus Networks."* ACM SIGCOMM CCR, 2008.
2. Feamster, N., Rexford, J., & Zegura, E. *"The Road to SDN."* ACM SIGCOMM CCR.
3. [POX Controller Documentation](https://noxrepo.github.io/pox-doc/html/)
4. [Mininet Documentation](http://mininet.org/documentation/)
5. [Open Networking Foundation](https://opennetworking.org)

---

## 👨‍💻 Authors

<table>
  <tr>
    <td align="center">
      <b>Muhammad Umer Fazal</b><br/>
    </td>
  </tr>
</table>

**Course:** CS 3001 — Computer Networks
**University:** National University of Computer & Emerging Sciences (FAST-NUCES)

---

<div align="center">

Made with ❤️ for CS 3001 — Computer Networks

</div>
