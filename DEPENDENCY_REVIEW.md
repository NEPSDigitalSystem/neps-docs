# Dependency Management Guide

## Overview

This document outlines the industry-standard dependency management approach for the NEPS Digital System, including automated security checks, review processes, and merge guidelines.

## Philosophy

**Manual review for production dependencies, automated checks for dev dependencies**

Given the healthcare/medical data domain and complex multi-service architecture, we prioritize security and stability over automation speed.

## Dependabot Configuration

### Configuration Strategy

Each service has a tiered Dependabot configuration:

1. **Production Dependencies** - Require manual review
   - Patch updates grouped together (bug fixes)
   - Minor updates grouped by critical packages
   - Major updates always individual
   - Label: `dependencies`, `production`, `[service]`

2. **Development Dependencies** - Semi-automated
   - All dev deps grouped together
   - Patch updates can auto-approve if CI passes
   - Label: `dependencies`, `dev`, `[service]`

3. **Docker Images** - Manual review
   - Base image updates grouped
   - Require testing of full stack
   - Label: `dependencies`, `docker`

4. **GitHub Actions** - Manual review
   - CI/CD pipeline changes grouped
   - Require careful review
   - Label: `dependencies`, `ci-cd`

### Services with Configurations

- `neps-portal` - npm (frontend)
- `neps-backend` - pip (Python backend)
- `neps-data-platform` - pip (Python ETL)
- `neps-ml-ai` - pip (Python ML/AI)
- `neps-infrastructure` - docker, github-actions

## Automated Workflows

### 1. Dependency Review Workflow

Each service has a `.github/workflows/dependency-review.yml` that runs on pull requests:

**Jobs:**
- `dependency-review` - GitHub's dependency review action
  - Non-blocking (continue-on-error: true)
  - Fails on moderate severity vulnerabilities (if Dependency Graph is enabled)
  - Blocks GPL-3.0 and AGPL-3.0 licenses (copyleft licenses)
  - Allows all other licenses (MIT, Apache-2.0, BSD, ISC, etc.)
  - **Note**: Requires Dependency Graph to be enabled in repo settings
    - To enable: Go to repo Settings > Security & Analysis > Dependency Graph

- `npm-audit` / `pip-audit` - Language-specific security audit
  - Runs npm audit for Node.js projects
  - Runs pip-audit for Python projects
  - Continues on error (doesn't block, but reports)

- `check-patch-updates` - Determines update type
  - Checks if PR is from dependabot
  - Identifies patch-only updates
  - Distinguishes production vs dev dependencies

- `auto-approve-dev-deps` - Conditional auto-approval
  - Only runs for dependabot PRs
  - Only for patch updates to dev dependencies
  - Only if security checks pass
  - Adds comment: "✅ Safe to merge after CI passes"

### 2. CI/CD Integration

Existing CI/CD workflows include:

- **Security scanning** - Trivy vulnerability scanner
  - PR: filesystem scan (report only)
  - Push: image scan (fails on CRITICAL/HIGH)

- **Testing** - Automated test suite
  - All services run tests before building
  - Tests must pass for merge consideration

## Review Checklist

### For All Dependency Updates

- [ ] CI/CD pipeline passes
- [ ] Security scans pass (Trivy, npm/pip audit)
- [ ] Dependency review passes (no blocked licenses)
- [ ] Review changelog for breaking changes
- [ ] Test locally in development environment

### For Production Dependencies (Manual Review Required)

- [ ] Security advisories reviewed
- [ ] Breaking changes documented
- [ ] Backward compatibility verified
- [ ] Docker images build successfully
- [ ] Integration tests pass
- [ ] No conflicts with NDS_Team changes (if applicable)
- [ ] Database migrations reviewed (if applicable)
- [ ] Performance impact assessed

### For Development Dependencies (Semi-Automated)

- [ ] Auto-approve if: patch-only + dev deps + security checks pass
- [ ] Manual review if: minor/major updates or security failures
- [ ] Test build and linting pass

### For Docker Images

- [ ] Full stack tested with new base image
- [ ] All services start successfully
- [ ] No performance degradation
- [ ] Security vulnerabilities in base image reviewed

### For GitHub Actions

- [ ] Workflow changes reviewed carefully
- [ ] No breaking changes to CI/CD pipeline
- [ ] Secrets access unchanged
- [ ] Permissions unchanged
- [ ] Test on feature branch first

## Merge Guidelines

### Auto-Merge Criteria (Dev Dependencies Only)

A dependency PR **CAN** be auto-merged if:

1. ✅ From dependabot[bot]
2. ✅ Only patch updates (e.g., 1.2.3 → 1.2.4)
3. ✅ Development dependencies only (not production)
4. ✅ All security checks pass
5. ✅ CI/CD pipeline passes
6. ✅ No blocked licenses

### Manual Review Required

Manual review is REQUIRED for:

- ❌ Any production dependency update
- ❌ Minor or major version updates
- ❌ Docker base image updates
- ❌ GitHub Actions updates
- ❌ Security-critical packages (auth, crypto, database drivers)
- ❌ Any update that fails security checks
- ❌ Updates with blocked licenses
- ❌ Updates with breaking changes

## Security-Critical Packages

These packages always require manual review:

### Backend (Python)
- pyjwt (authentication)
- cryptography (encryption)
- passlib (password hashing)
- sqlalchemy (ORM)
- alembic (migrations)
- python-multipart (file uploads)

### Data Platform (Python)
- pyjwt (authentication)
- cryptography (encryption)
- sqlalchemy (ORM)
- alembic (migrations)
- requests (HTTP client)
- pandas (data processing)

### Portal (Node.js)
- next (framework)
- react (framework)
- next-auth (authentication)

## Emergency Security Updates

For CRITICAL security vulnerabilities (CVSS 9.0+):

1. Immediate manual review
2. Apply patch regardless of version type
3. Test in staging environment
4. Deploy to production with monitoring
5. Document in security incident log

## Rollback Procedure

If a dependency update causes issues:

1. Identify the problematic PR
2. Revert the commit
3. Push to main/staging
4. CI/CD will automatically deploy previous version
5. Investigate the issue in a feature branch
6. Apply fix or wait for upstream patch

## Monitoring

### Alerts to Monitor

- Dependabot security alerts
- GitHub Security tab
- CI/CD build failures
- Application error rates post-deploy
- Performance metrics post-deploy

### Review Cadence

- **Weekly**: Review all open dependency PRs
- **Monthly**: Review security advisories
- **Quarterly**: Review base images and major versions

## Best Practices

1. **Stay Updated**: Review dependency PRs weekly to prevent backlog
2. **Test Locally**: Always test locally before merging production deps
3. **Read Changelogs**: Understand what changed in each update
4. **Security First**: Never skip security checks for convenience
5. **Incremental Updates**: Prefer patch updates over major jumps
6. **Pin Critical Versions**: Consider pinning security-critical packages
7. **Document Decisions**: Note why specific updates were blocked/accepted

## Prerequisites

### Enable Dependency Graph (Required for Dependency Review)

For the dependency review workflow to work, you must enable the Dependency Graph in each repository:

1. Go to repository Settings
2. Navigate to **Security & Analysis**
3. Enable **Dependency Graph**
4. Click **Configure Dependency Graph** and select the appropriate settings

If Dependency Graph is not enabled, the dependency review job will be skipped (non-blocking) and CI will continue with other checks.

## References

- [Dependabot Documentation](https://docs.github.com/en/code-security/dependabot)
- [Dependency Review Action](https://github.com/actions/dependency-review-action)
- [Enabling Dependency Graph](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph)
- [Trivy Scanner](https://aquasecurity.github.io/trivy/)
- [Semantic Versioning](https://semver.org/)
- [OWASP Dependency Check](https://owasp.org/www-project-dependency-check/)

## Contact

For questions about dependency management:
- DevOps team
- Security team
- Service maintainers
