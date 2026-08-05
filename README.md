# ☁️ Azure AZ-104 Project 02 — Secure Storage with Private Endpoint

![Microsoft Azure](https://img.shields.io/badge/Microsoft_Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Azure Storage](https://img.shields.io/badge/Azure_Storage-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Private Link](https://img.shields.io/badge/Private_Link-Implemented-success?style=for-the-badge)
![ARM Template](https://img.shields.io/badge/ARM_Template-Infrastructure_as_Code-blueviolet?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)

---

## 📖 Business Scenario

A company needs to store internal documents in Azure while preventing access through the public internet.

The environment must allow only workloads deployed inside an approved Azure Virtual Network to access Blob Storage. Access to Blob data must be controlled through Microsoft Entra ID and Azure RBAC. The complete environment must also be reproducible through an ARM template.

---

## 🎯 Objectives

- Create a Virtual Network with separate workload and private-endpoint subnets
- Protect the workload subnet with a Network Security Group
- Allow RDP only from a trusted public IP
- Allow HTTPS traffic from the workload subnet to the private-endpoint subnet
- Deploy an Azure Storage Account with public network access disabled
- Create a private Blob container named `documents`
- Connect Blob Storage to the VNet through a Private Endpoint
- Configure Private DNS resolution
- Validate DNS and HTTPS connectivity from a test VM
- Demonstrate management-plane and data-plane RBAC
- Recreate the complete environment using an ARM template

---

## 🏗️ Architecture

```text
                              Public Internet
                                    |
                     Storage public access disabled
                                    X
                                    |
                         Azure Storage Account
                         Blob container: documents
                                    ^
                                    |
                             Azure Private Link
                                    |
                          Private Endpoint 10.20.2.x
                                    |
+------------------------------------------------------------------+
|                 Virtual Network: 10.20.0.0/16                     |
|                                                                  |
|  Workload subnet                     Private endpoint subnet      |
|  10.20.1.0/24                        10.20.2.0/24                  |
|                                                                  |
|  Test VM 10.20.1.x  -- HTTPS 443 --> Private Endpoint            |
|         |                                                        |
|         +-- NSG                                                   |
|             - Allow RDP 3389 from trusted IP only                |
|             - Allow HTTPS 443 to 10.20.2.0/24                    |
|             - Deny other outbound traffic                        |
+------------------------------------------------------------------+
                                    |
                         Private DNS Zone
                privatelink.blob.core.windows.net
```

---

## 🧱 Resources Deployed

| Resource | Purpose |
|---|---|
| Resource Group | Logical lifecycle and deployment boundary |
| Virtual Network | Private Azure network |
| Workload Subnet | Hosts the test VM or application workloads |
| Private Endpoint Subnet | Hosts the Blob private endpoint |
| Network Security Group | Restricts inbound and outbound traffic |
| Storage Account | Stores private Blob data |
| Blob Container | Stores project documents |
| Private Endpoint | Provides a private IP for Blob Storage |
| Private DNS Zone | Resolves the normal Blob hostname to the private IP |
| Test VM | Validates DNS, HTTPS, and RBAC |
| ARM Template | Recreates the environment automatically |

---

## 🌐 Network Design

### Virtual Network

```text
10.20.0.0/16
```

### Subnets

```text
snet-workload-01           10.20.1.0/24
snet-private-endpoints-01  10.20.2.0/24
```

The workload and private endpoint are separated because they have different security and lifecycle requirements.

---

## 🔥 NSG Rules

### Inbound

```text
Allow-RDP-MyIP
Protocol: TCP
Port: 3389
Source: trusted public IP /32
Action: Allow
```

### Outbound

```text
Allow-HTTPS-PrivateEndpoint
Protocol: TCP
Port: 443
Source: 10.20.1.0/24
Destination: 10.20.2.0/24
Action: Allow
```

```text
Deny-All-Other-Outbound
Action: Deny
```

The test VM can reach Blob Storage over HTTPS through the private endpoint, while unnecessary outbound traffic is blocked.

---

## 💾 Storage Security

The Storage Account is configured with:

- Public network access: **Disabled**
- Secure transfer required: **Enabled**
- Minimum TLS version: **TLS 1.2**
- Anonymous Blob access: **Disabled**
- Default authorization: **Microsoft Entra ID**
- Storage encryption: **Microsoft-managed keys**
- Blob soft delete: **7 days**
- Container soft delete: **7 days**

The Blob container is:

```text
documents
```

with public access disabled.

---

## 🔐 Private Endpoint and DNS

The private endpoint connects specifically to the Blob sub-resource.

Applications continue to use the normal hostname:

```text
<storage-account>.blob.core.windows.net
```

Inside the VNet, Azure Private DNS resolves it to:

```text
<storage-account>.privatelink.blob.core.windows.net
        ↓
10.20.2.x
```

DNS determines the destination IP. HTTPS then carries encrypted traffic to that private IP over TCP port 443.

---

## 👤 RBAC Model

Two roles were used for the test user:

### Reader

Provides management-plane visibility:

- Can see the Storage Account
- Can view its settings
- Cannot modify networking or configuration

### Storage Blob Data Contributor

Provides data-plane permissions:

- Can list containers
- Can upload blobs
- Can download blobs
- Can delete blobs
- Cannot administer the Storage Account

This demonstrates that network access, management permissions, and data permissions are separate controls.

---

## 🧪 Validation

### DNS test from the Azure VM

```powershell
nslookup <storage-account>.blob.core.windows.net
```

Expected result:

```text
10.20.2.x
```

### HTTPS test from the Azure VM

```powershell
Test-NetConnection <storage-account>.blob.core.windows.net -Port 443
```

Expected result:

```text
TcpTestSucceeded : True
```

### DNS test from the local PC

The same hostname resolves to a public Azure IP outside the VNet. Access still fails because the Storage Account public network endpoint is disabled.

### RBAC tests

| Test | Expected Result |
|---|---|
| View Storage Account | Allowed with Reader |
| Upload Blob | Allowed |
| Download Blob | Allowed |
| Delete Blob | Allowed |
| Change Storage networking | Denied |
| Change IAM assignments | Denied |

---

## 🚀 ARM Deployment

The `arm/azuredeploy.json` template deploys the complete environment.

### Portal deployment

1. Open **Deploy a custom template**
2. Select **Build your own template in the editor**
3. Load `arm/azuredeploy.json`
4. Create or select a Resource Group
5. Enter:
   - A globally unique Storage Account name
   - Your current public IP in `/32` format
   - A secure VM administrator password
   - Optional Entra object ID for automatic RBAC assignment
6. Select **Review + create**
7. Deploy

### Important

Do not commit real passwords, access keys, SAS tokens, or private credentials to GitHub.

---

## 📁 Repository Structure

```text
azure-az104-project-02-secure-storage/
├── README.md
├── arm/
│   ├── azuredeploy.json
│   └── azuredeploy.parameters.example.json
├── images/
└── scripts/
```

---

## 📸 Recommended Screenshots

Add these files to the `images/` directory:

```text
01-resource-group.png
02-vnet-subnets.png
03-nsg-rules.png
04-storage-networking.png
05-private-endpoint.png
06-private-dns.png
07-nslookup-private-ip.png
08-test-netconnection.png
09-rbac-assignments.png
10-blob-container.png
11-arm-deployment.png
```

Embed them in the README using:

```markdown
![VNet and subnets](images/02-vnet-subnets.png)
```

---

## 🛡️ Security Decisions

| Decision | Reason |
|---|---|
| Separate subnets | Different security and lifecycle requirements |
| RDP restricted to one IP | Reduces exposure to scanning and brute-force attacks |
| Public Storage access disabled | Prevents access through the internet-facing endpoint |
| Private Endpoint | Gives Blob Storage a private IP inside the VNet |
| Private DNS | Ensures the Blob hostname resolves to the private endpoint |
| HTTPS only | Encrypts data in transit |
| RBAC separation | Enforces least privilege across management and data planes |
| ARM template | Makes the deployment repeatable and auditable |

---

## 📚 Lessons Learned

- HTTPS does not require the public internet; it only requires an IP network path.
- Private Endpoints bring a private interface for an Azure PaaS service into a VNet.
- Private DNS is essential because applications still use the normal Azure service hostname.
- NSGs control network reachability, while Azure RBAC controls permitted actions.
- Management-plane roles and data-plane roles solve different access problems.
- Exported ARM templates often include runtime-generated resources and hardcoded values that must be cleaned before reuse.
- Infrastructure as Code should be validated through a fresh redeployment.

---

## 💼 Skills Demonstrated

- Azure Virtual Networking
- Subnet design
- Network Security Groups
- Azure Storage
- Blob Storage
- Azure Private Link
- Private Endpoints
- Azure Private DNS
- Microsoft Entra ID
- Azure RBAC
- Infrastructure as Code
- ARM Templates
- Connectivity troubleshooting
- Cloud security hardening

---

## 👨‍💻 Author

**Orestis Prf**

AZ-104 hands-on learning portfolio
