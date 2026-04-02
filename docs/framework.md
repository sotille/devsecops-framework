# DevSecOps Framework

## Table of Contents

- [DevSecOps Principles and Philosophy](#devsecops-principles-and-philosophy)
- [DevSecOps Lifecycle](#devsecops-lifecycle)
- [Secure SDLC Model](#secure-sdlc-model)
- [Security Controls per Phase](#security-controls-per-phase)
- [Roles and Responsibilities](#roles-and-responsibilities)
- [Security Toolchain Reference](#security-toolchain-reference)
- [CI/CD Security Integration Points](#cicd-security-integration-points)
- [Infrastructure as Code Security](#infrastructure-as-code-security)
- [Container and Kubernetes Security](#container-and-kubernetes-security)
- [Secrets Management](#secrets-management)
- [Monitoring and Logging](#monitoring-and-logging)
- [Incident Response Integration](#incident-response-integration)

---

## DevSecOps Principles and Philosophy

### Principle 1: Security is a Shared Responsibility

Security cannot be the exclusive domain of a security team. Every person who participates in the software delivery process bears a proportional responsibility for security outcomes. Developers are responsible for writing code that does not introduce vulnerabilities. Operations engineers are responsible for maintaining secure infrastructure configurations. Platform engineers are responsible for building pipelines that enforce security standards. Security engineers are responsible for enabling all of these teams with knowledge, tooling, and feedback.

This does not mean that every engineer must become a security expert. It means that security expertise must be distributed through the organization via security champions, training, documentation, and accessible tooling — and that security outcomes are measured at the team level.

### Principle 2: Security Shifts Left

Security activities must be moved as early as possible in the SDLC. Every phase shift to the right multiplies the cost of remediation. Security defects found during development cost a fraction of those found in production. Automated security checks in CI pipelines provide fast, continuous feedback that eliminates the delay between vulnerability introduction and discovery.

### Principle 3: Automate Security Controls

Manual security processes do not scale to the pace of modern software delivery. Automated security controls — SAST, SCA, secrets scanning, IaC analysis, container scanning, compliance checks — run consistently on every code change, providing continuous security validation without human bottlenecks. Human security expertise is preserved for activities that require judgment: threat modeling, complex architecture review, penetration testing, and incident response.

### Principle 4: Fail Fast, Fix Fast

Security gates should block pipeline progression when significant security thresholds are exceeded. A failed security gate provides immediate, actionable feedback to the developer while the context is fresh. The cost of fixing a finding at the moment of discovery is orders of magnitude lower than the cost of fixing it weeks later, or of deploying it to production.

### Principle 5: Security Enables Velocity

The conventional view that security slows delivery is a product of late-stage security enforcement. When security is integrated continuously throughout the SDLC, it prevents the rework, production incidents, and emergency patches that are the true enemies of delivery velocity. Organizations with mature DevSecOps practices consistently demonstrate higher deployment frequency and lower change failure rates than those with traditional security models.

### Principle 6: Treat Security Configuration as Code

Security policies, compliance rules, admission controls, and network policies must be expressed as code — version-controlled, peer-reviewed, tested, and deployed through automated pipelines. Security configuration drift is one of the most common causes of security incidents. Codified security configuration provides auditability, repeatability, and the ability to detect and correct drift.

### Principle 7: Defense in Depth

No single security control is sufficient. Security architecture must layer multiple independent controls so that a failure in one layer does not result in a compromise. A vulnerability that bypasses SAST should be caught by DAST. A vulnerability deployed to staging should be caught before reaching production. Defense in depth means each control layer is designed with the assumption that other layers may fail.

### Principle 8: Continuous Compliance

Compliance requirements must be continuously validated, not periodically audited. Point-in-time compliance assessments create long windows of non-compliance between assessments. Continuous compliance monitoring — implemented through automated policy checks, CSPM tools, and compliance-as-code frameworks — provides real-time visibility into compliance posture and eliminates audit scrambles.

### Principle 9: Assume Breach

Design security controls with the assumption that adversaries have already gained a foothold somewhere in the system. This assumption drives investments in detection, containment, and response capabilities in addition to prevention. It leads to minimizing blast radius through least-privilege, network segmentation, and workload isolation. It motivates comprehensive logging and monitoring so that breaches are detected quickly.

### Principle 10: Measure and Improve

DevSecOps maturity is measurable. Mean time to detect (MTTD) vulnerabilities, mean time to remediate (MTTR), vulnerability density trends, false positive rates, and security gate pass rates are all quantifiable metrics. Organizations must baseline their current security posture, define improvement targets, and track progress against KPIs. Metrics drive resource allocation, communicate value to leadership, and identify process improvements.

### Principle 11: Secure the Software Supply Chain

The software supply chain — the sum of all tools, dependencies, libraries, and services that contribute to software being built and deployed — is attack surface that must be actively managed. Organizations must maintain software bills of materials, scan all dependencies, enforce artifact integrity, and monitor for supply chain threats as diligently as they monitor for application vulnerabilities.

### Principle 12: Build Security Culture, Not Just Tools

Tooling without culture produces alert fatigue, suppressed findings, and workarounds. The most effective DevSecOps programs invest as much in culture change — security champions programs, developer training, blameless post-mortems, recognition of security contributions — as in tools. Security must be perceived as an enabler of engineering excellence, not a bureaucratic obstacle.

---

## DevSecOps Lifecycle

The DevSecOps lifecycle consists of eight continuous phases, each with defined security activities, controls, and quality criteria.

### Phase 1: Plan

The plan phase encompasses all activities that occur before code is written — feature definition, architectural design, risk assessment, and sprint planning.

**Security activities:**
- **Threat modeling** — Identify potential threats, attack vectors, and mitigations for planned features and architectural changes using STRIDE or PASTA methodology
- **Security requirements definition** — Define security acceptance criteria alongside functional requirements in user stories
- **Abuse case development** — Define scenarios describing how the system could be misused or attacked
- **Third-party risk assessment** — Evaluate security posture of new vendors, SaaS products, or open-source dependencies being introduced
- **Architecture security review** — Security architect reviews significant design decisions before implementation begins

**Outputs:**
- Threat model document for significant features
- Security acceptance criteria in user stories
- Architectural Decision Records (ADRs) with security considerations documented

### Phase 2: Develop

The develop phase covers all activities performed by developers writing code — including IDE usage, local testing, and preparing code for peer review.

**Security activities:**
- **Secure coding practices** — Developers follow language-specific secure coding guidelines (OWASP Secure Coding Practices, CWE Top 25)
- **IDE security plugins** — Real-time vulnerability highlighting surfaces findings as code is written
- **Pre-commit hooks** — Automated checks prevent secrets, vulnerable patterns, and formatting violations from entering the repository
- **Dependency management** — Developers use approved, pinned dependency versions; understand license implications
- **Security self-review** — Developers review their own code for security implications before requesting peer review

**Outputs:**
- Code that passes pre-commit security checks
- Pull request with security-aware code review

### Phase 3: Build

The build phase covers the CI pipeline — automated compilation, testing, and security scanning of code changes.

**Security activities:**
- **SAST** — Static analysis of application source code for vulnerability patterns
- **SCA** — Dependency scanning for known CVEs and license compliance
- **Secrets detection** — Scanning for credentials, API keys, and sensitive data in code and history
- **IaC security scanning** — Analysis of Terraform, CloudFormation, Helm, and Kubernetes manifests for security misconfigurations
- **Container image scanning** — Base image and application layer scanning for OS and package vulnerabilities
- **SBOM generation** — Automated generation of software bill of materials for each build artifact
- **Build provenance recording** — SLSA provenance attestations generated for build artifacts

**Outputs:**
- Signed, scanned build artifacts
- SBOM artifact per build
- Security scan results published to vulnerability management platform
- Pipeline status indicating security gate pass/fail

### Phase 4: Test

The test phase covers integration testing, dynamic security testing, and compliance validation against a running application.

**Security activities:**
- **DAST** — Dynamic testing against a running instance of the application in the staging environment
- **API security testing** — Automated testing of API endpoints for OWASP API Top 10 vulnerabilities
- **Penetration testing** — Manual security testing for complex applications or major releases
- **Security regression testing** — Automated tests for previously identified vulnerabilities confirm they have not recurred
- **Compliance validation** — Automated checks confirm that the application and infrastructure meet relevant compliance requirements
- **Fuzzing** — Automated input fuzzing for security-sensitive code paths (parsers, deserialization, authentication flows)

**Outputs:**
- DAST scan report
- Security regression test results
- Compliance validation report
- Penetration test findings (for applicable releases)

### Phase 5: Release

The release phase covers the promotion of validated artifacts from staging to a production-ready state.

**Security activities:**
- **Artifact signing and verification** — All production artifacts are cryptographically signed; signatures verified before promotion
- **Release approval** — Production releases require explicit approval from authorized release engineers
- **Change advisory board review** — Significant releases reviewed by CAB for risk assessment
- **Security sign-off** — Security team provides sign-off for major releases or those with significant security changes

**Outputs:**
- Signed, immutable production artifacts in production artifact registry
- Release approval record with audit trail
- Security sign-off documentation

### Phase 6: Deploy

The deploy phase covers the actual deployment of software to production environments.

**Security activities:**
- **Admission control** — Kubernetes admission controllers verify image signatures and security policy compliance before deployment
- **Progressive deployment** — Canary or blue/green deployment strategies reduce blast radius
- **Secrets injection** — Dynamic secret injection from Vault or cloud secrets manager at deploy time
- **Infrastructure drift detection** — Automated comparison of deployed state to desired IaC state
- **Deployment approval gate** — Manual approval required for production deployments above a risk threshold

**Outputs:**
- Deployed production artifact with verified provenance
- Deployment audit trail in change management system
- Pre-deployment and post-deployment infrastructure state snapshots

### Phase 7: Operate

The operate phase covers the ongoing management and maintenance of software in production.

**Security activities:**
- **Patch management** — Systematic identification and deployment of security patches for OS, runtime, and application dependencies
- **Vulnerability scanning** — Continuous or scheduled scanning of production workloads
- **Cloud security posture management** — Continuous monitoring of cloud configuration compliance
- **Access review** — Periodic review of user and service account access to production systems
- **Security policy review** — Periodic review of firewall rules, IAM policies, and security group configurations

**Outputs:**
- Patched, current production workloads
- Access review records
- CSPM compliance reports

### Phase 8: Monitor

The monitor phase covers security observability and threat detection in production environments.

**Security activities:**
- **Security event monitoring** — Continuous collection and analysis of security events from all sources
- **Threat detection** — SIEM-based correlation rules and behavioral analytics detect anomalous activity
- **Anomaly detection** — Baseline-based detection of unusual patterns in API usage, authentication, and data access
- **Audit log review** — Regular review of privileged access logs, secrets access logs, and pipeline audit logs
- **Security metrics reporting** — Monthly security KPI reports to engineering and leadership stakeholders

**Outputs:**
- Security dashboards and alerts
- Monthly security metrics report
- Threat intelligence briefings

---

## Secure SDLC Model

The Secure SDLC (Software Development Lifecycle) integrates security checkpoints, reviews, and automated controls at each SDLC stage. It maps the DevSecOps lifecycle to formal SDLC phases and defines the minimum security activities required at each stage.

### SDLC Security Requirements by Phase

| SDLC Phase | Required Security Activity | Evidence Required |
|---|---|---|
| Requirements | Threat model initiated; security requirements documented | Threat model document |
| Design | Architecture security review for significant changes; threat model completed | ADR with security section; threat model |
| Development | IDE plugins active; pre-commit hooks enforced | Pipeline configuration showing pre-commit enforcement |
| Code Review | Security-aware code review; SAST results reviewed | PR review comments addressing security findings |
| Build | SAST, SCA, secrets scan, IaC scan pass; SBOM generated | CI pipeline run results; SBOM artifact |
| Integration Testing | DAST scan completed; security regression tests pass | DAST report; test results |
| Staging | Full security scan suite passed; penetration test (for major releases) | All scan reports; pentest report if applicable |
| Production Release | Artifact signed; release approval obtained; security sign-off | Signed artifact; approval record |
| Post-Deployment | Runtime monitoring active; vulnerability scanning configured | Monitoring dashboard; scan schedule |

---

## Security Controls per Phase

### Pre-Code Controls (Plan / Design)

| Control | Tool / Process | Frequency |
|---|---|---|
| Threat modeling | STRIDE / PASTA / MITRE ATT&CK | Per significant feature or quarterly |
| Security architecture review | Manual review by Security Architect | Per architectural change |
| Third-party risk assessment | Vendor security questionnaire + BlackKite / SecurityScorecard | Per new vendor |
| Security training | OWASP, SANS, platform-specific | Annually + upon role change |

### Developer-Side Controls (Develop)

| Control | Tool | Trigger |
|---|---|---|
| SAST (IDE) | SonarLint, Semgrep, Snyk IDE | On file save / real-time |
| Secret detection | Gitleaks, detect-secrets | Pre-commit |
| Dependency vulnerability check | `npm audit`, `pip-audit`, Dependabot | Pre-commit / PR |
| Code linting with security rules | ESLint security plugin, Bandit, SpotBugs | Pre-commit |
| License check | FOSSA CLI | Pre-commit |

### Build-Time Controls (Build)

| Control | Tool | Blocking Threshold |
|---|---|---|
| SAST | Semgrep, CodeQL, Checkmarx, Fortify | Critical: block; High: warn or block |
| SCA | Snyk, OWASP Dependency-Check, Mend | Critical/High CVE: block |
| Secrets scanning | Gitleaks, Semgrep, GitHub Secret Scanning | Any secret: block |
| IaC scanning | Checkov, tfsec, KICS | High severity: block |
| Container image scanning | Trivy, Grype | Critical: block |
| License compliance | FOSSA, Black Duck | License violation: block |
| SBOM generation | Syft, CycloneDX | Generated per build |

### Test-Time Controls (Test)

| Control | Tool | Blocking Threshold |
|---|---|---|
| DAST | OWASP ZAP, Burp Suite Enterprise | Critical: block production promotion |
| API security testing | OWASP ZAP, Postman Security Tests | High/Critical: block |
| Penetration testing | Manual (for major releases) | Critical findings: block release |
| Security regression tests | Custom test suite | Any regression: block |
| Compliance check | Chef InSpec, OpenSCAP | Critical control failure: block |

### Deployment Controls (Deploy)

| Control | Tool | Enforcement |
|---|---|---|
| Image signature verification | Cosign + Kyverno / OPA Gatekeeper | Block unsigned images |
| Admission control | Kyverno, OPA Gatekeeper | Block policy violations |
| Secrets injection | HashiCorp Vault, AWS Secrets Manager | Runtime injection only |
| Progressive rollout | Argo Rollouts, Flagger | Canary analysis gates |
| Deployment approval | CD pipeline approval gate | Manual approval for production |

---

## Roles and Responsibilities

### Developer

**Core responsibilities:**
- Write code that follows secure coding guidelines and language-specific security patterns
- Maintain awareness of OWASP Top 10 and CWE Top 25 for their primary languages
- Act promptly on security findings from automated tools and code reviewers
- Use approved dependency management practices; keep dependencies current
- Never hard-code secrets, credentials, or sensitive configuration in source code
- Participate in team threat modeling sessions for features they are building

**Security activities:**
- Resolve all Critical SAST/SCA findings blocking their PRs before requesting review
- Write security-focused unit tests for security-sensitive code paths
- Review security implications of their own code during self-review

### Security Champion

**Core responsibilities:**
- Serve as the team's primary resource for security guidance and tool interpretation
- Facilitate threat modeling sessions for significant features
- Triage automated security findings and coordinate remediation with team members
- Maintain awareness of current security vulnerabilities relevant to the team's technology stack
- Participate in cross-functional security champion community meetings

**Security activities:**
- First-pass review of High/Critical findings from automated tools
- Lead team threat modeling for new features
- Represent the team in security-related architectural discussions
- Escalate complex security questions to the Security Engineering team

### Security Engineer / AppSec Engineer

**Core responsibilities:**
- Maintain and configure the security toolchain (SAST, DAST, SCA, secrets detection tools)
- Define security gate policies and thresholds
- Conduct security design reviews for high-risk features and architectural changes
- Perform penetration testing and red team activities
- Develop and deliver security training for development teams
- Manage the vulnerability management program and track remediation SLAs

**Security activities:**
- Review and tune SAST/DAST rulesets to minimize false positives
- Conduct architecture threat modeling for platform-level changes
- Respond to security incidents involving application vulnerabilities
- Produce monthly security metrics reports for engineering leadership

### Operations / Platform Engineer

**Core responsibilities:**
- Maintain secure CI/CD pipeline infrastructure
- Implement and maintain runtime security controls (Falco, CSPM tools)
- Manage secrets management infrastructure (Vault, cloud secrets managers)
- Configure and maintain network security controls for pipeline and production environments
- Implement and maintain Kubernetes security controls (RBAC, PSA, network policies)

**Security activities:**
- Apply security patches to CI/CD infrastructure within SLA
- Monitor and respond to runtime security alerts
- Conduct periodic access reviews for CI/CD platform and infrastructure
- Maintain IaC security scanning configurations

### CISO / Security Leadership

**Core responsibilities:**
- Define the organization's DevSecOps security strategy and policy framework
- Ensure adequate resources are allocated to DevSecOps tooling and training
- Communicate security posture and risk to executive leadership and board
- Manage relationships with external auditors, regulators, and cyber insurance providers
- Oversee incident response program and business continuity planning

**Security activities:**
- Monthly security KPI review against defined targets
- Quarterly security risk assessment and board reporting
- Annual security program review and budget planning
- Executive-level incident response participation for significant incidents

### Platform / Infrastructure Engineer

**Core responsibilities:**
- Design and maintain the CI/CD platform infrastructure
- Implement infrastructure-as-code practices for all platform components
- Ensure platform reliability, scalability, and security
- Maintain runner/agent infrastructure and security hardening
- Implement and maintain observability infrastructure

**Security activities:**
- Implement ephemeral runner infrastructure
- Maintain runner hardening standards
- Audit platform access controls quarterly
- Respond to platform security incidents

---

## Security Toolchain Reference

### Comprehensive Toolchain by Category

| Category | Open Source Options | Commercial Options | Managed Cloud Options |
|---|---|---|---|
| SAST | Semgrep, CodeQL, SpotBugs, Bandit | Checkmarx, Fortify, Veracode | GitHub Advanced Security, GitLab Ultimate |
| DAST | OWASP ZAP, Nuclei, Nikto | Burp Suite Enterprise, Invicti, Rapid7 | AWS Inspector, GitLab DAST |
| SCA | OWASP Dependency-Check, Grype | Snyk, Mend, Black Duck, FOSSA | GitHub Dependabot, GitLab Dependency Scanning |
| Secrets Detection | Gitleaks, TruffleHog, detect-secrets | Nightfall, GitGuardian | GitHub Secret Scanning, GitLab Secret Detection |
| IaC Scanning | Checkov, tfsec, KICS, Terrascan | Prisma Cloud IaC, Wiz IaC | AWS Config Rules, Azure Policy |
| Container Scanning | Trivy, Grype, Clair, Syft | Snyk Container, Anchore Enterprise | AWS ECR Scanning, GCR Vulnerability Scanning |
| Runtime Security | Falco, Tetragon | Aqua Security, Sysdig, Twistlock | AWS GuardDuty, GKE Threat Detection |
| Secrets Management | HashiCorp Vault (OSS) | HashiCorp Vault Enterprise | AWS Secrets Manager, Azure Key Vault, GCP Secret Manager |
| Policy Enforcement | OPA, Kyverno, Conftest | Snyk Policies, Prisma Cloud | AWS Config, Azure Policy |
| SIEM | Elastic SIEM (OSS) | Splunk, IBM QRadar, Microsoft Sentinel | AWS Security Hub, Google Chronicle |
| CSPM | Prowler, ScoutSuite | Wiz, Prisma Cloud, Lacework | AWS Security Hub, GCP SCC |
| Artifact Signing | Cosign, Notary v2 | JFrog Xray | AWS Signer, GitHub Artifact Attestations |
| SBOM | Syft, CycloneDX tools | FOSSA, Snyk | GitHub SBOM, GitLab SBOM |

---

## CI/CD Security Integration Points

### GitHub Actions Integration

```yaml
name: DevSecOps Pipeline

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write  # For SARIF upload to GitHub Security tab
  id-token: write         # For OIDC cloud authentication

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for secret scanning

      # SAST: Semgrep
      - name: Run Semgrep SAST
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/owasp-top-ten
            p/secrets
            p/python
          publishToken: ${{ secrets.SEMGREP_APP_TOKEN }}

      # SCA: Snyk
      - name: Run Snyk SCA
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --fail-on=all

      # Secrets: Gitleaks
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

      # Container scan: Trivy
      - name: Build and scan container image
        run: |
          docker build -t myapp:${{ github.sha }} .
          trivy image \
            --exit-code 1 \
            --severity CRITICAL,HIGH \
            --format sarif \
            --output trivy-results.sarif \
            myapp:${{ github.sha }}

      - name: Upload Trivy results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
```

---

## Infrastructure as Code Security

### IaC Security Scanning with Checkov

Checkov evaluates Terraform, CloudFormation, Helm, Kubernetes manifests, and Dockerfile configurations against hundreds of built-in security policies.

```bash
# Scan Terraform directory
checkov -d ./terraform \
  --framework terraform \
  --check CKV_AWS_* \
  --soft-fail-on MEDIUM \
  --hard-fail-on HIGH,CRITICAL \
  --output sarif \
  --output-file-path checkov-results.sarif

# Scan Kubernetes manifests
checkov -d ./k8s \
  --framework kubernetes \
  --hard-fail-on HIGH,CRITICAL
```

### Terraform Security Best Practices

**Enforce state encryption:**
```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "production/app/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true                # Server-side encryption
    kms_key_id     = "alias/terraform-state-key"  # CMK encryption
    dynamodb_table = "terraform-state-lock"
  }
}
```

**Restrict resource policies:**
```hcl
# Deny public S3 bucket access
resource "aws_s3_bucket_public_access_block" "app_bucket" {
  bucket = aws_s3_bucket.app_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## Container and Kubernetes Security

### Container Image Security

**Secure Dockerfile best practices:**
```dockerfile
# Use minimal, pinned base image
FROM python:3.12.2-slim@sha256:<digest>

# Run as non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Install only required packages; clean apt cache
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Copy application code
WORKDIR /app
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=appuser:appuser . .

# Switch to non-root user before running
USER appuser

# Use ENTRYPOINT for production; avoid running as shell
ENTRYPOINT ["python", "-m", "uvicorn", "app.main:app"]
```

### Kubernetes Security Controls

**Pod Security Standards:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted pod security standard
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.29
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Security Context for all pods:**
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

**Network policies:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # Apply to all pods
  policyTypes:
    - Ingress
    - Egress
  # No ingress or egress rules = deny all; selectively open in separate policies
```

---

## Secrets Management

### Secrets Management Architecture

**HashiCorp Vault with Kubernetes:**
```yaml
# vault-agent-injector: automatically injects secrets into pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp-production"
        vault.hashicorp.com/agent-inject-secret-db-credentials: "secret/data/production/myapp/database"
        vault.hashicorp.com/agent-inject-template-db-credentials: |
          {{- with secret "secret/data/production/myapp/database" -}}
          DB_HOST={{ .Data.data.host }}
          DB_PASSWORD={{ .Data.data.password }}
          {{- end -}}
```

### Secret Rotation Policy

| Secret Type | Rotation Frequency | Rotation Method |
|---|---|---|
| Database passwords | 30 days | Vault dynamic secrets |
| API keys (external services) | 90 days | Automated rotation via Vault |
| TLS certificates | 90 days (or at renewal) | cert-manager automatic rotation |
| Cloud access keys | Eliminated via OIDC | N/A |
| SSH keys | 180 days | Automated via Vault SSH secrets engine |
| Service-to-service tokens | 1 hour | OIDC / short-lived tokens |

---

## Monitoring and Logging

### Security Monitoring Requirements

All production systems must emit the following security-relevant event types to the centralized SIEM:

| Event Category | Required Events | Retention |
|---|---|---|
| Authentication | Login success/failure, MFA events, account lockouts | 365 days |
| Authorization | Privilege use, access denials, RBAC changes | 365 days |
| Secrets access | All vault reads, secrets manager access | 365 days |
| Infrastructure changes | IaC deployments, manual console actions, config changes | 365 days |
| Network | DNS queries, outbound connections (anomaly detection) | 90 days |
| Container runtime | Falco alerts, container exec events, privilege escalation | 365 days |
| Pipeline | Build triggers, artifact promotions, secret access in pipelines | 365 days |

### Security KPIs and Metrics

| Metric | Target | Measurement Source |
|---|---|---|
| Mean time to detect (MTTD) vulnerabilities | < 24 hours for Critical | SIEM / vulnerability platform |
| Mean time to remediate (MTTR) — Critical | < 24 hours | Vulnerability management platform |
| MTTR — High | < 7 days | Vulnerability management platform |
| Vulnerability density (new Critical/High per 1k LOC) | Decreasing trend | SAST/SCA tooling |
| Security gate pass rate | > 95% on first pass | CI/CD pipeline metrics |
| False positive rate | < 15% | Developer-reported suppressions |
| Secret leak incidents | 0 per quarter | Secret scanning alerts |
| % of builds with SBOM | 100% | Artifact metadata |

---

## Incident Response Integration

### Security Incident Severity Classification

| Severity | Description | Response Time | Example |
|---|---|---|---|
| P1 — Critical | Active exploitation; data breach; production system compromised | 15 minutes | Ransomware, active data exfiltration, compromised production credentials |
| P2 — High | Confirmed vulnerability with imminent exploitation risk; service degraded | 1 hour | Critical CVE exploited in wild affecting production; pipeline credential compromise |
| P3 — Medium | Vulnerability requiring remediation; no immediate exploitation | 4 hours | High CVSS vulnerability in production dependency; suspicious pipeline activity |
| P4 — Low | Security misconfiguration; policy violation without immediate risk | 24 hours | Non-critical compliance violation; low-severity misconfiguration |

### Incident Response Playbooks

DevSecOps-specific incident response playbooks must be developed and tested for:

1. **Compromised CI/CD credentials** — Revoke credentials, audit recent pipeline runs for unauthorized access, review artifact integrity
2. **Secrets exposure in source code** — Rotate all exposed secrets immediately, scan git history, review who had access
3. **Malicious dependency injection** — Identify affected builds and deployments, remove malicious package, audit downstream impact
4. **Compromised build artifact** — Revoke deployed artifacts, identify scope of deployment, remediate affected systems
5. **Container runtime anomaly** — Isolate affected pod, capture forensic data, investigate root cause

### Forensic Evidence Preservation

CI/CD systems must be configured to preserve forensic evidence:
- Build logs retained for minimum 365 days (immutable)
- Pipeline artifact provenance records retained for minimum 365 days
- Secret access audit logs retained for minimum 365 days
- Container runtime event logs retained for minimum 90 days
- Network flow logs retained for minimum 90 days
