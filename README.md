# VNET-Encryption
Documentation and testing guide for Azure Virtual Network (VNet) Encryption — covers what it is, how to use it, and how to verify encrypted traffic between subnets.

# Azure Virtual Network (VNet) Encryption

## Overview

**Azure Virtual Network (VNet) Encryption** provides seamless encryption and decryption of traffic between supported Azure Virtual Machines within the same Virtual Network (VNet) or across peered VNets. It enhances data security by ensuring that communication between VMs is encrypted at the network layer, protecting against unauthorized access and packet sniffing.

This feature uses **MACsec (Media Access Control Security)** on the underlying physical links to secure data-in-transit between virtual machines, ensuring confidentiality and integrity without requiring application-level changes.

---

## Why Use VNet Encryption?

- 🔒 **Enhanced Security:** Protects east-west traffic (VM-to-VM) inside VNets from interception.  
- ⚙️ **Transparent Implementation:** No configuration changes needed in applications.  
- ⚡ **Hardware-Accelerated Performance:** Supported on specific VM series (e.g., DV5) to minimize latency and CPU overhead.  
- 🌍 **Regulatory Compliance:** Meets encryption-in-transit requirements for sensitive workloads.

---

## Supported Scenarios

| Scenario | Supported | Notes |
|-----------|------------|-------|
| Same VNet communication | ✅ | Encrypted automatically between supported VMs. |
| Peered VNets (same region) | ✅ | Supported if both VNets have encryption enabled. |
| Cross-region VNet peering | ❌ | Not supported. |
| Hybrid (VPN/ExpressRoute) | ❌ | Not applicable — use IPsec or MACsec separately. |

---

## Prerequisites

1. **Supported VM sizes** — DV5, EV5, or later.  
2. **Accelerated Networking** must be enabled.  
3. **Virtual Network Encryption** must be enabled at the VNet level.  
4. **Azure Network Watcher** should be enabled for monitoring and verification.

Reference: [What is Azure Virtual Network encryption?](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-encryption)

---

## Step-by-Step Guide

### 1. Create the Virtual Network

Define three subnets:
- **Client**
- **Server**
- **Application Gateway (AppGW)**

```bash
az network vnet create \
  --name TestVNet \
  --resource-group MyRG \
  --address-prefix 10.0.0.0/16
