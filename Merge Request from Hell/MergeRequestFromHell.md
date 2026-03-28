# Backdoors & Breaches Campaign: Merge Request from Hell

*A scenario loosely based on the 2025 wave of CI/CD supply chain attacks exploiting GitHub Actions and developer infrastructure. An attacker has compromised a popular open-source dependency used in your build pipeline, injecting malicious code that exfiltrates secrets during every deployment. No server was hacked. No endpoint was breached. The poison is in your software factory. Can your team trace a supply chain compromise back to its source?*

---

## Background

Software supply chain attacks through CI/CD pipelines have become one of the most potent and difficult-to-detect attack vectors in modern environments. In 2025, multiple campaigns exploited GitHub Actions workflows, poisoned open-source packages, and compromised build infrastructure to inject malicious code into downstream applications.

The pattern is consistent: an attacker compromises a widely used open-source library or GitHub Action either through a stolen maintainer token, a typosquatted package, or a subtle pull request that passes code review. The malicious payload activates only during the CI/CD build process, where it harvests environment variables (which commonly contain API keys, cloud credentials, and deployment tokens), exfiltrates them to an attacker-controlled endpoint, and often modifies the build artifacts to include a persistent backdoor.

For the victim organization, the compromise doesn't look like a hack. It looks like a normal software deployment. Their own CI/CD pipeline built it, tested it, signed it, and shipped it. The malicious code was never on a developer's workstation, it existed only in the ephemeral build environment.

---

## The Attack Chain

| Category | Card | Why This Card |
|---|---|---|
| **Initial Compromise** | **Supply Chain Attack** | The attacker compromises the GitHub account of a maintainer of a widely used open-source JavaScript utility package included in the organization's application dependencies. A malicious update is published that includes code which activates only when the `CI=true` environment variable is set (indicating a CI/CD build environment). The organization's automated dependency update tool pulls the new version into a pull request, and it is merged after routine review. |
| **Pivot and Escalate** | **New Service Creation** | During the next CI/CD build, the malicious dependency executes, reads environment variables from the GitHub Actions runner (including AWS deployment credentials, database connection strings, and API tokens for internal services), and posts them to an attacker-controlled HTTPS endpoint disguised as a telemetry call. The attacker uses the stolen AWS credentials to provision a new EC2 instance within the organization's cloud account or a "new service" running the attacker's tooling inside the victim's own infrastructure. |
| **C2 and Exfil** | **Domain Fronting as C2** | The web shell communicates through a domain-fronting technique: outbound requests appear to go to a legitimate CDN hostname (e.g., a popular cloud provider's CDN), but the actual traffic is routed to the attacker's backend server. This makes the C2 traffic nearly impossible to distinguish from normal CDN requests at the network layer. |
| **Persistence** | **Accessibility Features** | The attacker modifies the organization's production application by injecting a hidden administrative endpoint (a web shell accessible via a specific URL path and parameter combination). This modification was introduced through a second, more subtle commit to the compromised dependency that alters build output. Because it was deployed through the normal CI/CD pipeline, it passed automated testing, was code-signed, and is treated as a legitimate production artifact. |

---

## Procedure Cards

Deal the following cards as the **Established Procedures** (these get the +3 modifier):

| Procedure Card | Rationale |
|---|---|
| **Server Analysis** | The team can analyze production servers and cloud infrastructure. |
| **SIEM (Security Information and Event Management)** | Cloud audit logs (AWS CloudTrail, GitHub audit logs) are forwarded to the SIEM. |
| **Isolation** | Because sometimes you just need a card that isn't helpful. |
| **Endpoint Security Protection Analysis** | EDR-equivalent monitoring is deployed on production servers and CI/CD runners. |


---

## Starting Narrative

*Read the following to the Defenders:*

> Alright team, here's what we know. Yesterday at 4:30 PM, our cloud operations team received an automated AWS Cost Anomaly alert. Our monthly EC2 spend spiked by $1,800 overnight. When they investigated, they found a running EC2 instance in us-east-1 that nobody on the team provisioned. It's a t3.xlarge running Amazon Linux, and it was launched using our production deployment IAM role credentials.
>
> The instance was launched at 2:17 AM last night. It has an attached security group allowing inbound SSH from a single external IP address and outbound access on port 443. Our ops team says the deployment role credentials shouldn't be available outside our GitHub Actions runners.
>
> This morning, our application security team was doing a routine review and found something else: a URL path in our production web application that doesn't match anything in our codebase. Hitting the path returns a blank page, but the response headers are unusual. They swear it wasn't there last week.
>
> The last production deployment was two days ago. It went through our standard pipeline and all tests passed, code review was approved, deployment was automated. Everything looked normal.
>
> Something is very wrong, and we think these two findings might be connected. We need you to figure out what happened and how it got past our pipeline.

---

## Incident Captain Guidance

### Why Procedures Succeed or Fail

**Network Threat Hunting** - *Can reveal Initial Compromise.* If defenders look for anomalous connections, they may find the traffic to abnormal CDNs and web locations outside of the baseline. 

**Server Analysis** - *Can reveal Pivot and Escalate or Persistence.* If the defenders examine the rogue EC2 instance, they'll find attacker tooling, SSH keys, and evidence of the stolen credentials being used — revealing the New Service Creation pivot. If they examine the production application server and compare the deployed artifacts against the expected build output (e.g., diffing the production bundle against a clean local build), they can discover the injected web shell. If they only look at server configurations and access logs without comparing build artifacts, the web shell may be missed — it looks like part of the legitimate application.

**SIEM** - *Can reveal Pivot and Escalate.* If the defenders query AWS CloudTrail for API calls made with the production deployment role, they'll see the EC2 `RunInstances` call from an IP address that doesn't match any GitHub Actions runner IP range. If they query GitHub audit logs for recent dependency updates and workflow runs, they can trace the timeline to the malicious dependency update and the build where secrets were exfiltrated. If they're searching for traditional login anomalies or malware detections, the SIEM is quiet — the CI/CD pipeline behaved exactly as designed.

**Firewall Log Review** - *Can reveal C2 and Exfil.* If the defenders review outbound traffic from the CI/CD runners during the build window, they may find an unexpected HTTPS POST to an external endpoint that occurred during the dependency installation or build step. This is the secret exfiltration call. For the C2 channel, traffic from the production web servers to CDN endpoints should be compared against expected CDN targets — domain-fronted traffic will show a mismatch between the SNI hostname and the actual destination at the TLS layer (visible with TLS inspection or JA3 fingerprinting). If the defenders only look at production server traffic and ignore CI/CD runner traffic, they'll miss the initial exfiltration.

**Endpoint Security Protection Analysis** - *Can reveal Initial Compromise or Persistence.* If the CI/CD runners have runtime monitoring, the EDR-equivalent may have recorded the malicious dependency executing system commands or reading environment variables during the build process. However, many organizations treat CI/CD runners as ephemeral and don't monitor them with the same rigor as production servers — the Incident Captain should ask whether the defenders' monitoring covers their build environment. On the production server, EDR might flag the web shell if it detects unusual process spawning from the web server process.

**UEBA (Other Procedure, no modifier)** - *Can reveal Pivot and Escalate.* Behavioral analytics on the AWS deployment role would show anomalous API call patterns: the role suddenly making `RunInstances` calls when it normally only performs `ECS` or `Lambda` deployment operations.

**Isolation (Other Procedure, no modifier)** - *Can support containment.* If the defenders terminate the rogue EC2 instance and rotate the compromised AWS credentials, they sever the attacker's cloud access. If they also roll back the production deployment to the previous known-good version, they remove the web shell. However, if they don't also address the poisoned dependency, the *next* CI/CD build will re-compromise the environment.

---

## Discussion Points for Debrief

1. **CI/CD Pipeline as an Attack Surface:** Do you treat your CI/CD runners as security-critical infrastructure? Are secrets available to build steps audited, rotated, and scoped to minimum necessary permissions? Would you detect if a build step exfiltrated environment variables?

2. **Dependency Management:** How does your team vet open-source dependencies and updates? Do you pin dependency versions? Do you use lock files? Do you review changes when updating dependencies, or do you auto-merge Dependabot PRs?

3. **Build Artifact Integrity:** Could you detect if a production deployment contained code that wasn't in your source repository? Do you compare build outputs against expected baselines? Do you use reproducible builds or artifact signing?

4. **Secrets in CI/CD:** What secrets are accessible to your CI/CD pipeline? If a build step was compromised, what's the blast radius? Are deployment credentials short-lived, scoped, and rotated or are they long-lived IAM keys sitting in environment variables?

5. **Domain Fronting Detection:** Can your network monitoring distinguish between legitimate CDN traffic and domain-fronted C2? Do you inspect TLS metadata (SNI vs. Host header) on outbound connections from production?

---

## MITRE ATT&CK Mapping

| Attack Chain Step | MITRE ATT&CK Technique |
|---|---|
| Trusted Relationship (Supply Chain) | T1195.002 (Supply Chain Compromise: Compromise Software Supply Chain), T1199 (Trusted Relationship) |
| New Service Creation (Cloud Pivot) | T1578.002 (Modify Cloud Compute Infrastructure: Create Cloud Instance), T1552.001 (Unsecured Credentials: Credentials in Files) |
| Accessibility Features (Web Shell via Pipeline) | T1505.003 (Server Software Component: Web Shell), T1195.002 (Supply Chain Compromise) |
| Domain Fronting as C2 | T1090.004 (Proxy: Domain Fronting), T1071.001 (Application Layer Protocol: Web Protocols) |

---

## References and additional data

- GitHub Actions supply chain attacks and runner secret exfiltration campaigns like https://www.aikido.dev/blog/github-actions-incident-shai-hulud-supply-chain-attack
- SalesLoft supply chain incident affecting Salesforce-integrated organizations https://cloud.google.com/blog/topics/threat-intelligence/data-theft-salesforce-instances-via-salesloft-drift
- npm and PyPI malicious package campaigns targeting CI/CD environments https://www.paloaltonetworks.com/blog/cloud-security/npm-supply-chain-attack/
- CISA guidance on securing software supply chains and CI/CD pipelines https://www.cisa.gov/resources-tools/resources/securing-software-supply-chain-recommended-practices-guide-customers-and
- MITRE ATT&CK Techniques T1195.002, T1578.002, T1505.003, T1090.004
- Backdoors & Breaches by Black Hills Information Security — [https://www.backdoorsandbreaches.com](https://www.backdoorsandbreaches.com)
