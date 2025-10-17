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
az network vnet create   --name TestVNet   --resource-group MyRG   --address-prefix 10.0.0.0/16
```

### 2. Enable VNet Encryption

```bash
az network vnet update   --name TestVNet   --resource-group MyRG   --enable-encryption true
```

Verify encryption:
```bash
az network vnet show   --name TestVNet   --resource-group MyRG   --query "enableEncryption"
```

---

### 3. Deploy Virtual Machines

Deploy **DV5-series VMs** in each subnet:
```bash
az vm create   --name ClientVM   --resource-group MyRG   --image Ubuntu2204   --size Standard_D2s_v5   --vnet-name TestVNet   --subnet Client
```

Enable **Accelerated Networking**:
```bash
az network nic show   --name ClientVMNic   --resource-group MyRG   --query "enableAcceleratedNetworking"
```

If disabled:
```bash
az network nic update   --name ClientVMNic   --resource-group MyRG   --accelerated-networking true
az vm restart --name ClientVM --resource-group MyRG
```

---

### 4. Test Connectivity

From Client VM:
```bash
curl -v http://<ServerPrivateIP>
```

From Application Gateway:
```bash
curl -vk https://<BackendVMPrivateIP>
```

---

### 5. Verify Traffic Encryption

Use **Azure Network Watcher → Flow Logs**:
1. Open **Network Watcher** → **Flow Logs**.  
2. Create a new log for the encrypted VNet.  
3. Enable analytics and send logs to a storage account.  
4. Generate traffic between VMs.  
5. Inspect flow logs (`.json`) in your storage account.

Example:
```json
{
  "flowTuples": [
    "2025-10-17T15:45:00Z,10.0.0.4,10.0.0.5,443,51234,T,I,A,0,0,0,0"
  ],
  "vnetEncryptionEnabled": true
}
```

Reference: [Virtual Network Flow Logs | Azure Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/traffic-analytics)

---

## Troubleshooting

| Issue | Possible Cause | Resolution |
|--------|----------------|------------|
| Traffic not encrypted | VM size not supported | Use DV5 or EV5 series |
| Connectivity issues | VM not restarted | Stop/start VM to apply encryption |
| Accelerated Networking disabled | NIC misconfiguration | Enable Accelerated Networking |

---

## Summary

Azure VNet Encryption ensures **secure, low-latency, in-transit encryption** for VM traffic within VNets or peered VNets. It’s transparent to applications, simple to enable, and integrates seamlessly with **Network Watcher** for verification.

---

**References**
- [Azure Virtual Network Encryption](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-encryption)  
- [Virtual Network Flow Logs - Azure Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/traffic-analytics)

