# gha-ecr-deploy

[![CI](https://github.com/Natiwo/gha-ecr-deploy/actions/workflows/ci.yml/badge.svg)](https://github.com/Natiwo/gha-ecr-deploy/actions/workflows/ci.yml)
[![Security](https://github.com/Natiwo/gha-ecr-deploy/actions/workflows/security.yml/badge.svg)](https://github.com/Natiwo/gha-ecr-deploy/actions/workflows/security.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Production-grade CI/CD pattern for GitHub Actions with AWS ECR.

## Overview

A complete, battle-tested CI/CD pipeline that implements:

- **Artifact caching** between CI and CD (no rebuild)
- **Environment gates** (staging auto, production with approval)
- **Automatic rollback** on deployment failure
- **Security scanning** (secrets, vulnerabilities, IaC)
- **SBOM generation** for compliance
- **Cleanup automation** to prevent registry bloat

## Quick Start

1. **Use this template** or copy the `.github/workflows/` directory
2. Configure required secrets (see [Configuration](#configuration))
3. Create GitHub environments (`staging`, `production`)
4. Push to `main` for staging, tag `v*` for production

## Architecture

```
                              CI Pipeline
+-------------------------------------------------------------------------+
|  +-------------+  +-------------+                                       |
|  |   Quality   |  |  Security   |  (parallel)                           |
|  |   Gates     |  |    Scan     |                                       |
|  +------+------+  +------+------+                                       |
|         +--------+-------+                                              |
|                  v                                                      |
|         +---------------+                                               |
|         |  Build Image  |  --> Upload Artifact                          |
|         |  (matrix)     |                                               |
|         +-------+-------+                                               |
|                 v                                                       |
|         +---------------+                                               |
|         |  Test Image   |  <-- Download Artifact                        |
|         |  (matrix)     |                                               |
|         +-------+-------+                                               |
|                 v                                                       |
|         +---------------+                                               |
|         |   Summary     |                                               |
|         +---------------+                                               |
+-------------------------------------------------------------------------+
                                   |
                                   | workflow_run (on success)
                                   v
                              CD Pipeline
+-------------------------------------------------------------------------+
|         +---------------+                                               |
|         |   Prepare     |  --> Download CI Artifacts                    |
|         +-------+-------+                                               |
|                 |                                                       |
|    +------------+------------+                                          |
|    v                         v                                          |
| +--------------+    +--------------+                                    |
| |   Staging    |    |  Production  |  (environment gates)               |
| |   (auto)     |    |  (approval)  |                                    |
| +------+-------+    +------+-------+                                    |
|        |                   |                                            |
|        |                   +----------+                                 |
|        |                   |          |                                 |
|        v                   v          v                                 |
| +-------------+    +-------------+  +----------+                        |
| | Smoke Test  |    | Smoke Test  |  | Rollback |  (on failure)          |
| +-------------+    +------+------+  +----------+                        |
|                          |                                              |
|                          v                                              |
|                   +-------------+                                       |
|                   |  Release    |  (on tag)                             |
|                   +-------------+                                       |
+-------------------------------------------------------------------------+
```

## Comparison with Branching Strategies

### Git Flow

```
main -------------------------------------------------->
  |                                    ^
  |                                    | merge
  v                                    |
develop --+----------+----------------+--------------->
          |          |          ^
          v          v          |
       feature/   feature/   release/
         foo        bar       1.0
```

| Aspect | Git Flow | This Pattern |
|--------|----------|--------------|
| Branches | main, develop, feature/*, release/*, hotfix/* | main only |
| Complexity | High (5+ branch types) | Low (1 branch) |
| Staging deploy | develop branch | Push to main |
| Production deploy | Merge to main via release branch | Tag v* |
| Rollback | Hotfix branch + merge | Automatic via CD |
| Overhead | High (many merges) | Minimal |

**Verdict**: Git Flow was designed for software with discrete releases (v1.0, v2.0). Excessive ceremony for continuous deployment.

### GitHub Flow

```
main ----+--------+--------+----------------------------->
         |        |        |
         v        v        v
      feature  feature  feature
         |        |        |
         +--------+--------+
              PR + merge
```

| Aspect | GitHub Flow | This Pattern |
|--------|-------------|--------------|
| Branches | main + feature branches | main only |
| Deploy trigger | Merge to main | Push to main |
| Single environment | Yes (production) | No (staging + prod) |
| Rollback | Revert commit | Automatic |
| Feature flags | Required for staging | Not required |

**Verdict**: GitHub Flow is simple but assumes direct production deployment. Requires feature flags for staging testing. This pattern adds the staging layer without extra branches.

### Branch-Environment Pattern

```
main ----------------------------------------------------->
  |
  +---> staging ------------------------------------------>
  |         |
  +---> production --------------------------------------->
```

| Aspect | Branch-Environment | This Pattern |
|--------|-------------------|--------------|
| Branches | 1 per environment | 1 total |
| Branch sync | Manual (cherry-pick/merge) | Automatic |
| Drift risk | High | Zero |
| Staging deploy | Push to staging branch | Push to main |
| Production deploy | Push/merge to prod branch | Tag v* |

**Verdict**: Branch-Environment creates drift between environments. Staging code can diverge from production. This pattern guarantees production always receives exactly what passed through staging.

### Comparison Matrix

```
                    Complexity    Drift Risk    Staging    Auto-Rollback
                    ----------    ----------    -------    -------------
Git Flow            High          Medium        Yes        No
GitHub Flow         Low           Zero          No*        No
Branch-Env          Medium        High          Yes        No
This Pattern        Low           Zero          Yes        Yes

* Requires feature flags
```

## Flow Diagram

```
Developer                    CI                      CD
    |                        |                       |
    |  push main             |                       |
    +----------------------->|                       |
    |                        |  build + test         |
    |                        +---------------------->|
    |                        |                       |  staging (auto)
    |                        |                       |
    |  git tag v1.0.0        |                       |
    +----------------------->|                       |
    |                        |  (reuse artifacts)    |
    |                        +---------------------->|
    |                        |                       |  production
    |                        |                       |
    |                        |     [smoke test]      |
    |                        |<----------------------+
    |                        |                       |
    |                        |     [rollback if      |
    |                        |      failure]         |
```

## Configuration

### Required Secrets

| Secret | Description |
|--------|-------------|
| `AWS_ROLE_ARN` | IAM role ARN for OIDC authentication |
| `AWS_ACCOUNT_ID` | AWS account ID (optional, for documentation) |

### Optional Secrets

| Secret | Description |
|--------|-------------|
| `SLACK_WEBHOOK_URL` | Slack webhook for notifications |
| `DISCORD_WEBHOOK_URL` | Discord webhook for notifications |

### Environment Variables

Edit these in the workflow files:

```yaml
env:
  AWS_REGION: us-east-1           # Your AWS region
  ECR_REPOSITORY: your-org/repo   # Your ECR repository name
```

### GitHub Environments

Create two environments in Settings > Environments:

#### `staging`
- No protection rules
- Automatic deployment

#### `production`
- Required reviewers (optional)
- Deployment branches: `main` and tags `v*`

### IAM Role for OIDC

Trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

Minimum permissions policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ecr:GetAuthorizationToken"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:DescribeImages",
        "ecr:ListImages",
        "ecr:BatchDeleteImage"
      ],
      "Resource": "arn:aws:ecr:*:ACCOUNT_ID:repository/REPO_NAME"
    }
  ]
}
```

## Workflows

### ci.yml

Handles quality gates, security scanning, building, and testing.

Key features:
- Parallel quality and security jobs
- Matrix build for multiple targets
- Artifact upload for CD reuse
- Comprehensive test suite

### cd.yml

Handles deployment to staging and production.

Key features:
- Artifact download (no rebuild)
- Environment-based routing
- Automatic rollback on failure
- GitHub Release creation on tags
- Image cleanup automation

### security.yml

Scheduled and on-demand security scanning.

Key features:
- Secret detection (Trivy, TruffleHog)
- Vulnerability scanning (Trivy, Grype)
- IaC security (Trivy, Checkov)
- SBOM generation
- Automatic issue creation on findings

### notify.yml

Reusable workflow for deployment notifications.

Key features:
- Slack integration
- Discord integration
- Callable from other workflows

## Customization

### Adding Build Targets

In `ci.yml` and `cd.yml`:

```yaml
strategy:
  matrix:
    target: [minimal, runtime, your-target]
```

### Custom Smoke Tests

In `cd.yml`, modify the smoke test step:

```yaml
- name: Smoke test
  run: |
    docker run --rm $IMAGE bash -c '
      # Your custom tests here
      curl -sf http://localhost:8080/health
    '
```

### Adding Notifications

Configure `SLACK_WEBHOOK_URL` or `DISCORD_WEBHOOK_URL` secrets, then call from other workflows:

```yaml
notify:
  needs: [deploy]
  if: always()
  uses: ./.github/workflows/notify.yml
  with:
    status: ${{ needs.deploy.result }}
    environment: production
    version: ${{ needs.prepare.outputs.version }}
  secrets: inherit
```

## When to Use

| Scenario | Recommended Strategy |
|----------|---------------------|
| Packaged software (libs, CLIs) | Git Flow |
| Simple SaaS, small team | GitHub Flow |
| Strict compliance, audit trail | Branch-Environment |
| **Continuous deployment, multiple environments** | **This Pattern** |

## Troubleshooting

### CD not triggering after CI

1. Verify CI workflow name is exactly `CI`
2. Check CI completed successfully
3. Verify `workflow_run` configuration

### Artifacts not found

1. Artifacts expire in 1 day by default
2. Verify CI uploaded correctly
3. Check `run_id` in prepare job

### Rollback not working

1. Requires a previous image with `latest-{target}` tag
2. Rollback only triggers on production deployment failure
3. Verify ECR permissions

### Environment protection not appearing

1. Create environment in Settings > Environments
2. Add required reviewers
3. Verify job has `environment: production`

## Contributing

Contributions are welcome. Please feel free to submit issues and pull requests.

Before submitting:
1. Test your changes with a real AWS account
2. Update documentation if needed
3. Follow existing code style

## License

MIT License - see [LICENSE](LICENSE) for details.

---

This pattern was developed as a practical solution for teams needing reliable, automated deployments without the complexity of multi-branch strategies. We welcome feedback and suggestions for improvement.
