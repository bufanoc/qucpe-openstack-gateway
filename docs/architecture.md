# Architecture Guide

## OVN/OVS Architecture on QuCPE-7012

This document explains how Open Virtual Network (OVN) and Open vSwitch (OVS) are deployed on the QNAP QuCPE-7012, and how this architecture enables integration with OpenStack.

## Component Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     QuCPE-7012 SDN Software Stack                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        Management Plane                              │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │   │
│   │  │   Service   │  │    vnd      │  │      QNAP QNE APIs          │  │   │
│   │  │  Composer   │──│  (Virtual   │──│  (qne-qs-network, etc.)     │  │   │
│   │  │   Web UI    │  │   Network   │  │                             │  │   │
│   │  │             │  │   Daemon)   │  │                             │  │   │
│   │  └─────────────┘  └──────┬──────┘  └─────────────────────────────┘  │   │
│   │                          │                                           │   │
│   │                          │ Monitors & Publishes to Redis             │   │
│   │                          ▼                                           │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         Control Plane                                │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │                    OVN Northbound Database                   │    │   │
│   │  │                  /etc/config/ovnnb_db.db                     │    │   │
│   │  │                                                              │    │   │
│   │  │  • Logical switches, routers, ports                          │    │   │
│   │  │  • ACLs, NAT rules, load balancers                          │    │   │
│   │  │  • L2 gateway configurations                                 │    │   │
│   │  └──────────────────────────┬──────────────────────────────────┘    │   │
│   │                             │                                        │   │
│   │                             ▼                                        │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │                      ovn-northd                              │    │   │
│   │  │              (Northbound to Southbound translator)           │    │   │
│   │  └──────────────────────────┬──────────────────────────────────┘    │   │
│   │                             │                                        │   │
│   │                             ▼                                        │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │                   OVN Southbound Database                    │    │   │
│   │  │                 /var/lib/ovn/ovnsb_db.db                     │    │   │
│   │  │                                                              │    │   │
│   │  │  • Physical bindings (chassis, ports)                        │    │   │
│   │  │  • Logical flows (compiled from NB)                          │    │   │
│   │  │  • Encapsulation info (Geneve tunnels)                       │    │   │
│   │  └──────────────────────────┬──────────────────────────────────┘    │   │
│   │                             │                                        │   │
│   └─────────────────────────────┼───────────────────────────────────────┘   │
│                                 │                                            │
│   ┌─────────────────────────────┼───────────────────────────────────────┐   │
│   │                         Data Plane                                   │   │
│   │                             │                                        │   │
│   │                             ▼                                        │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │                    ovn-controller                            │    │   │
│   │  │           (Local agent - programs OVS flows)                 │    │   │
│   │  └──────────────────────────┬──────────────────────────────────┘    │   │
│   │                             │                                        │   │
│   │                             ▼                                        │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │                    Open vSwitch                              │    │   │
│   │  │                                                              │    │   │
│   │  │   ┌─────────────────────────────────────────────────────┐   │    │   │
│   │  │   │                    br-int                            │   │    │   │
│   │  │   │              (Integration Bridge)                    │   │    │   │
│   │  │   │                                                      │   │    │   │
│   │  │   │  • All VM/Container ports connect here               │   │    │   │
│   │  │   │  • OVN controller programs flows here                │   │    │   │
│   │  │   │  • Patch ports to provider bridges                   │   │    │   │
│   │  │   └─────────────────────────────────────────────────────┘   │    │   │
│   │  │                         │                                    │    │   │
│   │  │         ┌───────────────┼───────────────┐                    │    │   │
│   │  │         ▼               ▼               ▼                    │    │   │
│   │  │   ┌───────────┐   ┌───────────┐   ┌───────────┐             │    │   │
│   │  │   │ VS 1 Br   │   │ VS 2 Br   │   │ VS 3 Br   │             │    │   │
│   │  │   │ (L2GW)    │   │ (L2GW)    │   │ (L2GW)    │             │    │   │
│   │  │   │           │   │           │   │           │             │    │   │
│   │  │   │   eno5    │   │   eno6    │   │   eno9    │             │    │   │
│   │  │   └─────┬─────┘   └─────┬─────┘   └─────┬─────┘             │    │   │
│   │  │         │               │               │                    │    │   │
│   │  └─────────┼───────────────┼───────────────┼────────────────────┘    │   │
│   │            │               │               │                         │   │
│   └────────────┼───────────────┼───────────────┼─────────────────────────┘   │
│                │               │               │                             │
│                ▼               ▼               ▼                             │
│           Physical         Physical        Physical                          │
│           Network          Network         Network                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## OVN Databases

### Northbound Database (NB)

The NB database stores the **logical** network configuration - what the network should look like.

**Location:** `/etc/config/ovnnb_db.db`

**Key Tables:**

| Table | Purpose |
|-------|---------|
| `Logical_Switch` | Virtual L2 networks |
| `Logical_Router` | Virtual L3 routers |
| `Logical_Switch_Port` | Ports on switches (VMs, containers, router ports) |
| `Logical_Router_Port` | Router interfaces |
| `ACL` | Access control lists |
| `NAT` | NAT rules |
| `Load_Balancer` | L4 load balancing |

### Southbound Database (SB)

The SB database stores the **physical** bindings and compiled flows.

**Location:** `/var/lib/ovn/ovnsb_db.db`

**Key Tables:**

| Table | Purpose |
|-------|---------|
| `Chassis` | Physical hosts running ovn-controller |
| `Encap` | Tunnel encapsulation (Geneve, VXLAN) |
| `Port_Binding` | Maps logical ports to physical locations |
| `Logical_Flow` | Compiled flow rules from NB |
| `Multicast_Group` | BUM traffic handling |

## Current Standalone Configuration

### OVS External IDs

```bash
$ ovs-vsctl get open_vswitch . external_ids

{
  hostname=GATEPROD01,
  ovn-bridge-mappings="...",
  ovn-encap-ip="127.0.0.1",           # ◄── Local only (standalone)
  ovn-encap-type=geneve,
  ovn-nb="unix:/var/run/ovn/ovnnb_db.sock",
  ovn-remote="unix:/var/run/ovn/ovnsb_db.sock",  # ◄── Local socket
  system-id=<your-system-uuid>
}
```

### Chassis Registration

```bash
$ ovn-sbctl show

Chassis <your-system-uuid>
    hostname: GATEPROD01
    Encap geneve
        ip: "127.0.0.1"
        options: {csum="true"}
    Port_Binding "Virtual Switch 1-l2gw"
    Port_Binding "Virtual Switch 2-l2gw"
    Port_Binding "Virtual Switch 3-l2gw"
```

## L2 Gateway Architecture

The QuCPE uses OVN's `l2gateway` port type to bridge logical networks to physical ports.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         L2 Gateway Port Flow                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   OVN Logical Switch                    OVS Bridges                          │
│   ══════════════════                    ══════════════                        │
│                                                                              │
│   ┌─────────────────────┐              ┌─────────────────────┐              │
│   │  "Virtual Switch 1" │              │       br-int        │              │
│   │                     │              │                     │              │
│   │  ┌───────────────┐  │   patch      │  ┌───────────────┐  │              │
│   │  │  l2gw port    │──┼──────────────┼──│  patch port   │  │              │
│   │  │  (type:       │  │   ports      │  │               │  │              │
│   │  │   l2gateway)  │  │              │  └───────┬───────┘  │              │
│   │  └───────────────┘  │              │          │          │              │
│   │                     │              └──────────┼──────────┘              │
│   │  ┌───────────────┐  │                        │                          │
│   │  │  VM port      │  │              ┌──────────┼──────────┐              │
│   │  │  (regular)    │  │              │  Provider Bridge    │              │
│   │  └───────────────┘  │              │  (UUID-named)       │              │
│   │                     │              │          │          │              │
│   └─────────────────────┘              │  ┌───────┴───────┐  │              │
│                                        │  │    eno5       │  │              │
│                                        │  │  (physical)   │  │              │
│                                        │  └───────┬───────┘  │              │
│                                        └──────────┼──────────┘              │
│                                                   │                          │
│                                                   ▼                          │
│                                            Physical Network                  │
│                                            (Servers, switches)               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bridge Mappings

The `ovn-bridge-mappings` external_id connects OVN logical networks to OVS provider bridges:

```
ovn-bridge-mappings="<logical-net-uuid>:<ovs-bridge-uuid>,..."
```

This tells ovn-controller which OVS bridge to use for each L2 gateway.

## OpenStack Integration Architecture

### Before Integration (Standalone)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              QuCPE Standalone                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         ┌─────────────────┐                                  │
│                         │   ovn-northd    │                                  │
│                         └────────┬────────┘                                  │
│                                  │                                           │
│   ┌──────────────────┐          │          ┌──────────────────┐             │
│   │  OVN NB (local)  │◄─────────┴─────────►│  OVN SB (local)  │             │
│   │  Unix socket     │                      │  Unix socket     │             │
│   └──────────────────┘                      └────────┬─────────┘             │
│                                                      │                       │
│                                                      ▼                       │
│                                             ┌────────────────┐               │
│                                             │ ovn-controller │               │
│                                             └────────────────┘               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### After Integration (OpenStack L2 Gateway)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           OpenStack Cloud                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│   │  Neutron API    │───►│   OVN NB        │───►│   ovn-northd    │         │
│   │  (ML2/OVN)      │    │   Database      │    │                 │         │
│   └─────────────────┘    └─────────────────┘    └────────┬────────┘         │
│                                                          │                   │
│                                                          ▼                   │
│                                                 ┌─────────────────┐          │
│                                                 │   OVN SB        │          │
│                                                 │   Database      │          │
│                                                 │   TCP:6642      │          │
│                                                 └────────┬────────┘          │
│                                                          │                   │
└──────────────────────────────────────────────────────────┼───────────────────┘
                                                           │
                                                    Geneve Tunnel
                                                    (over WAN/VPN)
                                                           │
┌──────────────────────────────────────────────────────────┼───────────────────┐
│                              QuCPE-7012                  │                    │
├──────────────────────────────────────────────────────────┼───────────────────┤
│                                                          │                    │
│                         ┌────────────────────────────────┘                    │
│                         │                                                     │
│                         ▼                                                     │
│                ┌─────────────────┐                                            │
│                │ ovn-controller  │◄── Connects to remote OVN SB               │
│                │                 │    instead of local socket                 │
│                └────────┬────────┘                                            │
│                         │                                                     │
│                         ▼                                                     │
│                ┌─────────────────┐                                            │
│                │     br-int      │                                            │
│                │  (programmed    │                                            │
│                │   by remote     │                                            │
│                │   OVN flows)    │                                            │
│                └────────┬────────┘                                            │
│                         │                                                     │
│              ┌──────────┼──────────┐                                          │
│              ▼          ▼          ▼                                          │
│           L2GW       L2GW       L2GW                                          │
│           eno5       eno6       eno9                                          │
│              │          │          │                                          │
└──────────────┼──────────┼──────────┼──────────────────────────────────────────┘
               │          │          │
               ▼          ▼          ▼
          Physical    Physical    Physical
          Servers     Servers     Servers
          (on OpenStack tenant networks)
```

## Key Configuration Changes

### Standalone → OpenStack

| Parameter | Standalone Value | OpenStack Value |
|-----------|------------------|-----------------|
| `ovn-remote` | `unix:/var/run/ovn/ovnsb_db.sock` | `tcp:<openstack-sb>:6642` |
| `ovn-encap-ip` | `127.0.0.1` | `<qucpe-public-ip>` |
| `ovn-northd` | Running locally | Not needed (runs on OpenStack) |
| `ovn-nb-db` | Running locally | Not needed |
| `ovn-sb-db` | Running locally | Not needed |

### Services State

| Service | Standalone | OpenStack Mode |
|---------|------------|----------------|
| `ovn-ovsdb-server-nb` | Running | Stopped (optional) |
| `ovn-ovsdb-server-sb` | Running | Stopped (optional) |
| `ovn-northd` | Running | Stopped |
| `ovn-controller` | Running (local) | Running (remote) |
| `openvswitch-switch` | Running | Running |
| `vnd` | Running | Stopped (optional) |

## Geneve Tunnel Details

### Encapsulation Format

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Geneve Packet Format                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Outer Ethernet Header                             │   │
│   │   Dst MAC: Next-hop MAC                                              │   │
│   │   Src MAC: QuCPE MAC                                                 │   │
│   │   EtherType: 0x0800 (IPv4)                                           │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Outer IP Header                                   │   │
│   │   Src IP: QuCPE tunnel endpoint (ovn-encap-ip)                       │   │
│   │   Dst IP: OpenStack compute/network node                             │   │
│   │   Protocol: UDP (17)                                                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Outer UDP Header                                  │   │
│   │   Src Port: Entropy (hash-based)                                     │   │
│   │   Dst Port: 6081 (Geneve)                                            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Geneve Header                                     │   │
│   │   VNI: Logical network identifier (24-bit)                           │   │
│   │   Options: OVN metadata (logical ports, etc.)                        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Inner Ethernet Frame                              │   │
│   │   (Original VM/Server traffic)                                       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### MTU Considerations

| Layer | Overhead | Notes |
|-------|----------|-------|
| Outer Ethernet | 14 bytes | Standard Ethernet |
| Outer IP | 20 bytes | IPv4 header |
| Outer UDP | 8 bytes | UDP header |
| Geneve | 8+ bytes | Base + options |
| **Total Overhead** | **~50-100 bytes** | Depends on options |

**Recommendation:** Set physical MTU to 1550+ or inner MTU to 1400 to avoid fragmentation.

## QNAP-Specific Components

### vnd (Virtual Network Daemon)

QNAP's `vnd` daemon provides:
- Monitoring of OVN NB database
- Publishing network state to Redis
- Integration with Service Composer UI

**In OpenStack mode:** Can be stopped, as Service Composer won't manage the network.

### Service Composer

Web-based GUI for visual network design. Relies on:
- `vnd` for OVN state
- `qne-qs-network` API
- Redis for pub/sub

**In OpenStack mode:** Non-functional (network managed by OpenStack Horizon/CLI).

## Next Steps

See [Integration Guide](integration.md) for step-by-step instructions on connecting to OpenStack.
