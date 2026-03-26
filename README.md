# 🌐 Linux Network Namespace Simulation

> A network simulation with two isolated networks connected via a virtual router using Linux network namespaces, bridges, and virtual ethernet pairs.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Network Topology](#network-topology)
- [IP Addressing Scheme](#ip-addressing-scheme)
- [Prerequisites](#prerequisites)
- [Step-by-Step Implementation](#step-by-step-implementation)
- [Automation Script](#automation-script)
- [Testing & Verification](#testing--verification)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)

---

## Overview

This project demonstrates how to create isolated network environments within a single Linux host using **network namespaces**. Two separate networks (`ns1` and `ns2`) are connected through a `router-ns` namespace that acts as a virtual router between two Linux bridges (`br0` and `br1`).

**Key concepts used:**
- Linux Network Namespaces
- Virtual Ethernet (veth) pairs
- Linux Bridges
- IP Routing & Forwarding

---

## Network Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                        HOST SYSTEM                              │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │   ns1    │    │   br0    │    │   br1    │    │   ns2    │  │
│  │          │    │          │    │          │    │          │  │
│  │10.0.1.2/24◄──►(bridge)  │    │ (bridge) ◄──►10.0.2.2/24│  │
│  └──────────┘    └────┬─────┘    └────┬─────┘    └──────────┘  │
│                       │              │                          │
│                  ┌────▼──────────────▼────┐                    │
│                  │       router-ns         │                    │
│                  │  veth-r1: 10.0.1.1/24  │                    │
│                  │  veth-r2: 10.0.2.1/24  │                    │
│                  │  ip_forward = 1         │                    │
│                  └────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

### Interface Connections

| veth Pair | One End | Other End | Bridge |
|-----------|---------|-----------|--------|
| `veth-ns1` ↔ `veth-br0-ns` | ns1 | br0 | br0 |
| `veth-r1` ↔ `veth-br0-r` | router-ns | br0 | br0 |
| `veth-ns2` ↔ `veth-br1-ns` | ns2 | br1 | br1 |
| `veth-r2` ↔ `veth-br1-r` | router-ns | br1 | br1 |

---

## IP Addressing Scheme

| Namespace | Interface | IP Address | Role |
|-----------|-----------|------------|------|
| `ns1` | `veth-ns1` | `10.0.1.2/24` | Host on Network 1 |
| `router-ns` | `veth-r1` | `10.0.1.1/24` | Gateway for Network 1 |
| `router-ns` | `veth-r2` | `10.0.2.1/24` | Gateway for Network 2 |
| `ns2` | `veth-ns2` | `10.0.2.2/24` | Host on Network 2 |

### Routing Table

| Namespace | Destination | Gateway | Notes |
|-----------|-------------|---------|-------|
| `ns1` | `0.0.0.0/0` (default) | `10.0.1.1` | All traffic → router |
| `ns2` | `0.0.0.0/0` (default) | `10.0.2.1` | All traffic → router |
| `router-ns` | `10.0.1.0/24` | — | Direct (veth-r1) |
| `router-ns` | `10.0.2.0/24` | — | Direct (veth-r2) |

---

## Prerequisites

- Linux system (Ubuntu/Debian recommended)
- Root/sudo privileges
- `iproute2` package installed (`ip`, `bridge` commands)

```bash
# Verify tools are available
ip --version
bridge --version
```

---

## Step-by-Step Implementation

### STEP 1 — Create Network Bridges

Create and bring up two Linux bridges: `br0` (for Network 1) and `br1` (for Network 2).

```bash
sudo ip link add br0 type bridge
sudo ip link add br1 type bridge
sudo ip link set br0 up
sudo ip link set br1 up

# Verify
ip link show type bridge
```

**Expected output:** `br0` and `br1` listed as `BROADCAST,MULTICAST,UP`.

---

### STEP 2 — Create Network Namespaces

Create three isolated network namespaces.

```bash
sudo ip netns add ns1
sudo ip netns add ns2
sudo ip netns add router-ns

# Verify
ip netns list
```

**Expected output:**
```
router-ns
ns2
ns1
```

---

### STEP 3 — Create Virtual Ethernet (veth) Pairs

Each veth pair acts like a virtual cable between two points.

```bash
# ns1 <-> br0
sudo ip link add veth-ns1 type veth peer name veth-br0-ns

# ns2 <-> br1
sudo ip link add veth-ns2 type veth peer name veth-br1-ns

# router <-> br0
sudo ip link add veth-r1 type veth peer name veth-br0-r

# router <-> br1
sudo ip link add veth-r2 type veth peer name veth-br1-r

# Verify — all 8 veth interfaces should appear
ip link show type veth
```

---

### STEP 4 — Move Interfaces into Namespaces

Assign each veth end to its correct namespace.

```bash
sudo ip link set veth-ns1 netns ns1
sudo ip link set veth-ns2 netns ns2
sudo ip link set veth-r1 netns router-ns
sudo ip link set veth-r2 netns router-ns

# Verify
sudo ip netns exec ns1 ip link
sudo ip netns exec router-ns ip link
```

---

### STEP 5 — Connect Interfaces to Bridges

Attach the host-side veth ends to their respective bridges and bring them up.

```bash
# Attach to bridges
sudo ip link set veth-br0-ns master br0
sudo ip link set veth-br0-r  master br0

sudo ip link set veth-br1-ns master br1
sudo ip link set veth-br1-r  master br1

# Bring up bridge-side interfaces
sudo ip link set veth-br0-ns up
sudo ip link set veth-br0-r  up
sudo ip link set veth-br1-ns up
sudo ip link set veth-br1-r  up

# Verify
bridge link
```

**Expected output:** All 4 interfaces shown as members of `br0` or `br1`.

---

### STEP 6 — Assign IP Addresses

#### ns1

```bash
sudo ip netns exec ns1 ip addr add 10.0.1.2/24 dev veth-ns1
sudo ip netns exec ns1 ip link set veth-ns1 up
sudo ip netns exec ns1 ip link set lo up

# Verify
sudo ip netns exec ns1 ip addr
```

#### ns2

```bash
sudo ip netns exec ns2 ip addr add 10.0.2.2/24 dev veth-ns2
sudo ip netns exec ns2 ip link set veth-ns2 up
sudo ip netns exec ns2 ip link set lo up

# Verify
sudo ip netns exec ns2 ip addr
```

#### router-ns

```bash
sudo ip netns exec router-ns ip addr add 10.0.1.1/24 dev veth-r1
sudo ip netns exec router-ns ip addr add 10.0.2.1/24 dev veth-r2
sudo ip netns exec router-ns ip link set veth-r1 up
sudo ip netns exec router-ns ip link set veth-r2 up
sudo ip netns exec router-ns ip link set lo up

# Verify
sudo ip netns exec router-ns ip addr
```

---

### STEP 7 — Configure Default Routes

Tell `ns1` and `ns2` to send all traffic through the router.

```bash
sudo ip netns exec ns1 ip route add default via 10.0.1.1
sudo ip netns exec ns2 ip route add default via 10.0.2.1

# Verify
sudo ip netns exec ns1 ip route
sudo ip netns exec ns2 ip route
```

**Expected output for ns1:**
```
default via 10.0.1.1 dev veth-ns1
10.0.1.0/24 dev veth-ns1 proto kernel scope link src 10.0.1.2
```

---

### STEP 8 — Enable IP Forwarding in Router

Allow the router namespace to forward packets between its two interfaces.

```bash
sudo ip netns exec router-ns sysctl -w net.ipv4.ip_forward=1

# Verify
sudo ip netns exec router-ns sysctl net.ipv4.ip_forward
```

**Expected output:** `net.ipv4.ip_forward = 1`

---

## Automation Script

The entire setup can be automated with a single bash script:

```bash
#!/bin/bash
# network_setup.sh — Full setup and teardown for namespace simulation

set -e

setup() {
    echo "=== Setting up network simulation ==="

    # Bridges
    ip link add br0 type bridge
    ip link add br1 type bridge
    ip link set br0 up
    ip link set br1 up

    # Namespaces
    ip netns add ns1
    ip netns add ns2
    ip netns add router-ns

    # veth pairs
    ip link add veth-ns1 type veth peer name veth-br0-ns
    ip link add veth-ns2 type veth peer name veth-br1-ns
    ip link add veth-r1  type veth peer name veth-br0-r
    ip link add veth-r2  type veth peer name veth-br1-r

    # Move into namespaces
    ip link set veth-ns1 netns ns1
    ip link set veth-ns2 netns ns2
    ip link set veth-r1  netns router-ns
    ip link set veth-r2  netns router-ns

    # Attach to bridges & bring up
    ip link set veth-br0-ns master br0 && ip link set veth-br0-ns up
    ip link set veth-br0-r  master br0 && ip link set veth-br0-r  up
    ip link set veth-br1-ns master br1 && ip link set veth-br1-ns up
    ip link set veth-br1-r  master br1 && ip link set veth-br1-r  up

    # IP addresses
    ip netns exec ns1       ip addr add 10.0.1.2/24 dev veth-ns1
    ip netns exec ns1       ip link set veth-ns1 up
    ip netns exec ns1       ip link set lo up

    ip netns exec ns2       ip addr add 10.0.2.2/24 dev veth-ns2
    ip netns exec ns2       ip link set veth-ns2 up
    ip netns exec ns2       ip link set lo up

    ip netns exec router-ns ip addr add 10.0.1.1/24 dev veth-r1
    ip netns exec router-ns ip addr add 10.0.2.1/24 dev veth-r2
    ip netns exec router-ns ip link set veth-r1 up
    ip netns exec router-ns ip link set veth-r2 up
    ip netns exec router-ns ip link set lo up

    # Default routes
    ip netns exec ns1       ip route add default via 10.0.1.1
    ip netns exec ns2       ip route add default via 10.0.2.1

    # IP forwarding
    ip netns exec router-ns sysctl -w net.ipv4.ip_forward=1

    echo "=== Setup complete! ==="
    echo "Test with: sudo ip netns exec ns1 ping -c 4 10.0.2.2"
}

teardown() {
    echo "=== Tearing down network simulation ==="

    # Delete namespaces (also removes veth interfaces inside them)
    ip netns del ns1       2>/dev/null || true
    ip netns del ns2       2>/dev/null || true
    ip netns del router-ns 2>/dev/null || true

    # Remove bridge-side veth interfaces
    ip link del veth-br0-ns 2>/dev/null || true
    ip link del veth-br1-ns 2>/dev/null || true
    ip link del veth-br0-r  2>/dev/null || true
    ip link del veth-br1-r  2>/dev/null || true

    # Remove bridges
    ip link set br0 down && ip link del br0 2>/dev/null || true
    ip link set br1 down && ip link del br1 2>/dev/null || true

    echo "=== Teardown complete! ==="
}

case "$1" in
    setup)    setup    ;;
    teardown) teardown ;;
    *)
        echo "Usage: sudo bash network_setup.sh [setup|teardown]"
        exit 1
        ;;
esac
```

**Usage:**
```bash
sudo bash network_setup.sh setup
sudo bash network_setup.sh teardown
```

---

## Testing & Verification

### Final Connectivity Test

```bash
sudo ip netns exec ns1 ping -c 4 10.0.2.2
```

**Expected output:**
```
PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.
64 bytes from 10.0.2.2: icmp_seq=1 ttl=63 time=X ms
64 bytes from 10.0.2.2: icmp_seq=2 ttl=63 time=X ms
64 bytes from 10.0.2.2: icmp_seq=3 ttl=63 time=X ms
64 bytes from 10.0.2.2: icmp_seq=4 ttl=63 time=X ms

--- 10.0.2.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
```

> **Note:** TTL is 63 (not 64) because the packet hops through the router namespace, which decrements TTL by 1. This confirms routing is working correctly.

### Additional Test Commands

```bash
# Test ns1 → router gateway
sudo ip netns exec ns1 ping -c 2 10.0.1.1

# Test ns2 → router gateway
sudo ip netns exec ns2 ping -c 2 10.0.2.1

# Test ns2 → ns1 (reverse direction)
sudo ip netns exec ns2 ping -c 4 10.0.1.2

# Check all routes
sudo ip netns exec ns1 ip route
sudo ip netns exec ns2 ip route
sudo ip netns exec router-ns ip route

# Verify bridge members
bridge link

# Verify IP forwarding is ON in router
sudo ip netns exec router-ns sysctl net.ipv4.ip_forward
```

---

## Cleanup

Always clean up after testing to avoid conflicts with other network configurations.

```bash
# Delete namespaces
sudo ip netns del ns1
sudo ip netns del ns2
sudo ip netns del router-ns

# Delete remaining bridge-side veth interfaces
sudo ip link del veth-br0-ns
sudo ip link del veth-br1-ns
sudo ip link del veth-br0-r
sudo ip link del veth-br1-r

# Delete bridges
sudo ip link set br0 down && sudo ip link del br0
sudo ip link set br1 down && sudo ip link del br1

# Verify everything is gone
ip netns list
ip link show type bridge
ip link show type veth
```

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Ping fails between ns1 and ns2 | IP forwarding not enabled | `sudo ip netns exec router-ns sysctl -w net.ipv4.ip_forward=1` |
| Interface not found | veth already moved to namespace | Use `ip netns exec <ns> ip link` to check inside the namespace |
| `File exists` error on IP assign | IP already assigned | Run teardown first, then setup again |
| Bridge not forwarding | STP blocking port | Wait a few seconds or disable STP: `ip link set br0 type bridge stp_state 0` |
| `RTNETLINK answers: File exists` | Duplicate IP assignment | Namespace already configured; run teardown and retry |

---

## Technical Notes

- All commands require **root privileges** (`sudo`)
- Tested on **Ubuntu** via VS Code / Poridhi cloud environment
- Deleting a namespace automatically removes all veth interfaces **inside** it; only the host-side peers need manual cleanup
- The `ttl=63` in ping output confirms proper routing through `router-ns` (one hop decrements TTL by 1 from the default 64)

---

*Project by: Poridhian | Linux Network Namespaces Assignment*
