# Hardware Specifications

## QNAP QuCPE-7012-D2166NT-64G

The QuCPE-7012 is a carrier-grade Network Virtualization Premise Equipment (vCPE) designed for Software-Defined WAN (SD-WAN) and Network Function Virtualization (NFV) deployments.

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        QNAP QuCPE-7012 Front Panel                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────┐  ┌────────────────────────────────┐  ┌────────────────────────────┐ │
│  │PWR │  │     2.5GbE RJ45 (eno1-eno8)    │  │   10GbE SFP+ (eno9-eno12)  │ │
│  │LED │  │  ┌──┬──┬──┬──┬──┬──┬──┬──┐     │  │   ┌──┬──┬──┬──┐           │ │
│  │    │  │  │1 │2 │3 │4 │5 │6 │7 │8 │     │  │   │9 │10│11│12│           │ │
│  │SYS │  │  └──┴──┴──┴──┴──┴──┴──┴──┘     │  │   └──┴──┴──┴──┘           │ │
│  │LED │  │                                 │  │                           │ │
│  └────┘  └────────────────────────────────┘  └────────────────────────────┘ │
│                                                                              │
│  ┌──────┐  ┌──────┐  ┌────────────────────────────────────────────────────┐ │
│  │ USB  │  │ IPMI │  │              Console / Serial Port                 │ │
│  │ 3.0  │  │ RJ45 │  │                                                    │ │
│  └──────┘  └──────┘  └────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## CPU Specifications

| Property | Value |
|----------|-------|
| **Model** | Intel Xeon D-2166NT |
| **Microarchitecture** | Skylake-D |
| **Cores / Threads** | 12 / 24 |
| **Base Frequency** | 2.0 GHz |
| **Turbo Frequency** | 3.0 GHz |
| **L3 Cache** | 16.5 MB |
| **TDP** | 85W |
| **Lithography** | 14nm |
| **Memory Support** | DDR4-2133/2400 ECC |
| **Max Memory** | 512 GB |
| **PCIe Lanes** | 32 |

### CPU Features for NFV/SDN

| Feature | Support | Benefit |
|---------|---------|---------|
| Intel VT-x | Yes | Hardware virtualization |
| Intel VT-d | Yes | I/O virtualization (SR-IOV) |
| AES-NI | Yes | Hardware crypto acceleration |
| Intel QAT | Yes | Crypto/compression offload |
| Intel TSX | Yes | Transactional memory |
| AVX-512 | Yes | SIMD acceleration for DPDK |

## Memory Configuration

| Property | Value |
|----------|-------|
| **Installed** | 64 GB (2x32GB) |
| **Type** | DDR4 ECC RDIMM |
| **Speed** | 2400 MHz |
| **Slots** | 4 (2 populated) |
| **Max Supported** | 512 GB |

### Memory Topology

```
┌─────────────────────────────────────────────┐
│            Memory Configuration             │
├─────────────────────────────────────────────┤
│                                             │
│   Channel A          Channel B              │
│   ┌───────────┐      ┌───────────┐          │
│   │ DIMM A1   │      │ DIMM B1   │          │
│   │ 32GB DDR4 │      │ 32GB DDR4 │          │
│   │ ECC RDIMM │      │ ECC RDIMM │          │
│   └───────────┘      └───────────┘          │
│   ┌───────────┐      ┌───────────┐          │
│   │ DIMM A2   │      │ DIMM B2   │          │
│   │  (empty)  │      │  (empty)  │          │
│   └───────────┘      └───────────┘          │
│                                             │
│   Total: 64 GB in Dual-Channel Mode         │
│                                             │
└─────────────────────────────────────────────┘
```

## Network Interfaces

### 2.5 Gigabit Ethernet (RJ45)

| Port | Interface | MAC Address | Default Use | SR-IOV |
|------|-----------|-------------|-------------|--------|
| 1 | eno1 | 24:5e:be:7b:74:1b | Management | No |
| 2 | eno2 | 24:5e:be:7b:74:1c | WAN/LAN | No |
| 3 | eno3 | 24:5e:be:7b:74:1d | LAN | No |
| 4 | eno4 | 24:5e:be:7b:74:1e | LAN | No |
| 5 | eno5 | 24:5e:be:7b:74:1f | L2 Gateway | No |
| 6 | eno6 | 24:5e:be:7b:74:20 | L2 Gateway | No |
| 7 | eno7 | 24:5e:be:7b:74:21 | L2 Gateway | No |
| 8 | eno8 | 24:5e:be:7b:74:22 | L2 Gateway | No |

**Controller:** Intel I225-LM (2.5GbE)

### 10 Gigabit Ethernet (SFP+)

| Port | Interface | SR-IOV VFs | Default Use |
|------|-----------|------------|-------------|
| 9 | eno9 | 32 | High-speed L2GW / SR-IOV |
| 10 | eno10 | 32 | High-speed L2GW / SR-IOV |
| 11 | eno11 | 32 | High-speed L2GW / SR-IOV |
| 12 | eno12 | 32 | High-speed L2GW / SR-IOV |

**Controller:** Intel X710 (10GbE SFP+)

### SR-IOV Configuration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SR-IOV Virtual Functions                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   eno9 (10GbE SFP+)          eno10 (10GbE SFP+)                             │
│   ┌─────────────────┐        ┌─────────────────┐                            │
│   │ PF: eno9        │        │ PF: eno10       │                            │
│   │ VF0  VF1  VF2   │        │ VF0  VF1  VF2   │                            │
│   │ VF3  VF4  VF5   │        │ VF3  VF4  VF5   │                            │
│   │ ...            │        │ ...            │                            │
│   │ VF30 VF31      │        │ VF30 VF31      │                            │
│   │ (32 VFs total)  │        │ (32 VFs total)  │                            │
│   └─────────────────┘        └─────────────────┘                            │
│                                                                              │
│   eno11 (10GbE SFP+)         eno12 (10GbE SFP+)                             │
│   ┌─────────────────┐        ┌─────────────────┐                            │
│   │ PF: eno11       │        │ PF: eno12       │                            │
│   │ VF0  VF1  VF2   │        │ VF0  VF1  VF2   │                            │
│   │ VF3  VF4  VF5   │        │ VF3  VF4  VF5   │                            │
│   │ ...            │        │ ...            │                            │
│   │ VF30 VF31      │        │ VF30 VF31      │                            │
│   │ (32 VFs total)  │        │ (32 VFs total)  │                            │
│   └─────────────────┘        └─────────────────┘                            │
│                                                                              │
│   Total Available VFs: 128 (for VM/Container direct NIC passthrough)        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Storage

### Internal Storage

| Device | Type | Capacity | Interface | Use |
|--------|------|----------|-----------|-----|
| nvme0n1 | NVMe SSD | 256 GB | PCIe 3.0 x4 | System / OS |
| nvme1n1 | NVMe SSD | 256 GB | PCIe 3.0 x4 | System / OS (RAID1) |
| sda | SATA SSD | 2 TB | SATA III | Data volume |
| sdb | SATA SSD | 2 TB | SATA III | Data volume (RAID1) |

### Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Storage Configuration                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   System Volume (RAID1)                Data Volume (RAID1)                  │
│   ┌───────────────────────┐           ┌───────────────────────┐             │
│   │   nvme0n1 + nvme1n1   │           │    sda + sdb          │             │
│   │   256GB + 256GB       │           │    2TB + 2TB          │             │
│   │   ═══════════════     │           │    ═══════════════    │             │
│   │   MD RAID1 (md127)    │           │    MD RAID1 (md126)   │             │
│   │                       │           │                       │             │
│   │   ┌─────────────┐     │           │   ┌─────────────┐     │             │
│   │   │    /        │     │           │   │ /mnt/data   │     │             │
│   │   │   (root)    │     │           │   │  volume     │     │             │
│   │   └─────────────┘     │           │   └─────────────┘     │             │
│   │   ┌─────────────┐     │           │                       │             │
│   │   │   /boot     │     │           │   LVM on top for      │             │
│   │   │   (EFI)     │     │           │   VM/Container storage│             │
│   │   └─────────────┘     │           │                       │             │
│   └───────────────────────┘           └───────────────────────┘             │
│                                                                              │
│   DRBD: Configured for HA replication to peer QuCPE (optional)              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Form Factor

| Property | Value |
|----------|-------|
| **Form Factor** | 1U Rackmount |
| **Dimensions** | 430 x 350 x 44 mm (W x D x H) |
| **Weight** | ~8 kg |
| **Cooling** | Redundant fans |
| **Power Supply** | Dual redundant (optional) |
| **Power Consumption** | 150-250W typical |

## Environmental

| Property | Value |
|----------|-------|
| **Operating Temp** | 0°C to 40°C |
| **Storage Temp** | -20°C to 60°C |
| **Humidity** | 5% to 95% non-condensing |
| **Altitude** | Up to 3,000m |

## Comparison with Typical L2 Gateway Solutions

| Feature | QuCPE-7012 | Generic x86 Server | White-box Switch |
|---------|------------|-------------------|------------------|
| **OVN Pre-installed** | Yes | No | No |
| **SR-IOV** | 128 VFs | Varies | Limited |
| **DPDK** | Ready | Requires setup | Limited |
| **Form Factor** | 1U optimized | 1U-4U | 1U |
| **Power Efficiency** | High (Xeon D) | Lower | Varies |
| **Management** | Web UI + CLI | CLI | CLI |
| **Price Point** | Mid-range | Varies | Low-Mid |
| **Carrier Grade** | Yes | No | Varies |

## Performance Benchmarks (Expected)

> Note: These are estimated figures based on hardware capabilities. Actual performance depends on configuration.

| Metric | Value | Notes |
|--------|-------|-------|
| **L2 Forwarding (OVS)** | ~10 Mpps | Software path |
| **L2 Forwarding (DPDK)** | ~40 Mpps | With DPDK acceleration |
| **SR-IOV Passthrough** | Line rate | Near-zero latency |
| **Geneve Encap/Decap** | ~5 Gbps | Per tunnel |
| **Concurrent Flows** | 1M+ | OVS flow table |

## Why This Hardware Matters for OpenStack

1. **Native OVN Compatibility**: Same software stack as OpenStack Neutron ML2/OVN
2. **High Port Density**: 12 ports in 1U for connecting many physical servers
3. **SR-IOV for VMs**: 128 VFs enable high-performance VM networking
4. **DPDK Ready**: Can accelerate VNF workloads when used as edge compute
5. **ECC Memory**: Required for reliable compute workloads
6. **Carrier-Grade**: Designed for always-on network infrastructure
