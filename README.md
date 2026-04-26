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

gpedit.msc

## Step 4: Configure Wazuh to Collect PowerShell Logs 📡

### 🎯 Purpose

After enabling PowerShell Script Block Logging on `win10-lab`, the next step was to make sure Wazuh could collect those logs.

This allowed PowerShell activity from the Windows 10 endpoint to be reviewed centrally in the Wazuh dashboard during the investigation.

---

### 🛠️ Actions Performed

On the Windows 10 endpoint `win10-lab`, Notepad was opened as Administrator.

The Wazuh agent configuration file was opened:


C:\Program Files (x86)\ossec-agent\ossec.conf
<img width="1002" height="716" alt="Screenshot 2026-04-26 151551" src="https://github.com/user-attachments/assets/50addf23-3db3-4a37-885e-671f7de762e8" />

## Step 5: Enable Process Creation Logging 🧾

### 🎯 Purpose

PowerShell logging is useful, but it does not always show every external executable that runs on a Windows endpoint.

To improve visibility, Windows process creation logging was enabled on `win10-lab`. This allowed the lab to capture Event ID `4688`, which records when a new process is created.

This telemetry is useful for identifying tools and commands such as:

powershell.exe
cmd.exe
setspn.exe
klist.exe

### 🛠️ Actions Performed

On the Windows 10 endpoint win10-lab, Local Group Policy was opened:

gpedit.msc

Process creation auditing was enabled through the following path:

Computer Configuration
→ Windows Settings
→ Security Settings
→ Advanced Audit Policy Configuration
→ Audit Policies
→ Detailed Tracking
→ Audit Process Creation

### <img width="947" height="706" alt="Screenshot 2026-04-26 154600" src="https://github.com/user-attachments/assets/90447d0b-9de9-490a-a734-9b10b19c1f6a" />
🔎 Evidence Collected

After enabling process creation auditing, Windows generated Event ID 4688 logs on win10-lab.

Wazuh successfully collected these process creation events.

Key fields observed in Wazuh included:

agent.name: win10-lab
data.win.system.eventID: 4688
data.win.eventdata.newProcessName
data.win.eventdata.parentProcessName
data.win.eventdata.commandLine

## Step 6: Enumerate SPNs from win10-lab 🔎

### 🎯 Purpose

Before requesting a Kerberos service ticket, an attacker or analyst must identify accounts with Service Principal Names, or SPNs.

In this step, SPNs were enumerated from the domain-joined Windows 10 endpoint `win10-lab` to confirm that the `svc_sql` service account was discoverable in Active Directory.

---

### 🛠️ Actions Performed

On `win10-lab`, PowerShell was used to enumerate SPNs in the `lab.local` domain:


setspn -T lab.local -Q */*

### 🔎 Evidence Collected

The output showed the lab service account svc_sql and its assigned SPN:

CN=svc_sql,CN=Users,DC=lab,DC=local
    MSSQLSvc/dc01.lab.local:1433

 <img width="1011" height="826" alt="step-6-spn-enumeration-svc-sql" src="https://github.com/user-attachments/assets/f554ded5-c176-40b1-9dd9-1ca3f7d3ba27" />

## Step 7: Request a Kerberos Service Ticket 🎟️

### 🎯 Purpose

After identifying the `svc_sql` service account and its SPN, the next step was to request a Kerberos service ticket for that SPN.

This simulates Kerberoasting-style activity because Kerberoasting relies on requesting service tickets for SPN-backed service accounts.

---

### 🛠️ Actions Performed

On `win10-lab`, the Kerberos ticket cache was cleared:

klist purge

### 🔎 Evidence Collected

The klist output confirmed that a Kerberos service ticket was requested and cached for:

MSSQLSvc/dc01.lab.local:1433

The ticket also showed that the KDC involved was:

DC01.lab.local

<img width="953" height="707" alt="klist" src="https://github.com/user-attachments/assets/3f5a279e-6135-4abb-a0a9-71b0cda6761e" />

## Step 8: Validate Kerberos Event ID 4769 for the Service Account 🔐

### 🎯 Purpose

After requesting the Kerberos service ticket, the next step was to validate that the domain controller logged the activity as a **Kerberos service ticket request**.

This is an important detection point because **Windows Security Event ID 4769** is commonly used to identify Kerberoasting-related activity. In this lab, I specifically looked for a ticket request tied to the `svc_sql` service account.

---

### 🛠️ Actions Performed

On `dc01`, I reviewed Security Event ID `4769` events with PowerShell:

Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4769} -MaxEvents 20 | Format-List TimeCreated, Id, Message

### 🔎 Evidence Collected

The event showed:

Event ID: 4769
Service Name: svc_sql
Account Name: Administrator@LAB.LOCAL
Client Address: ::ffff:192.168.56.20
Ticket Encryption Type: 0x17

The 0x17 encryption type is especially important because it indicates RC4-HMAC, which is commonly associated with Kerberoasting-style ticket requests.

This confirmed that the Kerberos ticket request for the svc_sql SPN was successfully generated and logged on the domain controller.

<img width="963" height="717" alt="svc_sql" src="https://github.com/user-attachments/assets/68bb15f3-a21c-4973-98ed-57191d9c8b35" />


## Step 9: Finalize the Investigation and Document Findings 🏁

### 🎯 Purpose

The final step of this lab was to summarize the investigation, connect the evidence across systems, and document what the activity would mean from a SOC analyst perspective.

This step turns the lab from a technical exercise into a recruiter-ready security investigation.

---

### 🧠 Investigation Summary

The lab successfully demonstrated Kerberoasting-style activity in an Active Directory environment.

A service account named `svc_sql` was created with the following SPN:

MSSQLSvc/dc01.lab.local:1433

<img width="963" height="717" alt="svc_sql" src="https://github.com/user-attachments/assets/1bc79767-7760-42dc-8faa-5f0f0594258a" />


### 🚨 Detection Logic

This activity can be investigated by looking for:

Windows Security Event ID 4769
Service Name: svc_sql
Client Address: 192.168.56.20
Ticket Encryption Type: 0x17
Failure Code: 0x0

Additional supporting telemetry includes:

PowerShell Event ID 4104
Windows Process Creation Event ID 4688
setspn.exe execution
klist usage
Kerberos service ticket cache activity

### 🛡️ Defensive Takeaways

This lab showed that defenders should monitor for:

Service ticket requests for SPN-backed accounts
RC4 Kerberos encryption type 0x17
SPN enumeration activity
setspn.exe execution from workstations
Unusual Kerberos ticket requests from non-administrative endpoints
PowerShell commands related to Kerberos, SPNs, or ticket activity
