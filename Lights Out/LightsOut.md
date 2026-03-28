# Backdoors & Breaches Campaign: Lights Out

*A scenario loosely based on the 2025 trend of ransomware operators targeting VMware ESXi hypervisors after disabling endpoint protection. Your virtualized infrastructure can host everything, from domain controllers, file servers, to databases and email. The attacker isn't encrypting individual machines. They're going after the hypervisor layer to take down everything at once. Can your team detect the attack before the lights go out?*

---

## For in person or at play.backdoorsandbreaches.com. I typically use the Core + Expansion 
For those who don't know me, I'm Steven, and I've led over 1000 Backdoors & Breaches sessions at this point. I am trying to write up some of the scenarios to share to hopefully help others leverage Backdoors & Breaches to teach or run tabletops. Some scenarios are based on real events, others are more "well, what if" scenarios. 

This was a time pressure/ hard mode scenario that was pretty fun to run. Again, we;re looking for the "why this card" portions but you can play this more closely aligned to the traditional rules. 

**Thank you to the Black Hills Information Security team for making and sharing this game with the community!** I stand on the shoulders of giants and often leverage tools made by others to help make training more enjoyable or impactful. 

Note: Sometimes I modify some rules. Where I do, I try to note that in the scenario. Original rules are here: https://www.blackhillsinfosec.com/tools/backdoorsandbreaches/ 

Play online at https://play.backdoorsandbreaches.com or purchase cards from the Spearphish General Store (https://spearphish-general-store.myshopify.com/collections/backdoors-breaches-incident-response-card-game)
---

## Background

Throughout 2025, ransomware operators increasingly targeted VMware ESXi hypervisors to maximize impact. Rather than encrypting individual virtual machines one by one, attackers realized they could encrypt the virtual disk files (.vmdk) at the hypervisor level, taking down dozens of VMs in a single operation. ESXi hosts often lack EDR coverage entirely, run SSH with weak or reused credentials, and sit outside the scope of many organizations' patch management programs.

The attack pattern typically follows a well-worn path: gain initial access through an exposed service or compromised MSP, escalate privileges on the Windows domain to obtain vCenter or ESXi root credentials, disable or uninstall EDR on key systems to avoid alerting the SOC, then SSH into ESXi hosts directly to deploy a Linux-based ransomware encryptor. By the time anyone notices, every virtualized workload is offline and the ransom note is the only thing on screen.

This campaign puts your team in an environment where a mid-size company runs its entire infrastructure on a VMware cluster managed partially by an external MSP. The attacker is already inside — and they're disabling your defenses before you know it.

---

## The Attack Chain

| Category | Card | Why This Card |
|---|---|---|
| **Initial Compromise** | **Compromised Trusted Relationship** | The attacker compromises the credentials of a technician at the organization's managed service provider (MSP). The MSP has a persistent site-to-site VPN tunnel into the customer environment for remote management. The attacker uses these credentials to access the internal network through the MSP's infrastructure, a trusted connection that bypasses the organization's own perimeter controls. |
| **Pivot and Escalate** | **Kerberoasting** | From the MSP's management subnet, the attacker performs Kerberoasting against the customer's Active Directory, targeting service accounts. They crack the ticket for the vCenter service account, which has local admin privileges on the vCenter Server Appliance and ESXi host access. |
| **C2 and Exfil** | **DNS as Exfil** | Before deploying the ransomware encryptor, the attacker exfiltrates a listing of all VMs, their configurations, backup job metadata (stolen from the backup server VM), and a sample of sensitive files through DNS tunneling. This information is used to tailor the ransom demand and to prove they have the data during extortion negotiations. |
| **Persistence** | **Malicious Firmware** | The attacker enables SSH on ESXi hosts (if not already enabled) and places a cron job in the ESXi host's persistent storage partition that re-enables SSH and resets the root password on reboot. This survives reboots and persists at the hypervisor layer completely invisible to any VM-level monitoring. |


---

## Procedure Cards

Deal the following cards as the **Established Procedures** (these get the +3 modifier):

| Procedure Card | Rationale |
|---|---|
| **Firewall Log Review** | Internal firewall and VPN logs are maintained and accessible. |
| **SIEM (Security Information and Event Management)** | A SIEM is deployed ingesting Windows Event Logs, vCenter events, and VPN authentication logs. |
| **Server Analysis** | The team has procedures for analyzing Windows and Linux servers. |
| **Internal Segmentation** | Network segmentation exists between user VLANs, server VLANs, and the management VLAN (though enforcement varies). |



---

## Starting Narrative

*Read the following to the Defenders:*

> Everyone, drop what you're doing. We have a critical situation.
>
> At 5:47 AM this morning, about thirty minutes before the first shift, our monitoring dashboard went dark. Nagios stopped receiving SNMP polls from most of our infrastructure. Our on-call engineer tried to RDP into the domain controller and got nothing. He tried to access vCenter and got a connection timeout.
>
> He drove to the office and connected directly to the ESXi management interface via the IPMI/iLO console. What he found is alarming: all virtual machines on hosts ESX01, ESX02, and ESX03 are powered off. The datastores show that every .vmdk file has been renamed with a `.locked` extension. There's a text file on each datastore named `README_TO_RESTORE.txt` with a ransom demand for 75 Bitcoin.
>
> However, and this is the one piece of good news, ESX04 and ESX05 have NOT been encrypted yet. They're still running. Our backup server VM is on ESX04. Some of our less critical workloads are on ESX05.
>
> We don't know how the attacker got in, how long they've been here, or whether they're still in the environment right now. The clock is ticking on those two remaining hosts. What do we do?

---

## Incident Captain Guidance

### This Campaign Has Urgency Built In

Unlike other campaigns where defenders are investigating a past event, here the attack is *in progress*. ESX04 and ESX05 have not been encrypted yet. Every turn the defenders spend investigating without also taking protective action is a turn where the attacker might finish the job. The Incident Captain should apply time pressure: after turn 5, if the defenders haven't taken any containment actions on the remaining ESXi hosts, describe the attacker beginning to SSH into ESX04.

### Why Procedures Succeed or Fail

**Firewall Log Review** - *Can reveal Initial Compromise.* If the defenders review VPN logs and connections from the MSP's management subnet, they'll find an active VPN session from the MSP that's been connected for over 72 hours. The source IP and session timing don't match the MSP's normal maintenance windows. Cross-referencing with the MSP reveals that no technician had a scheduled session.

**SIEM** - *Can reveal Pivot and Escalate or Persistence.* If the defenders search for Kerberos TGS-REQ events (4769) targeting the vCenter service account from an unusual source (the MSP subnet), the Kerberoasting activity is visible. If they check vCenter event logs for SSH enablement on ESXi hosts or root password changes, they can find the persistence mechanism. If they only search the SIEM for events from the now-encrypted VMs... those logs stopped flowing when the VMs went down. Any log data not already forwarded to the SIEM is gone.

**Server Analysis** - *Can reveal Persistence or C2 and Exfil.* If the defenders connect directly to one of the still-running ESXi hosts (ESX04 or ESX05) via the DCUI or SSH and examine running processes, cron jobs, and the persistent storage partition, they may find the attacker's persistence mechanism (the cron job) and any staging artifacts. If they examine the backup server VM on ESX04, they can check whether backup metadata was accessed. On the Windows side, if any domain controller was recently re-built from backup, they can look for Kerberoasting artifacts in the event logs.

**Internal Segmentation** - *Can reveal Initial Compromise and support containment.* If the defenders examine whether the management VLAN (where ESXi hosts and vCenter live) is properly segmented from other network zones, they may discover that the MSP's VPN subnet has unrestricted access to the management VLAN, the segmentation policy exists on paper but isn't enforced for the MSP connection. This architectural flaw is how the attacker reached the hypervisors from the MSP foothold. If they enforce segmentation now by blocking the MSP tunnel and isolating the management VLAN, they can contain the attack and protect ESX04/ESX05.

**Endpoint Security Protection Analysis (Other Procedure, no modifier)** - *Can reveal pre-attack activity.* If the defenders check EDR status across the environment, they'll find something alarming: several hours before the encryption event, the EDR agents on key servers were uninstalled or had their tamper protection disabled. The uninstall commands were issued using the vCenter service account's domain admin credentials. This tells the defenders that EDR was deliberately neutralized before the ransomware was deployed, and it points to the compromised vCenter service account.

**Isolation (Other Procedure, no modifier)** - *Critical for saving ESX04/ESX05.* If the defenders isolate the remaining ESXi hosts by disabling SSH, changing root passwords, and disconnecting the management network from the MSP tunnel, they can prevent the attacker from completing the encryption. This is the highest-impact action in the campaign. If they do this before turn 5, they save the backups and remaining workloads.

---

## Discussion Points for Debrief

1. **Hypervisor Security:** Do you have EDR or monitoring on your ESXi hosts? Are ESXi hosts treated as critical assets in your security program, or are they managed infrastructure that falls between the cracks?

2. **MSP Access Controls:** What level of access does your MSP have to your environment? Is it always-on, or is it just-in-time? Do you monitor MSP sessions with the same rigor as internal administrator sessions? Could you revoke MSP access within minutes if needed?

3. **Backup Integrity Under Attack:** If your backup server is on the same infrastructure being targeted, are your backups actually safe? Do you maintain offline or immutable backups that can survive a hypervisor-level encryption attack?

4. **EDR Tamper Protection:** Could an attacker with domain admin credentials disable your EDR? Does your EDR require a separate, cloud-hosted authorization to uninstall? Would you be alerted if agents were being removed en masse?

5. **Ransomware Readiness:** This scenario has a time-pressure element. Does your team have a rehearsed ransomware playbook that includes hypervisor-level attacks? Do you know how to perform ESXi forensics?

---

## MITRE ATT&CK Mapping

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| External Cloud Access (MSP VPN) | T1133 (External Remote Services), T1199 (Trusted Relationship) |
| Kerberoasting (vCenter Service Account) | T1558.003 (Steal or Forge Kerberos Tickets: Kerberoasting) |
| Evil Firmware (ESXi Persistence) | T1053.003 (Scheduled Task/Job: Cron), T1542 (Pre-OS Boot) |
| DNS as Exfil | T1048.003 (Exfiltration Over Alternative Protocol), T1071.004 (Application Layer Protocol: DNS) |

---

## References and background info

- ESXi-targeted ransomware campaigns by Qilin, BlackCat/ALPHV, and other groups, 2024–2025
- Huntress reporting on ESXi ransomware attacks https://www.huntress.com/blog/esxi-vm-escape-exploit
- VMware security advisories on ESXi SSH and management interface hardening https://www.vmware.com/docs/vmware-best-practices-for-hardening-your-infrastructure
- www.ic3.gov/CSA/2023/230208.pdf
- MITRE ATT&CK Techniques T1133, T1199, T1558.003, T1053.003, T1048.003
- Backdoors & Breaches by Black Hills Information Security — [https://www.backdoorsandbreaches.com](https://www.backdoorsandbreaches.com)
