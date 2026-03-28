# Backdoors & Breaches Campaign: Scattered in the Aisles
## Author: Steven Lorenz

> A note on hard mode: Sometimes when I'm using backdoors and breaches as a teaching tool, I like to get the defenders to try to be more specific with why they chose a procedure. "Hard mode" adds a little specificity to a procedure to reimforce knowledge and learning potential. Can also just be discussion points about a procedure. 

*A scenario based on the 2025 Scattered Spider retail attacks. Your organization's third-party service provider has been compromised, and the attackers are using social engineering to pivot deep into retail networks. Can your team detect a supply chain intrusion before customer data is exfiltrated and ransomware is deployed?*

---

## Background
In mid-2025, a coordinated campaign attributed to the Scattered Spider threat group targeted multiple major retailers through a shared third-party service provider. The attackers used sophisticated social engineering, including vishing (voice phishing) of IT help desk staff to obtain valid credentials and MFA bypass codes. From that initial foothold in the service provider's environment, they pivoted laterally into client retail networks, deployed ransomware payloads, and exfiltrated millions of customer records before issuing extortion demands.
This campaign illustrates the growing danger of third-party supply chain compromise combined with human-targeted social engineering. The attackers did not need a zero-day exploit. They needed a convincing phone call.

---

## The Attack Chain

| Category | Card | Why This Card |
|---|---|---|
| **Initial Compromise** | **Social Engineering** | Attackers called the help desk of a third-party provider, impersonating an employee, and convinced staff to reset credentials and provide MFA enrollment codes. |
| **Pivot and Escalate** | **Weaponizing Active Directory** | Once inside the provider's network with valid admin level credentials, the attackers enumerated Active Directory trusts between the provider and retail client environments, then used those trust relationships to move laterally into production retail networks. |
| **Persistence** | **New User Added** | The attackers created new privileged accounts across multiple domains and cloud tenants to ensure they could regain access even if the original compromised credentials were revoked. |
| **C2 and Exfil** | **HTTPS as Exfil** | Customer PII databases and internal documents were staged and exfiltrated over encrypted HTTPS connections to attacker-controlled cloud storage, blending in with normal outbound web traffic. |

---

## Procedure Cards

Deal the following cards as the **Established Procedures** (these get the +3 modifier):

| Procedure Card | Rationale |
|---|---|
| **Crisis Management** | While it isn't printed on the card, nothing says we can't modify the card to the scenario |
| **Firewall Log Review** | Perimeter firewalls log all outbound traffic and are reviewed on a schedule. |
| **SIEM (Security Information and Event Management)** | A SIEM is deployed and ingesting logs from key sources. |
| **Endpoint Security Protection Analysis** | EDR/AV solutions are deployed on endpoints across the retail environment. |

Place the following as **Other Procedures** (no modifier):

- Server Analysis
- Endpoint Analysis
- NetFlow / Zeek / RITA Analysis
- User and Entity Behavior Analytics (UEBA)
- Isolation
- Internal Segmentation

---

## Starting Narrative

*Read the following to the Defenders:*
> Good morning, team. We've got a situation developing. About forty-five minutes ago, our Security Operations Center received an automated alert: a new Global Administrator account was created in our Azure AD tenant. Nobody on our IAM team made that change, and the account name doesn't match our naming conventions.
> At roughly the same time, we started getting reports from store managers that the point-of-sale reporting dashboard is running unusually slow, and two overnight batch jobs that sync customer loyalty data failed with authentication errors.
> Our third-party managed services provider, "PeakPoint Solutions," handles a portion of our identity infrastructure. We reached out to them, and they say they're "looking into some anomalous activity on their end as well" but haven't provided details.
> Your job is to figure out what's going on. Was this an internal mistake? A misconfiguration? Or is something much worse happening? The clock is ticking.

---

## Incident Captain Guidance

### Why Procedures Succeed or Fail
**SIEM** - *Can reveal Initial Compromise or Persistence.* **Hard mode:** If the defenders say they are looking for unusual account creation events, authentication anomalies from the third-party provider's IP ranges, or impossible-travel logins, the SIEM could surface the Social Engineering compromise (new sessions from unusual locations using reset credentials) or the New User Added persistence (event ID 4720 / Azure AD audit logs showing the rogue account creation). If they are searching for something unrelated, such as malware signatures, explain that the SIEM returns nothing unusual in that category because the attackers used legitimate credentials — no malware was involved in the initial stages.

**Firewall Log Review** - *Can reveal C2 and Exfil.* **Hard mode:** If the defenders say they are reviewing outbound traffic for large data transfers, unusual destinations, or connections to newly registered domains, they could discover the HTTPS exfiltration streams. The data was sent to cloud storage endpoints (e.g., blob storage URLs) that wouldn't be on any blocklist but would show abnormally large upload volumes during off-hours. If they're just looking for blocked connections or known bad IPs, nothing stands outm the traffic was encrypted, outbound on port 443, and went to legitimate cloud infrastructure.

**Crisis Management** - **Hard Mode:** If the defenders invoke Crisis Management to coordinate with PeakPoint Solutions and demand a joint investigation call, PeakPoint may disclose that one of their technicians received a suspicious call and reset an account. This directly points to the Social Engineering initial compromise. If the defenders use Crisis Management only for internal communication and don't engage the third party, it helps organize the response but doesn't reveal a card on its own.

**Endpoint Security Protection Analysis** - *Can reveal Persistence.* **Hard mode:** If the defenders check EDR telemetry for newly created local admin accounts or unusual RDP sessions originating from the service provider's jump hosts, they can find evidence of the New User Added persistence. If they're only looking for malware detections or suspicious process trees, the EDR is quiet — the attackers operated entirely through legitimate remote access tools and native Windows administration.

**UEBA (Other Procedure, no modifier)** - *Can reveal Initial Compromise or Pivot and Escalate.* **Hard mode:** Behavioral analytics would flag the anomalous login patterns: a service account that normally authenticates from a narrow IP range suddenly logging in from a VPN exit node, or an account accessing dozens of systems it has never touched before. This is the strongest path to uncovering the Active Directory lateral movement.

**Isolation (Other Procedure, no modifier)** — *Can reveal Pivot and Escalate.* **Hard mode:** If the defenders decide to sever the VPN tunnel or federation trust to PeakPoint Solutions, they may notice the attacker's sessions drop, confirming the third-party vector. However, this is a high-impact action that could disrupt legitimate business operations, and the Incident Captain should make the defenders justify this decision before rewarding it.

### Inject Card Guidance for alternative start point

- **Data Uploaded to Pastebin** - A threat intelligence feed flags that a sample of 10,000 customer records from your loyalty program just appeared on a paste site, confirming exfiltration has already occurred and escalating urgency.

---

## Discussion Points for Debrief

After the game, facilitate a conversation around these themes:

1. **Third-Party Risk Management:** Does your organization have visibility into how your managed service providers access your environment? Do you monitor their authentication activity with the same rigor as internal users?
2. **Help Desk Social Engineering:** What controls are in place to verify the identity of callers requesting credential resets or MFA re-enrollment? Would a vishing attack succeed against your help desk today?
3. **Identity is the New Perimeter:** The attackers never exploited a software vulnerability. They abused trust relationships and valid credentials. How does your detection strategy account for "living off the land" attacks that use legitimate tools and accounts?
4. **Supply Chain Blast Radius:** If a single vendor is compromised, how many of your systems are exposed? Do you have the ability to rapidly revoke third-party access without taking down critical business services?

---

## MITRE ATT&CK Mapping

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| Social Engineering | T1566 (Phishing), T1598 (Phishing for Information) - adapted to voice/vishing |
| Weaponizing Active Directory | T1482 (Domain Trust Discovery), T1078 (Valid Accounts) |
| New User Added | T1136.001 (Create Account: Local Account), T1136.003 (Create Account: Cloud Account) |
| HTTPS as Exfil | T1048.002 (Exfiltration Over Alternative Protocol: Asymmetric Encrypted Non-C2 Protocol), T1567 (Exfiltration Over Web Service) |

---

## References

- Scattered Spider retail campaign targeting UK retailers (M&S, Co-op, Harrods), June–July 2025
- FBI Flash Alert on Scattered Spider targeting airline and retail sectors, June 2025
- MITRE ATT&CK Group: Scattered Spider (G1015)
- CISA Advisory on Social Engineering and Help Desk Targeting, 2024–2025
- Backdoors & Breaches by Black Hills Information Security — [https://www.backdoorsandbreaches.com](https://www.backdoorsandbreaches.com)
