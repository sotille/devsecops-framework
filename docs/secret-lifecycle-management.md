# Secret Lifecycle Management and Rotation Automation

Secrets — API keys, database credentials, TLS certificates, SSH keys, signing keys, and service account tokens — accumulate over time and become security liabilities if not actively managed. Static, long-lived secrets are one of the most exploitable attack vectors in cloud-native environments: they are over-permissioned by convention, rarely rotated in practice, and frequently leaked through developer tooling, version control, and log aggregation systems.

This guide covers the complete lifecycle of secrets in a DevSecOps environment: provisioning, distribution, rotation, revocation, and audit — with a focus on automation patterns that eliminate the manual rotation burden that causes most organizations to skip rotation entirely.

---

## Table of Contents

- [Secret Classification](#secret-classification)
- [Secret Provisioning Patterns](#secret-provisioning-patterns)
- [Secret Distribution to Applications](#secret-distribution-to-applications)
- [Automated Rotation Architecture](#automated-rotation-architecture)
- [Rotation Patterns by Secret Type](#rotation-patterns-by-secret-type)
- [Secret Sprawl Detection and Remediation](#secret-sprawl-detection-and-remediation)
- [Emergency Revocation](#emergency-revocation)
- [Secrets in CI/CD Pipelines](#secrets-in-cicd-pipelines)
- [Audit and Compliance](#audit-and-compliance)
- [Measurement](#measurement)

---

## Secret Classification

Before implementing lifecycle management, classify secrets by their rotation complexity, blast radius, and decay risk.

### Classification Tiers

| Tier | Secret Types | Rotation Complexity | Max Lifetime | Automation Priority |
|---|---|---|---|---|
| **Tier 1: Service credentials** | Database passwords, API keys, service account passwords | Medium | 90 days | Highest |
| **Tier 2: Long-lived tokens** | Long-lived API tokens (GitHub PATs, cloud API keys) | Low | 180 days | High |
| **Tier 3: Certificates** | TLS certificates, mTLS certs, signing certificates | Low (ACME/cert-manager) | 90 days (public) / 1 year (internal) | High |
| **Tier 4: Workload identity** | OIDC tokens, SPIFFE SVIDs, cloud instance profiles | Automatic | Minutes to hours | Inherent |
| **Tier 5: Human credentials** | SSH keys (personal), GPG signing keys | High (coordination) | 1 year | Medium |
| **Tier 6: HSM-protected keys** | Root CA keys, code signing keys, KMS CMKs | Low (hardware-managed) | 3–5 years (per standard) | Low (vendor-managed) |

### Goal State: Eliminate Tier 1 and 2 Entirely

The goal of a mature secrets management program is to replace all Tier 1 and Tier 2 static secrets with Tier 4 workload identity:

```
Current state (common):
  Application → static DB password (set once, never rotated)
  Service → static API key (360 days old, unknown owner)

Goal state:
  Application → Vault dynamic credential (60-minute TTL, auto-renewed)
  Service → OIDC workload identity → cloud API (no stored secret)
```

This is achievable for most modern cloud-native workloads. Legacy applications requiring static credentials should have a documented remediation timeline.

---

## Secret Provisioning Patterns

### Anti-Pattern: Manual Secret Creation

Manual secret creation — a developer generates a credential, pastes it into a secret manager, and documents it in a ticket — is the root cause of most secret lifecycle failures:

- **Owner ambiguity**: who is responsible for rotating this secret?
- **Undocumented dependencies**: which services depend on this credential?
- **Rotation difficulty**: can this credential be rotated without downtime?

### Pattern 1: Infrastructure as Code Secret Provisioning

All secrets should be provisioned through IaC, never manually. IaC provisioning creates an auditable record of what secrets exist, who created them, and what they're used for.

```hcl
# Terraform: provision a database password through AWS Secrets Manager
resource "aws_secretsmanager_secret" "database_password" {
  name        = "/${var.environment}/payment-service/db-password"
  description = "PostgreSQL password for payment-service RDS instance"

  tags = {
    "techstream:service"        = "payment-service"
    "techstream:secret-tier"    = "1"
    "techstream:rotation-days"  = "90"
    "techstream:owner"          = "platform-team"
    "techstream:environment"    = var.environment
  }
}

resource "aws_secretsmanager_secret_rotation" "database_password_rotation" {
  secret_id           = aws_secretsmanager_secret.database_password.id
  rotation_lambda_arn = aws_lambda_function.db_rotation.arn

  rotation_rules {
    automatically_after_days = 90
  }
}
```

### Pattern 2: Dynamic Secret Generation

For credentials where the underlying system supports it, generate short-lived credentials on-demand rather than managing long-lived credentials at all.

**HashiCorp Vault database secrets engine:**

```bash
# Enable the database secrets engine
vault secrets enable database

# Configure a PostgreSQL connection
vault write database/config/payment-db \
  plugin_name=postgresql-database-plugin \
  allowed_roles="payment-service-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.internal:5432/payment_db?sslmode=require" \
  username="vault-admin" \
  password="${VAULT_ADMIN_PASSWORD}"

# Define a role — credentials valid for 1 hour, max 12 hours
vault write database/roles/payment-service-role \
  db_name=payment-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="12h"
```

```python
# Application: fetch dynamic database credentials from Vault
import hvac
import psycopg2

def get_db_connection():
    vault_client = hvac.Client(url="https://vault.internal:8200")
    # Authentication via Kubernetes service account (workload identity)
    with open("/var/run/secrets/kubernetes.io/serviceaccount/token") as f:
        jwt = f.read()

    vault_client.auth.kubernetes.login(
        role="payment-service",
        jwt=jwt,
    )

    # Fetch dynamic credentials (Vault generates them; they expire automatically)
    creds = vault_client.secrets.database.generate_credentials(
        name="payment-service-role"
    )

    return psycopg2.connect(
        host="postgres.internal",
        dbname="payment_db",
        user=creds["data"]["username"],
        password=creds["data"]["password"],
        sslmode="require",
    )
```

---

## Secret Distribution to Applications

### Distribution Anti-Patterns

| Anti-Pattern | Risk | Correct Pattern |
|---|---|---|
| Environment variables at build time | Secrets embedded in container layers | Runtime injection via Vault agent / CSI driver |
| `.env` files committed to git | Permanent secret exposure | Secrets manager + `.gitignore` enforcement |
| Secrets in Kubernetes ConfigMaps | ConfigMaps stored in etcd unencrypted | Kubernetes Secrets with etcd encryption + Vault sync |
| Hardcoded in application code | Visible to all with code access | Dynamic credentials or secrets manager API |
| Passed as CI/CD pipeline arguments | Visible in pipeline logs | Platform-native secrets (GitHub Secrets, GitLab CI variables) with masked output |

### Pattern: Vault Agent Sidecar Injection

```yaml
# Kubernetes pod with Vault Agent sidecar auto-injection
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  annotations:
    # Vault Agent sidecar injection
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "payment-service"
    # Inject dynamic DB credentials as a file (not env var)
    vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/payment-service-role"
    vault.hashicorp.com/agent-inject-template-db-creds: |
      {{- with secret "database/creds/payment-service-role" -}}
      {
        "username": "{{ .Data.username }}",
        "password": "{{ .Data.password }}"
      }
      {{- end }}
    # Agent will renew credentials before expiry — no application restart required
    vault.hashicorp.com/agent-pre-populate-only: "false"
spec:
  template:
    spec:
      serviceAccountName: payment-service
      containers:
      - name: payment-service
        image: registry.example.com/payment-service:2.1.0
        # Application reads credentials from file, not environment variable
        env:
        - name: DB_CREDENTIALS_PATH
          value: /vault/secrets/db-creds
```

### Pattern: Kubernetes Secrets Store CSI Driver

For environments where a sidecar pattern is not appropriate:

```yaml
# SecretProviderClass: mount Vault secrets as Kubernetes volume
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: payment-service-secrets
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.internal:8200"
    roleName: "payment-service"
    objects: |
      - objectName: "db-credentials"
        secretPath: "database/creds/payment-service-role"
        secretKey: "password"
      - objectName: "api-key"
        secretPath: "secret/data/payment-service/third-party-api"
        secretKey: "api_key"
  secretObjects:
  - secretName: payment-service-secrets
    type: Opaque
    data:
    - objectName: db-credentials
      key: db-password
```

---

## Automated Rotation Architecture

### Rotation Pipeline Design

Automated secret rotation requires a pipeline that:

1. **Generates** a new credential (or requests one from the underlying system)
2. **Validates** the new credential works before revoking the old one
3. **Updates** all consuming systems with the new credential
4. **Revokes** the old credential only after all consumers have been updated
5. **Audits** the rotation event in the compliance evidence store

This dual-write pattern (new credential in use before old credential is revoked) prevents downtime during rotation.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Rotation Pipeline                                │
│                                                                     │
│  1. Generate new credential (Vault / AWS / cloud API)               │
│  2. Test new credential against target system                       │
│  3. Update Vault / Secrets Manager with new credential              │
│  4. Wait for all pods to reload (Vault agent handles this)          │
│  5. Verify all pods using new credential (metrics check)            │
│  6. Revoke old credential                                           │
│  7. Record rotation event in compliance evidence store              │
└─────────────────────────────────────────────────────────────────────┘
```

### AWS Secrets Manager Rotation Lambda Pattern

```python
# AWS Lambda: custom rotation function for third-party API keys
import boto3
import json
import requests

def lambda_handler(event, context):
    arn = event["SecretId"]
    token = event["ClientRequestToken"]
    step = event["Step"]

    secrets_client = boto3.client("secretsmanager")

    if step == "createSecret":
        create_secret(secrets_client, arn, token)
    elif step == "setSecret":
        set_secret(secrets_client, arn, token)
    elif step == "testSecret":
        test_secret(secrets_client, arn, token)
    elif step == "finishSecret":
        finish_secret(secrets_client, arn, token)

def create_secret(client, arn, token):
    """Generate a new API key via the third-party API."""
    # Get current secret for API auth during new key generation
    current = json.loads(
        client.get_secret_value(SecretId=arn)["SecretString"]
    )

    # Call third-party API to generate new key
    response = requests.post(
        "https://api.example.com/v1/api-keys",
        headers={"Authorization": f"Bearer {current['api_key']}"},
        json={"description": "Rotated by Techstream automation"},
    )
    response.raise_for_status()
    new_key = response.json()["key"]

    # Store new key as pending version
    client.put_secret_value(
        SecretId=arn,
        ClientRequestToken=token,
        SecretString=json.dumps({
            **current,
            "api_key": new_key,
            "old_api_key": current["api_key"],  # Keep for revocation in finishSecret
        }),
        VersionStages=["AWSPENDING"],
    )

def test_secret(client, arn, token):
    """Validate the new key works before proceeding."""
    pending = json.loads(
        client.get_secret_value(
            SecretId=arn,
            VersionStage="AWSPENDING",
        )["SecretString"]
    )

    # Test the new key
    response = requests.get(
        "https://api.example.com/v1/whoami",
        headers={"Authorization": f"Bearer {pending['api_key']}"},
    )
    if response.status_code != 200:
        raise ValueError(f"New API key validation failed: {response.status_code}")

def finish_secret(client, arn, token):
    """Promote new key to current and revoke old key."""
    # Get version IDs for promotion
    metadata = client.describe_secret(SecretId=arn)
    for version_id, stages in metadata["VersionIdsToStages"].items():
        if "AWSCURRENT" in stages:
            if version_id == token:
                return  # Already promoted
            old_version = version_id

    # Promote pending to current
    client.update_secret_version_stage(
        SecretId=arn,
        VersionStage="AWSCURRENT",
        MoveToVersionId=token,
        RemoveFromVersionId=old_version,
    )

    # Revoke old API key
    pending = json.loads(
        client.get_secret_value(SecretId=arn)["SecretString"]
    )
    if "old_api_key" in pending:
        requests.delete(
            "https://api.example.com/v1/api-keys",
            headers={"Authorization": f"Bearer {pending['api_key']}"},
            json={"key": pending["old_api_key"]},
        )
```

---

## Rotation Patterns by Secret Type

### Database Credentials

**Preferred pattern:** Vault dynamic secrets (no rotation needed — credentials are generated per-session with short TTLs)

**Fallback pattern (static credential rotation):**
- Use AWS Secrets Manager rotation with the [AWS-provided Lambda rotation functions](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-rotation-templates.html)
- PostgreSQL, MySQL, MongoDB, and most managed databases have pre-built rotation functions
- Rotation window: off-peak hours; ensure application reconnect logic handles transient credential errors

### TLS Certificates

**Automated pattern:** cert-manager with Let's Encrypt (public-facing) or internal CA

```yaml
# cert-manager: auto-renewing TLS certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: payment-service-tls
  namespace: production
spec:
  secretName: payment-service-tls-secret
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  dnsNames:
  - payments.example.com
  duration: 2160h    # 90 days
  renewBefore: 360h  # Renew 15 days before expiry
```

**Internal mTLS:** Use SPIFFE/SPIRE for workload identity with automatic SVID rotation (default TTL: 1 hour).

### Cloud API Keys

**Preferred pattern:** Replace static API keys with OIDC workload identity:

```hcl
# AWS: OIDC provider trust for GitHub Actions (no stored AWS credentials)
resource "aws_iam_openid_connect_provider" "github_actions" {
  url = "https://token.actions.githubusercontent.com"
  client_id_list = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

resource "aws_iam_role" "github_actions_deploy" {
  name = "github-actions-deploy"
  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRoleWithWebIdentity"
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github_actions.arn
      }
      Condition = {
        StringLike = {
          "token.actions.githubusercontent.com:sub" = "repo:org/repo:*"
        }
      }
    }]
  })
}
```

### SSH Keys

SSH key rotation is operationally complex because it requires coordination across all systems that authorize the key. Automate using:

- **Vault SSH secrets engine** — issues short-lived signed SSH certificates instead of long-lived keys
- **AWS EC2 Instance Connect** — push temporary SSH public keys for a single connection
- **Google Cloud OS Login** — IAM-based SSH without static authorized_keys

---

## Secret Sprawl Detection and Remediation

### Continuous Scanning for Secret Leakage

Run Gitleaks or TruffleHog on every commit and on a scheduled scan of all repository history:

```yaml
# GitHub Actions: scan for secrets on every push + PR
- name: Scan for secrets (Gitleaks)
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

# Also run against full git history weekly (catches older commits)
- name: Scan full git history (scheduled)
  if: github.event_name == 'schedule'
  run: |
    gitleaks detect \
      --source . \
      --log-opts="--all" \
      --report-format json \
      --report-path gitleaks-history-report.json \
      --exit-code 0   # Report only; don't fail on historical findings
```

### Secret Inventory and Ownership

Maintain a secret inventory that maps every secret to its owner, consuming services, rotation schedule, and current status. The inventory enables impact assessment when a secret must be revoked:

```sql
-- Secret inventory schema
CREATE TABLE secret_inventory (
    secret_id       VARCHAR(255) PRIMARY KEY,
    secret_name     VARCHAR(500) NOT NULL,
    secret_store    VARCHAR(100) NOT NULL,  -- 'aws-secrets-manager', 'vault', 'github'
    tier            INTEGER NOT NULL,
    owner_team      VARCHAR(100) NOT NULL,
    consuming_services TEXT[],
    rotation_days   INTEGER,
    last_rotated_at TIMESTAMPTZ,
    next_rotation_at TIMESTAMPTZ GENERATED ALWAYS AS (
        last_rotated_at + (rotation_days || ' days')::INTERVAL
    ) STORED,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    tags            JSONB
);

-- Identify overdue rotations
SELECT
    secret_name,
    owner_team,
    last_rotated_at,
    next_rotation_at,
    NOW() - next_rotation_at AS overdue_by
FROM secret_inventory
WHERE next_rotation_at < NOW()
  AND tier IN (1, 2)
ORDER BY overdue_by DESC;
```

---

## Emergency Revocation

When a secret is confirmed to have been leaked, revocation must happen within minutes, not hours. Pre-plan emergency revocation for all Tier 1 and Tier 2 secrets.

### Emergency Revocation Runbook Template

```markdown
# Emergency Secret Revocation Runbook: [Secret Name]

**Secret ID:** [identifier in secrets manager]
**Tier:** [1-6]
**Owner:** [team/person]
**Consuming services:** [list]
**Estimated revocation impact:** [service outage? degraded functionality? no impact if rotation is clean]

## Pre-Revocation Steps (< 5 minutes)
1. Confirm the compromise (where was the secret observed? by whom?)
2. Notify [owner team] and [security on-call]
3. Open a P1 incident: [incident management link]

## Revocation Steps (< 15 minutes)
1. [Step-by-step revocation procedure specific to this secret type]
2. Verify the old secret is non-functional (test with the old value; expect auth failure)
3. Verify the new secret is functional (consuming services operating normally)

## Post-Revocation Steps (< 1 hour)
1. Confirm all consuming services have received the new credential
2. Rotate any secrets that could have been accessed using the compromised credential
3. Review audit logs for unauthorized use during the window between compromise and revocation
4. Document the incident timeline

## Escalation
If revocation cannot be completed within 15 minutes: [escalation path]
```

### Revocation Response Time Targets

| Tier | Discovery to Revocation | Acceptable Downtime |
|---|---|---|
| Tier 1 (service credentials) | < 15 minutes | < 2 minutes (blue/green credential swap) |
| Tier 2 (long-lived tokens) | < 30 minutes | Typically zero (service continues with new token) |
| Tier 3 (certificates) | < 1 hour | < 5 minutes (cert-manager re-issue) |
| Tier 5 (SSH keys) | < 2 hours | None (SSH key revocation doesn't affect running sessions) |

---

## Secrets in CI/CD Pipelines

### Pipeline Secret Hygiene Rules

1. **Never print secrets in logs** — configure log masking in the CI/CD platform for all registered secrets. Add explicit `set +x` before secret-handling steps in shell scripts.

2. **Use platform-native secrets** — GitHub Actions Secrets, GitLab CI Variables, Jenkins Credentials are masked in logs and audited.

3. **Scope secrets to minimum required workflows** — GitHub repository secrets are available to all workflows; use environment secrets to restrict to specific deployment workflows.

4. **Prefer OIDC over static keys** — use OIDC workload identity for AWS, Azure, GCP authentication from CI/CD. Eliminates cloud API keys from the pipeline entirely.

5. **Rotate CI/CD secrets on the same schedule as service secrets** — CI/CD platform secrets are often overlooked in rotation programs.

```yaml
# GitHub Actions: OIDC authentication to AWS (no stored AWS credentials)
- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
    role-session-name: GitHubActions-${{ github.run_id }}
    aws-region: us-east-1
    # No aws-access-key-id or aws-secret-access-key needed
```

---

## Audit and Compliance

### Required Audit Evidence for Secrets Management

| Evidence Type | Collection Method | Frequency | Retention |
|---|---|---|---|
| Secret inventory snapshot | Automated export from secrets managers | Weekly | 7 years |
| Rotation completion records | Rotation Lambda/function output to evidence store | Per rotation event | 7 years |
| Access logs (who accessed which secret, when) | Secrets manager audit logs (CloudTrail, Vault audit log) | Continuous | 7 years |
| Overdue rotation report | Automated query of secret inventory | Weekly | 3 years |
| Emergency revocation records | Incident management + manual evidence | Per event | 7 years |

### Compliance Mapping

| Control Requirement | Techstream Control | Measurement |
|---|---|---|
| SOC 2 CC6.1 — Logical access controls | Secret inventory, RBAC on secrets manager | Access review completion rate |
| ISO 27001 A.8.24 — Use of cryptography | Encryption key management, certificate rotation | Key rotation compliance % |
| PCI-DSS 8.6 — Manage all user IDs | System account credential rotation | Rotation SLA compliance % |
| NIST 800-53 IA-5 — Authenticator management | Secret rotation automation | Overdue rotation count |

---

## Measurement

| Metric | Definition | Target |
|---|---|---|
| **Rotation compliance rate** | % of Tier 1–2 secrets rotated within their scheduled interval | 100% |
| **Long-lived credential count** | Number of Tier 1–2 secrets with age > rotation threshold | 0 |
| **Vault dynamic secret adoption** | % of service database connections using dynamic credentials | ≥ 75% (L3); ≥ 95% (L4+) |
| **OIDC workload identity adoption** | % of CI/CD cloud auth using OIDC vs. static API keys | 100% for cloud providers supporting OIDC |
| **Secret leak MTTD** | Time from secret leak to detection (via Gitleaks/GitGuardian alert) | < 5 minutes (real-time scanning) |
| **Secret leak MTTR** | Time from detection to revocation and rotation | < 15 minutes for Tier 1 |
| **Secret inventory coverage** | % of secrets in the inventory vs. secrets discovered by audit | ≥ 99% |

---

## Related Techstream Resources

- [DevSecOps Framework — API Security](api-security.md)
- [DevSecOps Maturity Model — Secrets and Credential Hygiene Metrics](../devsecops-maturity-model/docs/metrics-kpis.md)
- [Secure CI/CD Reference Architecture — Pipeline Security Zones](../secure-ci-cd-reference-architecture/docs/architecture.md)
- [Secure Pipeline Templates — GitHub Actions Secure Pipeline](../secure-pipeline-templates/templates/github-actions-secure-pipeline.yml)
- [Software Supply Chain Security Framework — Build Security](../software-supply-chain-security-framework/docs/framework.md)
- [Cloud Security DevSecOps — Secrets Management](../cloud-security-devsecops/docs/framework.md)
