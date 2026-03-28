# Backdoors & Breaches Campaign: Supply & Demand

*A scenario based on the SolarWinds Orion / SUNBURST supply chain compromise. A nation-state adversary (APT29/Cozy Bear) has poisoned a routine software update for the network monitoring platform your organization relies on. The backdoor arrived signed, tested, and delivered through your own patch management process. It has been quietly beaconing from your network for months. 18,000 organizations received the same poisoned update. Can your team detect a backdoor that was designed by one of the most sophisticated threat actors on the planet to be invisible?*

## For in person or at play.backdoorsandbreaches.com. 

I typically use the Core + Expansion 
For those who don't know me, I'm Steven, and I've led over 1000 Backdoors & Breaches sessions at this point. I am trying to write up some of the scenarios to share to hopefully help others leverage Backdoors & Breaches to teach or run tabletops. Some scenarios are based on real events, others are more "well, what if" scenarios. 

**Thank you to the Black Hills Information Security team for making and sharing this game with the community!** I stand on the shoulders of giants and often leverage tools made by others to help make training more enjoyable or impactful. 

Note: Sometimes I modify some rules. Where I do, I try to note that in the scenario. Original rules are here: https://www.blackhillsinfosec.com/tools/backdoorsandbreaches/ 

Play online at https://play.backdoorsandbreaches.com or purchase cards from the Spearphish General Store (https://spearphish-general-store.myshopify.com/collections/backdoors-breaches-incident-response-card-game)

---

## Background

In September 2019, threat actors attributed to Russia's Foreign Intelligence Service (SVR) tracked as APT29, Cozy Bear, UNC2452, and later NOBELIUM, gained access to the internal network of SolarWinds, an IT management software company. Over the following months, they developed and tested a novel code injection tool (later named SUNSPOT) that silently modified the SolarWinds Orion Platform during the software build process.

Beginning in February 2020, the attackers injected a backdoor known as SUNBURST into a legitimate DLL component (`SolarWinds.Orion.Core.BusinessLayer.dll`). Because the backdoor was compiled and digitally signed as part of SolarWinds' official build pipeline, the malicious update was indistinguishable from a legitimate one. Starting in March 2020, SolarWinds distributed the trojanized updates to approximately 18,000 customers worldwide through Orion's standard automatic update mechanism.

Once installed, SUNBURST would lie dormant for approximately two weeks before activating. It then used a sophisticated DNS-based command-and-control protocol, with domain generation algorithm (DGA) subdomains under `avsvmcloud[.]com` designed to look like legitimate cloud API traffic, to beacon out and receive instructions. The attackers were highly selective: of the 18,000 infected organizations, only a small subset were chosen for follow-on intrusion. For those selected targets, the attackers deployed additional malware (TEARDROP, RAINDROP) to deliver customized Cobalt Strike beacons and began hands-on-keyboard operations to access email, steal data, and forge authentication tokens.

The compromise went undetected for approximately nine months. It was discovered on December 8, 2020, when cybersecurity firm FireEye (now Mandiant) disclosed that its own Red Team tools had been stolen — and subsequently traced the intrusion to the SolarWinds supply chain. The attack affected multiple U.S. federal government agencies (Treasury, Commerce, Homeland Security, State, Energy, and others), NATO, the European Parliament, Microsoft, and numerous Fortune 500 companies.

This campaign puts your team in the role of defenders at an organization that installed the trojanized Orion update months ago. The backdoor has been beaconing from your Orion server this entire time. You don't know it yet.

---

## The Attack Chain

| Category | Card | Why This Card |
|---|---|---|
| **Initial Compromise** | **Compromised Trusted Relationship** | The backdoor arrived through a digitally signed software update from SolarWinds, a trusted vendor whose Orion platform is deeply integrated into your network infrastructure. Your IT team installed the update through normal patch management procedures. No phishing, no exploitation, no social engineering — the poison came through the front door, packaged as medicine. |
| **Pivot and Escalate** | **Weaponizing Active Directory** | After the SUNBURST backdoor selected your organization for follow-on activity, the attackers used their access on the Orion server (which has broad network visibility and often holds privileged credentials for monitored systems) to harvest SAML token-signing certificates and forge authentication tokens. This allowed them to impersonate any user — including administrators — across on-premises Active Directory and federated cloud services (Azure AD/M365) without knowing any passwords. They also moved laterally to access email servers and file shares. |
| **C2 and Exfil** | **DNS as Exfil** | SUNBURST communicates with its command-and-control infrastructure using a DNS-based protocol. It generates DGA subdomains under `avsvmcloud[.]com` that encode the victim's domain name, allowing the attackers to identify which organization is calling home and decide whether to engage further. The subdomains were crafted to resemble legitimate AWS API endpoint patterns (e.g., `*.appsync-api.eu-west-1.avsvmcloud[.]com`), making them difficult to distinguish from normal cloud traffic. For selected targets, the C2 channel transitioned to HTTPS connections to dedicated infrastructure for hands-on operations and data exfiltration, including the theft of email content via Microsoft 365 APIs. |
| **Persistence** | **DLL Jijacking** | The SUNBURST backdoor itself IS the persistence mechanism — it's a malicious DLL compiled into the legitimate Orion application. Every time the SolarWinds Orion services start, the backdoor loads. It survives reboots, survives service restarts, and is protected by the application's own digital signature. It was designed to blend seamlessly with normal Orion operations, including anti-analysis checks that prevented it from executing in security researcher sandboxes. Additionally, the attackers deployed secondary backdoors (TEARDROP, RAINDROP, and later SUNSHUTTLE) on high-value targets to ensure persistence even if the primary SUNBURST backdoor was discovered and removed. |


---

## Procedure Cards

Deal the following cards as the **Established Procedures** (these get the +3 modifier):

| Procedure Card | Rationale |
|---|---|
| **SIEM (Security Information and Event Management)** | The organization has a SIEM ingesting Windows Event Logs, DNS query logs, and some cloud audit logs. |
| **NetFlow / Zeek / RITA Analysis** | Network traffic analysis tools are deployed and monitoring internal-to-external traffic. |
| **Firewall Log Review** | Perimeter firewall logs capture outbound DNS and HTTPS traffic. |
| **Server Analysis** | The team has procedures for examining servers, including the SolarWinds Orion server. |

Place the following as **Other Procedures** (no modifier):

- Endpoint Security Protection Analysis
- Endpoint Analysis
- User and Entity Behavior Analytics (UEBA)
- Isolation
- Crisis Management
- Internal Segmentation

---

## Starting Narrative

*Read the following to the Defenders:*

> Team, I need everyone's full attention. What I'm about to share stays in this room until we understand the scope.
>
> Thirty minutes ago, we received an emergency advisory from CISA, the Cybersecurity and Infrastructure Security Agency. They are reporting that SolarWinds, the company that makes the Orion network monitoring platform we use has been the victim of a sophisticated supply chain attack. A threat actor, believed to be a nation-state, injected a backdoor into Orion software updates distributed between March and June of this year.
>
> We checked. Our Orion server is running version 2020.2 HF1. This version is confirmed affected.
>
> The advisory says the backdoor has been active in affected organizations for up to nine months. It communicates using DNS queries to a domain called `avsvmcloud.com`. The advisory recommends immediately checking DNS logs for any queries to that domain.
>
> Our Orion server monitors over 400 systems across our network. It holds SNMP community strings, WMI credentials, and service account passwords for nearly every server and network device in our infrastructure. If this thing is real, and CISA says it ism the attackers may have had access to our most sensitive credentials for months.
>
> We need to determine: Is the backdoor active in our environment? Has the attacker moved beyond the Orion server? What have they accessed? And what's the safe way to respond without tipping them off or destroying evidence?

---

## Incident Captain Guidance

### The Dwell Time Problem

This campaign is unique because the compromise may have been active for **months** before discovery. Unlike other campaigns where defenders are responding to a fresh alert, here they're investigating historical activity and must assume the attacker has had time to establish deep persistence. The Incident Captain should reinforce this tension: every system the Orion server monitored is potentially compromised. Every credential it stored is potentially stolen. The blast radius is enormous and uncertain.

### A Note on Difficulty

This is one of the most challenging campaigns in the collection. SUNBURST was specifically engineered to evade detection, it checked for security tools before activating, used legitimate-looking DNS traffic, and operated within a signed, trusted application. If defenders are struggling, the Incident Captain should provide hints through the narrative ("The CISA advisory just updated, they're recommending checking for these specific indicators...") This one was incredibly fun to share live, and I even bent some rules to give extra turns. You guessed it, this one is also difficult because I wanted specific reasons a card was selected. Extra hints, looser success conditions, be ready to read your room. 

### Why Procedures Succeed or Fail

**SIEM** - *Can reveal C2 and Exfil or Pivot and Escalate.* If the defenders query DNS logs for any queries to `avsvmcloud[.]com` or its subdomains, they will find them originating from the Orion server, dating back months. This confirms the backdoor is active and has been beaconing. If they search for anomalous authentication events, particularly SAML token issuance from unexpected sources, or sign-ins to Microsoft 365 that bypass normal MFA flows, they can find evidence of the token-forging lateral movement. If they search for traditional malware alerts or failed login attempts, nothing appears. SUNBURST doesn't trigger signature-based detections because it's embedded in a signed, legitimate binary, and the attackers used forged tokens that succeed on the first attempt.

**Network Threat Hunting / Zeek / RITA Analysis** - *Can reveal C2 and Exfil or Persistence.* Network traffic analysis is one of the strongest detection paths. If the defenders analyze the Orion server's outbound traffic patterns, RITA's beacon detection can identify the periodic DNS queries to `avsvmcloud[.]com` as a beaconing pattern. If they also look for HTTPS connections from the Orion server to IP addresses not associated with SolarWinds' legitimate infrastructure, they may find the secondary C2 channels used for hands-on operations. If defenders look for unusual internal traffic FROM the Orion server TO other internal systems (especially domain controllers, email servers, and ADFS/federation servers), they can discover the lateral movement. The Orion server legitimately communicates with many systems for monitoring, so the Incident Captain should ask: "Can you distinguish the Orion server's normal monitoring traffic from the attacker's lateral movement? What looks different?"

**Firewall Log Review** - *Can reveal Initial Compromise or C2 and Exfil.* If the defenders check outbound DNS query logs at the perimeter, they can find the `avsvmcloud[.]com` queries and determine when the beaconing started — establishing a timeline for the compromise. If they also look for outbound HTTPS connections from the Orion server to IP addresses outside of SolarWinds' known infrastructure, they can find the exfiltration channel. If they only check for connections to known-bad IPs or blocked traffic, nothing stands out — the C2 infrastructure was purpose-built and wasn't on any threat intelligence blocklist when the campaign was active.

**Server Analysis** - *Can reveal Initial Compromise or Persistence.* If the defenders examine the Orion server itself, checking the installed Orion version, inspecting the `SolarWinds.Orion.Core.BusinessLayer.dll` file hash against the CISA-published indicators of compromise, and reviewing running processes, they can confirm the presence of the SUNBURST DLL. If they check for additional malware on the server (TEARDROP, Cobalt Strike beacons), they may find secondary persistence mechanisms. If they also examine the SAML token-signing certificate on the ADFS server and check for unexpected certificate changes or token issuance events, they can uncover the Weaponizing Active Directory pivot. **Critical nuance:** If the defenders decide to simply uninstall or update Orion immediately, warn them, this destroys forensic evidence on the server and does NOT address secondary persistence or stolen credentials. The attackers likely have other ways in by now.

**UEBA (Other Procedure, no modifier)** - *Can reveal Pivot and Escalate.* Behavioral analytics on service accounts and admin users would show anomalous access patterns: accounts that normally only authenticate during business hours suddenly making API calls to Microsoft 365 at 3 AM, or accounts accessing mailboxes of executives they've never interacted with.

**Endpoint Security Protection Analysis (Other Procedure, no modifier)** - *Limited effectiveness, and the Incident Captain should explain why.* SUNBURST was designed to detect and evade security products. Before activating, it checks for the presence of specific security tools (including CrowdStrike, Carbon Black, FireEye, ESET, and others) and alters its behavior accordingly. If defenders check their EDR for alerts on the Orion server, they'll likely find nothing as the backdoor specifically avoids triggering these tools. This is a powerful teaching moment about the limitations of endpoint detection against nation-state-level tradecraft.

**Isolation (Other Procedure, no modifier)** - *Critical, but complex.* If the defenders isolate the Orion server from the network, they sever the C2 channel but they also lose network monitoring for 400+ systems and alert the attacker that they've been discovered. If the attacker has already established secondary persistence on other systems (which, after months of access, is highly likely), isolating Orion alone doesn't contain the threat. **The right approach to reward:** Isolate the Orion server AND simultaneously rotate all credentials the Orion server had access to (SNMP strings, service account passwords, API keys), revoke SAML token-signing certificates and issue new ones, and hunt for secondary persistence across the environment. This is a massive, coordinated operation, not a simple "unplug it" action.

### Inject Card Guidance (Sometimes I pre-define what an inject means)

- **Data Uploaded to Pastebin** - CISA publishes an updated advisory listing specific government agencies confirmed compromised. Your organization does business with one of them. A congressional staffer calls asking whether your systems were affected and whether any government data transited your network. The incident just became political.
- **SIEM Analyst Returns from Splunk Training** - Your SIEM analyst discovers that DNS query logging was only enabled six weeks ago. Queries to `avsvmcloud[.]com` from before that date are not in the SIEM, the beaconing history is incomplete. You know it started, but you can't see how far back it goes. Grant a +2 on the next SIEM roll, but inform defenders of the gap.
- **Legal Takes Your Only Skilled Handler into a Meeting** - Your CISO is pulled into an emergency board call. The board wants to know: Are we affected? Is customer data at risk? Do we need to disclose? The technical investigation loses its leader for a turn.
- **Bobby the Intern Kills the System You are Reviewing** - Bobby, following the CISA advisory's recommendation to "disconnect affected SolarWinds servers," powers off the Orion server before the forensic team can image the disk or capture memory. Critical volatile evidence is lost. Damnit Bobby!

---

## Discussion Points for Debrief

After the game, facilitate a conversation around these themes. The SolarWinds attack reshaped how the industry thinks about supply chain security.

1. **Trusting Your Vendors' Build Pipelines:** The update was digitally signed by SolarWinds. Your patch management process installed it exactly as designed. Every best practice said "keep your software updated." And doing so is exactly what installed the backdoor. **How do you reconcile "patch everything" with "your patch might be poisoned"? What does defense in depth look like when the supply chain itself is compromised?**

2. **The Monitoring Platform Paradox:** SolarWinds Orion is a *network monitoring tool* designed to have visibility into everything. That makes it the perfect target. If your monitoring infrastructure is compromised, who monitors the monitor? **What privileged access does your network monitoring platform have? Are those credentials scoped to minimum necessary access? Would you detect if your monitoring server started behaving abnormally?**

3. **Dwell Time and Detection Gaps:** The attackers had access for up to nine months before discovery. Not a single security product detected SUNBURST during that time. **What is your average time to detect an intrusion? Do you rely primarily on signature-based detection (which SUNBURST evaded), or do you also hunt for anomalous behavior? Do you have the DNS query logs, NetFlow data, and authentication logs needed to investigate months-old activity?**

4. **SAML Token Forgery and Identity Trust:** The attackers forged SAML authentication tokens, allowing them to impersonate any user across on-premises and cloud environments. Resetting passwords alone doesn't help as the attacker doesn't need passwords when they can mint their own tokens. **Does your team understand SAML/OAuth token flows well enough to detect and respond to token forgery? Would you know if your ADFS token-signing certificate was compromised?**

5. **Coordinated Response at Scale:** Remediating this attack isn't just "remove the malware." It requires rotating every credential the Orion server touched, revoking and reissuing SAML certificates, hunting for secondary persistence across the entire environment, and rebuilding trust in the identity infrastructure. **Does your organization have the resources and expertise to execute a remediation of this scope? Would you need to bring in external incident response support?**

6. **Software Supply Chain Security Going Forward:** After SolarWinds, the industry accelerated efforts around software bills of materials (SBOMs), build provenance (SLSA framework), and Zero Trust for software updates. **Has your organization changed how it evaluates software vendor security? Do you require SBOMs from critical vendors? Do you have visibility into your own software supply chain?**

---

## MITRE ATT&CK Mapping

This campaign maps directly to MITRE ATT&CK Campaign [C0024: SolarWinds Compromise](https://attack.mitre.org/campaigns/C0024/).

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| Trusted Relationship (Supply Chain) | T1195.002 (Supply Chain Compromise: Compromise Software Supply Chain), T1199 (Trusted Relationship) |
| Weaponizing Active Directory (Token Forgery) | T1606.002 (Forge Web Credentials: SAML Tokens), T1550.001 (Use Alternate Authentication Material: Application Access Token), T1078.004 (Valid Accounts: Cloud Accounts) |
| DLL Attacks (SUNBURST + Secondary Backdoors) | T1574.002 (Hijack Execution Flow: DLL Side-Loading), T1027.009 (Obfuscated Files or Information: Embedded Payloads) |
| DNS as Exfil (DGA C2 + HTTPS Exfil) | T1071.004 (Application Layer Protocol: DNS), T1568.002 (Dynamic Resolution: Domain Generation Algorithms), T1048 (Exfiltration Over Alternative Protocol) |

### Additional MITRE Techniques Used in the Real Attack
- T1518.001 (Software Discovery: Security Software Discovery) - SUNBURST checked for security tools before activating
- T1497 (Virtualization/Sandbox Evasion) - SUNBURST detected analysis environments
- T1070.006 (Indicator Removal: Timestomping) - Attackers modified file timestamps
- T1114.002 (Email Collection: Remote Email Collection) - Attackers accessed M365 mailboxes
- T1213 (Data from Information Repositories) - Attackers accessed SharePoint and internal documentation

---

## References and extra info

- FireEye/Mandiant discovery and initial SUNBURST analysis https://cloud.google.com/blog/topics/threat-intelligence/sunburst-additional-technical-details
- CrowdStrike analysis of SUNSPOT build-injection tool https://www.crowdstrike.com/en-us/blog/sunspot-malware-technical-analysis/
- Symantec discovery of RAINDROP secondary malware https://www.security.com/threat-intelligence/solarwinds-raindrop-malware
- Microsoft analysis of Solorigate second-stage activation (TEARDROP, RAINDROP) https://www.microsoft.com/en-us/security/blog/2021/01/20/deep-dive-into-the-solorigate-second-stage-activation-from-sunburst-to-teardrop-and-raindrop/
- CISA Emergency Directive 21-01: Mitigate SolarWinds Orion Code Compromise https://www.cisa.gov/news-events/news/cisa-issues-emergency-directive-mitigate-compromise-solarwinds-orion-network-management-products https://www.cisa.gov/emergency-directive-21-01-older-supplemental-guidance
- US/UK government attribution of SolarWinds compromise to SVR/APT29 https://www.ncsc.gov.uk/news/uk-and-us-call-out-russia-for-solarwinds-compromise
- Palo Alto Networks Unit 42 SolarStorm timeline https://unit42.paloaltonetworks.com/solarstorm-supply-chain-attack-timeline/
- MITRE ATT&CK Campaign C0024: SolarWinds Compromise https://attack.mitre.org/campaigns/C0024/
- Backdoors & Breaches by Black Hills Information Security — [https://www.backdoorsandbreaches.com](https://www.backdoorsandbreaches.com)

---

