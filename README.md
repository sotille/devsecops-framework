# DevSecOps Framework

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)](https://github.com/techstream/devsecops-framework)
[![Documentation](https://img.shields.io/badge/docs-comprehensive-brightgreen.svg)](docs/)
[![Maintained](https://img.shields.io/badge/Maintained-yes-green.svg)](https://github.com/techstream/devsecops-framework)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

A comprehensive DevSecOps framework providing battle-tested principles, reference architectures, security controls, and implementation guidelines for integrating security seamlessly into modern software delivery pipelines across cloud-native and enterprise environments.

---

## Overview

The Techstream DevSecOps Framework consolidates industry best practices from NIST, OWASP, CIS, and CISA into an opinionated, actionable guide for engineering teams at every stage of their security maturity journey. Whether you are starting from a traditional waterfall security model or accelerating an existing DevOps program, this framework provides the structure, toolchain guidance, and governance model to embed security as a first-class engineering discipline.

Security cannot be bolted on at the end of the software delivery lifecycle. This framework operationalizes the principle of "shift-left security" — moving security testing, validation, and policy enforcement as early as possible in the development process — without sacrificing developer velocity.

---

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Documentation](#documentation)
  - [Introduction](docs/introduction.md)
  - [Architecture](docs/architecture.md)
  - [Framework](docs/framework.md)
  - [Implementation](docs/implementation.md)
  - [Developer Experience](docs/developer-experience.md)
  - [Best Practices](docs/best-practices.md)
  - [Roadmap](docs/roadmap.md)
- [Repository Structure](#repository-structure)
- [Who Should Use This Framework](#who-should-use-this-framework)
- [Contributing](#contributing)
- [License](#license)

---

## Quick Start

### Prerequisites

Before adopting this framework, ensure your organization has:

- A version-controlled source code repository (Git-based)
- A CI/CD pipeline (GitHub Actions, GitLab CI, Jenkins, CircleCI, or equivalent)
- Basic container infrastructure or cloud deployment capability
- Executive sponsorship for the DevSecOps transformation initiative

### Recommended Reading Order

For teams new to DevSecOps, read the documentation in this order:

1. **[Introduction](docs/introduction.md)** — Understand the DevSecOps philosophy, history, and business drivers. Gain the vocabulary and conceptual foundation needed to communicate effectively across teams.

2. **[Architecture](docs/architecture.md)** — Study the reference architecture diagrams and toolchain layer model. Understand how people, process, and technology interact in a mature DevSecOps organization.

3. **[Framework](docs/framework.md)** — Deep-dive into the DevSecOps lifecycle, security controls per phase, roles and responsibilities, and the full toolchain reference. This is the core technical document.

4. **[Implementation](docs/implementation.md)** — Follow the phased implementation guide to introduce security controls incrementally without overwhelming your engineering teams.

5. **[Best Practices](docs/best-practices.md)** — Reference this document continuously. Thirty-plus detailed best practices organized by domain provide day-to-day operational guidance.

6. **[Roadmap](docs/roadmap.md)** — Plan your 18-month transformation journey using the maturity model, quarterly milestones, and organizational change management guidance.

### Immediate Actions

```bash
# Clone this framework locally
git clone https://github.com/techstream/devsecops-framework.git
cd devsecops-framework

# Review your current maturity level against the framework
# See docs/roadmap.md — Maturity Model section for a self-assessment checklist

# Start with quick wins described in the implementation guide
# Phase 1 (Weeks 1-4) delivers measurable security improvements with minimal disruption
```

---

## Documentation

| Document | Description | Audience |
|---|---|---|
| [Introduction](docs/introduction.md) | History, philosophy, business case, terminology | All stakeholders |
| [Architecture](docs/architecture.md) | Reference architecture, toolchain layers, diagrams | Architects, Platform Engineers |
| [Framework](docs/framework.md) | Lifecycle, controls, roles, toolchain reference | Security, DevOps, Engineering leads |
| [AI Security](docs/ai-security.md) | Security controls for AI-assisted development, LLMs in pipelines, and AI-powered applications | Security Engineers, Architects, Developers |
| [Secret Lifecycle Management](docs/secret-lifecycle-management.md) | Secret classification, provisioning patterns, automated rotation architecture, emergency revocation, and compliance evidence | Security Engineers, Platform Engineers |
| [Implementation](docs/implementation.md) | Phased rollout guide, prerequisites, metrics | DevOps leads, Platform Engineers |
| [Brownfield Adoption Guide](docs/brownfield-adoption-guide.md) | Adopting DevSecOps in existing environments: assessment, phased pipeline instrumentation, delta-based gating for legacy codebases, secrets in git history, and legacy pipeline migration | Security Engineers, DevOps leads |
| [Best Practices](docs/best-practices.md) | 30+ domain-specific best practices | All engineering roles |
| [Roadmap](docs/roadmap.md) | 18-month roadmap, maturity model, KPIs | Leadership, Program Managers |

---

## Repository Structure

```
devsecops-framework/
├── README.md                   # This file
├── LICENSE                     # Apache 2.0 license
└── docs/
    ├── introduction.md         # What is DevSecOps, history, terminology
    ├── architecture.md         # Reference architecture and toolchain layers
    ├── framework.md            # Core framework: lifecycle, controls, roles
    ├── implementation.md       # Phased implementation guide
    ├── best-practices.md       # 30+ best practices by category
    └── roadmap.md              # 18-month roadmap and maturity model
```

---

## Who Should Use This Framework

**Security Engineers and Architects** will find the architecture and framework documents invaluable for designing security controls and integrating them into existing pipelines. The toolchain reference provides a curated, technology-agnostic catalog of security tools organized by pipeline phase.

**DevOps and Platform Engineers** will benefit from the implementation guide's step-by-step instructions for configuring security gates, hardening CI/CD infrastructure, and integrating scanning tools without disrupting existing delivery workflows.

**Engineering Leaders and CISOs** will find the roadmap and implementation documents useful for planning, budgeting, and communicating the DevSecOps transformation to senior stakeholders. The KPI and metrics frameworks provide the data needed to demonstrate progress and ROI.

**Developers** will use the best-practices document and framework reference daily as a guide for writing secure code, managing secrets, and understanding what security gates their commits must pass before production.

**Risk and Compliance Teams** will find the framework's mapping to NIST CSF, OWASP Top 10, CIS Controls, and SOC 2 requirements useful for audit and governance purposes.

---

## Learning Resources

The Techstream Book Series and hands-on lab companion extend the concepts in this framework with structured learning, exercises, and guided assessments.

- **[Book 1: DevSecOps — Foundations & Transformation](https://www.techstream.app/learn)** — The book volume aligned with this framework. Covers the DevSecOps philosophy, the TDMM maturity model, organizational transformation methodology, and DORA-aligned metrics.
- **[Hands-On Labs (techstream-learn/book-1-foundations/)](https://www.techstream.app/learn)** — Practical exercises including TDMM self-assessments, transformation roadmap building, and security champion program design.
- **[Book Series Overview (VOLUMES.md)](../techstream-books/VOLUMES.md)** — Index of all four Techstream volumes covering CI/CD security, cloud security, and release governance.
- **[Techstream Platform](https://www.techstream.app)** — The central portal for all Techstream frameworks, documentation, and learning resources.

---

## Contributing

We welcome contributions from the community. The Techstream DevSecOps Framework is a living document that evolves with the threat landscape, tooling ecosystem, and organizational learnings.

### How to Contribute

1. **Fork the repository** and create a feature branch from `main`.
2. **Make your changes**, ensuring all documentation follows the existing Markdown style and depth of coverage.
3. **Test your Markdown** for proper rendering (headings, tables, code blocks, and Mermaid diagrams).
4. **Open a pull request** with a clear description of what you changed and why.

### Contribution Guidelines

- All new best practices must include a rationale and implementation guidance section.
- Architecture changes must include updated diagrams (Mermaid or ASCII).
- Changes to the framework lifecycle must be reflected in both `framework.md` and `implementation.md`.
- Do not introduce vendor-specific guidance without also providing a vendor-neutral alternative.
- Maintain professional, concise prose. Avoid marketing language and unsubstantiated claims.

### Reporting Issues

If you find errors, outdated information, or missing coverage, please open a GitHub Issue with:
- The document and section affected
- A description of the problem
- A suggested correction if you have one

---

## License

Copyright 2024 Techstream

Licensed under the Apache License, Version 2.0. See the [LICENSE](LICENSE) file for the full license text.

You may use, modify, and distribute this framework freely under the terms of the Apache 2.0 License. Attribution to Techstream is appreciated but not required for internal use.
