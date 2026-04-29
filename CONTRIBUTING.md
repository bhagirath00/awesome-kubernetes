# Contributing to Awesome Kubernetes

Thank you for your interest in contributing to **Awesome Kubernetes**! We welcome contributions, bug fixes, enhancements, and architectural improvements from the community.

---

## How to Contribute

### Reporting Issues
If you identify a bug in any Kubernetes manifest, syntax error, broken link, or have a suggestion for new modules:
1. Check if a similar issue already exists.
2. Open a new issue with a clear, descriptive title and explanation.
3. Include your Kubernetes version (e.g. 1.28+), distribution (Kind, Minikube, EKS, GKE, AKS), and steps to reproduce.

### Making Changes

1. **Fork** the repository to your GitHub account.
2. **Clone** your fork locally:
   ```bash
   git clone https://github.com/bhagirath00/awesome-kubernetes.git
   cd awesome-kubernetes
   ```
3. Create a feature **branch** for your changes:
   ```bash
   git checkout -b feature/your-feature-name
   ```
4. Make your changes adhering to the style guide below.
5. **Commit** your changes:
   ```bash
   git commit -m "feat(module-20): add fluxcd helm release examples"
   ```
6. **Push** to your fork:
   ```bash
   git push origin feature/your-feature-name
   ```
7. Open a **Pull Request** against the `main` branch.

---

## Style Guide

### Kubernetes Manifests & Code
- **Declarative YAML**: Always provide clean, declarative Kubernetes manifests with explicit `apiVersion`, `kind`, and `metadata`.
- **Validation**: Validate manifests locally before committing:
  ```bash
  kubectl apply --dry-run=client -f <file.yaml>
  ```
- **Universal Compatibility**: Ensure examples run on standard local or managed clusters (Kind, Minikube, EKS, GKE, AKS).
- **Naming Conventions**: Use lowercase, hyphen-separated names for Kubernetes resources (e.g., `payment-service-deployment`).
- **Security Defaults**: Avoid `privileged: true`, `hostNetwork: true`, or running as `root` unless specifically demonstrating a low-level diagnostic scenario.

### Documentation & Guides
- Use clear, technical, professional English.
- Include working CLI command snippets and sample expected outputs.
- Include ASCII architectural flow diagrams or comparison tables where helpful.
- Reference official documentation ([kubernetes.io/docs](https://kubernetes.io/docs/home/)) for authoritative upstream specs.
- Maintain sequential chapter numbering (`01-`, `02-`, etc.) to preserve directory sorting.

### Commit Messages
We follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:
- `feat:` – New chapter, guide, or feature addition
- `fix:` – Bug fix, YAML correction, or typo fix
- `docs:` – Documentation updates or formatting adjustments
- `refactor:` – Reorganizing chapters, updating naming, or streamlining manifests

---

## Questions & Support

Feel free to open a GitHub Discussion or Issue for any questions.

Thank you for contributing to **Awesome Kubernetes**!
