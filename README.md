# Windows Server — Active Directory Trust Guide
> Bidirectional trust setup between two domains entirely via CLI (PowerShell/netdom).

---

## Table of Contents
1. [Environment Overview](#1-environment-overview)
2. [Network Connectivity Test](#2-network-connectivity-test)
3. [DNS Name Resolution Test](#3-dns-name-resolution-test)
4. [Validate GUI DNS Server](#4-validate-gui-dns-server)
5. [Configure Conditional Forwarder](#5-configure-conditional-forwarder)
6. [Create the Trust](#6-create-the-trust)
7. [Trust Verification](#7-trust-verification)

---

## 1. Environment Overview

| | Source (CLI) | Target (GUI) |
|---|---|---|
| **Hostname** | WS141CLI | WS141GUI |
| **Domain** | MazingerCLI141.lan | Mazinger141.lan |
| **IP** | 10.2.141.160 | 10.2.141.254 |

---

## 2. Network Connectivity Test

Before anything else, verify that both servers can reach each other over the network.

![Basic connectivity test](https://github.com/user-attachments/assets/0ee0e68a-9bf5-42fa-b049-0f61dd84bf4f)

All 4 packets responded successfully — basic network connectivity confirmed 

---

## 3. DNS Name Resolution Test

Attempt to resolve the GUI domain name (`Mazinger141.lan`) from the CLI server.

![DNS resolution failure](https://github.com/user-attachments/assets/0ed4cd45-a067-426f-a2b2-62c5c7842580)

The resolution fails — the CLI DNS has no knowledge of `Mazinger141.lan`.  
A **Conditional Forwarder** must be configured before proceeding.

---

## 4. Validate GUI DNS Server

Query the GUI DNS server directly by IP to confirm it resolves its own domain correctly.

```powershell
Resolve-DnsName -Name Mazinger141.lan -Server 10.2.141.254 -Type A
```

![GUI DNS validation](https://github.com/user-attachments/assets/6676ea51-961e-4a91-b5ed-6f1294165329)

The GUI DNS responds correctly and is ready to act as the forwarder target 

---

## 5. Configure Conditional Forwarder

Tell the CLI DNS server: *"For any query about `Mazinger141.lan`, forward it to `10.2.141.254`"*.

```powershell
Add-DnsServerConditionalForwarderZone `
    -Name "Mazinger141.lan" `
    -MasterServers 10.2.141.254 `
    -ReplicationScope "Forest"
```

![Forwarder creation](https://github.com/user-attachments/assets/403c90a2-0732-4d5c-a920-760dc5e84ee8)

No error returned — the forwarder was created successfully.

### Verification

Confirm the zone appears in the DNS server zone list.

![DNS zone list](https://github.com/user-attachments/assets/826a4890-57e9-4098-b77e-24a19a414419)

`Mazinger141.lan` is now listed as a **Forwarder** zone 

Test name resolution again without specifying a DNS server manually.

![DNS resolution success](https://github.com/user-attachments/assets/029b55d5-ca9b-4744-8eb0-08901355999e)

`Mazinger141.lan` resolves to `10.2.141.254` — DNS fully operational 

---

## 6. Create the Trust

### Check and import the Active Directory module

```powershell
Get-Module -ListAvailable -Name ActiveDirectory
```

![AD module check](https://github.com/user-attachments/assets/9b5c9194-9a0b-4a55-9019-3cd7e1b2e3de)

> If the module is not installed, run:
> ```powershell
> Install-WindowsFeature -Name RSAT-AD-PowerShell
> ```

```powershell
Import-Module ActiveDirectory
```

![Module import](https://github.com/user-attachments/assets/d8ca6717-d508-4ff4-882f-7caa4670922d)

### Create the bidirectional trust

Use `netdom` to establish the trust entirely from CLI.

```powershell
netdom trust MazingerCLI141.lan `
    /domain:Mazinger141.lan `
    /userD:Mazinger141\Administrador `
    /passwordD:* `
    /add `
    /twoway
```

![Trust creation](https://github.com/user-attachments/assets/37059a3f-2f9a-4550-845a-14ada2dea76f)

The command completed successfully — the trust has been created 

---

## 7. Trust Verification

Run the following commands to confirm the trust is active and the Kerberos secure channel is working.

```powershell
netdom query trust /domain:MazingerCLI141.lan
nltest /domain_trusts
nltest /sc_verify:Mazinger141.lan
```

![Trust verification](https://github.com/user-attachments/assets/717cea07-637c-4363-a628-de6017a691a8)

| Check | Result |
|---|---|
| `Direct Outbound + Direct Inbound` | Bidirectional trust confirmed |
| `HAS_IP / HAS_TIMESERV` | Network and NTP reachable |
| `NERR_Success` — connection status | Secure channel established |
| `NERR_Success` — trust verification | Trust fully operational |

> **Note:** The `Attr: quarantined` flag is expected on External Trusts.  
> It indicates that SID filtering is active, which is a security best practice.

---
