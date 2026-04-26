# 🔥 Kerberoasting Detection Lab: Active Directory + Wazuh SIEM

## 📌 Overview

This lab demonstrates how Kerberoasting-style activity can be detected and investigated in an Active Directory environment using Wazuh SIEM, Windows Security logs, PowerShell logging, and process creation telemetry.

The goal of this project was to build a realistic blue-team investigation lab where a domain workstation requests a Kerberos service ticket for a service account with an SPN, then analyze the resulting telemetry across the endpoint, domain controller, and SIEM.

---

## 🎯 Lab Objectives

- Build a working Active Directory detection lab
- Create a roastable service account with an SPN
- Enable Kerberos service ticket auditing on the domain controller
- Enable PowerShell Script Block Logging on the Windows endpoint
- Enable Windows process creation logging
- Generate Kerberoasting-style service ticket activity
- Investigate Windows Event IDs `4769`, `4104`, and `4688`
- Validate telemetry in Wazuh and on the source Windows systems

---

## 🧱 Lab Environment

| System | Role | Description |
|---|---|---|
| Ubuntu Server | Wazuh Server | SIEM / log analysis platform |
| Windows Server | `dc01` | Domain Controller for `LAB.LOCAL` |
| Windows 10 | `win10-lab` | Domain-joined endpoint |
| Kali Linux | Attacker VM | Reserved for future attack simulation |
| Metasploitable | Vulnerable host | Reserved for future expansion |

---

## 🛠️ Tools and Technologies

- Wazuh SIEM
- Active Directory Domain Services
- Windows Server
- Windows 10
- PowerShell
- Windows Event Viewer
- Group Policy
- Kerberos authentication
- VirtualBox

---

## 🔐 Attack Scenario

A test service account named `svc_sql` was created in Active Directory and assigned the following Service Principal Name:

## 🧪 Lab Walkthrough

This section documents the full build and investigation process used to create the Kerberoasting detection lab.

---

## Step 1: Create a Roastable Service Account

### Purpose

Kerberoasting targets Active Directory service accounts that have a Service Principal Name, or SPN, assigned. To simulate this safely, a lab service account named `svc_sql` was created.

### Actions Performed

On the domain controller `dc01`, a service account was created:

powershell
$password = ConvertTo-SecureString "Password123!" -AsPlainText -Force
New-ADUser -Name "svc_sql" -SamAccountName "svc_sql" -AccountPassword $password -Enabled $true
MSSQLSvc/dc01.lab.local:1433

## Step 2: Enable Kerberos Service Ticket Auditing

### Purpose

Kerberoasting activity generates Kerberos service ticket requests. On a Windows domain controller, these requests are logged as Windows Security Event ID `4769`.

### Actions Performed

<img width="1007" height="838" alt="Step 2" src="https://github.com/user-attachments/assets/a2fda17f-52c6-4705-9b5f-0316b521713a" />


On `dc01`, Kerberos service ticket auditing was enabled through Group Policy:

text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Advanced Audit Policy Configuration
→ Audit Policies
→ Account Logon
→ Audit Kerberos Service Ticket Operations

## Step 3: Enable PowerShell Script Block Logging ⚡

### 🎯 Purpose

PowerShell logging provides visibility into command activity on the Windows endpoint. Script Block Logging is especially valuable because it records the actual PowerShell content being executed.

For this lab, PowerShell logging was enabled on `win10-lab` so suspicious commands and Kerberos-related activity could be reviewed in Wazuh.

---

### 🛠️ Actions Performed

On the Windows 10 endpoint `win10-lab`, Local Group Policy was opened:

```text
gpedit.msc

![PowerShell Script Block Logging Event ID 4104 on win10-lab](images/step-3-powershell-4104-event-viewer.png)
