# Backdoors & Breaches Campaign: The Phantom Invoice
## Author: Steven Lorenz

---
>This scenario was from a late March, 2026 event. Modified rules apply, meaning that solves require a specific way of thinking outlined in the "why procedures succeed or fail" section. In different environments, you may use these as discussion prompts instead. Feel free to add to the story or change anything to fit your audience.
---
## For in person or at play.backdoorsandbreaches.com. 
I typically use the Core + Expansion 
For those who don't know me, I'm Steven, and I've led over 1000 Backdoors & Breaches sessions at this point. I am trying to write up some of the scenarios to share to hopefully help others leverage Backdoors & Breaches to teach or run tabletops. Some scenarios are based on real events, others are more "well, what if" scenarios. 

**Thank you to the Black Hills Information Security team for making and sharing this game with the community!** I stand on the shoulders of giants and often leverage tools made by others to help make training more enjoyable or impactful. 

Note: Sometimes I modify some rules. Where I do, I try to note that in the scenario. Original rules are here: https://www.blackhillsinfosec.com/tools/backdoorsandbreaches/ 

Play online at https://play.backdoorsandbreaches.com or purchase cards from the Spearphish General Store (https://spearphish-general-store.myshopify.com/collections/backdoors-breaches-incident-response-card-game)

---

*A scenario based on the 2025 wave of Adversary-in-the-Middle (AiTM) phishing attacks targeting Microsoft 365 environments. The attackers bypassed MFA, hijacked an executive's email session, and are now orchestrating a multi-million-dollar wire fraud all from inside your own tenant. Can your team detect a Business Email Compromise that looks like legitimate business activity?*

---

## Background

Business Email Compromise remains one of the most financially devastating attack categories, with losses exceeding $43 billion globally over the past several years according to the FBI's IC3. In 2025, these attacks evolved dramatically with the widespread adoption of Adversary-in-the-Middle (AiTM) phishing techniques and Phishing-as-a-Service (PhaaS) platforms like Tycoon2FA.

The attack works by placing a reverse proxy between the victim and the real Microsoft 365 login page. The victim completes a normal authentication flow, including MFA, but the proxy silently captures the resulting session token. The attacker then replays that token to access the mailbox directly, bypassing MFA entirely. Microsoft blocked over 13 million emails linked to Tycoon2FA in October 2025 alone.

Once inside, attackers register a new MFA device for persistence, create mailbox rules to hide their activity, and begin impersonating the compromised user to redirect financial transactions. By the time anyone notices, the wire transfer has cleared.

---

## The Attack Chain

| Category | Card | Why This Card |
|---|---|---|
| **Initial Compromise** | **Phishing** | The CFO receives a phishing email disguised as a SharePoint document-sharing notification from a trusted external partner (whose account was previously compromised). The link leads to an AiTM reverse proxy page that mirrors the real Microsoft 365 login. The CFO authenticates normally, including completing MFA, but the proxy captures the session token. |
| **Pivot and Escalate** | **Access Token Manipulation** | Using the stolen session token, the attacker accesses the CFO's Microsoft 365 account and immediately registers a new MFA authenticator device. They then use the CFO's delegated permissions and shared mailboxes to access finance team mailboxes and active deal correspondence, No additional password cracking required, just token-based lateral access through the M365 ecosystem. |
| **C2 and Exfil** | **Gmail, Tumblr, Salesforce, Twitter as C2** | The attacker uses the compromised Microsoft 365 account itself as the C2 and exfiltration channel. All fraudulent correspondence is sent from the CFO's legitimate email address. Sensitive financial documents, contracts, and banking details are forwarded to an external Gmail account controlled by the attacker. No separate C2 infrastructure is needed, the cloud productivity suite *is* the attack platform. |
| **Persistence** | **Malicious Email Rules** | The attacker creates Exchange Online inbox rules that automatically move incoming emails containing keywords like "wire," "payment," "invoice," and "fraud" into a hidden RSS Subscriptions folder or mark them as read and archive them. This ensures the real CFO never sees replies questioning the fraudulent payment instructions, and alerts about suspicious activity are suppressed. |

---

## Procedure Cards

Deal the following cards as the **Established Procedures** (these get the +3 modifier):

| Procedure Card | Rationale |
|---|---|
| **SIEM (Security Information and Event Management)** | Microsoft 365 audit logs and Entra ID sign-in logs are forwarded to a SIEM. |
| **User and Entity Behavior Analytics (UEBA)** | The organization has some form of identity analytics (e.g., Entra ID Protection or a third-party UEBA tool). |
| **Crisis Management** | The organization has a documented BEC response procedure including finance department coordination. |
| **Endpoint Analysis** | Endpoint forensic tools are available for the CFO's workstation. |

---

## Starting Narrative

*Read the following to the Defenders:*

> Team, we have a situation that started with a phone call from our bank twenty minutes ago. They called to verify a wire transfer request for $2.3 million, supposedly authorized by our CFO, Katherine Chen, to a new vendor account. Our treasury team says this doesn't match any approved transactions, and the beneficiary bank account is in Singapore, a jurisdiction we've never wired funds to before.
>
> We pulled Katherine in, and she's confused. She says she didn't authorize any wire transfer, but when we checked her Sent Items folder in Outlook, there is a thread going back four days between her email address and someone at our acquisition target's legal counsel, discussing adjusted payment terms. Katherine says she didn't write those emails. She doesn't recognize the thread.
>
> Here's where it gets stranger: Katherine mentioned that three days ago, she received a SharePoint link from a partner firm, clicked it, and logged in to view the document. She says the document seemed blank and she forgot about it.
>
> Finance has placed a hold on the wire. But we don't know who is in Katherine's account, what else they've done, what they've seen, or whether other accounts are compromised. We need answers... now.

---

## Incident Captain Guidance

### Why Procedures Succeed or Fail

**SIEM** - *Can reveal Initial Compromise or Persistence.* If the defenders query the Microsoft 365 Unified Audit Log or Entra ID sign-in logs for Katherine's account, they should find anomalous sign-in events: a successful authentication from Katherine's usual IP (the legitimate login through the AiTM proxy) immediately followed by token replay activity from a different IP address, potentially an anonymizing VPN or foreign ISP. If they search for Exchange Online `New-InboxRule` or `Set-InboxRule` operations, they'll find the malicious inbox rules — revealing the Logon Scripts persistence. If the defenders only search for failed logins or traditional brute-force patterns, they'll find nothing as the attacker never failed a login. They walked in with a valid token.

**UEBA** - *Can reveal Pivot and Escalate.* Identity analytics should flag several anomalies: a new MFA device registered from an unusual IP, access to shared mailboxes or delegated resources that Katherine rarely opens, and email-sending patterns that deviate from her historical baseline (sending emails at unusual hours, to unusual recipients, with unusual attachment types). If the defenders invoke UEBA early and know what to look for, this is the fastest path to understanding the scope of the compromise. If they only check Katherine's workstation-level behavior, UEBA may not provide cloud-specific insights.

**Crisis Management** - *Can reveal any card.* This is a BEC scenario where business process controls are as important as technical investigation. If the defenders coordinate with the finance department to review all outbound wire transfer requests from the past week, they may discover additional fraudulent payment attempts confirming the scope. If they coordinate with legal to engage the bank for a funds recovery hold, they buy time. If they contact the external partner firm to verify whether the SharePoint notification was legitimate, the partner may confirm their own account was compromised, directly pointing to the phishing Initial Compromise. Reward teams that think about business process alongside technical investigation.

**Endpoint Analysis** - *Can reveal Initial Compromise or Persistence.* If the defenders forensically examine Katherine's workstation, they can find the browser history showing the phishing URL (likely a lookalike domain or URL shortener redirect). They may find cached credentials or cookie artifacts. However, because this is a cloud-based attack, much of the evidence lives in Microsoft 365 audit logs rather than on the endpoint. Endpoint Analysis is useful but not the primary evidence source here.

**Firewall Log Review (Other Procedure, no modifier)** - *Can reveal C2 and Exfil.* If the defenders review outbound email connections or check for auto-forwarding rules sending email to external addresses (the Gmail account), they can discover the exfiltration path. Some organizations route Microsoft 365 traffic through a CASB or proxy that logs email forwarding destinations, if so, the external Gmail forward is visible.

**Isolation (Other Procedure, no modifier)** - *Critical but dangerous timing.* If the defenders disable Katherine's Microsoft 365 account or revoke all active sessions, they immediately cut off the attacker's access, but they also alert the attacker that they've been discovered, and any evidence in active sessions is lost. **The Incident Captain should challenge the defenders:** Have you gathered enough evidence first? Have you identified all compromised accounts, or just Katherine's? If they isolate too early, they may miss that the attacker has already pivoted to other accounts.

---

## Discussion Points for Debrief

1. **MFA Is Not Enough:** This attack succeeds even though the CFO had MFA enabled. Is your organization using *phishing-resistant* MFA (FIDO2/WebAuthn security keys) for privileged accounts? Do you understand the difference between MFA and phishing-resistant MFA?

2. **Token Theft and Session Management:** Once a session token is stolen, resetting the user's password does not revoke the attacker's access. Does your team know how to revoke active sessions and refresh tokens in Entra ID? Is Continuous Access Evaluation (CAE) enabled?

3. **Inbox Rule Monitoring:** Malicious inbox rules are a hallmark of BEC persistence. Do you actively monitor for new inbox rules, especially those that delete or redirect mail? Can you query the Exchange Online audit log for `New-InboxRule` events?

4. **Financial Process Controls:** The ultimate goal of BEC is financial fraud. Does your organization require out-of-band verification (phone call to a known number, not one provided in the email) for changes to payment instructions or large wire transfers?

5. **Cloud-Native Detection:** Almost all of this attack occurred inside Microsoft 365 with no on-premises footprint. Is your SOC equipped to investigate cloud-native incidents, or is your tooling and training primarily focused on endpoint and network detection?

---

## MITRE ATT&CK Mapping

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| Phish (AiTM Token Theft) | T1566.002 (Phishing: Spearphishing Link), T1557 (Adversary-in-the-Middle), T1528 (Steal Application Access Token) |
| Credential Stuffing (Token Reuse + MFA Manipulation) | T1078.004 (Valid Accounts: Cloud Accounts), T1098.005 (Account Manipulation: Device Registration) |
| Logon Scripts (Inbox Rules) | T1564.008 (Hide Artifacts: Email Hiding Rules), T1114.003 (Email Collection: Email Forwarding Rule) |
| Gmail/Cloud Services as C2 | T1567 (Exfiltration Over Web Service), T1114.002 (Email Collection: Remote Email Collection) |

---
 
## References and additional data

- Microsoft Security Blog on multi-stage AiTM phishing and BEC campaigns abusing SharePoint https://www.microsoft.com/en-us/security/blog/2026/01/21/multistage-aitm-phishing-bec-campaign-abusing-sharepoint/
- Microsoft Security Blog on domain spoofing via complex routing misconfigurations https://www.microsoft.com/en-us/security/blog/2026/01/06/phishing-actors-exploit-complex-routing-and-misconfigurations-to-spoof-domains/
- ThreatLocker analysis of AiTM attacks against Microsoft 365 https://www.threatlocker.com/blog/aitm-phishing-attacks-against-microsoft-365-mfa-bypasses-session-hijacking-and-bec
- Tycoon2FA PhaaS platform analysis https://www.microsoft.com/en-us/security/blog/2026/03/04/inside-tycoon2fa-how-a-leading-aitm-phishing-kit-operated-at-scale/
- MITRE ATT&CK Techniques T1557, T1528, T1564.008
- Backdoors & Breaches by Black Hills Information Security - [https://www.backdoorsandbreaches.com](https://www.backdoorsandbreaches.com)
