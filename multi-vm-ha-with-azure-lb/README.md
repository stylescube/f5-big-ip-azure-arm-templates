# F5 BIG-IP 2-VM HA "Load Balancer Sandwich" Azure ARM Template

This document explains the Azure Resource Manager (ARM) template for deploying a **High Availability (HA) pair of 3-NIC F5 BIG-IP Virtual Machines** placed between an **External (Public) Azure Standard Load Balancer** and an **Internal (Private) Azure Standard Load Balancer** (commonly known as the **"Load Balancer Sandwich"** architecture).

## Files

- **ARM Template**: [`azuredeploy-ha-sandwich.json`](./azuredeploy-ha-sandwich.json)
- **Parameters File**: [`azuredeploy-ha-sandwich.parameters.json`](./azuredeploy-ha-sandwich.parameters.json)

---

## Architecture Overview

```
                                   [ Internet ]
                                        |
                                        v
                        +-------------------------------+
                        |    External Standard ALB      |
                        |  (Public IP, Ports 80 & 443)  |
                        |      Floating IP (DSR)        |
                        +---------------+---------------+
                                        |
                   +--------------------+--------------------+  external-subnet (10.0.1.0/24)
                   |                                         |  + eth1-ext-nsg
             [ eth1: 1.1 ]                             [ eth1: 1.1 ]
           +---------------+                         +---------------+
  eth0 <-- |  BIG-IP VM 1  |  <=== HA Sync / ===>    |  BIG-IP VM 2  | --> eth0
 (Mgmt)    | (AvSet FD/UD) |       Heartbeat         | (AvSet FD/UD) |    (Mgmt)
           +---------------+                         +---------------+
             [ eth2: 1.2 ]                             [ eth2: 1.2 ]
                   |                                         |  internal-subnet (10.0.2.0/24)
                   +--------------------+--------------------+  + eth2-int-nsg
                                        |
                        +---------------+---------------+
                        |    Internal Standard ALB      |
                        |   (Static IP: 10.0.2.100)     |
                        |  HA Ports (All Protocols/0)   |
                        +---------------+---------------+
                                        |
                                        v
                           [ Internal App Workloads ]
```

### Interface Mapping (Per BIG-IP VM)

| Linux / Azure NIC | BIG-IP Interface | Azure Resource Name | Subnet | NSG | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`eth0` (Primary)** | `mgmt` | `<prefix>-vm<1\|2>-eth0-mgmt-nic` | `mgmt-subnet` (`10.0.0.0/24`) | `<prefix>-eth0-mgmt-nsg` | Dedicated management plane with per-VM Public IP (`<prefix>-vm<1\|2>-eth0-mgmt-pip`). Ports `22` & `443`. |
| **`eth1` (Secondary)** | `1.1` | `<prefix>-vm<1\|2>-eth1-ext-nic` | `external-subnet` (`10.0.1.0/24`) | `<prefix>-eth1-ext-nsg` | Public traffic plane attached to the **External Azure Load Balancer** backend pool. IP Forwarding enabled. |
| **`eth2` (Tertiary)** | `1.2` | `<prefix>-vm<1\|2>-eth2-int-nic` | `internal-subnet` (`10.0.2.0/24`) | `<prefix>-eth2-int-nsg` | Internal traffic & HA sync plane attached to the **Internal Azure Load Balancer** backend pool. IP Forwarding enabled. |

---

## How the Load Balancer Sandwich Works

1. **Inbound North-South Traffic (External ALB -> `eth1`)**:
   - Internet clients connect to the Public IP on the **External Standard Load Balancer (`<prefix>-ext-alb`)**.
   - The External ALB probes both BIG-IP VMs on TCP port `40000` (configurable via `healthProbePort`).
   - Traffic on TCP `80` and `443` is forwarded to the healthy BIG-IP node(s) on `eth1` with **Floating IP (`enableFloatingIP: true`)** enabled, plus an explicit **Outbound Rule** (`External-Outbound-SNAT`) for deterministic egress SNAT.
2. **Outbound / East-West Traffic (Internal ALB -> `eth2`)**:
   - Internal application tiers can use a User Defined Route (UDR) pointing `0.0.0.0/0` (or internal CIDRs) to the static private IP (`10.0.2.100`) of the **Internal Standard Load Balancer (`<prefix>-int-alb`)**.
   - The Internal ALB uses an **HA Ports rule (`protocol: "All"`, `frontendPort: 0`, `backendPort: 0`, `enableFloatingIP: true`)** to distribute all TCP/UDP flows across the healthy BIG-IP `eth2` interfaces.
3. **High Availability & Fault Tolerance**:
   - Both BIG-IP VMs are placed in a Managed **Availability Set (`<prefix>-avset`)** with separate Fault Domains and Update Domains.
   - Active/Standby or Active/Active state is controlled directly by BIG-IP health listeners responding to the Azure Load Balancer health probes on `healthProbePort`.

---

## Resources Deployed

- **1x Availability Set**: `<prefix>-avset` (2 Fault Domains, 2 Update Domains)
- **3x Network Security Groups**:
  - `<prefix>-eth0-mgmt-nsg` (Inbound SSH `22`, HTTPS `443` from `restrictedSrcAddress`)
  - `<prefix>-eth1-ext-nsg` (Inbound HTTP `80`, HTTPS `443` from `Internet`, plus `AzureLoadBalancer` health probes)
  - `<prefix>-eth2-int-nsg` (Inbound HA CMI ConfigSync `TCP 4353, 443, 1026`, Failover Heartbeat `UDP 1026`, Internal VNet traffic, plus `AzureLoadBalancer` probes)
- **1x Virtual Network**: `<prefix>-vnet` with `mgmt-subnet`, `external-subnet`, and `internal-subnet`
- **3x Standard Public IPs**:
  - 1x Frontend Public IP for the External Load Balancer (`<prefix>-ext-alb-pip`)
  - 2x Management Public IPs for `eth0` on VM1 and VM2 (`<prefix>-vm1-eth0-mgmt-pip`, `<prefix>-vm2-eth0-mgmt-pip`)
- **2x Standard Azure Load Balancers**:
  - `<prefix>-ext-alb` (Public frontend, backend pool on `eth1`, TCP probe on `40000`, Floating IP rules for `80`/`443`, Outbound SNAT rule)
  - `<prefix>-int-alb` (Private static IP `10.0.2.100` frontend, backend pool on `eth2`, TCP probe on `40000`, HA Ports rule)
- **6x Network Interfaces** (3 per VM: `eth0-mgmt-nic`, `eth1-ext-nic`, `eth2-int-nic`)
- **2x F5 BIG-IP Virtual Machines** (`<prefix>-vm1`, `<prefix>-vm2`)

---

## Parameters

| Parameter | Type | Default Value | Description |
| :--- | :--- | :--- | :--- |
| `deploymentPrefix` | `string` | `bigip-ha` | Naming prefix for all deployed resources. |
| `location` | `string` | `[resourceGroup().location]` | Azure region for all resources. |
| `adminUsername` | `string` | `azureuser` | Administrator username for both BIG-IP VMs. |
| `authenticationType` | `string` | `password` | `password` or `sshPublicKey`. |
| `adminPasswordOrKey` | `securestring` | *(Required)* | Administrator password or SSH public key. |
| `vmSize` | `string` | `Standard_DS4_v2` | VM size (must support at least 3 NICs). |
| `imageOffer` | `string` | `f5-big-ip-byol` | F5 Marketplace offer. |
| `imageSku` | `string` | `f5-big-all-2slot-byol` | F5 Marketplace SKU. |
| `bigipVersion` | `string` | `latest` | F5 BIG-IP version. |
| `vnetAddressPrefix` | `string` | `10.0.0.0/16` | Virtual Network CIDR block. |
| `mgmtSubnetPrefix` | `string` | `10.0.0.0/24` | Management subnet prefix (`eth0`). |
| `externalSubnetPrefix` | `string` | `10.0.1.0/24` | External/public subnet prefix (`eth1`). |
| `internalSubnetPrefix` | `string` | `10.0.2.0/24` | Internal subnet prefix (`eth2`). |
| `internalLbPrivateIp` | `string` | `10.0.2.100` | Static Private IP for the Internal Load Balancer frontend. |
| `healthProbePort` | `int` | `40000` | TCP port probed by External and Internal Load Balancers. |
| `restrictedSrcAddress` | `string` | `*` | Source IP/CIDR allowed to access Management ports (`22`, `443`) on `eth0`. |

---

## Deployment Instructions

```bash
# 1. Create a Resource Group
az group create \
  --name bigip-ha-sandwich-rg \
  --location eastus

# 2. Deploy the HA Load Balancer Sandwich template
az deployment group create \
  --resource-group bigip-ha-sandwich-rg \
  --template-file azuredeploy-ha-sandwich.json \
  --parameters @azuredeploy-ha-sandwich.parameters.json \
  --parameters restrictedSrcAddress="<YOUR_PUBLIC_IP>/32"
```

---

## Post-Deployment BIG-IP Configuration Notes

1. **VLANs & Self-IPs**:
   - Create VLAN `external` on interface `1.1` (`eth1`) and assign each BIG-IP its Azure-allocated private IP.
   - Create VLAN `internal` on interface `1.2` (`eth2`) and assign each BIG-IP its Azure-allocated private IP.
2. **Device Service Clustering (DSC)**:
   - Configure ConfigSync and Network Failover between `vm1` and `vm2` over the `internal` (`1.2` / `eth2`) Self-IPs.
3. **Azure Load Balancer Health Probe Listener**:
   - Create a Virtual Server listening on TCP port `40000` (matching `healthProbePort`) bound to the Active Traffic Group (`traffic-group-1`). Because `traffic-group-1` is only active on the Active BIG-IP unit, Azure Load Balancers will automatically direct traffic only to the Active BIG-IP and fail over within seconds when `traffic-group-1` moves to the peer.
4. **Floating IP Virtual Servers**:
   - Because `enableFloatingIP: true` (Direct Server Return) is enabled on the External and Internal Load Balancers, configure your BIG-IP Virtual Servers on `external` (`eth1`) with the **External Load Balancer's Public IP** as the destination address.
