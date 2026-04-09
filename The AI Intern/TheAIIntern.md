# Backdoors & Breaches Campaign: The AI Intern
## Author: Steven Lorenz

*An employee plugged an unauthorized AI tool into your internal systems to "work smarter." The AI startup behind it just got breached. Every prompt, every document, every database query your team fed the tool is now in the hands of attackers along with the API keys that gave it access to your systems. Nobody in IT knew this tool existed. This scenario explores shadow AI, third-party data exposure, and what happens when convenience outpaces governance.*

While not based on a real event, drive home, "Could this happen in your organizaion"

---

## Who Is This Campaign For?

This campaign is designed for **mixed audiences** with the intent of being equally engaging for technical security teams, IT governance groups, legal and compliance professionals, and business leadership. The scenario touches on:

- Shadow IT and unauthorized tool adoption
- AI governance and data handling policies
- Third-party breach exposure
- API key management and secrets hygiene
- The human tension between productivity and security

Every department has someone who has done some version of what happens in this scenario. That's what makes it resonate.

If you've read through any of my other scenarios, you probably noticed that I redefine one of the procedures, Crisis Management. Feel free to not follow my lead on that.

---

## Background

The adoption of generative AI tools exploded across enterprises in 2024 and 2025. While many organizations established policies around approved AI platforms, the reality on the ground was messier. Employees across departments like sales, marketing, engineering, HR, legal began using a sprawling ecosystem of AI-powered tools, wrappers, plugins, and integrations, many of which were built by small startups with minimal security maturity.

These tools often require deep access to function: API keys to internal databases, OAuth tokens to cloud services, read access to email, CRM integrations, document repositories. Employees grant this access because the tool is useful and because nobody told them not to.

The risk isn't just what the AI tool does with the data. It's what happens when the *AI tool's provider* gets breached. Every prompt, every uploaded document, every API key stored in their infrastructure becomes part of the attacker's haul. And unlike a breach of your own systems, you have zero visibility into the AI vendor's security posture, no contractual notification obligations, and often no idea the tool was even in use.

---

## The Attack Chain

| Category | Card | Why This Card |
|---|---|---|
| **Initial Compromise** | **Trusted Relationship** | A sales operations analyst, Derek, signed up for an AI-powered sales intelligence startup called "DealMind AI." To make it work, he connected it to the company's Salesforce instance via OAuth, gave it read access to the customer database, and pasted in an API key for the company's internal data warehouse so DealMind could "enrich" pipeline data. He also routinely uploads internal pricing sheets, competitive analyses, and customer contract summaries into DealMind's chat interface to get AI-generated insights. DealMind's infrastructure was breached by attackers who stole the entire customer database including every API key, OAuth token, and uploaded document from every DealMind user. |
| **Pivot and Escalate** | **Credential Harvesting** | The attackers extract Derek's company's data warehouse API key from DealMind's breached database. This key has read access to the entire customer database, financial reporting tables, and employee records. They also find the Salesforce OAuth token, which gives them access to the full CRM, including the opportunity pipeline, customer contacts, deal values, and internal notes. The attackers don't need to "hack" anything. They just use the keys your employee already handed over. |
| **C2 and Exfil** | **HTTPS as Exfil** | All stolen data including customer records, pricing strategies, employee PII, financial reports, and the full contents of every document and prompt Derek uploaded to DealMind was exfiltrated over standard HTTPS to the attackers' infrastructure. From the company's perspective, there's nothing to detect on their own network. The data left through a third-party SaaS tool the company didn't know existed, to infrastructure the company doesn't control, via a channel the company never monitored. |
| **Persistence** | **Malicious Browser Plugin** | During the breach investigation, IT discovers that Derek also installed a DealMind browser extension that has permissions to read all browser data, access cookies, and inject content into web pages. This extension has been silently exfiltrating session cookies for every web application Derek logs into, including the internal HR portal, the finance dashboard, and the corporate wiki. Even after the API keys are revoked, the stolen cookies provide ongoing access. |


---

## Procedure Cards

Choose afew for the established Procedures:
| Actual Card Name | Mixed-Audience Name | What It Represents |
|---|---|---|
| **Crisis Management** | **"Get Everyone in the Room"** | Coordinating IT, legal, HR, sales leadership, and security to understand the scope and respond. |
| **SIEM** | **"Search the Audit Logs"** | Checking cloud service audit logs (Salesforce, data warehouse, Azure AD) for unauthorized access using the stolen credentials. |
| **User and Entity Behavior Analytics** | **"Track What Derek's Accounts Touched"** | Reviewing Derek's access patterns and identifying every system and data set the compromised credentials could reach. |
| **Endpoint Analysis** | **"Check Derek's Computer"** | Examining Derek's workstation for the browser extension and any other unauthorized tools. |

**Other Procedures:**
- Endpoint Security Protection Analysis / "Check Security Tools"
- Firewall Log Review / "Review Outbound Traffic"
- Isolation / "Revoke Everything"
- Server Analysis / "Examine the Data Warehouse"
- Internal Segmentation / "Audit All Third-Party Integrations"
- NetFlow / Zeek / RITA Analysis / "Check Network Traffic"

---

## Starting Narrative

*Read the following to the Defenders:*

> OK team, this one is going to get uncomfortable, so let's stay focused on the problem, not the blame.
>
> Twenty minutes ago, we received an email from a company called DealMind AI, a "sales intelligence platform" that claims to use AI to help sales teams analyze deals and generate insights. Their email says they experienced a security incident and that data belonging to users of their platform may have been exposed. They're offering free credit monitoring.
>
> Here's the problem: nobody in IT, nobody in security, and nobody in procurement has ever heard of DealMind AI. It's not on our approved vendor list. It hasn't been through a security review. There's no contract on file.
>
> But when we searched our Salesforce audit logs for the name "DealMind," we found it immediately. Someone granted an OAuth integration called "DealMind Sales Intelligence" read access to our entire Salesforce org four months ago. The authorization was performed by Derek in Sales Operations.
>
> We pulled Derek in. He's cooperating. He says he found DealMind through a LinkedIn ad, thought it was "really useful" for analyzing pipeline data, and connected it himself. He also says he gave it an API key for our data warehouse, with read access to customer records and financial tables, because DealMind needed it for "data enrichment." He's also been uploading internal documents including pricing sheets, competitive battle cards, customer contracts directly into DealMind's chat interface for months.
>
> Derek says half his team knows about it. He says three other people on the sales team also have DealMind accounts. He genuinely thought he was just "using an AI tool to be more productive."
>
> We don't know what DealMind's breach exposed. We don't know who has our API keys. We don't know if anyone has already used them. And we just found out that Derek also installed a DealMind browser extension. 
>
> What do we do?

---

## Incident Captain Guidance

### The Blame Trap

This campaign has a built-in emotional dynamic: Derek did something he shouldn't have, but he didn't do it maliciously. He was trying to be good at his job. The Incident Captain should steer the table away from a blamestorm and toward systemic questions: *Why was it so easy for Derek to connect an unauthorized tool? Why did nobody notice for four months? Why does a sales analyst have an API key with that much access?*

If someone at the table says "just fire Derek," push back gently: "Derek isn't the only one. He said three other team members also use it. And this is just one AI tool... how many others are being used across the company that nobody knows about?"

### How Actions Succeed or Fail

**"Get Everyone in the Room" (Crisis Management)** - *Can reveal any card.* This is the most important action because the response requires coordination across departments. IT needs to understand the technical exposure. Legal needs to assess notification obligations (customer PII may be involved). Sales leadership needs to identify every employee who used DealMind. Procurement needs to check whether any other unauthorized AI tools have been connected to corporate systems. If the defenders coordinate well and assign these workstreams in parallel, reward them generously. If they focus only on the technical side and ignore legal/business coordination, announce a consequence: "A reporter from TechCrunch just published a story about the DealMind breach. Your company is named in the article because DealMind's leaked database included your corporate domain. Your PR team found out from the article, not from you."

**"Search the Audit Logs" (SIEM)** - *Can reveal Pivot and Escalate.* If the defenders check Salesforce audit logs for the DealMind OAuth integration, they'll see four months of read access across the entire CRM, including contacts, opportunities, account records, internal notes. If they check data warehouse access logs for Derek's API key, they'll find normal DealMind activity (periodic queries enriching pipeline data) but also, starting 48 hours ago, a burst of queries from a different IP address pulling entire tables of customer PII and financial data. That's the attacker using the stolen key. **Key moment:** "The API key is still active right now. Every minute you spend discussing this, the attacker may be pulling more data."

**"Revoke Everything" (Isolation)** - *Can reveal Initial Compromise and stop the bleeding.* If the defenders immediately revoke the Salesforce OAuth token, rotate the data warehouse API key, and disable the DealMind browser extension, they cut off the attacker's access. This is the single most urgent action. However, the data that was already in DealMind's platform, every uploaded document, every prompt, every AI-generated summary is already beyond your reach. That data is in DealMind's infrastructure, which has been breached. You can't un-ring that bell. **Discussion moment:** "The pricing sheets, competitive analyses, and customer contracts Derek uploaded to DealMind over the past four months, where is that data now? Can we get it back? Does DealMind's terms of service even address data ownership?"

**"Check Derek's Computer" (Endpoint Analysis)** - *Can reveal Persistence.* Examining Derek's workstation reveals the DealMind browser extension with permissions to read all website data, access cookies, and modify page content. A forensic review shows the extension has been sending session cookies for every authenticated web application Derek visits to DealMind's telemetry endpoint, which is now attacker-controlled. This means the attackers potentially have session cookies for any internal application Derek accessed in his browser. **Follow-up question:** "Derek also logs into the HR portal, the finance dashboard, and the admin panel for our marketing automation platform through his browser. Should we treat all of those as compromised?"

**"Audit All Third-Party Integrations" (Internal Segmentation)** - *Can reveal scope beyond DealMind.* If the defenders audit all OAuth grants, API keys, and third-party integrations across their cloud services, they may discover that DealMind is just the tip of the iceberg. Marketing connected an AI content tool to the CMS. Engineering has an AI code assistant connected to the GitHub org. HR uploaded employee performance reviews to an AI summarization tool. The shadow AI footprint is much larger than anyone realized. **This is the real lesson.**

### Inject Card Guidance

- **Data Uploaded to Pastebin** - A sample of your customer database, complete with contact details and contract values, appears on a dark web forum with the tag "from DealMind AI breach, piping hot and verified fresh." Your customers' data is now public.
- **Bobby the Intern Kills the System You are Reviewing** - Bobby, hearing that "AI tools are being investigated," panics and deletes the AI code assistant extension from his machine, the one connected to the company's private GitHub repos. Evidence of what was shared with it is now gone. (yes, I seem to pick on Bobby in many of these)

---

## Discussion Points for Debrief

1. **Shadow AI Is the New Shadow IT:** Derek connected an AI tool to critical business systems without anyone's knowledge or approval. **Does your organization have a policy on AI tool usage? Does it cover API connections, data uploads, and browser extensions? Do employees know the policy exists and does it actually get enforced?**

2. **The Data You Feed the AI:** Every document, prompt, and query fed into a third-party AI tool is potentially stored on that vendor's infrastructure. **When an employee pastes a customer contract into an AI chat, where does that data go? Who owns it? What happens to it if the vendor gets breached?**

3. **API Key Management:** Derek had a data warehouse API key with read access to customer records and financial data. **Why did a sales analyst have that key? Was it scoped to minimum access? Was it rotated regularly? Was anyone monitoring its usage?**

4. **Third-Party Risk for AI Vendors:** DealMind is a startup. Startups often prioritize growth over security. **Do you apply the same vendor security assessment to a "free AI tool an employee signed up for" as you do to an enterprise software vendor? Should you?**

5. **The Convenience vs. Security Tension:** Derek used DealMind because it genuinely made him more productive. Banning all AI tools isn't realistic. **How does your organization balance enabling employees to use modern tools while maintaining security and governance? Is your approved AI tooling good enough that employees don't feel the need to go rogue?**

6. **Incident Response for Third-Party Breaches:** Most of the compromised data isn't on your systems, it's on DealMind's. **How do you conduct incident response when the breach happened at a vendor you didn't know you were using? What visibility do you have? What leverage do you have?**

---

## MITRE ATT&CK Mapping

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| Trusted Relationship (Shadow AI Integration) | T1199 (Trusted Relationship), T1528 (Steal Application Access Token) |
| Credential Stuffing (Stolen API Keys & OAuth Tokens) | T1552.001 (Unsecured Credentials: Credentials in Files), T1078.004 (Valid Accounts: Cloud Accounts) |
| Malicious Browser Plugin (Cookie Theft) | T1176 (Browser Extensions), T1539 (Steal Web Session Cookie) |
| HTTPS as Exfil (Third-Party Exfiltration) | T1567 (Exfiltration Over Web Service), T1530 (Data from Cloud Storage) |

