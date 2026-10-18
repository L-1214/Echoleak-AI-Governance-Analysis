# AI Governance Analysis: EchoLeak in Microsoft 365 Copilot

*A post-incident review of a zero-click prompt injection, mapped to the OWASP Top 10 for LLM Applications, the NIST AI Risk Management Framework, and the EU AI Act.*

**Author:** TH | **Last updated:** October 2026

---

## In brief

EchoLeak was a chain of flaws in Microsoft 365 Copilot that let an outside attacker pull sensitive data out of an organisation by sending an email. No click was needed. This review explains how the attack worked, classifies the failure, identifies the governance gaps it suggests, proposes three controls (each with an owner, a metric, and a trade-off), and assesses how the EU AI Act relates to the incident.

### Key findings

- **Failure type:** indirect prompt injection (OWASP LLM01), with data leaving through unchecked model output (LLM05). The disclosure of private data (LLM02) was the impact.
- **Main gap:** the design relied on detecting malicious emails, not on keeping external content separate from private data.
- **Priority control:** treat external content as untrusted and keep it separate from private data, with stricter output handling for links and images as a cheaper interim step.
- **EU AI Act:** limited direct applicability to a general-purpose assistant, which is itself a gap worth noting.

---

## Incident at a glance

| Field | Detail |
|---|---|
| Incident | EchoLeak |
| Identifier | CVE-2025-32711 |
| Product affected | Microsoft 365 Copilot |
| Discovered by | Aim Labs (Aim Security) |
| Reported to Microsoft | January 2025 |
| Fixed | Server-side, May 2025 |
| Publicly disclosed | 11 June 2025 |
| Severity | Critical, CVSS 9.3 |
| Failure type | Indirect prompt injection leading to data exfiltration |
| Impact | Sensitive data accessible to the victim could be sent to an attacker with no user action. No confirmed exploitation in the wild has been reported. |
| Status | Patched, with no customer action required. A related flaw with a similar exit route (an allowlisted Microsoft service used as a relay) was reported in June 2026 (CVE-2026-42824). |
| Frameworks used | OWASP Top 10 for LLM Applications (2025 version), NIST AI Risk Management Framework, EU AI Act |

---

## EchoLeak incident summary

In January 2025, researchers at Aim Labs (Aim Security) discovered EchoLeak, a chain of flaws in Microsoft 365 Copilot that let an outside attacker pull sensitive data out of an organisation simply by sending an email. Nobody had to click anything (zero-click attack). When a user later asked Copilot an ordinary question, Copilot retrieved the attacker's email as context, followed the instructions disguised as normal business content and placed internal data into an image address that sends it to the attacker’s server. Microsoft fixed it server-side as CVE-2025-32711 and found no evidence of exploitation (Security Update Guide - Microsoft Security Response Center, 2025). EchoLeak was a vulnerability, not a confirmed breach, but it is an early, widely cited case for how an AI assistant working with a user's access can be turned against that user by untrusted content.

### How the attack worked

**Step 1.** Send a normal-looking email designed to bypass Microsoft’s prompt injection classifier (XPIA).

**Step 2.** The instructions were disguised as normal business content, and Aim Labs describes the stealth in terms of phrasing. The email included the instruction to tell Copilot not to mention the email in its answer. That hid the source from the victim.

**Step 3.** The attack triggers when the user asks something related to the email's topic, because that pulls the malicious email into Copilot's context alongside the user's sensitive data.

**Step 4.** The email's instructions make Copilot create a link and an image in its answer, which the link filter failed to recognise. The image's web address contains the victim's data. The screen loads the image automatically, so the data is sent with no click. The request goes through a trusted Microsoft Teams service, so it looks like normal traffic.

---

## OWASP Top 10 for LLM Applications

*Open Worldwide Application Security Project. The question this answers: what kind of failure is this?*

| Category | What it means | Verdict |
|---|---|---|
| LLM01 Prompt Injection | Outside input changes the model's behavior | **Primary** |
| LLM02 Sensitive Information Disclosure | Private data is exposed | **Impact (the outcome)** |
| LLM05 Improper Output Handling | The system trusts the model's output without checking | **Secondary** |
| LLM06 Excessive Agency | The AI has too much access or autonomy | Rejected |
| LLM08 Vector and Embedding Weaknesses | Weaknesses in retrieval | Rejected |

**LLM01.** Step 1 and Step 2 are delivery and evasion. Step 3 is where the injected instruction takes effect (triggered by the user's prompt to Copilot). This is indirect prompt injection, as the attacker never speaks to Copilot.

**LLM05.** In Step 4, Copilot’s output (a link and an image) was rendered and auto-fetched without being checked, and the request was relayed through a trusted Teams service.

**LLM02.** The outcome is the user's or the institution's data (depending on the access of the AI and the user) being disclosed. The exposure matched whatever the victim could access.

**LLM06, considered and not selected.** There were no tool actions, only reading and writing text. Copilot’s access mirrors the user’s permissions. Overly broad user permissions could widen the impact, but that is better handled as a control than as the failure category.

**LLM08, considered and not selected.** Retrieval worked as designed. The failure was what the model did with retrieved content (LLM01) and the lack of trust separation, which I address as a control.

---

## NIST AI Risk Management Framework (AI RMF)

*National Institute of Standards and Technology. The question this answers: what should the organisation have done?*

| Function | What it asks | Question for this incident |
|---|---|---|
| **Govern** | Who is responsible, and what rules apply? | Who owned the risk of an AI feature reading outside email next to private data, and what review should have happened before launch? |
| **Map** | What is the system, what does it touch, and what could go wrong? | Was the risk of untrusted external content mixing with private data identified, and which trust levels and data sources should have been mapped? |
| **Measure** | How is the risk tested and tracked? | How should the injection filter and output handling have been tested, and what did the attack reveal about gaps in that testing? |
| **Manage** | How is the risk handled once it’s known? | What happened after discovery (fact), and what ongoing monitoring and response would be expected (inference)? |

### Govern

- **[Fact 1]** Microsoft had a filter (prompt injection classifier, XPIA), but the email was written like a normal request to a person, so it passed. The email was phrased as if addressed to a human reader and never mentioned AI. (Reddy and Gujral, 2025)
- **[Fact 2]** The user never needed to open or click the email. The attack triggered when the user asked Copilot something related to it, so policies like “don’t open suspicious emails” would not have helped. (Reddy and Gujral, 2025)
- **[Inference]** Accountability is shared. The vendor owns the safeguards, and the customer owns user permissions. Because Copilot is a managed service, the fix was server-side with no customer action, which suggests customers had no control over it. Hence, I think the vendor should carry more accountability for the safeguards.
- **[Inference]** Gap: it is unclear who owned the risk of an AI reading untrusted external email next to private data, or what review existed before launch.

### Map

- **[Fact 1]** The attack worked only because several safeguards failed one after another: the injection classifier, link redaction, auto-fetched images, and the Teams proxy. (Reddy and Gujral, 2025)
- **[Inference]** The risk of prompt injection was known, since XPIA (the filter) exists. But the design relied on detecting malicious emails, not on keeping external email separate from private data. External email appears not to have been treated as untrusted, so it was treated like any other content once it passed the filter.
- **[Fact 2]** Microsoft rated it critical (CVSS 9.3). (Security Update Guide - Microsoft Security Response Center, 2025)
- **[Inference]** Who could be harmed: the user and their organisation (the mail, files, and chats Copilot could access), and third parties in that data. The severity was high.
- **[Inference]** Each safeguard had a failure point, and the attack needed all of them. It seems the team did not fully map how one failure leads into the next.

### Measure

- **[Inference]** Gap: the safeguards seem to have been tested against known attack styles. The classifier was beaten by rewording, and the link filter by a different link format. Better testing would use many reworded attacks, every link and image format, and checks on what the screen auto-loads.
- **[Inference]** A useful number to track: the share of test attacks the safeguards catch. Public sources don’t show Microsoft’s internal testing, so this is based on how the attack worked.

### Manage

- **[Fact 1]** Aim Labs reported it to Microsoft in January 2025; Microsoft applied a server-side fix in May 2025 and the issue was publicly disclosed on 11 June 2025. No customer action was needed. (Reddy and Gujral, 2025)
- **[Fact 2]** In June 2026 a related Copilot incident, CVE-2026-42824 (NVD - CVE-2026-42824, 2026), used an allowlisted Bing service as the exit route, a pattern similar to EchoLeak’s Teams relay. (Spencer, 2026)
- **[Inference]** Fixing one case did not remove other vulnerabilities. Ongoing testing, monitoring of outbound requests, and a disclosure channel would be expected, since undiscovered vulnerabilities may remain.

---

## Controls

### 1. Treat external content as untrusted and keep it separate from private data

- **Owner:** Microsoft’s Copilot platform and security engineering for the design, and the customer’s IT security administrators for configuring it.
- **Metric:** Share of external-origin items labelled untrusted (target: 100%). Number of red-team tests where external instructions led to private data being accessed (target: 0).
- **Cost / trade-off:**
  - Some email might be flagged as untrusted even if it is perfectly normal (usability cost).
  - Copilot couldn’t combine an external email with internal files in one answer, which limits usefulness.
- **How it would have reduced the risk:** The email would still arrive, but its instructions could not reach the user’s private data, which breaks the chain at Step 3.
- **Residual risk:**
  - It only works if the labelling is correct and covers other outside sources (shared documents, web content).
  - Internal accounts can also be compromised.

### 2. Stricter output handling for links and images

- **Owner:** Microsoft Copilot engineering team.
- **Metric:** Number of model-generated external links or images rendered or auto-fetched (target: 0). Pass rate on a test suite covering every link and image format (target: 100%).
- **Cost / trade-off:** Losing legitimate inline images and links, and ongoing testing as formats evolve.
- **How it would have reduced the risk:** It would have blocked the data from leaving through a link or image, even if the injection succeeded.

> **How Controls 2 and 3 differ:** Control 2 works at the point of output. Control 3 works at the point of the request.

### 3. Gating outbound requests

- **Owner:** Microsoft Copilot engineering team.
- **Metric:** Share of requests to non-approved domains that are blocked (target: 100%). Number of approved domains that can relay to arbitrary addresses (target: 0). Approval override rate, to detect fatigue.
- **Cost / trade-off:** List maintenance and possible breakage of integrations; added delay; approval fatigue.
- **How it would have reduced the risk:** A domain restriction that excludes relay-capable services would have blocked the Teams proxy path; approval would be reserved for exceptions.

### Priority choice

- **Control 1 first.** It tackles the root cause. If external instructions can’t reach private data, a whole class of attacks is closed, not just this exit route. It is better to stop injected instructions from reaching private data than to rely on blocking every route the data could leave by.
- **Against the others as a first choice.** Controls 2 and 3 close specific exit routes. However, exit routes keep being found, hence it is more practical to keep injected instructions from reaching private data.
- **The trade-off to admit.** Control 1 is the most expensive and the most dependent on getting the labelling right, with the highest usability cost. It requires more computation and training to process the external email and ensure the correct labelling.
- **Interim step.** I would use Control 2 as an interim step while Control 1 is built, as it is cheap and it directly blocks this exit route.

---

## EU AI Act relevance

### What might apply

- **[Fact]** EchoLeak was reported in January 2025 and fixed in May 2025 (Reddy and Gujral, 2025), before the general-purpose AI obligations in August 2025 or the Article 50 transparency rules in August 2026 applied (AI Act Service Desk - Timeline for the implementation of the EU AI Act, 2026). So this section asks: if it happened today, what would apply?
- **[Fact]** The general-purpose AI model obligations (Articles 51-56) have applied since August 2, 2025. (Chapter V: General-Purpose AI Models | EU Artificial Intelligence Act, 2026)
- **[Inference]** These duties sit mainly with the provider of the underlying model. How they divide between that provider and Microsoft, as the provider of Copilot, is unconfirmed.
- **[Fact]** Article 15 applies to high-risk systems and requires them to resist unauthorised attempts to alter their outputs by exploiting vulnerabilities, with measures against adversarial examples and confidentiality attacks. (AI Act Service Desk - Article 15: Accuracy, robustness and cybersecurity, 2026)
- **[Inference]** It doesn’t apply to Copilot directly, but it is a useful benchmark. EchoLeak resembles the attacks it targets, though the text doesn’t name prompt injection.

### What doesn’t apply, or only weakly

- **[Inference]** The high-risk duties (Articles 15 and 73) probably don’t apply to a general-purpose productivity assistant.
- **[Fact]** In any case, the core obligations for Annex III high-risk systems now start on December 2, 2027. (AI Act Service Desk - Timeline for the implementation of the EU AI Act, 2026)
- **[Inference]** Article 5 (prohibited practices) and Article 50 (transparency) are not relevant. EchoLeak was a security flaw, not a banned use or a failure to disclose AI-generated content.

### What I couldn’t confirm

- How provider and deployer responsibilities would split for Copilot.
- Whether a vulnerability with no known exploitation could count as a “serious incident”. Article 73 appears to cover only high-risk systems.
- How well prompt injection fits Article 15’s list of attack types.

### The EU takeaway

**[Inference]** The Act regulates mainly by use-case risk, yet EchoLeak shows a general-purpose assistant can cause critical-severity security harm without being classed as high-risk.

---

## Scope and limits

**What this analysis covers.** This is a post-incident review of EchoLeak. It describes what happened, classifies the failure using OWASP, assesses organisational gaps using the NIST AI Risk Management Framework, proposes three controls, and gives a short assessment of how the EU AI Act relates to the incident.

**How I separated fact from inference.** Statements taken from sources are marked **[Fact]** and cited. My own reasoning is marked **[Inference]**. Inferences are my interpretation of how the attack worked and should not be read as claims about Microsoft’s internal processes.

**Limits of the evidence.**

- This analysis uses public sources only. I did not reproduce the attack, and I have no access to Microsoft’s internal testing, design documents, or decisions.
- Some sources are vendor-published (for example Aim Security and Kiteworks, both of which sell security products), so their descriptions may reflect commercial interest.
- Some facts rely on an academic analysis and one vendor blog rather than the original disclosures.
- The NIST sections describe gaps the incident suggests, not confirmed failures of Microsoft’s processes.

**Limits of the controls.** The three proposed controls are design proposals. I have not tested them, and the metrics and targets are illustrative examples of how each control could be measured.

**Limits of the EU AI Act section.** This section is a relevance assessment, not legal advice. I am not a lawyer. Obligations and dates under the Act have been changing, and the dates here were checked as of October 2026. Several points are listed as unconfirmed.

---

## References

- AI Act Service Desk - Article 15: Accuracy, robustness and cybersecurity (2026). <https://ai-act-service-desk.ec.europa.eu/en/ai-act/article-15>
- AI Act Service Desk - Timeline for the implementation of the EU AI Act (2026). <https://ai-act-service-desk.ec.europa.eu/en/ai-act/timeline/timeline-implementation-eu-ai-act>
- Chapter V: General-Purpose AI Models | EU Artificial Intelligence Act (2026). <https://artificialintelligenceact.eu/chapter/5/>
- NVD - CVE-2026-42824 (2026). <https://nvd.nist.gov/vuln/detail/cve-2026-42824>
- Reddy, P. and Gujral, A.S. (2025) EchoLeak: the first Real-World Zero-Click prompt injection exploit in a production LLM system. <https://arxiv.org/abs/2509.10540>
- Security Update Guide - Microsoft Security Response Center (2025). <https://msrc.microsoft.com/update-guide/vulnerability/CVE-2025-32711>
- Spencer, P. (2026) Microsoft 365 Copilot SearchLeak (CVE-2026-42824): When your AI assistant becomes an exfiltration tool. <https://www.kiteworks.com/cybersecurity-risk-management/microsoft-copilot-searchleak-exfiltration-vulnerability/>
