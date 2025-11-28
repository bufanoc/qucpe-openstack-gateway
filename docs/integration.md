# Integration Guide

## Connecting QuCPE-7012 to OpenStack as L2 Gateway

This guide walks through connecting a QNAP QuCPE-7012 to an OpenStack cloud using OVN ML2 mechanism driver.

## Prerequisites

### OpenStack Requirements

- OpenStack deployment using **Neutron with ML2/OVN** mechanism driver
- OVN Southbound database accessible from QuCPE (TCP port 6642)
- Network connectivity between QuCPE and OpenStack infrastructure
- Administrative access to OpenStack

### QuCPE Requirements

- QNAP QuCPE-7012 running QNE 1.0.6 or later
- SSH/console access with root privileges
- Network connectivity to OpenStack (can be through VPN, direct link, or internet)

### Network Requirements

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| QuCPE | OpenStack OVN SB | 6642 | TCP | OVSDB connection |
| QuCPE | OpenStack compute nodes | 6081 | UDP | Geneve tunnels |
| OpenStack | QuCPE | 6081 | UDP | Geneve tunnels |

## Step 1: Backup Current Configuration

**Critical:** Always backup before making changes.

```bash
# Create backup directory
mkdir -p /root/qucpe-backup-$(date +%Y%m%d)
cd /root/qucpe-backup-$(date +%Y%m%d)

# Backup OVS configuration
ovs-vsctl get open_vswitch . external_ids > ovs-external-ids.txt

# Backup OVN databases
cp /etc/config/ovnnb_db.db ./ovnnb_db.db.backup
cp /var/lib/ovn/ovnsb_db.db ./ovnsb_db.db.backup

# Backup service states
systemctl list-units --type=service | grep -E "ovn|ovs|vnd" > services-state.txt

# Save current OVN topology
ovn-nbctl show > ovn-nb-topology.txt
ovn-sbctl show > ovn-sb-topology.txt
ovs-vsctl show > ovs-topology.txt

echo "Backup completed in $(pwd)"
```

## Step 2: Gather OpenStack Information

From your OpenStack deployment, collect:

```bash
# On OpenStack network/controller node:

# Get OVN Southbound DB connection info
ovn-sbctl get-connection
# Example output: ptcp:6642:0.0.0.0

# Get the IP address of the OVN SB server
# This is typically the network node or a VIP
hostname -I | awk '{print $1}'

# Verify SB database is accessible
ovn-sbctl show
```

**Required Information:**

| Parameter | Example Value | Your Value |
|-----------|---------------|------------|
| OVN SB IP | 10.0.0.10 | __________ |
| OVN SB Port | 6642 | __________ |
| QuCPE Tunnel IP | 192.168.1.100 | __________ |

## Step 3: Test Connectivity

From the QuCPE, verify you can reach the OpenStack OVN SB:

```bash
# Test TCP connectivity to OVN SB
nc -zv <OPENSTACK_OVN_SB_IP> 6642

# Test OVSDB protocol (should show some output)
ovsdb-client list-dbs tcp:<OPENSTACK_OVN_SB_IP>:6642

# Expected output:
# OVN_Southbound
```

If connectivity fails, check:
- Firewall rules on OpenStack side
- Network routing between QuCPE and OpenStack
- VPN tunnel status (if applicable)

## Step 4: Stop Local OVN Central Services

The QuCPE will no longer need its local OVN northbound/southbound databases:

```bash
# Stop local OVN central services
systemctl stop ovn-northd
systemctl stop ovn-ovsdb-server-sb
# Note: Keep ovn-ovsdb-server-nb running if you want to preserve local config

# Optionally stop QNAP's vnd (Service Composer won't work anyway)
systemctl stop vnd

# Verify services stopped
systemctl status ovn-northd ovn-ovsdb-server-sb vnd
```

## Step 5: Configure OVS to Connect to OpenStack

This is the key step - point ovn-controller to OpenStack's OVN SB:

```bash
# Set the remote OVN Southbound database
ovs-vsctl set open . external_ids:ovn-remote="tcp:<OPENSTACK_OVN_SB_IP>:6642"

# Set the tunnel endpoint IP (QuCPE's IP reachable from OpenStack)
ovs-vsctl set open . external_ids:ovn-encap-ip="<QUCPE_TUNNEL_IP>"

# Verify the configuration
ovs-vsctl get open_vswitch . external_ids
```

**Example:**

```bash
ovs-vsctl set open . external_ids:ovn-remote="tcp:10.0.0.10:6642"
ovs-vsctl set open . external_ids:ovn-encap-ip="192.168.1.100"
```

## Step 6: Restart ovn-controller

```bash
# Restart ovn-controller to connect to remote OVN SB
systemctl restart ovn-controller

# Check status
systemctl status ovn-controller

# Check logs for connection status
journalctl -u ovn-controller -n 50
```

## Step 7: Verify Chassis Registration

On **OpenStack** (network node), verify the QuCPE appears as a chassis:

```bash
# List all chassis
ovn-sbctl show

# You should see the QuCPE:
# Chassis <your-system-uuid>
#     hostname: GATEPROD01
#     Encap geneve
#         ip: "192.168.1.100"  # QuCPE's tunnel IP
#         options: {csum="true"}
```

On **QuCPE**, verify the connection:

```bash
# Check ovn-controller is connected
ovs-appctl -t ovn-controller connection-status
# Expected: connected

# List learned chassis (should include OpenStack nodes)
ovn-sbctl list Chassis
```

## Step 8: Create Provider Network Mapping

To map an OpenStack provider/external network to a QuCPE physical port:

### On OpenStack:

```bash
# Create a provider network that will be bridged to QuCPE
openstack network create \
  --provider-network-type flat \
  --provider-physical-network physnet-qucpe \
  --external \
  qucpe-external

# Create subnet
openstack subnet create \
  --network qucpe-external \
  --subnet-range 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  --allocation-pool start=192.168.100.100,end=192.168.100.200 \
  qucpe-external-subnet
```

### On QuCPE:

```bash
# Create OVS bridge for the physical network
ovs-vsctl add-br br-qucpe

# Add physical port to the bridge
ovs-vsctl add-port br-qucpe eno5

# Configure bridge mapping
ovs-vsctl set open . external_ids:ovn-bridge-mappings="physnet-qucpe:br-qucpe"

# Restart ovn-controller to pick up the mapping
systemctl restart ovn-controller
```

## Step 9: Test L2 Gateway Functionality

### Create Test Network on OpenStack:

```bash
# Create tenant network
openstack network create test-tenant-net

# Create subnet
openstack subnet create \
  --network test-tenant-net \
  --subnet-range 10.100.0.0/24 \
  test-tenant-subnet

# Boot a test VM
openstack server create \
  --flavor m1.small \
  --image cirros \
  --network test-tenant-net \
  test-vm
```

### Configure L2 Gateway (using Neutron L2GW or manual OVN):

```bash
# Option A: Using Neutron L2 Gateway service (if installed)
# Create L2 gateway
openstack l2-gateway create qucpe-gw --device name=GATEPROD01,interface_names=eno5

# Create L2 gateway connection
openstack l2-gateway-connection create \
  --l2-gateway qucpe-gw \
  --network test-tenant-net \
  qucpe-connection

# Option B: Manual OVN configuration
# On OpenStack OVN NB:
ovn-nbctl lsp-add <switch-uuid> qucpe-l2gw-port
ovn-nbctl lsp-set-type qucpe-l2gw-port l2gateway
ovn-nbctl lsp-set-options qucpe-l2gw-port \
  network_name=physnet-qucpe \
  l2gateway-chassis=<your-system-uuid>
```

### Verify Connectivity:

```bash
# On a physical server connected to QuCPE eno5:
# Configure IP in test-tenant-net subnet
ip addr add 10.100.0.50/24 dev eth0

# Ping the OpenStack VM
ping 10.100.0.x  # VM's IP

# From OpenStack VM, ping physical server
ping 10.100.0.50
```

## Rollback Procedure

To disconnect from OpenStack and restore standalone mode:

```bash
# 1. Restore local OVN SB connection
ovs-vsctl set open . external_ids:ovn-remote="unix:/var/run/ovn/ovnsb_db.sock"
ovs-vsctl set open . external_ids:ovn-encap-ip="127.0.0.1"

# 2. Restart local OVN central services
systemctl start ovn-ovsdb-server-sb
systemctl start ovn-northd

# 3. Restart ovn-controller
systemctl restart ovn-controller

# 4. Restart QNAP vnd for Service Composer
systemctl start vnd

# 5. Verify local operation
ovn-sbctl show
# Should show local chassis at 127.0.0.1
```

## Troubleshooting

### ovn-controller won't connect

```bash
# Check logs
journalctl -u ovn-controller -f

# Common issues:
# - Firewall blocking port 6642
# - Wrong IP address
# - SSL required but not configured
```

### Chassis not appearing in OpenStack

```bash
# On QuCPE, verify external_ids
ovs-vsctl get open_vswitch . external_ids

# Check ovn-controller is running
systemctl status ovn-controller

# Verify network connectivity
telnet <OPENSTACK_OVN_SB_IP> 6642
```

### Geneve tunnels not forming

```bash
# Check tunnel interface
ip link show type geneve

# Verify encap IP is routable
ping <openstack-compute-ip>

# Check OVS flows
ovs-ofctl dump-flows br-int | grep tun
```

### L2 traffic not flowing

```bash
# Verify port binding
ovn-sbctl list Port_Binding | grep qucpe

# Check physical interface is up
ip link show eno5

# Verify bridge mapping
ovs-vsctl get open . external_ids:ovn-bridge-mappings

# Check for flow rules
ovs-ofctl dump-flows br-qucpe
```

## Security Considerations

### SSL/TLS for OVN Connection

For production, enable SSL:

```bash
# Generate certificates (or use existing PKI)
# On QuCPE:
ovs-vsctl set open . external_ids:ovn-remote="ssl:<IP>:6642"
ovs-vsctl set-ssl /path/to/privkey.pem /path/to/cert.pem /path/to/ca-cert.pem
```

### Firewall Rules

Minimal required rules:

```bash
# On QuCPE (outbound)
iptables -A OUTPUT -p tcp --dport 6642 -j ACCEPT  # OVN SB
iptables -A OUTPUT -p udp --dport 6081 -j ACCEPT  # Geneve

# On QuCPE (inbound)
iptables -A INPUT -p udp --dport 6081 -j ACCEPT   # Geneve from OpenStack
```

## Next Steps

- [Roadmap](roadmap.md) - Future enhancements including edge compute
- [Troubleshooting](troubleshooting.md) - Detailed troubleshooting guide
- [Architecture](architecture.md) - Deep dive into OVN architecture
