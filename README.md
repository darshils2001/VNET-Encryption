# Azure Virtual Network (VNet) Encryption

## Overview

**Azure Virtual Network (VNet) Encryption** provides seamless encryption and decryption of traffic between supported Azure Virtual Machines within the same Virtual Network (VNet) or across peered VNets. It enhances data security by ensuring that communication between VMs is encrypted at the network layer, protecting against unauthorized access and packet sniffing.

This feature uses **MACsec (Media Access Control Security)** on the underlying physical links to secure data-in-transit between virtual machines, ensuring confidentiality and integrity without requiring application-level changes.

---

## Step-by-Step Guide

### 1. Create the Virtual Network

Create a **Virtual Network** with **three subnets**:
- **AppGWSubnet** – for the Application Gateway  
- **ClientSubnet** – for the client VM  
- **ServerSubnet** – for the server VM (which will act as the backend for the Application Gateway)

```bash
az network vnet create \
  --name TestVNet \
  --resource-group MyRG \
  --address-prefix 10.0.0.0/16 \
  --subnet-name AppGWSubnet \
  --subnet-prefix 10.0.1.0/24
```

Add the remaining two subnets:
```bash
az network vnet subnet create \
  --resource-group MyRG \
  --vnet-name TestVNet \
  --name ClientSubnet \
  --address-prefix 10.0.2.0/24

az network vnet subnet create \
  --resource-group MyRG \
  --vnet-name TestVNet \
  --name ServerSubnet \
  --address-prefix 10.0.3.0/24
```

---

### 2. Enable VNet Encryption

Enable encryption at the VNet level:
```bash
az network vnet update \
  --name TestVNet \
  --resource-group MyRG \
  --enable-encryption true
```

Verify the encryption status:
```bash
az network vnet show \
  --name TestVNet \
  --resource-group MyRG \
  --query "enableEncryption"
```

---

### 3. Deploy Virtual Machines

Deploy **DV5-series or later VMs** into each subnet.

**Client VM:**
```bash
az vm create \
  --name ClientVM \
  --resource-group MyRG \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --vnet-name TestVNet \
  --subnet ClientSubnet
```

**Server VM (AppGW backend):**
```bash
az vm create \
  --name ServerVM \
  --resource-group MyRG \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --vnet-name TestVNet \
  --subnet ServerSubnet
```

Ensure **Accelerated Networking** is enabled:
```bash
az network nic show \
  --name ClientVMNic \
  --resource-group MyRG \
  --query "enableAcceleratedNetworking"
```

If it’s disabled:
```bash
az network nic update \
  --name ClientVMNic \
  --resource-group MyRG \
  --accelerated-networking true
az vm restart --name ClientVM --resource-group MyRG
```

Repeat the same for **ServerVMNic**.

---

### 4. Deploy Application Gateway

Deploy an **Application Gateway** in the AppGW subnet and configure **ServerVM** as its backend target.

```bash
az network application-gateway create \
  --name TestAppGW \
  --resource-group MyRG \
  --vnet-name TestVNet \
  --subnet AppGWSubnet \
  --capacity 2 \
  --sku Standard_v2 \
  --http-settings-cookie-based-affinity Disabled \
  --frontend-port 80 \
  --routing-rule-type Basic \
  --servers <ServerVMPrivateIP>
```

---

### 5. Verify Communication and Encryption

From **ClientVM**, test communication through **AppGW**:
```bash
curl -v http://<AppGWFrontendIP>
```

Then, verify VNet encryption logs using **Network Watcher → Flow Logs**, confirming traffic between the subnets is encrypted.

---

## Summary

Azure VNet Encryption ensures **secure, low-latency, in-transit encryption** for VM traffic within VNets or peered VNets. It’s transparent to applications, simple to enable, and integrates seamlessly with **Network Watcher** for verification.
