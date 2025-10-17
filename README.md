# Azure Virtual Network (VNet) Encryption

## Overview

**Azure Virtual Network (VNet) Encryption** provides seamless encryption and decryption of traffic between supported Azure Virtual Machines within the same Virtual Network (VNet) or across peered VNets. It secures east–west traffic at the network layer, protecting data-in-transit from unauthorized access and packet sniffing.

This capability leverages **MACsec (Media Access Control Security)** on the underlying physical network to encrypt traffic transparently, maintaining confidentiality and integrity without requiring any application-level changes.

---

## Step-by-Step Deployment Guide

### 1. Create the Virtual Network and Subnets

Create a **Virtual Network** with three subnets:
- **AppGWSubnet** – for the Application Gateway  
- **ClientSubnet** – for the client VM  
- **ServerSubnet** – for the server VM (backend for the Application Gateway)

```bash
az network vnet create \
  --name TestVNet \
  --resource-group MyRG \
  --address-prefix 10.0.0.0/16 \
  --subnet-name AppGWSubnet \
  --subnet-prefix 10.0.1.0/24
````

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

Enable encryption for the VNet:

```bash
az network vnet update \
  --name TestVNet \
  --resource-group MyRG \
  --enable-encryption true
```

Verify:

```bash
az network vnet show \
  --name TestVNet \
  --resource-group MyRG \
  --query "enableEncryption"
```

---

### 3. Deploy Virtual Machines

Deploy **DV5-series or later VMs** in each subnet.

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

**Server VM (backend):**

```bash
az vm create \
  --name ServerVM \
  --resource-group MyRG \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --vnet-name TestVNet \
  --subnet ServerSubnet
```

Check and enable **Accelerated Networking**:

```bash
az network nic show \
  --name ClientVMNic \
  --resource-group MyRG \
  --query "enableAcceleratedNetworking"
```

If disabled:

```bash
az network nic update \
  --name ClientVMNic \
  --resource-group MyRG \
  --accelerated-networking true
az vm restart --name ClientVM --resource-group MyRG
```

Repeat for **ServerVMNic**.

> **Note:** Stop/start the VM to ensure Accelerated Networking is applied properly.

---

### 4. Deploy the Application Gateway

Deploy an **Application Gateway** in the AppGW subnet and configure **ServerVM** as the backend.

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

## 🧩 Verify Encrypted Traffic Using Network Watcher Flow Logs

### 1. Enable Network Watcher

1. In the **Azure Portal**, search for **Network Watcher**.
2. Select the **region** where your VNet resides.
3. If not enabled, click **Enable Network Watcher**.

---

### 2. Create a Flow Log

1. Go to **Network Watcher → Flow Logs** → click **+ Create**.
2. Configure as follows:

| Setting                 | Recommended Value            | Description                      |
| ----------------------- | ---------------------------- | -------------------------------- |
| **Subscription**        | Your active subscription     | Must match the VNet region       |
| **Resource Group**      | Same as your VNet            | Keeps related resources together |
| **NSG**                 | Attached to the ServerSubnet | Captures VM-to-VM flows          |
| **Flow log version**    | v2                           | Provides detailed flow data      |
| **Storage account**     | e.g., `vnetencryptionlogs`   | Stores `.json` flow logs         |
| **Retention (days)**    | 7–30                         | Choose per your test duration    |
| **Traffic analytics**   | ✅ Enabled                    | Enables visual insights          |
| **Processing interval** | 10 minutes                   | Near real-time updates           |

Click **Review + Create** → **Create**.

---

### 3. Generate Test Traffic

From the **Client VM**, send traffic through the Application Gateway:

```bash
curl -v http://<AppGWFrontendIP>
```

Wait 10–15 minutes for logs to appear.

---

### 4. View and Verify Flow Logs

In your **Storage Account**, navigate to:

```
insights-logs-networksecuritygroupflowevent/
  resourceId=/SUBSCRIPTIONS/<subId>/RESOURCEGROUPS/<rg>/PROVIDERS/MICROSOFT.NETWORK/NETWORKSECURITYGROUPS/<nsgName>/
  year=<YYYY>/month=<MM>/day=<DD>/hour=<HH>/
```

Download a `.json` log and inspect it:

```json
{
  "records": [
    {
      "time": "2025-10-17T20:10:00Z",
      "flowTuples": [
        "10.0.2.4,10.0.3.5,443,52000,T,I,A,0,0,0,0"
      ],
      "vnetEncryptionEnabled": true
    }
  ]
}
```

---

### 5. Analyze with Traffic Analytics

1. In **Network Watcher → Traffic Analytics**, open your workspace.
2. Review dashboards for:

   * **Top Talkers** (VMs/Subnets)
   * **Encrypted Flows**
   * **Inbound/Outbound Traffic**
   * **Allowed vs. Denied Flows**

---

## ✅ Summary

By enabling **VNet Encryption** and configuring **Flow Logs with Traffic Analytics**, you can confirm that communication between **Client**, **AppGW**, and **Server** subnets is securely encrypted in transit.



---

