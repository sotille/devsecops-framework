# Federal-Standards Alignment Matrix

This document maps the eight-phase DevSecOps lifecycle defined in this framework to the corresponding sections of major United States federal cybersecurity standards and guidance.

## Alignment table

| Lifecycle Phase | NIST SP 800-218 (SSDF) | EO 14028 Section | DoD DevSecOps Fundamentals v2.5 | CISA Secure-by-Design |
|---|---|---|---|---|
| 1. Plan & Threat Model | PO.1, PO.2 | §4(b) | Plan Phase | Threat-informed Design |
| 2. Develop & Secure | PS.1, PS.2, PS.3 | §4(e)(i) | Develop Phase | Memory Safety; Default Strong Authn |
| 3. Build & SBOM | PS.3, PW.4 | §4(e)(vii) (SBOM) | Build Phase | Software Bill of Materials |
| 4. Test (SAST/DAST/SCA) | PW.7, PW.8 | §4(e)(iv) | Test Phase | Vulnerability Disclosure |
| 5. Release & Sign | PW.4 | §4(e)(iii), §4(e)(viii) | Release Phase | Provenance |
| 6. Deploy & Verify | RV.1, RV.3 | §4(e)(ix) | Deploy Phase | Default-Secure Configuration |
| 7. Operate & Monitor | RV.1, RV.2 | §4(e)(ix) | Operate Phase | Continuous Monitoring |
| 8. Respond & Recover | RV.3 | §4(f) | Monitor Phase | Coordinated Disclosure |

## Cross-references

- **EO 14306 (June 2025):** extends EO 14028 mandates and reinforces the consortium model under which NIST NCCoE produces operational guidance.
- **NIST SP 1800-44** (preliminary draft, March 2026): operational reference implementation guide for SSDF in CI/CD environments. This framework is aligned with the practices in the draft.
- **NIST CSF 2.0:** the lifecycle phases map to the CSF Functions (Identify, Protect, Detect, Respond, Recover) at the program level.

## How to use this document

When responding to compliance questions, RFPs, or security audits referencing any of the federal standards above, refer to the corresponding lifecycle phase in this framework. Each phase has detailed implementation guidance in `docs/`.

## Maintenance

This matrix is updated when major revisions to federal guidance are published. See `CHANGELOG.md` for revision history.
