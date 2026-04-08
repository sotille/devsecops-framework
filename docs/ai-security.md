# AI and LLM Security Integration

The integration of large language models (LLMs) and AI-assisted development tools into software delivery pipelines introduces a new class of security risks that extend beyond traditional application security concerns. This document defines security controls, architectural guidance, and governance requirements for organizations that use AI in their development workflow.

---

## Scope

This document covers three distinct uses of AI in software delivery:

1. **AI-assisted code generation** — Developers use AI coding assistants (GitHub Copilot, Amazon Q Developer, Cursor, JetBrains AI) to generate, complete, or refactor code
2. **AI in CI/CD pipelines** — LLMs are invoked as part of the automated pipeline (code review automation, vulnerability explanation, release notes generation)
3. **AI-powered applications** — The software being built includes LLM inference endpoints, RAG pipelines, or agent frameworks as product features

Security considerations for each use case are distinct and addressed separately.

---

## Use Case 1: AI-Assisted Code Generation

### Risk Profile

AI coding assistants generate code based on training data and context window input. The generated code:
- May reproduce patterns from vulnerable training data (known CVEs, insecure coding patterns)
- May include hardcoded credentials or API keys from examples in the training corpus
- May introduce dependencies on packages that do not exist (hallucinated package names) — which can be exploited through dependency confusion attacks
- Is generated without awareness of the application's specific security requirements or threat model

### Controls

**1.1 Treat AI-generated code as untrusted input**

Apply the same SAST, SCA, and secrets scanning to AI-generated code as to human-authored code. Do not exempt AI-generated code from any security gate.

**1.2 Enable hallucinated package detection**

AI models can generate `import` or `require` statements referencing packages that do not exist in the target package registry. An attacker who monitors common AI code generation patterns can register these names and serve malicious code.

- Run SCA with dependency confusion detection mode enabled
- Verify every new dependency is intentionally introduced, not hallucinated
- Use a private registry mirror that restricts package resolution to approved sources

**1.3 Define an acceptable use policy for AI coding assistants**

| Topic | Policy |
|-------|--------|
| Code containing secrets | AI assistants must not be given context containing production secrets, API keys, or credentials |
| Proprietary code in context | Assess whether AI assistant vendor processes prompts; classify code accordingly |
| Generated code review | AI-generated code blocks must be reviewed by a human before merging, particularly for authentication, authorization, and data handling logic |
| Model selection | Use approved AI assistant tools; do not connect unofficial or personal AI tools to company codebases |

**1.4 Security requirements apply regardless of authorship**

Secure coding standards (input validation, parameterized queries, output encoding, proper error handling) apply equally to human-authored and AI-generated code. Security champions should treat AI-generated code with heightened scrutiny, as the model may optimize for functionality over security.

---

## Use Case 2: AI in CI/CD Pipelines

### Risk Profile

Invoking an LLM as part of an automated pipeline introduces:
- **Prompt injection risk** — code or commit messages may contain adversarial instructions that manipulate the LLM's output (e.g., causing a code review bot to approve malicious changes)
- **Confidentiality risk** — pipeline context sent to an LLM API may contain secrets, PII, or proprietary IP
- **Non-determinism** — LLM outputs are non-deterministic; using them in security-critical decisions (approve/block) without deterministic guardrails is unreliable
- **Supply chain risk** — LLM provider APIs are external dependencies in the pipeline

### Controls

**2.1 Do not use LLM output as the sole basis for security-critical pipeline decisions**

LLM-assisted tools may be used to explain findings, prioritize issues, or generate remediation suggestions. They must not be the sole mechanism for approving or blocking a build or deployment. All security gates must be based on deterministic, policy-driven controls.

**2.2 Sanitize pipeline context before sending to LLM APIs**

Before sending any pipeline data to an external LLM API:
- Remove environment variables, secrets, and credentials from the context
- Assess whether code contents are appropriate to send externally; apply data classification
- Use LLM APIs within a private endpoint or VPC if the vendor supports it

**2.3 Treat LLM-generated pipeline output as untrusted**

LLM-generated output that influences pipeline behavior (e.g., a list of files to scan, a risk score) must be validated against a deterministic schema before use. Reject any LLM output that does not conform to the expected structure.

**2.4 Prompt injection mitigation**

When the LLM is given code, commit messages, PR titles, or user-provided content as input, apply prompt injection defenses:
- Separate system instructions from user-controlled content with clear delimiters
- Validate that LLM output does not contain instructions to modify pipeline behavior
- Use output parsing that rejects unexpected tokens

**2.5 LLM API key management**

Treat LLM API keys as high-value credentials:
- Store in secrets manager, not in environment variables or repository
- Scope API keys to the minimum capabilities required
- Rotate API keys on a quarterly schedule
- Alert on API key usage outside expected patterns (rate, time of day, calling IP)

---

## Use Case 3: AI-Powered Applications (LLM as a Product Feature)

### Risk Profile

Applications that include LLM inference as a product feature introduce risks to both the application and the users it serves:

- **Prompt injection** — user-supplied input may instruct the LLM to bypass application instructions, leak system prompts, or take unintended actions
- **Indirect prompt injection** — content retrieved from external sources (documents, web pages, emails) may contain injected instructions that execute when the LLM processes the retrieved content
- **Data exfiltration via LLM** — a compromised or manipulated LLM agent may exfiltrate data through its output or through tool calls
- **Insecure tool use** — LLM agents with access to tools (file system, database, API calls) may be manipulated into taking destructive actions
- **Model confidentiality** — system prompts may contain proprietary business logic or data that should not be disclosed to users
- **Hallucination in security-relevant contexts** — LLMs may generate plausible but incorrect output in contexts where accuracy is security-critical

### OWASP LLM Top 10 Control Mapping

| OWASP LLM Risk | Control |
|----------------|---------|
| LLM01: Prompt Injection | Input sanitization; separate untrusted content from instructions; output validation |
| LLM02: Insecure Output Handling | Validate LLM output before use; encode before rendering; do not execute LLM output as code |
| LLM03: Training Data Poisoning | Use vetted base models; monitor model behavior for anomalies |
| LLM04: Model Denial of Service | Rate limiting on inference endpoints; input length limits; cost controls |
| LLM05: Supply Chain Vulnerabilities | Pin model versions; verify model checksums; audit third-party plugins |
| LLM06: Sensitive Information Disclosure | Classify data before including in prompts; apply output filtering |
| LLM07: Insecure Plugin Design | Least-privilege tool access; validate all tool inputs; human-in-the-loop for destructive actions |
| LLM08: Excessive Agency | Limit scope of agent actions; require confirmation for irreversible actions |
| LLM09: Overreliance | Do not use LLM output as the sole decision in security-relevant contexts |
| LLM10: Model Theft | Access controls on model weights and inference endpoints; audit model access |

---

### Architectural Controls for LLM Applications

**3.1 Defense in depth for prompt injection**

No single control fully prevents prompt injection. Apply multiple layers:

```
User Input
    ↓
Input validation (length, format, content filter)
    ↓
Prompt construction (separate system/user/context with clear markers)
    ↓
LLM inference
    ↓
Output validation (schema validation, content filter, anomaly detection)
    ↓
Action authorization (check permissions before executing tool calls)
    ↓
Output rendering (encode for context — HTML encoding, markdown sanitization)
```

**3.2 Least-privilege tool access for agents**

LLM agents that invoke tools (APIs, databases, file systems) must operate under least-privilege:
- Each tool should be scoped to the minimum data required (read-only where possible)
- Destructive operations (delete, update, send message) require explicit user confirmation
- Tool calls should be logged and auditable
- Agent identity should be separate from the application's service identity

```python
# Example: tool with explicit permission check before destructive action
def delete_record(record_id: str, user_context: UserContext) -> str:
    # Never execute destructive actions based solely on LLM instruction
    if not user_context.has_permission("delete_record"):
        raise PermissionError("User does not have delete permission")
    if not user_context.confirmation_received:
        return "Confirmation required before deleting record. Please confirm."
    # Proceed only after human confirmation
    return db.delete(record_id)
```

**3.3 System prompt protection**

System prompts often contain proprietary logic and should be treated as sensitive configuration:
- Do not expose system prompts in client-side code or API responses
- Monitor for prompt extraction attacks (inputs designed to elicit system prompt disclosure)
- If the system prompt contains business logic, treat it as an application secret

**3.4 RAG pipeline security**

Retrieval-Augmented Generation (RAG) pipelines that inject retrieved content into LLM prompts are particularly vulnerable to indirect prompt injection:
- Sanitize retrieved content before injecting it into prompts — strip HTML, limit markdown
- Apply content filtering to retrieved content to detect injection attempts
- Consider whether retrieved content should be clearly labeled as untrusted in the prompt

**3.5 LLM security testing in CI/CD**

Include LLM-specific security tests in the CI pipeline:

```yaml
# Example: automated prompt injection testing
- name: Run prompt injection tests
  run: |
    python -m pytest tests/security/test_prompt_injection.py \
      --model-endpoint ${{ env.LLM_ENDPOINT }} \
      --test-suite owasp-llm-top10
```

Test categories to cover:
- Direct prompt injection via user input fields
- Indirect prompt injection via document upload
- System prompt extraction attempts
- Tool call manipulation (attempting to invoke tools with unauthorized parameters)
- Output filter bypass attempts

---

## Governance Requirements

### AI Usage Policy

Organizations using AI in their development workflow should document and communicate:

1. **Approved AI tools** — which AI coding assistants, LLM APIs, and model providers are approved for use
2. **Data classification constraints** — what types of data may be sent to external AI APIs
3. **Code review requirements** — minimum human review requirements for AI-generated code
4. **Incident response** — how to respond to AI-specific security incidents (compromised API key, prompt injection exploit, hallucinated dependency attack)

### Audit and Monitoring

- Log all LLM API calls including model name, approximate token count, and calling service — not content
- Alert on anomalous API key usage (unusual request volume, unexpected calling IP)
- Include AI-generated code sections in security-focused code review checklists
- Track the ratio of AI-generated to human-authored code — a sudden spike may indicate an understaffed team taking excessive shortcuts

### Model and Dependency Versioning

Treat LLM model versions as software dependencies:
- Pin to specific model versions where the API supports it
- Test application behavior when upgrading to a new model version
- Include LLM provider APIs in third-party dependency risk assessments

---

---

## Use Case 4: Agentic AI Systems and Autonomous Pipelines

As of 2026, AI agents — systems that autonomously plan, take actions, and invoke tools across multi-step workflows — are deployed at scale in engineering organizations. Agents orchestrate CI/CD steps, manage infrastructure, perform code review, run security scans, and interact with external systems. This operational model introduces a substantially different threat surface from interactive LLM usage.

### Risk Profile

Agentic systems differ from conventional LLM integrations in the following ways:

- **Autonomous, multi-step action sequences** — an agent may execute dozens of tool calls without human review between steps. A single compromised decision early in a sequence can propagate downstream
- **Persistent state and memory** — agents with long-term memory or context storage retain sensitive information across sessions, creating data classification and retention risks
- **Lateral movement via tool access** — agents with broad tool access (API calls, file system, shell execution) that are compromised can traverse organizational systems similarly to a compromised service account
- **Opaque reasoning chains** — complex agent reasoning is not easily auditable; malicious or erroneous intermediate steps may not surface in output logs
- **Cross-agent trust** — multi-agent systems where one agent invokes another create trust delegation chains; compromise of a subordinate agent can propagate upward

### Controls

**4.1 Enforce minimum tool scope per agent**

Define tool access lists per agent and enforce at the orchestration layer. An agent performing code review must not have write access to production infrastructure tools, regardless of its prompt instructions.

```yaml
# Example: Agent tool access manifest
agent: code-review-agent
allowed_tools:
  - github.read_pull_request
  - github.post_review_comment
  - semgrep.run_scan
denied_tools:
  - github.merge_pull_request
  - aws.*
  - kubectl.*
  - shell.execute
```

**4.2 Require human-in-the-loop for irreversible actions**

Define a policy of irreversible action categories that require explicit human approval before an agent proceeds. Irreversible actions include: deploying to production, deleting resources, sending external communications, modifying access control policies.

**4.3 Agent identity isolation**

Each agent must have a dedicated service identity with its own credentials, distinct from human user identities and application service accounts. Agent identities must be auditable in access logs separate from human activity.

**4.4 Constrain agent memory and context storage**

- Agent memory stores must not retain PII, credentials, or internal IP beyond the operational session unless explicitly required and authorized
- Apply data classification controls to memory contents before persistence
- Enforce retention limits: session-scoped memory must be purged on session end; long-term memory must expire and be reviewed quarterly

**4.5 Audit agent action traces**

Log every tool call made by an agent including: agent identity, tool invoked, parameters (sanitized), result status, and timestamp. Ship traces to a tamper-evident log store. Agent action traces are audit evidence for compliance and incident response.

```python
# Example: Structured agent action log entry
{
    "timestamp": "2026-04-07T04:15:00Z",
    "agent_id": "code-review-agent-7f3a",
    "session_id": "sess-9c2d1e",
    "action": "semgrep.run_scan",
    "parameters": {"target": "pr/4821", "rules": "p/default"},
    "result_status": "success",
    "findings_count": 3,
    "duration_ms": 4820
}
```

**4.6 Prompt injection defenses for agent inputs**

Agents that process external content (pull request descriptions, issue comments, document uploads, web retrieval results) must apply prompt injection defenses before that content influences further tool calls:

- Treat all externally sourced content as untrusted
- Process retrieved content in a sandboxed context before injecting into the agent's planning prompt
- Alert on agent behavior anomalies: unexpected tool calls, unusual parameter patterns, self-modification attempts

---

## Use Case 5: MCP Server Security (Model Context Protocol)

The Model Context Protocol (MCP) is an open standard for connecting LLMs and agents to external tools, data sources, and services. MCP servers expose capabilities that agents invoke as structured tool calls. As MCP adoption grows, MCP servers are becoming a critical security surface in agentic architectures.

### Risk Profile

- **Malicious MCP server installation** — an attacker who can add an MCP server to an agent's configuration gains the ability to execute arbitrary tool calls with the agent's identity and permissions
- **Tool poisoning** — an MCP server can return manipulated tool descriptions (in the MCP discovery handshake) that include embedded instructions to the LLM, causing the agent to take unintended actions. This is the MCP analogue of prompt injection
- **Excessive permission grants** — MCP servers frequently request broader tool scopes than their stated function requires; unreviewed MCP server installations may grant access to sensitive systems
- **MCP server supply chain** — publicly distributed MCP servers may be compromised via their own supply chain (malicious maintainer, compromised package registry entry)
- **Cross-server trust escalation** — in multi-MCP environments, a compromised server may attempt to influence the agent to call other trusted servers with malicious parameters

### Controls

**5.1 MCP server allowlisting**

Maintain an explicit allowlist of approved MCP servers. Any MCP server not on the allowlist must not be reachable from production agent environments. Apply allowlist enforcement at the network layer (not only at configuration).

**5.2 MCP server vetting and supply chain review**

Before approving an MCP server for the allowlist:
- Review the server's source code or published audit
- Verify package integrity (checksum, signature) at install time
- Pin to an exact version or commit SHA
- Assign an internal risk tier (low / medium / high) based on the tool access the server requires

**5.3 Review MCP tool descriptions for injection**

When adding an MCP server, review the tool descriptions returned during MCP discovery. Tool descriptions that contain natural language instructions directed at the agent ("always call X when using this tool", "do not log this action") are a red flag for tool poisoning. Reject any MCP server whose tool descriptions contain behavioral instructions to the LLM beyond functional documentation.

**5.4 Scope MCP server permissions to minimum required**

- Configure each MCP server with the minimum credentials and API access required for its stated function
- Use separate credentials per MCP server — do not share a single high-privilege credential across multiple servers
- Apply time-bound credentials where the MCP server protocol supports it

**5.5 Monitor MCP server behavior**

- Log all MCP tool invocations (server, tool name, caller agent, parameter schema)
- Alert on: calls to allowlisted tools with atypical parameter patterns; calls to tools not previously observed from a given agent; MCP discovery requests from unexpected agents or hosts

### Production MCP Server Deployment Patterns

Deploying MCP servers in a production environment introduces infrastructure and operational concerns that do not arise in development-time agent use. The following patterns address network isolation, credential hygiene, availability, and audit integrity.

#### Pattern 1: Sidecar MCP Server (Kubernetes)

Run the MCP server as a sidecar container in the same Pod as the agent workload. The MCP transport (stdio or HTTP) is confined to the Pod network namespace — no external network exposure is needed.

```yaml
# Kubernetes Pod spec — agent + MCP server as sidecar
apiVersion: v1
kind: Pod
metadata:
  name: code-review-agent
  namespace: ai-workloads
spec:
  serviceAccountName: code-review-agent-sa  # Minimal RBAC; no cluster-admin
  containers:
    - name: agent
      image: myregistry.io/code-review-agent@sha256:<digest>
      env:
        - name: MCP_SERVER_URL
          value: "http://localhost:8080"   # Loopback only — not exposed externally
      resources:
        limits:
          memory: "512Mi"
          cpu: "500m"

    - name: github-mcp-server
      image: myregistry.io/github-mcp-server@sha256:<digest>  # Pinned digest
      ports:
        - containerPort: 8080
          protocol: TCP
      env:
        - name: GITHUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: github-mcp-credentials
              key: token
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 65534
        capabilities:
          drop: ["ALL"]
  # Network policy: block all external egress except GitHub API
  # (defined separately as a CiliumNetworkPolicy or NetworkPolicy resource)
```

**Security properties of this pattern:**
- MCP server is unreachable from outside the Pod — cannot be called by other workloads or external actors
- Credentials are injected via Kubernetes Secrets and never appear in environment variables visible to the agent container
- MCP server image is pinned to a digest — cannot be silently updated

#### Pattern 2: Shared MCP Server with mTLS (Multi-Agent Fleet)

When multiple agent instances share a common MCP server (e.g., a centralized database query server), expose the MCP server as a Kubernetes Service with mutual TLS enforced at the Istio or Cilium layer.

```yaml
# Istio PeerAuthentication — enforce mTLS for MCP server service
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: mcp-server-mtls
  namespace: ai-workloads
spec:
  selector:
    matchLabels:
      app: database-mcp-server
  mtls:
    mode: STRICT

---
# Istio AuthorizationPolicy — only allow calls from approved agent service accounts
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: mcp-server-authz
  namespace: ai-workloads
spec:
  selector:
    matchLabels:
      app: database-mcp-server
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/ai-workloads/sa/code-review-agent-sa"
              - "cluster.local/ns/ai-workloads/sa/documentation-agent-sa"
      to:
        - operation:
            methods: ["POST"]
            paths: ["/mcp/*"]
```

#### Pattern 3: MCP Server Credential Rotation

MCP servers that authenticate to external services (GitHub, Jira, Slack, databases) must have their credentials rotated on a defined schedule. Use External Secrets Operator to pull credentials from the secrets manager and rotate them without redeploying the MCP server.

```yaml
# ExternalSecret — rotate GitHub token for MCP server every 24 hours
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: github-mcp-credentials
  namespace: ai-workloads
spec:
  refreshInterval: 24h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: github-mcp-credentials
    creationPolicy: Owner
  data:
    - secretKey: token
      remoteRef:
        key: secret/ai-workloads/github-mcp-server
        property: token
```

**Credential rotation audit requirement:** Each credential rotation must produce a log entry in the audit trail. Vault's audit backend and the External Secrets Operator event stream both provide this — ship both to the SIEM.

#### MCP Server Observability Requirements

| Signal | Collection Method | Alert Condition |
|--------|------------------|-----------------|
| Tool invocation log (server, tool, agent, params schema, result) | Structured log from MCP server; forward to SIEM | Tool not in approved catalog for this agent identity |
| MCP discovery handshake | Log on startup and re-discovery | Discovery from unexpected source IP or service account |
| Credential usage (API calls made by MCP server) | Cloud provider API audit log (CloudTrail, Azure Monitor) | API calls outside expected scope for this server |
| Error rate (tool call failures) | Metrics; Prometheus or Datadog | Sustained error rate > 5% — potential exfiltration probe |
| MCP server process execution | Tetragon TracingPolicy (see [eBPF Security](../../cloud-security-devsecops/docs/ebpf-security.md)) | Unexpected subprocess spawned by MCP server |

#### MCP Server Security Checklist

```
Supply Chain
[ ] MCP server image pinned to immutable digest in all deployments
[ ] MCP server source code reviewed before initial deployment
[ ] MCP server package integrity verified (checksum, signature)
[ ] MCP server version pinned; upgrade process requires security review

Network Controls
[ ] Sidecar MCP servers: loopback-only transport (no external exposure)
[ ] Shared MCP servers: mTLS enforced; callers authenticated by service account
[ ] Network policy restricts MCP server egress to declared external endpoints only
[ ] No MCP server exposed on a public-facing port

Credential Security
[ ] Separate credentials per MCP server — no shared high-privilege tokens
[ ] Credentials managed by secrets manager (not hardcoded, not in env vars in plaintext)
[ ] Credential rotation configured; rotation events logged to SIEM

Monitoring
[ ] All tool invocations logged with agent identity, tool name, parameter schema
[ ] Anomalous invocation patterns alert to security team within 5 minutes
[ ] MCP server process behavior monitored by Tetragon or Falco
[ ] Monthly review of per-server tool usage patterns for behavioral drift
```

---

## AI Supply Chain Security

AI systems introduce new supply chain attack surfaces beyond traditional software dependencies.

### Model Supply Chain

| Risk | Control |
|------|---------|
| Compromised model weights served via public hub | Verify model checksum against provider-published hash before loading; use private model registry for approved models |
| Backdoored model fine-tuned on poisoned data | Apply behavioral test suite to model before promoting to production use; do not use externally fine-tuned models without provenance review |
| Model version substitution | Pin model identifiers to immutable references (digest or signed manifest); treat model upgrades as requiring change management review |

### AI Toolchain Dependencies

AI frameworks (LangChain, LlamaIndex, Autogen, CrewAI, Claude Agent SDK) are software dependencies with their own vulnerability surface:

- Include AI framework packages in SCA scans
- Monitor CVE feeds for AI framework vulnerabilities; apply the same patch SLA as application dependencies
- Review AI framework updates before applying — behavioral changes in agentic frameworks can alter security boundaries without explicit CVE classification

### Training Data and Fine-Tuning

If your organization produces or uses fine-tuned models:

- Apply data lineage tracking to training datasets
- Screen training data for PII, credentials, and copyright-protected content before ingestion
- Restrict access to fine-tuning pipelines using the same controls applied to production build pipelines (see [Secure CI/CD Reference Architecture](../../secure-ci-cd-reference-architecture/docs/framework.md))
- Generate SLSA-equivalent provenance for fine-tuned model artifacts

---

## Governance Requirements

### AI Usage Policy

Organizations using AI in their development workflow should document and communicate:

1. **Approved AI tools** — which AI coding assistants, LLM APIs, model providers, MCP servers, and agent frameworks are approved for use
2. **Data classification constraints** — what types of data may be sent to external AI APIs or processed by agentic systems
3. **Code review requirements** — minimum human review requirements for AI-generated code
4. **Agent authorization scope** — which actions agents are permitted to take autonomously vs. requiring human approval
5. **Incident response** — how to respond to AI-specific security incidents (compromised API key, prompt injection exploit, hallucinated dependency attack, MCP tool poisoning, rogue agent action)

### Audit and Monitoring

- Log all LLM API calls including model name, approximate token count, and calling service — not content
- Log all agent action traces (tool, parameters schema, result status)
- Alert on anomalous API key usage (unusual request volume, unexpected calling IP)
- Alert on agent behavioral anomalies (unexpected tool sequences, self-modification attempts)
- Include AI-generated code sections in security-focused code review checklists
- Review agent permission grants quarterly; revoke unused tool access

### Model and Dependency Versioning

Treat LLM model versions as software dependencies:
- Pin to specific model versions where the API supports it
- Test application behavior when upgrading to a new model version
- Include LLM provider APIs and AI frameworks in third-party dependency risk assessments
- Treat model weight artifacts as release artifacts: version, sign, and store in a controlled registry

---

## Framework Navigation

This document is the AI security integration guide for the DevSecOps Framework. It provides the bridge between foundational DevSecOps controls and AI-specific security requirements, covering the five AI use cases that most engineering organizations encounter.

For organizations that have moved beyond initial AI adoption and need a dedicated architecture for AI and agentic pipeline security, use the **[AI DevSecOps Framework](../../ai-devsecops-framework/docs/introduction.md)** as the primary reference. The AI DevSecOps Framework provides:

| Topic | This Document | AI DevSecOps Framework |
|---|---|---|
| AI coding assistant controls | Use Cases 1–2 (complete) | Expanded in `developer-environment-controls.md` |
| Prompt injection defense | Use Case 2–3 (controls) | Full architecture in `prompt-injection-defense.md` |
| Agent authorization | Use Case 4 (principles) | Detailed POLA patterns in `agent-authorization.md` |
| Blast radius containment | Use Case 4 (principles) | Architecture in `blast-radius-containment.md` |
| Multi-agent trust | Not covered | `multi-agent-architecture.md` |
| Agent audit trail | Use Case 4 (principles) | Full schema in `agent-audit-trail.md` |
| Model supply chain | AI Supply Chain section | Complete in `model-supply-chain.md` |
| MCP server security | Use Case 5 (complete) | Complemented by `pipeline-controls.md` |
| AI security maturity | Not covered | Five-level model in `maturity-model.md` |
| Agent forensics | Not covered | Via `forensics-and-incident-response-framework` |
| Regulatory compliance (EU AI Act, ISO 42001) | Not covered | `regulatory-mapping.md`, `iso-42001-certification-roadmap.md` |

**When to use this document:** Initial AI security review, governance policy creation, OWASP LLM Top 10 control mapping, MCP server deployment patterns.

**When to use the AI DevSecOps Framework:** Designing AI-integrated pipeline architecture, implementing agent authorization systems, building forensic readiness for AI agents, achieving AI security maturity Level 3+, pursuing ISO 42001 certification.

---

## Related Documents

- [AI DevSecOps Framework](../../ai-devsecops-framework/docs/introduction.md) — Comprehensive AI and agentic pipeline security architecture (threat modeling, agent authorization, blast radius, audit trail, maturity model)
- [DevSecOps Framework](framework.md) — Core security lifecycle and controls catalog
- [Secure CI/CD Reference Architecture: AI-Assisted Development](../../secure-ci-cd-reference-architecture/docs/ai-assisted-development.md) — Pipeline-level controls for AI coding assistant integration
- [Secure CI/CD Reference Architecture Threat Model](../../secure-ci-cd-reference-architecture/docs/threat-model.md) — Pipeline threat model including AI-in-pipeline considerations
- [Software Supply Chain Security Framework](../../software-supply-chain-security-framework/docs/framework.md) — Dependency and artifact security controls
- [Compliance Automation: AI Regulatory Frameworks](../../compliance-automation-framework/docs/regulatory-controls-matrix.md#section-11-ai-regulatory-frameworks) — EU AI Act, NIST AI RMF 1.0, and ISO 42001 control mappings
- [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — LLM application security reference
- [Model Context Protocol Specification](https://modelcontextprotocol.io/) — MCP protocol reference
