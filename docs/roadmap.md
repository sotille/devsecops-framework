# DevSecOps 18-Month Implementation Roadmap

## Table of Contents

- [Roadmap Overview](#roadmap-overview)
- [Maturity Model (5 Levels)](#maturity-model-5-levels)
- [18-Month Roadmap by Quarter](#18-month-roadmap-by-quarter)
- [KPIs per Maturity Stage](#kpis-per-maturity-stage)
- [Organizational Change Management](#organizational-change-management)
- [Technology Adoption Sequence](#technology-adoption-sequence)
- [Risk Mitigation During Transition](#risk-mitigation-during-transition)

---

## Roadmap Overview

The DevSecOps 18-Month Roadmap provides a structured, achievable path from minimal security integration to a mature, continuously improving DevSecOps program. The roadmap is organized around quarterly milestones and aligned to the five-level DevSecOps Maturity Model.

### Guiding Principles for the Roadmap

- **Incremental over revolutionary:** Each phase delivers measurable security improvements without disrupting delivery velocity
- **Prove value early:** Phase 1 targets the highest-impact, lowest-friction improvements to build organizational trust
- **Technology follows process:** Tooling is introduced only after the processes and roles it supports are defined
- **Measure everything:** Every phase has measurable KPIs that demonstrate progress and justify continued investment
- **Culture is the hardest part:** Technical controls are achievable in months; cultural change requires sustained effort over years

### Key Stakeholders

| Stakeholder | Role in Roadmap |
|---|---|
| CISO / Security Leadership | Executive sponsor; resource allocation; board reporting |
| VP Engineering | Co-sponsor; ensures engineering teams prioritize security work |
| Security Engineers | Technical implementation leads for security toolchain |
| Platform Engineers | CI/CD platform implementation and maintenance |
| Engineering Managers | Team-level adoption; security OKR ownership |
| Security Champions | Team-level advocates and first-line security responders |
| Developers | Tool adoption; finding remediation; culture shift |

---

## Maturity Model (5 Levels)

### Level 1: Initial (Ad Hoc)

**Description:** Security is applied inconsistently and reactively. Security activities are not integrated into the development workflow and depend on individual initiative or periodic external assessments.

**Characteristics:**
- No consistent SAST, DAST, or SCA in CI pipelines
- Secrets occasionally hardcoded or poorly managed
- Security issues discovered primarily through production incidents or annual pen tests
- No defined vulnerability management process or SLAs
- Security team is perceived as a blocker
- No security training program for developers
- IaC exists but is not scanned for security misconfigurations

**Risk Level:** Very High — significant probability of undetected vulnerabilities reaching production; high likelihood of a security incident within 12–24 months

---

### Level 2: Foundation (Reactive to Proactive)

**Description:** Core security controls are in place. Automated scanning exists but is not consistently enforced. Vulnerability management process is defined but compliance is variable.

**Characteristics:**
- SAST and SCA run in CI pipelines but are not always blocking
- Secret scanning enabled; some historical secrets cleaned up
- Basic vulnerability management process with defined SLAs
- Pre-commit hooks deployed on most repositories
- Branch protection enabled on default branches
- Some security training completed; no champions program yet
- IaC scanning introduced

**Risk Level:** High — significant reduction in critical vulnerability exposure; major vulnerability classes are being detected

**Transition from Level 1 to Level 2:** Approximately 2–4 months with dedicated effort

---

### Level 3: Defined (Consistent and Enforced)

**Description:** Security controls are consistently applied across all production repositories. Security gates enforce minimum standards. Vulnerability management SLAs are tracked and met with > 80% compliance.

**Characteristics:**
- SAST, SCA, secrets, IaC, and container scanning fully operational and blocking
- Zero secrets in repositories or pipelines
- Vulnerability management SLA compliance > 80% for Critical/High
- Security Champions program operational
- Secrets management infrastructure deployed (Vault or cloud-native)
- OIDC federation with cloud providers
- DAST integrated into pre-production pipeline
- SBOM generated for all production artifacts
- Security KPIs baselined and improving

**Risk Level:** Medium — consistent security baseline; most common vulnerability classes detected and remediated before production

**Transition from Level 2 to Level 3:** Approximately 4–6 months

---

### Level 4: Managed (Measured and Optimized)

**Description:** Security is a measurable, continuously improving engineering function. Advanced security controls provide defense in depth. Security metrics are visible to leadership and drive resource allocation.

**Characteristics:**
- All artifacts cryptographically signed; admission control enforces signatures
- Ephemeral runner infrastructure deployed
- Runtime security monitoring (Falco, CSPM) fully operational
- SAST false positive rate < 15%
- Monthly security KPI report to engineering leadership
- Security OKRs integrated into team objectives
- Threat modeling practiced consistently for significant features
- Supply chain security controls in place (SLSA provenance)
- Compliance evidence collection largely automated

**Risk Level:** Low-Medium — mature security posture; sophisticated attacks are still possible but organization detects and responds quickly

**Transition from Level 3 to Level 4:** Approximately 6–9 months

---

### Level 5: Optimizing (Continuously Improving)

**Description:** Security is an embedded organizational capability that continuously improves. The security program operates proactively, anticipates emerging threats, and drives innovation in security practice.

**Characteristics:**
- All Level 4 characteristics plus:
- Automated threat modeling with continuous updates
- Bug bounty program or continuous penetration testing program
- Security excellence is recognized and rewarded culturally
- Organization contributes to open-source security projects and industry standards
- Emerging threat intelligence integrated into security toolchain
- SLSA Level 3+ provenance for all production artifacts
- Regulatory compliance achieved and maintained continuously through automation
- Security KPIs trend positively quarter-over-quarter without significant security incidents

**Risk Level:** Low — industry-leading security posture; breach risk significantly reduced

**Transition from Level 4 to Level 5:** 12–18 months of sustained optimization

---

## 18-Month Roadmap by Quarter

### Q1 (Months 1–3): Foundation

**Theme:** Establish visibility and eliminate the most critical risks without disrupting delivery

**Milestone 1.1: Complete Security Baseline Assessment** (Month 1)
- Conduct full inventory of all repositories and CI/CD pipelines
- Run baseline SAST and SCA scans across all repositories (report-only mode)
- Audit all CI/CD pipeline credentials and permissions
- Assess current vulnerability management capabilities
- Identify all existing secrets in version control (historical scan)

**Deliverables:**
- Baseline security assessment report
- Repository and pipeline inventory
- Critical risk register with prioritized remediation plan

---

**Milestone 1.2: Critical Risk Remediation** (Month 1–2)
- Rotate all exposed secrets discovered in baseline scan
- Remediate all Critical CVEs identified in SCA baseline
- Replace all hardcoded pipeline credentials with proper secrets management
- Implement OIDC federation for at least 3 highest-risk pipelines

**Deliverables:**
- Zero exposed secrets in version control
- Critical CVE count reduced by > 80%
- Secrets manager deployed and initial migration complete

---

**Milestone 1.3: Pre-Commit and SCM Security Controls** (Month 2–3)
- Deploy pre-commit hooks framework across all production repositories
- Enable branch protection on all default branches
- Enable secret scanning on all repositories
- Enable dependency vulnerability scanning (Dependabot or GitLab Security)
- Begin developer security communication and awareness campaign

**Deliverables:**
- Pre-commit hooks deployed and documented
- Branch protection policy enforced on all production repositories
- Developer security awareness communication distributed

---

**Milestone 1.4: CI Security Scan Deployment (Warning Mode)** (Month 3)
- Add SAST scan to all CI pipelines (warning mode — do not block)
- Add SCA scan to all CI pipelines (warning mode)
- Set up vulnerability management platform (DefectDojo or commercial equivalent)
- Establish baseline metrics for Phase 1

**Deliverables:**
- SAST and SCA running in all CI pipelines
- Vulnerability management platform operational
- Q1 security baseline metrics documented

**Q1 KPIs:**
- Exposed secrets in VCS: 0 (from baseline)
- % pipelines with basic security scanning: 100%
- Critical CVE count: reduced > 80% from baseline

---

### Q2 (Months 4–6): Integration

**Theme:** Move from visibility to enforcement; integrate security findings into the developer workflow

**Milestone 2.1: Security Gate Activation** (Month 4)
- Activate blocking SAST gates for Critical findings
- Activate blocking SCA gates for Critical CVEs
- Activate zero-tolerance secrets gate
- Publish remediation guidance in gate failure messages
- Define and communicate exception management process

**Deliverables:**
- All blocking security gates active in all production pipelines
- Exception management process documented and operational
- Security gate failure notifications to development teams operational

---

**Milestone 2.2: Container and IaC Security** (Month 4–5)
- Implement IaC security scanning (Checkov or tfsec) across all Terraform/CloudFormation repositories
- Implement container image scanning (Trivy) for all container image builds
- Add SBOM generation to all container image builds
- Activate blocking gates for Critical container vulnerabilities

**Deliverables:**
- IaC scanning active in all infrastructure repositories
- Container scanning active in all container build pipelines
- SBOM generated and stored for every container image build

---

**Milestone 2.3: Secrets Management Infrastructure** (Month 5–6)
- Deploy HashiCorp Vault or cloud-native secrets manager
- Migrate all remaining CI/CD secrets to Vault
- Implement dynamic secrets for database credentials
- Complete OIDC federation rollout for all cloud-targeting pipelines
- Delete all long-lived cloud credentials from CI/CD systems

**Deliverables:**
- Centralized secrets management platform operational
- Zero long-lived cloud credentials in CI/CD platforms
- Dynamic database credentials in use for all production databases

---

**Milestone 2.4: DAST Integration** (Month 6)
- Deploy staging environment with DAST capability
- Integrate OWASP ZAP into pre-production pipeline
- Configure DAST to gate production promotion on Critical findings
- Run initial DAST scan against all production applications; prioritize findings

**Deliverables:**
- DAST scanning integrated into all staging pipelines
- DAST blocking gate active for production promotion
- Initial DAST findings triaged and in remediation pipeline

**Q2 KPIs:**
- Security gate pass rate on first attempt: > 80%
- Vulnerability SLA compliance (Critical): > 85%
- Vulnerability SLA compliance (High): > 70%
- % artifacts with SBOM: 100%
- Long-lived cloud credentials in pipelines: 0

---

### Q3 (Months 7–9): Hardening

**Theme:** Harden CI/CD infrastructure itself; deploy advanced controls; launch people program

**Milestone 3.1: Pipeline Infrastructure Hardening** (Month 7–8)
- Deploy ephemeral runner infrastructure using Kubernetes Runner Scale Sets
- Implement runner network egress controls (allowlist-based)
- Harden runner security contexts (non-root, read-only rootfs, dropped capabilities)
- Audit and reduce pipeline token permissions to minimum necessary
- Implement pipeline configuration monitoring for unauthorized changes

**Deliverables:**
- Ephemeral runners deployed for all CI workloads
- Network egress policies active for all build environments
- Pipeline token permissions audit complete with remediation

---

**Milestone 3.2: Artifact Signing and Admission Control** (Month 7–8)
- Implement Cosign/Sigstore keyless signing for all container image builds
- Deploy Kyverno admission controller with image verification policies
- Activate enforcement for all production namespaces
- Test and verify signature verification is working as expected

**Deliverables:**
- 100% of container images signed after build
- Admission controller blocking unsigned image deployments in production
- Signature verification testing documented and passing

---

**Milestone 3.3: Runtime Security Deployment** (Month 8–9)
- Deploy Falco to all production Kubernetes namespaces
- Configure Falco rules for organization-specific threat patterns
- Integrate Falco alerts with SIEM
- Deploy CSPM tooling (Wiz, Prisma Cloud, or cloud-native)
- Configure CSPM compliance frameworks (CIS benchmarks)

**Deliverables:**
- Runtime security monitoring active in all production namespaces
- SIEM receiving and processing runtime security alerts
- CSPM dashboard operational with compliance baseline established

---

**Milestone 3.4: Security Champions Program Launch** (Month 8–9)
- Identify and formally designate Security Champions for all development teams
- Deliver initial Security Champion training (16-hour curriculum)
- Launch monthly Security Champion community of practice
- Define Security Champion responsibilities and success metrics
- Integrate Security Champions into security review processes

**Deliverables:**
- Security Champions program officially launched
- > 80% of development teams have a designated Security Champion
- Monthly Security Champion community meetings scheduled

**Q3 KPIs:**
- % CI workloads using ephemeral runners: 100%
- % production images signed and verified: 100%
- % production namespaces with runtime security monitoring: 100%
- % development teams with active Security Champion: > 80%
- Runtime security alerts actioned within SLA: > 90%

---

### Q4 (Months 10–12): Measurement

**Theme:** Establish comprehensive metrics; validate security posture; begin optimization

**Milestone 4.1: Security Metrics Program** (Month 10)
- Deploy security KPI dashboard visible to engineering and security leadership
- Automate metric collection from all security tooling
- Establish quarterly improvement targets for all KPIs
- Integrate security KPIs into engineering team OKRs

**Deliverables:**
- Monthly security KPI dashboard published and reviewed by leadership
- Security OKRs integrated into engineering team quarterly objectives
- Automated metric collection from SAST/SCA/DAST/container scanning platforms

---

**Milestone 4.2: SAST Tuning and False Positive Reduction** (Month 10–11)
- Conduct SAST false positive analysis across all teams
- Tune rulesets to eliminate the highest-frequency false positives
- Develop organization-specific custom rules for internal vulnerability patterns
- Target: SAST false positive rate < 20%

**Deliverables:**
- SAST false positive rate measured and documented
- Custom ruleset developed and deployed
- Developer satisfaction survey showing improved perception of SAST tools

---

**Milestone 4.3: Compliance Automation** (Month 11–12)
- Map all DevSecOps controls to applicable compliance frameworks (SOC 2, PCI-DSS, ISO 27001)
- Implement automated compliance evidence collection
- Conduct internal compliance pre-audit
- Identify and remediate compliance gaps

**Deliverables:**
- Compliance control mapping document published
- Automated evidence collection operational
- Internal pre-audit completed with findings remediated

**Q4 KPIs:**
- SAST false positive rate: < 20%
- Vulnerability SLA compliance (Critical): > 95%
- Vulnerability SLA compliance (High): > 85%
- Security KPI dashboard: operational and reviewed monthly
- Compliance readiness: > 90% of controls implemented

---

### Q5–Q6 (Months 13–18): Optimization

**Theme:** Optimize, advance maturity, and continuously improve

**Milestone 5.1: Advanced Supply Chain Security** (Months 13–14)
- Implement SLSA Level 2+ provenance for all artifacts
- Deploy SBOM vulnerability monitoring (Dependency-Track or similar)
- Implement automated new CVE → SBOM impact analysis
- Evaluate private package mirroring for high-risk dependencies

**Milestone 5.2: Advanced Threat Detection** (Months 14–15)
- Implement behavioral anomaly detection for CI/CD activity
- Deploy advanced SIEM detection rules for pipeline-specific threats
- Conduct tabletop exercise simulating a CI/CD supply chain attack
- Tune anomaly detection to reduce false positive rate to < 10%

**Milestone 5.3: Penetration Testing Program** (Months 15–16)
- Conduct comprehensive penetration test of CI/CD infrastructure
- Remediate all Critical/High findings from penetration test
- Establish annual penetration testing cadence
- Evaluate bug bounty program as complement to penetration testing

**Milestone 5.4: Level 5 Maturity Validation** (Months 16–18)
- Conduct third-party DevSecOps maturity assessment
- Benchmark against industry peers (BSIMM or OSAMM assessment)
- Publish internal DevSecOps maturity roadmap for next 12 months
- Celebrate and recognize team achievements in DevSecOps adoption

---

## KPIs per Maturity Stage

| KPI | Level 1 | Level 2 | Level 3 | Level 4 | Level 5 |
|---|---|---|---|---|---|
| MTTD (Critical vuln) | Weeks–months | Days–weeks | 24–48 hours | < 24 hours | < 4 hours |
| MTTR (Critical vuln) | Weeks | 7–14 days | 3–7 days | < 48 hours | < 24 hours |
| Vuln SLA compliance (High) | < 30% | 50–60% | 70–80% | > 90% | > 98% |
| Security gate pass rate | N/A | 50–60% | 75–85% | > 90% | > 97% |
| SAST false positive rate | N/A | > 40% | 20–30% | 10–20% | < 10% |
| % artifacts with SBOM | 0% | < 50% | 80–90% | 100% | 100% (+monitor) |
| % images signed | 0% | < 30% | 70–80% | 100% | 100% (+verify) |
| Secret scan coverage | 0% | 50–70% | 90–95% | 100% | 100% |
| Security training completion | < 20% | 40–50% | 70–80% | > 90% | > 95% |
| % teams with champion | 0% | 20–30% | 60–70% | > 80% | 100% |

---

## Organizational Change Management

### The Three Waves of Resistance

Every DevSecOps transformation encounters resistance in three waves. Anticipating and planning for these waves dramatically increases the probability of success.

**Wave 1: "This will slow us down"** (occurs in Q1–Q2)

This resistance comes primarily from developers and engineering managers who fear that security gates will block deployments and reduce velocity. Address it by:
- Demonstrating that Phase 1 controls are visibility-only and non-blocking
- Showing data: time spent on security gates vs. time spent remediating production incidents
- Celebrating early wins: "Our pipeline caught a hardcoded API key before it went public"
- Making security tooling fast: gate overhead should be < 5 minutes on a typical PR

**Wave 2: "These tools are wrong all the time"** (occurs in Q2–Q3)

This resistance comes from developers who encounter high false positive rates from SAST tools and conclude that the tools are useless. Address it by:
- Investing in SAST tuning as a first-class engineering priority
- Creating a clear process for developers to flag false positives
- Acting on false positive reports quickly (within 48 hours for High-frequency patterns)
- Tracking and publishing false positive rate metrics to demonstrate improvement

**Wave 3: "We're different, this doesn't apply to us"** (occurs throughout)

This resistance comes from specific teams claiming their application, technology, or deployment pattern doesn't fit the framework. Address it by:
- Engaging with the team to understand their specific context
- Showing how the principles apply even if the specific tools need adaptation
- Involving the team in designing the controls for their context
- Maintaining firm minimums (secrets, Critical CVEs) while allowing flexibility in implementation

### Change Management Activities by Quarter

| Quarter | Change Activity | Owner |
|---|---|---|
| Q1 | Executive communication on DevSecOps program objectives | CISO + VP Engineering |
| Q1 | Developer awareness campaign: why DevSecOps matters | Security Team |
| Q1–Q2 | Security training rollout (Level 1 — All engineers) | Security Team + L&D |
| Q2 | Security Champions identification and initial communication | Engineering Managers |
| Q3 | Security Champions training program delivery | Security Team |
| Q3 | Recognition program: celebrate security achievements | Engineering Leadership |
| Q4 | Security OKR integration into engineering planning cycle | VP Engineering |
| Q5 | Mature program celebration: share metrics, recognize contributors | CISO + VP Engineering |

---

## Technology Adoption Sequence

Technology must be introduced in a sequence that manages complexity and builds on each preceding capability. The following sequence is recommended:

```
QUARTER 1:
 Secret Scanning → SCA (report mode) → SAST (report mode) → Vuln Mgmt Platform

QUARTER 2:
 SAST Gates → SCA Gates → IaC Scanning → Container Scanning → SBOM
                ↓
           Vault / Secrets Manager → OIDC Federation
                ↓
           DAST (staging)

QUARTER 3:
 Ephemeral Runners → Network Egress Controls → Artifact Signing → Admission Control
                ↓
          Runtime Security (Falco) → CSPM

QUARTER 4:
 Security KPI Dashboard → SAST Tuning → Compliance Automation

QUARTER 5-6:
 SLSA Provenance → SBOM Monitoring → Behavioral Anomaly Detection → Pen Test Program
```

**Rationale for sequencing:**
- Visibility before enforcement — establish baselines before blocking to avoid delivery disruption
- People before advanced tooling — Security Champions program must be running before deploying tools that require human interpretation
- Foundation before advanced controls — ephemeral runners and admission control require mature pipeline infrastructure to support
- OIDC before Vault migration — eliminating long-lived cloud credentials reduces blast radius during the transition to centralized secrets management

---

## Risk Mitigation During Transition

### Risk 1: Security Gate Disrupts Delivery at High-Priority Release

**Probability:** High (especially in Q2 when gates activate)
**Impact:** Medium — causes short-term velocity reduction and organizational friction

**Mitigation:**
- Maintain a documented emergency exception process with clear criteria (production outage, critical security fix)
- Emergency exceptions require sign-off from a defined approval chain (Security Lead + Engineering VP)
- All emergency exceptions are reviewed in the next security steering committee meeting
- Monitor for exception abuse; > 3 exceptions per team per quarter triggers a process review

### Risk 2: SAST Tool Generates High False Positive Rate, Adoption Collapses

**Probability:** Medium — common with unconfigured enterprise SAST tools
**Impact:** High — undermines trust in the entire DevSecOps program

**Mitigation:**
- Budget 2–4 weeks per language for SAST configuration and tuning before activating gates
- Set a false positive rate SLO of < 25% during Q2, < 20% during Q3, < 15% ongoing
- Establish a rapid response process for false positive reports (48-hour SLA)
- Use incremental/differential scanning to limit findings to PR-introduced issues

### Risk 3: Key Security Champion Leaves Organization

**Probability:** Medium — champions are typically high performers who are recruited aggressively
**Impact:** Low-Medium — team loses security advocate; capability degraded temporarily

**Mitigation:**
- Ensure Security Champion role is not single-threaded: identify backup champions for each team
- Document security knowledge in team-accessible wikis rather than in individuals
- Include security champion context in team knowledge transfer during offboarding
- Recruit replacement champions rapidly; set 30-day target for replacement identification

### Risk 4: Significant Security Incident During Transition Period

**Probability:** Low-Medium — transition period may have temporary security gaps as new controls are deployed
**Impact:** Very High — could undermine the DevSecOps program politically and reputationally

**Mitigation:**
- Prioritize Q1 critical risk remediation — the most impactful controls against common attacks deploy first
- Maintain existing security controls in parallel during transition; do not disable old controls before new ones are verified operational
- Conduct quarterly threat assessment to identify and prioritize the highest-risk gaps
- Ensure incident response playbooks are current throughout the transition; conduct tabletop exercises quarterly
