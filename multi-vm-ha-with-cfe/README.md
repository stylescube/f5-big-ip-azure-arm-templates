# F5 BIG-IP 2-VM HA with F5 Cloud Failover Extension (CFE) Azure ARM Template

This document explains the Azure Resource Manager (ARM) template for deploying a **High Availability (HA) Active/Standby pair of 3-NIC F5 BIG-IP Virtual Machines** using the **[F5 Cloud Failover Extension (CFE)](https://github.com/F5Networks/f5-cloud-failover-extension)** instead of Azure Load Balancers.

## Files

- **ARM Template**: [`azuredeploy-ha-cfe.json`](./azuredeploy-ha-cfe.json)
- **Parameters File**: [`azuredeploy-ha-cfe.parameters.json`](./azuredeploy-ha-cfe.parameters.json)

---

## Architecture Overview

Unlike the Load Balancer Sandwich design, **F5 Cloud Failover Extension (CFE)** eliminates the external and internal Azure Load Balancers. When a BIG-IP failover event occurs, CFE calls the Azure Resource Manager API (authenticated via a **User-Assigned Managed Identity**) to:
1. **Reassign Secondary VIP IP Configurations (`failoverAddresses`)**: Moves the secondary private IP (`10.0.1.100`) and its bound **Floating Public IP (`<prefix>-eth1-floating-vip-pip`)** from the standby BIG-IP's `eth1` NIC to the newly active BIG-IP's `eth1` NIC.
2. **Update User Defined Routes (`failoverRoutes`)**: Updates the `nextHopIpAddress` in the Azure Route Table (`<prefix>-cfe-rt`) from `10.0.2.4` (VM1 `eth2`) to `10.0.2.5` (VM2 `eth2`).

```
                                [ Internet Clients ]
                                         |
                                         v
                     +---------------------------------------+
                     |     Floating VIP Public IP (PIP)      |
                     |  Bound to Secondary IP: 10.0.1.100    |
                     +-------------------+-------------------+
                                         |
                    (CFE moves secondary ipconfig on failover)
                                         |
                   +---------------------+---------------------+  external-subnet (10.0.1.0/24)
                   | (Active Initially)                        |  + eth1-ext-nsg
             [ eth1: 1.1 ]                               [ eth1: 1.1 ]
             Self: 10.0.1.4                              Self: 10.0.1.5
             VIP:  10.0.1.100                            (Standby)
           +---------------+                           +---------------+
  eth0 <-- |  BIG-IP VM 1  |   <=== HA ConfigSync /    |  BIG-IP VM 2  | --> eth0
10.0.0.4   | (Managed ID)  |        Heartbeat ===>     | (Managed ID)  |   10.0.0.5
           +---------------+                           +---------------+
             [ eth2: 1.2 ]                               [ eth2: 1.2 ]
             Self: 10.0.2.4                              Self: 10.0.2.5
                   |                                           |  internal-subnet (10.0.2.0/24)
                   +---------------------+---------------------+  + eth2-int-nsg
                                         |
                     +-------------------+-------------------+
                     |     Azure Route Table (CFE UDR)       |
                     |   Prefix: 10.10.0.0/16 -> NextHop:    |
                     |   VirtualAppliance 10.0.2.4 / .5      |
                     +---------------------------------------+
```

### Interface Mapping (Per BIG-IP VM)

| Linux / Azure NIC | BIG-IP Interface | Static Private IPs (VM1 / VM2) | CFE Discovery Tags | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **`eth0` (Primary)** | `mgmt` | `10.0.0.4` / `10.0.0.5` | *None* | Management SSH (`22`) & TMUI GUI (`443`) with per-VM Public IP (`<prefix>-vm<1\|2>-eth0-mgmt-pip`). |
| **`eth1` (Secondary)** | `1.1` | Self: `10.0.1.4` / `10.0.1.5`<br>Floating VIP: `10.0.1.100` (on VM1) | `f5_cloud_failover_label`<br>`f5_cloud_failover_nic_map: external` | Public traffic plane. Per-VM Self Public IPs ensure uninterrupted outbound access to Azure APIs during failover; secondary VIP IP config floats via CFE. |
| **`eth2` (Tertiary)** | `1.2` | Self: `10.0.2.4` / `10.0.2.5` | `f5_cloud_failover_label`<br>`f5_cloud_failover_nic_map: internal` | Internal traffic & HA ConfigSync/Failover heartbeat plane. Target of CFE-managed Azure Route Table UDRs. |

---

## Resources Deployed for CFE

1. **User-Assigned Managed Identity & RBAC (`Microsoft.ManagedIdentity/userAssignedIdentities`)**:
   - Creates `<prefix>-cfe-identity`, attaches it to both BIG-IP VMs, and assigns **Contributor** and **Storage Blob Data Contributor** roles scoped to the Resource Group.
2. **CFE State Storage Account (`Microsoft.Storage/storageAccounts`)**:
   - Creates a `StorageV2` account (`f5cfe<uniqueString>`) with tag `"f5_cloud_failover_label": "<cfeFailoverLabel>"`, used by CFE to persist failover state locks.
3. **CFE-Managed Route Table (`Microsoft.Network/routeTables`)**:
   - Creates `<prefix>-cfe-rt` tagged with `f5_cloud_failover_label` and `f5_self_ips: "10.0.2.4,10.0.2.5"`.
   - Includes a pre-configured route (`CFE-Managed-Internal-Route`) for `cfeManagedRoutePrefix` (`10.10.0.0/16`) with `nextHopType: "VirtualAppliance"` pointing to `10.0.2.4`.
4. **3x Network Security Groups (`Microsoft.Network/networkSecurityGroups`)**:
   - `<prefix>-eth0-mgmt-nsg` (SSH `22`, HTTPS `443` from `restrictedSrcAddress`)
   - `<prefix>-eth1-ext-nsg` (HTTP `80`, HTTPS `443` from `Internet`)
   - `<prefix>-eth2-int-nsg` (HA ConfigSync `TCP 4353, 443, 1026`, Heartbeat `UDP 1026`, and Internal VNet traffic)
5. **5x Standard Public IP Addresses (`Microsoft.Network/publicIPAddresses`)**:
   - 1x Floating VIP Public IP (`<prefix>-eth1-floating-vip-pip`, tagged with `f5_cloud_failover_label`)
   - 2x Management Public IPs (`<prefix>-vm1-eth0-mgmt-pip`, `<prefix>-vm2-eth0-mgmt-pip`)
   - 2x Per-VM External Self Public IPs (`<prefix>-vm1-eth1-self-pip`, `<prefix>-vm2-eth1-self-pip`) so both active and standby BIG-IPs always have outbound internet access to talk to `management.azure.com` and `blob.core.windows.net`.
6. **2x 3-NIC BIG-IP VMs + Optional Automated CFE RPM Installer**:
   - Deployed in an Availability Set (`<prefix>-avset`).
   - When `installCfePackage: true`, an Azure `CustomScript` extension downloads the F5 CFE RPM from GitHub and installs it via the BIG-IP iControl LX package manager.

---

## Parameters

| Parameter | Type | Default Value | Description |
| :--- | :--- | :--- | :--- |
| `deploymentPrefix` | `string` | `bigip-cfe` | Naming prefix for all deployed resources. |
| `location` | `string` | `[resourceGroup().location]` | Azure region for all resources. |
| `adminUsername` | `string` | `azureuser` | Administrator username for both BIG-IP VMs. |
| `authenticationType` | `string` | `password` | `password` or `sshPublicKey`. |
| `adminPasswordOrKey` | `securestring` | *(Required)* | Administrator password or SSH public key. |
| `vmSize` | `string` | `Standard_DS4_v2` | VM size (must support at least 3 NICs). |
| `imageOffer` | `string` | `f5-big-ip-byol` | F5 Marketplace offer. |
| `imageSku` | `string` | `f5-big-all-2slot-byol` | F5 Marketplace SKU. |
| `bigipVersion` | `string` | `latest` | F5 BIG-IP image version. |
| `cfeFailoverLabel` | `string` | `bigip-ha-cfe-deployment` | Value for `f5_cloud_failover_label` resource tag. |
| `cfeVersion` | `string` | `2.1.1` | Version of F5 Cloud Failover Extension RPM to install. |
| `installCfePackage` | `bool` | `true` | Automatically install CFE RPM on both VMs via `CustomScript` extension. |
| `floatingExternalVipPrivateIp` | `string` | `10.0.1.100` | Secondary floating private IP on `eth1`. |
| `cfeManagedRoutePrefix` | `string` | `10.10.0.0/16` | Route CIDR in the Azure Route Table managed by CFE. |
| `restrictedSrcAddress` | `string` | `*` | Source IP/CIDR allowed to access Management ports (`22`, `443`) on `eth0`. |

---

## Deployment & CFE Activation Instructions

### 1. Deploy the ARM Template

```bash
az group create \
  --name bigip-cfe-rg \
  --location eastus

az deployment group create \
  --resource-group bigip-cfe-rg \
  --template-file azuredeploy-ha-cfe.json \
  --parameters @azuredeploy-ha-cfe.parameters.json \
  --parameters restrictedSrcAddress="<YOUR_PUBLIC_IP>/32"
```

### 2. Configure BIG-IP Self-IPs & HA Clustering (DSC)
- **VM1**:
  - `external` VLAN (`1.1` / `eth1`) Self-IP: `10.0.1.4/24`
  - `internal` VLAN (`1.2` / `eth2`) Self-IP: `10.0.2.4/24`
- **VM2**:
  - `external` VLAN (`1.1` / `eth1`) Self-IP: `10.0.1.5/24`
  - `internal` VLAN (`1.2` / `eth2`) Self-IP: `10.0.2.5/24`
- Establish Device Trust, ConfigSync, and Failover between VM1 and VM2 over `10.0.2.4` and `10.0.2.5`.
- Create your Virtual Server on `traffic-group-1` with destination address `10.0.1.100` (`floatingExternalVipPrivateIp`).

### 3. Submit the CFE Declaration to Both BIG-IP VMs
Retrieve the pre-built `cfePayloadJson` output from the ARM deployment and `POST` it to `/mgmt/shared/cloud-failover/declare` on both BIG-IPs:

```bash
curl -k -u azureuser:'<PASSWORD>' \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "class": "Cloud_Failover",
    "environment": "azure",
    "externalStorage": {
      "scopingTags": {
        "f5_cloud_failover_label": "bigip-ha-cfe-deployment"
      }
    },
    "failoverAddresses": {
      "enabled": true,
      "scopingTags": {
        "f5_cloud_failover_label": "bigip-ha-cfe-deployment"
      }
    },
    "failoverRoutes": {
      "enabled": true,
      "scopingTags": {
        "f5_cloud_failover_label": "bigip-ha-cfe-deployment"
      },
      "scopingAddressRanges": [
        {
          "range": "10.10.0.0/16"
        }
      ],
      "defaultNextHopAddresses": {
        "discoveryType": "static",
        "items": [
          "10.0.2.4",
          "10.0.2.5"
        ]
      }
    }
  }' \
  https://<BIGIP_MGMT_FQDN>/mgmt/shared/cloud-failover/declare
```

### 4. Verify CFE Trigger Script on BIG-IP
Ensure the BIG-IP failover scripts call CFE upon state transition (configured automatically by CFE when declared, verifiable via):
```bash
tmsh list sys failover
curl -k -u azureuser:'<PASSWORD>' https://localhost/mgmt/shared/cloud-failover/inspect
```
