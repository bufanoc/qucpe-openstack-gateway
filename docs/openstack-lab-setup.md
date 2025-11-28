# OpenStack Test Environment for QuCPE L2 Gateway Project

## Overview

Deploy a **Kolla-Ansible OpenStack Caracal (2024.1)** two-node environment (controller + compute) on vSphere VMs to serve as a development lab for testing the QuCPE-7012 L2 Gateway integration through all project phases.

## Test Strategy

| Component | Role | Purpose |
|-----------|------|---------|
| OpenStack (2 nodes) | Controller + Compute | Real Geneve overlay traffic between nodes |
| Real QuCPE-7012 | **Edge Node** | Registered OpenStack node for L2GW + future VNF compute |
| Virtual QuCPE (ESXi) | Canary/Sandbox | Test risky commands before running on real hardware |

**Virtual QuCPE Use Case:** Run aggressive/destructive commands first on the clone to detect QNAP booby traps (watchdogs, license checks, hidden dependencies) before applying to production hardware.

## QuCPE as OpenStack Edge Node

The QuCPE will be registered as a **full OpenStack node** from day one, not just a standalone OVN chassis. This provides:

- Visibility in `openstack hypervisor list`
- Centralized monitoring and management
- Clean upgrade path to VNF compute
- Proper integration with Neutron/Nova

### Phased Capability Enablement

| Phase | Capability | QuCPE Components |
|-------|------------|------------------|
| 1 | L2 Gateway | nova-compute (disabled), ovn-controller, neutron-ovn-agent |
| 1.5 | Edge Compute (VNF) | nova-compute (enabled), SR-IOV agent, host aggregate |
| 2 | Full Edge | DPDK optimization, SFC, hardware offload |

### Host Aggregate Strategy

```bash
# Create edge aggregate (Phase 1.5+)
openstack aggregate create --zone edge-vnf edge-nodes
openstack aggregate add host edge-nodes qucpe-7012

# Create VNF-specific flavor with SR-IOV
openstack flavor create --ram 2048 --disk 10 --vcpus 2 vnf.small
openstack flavor set vnf.small --property aggregate_instance_extra_specs:edge=true
openstack flavor set vnf.small --property hw:cpu_policy=dedicated
openstack flavor set vnf.small --property hw:mem_page_size=large
```

This ensures regular workloads never accidentally land on the QuCPE - only VNF instances with the right flavor.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            vSphere Cluster                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐   │
│  │ OpenStack Controller │  │ OpenStack Compute    │  │ QuCPE Clone VM   │   │
│  │ ──────────────────── │  │ ──────────────────── │  │ ──────────────── │   │
│  │ Ubuntu 22.04 LTS     │  │ Ubuntu 22.04 LTS     │  │ QNAP QNE 1.0.6   │   │
│  │ 16 vCPU / 32GB RAM   │  │ 8 vCPU / 16GB RAM    │  │ (Canary only)    │   │
│  │                      │  │                      │  │                  │   │
│  │ - Keystone, Nova API │  │ - nova-compute       │  │ Test risky cmds  │   │
│  │ - Neutron (ML2/OVN)  │◄─┼─► ovn-controller     │  │ before applying  │   │
│  │ - Glance, Horizon    │  │ - libvirt/QEMU       │  │ to real hardware │   │
│  │ - OVN-NB/SB/northd   │  │                      │  │                  │   │
│  │                      │  │                      │  │                  │   │
│  └──────────┬───────────┘  └──────────┬───────────┘  └──────────────────┘   │
│             │                         │                                      │
│             │      Geneve Overlay     │                                      │
│             └────────────┬────────────┘                                      │
│                          │                                                   │
└──────────────────────────┼───────────────────────────────────────────────────┘
                           │
                    WAN / Site Link
                           │
              ┌────────────┴────────────┐
              │                         │
              │   Real QuCPE-7012       │
              │   ═══════════════       │
              │   OpenStack EDGE NODE   │  ◄── Registered in OpenStack
              │                         │
              │   - nova-compute        │  (disabled Phase 1, enabled Phase 1.5)
              │   - ovn-controller      │  L2GW functionality
              │   - neutron-ovn-agent   │  Proper Neutron integration
              │   - sr-iov-agent        │  (Phase 1.5+ for VNF)
              │                         │
              │   br-int                │
              │   L2GW ports (eno5-9)   │
              │   SR-IOV VFs (128)      │  (Phase 1.5+ for VNF passthrough)
              │                         │
              └────────────┬────────────┘
                           │
                    Physical Servers
                    (OpenStack tenant network)
```

## VM Specifications

### OpenStack Controller VM

| Resource | Value |
|----------|-------|
| Guest OS | Ubuntu 22.04.4 LTS |
| vCPU | 16 |
| RAM | 32 GB |
| Disk 1 | 200 GB (thin) - OS + Docker + Glance images |
| NIC 1 | Management (API access, SSH) |
| NIC 2 | Tunnel network (Geneve overlay to compute) |
| Nested Virt | Not required (no VMs run here) |

### OpenStack Compute VM

| Resource | Value |
|----------|-------|
| Guest OS | Ubuntu 22.04.4 LTS |
| vCPU | 8 |
| RAM | 16 GB |
| Disk 1 | 100 GB (thin) - OS + Docker + instance ephemeral |
| NIC 1 | Management |
| NIC 2 | Tunnel network (Geneve overlay) |
| Nested Virt | **Enabled** (for Nova instances) |

### Network Configuration

| Network | Purpose | VLAN/Portgroup | IP Range |
|---------|---------|----------------|----------|
| Management | API access, SSH | Existing LAN | DHCP or static |
| Tunnel | Geneve overlay | Dedicated portgroup | 10.0.0.0/24 (example) |
| Provider (optional) | External/floating IPs | Trunk or dedicated | As needed |

**Critical vSwitch Setting:** Portgroup must have **Promiscuous Mode = Accept** for OVN Geneve tunnels to work in nested virtualization.

## Deployment Steps

### Step 1: Create VMs in vCenter

1. Create new VMs with specs above
2. Enable "Expose hardware assisted virtualization to guest OS" on compute node
3. Attach to appropriate portgroups
4. Install Ubuntu 22.04 LTS server (minimal)

### Step 2: Prepare Ubuntu Hosts (Both Nodes)

```bash
# Update system
apt update && apt upgrade -y

# Install prerequisites
apt install -y python3-dev python3-pip python3-venv git gcc libffi-dev libssl-dev

# Install Docker
curl -fsSL https://get.docker.com | bash
systemctl enable docker
usermod -aG docker $USER

# Configure Docker for Kolla
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
systemctl restart docker
```

### Step 3: Install Kolla-Ansible (Controller Only)

```bash
# Create virtual environment
python3 -m venv /opt/kolla-venv
source /opt/kolla-venv/bin/activate

# Install Kolla-Ansible for Caracal
pip install -U pip
pip install 'ansible-core>=2.14,<2.16'
pip install kolla-ansible==18.0.0  # Caracal release

# Create Kolla directories
mkdir -p /etc/kolla
cp -r /opt/kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
cp /opt/kolla-venv/share/kolla-ansible/ansible/inventory/multinode /etc/kolla/

# Generate passwords
kolla-genpwd
```

### Step 4: Configure Multinode Inventory

Edit `/etc/kolla/multinode`:

```ini
[control]
controller ansible_host=<controller-ip> ansible_user=root

[network]
controller

[compute]
compute01 ansible_host=<compute-ip> ansible_user=root

[monitoring]
controller

[storage]
# Empty - no Cinder

[deployment]
localhost ansible_connection=local
```

**Note:** Set up SSH key auth from controller to compute node:
```bash
ssh-keygen -t ed25519
ssh-copy-id root@<compute-ip>
```

### Step 5: Configure Kolla for OVN

Edit `/etc/kolla/globals.yml`:

```yaml
---
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "2024.1"

# Network interfaces
network_interface: "ens192"           # Management NIC
neutron_external_interface: "ens224"  # Tunnel/provider NIC
kolla_internal_vip_address: "<controller-ip>"

# Enable OVN (critical for L2GW project)
neutron_plugin_agent: "ovn"
neutron_ovn_distributed_fip: "yes"

# Expose OVN SB for remote chassis (QuCPE)
ovn_sb_connection: "ptcp:6642:0.0.0.0"

# Enable services
enable_neutron: "yes"
enable_nova: "yes"
enable_horizon: "yes"
enable_glance: "yes"
enable_cinder: "no"  # Optional, enable if testing block storage

# Nested virt support
nova_compute_virt_type: "kvm"  # or "qemu" if nested virt not working
```

### Step 6: Deploy OpenStack

```bash
source /opt/kolla-venv/bin/activate

# Bootstrap servers (both controller and compute)
kolla-ansible -i /etc/kolla/multinode bootstrap-servers

# Pre-deployment checks
kolla-ansible -i /etc/kolla/multinode prechecks

# Deploy
kolla-ansible -i /etc/kolla/multinode deploy

# Post-deploy (creates admin-openrc.sh)
kolla-ansible -i /etc/kolla/multinode post-deploy
```

### Step 7: Verify OVN SB is Accessible

```bash
# On OpenStack controller
docker exec ovn_sb_db ovn-sbctl show

# Verify port is listening
ss -tlnp | grep 6642

# From QuCPE (test connectivity)
nc -zv <openstack-ip> 6642
ovsdb-client list-dbs tcp:<openstack-ip>:6642
```

### Step 8: Create Test Resources

```bash
# Source credentials
source /etc/kolla/admin-openrc.sh

# Download test image
wget https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
openstack image create --disk-format qcow2 --container-format bare \
  --public --file cirros-0.6.2-x86_64-disk.img cirros

# Create flavor
openstack flavor create --ram 512 --disk 1 --vcpus 1 m1.tiny

# Create test network (this will be bridged to QuCPE)
openstack network create test-tenant-net
openstack subnet create --network test-tenant-net \
  --subnet-range 10.100.0.0/24 \
  --gateway 10.100.0.1 \
  test-tenant-subnet
```

## Connecting QuCPE to OpenStack as Edge Node

### Phase 1: Register as OpenStack Node with L2GW

Unlike a simple OVN chassis, we'll install OpenStack agents on QuCPE so it's a proper managed node.

#### Step 1: Test on Virtual QuCPE First

Before touching real hardware, test on the ESXi clone:

```bash
# On virtual QuCPE - test if we can install nova-compute
apt update
apt install -y nova-compute neutron-ovn-agent

# Watch for QNAP booby traps:
# - Does vnd service interfere?
# - Do any watchdogs restart services?
# - Are there package conflicts?
```

#### Step 2: Configure QuCPE as Edge Node

```bash
# Point OVN to OpenStack
ovs-vsctl set open . external_ids:ovn-remote="tcp:<openstack-ip>:6642"
ovs-vsctl set open . external_ids:ovn-encap-ip="<qucpe-tunnel-ip>"

# Configure nova-compute to connect to controller
# Edit /etc/nova/nova.conf with RabbitMQ, Keystone endpoints

# Disable nova-compute scheduling initially (Phase 1)
openstack compute service set --disable qucpe-edge nova-compute
```

#### Step 3: Verify Registration

```bash
# On controller
openstack hypervisor list
# Should show: qucpe-edge

openstack network agent list
# Should show: ovn-controller on qucpe-edge

ovn-sbctl show
# Should show: qucpe-edge as chassis
```

### Phase 1.5: Enable VNF Compute

```bash
# Create edge aggregate
openstack aggregate create --zone edge-vnf edge-nodes
openstack aggregate add host edge-nodes qucpe-edge

# Enable compute service
openstack compute service set --enable qucpe-edge nova-compute

# Create VNF flavor (only schedules to edge aggregate)
openstack flavor create --ram 2048 --disk 10 --vcpus 2 vnf.small
openstack flavor set vnf.small --property aggregate_instance_extra_specs:zone=edge-vnf

# Test VNF deployment
openstack server create --flavor vnf.small --image cirros --network test-net test-vnf
```

## Future Expansion (HA Testing)

When ready for Phase 2 HA testing:

1. Add 2 more controller VMs for control plane HA (3 total)
2. Add second QuCPE as standby chassis
3. Update inventory with additional nodes
4. Re-run `kolla-ansible -i /etc/kolla/multinode deploy`
5. Test QuCPE chassis failover scenarios

## Troubleshooting

### OVN SB not accessible from QuCPE
- Check firewall: `ufw allow 6642/tcp`
- Verify Docker port mapping: `docker port ovn_sb_db`
- Check `ovn_sb_connection` in globals.yml

### Geneve tunnels not forming
- Ensure promiscuous mode on vSwitch portgroup
- Verify encap IPs are routable between hosts
- Check OVS flows: `docker exec openvswitch_vswitchd ovs-ofctl dump-flows br-int`

### Nova VMs won't start
- Enable nested virtualization in vSphere
- Or set `nova_compute_virt_type: "qemu"` in globals.yml

## References

- [Kolla-Ansible Docs](https://docs.openstack.org/kolla-ansible/2024.1/)
- [OVN Architecture](https://docs.openstack.org/neutron/2024.1/admin/ovn/)
- [Integration Guide](integration.md)
