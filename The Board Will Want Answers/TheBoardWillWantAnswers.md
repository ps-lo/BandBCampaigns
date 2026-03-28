# Backdoors & Breaches Campaign: The Board Will Want Answers
## Author: Steven Lorenz

*It's Monday morning and nothing works. Email is down. The customer database is encrypted. The phone system runs through the same servers. Your 200 employees are standing around wondering what to do. A ransom note on every screen demands $2 million in cryptocurrency within 72 hours. The CEO needs a plan in the next 30 minutes. This scenario is designed for executives, board members, and leadership teams, the people who make decisions when everything goes wrong.*

---

## Who Is This Campaign For?

I wrote this one specifically to tabletop with a client with zero on site IT staff. I've added some of the feedback I recived, like plain language definitions. Updated with newer references to freshen up the talking points.

This campaign is designed for **senior leadership, board members, and the people who will be making crisis decisions** during a real incident. It is deliberately light on technical detail and heavy on the business decisions that matter most:

- Should we pay the ransom?
- Who do we need to notify and and when?
- What do we tell employees, customers, and the press?
- Can we keep operating?
- Who is in charge?

It works well for:

- Board of directors tabletop exercises
- C-suite crisis simulation workshops
- Business continuity and disaster recovery planning sessions
- Insurance and legal readiness discussions
- Any leadership team that hasn't rehearsed a major cyber incident

**No technical security knowledge is required.** If you've ever managed a crisis of any kind, a product recall, a PR issue, a natural disaster, you have the skills to play this game.

---

## Background

Ransomware attacks don't just affect computers, they shut down businesses. In 2025, ransomware groups publicly claimed thousands of successful attacks, and the median ransom demand climbed to nearly $60,000, with demands against larger organizations routinely reaching millions. But the ransom itself is often the smallest cost. Lost revenue during downtime, emergency IT services, legal fees, regulatory fines, customer notification costs, and reputational damage can dwarf the ransom amount many times over.

The hardest part of a ransomware incident isn't the technology. It's the decisions. Leadership teams that have never rehearsed this scenario often freeze, argue, or make impulsive choices that make things worse. This campaign puts your leadership team through those decisions in a safe environment, so the first time isn't the real thing.

---

## The Attack Chain (Simplified Explanations)

| Category | Step | Card | What Happened (Plain Language) |
|---|---|---|---|
| Initial Compromise | **How did the attackers get in?** | **Exploitable External Service** | The company uses a firewall/VPN appliance to allow employees to work remotely. This appliance had a known security vulnerability that the manufacturer issued a fix for two months ago. The IT team hadn't applied the fix yet because it required a maintenance window, and scheduling kept getting pushed back. The attackers used this vulnerability to get into the network without needing anyone's password. |
| Pivot and Escalate | **How did they spread through the network?** | **Internal Password Spray** | Once inside, the attackers tried common passwords against every account in the system. Several employees were using simple passwords like "Summer2025!" and "CompanyName1". Three accounts, including one with administrator access, were compromised. With the admin account, the attackers could reach every system in the company. |
| C2 and Exfil | **How did they steal data and deploy ransomware?** | **HTTP as Exfil** | Before encrypting anything, the attackers spent five days quietly copying sensitive data including customer records, employee HR files, financial statements, and intellectual property to their own servers. Only after the data was safely stolen did they deploy the ransomware that encrypted every server and workstation. Now they have leverage: even if you restore from backups, they still have your data and will publish it unless you pay. |
| Persistence | **How did they make sure they could stay?** | **New User Added** | The attackers created a new user account with administrator privileges, named to blend in with existing IT staff accounts. Even if the original compromised accounts were discovered and disabled, this hidden backdoor account ensures the attackers can get back in. |

---

## Procedure Cards

For an executive audience, redefine the Procedure Cards to reflect business decisions:

| Actual Card Name | Plain Language Name | What It Represents |
|---|---|---|
| **Crisis Management** | **"Activate the Crisis Team"** | Assembling leadership, legal, communications, IT, HR, and external advisors into a coordinated response. |
| **Isolation** | **"Contain the Damage"** | Disconnecting systems, revoking access, and stopping the spread even if it means shutting things down further. |
| **Endpoint Security Protection Analysis** | **"Assess the Scope"** | Understanding which systems are affected, which are still operational, and whether backups are intact. |
| **Firewall Log Review** | **"Find the Entry Point"** | Identifying how the attackers got in so you can close the door and prevent re-entry. |

**Established Procedures** (these get the +3 modifier) Feel free to add additional:
- Crisis Management / "Activate the Crisis Team"
- Endpoint Security Protection Analysis / "Assess the Scope"

**Other Procedures** (no modifier):
- Isolation / "Contain the Damage"
- Firewall Log Review / "Find the Entry Point"
- SIEM / "Search the Security Logs"
- Server Analysis / "Examine the Servers"
- Endpoint Analysis / "Check Individual Computers"
- Internal Segmentation / "Review Network Architecture"
- User and Entity Behavior Analytics / "Look for Compromised Accounts"
- NetFlow / Zeek / RITA Analysis / "Analyze Network Traffic"

---

## Starting Narrative

*Read the following to the Defenders. Set the scene, this is a crisis, and the people at the table are in charge.*

> It's 7:45 AM on a Monday. You're the leadership team of a 200-person professional services firm. You have offices in three cities, about 300 active clients, and your business runs on a client management system, email, a document management platform, and a VoIP phone system, all hosted on servers in your main office.
>
> At 6:12 AM, your IT director called the CEO. Every server is encrypted. Every workstation that was powered on over the weekend is encrypted. The phone system is down. Email is down. The client management system is down. Employees arriving at the office are seeing a red screen with a padlock icon and a message that reads:
>
> *"Your files have been encrypted by NOVA SYNDICATE. You have 72 hours to pay $2,000,000 in Bitcoin. If you do not pay, your client data including personal information, case files, and financial records will be published on our website. Contact us at [Tor address] to negotiate."*
>
> Your IT director has confirmed: the backup server is also encrypted. However, your cloud-based offsite backup from last Tuesday appears to be intact you'd lose about five days of work if you restore from it.
>
> The CEO just gathered all of you in the conference room, which still has working Wi-Fi on a separate consumer internet connection. Here's what she said:
>
> *"We need a plan. Right now. What do we do first? And someone needs to figure out what we tell our employees since there are about forty people standing in their workspace right now staring at locked computers wondering if they still have jobs."*
>
> You have the floor. What do you do?

---

## Incident Captain Guidance

### This Campaign Is About Decisions, Not Detection

Unlike technically focused campaigns, the Incident Captain here should focus on **business decisions and their consequences**. The technical investigation matters, but the primary gameplay should be about:

- **Communication:** Who needs to know, what do you tell them, and when?
- **Business continuity:** How do you keep serving clients today?
- **The ransom question:** Do you pay? Why or why not?
- **Legal and regulatory obligations:** Who must be notified? By when?

### How Actions Succeed or Fail, but keep this one looser than a tech only game.

**"Activate the Crisis Team" (Crisis Management)** - *Can reveal any card, and is the single most important action.* If the defenders organize a structured response by assigning roles (incident commander, communications lead, legal liaison, technical lead), establishing a communication channel that doesn't rely on compromised systems, and setting a regular cadence of briefings, reward this generously. The Incident Captain can reveal whichever card is most relevant to the defenders' current focus.

**What to challenge:** If nobody mentions employee communication within the first two turns, ask: "There are forty confused employees in the work area. Two of them have already texted friends that 'the company got hacked.' A client just called the receptionist's personal cell phone asking why nobody's answering the main line. What are you going to say?"

**"Assess the Scope" (Endpoint Security Protection Analysis)** - *Can reveal How They Stole Data (C2 and Exfil).* If the defenders ask IT to determine what's still working, they'll learn: servers are encrypted, most workstations are encrypted, but the offsite cloud backup appears intact (it was air-gapped from the production network). IT will also report that the ransomware note mentions data exfiltration and preliminary review of the few surviving logs suggests large outbound data transfers occurred over the past five days. **Key decision moment:** "The attackers say they stole client data. Do you believe them? How does this change your decision about paying the ransom?"

**"Contain the Damage" (Isolation)** - *Can reveal How They Spread (Pivot and Escalate).* If the defenders order IT to isolate the network, disconnect internet, shut down remaining servers, revoke all admin credentials, they stop any ongoing attacker access. IT reports that during the shutdown, they found an unknown admin account that nobody recognizes. **Key decision moment:** "Shutting everything down means your employees definitively cannot work today. Your clients cannot reach you. Are you comfortable with that? For how long?"

**"Find the Entry Point" (Firewall Log Review)** - *Can reveal How They Got In (Initial Compromise).* If the defenders ask IT to check how the attackers got in, the VPN appliance vulnerability is identified. The fix was available for two months but wasn't applied. **Key decision moment:** This will likely come up again. "The board is going to ask: why wasn't this patched? Who made that decision? What's your answer?"

### Special Mechanics: The Ransom Decision

At some point during the game (ideally around turn 4–5), the Incident Captain should force the ransom question:

> "Your outside legal counsel just called. They've made contact with the attackers through the Tor portal. The attackers have provided a sample of stolen data as proof, including client names, social security numbers, and financial records from your system. They're offering to reduce the demand to $800,000 if you pay within 24 hours. Your cyber insurance policy has a $500,000 sublimit for ransom payments. The FBI recommends against paying but says it's your decision. What do you do?"

Let the table debate. There's no right answer. The purpose is to rehearse the decision-making process.

---

## Discussion Points for Debrief

These conversations are the real value of this exercise.

1. **The First 60 Minutes:** Did your team know what to do immediately? Was there a plan, or did people talk in circles? **Does your organization have a documented incident response plan that non-technical leaders can follow? Has it been tested?**

2. **Communication Under Pressure:** Who did you think to notify and who did you forget? Employees? Clients? Regulators? Your insurance carrier? The board? **Do you have pre-drafted communication templates for a ransomware scenario? Do you know the regulatory notification deadlines that apply to your industry (e.g., 72 hours under GDPR, state breach notification laws, SEC disclosure rules)?**

3. **To Pay or Not to Pay:** What was the discussion like? What factors influenced the decision? **There's no universally right answer, but the factors include: do you have usable backups, is client data at risk of publication, what does your insurance cover, what are the legal implications of paying, and will the attackers actually provide a decryption key?**

4. **Backups Are Your Lifeline:** The offsite backup survived because it was isolated from the main network. **If your backups are connected to the same network as your production systems, they'll be encrypted too. Do you have truly offline or immutable backups? Have you tested restoring from them recently?**

5. **Patch Management Is a Leadership Issue:** The attackers got in through a vulnerability that had a fix available for two months. The patch wasn't applied because of scheduling delays. **Patching isn't just an IT task, it's a risk management decision. Does leadership have visibility into the organization's patching status for critical systems? Was maintenance deferred by senior leadership?**

6. **Business Continuity:** Could your organization operate for a day without email, phones, and your core business applications? A week? **What's your manual fallback plan? Can employees access anything they need from personal devices or alternative systems?**

---

## MITRE ATT&CK Mapping (For the Technically Curious)

| Attack Step | MITRE ATT&CK Technique |
|---|---|
| Exploitable External Service | T1190 (Exploit Public-Facing Application) |
| Internal Password Spray | T1110.003 (Brute Force: Password Spraying) |
| New User Added | T1136.001 (Create Account: Local Account) |
| HTTP as Exfil + Ransomware | T1486 (Data Encrypted for Impact), T1048 (Exfiltration Over Alternative Protocol) |

---

## References

- FBI IC3 Internet Crime Report https://www.ic3.gov/AnnualReport/Reports/2024_IC3Report.pdf
- Chainalysis 2026 Crypto Crime Report on ransomware payment trends https://www.chainalysis.com/blog/crypto-ransomware-2026/
- CISA "Stop Ransomware" campaign https://www.cisa.gov/stopransomware
- Verizon DBIR https://www.verizon.com/business/resources/reports/2025-dbir-data-breach-investigations-report.pdf
- NIST Cybersecurity Framework incident response guidance https://csrc.nist.gov/projects/incident-response
- Backdoors & Breaches by Black Hills Information Security - [https://www.backdoorsandbreaches.com](https://www.backdoorsandbreaches.com)
