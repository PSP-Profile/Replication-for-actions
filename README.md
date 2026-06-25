
# Pedagogical SAST Profile (PSP) Action

## What is the PSP?

The **Pedagogical SAST Profile (PSP)** is a specialized Static Application Security Testing (SAST) tool built on top of Semgrep, specifically architected for **educational environments**, coding tutorials, bootcamps, and beginner-to-intermediate developers who are learning secure coding practices.

### The Problem with Standard SAST in Education

Traditional SAST tools (CodeQL, SonarCloud, commercial enterprise scanners) are designed for production-grade applications and expert security engineers. When applied to learning contexts, they create three critical pedagogical barriers:

1. **Noise Overload**: They scan everything—dependencies, generated code, framework internals, build artifacts—producing hundreds of alerts on code the learner didn't write and can't control.
2. **Abstraction Gap**: Findings say things like "Review this input validation" or "Potential injection vulnerability" without telling a Django learner to use `url_has_allowed_host_and_scheme()` or a Spring learner to add `@PreAuthorize`.
3. **Concept Dilution**: With 200+ rule categories, beginners cannot see patterns. They miss the forest for the trees, failing to connect a missing auth decorator in one file with broken access control in another.

### The PSP Solution

The PSP is a **curated, pedagogically-scoped rule set** (50 rules: 30 Django, 20 Spring) that treats security education as a first-class concern rather than a byproduct of compliance scanning.

---

## Why the PSP is Easy to Understand

### 1. Automatic Language Detection (Zero-Config Setup)
The PSP now features **auto-detection**. Simply add the action to your workflow—it automatically identifies whether your repository contains Java (Spring) or Python (Django) source files and loads the appropriate rule set. No manual `framework` configuration required:

```yaml
- uses: research-projects-all/psp-semgrep-action@v2
  with:
    framework: auto  # default; detects Java → Spring, Python → Django
```

This removes the cognitive burden of configuration from learners and instructors alike.

### 2. Scope-Limited to Tutorial-Authored Code
The PSP aggressively excludes code that learners did not write:
- **Python**: `.venv/`, `__pycache__/`, `migrations/`, `*.pyc`
- **Java**: `target/`, `build/`, `node_modules/`
- **General**: `tests/`, `.git/`

Learners only see alerts on code they can actually modify, creating a direct feedback loop between their mistakes and their learning.

### 3. Concept-Concentrated OWASP Top 10 Coverage
Instead of 200+ disparate rules, the PSP concentrates on **50 rules mapped to the OWASP Top 10**. This intentional constraint:

| OWASP Category | What Learners See | Pedagogical Goal |
|----------------|-------------------|------------------|
| A01 Broken Access Control | Missing `@login_required`, object-level auth failures | "Always check who the user is before serving data" |
| A02 Cryptographic Failures | Hardcoded passwords, MD5/SHA-1 usage | "Use the framework's built-in security, never roll your own crypto" |
| A03 Injection | SQLi, XSS, Command Injection, SSTI, XXE, LDAP | "Never trust user input; always use parameterized APIs" |
| A04 Insecure Design | Mass assignment, business logic flaws | "Validate sensitive fields server-side, never bind user input directly to models" |
| A05 Security Misconfiguration | DEBUG=True, CSRF disabled, permissive CORS | "Secure defaults matter; explicit overrides are dangerous" |
| A06 Vulnerable Components | Outdated Django/Spring Boot versions | "Dependency hygiene is security hygiene" |
| A08 Integrity Failures | Insecure deserialization (pickle, ObjectInputStream) | "Don't execute data; use JSON, not binary serialization" |
| A09 Logging Failures | Log injection via string concatenation | "Logs are attack surfaces too; use parameterized logging" |
| A10 SSRF | Unvalidated URL fetching | "Validate URLs against allowlists, especially for internal services" |

By seeing the **same 10 concepts** across different vulnerability manifestations, learners build mental models rather than memorizing isolated fixes.

### 4. Framework-Concrete Remediation Messages
Every PSP message tells the learner **exactly which API to use**:

| Vulnerability | Generic SAST Message | PSP Message |
|---------------|---------------------|-------------|
| Missing auth | "Implement access control" | "Add `@login_required` for logged-in users, or `@permission_required` for specific roles" |
| Open redirect | "Validate redirect URL" | "Use Django's `url_has_allowed_host_and_scheme()` to verify URLs are safe" |
| Missing Spring auth | "Add authorization" | "Add `@PreAuthorize` or `@Secured` annotations to restrict access" |
| Weak hashing | "Use strong hashing" | "Use `BCryptPasswordEncoder` to securely hash passwords before storage" |

This **concreteness** eliminates the translation layer that typically requires expert mentorship.

### 5. Severity as Pedagogical Progression
The PSP uses Semgrep severity levels as a teaching scaffold:
- **ERROR**: Critical security flaws that will definitely be exploited (Injection, Broken Auth, SSRF). These fail the build if `fail-on-finding: true`.
- **WARNING**: Design-level issues that enable future vulnerabilities (Mass Assignment, CSRF disabled, permissive CORS).
- **INFO**: Hygiene issues that build professional discipline (Log injection formatting).

Instructors can tune the rigor: start with `fail-on-finding: false` to teach, then enable it for assessment.

### 6. Native GitHub Code Scanning Integration
Results are emitted in SARIF format and uploaded directly to GitHub's Security tab. Learners see findings inline with their code in pull requests, with OWASP categories, CWE IDs, and direct links to official cheat sheets.

---

## Usage

### Basic (Auto-Detect)
```yaml
name: Security Learning Scan
on: [push, pull_request]

jobs:
  psp:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: research-projects-all/psp-semgrep-action@v2
```

### Explicit Framework
```yaml
      - uses: research-projects-all/psp-semgrep-action@v2
        with:
          framework: django    # or spring, both, auto
          scan-path: ./src
          fail-on-finding: true
```

### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `framework` | Target framework: `django`, `spring`, `both`, or `auto` | No | `auto` |
| `scan-path` | Path to scan | No | `.` |
| `output-format` | `text`, `json`, or `sarif` | No | `sarif` |
| `output-file` | Output file path | No | `psp-results.sarif` |
| `fail-on-finding` | Fail build on ERROR severity | No | `false` |

---

## Rule Inventory

### Django (30 Rules)
Covers: Missing auth decorators, object-level authorization, BOLA, open redirects, hardcoded passwords, weak hashing, SQL injection (raw queries), XSS (`mark_safe`), command injection, insecure file uploads, SSTI, LDAP injection, XXE, email header injection, mass assignment, business logic flaws, DEBUG mode, hardcoded `SECRET_KEY`, CSRF exemptions, permissive CORS, HTTP method validation, missing rate limiting, outdated dependencies, insecure pickle/YAML deserialization, log injection, SSRF, cache poisoning.

### Spring (20 Rules)
Covers: Missing authz annotations (`@PreAuthorize`), IDOR/object-level auth, open redirects, `NoOpPasswordEncoder`, weak MD5/SHA-1, SQLi via `JdbcTemplate`, XSS in response bodies, command injection via `Runtime.exec`, unrestricted file uploads, SSTI (`SpelExpressionParser`), XXE via `DocumentBuilderFactory`, LDAP injection, mass assignment via `@RequestBody`, disabled CSRF, permissive CORS, outdated Spring Boot, insecure deserialization (`ObjectInputStream`), log injection concatenation, SSRF via `RestTemplate`.

---

## Release Notes

### v2.0.0 — Intelligent Language Detection
- **Added**: `framework: auto` (default) automatically detects Java and Python source files in the repository and selects the appropriate rule set.
- **Added**: Dynamic `--include` filters optimize scan performance by targeting only relevant file extensions per detected framework.
- **Improved**: Detection step emits GitHub Actions notices so workflow logs clearly show which framework was selected and why.
- **Maintained**: Full backward compatibility with explicit `django`, `spring`, and `both` values.

### v1.0.0 — Initial Release
- 50 curated Semgrep rules (30 Django, 20 Spring) aligned with OWASP Top 10.
- SARIF output with native GitHub Code Scanning upload.
- Scope-limited scanning excluding dependencies, tests, and generated code.
- Framework-concrete remediation messages with official OWASP cheat sheet references.


# Release Notes

## v2.0.0 — Language Detection
**Release Date**: 2026-05-27

### New Features
- **Auto-Detection (`framework: auto`)**: The action now inspects the repository for `.java` and `.py` files and automatically selects Spring, Django, or both rule sets. This makes the action truly zero-config for mixed classrooms.
- **Dynamic File Targeting**: When running in `spring` or `django` mode, Semgrep only includes the relevant file extensions (`*.java` / `*.py`), reducing scan time and noise.
- **Transparent Logging**: The detection step prints GitHub Actions notices (`::notice::`) so instructors and students can see exactly why a specific framework was chosen.

### Improvements
- Backward compatibility maintained for all explicit `framework` values (`django`, `spring`, `both`).
- `framework` input changed from `required: true` to `required: false` with `default: 'auto'`.

## v1.0.0 — Pedagogical SAST Profile Launch
**Release Date**: 2026-05-20

### Features
- 50 curated Semgrep rules (30 Django, 20 Spring) mapped to OWASP Top 10 2021.
- SARIF, JSON, and plain-text output formats.
- Native GitHub Code Scanning integration via `upload-sarif`.
- Scope-limited scanning that excludes dependencies, virtual environments, build artifacts, and test code.
- Framework-concrete remediation messages referencing exact APIs (`@login_required`, `BCryptPasswordEncoder`, `url_has_allowed_host_and_scheme()`, etc.).
```

