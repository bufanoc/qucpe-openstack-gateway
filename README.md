# QuCPE-OpenStack Gateway

> **Extending OpenStack tenant networks to physical infrastructure using QNAP QuCPE-7012 as an OVN L2 Gateway**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![OpenStack](https://img.shields.io/badge/OpenStack-Compatible-red.svg)](https://www.openstack.org/)
[![OVN](https://img.shields.io/badge/OVN-20.12.0-blue.svg)](https://www.ovn.org/)

## Overview

This project documents how to repurpose a **QNAP QuCPE-7012** Network Virtualization Premise Equipment (vCPE) as an **OpenStack L2 Gateway**, enabling seamless connectivity between OpenStack tenant networks (Geneve overlays) and physical infrastructure at remote/edge sites.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        OpenStack Cloud                                   │
│                                                                          │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐         │
│   │   Compute   │    │   Compute   │    │   Network Node      │         │
│   │    Node 1   │    │    Node 2   │    │   (OVN Central)     │         │
│   │             │    │             │    │                     │         │
│   │ ovn-controller   │ ovn-controller   │  ovn-northd         │         │
│   └──────┬──────┘    └──────┬──────┘    │  ovn-nb-db          │         │
│          │                  │           │  ovn-sb-db          │         │
│          │                  │           └──────────┬──────────┘         │
│          │                  │                      │                    │
│          └──────────────────┴──────────────────────┘                    │
│                              │                                          │
│                        Geneve Overlay                                   │
│                     (Tenant Networks)                                   │
│                              │                                          │
└──────────────────────────────┼──────────────────────────────────────────┘
                               │
                         WAN / Internet
                               │
                    ┌──────────┴──────────┐
                    │                     │
                    │   QNAP QuCPE-7012   │
                    │   ═══════════════   │
                    │                     │
                    │   ovn-controller ◄──┼── Joins as remote chassis
                    │   br-int            │
                    │      │              │
                    │   L2 Gateway        │
                    │   ┌──┴───┐          │
                    │   │eno5-9│          │
                    │   └──┬───┘          │
                    └──────┼──────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
        ┌─────┴─────┐            ┌──────┴─────┐
        │  Physical │            │  Physical  │
        │  Server 1 │            │  Server 2  │
        │           │            │            │
        │ (OpenStack│            │ (OpenStack │
        │  Tenant   │            │  Tenant    │
        │  Network) │            │  Network)  │
        └───────────┘            └────────────┘
```

## Why QuCPE-7012?

The QNAP QuCPE-7012 is uniquely suited for this role because it **already runs the same SDN stack** that OpenStack Neutron ML2/OVN uses:

| Feature | QuCPE-7012 | Typical L2GW Solutions |
|---------|------------|------------------------|
| OVN Version | 20.12.0 (native) | Requires installation |
| OVS Version | 2.14.90 with DPDK | Requires installation |
| Geneve Support | Built-in | Requires configuration |
| L2 Gateway | Pre-configured | Complex setup |
| SR-IOV | 128 VFs (32 per 10GbE) | Hardware dependent |
| Management | Web UI + CLI | CLI only |
| Form Factor | 1U Rackmount | Varies |

### The Key Insight

QNAP built their "Service Composer" network virtualization on top of **standard OVN/OVS**. By simply pointing the QuCPE's `ovn-controller` at an OpenStack OVN Southbound database instead of its local one, the device seamlessly joins the OpenStack cluster as a remote chassis.

**No firmware modifications. No package installations. Fully reversible.**

## Hardware Specifications

### QNAP QuCPE-7012-D2166NT-64G

| Component | Specification | OpenStack Relevance |
|-----------|---------------|---------------------|
| **CPU** | Intel Xeon D-2166NT | 12 cores / 24 threads @ 2.0GHz (3.0GHz boost) |
| **Architecture** | x86_64 | Native OpenStack compatibility |
| **Memory** | 64GB DDR4 ECC (expandable) | Sufficient for VNF workloads |
| **Storage** | 2x NVMe + 2x SATA (RAID1) | Local storage for edge computing |

### Network Interfaces

```
┌─────────────────────────────────────────────────────────────────────┐
│                      QuCPE-7012 Network Ports                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   MANAGEMENT        2.5GbE COPPER (RJ45)         10GbE SFP+         │
│   ┌───────┐    ┌───┬───┬───┬───┬───┬───┬───┐   ┌───┬───┬───┬───┐   │
│   │ IPMI  │    │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │   │ 9 │10 │11 │12 │   │
│   │ (OOB) │    │eno│eno│eno│eno│eno│eno│eno│   │eno│eno│eno│eno│   │
│   └───────┘    └───┴───┴───┴───┴───┴───┴───┘   └───┴───┴───┴───┘   │
│                  │                   │             │               │
│                  │                   │             │               │
│              Management           L2 Gateway    High-Speed         │
│              + WAN Uplink         to Physical   Tenant Traffic     │
│                                   Servers       (SR-IOV capable)   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

| Port(s) | Type | Speed | SR-IOV VFs | Recommended Use |
|---------|------|-------|------------|-----------------|
| eno1 | RJ45 | 2.5GbE | N/A | Management / WAN uplink |
| eno2-4 | RJ45 | 2.5GbE | N/A | L2GW to physical servers |
| eno5-8 | RJ45 | 2.5GbE | N/A | L2GW to physical servers |
| eno9-12 | SFP+ | 10GbE | 32 each | High-speed tenant traffic / SR-IOV |

### SR-IOV Capabilities

Each 10GbE SFP+ port supports **32 Virtual Functions**, enabling:
- Direct VM-to-wire connectivity
- Hardware-accelerated packet processing
- Bypass of software switching for latency-sensitive workloads

```
Total SR-IOV VFs Available: 4 ports × 32 VFs = 128 Virtual Functions
```

## Software Stack

### Pre-installed SDN Components

| Component | Version | Package |
|-----------|---------|---------|
| Open vSwitch | 2.14.90 | `openvswitch-switch` (QNAP +q47) |
| OVS DPDK | 2.14.90 | `openvswitch-switch-dpdk` |
| OVN Central | 20.12.0 | `ovn-central` (QNAP +q18) |
| OVN Host | 20.12.0 | `ovn-host` |
| DPDK | 20.11 | Pre-installed |

### QNAP-Specific Components

| Component | Purpose | Impact on Integration |
|-----------|---------|----------------------|
| `vnd` | Virtual Network Daemon - syncs OVN to Service Composer | Non-essential for OpenStack mode |
| `vn-utils` | CLI utilities | Useful for debugging |
| Service Composer | Web UI for network management | Disabled in OpenStack mode |

### Operating System

| Property | Value |
|----------|-------|
| Base OS | Ubuntu 18.04 LTS |
| Kernel | 5.10.60-qnap (customized) |
| QNE Version | 1.0.6.q322 |

## Current OVN Topology (Standalone Mode)

Before integration, the QuCPE runs OVN in standalone mode:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    QuCPE OVN Architecture (Standalone)               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    OVN Northbound DB                         │   │
│   │                  /etc/config/ovnnb_db.db                     │   │
│   │                                                              │   │
│   │  Logical Router: r0                                          │   │
│   │    ├── r0-System default virtual switch (10.222.0.1/16)     │   │
│   │    ├── r0-docker-browser-station (10.3.1.1/24)              │   │
│   │    └── gw-host (192.0.0.255/31)                             │   │
│   │                                                              │   │
│   │  Logical Switches:                                           │   │
│   │    ├── System default virtual switch (VMs/Containers)       │   │
│   │    ├── Virtual Switch 1 ──► L2GW ──► eno5 (physical)        │   │
│   │    ├── Virtual Switch 2 ──► L2GW ──► eno6 (physical)        │   │
│   │    └── Virtual Switch 3 ──► L2GW ──► eno9 (physical)        │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    OVN Southbound DB                         │   │
│   │                  /var/lib/ovn/ovnsb_db.db                    │   │
│   │                                                              │   │
│   │  Chassis: <your-system-uuid>                   │   │
│   │    Encap: geneve @ 127.0.0.1  ◄── LOCAL ONLY                │   │
│   │    Port Bindings: L2GW ports for physical NICs              │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                     ovn-controller                           │   │
│   │              Connects to: unix:/var/run/ovn/ovnsb_db.sock    │   │
│   │              Programs: br-int flow rules                     │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Integration with OpenStack

### Phase 1: L2 Gateway Mode (Network Extension)

Connect physical infrastructure to OpenStack tenant networks:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    QuCPE as OpenStack L2 Gateway                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   OpenStack Cloud                          QuCPE-7012                │
│   ══════════════                          ═══════════                │
│                                                                      │
│   ┌─────────────────┐                    ┌─────────────────┐        │
│   │  OVN Northbound │                    │  ovn-controller │        │
│   │     Database    │                    │                 │        │
│   └────────┬────────┘                    │  Receives flow  │        │
│            │                             │  rules from     │        │
│            ▼                             │  OpenStack OVN  │        │
│   ┌─────────────────┐    TCP:6642        └────────┬────────┘        │
│   │  OVN Southbound │◄───────────────────────────►│                 │
│   │     Database    │    Geneve Tunnel           │                 │
│   └─────────────────┘                             ▼                 │
│                                          ┌─────────────────┐        │
│   Tenant Network: web-tier               │     br-int      │        │
│   Subnet: 10.0.1.0/24                    │                 │        │
│   Geneve VNI: 12345                      │  L2GW Ports     │        │
│                                          │    │   │   │    │        │
│                                          └────┼───┼───┼────┘        │
│                                               │   │   │             │
│                                             eno5 eno6 eno9          │
│                                               │   │   │             │
│                                               ▼   ▼   ▼             │
│                                          Physical Servers           │
│                                          on 10.0.1.0/24             │
│                                          (OpenStack tenant net)     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Phase 2: Edge Compute Node (Future)

Leverage the Xeon D-2166NT for lightweight VNF workloads:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    QuCPE as OpenStack Edge Node                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                      QuCPE-7012                              │   │
│   │                                                              │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│   │   │   VNF 1     │  │   VNF 2     │  │   VNF 3     │         │   │
│   │   │  (Firewall) │  │   (Router)  │  │  (Load Bal) │         │   │
│   │   │             │  │             │  │             │         │   │
│   │   │  SR-IOV VF  │  │  SR-IOV VF  │  │  DPDK Port  │         │   │
│   │   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │   │
│   │          │                │                │                 │   │
│   │          └────────────────┼────────────────┘                 │   │
│   │                           │                                  │   │
│   │                      ┌────┴────┐                             │   │
│   │                      │  br-int │                             │   │
│   │                      └────┬────┘                             │   │
│   │                           │                                  │   │
│   │   ┌───────────────────────┼───────────────────────┐         │   │
│   │   │         L2 Gateway Physical Ports              │         │   │
│   │   │    eno5    eno6    eno9    eno10   eno11      │         │   │
│   │   └─────┬───────┬───────┬───────┬───────┬─────────┘         │   │
│   │         │       │       │       │       │                    │   │
│   └─────────┼───────┼───────┼───────┼───────┼────────────────────┘   │
│             │       │       │       │       │                        │
│             ▼       ▼       ▼       ▼       ▼                        │
│        Physical  Physical  High-Speed Tenant Traffic                 │
│        Servers   Servers   (10GbE with SR-IOV passthrough)          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Project Roadmap

### Milestone 1: L2 Gateway Integration *(Current Focus)*
- [ ] Document standalone OVN configuration
- [ ] Create backup/restore procedures
- [ ] Test connection to OpenStack OVN SB
- [ ] Validate L2 gateway functionality
- [ ] Document rollback procedures

### Milestone 2: Production Hardening
- [ ] SSL/TLS for OVN SB connection
- [ ] High availability configuration
- [ ] Monitoring and alerting integration
- [ ] Performance benchmarking

### Milestone 3: Edge Compute
- [ ] Nova compute agent installation
- [ ] SR-IOV configuration for VMs
- [ ] DPDK optimization
- [ ] VNF deployment testing

### Milestone 4: Advanced Features
- [ ] Service function chaining
- [ ] Hardware offload exploration
- [ ] Multi-site federation
- [ ] Integration with OpenStack Neutron L2GW service

## Quick Start

### Prerequisites

- QNAP QuCPE-7012 running QNE 1.0.6 or later
- OpenStack cloud with OVN ML2 mechanism driver
- Network connectivity between QuCPE and OpenStack OVN SB database
- SSH access to QuCPE

### Backup Current Configuration

```bash
# Save current OVS external_ids
ovs-vsctl get open_vswitch . external_ids > ~/ovs-backup.txt

# Backup OVN databases
cp /etc/config/ovnnb_db.db /etc/config/ovnnb_db.db.backup
cp /var/lib/ovn/ovnsb_db.db /var/lib/ovn/ovnsb_db.db.backup
```

### Connect to OpenStack

```bash
# Point ovn-controller to OpenStack's OVN Southbound DB
ovs-vsctl set open . external_ids:ovn-remote="tcp:<OPENSTACK_OVN_SB_IP>:6642"

# Set tunnel endpoint IP (QuCPE's IP reachable from OpenStack)
ovs-vsctl set open . external_ids:ovn-encap-ip="<QUCPE_TUNNEL_IP>"

# Restart ovn-controller to connect
systemctl restart ovn-controller
```

### Verify Connection

```bash
# Check chassis registration
ovn-sbctl show

# Should show QuCPE as a chassis in the cluster
```

### Rollback to Standalone

```bash
# Restore local OVN connection
ovs-vsctl set open . external_ids:ovn-remote="unix:/var/run/ovn/ovnsb_db.sock"
ovs-vsctl set open . external_ids:ovn-encap-ip="127.0.0.1"

# Restart services
systemctl restart ovn-ovsdb-server-nb ovn-ovsdb-server-sb ovn-northd ovn-controller

# Restart QNAP's vnd for Service Composer
systemctl restart vnd
```

## Documentation

| Document | Description |
|----------|-------------|
| [Hardware Specifications](docs/hardware.md) | Detailed hardware specs and capabilities |
| [Architecture Guide](docs/architecture.md) | OVN/OVS architecture on QuCPE |
| [Integration Guide](docs/integration.md) | Step-by-step OpenStack integration |
| [Troubleshooting](docs/troubleshooting.md) | Common issues and solutions |
| [Roadmap](docs/roadmap.md) | Detailed project roadmap |

## Contributing

Contributions are welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) file for details.

## Acknowledgments

- QNAP for building a solid OVN-based vCPE platform
- The OVN/OVS community for excellent documentation
- OpenStack Neutron team for ML2/OVN integration

---

**Disclaimer:** This project is not affiliated with or endorsed by QNAP Systems, Inc. QNAP and QuCPE are trademarks of QNAP Systems, Inc. OpenStack is a trademark of the OpenStack Foundation.
