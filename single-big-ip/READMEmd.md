# F5 BIG-IP Single-VM (Multi-NIC) Azure ARM Template

This repository contains the Azure Resource Manager (ARM) template for deploying a standalone, **multi-NIC F5 BIG-IP Virtual Machine** on Microsoft Azure with dedicated Network Security Groups (NSGs) and separated Management (`eth0`) and Public Traffic (`eth1`) network planes.

## Files

- **ARM Template**: [`azuredeploy.json`](./azuredeploy.json)
- **Parameters File**: [`azuredeploy.parameters.json`](./azuredeploy.parameters.json)

---

## Architecture Overview

```
                     [ Administrator ]                [ Internet Clients ]
                             |                                  |
                             | (SSH 22 / HTTPS 443)             | (HTTP 80 / HTTPS 443)
                             v                                  v
                  +---------------------+            +---------------------+
                  |  eth0 Mgmt PIP      |            |  eth1 External PIP  |
                  +----------+----------+            +----------+----------+
                             |                                  |
                  +----------+----------+            +----------+----------+
                  |   eth0-mgmt-nsg     |            |    eth1-ext-nsg     |
                  |  (mgmt-subnet)      |            |  (external-subnet)  |
                  |   10.0.0.0/24       |            |    10.0.1.0/24      |
                  +----------+----------+            +----------+----------+
                             |                                  |
                       [ eth0: mgmt ]                    [ eth1: 1.1 ]
                             |                                  |
                             +----------------+-----------------+
                                              |
                                    +---------+---------+
                                    |   F5 BIG-IP VM    |
                                    | (Standard_DS3_v2) |
                                    +-------------------+
```

### Interface Mapping

| Linux / Azure NIC | BIG-IP Interface | Azure Resource Name | Subnet | NSG | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`eth0` (Primary)** | `mgmt` | `<vmName>-eth0-mgmt-nic` | `mgmt-subnet` (`10.0.0.0/24`) | `<vmName>-eth0-mgmt-nsg` | Out-of-band management (SSH `22` & TMUI GUI/REST `443`). IP Forwarding disabled. |
| **`eth1` (Secondary)** | `1.1` | `<vmName>-eth1-ext-nic` | `external-subnet` (`10.0.1.0/24`) | `<vmName>-eth1-ext-nsg` | Public application VIP traffic (HTTP `80` & HTTPS `443`). IP Forwarding enabled. |

---

## Resources Deployed

1. **Network Security Groups (`Microsoft.Network/networkSecurityGroups`)**:
   - **`<vmName>-eth0-mgmt-nsg`**: Allows inbound TCP `22` (SSH) and TCP `443` (TMUI Management GUI / iControl REST) from `restrictedSrcAddress`.
   - **`<vmName>-eth1-ext-nsg`**: Allows inbound TCP `80` (HTTP) and TCP `443` (HTTPS) from `Internet`.
2. **Virtual Network (`Microsoft.Network/virtualNetworks`)**:
   - Address space `10.0.0.0/16` with two subnets (`mgmt-subnet` and `external-subnet`), each associated with its respective NSG.
3. **Standard Public IP Addresses (`Microsoft.Network/publicIPAddresses`)**:
   - **`<vmName>-eth0-mgmt-pip`**: Static Standard Public IP with DNS label for BIG-IP management.
   - **`<vmName>-eth1-ext-pip`**: Static Standard Public IP with DNS label for public Virtual Server (VIP) traffic.
4. **Network Interfaces (`Microsoft.Network/networkInterfaces`)**:
   - **`<vmName>-eth0-mgmt-nic`**: Attached to `mgmt-subnet` as the primary NIC (`eth0`).
   - **`<vmName>-eth1-ext-nic`**: Attached to `external-subnet` as the secondary NIC (`eth1`) with `enableIPForwarding: true`.
5. **F5 BIG-IP Virtual Machine (`Microsoft.Compute/virtualMachines`)**:
   - Deploys the specified F5 Marketplace image/offer (defaults to BYOL `f5-big-all-2slot-byol`) with a Premium SSD OS Disk.

---

## Parameters

| Parameter | Type | Default Value | Description |
| :--- | :--- | :--- | :--- |
| `vmName` | `string` | `bigip-vm` | Name of the F5 BIG-IP Virtual Machine. |
| `location` | `string` | `[resourceGroup().location]` | Azure region for all resources. |
| `adminUsername` | `string` | `azureuser` | Administrator username for the BIG-IP VM. |
| `authenticationType` | `string` | `password` | Authentication type (`password` or `sshPublicKey`). |
| `adminPasswordOrKey` | `securestring` | *(Required)* | SSH public key or password for the VM. |
| `vmSize` | `string` | `Standard_DS3_v2` | Azure VM size (must support 2+ NICs). |
| `imageOffer` | `string` | `f5-big-ip-byol` | F5 Marketplace offer (`f5-big-ip-byol`, `f5-big-ip-best`, `f5-big-ip-good`, `f5-big-ip-better`). |
| `imageSku` | `string` | `f5-big-all-2slot-byol` | F5 Marketplace SKU. |
| `bigipVersion` | `string` | `latest` | F5 BIG-IP image version. |
| `vnetName` | `string` | `bigip-vnet` | Virtual Network name. |
| `vnetAddressPrefix` | `string` | `10.0.0.0/16` | Virtual Network CIDR block. |
| `mgmtSubnetName` | `string` | `mgmt-subnet` | Management subnet name (`eth0`). |
| `mgmtSubnetPrefix` | `string` | `10.0.0.0/24` | Management subnet CIDR block (`eth0`). |
| `externalSubnetName` | `string` | `external-subnet` | External/public subnet name (`eth1`). |
| `externalSubnetPrefix` | `string` | `10.0.1.0/24` | External/public subnet CIDR block (`eth1`). |
| `restrictedSrcAddress` | `string` | `*` | Source IP/CIDR permitted to reach management ports (`22` and `443`) on `eth0`. |

---

## Deployment Instructions

### 1. Accept F5 Marketplace Terms
Before your first deployment in an Azure subscription, accept the Marketplace legal terms:

```bash
az vm image terms accept \
  --publisher f5-networks \
  --offer f5-big-ip-byol \
  --plan f5-big-all-2slot-byol
```

### 2. Create Resource Group & Deploy Template

```bash
az group create \
  --name bigip-single-rg \
  --location eastus

az deployment group create \
  --resource-group bigip-single-rg \
  --template-file azuredeploy.json \
  --parameters @azuredeploy.parameters.json \
  --parameters restrictedSrcAddress="<YOUR_PUBLIC_IP>/32"
```

---

## Template Outputs

| Output Name | Description |
| :--- | :--- |
| `eth0MgmtPublicIP` | Public IP address assigned to `eth0` for Management access. |
| `eth0MgmtFqdn` | Fully Qualified Domain Name (FQDN) for `eth0` Management access. |
| `managementGuiUrl` | HTTPS URL (`https://<fqdn>`) for the BIG-IP TMUI Configuration Utility. |
| `sshCommand` | Ready-to-use SSH command to connect to `eth0`. |
| `eth1PublicTrafficVIP` | Public IP address assigned to `eth1` for public application traffic. |
| `eth1PublicTrafficFqdn` | FQDN assigned to `eth1` for public application traffic. |
