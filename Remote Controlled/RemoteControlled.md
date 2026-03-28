# Backdoors & Breaches Campaign: Remote Controlled
## Author: Steven Lorenz

*A scenario based on the 2025 surge in Remote Monitoring and Management (RMM) tool abuse. Attackers are weaponizing the same legitimate tools your IT team depends on, ScreenConnect, AnyDesk, and others, to gain persistent, stealthy access to your network. Traditional malware is nowhere to be found. Can your team detect an attacker hiding in plain sight inside your own management tooling?*

---

## For in person or at play.backdoorsandbreaches.com. 
Uses the Core + Expansion 
For those who don't know me, I'm Steven, and I've led over 1000 Backdoors & Breaches sessions at this point. I am trying to write up some of the scenarios to share to hopefully help others leverage Backdoors & Breaches to teach or run tabletops. Some scenarios are based on real events, others are more "well, what if" scenarios. 

**Thank you to the Black Hills Information Security team for making and sharing this game with the community!** I stand on the shoulders of giants and often leverage tools made by others to help make training more enjoyable or impactful. 

Note: Sometimes I modify some rules. Where I do, I try to note that in the scenario. Original rules are here: https://www.blackhillsinfosec.com/tools/backdoorsandbreaches/ 

Play online at https://play.backdoorsandbreaches.com or purchase cards from the Spearphish General Store (https://spearphish-general-store.myshopify.com/collections/backdoors-breaches-incident-response-card-game)

---

## Background

Throughout 2025 and into early 2026, cybersecurity researchers documented an explosive increase in attackers abusing legitimate RMM tools as their primary means of initial access, persistence, and lateral movement. Huntress research revealed that RMM-based intrusions surged dramatically while traditional malware and RAT usage dropped in parallel, attackers were ditching custom hacking tools in favor of "living off the land" through software that IT departments already trust and allow.

The attack pattern is insidious: a phishing email delivers a trojanized ScreenConnect installer (disguised as an invoice, support document, or software update). Once installed, the attacker has a fully functional remote access session using a signed, legitimate binary that most endpoint protection tools explicitly whitelist. From there, attackers harvest credentials, deploy additional RMM tools for redundancy, and stage ransomware, all without ever dropping traditional malware.

In May 2025, ConnectWise itself disclosed a breach by a nation-state actor that affected ScreenConnect cloud customers, further demonstrating that the RMM supply chain is under active siege.

---

## The Attack Chain

| Category | Card | Why This Card |
|---|---|---|
| **Initial Compromise** | **Phish** | An employee receives a phishing email disguised as a document from a trusted vendor. The attachment is a PDF containing a blurred image with a button labeled "Open in Adobe." Clicking it downloads a trojanized ScreenConnect installer that connects back to the attacker's self-hosted ScreenConnect server. |
| **Pivot and Escalate** | **Internal Password Spray** | Using the ScreenConnect session, the attacker runs encoded PowerShell commands to enumerate the domain and performs an internal password spray against service accounts and admin accounts using commonly reused passwords. Several accounts with weak passwords are compromised. |
| **C2 and Exfil** | **HTTPS as Exfil** | All command-and-control communication occurs over standard HTTPS to the attacker's self-hosted RMM infrastructure. Data exfiltration happens through the RMM tools' built-in file transfer capabilities, encrypted, signed, and indistinguishable from legitimate RMM traffic without deep inspection. |
| **Persistence** | **Malicious Service** | The attacker deploys a second RMM tool (Tactical RMM with MeshAgent) registered as a Windows service alongside the ScreenConnect session thus establishing redundant remote access channels. If one is discovered and removed, the other persists. The service names mimic legitimate IT management software. |


---

## Procedure Cards

Pick 2-3 of the following cards as the **Established Procedures** (these get the +3 modifier):

| Procedure Card | Rationale |
|---|---|
| **Endpoint Security Protection Analysis** | EDR is deployed across endpoints, though RMM tools may be on the allowlist. |
| **SIEM (Security Information and Event Management)** | A SIEM ingests Windows Event Logs and some application installation events. |
| **Firewall Log Review** | Perimeter firewall logs are available with outbound traffic visibility. |
| **Endpoint Analysis** | The team has tools and procedures for forensic endpoint examination. |

---

## Starting Narrative

*Read the following to the Defenders:*

> Alright everyone, we have a weird one. Yesterday afternoon, one of our accounting staff, Marcus, opened what he thought was an invoice PDF from a vendor we work with regularly. He says the document looked blurry and had a button that said "Open in Adobe" so he clicked it, and his browser downloaded something. He's not sure what happened after that, but his machine seemed fine.
>
> Fast forward to this morning. Our IT service desk noticed something strange: Marcus's workstation is showing two active remote management sessions in our asset management console. One is our standard corporate RMM agent, that's expected. But the second one is a ScreenConnect client connecting to a server we don't recognize. Nobody on the IT team set it up. Nobody authorized it.
>
> About an hour ago, our domain controller event logs started showing a burst of failed authentication attempts against multiple service accounts, all originating from Marcus's workstation IP address. Then the failures stopped and a few succeeded.
>
> We don't know how far this has gone. Your job is to figure out what's installed, what it's talking to, who now has access, and how to stop it, without tipping off the attacker and losing our window.

---

## Incident Captain Guidance

### Why Procedures Succeed or Fail

**Endpoint Security Protection Analysis** - *Can reveal Initial Compromise or Persistence.* This is a tricky one, and the Incident Captain should lean into the difficulty. If the defenders examine the EDR telemetry for Marcus's workstation, they should find the ScreenConnect installation event, but it may not have triggered an alert because ScreenConnect binaries are signed and potentially allowlisted. **The key question is: does the team think to look for *unauthorized* RMM installations, not just malware?** If they ask specifically about new software installations, unsigned MSI packages, or ScreenConnect client deployments connecting to non-corporate relay servers, they can uncover the Initial Compromise. If they look for the second RMM tool (Tactical RMM / MeshAgent), they find the Persistence. If they're only hunting for traditional malware signatures, their EDR shows a clean bill of health which is the whole point of RMM abuse.

**SIEM** - *Can reveal Pivot and Escalate.* If the defenders query for patterns consistent with password spraying (Event ID 4625 — failed logon attempts from a single source IP against many accounts in rapid succession, followed by Event ID 4624 - successful logons), they will clearly see the Internal Password Spray. If they search for the specific service account names that were compromised, they can trace the lateral movement. If they search for generic "malware" or "alert" categories, nothing stands out.

**Firewall Log Review** - *Can reveal C2 and Exfil.* If the defenders inspect outbound HTTPS connections from Marcus's workstation (and subsequently other compromised systems), they may notice connections to unfamiliar domains or IP addresses hosting ScreenConnect relay infrastructure. The domains likely use suspicious TLDs (.top, .xyz, .icu) or newly registered domains. If the defenders check for known C2 indicators or blocklisted IPs only, the traffic passes through cleanly because it's going to a self-hosted ScreenConnect server using standard HTTPS, the protocol looks identical to legitimate RMM traffic.

**Endpoint Analysis** - *Can reveal any card.* Forensic examination of Marcus's workstation is the gold mine. If the defenders analyze browser download history, they'll find the trojanized installer. If they examine running services, they'll find both the ScreenConnect and Tactical RMM services. If they check the registry for ScreenConnect client configuration, they'll find the attacker's relay server URL embedded in the configuration. If they examine PowerShell logs (if ScriptBlock logging is enabled), they can find the password spray commands. The Incident Captain should reward thorough forensic work here.

**Network Threat Hunting / Zeek / RITA Analysis (Other Procedure, no modifier)** - *Can reveal C2 and Exfil or Persistence.* Network flow analysis would reveal beaconing patterns: periodic, consistent HTTPS connections from Marcus's workstation (and later from other systems) to unfamiliar external IPs. If the team is running RITA, the beacon detection algorithm would flag the ScreenConnect relay connections. If they also spot a second beaconing pattern on a different port or to a different IP, that reveals the redundant RMM persistence.

**UEBA (Other Procedure, no modifier)** - *Can reveal Pivot and Escalate.* Behavioral analytics would flag that Marcus's workstation is accessing administrative shares, running PowerShell at unusual hours, and connecting to systems Marcus has never interacted with before.

### Inject Card Guidance

- **Take One Procedure Card Away from the Defenders** - Your RMM vendor pushes an automatic update that temporarily breaks your EDR integration. Remove the Endpoint Security Protection Analysis card.
- **Honeypots Deployed** - One of the service accounts the attacker sprayed against was actually a honeypot account. The attacker just triggered a honeytoken alert, revealing the password spray without needing a die roll. Reveal the Pivot and Escalate card.
- **Bobby the Intern Kills the System You are Reviewing** - Bobby sees the "unauthorized" ScreenConnect on Marcus's machine and helpfully uninstalls it, thinking it's bloatware. The attacker still has the Tactical RMM backdoor but now knows they've been partially discovered and they may accelerate their timeline.

---

## Discussion Points for Debrief

1. **RMM Allowlisting:** Does your organization maintain an allowlist of authorized RMM tools? Would your EDR flag the installation of an unauthorized RMM client, or does it treat all signed RMM binaries as trusted? How quickly can you distinguish a legitimate ScreenConnect deployment from a malicious one?
2. **RMM Inventory and Monitoring:** Do you know how many different RMM tools are actively running in your environment right now? Many organizations discover during incident response that three or four different RMM platforms are in use across departments.
3. **PowerShell Logging:** Is ScriptBlock logging enabled? Are PowerShell transcription logs being collected and forwarded to your SIEM? Without this visibility, the entire password spray phase is invisible.
4. **The "Signed Binary" Trust Problem:** Attackers increasingly use signed, legitimate software as their tooling. How does your detection strategy adapt when the "malware" has a valid code signature?
5. **Redundant Persistence:** The attacker deployed two separate RMM tools. If your team found and removed one, would they keep looking for a second? How does your IR runbook handle the possibility of redundant backdoors?

---

## MITRE ATT&CK Mapping

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| Phish (Trojanized RMM) | T1566.001 (Phishing: Spearphishing Attachment), T1219 (Remote Access Software) |
| Internal Password Spray | T1110.003 (Brute Force: Password Spraying) |
| Malicious Service (Dual RMM) | T1543.003 (Create or Modify System Process: Windows Service), T1219 (Remote Access Software) |
| HTTPS as Exfil (via RMM) | T1041 (Exfiltration Over C2 Channel), T1071.001 (Application Layer Protocol: Web Protocols) |

---

## References and additional info

- Huntress 2025 report on RMM abuse surge https://www.huntress.com/blog/rmm-abuse-when-it-convenience-bites-back
- Microsoft Security Blog on signed malware deploying RMM backdoors (ScreenConnect, Tactical RMM, MeshAgent) https://www.microsoft.com/en-us/security/blog/2026/03/03/signed-malware-impersonating-workplace-apps-deploys-rmm-backdoors/
- CyberProof research on CHAINVERB/UNC5952 ScreenConnect abuse campaigns https://www.cyberproof.com/blog/connectwise-screenconnect-attacks-continued-surge-in-rmm-tool-abuse/
- MITRE ATT&CK Technique T1219 (Remote Access Software) https://attack.mitre.org/techniques/T1219/
- Backdoors & Breaches by Black Hills Information Security - [https://www.backdoorsandbreaches.com](https://www.backdoorsandbreaches.com)
