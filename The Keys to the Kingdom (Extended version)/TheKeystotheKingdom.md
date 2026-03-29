# Backdoors & Breaches Campaign: The Keys to the Kingdom
Author: Steven Lorenz

*A full Active Directory domain compromise. Starting with a phished user to Golden Ticket persistence. The attacker abused a misconfigured AD CS certificate template to escalate privileges, performed a DCSync to steal the KRBTGT hash, forged a Golden Ticket for unlimited domain access, and modified the AdminSDHolder object to ensure their privileges survive any cleanup attempt. Your identity infrastructure has been turned inside out.*

*This campaign is written as both a playable scenario and a learning guide. Every detection path is explained in depth so that players understand not just WHAT to do, but WHY it works. This scenario was written as an opener for an Active Directory defense workshop and took about an hour to work through and I've modified the scenario into a standalone playable B&B scenario.*

**Required Decks:** Core Deck + Trimarc Expansion Deck

---

## Who Is This Campaign For?

This campaign is designed for **students, early career security professionals, pr anyone interested in learning Active Directory security** through hands-on tabletop play. The whole point is to *learn* through the experience.

If you've never heard of Kerberoasting, AdminSDHolder, or ESC1 before today, that's OK. This guide will teach you what they are, why they matter, and how to detect them all within the structure of a Backdoors & Breaches game.

The Incident Captain should use this campaign as a **teaching tool**. When defenders propose an action, don't just say "it works" or "it doesn't", I added some basic explainers for *why* it worked. Take time to teach with the game as a guide.

---

## Key Concepts You'll Need (A Quick Primer)

Before play, the Incident Captain can read these aloud or share them with the table.

### What Is Active Directory?
Active Directory (AD) is Microsoft's directory service managing who can log in to what in most corporate Windows environments. When you type your username and password at work, AD checks those credentials and determines what you're allowed to access.

### What Is Kerberos?
Kerberos is the authentication protocol AD uses. Instead of sending your password to every server, Kerberos gives you a "ticket" (a TGT or Ticket Granting Ticket) that proves your identity. You present this ticket to access resources. The **KRBTGT account** is the special account whose password encrypts and signs all tickets. If an attacker gets the KRBTGT hash, they can forge their own tickets, that is known as a **Golden Ticket** attack.

### What Is AD CS (Active Directory Certificate Services)?
AD CS is Microsoft's built in system for issuing digital certificates used for Wi-Fi authentication, VPN access, email encryption, and smart card login. AD CS uses **certificate templates** that define what kind of certificate can be issued and who can request one. If a template is misconfigured, an attacker can request a certificate that lets them impersonate any user including Domain Admin.

### What Is AdminSDHolder?
AdminSDHolder is a special container in AD that acts as a security template for all "protected groups" (Domain Admins, Enterprise Admins, etc.). Every 60 minutes, a background process called **SDProp** copies the permissions from AdminSDHolder onto every protected group. This is meant to prevent accidental permission changes. But if an attacker modifies AdminSDHolder, SDProp becomes a persistence mechanism as it automatically re-applies the attacker's permissions every hour, even if a defender removes them manually.

### What Is a Golden Ticket?
A Golden Ticket is a forged Kerberos TGT. Normally, only the domain controller can issue TGTs (using the KRBTGT hash). But if an attacker steals the KRBTGT hash, they can create their own TGTs for any user, with any permissions, with any expiration date. A Golden Ticket is the master key to the entire domain. The only way to invalidate it is to reset the KRBTGT password **twice*

---

## The Attack Chain

| Category | Card (Deck) | What Happened |
|---|---|---|
| **Initial Compromise** | **Phishing** *(Core Deck)* | An HR coordinator named Maria clicked a link in a spearphishing email disguised as a benefits enrollment update. The link opened a Word document with a malicious macro. When Maria clicked "Enable Content," the macro ran a hidden PowerShell command that downloaded an attacker tool called Cobalt Strike directly into memory. No files on disk here, the attack lives entirely in RAM, invisible to traditional antivirus. The attacker now has a remote session on Maria's workstation. |
| **Pivot and Escalate** | **Misconfigured Certificate Templates (ESC1)** *(Trimarc Expansion)* | The attacker runs Certipy to scan for misconfigured certificate templates. They find `UserAuthCert` with a dangerous setting: it lets the requester choose *who the certificate is for* (the Subject Alternative Name), and *any domain user* can enroll. The attacker requests a certificate with the SAN set to `administrator@corp.local`. The CA issues it. The attacker authenticates as Domain Admin using certificate-based Kerberos (PKINIT). **Standard user to Domain Admin in one certificate request.** They then perform a DCSync to steal the KRBTGT hash, enabling Golden Ticket forging. |
| **Persistence** | **AdminSDHolder Rights Modification** *(Trimarc Expansion)* | The attacker modifies the ACL on AdminSDHolder, adding full-control permissions for a dormant service account they've taken over (`svc-monitoring`). Every 60 minutes, SDProp copies this permission to Domain Admins, Enterprise Admins, and every other protected group. Even if a defender removes the attacker's access from Domain Admins, SDProp silently restores it within the hour. The attacker has programmed AD to re-hack itself on a timer. |
| **C2 and Exfil** | **Living Off Trusted Sites (LOTS)** *(Trimarc Expansion)* | Commands are posted to a private GitHub Gist. Stolen data from the NTDS.dit dump (every password hash in the domain), executive email exports, HR compensation files is uploaded to Azure Blob Storage. All traffic goes to github.com and blob.core.windows.net over HTTPS. The attacker's traffic is invisible in normal cloud traffic. |

---

## Procedure Cards

**Established Procedures** (+3 modifier) pick some or all of these:

| Procedure Card | What It Represents |
|---|---|
| **SIEM** *(Core)* | Collects and correlates logs from domain controllers, the CA, endpoints, and network devices. |
| **Server Analysis** *(Core)* | Direct examination of server configurations, AD objects, certificate templates, and running processes. |
| **Firewall Log Review** *(Core)* | Reviewing perimeter and internal firewall logs for suspicious outbound connections and traffic patterns. |
| **UEBA** *(Core)* | Behavioral analytics that baseline "normal" and alert when users or machines deviate from their patterns. |

---

## Starting Narrative

*Read the following to the Defenders:*

> Team, we've got a developing situation that just escalated from "suspicious" to "hair on fire."
>
> Yesterday afternoon, our SOC analyst noticed an unusual Kerberos event on DC01: a TGT was issued for the built in Administrator account using *certificate based authentication*, not a password. The Administrator account hasn't been used interactively in months. Its password is stored in a sealed envelope in the company safe. Nobody opened that envelope. Nobody typed that password. But someone just became Administrator.
>
> Our analyst traced the certificate back to our internal CA. It was enrolled two hours before the TGT, using a template called `UserAuthCert`. The enrollment request came from an HR workstation: `WS-HR-023`, assigned to Maria Gonzalez.
>
> Maria remembers clicking a link in an email about benefits enrollment yesterday morning. She thought it was legitimate.
>
> Then things got worse. Our AD team checked the AdminSDHolder object and found a permission entry that wasn't there last quarter: an account called `svc-monitoring` has full control. That account was supposed to be decommissioned six months ago. It's still active.
>
> If AdminSDHolder has been modified, the SDProp process has been silently propagating the attacker's permissions to every protected group in the domain.
>
> We need to figure out: What happened on Maria's workstation? How did someone get a certificate as Administrator? What is `svc-monitoring` doing? And how do we remediate without the attacker walking back in through a Golden Ticket?

---

## Incident Captain Guidance: Detection Deep Dive

For each card, every procedure that can reveal it is explained with **what to look for** and **why it works**. Use these explanations when defenders propose actions.

---

### INITIAL COMPROMISE: Phishing

*Maria clicked a phishing link, a macro executed PowerShell, and a Cobalt Strike beacon loaded into memory.*

**Can be solved by: SIEM, Server Analysis, Cloud Event Log Analysis, or UEBA**

---

#### SIEM

**Why it works:** Even though Cobalt Strike runs in memory with no files on disk, the *process that launched it* leaves log traces.

**What to look for:**
- **Event ID 4688 (Process Creation):** Shows `WINWORD.EXE` spawning `powershell.exe`. Word doesn't normally launch PowerShell. This parent child process relationship is a classic malicious macro indicator.
- **Event ID 4104 (PowerShell ScriptBlock Logging):** If enabled, records the exact PowerShell commands the macro executed including the download URL for Cobalt Strike.
- **AV/Defender events:** Even "fileless" attacks can trigger behavioral detections when PowerShell downloads and executes content from the internet.

**If the dice roll failed, then suggest they search for the wrong thing:** "You searched for failed logins and firewall blocks seeing nothing unusual. Maria's login was legitimate. The phishing attack didn't involve a failed login. Think about what *process activity* on Maria's machine would look different from a normal day."

---

#### Server Analysis

**Why it works:** The email server or email security gateway processed the phishing email. Examining it reveals the attack's entry point.

**What to look for:**
- **Mail server / gateway logs:** The original phishing email metadata, sender, subject, URL, attachments is logged. This confirms the vector and provides IOCs.
- **Message trace:** Search for the same phishing email sent to other employees. Maria might not be the only target, maybe she was just the only one who clicked.

**If the dice roll failed, then state they only checked domain controllers:** "DCs look normal as the phish happened on a workstation. But think about what *other servers* have evidence of how the email arrived. Where do emails go before reaching Maria's inbox?"

---

#### Cloud Event Log Analysis

**Why it works:** If the organization uses Microsoft 365, cloud audit logs provide additional evidence of email delivery and user interaction.

**What to look for:**
- **M365 Unified Audit Log/Message Trace:** When the email was received, if Maria clicked the link (if Safe Links is enabled), and whether automated scans flagged it.
- **Defender for Office 365 alerts:** Post delivery detections that may not have been acted on.

**If the dice roll failed, then cloud logs aren't available:** "Your M365 admin says the Unified Audit Log was never enabled. That's a gap to note for the debrief."

---

#### UEBA 

**Why it works:** UEBA baselines "normal" behavior per user and machine, then alerts on deviations. Maria's workstation suddenly doing new things is exactly what UEBA catches.

**What to look for:**
- **Process anomaly:** Maria's machine has never run PowerShell before (she's in HR). First ever PowerShell execution = behavioral anomaly.
- **Network anomaly:** Her workstation is connecting to an IP/domain it has never contacted (the Cobalt Strike team server).
- **Application anomaly:** `WINWORD.EXE` spawning `powershell.exe` is abnormal for any user. UEBA flags this never-before-seen parent-child relationship.

**If the dice roll failed:** "What specifically are you asking UEBA to look for? What kind of deviation would you expect if malicious code just ran on an HR workstation?"

---

### PIVOT AND ESCALATE: Misconfigured Certificate Templates (ESC1)

*Attacker used a misconfigured AD CS template to get a certificate as Domain Admin, authenticated via PKINIT, performed DCSync, and obtained KRBTGT hash.*

**Can be solved by: SIEM, Cyber Deception, Server Analysis, or Permission Audit**

---

#### SIEM

**Why it works:** Certificate enrollment and Kerberos authentication generate specific Event IDs that tell the ESC1 story step by step.

**What to look for:**
- **Event ID 4886 (Certificate request received)** on the CA: Logs *who* requested a cert, *from which machine*, and *which template*. You'll find a request from Maria's workstation for `UserAuthCert` with a SAN of `administrator@corp.local`. That's the smoking gun, why would Maria's machine asked for a Domain Admin certificate?
- **Event ID 4887 (Certificate issued):** Confirms the CA approved and issued it. The CA issued a Domain Admin certificate because the template allowed it.
- **Event ID 4768 (TGT requested)** on the DC: Shows PKINIT authentication for the Administrator account. The key detail: **pre-authentication type 16** (certificate-based) instead of the normal **type 2** (password). If Administrator normally uses a password and suddenly appears with PKINIT, that's anomalous.
- **Event ID 4662 (Directory operation):** Catches the DCSync. Logs replication requests from accounts that aren't domain controllers, a DCSync indicator.

**If the dice roll failed, then they searched for failed logins:** "No failed logins. The attacker never guessed a password because they used a *certificate* and it worked on the first try. What events fire when certificates are issued or Kerberos tickets are created?"

---

#### Cyber Deception

**Why it works:** Honeytokens and decoys planted in AD catch attackers during their reconnaissance phase before they find the real vulnerability.

**What to look for:**
- **Honeytoken service account with a Kerberos SPN:** If a decoy account with a weak password and SPN was planted, the attacker likely Kerberoasted it during initial enumeration (before discovering ESC1). The Kerberoasting attempt triggers an alert, catching the attacker during recon even though they used a different escalation path.
- **Decoy certificate template:** An advanced deception strategy might include a monitored fake template that looks misconfigured. Any enrollment attempt is definitively malicious.
- **Honeytoken in privileged groups:** A fake account in Domain Admins that the attacker tries to use after escalation.

**If the dice roll failed, deception isn't deployed:** "You don't have honeytokens in AD. Note that for the debrief as even a few decoy accounts with SPNs are one of the cheapest, highest-fidelity detection strategies available. A legitimate user will *never* interact with a honeytoken, so any activity against one is definitively suspicious."

---

#### Server Analysis

**Why it works:** Direct examination of the CA server and AD configuration reveals both the misconfiguration (root cause) and the evidence of exploitation.

**What to look for:**
- **Certificate template settings:** Examine `UserAuthCert` using `certtmpl.msc`, Certify, PSPKIAudit, or Certipy. The dangerous settings: `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT` is enabled (requester specifies the SAN), and enrollment permissions include `Authenticated Users` (anyone can request). This IS the ESC1 vulnerability.
- **Issued certificates log:** The CA database (via `certsrv.msc` or `certutil`) shows the certificate issued to `administrator@corp.local` from Maria's workstation. The timestamp aligns with the attack.
- **AD object inspection:** Check `svc-monitoring` properties: creation date, last logon, group memberships. Is it being used as a backdoor?

**If the dice roll failed, then they only check the domain controller:** "The DC shows authentication events, but the *certificate* was issued by a different server, the Enterprise CA. Which server runs your CA? Have you ever examined its certificate templates?"

---

#### Permission Audit

**Why it works:** Auditing AD CS enrollment permissions cuts straight to the root cause, **who can request which certificates**.

**What to look for:**
- **Template enrollment permissions:** Who has "Enroll" and "Autoenroll" rights on each template. `UserAuthCert` grants enrollment to `Authenticated Users` which is far too broad. Should be restricted to specific security groups.
- **CA security permissions:** Who has "Issue and Manage Certificates" and "Request Certificates" on the CA itself.
- **BloodHound AD CS analysis:** BloodHound (with Certipy data) visually maps all certificate based attack paths, showing which users can escalate through which templates. This is the most powerful way to find ESC1-ESC8 vulnerabilities.

**If the dice roll failed:** "Think of the certificate template as a form anyone can fill out to request a VIP badge. The 'permission audit' checks *who is allowed to fill out that form*. If the answer is 'literally everyone,' that's a problem."

---

### C2 AND EXFIL: Living Off Trusted Sites (LOTS)

*Commands via GitHub Gists. Data exfiltrated to Azure Blob Storage. All over HTTPS to allowlisted domains.*

**Can be solved by: Network Threat Hunting, SIEM, Firewall Log Review, or UEBA**

---

#### Network Threat Hunting

**Why it works:** Goes beyond automated alerts because an analyst *actively searches* for anomalous patterns when traffic goes to legitimate domains.

**What to look for:**
- **Unusual source-destination pairs:** Maria's HR workstation making regular connections to `gist.githubusercontent.com`. GitHub is normal for developers but not for HR workstations. "Which machines are talking to GitHub?" instantly flags it.
- **Data volume anomalies:** Multi-gigabyte uploads to Azure Blob Storage from a single workstation or DC. Even to a legitimate domain, the *volume* is abnormal.
- **Beaconing patterns:** The beacon checks the GitHub Gist on a regular interval (e.g., every 60 seconds). RITA or similar tools flag this periodic timing pattern as C2 beaconing regardless of the destination's reputation.

**If the dice roll failed, then maybe they just check a blocklist:** "GitHub and Azure are categorized as 'Cloud Services' both allowed. The domains are legitimate. The attacker is hiding in YOUR trusted infrastructure. How else could you spot this?"

---

#### SIEM

**Why it works:** Correlates web proxy logs, DNS logs, and endpoint identity to surface anomalies invisible in isolation.

**What to look for:**
- **Web proxy + endpoint correlation:** SIEM links proxy logs (destination URL) with source machine identity. `WS-HR-023` repeatedly accessing `gist.githubusercontent.com` is anomalous for an HR workstation.
- **DNS query patterns:** Unusual number of queries to GitHub and Azure subdomains with regular polling intervals which is characteristic of C2 check-ins.
- **Outbound data volume:** Proxy or firewall byte counts show multi-gigabyte uploads to cloud storage endpoints from machines that normally transfer kilobytes.

**If the dice roll failed**, then use my catch phrase "just because your team is good at something, doesn't mean it works every time" or some other reason for the failure.

---

#### Firewall Log Review

**Why it works:** Firewall logs capture connection metadata: source, destination, bytes transferred, duration, frequency even for encrypted traffic.

**What to look for:**
- **Upload volume by host:** Maria's workstation uploaded 4.7 GB to `blob.core.windows.net`. HR workstations don't normally upload gigabytes to Azure. The volume, not the destination, is the indicator.
- **Connection frequency:** Dozens of short connections to `gist.githubusercontent.com` at regular intervals from a non-developer workstation.
- **Unusual source-destination pairs:** Domain controllers making outbound data uploads to Azure Blob Storage. DCs should never do this.

**If the dice roll failed, then they only check blocked connections:** "Everything was allowed, traffic went to legitimate domains on port 443. The firewall did its job per its rules. Now think about what you can learn from *allowed* connections. The blocked list won't help here."

---

#### UEBA

**Why it works:** Compares current behavior against historical baselines. An HR workstation talking to developer infrastructure deviates dramatically.

**What to look for:**
- **New destination:** `WS-HR-023` has never connected to GitHub or Azure Blob Storage. UEBA flags these as first seen destinations that don't match the machine's role.
- **Data transfer spike:** Volume leaving the workstation is orders of magnitude above baseline.
- **Off-hours activity:** If exfiltration happens at 2 AM while Maria is asleep, UEBA flags the off-hours network activity.

**If the dice roll failed,** then tell the defenders that there was a payment failure and the vendor blocked logins or some other statement.

---

### PERSISTENCE: AdminSDHolder Rights Modification

*Attacker modified AdminSDHolder ACL, and SDProp auto-propagates backdoor permissions to all protected groups every 60 minutes.*

**Can be solved by: SIEM, UEBA, or Server Analysis**

---

#### SIEM

**Why it works:** AD object modifications generate specific Event IDs that record exactly what changed, when, and by whom.

**What to look for:**
- **Event ID 5136 (Directory object modified):** Fires when any AD object's attributes change. Filter for modifications to `CN=AdminSDHolder,CN=System,DC=corp,DC=local`. Shows the exact timestamp, the account that made the change, and the old vs. new ACL values (the `nTSecurityDescriptor` attribute).
- **Event ID 4662 (Directory operation):** Logs Write DACL operations against AdminSDHolder. If the source account isn't a legitimate admin performing authorized maintenance, it's an IOC.
- **Secondary SDProp evidence:** As SDProp propagates the backdoored ACL, Events 4738 or 4732 may fire showing permission changes on Domain Admins and other protected groups, all originating from the SDProp process with the *new, unauthorized* content.

**If the dice roll failed, then they searched for account creation:** "You searched for Event ID 4720 (new account) finding nothing. The attacker didn't create a new account. They used the existing `svc-monitoring` account. They modified *permissions*, not *accounts*. What Event IDs fire when an AD object's permissions change?"

---

#### UEBA

**Why it works:** Detects dormant accounts reactivating and accounts performing actions outside their historical profile.

**What to look for:**
- **Dormant account activation:** `svc-monitoring` hasn't been used in months. UEBA flags its sudden reactivation: logging in, performing directory queries, accessing resources as a high-severity anomaly. Dormant accounts coming to life is one of the strongest compromise indicators.
- **Privilege usage anomaly:** `svc-monitoring` was a monitoring service account. It never performed AD administrative operations. UEBA flags ACL modifications on privileged objects as completely outside its profile.
- **Protected group permission changes:** UEBA tools monitoring group membership can detect when effective permissions on protected groups change even from SDProp propagation.

**If the dice roll failed, then they're looking at the wrong accounts:** "You checked the Administrator account and it looks normal because the attacker isn't using it directly. They're using a forgotten service account. Try asking UEBA: 'Show me every account inactive for 90+ days that suddenly became active this week.'"

---

#### Server Analysis

**Why it works:** Direct examination of AD objects reveals the modification itself with no log correlation needed.

**What to look for:**
- **AdminSDHolder ACL inspection:** Use PowerShell (`Get-ACL "AD:\CN=AdminSDHolder,CN=System,DC=corp,DC=local"`), `dsacls`, or Sysinternals AD Explorer to view the current ACL. The unauthorized ACE granting `svc-monitoring` full control is immediately visible.
- **Protected group ACL check:** Check Domain Admins, Enterprise Admins, Schema Admins ACLs — `svc-monitoring` already has full control on all of them (propagated by SDProp). Same unauthorized ACE on AdminSDHolder AND every protected group confirms active propagation.
- **Service account audit:** Examine `svc-monitoring` properties — last logon, password last set, group memberships, description. A legacy account with no current purpose that has full control on AdminSDHolder is a critical finding.

**If they check GPOs instead:** "GPOs and OUs are clean. The attacker modified something more subtle: AdminSDHolder. Most AD admins never look at it. Have you ever examined the AdminSDHolder object in your environment?"

---

## The Golden Ticket: The Capability Between the Lines

The Golden Ticket isn't a separate card in the layout, but it's the **capability** the attacker gains between ESC1 and AdminSDHolder. The Incident Captain should weave it in:

After getting Domain Admin via ESC1, the attacker runs DCSync, impersonating a DC to request the KRBTGT hash. With it, they forge a Golden Ticket.

**Key teaching moment:** If defenders suggest "reset the Administrator password," explain:

> "The Golden Ticket doesn't USE the Administrator password. It's encrypted with the KRBTGT hash which is a completely different key. As long as KRBTGT hasn't changed, the attacker can forge tickets for ANY user. The only fix is resetting KRBTGT *twice*, once to invalidate the current key, once to invalidate the previous key (Kerberos retains both). When was your KRBTGT last reset?" 

*Also, don't just rapid fire this password rotation twice as you will probably have a really bad time.*

Most organizations discover it's never been reset. I've seen multiple krbtgt account passwords old enough to take out for a beer. 

---

## Inject Card Guidance

- **SIEM Analyst Returns from Splunk Training** - Analyst knows Event IDs 4886/4887 (cert enrollment), 4768 pre-auth type 16 (PKINIT), 4662 (DCSync), and 5136 (AdminSDHolder). Grant +2 on next SIEM roll.
- **Honeypots Deployed** - A honeytoken service account with a Kerberos SPN was Kerberoasted by the attacker during recon (before finding ESC1). The alert fired three days ago but wasn't reviewed. Reveal one card and ask: "What if someone had looked at that alert?"
- **Bobby the Intern Kills the System You are Reviewing** - Bobby disables the CA's web enrollment. Stops new malicious certs but breaks Wi-Fi authentication company-wide (which uses certificate auto-enrollment). Help desk floods.
- **Take One Procedure Card Away** - The attacker uses the Golden Ticket to access the SIEM and begins deleting logs. Remove SIEM. This is realistic: with a Golden Ticket, the attacker can access *any system*.
- **Legal Takes Your Only Skilled Handler Into a Meeting** - The NTDS.dit dump means every password hash was stolen. Breach notification discussions begin. Your AD expert is unavailable.

---

## Why the Other Procedures Fail (And When They Could Work)

This section is the Incident Captain's cheat sheet for when defenders propose a procedure that *doesn't* solve the current card. Instead of just saying "nothing comes up," the Incident Captain should explain **why** the procedure doesn't work here and, where applicable, **how it could work in a real-world variant** of this attack. These explanations are where some of the deepest learning happens.

### Procedures That DON'T Solve PHISH And Why

*Reminder: Phish IS solved by SIEM, Server Analysis, Cloud Event Log Analysis, and UEBA.*

---

**Firewall Log Review**

**Why it fails here:** The phishing email arrived through the organization's normal email flow, it wasn't blocked by the firewall because inbound email on port 25/443 is allowed traffic. The subsequent Cobalt Strike beacon communicates outbound over HTTPS (port 443) or DNS, which also passes through the firewall without being blocked. Firewall logs show an allowed HTTPS connection from Maria's workstation to an external IP but so do thousands of other legitimate web connections every hour. Without context about *what process* initiated the connection (which firewalls don't log), the phishing beacon blends in.

**When it COULD work in real life:** If the organization's firewall performs TLS inspection and has a URL categorization or reputation service, newly registered or uncategorized domains used for the Cobalt Strike team server might be flagged or blocked. If the C2 domain was registered recently (hours or days before the attack), a firewall with domain age filtering would catch it. Additionally, if the firewall logs include the SNI hostname (Server Name Indication from the TLS handshake), a threat hunter reviewing outbound connections to suspicious hostnames could find the C2. But these are proactive hunting actions, not something the firewall log review procedure would surface on a standard review.

---

**Endpoint Security Protection Analysis**

**Why it fails here:** The Cobalt Strike beacon runs entirely in memory. No malicious files were written to disk. Traditional antivirus and many EDR solutions rely on scanning files for known malicious signatures. Since there's no file to scan, signature based detection comes up empty. The beacon was loaded through a "PowerShell download cradle", a legitimate Windows tool (PowerShell) downloading and executing code directly into memory. Many organizations have PowerShell execution allowed for legitimate business purposes, so the EDR may not flag it.

**When it COULD work in real life:** Modern EDR platforms with behavioral detection (not just signature-based) can catch this. Products like CrowdStrike Falcon, Microsoft Defender for Endpoint, and SentinelOne have specific detections for "fileless malware" patterns like PowerShell making outbound HTTP requests and executing reflectively loaded .NET assemblies. If the organization's EDR has these behavioral rules enabled and tuned, it might alert on the initial PowerShell execution or on the Cobalt Strike beacon's in-memory activity patterns (named pipes, sleep/jitter beaconing behavior, process injection). The reason it fails *in this scenario* is to teach the lesson that not all EDR deployments are created equal and an EDR that only does signature scanning is fundamentally different from one that does behavioral analysis.

---

**Endpoint Analysis**

**Why it fails here:** Endpoint Analysis (forensic examination of the workstation) actually *can* find evidence of the phish: the email in the Outlook cache, the downloaded document, PowerShell transcript logs, and browser history showing the phishing click. However, in the game structure, we can't have everything that works work. In scenario, because it requires someone to already suspect Maria's specific workstation and go look at it. At the start of the game, defenders don't yet know which workstation is compromised. The SIEM and UEBA procedures are more likely to *identify* which machine to look at in the first place.

**When it COULD work in real life:** Endpoint Analysis is extremely powerful once you know which machine to examine it's just not a good *discovery* mechanism. It's a *confirmation and evidence-gathering* mechanism. In a real incident, the SIEM or UEBA would flag the anomaly, and Endpoint Analysis would follow to build the forensic timeline. The Incident Captain can allow Endpoint Analysis to reveal the Phish card if the defenders specifically say "we want to examine Maria's workstation" but only if they have a reason to suspect her machine. If they say "we want to do endpoint analysis" without specifying which endpoint, ask: "Which machine? You have 400 workstations. What's drawing you to a specific one?"

---

**Network Threat Hunting / Zeek / RITA Analysis**

**Why it fails here:** Network flow analysis shows connections: source IP, destination IP, port, bytes, duration. The Cobalt Strike beacon's C2 connection looks like a normal HTTPS session to an external server. Without deep packet inspection (which HTTPS encryption prevents), flow data alone can't distinguish the beacon from a user browsing the web. The connection volume is small (command-and-control check-ins are typically just a few hundred bytes), so it doesn't trigger volume-based alerts.

**When it COULD work in real life:** RITA (Real Intelligence Threat Analytics) specifically analyzes flow data for *beaconing patterns*, the regular, periodic connections to the same destination with consistent timing intervals. Cobalt Strike beacons, even with jitter configured, often show detectable periodicity. If RITA is deployed and has been collecting data since before the attack, it could identify the beacon's C2 connection as a statistical outlier in timing regularity. However, this is better suited for revealing the C2/Exfil card (LOTS) than the initial phish itself bewcause the beaconing pattern tells you "this machine is talking to a C2 server" but doesn't directly reveal "this machine was phished." The distinction matters: finding the C2 is a consequence of the phish, not the phish itself.

---

**Crisis Management**

**Why it fails here:** Crisis Management is an organizational coordination procedure, assembling leadership, assigning roles, establishing communication channels. It doesn't involve any technical investigation. You can't discover a phishing attack by holding a meeting. Crisis Management becomes critical *after* a card is revealed (coordinating the response), but it doesn't reveal cards on its own in this scenario.

---

**Isolation**

**Why it fails here:** Isolation is a containment action, not a discovery mechanism. You can isolate Maria's workstation but you first need to know it's Maria's workstation that's compromised. Isolating a machine doesn't tell you *how* it was compromised (phishing vs. drive-by download vs. USB attack). It stops the bleeding but doesn't diagnose the wound.

**When it COULD work in real life:** If the defenders isolate the compromised workstation and then see the C2 beacon drop from the network and simultaneously notice that the LOTS exfiltration traffic also stops, the correlation confirms that this specific machine was the source. That's useful context but still doesn't reveal the phishing mechanism. Isolation is always a *response* to a finding, not the finding itself.

---

**Internal Segmentation**

**Why it fails here:** Internal Segmentation involves reviewing or enforcing network segmentation between zones (e.g., user VLAN vs. server VLAN vs. management VLAN). The phishing attack happened on a standard user workstation in the user VLAN. Reviewing segmentation policies won't surface the phishing email or the beacon, it will only tell you whether the attacker could move laterally *after* compromising the workstation. That's relevant to later attack stages, not the initial compromise.

**When it COULD work in real life:** Strong segmentation is a *preventive* control, not a *detective* one. If the HR VLAN was properly segmented from the server VLAN and the management VLAN, the attacker's ability to reach the Certificate Authority from Maria's workstation would be limited. This wouldn't *reveal* the phish, but it would *prevent or slow* the escalation that follows. The Incident Captain can note this during the debrief as a defense-in-depth discussion point.

---

**Permissions Audit**

**Why it fails here:** A Permissions Audit reviews who has access to what: AD group memberships, certificate template enrollment rights, file share permissions, etc. The phishing attack exploited Maria's standard user credentials. A permissions audit would confirm that Maria has normal user permissions (which she's supposed to have). Nothing about her permissions is anomalous. The attack didn't exploit excessive permissions at this stage, it exploited human trust (clicking a malicious link).

**When it COULD work in real life:** A proactive permissions audit *before* the attack could identify that Maria (and all Authenticated Users) can enroll in the misconfigured certificate template which sets up the ESC1 discovery. But that's the *next* card, not this one. For the phish itself, permissions aren't the issue.

---

### Procedures That DON'T Solve ESC1 and Why

*Reminder: ESC1 IS solved by SIEM, Cyber Deception, Server Analysis, and Permissions Audit.*

---

**Firewall Log Review**

**Why it fails here:** The ESC1 attack is entirely internal. The certificate request goes from Maria's workstation to the internal Enterprise CA server and both are on the internal network. No traffic crosses the firewall. Firewall logs capture perimeter traffic (internal-to-external), not internal-to-internal communication between a workstation and an internal CA.

**When it COULD work in real life:** If the organization has *internal* firewall segmentation (micro-segmentation) with logging between the user VLAN and the server VLAN, the certificate enrollment traffic (typically DCOM/RPC or HTTPS to the CA's web enrollment interface) would appear in those logs. However, even then, it would look like a legitimate certificate request since the firewall can't distinguish a normal enrollment from a malicious one with a forged SAN. Only the *content* of the request reveals the abuse, and firewalls don't inspect certificate enrollment payloads.

---

**UEBA**

**Why it fails here:** UEBA tracks behavioral patterns but certificate enrollment may not be in the behavioral baseline at all. If UEBA doesn't have visibility into AD CS enrollment events, it has no baseline to compare against. Many UEBA deployments focus on authentication events, file access, and network connections. Certificate enrollment is a specialized data source that most UEBA tools don't ingest by default.

**When it COULD work in real life:** If the UEBA tool is configured to ingest CA enrollment logs or Windows Event ID 4886/4887, it could detect that Maria's workstation has never requested a certificate before (behavioral anomaly) or that a certificate was requested with a SAN that doesn't match the enrolling user (identity mismatch anomaly). Advanced UEBA deployments with AD CS visibility absolutely could catch this. The reason it fails in this scenario is to highlight the gap: **most organizations don't feed certificate enrollment data into their UEBA**. That's the teaching moment.

---

**Endpoint Security Protection Analysis**

**Why it fails here:** The ESC1 attack uses Certipy or Certify, tools that make standard Windows API calls to request a certificate from the CA. From the endpoint's perspective, a certificate request is a normal Windows operation. EDR may flag Certipy as a known offensive tool (if it has the signature), but if the attacker uses a custom-compiled version or a living-off-the-land approach using built-in `certreq.exe`, the endpoint shows nothing unusual. The certificate request looks like any other administrative action.

**When it COULD work in real life:** If the EDR has specific rules for detecting certificate abuse tools (Certipy, Certify, Rubeus certify module), or if it flags the execution of `certreq.exe` with unusual command-line parameters (specifically the `-attrib "SAN:administrator@corp.local"` flag), the ESC1 attempt could be caught at the endpoint level. CrowdStrike and Microsoft Defender for Endpoint have added detections for some AD CS abuse patterns. But many EDR deployments don't have these rules tuned or enabled.

---

**Endpoint Analysis**

**Why it fails here:** Forensic examination of Maria's workstation could find the Certipy tool and its output but this requires the defenders to already know which machine to examine and what to look for. If they examine the endpoint and find Certipy or certificate artifacts, they *could* piece together the ESC1 attack. However, without knowing that certificates are involved, they're unlikely to look for these specific artifacts among the broader phishing evidence.

**When it COULD work in real life:** If the defenders found Certipy during the phishing investigation (while examining Maria's workstation for the beacon), they might follow the thread: "Why is a certificate abuse tool on this machine?" to "What certificate was requested?" to ESC1 discovery. This is a realistic investigative path. The Incident Captain could audible this and allow Endpoint Analysis to contribute to revealing ESC1 if the defenders explicitly say "we found the beacon on Maria's machine and now we want to see what other tools the attacker ran from there." That's good investigative thinking and should be rewarded.

---

**Network Threat Hunting/ Zeek / RITA Analysis**

**Why it fails here:** The certificate enrollment traffic between Maria's workstation and the internal CA is standard DCOM/RPC or HTTPS traffic on the internal network. NetFlow data shows a connection from `WS-HR-023` to the CA server but workstations legitimately communicate with the CA for certificate auto-enrollment, so this connection isn't anomalous in flow data. The flow metadata doesn't reveal the *content* of the certificate request (the malicious SAN).

**When it COULD work in real life:** Zeek (Bro) with TLS/SSL certificate logging can extract certificate details from network traffic, potentially revealing the SAN mismatch. If Zeek is deployed on internal network segments and has TLS inspection capabilities, it could log the certificate content and flag a certificate issued to `administrator@corp.local` from a workstation that shouldn't be requesting Domain Admin certificates. This is a sophisticated deployment but technically feasible.

---

**Cloud Event Log Analysis**

**Why it fails here:** The ESC1 attack is entirely on-premises. It targets an on-premises Enterprise CA and an on-premises Active Directory domain. Cloud audit logs (Azure AD / Entra ID sign-in logs, Microsoft 365 Unified Audit Log) don't capture on-premises certificate enrollment events. The CA is not a cloud service, it's a Windows Server running AD CS.

**When it COULD work in real life:** If the organization uses Azure AD/Entra ID and the attacker uses the certificate to authenticate to cloud resources (like Azure AD via federated authentication or cloud-connected workloads), the cloud sign-in logs would show the certificate-based authentication event. Additionally, if the organization uses Microsoft Defender for Identity (which monitors on-premises AD traffic and forwards alerts to the cloud portal), it could detect the ESC1 abuse and surface it through cloud-side alerting. But the ESC1 enrollment itself is on-prem and cloud logs alone can't see it.

---

**Crisis Management**

**Why it fails here:** Same as with Phish, Crisis Management is coordination, not detection. You can't discover a certificate template misconfiguration by holding a meeting.

---

**Isolation**

**Why it fails here:** Isolating Maria's workstation kills the beacon and prevents the attacker from requesting *more* certificates. But it doesn't reveal *that* a certificate was already issued. The malicious certificate has already been generated, and the attacker may have already used it to authenticate and perform DCSync. Isolation addresses the foothold but not the escalation that already occurred.

**When it COULD work in real life:** If isolation is combined with a review of the CA's issued certificates log (Server Analysis), the timeline becomes clear: "We isolated the workstation at 2:00 PM. The malicious certificate was issued at 11:30 AM. The attacker had 2.5 hours of Domain Admin access before containment." Isolation is a response action, not a discovery mechanism but it helps frame the timeline.

---

**Internal Segmentation**

**Why it fails here:** Reviewing segmentation tells you whether Maria's workstation *can reach* the CA server. In this environment, it can since the CA is accessible from the user VLAN for legitimate certificate enrollment. The segmentation is working as designed. The problem isn't network access to the CA, it's the template misconfiguration on the CA itself. Segmentation can't prevent an authorized user from making an authorized (but maliciously configured) certificate request.

**When it COULD work in real life:** If the CA were placed in a dedicated management VLAN with restricted access (only IT admin workstations can reach it), the ESC1 attack from Maria's HR workstation would fail because her machine couldn't connect to the CA at all. This is a valid defense-in-depth recommendation for the debrief: **should standard user workstations have direct network access to the Certificate Authority?** In many environments, the answer should be no.

---

### Procedures That DON'T Solve LOTS and Why

*Reminder: LOTS IS solved by Network Threat Hunting, SIEM, Firewall Log Review, and UEBA.*

---

**Server Analysis**

**Why it fails here:** The LOTS C2 and exfiltration happen FROM endpoints and domain controllers TO external cloud services. Server Analysis focuses on examining the configuration and logs of internal servers. The domain controllers themselves don't log "I uploaded data to Azure Blob Storage" the upload is initiated by the Cobalt Strike beacon running on the compromised system, and the destination is external. The server's own logs show normal AD operations, not the exfiltration traffic that passes through it.

**When it COULD work in real life:** If the defenders examine the domain controller's process list and find the Cobalt Strike beacon running there (the attacker pivoted to the DC using the Golden Ticket), that's a finding but it reveals the *persistence* or *pivot*, not the exfiltration channel. Finding the beacon leads to C2 discovery indirectly. Additionally, PowerShell transcript logs on the DC might show commands used to stage data for exfiltration.

---

**Endpoint Security Protection Analysis**

**Why it fails here:** EDR looks at process behavior, file operations, and local network connections on the endpoint. The LOTS exfiltration is outbound HTTPS to github.com and blob.core.windows.net, both legitimate domains. EDR typically doesn't flag connections to GitHub or Azure as malicious because they're in pretty much every allowlist. The data being uploaded is compressed and encrypted before transmission, so data loss prevention (DLP) based on content inspection doesn't trigger either.

**When it COULD work in real life:** Some advanced EDR platforms can detect when processes access sensitive files (like the NTDS.dit) and then make outbound network connections correlating file access with network activity on the same endpoint. If the EDR detects "process X read NTDS.dit and then connected to blob.core.windows.net," that's a high-fidelity detection. But this requires file-to-network correlation capabilities that many EDR deployments don't have configured.

---

**Endpoint Analysis**

**Why it fails here:** Forensic endpoint analysis could find evidence of data staging (compressed archives in temp directories, PowerShell history showing upload commands), but the exfiltrated data went to external cloud services — once it leaves the machine, it's beyond the endpoint's visibility. Endpoint Analysis shows what the attacker *did* on the machine, but the LOTS *channel* (GitHub Gists, Azure Blob) is a network-level observation, not an endpoint-level one.

**When it COULD work in real life:** If the attacker used a browser to access the GitHub Gist for commands, browser history and cached pages on the endpoint would show the Gist URL thus linking the endpoint to the C2 channel. PowerShell command history (if not cleared) might show `Invoke-WebRequest` or `az storage blob upload` commands that directly reveal the exfiltration method. This is realistic and the Incident Captain could allow it if defenders specifically describe searching for command history and browser artifacts related to cloud service interaction.

---

**Cloud Event Log Analysis**

**Why it fails here:** Cloud Event Log Analysis refers to reviewing logs from *your organization's* cloud services (Azure AD, M365, GCP audit logs). The LOTS exfiltration goes to the *attacker's* Azure Blob Storage and the *attacker's* GitHub Gist, resources in accounts your organization doesn't control and has no logs for. Your cloud logs won't show the attacker uploading data to *their* Azure storage container.

**When it COULD work in real life:** If the attacker exfiltrated data through your organization's *own* cloud services (e.g., uploading to a SharePoint site, emailing via M365, or using OneDrive), cloud event logs would catch it. Microsoft 365's Unified Audit Log records file uploads, sharing events, and email sends. But LOTS specifically involves *trusted third-party sites* like GitHub, Azure Blob (under the attacker's account), paste sites, that are outside your organization's logging boundary. This is the whole point of LOTS: the attacker uses infrastructure you don't monitor.

---

**Permissions Audit**

**Why it fails here:** A Permissions Audit reviews who has access to what within your AD and cloud environment. It's excellent for finding misconfigurations (like ESC1) but doesn't examine network traffic patterns. The LOTS exfiltration is a network behavior, not a permissions issue — the attacker already has the permissions (via Golden Ticket) to access the data. The problem isn't *who can access the data* but *where the data is going after access*.

**When it COULD work in real life:** A Permissions Audit could reveal *what data the attacker could access* given their current privileges, helping to scope the breach. If the audit shows that the Golden Ticket-level privileges can reach every file share, database, and mailbox in the environment, that tells you the potential scope of exfiltration, even if it doesn't reveal the exfiltration channel itself. That scoping information is valuable for breach notification and remediation.

---

**Crisis Management**

**Why it fails here:** Coordination doesn't detect network exfiltration. However, Crisis Management becomes critical *after* LOTS is discovered: coordinating with GitHub (requesting takedown of the C2 Gist), Azure (requesting suspension of the attacker's storage account), and legal (assessing breach notification obligations based on the data confirmed exfiltrated).

---

**Isolation**

**Why it fails here:** Isolation stops the exfiltration by disconnecting compromised machines from the network but it doesn't tell you *how* the data left or *where it went*. You know the bleeding stopped, but you don't know how much blood was lost or where it went.

**When it COULD work in real life:** If you isolate machines and simultaneously capture a final packet capture (PCAP) as the connection terminates, you might catch the destination URL in the TLS SNI field or DNS query, revealing the exfiltration endpoint. But this requires precise timing and a network tap already in place.

---

**Internal Segmentation**

**Why it fails here:** Internal Segmentation controls traffic *between internal zones*. LOTS exfiltration goes outbound to the internet making it a perimeter problem, not a segmentation problem. Even perfect internal segmentation doesn't prevent a compromised machine in any zone from making outbound HTTPS connections to GitHub.

**When it COULD work in real life:** If the organization enforces *egress* segmentation or restricting which internal zones can make outbound internet connections, a domain controller reaching out to GitHub would be blocked. Domain controllers typically shouldn't need internet access. This is a strong defense-in-depth recommendation: **restrict outbound internet access from your domain controllers to only the destinations they actually need** (Windows Update, time sync, AV updates). If DC01 can't reach github.com, the LOTS exfiltration from the DC fails.

---

### Procedures That DON'T Solve AdminSDHolder and Why

*Reminder: AdminSDHolder IS solved by SIEM, UEBA, and Server Analysis.*

---

**Firewall Log Review**

**Why it fails here:** The AdminSDHolder modification is an internal Active Directory operation, a change to an AD object's ACL performed via LDAP on a domain controller. No traffic crosses the firewall. The modification is invisible at the network perimeter.

**When it COULD work in real life:** This cannot work for detecting AdminSDHolder modifications because the operation is entirely internal to AD. However, if the attacker uses the AdminSDHolder-granted permissions to subsequently access external resources (like the LOTS exfiltration), the firewall *could* see the *consequences* of the persistence, just not the persistence mechanism itself.

---

**Endpoint Security Protection Analysis**

**Why it fails here:** The AdminSDHolder modification is made against the AD database on the domain controller using standard LDAP/AD administrative tools or APIs. From an endpoint perspective, this looks like normal AD administration, the same operations that legitimate admins perform. EDR on the domain controller doesn't flag standard LDAP operations as malicious.

**When it COULD work in real life:** Microsoft Defender for Identity (a cloud-connected EDR-like product for domain controllers) specifically monitors for suspicious AD object modifications, including AdminSDHolder ACL changes. If the organization has Defender for Identity deployed, it would alert on the modification. However, standard EDR products (designed for workstation threats) don't have this AD-specific detection capability. This is a key distinction for the debrief: **general-purpose EDR and AD-specific monitoring tools solve different problems**.

---

**Endpoint Analysis**

**Why it fails here:** Endpoint forensics on the workstation or even the domain controller won't directly reveal the AdminSDHolder change. The modification is stored in the AD database (NTDS.dit), not in files, processes, or registry entries that endpoint forensics typically examines. You need to query the AD object directly (Server Analysis) or find the modification event in the logs (SIEM), not examine the filesystem.

**When it COULD work in real life:** If the attacker ran PowerShell commands on the DC to modify AdminSDHolder, and PowerShell transcription logging was enabled, the transcript logs on the DC would show the exact commands used (e.g., `Set-ACL` or `Add-ADPermission` targeting the AdminSDHolder DN). This bridges endpoint forensics with AD object changes, but it requires PowerShell logging to be enabled on domain controllers, which many organizations don't do.

---

**Network Threat Hunting / Zeek / RITA Analysis**

**Why it fails here:** The AdminSDHolder modification is an LDAP operation between the attacker's session (on a compromised machine) and the domain controller. In network flow data, this looks like normal LDAP traffic (port 389 or 636). Workstations and servers communicate with domain controllers via LDAP constantly for authentication, group policy processing, and directory queries. The AdminSDHolder modification is a single LDAP write operation buried in thousands of normal LDAP transactions. Network metadata doesn't have the granularity to distinguish "LDAP: modify AdminSDHolder ACL" from "LDAP: look up a user's email address."

**When it COULD work in real life:** Zeek has LDAP protocol parsing capabilities that can extract operation types and target DNs from LDAP traffic. If Zeek is deployed on the internal network and configured to log LDAP operations in detail, it could potentially log the Write DACL operation against the AdminSDHolder distinguished name. This is an advanced deployment since most organizations don't run Zeek with full LDAP parsing on internal segments but it's technically possible and is an emerging detection strategy.

---

**Cloud Event Log Analysis**

**Why it fails here:** AdminSDHolder is an on-premises Active Directory object. Its modification is logged in on-premises Windows Event Logs (Event ID 5136), not in cloud audit logs. Azure AD / Entra ID doesn't replicate or monitor AdminSDHolder changes.

**When it COULD work in real life:** If the organization uses Microsoft Defender for Identity (cloud-connected, but monitoring on-premises AD), AdminSDHolder modifications would be surfaced through the cloud security portal even though the change happened on-prem. Defender for Identity bridges the on-prem/cloud visibility gap for exactly these kinds of AD-specific attacks. Also, if the organization has Azure AD Connect syncing with on-premises AD, certain downstream effects of the AdminSDHolder change (like permission propagation to synced groups) *might* generate cloud-side audit events but this is indirect and unreliable.

---

**Permissions Audit**

**Why it fails here (in the game):** This is a borderline case. A Permissions Audit *could* reveal AdminSDHolder checking ACLs on AD objects is essentially what a permissions audit does. However, in the game structure, this procedure just doesn't work.

**When it COULD work in real life:** A comprehensive AD permissions audit using tools like BloodHound, PingCastle, or Purple Knight would absolutely flag an unauthorized ACE on AdminSDHolder. In fact, this is one of the highest-value checks in any AD security assessment. If your team runs a Permissions Audit and specifically scopes it to include the AdminSDHolder object, they would find the backdoor immediately. **The Incident Captain can choose to allow Permissions Audit to reveal AdminSDHolder if the defenders specifically mention checking AdminSDHolder permissions** if you want to reward precise thinking.

---

**Cyber Deception**

**Why it fails here:** Honeytokens and decoys are typically placed to detect *attacker reconnaissance and access*, and these consist of fake accounts that attract Kerberoasting, fake credentials that attract credential stuffing, fake shares that attract lateral movement. The AdminSDHolder modification is a *persistence* action that happens after the attacker already has Domain Admin, At this point they're modifying AD infrastructure, not interacting with decoy resources.

**When it COULD work in real life:** If a honeytoken ACE was placed on the AdminSDHolder object itself (an ACE for a fake monitoring account that triggers an alert when the AdminSDHolder object is read or written), it could detect both the attacker's *reconnaissance* of AdminSDHolder and their subsequent *modification* of it. This is an advanced deception technique essentially making AdminSDHolder itself a trap. I don't know if this is done by many, but it's a creative and potentially effective strategy.

---

**Crisis Management**

**Why it fails here:** Coordination doesn't detect AD object modifications. Crisis Management is critical for the *response* (coordinating the KRBTGT double-reset, planning the AdminSDHolder cleanup, managing the domain-wide password reset) but doesn't discover the persistence mechanism.

---

**Isolation**

**Why it fails here:** You can isolate every compromised workstation in the environment, and the AdminSDHolder backdoor remains untouched. It's embedded in the AD database itself, not on any endpoint. Isolation addresses *network access* where AdminSDHolder is a *directory services* problem. The attacker doesn't need any currently-compromised machine to exploit the AdminSDHolder backdoor since they can authenticate from any machine using a Golden Ticket and the `svc-monitoring` account will have full control of Domain Admins, freshly re-applied by SDProp.

**When it COULD work in real life:** Isolation can *indirectly* support AdminSDHolder remediation by cutting off the attacker's active sessions while the AD team cleans the ACL. But it doesn't discover or fix the persistence, only direct AD object modification does that.

---

**Internal Segmentation**

**Why it fails here:** Internal Segmentation controls which network zones can communicate. The AdminSDHolder modification is an LDAP write operation from a compromised session to a domain controller traffic that is allowed by design (every domain-joined machine needs to talk to DCs). You can't segment away from the domain controller without breaking Active Directory entirely.

**When it COULD work in real life:** Tiered administration models (like Microsoft's Enterprise Access Model) restrict *which accounts* can log into *which systems*. If Tier 0 (domain controllers and AD infrastructure) is properly isolated so that only dedicated admin workstations can make administrative LDAP changes, the attacker's attempt to modify AdminSDHolder from Maria's workstation would fail because her machine isn't authorized for Tier 0 administration. This is a preventive control, not a detective one but it's one of the most effective defenses against AD-level attacks.

---

## Remediation Guide (For the Debrief)

Walk through the full remediation sequence after the game:

1. **Contain the foothold:** Isolate Maria's workstation. Kill the beacon.
2. **Fix ESC1:** Remove `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT` from `UserAuthCert`. Restrict enrollment permissions. Audit all templates.
3. **Revoke the certificate:** Revoke the cert issued to `administrator@corp.local`. Publish updated CRL.
4. **Clean AdminSDHolder:** Remove the `svc-monitoring` ACE. Wait 60 minutes for SDProp to propagate clean permissions. Verify.
5. **Disable `svc-monitoring`:** Disable, reset password, remove from groups. Audit its history.
6. **Reset KRBTGT TWICE:** Reset once, wait for replication, reset again. This invalidates all Golden Tickets. **Plan for disruption.**
7. **Reset all passwords:** The NTDS.dit dump means every hash was stolen. Force domain-wide password reset. Prioritize privileged accounts.
8. **Hunt for more:** Scheduled tasks, GPO modifications, additional rogue accounts, other malicious certificates.

## After-Action Discussion

This is the most important part of the exercise. The game itself is the hook, the debrief is where the learning happens. The Incident Captain should plan for **at least 20–30 minutes of open discussion** after the final card is revealed (or after turn 10 if the defenders lose). The sections below provide a structured framework, but follow the energy of the room, if a topic sparks a great conversation, stay on it.

---

### Part 1: What Just Happened? (Attack Chain Walkthrough)

Start by walking through the full attack chain from end to end. Many players will have been focused on the card they were investigating and may not have the complete picture. Lay it out as a story:

> "Let's rewind and walk through this from the attacker's perspective, start to finish.
>
> The attacker sent a phishing email to Maria in HR. She clicked it. A macro ran PowerShell, which loaded Cobalt Strike into memory. At this point, the attacker has a remote shell on a standard user workstation. They're inside your network, but they're nobody special. They have Maria's permissions, which provide basic user access.
>
> From that workstation, the attacker ran Certipy to scan your Certificate Authority for misconfigured templates. They found one, `UserAuthCert`, that lets any authenticated user request a certificate for *any other user*, including Domain Admins. This misconfiguration has probably been there since the CA was first deployed. The attacker requested a certificate as the built-in Administrator, and the CA issued it because the template allowed it. One certificate request: standard user to Domain Admin.
>
> With Domain Admin privileges, the attacker performed a DCSync by impersonating a domain controller, requesting the password hash for every account in the domain, including the KRBTGT account. With the KRBTGT hash, they forged a Golden Ticket: a fabricated Kerberos TGT that lets them impersonate anyone, access anything, and persist indefinitely.
>
> To make sure they could survive a cleanup, the attacker modified the AdminSDHolder object adding full-control permissions for a dormant service account they'd taken over. Every 60 minutes, the SDProp process silently copies those permissions onto every protected group in the domain. Even if you manually remove the attacker's access from Domain Admins, it comes back automatically within the hour.
>
> Finally, the attacker exfiltrated the NTDS.dit (every password hash in the domain), executive email exports, and HR compensation data and then uploaded to Azure Blob Storage through a pre-signed URL, with C2 commands retrieved from a GitHub Gist. Every byte of exfiltration traffic went to legitimate, allowlisted domains over HTTPS.
>
> From phish to total domain compromise: probably four to six hours. The attacker now has every password hash, a Golden Ticket, and a self-healing persistence mechanism. And the exfiltration went to domains your firewall was configured to trust."

Pause. Let the room absorb this. Then ask: **"Where could we have broken this chain?"**

---

### Part 2: Where the Chain Could Have Been Broken

Walk through each transition point in the attack chain. At each point, ask the table what control could have prevented the attacker from reaching the next stage. This helps players see that defense-in-depth means creating *multiple* opportunities to stop an attack and not relying on any single control.

**Break Point 1: The Phishing Foothold**

The phishing email reached Maria's inbox, she clicked the link, and the macro executed.

Questions for the table:

- Did our email security gateway scan the link and the attachment? Should it have caught this? What are the limitations of email security against targeted spearphishing?
- Did the macro execute because Office macros are enabled by default in our environment? Microsoft has been disabling macros in files downloaded from the internet by default since 2022 but have we enforced this policy?
- Could attack surface reduction (ASR) rules have blocked Office from spawning PowerShell? Does our organization use ASR rules? Do we even know what they are?
- If Maria had been trained on phishing awareness, would she have recognized this email? It was targeted and used a believable benefits enrollment pretext. Is it fair to rely on human judgment as a security control when the phish is this well-crafted?

**Key takeaway to share:** "Phishing is the most common initial access vector. You will never reduce click rates to zero. The question isn't 'how do we stop people from clicking', it's 'what happens AFTER someone clicks, and how quickly do we detect it.' If your entire security strategy depends on nobody ever clicking a malicious link, you don't have a security strategy, you have hopes and prayers"

---

**Break Point 2: Foothold to Domain Admin via ESC1**

The attacker went from a standard user workstation to Domain Admin using a single misconfigured certificate template.

Questions for the table:

- Has anyone in this room ever audited our AD CS certificate templates? Do we know how many templates are published? Do we know which ones have the `ENROLLEE_SUPPLIES_SUBJECT` flag enabled?
- Why did the `UserAuthCert` template allow any authenticated user to enroll? Who configured it this way? Was it a deliberate decision or a default setting that nobody changed?
- How long has this misconfiguration existed? If it's been there since the CA was deployed (potentially years), how many audits, penetration tests, and security reviews missed it?
- Do we have any monitoring on certificate enrollment events? Are Events 4886 and 4887 from the CA being forwarded to the SIEM? If not, we are completely blind to certificate-based attacks.

**Key takeaway to share:** "AD CS is one of the most overlooked attack surfaces in enterprise AD. The SpecterOps 'Certified Pre-Owned' research identified eight categories of AD CS vulnerabilities (ESC1 through ESC8), and found exploitable misconfigurations in the vast majority of environments they tested. If you've never audited your certificate templates, you almost certainly have at least one ESC1 or ESC8 vulnerability. The fix for ESC1 is straightforward: remove the `ENROLLEE_SUPPLIES_SUBJECT` flag from templates that don't need it, and restrict enrollment permissions to specific security groups. This is a 15-minute configuration change that eliminates one of the most powerful privilege escalation paths in AD."

---

**Break Point 3: Domain Admin to Golden Ticket (DCSync + KRBTGT)**

The attacker used Domain Admin to perform a DCSync and extract the KRBTGT hash.

Questions for the table:

- Do we monitor for DCSync activity? Event ID 4662 with the `Replicating Directory Changes All` extended right, from a source that is not a domain controller, is the definitive DCSync indicator. Is this in our SIEM detection rules?
- How many accounts in our domain have DCSync rights (specifically, the `DS-Replication-Get-Changes-All` extended right)? By default, this includes Domain Admins, Enterprise Admins, and domain controllers. Are there any other accounts with this right? (Check with: `Get-ADUser -Filter * -Properties * | Where-Object { $_.objectSid -match "..." }` or use BloodHound.)
- When was the last time our KRBTGT password was reset? If the answer is "never" or "we don't know," that means a Golden Ticket forged at any point in the domain's history could still be valid today. The KRBTGT password should be reset periodically (Microsoft recommends at least every 180 days) and always after a suspected domain compromise.
- Do we understand the operational impact of a KRBTGT double-reset? All existing Kerberos tickets (TGTs and service tickets) in the domain become invalid. Every user, service, and application will need to re-authenticate. This means brief disruptions to SSO, mapped drives, running applications, and background services. It's necessary but disruptive in some situations, so have we planned for it?

**Key takeaway to share:** "The KRBTGT account is the single most important account in your Active Directory. It's more important than the Domain Admin account because it's the key that makes ALL other keys. If you take away one action item from this exercise, it should be: go check when your KRBTGT password was last reset. If it's been more than 180 days, and especially if it's been *never* been reset, schedule a reset. The process for a controlled KRBTGT reset during a non-emergency is well-documented. Doing it calmly on a Tuesday morning is vastly preferable to doing it at 2 AM during an active incident."

---

**Break Point 4: Domain Admin to Persistent Access (AdminSDHolder)**

The attacker modified AdminSDHolder to ensure their permissions are automatically re-applied every 60 minutes.

Questions for the table:

- Before today, how many people at this table had heard of AdminSDHolder? How many knew what SDProp does? (Be honest, this is a learning exercise, not a test.)
- Is AdminSDHolder in our security monitoring scope? Do we have a SIEM rule that alerts when the AdminSDHolder object's `nTSecurityDescriptor` attribute is modified (Event ID 5136)?
- Do we have a known-good baseline of the AdminSDHolder ACL? Without a baseline, we can't detect unauthorized changes because we don't know what "normal" looks like.
- How many decommissioned service accounts are still active in our domain? The attacker used `svc-monitoring`, an account that should have been disabled six months ago. Stale accounts are backdoors waiting to be activated. When was the last time we audited service accounts?

**Key takeaway to share:** "AdminSDHolder is a masterclass in how attackers weaponize AD's own security mechanisms. The SDProp process was designed to *protect* privileged groups from accidental permission changes. The attacker turned it into an automatic re-infection mechanism. Most AD security audits don't check AdminSDHolder, it's one of those 'deep AD' objects that even experienced admins rarely examine. Adding AdminSDHolder ACL monitoring to your SIEM takes about 30 minutes to configure, and it provides detection for one of the stealthiest persistence techniques in the AD attacker playbook."

---

**Break Point 5: Data Access to Exfiltration (LOTS)**

The attacker exfiltrated gigabytes of sensitive data to legitimate cloud services without triggering any alerts.

Questions for the table:

- Do our domain controllers have unrestricted outbound internet access? Should they? A domain controller uploading data to Azure Blob Storage is abnormal, but only if our firewall is configured to notice. If DCs can reach any destination on port 443, we have no containment.
- Do we differentiate between legitimate and suspicious outbound connections based on the *source* machine? An HR workstation connecting to GitHub is anomalous. A developer workstation connecting to GitHub is normal. Do our security tools distinguish between these?
- Are we performing any egress data volume monitoring? Would we notice if 5 GB of data left our network to a single cloud destination in a two-hour window?
- Do we inspect or categorize TLS traffic to cloud services? If our web proxy is configured to bypass inspection for "trusted" domains like *.github.com and *.blob.core.windows.net, we're blind to LOTS exfiltration by design.

**Key takeaway to share:** "Living Off Trusted Sites is the evolution of 'living off the land.' Attackers aren't just using your legitimate tools, they're using your legitimate *internet destinations*. Every allowlist entry, every TLS inspection bypass, every 'trusted domain' exemption in your proxy is a potential exfiltration channel. The defense isn't to block GitHub and Azure since that breaks legitimate work. The defense is contextual analysis: which machines are talking to which destinations, in what volumes, at what times? That requires correlating network data with endpoint identity, which most organizations aren't doing."

---

### Part 3: Remediation Sequence

Walk through the full remediation in order. This section is critical because remediation after a Golden Ticket compromise is one of the most complex operations in incident response, and getting the sequence wrong means the attacker walks back in.

**Step 1: Contain the Initial Foothold**

Isolate Maria's workstation from the network. Disable her AD account. This kills the original Cobalt Strike beacon and prevents the attacker from using Maria's credentials further.

*But this is necessary and insufficient.* By this point, the attacker has the Golden Ticket and the AdminSDHolder backdoor. They no longer need Maria's machine or Maria's credentials. Containment at the endpoint buys time but it doesn't solve the problem.

**Step 2: Fix the ESC1 Certificate Template**

On the Certificate Authority server, open `certtmpl.msc` and edit the `UserAuthCert` template. Remove the `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT` flag (in the Subject Name tab, uncheck "Supply in the request"). Change the enrollment permissions from `Authenticated Users` to only the specific security groups that need this template.

While you're there, audit *every other template* for the same misconfiguration. The attacker found this one, but there may be others.

This step prevents the attacker from using the same escalation path again, but it doesn't invalidate the certificates or the Golden Ticket they already have.

**Step 3: Revoke the Malicious Certificate**

On the CA, locate the certificate issued to `administrator@corp.local` from Maria's workstation. Revoke it. Publish an updated Certificate Revocation List (CRL). Ensure that all systems that validate certificates are checking the CRL or OCSP responder.

Note: Certificate revocation in AD environments is notoriously unreliable. Many systems cache certificates and don't check revocation status aggressively. PKINIT authentication to domain controllers may or may not respect the CRL depending on configuration. Revocation is a defense-in-depth step, not a guaranteed fix.

**Step 4: Clean the AdminSDHolder ACL**

Using PowerShell or AD Explorer, remove the unauthorized ACE for `svc-monitoring` from the AdminSDHolder object. Then wait. SDProp runs every 60 minutes so you need to wait for the next SDProp cycle to propagate the cleaned ACL to all protected groups.

After 60 minutes, verify that the `svc-monitoring` ACE has been removed from Domain Admins, Enterprise Admins, Schema Admins, and every other protected group.

*Trap for the unwary:* If you clean Domain Admins but forget to clean AdminSDHolder, SDProp will re-apply the attacker's ACE within the hour. Always fix AdminSDHolder first and let SDProp do the propagation.

**Step 5: Disable and Investigate `svc-monitoring`**

Disable the `svc-monitoring` account immediately. Reset its password to a long random value. Remove it from all groups. Do NOT delete it yet, preserve it for forensic investigation. Review its activity logs: when was it last used? What did it access? Were there other accounts the attacker similarly co-opted?

This is also the time to audit all service accounts in the domain. How many are active but no longer needed? How many have passwords that haven't been changed in years? How many have excessive permissions? Each one is a potential persistence mechanism.

**Step 6: Reset the KRBTGT Password TWICE**

This is the big one. The KRBTGT double-reset invalidates all existing Golden Tickets.

The process:

1. Reset the KRBTGT password once using `Set-ADAccountPassword` or the Active Directory Users and Computers console.
2. Wait for the password change to replicate to ALL domain controllers. Verify with `repadmin /showrepl`. Every DC must show successful replication.
3. Reset the KRBTGT password a second time. Wait for replication again.

The first reset invalidates the current KRBTGT key. But Kerberos retains the *previous* key for backward compatibility during key rollover so a Golden Ticket forged with the old key would still work after a single reset. The second reset pushes the old key out of the history, fully invalidating all previously forged tickets.

**Operational impact:** Every existing Kerberos TGT and service ticket in the domain becomes invalid after the double-reset. Every user will be forced to re-authenticate. Running applications, mapped drives, SSO sessions, and background services that rely on cached Kerberos tickets will experience interruptions. SQL Server connections, IIS application pools, scheduled tasks running as service accounts all will need to re-authenticate. This typically isn't as big of a thing as it sounds, but it can be in some user populations.

**Recommendation for the table:** "In a real incident, plan the KRBTGT double-reset for a maintenance window if possible. Communicate to the business that a brief disruption is expected. Have your service desk ready for a spike in 'I got logged out' calls. Test the process in a lab environment first if you've never done it. And document everything since you'll want a detailed log of exactly when each reset occurred and when replication completed."

**Step 7: Force a Domain-Wide Password Reset**

The NTDS.dit was dumped. The attacker has the password hash for every account in the domain. Even though Kerberos tickets are now invalidated (from the KRBTGT reset), the attacker still has NTLM hashes that can be used for pass-the-hash attacks, credential stuffing against cloud services where employees reused their AD passwords, and offline cracking to recover plaintext passwords.

Force a password reset for all user accounts and service accounts in the domain. Prioritize Domain Admins, Enterprise Admins, service accounts, and any accounts with elevated privileges.

**Discussion question for the table:** "If all 500 of our employees need to reset their passwords simultaneously, do we have a process for that? Can our help desk handle the volume? Do we have a self-service password reset tool? What about service accounts? Do we know which services depend on which accounts and who knows the new passwords?"

**Step 8: Hunt for Additional Persistence**

The attacker had Domain Admin access for some period of time. During that window, they could have planted any number of additional persistence mechanisms. Now begins the hunt:

- **Scheduled tasks and cron jobs:** Check every domain controller and server for scheduled tasks created or modified during the attack window.
- **Group Policy Objects:** Review all GPOs for unauthorized modifications, especially logon scripts, startup scripts, and software installation policies.
- **Additional rogue accounts:** Search for user accounts created during the attack window, or existing accounts with newly elevated privileges.
- **Additional malicious certificates:** Review the CA's issued certificate database for any other certificates with suspicious SANs or enrollment patterns.
- **SAML/federation trusts:** If the environment uses ADFS or Azure AD Connect, check for modified trust configurations or SAML token-signing certificate export.
- **Skeleton Key / Directory Service Restore Mode (DSRM) password:** Advanced attackers may have installed a Skeleton Key on a DC (allowing authentication as any user with a universal password) or changed the DSRM password for direct DC access. These require memory forensics or DSRM password auditing to detect.
- **WMI event subscriptions:** Check for WMI permanent event subscriptions that persist across reboots and can execute arbitrary code.
- **Registry run keys and services:** Standard persistence locations that should be audited across all servers.

**Discussion question for the table:** "How confident are we that we found everything? The attacker had Domain Admin access, potentially for hours or days. Domain Admin can modify anything in the domain. Can we truly guarantee that we've found and removed every backdoor? What's the alternative if we can't guarantee that?" (The alternative, in extreme cases, is a full AD forest recovery from known-good backup, essentially rebuilding the domain from scratch. This is the nuclear option and should be discussed even if the team doesn't execute it.)

---

### Part 4: What Should We Do Monday Morning?

Transition the conversation from "what happened in the game" to "what should we actually do in our real environment." These are concrete, prioritized action items that the team can take away.

**This week (quick wins):**

- Run Certipy or PSPKIAudit against your Enterprise CA and review the output. Fix any ESC1/ESC8 misconfigurations immediately. This is a 15-minute scan and a 15-minute fix that eliminates one of the most powerful escalation paths.
- Check when your KRBTGT password was last reset. (`Get-ADUser krbtgt -Properties PasswordLastSet`). If it's been more than 180 days, schedule a controlled double-reset.
- Inspect the AdminSDHolder ACL and compare it against Microsoft's documented default. (`Get-ACL "AD:\CN=AdminSDHolder,CN=System,DC=yourdomain,DC=com" | Format-List`). Document the current state as a baseline.
- Audit active service accounts. Disable any that are no longer needed. Reset passwords on all remaining service accounts to long, random values.
- Verify that certificate enrollment events (Event IDs 4886, 4887) from the CA are being forwarded to your SIEM.

**This month (detection improvements):**

- Create SIEM detection rules for: DCSync (Event ID 4662 with replication rights from non-DC sources), PKINIT authentication (Event ID 4768 with pre-auth type 16), AdminSDHolder modification (Event ID 5136 targeting the AdminSDHolder DN), new MFA device registration (if cloud-connected), and certificate enrollment with SAN mismatch (4886/4887 where the SAN doesn't match the enrolling user).
- Deploy honeytoken accounts: create 2–3 fake service accounts with Kerberos SPNs and deliberately weak passwords. Monitor for any Kerberos TGS requests (Event ID 4769) targeting these accounts, any activity is definitively malicious.
- Review your email security configuration: are Office macros disabled for files downloaded from the internet? Are ASR rules enabled? Is your email gateway scanning embedded URLs?
- Audit egress traffic from domain controllers. Document which external destinations DCs legitimately need to reach (Windows Update, NTP, AV updates). Block everything else.

**This quarter (architectural improvements):**

- Implement a tiered administration model (Microsoft's Enterprise Access Model). Tier 0 (identity infrastructure: DCs, CA, ADFS) should only be accessible from dedicated Privileged Access Workstations (PAWs), not from standard user workstations.
- Evaluate Group Managed Service Accounts (gMSAs) as replacements for traditional service accounts. gMSAs have automatically rotated, 120-character random passwords that humans never know eliminating Kerberoasting and password-based service account compromise entirely.
- Deploy Microsoft Defender for Identity or a comparable AD monitoring solution that provides real-time detection of credential theft, lateral movement, and privilege escalation within AD.
- Conduct a full AD security assessment using tools like PingCastle, Purple Knight, or Trimarc Vision. These provide a comprehensive security posture score and identify the highest-risk misconfigurations.
- Review your incident response plan: does it include AD-specific runbooks for KRBTGT reset, AdminSDHolder cleanup, and forest recovery? If not, write them now AS during an incident is the wrong time to figure out the KRBTGT double-reset procedure.

---

### Part 5: Reflection Questions for the Table

Close the exercise with open-ended questions. Let the room talk. These questions don't have right answers as they are designed to surface assumptions, gaps, and priorities.

1. **"What surprised you most about this attack chain?"** Let people share their reactions. Many teams are surprised by the ESC1 escalation (one certificate request = Domain Admin) or the AdminSDHolder persistence (AD re-infects itself every hour).

2. **"Which break point in the chain would have been cheapest and fastest to fix?"** Guide the conversation toward the ESC1 template fix (15 minutes) versus the broader architectural changes (months). Help the team see that some of the highest-impact defensive improvements are simple configuration changes that just need someone to know to check.

3. **"If this happened tomorrow in our real environment, how confident are you that we'd detect it within 24 hours? Within a week? At all?"** Be honest. If the answer is "we wouldn't detect it," that's valuable to say out loud. It creates motivation for the action items above.

4. **"Who in our organization owns AD security?"** In many organizations, AD is managed by the infrastructure team, but monitored (or not) by the security team, with neither group taking ownership of AD-specific attack surface management. This question surfaces organizational gaps.

5. **"What was the hardest part of this exercise for our team?"** Was it the technical detection? The AD-specific knowledge? Knowing which Event IDs to search for? Coordinating between team members? Understanding the remediation sequence? The answers tell you where to focus training.

6. **"If you could only implement one defensive improvement from today's exercise, what would it be?"** Force prioritization. There's a long list of improvements above, but resources are finite. What's the single highest-value change?

---

## MITRE ATT&CK Mapping

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| Phish | T1566.001 (Spearphishing Attachment), T1059.001 (PowerShell) |
| ESC1 | T1649 (Steal or Forge Authentication Certificates) |
| DCSync / Golden Ticket | T1003.006 (DCSync), T1558.001 (Golden Ticket) |
| AdminSDHolder | T1222.001 (Permissions Modification), T1098 (Account Manipulation) |
| LOTS | T1567 (Exfiltration Over Web Service), T1102 (Web Service) |

---

## References

- Sean Metcalf's ADSecurity.org
- SpecterOps "Certified Pre-Owned" whitepaper https://specterops.io/blog/2021/06/17/certified-pre-owned/
- Backdoors & Breaches [https://www.backdoorsandbreaches.com](https://www.backdoorsandbreaches.com)
- Trimarc Expansion Deck for Backdoors & Breaches
