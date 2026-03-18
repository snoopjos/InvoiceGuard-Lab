# 🧾 InvoiceGuard-Lab: Implementing a Secure and Cost-Optimised Billing Repository in Azure

This project demonstrates how to move invoice storage away from a VM's local virtual disk and into a dedicated, secure, and cost-effective Azure Blob Storage solution — while giving auditors scoped access without exposing the server or Azure account.

## 🚀 Business Situation

Our invoicing server is generating thousands of PDF invoices every month, and all of these files are currently saved directly to the server's local hard drive (the virtual disk).

As CEO, three major risks have been identified:
1. **Cost and Scalability:** Constantly buying larger and more expensive disks just to store old files is not economically justifiable.
2. **Data Security:** If the server experiences an error or someone accidentally deletes a file, we risk losing business-critical information we are legally required to retain for **seven years**.
3. **Access:** External auditors need to view certain invoices from time to time. Giving them login details to the server or full access to our Azure account is unacceptable.

The goal is to move invoice storage to a **dedicated, secure and cost-effective cloud solution**.

## 🔒 Security Requirements

1. Invoices **must not** be stored long-term on the VM's local disk.
2. Auditors may only access files via a **read-only, IP-restricted SAS token** — no Azure credentials.
3. Storage must be protected by a **firewall** using private or service endpoints.
4. Data must be replicated to protect against hardware failure.

## 🎯 Objective

Deploy a minimal but secure Azure Blob Storage environment that offloads invoice storage from the VM, enforces lifecycle-based cost management, restricts auditor access to scoped read-only tokens, and protects the storage account behind a network firewall.

---

## 🧠 Thought Process

### Storage Account

To store any blob data in Azure, we first need a Storage Account.
The Storage Account is the top-level container for all of our invoice files.

We configure it with:
- **ZRS (Zone Redundant Storage)** — provides enough redundancy and is more cost effective than GRS
- **Blob Soft Delete** — protects against accidental deletion with a 7-day recovery window
- **No public blob access** — all access is controlled via firewall rules and SAS tokens

### Blob Container

Inside the Storage Account, we create a private container called `invoices`.
This is where all PDF invoices will live. Public access is fully disabled.

### Lifecycle Management

Rather than manually managing storage costs, we configure a Lifecycle Management Rule.
Any blob untouched for **30 days** is automatically moved to the **Cool tier**, reducing costs for older invoices without any manual effort. As Auditers will need access from time to time we do not want to place it in Cold tier as this is the most expensive to access.

### Storage Firewall

We lock the Storage Account down so only trusted sources can reach it.
Our billing subnet is added as an allowed Virtual Network via a **Service Endpoint**, meaning traffic from the VM to storage stays private within the Azure backbone.

### RBAC

To manage blobs using our own identity rather than raw access keys, we assign ourselves the **Storage Blob Data Contributor** role.
This follows the principle of least privilege — no one gets more access than they need.

### SAS Token for Auditors

Rather than sharing any credentials, we generate a **Shared Access Signature (SAS) token** that is:
- **Read-only** — auditors cannot modify or delete anything
- **IP-restricted** — only works from their known office IP range
- **Time-limited** — expires after the agreed access window

They receive a single URL. No Azure account. No server access.

---
 
## 🔨 Implementation Steps
 
### Step 1 — Create a Resource Group
- Create a new Resource Group named **`rg-invoiceguard-lab`**
 
---
 
### Step 2 — Confirm Your VNet and Subnet
- Confirm your existing **VNet** and billing **subnet** are in place from the previous lab
- Note the subnet address range — you will need it when configuring the Storage Firewall
 
---
 
### Step 3 — Create the Billing Server VM
- Create a new **Virtual Machine** inside the resource group
- Place it in the existing billing **subnet**
- Choose **Windows Server** as the OS (this will act as the invoicing server)
- Do **not** assign a public IP — access will be handled via Bastion
- Connect to the VM using **Azure Bastion** to confirm it is running
 
---
 
### Step 4 — Create a Storage Account
- Create a new **Storage Account** inside the resource group
- Set redundancy to **LRS**
- Under Data Protection, enable **Blob Soft Delete** → 7-day retention
- Ensure public blob access is **disabled**
 
---
 
### Step 5 — Create a Blob Container
- Inside the Storage Account, create a new container named **`invoices`**
- Set public access level to **Private**
 
---
 
### Step 6 — Configure a Lifecycle Management Rule
- Add a new **Lifecycle Management Rule**
- Set action: move blobs to **Cool tier** after **30 days** of last modification
 
---
 
### Step 7 — Configure the Storage Firewall
- Go to **Networking** on the Storage Account
- Set public network access to: enabled from **selected virtual networks and IP addresses**
- Add your existing billing subnet as an allowed Virtual Network
  - This creates a **Service Endpoint** (`Microsoft.Storage`) on the subnet
 
---
 
### Step 8 — Assign RBAC for Admin Access
- Go to **Access Control (IAM)** on the Storage Account
- Assign yourself the **Storage Blob Data Contributor** role
 
---
 
### Step 9 — Generate a SAS Token for Auditors
- Go to **Shared Access Signature** on the Storage Account
- Set permissions to **Read only**
- Restrict to the auditor's **known external IP range**
- Set an appropriate **expiry date**
- Generate and share only the **Blob SAS URL** — no Azure credentials required
 
---
 
### Step 10 — Install AzCopy on the Billing Server VM
- Connect to the VM via **Azure Bastion**
- Download and install **AzCopy** directly on the VM
- This simulates how the billing server would push invoices to storage in production
 
---
 
### Step 11 — Simulate Invoice Administration
- Authenticate AzCopy using **`azcopy login`**
- Upload test PDF invoices to the **`invoices`** container
- Verify the files appear in the Azure Portal under your container