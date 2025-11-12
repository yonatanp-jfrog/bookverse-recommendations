# 🔄 BookVerse Recommendations - CI/CD Workflow Reference

## 📋 Table of Contents

- [Quick Reference](#-quick-reference)
- [Workflow Steps Breakdown](#-workflow-steps-breakdown)
- [Environment Variables](#-environment-variables)
- [Artifact Specifications](#-artifact-specifications)
- [Evidence Framework](#️-evidence-framework)
- [Troubleshooting](#-troubleshooting)
- [Best Practices](#-best-practices)

## ⚡ Quick Reference

### Workflow Overview

```yaml
Workflow: .github/workflows/ci.yml
Total Jobs: 3
Average Duration: 12-15 minutes
Artifacts Produced: 4 (2 Docker + 2 Generic)
Evidence Types: 6 different types
Promotion Stages: 4 (DEV → QA → STAGING → PROD)
```

### Trigger Conditions

| Trigger | Condition | App Version Created |
|---------|-----------|-------------------|
| `push` to main | Always runs | Based on commit analysis |
| `pull_request` to main | Always runs | No (build-info only) |
| `workflow_dispatch` | Manual trigger | Based on input parameter |

### Job Dependencies

```mermaid
graph LR
    A[analyze-commit] --> B[build-test-publish]
    B --> C[create-promote]
    A -.-> C
```

## 🔧 Workflow Steps Breakdown

### Job 1: `analyze-commit`

#### Purpose

Intelligently determines whether to create an AppTrust application version based on commit content analysis.

#### Steps Detail

##### 1. Repository Setup

```yaml
- name: "[Setup] Checkout"
  uses: actions/checkout@v4
  with:
    fetch-depth: 2  # Need 2 commits for diff analysis
```

##### 2. Shared Infrastructure Access

```yaml
- name: "[Setup] Checkout bookverse-infra"
  # Provides access to shared DevOps scripts and libraries
  # Location: bookverse-infra/libraries/bookverse-devops/scripts/
```

##### 3. Commit Analysis

```bash
# Script: analyze-commit.sh
# Analyzes:
# - Changed file patterns
# - Commit message keywords
# - File type significance
# - Business rules for version creation

# Output Variables:
create_app_version: true|false
decision_reason: "Human readable explanation"
commit_type: "code|docs|config|tests"
```

#### Decision Logic

```bash
# Code changes → create_app_version: true
if [[ $(git diff --name-only HEAD~1 | grep -E '\.(py|js|ts|yaml)$') ]]; then
  create_app_version=true
  decision_reason="Code changes detected"
fi

# Documentation only → create_app_version: false
if [[ $(git diff --name-only HEAD~1 | grep -v -E '\.(md|txt)$' | wc -l) -eq 0 ]]; then
  create_app_version=false
  decision_reason="Documentation changes only"
fi
```

### Job 2: `build-test-publish`

#### Build & Test Purpose

Core CI/CD pipeline that builds, tests, and publishes all service artifacts.

#### Phase Breakdown

##### Phase 1: Environment Setup (Steps 1-5)

```yaml
# Duration: ~2 minutes
# Steps:
1. Repository checkout
2. Shared infrastructure checkout  
3. Trigger information display
4. Build metadata configuration
5. JFrog CLI setup with OIDC
```

##### Phase 2: Authentication & Security (Steps 6-7)

```yaml
# Duration: ~1 minute
# Steps:
6. JFrog authentication verification
7. OIDC token exchange for API access
```

##### Phase 3: Version Management (Steps 8-9)

```yaml
# Duration: ~1 minute
# Steps:
8. Python dependencies installation
9. SemVer determination for all artifacts
```

**SemVer Process**:

```bash
# Uses shared semver script from bookverse-infra
./determine-semver.sh \
  --application-key "bookverse-recommendations" \
  --version-map "./config/version-map.yaml" \
  --jfrog-url "$JFROG_URL" \
  --jfrog-token "$JF_OIDC_TOKEN" \
  --project-key "$PROJECT_KEY" \
  --packages "recommendations,recommendations-worker,recommendation-config,resources"

# Output: Individual versions for each package
RECOMMENDATIONS_VERSION=1.2.3
WORKER_VERSION=1.2.4  
CONFIG_VERSION=1.2.1
RESOURCES_VERSION=1.2.2
APP_VERSION=1.2.3
```

##### Phase 4: Build & Test (Steps 10-13)

```yaml
# Duration: ~3 minutes
# Steps:
10. Python environment setup
11. JFrog environment configuration
12. Python dependency installation via JFrog
13. Test execution with coverage
```

**Test Configuration**:

```bash
# Test Command
python -m pytest tests/test_simple.py -v --cov-report=xml --cov-report=term-missing

# Coverage Requirements
minimum_coverage: 80%
current_coverage: 85.5%
status: PASSED
```

##### Phase 5: Multi-Artifact Build (Steps 14-22)

```yaml
# Duration: ~4 minutes
# Steps:
14. API Docker image build & push
15. API image evidence collection
16. Config artifact creation & upload
17. Config evidence collection  
18. Resources artifact creation & upload
19. Resources evidence collection
20. Worker Docker image build & push
21. Worker image evidence collection
22. Build-info publishing
```

**Docker Build Process**:

```bash
# API Image
IMAGE_NAME="$REGISTRY_URL/$REPO_KEY/recommendations:$RECOMMENDATIONS_VERSION"
jf docker build --pull -t "$IMAGE_NAME" --build-name "$BUILD_NAME" --build-number "$BUILD_NUMBER" .
docker push "$IMAGE_NAME"

# Worker Image  
WORKER_IMAGE="$REGISTRY_URL/$REPO_KEY/recommendations-worker:$WORKER_VERSION"
jf docker build -f Dockerfile.worker -t "$WORKER_IMAGE" --build-name "$BUILD_NAME" --build-number "$BUILD_NUMBER" .
docker push "$WORKER_IMAGE"
```

**Generic Artifact Process**:

```bash
# Config Bundle
tar -czf "recommendations-config-$CONFIG_VERSION.tar.gz" recommendations-settings.yaml
jf rt upload "dist/config/recommendations-config-$CONFIG_VERSION.tar.gz" \
  "$GENERIC_REPO/recommendations/config/$CONFIG_VERSION/"

# Resources Bundle
tar -czf "recommendations-resources-$RESOURCES_VERSION.tar.gz" stopwords.txt  
jf rt upload "dist/resources/recommendations-resources-$RESOURCES_VERSION.tar.gz" \
  "$GENERIC_REPO/recommendations/resources/$RESOURCES_VERSION/"
```

##### Phase 6: Evidence & Build-Info (Steps 23-24)

```yaml
# Duration: ~2 minutes
# Steps:
23. Build evidence collection
24. Build-info publishing with artifact association
```

### Job 3: `create-promote`

#### Promotion Purpose

Creates AppTrust application version and promotes through all deployment stages.

#### Execution Condition

```yaml
if: needs.analyze-commit.outputs.create_app_version == 'true' && 
    needs.build-test-publish.result == 'success'
```

#### Steps Overview

```yaml
# Duration: ~6 minutes
# Steps:
1-5.   Setup (checkout, JFrog CLI, OIDC exchange)
6.     Application version creation
7.     Application version evidence
8.     DEV stage promotion & evidence
9.     QA stage promotion & evidence  
10.    STAGING stage promotion & evidence
11.    PROD stage promotion & evidence
12.    Build summary generation
```

#### AppTrust Integration

##### Application Version Creation

```bash
# API Call
POST /apptrust/api/v1/applications/bookverse-recommendations/versions

# Payload
{
  "version": "$APP_VERSION",
  "tag": "release",
  "sources": {
    "builds": [{
      "name": "$BUILD_NAME",
      "number": "$BUILD_NUMBER",
      "repository_key": "bookverse-build-info",
      "include_dependencies": false
    }]
  }
}
```

##### Promotion Process

```bash
# For each stage (DEV, QA, STAGING, PROD):
1. Setup promotion environment
2. Execute promotion via shared script
3. Attach stage-specific evidence
4. Validate promotion success
```

## 🌍 Environment Variables

### Required Repository Variables

```yaml
# JFrog Configuration
JFROG_URL: "https://apptrusttraining1.jfrog.io"
PROJECT_KEY: "bookverse"
DOCKER_REGISTRY: "apptrusttraining1.jfrog.io"

# Evidence Configuration  
EVIDENCE_KEY_ALIAS: "bookverse-evidence-key"
```

### Required Repository Secrets

```yaml
# Evidence Signing
EVIDENCE_PRIVATE_KEY: |
  -----BEGIN PRIVATE KEY-----
  [PEM formatted private key for evidence signing]
  -----END PRIVATE KEY-----
```

### Generated Environment Variables

```yaml
# Build Identity
BUILD_NAME: "bookverse-recommendations_CI_build-test-publish"
BUILD_NUMBER: "127-1"
SERVICE_NAME: "recommendations"
APPLICATION_KEY: "bookverse-recommendations"

# Versions
APP_VERSION: "1.2.3"
RECOMMENDATIONS_VERSION: "1.2.3"
WORKER_VERSION: "1.2.4"
CONFIG_VERSION: "1.2.1"
RESOURCES_VERSION: "1.2.2"

# Images
IMAGE_NAME: "apptrusttraining1.jfrog.io/bookverse-recommendations-internal-docker-nonprod-local/recommendations:1.2.3"
WORKER_IMAGE: "apptrusttraining1.jfrog.io/bookverse-recommendations-internal-docker-nonprod-local/recommendations-worker:1.2.4"

# Authentication
JF_OIDC_TOKEN: "[Generated OIDC access token]"
```

## 📦 Artifact Specifications

### Docker Images

#### Recommendations API Image

```yaml
Name: recommendations
Base Image: python:3.11-slim
Size: ~1.2GB
Ports: 8000
Health Check: GET /health
Registry: bookverse-recommendations-internal-docker-nonprod-local
Versioning: SemVer (e.g., 1.2.3)
```

**Image Labels**:

```dockerfile
LABEL org.opencontainers.image.title="BookVerse Recommendations API"
LABEL org.opencontainers.image.description="AI-powered book recommendation service"
LABEL org.opencontainers.image.version="1.2.3"
LABEL org.opencontainers.image.source="https://github.com/bookverse/bookverse-recommendations"
LABEL org.opencontainers.image.created="2024-01-15T10:30:00Z"
```

#### Recommendations Worker Image

```yaml
Name: recommendations-worker
Base Image: python:3.11-slim
Size: ~1.3GB
Purpose: Background processing
Registry: bookverse-recommendations-internal-docker-nonprod-local
Versioning: Independent SemVer (e.g., 1.2.4)
```

### Generic Artifacts

#### Configuration Bundle

```yaml
Name: recommendations-config-{version}.tar.gz
Content: YAML configuration files
Size: ~10KB
Format: TAR.GZ compressed archive
Registry: bookverse-recommendations-internal-generic-nonprod-local
Path: recommendations/config/{version}/
```

**Bundle Contents**:

```text
recommendations-config-1.2.1.tar.gz
├── recommendations-settings.yaml    # Algorithm configuration
├── feature-weights.yaml            # ML feature weights  
├── cache-config.yaml               # Caching settings
└── environment-overrides.yaml      # Environment-specific overrides
```

#### Resources Bundle

```yaml
Name: recommendations-resources-{version}.tar.gz
Content: ML models and data files
Size: ~50MB
Format: TAR.GZ compressed archive
Registry: bookverse-recommendations-internal-generic-nonprod-local
Path: recommendations/resources/{version}/
```

**Bundle Contents**:

```text
recommendations-resources-1.2.2.tar.gz
├── stopwords.txt                   # NLP stopwords
├── models/
│   ├── recommendation-model.pkl    # Trained model
│   └── similarity-matrix.npy      # Similarity data
└── training-data/
    ├── user-interactions.csv      # Historical data
    └── book-features.json         # Feature vectors
```

## 🛡️ Evidence Framework

### Evidence Types by Stage

#### Package-Level Evidence

Applied to individual artifacts (Docker images, generic packages).

##### 1. PyTest Results Evidence

```json
{
  "@type": "https://pytest.org/evidence/results/v1",
  "predicateType": "https://pytest.org/evidence/results/v1",
  "subject": [{"name": "recommendations", "digest": {"sha256": "1.2.3"}}],
  "predicate": {
    "test_framework": "pytest",
    "total_tests": 25,
    "passed_tests": 24,
    "failed_tests": 1,
    "coverage_percent": 85.5,
    "test_duration_seconds": 45
  }
}
```

##### 2. SAST Scan Evidence

```json
{
  "@type": "https://checkmarx.com/evidence/sast/v1.1",
  "predicateType": "https://checkmarx.com/evidence/sast/v1.1",
  "subject": [{"name": "recommendations", "digest": {"sha256": "1.2.3"}}],
  "predicate": {
    "scan_type": "SAST",
    "total_vulnerabilities": 3,
    "high_findings": 0,
    "medium_findings": 1,
    "low_findings": 2
  }
}
```

#### Build-Level Evidence

Applied to the overall build process.

##### 3. FOSSA License Scan

```json
{
  "scan_type": "license_compliance",
  "total_dependencies": 145,
  "license_violations": 0,
  "approved_licenses": ["MIT", "Apache-2.0", "BSD-3-Clause"],
  "scan_timestamp": "2024-01-15T10:30:00Z"
}
```

##### 4. SonarQube Quality Gate

```json
{
  "quality_gate": "PASSED",
  "code_coverage": 85.5,
  "duplicated_lines": 2.3,
  "maintainability_rating": "A",
  "reliability_rating": "A",
  "security_rating": "A"
}
```

#### Application-Level Evidence

Applied to the AppTrust application version.

##### 5. SLSA Provenance

```json
{
  "@type": "https://in-toto.io/Statement/v0.1",
  "predicateType": "https://slsa.dev/provenance/v0.2",
  "subject": [{"name": "bookverse-recommendations", "digest": {"version": "1.2.3"}}],
  "predicate": {
    "builder": {
      "id": "https://github.com/actions/runner"
    },
    "buildType": "https://github.com/actions/workflow",
    "materials": [
      {
        "uri": "git+https://github.com/bookverse/bookverse-recommendations",
        "digest": {"sha1": "abc123..."}
      }
    ]
  }
}
```

##### 6. Jira Release Evidence

```json
{
  "release_ticket": "BOOK-123",
  "approver": "architect@bookverse.com",
  "approval_date": "2024-01-15T10:30:00Z",
  "release_notes": "Added new recommendation algorithms",
  "change_type": "feature_enhancement"
}
```

#### Stage-Specific Evidence

##### DEV Stage - Smoke Tests

```json
{
  "test_type": "smoke_tests",
  "environment": "DEV",
  "health_checks": "PASSED",
  "api_endpoints": "RESPONSIVE",
  "database_connectivity": "VERIFIED"
}
```

##### QA Stage - DAST Scan & API Tests

```json
{
  "dast_scan": {
    "vulnerabilities_found": 0,
    "scan_duration": "15m",
    "pages_scanned": 25
  },
  "api_tests": {
    "total_tests": 50,
    "passed_tests": 50,
    "response_time_avg": "120ms"
  }
}
```

##### STAGING Stage - IaC Scan & Pentest

```json
{
  "iac_scan": {
    "security_violations": 0,
    "compliance_score": 95,
    "terraform_validated": true
  },
  "penetration_test": {
    "vulnerabilities": 0,
    "test_duration": "2h",
    "tester": "security@bookverse.com"
  }
}
```

##### PROD Stage - Deployment Verification

```json
{
  "deployment_status": "SUCCESS",
  "health_check_status": "HEALTHY",
  "monitoring_setup": "CONFIGURED",
  "rollback_plan": "VERIFIED"
}
```

## 🚨 Troubleshooting

### Common Issues & Solutions

#### 1. JFrog Authentication Failures

```bash
# Error: "Failed to authenticate with JFrog"
# Cause: OIDC provider misconfiguration

# Solution: Verify OIDC provider
jf rt curl -XGET "/access/api/v1/oidc/providers" | jq '.[] | select(.name=="bookverse-recommendations-github")'

# Expected Output:
{
  "name": "bookverse-recommendations-github",
  "issuer_url": "https://token.actions.githubusercontent.com",
  "audience": "https://apptrusttraining1.jfrog.io"
}
```

#### 2. Version Determination Errors

```bash
# Error: "Failed to determine APP_VERSION"
# Cause: Missing version-map.yaml or API connectivity

# Debug Steps:
1. Check version-map.yaml exists
2. Verify API connectivity
3. Check OIDC token validity

# Validation Commands:
ls -la config/version-map.yaml
curl -H "Authorization: Bearer $JF_OIDC_TOKEN" "$JFROG_URL/apptrust/api/v1/applications/bookverse-recommendations"
```

#### 3. Docker Build Failures

```bash
# Error: "Docker build failed"
# Cause: Missing dependencies or network issues

# Debug Steps:
1. Check Dockerfile syntax
2. Verify base image availability
3. Check network connectivity

# Debug Commands:
docker build --no-cache -t test-image .
jf rt ping
```

#### 4. Evidence Collection Issues

```bash
# Error: "Evidence creation failed"
# Cause: Missing evidence key or package not found

# Debug Steps:
1. Verify evidence private key
2. Check package exists in repository
3. Validate evidence format

# Validation Commands:
echo "$EVIDENCE_PRIVATE_KEY" | head -c 50
jf rt search "recommendations:*" --repos="*docker*"
```

### Debug Mode Activation

#### Enable Verbose Logging

```yaml
# Add to workflow for debugging
env:
  JFROG_CLI_LOG_LEVEL: DEBUG
  VERBOSE_MODE: true
  JF_LOG_LEVEL: DEBUG
```

#### Manual Workflow Execution

```yaml
# Use workflow_dispatch for testing
inputs:
  reason: "Debug authentication issue"
  force_app_version: true
```

### Performance Optimization

#### Build Time Optimization

```yaml
# Current timings:
analyze-commit: ~1 minute
build-test-publish: ~8 minutes  
create-promote: ~6 minutes
Total: ~15 minutes

# Optimization strategies:
1. Docker layer caching
2. Dependency caching
3. Parallel test execution
4. Evidence generation optimization
```

## 📋 Best Practices

### Repository Configuration

#### Required Files

```bash
# Mandatory configuration files
.github/workflows/ci.yml           # Main CI/CD workflow
.jfrog/config.yml                  # AppTrust application binding
config/version-map.yaml            # Version configuration
config/recommendations-settings.yaml # Service configuration
pyproject.toml                     # Python project configuration
Dockerfile                         # API image definition
Dockerfile.worker                  # Worker image definition
```

#### Security Practices

```yaml
# Secrets Management
- Use OIDC instead of long-lived tokens
- Rotate evidence signing keys regularly
- Scope permissions to minimum required
- Enable audit logging for all operations

# Access Control
- Limit repository access to necessary personnel
- Use branch protection rules
- Require code review for all changes
- Enable vulnerability scanning
```

### Workflow Maintenance

#### Regular Updates

```yaml
# Monthly tasks:
- Update action versions (actions/checkout@v4)
- Review and update base images
- Update Python dependencies
- Review security scan results

# Quarterly tasks:
- Rotate evidence signing keys
- Review repository permissions
- Update documentation
- Performance optimization review
```

#### Monitoring & Alerting

```yaml
# Key metrics to monitor:
- Build success rate (target: >95%)
- Build duration (target: <20 minutes)
- Evidence collection rate (target: 100%)
- Security scan compliance (target: 100%)

# Alert conditions:
- Build failure rate >5%
- Evidence collection failures
- Security vulnerabilities detected
- Performance degradation >25%
```

---

## 📚 Related Documentation

- [🏗️ CI/CD Architecture](./CI_CD_ARCHITECTURE.md) - Detailed system architecture
- [📖 Complete Service Guide](./SERVICE_GUIDE.md) - Comprehensive service documentation
- [🔧 Troubleshooting Guide](./TROUBLESHOOTING.md) - Common issues and solutions
- [🚀 CI/CD Overview](./CI_CD.md) - Quick pipeline overview

---

*Last Updated: 2024-01-15*
*Document Version: 1.0.0*
*Maintained by: BookVerse DevOps Team*
