# API Security Integration Guide

APIs are primary attack surfaces in modern software systems: they expose business logic, handle authentication tokens, process untrusted inputs, and bridge internal services. This guide integrates API security into the DevSecOps pipeline, covering threat modeling, automated testing, runtime protection, and governance. It is scoped to REST and GraphQL APIs; gRPC considerations are noted where they differ significantly.

---

## Why APIs Require Dedicated DevSecOps Controls

Standard application security controls — SAST, SCA, container scanning — address code and dependency risks but do not fully cover API-specific threats:

- **Business logic abuse** is not detectable by syntax analysis. An attacker who authenticates legitimately and calls an endpoint in an unintended sequence (e.g., price manipulation, BOLA — Broken Object Level Authorization) bypasses SAST entirely.
- **Schema drift** — where the deployed API behavior diverges from its specification — creates gaps that neither developers nor security teams are aware of.
- **API sprawl** — undocumented endpoints, shadow APIs, deprecated but still-live routes — cannot be protected if they are not known.

The OWASP API Security Top 10 (2023) frames these risks. This guide provides controls for each category, integrated into the DevSecOps pipeline.

---

## OWASP API Security Top 10 (2023) — Control Mapping

| Risk | Shift-Left Control | Pipeline Control | Runtime Control |
|------|--------------------|------------------|-----------------|
| API1: Broken Object Level Authorization (BOLA) | Threat model + code review checklist | Integration test suite (authorization matrix) | API gateway rate limiting + anomaly detection |
| API2: Broken Authentication | Auth library selection standard; OIDC/JWT best practices doc | DAST authentication probe | Token validation middleware; short expiry enforcement |
| API3: Broken Object Property Level Authorization | Code review; field-level authorization testing | Fuzzing + schema validation tests | Request/response logging; anomaly detection |
| API4: Unrestricted Resource Consumption | Rate limit design review | Load and spike tests in staging | Gateway throttling; quota enforcement |
| API5: Broken Function Level Authorization | Threat model; privilege separation design | Automated authorization matrix test | RBAC enforcement at gateway; admin endpoint audit |
| API6: Unrestricted Access to Sensitive Business Flows | Business logic threat model | End-to-end integration tests; abuse case tests | Behavioral analytics; bot detection |
| API7: Server-Side Request Forgery (SSRF) | Secure coding checklist | DAST SSRF probe | Egress filtering; metadata service blocking |
| API8: Security Misconfiguration | Linting; OpenAPI schema review | Configuration scanning (headers, CORS, TLS) | Runtime header enforcement; TLS policy |
| API9: Improper Inventory Management | API catalog requirement; discovery scans | API changelog enforcement | Traffic-based shadow API detection |
| API10: Unsafe Consumption of APIs | Third-party API risk assessment | Dependency scanning for API client libraries | Mutual TLS; response validation |

---

## Phase 1: Design (Shift-Left)

### API Threat Modeling

Every API that exposes sensitive data, performs write operations, or accepts unauthenticated input requires a threat model before implementation. Use the following checklist as a starting point:

**Authentication and authorization:**
- [ ] What identity provider issues tokens? Are tokens short-lived (< 1 hour for access tokens)?
- [ ] Is Object Level Authorization (BOLA) enforced at the data access layer, not just at the route handler?
- [ ] Does each endpoint enforce the minimum required scope? Is scope validation tested?
- [ ] Are admin and internal endpoints segregated from public API surfaces (separate route prefix, network policy)?

**Data handling:**
- [ ] What PII or sensitive data does the endpoint return? Is it the minimum necessary?
- [ ] Are response fields filtered per caller role (no over-fetching of sensitive fields)?
- [ ] Does the endpoint accept file uploads? What type and size validation is enforced?

**Rate limiting and resource consumption:**
- [ ] Does each endpoint have a rate limit appropriate to its expected usage pattern?
- [ ] Are expensive operations (search, aggregation, export) protected by stricter limits than cheap reads?
- [ ] Is there a maximum page size on paginated endpoints?

**External API consumption:**
- [ ] What third-party APIs does this service call? Are those calls validated and bounded?
- [ ] If a called URL is user-supplied, is SSRF mitigated (allowlist validation, no internal network reach)?

---

### OpenAPI Specification Requirement

All APIs must have an OpenAPI (v3.x) specification committed to the repository before the first deployment. The specification serves as the contract for:

- Automated schema validation in tests
- Fuzz testing input generation
- Gateway routing and validation configuration
- API documentation generation

Schema quality gates (enforced in CI):

```yaml
# Example: spectral linting in GitHub Actions
- name: Lint OpenAPI spec
  uses: stoplightio/spectral-action@latest
  with:
    file_glob: 'api/openapi.yaml'
    spectral_ruleset: '.spectral.yaml'
```

Minimum ruleset (`.spectral.yaml`):

```yaml
rules:
  oas3-schema: error
  operation-operationId: error
  operation-description: warn
  operation-tags: warn
  no-$ref-siblings: error
  typed-enum: error
  # Require security scheme on all operations
  operation-security-defined:
    severity: error
    given: "$.paths[*][get,post,put,patch,delete]"
    then:
      field: security
      function: defined
```

---

## Phase 2: Build (Pipeline Controls)

### SAST Rules for API Code

Configure SAST tools with API-specific rules in addition to standard rulesets:

**Semgrep rules to enable:**

```yaml
# Detect direct object reference without authorization check
rules:
  - id: bola-risk-missing-ownership-check
    patterns:
      - pattern: |
          def $FUNC(request, $ID, ...):
            $OBJ = $MODEL.objects.get(id=$ID)
            ...
      - pattern-not: |
          def $FUNC(request, $ID, ...):
            $OBJ = $MODEL.objects.get(id=$ID, user=request.user)
            ...
    message: "Potential BOLA: object fetched by ID without ownership check"
    severity: WARNING
    languages: [python]
```

**SAST checklist for API code reviews:**
- Input deserialization from request bodies — check for unsafe deserialization
- URL parameter handling — verify type coercion and bounds
- Error responses — confirm stack traces and internal paths are not exposed in production error messages
- CORS configuration — confirm `Access-Control-Allow-Origin` is not wildcard on authenticated endpoints
- Security headers — `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`

---

### DAST API Scanning

Dynamic API scanning must run against a deployed staging or review environment. Do not run aggressive DAST against production.

**Recommended toolchain:**

| Tool | Use Case | Integration Point |
|------|---------|-------------------|
| **OWASP ZAP** (API scan mode) | Broad vulnerability scanning using OpenAPI spec | CI/CD post-deploy stage |
| **Nuclei** (api templates) | CVE and misconfiguration probes | CI/CD post-deploy stage |
| **Schemathesis** | Property-based fuzzing from OpenAPI spec | CI/CD post-deploy stage |
| **42Crunch API Security Audit** | Spec-level security analysis (no deployment needed) | Pre-merge CI gate |

**Example: Schemathesis in GitHub Actions:**

```yaml
- name: API Fuzz Testing
  run: |
    pip install schemathesis
    st run https://staging.internal/openapi.json \
      --checks all \
      --stateful=links \
      --base-url https://staging.internal \
      --header "Authorization: Bearer ${{ secrets.STAGING_API_TOKEN }}" \
      --junit-xml test-results/schemathesis.xml
```

**Example: ZAP API scan:**

```yaml
- name: ZAP API Scan
  uses: zaproxy/action-api-scan@v0.9.0
  with:
    target: 'https://staging.internal/openapi.json'
    format: openapi
    fail_action: true
    cmd_options: '-a -j'
```

---

### Authorization Matrix Testing

BOLA (API1) and BFLA (API5) cannot be reliably detected by automated scanning alone. Implement an authorization matrix test suite:

```python
# Example: pytest authorization matrix for a resource API
import pytest
import requests

# Matrix: (role, endpoint, method, expected_status)
AUTH_MATRIX = [
    ("admin",    "/api/v1/users",          "GET",    200),
    ("admin",    "/api/v1/users/42",       "DELETE", 204),
    ("user",     "/api/v1/users",          "GET",    403),
    ("user",     "/api/v1/users/42",       "DELETE", 403),
    ("user",     "/api/v1/users/me",       "GET",    200),
    ("user",     "/api/v1/users/99",       "GET",    403),  # BOLA: user accessing another user's resource
    ("readonly", "/api/v1/reports",        "GET",    200),
    ("readonly", "/api/v1/reports",        "POST",   403),
    ("anon",     "/api/v1/health",         "GET",    200),
    ("anon",     "/api/v1/users",          "GET",    401),
]

@pytest.mark.parametrize("role,path,method,expected", AUTH_MATRIX)
def test_authorization_matrix(role, path, method, expected, api_tokens):
    token = api_tokens[role]
    headers = {"Authorization": f"Bearer {token}"} if token else {}
    resp = getattr(requests, method.lower())(f"{BASE_URL}{path}", headers=headers)
    assert resp.status_code == expected, \
        f"AUTHORIZATION FAILURE: {role} {method} {path} → {resp.status_code} (expected {expected})"
```

Run this suite in CI as a required quality gate before any PR can merge changes to API route handlers or authorization logic.

---

### API Schema Governance Gate

Enforce the following checks as CI gates:

```yaml
api-governance:
  stage: security
  script:
    # 1. OpenAPI spec must exist
    - test -f api/openapi.yaml || (echo "ERROR: OpenAPI spec missing" && exit 1)

    # 2. Spec must pass schema validation
    - spectral lint api/openapi.yaml --ruleset .spectral.yaml

    # 3. No endpoints added without security scheme
    - python scripts/check_api_security_coverage.py api/openapi.yaml

    # 4. Breaking changes require explicit approval
    - oasdiff breaking api/openapi.yaml.prev api/openapi.yaml --fail-on ERR
```

---

## Phase 3: Runtime Controls

### API Gateway Security Configuration

All APIs must be fronted by an API gateway or service mesh that enforces:

| Control | Configuration |
|---------|--------------|
| **Authentication** | JWT validation at gateway; reject requests without valid token (except public endpoints) |
| **Rate limiting** | Per-client and per-IP limits; global endpoint limits for expensive operations |
| **Request size limits** | Maximum body size (default: 1 MB; adjust per endpoint) |
| **TLS enforcement** | Minimum TLS 1.2; TLS 1.3 preferred; HSTS header |
| **Security headers** | `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Cache-Control: no-store` on authenticated responses |
| **CORS policy** | Explicit allowlist; no wildcard on credentialed requests |
| **Schema validation** | Request body validated against OpenAPI schema before forwarding to backend |

**Example: Kong Gateway rate limiting plugin:**

```yaml
plugins:
  - name: rate-limiting
    config:
      minute: 100          # per consumer per minute
      hour: 2000           # per consumer per hour
      policy: redis         # use Redis for distributed rate limit state
      fault_tolerant: false # block requests if Redis is unavailable
      hide_client_headers: false
```

**Example: Envoy CORS policy:**

```yaml
cors:
  allow_origin_string_match:
    - exact: "https://app.example.com"
    - exact: "https://admin.example.com"
  allow_methods: GET, POST, PUT, DELETE, OPTIONS
  allow_headers: Authorization, Content-Type, X-Request-ID
  allow_credentials: true
  max_age: "3600"
```

---

### Runtime API Discovery and Shadow API Detection

Shadow APIs — endpoints receiving production traffic that are not present in any known API specification — represent uncontrolled attack surface. Detect them by:

1. **Traffic analysis:** aggregate API access logs and compare routes receiving traffic against the registered OpenAPI specification.
2. **Gateway routing coverage:** any request that hits a catch-all route (404 → backend) and receives a 2xx is a candidate shadow API.
3. **Scheduled discovery scans:** run Nuclei or an internal discovery script against the production hostname monthly, using the OpenAPI spec as the baseline.

```python
# Shadow API detection: compare gateway traffic routes to OpenAPI spec
import yaml, json, re
from collections import Counter

with open("api/openapi.yaml") as f:
    spec = yaml.safe_load(f)

spec_paths = set(spec.get("paths", {}).keys())
# Convert path params: /users/{id} → regex
spec_patterns = [re.compile("^" + re.sub(r"\{[^}]+\}", r"[^/]+", p) + "$") for p in spec_paths]

# Load traffic routes from gateway access log aggregation
with open("traffic_routes.json") as f:
    traffic_routes = json.load(f)  # {"route": count, ...}

shadow_apis = []
for route in traffic_routes:
    if not any(p.match(route) for p in spec_patterns):
        shadow_apis.append({"route": route, "request_count": traffic_routes[route]})

if shadow_apis:
    print("SHADOW APIs DETECTED (not in OpenAPI spec):")
    for s in sorted(shadow_apis, key=lambda x: -x["request_count"]):
        print(f"  {s['route']}: {s['request_count']} requests")
```

---

### GraphQL-Specific Controls

GraphQL requires additional controls beyond standard REST API guidance:

| Risk | Control |
|------|---------|
| **Introspection in production** | Disable GraphQL introspection in production; enable only in development environments |
| **Query depth attacks** | Enforce maximum query depth (recommended: 5–10 levels) |
| **Query complexity attacks** | Implement query complexity analysis; reject queries exceeding a complexity budget |
| **Batching attacks** | Limit batch query size; apply per-query rate limiting |
| **Persistent queries** | Use persisted query allowlists in production (reject arbitrary queries) |
| **Field-level authorization** | Enforce authorization at resolver level; never rely solely on gateway-level auth |

```python
# Example: graphene-django depth and complexity limits
from graphql import build_ast_schema
from graphql_depth_limit import depth_limit_validator

schema = graphene.Schema(query=Query)
schema.execute(
    query,
    validation_rules=[depth_limit_validator(max_depth=7)]
)
```

---

## API Inventory and Governance

Maintaining an accurate API inventory is foundational to all other controls. Establish the following:

| Requirement | Implementation |
|-------------|----------------|
| **API registration** | All APIs must be registered in the API catalog before first deployment. Registration includes: service name, owner team, OpenAPI spec location, environment URLs, authentication scheme, data classification |
| **API changelog** | Breaking changes require a changelog entry and deprecation notice minimum 30 days before removal |
| **API data classification** | Each API tagged with highest sensitivity of data processed (Public / Internal / Confidential / Restricted) |
| **API retirement** | Deprecated APIs have an explicit sunset date; removed APIs are redirected to a documented tombstone response (410 Gone) |
| **Periodic review** | All registered APIs reviewed quarterly for accuracy; shadow API scans reconciled against inventory |

---

## Integration with the Techstream Framework

- **Threat modeling:** [DevSecOps Framework — Architecture](architecture.md) — integrate API threat models into the STRIDE process
- **Pipeline templates:** [Secure Pipeline Templates](../../secure-pipeline-templates/docs/framework.md) — add DAST and schema validation stages
- **Compliance mapping:** [Regulatory Controls Matrix](../../compliance-automation-framework/docs/regulatory-controls-matrix.md) — API security controls map to NIST 800-53 SC-8, AC-3, AC-4; SOC2 CC6.1, CC6.6; PCI DSS 6.4, 6.5
- **Supply chain:** [Supply Chain Security Framework](../../software-supply-chain-security-framework/docs/framework.md) — third-party API clients treated as supply chain dependencies
- **Maturity assessment:** [Assessment Scorecard](../../devsecops-maturity-model/docs/assessment-scorecard.md) — Domain 3: Application Security includes API security coverage

---

*Part of the Techstream DevSecOps Framework. Licensed under Apache 2.0.*
