# Backdoors & Breaches Campaign: The ToolShell Game

*A scenario based on the 2025 SharePoint "ToolShell" zero-day exploitation campaign. Nation-state actors are chaining vulnerabilities in your internet-facing SharePoint servers to gain a foothold, escalate privileges, and establish persistent access. Can your team detect the intrusion before sensitive corporate data walks out the door?*

>This scenario was part of a series presented to undergaduate Comp Sci students. Modified rules apply, meaning that solves require a specific way of thinking outlined in the "why procedures succeed or fail" section. In different environments, you may use these as discussion prompts instead. In some other scenarions I may call this "hard mode" It is an educational game, just play and help everyone learn. Feel free to add to the story or change anything to fit your audience.
---
## For in person or at play.backdoorsandbreaches.com. 
Use Core + Trimarc Expansion 
For those who don't know me, I'm Steven, and I've led over 1000 Backdoors & Breaches sessions at this point. I am trying to write up some of the scenarios to share to hopefully help others leverage Backdoors & Breaches to teach or run tabletops. Some scenarios are based on real events, others are more "well, what if" scenarios. 

**Thank you to the Black Hills Information Security team for making and sharing this game with the community!** I stand on the shoulders of giants and often leverage tools made by others to help make training more enjoyable or impactful. 

Note: Sometimes I modify some rules. :) Where I do, I try to note that in the scenario. Original rules are here: https://www.blackhillsinfosec.com/tools/backdoorsandbreaches/ 

Play online at https://play.backdoorsandbreaches.com or purchase cards from the Spearphish General Store (https://spearphish-general-store.myshopify.com/collections/backdoors-breaches-incident-response-card-game)
---

## Background

In July 2025, cybersecurity researchers discovered that multiple advanced persistent threat (APT) groups, including several Chinese-aligned actors, were actively exploiting a pair of chained zero-day vulnerabilities in on-premises Microsoft SharePoint Server. The exploit chain, dubbed "ToolShell" by the security community, combined a critical authentication bypass with a high-severity remote code execution flaw, allowing unauthenticated attackers to gain SYSTEM-level access on unpatched servers.

Within weeks of the initial discovery, hundreds of SharePoint instances worldwide were compromised. Some threat actors used the access for espionage and data theft. Others, including hybrid groups, deployed ransomware. By October 2025, nearly 40% of incident response engagements at one major threat intelligence firm involved ToolShell exploitation.

This campaign puts your team in the shoes of defenders at a mid-size organization that runs an on-premises SharePoint environment for internal collaboration and document management. The server is internet-facing for remote employee access — and it hasn't been patched in months.

---

## The Attack Chain

| Category | Card | Why This Card |
|---|---|---|
| **Initial Compromise** | **Exploitable External Service** | The attackers exploit the chained SharePoint vulnerabilities (ToolShell) against an internet-facing server to gain unauthenticated remote code execution with SYSTEM privileges. No user interaction or credentials were required. |
| **Pivot and Escalate** | **Kerberoasting** | With SYSTEM access on the SharePoint server (which is domain-joined), the attackers request Kerberos service tickets for service accounts, then crack them offline to obtain plaintext passwords for highly privileged domain service accounts. |
| **C2 and Exfil** | **DNS as C2** | To avoid triggering web proxy or firewall alerts on large HTTPS transfers, the attackers encode stolen documents into DNS TXT queries to attacker-controlled domains, slowly exfiltrating data through a channel that many organizations do not inspect closely. |
| **Persistence** | **DLL Hijacking** | The attackers place a malicious DLL in a directory used by a legitimate SharePoint service, ensuring their code is loaded every time the service restarts. This survives reboots and blends in with normal SharePoint operations. |


---

## Procedure Cards

Deal the following cards as the **Established Procedures** (these get the +3 modifier):

Sometimes the IC can be nice and give four viable established procedures. Feel free to change these up (And yes, I have accidentally run online games with no viable established procedures, oops)

| Procedure Card | Rationale |
|---|---|
| **Server Analysis** | The organization has a documented procedure for analyzing server logs, configurations, and running processes. |
| **Firewall Log Review** | Perimeter firewall logs are available and reviewed. |
| **Endpoint Security Protection Analysis** | EDR agents are deployed on servers, including the SharePoint server. |
| **SIEM (Security Information and Event Management)** | A SIEM is deployed, ingesting Windows Event Logs and IIS logs from the SharePoint server. |

---

## Starting Narrative

*Read the following to the Defenders:*

> Alright, team. We need your help. This morning our vulnerability management team ran their monthly scan and flagged something concerning: our on-premises SharePoint server. You know, the one that hosts HR policies, project documentation, and a good chunk of our internal collaboration. Well, it is missing several critical patches, including ones released months ago.
>
> That alone would be bad enough. But here's where it gets worse. About an hour ago, our network monitoring system flagged an unusual spike in DNS query volume originating from the SharePoint server. The queries are going to external domains we don't recognize, and the query patterns look... strange. Long, encoded-looking subdomain strings.
>
> We also found that the SharePoint application pool recycled unexpectedly twice last night, and IIS worker processes have been consuming more CPU than usual this week.
>
> We don't know yet if this is a misconfiguration, a performance issue, or something more serious. That's what you're here to find out.

---

## Incident Captain Guidance

### Why Procedures Succeed or Fail

**Server Analysis** - *Can reveal Initial Compromise or Persistence.* If the defenders examine the SharePoint server's IIS logs, they will find HTTP requests with unusual URI patterns targeting specific SharePoint API endpoints — the exploitation artifacts of the ToolShell chain. If they inspect loaded DLLs or compare file hashes against known-good baselines, they can discover the malicious DLL persistence. If they only check Windows Event Logs for failed logins or standard errors, they won't find much — the exploit didn't generate authentication events, and the DLL doesn't trigger application errors.

**SIEM** - *Can reveal Pivot and Escalate.* If the defenders query for Kerberos TGS-REQ events (Event ID 4769) with RC4 encryption from the SharePoint server's machine account targeting service accounts, this is a textbook Kerberoasting indicator. The SIEM could also surface the unusual DNS traffic if DNS query logs are being ingested. If the defenders search the SIEM for generic alerts or malware detections, nothing will appear — the attackers used native Windows protocols and legitimate-looking processes.

**Firewall Log Review** - *Can reveal C2 and Exfil.* If the defenders inspect DNS traffic leaving the network and notice the SharePoint server is making an abnormally high volume of DNS queries to a small set of external domains with long, high-entropy subdomain labels, this points directly to the DNS exfiltration channel. If they are only reviewing HTTP/HTTPS traffic or looking for connections to known malicious IPs, the firewall logs appear clean — DNS often passes through unfiltered or is forwarded to an external resolver without deep inspection.

**Endpoint Security Protection Analysis** - *Can reveal Persistence.* If the EDR agent on the SharePoint server is configured to monitor for unsigned DLLs being loaded by IIS worker processes, or if the defenders hunt for DLLs in non-standard directories, they can uncover the malicious DLL. If the defenders only check for known malware signatures or behavioral alerts from user workstations, the SharePoint server may not be their focus and the DLL (which is custom and unsigned but not in any signature database) won't trigger a detection.

**NetFlow / Zeek / RITA Analysis (Other Procedure, no modifier)** - *Can reveal C2 and Exfil.* Network flow analysis would reveal the beaconing pattern in DNS: regular, periodic bursts of DNS queries from the SharePoint server to external domains, with consistent timing intervals. This is a strong indicator of automated data exfiltration.

**Isolation (Other Procedure, no modifier)** - *Can support revealing any card.* If defenders isolate the SharePoint server from the network, the DNS exfiltration stops, and the attackers lose their access. However, this also takes a major collaboration platform offline. The Incident Captain should challenge the defenders to justify this decision and explore whether they've gathered enough evidence first.

---

## Discussion Points for Debrief

After the game, facilitate a conversation around these themes:

1. **Patch Management and Internet-Facing Assets:** The SharePoint server was missing patches for over two months. What is your organization's patch cadence for internet-facing systems? Is there a separate, accelerated timeline for critical vulnerabilities versus internal-only systems?

2. **DNS Visibility:** Many organizations have limited visibility into DNS traffic. Do you log and inspect DNS queries? Would you detect DNS tunneling or exfiltration in your environment today?

3. **Zero-Day Readiness:** When a zero-day is announced for software you run, what is your emergency response process? Can you identify all instances of the affected software in your environment within hours?

4. **Defense in Depth for Servers:** Domain-joined servers with SYSTEM-level compromise give attackers immediate access to Active Directory attacks like Kerberoasting. How do you limit the blast radius of a single compromised server? Are service account passwords long, random, and rotated? Are you using Group Managed Service Accounts (gMSAs)?

---

## MITRE ATT&CK Mapping

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| Exploitable External Service | T1190 (Exploit Public-Facing Application) |
| Kerberoasting | T1558.003 (Steal or Forge Kerberos Tickets: Kerberoasting) |
| DLL Attacks | T1574.001 (Hijack Execution Flow: DLL Search Order Hijacking) |
| DNS as Exfil | T1048.003 (Exfiltration Over Alternative Protocol: Exfiltration Over Unencrypted Non-C2 Protocol), T1071.004 (Application Layer Protocol: DNS) |

---

## References and additional data

- "ToolShell" SharePoint zero-day exploitation campaign (CVE-2025-53770 & CVE-2025-53771) https://www.microsoft.com/en-us/security/blog/2025/07/22/disrupting-active-exploitation-of-on-premises-sharepoint-vulnerabilities/
- Eye Security disclosure on global SharePoint compromises https://www.eye.security/press/eye-security-detects-large-scale-exploitation-of-critical-microsoft-sharepoint-vulnerability https://www.eye.security/blog/eye-security-uncovers-actively-exploited-zero-day-in-microsoft-sharepoint-cve-2025-53770
- Cisco Talos IR reporting on ToolShell exploitation engagements https://blog.talosintelligence.com/ir-trends-q4-2025/
- MITRE ATT&CK Technique T1190, T1558.003, T1574.001, T1048.003
- Backdoors & Breaches by Black Hills Information Security — [https://www.backdoorsandbreaches.com](https://www.backdoorsandbreaches.com)
