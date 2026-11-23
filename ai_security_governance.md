# AI Security & Governance Interview Master Guide

**Basic → Intermediate → Advanced → Production → System Design**

**Target:** Senior AI Engineer / Principal AI Engineer / AI Security Architect  
**Verified reference date:** 23 November 2026 (Singapore time)  
**Scope:** LLM applications, tool-using agents, MCP, A2A, sensitive-data handling, cloud deployment, Singapore healthcare, security testing, and AI governance.  
**Explicit exclusions:** RAG architecture and security, document-retrieval mechanics, retrieval poisoning, vector and embedding weaknesses, and vector database controls. Discussion of *untrusted tool responses* is included because it is a general agent-security boundary, not a retrieval-specific mechanism.

> **Usage and precision note:** This is an engineering/interview guide, not a legal opinion or certification assertion. Sources are official primary publications at the links in [Reference register](#reference-register). **Law**, **formal standard**, **published guidance**, **protocol specification**, **vendor-dependent capability**, and **engineering recommendation** are identified separately. Where a legal exception, organizational role, regulatory commencement date, or cloud feature matters, verify it for the exact deployment before relying on it.

## Contents

- [How to study this guide](#how-to-study-this-guide)
- [Security taxonomy and regulatory sanity check](#taxonomy-and-regulatory-sanity-check)
- [Reference threat model](#reference-threat-model)
- [Part I — Basic (Q001–Q020)](#part-i--basic-q001q020)
- [Part II — Intermediate (Q021–Q050)](#part-ii--intermediate-q021q050)
- [Part III — Advanced (Q051–Q080)](#part-iii--advanced-q051q080)
- [Part IV — Production (Q081–Q106)](#part-iv--production-q081q106)
- [Part V — System Design (Q107–Q120)](#part-v--system-design-and-principal-level-scenarios-q107q120)
- [Enterprise architecture patterns](#enterprise-architecture-patterns)
- [Production decision matrices and security checklists](#production-decision-matrices-and-security-checklists)
- [Production release checklist](#production-release-checklist)
- [Security regression tests](#security-regression-test-catalogue)
- [Incident examples](#incident-examples--hypothetical-training-scenarios-not-claims-about-real-incidents)
- [Interview challenges](#ten-principal-level-interview-challenges-to-practice) and [scoring rubric](#interview-scoring-rubric)
- [Reference register](#reference-register)

## How to study this guide

1. **Basic (Q001–Q020):** State the threat, the trust boundary, and why prompt instructions alone are not a security control.
2. **Intermediate (Q021–Q050):** Design enforcement points for tools, delegated identity, sensitive data, and model providers.
3. **Advanced (Q051–Q080):** Analyze cross-boundary failure modes, PDPA transfers, healthcare privacy, standards, and governance.
4. **Production (Q081–Q106):** Explain evidence, testing, runtime controls, operations, incident response, and auditability.
5. **System Design (Q107–Q120):** Defend actual architecture choices and counter predictable interviewer objections.

For each answer, practice: **asset → actor → entry point → trust boundary → impact → preventive control → detective/response control → residual risk → evidence**. There is no single universally sufficient defense, and an LLM must never be the final authority for access control or critical side effects.

## Taxonomy and regulatory sanity check

### What changed in 2026

The **OWASP Top 10 for LLM Applications 2026** identifies: `LLM01 Prompt Injection`, `LLM02 Sensitive Information Disclosure`, `LLM03 Excessive Agency`, `LLM04 Supply Chain`, `LLM05 Data and Model Poisoning`, `LLM06 Unbounded Consumption`, `LLM07 Misinformation`, `LLM08 Hidden Context Exposure`, `LLM09 Vector and Embedding Weaknesses`, `LLM10 Improper Output Handling`. We deliberately omit the LLM09 subject area. **Do not reuse 2025 numeric IDs** (in 2025, for example, Excessive Agency was LLM06). [S01]

The **OWASP Top 10 for Agentic Applications 2026** categories are `ASI01 Agent Goal Hijack`, `ASI02 Tool Misuse and Exploitation`, `ASI03 Identity and Privilege Abuse`, `ASI04 Agentic Supply Chain Vulnerabilities`, `ASI05 Unexpected Code Execution`, `ASI06 Memory & Context Poisoning`, `ASI07 Insecure Inter-Agent Communication`, `ASI08 Cascading Failures`, `ASI09 Human-Agent Trust Exploitation`, `ASI10 Rogue Agents`. Agent memory is discussed here as persisted *instructions/state*, not as retrieval infrastructure. [S02]

The **MCP protocol revision 2026-07-28** brought changes to the protocol core and authorization flows. Its HTTP authorization specification uses an OAuth-based resource-server model **when authorization is implemented**; MCP authorization is *optional at the protocol level*, not an endorsement of anonymous internal enterprise services. **A2A 1.0.1** specifies enterprise-style authentication and authorization, with scheme advertisement through Agent Cards. Verify compatibility and exact revision across clients and servers. [S03], [S04]

### Distinguish source type

| Label | What it means | Examples | What it does **not** prove |
|---|---|---|---|
| **LAW** | Binding statute/regulation where applicable | Singapore PDPA, its regulations; applicable HIA provisions; applicable US HIPAA rules | That a system is compliant just because it uses a particular cloud region |
| **STANDARD** | Normative requirements for a defined management-system or technical scope, often voluntary until adopted | ISO/IEC 42001:2023 | That certification validates every AI application, processing activity, or country-specific law |
| **GUIDANCE** | Official but generally non-binding interpretation or practice | PDPC AI advisory guidelines; NIST AI RMF | A new standalone legal obligation |
| **PROTOCOL** | Contract of behavior for interoperating implementations | MCP 2026-07-28; A2A 1.0.1 | That a particular installation enforces its intended security correctly |
| **VENDOR** | Version-, contract-, region-, and configuration-specific behavior | Model logging, abuse monitoring, training use, cross-region routing | Universally consistent handling across all models, accounts, and services |
| **RECOMMENDATION** | Proposed defense supported by engineering risk analysis | Per-call PDP, deny-by-default egress, short-lived scoped credentials | A zero-risk guarantee or a legal requirement without legal authority |

## Reference threat model

### Assets, actors and boundaries

- **Assets:** patient and employee data; secrets; delegated credentials; model/tool configuration; privileged side effects; audit evidence; tenant isolation; financial and operational budgets.
- **Threat actors:** malicious end users; third-party content authors; compromised MCP/A2A servers or tools; malicious plugins/packages; compromised credentials; insiders; faulty model/vendor integrations; accidents and misconfiguration.
- **Entry points:** user messages; tool descriptions and tool responses; agent-to-agent messages; persisted agent state; code execution outputs; model-generated structured data; administrative configuration; vendor and observability integrations.
- **Boundaries:** human → application; untrusted text → model context; model intent → executable tool; user identity → service identity; agent → agent; tenant → tenant; organization → processor/subprocessor; Singapore → overseas processing; operational logs → third-party analytics.

```text
                   HUMAN / EXTERNAL CLIENT  (untrusted intent)
                              |
                         [IDP / API]
                              |
          +-------------------v-----------------------+
          | APPLICATION: authenticated principal      |
          | risk tier + tenant + purpose + consent    |
          +-------------------+-----------------------+
                              | B1: request ingress
          +-------------------v-----------------------+
          | AI ORCHESTRATOR / LLM GATEWAY             |
          | minimize inputs; model policy; budgets    |
          | treat tool messages as untrusted data     |
          +------------+-----------------+------------+
                       | B2: model       | B3: proposed action
                       v                 v
                MODEL PROVIDER      TOOL POLICY GATEWAY
                  (processor?)       identity/purpose/ABAC
                       |              approval + per-call auth
                       |                  |
                       |             B4: MCP / A2A
                       |                  v
                       |        BACKEND / PARTNER SYSTEMS
                       |         tenant and row-level auth
                       |
          B5: international transfers / vendor sub-processors
                       |
                       v
           AUDIT + TELEMETRY (minimal, protected, retained)
```

**Design principle:** Model text is **not a security principal, policy decision, authorization grant, or sufficient audit record**. A gateway and business services validate the real caller and action **independently of the model**. TLS, a private network, an account role, or a system prompt cannot substitute for object-level authorization. [S01], [S02], [S03]

### Control types and evidence

| Risk | Preventive control | Detective control | Response control | Governance procedure | Verification evidence |
|---|---|---|---|---|---|
| Prompt injection induces prohibited tool call | Tool allowlist + execution-time ABAC; untrusted-content separation | Policy-denial and anomalous call alerts | Suspend tool/agent, invalidate token | Threat model and risk acceptance | Injection regression cases; denied policy traces |
| Patient data sent to wrong party | Tenant/patient/purpose enforcement; minimization | DLP and access anomalies | Quarantine, revoke, breach assessment | DPIA; data-flow approvals | Object-level auth tests, sample trace inspections |
| Unapproved external processor | Approved routing; egress allowlist | Destination and billing anomalies | Block route and assess data exposure | Vendor register, transfer assessment | Contract, data-flow map, route traces |
| Code tool compromises worker | Isolation + no host secrets; egress controls | Runtime/syscall/egress alerts | Kill sandbox and rotate credentials | Sandboxing exception review | Isolation escape tests, policy snapshots |
| Model change silently alters behavior | Evaluated/approved immutable release | Canary and security regression telemetry | Rollback + kill switch | Model change advisory | Model/version/prompt hashes and test report |
| Cross-tenant delegated identity | Bound token audiences/scopes; per-tool tenant check | Cross-tenant denial monitoring | Disable grant, revoke credentials | Identity and delegation policy | Negative tests for tenant swaps |

### Security architecture quality bar

For every meaningful AI feature, produce a **data-flow diagram, risk register, subject/resource/action/purpose policy, tool registry, vendor and subprocessor record, test evidence, logging and retention schedule, incident runbook, and named accountable owner**. Add clinical validation and appropriate human review where outcomes may affect patient care.

---

## Part I — Basic (Q001–Q020)

### Q001. What is AI application security, and how is it different from traditional application security?

**Answer.** Traditional controls still matter: authentication, object authorization, input handling, secure dependencies, isolation, and logging. AI adds a probabilistic interpreter that reads untrusted natural language and can propose actions. The critical new failure is **treating lower-trust content as higher-authority instructions**. An untrusted message can influence a model even if every HTTP request is syntactically valid. Therefore, protect both **information flows** and **action execution**. The model may help classify intent, but deterministic controls decide whether a tool can run. **Evidence:** show a policy denial when a model requests an out-of-scope action. **Residual:** incorrect but apparently plausible answers and new language variants may still occur. [S01]

### Q002. Define asset, threat, vulnerability, risk, exploitability, and impact in an LLM system

**Answer.** An **asset** is what must be protected (a patient record); a **threat** is a harmful event (unauthorized disclosure); an **actor** is who may cause it; a **vulnerability** is a weakness (the assistant can call a patient-data tool with a broad service credential); **exploitability** concerns how feasibly the weakness can be triggered; **impact** is resulting harm; **risk** combines likelihood and consequences under the organization's risk methodology. Do not confuse a prompt-injection *attempt* with an actual breach. Record the precise pathway and whether controls stopped the outcome. **Evidence:** scenario-based threat registers and denied cross-tenant attempts, not merely generic vulnerability counts.

### Q003. What are the main LLM application attack surfaces?

**Answer.** Consider ingress text, prompt templates, model routing, tool specifications, tool results, persisted conversation/agent state, structured outputs, plugins, secrets, code execution, network egress, and external processors. A malicious tool description can be as important as a malicious user message because both enter context. **Boundary:** none of these surfaces should be automatically elevated to an authorized operation. **Control:** explicitly label provenance, restrict tools by task and principal, enforce bounded execution and output handling. **Verify:** trace the provenance of every proposed action and attempt adversarial input at each boundary. [S01], [S02]

### Q004. What is a trust boundary, and where does one exist in an AI assistant?

**Answer.** A trust boundary separates zones with different authority or assurance. Examples include external input crossing into an orchestrator, model-generated JSON crossing into a business API, and an MCP server crossing into a protected service. Crossing the boundary must trigger **new validation and authorization**; trust must not be inferred from a previous hop. In healthcare, user → tenant → authorized patient → purpose is an additional object-level chain. **Verification:** swap patient IDs while retaining a valid login, and require a denial. TLS establishes channel protection, not a legitimate clinical purpose.

### Q005. What is direct prompt injection?

**Answer.** Direct prompt injection occurs when a user-controlled message attempts to override the application's intended instructions or constraints. A user might ask the assistant to disregard its role and perform a privileged task. This is distinct from an ordinary, permitted request: the attempted instruction is trying to cross a higher-priority control boundary. **Prevention:** limit what the model can cause; validate capabilities per request; authenticate and authorize externally. **Detection:** count attempted policy overrides and forbidden tool requests. **Residual:** detecting malicious wording by a second model is not perfectly reliable. [S01]

### Q006. What is indirect prompt injection without involving retrieval?

**Answer.** Indirect injection occurs when lower-trust content **from a tool result, webhook, email body, third-party API response, or A2A message** is written as if it were a command to the assistant. The model might misinterpret that content as authority. The architecture should treat such content as **data**, not as developer instructions, and never let that content grant a tool permission. **Verify:** use a harmless synthetic tool response containing conflicting instructions; the model should continue within the permitted task and the tool policy must still deny unauthorized side effects. [S01], [S02]

### Q007. Is an instruction hierarchy sufficient to prevent injection?

**Answer.** No. An instruction hierarchy is a useful behavioral signal about which messages ought to take precedence, but it is **not a security boundary**. Long, convincing, or cleverly packaged untrusted content can still influence output and tool choice. Use instruction separation as one layer, then place capability and data authorization at execution time. A secure design remains safe **even if the model proposes an unsafe action**. **Verify:** force the agent to request a prohibited tool in testing and confirm a policy rejection rather than a successful action. [S01]

### Q008. What is the difference between a jailbreak and prompt injection?

**Answer.** A **jailbreak** generally seeks to bypass the model's behavioral constraints; **prompt injection** seeks to redirect the model using text from a less-trusted surface or requester. They overlap but are not identical. A jailbreak may produce an undesirable answer without triggering any external effect; an injection against a tool-enabled agent may cause unauthorized side effects. **Prioritization:** assess actual confidentiality, integrity, availability, and safety impact rather than treating every refusal failure as equally severe. **Evidence:** distinguish policy violation rates from successful prohibited tool execution.

### Q009. What is excessive agency?

**Answer.** Excessive agency means the application gives an AI component more functions, permissions, autonomy, or unsupervised scope than necessary. Example: a summarization assistant also has general database-write and outbound-HTTP capabilities. The danger is architectural, even before any injection is attempted. **Controls:** per-feature allowlists; separated read/write credentials; policy-driven approval for sensitive actions; small budgets; no generic command executor unless justified. **Verify:** enumerate all reachable privileges and compare them with explicitly approved use cases. The 2026 LLM taxonomy identifies Excessive Agency as **LLM03:2026**. [S01]

### Q010. Why must the LLM not be the authorization engine?

**Answer.** The LLM cannot reliably determine whether an authenticated user may read or modify a particular object from natural language alone. Authorization must use trusted identity claims, server-side roles/attributes, tenant, patient relationship, action, purpose, and context. The business service or policy enforcement point must reject mismatched requests **even when arguments look valid**. **Verify:** a correctly formatted `patientId` belonging to another provider must be denied. The model may propose a candidate action, never create or override entitlements.

### Q011. How do authentication and authorization differ for AI tools?

**Answer.** Authentication establishes who the requester or service is; authorization decides if that identity may perform **this action on this resource now**. For agents, also record *on whose behalf* the operation occurs. A valid service JWT does not imply every patient is in scope. **Controls:** short-lived audience-scoped credentials, execution-time ABAC or domain policy, and separate runtime versus user identity. **Verify:** valid identity + insufficient permission yields a clear denial and an auditable policy decision. [S03], [S04]

### Q012. Define least privilege for a tool-using agent

**Answer.** Least privilege applies to **tools discoverable, operations executable, objects accessible, credentials held, outbound destinations, and permitted lifetime**. An appointment assistant might read slots and create a booking, but should not alter diagnosis records or arbitrary accounts. Give each task a limited grant; do not attach a platform-wide role to every worker. **Verify:** an inventory comparison of reachable permissions against use-case approval; negative tests for omitted operations. **Residual:** a legitimate authorized action can still be requested at an inappropriate time, so use business rules and explicit confirmation for consequential operations.

### Q013. What is sensitive information disclosure in an LLM application?

**Answer.** Disclosure can occur in user-facing responses, prompts to third-party models, tool arguments, traces, error logs, support tickets, or outputs rendered in external systems. The source may be patient data, a secret, a hidden instruction, or private organization data. **Prevention:** classify data at every boundary, minimize payloads, redact where compatible with task semantics, validate recipient authorization, prohibit secrets in prompts. **Detection:** monitor privileged accesses and sensitive-data egress. **Response:** revoke access, preserve minimal evidence, assess breach notification. [S01], [S11]

### Q014. Is a system prompt a secure place to store API keys or secrets?

**Answer.** No. System instructions may be disclosed accidentally, intentionally, or through logging and support workflows; the 2026 OWASP taxonomy includes **Hidden Context Exposure (LLM08)**. A secret in context is already too widely exposed. Use a secret manager, workload identity, and server-side tool adapter. The model receives only a capability name and allowed schema, not its credential. **Verification:** inspect prompts, traces, crash reports, client bundles, and tool errors for secret-shaped data. Rotate anything already exposed. [S01]

### Q015. What does defense in depth mean for prompt injection?

**Answer.** It means assuming that one layer will sometimes fail. Use task-scoped instructions and content provenance **plus** strict tool registry, input schema validation, per-call authorization, constrained egress, secret isolation, action approval, anomaly monitoring, and tested incident response. A prompt filter might reduce suspicious requests but cannot prove safety. **Residual:** a model may still choose harmful but ostensibly allowed actions, requiring purpose checks, quotas, and reviews. **Proof:** demonstrate the same forbidden objective is blocked by an independent downstream control if the prompt layer fails. [S01], [S02]

### Q016. What is output handling risk?

**Answer.** LLM output may be text, Markdown, HTML, SQL fragments, commands, URLs, or JSON. Never assume it is safe because it came from the model. Treat it as untrusted program output: use context-aware encoding for UI, parameterized queries, URL/host policies, schema validation, and fixed operation mappings. **Boundary:** generated content → renderer or execution engine. **Verify:** security tests with harmless markup/special characters and malformed structures; confirm no unintended rendering, network access, or side effect. The 2026 OWASP entry is **LLM10:2026 Improper Output Handling**. [S01]

### Q017. Does a valid JSON Schema make a tool invocation secure?

**Answer.** JSON Schema can ensure shape, type, limits, enumerations, and required fields. It does **not** establish the caller's permissions, business purpose, data sensitivity, or appropriateness of executing the operation. Example: a valid UUID may identify a patient outside the caller's care relationship. Use schema validation **and** authenticated, server-side policy checks plus domain invariants. **Evidence:** type-correct unauthorized requests are rejected; input errors do not disclose other tenant identifiers. Separate model-format compliance metrics from security authorization metrics.

### Q018. What is the role of a threat model in AI design review?

**Answer.** A threat model makes explicit **assets, actors, trust boundaries, abuse cases, impact, controls, owners, and residual risk** before shipping. Start with a data-flow diagram and ask what happens when the model misinterprets a message or a tool returns adversarial text. Include model providers and telemetry processors, not just your API. **Evidence:** reviewed scenarios linked to automated tests, controls, and acceptance signatures. **Update triggers:** new tool, model, vendor, permission, tenant scope, data category, or autonomous action.

### Q019. How do you prioritize AI security risks?

**Answer.** Prioritize by sensitivity and blast radius: patient harm or unauthorized record exposure usually outranks a benign answer-format violation. Consider probability, reachability, attacker control, autonomy, downstream permissions, ability to detect, recovery time, and applicable legal/contract consequences. Use a **risk-based**, not a purely model-score-based, ranking. **Verify:** each high risk has a tested control and owner, and exceptions have time-bound approval. Do not infer risk solely from OWASP category rank or a vendor benchmark percentage. [S01], [S05]

### Q020. What would convince you an AI feature is secure enough to launch?

**Answer.** There is no proof of absolute safety. I would demand a defined use case and data classification; approved threat model; authenticated, scoped tools; fail-closed object authorization; relevant privacy and transfer assessments; meaningful adversarial tests; documented residual risks; prepared responders; and a rollback/kill switch. For high-impact healthcare uses, include clinical owner and suitable human oversight. **Go/no-go evidence:** signed risk decision with known failures and controls, test artifacts, vendor terms, and production alert routes—not a declaration that the prompt is 'safe'.

---

## Part II — Intermediate (Q021–Q050)

### Q021. How can tool-output manipulation compromise an agent?

**Answer.** A tool's reply is external data, even when the tool was legitimately invoked. An attacker or compromised upstream provider can place an instruction-like string in a status field, note, or error message. If the orchestrator appends that text as trusted guidance, the model may change course. **Controls:** supply provenance and typed response contracts; avoid concatenating tool text into high-priority instructions; permit tools to return facts, not new privileges; enforce a downstream policy gate. **Verification:** synthetic poisoned status and error fields should not lead to unauthorized tool calls. **Residual:** legitimate tool data can still bias the answer without causing a prohibited call. [S01], [S02]

### Q022. How should you design a secure tool registry?

**Answer.** Maintain a versioned registry with immutable `tool_id`, owner, description, input/output schema, data classes, effect (`read`, `write`, `external-send`, `code`), tenant scope, authentication mode, network destinations, required permissions, and approval tier. Dynamic discovery must not silently expand the executor's allowed set. Present only task-relevant tools to the model and authorize again at execution. **Verify:** a newly published or renamed tool has no effect until reviewed and enabled for that workload. **Residual:** a permitted tool may have a vulnerable implementation, so service security testing remains necessary.

### Q023. What is the difference between tool *discovery* and tool *execution* authorization?

**Answer.** Discovery controls what the model can see and propose. Execution authorization answers whether a concrete call is permitted for the actual user, tenant, resource, purpose, and state. The first reduces confusion and exposure; the second is mandatory protection. **Do not rely on** hidden tools, tool descriptions such as “admins only,” or frontend filtering. **Test:** make a direct, valid backend call for an undisclosed tool and confirm a deny; separately verify unauthorized tools are absent from the model's context.

### Q024. What is the confused deputy problem in agent systems?

**Answer.** A *deputy* is a service with legitimate authority that is tricked into using that authority for an unauthorized caller or purpose. Example: an MCP proxy has broad API credentials and accepts a user-directed operation that the user could not perform directly. **Mitigate:** bind the principal and intended audience at each hop, avoid blanket service tokens, validate object-level access, retain delegation provenance, and secure consent/authorization flows. **Verify:** cross-client and cross-tenant delegation attempts cannot reuse another user's grant. MCP security guidance explicitly discusses confused deputies. [S03]

### Q025. How should human approval work for sensitive agent actions?

**Answer.** Approval must cover a **specific immutable action proposal**: tool, normalized arguments, affected resources, reason, data recipient, and expected effect. The user should see material consequences before authorizing. Record the approver's identity and bind authorization to a hash or version of that proposal; check authorization **again** just before execution. Require reapproval if arguments, scope, beneficiary, or destination change. **Verify:** changing an appointment recipient after approval invalidates the grant. **Residual:** approvers can be misled; readable confirmation UX and separation of duties mitigate—but do not eliminate—that risk.

### Q026. What is privilege escalation in an agent workflow?

**Answer.** Escalation occurs when an agent obtains capabilities beyond its authorized role, perhaps by abusing a shared service token, invoking an administrative MCP tool, or crossing a tenant boundary. Distinguish **model influence** from actual **privilege escalation**: a model asking for admin access is only a proposal until an executor grants it. **Controls:** separate runtime identities, prohibit token forwarding to unrelated audiences, per-call policy, credential rotation, break-glass process. **Evidence:** trace shows denied scope and auditable issuer/audience validation; no elevated side effect. [S02], [S03]

### Q027. How do you handle read-only versus write-capable tools?

**Answer.** Separate them both in identity and policy, not only in prompt descriptions. Read tools may still disclose sensitive information; write tools may alter records, send notifications, or trigger transactions. Classify each by **impact and reversibility**, define approval thresholds, and apply tighter quotas and alerting to high-risk writes. An apparently read-only tool with broad outbound URLs can still be an exfiltration path. **Verify:** each declared `read` tool has no hidden write, mutation, or network side effect; write operations use domain API invariants and authorization.

### Q028. Why is a generic browser, HTTP, SQL, or shell tool high risk?

**Answer.** General-purpose capabilities dramatically widen the set of possible actions and destinations. A single broad tool often bypasses the domain-level permissions that a dedicated `create_appointment` tool could enforce. Prefer narrow operations, constrained endpoints, query templates, read-only database views, and sandboxes for code. **Controls:** operation allowlist, explicit URL destinations, network and filesystem isolation, scope-limited identity, and audit records. **Verify:** forbidden destinations and unauthorized data classes cannot be reached, even when the generated request is syntactically valid.

### Q029. What is secure delegation of user identity to an agent?

**Answer.** A user session can create an explicit, constrained grant for an agent, but it should not expose the user's full credential to the model. The grant should specify **subject**, **actor/client**, **audience**, **scope**, **expiry**, **tenant**, and where appropriate **resource or purpose**. A token broker can mint/exchange a short-lived token for the target tool. **Verification:** replay, wrong-audience, expired, and unauthorized-resource attempts are rejected. **Residual:** a correctly issued but excessively broad grant can still be abused; re-evaluate scope at every tool call.

### Q030. What is a policy decision point (PDP) versus policy enforcement point (PEP)?

**Answer.** A **PDP** evaluates whether an action is allowed under policy; a **PEP** actually blocks or permits execution. For agent tooling, the PEP should sit immediately before the side-effecting adapter or business service, and the PDP should evaluate trusted claims and server-side data. Example: `allow(user, tenant, patient, action, care_relationship, purpose)`. The model is neither PDP nor PEP. **Verify:** a PDP denial results in no downstream request, with a tamper-resistant decision ID and a minimal audit event.

### Q031. What is purpose-based authorization, and why might healthcare require it?

**Answer.** Traditional role checks ask whether a clinician is authorized in general. Purpose-aware policy also asks **why** the data is used—e.g., treatment, administrative workflow, or another explicitly supported purpose. For healthcare, the same professional role does not necessarily permit unrelated access. Purpose must be tied to a legitimate workflow and verified against server-side facts, not accepted solely from a model- or user-written string. **Test:** a provider lacking an allowed patient relationship receives a deny even with a correct role and asserted “treatment” purpose. Legal rules vary by data source and application.

### Q032. How do you implement multi-tenant isolation for an assistant?

**Answer.** Bind every request to an authenticated tenant and resource relationship in server-side context; propagate tenant context through the gateway, tool policy, services, caches, job state, and audit. Avoid relying solely on model-generated tenant IDs or UI filters. Apply database access controls and tenant-specific credential/domain boundaries where justified by sensitivity. **Negative testing:** compare valid patient IDs across tenants, race asynchronous jobs from two tenants, and attempt reuse of conversation/task IDs. **Residual:** shared processors and operational telemetry can still mix data unless separately isolated.

### Q033. What is MCP, from a security perspective?

**Answer.** MCP standardizes how AI clients interact with exposed tools and other protocol resources. From a threat-model standpoint, an MCP server is a **remote or local execution and data boundary**, not just a catalog. Its declared tool names, descriptions, and arguments are not inherently trustworthy. For remote enterprise MCP, require authentication where needed, scoped authorization, allowlisted servers, validated identity, egress policy, and logging. **Version note:** use the exact negotiated MCP protocol revision; 2026-07-28 differs materially from prior handshake-era revisions. [S03]

### Q034. Does MCP automatically secure tools through OAuth?

**Answer.** No. MCP defines authorization behavior for implementations that use it, but the specification states authorization is optional in MCP and distinguishes HTTP-based implementations from STDIO. When using HTTP authorization, the MCP server is an OAuth resource server and must validate appropriate tokens. OAuth itself is not object authorization: a bearer token may be valid yet insufficient for a patient or tenant. **Verification:** inspect transport, issuer, audience, scopes, client authentication, delegated identity, and downstream domain authorization—not merely presence of an `Authorization` header. [S03]

### Q035. What is the security difference between local MCP and remote MCP?

**Answer.** A local STDIO server may inherit filesystem, process, and environment access from the user's host; a remote HTTP server exposes a network/API trust boundary. Local does not mean safe: an untrusted package with host credentials is often higher risk than a tightly constrained remote adapter. For local servers, vet code, permissions, child-process isolation, and secrets; for remote, add TLS, authorization, identity, origin/destination controls, and server lifecycle management. **Verify:** credential visibility and effective permissions in both execution modes. [S03]

### Q036. What is token passthrough and why is it dangerous in MCP?

**Answer.** Token passthrough occurs when a server forwards a credential issued for one service to another service instead of obtaining a suitable audience-bound credential. This can defeat intended audience boundaries and confuse attribution. Prefer proper OAuth delegation or token exchange and validate token **audience, issuer, subject, scopes, and expiry** at every resource server. Keep raw tokens out of model context, logs, and A2A task payloads. **Verify:** a token issued for MCP service A cannot be used directly against downstream service B. [S03]

### Q037. How would you secure MCP tool descriptions and server discovery?

**Answer.** Tool descriptions influence what the model decides to call, so they are an instruction-like attack surface. Allowlist trusted MCP servers, record package/source provenance, review tools and schema diffs, freeze or approve capability sets, and prevent unreviewed servers from registering privileged tools. **At runtime:** use a registry policy independently of descriptions and check each invocation. **Verify:** renaming or adding a dangerous tool cannot bypass approval; replacement server identity or signing key changes trigger review. **Residual:** even a vetted tool can later become compromised. [S02], [S03]

### Q038. What is A2A and what additional security problem does it introduce?

**Answer.** A2A provides conventions for agents or agent applications to communicate as services. Every agent boundary introduces remote identity, authorization, data disclosure, task ownership, and inter-agent content integrity concerns. Agent messages are **not trusted instructions simply because another agent authored them**. Authenticate peers; authorize each resource/task; protect task histories, artifacts, and callbacks; and bind delegated work to a known principal and purpose. **Verify:** an authenticated but unrelated agent cannot enumerate or cancel another caller's tasks. [S04], [S02]

### Q039. How should A2A Agent Cards be trusted?

**Answer.** Agent Cards advertise endpoints, capabilities, and supported security schemes. They help with discovery; they are **not proof** that a server is entitled to sensitive data. Obtain cards only from approved origins or registries, verify server identity, use signing/verification mechanisms where the chosen version and deployment support them, and protect nonpublic cards. Authorize actual calls separately. **Verify:** a spoofed or substituted card does not redirect sensitive tasks to a new destination. Check the precise A2A release and signature-validation behavior. [S04]

### Q040. What is the difference between agent identity and end-user identity?

**Answer.** Agent identity identifies the running workload; end-user identity identifies the human on whose behalf it acts. A downstream service often needs both: `actor=appointment-agent`, `subject=doctor-123`, `tenant=clinic-A`, `action=create`, plus trusted purpose and policy. Never infer human entitlement from the service role alone. **Controls:** OIDC/workload identity, constrained delegation, service authentication, and auditable attribution. **Verify:** an agent can access a backend operational endpoint but cannot act for a patient or clinician without a valid delegated grant.

### Q041. How do you prevent a cross-agent confused deputy?

**Answer.** A receiving agent must not execute with broader privileges merely because the calling agent is trusted. Require authentic caller identity, signed or securely bound delegated authority where appropriate, audience and task binding, tenant/resource authorization, and anti-replay protections. Do not trust the calling agent's natural-language claim “the user approved this.” **Test:** an approved agent sends a task for a resource outside its delegation; the receiver denies it. **Residual:** a sufficiently broad policy can still authorize unintended outcomes and needs regular review. [S02], [S04]

### Q042. When would you use mTLS versus OAuth for agent communication?

**Answer.** They solve different problems. **mTLS** authenticates service endpoints and protects the transport, useful for workload-to-workload trust. **OAuth-style access tokens** carry authorization context such as audience, scope, and sometimes delegated user identity. A sensitive enterprise design may use both. Neither substitutes for patient/tenant object authorization. **Verification:** incorrect client certificate fails service authentication; valid certificate with insufficient token scope fails operation authorization. Match requirements to protocol version, mesh, and identity provider rather than assuming every SDK supplies them automatically. [S03], [S04]

### Q043. What are common AI data classes and why classify them before a model call?

**Answer.** At minimum distinguish public, internal, confidential, secrets, personal data, health/clinical data, financial data, and regulated/contract-restricted data. Categories may overlap; health information remains sensitive even without a name attached if re-identification is possible. Classification drives model routing, masking, approval, retention, monitoring, and who can see the response. **Verify:** disallowed classes cannot reach an unapproved model or observability vendor and that classifiers are evaluated for false negatives. **Residual:** data classification errors require conservative defaults and audit samples.

### Q044. Define PII, PHI, and personal data without conflating them

**Answer.** **PII** is a broad privacy/security term for information linked or linkable to a person; definitions vary by framework. **PHI** is a term with a specific meaning under applicable US HIPAA rules. **Singapore PDPA personal data** concerns individuals identifiable from the data alone or with other information the organization has or is likely to access, subject to statutory context and exceptions. A Singapore clinical record is not automatically HIPAA PHI just because it is medical. **Design:** classify by the actually applicable law, contracts, and identifiability, not by an informal label. [S09], [S10], [S16]

### Q045. What is data minimization for LLM inference?

**Answer.** Send the **minimum information necessary for the authorized task**. For example, for an appointment explanation, omit unrelated clinical history, full identifiers, and provider notes; use derived facts if the task allows them. Apply minimization before the model call, not only when writing logs. Track fields and justify each one; maintain a deny-by-default classification policy for new fields. **Verification:** compare model request payload against a field-level allowlist and fail tests when an unrelated sensitive field is introduced. PDPC AI guidance encourages minimization as good practice; this does not replace applicable PDPA obligations. [S11]

### Q046. What is the difference between redaction, pseudonymization, de-identification, and anonymization?

**Answer.** **Redaction** removes or obscures selected fields. **Pseudonymization** replaces identifiers while retaining a way to reconnect records, directly or via other data. **De-identification** is a broad set of techniques whose legal sufficiency depends on context. **Anonymization** requires a stronger assessment of whether re-identification remains a serious possibility; under PDPC guidance, genuinely anonymized data can fall outside PDPA personal-data scope. Removal of name/NRIC alone is often insufficient for rare clinical attributes. **Verification:** motivated-intruder or re-identification assessment using likely auxiliary information. [S10], [S11], [S13]

### Q047. Why does removing `patientId` from Langfuse or tracing logs not automatically make them anonymous?

**Answer.** The trace may retain exact dates, measurements, rare diagnoses, clinic location, message context, or stable pseudonyms. Combined with other accessible information, these can identify a patient. A random run ID may also enable linkage. **Controls:** field allowlisting, structured redaction, separation of identifiers, truncation, limited access and retention, region/processor approval, and no raw prompt content unless justified. **Verification:** inspect real traces under restricted conditions and conduct a contextual re-identification assessment. **Residual:** imperfect detectors and contextual uniqueness still create privacy risk. [S10], [S11]

### Q048. How should prompt and completion logs be designed for healthcare?

**Answer.** Log **metadata by default**: request ID, principal/tenant pseudonym as needed, task/tool version, model configuration, policy decisions, latency, cost, and safety event. Keep raw clinical prompt/completion content only when a clearly authorized purpose justifies it, with additional encryption, access control, retention limits, and incident procedures. Sanitize error messages and debugging exports. **Verify:** log sampling, access-control tests, data-retention deletion tests, and monitoring-vendor subprocessor review. **Residual:** even apparently metadata-only events may be linkable to individuals.

### Q049. How should a DLP gate handle both inputs and outputs?

**Answer.** Input DLP reduces accidental or unauthorized disclosure to models; output DLP reduces leakage to users or external channels. Neither can prove compliance: detectors can miss clinical context and may break needed medical meaning. Use structured field policies and deterministic recipient authorization first, DLP as defense in depth. Maintain a controlled exception process for authorized clinical use. **Verify:** representative false-positive/false-negative tests, sensitive-field canaries, and tool/network egress correlation. Record why blocked or allowed without logging the sensitive content itself.

### Q050. Why are data residency, sovereignty, and privacy compliance different questions?

**Answer.** **Residency** concerns where data is stored or processed; **sovereignty** concerns legal jurisdiction and control; **compliance** concerns all applicable purposes, safeguards, transfers, contracts, retention, access, and accountability. A Singapore database does not prove inference, backups, abuse logs, human support, or telemetry remain in Singapore. Conversely, PDPA does not impose a blanket requirement to host all personal data only in Singapore: its transfer restrictions may allow suitably protected overseas transfers. **Verify:** complete data-flow and subprocessor inventory, precise service behavior, contractual terms, and applicable legal assessment. [S09], [S10]

---

## Part III — Advanced (Q051–Q080)

### Q051. When does Singapore's PDPA apply to a GenAI assistant?

**Answer.** The PDPA governs relevant collection, use, disclosure, protection, retention, and transfers of **personal data by organizations** within its scope, subject to the statute's definitions and exceptions. Calling an LLM does not change the character of data processing. Identify whether the provider, customer organization, and subcontractors are organizations or data intermediaries and map what each does. **Engineering action:** document personal-data purpose and flows from user request through inference, telemetry, support, and deletion. **Legal precision:** PDPC's AI-specific advisory guidelines are **non-binding guidance**; the underlying PDPA and regulations remain law. [S09], [S10], [S11]

### Q052. Does consent always have to be collected separately for every AI use?

**Answer.** Not necessarily. PDPA consent and notification obligations depend on the **specific purpose and legal basis**; relevant deemed-consent provisions or statutory exceptions may apply if their conditions are met. The PDPC AI advisory discusses business-improvement, research, and other exceptions in defined contexts; it is **not a blanket permission** to train models on patients' records. **Process:** obtain a privacy/legal determination for each purpose (inference, evaluation, model improvement, analytics), document notification/consent or the exception relied upon, and re-evaluate materially new purposes. **Verify:** no engineering flag labeled `ai_allowed=true` substitutes for a defensible legal analysis. [S09], [S11]

### Q053. What is Singapore's Transfer Limitation Obligation?

**Answer.** Under PDPA and the Personal Data Protection Regulations, an organization generally must ensure overseas personal-data recipients provide a **standard of protection comparable to PDPA**, subject to the statutory rules and exceptions. Mechanisms may include legally enforceable obligations (such as suitable contracts or binding corporate rules) or prescribed certifications under applicable conditions. **Not true:** “PDPA requires every patient byte to remain in Singapore,” or “signing any DPA automatically fixes all transfer issues.” **Verify:** recipients, countries, onward transfers, protections, and allowed uses. Use the actual regulations and PDPC Key Concepts guidance; obtain legal review for exceptions. [S09], [S10]

### Q054. Is a Singapore inference endpoint proof that all processing stays in Singapore?

**Answer.** No. A provider may separately process support tickets, security-abuse signals, backup copies, billing, traces, or model telemetry; routing or fallback may differ by feature, account, and model. Some claims are vendor-specific and can change. **Due diligence:** inspect product/region documentation **and contractual commitments** for inference, at-rest storage, temporary buffers, logs, human access, training, subprocessors, failover, and deletion. **Verification:** collect vendor attestations, contract schedules, configuration evidence, and actual outbound destination observations. Do not invent or assume a vendor's regional behavior. [S10]

### Q055. How do you assess a US-based observability service receiving Singapore clinical traces?

**Answer.** First determine what trace fields can identify an individual; pseudonymization and simple identifier removal may not be enough. Then enumerate processing countries, service providers, retention, training/secondary uses, access controls, onward transfers, and transfer arrangements. Consider whether minimization or deployment of a regional/self-hosted collector can meet the need better. **Decision:** block raw clinical tracing until privacy/legal review and controls are approved; prefer metadata-only traces. **Verify:** sampled payloads and contract/data-flow evidence, not solely a vendor security badge. [S10], [S11]

### Q056. Is anonymized health data outside Singapore PDPA?

**Answer.** Data effectively anonymized so individuals are not identifiable may fall outside PDPA's personal-data definition, but the threshold is contextual. PDPC discusses whether there is **no serious possibility of re-identification**, considering reversibility, recipients, auxiliary information, and protections. Merely removing direct identifiers or calling data “anonymous” is inadequate. **Design review:** establish how identifiability was assessed and re-assess when sharing scope or external data availability changes. **Residual:** a motivated actor may combine seemingly innocuous clinical facts to identify someone. [S10], [S11], [S13]

### Q057. What is the PDPA Protection Obligation and how does it affect AI?

**Answer.** Section 24 requires organizations to make **reasonable security arrangements** to protect personal data against relevant risks; it does not prescribe one AI-specific cloud architecture. An LLM workflow must therefore address identity, tenant isolation, model/tool permissions, encryption, processor access, telemetry, incident response, and other reasonably foreseeable exposures. **Verification:** risk-proportionate control evidence, testing, and documented exceptions. **Do not claim:** ISO certification or a regional model endpoint conclusively discharges section 24. PDPC's AI advisory explicitly references section 24 in development environments. [S09], [S11]

### Q058. What is Singapore's Retention Limitation Obligation in this context?

**Answer.** Organizations should cease retaining personal data or remove means of association when retaining it no longer serves the purposes for which it was collected and is no longer necessary for legal/business purposes, subject to applicable obligations. A permanent prompt archive “for debugging someday” needs a defensible rationale. **Design:** purpose-specific TTLs for conversations, tool payloads, metrics, support exports, and audit events; separate retention for indispensable security evidence. **Verify:** deletion jobs, backup lifecycle, legal holds, and vendor deletion attestations. Avoid universal retention periods not grounded in specific law or agreements. [S09], [S10]

### Q059. What must a Singapore organization do after a potentially notifiable personal-data breach?

**Answer.** Contain and assess the breach reasonably and expeditiously; document the assessment. PDPC guidance discusses assessment **within 30 calendar days** of having credible grounds to believe a breach occurred. Notify PDPC **as soon as practicable and in any case within three calendar days after determining it is notifiable**. Notify affected individuals **as soon as practicable where required**. A breach can be notifiable due to likely **significant harm**, **significant scale (500 or more affected individuals)**, or other applicable statutory criteria; notification to people is not always required for a scale-only event. Do not treat “three days from detection” as the legal rule. [S12]

### Q060. When is HIPAA relevant to a Singapore healthcare AI system?

**Answer.** HIPAA is **not automatically applicable** simply because data is medical, a vendor is US-based, or a vendor markets “HIPAA-ready.” Assess whether the specific processing involves a US HIPAA-covered entity or business associate and information protected by the HIPAA rules. If applicable, determine Privacy/Security/Breach Notification duties and required business associate agreements. A signed **BAA is necessary in many covered arrangements but not sufficient** for compliance. Conversely, a vendor's HIPAA claim does not establish Singapore PDPA compliance. **Verify:** entity roles, data flows, agreements, and technical safeguards with counsel. [S16]

### Q061. What should a Singapore healthcare architect know about the Health Information Act 2026?

**Answer.** Singapore enacted the **Health Information Act 2026** (passed 12 January; assented 3 February 2026). It establishes frameworks concerning the National Electronic Health Record (NEHR), relevant sharing, and cybersecurity/data-security measures. **Critical caveat:** statutory enactment, staged implementation, and commencement of specific obligations are distinct. Determine whether the entity is a licensed provider, retail pharmacy, relevant approved user, or health-information-system provider, and check the **current commencement notifications and MOH requirements** before stating that a particular duty is operational. It is not a blanket authorization for an LLM to access NEHR. [S17]

### Q062. How do you conduct a Data Protection Impact Assessment (DPIA) for an AI use case?

**Answer.** Describe processing purpose, necessity and proportionality, data subjects, data categories, controller/intermediary roles, cross-border flows, retention, threat scenarios, potential harm, safeguards, residual risk, and approvers. Evaluate the same flow's inference, logging, testing, and support paths. Include clinical and vulnerable-individual considerations where relevant. **Output:** decision to proceed, proceed with conditions, or block, with owner and review triggers. PDPC AI advisory **encourages** DPIAs as a practice; do not misstate every DPIA as a separately mandatory PDPA filing. [S11]

### Q063. What is a vendor Data Processing Agreement (DPA) expected to cover?

**Answer.** As a **contractual control**, a DPA or equivalent terms may specify purposes, processor roles, security, access, locations, onward transfers, retention, training/secondary use, incident notification, audit rights, and exit/deletion. Wording must align with actual services and applicable law; “DPA signed” does not establish compliance with any transfer mechanism by itself. Request contract precedence clarity where vendor public documentation is less restrictive. **Verify:** subprocessor schedule and actual configurations match written obligations. **Residual:** third-party misconfiguration and enforcement challenges remain even with strong contracts.

### Q064. What is the difference between 'no model training', 'no retention', and 'no logging'?

**Answer.** These are **different vendor assertions**. “No training” means the data is not used for model training under defined terms; it may still be temporarily processed, logged for security, or stored for operations. “No retention” can have scope, exceptions, and feature differences. “No logging” may not cover platform, billing, network, abuse, or customer-side logging. **Procurement:** require explicit definitions, eligible products, exceptions, subprocessors, contractual assurance, and verification avenues. Never generalize an advertised property from one model/API to every service or region.

### Q065. When should an enterprise select managed models versus self-hosted models for sensitive data?

**Answer.** Use a documented tradeoff, not an ideological preference. Managed inference can reduce infrastructure burden but introduces provider and contract boundaries. Self-hosting may improve locality/control but makes the organization responsible for isolation, model supply chain, security patches, abuse protection, and GPU/runtime hardening. Evaluate clinical necessity, provider commitments, supported regions, quality, availability, cost, and operational capability. **Decision matrix:** permitted data flows and legal basis first; engineering quality and economics second. **Residual:** hosting internally does not make model outputs trustworthy or solve authorization.

### Q066. How do you evaluate AI supply-chain security?

**Answer.** Inventory model provenance, model weights/licenses, container images, package dependencies, prompts and tool templates, MCP servers, runtime plugins, CI artifacts, and hosted vendor integrations. Review malicious updates, tampering, unreviewed permissions, and compromised delivery paths. **Controls:** pin immutable versions/digests, scan and review dependencies, use trusted build pipelines and provenance, require approval before new tool capabilities, and run runtime isolation. **Verify:** an unexpected package, image digest, model revision, or tool schema triggers a blocked deployment or review. [S01], [S02]

### Q067. What is model or data poisoning, and how is it distinct from prompt injection?

**Answer.** Poisoning targets **training, fine-tuning, evaluation, or persistent state/data** so the model/system behavior changes beyond a single untrusted message. Prompt injection targets the model's behavior **at use time** through instruction-like content. In this guide we discuss poisoning of **model artifacts and agent state**, not retrieval datasets. **Controls:** provenance, role-restricted updates, integrity checks, dataset review, approval and versioning, and comparative tests. **Verify:** unauthorized modifications cannot become an approved model or persistent instruction. [S01], [S02]

### Q068. What is agent memory poisoning when no retrieval system is involved?

**Answer.** An attacker may try to alter **persisted preferences, working notes, checkpoints, tool state, or long-lived instructions** so later runs act on a false authority or instruction. Keep state data distinct from policy; require authenticated writers, ownership/tenant scope, immutable audit, TTL, and validation of remembered assertions before action. **Verify:** a lower-trust conversation cannot store a new instruction that enlarges permissions or persists across other users. **Residual:** some malicious but plausible notes may evade content checks, so all side effects still need runtime authorization. [S02]

### Q069. What are realistic limitations of prompt firewalls and LLM guardrails?

**Answer.** They can detect patterns, classify risk, and reduce known attacks, but they are probabilistic, context-sensitive, and prone to false positives/negatives. They can also be bypassed by benign-looking variants. **Do not** treat a moderation score as identity, permission, data-classification truth, or a substitute for egress restrictions. Measure attack success against the **protected outcome**, not merely whether the classifier flags the text. **Verify:** adaptive regression suites, human-reviewed samples, shadow evaluation, and monitoring for drift. [S01], [S05]

### Q070. How do you secure dynamically generated tool arguments?

**Answer.** Parse into a strict typed schema, normalize values, reject unknown fields, validate object existence and authorized relationship, apply business invariants, and enforce operation/destination controls before calling the service. Avoid permitting opaque user-supplied HTTP requests or SQL. **Separate:** syntax checks, domain checks, and policy checks. **Verify:** a valid-looking but cross-tenant resource ID and an unapproved recipient both fail; no side effect occurs before the policy decision. Use identical checks for retries and asynchronous execution.

### Q071. How do you handle sensitive tool results returned to the model?

**Answer.** A tool may lawfully return more than the model needs. Apply **server-side result projection**: only authorized fields required for the task, minimized and bound to the caller's context. Avoid including secrets or unrelated clinical notes; label the data class, source, and disclosure constraints. **Output authorization** is still required because a model could combine several permitted fields into a sensitive inference. **Verify:** captured tool payloads and replies respect field-level permissions; cross-tenant fields are never retrieved merely to be redacted afterward.

### Q072. How do you stop exfiltration through seemingly benign tools?

**Answer.** A send-email, web-request, webhook, image-render, or report-sharing tool can disclose sensitive data as a side effect. Constrain **recipient and destination** via policy, require domain approval for external sends, apply DLP as an extra check, and never allow model-proposed arbitrary URLs with sensitive payloads. **Verify:** an unauthorized external recipient is rejected even if the tool call is formatted correctly; observed egress matches the allowlist. **Residual:** sending approved content to an approved but compromised recipient still creates risk.

### Q073. What does a secure execution sandbox protect against—and what not?

**Answer.** A sandbox confines untrusted code's **filesystem, process, network, secret, and resource access**. It does not establish consent, authorization to access patient data, or correctness of an agent's clinical decision. Prefer no code execution at all for routine business agents. When required, use ephemeral isolated workers, non-root identity, read-only base images, seccomp/runtime controls, no shared credentials, no broad host mounts, quotas, and egress policy. **Verify:** harmless isolation probes and forbidden egress attempts fail. **Residual:** runtime/kernel flaws and misconfiguration remain.

### Q074. What must be considered in a Kubernetes design for AI workers?

**Answer.** Use separate namespaces/workloads for risk tiers, minimum RBAC, dedicated service accounts, workload identity, restricted pods, controlled secrets, network policies, egress gateways, image admission, and runtime monitoring. Kubernetes NetworkPolicy alone may depend on CNI capabilities and cannot replace L7 destination control. **Design:** chat/API pods should not receive broad database access if they only call backend services. **Verify:** effective access from inside the pod, not YAML alone; test DNS/egress and permission boundaries across tenants and environments.

### Q075. How does a model gateway enforce enterprise policy?

**Answer.** A gateway can centralize approved model/region routing, tenant quota, data-class decisions, minimization, request/response metadata, vendor credentials, version pinning, and fail-closed behavior. It must not become a superuser that bypasses application permissions. **Architecture:** authorize caller and allowed use case first; then select vendor and attach only scoped credentials. **Verify:** restricted clinical payloads never route to unapproved providers; fallback models cannot silently violate policy. **Residual:** a compliant routing path can still produce wrong answers.

### Q076. How would you model risk in NIST AI RMF versus OWASP?

**Answer.** **OWASP** provides security risk categories and attack-oriented guidance. **NIST AI RMF 1.0 (2023)** provides voluntary governance and risk-management functions **GOVERN, MAP, MEASURE, MANAGE**. **NIST AI 600-1 (2024)** is a GenAI companion profile, not a law or security certificate. **Application:** map an OWASP scenario to an asset, business impact, measurement, mitigation, owner, and monitoring plan. **Verify:** a risk register links concrete tests/evidence to governance decisions instead of merely listing framework names. [S01], [S02], [S05], [S06]

### Q077. What is ISO/IEC 42001, and what does certification mean?

**Answer.** **ISO/IEC 42001:2023** specifies requirements for an organizational **AI management system** (AIMS): scope, policy, leadership, processes, controls, and continual improvement. Certification can offer third-party assurance **within its defined scope**, but it does not certify that each model output is accurate, every tool is immune to injection, every data transfer is lawful, or HIPAA/PDPA compliance is automatic. **Verify:** certificate scope, legal entity, audit body, current validity, and evidence for the actual workload. **Complement:** ISO/IEC 23894:2023 is AI risk-management guidance. [S07], [S08]

### Q078. What can Singapore's IMDA AI governance frameworks add to an enterprise program?

**Answer.** Singapore published a Model AI Governance Framework for Generative AI (2024) and **Model AI Governance Framework for Agentic AI, v1.0, 22 January 2026**. These are useful **policy and engineering guidance**, not automatic binding laws. Apply their governance themes to autonomy boundaries, human responsibility, transparency, risk-based safeguards, testing, and accountability. **Verification:** show an enterprise adoption matrix matching framework themes to procedures and technical controls. **Do not** cite a governance framework as if it creates a PDPA exemption. [S14], [S15]

### Q079. What is an AI system inventory and what must it contain?

**Answer.** Inventory each deployed or approved AI use case, product owner, technical owner, model/vendor and version, purpose, data categories, geography, users, tools, autonomy/risk tier, dependencies, subprocessors, approvals, monitoring, and decommission date. An inventory should track **shadow AI** discovery and upgrades, not simply list cloud services. **Verify:** reconcile model gateway activity, cloud billing, code repositories, vendor contracts, and service catalog. **Residual:** unregistered employee tooling can escape central visibility; address with policy, discovery, and safe approved alternatives.

### Q080. How do you design AI lifecycle governance without slowing every low-risk change?

**Answer.** Tier workflows by **data sensitivity, external effects, autonomy, and harm**. A low-risk internal drafting feature may follow standard CI; a patient-affecting tool requires specialist review and domain sign-off. Use lightweight templates plus automated evidence collection. Require re-review when model/provider, permissions, geography, data purpose, tools, or user group materially changes. **Governance controls:** named owners, time-boxed exceptions, release gates, review cadence, and retirement/erase plans. **Verify:** the release system can demonstrate who approved what version and why. [S05], [S07], [S15]

---

## Part IV — Production (Q081–Q106)

### Q081. What does a production AI security review require beyond penetration testing?

**Answer.** Traditional SAST, SCA, DAST, IAM, API, and cloud reviews remain necessary. Add **model/tool trust-boundary testing**, adversarial input, data-classification flow review, tool-call authorization, workflow state integrity, model-provider assessment, and sensitive trace inspection. **Output:** scenarios mapped to controls, failed and passed test cases, owned residual risks, and a deployment decision. **Verify:** no critical finding is closed solely because the prompt was reworded; require an enforceable and regression-tested mitigation. [S01], [S02]

### Q082. How would you create a threat-model-driven security test suite?

**Answer.** List assets and forbidden outcomes first: unauthorized patient read, prohibited write, external send, role escalation, tenant crossover, insecure tool attachment, model data leak. For each, generate controlled **harmless synthetic cases** across entry surfaces and expected policy behavior. Test both attempted action and final effect, preserving seed/configuration where relevant. Categorize by risk, surface, and principal; require coverage for each high-risk trust boundary. **Verify:** CI produces traceable allow/deny evidence, not just a binary “guardrail passed” score.

### Q083. How do you measure prompt-injection resistance?

**Answer.** Define success in terms of the attacker's *prohibited outcome*—e.g., did an unauthorized tool execute?—rather than whether a response contains suspicious text. Record attack-attempt rate, blocked-outcome rate, false positives against benign tasks, severity-weighted failure rate, and test coverage. Include varied but safe adversarial cases and re-test after model or prompt changes. **Residual:** a low observed success rate on a fixed suite says little about novel or adaptive attacks. Avoid asserting “100% injection-proof.” [S01]

### Q084. What is the difference between red teaming, adversarial testing, and penetration testing?

**Answer.** **Adversarial testing** probes model/workflow behavior with deliberately challenging inputs. **Red teaming** explores realistic attacker objectives, trust boundaries, and organizational response under authorized rules of engagement. **Penetration testing** often focuses on exploitable system vulnerabilities and validation. They overlap, but AI security also requires business-object abuse and data-flow tests. **Safety:** use synthetic records and isolated test systems; do not expose real patients or run harmful actions in production. **Output:** severity, cause, affected boundary, fix, retest, owner.

### Q085. What is an appropriate AI security test environment?

**Answer.** Use a production-like topology with synthetic or properly governed data, seeded identities/tenants, equivalent authorization policies, controlled model/tool endpoints, and tightly bounded external effects. Replace real outbound notifications and payments with sinks. Exercise approved adversarial conditions without live patient exposure. **Verification:** test policies and runtime identities match production intent, while data and side effects are demonstrably isolated. **Residual:** staging does not perfectly replicate production vendor configuration, load, or unanticipated behavior.

### Q086. How can you test agent authorization without depending on model nondeterminism?

**Answer.** Split **policy tests** from **model behavioral tests**. Directly call the tool enforcement point with hand-crafted but harmless requests representing authorized and unauthorized combinations, then separately run full agent scenarios to see what requests the model proposes. The policy tests should be deterministic. **Verify:** a broad range of valid JSON requests with invalid tenant, patient relationship, purpose, role, or audience produce a denial with no side effects. This preserves assurance even when the model changes.

### Q087. How do you test approval integrity and race conditions?

**Answer.** Confirm that the approval is tied to the **exact** normalized resource, operation, arguments, identity, audience, and expiry. Check that execution re-evaluates policy if account permissions, patient relationship, or resource state changes between proposal and action. Treat approval tokens as one-time and revocable where appropriate. **Test:** mutate a harmless destination or revoke role after approval; execution must re-check and deny. **Residual:** human approval may still be poorly informed; ensure clear display of effects and audit.

### Q088. What are the highest-priority monitoring signals for a healthcare AI agent?

**Answer.** Focus on abnormal **authorization denials, high-volume patient access, cross-tenant attempts, sensitive egress, new external recipients, tool permission changes, policy overrides, sandbox violations, vendor route changes, and unusual administrative actions**. Model refusal rates alone are too indirect. Emit privacy-preserving structured events with correlation IDs and safe principal references. **Verify:** an approved synthetic incident triggers an actionable alert to an actual on-call owner, and alert payloads do not include raw health records.

### Q089. How do you make AI activity auditable without storing full prompts?

**Answer.** Record actor, authorized subject, tenant, action, resource reference or protected pseudonym, purpose, model/tool/policy version, decision and reason code, approval reference, timestamp, correlation ID, and result category. Prefer hashing or references for sensitive proposals, with a separately protected evidence store if necessary. Keep audit records integrity-protected and access-controlled. **Verify:** an investigator can reconstruct who initiated and authorized a high-risk action without unrestricted clinical content retention. **Tradeoff:** exact content reconstruction may be limited by minimization; define evidence needs upfront.

### Q090. How would you handle a confirmed prompt-injection attempt that did not cause a side effect?

**Answer.** Capture minimal indicators, identify entry surface and principal, verify execution/egress logs, and determine whether the attempt crossed any policy boundary. Update tests and monitoring; avoid treating every attempt as a mandatory legal data-breach notification. **Response:** block source or tighten controls if warranted; adjust rate/permission policies. **Evidence:** trace proves no protected data disclosed and no unauthorized operation occurred, to the extent observable. If facts are incomplete, document uncertainty and continue assessment. [S01], [S12]

### Q091. How do you respond if an agent actually disclosed patient data?

**Answer.** Stop the affected workflow and external route, contain credentials and access, preserve proportionate evidence, identify recipients and affected records, notify the incident-response and privacy/legal teams, and assess patient harm and statutory/contractual notification duties. Investigate model requests, tool authorization, output channels, and telemetry vendors. **Do not assume** it is automatically notifiable or automatically exempt: use PDPA criteria and any applicable sector-specific rules. **Recovery:** correct policy, regression test, verify containment, and review vendor access. [S09], [S12], [S17]

### Q092. What does an AI incident-response playbook contain?

**Answer.** A clear severity taxonomy, responsible security/privacy/clinical owners, emergency contacts and decision authority, stop/disable switches, model/tool/tenant isolation paths, credential revocation, event preservation without overcollection, breach assessment timeline, vendor notification procedures, communications, and restart criteria. Maintain runbooks for disclosure, unauthorized execution, unsafe clinical output, compromised tool, and external-processor incident. **Verify:** tabletop exercises and technical drills prove that responders can actually disable and trace the system before an emergency.

### Q093. How do you design kill switches for agent systems?

**Answer.** Provide independent controls to disable the **entire feature**, a **specific model route**, a **single tool**, an **external destination**, or a **tenant-level action** without deploying new code. Preserve emergency read-only capability if it is safe and needed. Require a named operator and audit each change; avoid relying on a model to respect a “stop” prompt. **Verify:** drill the switch against a synthetic in-flight action and prove no new prohibited side effect occurs after cutoff. Handle already-committed actions through domain remediation, not assumptions of rollback.

### Q094. How should secrets be isolated across model gateways and tools?

**Answer.** Prefer short-lived workload identity and secrets-manager access at the adapter layer. Never inject provider keys, database credentials, OAuth refresh tokens, or private signing keys into LLM context, tool text, or generic execution sandboxes. Separate identities by environment, agent, and purpose. Rotate on exposure; scope egress to permitted endpoints. **Verify:** a model prompt dump or compromised low-privilege tool cannot reveal production credentials, and revoke/rotation procedures are tested.

### Q095. How do you enforce secure network egress in AWS EKS?

**Answer.** Use a combination of Kubernetes/CNI controls, security groups where appropriate, subnets/routes, proxy or egress gateway, DNS/HTTP destination allowlists, private service endpoints, and AWS IAM identity policies. Separate **model inference**, **tool adapters**, **code sandboxes**, and **telemetry** by destination class. NetworkPolicy does not inherently inspect application-layer hosts or authorize clinical data release. **Verify:** run harmless denied-destination tests from the actual pods and measure traffic at the gateway. Keep exceptions in version control with owner and expiry.

### Q096. How do you design tenant-specific rate, budget, and action limits?

**Answer.** Limit model tokens and requests, tool invocations, sensitive records per request, external sends, approvals, and concurrency per authenticated tenant/user/use case. These controls reduce unbounded consumption and blast radius but are not access control. Choose thresholds based on legitimate clinical workflows and monitor false positives. **Verify:** tenant A cannot exhaust B's reserved quotas or bypass rate limits with new task IDs. The OWASP 2026 LLM taxonomy includes **Unbounded Consumption (LLM06)**. [S01]

### Q097. How do you protect clinical safety when an LLM generates a misleading answer?

**Answer.** Separate answer quality and clinical authority. Constrain use cases, disclose uncertainty where suitable, enforce deterministic clinical rules for high-impact decisions, and require appropriate qualified human review before patient-affecting actions. Maintain an escalation route when information is insufficient or conflicting. **Measure:** clinically reviewed scenario outcomes, harmful recommendation rates, and escalation performance. **Residual:** a well-authorized agent can still produce plausible misinformation; that is a safety issue even if no attacker is involved. OWASP lists **Misinformation (LLM07:2026)**. [S01], [S17]

### Q098. How should an enterprise manage model updates and silent vendor changes?

**Answer.** Maintain an approved **model and endpoint version register** with prompt templates, tool schemas, policy versions, evaluation baselines, and owner. Ask vendors what they can change without customer action and where notifications are available; don't assume pinned names always mean immutable weights. Use controlled canaries, security regression suites, data-flow checks, and rollback paths. **Verify:** unauthorized provider/region changes fail route policy and releases retain a trace of the actual model identifier returned.

### Q099. What is security regression testing for AI agents?

**Answer.** It converts incidents, red-team findings, and threat-model scenarios into versioned reproducible **negative cases**. Cover direct/indirect injection, valid-but-unauthorized tool arguments, missing approvals, token audience violations, cross-tenant data, unapproved external recipients, and changed tool definitions. Evaluate both **policy outcome** and model behavior, including non-deterministic runs where appropriate. **Release gate:** no new critical unauthorized outcome; document exceptions and remaining uncertainty. **Residual:** regression suites only cover modeled cases and must evolve.

### Q100. How would you evaluate an AI security product or guardrail vendor?

**Answer.** Define the exact job: content filtering, data classification, tool authorization, monitoring, adversarial testing, or runtime isolation. For each, ask **what it enforces technically**, how it fails, what data it sends offsite, where it operates, false-negative rates on your cases, latency/cost, support access, and logging/retention. Test it using controlled known-good and known-bad cases. A product that flags a prompt but cannot prevent a side-effect is **detective**, not a reliable execution guard. **Procurement:** compare independent enforcement to marketing claims.

### Q101. How do you prevent logging and tracing vendors from becoming shadow data processors?

**Answer.** Use egress registration and default-deny integrations; inventory SaaS SDKs, exception/error monitors, APM exporters, session replay, support tooling, and analytics. Enforce trace payload allowlists, avoid raw prompts in OTLP spans by default, and route data through an approved collector. **Verify:** examine emitted events under realistic flows and block unauthorized destinations. A service may act as a data intermediary or other processor depending on actual contractual and processing roles; legal classification cannot be inferred from “observability.” [S10], [S11]

### Q102. What is an effective governance exception process?

**Answer.** Record risk statement, affected assets and individuals, control gap, compensating controls, business rationale, owner, approver with authority, expiry, monitoring, and remediation plan. Distinguish **non-negotiable legal compliance** from policies that may be risk-accepted internally; a business owner cannot waive statute. High-impact clinical or patient-data exceptions require privacy/security and appropriate clinical input. **Verify:** exceptions expire automatically or block deployments when unrenewed, and activity is reconciled with the granted scope.

### Q103. How do you make an AI security program measurable at executive level?

**Answer.** Measure coverage and outcomes: percent AI systems inventoried and risk-tiered; proportion of high-risk tools with per-call authorization and negative tests; external processors assessed; sensitive traces minimized; expired exceptions; detection and containment drill performance; unresolved critical findings; and security regression outcomes. Avoid presenting “number of malicious prompts blocked” as a stand-alone measure of safety. **Report:** trends, material incidents, residual high risks, and named decisions with provenance. Tie metrics to actual exposed workflows, not vanity usage counts. [S05], [S07]

### Q104. What should a production security readiness checklist cover?

**Answer.** It should cover scope and owners, data classification and legal basis, model/vendor selection and regions, identities and policies, tool/action permissions, sensitive trace controls, prompt injection defenses beyond prompting, approval integrity, sandbox/network restrictions, reproducible security testing, alerts and audit, retention/deletion, third-party terms, incident playbooks, rollback, and governance sign-off. **Verify:** collect evidence from CI, cloud configuration, source code, contracts, and exercises rather than self-attested checkbox answers. A detailed gate appears in [Production release checklist](#production-release-checklist).

### Q105. What is the difference between a security architecture decision and a risk acceptance?

**Answer.** An architecture decision records why a control/design was chosen, alternatives, assumptions, and consequences. A **risk acceptance** explicitly acknowledges a known, remaining risk and its owner, limits, expiry, monitoring, and approval authority. Do not hide residual risk in an ADR as if design documentation removed it. **Evidence:** link ADR, threat model, measured failure modes, release decision, and exceptions. Example: adopting a managed model under approved transfer controls is an architecture choice; permitting raw clinical tracing overseas despite incomplete safeguards would require a separate legal/risk decision and may not be permissible.

### Q106. What does a mature Principal-level AI security review sound like?

**Answer.** It names the **failure mode and exact enforcement point**: “The model may be manipulated, so our policy gateway requires authenticated subject, patient relationship, approved purpose, and recipient before any tool runs. The model receives no privileged credentials. We test type-correct cross-tenant calls, record policy decisions, minimize traces, and have a tool kill switch. The remaining risk is misleading clinical text, which requires clinical validation and monitoring.” This is stronger than “we use guardrails, encryption, and a private VPC” because it is **specific, testable, and honest about residual risk**.

---

## Part V — System Design and Principal-Level Scenarios (Q107–Q120)

### Q107. Design a patient-data assistant for clinicians working across multiple provider organizations

**Answer.** Use enterprise SSO, backend-resolved tenant and clinician identity, an approved-purpose policy, and a patient-relationship service as the **authoritative source** of which provider may access which patient. Place a tool PEP between the LLM and patient service. The assistant never chooses its own tenant or receives a broad database credential. Read tools return **only allowed fields**; clinical outputs go only to the authorized clinician, with audit and human validation for significant decisions. **Boundary:** model-generated `patientId` → trusted backend. **Tests:** first-visit versus unrelated-provider access, token expiry, tenant swap, forbidden patient fields. **Residual:** clinical interpretation error, legitimate insider misuse, and incomplete care-relationship data. See [Case A](#case-a--secure-patient-data-assistant).

### Q108. A model says 'the user confirmed the payment'. Should the tool run?

**Answer.** No. That sentence is model output and does not prove an authenticated human approved a transaction. Build an immutable payment proposal and show the amount, beneficiary, currency, and purpose through a trusted UI. Bind a time-limited approval to the proposal and re-check identity, financial authority, recipient, limits, and state at execution time. **Verify:** a changed beneficiary or expired approval fails, and a repeated invocation cannot cause duplicate charges. The general security principle is independent confirmation of consequential side effects, not a better approval prompt.

### Q109. Your agent is connected to five MCP servers, one of which becomes compromised. What is the containment design?

**Answer.** Each server is individually allowlisted and identified, with scoped credentials and network destinations. Tool-policy enforcement remains outside all five servers; no server can directly obtain keys for another. Detect changed tool manifests, abnormal results, and denied actions. **Contain:** disable the compromised server and rotate its credentials without taking down unrelated tools; preserve minimal audit evidence; evaluate data exposure. **Verification:** an altered tool name/description cannot unlock new privileges, and the compromised server cannot pivot through other tool credentials. **Residual:** data already legitimately disclosed to it may be exposed. [S03], [S02]

### Q110. Design an A2A workflow where a scheduling agent delegates to a billing agent

**Answer.** The scheduler authenticates as a service and supplies a delegated user authorization **for a specific billing task** using a mechanism supported by the environment. The billing agent validates the issuing authority, audience, scope, tenant, purpose, task ownership, and amount limits before acting. A2A message content and Agent Cards are never the authority for payment permissions. **Boundary:** scheduler identity → billing service/user delegation. **Verification:** expired/wrong-audience grant, task-ID swap, and cross-tenant request fail. **Residual:** fraudulent but policy-permitted user requests still require risk controls. See [Case B](#case-b--enterprise-agent-and-tool-gateway). [S04]

### Q111. Design an enterprise agent gateway used by many engineering teams

**Answer.** Centralize **identity-aware routing, model allowlists, sensitive data classes, egress, tool registry, quotas, and audit**. Keep domain authorization at each backend service and tool PEP; otherwise the gateway becomes a dangerous superuser. Allow policies by application, tenant, model, tool, and region, with approval workflows for changes. Use short-lived service identity and separate runtime secrets. **Verification:** onboarding a new team/model cannot inherit patient-data permissions by default. **Residual:** centralized outage/misconfiguration increases blast radius; separate policy administration, stages, and emergency rollback.

### Q112. A vendor offers an excellent model but only overseas inference and unclear logs. What is your decision?

**Answer.** Classify the specific intended data and purpose. Ask for exact processing locations, inference logs, abuse monitoring, human access, retention, model-training use, subprocessors, and enforceable transfer protections. Involve privacy/legal stakeholders to assess PDPA and any sector/client restrictions. If needed terms cannot be verified, **do not route identifiable patient data**. Consider minimized/de-identified data if suitable or an approved alternate deployment. **Evidence:** documented vendor assessment and transfer decision, not “vendor says HIPAA compliant.” **Residual:** even approved vendors may change subprocessors or have security incidents. [S09], [S10], [S16]

### Q113. How would you secure a sensitive-data workflow that summarizes a health screening result?

**Answer.** Authorize the initiating clinician and allowed patient/report. Have a backend project only permitted clinical fields, attach task purpose and prohibited disclosure destinations, route through an approved model, and return output via an authorized UI. Do not send raw reports to general observability; retain a minimal audit and clinical review state. Use specialist review before generating patient-facing recommendations that may affect care. **Tests:** unrelated-provider denial, route policy for restricted fields, trace inspection, unsafe/missing-result scenario. **Residual:** hallucination and clinical overinterpretation. See [Case C](#case-c--sensitive-health-screening-workflow).

### Q114. A legitimate user asks an agent to email an entire patient record to an external address. What controls apply?

**Answer.** Authentication alone is insufficient. Check whether the user is authorized to disclose the full record **for the purpose and recipient** under applicable policies and law. Minimize scope, validate destination, require appropriate consent/legal basis or authorization, provide preview/approval when required, and maintain an audit. If not allowed, refuse execution regardless of the model's enthusiasm. **Test:** valid clinician role + unapproved recipient yields deny. **Residual:** even approved recipients can mishandle data; contracts and external sharing controls matter.

### Q115. A company wants to turn on raw LLM traces for debugging in a US SaaS account. How would you respond?

**Answer.** Start from necessity: which incidents cannot be diagnosed with metadata? Assess actual personal-data content, whether the US SaaS is a processor, transfer controls, access and retention, and the ease of patient re-identification. Prefer allowlisted metrics and privacy-preserving structured spans. If a temporary raw capture is justified, seek explicit scope/owner/TTL approval and limit access; check all legal and contract requirements first. **Verify:** sampled spans and deletion outcomes. **Do not assert:** removing `patientId`, hosting in Singapore, or a HIPAA badge resolves the issue. [S10], [S11], [S16]

### Q116. An agent uses an admin-level service role to call microservices. How would you refactor safely?

**Answer.** First inventory actual tool operations and map each to minimal permissions; then split read and write tools, use workload identities, delegate user context, enforce resource/tenant policy at each service, and eliminate broad database access from agent pods. Introduce shadow deny logging to discover legitimate use before hard cutover, then fail closed. **Verification:** negative authorization tests and production access logs show no unauthorized operation path; emergency access is separated and controlled. **Residual:** operator misconfiguration requires periodic permission review.

### Q117. An internal AI code-review agent receives source code and CI secrets. What is your security architecture?

**Answer.** Provide only code/diff and metadata needed for review; avoid exporting CI secrets, production values, or unrelated repositories. Use a **read-only default identity**, isolated execution if tools are needed, a vetted model endpoint, approved outbound routes, and purpose-specific retention. AI review findings are advisory unless backed by verifiable policies/tests; human Code Owners retain approval. **Verify:** a repo with maliciously phrased comments cannot cause the reviewer to run privileged tools or disclose secrets. **Residual:** inaccurate findings and unseen bugs remain possible.

### Q118. What if a production agent starts making unusually many allowed-but-sensitive patient reads?

**Answer.** An allow decision for each individual record does not guarantee the **aggregate pattern** is appropriate. Apply patient-access quotas, rate limits, purpose/relationship constraints, anomaly detection, and privileged-read monitoring. Suspend or step up review on anomalous behavior while preserving legitimate clinical workflows. **Verify:** seeded bursts cross a threshold and reach responders with minimal PHI, with false-positive checks. **Residual:** low-rate insider misuse may evade thresholds, requiring audits and organizational oversight.

### Q119. How would you run governance across 30 microservices and multiple AI vendors?

**Answer.** Establish one central registry of AI uses and approved processors, but enforce policies at **gateway + service** boundaries. Provide reusable IAM, PEP, safe telemetry, and release evidence patterns so teams do not invent them. Assign one business owner and a technical/security owner per high-risk application; distinguish vendor deployment approvals from model security claims. **Verify:** discovery reconciles real model traffic and tool registrations with inventory, and every production integration has a responsible owner and risk decision. **Residual:** exceptions, shadow tools, and legacy applications need targeted remediation.

### Q120. In a Principal interview, defend a choice between 'more prompt guardrails' and an external execution policy engine

**Answer.** Use both only where each adds measurable value, but **never substitute prompt guardrails for execution policy**. A model-level guardrail may reduce attempts and improve helpfulness. A policy engine independently prevents unauthorized patient access, unapproved recipients, and out-of-scope writes even when the model is manipulated. Describe the protected resource/action, trusted identity and context, deny behavior, audit evidence, and failure handling. **Decision:** invest first in deterministic enforcement for high-impact side effects; then test additional detection/guardrails for incremental risk reduction. **Residual:** legitimate but unsafe advice still needs clinical safeguards.

---

## Enterprise architecture patterns

The following designs are reference patterns for interview discussion, **not deployable security guarantees**. Actual policies must follow the legal role and data model of the organization.

### Case A — Secure patient-data assistant

**Business need:** Clinicians at authorized provider organizations ask questions about a patient's care context. A patient's existence in a global account directory **does not establish** a provider's right to access that patient's clinical data.

```text
[Clinician SSO]
      |
      v  TB1: verified identity, device/session policy
[Care Web/API BFF] -----> [Tenant + Care Relationship Policy Service]
      |                              |
      |                              v
      |                         [PDP: allow/deny]
      v                              ^
[AI Gateway: approved model]         |
      | proposes bounded query        |
      v                              |
[Agent Tool PEP] --------------------+
      |  TB2: tool call -> authorization + purpose
      v
[Patient Service] -----> [Clinical DB / encrypted sensitive data]
      |
      | projected, minimized, source-labeled fields
      v
[Model] -> [Clinician UI + clinical review if indicated]
      |
      +------> [Metadata-only Audit / Alerting]

TB3: Gateway -> model provider; evaluate geography, purpose, contracts
TB4: Service -> telemetry; minimize and control processors
```

| Threat | Prevent | Detect | Respond | Residual risk | Verify |
|---|---|---|---|---|---|
| Provider B reads patient seen only by A | Server-side care-relationship and purpose policy | Cross-provider denial alerts | Suspend grant, investigate | Stale/mislinked relationship | B→A ID swap regression |
| Model requests broad patient list | Patient-scoped tool schema, query limits, PEP | Bulk-access anomaly | Disable bulk tool | Allowed bulk read still risky | Policy negative tests |
| Raw report copied to overseas telemetry | Trace allowlist; approved vendor route | Data-egress checks | Disable export, breach assessment | Contextual re-identification | Sample actual OTLP spans |
| Incorrect medical interpretation | Validated clinical scope; qualified review | Clinical QA and feedback | Withdraw output / escalate | Known limits of generative answers | Clinically scored test cases |

**Launch gate:** demonstrate a denied, type-valid cross-provider request; secure tool calls even if model requests unauthorized access; verify all outbound data paths; confirm clinical review ownership and incident plan.

### Case B — Enterprise agent and tool gateway

**Business need:** Engineering teams deploy multiple LLM-powered agents through one security policy plane while retaining domain-level authorization.

```text
[Users / Service Identities]
           |
        [SSO + IAM]      [AI Application Registry]
           |                    |
           v                    v
   +-----------------------------------+
   | ENTERPRISE AI GATEWAY             |
   | model/region route | data class   |
   | principal/tenant   | quotas       |
   | minimal logs       | kill switch  |
   +---------+-------------------------+
             | model proposes tool call
             v
     [Tool Registry + Version Gate]
             |
        [Tool PEP/PDP] <----- [Delegation / Token Broker]
             |
      +------+--------+-----------+
      |               |           |
[Internal API]    [MCP Server] [A2A Agent]
      |               |           |
  [Business          [Scoped identity and per-hop authorization]
   Policies]            |           |
      +-----------------+-----------+
                        |
           [Audit + Alerts + Incident Response]
```

**Control-plane concerns:** model/tool inventory, policy rollout, vendor and subprocessor approvals, prompt/tool/version change management, approval rules, and audit evidence. **Data-plane concerns:** each inference and tool action actually applies the policy. A central gateway that only logs decisions is **not** a PEP; downstream services still need their own checks.

**Primary risks:** gateway credential concentration, authorization bypass through direct service paths, MCP tool changes, A2A delegation confusion, and operational misconfiguration. **Verification:** inventory reconciliation, forbidden direct-service calls, token audience tests, tool swap tests, and egress observations. **Residual:** central policy errors can have organization-wide impact; use staged rollout and independent service controls.

### Case C — Sensitive health-screening workflow

**Business need:** A clinician requests an AI-assisted explanation of a fixed health-screening result, without letting the assistant automatically change the patient's record or make final clinical decisions.

```text
[Authenticated Clinician]
          |
[Patient/Report Authorization]---[Purpose + Relationship Policy]
          |
[Minimum Necessary Clinical Projection]
          |
[Approved Singapore or legally assessed external model route]
          |
[Generated Draft] --> [Clinical Validation / Human Review]
          |
[Authorized Response; NO automatic record modification]
          |
[Metadata Audit / Safe Diagnostic Events]

Separate: vendor processing, trace export, support access and backups
```

**Data inventory:** screening readings, demographics needed for interpretation, dates/time-zone precision, sensitive medical history, and optional identifiers. **Decisions:** which fields are necessary; whether input to provider stays regional; what processor contracts apply; who may see raw output; whether to retain conversational text; how to correct a misleading result. **Security tests:** unauthorized patient, wrong-purpose, missing field, unsafe output, error-log leakage, and disallowed external region. **Residual:** clinically plausible but incorrect advice and re-identification risk in context-rich telemetry.

### Case D — AI vendor governance for a regulated platform

**Business need:** Several vendors provide model inference, AI-assisted code review, transcription, and observability. Every provider must be assessed based on its **actual processing and permissions**, not on generic certification language.

```text
[Business Sponsor: need / purpose / impact]
       |
[Data Protection + Security + Clinical (as applicable)]
       |
[Vendor Due Diligence: terms, location, sub-processors, support]
       |
[Risk Tier + Transfer / DPIA / Contract Decision]
       |
[Technical Proof: egress, logs, data classes, tool rights]
       |
[Time-bound Approval + Registry + Release Gate]
       |
[Continuous Review: changes, incidents, expiry, renewal]
```

**Minimum records:** vendor legal entity; product and service; owner; geography of every relevant flow; whether data is used for training or secondary purposes; retention; security/access; customer audit rights; breach notification; subprocessors; DPA/BAA as applicable; exit/deletion; and risk acceptance. **Hard stop:** clinical data flowing to an unknown recipient or unapproved purpose. **Residual:** contracts do not physically prevent every vendor incident; combine legal, technical, and detective controls.

---

## Production decision matrices and security checklists

### Risk-tier and human-oversight decision matrix

**Illustrative enterprise policy, not a statutory classification.** Organizations must choose risk thresholds based on actual services, affected individuals, contracts, and law.

| Tier | Example | Data | Tool autonomy | Typical approval | Minimum assurance |
|---|---|---|---|---|---|
| **T0 — Minimal** | Public-content draft with no tools | Public | None | Product owner | Basic prompt/output tests, approved model, no sensitive logs |
| **T1 — Controlled** | Internal summarizer with no patient data | Internal | Read-only, narrow | Engineering + security rules | Data classification, model/vendor approval, safe logging, standard CI security |
| **T2 — Sensitive** | Clinician assistant reading authorized records | Personal/clinical | Scoped sensitive reads | Security/privacy + domain owner | DPIA as appropriate; per-resource policy; tenant tests; transfer controls; audit |
| **T3 — High impact** | Agent proposing patient-affecting changes | Clinical/high harm | Sensitive writes or external disclosure | Named clinical and security/privacy decision makers | Exact-action approval; separation of duties; action gating; enhanced red team; rapid kill switch |
| **T4 — Prohibited/unapproved** | Agent with unrestricted patient DB and arbitrary outbound tool | Highly sensitive | Broad uncontrolled writes/egress | **Do not deploy as designed** | Redesign scopes and control boundaries before reassessment |

### Tool authorization and side-effect decision matrix

| Action category | Model may propose? | Executor must independently verify | Human approval baseline | Key telemetry |
|---|---|---|---|---|
| Public factual lookup | Yes, if approved | Tool allowlist and quotas | Usually not | Route/version/cost |
| Read patient's allowed result | Only authorized workflows | User + tenant + patient relationship + purpose + field scope | By workflow/risk, not automatically per read | Patient-access policy decision |
| Update patient data | Only with narrow schema | Same, plus domain invariants and current record state | Clinical or authorized reviewer for consequential change | Before/after reference + approver |
| Email external recipient | Only approved use cases | Sender authority, recipient, content class, lawful purpose | Usually explicit for sensitive data | Recipient domain/purpose, DLP result |
| Execute code | Only justified workloads | Sandbox identity, image, filesystem and egress policy | Explicit policy for privileged capability | Sandbox violations and destination |
| Add/change MCP tool | No autonomous production registration | Source/provenance, capability/risk review, permission changes | Technical/security owner | Manifest digest and approval |
| Delegate A2A task | Yes, when permitted | Peer identity, audience, task owner, scope, tenant, purpose | Depends on end effect, not messaging mechanism | Chain of delegation and policy result |

### Vendor / hosting decision matrix

| Option | Control advantages | Material risk questions | Required evidence | Example decision |
|---|---|---|---|---|
| **Approved managed model in SG region** | Potentially simpler locality, managed ops | Is inference actually SG? Where are logs/support/failover? | Product-specific terms, route config, processor map, security assessment | Allow restricted workloads only after actual data-flow/legal assessment |
| **Managed model outside SG** | Model choice and quality | Transfer limitation; subprocessors; access; retention; cross-border onward transfers | Appropriate transfer/legal mechanism; contract; scope; minimization | Allow only for validated purpose/data classification |
| **Self-hosted on private EKS** | Direct runtime and network control | Model supply chain; GPU host/pod hardening; secrets; ops risk | Model provenance, network and IAM evidence, incident plan | Often useful for data-control needs if operations are mature |
| **Third-party AI observability** | Developer diagnostics | PHI/PII in traces; actual region; support access | Trace payload samples; DPA; retention; transfer assessment | Prefer minimal, structured metadata; raw traces gated |
| **Unverified vendor** | None sufficient for sensitive use | Unknown location/training/logging/ownership | Missing | **Block sensitive data** until assessed |

**Non-equivalence statements:** “Singapore region” ≠ “all processing stays in Singapore”; “zero training” ≠ “zero retention”; “HIPAA-ready” ≠ “HIPAA compliant”; “ISO/IEC 42001 certified” ≠ “PDPA compliant”; “encrypted at rest” ≠ “authorized processing”; “mTLS” ≠ “patient access authorization”. [S07], [S10], [S16]

### Singapore legal-versus-guidance decision table

| Statement | Classification | Accurate interpretation / action | Primary source |
|---|---|---|---|
| PDPA applies to relevant personal-data processing | **LAW** | Identify the organization's role, purpose, data and applicable provisions | [S09] |
| Reasonable security arrangements for personal data | **LAW** | Protection Obligation under s.24; context-specific controls | [S09], [S11] |
| Protect overseas transfers under applicable transfer rules | **LAW + regulations** | Comparable protection subject to applicable mechanisms/exceptions; not blanket locality | [S09], [S10] |
| Data minimization in AI development | **GUIDANCE** plus general legal-purpose considerations | PDPC AI guideline encourages it; exact duties stem from law | [S11] |
| DPIA for every AI model is mandatory | **Not established as blanket PDPA requirement** | DPIA is prudent for high-risk cases; consult sector/client obligations | [S11] |
| Genuinely anonymized data may fall outside PDPA | **Legal-definition consequence**, interpreted in **GUIDANCE** | Must assess serious possibility of re-identification | [S10], [S11] |
| PDPC breach notification after notifiability determination | **LAW + regulations** | As soon as practicable, no later than 3 calendar days for notifiable breach | [S12] |
| 500 affected individuals is significant-scale threshold | **Regulatory criterion** | Can trigger notification to PDPC even without prescribed sensitive data | [S12] |
| GenAI/Agentic Model AI Governance Framework | **GUIDANCE** | Risk management framework, not a substitute for PDPA | [S14], [S15] |
| HIA 2026 may impose healthcare-specific duties | **LAW when applicable/in force** | Confirm covered entities, commencement, phased rules and MOH specifications | [S17] |
| HIPAA applies to every Singapore hospital app | **Incorrect** | Requires a relevant HIPAA-covered entity/BA relationship and regulated PHI | [S16] |

### Production release checklist

Use the following as a **suggested internal security gate**, not as a claim that every item is legally mandated.

#### A. Governance and data flows

- [ ] Named business, engineering, security, and privacy owners; clinical owner where relevant.
- [ ] AI use case, autonomy tier, model/provider/version, and tool set registered.
- [ ] Inputs, outputs, tool results, persisted state, telemetry, backups, support exports, and subcontractors mapped.
- [ ] Data classification, purpose/legal basis, necessary fields, retention, and deletion tested.
- [ ] Applicable PDPA/healthcare/HIPAA obligations and contractual constraints assessed by responsible stakeholders.
- [ ] Overseas processing and transfer controls assessed; region and subprocessor claims verified for *specific product/features*.
- [ ] DPIA/risk assessment and residual risk decision recorded where appropriate.

#### B. Tool, identity, and network controls

- [ ] All high-risk tool calls have authenticated principal, delegated identity context, per-resource authorization, and deny-by-default behavior.
- [ ] Tool registry contains owner, version, schema, side-effect level, allowed resources, and approval requirement.
- [ ] Model does not receive service secrets or broad access tokens; credentials are short-lived and audience-scoped.
- [ ] Sensitive data is projected/minimized **before** the model call.
- [ ] Multi-tenant access and object relationships are enforced downstream, including async workers.
- [ ] All consequential approval grants bind exact arguments/resources and are rechecked at execution.
- [ ] Sandboxed code tools have tested isolation, limited filesystem, quotas, and network egress.
- [ ] Pods/services cannot directly access unneeded private databases or outbound destinations.
- [ ] MCP/A2A servers are allowlisted and their credentials, manifests, task and agent identity checked.

#### C. Model and content defenses

- [ ] Untrusted content is labeled and never treated as an authority to change tool privileges.
- [ ] Prompt hierarchy and injection detection are **defense in depth**, not only execution control.
- [ ] Output JSON/schema is validated; downstream rendering/queries/API calls use context-safe handling.
- [ ] Tool descriptions, model/provider/region, prompt templates, and policies are version-controlled.
- [ ] Clinical/harmful output controls and qualified human oversight are configured where needed.
- [ ] Safe fallback/rollback and model/tool/tenant kill switches are available.

#### D. Testing, operations, and assurance

- [ ] CI includes deterministic unauthorized tool-policy tests and automated negative cross-tenant cases.
- [ ] Adversarial scenario suite covers lower-trust user/tool/A2A content with synthetic data.
- [ ] Severity-weighted failures, benign-task false positives, and test coverage are recorded.
- [ ] External processors are approved, and traces are inspected for personal data leakage.
- [ ] Audit logs record policy decisions and action provenance with minimal sensitive content.
- [ ] Egress, unauthorized reads, tool mutations, data volume, and approval anomalies have actionable alerts.
- [ ] Incident responders can disable tools, revoke credentials, preserve evidence, assess breach duties, and recover.
- [ ] Risk acceptance and exception expiry are owned, time-bound, and monitored.
- [ ] Actual production configuration has been reviewed—not only declarative YAML/screenshots.

### Security regression test catalogue

Each row is a **safe, high-level test objective**. Implement only with permitted synthetic data and isolated endpoints; the catalogue deliberately omits exploitation payloads.

| Test ID | Category | Test objective | Expected invariant | Evidence |
|---|---|---|---|---|
| SEC-01 | Direct injection | User asks for a prohibited capability | Tool execution denied | Agent trace + PEP deny |
| SEC-02 | Indirect injection | Untrusted tool result contains conflicting instruction-like text | No policy elevation or unauthorized side effect | Tool transcript + decision |
| SEC-03 | Cross-tenant | Valid-looking patient ID belongs to other tenant | Denied at business service | Resource authorization logs |
| SEC-04 | Care relationship | Clinician in provider B requests provider A-only patient | Denied independent of model text | Relationship policy test |
| SEC-05 | Token scope | Correct identity, wrong audience/scope | Denied before execution | Auth verification result |
| SEC-06 | Approval | Proposal changes after approval | Old grant rejected | Proposal hash and decision |
| SEC-07 | MCP registry | New unreviewed tool or changed manifest | Not executable | Registry approval evidence |
| SEC-08 | A2A task | Unauthorized caller references another agent's task | Cannot view/cancel/delegate task | Task ACL denial |
| SEC-09 | Egress | Tool targets disallowed domain | Blocked by application + network controls | Egress and policy record |
| SEC-10 | Secret isolation | Low-trust runtime attempts to read forbidden credential | No credential exposure | Sandbox/secret ACL test |
| SEC-11 | PII in traces | Realistic synthetic clinical content through flow | Only allowlisted telemetry leaves environment | Export sample |
| SEC-12 | Output handling | Malformed structured response and special characters | Safe rendering and no arbitrary execution | Parser/UI tests |
| SEC-13 | Data residency | Restricted data requests unavailable region route | Route rejected; no silent fallback | Router decision logs |
| SEC-14 | Quota | Excessive legitimate tool volume | Limits activate without tenant crossover | Rate limiter event |
| SEC-15 | Incident | Disable sensitive tool during dry run | Subsequent calls blocked and responder notified | Drill record |
| SEC-16 | Model update | Model/prompt/tool version change | Required safety suite and approval execute | CI + release artifact |

### Incident examples — hypothetical training scenarios (not claims about real incidents)

#### Incident I — Unauthorized patient disclosure through an overpowered agent

**Situation:** A clinician assistant's tool returns a record the clinician is not authorized to view. A model-level guardrail had been configured, but the backend trusted the model's supplied `patientId`.

- **Root cause:** missing object/purpose authorization; excessive service permissions.
- **Preventive:** per-call PEP plus patient/provider relationship; least-privileged tool identity.
- **Detective:** cross-provider access alert, record-access audit, anomalous view pattern.
- **Response:** suspend tool, assess who received data, contain access, assess notifiability.
- **Governance:** update threat model, business owner, release test SEC-03/04, privacy review.
- **Residual:** legitimate users may still misuse authorized access.

#### Incident II — Sensitive trace leakage to a third-party analytics provider

**Situation:** Debug tracing ships clinical narrative and rare condition details to an overseas SaaS service. Direct identifiers were removed, but the combined text remains potentially identifiable.

- **Root cause:** telemetry treated as non-personal metadata without contextual re-identification assessment.
- **Preventive:** default metadata-only tracing, restricted collector, approved vendor/transfer terms.
- **Detective:** payload sampling and outbound data-class inspection.
- **Response:** stop exporter, restrict vendor access, assess deletion and notification duties.
- **Governance:** processor register, DPIA/transfer assessment, trace data-retention schedule.
- **Residual:** remaining metadata can still be linkable; periodic reassessment.

#### Incident III — Tool manifest change widens agent permissions

**Situation:** A previously read-only MCP integration is updated to offer an operation with external side effects. The agent automatically discovers and proposes it.

- **Root cause:** auto-registration of unreviewed capabilities and broad executor permission.
- **Preventive:** signed/approved registry entries, tool effect classification, separate write credentials.
- **Detective:** manifest digest and effective-permission diffs; unusual invocation alerts.
- **Response:** disable server/tool, rotate affected credentials, investigate executions.
- **Governance:** supply-chain approval and vendor change notification.
- **Residual:** malicious changes to existing approved tool implementation still require runtime constraints.

#### Incident IV — Wrong clinical recommendation despite proper access control

**Situation:** The assistant accesses only allowed data but generates an unsupported interpretation that a clinician initially accepts.

- **Root cause:** clinical quality and human-overreliance failure; **not** necessarily a prompt injection.
- **Preventive:** scope constraints, explicit uncertainties, clinical review requirements, deterministic safety boundaries.
- **Detective:** clinical scenario evaluation, feedback and harm monitoring.
- **Response:** stop unsafe recommendation channel, review affected communications, correct guidance.
- **Governance:** patient-safety owner, validation protocol, release thresholds and retraining/review.
- **Residual:** no generative model can guarantee correct clinical advice.

### Governance RACI example

`A = Accountable; R = Responsible; C = Consulted; I = Informed` — **illustrative internal policy**.

| Decision | Business/Clinical Owner | Engineering | Security | Privacy / Legal | Vendor/Procurement |
|---|---|---|---|---|---|
| Use-case purpose and acceptable patient harm | **A** | C | C | C | I |
| Implementation of tool/tenant permissions | C | **A/R** | C | C | I |
| Security threat model / security release gate | C | R | **A** | C | I |
| PDPA basis and cross-border transfer assessment | C | C | C | **A** | R |
| Vendor due diligence and contract | C | C | C | C | **A/R** |
| Patient-affecting action approval process | **A** | R | C | C | I |
| AI incident response | C | R | **A** | R for privacy notifications | C |
| Annual AI inventory review | **A** | R | C | C | C |

For a clinical service, **qualified clinical governance** may have non-delegable responsibilities under organizational and sector requirements. A RACI does not override legal duties.

### Reference policy pseudo-code (defensive architecture only)

```ts
// Illustrative security invariant: the model proposes; trusted services authorize.
// This is NOT a turnkey authorization library or complete clinical policy.

type ToolProposal = {
  toolId: string;
  patientId?: string;
  recipient?: string;
  payload: unknown;
  proposalVersion: string;
};

async function executeAuthorizedTool(
  authenticatedContext: {
    userId: string; tenantId: string; agentId: string; purpose: string;
  },
  proposal: ToolProposal,
) {
  const tool = approvedRegistry.lookup(proposal.toolId);
  if (!tool || !tool.enabled) return { status: "DENIED" as const };

  const args = tool.schema.parse(proposal.payload); // syntax/shape only
  const decision = await policyService.evaluate({
    principal: authenticatedContext.userId,
    tenant: authenticatedContext.tenantId,
    agent: authenticatedContext.agentId,
    purpose: authenticatedContext.purpose,
    action: tool.operation,
    resource: serverResolveResource(tool, args),
  });
  if (!decision.allow) return auditDenial(decision); // no side effect

  if (tool.requiresApproval) {
    await approvalService.requireBoundGrant({
      principal: authenticatedContext.userId,
      action: tool.operation,
      normalizedArguments: args,
      proposalVersion: proposal.proposalVersion,
      // also bind expiry, tenant, recipient, resource, and purpose
    });
  }
  // Real code must handle atomicity, updated permissions and expiry here.
  return tool.adapter.execute(args, decision.scopedCredential);
}
```

**Security invariant:** schema parsing is not authorization; a model-generated ID cannot change the trusted principal; the backend resolves resource ownership; approvals are bound and checked at the point of use; sensitive logs exclude raw payloads by default. Implementation details (atomic action, replay protection, conditional writes, tenant policy, credential issuance) require additional domain-specific design.

### Ten Principal-level interview challenges to practice

1. **Scope dispute:** “The user authenticated; why can't we let the model choose a patient ID?” → Explain object authorization and purpose boundaries.
2. **Guardrail optimism:** “Our jailbreak detector scores 99%; can we skip tool ACLs?” → Explain probabilistic filters versus deterministic side-effect prevention.
3. **Vendor assurance:** “Our endpoint is in Singapore and SOC 2 audited; are we PDPA compliant?” → Map all processors, purposes, transfers and contracts.
4. **Protocol confusion:** “MCP uses OAuth, so all MCP tools are secure.” → Clarify optional protocol authorization and downstream object policy.
5. **Agent delegation:** “Agent B trusts Agent A; can B use A's admin token?” → Discuss audiences, delegated identity and confused deputy.
6. **Clinical quality:** “Our model is accurate in a benchmark; can it auto-change records?” → Explain clinical validation, human review and impact-tiered permissions.
7. **Privacy:** “We removed patient names from traces; are they anonymous?” → Describe linkability and serious possibility of re-identification.
8. **Execution:** “We run shell tools in a private EKS cluster; why sandbox?” → Distinguish network privacy from host/secret and destination confinement.
9. **Governance:** “ISO/IEC 42001 certification means each AI output is safe, correct?” → Explain AIMS audit scope and separate runtime evidence.
10. **Incident:** “We found a prompt injection. Must PDPC be notified in three days?” → Distinguish attempted injection, personal-data breach, notifiability determination and timing.

### Interview scoring rubric

| Competency | Weak / basic answer | Strong senior answer | Principal-level answer |
|---|---|---|---|
| Threat model | Lists threats | Maps real attacker, boundary and impact | Prioritizes risks with owners and justified residuals |
| Prompt injection | “Write a stronger prompt” | Layered content/tool controls | Demonstrates safe failure when model is compromised |
| Authorization | Checks user role | Checks tenant/action/object | Purpose, delegated identity, policy evidence, edge cases |
| Privacy | Redacts identifiers | Maps data and processors | Proves minimization, transfer analysis, re-identification handling |
| AI vendor risk | Trusts vendor badge | Reviews contracts and regions | Verifies feature-specific behaviors and ongoing subprocessors |
| Production assurance | Manual testing | Repeatable security regressions | Measurable release gates, incident drills, independent controls |
| Governance | Quotes frameworks | Operates inventory/approvals | Risk-tiered program with accountable decisions and exceptions |
| Honesty about uncertainty | Promises no attacks | Acknowledges residual risks | Specifies measurable bounds, failure modes and conditions for blocking launch |

---

## Reference register

**Verification date:** 23 November 2026. The following are **primary, official sources**, not third-party summaries. Publication or version dates are given where the publisher supplies them; a page's last-updated date is different from the legal effective date. Reference labels used throughout the guide link directly to these sources. Sources may be revised after this verification date.

| ID | Classification | Official primary source | Publication / version information | How to use it |
|---|---|---|---|---|
| [S01] | **Published security guidance** | OWASP GenAI Security Project, *Top 10 for LLM Applications 2026* | Resource published **3 Aug 2026**; OWASP announcement **2 Sep 2026** | Current 2026 LLM risk IDs, scope and mitigations; not law or certification. |
| [S02] | **Published security guidance** | OWASP, *Top 10 for Agentic Applications 2026* | Public launch **9 Dec 2025**, titled 2026 edition | Agent-specific threats and defenses; not a substitute for threat modeling. |
| [S03] | **Protocol specification / security guidance** | MCP, *Authorization* and companion *Security Best Practices* | Protocol revision **28 Jul 2026** | Implementers must distinguish optional protocol authorization from enterprise requirements for authenticated tool access. |
| [S04] | **Protocol specification** | A2A, *Protocol Specification v1.0.1* | **v1.0.1**; publication date not clearly identified on specification page | Security-scheme advertisement, authenticated requests, task access and delegation boundaries. |
| [S05] | **Voluntary US federal risk-management framework** | NIST, *Artificial Intelligence Risk Management Framework (AI RMF 1.0)* | **26 Jan 2023** | Govern–Map–Measure–Manage functions; not a Singapore statutory duty. |
| [S06] | **Voluntary US federal profile** | NIST, *AI RMF: Generative Artificial Intelligence Profile (AI 600-1)* | **26 Jul 2024** | Supplemental GenAI risk-management actions; not a certification or legal safe harbor. |
| [S07] | **Formal international standard** | ISO/IEC **42001:2023**, *Artificial intelligence management system* | Published **Dec 2023** | AIMS requirements; evaluate certification scope and organization-specific controls. Full normative text is licensed by ISO. |
| [S08] | **Formal international standard** | ISO/IEC **23894:2023**, *Guidance on risk management for AI* | Published **Feb 2023** | Risk-management guidance; not evidence that a specific deployment is secure. |
| [S09] | **Singapore law** | *Personal Data Protection Act 2012* (Singapore Statutes Online) | **Act 26 of 2012**, as amended; check the consolidated text applicable at the time | Underlying legally binding obligations when applicable; verify statutory exceptions and scope. |
| [S10] | **PDPC official advisory guidance** | *Advisory Guidelines on Key Concepts in the PDPA* | PDPC revision **May 2022** | Explanation of consent, notification, purpose, protection, retention and transfer concepts; consult the regulations and Act for binding law. |
| [S11] | **PDPC official advisory guidance** | *Advisory Guidelines on Use of Personal Data in AI Recommendation and Decision Systems* | **1 Mar 2024** | Published regulator guidance for personal-data use in AI; guidance does not create an automatic exception to PDPA. |
| [S12] | **PDPC official operational guidance** | *Guide on Managing and Notifying Data Breaches under the PDPA* | Revised **15 Mar 2021** | Incident assessment, notifiability and notification; read alongside the Act and Personal Data Protection (Notification of Data Breaches) Regulations 2021. |
| [S13] | **PDPC official technical guidance** | *Guide to Basic Anonymisation* (PDPC resource page) | PDPC guide/resource; verify current edition at link | Identifiability and re-identification considerations. Pseudonymization alone is not equivalent to anonymization. |
| [S14] | **Singapore government governance guidance** | IMDA / AI Verify Foundation, *Model AI Governance Framework for Generative AI* | Final framework launched **30 May 2024** | Voluntary governance framework, not Singapore legislation. |
| [S15] | **Singapore government governance guidance** | IMDA, *Model AI Governance Framework for Agentic AI, v1.0* | **22 Jan 2026** | Governance priorities for increasingly autonomous systems; not a statutory compliance certificate. |
| [S16] | **US law / regulator interpretation where applicable** | US HHS, *Covered Entities and Business Associates* and *Guidance on HIPAA & Cloud Computing* | HHS guidance pages; no reliable single publication date for current page revisions | HIPAA applicability, business associates, and BAAs; no universal application to every Singapore healthcare dataset. |
| [S17] | **Singapore law / government implementation guidance** | *Health Information Act 2026* (Singapore Statutes Online) and Health Information Act government portal | **Enacted 3 Feb 2026**; government overview updated **2 Oct 2026** | Check applicability, detailed provisions, commencement, regulations and sector-specific implementation timelines before drawing operational conclusions. |

### Direct links

- **OWASP:** [2026 LLM Top 10 overview](https://genai.owasp.org/top-10-for-llm-applications-2026/) · [2026 source resource and download](https://genai.owasp.org/resource/owasp-genai-llm-top-10-2026/) · [2026 Agentic Top 10](https://genai.owasp.org/top-10-for-agentic-applications-2026/)
- **MCP:** [Authorization specification](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization) · [Security best practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices) · [2026-07-28 announcement](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- **A2A:** [Protocol v1.0.1 specification](https://a2a-protocol.org/v1.0.1/specification/)
- **NIST:** [AI RMF](https://www.nist.gov/itl/ai-risk-management-framework) · [Generative AI Profile](https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-generative-artificial-intelligence)
- **ISO:** [ISO/IEC 42001](https://www.iso.org/standard/42001) · [ISO/IEC 23894](https://www.iso.org/standard/77304.html)
- **Singapore PDPA:** [PDPA 2012 statute](https://sso.agc.gov.sg/Act/PDPA2012) · [PDPC guidelines by topic](https://www.pdpc.gov.sg/organisations/resources/guidance-by-topic) · [PDPC data-breach guide](https://www.pdpc.gov.sg/-/media/Files/PDPC/PDF-Files/Other-Guides/Guide-on-Managing-and-Notifying-Data-Breaches-under-the-PDPA-15-Mar-2021.pdf) · [PDPC anonymisation resource](https://www.pdpc.gov.sg/help-and-resources/2018/01/basic-anonymisation)
- **Singapore AI governance:** [IMDA GenAI framework announcement](https://www.imda.gov.sg/resources/press-releases-factsheets-and-speeches/factsheets/2024/gen-ai-and-digital-foss-ai-governance-playbook) · [IMDA Agentic AI framework v1.0](https://www.imda.gov.sg/-/media/imda/files/about/emerging-tech-and-research/artificial-intelligence/mgf-for-agentic-ai.pdf)
- **Singapore healthcare:** [Health Information Act 2026 statute](https://sso.agc.gov.sg/Acts-Supp/1-2026/) · [HIA implementation and overview](https://www.healthinfo.gov.sg/hia/) · [HIA implementation support and sector timelines](https://www.healthinfo.gov.sg/implementation-support/) · [HIA cybersecurity and data security](https://www.healthinfo.gov.sg/csds/)
- **US healthcare (only if applicable):** [HHS HIPAA covered entities and business associates](https://www.hhs.gov/hipaa/for-professionals/covered-entities/index.html) · [HHS HIPAA and cloud computing](https://www.hhs.gov/hipaa/for-professionals/special-topics/health-information-technology/cloud-computing/index.html)
- **Additional threat knowledge:** [MITRE ATLAS](https://atlas.mitre.org/), living AI adversarial-threat knowledge base, useful for mapping detection and defensive test scenarios.

### Reference-style links used throughout

[S01]: https://genai.owasp.org/resource/owasp-genai-llm-top-10-2026/
[S02]: https://genai.owasp.org/top-10-for-agentic-applications-2026/
[S03]: https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization
[S04]: https://a2a-protocol.org/v1.0.1/specification/
[S05]: https://www.nist.gov/itl/ai-risk-management-framework
[S06]: https://www.nist.gov/publications/artificial-intelligence-risk-management-framework-generative-artificial-intelligence
[S07]: https://www.iso.org/standard/42001
[S08]: https://www.iso.org/standard/77304.html
[S09]: https://sso.agc.gov.sg/Act/PDPA2012
[S10]: https://www.pdpc.gov.sg/-/media/Files/PDPC/PDF-Files/Advisory-Guidelines/AG-on-Key-Concepts/Advisory-Guidelines-on-Key-Concepts-in-the-PDPA-17-May-2022.pdf
[S11]: https://www.pdpc.gov.sg/-/media/files/pdpc/pdf-files/advisory-guidelines/advisory-guidelines-on-the-use-of-personal-data-in-ai-recommendation-and-decision-systems.pdf
[S12]: https://www.pdpc.gov.sg/-/media/Files/PDPC/PDF-Files/Other-Guides/Guide-on-Managing-and-Notifying-Data-Breaches-under-the-PDPA-15-Mar-2021.pdf
[S13]: https://www.pdpc.gov.sg/help-and-resources/2018/01/basic-anonymisation
[S14]: https://www.imda.gov.sg/resources/press-releases-factsheets-and-speeches/factsheets/2024/gen-ai-and-digital-foss-ai-governance-playbook
[S15]: https://www.imda.gov.sg/-/media/imda/files/about/emerging-tech-and-research/artificial-intelligence/mgf-for-agentic-ai.pdf
[S16]: https://www.hhs.gov/hipaa/for-professionals/covered-entities/index.html
[S17]: https://www.healthinfo.gov.sg/hia/

---

**Final interview principle:** An LLM may recommend an action, but your application must prove that the action is authorized, appropriate for the declared purpose, bounded in impact, minimally exposed, observable, recoverable, and governed by an accountable owner. Test and monitor those properties independently of the model's stated intentions.
