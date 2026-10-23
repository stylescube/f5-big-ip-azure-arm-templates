# F5 BIG-IP 2-VM HA with Azure Cross-Region (Global) Load Balancer ARM Template

This document explains the Azure Resource Manager (ARM) template for deploying an **Azure Cross-Region Load Balancer (`sku.tier: "Global"`)** with a **Global Anycast Public IP** fronting a regional **HA pair of 3-NIC F5 BIG-IP Virtual Machines** in a Load Balancer Sandwich topology.

## Files

- **ARM Template**: [`azuredeploy-ha-global-lb.json`](./azuredeploy-ha-global-lb.json)
- **Parameters File**: [`azuredeploy-ha-global-lb.parameters.json`](./azuredeploy-ha-global-lb.parameters.json)

---

## Architecture Overview

In Azure, a **Cross-Region Load Balancer (`sku.tier: "Global"`)** cannot attach directly to VM NICs; instead, its backend pool references the frontend IP configuration of one or more **Regional Standard Public Load Balancers (`sku.tier: "Regional"`)**.

```
                                [ Global Internet Clients ]
                                             |
                                             v
                         +---------------------------------------+
                         |    Global Anycast Public IP (PIP)     |
                         |         (sku.tier = "Global")         |
                         +-------------------+-------------------+
                                             |
                         +-------------------+-------------------+
                         |   Azure Cross-Region Load Balancer    |
                         |         (sku.tier = "Global")         |
                         |      Floating IP: Ports 80 & 443      |
                         +-------------------+-------------------+
                                             |
                       (Backend Pool references Regional ALB Frontend)
                                             |
                         +-------------------+-------------------+
                         |   Regional External Standard ALB      |
                         |        (sku.tier = "Regional")        |
                         |      Health Probe: TCP Port 40000     |
                         +-------------------+-------------------+
                                             |
                    +------------------------+------------------------+  external-subnet (10.0.1.0/24)
                    |                                                 |  + eth1-ext-nsg
              [ eth1: 1.1 ]                                     [ eth1: 1.1 ]
            +---------------+                                 +---------------+
  eth0  <-- |  BIG-IP VM 1  |     <=== HA ConfigSync /        |  BIG-IP VM 2  | --> eth0
 (Mgmt)     | (AvSet FD/UD) |          Heartbeat ===>         | (AvSet FD/UD) |    (Mgmt)
            +---------------+                                 +---------------+
              [ eth2: 1.2 ]                                     [ eth2: 1.2 ]
                    |                                                 |  internal-subnet (10.0.2.0/24)
                    +------------------------+------------------------+  + eth2-int-nsg
                                             |
                         +-------------------+-------------------+
                         |   Regional Internal Standard ALB      |
                         |       (Static IP: 10.0.2.100)         |
                         |     HA Ports (All Protocols/0)        |
                         +-------------------+-------------------+
                                             |
                                             v
                                [ Internal App Workloads ]
```

### Interface Mapping (Per BIG-IP VM)

| Linux / Azure NIC | BIG-IP Interface | Azure Resource Name | Subnet | NSG | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`eth0` (Primary)** | `mgmt` | `<prefix>-vm<1\|2>-eth0-mgmt-nic` | `mgmt-subnet` (`10.0.0.0/24`) | `<prefix>-eth0-mgmt-nsg` | Dedicated management plane with per-VM Public IP (`<prefix>-vm<1\|2>-eth0-mgmt-pip`). Ports `22` & `443`. |
| **`eth1` (Secondary)** | `1.1` | `<prefix>-vm<1\|2>-eth1-ext-nic` | `external-subnet` (`10.0.1.0/24`) | `<prefix>-eth1-ext-nsg` | Public traffic plane attached to the Regional External ALB backend pool (fronted by the Global ALB). IP Forwarding enabled. |
| **`eth2` (Tertiary)** | `1.2` | `<prefix>-vm<1\|2>-eth2-int-nic` | `internal-subnet` (`10.0.2.0/24`) | `<prefix>-eth2-int-nsg` | Internal traffic & HA sync plane attached to the Regional Internal ALB backend pool. IP Forwarding enabled. |

---

## Key Architectural Details

1. **Global Anycast Frontend (`<prefix>-global-alb` & `<prefix>-global-pip`)**:
   - Deployed with `sku.name: "Standard"` and `sku.tier: "Global"` in a supported home region (`globalLbLocation`, default `eastus`).
   - Can have additional Regional External Load Balancers from other Azure regions added to its `GlobalRegionalAlbsPool` backend pool for active/active or active/passive multi-region global failover.
2. **Packet Destination IP (Floating IP / DSR)**:
   - Both the Global Load Balancer and the Regional External Load Balancer have `enableFloatingIP: true`.
   - When client traffic arrives at the **Global Anycast Public IP**, the destination IP in the TCP packet delivered to the BIG-IP `eth1` interface is the **Global Anycast Public IP** (or the Regional Public IP if accessed directly via the regional PIP).
   - Therefore, configure your BIG-IP Virtual Server destination IP on `eth1` to match the **`globalAnycastPublicIp`** output (or `0.0.0.0/0`).
3. **Health Probing Hierarchy**:
   - Azure Cross-Region (Global) Load Balancers do not define separate health probes; they automatically track the health of each Regional Load Balancer based on the Regional Load Balancer's health probe (`healthProbePort`, default TCP `40000`).

---

## Parameters

| Parameter | Type | Default Value | Description |
| :--- | :--- | :--- | :--- |
| `deploymentPrefix` | `string` | `bigip-glb` | Naming prefix for all deployed resources. |
| `location` | `string` | `[resourceGroup().location]` | Regional Azure location for the BIG-IP VMs, VNet, NSGs, and Regional Load Balancers. |
| `globalLbLocation` | `string` | `eastus` | Supported home region for the Global Tier Load Balancer and Global Public IP (`eastus`, `eastus2`, `westus`, `westus2`, `centralus`, `northeurope`, `westeurope`, etc.). |
| `adminUsername` | `string` | `azureuser` | Administrator username for both BIG-IP VMs. |
| `authenticationType` | `string` | `password` | `password` or `sshPublicKey`. |
| `adminPasswordOrKey` | `securestring` | *(Required)* | Administrator password or SSH public key. |
| `vmSize` | `string` | `Standard_DS4_v2` | VM size (must support at least 3 NICs). |
| `imageOffer` | `string` | `f5-big-ip-byol` | F5 Marketplace offer. |
| `imageSku` | `string` | `f5-big-all-2slot-byol` | F5 Marketplace SKU. |
| `bigipVersion` | `string` | `latest` | F5 BIG-IP image version. |
| `internalLbPrivateIp` | `string` | `10.0.2.100` | Static Private IP for the Regional Internal Load Balancer frontend. |
| `healthProbePort` | `int` | `40000` | TCP port probed by the Regional Load Balancers. |
| `restrictedSrcAddress` | `string` | `*` | Source IP/CIDR allowed to access Management ports (`22`, `443`) on `eth0`. |

---

## Deployment Instructions

```bash
# 1. Create a Resource Group
az group create \
  --name bigip-global-lb-rg \
  --location eastus

# 2. Deploy the Global Load Balancer + HA BIG-IP template
az deployment group create \
  --resource-group bigip-global-lb-rg \
  --template-file azuredeploy-ha-global-lb.json \
  --parameters @azuredeploy-ha-global-lb.parameters.json \
  --parameters restrictedSrcAddress="<YOUR_PUBLIC_IP>/32"
```

---

## Template Outputs

| Output Name | Description |
| :--- | :--- |
| `globalAnycastPublicIp` | Global Anycast Public IP on the Cross-Region Load Balancer (use this as your BIG-IP Virtual Server VIP). |
| `globalAnycastFqdn` | Global FQDN for client traffic. |
| `regionalExternalAlbPublicIp` | Regional External Load Balancer Public IP. |
| `regionalInternalAlbPrivateIp` | Regional Internal Load Balancer static private IP (`10.0.2.100`). |
| `bigip1MgmtGuiUrl` | HTTPS Management URL for BIG-IP VM 1 (`eth0`). |
| `bigip2MgmtGuiUrl` | HTTPS Management URL for BIG-IP VM 2 (`eth0`). |
