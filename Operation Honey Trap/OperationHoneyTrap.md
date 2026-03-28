# Backdoors & Breaches Campaign: Operation Honey Trap
## Author: Steven Lorenz

I'll be honest, I've never run this campaign. I think it would be hella fun running a multi-team event on one dual-hand gameboard.

*Your organization hired a red team to test your defenses. Three days into the engagement, the red team calls an emergency meeting: while conducting reconnaissance, they found evidence of a REAL attacker already inside your network. This isn't the pentest anymore. This is the real thing and the attacker may know the red team is there. The script just got flipped. Can the defenders, the red team, and leadership work together to hunt a live threat actor without spooking them?*

---

## Who Is This Campaign For?

This campaign is designed for **mixed audiences** and is one of the most unique I've thought up. It works because it creates a dynamic that doesn't exist in any other B&B campaign that I have seen or been a part of: **three parties at the table with different knowledge, different goals, and different constraints.**

It works well for:

- Joint red team / blue team training exercises
- Security teams that want to practice the "what if this were real" transition
- Leadership teams that need to understand how pentests and real incidents differ
- Mixed groups where some players take the "red team" role and others play "defenders"
- Conference workshops and security meetups (this one will likely generate *great* table discussion)

**Gameplay Twist:** Consider splitting the table into two groups, the "Red Team" and "Blue Team / SOC" — who receive slightly different information from the Incident Captain. The red team knows what they did versus what they didn't do. The SOC knows what they're seeing in alerts but can't yet distinguish pentest activity from real attacker activity. Forcing the two groups to communicate and deconflict is the entire point.

---

## Background

Penetration tests are supposed to be controlled exercises. An organization hires a team of security professionals to simulate an attack under agreed-upon rules of engagement, with the goal of finding vulnerabilities before a real attacker does. What happens when the pentest discovers a real attacker is already there?

This scenario is based on a pattern that has occurred in real-world engagements more often than the industry likes to admit. Red teams, conducting authorized intrusion testing, have stumbled upon evidence of prior compromise such as indicators of command-and-control infrastructure, unauthorized accounts, web shells, and data staging that clearly aren't from the pentest. In some cases, the red team's own activity inadvertently alerts the real attacker, who then accelerates their operation or begins covering their tracks.

The result is a chaotic collision of authorized testing and real incident response, where the SOC is simultaneously receiving alerts from the pentest and the real attacker, and nobody is sure which is which.

---

## The Attack Chain

*There are TWO attack chains in play. Two Attack Boards as it were. The Incident Captain manages both.*

My thought here was that a Persistence card exists on both boards, but initially out of play. Once adequate deconfliction occurs to know which attack chain is the authorised event, notify the defenders that the Persistence card is now solveable. 

### The Pentest (Known to the Red Team)

| Category | Card | What the Red Team Did |
|---|---|---|
| **Initial Compromise** | **Phishing** | The red team sent targeted phishing emails to the marketing department as part of the authorized engagement. Two employees clicked. The red team has Cobalt Strike beacons on two marketing workstations. |
| **Pivot and Escalate** | **Weaponizing Active Directory** | From the marketing workstations, the red team used BloodHound to map Active Directory and identified a path to Domain Admin through a misconfigured Group Policy Object. They haven't exploited it yet, they're still in the reconnaissance phase. |
| **C2 and Exfil** | **Living OFF Trusted Sites** | The red team has two beacons active, has mapped AD, and is planning their persistence and Escalation for tomorrow. They have NOT touched any servers, created any accounts, or accessed any databases. |

### The Real Attacker (Unknown to Everyone at Start)

| Category | Card | What Actually Happened |
|---|---|---|
| **Initial Compromise** | **Web Server Compromise** | Six weeks ago, a threat actor exploited a vulnerability in the company's public-facing customer portal (a Java-based web application) and deployed a web shell on the application server. This happened well before the pentest started. Nobody detected it. |
| **Pivot and Escalate** | **Internal Password Spray** | The real attacker used the web shell to conduct a slow, methodical internal password spray over several weeks, with only a few attempts per account per day to avoid lockout thresholds. They compromised a service desk account that has elevated privileges for password resets. |
| **Persistence** | **Logon Scripts** | The attacker added a PowerShell command to a Group Policy logon script that executes on every user login, downloading and running a lightweight reverse shell from an attacker-controlled domain. Every workstation that processes Group Policy is now a potential backdoor. Every time someone logs in, the attacker gets a new connection. |
| **C2 and Exfil** | **HTTPS as Exfil** | The attacker is exfiltrating data from the HR shared drive - employee records, salary information, Social Security numbers over HTTPS to a cloud storage endpoint. They are doing this during business hours at low volume to blend in with normal traffic. |

---

## Procedure Cards

| Actual Card Name | Mixed-Audience Name | What It Represents |
|---|---|---|
| **SIEM** | **"Check the Alert Queue"** | Reviewing the SIEM — which is currently full of alerts from BOTH the authorized pentest and the unknown real attacker. |
| **Endpoint Security Protection Analysis** | **"Review EDR Telemetry"** | Examining endpoint detection data, which shows activity from the red team's Cobalt Strike AND the real attacker's reverse shells. |
| **Crisis Management** | **"Call a Timeout"** | Pausing the pentest, establishing a joint operations channel between the red team and the SOC, and coordinating the transition from test to real incident. |
| **Server Analysis** | **"Examine the Servers the Red Team Didn't Touch"** | Looking at servers and systems that are outside the pentest scope for signs of the real compromise. |


**Other Procedures** (no modifier):
- Endpoint Security Protection Analysis / "Review EDR Telemetry"
- Server Analysis / "Examine the Servers the Red Team Didn't Touch"
- Firewall Log Review / "Sort Out the Network Traffic"
- Network Threat Hunting / Zeek / RITA Analysis / "Find the C2 That Isn't Ours"
- Endpoint Analysis / "Deep Dive on Suspicious Hosts"
- Isolation / "Contain the Real Threat"
- User and Entity Behavior Analytics / "Separate Red Team Activity from Real Attacker"
- Internal Segmentation / "Map the Blast Radius"
- Call a Consultant / "Bust out the credit card and bring in steven, or someone better"

---

## Starting Narrative

*Read the following to the Defenders:*

> OK, everyone. Something unexpected just happened and we need to adjust fast.
>
> As you know, we engaged Technically Secure Security to conduct an authorized red team engagement/penetration test of our corporate network. The engagement started Monday. Today is Thursday. The rules of engagement are documented and signed. The red team's scope includes the corporate network, Active Directory, and email servers are in scope for access but not for destructive actions.
>
> Ten minutes ago, the red team lead, Ava, called our CISO on his personal cell phone, not the normal reporting channel. She said, and I quote: "We need to stop what we're doing and talk. We found something that isn't us."
>
> Ava says that while mapping Active Directory, her team noticed a Group Policy Object that was modified three weeks ago, well before their engagement started. The modification adds a PowerShell download cradle to the user logon script. Her team didn't do this. It's not in their playbook.
>
> She also says they found a web shell on the customer portal server. Her team never touched that server; since it's a production web application they intentionally avoided it. The web shell has a last-modified timestamp from six weeks ago.
>
> Ava is professional and direct. She says: "You have an uninvited guest. They've been here a lot longer than us. We should pause our engagement immediately and figure this out together."
>
> Here's our problem: the SOC alert queue right now is full of alerts. Some are from the red team's authorized Cobalt Strike beacons. Some appear to be from the real attacker's infrastructure. And right now, our SOC analysts cannot tell which is which.
>
> Ava and her team are standing by. What do we do?

---

## Incident Captain Guidance

### The Deconfliction Challenge

This is the core gameplay mechanic that makes this campaign unique. There are two sets of malicious-looking activity in the environment, and the defenders must **sort signal from noise** before they can effectively respond to the real threat. Every action the defenders take needs to account for this dual reality.

The Incident Captain should keep track of which alerts and findings come from the red team and which come from the real attacker. When defenders investigate something, the Incident Captain reveals which source it belongs to.

**Red Team Artifacts (authorized and can be ignored once identified):**
- Cobalt Strike beacons on two marketing workstations
- BloodHound/SharpHound AD enumeration from those same workstations
- Phishing emails to the marketing department
- DNS queries to the red team's C2 domain (which should be in the rules of engagement documentation)

**Real Attacker Artifacts (the actual threat):**
- Web shell on the customer portal server
- Modified GPO logon script with PowerShell download cradle
- Slow password spray from the customer portal server against internal accounts
- Compromised service desk account
- HTTPS exfiltration from the HR shared drive
- Reverse shell connections to an unknown external domain

### How Actions Succeed or Fail

**"Call a Timeout" (Crisis Management)** - *Can reveal any card. This should be the FIRST action.* If the defenders immediately pause the pentest, establish a joint operations channel with Ava's red team, and begin deconflicting artifacts, they dramatically accelerate the investigation. The red team can provide their exact IOCs (C2 domains, beacon IPs, compromised accounts, tools deployed), allowing the SOC to filter those out and focus on what's left, the real attacker. **If the defenders DON'T pause the pentest and try to investigate while it's still running**, the alert queue remains noisy, the SOC wastes time chasing red team artifacts, and the real attacker has more time to operate. Announce: "Your SOC analyst just spent 45 minutes investigating a Cobalt Strike beacon that turned out to be the red team's. Meanwhile, the attacker just pulled another batch of HR records."

**"Check the Alert Queue" (SIEM)** - *Can reveal Pivot and Escalate or C2 and Exfil, but ONLY effectively after deconfliction.* If the red team has provided their IOCs and the defenders filter those out, the remaining SIEM alerts become much more useful. Filtered alerts reveal: the slow password spray against internal accounts (from an IP that isn't the red team's), successful authentication with the compromised service desk account at unusual hours, and HTTPS uploads from an internal system to an unknown cloud endpoint. If the defenders try to use the SIEM without deconflicting first, they're drowning in alerts and can't distinguish signal from noise. Reward teams that deconflict first.

**"Review EDR Telemetry" (Endpoint Security Protection Analysis)** - *Can reveal Persistence.* EDR data shows two categories of suspicious processes: the red team's Cobalt Strike beacons (on the two marketing workstations, known) and PowerShell download cradles executing on dozens of workstations during morning logon (the GPO modification, unknown). Once the red team's activity is excluded, the GPO-based persistence becomes starkly visible. **Key moment:** "The red team has two beacons on two machines. But your EDR shows reverse shells spawning on forty-seven workstations. Those forty-seven are not from the red team."

**"Examine the Servers the Red Team Didn't Touch" (Server Analysis)** - *Can reveal Initial Compromise.* The red team explicitly avoided the customer portal server. If the defenders examine it, they find the web shell, a Java-based backdoor embedded in the web application's directory structure, with a creation date from six weeks ago. The server also shows evidence of the password spray originating from its IP address. **Discussion moment:** "The red team's rules of engagement excluded this server, which is exactly where the real attacker got in. What does that tell us about how we scoped this engagement?"

**"Find the C2 That Isn't Ours" (Network Threat Hunting / Zeek / RITA Analysis, no modifier)** - *Can reveal C2 and Exfil.* Network traffic analysis can identify two separate beaconing patterns: the red team's Cobalt Strike (going to their known C2 domain) and the real attacker's reverse shells (going to a completely different domain). RITA will flag both as beacons, but the red team's IOC list allows the defenders to tag and exclude theirs. The remaining beacon, plus the HTTPS exfiltration stream from the HR file server belongs to the real attacker.

**"Contain the Real Threat" (Isolation, no modifier)** - *Critical, but must be surgical.* If the defenders isolate the customer portal server, disable the compromised service desk account, and remove the malicious GPO logon script, they contain the active threat without disrupting the rest of the investigation. If they blanket isolate everything (including the red team's beacons), they lose their ability to distinguish red team vs. real attacker in the future, and the red team's engagement data is corrupted. **Best practice to reward:** Contain the real attacker's access points first. Then decide whether to resume, restructure, or cancel the pentest.

### Inject Card Guidance

- **Honeypots Deployed** - Turns out the red team deployed a honeytoken as part of their engagement to see if the blue team was actively hunting them. The real attacker triggered it. Reveal one card immediately. The honeytoken confirms the real attacker accessed a specific system.
- **Bobby the Intern Kills the System You are Reviewing** - Bobby, seeing "unauthorized" processes on his workstation (actually the GPO logon script), runs a "fix" he found on Reddit that corrupts the PowerShell transcript logs the forensic team needs.
- **SIEM Analyst Returns from Splunk Training** - Your detection engineer creates a brilliant filter: exclude all IOCs from the red team's engagement document, and surface only alerts from unknown sources. The remaining alerts light up like a Christmas tree. Grant +2 on the next SIEM roll.
- **Take One Procedure Card Away from the Defenders** - The real attacker, observing increased network scanning and investigation activity (which they can see from their compromised service desk account), realizes they've been discovered and begins accelerating exfiltration. Remove one Procedure card to represent the time pressure.

---

## Discussion Points for Debrief

1. **Pentest Deconfliction Planning:** When you hire a red team, do you establish a clear deconfliction process before the engagement begins? **Does the SOC know the red team's C2 domains, source IPs, and target systems? Is there a hotline for the red team to report unexpected findings? This scenario shows why deconfliction isn't optional, It's critical.**

2. **What If the Red Team Hadn't Found It?** Ava's team discovered the real attacker by accident while mapping AD. **If the pentest hadn't happened, how long would the real attacker have remained undetected? What does that say about your continuous monitoring capabilities?**

3. **Alert Fatigue and Noise:** When the SOC's alert queue is full of pentest activity, real alerts get lost. **How do you manage alert volume during a red team engagement? Do you tune your detection rules, or does the SOC just suffer through a noisy week?**

4. **Rules of Engagement and Scope:** The red team avoided the customer portal server because it was a production system. That's exactly where the real attacker was. **How do you scope penetration tests to balance risk (touching production) with coverage (testing everything that matters)? Are your "off limits" systems actually your highest risk?**

5. **The Red Team as an Ally:** In this scenario, the red team transitioned from adversary to ally in seconds. **Does your organization view the red team as a partner or an opponent? Would your SOC be comfortable working alongside them during a real incident?**

6. **Resuming the Pentest:** After the real incident is contained, do you resume the penetration test? **The engagement was disrupted, so do the findings so far still have value? Should the red team re-scope based on what was learned from the real compromise?**

---

## MITRE ATT&CK Mapping

### Real Attacker

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| Web Server Compromise | T1190 (Exploit Public-Facing Application), T1505.003 (Server Software Component: Web Shell) |
| Internal Password Spray | T1110.003 (Brute Force: Password Spraying) |
| Logon Scripts (GPO Modification) | T1547.001 (Boot or Logon Autostart Execution: Registry Run Keys / Startup Folder), T1484.001 (Domain Policy Modification: Group Policy Modification) |
| HTTPS as Exfil | T1567 (Exfiltration Over Web Service), T1041 (Exfiltration Over C2 Channel) |

### Authorized Red Team (For Reference)

| Red Team Activity | MITRE ATT&CK Technique |
|---|---|
| Phishing (Authorized) | T1566.001 (Phishing: Spearphishing Attachment) |
| AD Reconnaissance (Authorized) | T1087.002 (Account Discovery: Domain Account), T1482 (Domain Trust Discovery) |
| Cobalt Strike Beacons (Authorized) | T1219 (Remote Access Software) |

---

*Note: This campaign can optionally be run as a competitive or cooperative Purple Team exercise by splitting the table into two groups (Red Team and SOC) and having the Incident Captain feed each group different information. The Red Team knows what their artifacts look like; the SOC only sees raw alerts. Forcing them to collaborate in real time is where the magic happens.*
