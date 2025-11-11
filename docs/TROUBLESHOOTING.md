# 🔧 BookVerse Recommendations - Troubleshooting Guide

## 📋 Table of Contents

- [Quick Diagnostics](#quick-diagnostics)
- [Common Issues](#common-issues)
- [CI/CD Pipeline Issues](#cicd-pipeline-issues)
- [Authentication Problems](#authentication-problems)
- [Build Failures](#build-failures)
- [Evidence Collection Issues](#evidence-collection-issues)
- [Deployment Problems](#deployment-problems)
- [Performance Issues](#performance-issues)
- [Debug Tools](#debug-tools)
- [Support Escalation](#support-escalation)

## ⚡ Quick Diagnostics

### 🔍 System Health Check
```bash
# Basic service health
curl -s http://localhost:8000/health | jq .

# Detailed diagnostics
curl -s http://localhost:8000/api/v1/recommendations/health | jq .

# Worker status
docker ps | grep recommendations-worker

# Configuration validation
python -c "import yaml; print(yaml.safe_load(open('config/recommendations-settings.yaml')))"
```

### 🚀 CI/CD Pipeline Status
```bash
# Check latest workflow run
gh run list --workflow=ci.yml --limit=1

# View workflow logs
gh run view <run-id> --log

# Check repository variables
gh variable list

# Verify secrets (names only)
gh secret list
```

## 🐛 Common Issues

### 1. Service Won't Start

#### Symptoms
- Container exits immediately
- Health check endpoints return 500 errors
- "ModuleNotFoundError" in logs

#### Diagnosis
```bash
# Check container logs
docker logs <container-id>

# Verify Python dependencies
pip check

# Test configuration loading
python -c "from app.config import settings; print(settings)"
```

#### Solutions
```bash
# Fix dependency issues
pip install -r requirements.txt --force-reinstall

# Rebuild with fresh dependencies
docker build --no-cache -t bookverse-recommendations:debug .

# Check configuration format
yamllint config/recommendations-settings.yaml
```

### 2. Recommendation API Returns Empty Results

#### Symptoms
- API returns `{"recommendations": []}` for all requests
- No errors in logs but no results generated

#### Diagnosis
```bash
# Check inventory service connectivity
curl -s http://localhost:8001/health

# Verify recommendation settings
cat config/recommendations-settings.yaml

# Check indexer status
docker exec <container> python -c "from app.indexer import Indexer; i=Indexer(); print(i.get_stats())"
```

#### Solutions
```bash
# Update inventory service URL
export INVENTORY_BASE_URL="http://correct-inventory-url"

# Rebuild indices
curl -X POST http://localhost:8000/api/v1/recommendations/rebuild-index

# Reset cache
curl -X POST http://localhost:8000/api/v1/recommendations/clear-cache
```

### 3. Worker Process Failures

#### Symptoms
- Background tasks not processing
- High memory usage in worker container
- Worker container restarts frequently

#### Diagnosis
```bash
# Check worker logs
docker logs bookverse-recommendations-worker

# Monitor resource usage
docker stats bookverse-recommendations-worker

# Check task queue status
python -m app.worker --status
```

#### Solutions
```bash
# Increase worker memory limits
docker run -m 2g bookverse-recommendations-worker

# Optimize worker configuration
# Edit config/worker-settings.yaml
batch_size: 100  # Reduce if memory issues
worker_threads: 2  # Reduce for single-core systems

# Restart worker with fresh state
docker restart bookverse-recommendations-worker
```

## 🚀 CI/CD Pipeline Issues

### 1. Workflow Fails at Commit Analysis

#### Error Messages
```
Error: analyze-commit.sh: command not found
Error: Failed to determine create_app_version
```

#### Diagnosis
```bash
# Check if bookverse-infra checkout succeeded
ls -la bookverse-infra/libraries/bookverse-devops/scripts/

# Verify script permissions
ls -la bookverse-infra/libraries/bookverse-devops/scripts/analyze-commit.sh
```

#### Solutions
```yaml
# Add explicit permission setting in workflow
- name: "[Setup] Fix script permissions"
  run: |
    find bookverse-infra/libraries/bookverse-devops/scripts/ -name "*.sh" -exec chmod +x {} \;
```

### 2. Build Job Fails at SemVer Determination

#### Error Messages
```
Error: Failed to extract APP_VERSION from semver output
Error: Missing JF_OIDC_TOKEN
```

#### Diagnosis
```bash
# Check OIDC token exchange
echo "Token length: ${#JF_OIDC_TOKEN}"

# Verify version-map.yaml format
cat config/version-map.yaml | yq .

# Test semver script manually
./bookverse-infra/libraries/bookverse-devops/scripts/determine-semver.sh --help
```

#### Solutions
```yaml
# Add token validation
- name: "[Debug] Validate OIDC token"
  run: |
    if [[ -z "${JF_OIDC_TOKEN:-}" ]]; then
      echo "❌ Missing OIDC token"
      exit 1
    fi
    echo "✅ OIDC token present (length: ${#JF_OIDC_TOKEN})"

# Fix version-map.yaml format
packages:
  recommendations:
    current_version: "1.0.0"
    version_strategy: "semantic"
```

### 3. Docker Build Failures

#### Error Messages
```
Error: failed to solve: failed to copy files
Error: unable to prepare context: unable to evaluate symlinks
```

#### Diagnosis
```bash
# Check Docker build context
docker build --dry-run .

# Verify Dockerfile syntax
docker build --target=<stage> .

# Check for large files in context
find . -type f -size +100M
```

#### Solutions
```dockerfile
# Add .dockerignore to exclude large files
htmlcov/
*.egg-info/
__pycache__/
.git/

# Fix file permissions
RUN chmod +x /app/entrypoint.sh

# Use multi-stage builds for optimization
FROM python:3.11-slim as base
FROM base as dependencies
FROM dependencies as runtime
```

### 4. Evidence Collection Failures

#### Error Messages
```
Error: Evidence creation failed
Error: Package not found in repository
```

#### Diagnosis
```bash
# Check if packages exist
jf rt search "recommendations:*" --repos="*docker*"

# Verify evidence key
echo "$EVIDENCE_PRIVATE_KEY" | openssl rsa -in - -text -noout

# Test evidence creation manually
jf evd create-evidence --help
```

#### Solutions
```yaml
# Add package existence validation
- name: "[Debug] Verify package existence"
  run: |
    PACKAGES=$(jf rt search "recommendations:$RECOMMENDATIONS_VERSION" --repos="*docker*")
    if [[ -z "$PACKAGES" ]]; then
      echo "❌ Package not found"
      exit 1
    fi
    echo "✅ Package found: $PACKAGES"

# Fix evidence key format
- name: "[Debug] Validate evidence key"
  run: |
    echo "$EVIDENCE_PRIVATE_KEY" | openssl rsa -in - -check -noout
```

## 🔐 Authentication Problems

### 1. JFrog OIDC Authentication Fails

#### Error Messages
```
Error: Failed to authenticate with JFrog
Error: OIDC provider not found
```

#### Diagnosis
```bash
# Check OIDC provider configuration
jf rt curl -XGET "/access/api/v1/oidc/providers" | jq '.[] | select(.name=="bookverse-recommendations-github")'

# Verify GitHub OIDC token
curl -H "Authorization: bearer $ACTIONS_ID_TOKEN" $ACTIONS_ID_TOKEN_REQUEST_URL
```

#### Solutions
```bash
# Create OIDC provider if missing
jf rt curl -XPOST "/access/api/v1/oidc/providers" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "bookverse-recommendations-github",
    "issuer_url": "https://token.actions.githubusercontent.com",
    "audience": "'$JFROG_URL'"
  }'

# Update provider configuration
jf rt curl -XPUT "/access/api/v1/oidc/providers/bookverse-recommendations-github" \
  -H "Content-Type: application/json" \
  -d '{
    "audience": "'$JFROG_URL'",
    "claims": {
      "repository": "bookverse/bookverse-recommendations"
    }
  }'
```

### 2. AppTrust API Authentication Issues

#### Error Messages
```
Error: 401 Unauthorized
Error: Invalid token
```

#### Diagnosis
```bash
# Test token validity
curl -H "Authorization: Bearer $JF_OIDC_TOKEN" \
     "$JFROG_URL/apptrust/api/v1/applications" | jq .

# Check token expiration
echo "$JF_OIDC_TOKEN" | base64 -d | jq .exp
```

#### Solutions
```bash
# Refresh OIDC token
./bookverse-infra/libraries/bookverse-devops/scripts/exchange-oidc-token.sh \
  --service-name "recommendations" \
  --provider-name "bookverse-recommendations-github" \
  --jfrog-url "$JFROG_URL"

# Validate token permissions
jf rt curl -XGET "/access/api/v1/oidc/tokens/verify" \
  -H "Authorization: Bearer $JF_OIDC_TOKEN"
```

## 📦 Build Failures

### 1. Python Dependency Installation Failures

#### Error Messages
```
Error: Could not find a version that satisfies the requirement
Error: pip install failed
```

#### Diagnosis
```bash
# Check PyPI connectivity
jf rt ping

# Verify pip configuration
jf pip-config --repo-resolve "bookverse-pypi-virtual"

# Test manual installation
pip install pytest==8.3.2 --index-url https://swampupsec.jfrog.io/artifactory/api/pypi/bookverse-pypi-virtual/simple
```

#### Solutions
```bash
# Fix pip configuration
jf pip-config --repo-resolve "bookverse-pypi-virtual"

# Use fallback to public PyPI
pip install -r requirements.txt --index-url https://pypi.org/simple/

# Clear pip cache
pip cache purge
```

### 2. Test Execution Failures

#### Error Messages
```
Error: No tests ran
Error: pytest command not found
```

#### Diagnosis
```bash
# Check pytest installation
which pytest
pytest --version

# Verify test files exist
find tests/ -name "test_*.py"

# Check test configuration
cat pytest.ini
```

#### Solutions
```bash
# Install test dependencies
pip install pytest pytest-cov

# Fix test discovery
pytest tests/ -v --collect-only

# Update pytest configuration
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
```

## 🚀 Deployment Problems

### 1. Application Version Creation Fails

#### Error Messages
```
Error: Failed to create application version
Error: HTTP 409 - Version already exists
```

#### Diagnosis
```bash
# Check existing versions
curl -H "Authorization: Bearer $JF_OIDC_TOKEN" \
     "$JFROG_URL/apptrust/api/v1/applications/bookverse-recommendations/versions" | jq .

# Verify build-info exists
jf rt search "*" --build="$BUILD_NAME/$BUILD_NUMBER"
```

#### Solutions
```bash
# Handle version conflicts
if [[ "$HTTP_STATUS" == "409" ]]; then
  echo "📝 Version already exists, continuing..."
else
  echo "❌ Unexpected error: $HTTP_STATUS"
  exit 1
fi

# Force version increment
export FORCE_VERSION_INCREMENT=true
```

### 2. Promotion Failures

#### Error Messages
```
Error: Failed to promote to DEV
Error: Evidence validation failed
```

#### Diagnosis
```bash
# Check promotion status
curl -H "Authorization: Bearer $JF_OIDC_TOKEN" \
     "$JFROG_URL/apptrust/api/v1/applications/bookverse-recommendations/versions/$APP_VERSION" | jq .

# Verify evidence exists
jf evd list --package-name recommendations --package-version $RECOMMENDATIONS_VERSION
```

#### Solutions
```bash
# Re-attach missing evidence
jf evd create-evidence \
  --predicate pytest-results.json \
  --predicate-type "https://pytest.org/evidence/results/v1" \
  --package-name recommendations \
  --package-version $RECOMMENDATIONS_VERSION

# Retry promotion
source bookverse-infra/libraries/bookverse-devops/scripts/evidence-lib.sh
advance_one_step
```

## 📊 Performance Issues

### 1. Slow API Response Times

#### Symptoms
- API responses >1 second
- High CPU usage
- Memory leaks

#### Diagnosis
```bash
# Profile API endpoints
curl -w "%{time_total}\n" -o /dev/null -s http://localhost:8000/api/v1/recommendations/similar?book_id=123

# Check memory usage
docker stats bookverse-recommendations

# Monitor cache hit rates
curl http://localhost:8000/api/v1/recommendations/stats | jq .cache
```

#### Solutions
```python
# Optimize configuration
cache:
  ttl_seconds: 3600      # Increase cache duration
  max_entries: 50000     # Increase cache size

algorithm:
  max_recommendations: 10  # Reduce computation
  similarity_threshold: 0.2  # Filter low-quality results
```

### 2. High Build Times

#### Symptoms
- CI/CD pipeline takes >20 minutes
- Docker builds are slow
- Test execution is slow

#### Optimization Strategies
```yaml
# Enable Docker layer caching
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v2
  with:
    buildkitd-flags: --allow-insecure-entitlement network.host

# Parallel test execution
- name: Run tests in parallel
  run: |
    pytest -n auto --dist worksteal
```

## 🔧 Debug Tools

### CI/CD Debugging

#### Enable Debug Mode
```yaml
# Add to workflow environment
env:
  JFROG_CLI_LOG_LEVEL: DEBUG
  VERBOSE_MODE: true
  RUNNER_DEBUG: 1
```

#### Manual Workflow Trigger
```bash
# Trigger with debug options
gh workflow run ci.yml \
  -f reason="Debug authentication issue" \
  -f force_app_version=true
```

### Local Debugging

#### Docker Debug Container
```bash
# Start debug container
docker run -it --entrypoint /bin/bash bookverse-recommendations:latest

# Mount local code for debugging
docker run -it -v $(pwd):/app bookverse-recommendations:latest bash
```

#### Python Debugging
```python
# Add to code for debugging
import pdb; pdb.set_trace()

# Or use logging
import logging
logging.basicConfig(level=logging.DEBUG)
```

### JFrog Debugging

#### CLI Debug Commands
```bash
# Enable debug logging
export JFROG_CLI_LOG_LEVEL=DEBUG

# Test connectivity
jf rt ping --verbose

# Check configuration
jf config show

# Search for artifacts
jf rt search "*recommendations*" --recursive
```

## 📞 Support Escalation

### Escalation Matrix

| Issue Type | First Contact | Escalation 1 | Escalation 2 |
|------------|---------------|--------------|--------------|
| **Service Issues** | Backend Team | Platform Team | Architecture Team |
| **CI/CD Pipeline** | DevOps Team | Platform Team | Infrastructure Team |
| **Security/Evidence** | Security Team | Compliance Team | CISO |
| **JFrog Issues** | DevOps Team | JFrog Support | Platform Team |
| **Performance** | Backend Team | SRE Team | Architecture Team |

### Information to Gather

#### For Service Issues
```bash
# Service information
curl -s http://localhost:8000/info | jq .
docker logs bookverse-recommendations --tail=100
docker inspect bookverse-recommendations

# Configuration
cat config/recommendations-settings.yaml
env | grep -E "(INVENTORY|RECO_|SERVICE)"
```

#### For CI/CD Issues
```bash
# Workflow information
gh run view <run-id> --log
gh workflow view ci.yml

# Repository configuration
gh variable list
gh secret list (names only)
cat .github/workflows/ci.yml | head -20
```

#### For Authentication Issues
```bash
# JFrog information
jf config show
jf rt ping
jf access-token-create --expiry=3600

# OIDC information (DO NOT SHARE TOKEN VALUES)
echo "Token present: $([[ -n "$JF_OIDC_TOKEN" ]] && echo "YES" || echo "NO")"
echo "Token length: ${#JF_OIDC_TOKEN}"
```

### Contact Information

#### Internal Teams
- **DevOps Team**: devops@bookverse.com (Slack: #devops)
- **Backend Team**: backend@bookverse.com (Slack: #backend)
- **Security Team**: security@bookverse.com (Slack: #security)
- **Platform Team**: platform@bookverse.com (Slack: #platform)

#### External Support
- **JFrog Support**: [JFrog Support Portal](https://jfrog.com/support/)
- **GitHub Support**: [GitHub Support](https://support.github.com/)
- **AppTrust Support**: platform@bookverse.com

---

## 📚 Additional Resources

- [🏗️ CI/CD Architecture](./CI_CD_ARCHITECTURE.md) - Complete system design and architecture
- [📋 Workflow Reference](./WORKFLOW_REFERENCE.md) - Detailed workflow steps and configuration
- [📖 Complete Service Guide](./SERVICE_GUIDE.md) - Comprehensive service documentation
- [🚀 CI/CD Overview](./CI_CD.md) - Quick pipeline overview

---

*Last Updated: 2024-01-15*
*Document Version: 1.0.0*
*Maintained by: BookVerse DevOps Team*
