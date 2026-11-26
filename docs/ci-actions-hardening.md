# GitHub Actions Hardening and Authoring Guide

This document establishes the mandatory security and operational guidelines for authoring GitHub Actions workflows across the NeoNephos Katalis ecosystem. These rules enforce compliance with the NeoNephos Actions Hardening Guide, European supply chain security requirements, and Linux Foundation best practices.

---

## 1. Action Pinning (Full-Length Commit SHA)

### Mandatory Requirement
Every external or third-party GitHub Action must be referenced using its **full-length 40-character commit SHA**, accompanied by a trailing comment indicating the semantic release version.

### Rationale
Git tags (such as `@v4` or `@v5`) and branch names (such as `@main`) are mutable pointers. An attacker who compromises a maintainer account or upstream repository can update an existing tag to point to malicious code, executing unauthorized payloads inside the runner environment. A full-length commit SHA provides cryptographic immutability.

### Correct Pattern
```yaml
# Compliant: Immutable commit SHA with human-readable version comment
- name: Checkout code
  uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4.2.2

- name: Setup Go environment
  uses: actions/setup-go@40f1582b2485089dde7abd97c1529aa768e1baff # v5.3.0
  with:
    go-version-file: go.mod
```

### Non-Compliant Pattern
```yaml
# FORBIDDEN: Mutable tags can be maliciously modified upstream
- name: Checkout code
  uses: actions/checkout@v4

- name: Setup Go environment
  uses: actions/setup-go@v5
```

### Approved Action Sources
Per enterprise policy, only actions meeting one of the following criteria may be used:
1. Actions created by GitHub (`actions/*`).
2. Actions developed and hosted in enterprise-owned repositories (`neonephos/*` or `neonephos-katalis/*`).
3. Verified creators in the GitHub Marketplace.

---

## 2. Principle of Least Privilege (`permissions:`)

### Mandatory Requirement
Every workflow file must explicitly define the minimal necessary permissions required for execution using the top-level `permissions:` block.

### Rationale
By default, GitHub Actions workflows run with permissive token scopes. If a workflow is compromised, overly permissive tokens allow unauthorized code pushes, release tampering, or secret theft. Enforcing minimal permissions limits the blast radius.

### Standard Read-Only Configuration
For standard test, lint, and build workflows that only need to read the repository code:

```yaml
name: Test Suite

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Explicitly scope GITHUB_TOKEN permissions to read-only
permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4.2.2
```

### Permission Scopes Table

| Scope | Minimum Value | Use Case |
|---|---|---|
| `contents` | `read` | Checking out repository code |
| `contents` | `write` | Pushing tags, releases, or sync branches |
| `pull-requests` | `write` | Creating or commenting on Pull Requests |
| `issues` | `write` | Opening or closing issues |
| `packages` | `write` | Publishing container images to GHCR |
| `id-token` | `write` | OpenID Connect (OIDC) cloud authentication |

---

## 3. Script Injection Prevention

### Mandatory Requirement
Never evaluate GitHub context expressions (`${{ ... }}`) or secrets directly inside inline `run:` shell strings. Pass them as environment variables through the step's `env:` block.

### Rationale
When a context expression is evaluated directly in a `run:` block, GitHub Actions performs string interpolation *before* the shell executes the command. If a variable contains malicious characters (such as `;`, `&&`, or backticks), an attacker can escape the command and execute arbitrary code. Passing values via `env:` assigns them safely to process environment variables.

### Correct Pattern
```yaml
# Compliant: Variables are safely passed via environment variables
- name: Configure authentication
  env:
    GH_ACTOR: ${{ github.actor }}
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    echo "machine github.com login $GH_ACTOR password $GH_TOKEN" > "$HOME/.netrc"
    chmod 600 "$HOME/.netrc"
```

### Non-Compliant Pattern
```yaml
# FORBIDDEN: Direct interpolation allows command injection
- name: Configure authentication
  run: |
    echo "machine github.com login ${{ github.actor }} password ${{ secrets.GITHUB_TOKEN }}" > "$HOME/.netrc"
```

---

## 4. Safe Tool Installation and Deterministic Environments

### Mandatory Requirements
1. **Secure Flags:** Always invoke `curl` with fail-fast and silent flags: `curl -fsSL <url> | bash`.
2. **Pinned Tool Versions:** Never download unpinned or `latest` binaries in automated workflows. Always pin exact versions for CLI tools (`kind`, `kubectl`, `helm`, `kubebuilder`).
3. **Environment Cleanup:** Add post-execution teardown steps protected by `if: always()` to clean up ephemeral clusters or background containers.

### Example: Kind and Helm Setup
```yaml
env:
  KIND_VERSION: v0.26.0
  KIND_NODE_IMAGE: kindest/node:v1.32.0

steps:
  - name: Install kind binary
    run: |
      curl -Lo ./kind "https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64"
      chmod +x ./kind
      sudo mv ./kind /usr/local/bin/kind
      kind version

  - name: Create test cluster
    run: |
      kind create cluster --name test-cluster --image "${KIND_NODE_IMAGE}" --wait 120s

  - name: Teardown test cluster
    if: always()
    run: |
      kind delete cluster --name test-cluster || true
```

---

## 5. Concurrency Control

### Mandatory Requirement
Workflows triggered by pushes and pull requests should specify concurrency groups with `cancel-in-progress: true` to cancel redundant runs when new commits are pushed.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

---

## 6. Authoring and Review Checklist

Before opening or approving a Pull Request that modifies or introduces GitHub Actions workflows:

- [ ] Every `uses:` statement references a full 40-character commit SHA with a version comment.
- [ ] Only official GitHub (`actions/*`) or approved enterprise actions are used.
- [ ] Top-level `permissions:` block is declared with least-privilege scopes.
- [ ] No inline `${{ ... }}` expressions exist inside `run:` script bodies; all values are passed via `env:`.
- [ ] External CLI tools are downloaded with explicit pinned version numbers (no `latest`).
- [ ] Ephemeral resources (Kind clusters, Docker test networks) include automated teardown steps under `if: always()`.
- [ ] Concurrency rules prevent duplicate workflow executions.
