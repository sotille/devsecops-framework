# Runtime Threat Detection and Behavioral Security

Security controls applied at development, build, and deployment time create a strong pre-production security posture. Runtime threat detection extends this protection into the production environment — detecting threats that evade static controls, responding to zero-day exploitation, and generating forensic evidence when incidents occur.

This document covers the tools, techniques, and operational model for runtime security monitoring in cloud-native environments, with focus on Kubernetes workloads, container runtime behavior, and cloud infrastructure.

---

## Why Runtime Detection Is Required

Pre-production controls (SAST, SCA, container scanning) operate on static artifacts. They cannot detect:

- **Zero-day exploitation** — vulnerabilities with no known CVE at deployment time
- **Post-deployment supply chain compromise** — a dependency or base image compromised after your last build
- **Credential theft from running workloads** — an attacker extracting secrets from a container's environment or process memory
- **Lateral movement** — a compromised workload reaching other services it should not contact
- **Privilege escalation** — a container exploit gaining host-level capabilities
- **Data exfiltration** — a workload sending data to an unauthorized external endpoint
- **Insider threat** — a developer or admin action that abuses legitimate access patterns

Runtime detection is the last line of active defense and the first line of forensic capability.

---

## Runtime Threat Detection Architecture

```
┌───────────────────────────────────────────────────────────────────────┐
│                    Kubernetes Node                                      │
│                                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐ │
│  │  Container A │  │  Container B │  │       eBPF / Falco Agent      │ │
│  │  (app)       │  │  (sidecar)   │  │                               │ │
│  │              │  │              │  │  Intercepts syscalls at        │ │
│  │  syscalls    │  │  syscalls    │  │  kernel level — zero overhead  │ │
│  └──────┬───────┘  └──────┬───────┘  │  on application code         │ │
│         │                 │          └────────────────┬──────────────┘ │
│         └─────────────────┘                           │                │
│                            Linux Kernel               │                │
│  ┌─────────────────────────────────────────────────── │ ─────────────┐ │
│  │  Syscall table ←────────────────────── eBPF probe ─┘              │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────┬────────────────────────────────┘
                                       │
                                  Events stream
                                       │
                         ┌─────────────▼──────────────┐
                         │  Event Processing           │
                         │  - Rule evaluation          │
                         │  - Enrichment (k8s meta)    │
                         │  - Alert generation         │
                         └─────────────┬──────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                   │
                    ▼                  ▼                   ▼
             ┌────────────┐    ┌─────────────┐    ┌──────────────┐
             │   SIEM     │    │  PagerDuty  │    │  Falco       │
             │ (Splunk/   │    │  / OpsGenie │    │  Sidekick    │
             │  OpenSearch│    │  (alerts)   │    │  (fanout)    │
             └────────────┘    └─────────────┘    └──────────────┘
```

---

## Falco: Kernel Syscall Monitoring

Falco is the CNCF-graduated runtime security tool for Kubernetes and Linux. It monitors the kernel syscall stream using eBPF probes (or the legacy kernel module driver) and evaluates security rules against the observed behavior. Falco does not require modifying application code or using a sidecar.

### Deployment

**Helm installation (recommended for Kubernetes):**

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \
  --set falco.grpc.enabled=true \
  --set falco.grpc_output.enabled=true \
  --set falco.json_output=true \
  --set falco.log_level=info \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=false \
  --set-string falcosidekick.config.slack.webhookurl="${SLACK_WEBHOOK}" \
  --set-string falcosidekick.config.pagerduty.routingKey="${PAGERDUTY_KEY}"
```

**Verify deployment:**

```bash
kubectl get pods -n falco
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=50
```

### Core Rule Categories

Falco ships with a comprehensive default ruleset. The following categories map to the most critical detection scenarios for cloud-native environments.

**Container escape and privilege escalation:**

```yaml
# rules/container-escape.yaml
- rule: Container Run as Root
  desc: Detect a container running as root where it should not
  condition: >
    container.id != host
    and container.privileged = false
    and user.uid = 0
    and not user_known_run_as_root_containers
  output: >
    Container running as root (container=%container.name
    image=%container.image.repository:%container.image.tag
    user=%user.name pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: WARNING
  tags: [container, cis, mitre_privilege_escalation]

- rule: Privileged Container Started
  desc: Detect the start of a privileged container
  condition: >
    evt.type = container
    and container.privileged = true
    and not falco_privileged_containers
    and not user_privileged_containers
  output: >
    Privileged container started (container=%container.name
    image=%container.image.repository:%container.image.tag
    pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: WARNING
  tags: [container, cis, mitre_privilege_escalation]

- rule: Mount Sensitive Host Path
  desc: Detect mounting of sensitive host paths inside a container
  condition: >
    evt.type = container
    and container.mount.dest[/proc*] != ""
      or container.mount.dest[/sys*] != ""
      or container.mount.dest[/dev*] != ""
      or container.mount.dest[/run/docker*] != ""
  output: >
    Sensitive host path mounted in container (container=%container.name
    mounts=%container.mounts pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: ERROR
  tags: [container, mitre_privilege_escalation]
```

**Suspicious network activity:**

```yaml
- rule: Outbound Connection to Unexpected External IP
  desc: Detect a container making an outbound connection to an IP outside expected ranges
  condition: >
    outbound
    and not fd.sip in (allowed_external_ips)
    and not fd.snet in (private_ip_ranges)
    and container.id != host
    and not proc.name in (expected_external_network_procs)
  output: >
    Unexpected outbound network connection (command=%proc.cmdline
    container=%container.name dest=%fd.name pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: WARNING
  tags: [network, mitre_exfiltration, mitre_command_and_control]

- rule: Unexpected UDP Traffic
  desc: UDP traffic from containers that should only use TCP
  condition: >
    evt.type = connect
    and fd.l4proto = udp
    and fd.sport != 53
    and container.id != host
    and not expected_udp_procs
  output: >
    Unexpected UDP connection (command=%proc.cmdline
    container=%container.name dest=%fd.name)
  priority: WARNING
```

**Credential and secret access:**

```yaml
- rule: Read Shell Configuration File
  desc: Detect reading of shell configuration files that may contain credentials
  condition: >
    open_read
    and (fd.name = /root/.bashrc
         or fd.name = /root/.bash_profile
         or fd.name = /root/.profile
         or fd.name = /home/.*/\.bashrc
         or fd.name = /home/.*/\.bash_profile)
    and container.id != host
    and not proc.name in (shell_procs)
  output: >
    Shell config file read from container (command=%proc.cmdline
    file=%fd.name container=%container.name pod=%k8s.pod.name)
  priority: WARNING
  tags: [credential_access, mitre_credential_access]

- rule: Access Kubernetes Service Account Token
  desc: Detect reading of the Kubernetes service account token from within a container
  condition: >
    open_read
    and fd.name startswith /var/run/secrets/kubernetes.io/serviceaccount
    and not proc.name in (k8s_client_procs)
    and container.id != host
  output: >
    Service account token read unexpectedly (command=%proc.cmdline
    container=%container.name pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: WARNING
  tags: [credential_access, mitre_credential_access]
```

**Persistence mechanisms:**

```yaml
- rule: Cron Job Creation in Container
  desc: Detect creation of cron jobs inside containers (persistence mechanism)
  condition: >
    (evt.type = write or evt.type = creat)
    and (fd.name startswith /etc/cron
         or fd.name = /var/spool/cron/crontabs/root)
    and container.id != host
  output: >
    Cron job created inside container (command=%proc.cmdline
    file=%fd.name container=%container.name pod=%k8s.pod.name)
  priority: ERROR
  tags: [persistence, mitre_persistence]

- rule: Write Below Root
  desc: An attempt to write to the root directory tree (excluding /tmp, /var, /proc, /run)
  condition: >
    evt.type = write
    and fd.name startswith /
    and not fd.name startswith /tmp
    and not fd.name startswith /var
    and not fd.name startswith /run
    and not fd.name startswith /proc
    and not fd.name startswith /sys
    and container.id != host
    and not proc.name in (expected_root_write_procs)
  output: >
    Write below root directory (command=%proc.cmdline
    file=%fd.name container=%container.name)
  priority: ERROR
```

### Custom Rule Development

When writing custom Falco rules for your environment:

1. **Start with warn, not block**: Falco detects but does not block by default. Use alerts to tune before integrating with response automation.
2. **Establish baselines first**: Deploy Falco in logging-only mode for 2-4 weeks to capture normal behavior before writing alert rules.
3. **Use macros for reusable conditions**: Extract common conditions into macros to avoid duplication.
4. **Tag rules with MITRE ATT&CK**: Tags enable mapping detections to threat intelligence.

```yaml
# Macro: define containers that legitimately run as root
- macro: user_known_root_containers
  condition: >
    container.image.repository in (
      "nginx",
      "registry.internal/platform/init-container"
    )

# Macro: known external network services
- macro: allowed_external_endpoints
  condition: >
    fd.sip in ("8.8.8.8", "8.8.4.4")  # DNS servers
    or fd.snet in ("10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16")

# List: processes that legitimately access K8s SA tokens
- list: k8s_client_procs
  items: [kubectl, helm, argocd, flux]
```

---

## eBPF-Based Observability

Beyond Falco, eBPF (Extended Berkeley Packet Filter) provides low-overhead kernel-level observability for security, performance, and network telemetry. eBPF programs run in the kernel in a sandboxed environment verified by the kernel's eBPF verifier — they cannot crash the kernel or be exploited.

### Security Use Cases for eBPF

| Use Case | Tool | Signal Captured |
|----------|------|----------------|
| Syscall-level intrusion detection | Falco (eBPF driver), Tetragon | All syscalls with process and container context |
| Network flow visibility | Cilium, Hubble | All TCP/UDP connections with identity-aware metadata |
| Process execution tracking | Tetragon, Pixie | All process executions with arguments and parent chain |
| File integrity monitoring | Falco, Tetragon | All file opens, writes, and permission changes |
| Kubernetes-aware network policy | Cilium | L3/L4/L7 policy with identity-based enforcement |

### Tetragon: eBPF-Based Process Monitoring

Tetragon (from Isovalent/Cilium) provides process-level tracing with Kubernetes awareness. It can enforce process execution policies in-kernel — blocking, not just alerting.

```yaml
# Tetragon TracingPolicy: monitor process execution
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: monitor-sensitive-file-access
spec:
  kprobes:
    - call: "fd_install"
      syscall: false
      return: false
      args:
        - index: 0
          type: int
        - index: 1
          type: file
      selectors:
        - matchPIDs:
            - operator: NotIn
              isNamespacePID: true
              values: [1]
          matchArgs:
            - index: 1
              operator: Prefix
              values:
                - "/etc/shadow"
                - "/etc/passwd"
                - "/root/.ssh"
          matchActions:
            - action: Sigkill  # Kill the process accessing these files
```

### Cilium and Hubble: Network Observability with Identity Awareness

Cilium replaces kube-proxy and uses eBPF for Kubernetes network policy enforcement. Hubble provides network flow visibility built on Cilium.

```yaml
# Cilium NetworkPolicy: L7-aware policy (HTTP path-level enforcement)
apiVersion: cilium.io/v1
kind: CiliumNetworkPolicy
metadata:
  name: payment-api-policy
  namespace: payments
spec:
  endpointSelector:
    matchLabels:
      app: payment-processor
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: api-gateway
      toPorts:
        - ports:
            - port: "8443"
              protocol: TCP
          rules:
            http:
              - method: "POST"
                path: "/api/v1/payments"
              - method: "GET"
                path: "/api/v1/payments/[0-9]+"
    # Explicitly deny introspection and admin endpoints
    - fromEndpoints:
        - {}
      toPorts:
        - ports:
            - port: "8443"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/admin/.*"
          action: Deny
```

---

## Cloud Provider Runtime Security

Cloud providers offer managed runtime detection services that complement Falco and eBPF tools.

### AWS GuardDuty

GuardDuty analyzes VPC Flow Logs, CloudTrail events, DNS logs, and (with EKS Runtime Monitoring enabled) container runtime events.

| Finding Category | Examples | Severity |
|-----------------|---------|---------|
| Cryptocurrency mining | UnauthorizedAccess:EC2/BitcoinTool.B, CryptoCurrency:EC2/BitcoinTool.B | High |
| Credential compromise | UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS | Critical |
| Reconnaissance | Recon:IAMUser/MaliciousIPCaller | Medium |
| Backdoor | Backdoor:EC2/C&CActivity.B | High |
| EKS runtime | Execution:Kubernetes/NewBinaryExecutedInContainer, PrivilegeEscalation:Kubernetes/PrivilegedContainer | High |

Enable GuardDuty via Terraform:

```hcl
resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs {
      enable = true
    }
    kubernetes {
      audit_logs {
        enable = true
      }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          enable = true
        }
      }
    }
  }

  # Enable EKS Runtime Monitoring
  features {
    name   = "EKS_RUNTIME_MONITORING"
    status = "ENABLED"
  }
}

# Publish findings to Security Hub for centralized management
resource "aws_guardduty_publishing_destination" "security_hub" {
  detector_id     = aws_guardduty_detector.main.id
  destination_arn = aws_s3_bucket.findings.arn
  kms_key_arn     = aws_kms_key.findings.arn
}
```

### Azure Defender for Containers

Microsoft Defender for Containers (part of Defender for Cloud) provides threat detection for AKS clusters.

```hcl
resource "azurerm_security_center_subscription_pricing" "defender_containers" {
  tier          = "Standard"
  resource_type = "Containers"
}

# Enable Defender profile on AKS
resource "azurerm_kubernetes_cluster" "main" {
  # ... other config ...

  microsoft_defender {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.security.id
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.security.id
  }
}
```

### GCP Security Command Center + Container Threat Detection

```hcl
resource "google_project_service" "scc" {
  service = "securitycenter.googleapis.com"
}

# Enable Container Threat Detection (built into SCC Premium)
resource "google_scc_source" "container_threat_detection" {
  display_name = "Container Threat Detection"
  organization = var.organization_id
  description  = "GCP Container Threat Detection findings"
}
```

---

## Alert Prioritization and Response

Not every Falco alert warrants immediate human response. Use a tiered model:

| Priority | Criteria | Response |
|----------|---------|----------|
| P1 — Immediate | Privilege escalation, container escape, credential exfiltration, C2 communication | Automated containment (pod termination) + immediate SOC page |
| P2 — Urgent | Unexpected shell spawn, sensitive file access, unexpected outbound connections | SOC alert within 15 minutes; investigate before deciding on containment |
| P3 — Investigate | Policy drift, unexpected network connection within private ranges, unusual process activity | SIEM ticket; review within 4 hours |
| P4 — Audit | Informational baseline events, expected admin activity, normal framework operations | Log only; used for behavioral baseline |

### Automated Containment Response

For P1 events, automate initial containment while the human investigation begins:

```python
# Falco → Falcosidekick → custom webhook → containment action
# This function is called when a P1 Falco alert is received

import kubernetes
import logging

def contain_compromised_pod(namespace: str, pod_name: str, rule_name: str):
    """
    Apply network isolation and log the containment action.
    Does NOT delete the pod to preserve forensic evidence.
    """
    v1 = kubernetes.client.CoreV1Api()
    networking_v1 = kubernetes.client.NetworkingV1Api()

    # Step 1: Label pod as under investigation
    v1.patch_namespaced_pod(
        name=pod_name,
        namespace=namespace,
        body={"metadata": {"labels": {"security.techstream.io/status": "contained",
                                       "security.techstream.io/rule": rule_name}}}
    )

    # Step 2: Apply deny-all NetworkPolicy targeting the compromised pod
    isolation_policy = kubernetes.client.V1NetworkPolicy(
        metadata=kubernetes.client.V1ObjectMeta(
            name=f"isolate-{pod_name}",
            namespace=namespace,
            annotations={
                "security.techstream.io/reason": rule_name,
                "security.techstream.io/contained-at": datetime.utcnow().isoformat()
            }
        ),
        spec=kubernetes.client.V1NetworkPolicySpec(
            pod_selector=kubernetes.client.V1LabelSelector(
                match_labels={"security.techstream.io/status": "contained"}
            ),
            policy_types=["Ingress", "Egress"]
        )
    )

    networking_v1.create_namespaced_network_policy(namespace, isolation_policy)

    logging.info(f"Contained pod {namespace}/{pod_name} due to rule: {rule_name}")

    # Step 3: Create incident ticket (PagerDuty/ServiceNow webhook)
    create_security_incident(
        title=f"P1: Container isolation triggered — {rule_name}",
        description=f"Pod {namespace}/{pod_name} isolated. Pod is preserved for forensic analysis.",
        severity="P1"
    )
```

---

## Runtime Security Metrics

Track the effectiveness of your runtime detection program with these KPIs:

| Metric | Definition | Target |
|--------|-----------|--------|
| Mean Time to Detect (MTTD) | Time from threat event to alert generation | < 5 minutes for P1 |
| Alert fidelity rate | Percentage of alerts confirmed as true positives | > 80% (tuning target) |
| Rule coverage | Percentage of MITRE ATT&CK techniques with at least one detection rule | > 70% of applicable techniques |
| Containment time | Time from P1 alert to network isolation | < 10 minutes (automated) |
| Baseline drift rate | Percentage of workloads exhibiting behavioral changes week-over-week | Track; alert on spikes |
| Uncovered high-severity findings | GuardDuty/Defender findings with no corresponding Falco rule | Drive to zero |

---

## Integration with the Techstream Ecosystem

| Framework | Integration Point |
|-----------|-----------------|
| `secure-ci-cd-reference-architecture` | Falco and eBPF tools should be deployed via the same GitOps pipeline that deploys application workloads |
| `devsecops-maturity-model` | Runtime detection capability is scored in Domain 7 (Operations and Monitoring) — L1: no runtime detection, L3: Falco deployed with default rules, L5: tuned rules + automated containment + behavioral baselines |
| `compliance-automation-framework` | Falco JSON output feeds SIEM evidence for SOC 2 CC7.2, PCI-DSS Req 10.6, ISO 27001 A.8.16 |
| `cloud-security-devsecops` | GuardDuty/Defender/SCC provide the cloud control plane layer; Falco and eBPF provide the workload layer — both are required for full coverage |
| `release-orchestration-framework` | Post-deployment: smoke test period with elevated Falco alerting sensitivity before canary promotion |

---

## Related Documents

- [Architecture](architecture.md) — Six-layer toolchain including runtime monitoring layer
- [Best Practices](best-practices.md) — Container runtime security best practices
- [API Security](api-security.md) — API gateway runtime controls
- [Cloud Security: eBPF Security](../../cloud-security-devsecops/docs/ebpf-security.md) — Deep dive on eBPF-based security tools
- [Cloud Security: Zero Trust Architecture](../../cloud-security-devsecops/docs/zero-trust-architecture.md) — Zero trust principles for runtime access control
- [Compliance Automation: Evidence Collection](../../compliance-automation-framework/docs/evidence-collection-automation.md) — Collecting Falco and GuardDuty findings as compliance evidence
