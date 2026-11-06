# Conconnecting

**Conconnecting** is an open-source, high-performance Multi-WAN gateway daemon and CLI written in Rust. It transforms bare-metal mini-PCs and multi-port embedded hardware (e.g., Raspberry Pi CM4, x86 Quad-NIC appliances) into an intelligent edge router.

Conconnecting merges multiple physical networks (multiple Ethernet ports, LTE modems, Wi-Fi connections) into a single unified upstream pipe, providing deterministic failover, latency-aware flow balancing, rate limiting, and hardware-level IP masquerading to downstream clients.

---

## Table of Contents

* [Key Features](#-key-features)
* [System Architecture](#-system-architecture)
* [Physical Appliance Topology](#-physical-appliance-topology)
* [Prerequisites & Hardware](#-prerequisites--hardware)
* [Installation](#-installation)
* [Configuration](#-configuration)
* [CLI & TUI Usage](#-cli--tui-usage)
* [Technical Deep Dive](#-technical-deep-dive)
* [Roadmap](#-roadmap)
* [License](#-license)

---

## Key Features

1. **Hardware-Based IP Rewiring (Stateful SNAT)**
   * Acts as an inline physical gateway. Downstream computers connect through a dedicated LAN port with a single local IP, while outgoing flows are dynamically rewritten across all connected WAN interfaces.
2. **Deep Packet Management & Inspection**
   * Zero-copy Layer 2–Layer 4 header parsing and modification powered by `etherparse` and `pnet`.
   * Low-overhead fast-path packet steering via kernel-space eBPF/XDP hooks with user-space TUN fallback.
3. **Dynamic Interface Controller & Hotplugging**
   * Asynchronous Linux Netlink (`rtnetlink`) subsystem for real-time link detection.
   * Automatically rebalances traffic tables when Ethernet cables are plugged, unplugged, or when Wi-Fi associations drop.
4. **Rate Limiting & Metric-Based Preference Meter**
   * Hierarchical Token Bucket (HTB) rate limiting to throttle bandwidth per interface or host.
   * Interface weighting engine prioritizing high-speed, unmetered uplinks (e.g., Fiber) over metered or fallback paths (e.g., 5G/LTE).
5. **Dynamic Latency-Balancing Engine**
   * Sub-second non-blocking ICMP ping and passive TCP RTT tracking across all WAN interfaces.
   * Dynamic weight adjustment: reroutes latency-sensitive traffic (VoIP, SSH, gaming) to the lowest-jitter path in real time.
6. **Hardened Security & DNS Leak Protection**
   * Integrated stateful connection tracking (`conntrack`) dropping unsolicited inbound WAN packets.
   * Built-in DNS-over-HTTPS (DoH) / DNS-over-TLS (DoT) recursive caching resolver to stop upstream ISP eavesdropping.

---

## System Architecture

```
                       [ DOWNSTREAM CLIENT ]
                      (Desktop / Laptop / Switch)
                       IP: 192.168.10.2 (via DHCP)
                                  │
                                  ▼
                   [ CONCONNECTING APPLIANCE (LAN) ]
                     Interface: eth0 (192.168.10.1)
                                  │
      ┌───────────────────────────┴───────────────────────────┐
      │             Conconnecting Core Engine (Rust)          │
      │                                                       │
      │  ┌─────────────────┐ ┌───────────────┐ ┌───────────┐  │
      │  │ Flow Classifier │ │ Latency Meter │ │ Rate Lim. │  │
      │  │  (5-Tuple Hash) │ │ (RTT Prober)  │ │ (Bucket)  │  │
      │  └────────┬────────┘ └───────┬───────┘ └─────┬─────┘  │
      │           │                  │               │        │
      │           ▼                  ▼               ▼        │
      │       [ Multi-WAN Dispatcher & Dynamic NAT (SNAT) ]     │
      └───────┬──────────────────────┬────────────────┬───────┘
              │                      │                │
              ▼                      ▼                ▼
         [ WAN 1 (eth1) ]      [ WAN 2 (eth2) ]  [ WAN 3 (wlan0) ]
         Fiber ISP (1 Gbps)     Cable ISP (300M)  5G/LTE Backup
```

---

## Physical Appliance Topology

Conconnecting is designed to run on a physical appliance placed between your upstream internet connections and your local workstation or switch.

* **Dedicated LAN Port (`eth0`):** Serves DHCP (`192.168.10.0/24`) and DNS to your workstation. Your desktop only interacts with this single interface.
* **Aggregated WAN Ports (`eth1`, `eth2`, `wlan0`):** Connect directly to upstream modems, fiber ONTs, or mobile hotspots. Conconnecting manages routes and multiplexing transparently.

---

## Prerequisites & Hardware

### Recommended Hardware
* **x86-64 Appliance:** Intel N100 / Celeron J4125 or newer with 3–4x Intel 2.5GbE i225/i226 NICs.
* **ARM Alternative:** Raspberry Pi Compute Module 4 with dual-NIC carrier board, FriendlyElec NanoPi R4S/R5S.

### OS & Kernel Requirements
* Linux Kernel $\ge$ 5.15 (Kernel 6.x+ recommended for full eBPF/XDP multi-buffer support).
* Capabilities: `CAP_NET_ADMIN`, `CAP_NET_RAW`, and `CAP_BPF`.
* Dependencies: `iproute2`, `nftables`, `ethtool`.

---

## Installation

### From Pre-built Binaries

Download the latest binary release for your architecture from the [Releases](https://github.com/your-username/conconnecting/releases) page.

```bash
curl -LO https://github.com/your-username/conconnecting/releases/latest/download/conconnecting-x86_64-unknown-linux-gnu.tar.gz
tar -xvf conconnecting-x86_64-unknown-linux-gnu.tar.gz
sudo mv conconnecting /usr/local/bin/
```

### Building from Source

```bash
# Install Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# Install build dependencies (Debian/Ubuntu)
sudo apt-get update && sudo apt-get install -y build-essential libelf-dev pkg-config clang llvm

# Clone repository and compile
git clone https://github.com/your-username/conconnecting.git
cd conconnecting
cargo build --release

# Binary is located at target/release/conconnecting
sudo cp target/release/conconnecting /usr/local/bin/
```

---

## Configuration

Conconnecting is configured via a single TOML file located at `/etc/conconnecting/conconnecting.toml`.

```toml
[lan]
interface = "eth0"
ip_address = "192.168.10.1"
subnet_mask = "255.255.255.0"
dhcp_enabled = true
dhcp_range = ["192.168.10.100", "192.168.10.250"]

# Upstream WAN Interfaces
[[wan]]
name = "Primary-Fiber"
interface = "eth1"
type = "dhcp"
weight = 10                  # Static priority preference (1-10)
rate_limit_mbps = 1000
health_check_target = "1.1.1.1"

[[wan]]
name = "Secondary-Cable"
interface = "eth2"
type = "dhcp"
weight = 7
rate_limit_mbps = 300
health_check_target = "8.8.8.8"

[[wan]]
name = "LTE-Fallback"
interface = "wlan0"
type = "dhcp"
weight = 2
rate_limit_mbps = 50
health_check_target = "9.9.9.9"

[balancer]
strategy = "latency_weighted"   # Options: "round_robin", "latency_weighted", "failover_only"
probe_interval_ms = 1000
jitter_penalty_threshold_ms = 25

[security]
drop_unsolicited = true
syn_flood_protection = true
dns_over_https = true
upstream_doh_provider = "https://cloudflare-dns.com/dns-query"
```

---

## CLI & TUI Usage

Conconnecting features an interactive Terminal User Interface (TUI) as well as headless systemd daemon controls.

### Running as a Daemon

```bash
# Start Conconnecting service in background
sudo conconnecting daemon --config /etc/conconnecting/conconnecting.toml

# Or using systemd
sudo systemctl enable --now conconnecting
```

### Launching the Real-Time Terminal Dashboard

```bash
sudo conconnecting tui
```

*Displays real-time bandwidth graphs per interface, active RTT latency meters, connection pools, and hardware link states using `ratatui`.*

### Command Line Reference

```bash
# Check link status and dynamic latency scores
conconnecting status

# Manually trigger health check probes across all WANs
conconnecting probe

# Set a manual weight override on an interface
conconnecting set-weight --interface eth2 --weight 9

# Add or inspect active firewall / conntrack rules
conconnecting firewall status
```

---

## 🔬 Technical Deep Dive

### 1. Per-Flow Multipath Load Balancing

Internet connections break when individual packets of a single TCP stream arrive from different public IP addresses. To prevent TCP connection drops, Conconnecting uses **5-tuple flow hashing**:

$$
\text{Flow Hash} = \text{Hash}(\text{SrcIP}, \text{DstIP}, \text{SrcPort}, \text{DstPort}, \text{Protocol})
$$

All packets belonging to the same flow are persistently assigned to the selected WAN link for the duration of that session, maintaining consistent state without requiring an external relay server.

### 2. Latency Dynamic Weighting Equation

WAN weights are updated continuously according to actively measured Round Trip Times ($RTT$) and packet loss ($\alpha$):

$$
W_{\text{effective}} = W_{\text{base}} \times \left( \frac{1}{1 + \frac{RTT}{RTT_{\text{baseline}}}} \right) \times (1 - \alpha)^2
$$

If a primary link experiences bufferbloat or transient congestion, traffic shifts dynamically to secondary links before connections time out.

---

## Roadmap

- [x] Initial Netlink interface listener and link state daemon.
- [x] Userspace L3/L4 packet routing with `etherparse` and TUN virtual interfaces.
- [x] Real-time latency probe engine with dynamic weight recalculation.
- [ ] eBPF/XDP fast-path bypass for multi-gigabit throughput line-rates.
- [ ] Optional client/server True Packet Striping (MPTCP / aggregated proxy tunneling).
- [ ] Web-based telemetry dashboard (embedded Actix Web + WASM).

---

## License

Licensed under either of:

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.
