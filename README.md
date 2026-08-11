# Active-Directory-IAM-Lab

# 🛡️ Enterprise Active Directory & Identity Access Management (IAM) Lab

![Windows Server](https://img.shields.io/badge/OS-Windows%20Server%202025-blue)
![Hypervisor](https://img.shields.io/badge/Hypervisor-VMware%20Workstation%20Pro-orange)
![Security Focus](https://img.shields.io/badge/Focus-IAM%20%7C%20AD%20FS%20%7C%20RBAC-green)

## 📌 Executive Summary
This repository documents the step-by-step deployment, identity provisioning, and federated authentication setup for a miniature enterprise environment named **CDU Bank**. 

The goal of this multi-week project is to establish a centralized **Identity and Access Management (IAM)** framework, enforce **Role-Based Access Control (RBAC)**, and configure **Active Directory Federation Services (AD FS)** for Single Sign-On (SSO) using modern OAuth 2.0 and OpenID Connect protocols.

---

## 🛠️ Lab Environment & Specs
* **Hypervisor:** VMware Workstation Pro 17
* **Operating System:** Windows Server 2025
* **Target Domain:** `CDU.BANK` (NetBIOS: `CDU`)
* **Primary Domain Controller:** `CDUBANK-DC01`
* **Network Mode:** NAT / Host-Only (Isolated Sandbox)

---

## 📅 Weekly Implementation Breakdown

---

### 🔹 Week 2: Infrastructure Setup & Active Directory Deployment

#### Objectives
1. Deploy Windows Server 2025 inside VMware.
2. Troubleshoot administrative credential issues and configure a static IPv4 address.
3. Promote the server to a Domain Controller for `CDU.BANK`.
4. Create an initial system recovery snapshot.

#### Key Implementation Steps
* **VM Provisioning:** Allocated 8GB RAM and 60GB disk space in VMware. Set the computer host name to `CDUBANK-DC01`.
* **Credential Hardening:** Opened `cmd.exe` as Administrator to reset local account credentials (`net user Administrator S3cur3P@ssword1!`) to pass prerequisite checks.
* **Static Network Configuration:** Configured a static IPv4 address and set the Preferred DNS Server to the local loopback address (`127.0.0.1` / DC IP) on adapter `Ethernet0` to ensure proper DNS and AD mapping.
* **AD DS Promotion:** Installed Active Directory Domain Services (AD DS) and promoted the server to a new forest domain (`CDU.BANK`).
* **Backup:** Took a VMware guest state snapshot named `AD_Server` for safe system rollback.

![Week 2 Active Directory Promotion Proof](img/week2-ad-setup.png)
*Figure 1: Successful installation and promotion of Active Directory Domain Controller.*

---

### 🔹 Week 3: Identity Provisioning & Role-Based Access Control (RBAC)

#### Objectives
1. Create Organizational Units (OUs) reflecting organizational departments (`Finance`, `IT`).
2. Implement Role-Based Access Control (RBAC) via Security Groups.
3. Provision domain user accounts adhering to standardized User Principal Names (UPNs).
4. Audit compliance using PowerShell script `marking-script-1.ps1`.

#### Directory Architecture (`CDU Bank`)

```text
CDU.BANK (Domain Root)
│
├── 📁 Finance (Organizational Unit)
│   ├── 👥 Core Banking (Security Group)  ──> 👤 Jake Weather (jweather)
│   └── 👥 Front Desk (Security Group)     ──> 👤 Jess Black (jblack)
│
└── 📁 IT (Organizational Unit)
    ├── 👥 Service Desk (Security Group)   ──> 👤 Jane Smith (jsmith)
    └── 👥 Field Staff (Security Group)    ──> 👤 Joe Bloggs (jbloggs)

## Week 4: AD FS Deployment & OAuth / OpenID SSO Setup
Objectives
Create a dedicated service account (adfs-service) with a non-expiring password.

Deploy and configure Active Directory Federation Services (AD FS).

Configure an Application Group using OAuth 2.0 / OpenID Connect.

Validate federated claims using the external Claims X-Ray suite and marking-script-2.ps1.

Key Implementation Steps
Service Account Setup: Created user adfs-service in AD Administrative Center, set the password to YourSecurePassword123!, and checked Password Never Expires.

AD FS Installation: Executed automated script adfs-setup.ps1 to install AD FS binaries and generate local SSL certificates (sts.cdu.bank).

AD FS Configuration: Configured the AD FS Server role, bound it to the CDU\Administrator domain credentials, and named the service CDU Bank ADFS.

OAuth Application Setup: * Opened AD FS Management and added a new Application Group named Claims Xray (type: Web Browser Accessing a Web Application).

Configured Redirect URI to https://claimsxray.com/token.

Applied Access Control Policy set to Permit Everyone.

Token Handshake Test: Opened https://claimsxray.com inside the Server VM, selected OpenID, initiated the login process, and successfully authenticated against AD FS to receive federated claims.

Automated Audit: Executed marking-script-2.ps1 to confirm successful federated token issuance.
