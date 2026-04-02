# DevSecOps Best Practices

This document catalogs 30+ detailed best practices organized by security domain. Each best practice includes a rationale explaining why it matters and implementation guidance describing how to apply it in practice.

## Table of Contents

- [Pipeline Security](#pipeline-security)
- [Code Security](#code-security)
- [Secrets Management](#secrets-management)
- [Container Security](#container-security)
- [Infrastructure as Code Security](#infrastructure-as-code-security)
- [Monitoring and Incident Response](#monitoring-and-incident-response)
- [Culture and Governance](#culture-and-governance)

---

## Pipeline Security

### BP-P01: Pin All External Dependencies to Immutable References

**Rationale:** Mutable references (tags, branches, "latest") for CI/CD actions, Docker base images, and build tool versions can be changed by maintainers — intentionally or via compromise — causing your pipeline to execute different code than you intended. The SolarWinds and Codecov breaches demonstrated that even trusted tools can become attack vectors.

**Implementation:**
- Pin GitHub Actions to their commit SHA, not their tag: `uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`
- Pin Docker base images to their digest: `FROM python:3.12-slim@sha256:<digest>`
- Pin package manager lock files and commit them to version control
- Use Renovate or Dependabot to automate SHA/digest updates when new versions are released
- Add a comment noting the human-readable version alongside the SHA for maintainability

---

### BP-P02: Use OIDC Federation Instead of Long-Lived Cloud Credentials

**Rationale:** Long-lived cloud credentials (AWS access keys, GCP service account JSON files, Azure service principals) stored as pipeline secrets are a common attack target. They have a wide exposure window, are difficult to rotate across many pipelines, and often accumulate excessive permissions over time. OIDC federation eliminates the need to store cloud credentials entirely.

**Implementation:**
- Configure AWS IAM, GCP Workload Identity, or Azure Workload Identity to trust your CI/CD platform's OIDC provider
- Configure IAM role trust policies to be scoped to specific repositories, branches, and workflow names — not just the entire CI/CD organization
- Verify that temporary credentials have the minimum necessary permissions for each specific job
- Audit all pipelines for remaining long-lived cloud credentials; migrate to OIDC; revoke old credentials

---

### BP-P03: Enforce Ephemeral Build Environments

**Rationale:** Persistent build environments accumulate state between runs. An attacker who compromises a persistent runner can persist malware, modify build tools, steal secrets from subsequent runs, and pivot to other systems. Ephemeral environments that are created fresh and destroyed after each job prevent state accumulation and significantly limit the blast radius of a compromise.

**Implementation:**
- Use cloud-hosted runners when possible (GitHub-hosted, GitLab SaaS runners) — these are ephemeral by default
- For self-hosted runners, use Kubernetes with the GitHub Actions Runner Controller or GitLab Runner Kubernetes executor
- Configure self-hosted runners with `--ephemeral` flag to de-register after one job
- Mount work directories as in-memory filesystems (tmpfs) to prevent disk-based persistence
- Verify runner isolation by auditing whether build artifacts persist between runs

---

### BP-P04: Apply Least Privilege to All Pipeline Tokens and Service Accounts

**Rationale:** Over-permissioned pipeline tokens create an unnecessarily large blast radius when compromised. A token that can push to all repositories, modify organization settings, and deploy to all environments gives an attacker extraordinary leverage if obtained.

**Implementation:**
- Define the minimum permissions required for each CI/CD job and configure tokens accordingly
- For GitHub Actions, declare `permissions:` at the workflow and job level, overriding the default
- Use separate service accounts for each pipeline stage (build, test, deploy-staging, deploy-production)
- Audit all pipeline tokens quarterly; revoke unused permissions
- Use OIDC scoped to specific repos/branches rather than organization-wide tokens

---

### BP-P05: Implement Network Egress Controls for Build Environments

**Rationale:** Build environments have access to sensitive credentials and systems. Compromised builds without network egress controls can exfiltrate secrets via DNS, HTTP, or other protocols to attacker-controlled infrastructure. Egress controls limit the channels available for exfiltration.

**Implementation:**
- Implement network policies (Kubernetes NetworkPolicy or cloud security groups) that restrict build environment outbound traffic to known-good destinations
- Allowlist specific domains required for builds (package registries, artifact stores, dependency proxies)
- Route all package downloads through an internal proxy/mirror rather than directly to public registries
- Monitor DNS queries from build environments for anomalous patterns
- Alert on connections to domains not in the approved allowlist

---

### BP-P06: Protect Pipeline Configuration Files with the Same Rigor as Production Code

**Rationale:** Pipeline configuration files (`.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`) execute arbitrary code in privileged environments. Unauthorized modifications can introduce malicious build steps, exfiltrate secrets, or alter artifacts. Yet many organizations apply weaker review standards to pipeline files than to application code.

**Implementation:**
- Add pipeline configuration files to `CODEOWNERS` with required reviews from the platform security team
- Enable alerts for any modifications to pipeline configuration files
- Review pipeline configuration in the same pull request workflow as application code
- Use a dedicated, restricted service account for pipeline configuration; do not allow developers to modify pipelines that access production
- Consider storing pipeline templates centrally and consuming them in application repositories, so central security review applies to all pipelines

---

### BP-P07: Implement Immutable Artifact Management

**Rationale:** If build artifacts can be overwritten after creation, an attacker who gains write access to the artifact registry can replace legitimate artifacts with malicious ones. Immutable artifact management ensures that once an artifact is published, it cannot be changed.

**Implementation:**
- Configure artifact registries (JFrog Artifactory, Nexus, AWS ECR) with immutable tag policies for production artifacts
- Require artifact signing before promotion to staging and production registries
- Configure admission controllers to verify artifact signatures before allowing deployment
- Implement artifact retention policies that retain production artifacts for the full compliance retention period

---

## Code Security

### BP-C01: Implement Differential SAST Scanning for Pull Requests

**Rationale:** Running full SAST scans and reporting all findings on every pull request — including findings that predate the current change — creates noise, frustration, and alert fatigue. Developers become desensitized to findings that are not their fault. Differential scanning reports only findings introduced in the current change, making feedback actionable.

**Implementation:**
- Configure Semgrep with `SEMGREP_BASELINE_REF` to compare against the base branch
- Configure CodeQL with change-based analysis to report only new findings
- Maintain a "tracked findings" list in the vulnerability management platform for pre-existing issues
- Reserve pre-existing finding review for dedicated security debt sprint work, not PR reviews

---

### BP-C02: Establish and Enforce Secure Coding Standards

**Rationale:** Consistent secure coding standards reduce the frequency of vulnerability classes across the codebase by ensuring developers know what patterns to use and which to avoid. Language-agnostic standards like OWASP Secure Coding Practices provide a foundation; language-specific standards address platform-specific risks.

**Implementation:**
- Publish secure coding standards for all primary languages: input validation, output encoding, authentication, session management, cryptography, error handling
- Map coding standards to SAST rules — every standard should have a corresponding automated check
- Include secure coding standards in developer onboarding documentation
- Reference coding standards in code review checklists
- Update standards when new vulnerability patterns emerge; communicate updates to all developers

---

### BP-C03: Require Security-Focused Code Review

**Rationale:** Automated tools have blind spots. Complex business logic vulnerabilities, authentication bypass through logic errors, and authorization failures often require human review to detect. Code review is the last human-in-the-loop security check before code reaches the pipeline.

**Implementation:**
- Include security-focused review checklist items in pull request templates
- Ensure at least one reviewer per pull request has completed security training
- For security-sensitive code (authentication, authorization, cryptography, payment processing), require review by a Security Champion or Security Engineer
- Use CODEOWNERS to automatically assign Security Champions to review changes to security-sensitive paths
- Train reviewers on common vulnerability patterns and how to identify them in code reviews

---

### BP-C04: Enforce Consistent Input Validation at System Boundaries

**Rationale:** The majority of web application vulnerability classes — injection attacks, path traversal, XXE, deserialization vulnerabilities — result from trusting external input without validation. Input validation at every system boundary is the most fundamental defense against these attack classes.

**Implementation:**
- Define input validation as a non-negotiable architectural standard: validate and sanitize at all entry points (HTTP APIs, message queues, file uploads, database reads of externally sourced data)
- Use allowlist validation (define what is acceptable) rather than denylist validation (try to block known bad inputs)
- Implement centralized validation utilities so that validation logic is consistent and auditable
- Test input validation in security regression tests — fuzzing and boundary value testing
- Add SAST rules to flag common input validation anti-patterns (direct SQL concatenation, unsafe deserialization, etc.)

---

### BP-C05: Use Parameterized Queries and Prepared Statements Exclusively

**Rationale:** SQL injection has been in the OWASP Top 10 for over a decade and remains prevalent because it is easy to introduce and easy to exploit. The technical mitigation is straightforward: parameterized queries prevent user-supplied data from being interpreted as SQL syntax.

**Implementation:**
- Establish a coding standard: direct string concatenation in SQL queries is a critical violation
- Implement SAST rules to detect SQL string concatenation patterns in all supported languages
- Migrate existing raw SQL queries to ORM methods or parameterized queries
- Apply the same principle to LDAP queries (LDAP injection), OS commands (command injection), and XPath queries (XPath injection)

---

### BP-C06: Implement Output Encoding Appropriate to Context

**Rationale:** Cross-Site Scripting (XSS) results from rendering user-controlled data in HTML, JavaScript, CSS, or URL contexts without appropriate encoding. The correct encoding depends on where the data is rendered — HTML entity encoding is inappropriate for a JavaScript context.

**Implementation:**
- Use a rendering framework that performs automatic contextual output encoding (React, Angular, Thymeleaf) rather than manual string interpolation into templates
- For cases where manual encoding is required, use a well-tested encoding library for the target context
- Implement a Content Security Policy (CSP) header as defense-in-depth against XSS
- Include XSS test cases in security regression tests
- Scan with SAST rules targeting common XSS patterns for the frameworks in use

---

## Secrets Management

### BP-S01: Treat All Discovered Secrets as Compromised

**Rationale:** When a secret is discovered in version control, build logs, or any unintended location, the conservative assumption must be that it has already been accessed by unauthorized parties — through repository cloning, log access, public exposure, or other means. Any delay in rotation leaves systems vulnerable.

**Implementation:**
- Establish an incident response playbook specifically for secret exposure events
- When a secret is discovered: rotate it immediately, then investigate the root cause and scope of exposure
- After rotation, scan git history with `git log -p` and gitleaks to identify all occurrences
- Remove the secret from git history using `git filter-repo` (not `git filter-branch`)
- Audit access logs for the exposed secret to determine if unauthorized access occurred
- Document the incident and root cause; use it as a teaching moment

---

### BP-S02: Use Dynamic Secrets with Short TTLs Wherever Possible

**Rationale:** Static, long-lived credentials create a large exposure window. If a long-lived database password is compromised, it remains useful to an attacker until it is manually rotated — which organizations often neglect to do regularly. Dynamic secrets are generated on demand and expire automatically, limiting the window of exploitation.

**Implementation:**
- Use HashiCorp Vault's database secrets engine to generate per-request, short-lived database credentials
- Use Vault's AWS secrets engine to generate short-lived IAM credentials for cloud access
- Use OIDC for CI/CD-to-cloud authentication — tokens are valid for minutes, not months
- Configure TTLs aggressively: database credentials valid for 1 hour, cloud credentials for 15 minutes
- Ensure applications and pipelines handle credential renewal gracefully (lease renewal or re-authentication before TTL expiry)

---

### BP-S03: Implement Centralized Secrets Management with Audit Logging

**Rationale:** Secrets scattered across environment variables, configuration files, parameter stores, and developer laptops create an unmanageable attack surface. Centralized secrets management provides a single authoritative store with access control, audit logging, and rotation automation.

**Implementation:**
- Deploy HashiCorp Vault (or cloud-native equivalent: AWS Secrets Manager, Azure Key Vault, GCP Secret Manager)
- Migrate all secrets from environment variables, config files, and CI/CD platform secrets stores to the centralized manager
- Implement namespace-based access control: each application or team can only access its own secrets
- Enable audit logging on all secret access; forward logs to SIEM
- Implement automated secret rotation for all secrets that support it
- Review access policies quarterly; revoke access for team members who have changed roles

---

### BP-S04: Scope Secrets to the Minimum Necessary Consumer

**Rationale:** Secrets with broad scope create risk if compromised. A database credential that works against all databases, or a cloud credential that has access to all cloud resources, gives an attacker excessive leverage if stolen.

**Implementation:**
- Create separate credentials for each application, service, and environment
- Use resource-scoped database users with only the permissions needed (SELECT, INSERT, UPDATE — never blanket GRANT ALL)
- Scope cloud credentials to specific resources (specific S3 buckets, specific Lambda functions) rather than service-wide
- Never reuse the same secret across environments (production database credentials must not work in staging)

---

## Container Security

### BP-CO01: Use Minimal Base Images

**Rationale:** Every package installed in a container image is potential attack surface. OS-level vulnerabilities, package manager binaries, shells, and debug utilities all provide capabilities that attackers can exploit if they achieve code execution. Minimal images reduce the attack surface to only what the application needs.

**Implementation:**
- Use distroless images (`gcr.io/distroless/*`) for applications that don't need a package manager or shell
- Use Alpine-based images when a lightweight package manager is needed
- Avoid `ubuntu:latest` or `debian:latest` as base images — use their `-slim` or `-minimal` variants
- Audit your current base images with `trivy image --severity HIGH,CRITICAL <image>` to understand the vulnerability baseline
- Justify every package installed in the Dockerfile; remove packages that were added for debugging

---

### BP-CO02: Never Run Containers as Root

**Rationale:** A container process running as root that achieves a container escape has host-level privileges on the underlying node. Non-root processes have a dramatically reduced blast radius in the event of a container escape or process compromise.

**Implementation:**
- Add `USER nonroot` or `USER 1000` to all Dockerfiles before the `CMD`/`ENTRYPOINT` instruction
- Set `securityContext.runAsNonRoot: true` and `securityContext.runAsUser: <uid>` in Kubernetes pod specs
- Test that your application functions correctly as non-root before mandating this in production
- Enable Kubernetes Pod Security Standards with `restricted` profile to enforce this at the namespace level
- Fix build failures caused by applications that incorrectly require root by identifying the specific privilege needed and granting it minimally (e.g., CAP_NET_BIND_SERVICE for port 80, rather than running as root)

---

### BP-CO03: Implement Kubernetes Network Policies

**Rationale:** By default, Kubernetes pods can communicate freely with every other pod in the cluster. If an attacker achieves code execution in any pod, they can pivot to access databases, caches, internal APIs, and other sensitive services. Network policies implement zero-trust micro-segmentation for container workloads.

**Implementation:**
- Start with a default-deny-all NetworkPolicy in each production namespace
- Add explicit allow rules for only the required pod-to-pod and pod-to-external communication
- Use namespace selectors to control cross-namespace traffic
- Verify network policies with a tool like `netassert` or Kubernetes network policy visualizer
- Require network policy definitions for all new services as part of the deployment review process

---

### BP-CO04: Enable Container Runtime Security Monitoring

**Rationale:** Container security policies prevent known-bad configurations, but they cannot prevent all runtime attacks. A compromised container may behave maliciously in ways that are architecturally permitted but operationally anomalous — unusual outbound connections, unexpected process spawning, file system writes to unexpected paths. Runtime security tools detect these anomalies.

**Implementation:**
- Deploy Falco with organization-specific detection rules for the most sensitive workloads
- Configure alerts for: unexpected process execution in containers, unexpected outbound network connections, modifications to sensitive files, privilege escalation attempts
- Feed Falco alerts into SIEM for correlation with other security events
- Review and tune Falco rules monthly to reduce false positive rate

---

### BP-CO05: Enforce Admission Control for All Cluster Deployments

**Rationale:** Admission controllers are the last line of defense before a workload is admitted to the Kubernetes cluster. They can enforce security policies regardless of how the deployment was triggered — whether through a CD pipeline, a developer `kubectl apply`, or an automated tool.

**Implementation:**
- Deploy Kyverno or OPA Gatekeeper with policies that enforce:
  - Image signature verification (signed artifacts only)
  - No Critical CVEs in admitted images
  - Non-root user requirement
  - Read-only root filesystem
  - Required security context fields
  - No privileged containers
- Configure policies in `Enforce` mode for production, `Audit` mode for development
- Review admission controller policy violations weekly; address root causes

---

## Infrastructure as Code Security

### BP-I01: Scan IaC Before Every Deployment

**Rationale:** Infrastructure misconfigurations are among the leading causes of cloud security incidents. Public S3 buckets, overly permissive security groups, unencrypted storage volumes, and disabled logging are all configuration-time mistakes that automated scanning can catch before they reach production.

**Implementation:**
- Integrate Checkov, tfsec, or KICS into CI pipelines for all Terraform, CloudFormation, Helm, and Kubernetes manifests
- Block merges on High/Critical IaC findings
- Run IaC scans in pre-commit hooks for immediate developer feedback
- Configure scanning to use the organization's compliance frameworks (CIS benchmarks, PCI-DSS, HIPAA as applicable)
- Review IaC scan results in pull requests alongside code review

---

### BP-I02: Store Terraform State Securely

**Rationale:** Terraform state files contain sensitive data including resource IDs, passwords, private keys, and other secrets that were passed as inputs to resources. If state files are stored insecurely (e.g., in a local directory or unencrypted S3 bucket), attackers who access them gain knowledge of the entire infrastructure and potentially credentials.

**Implementation:**
- Store state in an encrypted, access-controlled remote backend (S3 with SSE-KMS, Terraform Cloud, GCS with CMEK)
- Enable state locking (DynamoDB for S3 backend) to prevent concurrent modifications
- Restrict access to state buckets/storage to CI/CD pipelines and authorized infrastructure engineers only
- Enable versioning on state storage to allow rollback
- Audit access to state storage quarterly

---

### BP-I03: Implement IaC Drift Detection

**Rationale:** Manual changes made directly to cloud resources outside of the IaC workflow create drift — the actual infrastructure state diverges from the declared state. Drifted resources bypass security controls defined in IaC, may introduce misconfigurations, and create compliance gaps.

**Implementation:**
- Run `terraform plan` on a schedule (daily or on every CI pipeline run) to detect drift
- Alert on any infrastructure drift that is not reflected in pending IaC changes
- Enforce an organizational policy against manual console changes in production
- Use AWS Config, Azure Policy, or GCP Organizational Policies to detect specific security misconfigurations
- Investigate all drift events; either remediate in IaC or accept with documented justification

---

## Monitoring and Incident Response

### BP-M01: Centralize All Security Event Logging

**Rationale:** Distributed logs across application servers, cloud accounts, CI/CD systems, and container platforms make correlation and investigation impossible. Security incidents rarely generate a single obvious log entry — they require correlating events across multiple sources to reconstruct the attack chain.

**Implementation:**
- Define a SIEM architecture and data sources list covering: application logs, cloud audit logs, VPC flow logs, Kubernetes audit logs, CI/CD pipeline audit logs, secrets access logs, identity provider logs
- Forward all security-relevant logs to the SIEM with consistent timestamps (UTC), structured formats (JSON), and source identifiers
- Define log retention policies aligned with compliance requirements: 365 days minimum for security events
- Implement log integrity protection to detect tampering (AWS CloudTrail log file validation, immutable S3 storage)
- Test log coverage quarterly by simulating events and verifying they appear in the SIEM

---

### BP-M02: Define and Test Incident Response Playbooks

**Rationale:** Incident response plans that exist only on paper are ineffective under the time pressure of an active incident. Teams that practice their response procedures — through tabletop exercises and simulated incidents — respond significantly faster and more effectively to real events.

**Implementation:**
- Develop written incident response playbooks for the most likely CI/CD security scenarios: compromised credentials, secrets exposure, malicious dependency, compromised build artifact, container runtime anomaly
- Include in each playbook: detection criteria, initial triage steps, containment actions, evidence collection procedures, communication templates, and remediation steps
- Conduct quarterly tabletop exercises using realistic scenarios
- Run annual simulated incidents (red team exercises or purple team exercises)
- Update playbooks after every real incident based on lessons learned

---

### BP-M03: Implement Anomaly Detection for Pipeline Activity

**Rationale:** Known-bad patterns can be blocked by security gates. Novel attacks — new techniques, insider threats, and zero-day exploits — may bypass signature-based controls but often produce anomalous behavioral patterns. Anomaly detection can identify these patterns even without a known signature.

**Implementation:**
- Establish behavioral baselines for CI/CD pipeline activity: normal build durations, typical artifact sizes, standard outbound network connections, usual deployment patterns
- Alert on deviations: builds that take significantly longer than baseline (possible data exfiltration), unusual outbound connections from build environments, deploys that occur outside of normal working hours without change ticket, artifacts that are significantly larger than historical baseline
- Correlate pipeline anomalies with other security events in the SIEM
- Review and tune anomaly detection rules monthly to reduce false positive rate

---

## Culture and Governance

### BP-G01: Establish and Maintain a Security Champions Program

**Rationale:** A small centralized security team cannot scale to meet the security needs of a large engineering organization operating at DevOps velocity. Security Champions — developers with additional security training who serve as security advocates within their teams — extend the security team's reach without requiring every developer to become a security specialist.

**Implementation:**
- Identify at least one Security Champion per development team (ideally one per 6–8 developers)
- Provide champions with dedicated security training covering threat modeling, secure coding, and OWASP Top 10
- Establish a regular Security Champion community meeting (monthly recommended) for knowledge sharing
- Give champions clear responsibilities: threat modeling facilitation, first-pass security finding triage, security question escalation
- Recognize and reward champion contributions to create positive reinforcement

---

### BP-G02: Measure Security Posture with Meaningful KPIs

**Rationale:** Organizations that measure security posture with meaningful metrics can demonstrate improvement over time, allocate resources to highest-impact areas, and communicate value to executive leadership. Without measurement, DevSecOps investment is based on intuition rather than evidence.

**Implementation:**
- Define a core security KPI set: MTTD and MTTR by severity, vulnerability density trend, security gate pass rate on first attempt, false positive rate, training completion rate, patch compliance rate
- Automate metric collection from security tooling; publish monthly dashboard to engineering and leadership
- Set quarterly improvement targets for each KPI
- Review KPIs in the DevSecOps steering committee monthly
- Adjust targets as maturity improves; don't allow targets to become complacent ceilings

---

### BP-G03: Run Blameless Post-Mortems for Security Incidents

**Rationale:** Organizations that blame individuals for security incidents create an environment where problems are hidden rather than reported and addressed. Blameless post-mortems focus on systemic failures — process gaps, tool inadequacies, training needs — that, when addressed, improve the security posture for everyone.

**Implementation:**
- Establish a blameless post-mortem process that explicitly separates human error from systemic failures
- For every P1/P2 security incident, conduct a post-mortem within 5 business days
- Post-mortem output should include: incident timeline, contributing factors, action items with owners and due dates
- Track action item completion and report on it in the monthly security steering committee
- Share post-mortem learnings (appropriately sanitized) across engineering teams
- Build action items from post-mortems into product and platform backlogs as first-class engineering work

---

### BP-G04: Integrate Security Metrics into Engineering OKRs

**Rationale:** Security improvements compete with feature development for engineering time. Unless security objectives are formally part of team and organizational goals, security work is perpetually deprioritized in favor of features. Making security a formal engineering objective ensures it receives the resources it requires.

**Implementation:**
- Work with engineering leadership to define security OKRs at the organizational, team, and individual contributor levels
- Example engineering OKRs with security components:
  - "Reduce Mean Time to Remediate High vulnerability SLA violations from 14 days to 7 days this quarter"
  - "Achieve 100% SBOM coverage for all production artifacts by end of quarter"
  - "Complete Security Champion training for all development teams by end of half"
- Include security KPI review in quarterly engineering all-hands or leadership reviews
- Acknowledge and celebrate security achievements alongside feature delivery milestones

---

### BP-G05: Conduct Regular Security Toolchain Reviews

**Rationale:** The DevSecOps toolchain is itself a technology product that requires maintenance. Security tool vendors release updates with improved detection capabilities, reduced false positive rates, and new vulnerability coverage. Tools that are not updated fall behind the vulnerability landscape and may miss emerging threat patterns.

**Implementation:**
- Establish a quarterly security toolchain review cadence
- Review: tool version currency, vulnerability database update frequency, false positive rate trends, new features in newer versions, alternative tools that may provide better coverage
- Budget for toolchain updates as part of platform engineering maintenance work
- Subscribe to security tool vendor security advisories and update tools when they address vulnerabilities in the tools themselves
- Evaluate new tool categories annually (new SAST capabilities, emerging threat areas, new compliance frameworks)
