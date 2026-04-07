# Contributing to the Techstream DevSecOps Framework

Thank you for your interest in contributing. The Techstream Framework Suite is an open, community-maintained reference library for DevSecOps, secure software delivery, and cloud security engineering. Contributions that improve technical accuracy, documentation clarity, and real-world applicability are welcome.

---

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [What We Welcome](#what-we-welcome)
- [What We Do Not Accept](#what-we-do-not-accept)
- [How to Contribute](#how-to-contribute)
- [Documentation Standards](#documentation-standards)
- [Review Process](#review-process)
- [License](#license)

---

## Code of Conduct

All contributors are expected to engage professionally and constructively. Technical disagreements should focus on substance, not individuals. Contributions that are dismissive, personal, or unprofessional will not be reviewed.

---

## What We Welcome

- **Corrections to technical inaccuracies** — if a tool reference, configuration pattern, or security control is described incorrectly, please open an issue or submit a pull request with the correction and a brief explanation.
- **Additions of missing tools, patterns, or practices** — the security tooling landscape evolves rapidly. If an important tool category, control pattern, or emerging practice is not covered, additions are welcome.
- **Improvements to clarity and depth** — sections that are vague, shallow, or ambiguous benefit from contributions that make them more specific and actionable.
- **New examples and implementation guidance** — concrete examples, configuration snippets, and step-by-step guidance are consistently among the most valuable contributions.
- **Cross-framework alignment** — if you identify inconsistencies in terminology, architecture, or recommendations between Techstream repositories, please open an issue so the core team can address them consistently.

---

## What We Do Not Accept

- **Vendor promotional content** — tool references must be accurate and based on technical merit. Contributions that read as product marketing will not be accepted.
- **Fabricated metrics or case studies** — all examples must be generic and realistic. Do not invent specific performance numbers, client names, or outcome data.
- **Scope creep beyond the framework's defined domain** — this repository covers DevSecOps principles, organizational models, and the software delivery security lifecycle. Contributions outside this scope should be directed to the appropriate Techstream repository.
- **Breaking changes to the document structure** — major restructuring of the documentation requires prior discussion in a GitHub Issue before a pull request is opened.

---

## How to Contribute

### Reporting Issues

Use GitHub Issues to report:
- Technical errors or outdated information
- Gaps in coverage of important topics
- Inconsistencies with other Techstream repositories
- Broken links or formatting problems

Include: the specific section or file affected, the problem description, and (for technical corrections) a reference or justification for the correction.

### Submitting Pull Requests

1. **Fork the repository** and create a branch from `main` with a descriptive name (e.g., `add-slsa-level3-guidance`, `fix-semgrep-configuration-example`).
2. **Make your changes** following the documentation standards below.
3. **Test your Markdown** — ensure all Mermaid diagrams render correctly, all internal links resolve, and tables are properly formatted.
4. **Open a pull request** against `main` with a clear description of:
   - What was changed and why
   - Which section(s) are affected
   - Any references or sources for technical claims
5. **Respond to review comments** — the core team reviews all pull requests. If changes are requested, please address them or explain your reasoning.

---

## Documentation Standards

All contributions must adhere to the documentation standards maintained across the Techstream suite:

**Tone and Style**
- Professional, direct, and technical. Avoid marketing language and superlatives.
- Write for a practitioner audience: security architects, DevSecOps engineers, platform engineers, and engineering managers.
- Use active voice and present tense where possible.
- Avoid filler phrases ("It is important to note that...", "As we can see...").

**Technical Accuracy**
- All tool names, command syntax, and configuration examples must be accurate for the current stable version of the tool.
- Include version numbers or caveats when guidance is version-specific.
- Code blocks must be syntactically correct and representative of real-world usage.

**Markdown Formatting**
- Use ATX-style headers (`#`, `##`, `###`).
- Use fenced code blocks with language identifiers (` ```bash `, ` ```yaml `, ` ```mermaid `).
- Tables should be used for structured comparisons; avoid tables for simple lists.
- Internal links should use relative paths to other files in the repository.

**Mermaid Diagrams**
- Use Mermaid for architecture and flow diagrams where visual representation adds clarity.
- Keep diagrams focused: one concept per diagram. Complex diagrams should be split.
- Include a brief text description before or after each diagram explaining what it depicts.

**Section Structure**
Each major documentation file follows a consistent structure:
- Table of Contents at the top
- Sections separated by `---` horizontal rules
- Subsections introduced with `###` headers
- Actionable guidance in numbered or bulleted lists

---

## Review Process

Pull requests are reviewed by the Techstream core team. The review focuses on:

1. **Technical correctness** — is the content accurate?
2. **Scope alignment** — does the contribution fit the framework's defined domain?
3. **Documentation standards** — does it meet the style and formatting requirements?
4. **Cross-repository consistency** — does it align with guidance in other Techstream frameworks?

Most pull requests receive an initial response within 5 business days. Substantive contributions may require multiple review cycles. If a pull request has not received a response within 10 business days, please comment on it to request an update.

---

## License

By contributing to this repository, you agree that your contributions will be licensed under the [Apache License 2.0](LICENSE), the same license that covers the existing content. You certify that you have the right to submit the contribution under this license.
