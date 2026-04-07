# Platform Engineering and Internal Developer Platform Security

Platform engineering organizations build Internal Developer Platforms (IDPs) — opinionated, paved paths that abstract infrastructure complexity and provide development teams with self-service access to compute, deployment pipelines, secrets management, service identity, and observability. From a security perspective, a well-designed IDP is the highest-leverage security investment available to an engineering organization: it embeds security controls into the default developer workflow, making secure patterns the path of least resistance.

This document covers the security architecture of IDPs, controls for the platform itself, and patterns for delivering security capabilities to development teams through the platform.

---

## Why Platform Security Matters

A platform that serves 50 development teams is a shared attack surface. A vulnerability in the platform — a misconfigured permissions model, a compromised pipeline template, an insecure secrets delivery mechanism — propagates to every team and every workload that runs on it. Platform security failures have uniquely high blast radius.

The flip side: a platform that correctly implements security controls distributes those controls to every team automatically. OIDC-based cloud credential federation, automatic artifact signing, network policy enforcement, and secrets management all become transparent to development teams when delivered through the platform. Security that requires each team to implement independently has variable quality; security delivered by the platform is uniform.

---

## Platform Security Architecture

The platform consists of four layers, each with distinct security requirements:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Layer 4: Developer-Facing Interfaces                                │
│  (Self-service portal, CLI, scaffolding CLI, service catalog)        │
│  Security: AuthN/AuthZ for portal; input validation; audit logging   │
├──────────────────────────────────────────────────────────────────────┤
│  Layer 3: Golden Path Pipelines                                      │
│  (Pre-built, security-hardened CI/CD templates and workflows)        │
│  Security: SAST, SCA, signing, SBOM, policy gates — all built-in    │
├──────────────────────────────────────────────────────────────────────┤
│  Layer 2: Platform Services                                          │
│  (Secrets management, service identity, artifact registry,           │
│   container runtime, GitOps engine, observability)                   │
│  Security: Hardened configuration; access isolation per team         │
├──────────────────────────────────────────────────────────────────────┤
│  Layer 1: Infrastructure                                             │
│  (Cloud accounts, Kubernetes clusters, networking)                   │
│  Security: IaC-managed; CIS benchmark; least privilege; audit logs   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Layer 1: Infrastructure Security

Platform infrastructure follows the same security baseline as all cloud infrastructure, per the [Cloud Security DevSecOps Framework](../../../cloud-security-devsecops/docs/framework.md). The platform-specific requirements are:

**Separation of platform and workload infrastructure:**

The platform control plane (CI/CD system, GitOps engine, artifact registry, secrets management, platform API) must run in a separate cluster or account from development team workloads. Compromise of a development team's workload must not provide lateral access to the platform control plane.

```
Platform Account / Cluster                    Workload Clusters
┌─────────────────────────────┐              ┌────────────────────────┐
│  ArgoCD / Flux control plane │              │  Cluster A (staging)   │
│  Artifact registry (Harbor)  │──deploy──► │  Team namespaces       │
│  Vault                       │              └────────────────────────┘
│  Tekton / CI system          │              ┌────────────────────────┐
│  Platform API / portal       │──deploy──► │  Cluster B (production)│
└─────────────────────────────┘              └────────────────────────┘
```

**Platform infrastructure must be managed as code:**

Every component of the platform — cluster configuration, RBAC, network policies, admission control policies, Vault configuration — must be declared in the GitOps repository and reconciled automatically. Manual console access to the platform control plane is prohibited except in documented break-glass scenarios.

---

## Layer 2: Platform Services Security

### Secrets Management

The platform provides a centralized secrets management service (HashiCorp Vault or cloud KMS equivalent) to all development teams. Platform security requirements:

- **Team namespace isolation**: Each team's secrets are stored in a Vault namespace or mount path accessible only to that team's service accounts
- **Dynamic secrets by default**: Wherever possible, platform provides dynamic credentials (database passwords, cloud credentials via OIDC) rather than static secrets
- **Secret zero protection**: The mechanism by which workloads authenticate to Vault (Kubernetes auth, AWS IAM auth) must not itself be a static secret
- **Audit logging**: All secret access events are logged and retained for the compliance period

```hcl
# Vault policy — team-a isolated namespace (Vault HCL)
path "secret/data/team-a/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "secret/metadata/team-a/*" {
  capabilities = ["list"]
}

# Explicit deny for cross-team access
path "secret/data/team-b/*" {
  capabilities = ["deny"]
}

# Dynamic database credentials — team-a's databases only
path "database/creds/team-a-*" {
  capabilities = ["read"]
}
```

### Service Identity

The platform provides each workload with a cryptographically verifiable service identity (SPIFFE/SPIRE or Kubernetes service account OIDC projection). This identity is the basis for:

- Cloud credential federation (OIDC to AWS/Azure/GCP IAM)
- Vault authentication
- mTLS between services (via Cilium or Istio)
- Audit log attribution

Development teams do not manage service credentials — the platform issues and rotates them automatically.

### Artifact Registry

The platform's artifact registry enforces:
- **Team namespace isolation**: Teams can only push to their own registry namespace
- **Signature verification at pull**: Images without a valid Cosign signature are rejected at the registry level
- **Vulnerability scan gate**: Images with CRITICAL CVEs are quarantined and cannot be promoted to the production registry
- **Immutable tags**: The production registry enforces digest-only references; mutable tags are blocked from the production repository

```yaml
# Harbor project policy — production registry
apiVersion: goharbor.io/v1beta1
kind: HarborProject
metadata:
  name: production
spec:
  projectName: production
  public: false
  autoScan: true
  scannerName: trivy
  preventVulnerable: true      # Block images with CRITICAL CVEs
  vulnerability:
    preventionSeverity: Critical
  cosignVerification:
    enabled: true
    rekorURL: "https://rekor.sigstore.dev"
```

---

## Layer 3: Golden Path Pipeline Security

The platform's golden path pipeline is a pre-approved, security-hardened CI/CD template that teams use instead of building pipelines from scratch. Security is built into the template — teams get it by default.

### What the Golden Path Pipeline Provides

| Stage | Security Control | Team Responsibility |
|---|---|---|
| Source checkout | Pinned action SHA, branch protection | None — platform-provided |
| Dependency install | Private registry mirror, lockfile enforcement | Maintain lockfile |
| SAST | Semgrep with security ruleset | Review findings; fix CRITICAL/HIGH |
| SCA | Trivy SCA, dependency confusion detection | Review findings; fix CRITICAL/HIGH |
| Secrets scanning | Gitleaks, pre-commit hooks | Ensure no secrets committed |
| Build | Ephemeral, isolated, network-restricted runner | Maintain build scripts |
| Container scan | Trivy image scan | Review findings |
| SBOM generation | Syft CycloneDX attestation | None — platform-provided |
| Artifact signing | Cosign keyless OIDC | None — platform-provided |
| Deployment (staging) | OIDC-based cloud credentials | Configure staging target |
| DAST | OWASP ZAP baseline scan | Review findings |
| Deployment (production) | Manual approval gate + OIDC credentials | Approve deployment |

The platform owns the security gates. Teams configure the business logic (application-specific SAST rules, test scripts, deployment targets), but cannot disable or weaken the platform security controls without a formal security exception.

**Enforcing the golden path:** Platform-provided pipelines are identified by a cryptographic signature on the pipeline definition file. Kyverno admission control verifies that any workload deployed to the cluster was built by a platform-signed pipeline — preventing teams from bypassing the golden path with a custom unsecured pipeline.

```yaml
# Kyverno policy — require platform pipeline build attestation
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-platform-pipeline-attestation
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-platform-attestation
    match:
      any:
      - resources:
          kinds: [Pod]
          namespaces: ["*"]
    verifyImages:
    - imageReferences: ["harbor.platform.internal/*"]
      attestations:
      - predicateType: "https://techstream.internal/provenance/v1"
        attestors:
        - entries:
          - keyless:
              subject: "https://github.com/platform-team/golden-path-pipeline/.github/workflows/*.yml@refs/heads/main"
              issuer: "https://token.actions.githubusercontent.com"
```

---

## Layer 4: Developer-Facing Interface Security

### Self-Service Portal and CLI

The platform portal (Backstage, Port, Cortex, or custom) and CLI are authentication-required systems. Security requirements:

- **Authentication**: Platform portal authenticates via SSO (OIDC/SAML); no local username/password accounts
- **Authorization**: Role-based access — developers can view and operate their team's services; they cannot view other teams' secrets, logs, or configuration
- **Input validation**: All scaffolding inputs (service name, team name, repository URL) are validated before any resource is created
- **Audit logging**: All portal actions (service creation, secret access, deployment approval) are logged with user identity and timestamp

### Scaffolding and Service Templates

Scaffolding tools (cookiecutter, Backstage software templates, Crossplane compositions) create new services with security defaults pre-configured:

```yaml
# Backstage software template — new microservice scaffold
# Generates: repository, CI/CD pipeline, service account, Vault namespace, Kubernetes namespace
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: secure-microservice
spec:
  parameters:
    - title: Service Configuration
      properties:
        name:
          type: string
          pattern: '^[a-z][a-z0-9-]{2,30}$'
        team:
          type: string
          enum: [payments, auth, platform, data]
        dataClassification:
          type: string
          enum: [public, internal, confidential, restricted]

  steps:
    - id: create-repo
      action: publish:github
      input:
        # Repository created with branch protection, CODEOWNERS, security scanning workflow

    - id: create-vault-namespace
      action: vault:create-namespace
      input:
        # Team-scoped Vault namespace with team-specific policies

    - id: create-k8s-namespace
      action: kubernetes:create-namespace
      input:
        # Namespace with PSS restricted profile, default-deny NetworkPolicy,
        # ResourceQuota, LimitRange — all pre-configured

    - id: configure-oidc-identity
      action: cloud:configure-oidc
      input:
        # OIDC trust between GitHub Actions repo and cloud IAM role
        # Scoped to this specific repository and branch
```

Every service created through the platform scaffold gets: branch protection, security scanning in CI, a Vault namespace for secrets, OIDC-based cloud credentials, a Kubernetes namespace with Pod Security Standards enforced, and a default-deny NetworkPolicy. Teams inherit security by default; they opt into weaker settings only through a documented exception process.

---

## Platform Security Anti-Patterns

These patterns are common in poorly designed platforms and should be explicitly avoided:

| Anti-Pattern | Risk | Correct Pattern |
|---|---|---|
| Shared service account for all CI/CD jobs | One compromise = blast radius across all teams | Per-repository OIDC-scoped identities |
| Admin-level cloud credentials in platform | Platform compromise = full cloud access | Least-privilege OIDC with job-scoped permissions |
| Secrets accessible across team namespaces | Cross-team secret leakage | Vault namespace isolation per team |
| Platform pipeline modifiable by teams | Teams bypass security gates | Pipeline as code owned by platform team; CODEOWNERS enforced |
| Shared runner pool without isolation | Build cache poisoning; secret leakage | Ephemeral, isolated runners per job |
| Unscanned base images in platform registry | Platform distributes vulnerable images | Base images continuously scanned; EOL images blocked |
| GitOps repository writable by all teams | Cross-team deployment manipulation | Namespace-scoped CODEOWNERS; separate repos per team |

---

## Platform Security Metrics

Track the health of the platform's security delivery:

| Metric | Definition | Target |
|---|---|---|
| Golden path adoption rate | % of development teams using the platform-provided pipeline | 100% |
| Golden path bypass rate | Count of admitted workloads without a valid platform attestation | 0 |
| Secrets manager adoption | % of team workloads retrieving secrets from Vault vs. static environment variables | ≥ 90% |
| OIDC adoption | % of CI/CD jobs using OIDC credentials vs. long-lived keys | 100% |
| Platform vulnerability SLA | % of platform components (base images, dependencies) within CVE remediation SLA | 100% Critical/High |
| Developer satisfaction score | Self-reported friction score for platform security features (1–5) | ≥ 4.0 |

---

## Integration with Techstream Frameworks

The platform engineering security model integrates across all Techstream frameworks:

| Framework | Integration Point |
|---|---|
| [Secure CI/CD Reference Architecture](../../secure-ci-cd-reference-architecture/docs/architecture.md) | Golden path pipeline architecture and security controls |
| [Secure Pipeline Templates](../../secure-pipeline-templates/templates/github-actions-secure-pipeline.yml) | Deployable pipeline template embedded in the golden path |
| [Cloud Security DevSecOps](../../cloud-security-devsecops/docs/architecture.md) | Platform infrastructure security and Kubernetes hardening |
| [Software Supply Chain Security](../../software-supply-chain-security-framework/docs/framework.md) | SBOM and signing built into the golden path |
| [Release Orchestration Framework](../../release-orchestration-framework/docs/gitops-architecture.md) | GitOps engine as the platform's deployment mechanism |
| [Compliance Automation Framework](../../compliance-automation-framework/docs/framework.md) | Platform as evidence generator; all pipelines produce compliance evidence |
| [DevSecOps Maturity Model](../../devsecops-maturity-model/docs/framework.md) | Platform adoption rate and security feature coverage as maturity indicators |

---

*Part of the Techstream DevSecOps Framework. Licensed under Apache 2.0.*
