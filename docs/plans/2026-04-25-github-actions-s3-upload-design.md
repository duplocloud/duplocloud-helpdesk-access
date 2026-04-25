# GitHub Actions S3 Upload Pipeline Design

**Date**: 2026-04-25  
**Status**: Approved

## Overview

Create two GitHub Actions workflows to automate uploading the `aws.yaml` CloudFormation template to S3. The dev pipeline uploads versioned files for testing, while the publish pipeline uploads the canonical production file.

## Requirements

### Business Goals
- Enable testing of CloudFormation templates from S3 before publishing
- Automate the manual S3 upload process mentioned in README line 84-94
- Provide clear separation between development and production deployments

### Technical Requirements
- Upload to S3 bucket: `duploservices-ai-access-227120241369`
- Use DuploCloud JIT AWS credentials via `duploctl`
- Dev pipeline: versioned filenames with branch and commit SHA
- Publish pipeline: canonical `aws.yaml` filename
- No manual triggers for dev pipeline
- Manual trigger implicit via PR merge for publish pipeline

## Branch Strategy

### Branch Structure
- **`main`** - Production branch, triggers publish workflow
- **`dev`** - Development branch (to be created), triggers dev workflow
- **Feature branches** - Created from `dev`, trigger dev workflow via PRs

### Workflow
1. Developer creates feature branch from `dev`
2. Opens PR to `dev` → dev pipeline uploads `aws-{branch}-{sha}.yaml`
3. PR merged to `dev` → dev pipeline uploads `aws-dev-{sha}.yaml`
4. When ready for production, open PR from `dev` to `main`
5. PR merged to `main` → publish pipeline uploads `aws.yaml`

## Architecture

### Approach: Two Separate Workflow Files

**Selected Approach**: Create two independent workflow files for clarity and maintainability.

**Alternatives Considered**:
- Single workflow with conditionals (rejected: too complex)
- Reusable workflow pattern (rejected: overkill for simple workflows)

### Workflow Triggers

#### Dev Upload Workflow (`.github/workflows/dev-upload.yml`)
```yaml
on:
  push:
    branches: [dev]
  pull_request:
    branches: [dev]
    types: [opened, synchronize, reopened]
```

#### Publish Workflow (`.github/workflows/publish.yml`)
```yaml
on:
  push:
    branches: [main]
```

## Workflow Design

### Common Steps (Both Workflows)

1. **Checkout code**: `actions/checkout@v4`
2. **Setup DuploCtl**: `duplocloud/actions/setup@v1`
   - `duplo_host`: `https://prod.duplocloud.net`
   - `duplo_token`: `${{ secrets.DUPLO_PROD_TOKEN }}`
   - `DUPLO_TENANT_ID`: `5cd3b897-e4d2-4049-85b8-8dd9ab5232de`
3. **Get AWS credentials**: Automatically provided by DuploCtl setup
4. **Upload to S3**: `aws s3 cp aws.yaml s3://duploservices-ai-access-227120241369/{filename}`

### Dev Workflow Filename Logic

Generate versioned filename using bash expression:
```bash
FILENAME="aws-${GITHUB_HEAD_REF:-${GITHUB_REF#refs/heads/}}-${GITHUB_SHA:0:7}.yaml"
```

**Logic**:
- For PRs: Use `GITHUB_HEAD_REF` (source branch name)
- For direct commits: Extract branch name from `GITHUB_REF`
- Append 7-character short SHA from `GITHUB_SHA`

**Examples**:
- PR from `feature-eks-admin`: `aws-feature-eks-admin-a1b2c3d.yaml`
- Commit to `dev`: `aws-dev-e4f5a6b.yaml`

### Publish Workflow Filename

Fixed filename: `aws.yaml`

## Configuration

### GitHub Secrets Required
- `DUPLO_PROD_TOKEN`: DuploCloud portal authentication token

### S3 Configuration
- **Bucket**: `duploservices-ai-access-227120241369`
- **Region**: Default from JIT credentials
- **ACL**: Bucket already public, no ACL flag needed
- **Content-Type**: Auto-detected by S3

### Permissions
- Standard `GITHUB_TOKEN` permissions
- No additional repository permissions required
- S3 write access provided by DuploCloud JIT credentials

## Error Handling

### Strategy
- Use default GitHub Actions failure behavior (fail fast, show errors)
- No automatic retries (manual workflow re-run if needed)
- Clear error messages from AWS CLI and DuploCtl

### Failure Scenarios
- **DuploCtl setup fails**: Workflow fails before S3 upload attempt
- **S3 upload fails**: Workflow fails with AWS CLI error message
- **Invalid credentials**: Workflow fails with authentication error

## Testing Strategy

### Dev Pipeline Testing
1. Create `dev` branch
2. Create test feature branch from `dev`
3. Open PR to `dev`
4. Verify workflow runs and uploads `aws-{branch}-{sha}.yaml` to S3
5. Merge PR to `dev`
6. Verify workflow uploads `aws-dev-{sha}.yaml` to S3

### Publish Pipeline Testing
1. Create PR from `dev` to `main` (or commit directly)
2. Merge to `main`
3. Verify workflow runs and uploads `aws.yaml` to S3
4. Verify file is accessible from CloudFormation quick-create URL

## Implementation Checklist

- [ ] Create `.github/workflows/` directory
- [ ] Create `dev-upload.yml` workflow file
- [ ] Create `publish.yml` workflow file
- [ ] Create `dev` branch from current `main`
- [ ] Verify `DUPLO_PROD_TOKEN` secret exists in repository settings
- [ ] Test dev workflow with sample PR
- [ ] Test publish workflow with sample commit to main
- [ ] Update README if needed to reference automated S3 uploads

## Success Criteria

- Dev workflow runs on every PR to `dev` and commit to `dev`
- Publish workflow runs only on commits to `main` (after merge)
- Versioned YAML files appear in S3 with correct naming pattern
- Canonical `aws.yaml` is updated in S3 after main branch commits
- CloudFormation quick-create URL works with published `aws.yaml`

## Future Enhancements (Out of Scope)

- Add workflow to clean up old versioned files from S3
- Add Slack/email notifications on publish
- Add CloudFormation template validation before upload
- Support for multiple S3 buckets/environments
