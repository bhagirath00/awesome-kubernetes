# Security Policy

## Content Security & Attribution

This repository is designed for educational and professional reference purposes. The architectural guidance, manifests, and documentation provided here are compiled in accordance with industry best practices and official documentation.

We acknowledge and credit the following foundational references:
- **Kubernetes Official Documentation**: [kubernetes.io/docs](https://kubernetes.io/docs/home/)
- **Cloud Native Computing Foundation (CNCF)**: [cncf.io](https://www.cncf.io/)
- **Helm Documentation**: [helm.sh](https://helm.sh/)
- **CIS Kubernetes Benchmarks**: [cisecurity.org](https://www.cisecurity.org/benchmark/kubernetes)

---

## Educational & Production Use

- Code snippets, CLI commands, and YAML configurations are intended for learning, experimentation, and testing in non-production environments (e.g., Minikube, Kind, sandbox clusters).
- While all manifests and security hardening guides are written to production standards, always validate configurations against your organization's security policies, RBAC requirements, and CIS Benchmarks before deploying to live clusters.
- Never commit real secrets, private keys, cloud access tokens, or unencrypted SOPS keys to version control.

---

## Supported Versions

We track actively maintained, upstream Kubernetes versions:

| Kubernetes Version | Documentation & Manifest Support |
| :--- | :--- |
| **1.30.x** | :white_check_mark: Supported |
| **1.29.x** | :white_check_mark: Supported |
| **1.28.x** | :white_check_mark: Supported |
| **< 1.28** | :x: Deprecated (API removals may apply) |

---

## Reporting Vulnerabilities & Content Issues

### Content & Configuration Inaccuracies
If you find any inaccuracies, broken links, outdated APIs, or configuration errors in the documentation, please:
1. Open an issue on GitHub with a clear description and reproduction steps.
2. Submit a Pull Request following our [Contributing Guidelines](CONTRIBUTING.md).

### Security Vulnerabilities
If you discover a security vulnerability (such as exposed credentials, high-risk security flaws in example manifests, or dependency vulnerabilities):
1. **Do NOT open a public issue.**
2. Send a report via GitHub Private Vulnerability Reporting or contact the maintainers directly.
3. Include details of the vulnerability, affected files, and suggested remediation steps.
4. Maintainers will review the report and coordinate a fix promptly.
