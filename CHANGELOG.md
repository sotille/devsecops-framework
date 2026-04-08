# Changelog

All notable changes to the DevSecOps Framework are documented here.
Format: `[version] — [date] — [summary of changes]`

---

## [Unreleased]

- [2026-04-08] docs/best-practices.md: Added AI-Assisted Development Security section (BP-AI01–BP-AI04) covering (1) AI-suggested dependency verification pre-commit hooks and registry mirror enforcement for slopsquatting prevention; (2) AI code review as advisory-only with deterministic SAST as the blocking gate; (3) agent tool authorization policies, session-scoped credentials, and approval gates for high-consequence actions; (4) AI tool inventory process with four-question security review. Closes the gap between traditional DevSecOps best practices and AI-specific controls; cross-references ai-devsecops-framework for full implementation guidance

- [2026-04-07] README.md: Updated Book Series Overview link text from "all four Techstream volumes" to "all five Techstream volumes" to include Book 5 (AI and Agentic Systems Security)
- Added CHANGELOG.md (this file) for version tracking
- Added "Learning Resources" section to README.md linking to techstream-learn, techstream-books, and techstream.app
- docs/architecture.md: Added False Positive Management at Scale section covering baseline metrics, suppression vs. exception distinction, SAST tuning process, SCA false positive patterns with VEX examples, tiered severity threshold design, and security gate program health dashboard (2026-04-07)

## [1.0.0] — 2024-01-15

- Initial public release of the DevSecOps Framework
- Core framework documentation: introduction, architecture, framework, implementation, best-practices, roadmap
- AI security controls guidance for LLM-integrated development pipelines
- Secret lifecycle management documentation including automated rotation architecture
- Brownfield adoption guide for legacy environment migration
- API security controls and platform engineering integration guidance
- Runtime threat detection reference documentation
- Windows and .NET security considerations for DevSecOps
- Apache 2.0 license and contribution guidelines
