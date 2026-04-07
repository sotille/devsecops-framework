# Brownfield DevSecOps Adoption Guide

Most DevSecOps transformations do not start from a blank slate. The majority of organizations adopting DevSecOps have existing systems, pipelines, teams, and technical debt that cannot be paused for a full rebuild. This guide addresses the specific challenges of adopting DevSecOps practices in brownfield environments — where applications are already in production, pipelines already exist (or are absent), and security work must happen alongside ongoing feature delivery.

---

## Defining the Brownfield Problem

A brownfield environment has one or more of the following characteristics:

- **Existing applications in production** with no test coverage, undocumented architecture, or security debt accumulated over years
- **Legacy CI/CD pipelines** built on older tooling (Jenkins 1.x, TeamCity, Bamboo, manual scripts) that are difficult to modify
- **No current pipeline security scanning** — no SAST, SCA, or container scanning in the delivery workflow
- **Monolithic architectures** that are difficult to scan, test, or deploy independently
- **Accumulated vulnerability debt** — many known and unknown vulnerabilities in production code
- **Cultural inertia** — teams that view security as overhead or that have failed security initiatives before

Greenfield guidance assumes you can set standards before code is written. Brownfield guidance assumes the code already exists and the challenge is improving it without breaking delivery.

---

## Brownfield Anti-Patterns

### Anti-Pattern 1: Boiling the Ocean

Attempting to bring all applications to DevSecOps Level 3 simultaneously. This creates widespread pipeline failures, developer frustration, and no prioritization of actual risk.

**Correct approach:** Tier applications by risk and start with 2-3 high-value targets where success can be demonstrated.

### Anti-Pattern 2: Break-Build from Day One

Deploying SAST with break-build enforcement on a legacy codebase with thousands of existing findings. The first pipeline run fails every build and security is immediately seen as blocking delivery.

**Correct approach:** Start in report-only mode. Establish a baseline. Enforce only on new findings in the first 90 days.

### Anti-Pattern 3: Full Security Debt Remediation Before Proceeding

Blocking all new feature work until all existing vulnerabilities are remediated. This is politically untenable and prioritizes old vulnerabilities over business delivery.

**Correct approach:** Separate the debt queue (existing findings, time-boxed SLA) from the gates queue (new findings, break-build). Work both in parallel.

### Anti-Pattern 4: Replacing All Existing Tooling at Once

Replacing Jenkins, existing deployment scripts, and all monitoring simultaneously to implement "the right architecture." This creates instability and slows delivery for months.

**Correct approach:** Add security controls to existing pipelines as a layer. Migrate tooling incrementally, one pipeline at a time.

---

## Brownfield Assessment

Before planning a brownfield adoption, understand the existing state across five dimensions:

### 1. Application Inventory and Risk Tiering

Build an inventory and prioritize:

```markdown
| Tier | Criteria | Count | Examples |
|------|---------|-------|---------|
| Tier 1 | Customer-facing, processes payment/PII, in regulated scope | 3-10 | User API, payment service, auth service |
| Tier 2 | Internal business-critical, not customer-facing | 10-30 | Order management, inventory, reporting |
| Tier 3 | Internal tooling, low-risk, non-customer data | >30 | Admin scripts, internal dashboards |
```

Start brownfield DevSecOps with Tier 1 applications. Results there justify expansion to Tier 2 and Tier 3.

### 2. Current Pipeline State Audit

| Dimension | Questions |
|-----------|----------|
| Pipeline existence | Does a CI/CD pipeline exist? Is it automated? |
| Security tools | Is any SAST, SCA, secrets scanning currently running? |
| Test coverage | What is the current test coverage? Are any security tests present? |
| Deployment process | Is deployment automated? Manual approvals? |
| Secret management | Where are secrets stored? Are any hardcoded? |

### 3. Technical Debt Assessment

Run existing code through a baseline scan without enforcement to understand the debt queue:

```bash
# Baseline SAST scan — report only, no enforcement
semgrep scan \
  --config auto \
  --json \
  --output baseline-sast.json \
  ./src

# Summarize findings by severity and rule category
jq '[.results | group_by(.extra.severity) |
     .[] | {severity: .[0].extra.severity, count: length}]' \
  baseline-sast.json

# Baseline SCA scan
trivy fs \
  --format json \
  --output baseline-sca.json \
  .

# Secrets scan
gitleaks detect \
  --source . \
  --report-path baseline-secrets.json \
  --report-format json \
  --no-git  # Scan working directory, not git history (for initial baseline)
```

**Interpreting baseline results:**

| Scenario | Recommended Approach |
|----------|---------------------|
| 0-50 findings | Enforce break-build immediately for HIGH/CRITICAL |
| 50-500 findings | 60-day remediation sprint for HIGH/CRITICAL + new-findings enforcement |
| 500+ findings | Risk-stratified remediation: fix exploitable findings first; gate new findings only |
| Secrets in codebase | Immediate incident response — treat as active secret exposure |

### 4. Team Readiness Assessment

| Factor | Indicator of Readiness |
|--------|----------------------|
| Security champion presence | Does any engineer have security knowledge or interest? |
| Recent security incidents | Teams that have suffered incidents are often more receptive |
| Leadership support | Is CISO or VP Engineering visibly backing the initiative? |
| Capacity for change | Is the team in a feature freeze or heavy sprint? |

### 5. Dependency and Architecture Assessment

```bash
# Generate a dependency graph to understand the blast radius of changes
# For Node.js projects
npx depcruise --output-type dot src | dot -T svg > dependency-graph.svg

# For Python projects
pipdeptree --graph-output svg > dependency-graph.svg

# Identify circular dependencies and outdated packages that may require refactoring
```

---

## Phased Brownfield Adoption

### Phase 1: Instrument Without Blocking (Weeks 1-4)

**Goal:** Add security scanning to all Tier 1 pipelines in report-only mode. Establish baselines. Fix secrets.

**Actions:**
1. Add SAST, SCA, and secrets scanning to existing pipelines — `continue-on-error: true`
2. Fix all secrets findings immediately (treat as active incidents)
3. Establish the baseline finding count per tool per repository
4. Set up a findings dashboard visible to developers and security team
5. Identify the top 10 highest-risk findings by exploitability

```yaml
# Jenkins: Add security scanning stage in report-only mode
stage('Security Scan (Report Only)') {
    steps {
        sh '''
            semgrep scan --config auto --json --output sast-report.json . || true
            trivy fs --format json --output sca-report.json . || true
            gitleaks detect --report-path secrets-report.json --report-format json . || true
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: '*-report.json', allowEmptyArchive: true
        }
    }
}
```

**Exit criteria for Phase 1:**
- SAST, SCA, secrets scanning running in all Tier 1 pipelines
- Zero secrets in any Tier 1 repository (finding or confirmed rotation)
- Baseline finding counts documented per repository
- Top 10 exploitable findings remediated

---

### Phase 2: New-Findings Gate (Weeks 5-12)

**Goal:** Prevent new security debt from being introduced while existing debt is managed in parallel.

**New-findings baseline enforcement (delta-based gating):**

The key insight for brownfield adoption: you cannot block builds based on the total finding count (the baseline has too many findings). You can block based on the *change* in finding count — new findings introduced by the current PR.

```yaml
# GitHub Actions: New-findings gate
- name: SAST Scan — New Findings Gate
  run: |
    # Scan current branch
    semgrep scan --config auto --json --output current-scan.json .

    # Scan base branch (main)
    git stash
    git checkout origin/main
    semgrep scan --config auto --json --output baseline-scan.json .
    git checkout -

    # Compare: only fail on new findings not present in baseline
    python3 scripts/delta_findings.py \
      --current current-scan.json \
      --baseline baseline-scan.json \
      --fail-on-new HIGH,CRITICAL \
      --output delta-report.json

- name: Upload Scan Results
  uses: actions/upload-artifact@v4
  with:
    name: security-scan-results
    path: '*-scan.json'
```

```python
# scripts/delta_findings.py — simplified delta logic
import json
import sys

def finding_fingerprint(f: dict) -> str:
    """Create a unique key for a finding based on location and rule."""
    return f"{f['check_id']}:{f['path']}:{f['start']['line']}"

def find_new_findings(current: list, baseline: list, fail_severities: set) -> list:
    baseline_fps = {finding_fingerprint(f) for f in baseline}
    new_findings = [
        f for f in current
        if finding_fingerprint(f) not in baseline_fps
        and f.get('extra', {}).get('severity', '').upper() in fail_severities
    ]
    return new_findings

if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument('--current', required=True)
    parser.add_argument('--baseline', required=True)
    parser.add_argument('--fail-on-new', default='HIGH,CRITICAL')
    parser.add_argument('--output', required=True)
    args = parser.parse_args()

    with open(args.current) as f:
        current = json.load(f)["results"]
    with open(args.baseline) as f:
        baseline = json.load(f)["results"]

    fail_severities = set(args.fail_on_new.upper().split(','))
    new_findings = find_new_findings(current, baseline, fail_severities)

    with open(args.output, 'w') as f:
        json.dump({"new_findings": new_findings, "count": len(new_findings)}, f, indent=2)

    if new_findings:
        print(f"FAIL: {len(new_findings)} new {args.fail_on_new} findings introduced.")
        for finding in new_findings:
            print(f"  {finding['check_id']} in {finding['path']}:{finding['start']['line']}")
        sys.exit(1)

    print("PASS: No new HIGH/CRITICAL findings introduced.")
```

**Debt remediation track:**

Run a parallel remediation track for existing findings:

```markdown
## Security Debt Backlog Management

Organize existing findings by remediation priority:

### Priority 1 (Fix in current sprint):
- Authentication bypass findings
- SQL injection findings
- XXE/XXS injection findings in user-facing endpoints
- Findings with public exploit available

### Priority 2 (Fix in next 2 sprints):
- All remaining HIGH severity findings in Tier 1 applications
- SCA: all CRITICAL CVEs in direct dependencies

### Priority 3 (Fix within 90 days):
- All HIGH severity findings in Tier 2 applications
- CRITICAL CVEs in transitive dependencies

### Priority 4 (Fix within 6 months):
- MEDIUM severity findings
- Remaining SCA findings
```

**Exit criteria for Phase 2:**
- New HIGH/CRITICAL findings blocked in all Tier 1 pipelines
- Priority 1 debt remediated
- Debt queue tracked in vulnerability management platform
- Security scanning running in all Tier 2 pipelines (report only)

---

### Phase 3: Full Enforcement and Tier 2 Expansion (Weeks 13-26)

**Goal:** Enforce full security gate suite on Tier 1 applications; bring Tier 2 to Phase 2 state.

**Transition from new-findings to total-findings enforcement:**

Once the baseline finding count is zero or near-zero for a repository, transition from delta-based to total-count enforcement:

```yaml
# GitHub Actions: Full enforcement (when baseline is clean)
- name: SAST — Full Enforcement
  run: |
    semgrep scan --config auto --error --severity ERROR .
    # --error exits non-zero on any ERROR-severity finding
    # Deploy when your baseline is zero HIGH/CRITICAL findings
```

**Tier 2 pipeline instrumentation:**

Apply the Phase 1 template to all Tier 2 pipelines. The lessons from Tier 1 (tuned rulesets, false positive lists, developer guidance) reduce the effort for Tier 2.

**Exit criteria for Phase 3:**
- All Tier 1 applications on full enforcement (total findings, not delta)
- Priority 2 debt remediated for Tier 1
- All Tier 2 pipelines running scanning in report-only mode
- Secret management: all Tier 1 credentials in vault

---

### Phase 4: Sustained Maturity (Month 6+)

**Goal:** Maintain the gains, extend to Tier 3, and advance to higher maturity levels.

- Extend full enforcement to Tier 2 applications
- Add container scanning and IaC scanning to all pipelines
- Implement supply chain controls (artifact signing, SBOM generation)
- Run first DevSecOps maturity assessment to measure progress
- Set roadmap for Level 3 capabilities (threat modeling, DAST automation, SOAR)

---

## Secrets in Legacy Codebases

Secrets committed to source code repositories require immediate treatment. The git history persists even after the secret is removed from the current working tree.

### Detection

```bash
# Scan full git history for secrets
gitleaks detect \
  --source . \
  --report-path git-history-secrets.json \
  --report-format json \
  --log-opts "HEAD"  # Scan all commits reachable from HEAD

# TruffleHog — scan git history for verified secrets
trufflehog git file://. --json > trufflehog-results.json
```

### Remediation Decision Matrix

| Scenario | Action |
|----------|--------|
| Secret committed, not yet pushed | Rewrite history (git filter-branch or BFG Repo Cleaner) |
| Secret pushed to private repo, not widely cloned | Rotate secret immediately; optionally rewrite history |
| Secret pushed to public repo (or was public) | Rotate immediately; assume secret is compromised; investigate access logs |
| Secret is ancient and rotated | Document as known finding; skip history rewrite if impractical |

```bash
# BFG Repo Cleaner — remove secrets from git history
# (Run after rotating the actual secret)
bfg --replace-text secrets-patterns.txt my-repo.git
cd my-repo
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force  # Coordinate with all team members to re-clone
```

---

## Legacy Pipeline Migration

Migrating from an older CI/CD platform (Jenkins, Bamboo, TeamCity) to a modern one provides an opportunity to embed security from the start.

**Migration strategy:**

1. **Do not migrate pipelines verbatim.** Legacy pipelines typically have no security controls. Migrating them without adding security controls perpetuates the problem.

2. **Use the secure pipeline templates** from the [Secure Pipeline Templates framework](../../secure-pipeline-templates/README.md) as the target state for all migrated pipelines.

3. **Migrate application by application**, starting with Tier 1. Each migration is an opportunity to add security gates.

4. **Dual-run during transition**: Run the old pipeline and new pipeline in parallel for one sprint to validate equivalence before switching over.

---

## Related Techstream Resources

| Topic | Document |
|-------|---------|
| Core DevSecOps principles | [DevSecOps Framework](framework.md) |
| Phased implementation guidance | [Implementation Guide](implementation.md) |
| Pipeline templates for new pipelines | [Secure Pipeline Templates](../../secure-pipeline-templates/README.md) |
| Transformation methodology | [DevSecOps Methodology](../../devsecops-methodology/README.md) |
| Maturity assessment | [DevSecOps Maturity Model](../../devsecops-maturity-model/README.md) |
| Legacy CI/CD migration | [Legacy CI/CD Migration](../../secure-ci-cd-reference-architecture/docs/legacy-cicd-migration.md) |

*Part of the Techstream DevSecOps Framework. Licensed under Apache 2.0.*
