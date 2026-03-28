# Backdoors & Breaches Campaign: Broker's Fee

*A scenario based on the 2025 surge in Initial Access Broker-fueled ransomware attacks against the healthcare sector. An employee's stolen credentials have been sold on a dark web marketplace, and a ransomware affiliate has purchased access to your hospital network. Can your team detect the intrusion before patient records are encrypted and lives are put at risk?*

>This scenario was part of a series presented to undergaduate Comp Sci students. Modified rules apply, meaning that solves require a specific way of thinking outlined in the "why procedures succeed or fail" section. In different environments, you may use these as discussion prompts instead. In some other scenarions I may call this "hard mode" It is an educational game, just play and help everyone learn. Feel free to add to the story or change anything to fit your audience.
---
## For in person or at play.backdoorsandbreaches.com. I typically use the Core + Expansion 
For those who don't know me, I'm Steven, and I've led over 1000 Backdoors & Breaches sessions at this point. I am trying to write up some of the scenarios to share to hopefully help others leverage Backdoors & Breaches to teach or run tabletops. Some scenarios are based on real events, others are more "well, what if" scenarios. 

**Thank you to the Black Hills Information Security team for making and sharing this game with the community!** I stand on the shoulders of giants and often leverage tools made by others to help make training more enjoyable or impactful. 

Note: Sometimes I modify some rules. Where I do, I try to note that in the scenario. Original rules are here: https://www.blackhillsinfosec.com/tools/backdoorsandbreaches/ 

Play online at https://play.backdoorsandbreaches.com or purchase cards from the Spearphish General Store (https://spearphish-general-store.myshopify.com/collections/backdoors-breaches-incident-response-card-game)

## Background

Throughout 2025, the ransomware ecosystem matured into a fully industrialized supply chain. Initial Access Brokers (IABs), specialized cybercriminals who compromise organizations and then sell that network access to the highest bidder, became a critical enablement layer. On chain analysis revealed that IAB activity often preceded ransomware deployments by roughly 30 days, functioning as a leading indicator of attacks to come.

The healthcare sector was hit especially hard. Ransomware groups like Qilin operated a Ransomware-as-a-Service (RaaS) model, providing affiliates with encryption tools, leak site infrastructure, and negotiation playbooks in exchange for a cut of ransom payments. One such affiliate purchased VPN credentials for a regional hospital network from an IAB who had harvested them months earlier through an infostealer malware campaign. The affiliate logged in, moved laterally, and deployed ransomware, disrupting patient care, encrypting clinical systems, and exfiltrating nearly three terabytes of protected health information (PHI).

This campaign puts your team at a mid-size regional healthcare system. You have multiple clinic locations, an electronic health record (EHR) system, and a remote-access VPN for clinical staff.

---

## The Attack Chain

| Category | Card | Why This Card |
|---|---|---|
| **Initial Compromise** | **Credential Stuffing** | An IAB harvested VPN credentials from a clinician's personal device using infostealer malware. Those credentials, reused from a personal email account, were sold on a dark web access marketplace. The ransomware affiliate used them to log into the hospital's SSL VPN. |
| **Pivot and Escalate** | **Local Privilege Escalation** | Once on the internal network via VPN, the affiliate exploited an unpatched local privilege escalation vulnerability on a clinical workstation to gain local administrator access, then used that access to dump cached credentials and move laterally to additional systems. |
| **Persistence** | **Malicious Service** | The affiliate deployed a lightweight remote access tool (RAT) registered as a Windows service on several servers, ensuring persistent access even if the VPN credentials were changed. The service name was crafted to resemble a legitimate medical software update service. |
| **C2 and Exfil** | **HTTPS as Exfil** | Patient records, insurance data, and internal documents were compressed, encrypted, and exfiltrated over HTTPS to attacker-controlled cloud infrastructure before the ransomware payload was detonated. The double extortion model means data theft happened first, encryption second. |

---

## Procedure Cards

Deal the following cards as the **Established Procedures** (these get the +3 modifier):

| Procedure Card | Rationale |
|---|---|
| **Endpoint Security Protection Analysis** | The hospital has an EDR solution deployed on servers and most workstations. |
| **SIEM (Security Information and Event Management)** | A SIEM is in place, ingesting VPN logs, Windows Event Logs, and some application logs. |
| **Permissions Audit** | Does not solve any of the attack chain. Feel free to choose a different card. |
| **Isolation** | The network team has the ability to isolate network segments and disable VPN access on short notice. |

---

## Starting Narrative

*Read the following to the Defenders:*

> Team, we have a developing situation, and time is not on our side. At 6:14 AM this morning, well before normal business hours, the VPN concentrator logged a successful authentication for Dr. Elaine Marsh, a cardiologist at our North Campus clinic. Nothing unusual about that on its own. But Dr. Marsh is currently on a medical conference in Vienna, Austria, and she swears she did not log in.
>
> The source IP for the VPN session traces to a residential ISP in Eastern Europe. The session has been active for over three hours.
>
> In the last hour, our help desk has received two separate calls from nurses at the South Campus clinic saying that a workstation in the cardiology unit displayed a brief command prompt window that opened and closed on its own. They thought it was a Windows update.
>
> We also just received an automated alert from our EDR platform: a new Windows service was registered on one of our file servers, FILESRV02. The service name is "EHRUpdateAgent" but our EHR vendor says they haven't pushed any updates this week.
>
> We need you to figure out what's happening, how far it's spread, and how to contain it before morning rounds begin and two hundred clinicians log in for the day.

---

## Incident Captain Guidance

### Why Procedures Succeed or Fail (Review the Attack Chain, there are additional solves)

**SIEM** - *Can reveal Initial Compromise.* If the defenders query VPN authentication logs for impossible-travel scenarios (a user authenticating from both a US-based IP and a European IP within an implausible timeframe), or for off-hours logins from unusual geolocations, they will immediately identify the Credential Stuffing compromise via Dr. Marsh's VPN account. If they search the SIEM for malware alerts or failed login attempts, they won't find much — the login succeeded on the first attempt with valid credentials.

**Endpoint Security Protection Analysis** - *Can reveal Persistence or Pivot and Escalate.* If the defenders investigate the EDR alert about the "EHRUpdateAgent" service on FILESRV02, they can trace the service binary to an unsigned executable, revealing the Malicious Service persistence. If they use EDR to hunt for evidence of credential dumping tools (such as LSASS memory access patterns) across clinical workstations, they can uncover the Local Privilege Escalation activity. If they only review top-level alert dashboards without digging into the specific service creation event, they may miss the connection.

**Crisis Management** - *Can reveal any card.* Healthcare incident response has unique constraints: you cannot simply shut down clinical systems without risking patient safety. If the defenders invoke Crisis Management to coordinate with clinical leadership and establish manual backup procedures for patient care (paper charting, verbal orders), this buys the technical team time to investigate without the pressure of keeping everything online. The Incident Captain can reward strong Crisis Management by revealing the card most closely related to whatever the defenders are currently investigating. If the defenders skip this step, inject complications: a department head calls demanding to know why systems are slow, and the CEO's office is asking for a statement.

**Isolation** - *Non-standard, but can reveal Pivot and Escalate with proper rationale* If the defenders immediately disable Dr. Marsh's VPN account and disconnect the compromised workstations from the network, the attacker's lateral movement is contained. Network isolation at the segment level (separating clinical VLAN from server VLAN) can halt the privilege escalation in progress. However, if they isolate systems without first gathering volatile evidence, they lose forensic data. The Incident Captain should ask what evidence they're collecting before they pull the plug.

**Firewall Log Review (Other Procedure, no modifier)** - *Can reveal C2 and Exfil.* If the defenders review outbound traffic from FILESRV02 and clinical workstations, they may notice large HTTPS uploads to cloud storage endpoints (e.g., AWS S3 or Azure Blob) occurring in the early morning hours. The volume and timing are abnormal for a healthcare file server. If they're only reviewing inbound traffic or looking for connections to known C2 IP addresses, the HTTPS exfiltration — going to legitimate cloud infrastructure — won't stand out.

**UEBA (Other Procedure, no modifier)** - *Can reveal Initial Compromise or Pivot and Escalate.* Behavioral analytics would flag that Dr. Marsh's account is accessing systems she has never accessed before (file servers, administrative shares, other clinics' workstations). The account's behavior pattern has completely changed from its historical baseline.

---

## Discussion Points for Debrief

After the game, facilitate a conversation around these themes:

1. **Credential Hygiene and Infostealer Exposure:** The initial compromise came from credentials stolen by infostealer malware on a personal device, not a corporate asset. Does your organization monitor dark web marketplaces for exposed employee credentials? Do you enforce unique passwords for VPN access, and is MFA phishing-resistant (e.g., FIDO2)?

2. **The IAB-to-Ransomware Pipeline:** Access brokers often compromise organizations weeks or months before a ransomware affiliate acts. What would it take for your team to detect a dormant, unauthorized VPN session that uses valid credentials?

3. **Healthcare-Specific IR Constraints:** You can't just "unplug everything" when patient care depends on those systems. How does your incident response plan balance containment speed with clinical safety? Have you rehearsed manual fallback procedures?

4. **Double Extortion Reality:** Even if you have perfect backups and can restore encrypted systems, the attackers already exfiltrated the data. How does your organization handle the extortion component? Who makes the decision about ransom payment, and what is the legal and regulatory exposure?

5. **Patch Management on Clinical Systems:** The local privilege escalation exploited an unpatched workstation. Medical environments often delay patching due to vendor certification requirements. How do you manage this tension between security and operational stability?

---

## MITRE ATT&CK Mapping

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| Credential Stuffing | T1078 (Valid Accounts), T1133 (External Remote Services) |
| Local Privilege Escalation | T1068 (Exploitation for Privilege Escalation), T1003 (OS Credential Dumping) |
| Malicious Service / Just Malware | T1543.003 (Create or Modify System Process: Windows Service), T1036 (Masquerading) |
| HTTPS as Exfil | T1567 (Exfiltration Over Web Service), T1048.002 (Exfiltration Over Asymmetric Encrypted Non-C2 Protocol) |

---

## References and background

- Qilin ransomware group operations and RaaS model https://www.threatlocker.com/blog/qilin-raas-group-technical-analysis-from-initial-access-to-beaconing
- Chainalysis 2026 Crypto Crime Report https://www.chainalysis.com/blog/crypto-ransomware-2026/
- DaVita healthcare ransomware breach, exposing 2.7 million patient records https://www.bitdefender.com/en-us/blog/hotforsecurity/davita-confirms-data-breach-impacting-2-4-million-patients
- Cyble research on 2025 ransomware trends https://cyble.com/blog/ransomware-groups-q4-2025-cyble-report/
- MITRE ATT&CK Techniques T1078, T1133, T1068, T1543.003, T1567
- Backdoors & Breaches by Black Hills Information Security — [https://www.backdoorsandbreaches.com](https://www.backdoorsandbreaches.com)
