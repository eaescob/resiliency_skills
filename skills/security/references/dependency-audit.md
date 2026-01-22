# Dependency Auditing

Automated vulnerability scanning for project dependencies.

## Table of Contents
- [Node.js Auditing](#nodejs-auditing)
- [Go Auditing](#go-auditing)
- [CI/CD Integration](#cicd-integration)
- [Fixing Vulnerabilities](#fixing-vulnerabilities)
- [Allowlisting](#allowlisting)

## Node.js Auditing

### npm audit

```bash
# Check for vulnerabilities
npm audit

# Auto-fix where possible
npm audit fix

# Force fixes (may include breaking changes)
npm audit fix --force

# JSON output for CI
npm audit --json
```

### better-npm-audit

More flexible than built-in npm audit:

```bash
npx better-npm-audit audit

# Ignore specific advisories
npx better-npm-audit audit --exclude 1234,5678

# Set severity threshold
npx better-npm-audit audit --level high
```

### Snyk

Comprehensive vulnerability database:

```bash
# Install
npm install -g snyk

# Authenticate
snyk auth

# Test project
snyk test

# Monitor continuously
snyk monitor

# Fix vulnerabilities
snyk fix
```

### npm-check-updates

Find outdated dependencies:

```bash
npx npm-check-updates

# Update package.json
npx npm-check-updates -u

# Interactive mode
npx npm-check-updates -i
```

## Go Auditing

### govulncheck (Official)

```bash
# Install
go install golang.org/x/vuln/cmd/govulncheck@latest

# Scan project
govulncheck ./...

# JSON output
govulncheck -json ./...

# Scan specific package
govulncheck -test ./pkg/...
```

### nancy (Sonatype)

```bash
# Install
go install github.com/sonatype-nexus-community/nancy@latest

# Scan
go list -json -deps ./... | nancy sleuth

# With allowlist
go list -json -deps ./... | nancy sleuth --exclude-vulnerability CVE-2022-1234
```

### Trivy

Container and dependency scanning:

```bash
# Install
brew install trivy  # or appropriate for your OS

# Scan filesystem
trivy fs .

# Scan specific language
trivy fs --scanners vuln --pkg-types go .
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * *'  # Daily

jobs:
  audit-node:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run audit
        run: npm audit --audit-level=high

  audit-go:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Install govulncheck
        run: go install golang.org/x/vuln/cmd/govulncheck@latest

      - name: Run govulncheck
        run: govulncheck ./...
```

### GitLab CI

```yaml
# .gitlab-ci.yml
security-audit:
  stage: test
  image: node:20
  script:
    - npm ci
    - npm audit --audit-level=high
  allow_failure: false
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

dependency-scanning:
  stage: test
  image:
    name: aquasec/trivy
    entrypoint: [""]
  script:
    - trivy fs --exit-code 1 --severity HIGH,CRITICAL .
```

### Pre-commit Hook

```bash
#!/bin/sh
# .git/hooks/pre-commit

echo "Running security audit..."

if [ -f "package.json" ]; then
  npm audit --audit-level=high
  if [ $? -ne 0 ]; then
    echo "Security vulnerabilities found. Fix before committing."
    exit 1
  fi
fi

if [ -f "go.mod" ]; then
  govulncheck ./...
  if [ $? -ne 0 ]; then
    echo "Security vulnerabilities found. Fix before committing."
    exit 1
  fi
fi
```

## Fixing Vulnerabilities

### Node.js

```bash
# 1. Check what needs updating
npm audit

# 2. Try automatic fix
npm audit fix

# 3. If that doesn't work, update specific package
npm update vulnerable-package

# 4. If transitive dependency, use overrides (npm 8.3+)
# package.json
{
  "overrides": {
    "vulnerable-package": "^2.0.0"
  }
}

# 5. For yarn, use resolutions
{
  "resolutions": {
    "vulnerable-package": "^2.0.0"
  }
}
```

### Go

```bash
# 1. Update specific dependency
go get package@latest

# 2. Update all dependencies
go get -u ./...

# 3. Tidy up
go mod tidy

# 4. Verify
govulncheck ./...
```

### When Updates Break Things

1. Check if vulnerability affects your usage
2. If not, consider allowlisting (temporarily)
3. Look for alternative packages
4. Pin to last safe version while working on fix
5. Consider forking and patching

## Allowlisting

### npm audit (package.json)

```json
{
  "overrides": {
    "vulnerable-but-unused": {
      ".": {
        "note": "Not exploitable in our usage",
        "expires": "2024-06-01"
      }
    }
  }
}
```

### better-npm-audit

```json
// .nsprc
{
  "exceptions": [
    "https://github.com/advisories/GHSA-xxxx-yyyy"
  ]
}
```

### Snyk (.snyk file)

```yaml
version: v1.5.0
ignore:
  SNYK-JS-LODASH-1234:
    - '*':
        reason: 'Not exploitable - input is trusted'
        expires: '2024-06-01T00:00:00.000Z'
```

### Go nancy

```yaml
# .nancy-ignore
CVE-2022-1234 # Description of why it's safe
CVE-2022-5678 until=2024-06-01 # Temporary ignore
```

### Documenting Exceptions

Always document why a vulnerability is allowlisted:

```markdown
## Security Exceptions

### CVE-2022-1234 (lodash prototype pollution)
- **Package**: lodash@4.17.20
- **Expires**: 2024-06-01
- **Reason**: The vulnerable function `set()` is not used in our codebase.
  We only use `get()` and `merge()` with trusted objects.
- **Mitigation**: Input validation prevents untrusted data reaching lodash.
- **Tracking**: JIRA-1234
```
