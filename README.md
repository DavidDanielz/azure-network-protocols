<p align="center">
  <img src="https://i.imgur.com/Ua7udoS.png" alt="Network traffic inspection"/>
</p>

<h1 align="center">Azure Network Security Groups & Packet Inspection</h1>
<h2 align="center">Controlling VM Traffic with NSG Rules and Verifying It in Wireshark</h2>

<h3 align="center">
Microsoft Azure | Network Security Groups | Wireshark | Windows Server | Ubuntu | DNS
</h3>

<div align="center">

![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Wireshark](https://img.shields.io/badge/Wireshark-1679A7?style=for-the-badge&logo=wireshark&logoColor=white)
![Windows Server](https://img.shields.io/badge/Windows_Server-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

</div>

---

## 📋 Project Overview

**Project type:** Cloud networking and security
**Goal:** Prove, with packet captures, that NSG rules do what I configured them to do.

I built a small Azure lab with Windows and Linux VMs, created NSG rules to allow and deny traffic, and used Wireshark to inspect what actually happened on the wire. For each test I recorded the rule, the expected result, the actual result and the packet evidence.

The same questions come up in identity and access work: who or what can reach a system, over which port, and how do I confirm it? NSG rules answer that at the network layer, so this project is a foundation for the access control work I'm moving toward.

---

## 🧭 Contents

1. [Lab environment](#-lab-environment)
2. [Results at a glance](#-results-at-a-glance)
3. [Step 1: Deploy the VMs](#1️⃣-step-1-deploy-the-vms)
4. [Step 2: Capture baseline traffic](#2️⃣-step-2-capture-baseline-traffic)
5. [Step 3: Block ICMP with an NSG rule](#3️⃣-step-3-block-icmp-with-an-nsg-rule)
6. [Step 4: Block SSH with an NSG rule](#4️⃣-step-4-block-ssh-with-an-nsg-rule)
7. [Step 5: Inspect DNS (A record, cache, CNAME)](#5️⃣-step-5-inspect-dns-a-record-cache-cname)
8. [Step 6: Block SMB file sharing](#6️⃣-step-6-block-smb-file-sharing)
9. [Troubleshooting](#-troubleshooting)
10. [Key takeaways](#-key-takeaways)
11. [What I would do in production](#-what-i-would-do-in-production)

---

## 🖥️ Lab Environment

| Machine | OS | Role |
|---|---|---|
| **Client-1** | Windows 10 | Test client, runs Wireshark |
| **Linux-1** | Ubuntu Server 20.04 | Target for ICMP and SSH tests |
| **DC-1** | Windows Server | DNS server and file share host |

*(Edit this table to match what you actually built.)*

**Tools:** Microsoft Azure (VMs, Virtual Network, NSGs), Wireshark, Remote Desktop (RDP), SSH, `ping`, `nslookup`, `ipconfig`
**Protocols inspected:** ICMP, SSH, RDP, DNS, SMB

All machines are in the same Virtual Network (`10.0.0.0/16`, *edit if different*).

---

## 📊 Results at a Glance

| Test | NSG rule | Expected | Actual | Wireshark evidence |
|---|---|---|---|---|
| Ping Linux-1 | None | Replies | *(fill in)* | Echo request and reply |
| Ping Linux-1 | Deny ICMP inbound | Timeout | *(fill in)* | Requests, no replies |
| SSH to Linux-1 | Deny TCP 22 inbound | Timeout | *(fill in)* | SYN, no SYN-ACK |
| Resolve `mainframe` | None | Fails, then works after A record | *(fill in)* | DNS query, NXDOMAIN, then answer |
| Open `\\DC-1` | Deny TCP 445 inbound | Share unreachable | *(fill in)* | SMB traffic stops |

---

## ⚙️ Hands-On Walkthrough

### 1️⃣ Step 1: Deploy the VMs

**Goal:** Create the VMs in one virtual network so they can reach each other.

<img src="images/step-1/01-resource-group.png" width="100%" alt="Resource group">

**What I did:** Created a resource group and a virtual network for the lab.

---

<img src="images/step-1/02-vms-running.png" width="100%" alt="VMs running">

**What I did:** Deployed Client-1, Linux-1 and DC-1 into the same virtual network and confirmed all three were running.

---

<img src="images/step-1/03-ipconfig.png" width="100%" alt="Private IP addresses">

**What I did:** Recorded each VM's private IP address with `ipconfig` and `ip a` to use in the tests.

**Result:** Three VMs deployed and reachable on the same network.

---

### 2️⃣ Step 2: Capture Baseline Traffic

**Goal:** See normal traffic first so blocked traffic is easy to recognize later.

<img src="images/step-2/01-icmp-baseline.png" width="100%" alt="ICMP baseline">

**What I did:** Started Wireshark on Client-1, pinged Linux-1 and filtered with `icmp`. *(Describe what you saw: matching echo request and echo reply packets.)*

---

<img src="images/step-2/02-ssh-baseline.png" width="100%" alt="SSH baseline">

**What I did:** Connected to Linux-1 over SSH and filtered with `tcp.port == 22`. *(Describe what you saw: the TCP three-way handshake, then encrypted SSH traffic.)*

---

<img src="images/step-2/03-rdp-baseline.png" width="100%" alt="RDP baseline">

**What I did:** Opened an RDP session and filtered with `tcp.port == 3389`. *(Describe what you saw.)*

**Result:** A baseline of what working ICMP, SSH and RDP traffic looks like.

---

### 3️⃣ Step 3: Block ICMP with an NSG Rule

**Goal:** Deny ping with an NSG rule and prove it in the capture.

**Rule created (inbound, on Linux-1's NSG):**

| Setting | Value |
|---|---|
| Source | Any |
| Destination | Any |
| Protocol | ICMP |
| Action | **Deny** |
| Priority | *(your number, lower is evaluated first)* |
| Name | `Deny-ICMP-Inbound` |

<img src="images/step-3/01-deny-icmp-rule.png" width="100%" alt="Deny ICMP rule">

**What I did:** Added the deny rule to the NSG.

---

<img src="images/step-3/02-ping-timeout.png" width="100%" alt="Ping timing out">

**What I did:** Pinged Linux-1 again. *(Result: request timed out.)*

---

<img src="images/step-3/03-icmp-no-reply.png" width="100%" alt="ICMP with no replies in Wireshark">

**What I did:** Checked Wireshark. *(What you saw: echo requests leaving with no replies coming back.)*

---

<img src="images/step-3/04-rule-removed-ping-works.png" width="100%" alt="Ping working after rule removed">

**What I did:** Deleted the rule and confirmed ping worked again.

**Result:** The rule blocked ICMP, and removing it restored it. Cause and effect are both confirmed.

---

### 4️⃣ Step 4: Block SSH with an NSG Rule

**Goal:** Show the same allow and deny behavior on a TCP service.

<img src="images/step-4/01-deny-ssh-rule.png" width="100%" alt="Deny SSH rule">

**What I did:** Added an inbound **Deny** rule for TCP port 22 on Linux-1's NSG.

---

<img src="images/step-4/02-ssh-timeout.png" width="100%" alt="SSH timing out">

**What I did:** Tried to SSH to Linux-1. *(Result: connection timed out.)*

---

<img src="images/step-4/03-syn-no-synack.png" width="100%" alt="SYN packets with no reply">

**What I did:** Captured with `tcp.port == 22`. *(What you saw: repeated SYN packets and no SYN-ACK.)*

**Result:** The NSG dropped the traffic silently. The client gets no answer, not a refusal.

---

### 5️⃣ Step 5: Inspect DNS (A Record, Cache, CNAME)

**Goal:** Capture DNS queries and responses and understand client-side caching.

| # | Action | Result |
|---|---|---|
| 1 | From Client-1, `ping mainframe` and `nslookup mainframe` | *(fails, no record)* |
| 2 | On DC-1, create an **A record** `mainframe` pointing to DC-1's private IP | |
| 3 | `ping mainframe` again | *(succeeds)* |
| 4 | Change the A record to `8.8.8.8`, ping again | *(still uses old address, cached)* |
| 5 | `ipconfig /displaydns`, then `ipconfig /flushdns` | *(cache cleared)* |
| 6 | `ping mainframe` again | *(new address)* |
| 7 | Create a **CNAME** `search` pointing to `www.google.com` | *(alias resolves)* |

<img src="images/step-5/01-nslookup-fail.png" width="100%" alt="nslookup failing">

**What I did:** Tried to resolve `mainframe` with no record in place.

---

<img src="images/step-5/02-a-record.png" width="100%" alt="Creating an A record">

**What I did:** Created the A record on DC-1 and confirmed the ping worked.

---

<img src="images/step-5/03-dns-cache.png" width="100%" alt="DNS cache">

**What I did:** Changed the record, saw the stale cached answer, then flushed the cache.

---

<img src="images/step-5/04-dns-wireshark.png" width="100%" alt="DNS in Wireshark">

**What I did:** Filtered with `dns` in Wireshark. *(Describe the query, the failed response, the later successful response, and that no new query appeared while the answer was cached.)*

**Result:** I can see exactly when a client asks DNS and when it uses its cache.

---

### 6️⃣ Step 6: Block SMB File Sharing

**Goal:** Show that a network rule can cut off a service regardless of its permissions.

<img src="images/step-6/01-smb-working.png" width="100%" alt="SMB share working">

**What I did:** Opened `\\DC-1` from Client-1 and captured with `tcp.port == 445`. *(What you saw: SMB traffic to DC-1.)*

---

<img src="images/step-6/02-deny-445-rule.png" width="100%" alt="Deny 445 rule">

**What I did:** Added an inbound **Deny** rule for TCP port 445 on DC-1's NSG.

---

<img src="images/step-6/03-share-unreachable.png" width="100%" alt="Share unreachable">

**What I did:** Tried to open `\\DC-1` again. *(Result: unreachable.)*

**Result:** The share stopped working at the network layer even though the share and file permissions were unchanged.

---

## 🛠️ Troubleshooting

*(Replace this with 2 or 3 real problems you hit. This section carries a lot of weight with hiring managers. Examples of the format:)*

| Problem | Cause | Fix |
|---|---|---|
| Deny rule had no effect | *(for example, a lower-numbered Allow rule matched first)* | *(for example, gave the Deny rule a lower priority number)* |
| Wireshark showed nothing | *(for example, wrong capture interface)* | *(for example, selected the Ethernet adapter)* |
| Ping to a VM failed from the start | *(for example, Windows Firewall blocks ICMP by default)* | *(for example, enabled the inbound ICMP rule)* |

---

## 🧠 Key Takeaways

- NSG rules are evaluated by **priority**, lowest number first, and the first match wins.
- A **blocked** connection looks like silence in a capture (no reply). A **refused** connection looks like a reset. They are different problems.
- **DNS caching** can make a changed record look broken. Flush the cache before you decide it is.
- **Network rules and permissions are separate layers.** Correct share permissions do not help if the network blocks the port.
- Testing means recording the **expected** result, the **actual** result and the **evidence**.

---

## 🔐 What I Would Do in Production

- Never leave RDP or SSH open to `Any`. Restrict them to a trusted IP range or use Azure Bastion.
- Use **deny by default** and open only the ports each system needs.
- Turn on **NSG flow logs** and review them, so the evidence is available without running a packet capture.
- Group VMs with **application security groups** instead of managing rules by IP address.

---

## ✅ Outcome

I configured NSG rules to allow and deny ICMP, SSH and SMB traffic between Azure VMs, and confirmed each result with Wireshark captures. I also captured DNS resolution and caching behavior. The project shows that I can change cloud network controls, test the change, and prove it with evidence.
