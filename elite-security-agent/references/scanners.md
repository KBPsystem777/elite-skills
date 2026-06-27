# Scanner belt

Scanners give breadth. They will not understand your business logic, tenancy,
or availability budget — that is what Step 4 (audit domains) is for. Run the
ones that match the stack, save raw output, dedupe, then cite findings.

Capture everything under `/tmp/elite-security-agent/<scanner>.json` (or `.sarif`
when available) so later steps can parse and dedupe.

## Universal — run on every audit

### Secret scanning

```bash
# gitleaks — git history + working tree
gitleaks detect --no-git -v --report-format json --report-path /tmp/elite-security-agent/gitleaks.json
gitleaks detect       -v --report-format json --report-path /tmp/elite-security-agent/gitleaks-history.json

# trufflehog — verified secrets only (lower false positives)
trufflehog filesystem . --json --only-verified > /tmp/elite-security-agent/trufflehog.json
```

A committed secret is compromised even after deletion — git history is forever.
Rotation is mandatory; deletion is cleanup.

### SAST — Semgrep

```bash
# auto config picks rule packs by language fingerprint
semgrep --config auto --json --output /tmp/elite-security-agent/semgrep.json .

# stricter: OWASP top 10 + secrets + supply chain
semgrep --config p/owasp-top-ten --config p/secrets --config p/supply-chain \
  --json --output /tmp/elite-security-agent/semgrep-strict.json .
```

### IaC + container config — Trivy / Checkov

```bash
trivy config . --format json --output /tmp/elite-security-agent/trivy-config.json
trivy fs     . --scanners vuln,secret,misconfig --format json \
  --output /tmp/elite-security-agent/trivy-fs.json

checkov -d . -o json --output-file-path /tmp/elite-security-agent
```

## Per-ecosystem

### Node.js / TypeScript

```bash
npm audit --json > /tmp/elite-security-agent/npm-audit.json  || true
pnpm audit --json > /tmp/elite-security-agent/pnpm-audit.json || true
yarn npm audit --json > /tmp/elite-security-agent/yarn-audit.json || true
osv-scanner --lockfile=package-lock.json --format json \
  --output /tmp/elite-security-agent/osv-node.json
```

### Python

```bash
pip-audit -r requirements.txt -f json -o /tmp/elite-security-agent/pip-audit.json
# or for poetry / uv lock files:
osv-scanner --lockfile=poetry.lock -f json -o /tmp/elite-security-agent/osv-py.json
bandit -r . -f json -o /tmp/elite-security-agent/bandit.json
```

### Rust

```bash
cargo audit --json > /tmp/elite-security-agent/cargo-audit.json
cargo deny check --format json > /tmp/elite-security-agent/cargo-deny.json
```

### Go

```bash
govulncheck -json ./... > /tmp/elite-security-agent/govulncheck.json
gosec -fmt=json -out=/tmp/elite-security-agent/gosec.json ./...
```

### Java / JVM

```bash
# Maven
mvn -q org.owasp:dependency-check-maven:check \
  -DfailBuildOnCVSS=0 -Dformat=JSON \
  -DoutputDirectory=/tmp/elite-security-agent
# Gradle
gradle dependencyCheckAnalyze \
  -PoutputFormat=JSON -PoutputDirectory=/tmp/elite-security-agent
```

### Solidity / EVM

```bash
slither . --json /tmp/elite-security-agent/slither.json
mythril analyze contracts/*.sol -o json > /tmp/elite-security-agent/mythril.json
```

## Container images

```bash
# For each image referenced by Dockerfile / compose / k8s manifests:
trivy image <image:tag> --format json --output /tmp/elite-security-agent/trivy-image.json
grype  <image:tag> -o json > /tmp/elite-security-agent/grype.json
```

Flag: root user, no `USER` directive, `latest` tag, no resource limits, exposed
ports beyond what the service needs, secrets baked into image layers (use
`dive` or `trivy image --scanners secret`).

## Kubernetes / Helm

```bash
kube-linter lint . --format json > /tmp/elite-security-agent/kube-linter.json
kubesec scan deployment.yaml > /tmp/elite-security-agent/kubesec.json
trivy k8s --report summary cluster   # if cluster access is in scope
```

## CI/CD

Read `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml` by
hand. Scanners catch some patterns, but the dangerous ones are
human-reviewable:

- `pull_request_target` with checkout of PR head → arbitrary code execution
  with secrets.
- Long-lived cloud keys in secrets instead of OIDC federation.
- No required reviewers, no branch protection, no signed commits.
- Third-party actions pinned by tag (`@v3`) instead of SHA.

```bash
# Lint workflows
actionlint -format '{{json .}}' > /tmp/elite-security-agent/actionlint.json
# Hunt for the dangerous pattern
grep -rE 'pull_request_target' .github/workflows/ || true
```

## Triage and dedupe

1. Drop duplicates (same file:line, same rule) — Semgrep often overlaps
   ecosystem scanners.
2. Drop test files and vendored deps from SAST results unless the finding is
   itself about test/vendored code being shipped.
3. Cross-reference SCA results against the lockfile — flag transitive vs.
   direct.
4. For every "Critical/High" from any scanner, manually verify reachability
   before promoting to the report. Unreachable code is "unconfirmed", not
   "Critical".

## Graceful fallback

If a scanner is not installable in the environment, say so explicitly in the
report rather than silently skipping the category. Manual review of that
category is the fallback — never an empty "no findings" section.
