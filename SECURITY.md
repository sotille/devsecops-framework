# Security Policy

## Supported Versions

The `devsecops-framework` framework follows semantic versioning. The current major version receives security updates.

| Version | Supported          |
| ------- | ------------------ |
| 1.x     | :white_check_mark: |
| < 1.0   | :x:                |

## Reporting a Vulnerability

If you discover a security vulnerability or implementation concern in the patterns documented in this framework, please report it responsibly:

- **Preferred channel:** open a GitHub Security Advisory at https://github.com/sotille/devsecops-framework/security/advisories/new
- **Alternative:** email `fsotille@gmail.com` with subject prefix `[SECURITY] devsecops-framework`

Please include:
1. A clear description of the issue
2. Steps to reproduce (if applicable)
3. Affected version(s) or documents
4. Suggested mitigation (if known)

You will receive an acknowledgment within 5 business days. A coordinated disclosure timeline will be agreed before any public disclosure.

## Scope

This framework documents **patterns, reference architectures, and methodology**, not production binaries. Security concerns include:
- Documentation describing patterns that would be insecure if implemented as written
- Reference code samples with security flaws
- Misaligned guidance versus current federal standards (NIST SSDF, EO 14028, etc.)
- Outdated references to deprecated practices

## Acknowledgments

Contributors who responsibly report security issues will be acknowledged in `CHANGELOG.md` unless they request otherwise.
