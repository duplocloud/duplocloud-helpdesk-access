# GitHub Actions S3 Upload Pipelines Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create two GitHub Actions workflows to automatically upload aws.yaml to S3 with versioned (dev) and canonical (publish) filenames using DuploCloud JIT credentials.

**Architecture:** Two independent workflow files with common DuploCloud authentication pattern. Dev workflow generates versioned filenames from branch name and commit SHA. Publish workflow uses fixed filename. Both use duploctl for AWS credential management.

**Tech Stack:** GitHub Actions, DuploCloud Actions (setup@v1), AWS CLI, Bash

---

## Task 1: Create GitHub Workflows Directory

**Files:**
- Create: `.github/workflows/` (directory)

**Step 1: Create workflows directory**

```bash
mkdir -p .github/workflows
```

**Step 2: Verify directory exists**

Run: `ls -la .github/`
Expected: `workflows/` directory visible

**Step 3: Commit**

```bash
git add .github/workflows/.gitkeep
git commit -m "chore: create GitHub workflows directory"
```

---

## Task 2: Create Dev Upload Workflow

**Files:**
- Create: `.github/workflows/dev-upload.yml`

**Step 1: Write dev-upload.yml workflow**

```yaml
name: "Dev: Upload to S3"

on:
  push:
    branches: [dev]
  pull_request:
    branches: [dev]
    types: [opened, synchronize, reopened]

jobs:
  upload:
    name: Upload versioned aws.yaml to S3
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup DuploCtl
        uses: duplocloud/actions/setup@v1
        with:
          duplo_host: https://prod.duplocloud.net
          duplo_token: ${{ secrets.DUPLO_PROD_TOKEN }}
        env:
          DUPLO_TENANT_ID: 5cd3b897-e4d2-4049-85b8-8dd9ab5232de
      
      - name: Generate versioned filename
        id: filename
        run: |
          BRANCH_NAME="${GITHUB_HEAD_REF:-${GITHUB_REF#refs/heads/}}"
          SHORT_SHA="${GITHUB_SHA:0:7}"
          FILENAME="aws-${BRANCH_NAME}-${SHORT_SHA}.yaml"
          echo "filename=${FILENAME}" >> $GITHUB_OUTPUT
          echo "Generated filename: ${FILENAME}"
      
      - name: Upload to S3
        run: |
          aws s3 cp aws.yaml s3://duploservices-ai-access-227120241369/${{ steps.filename.outputs.filename }}
          echo "Uploaded to: s3://duploservices-ai-access-227120241369/${{ steps.filename.outputs.filename }}"
```

**Step 2: Verify file syntax**

Run: `cat .github/workflows/dev-upload.yml`
Expected: Valid YAML content displayed

**Step 3: Commit**

```bash
git add .github/workflows/dev-upload.yml
git commit -m "feat: add dev upload workflow for versioned S3 uploads"
```

---

## Task 3: Create Publish Workflow

**Files:**
- Create: `.github/workflows/publish.yml`

**Step 1: Write publish.yml workflow**

```yaml
name: "Publish: Upload to S3"

on:
  push:
    branches: [main]

jobs:
  upload:
    name: Upload canonical aws.yaml to S3
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup DuploCtl
        uses: duplocloud/actions/setup@v1
        with:
          duplo_host: https://prod.duplocloud.net
          duplo_token: ${{ secrets.DUPLO_PROD_TOKEN }}
        env:
          DUPLO_TENANT_ID: 5cd3b897-e4d2-4049-85b8-8dd9ab5232de
      
      - name: Upload to S3
        run: |
          aws s3 cp aws.yaml s3://duploservices-ai-access-227120241369/aws.yaml
          echo "Uploaded canonical aws.yaml to S3"
```

**Step 2: Verify file syntax**

Run: `cat .github/workflows/publish.yml`
Expected: Valid YAML content displayed

**Step 3: Commit**

```bash
git add .github/workflows/publish.yml
git commit -m "feat: add publish workflow for canonical S3 uploads"
```

---

## Task 4: Create Dev Branch

**Files:**
- N/A (branch creation only)

**Step 1: Create dev branch from main**

```bash
git checkout -b dev
```

**Step 2: Verify branch created**

Run: `git branch --show-current`
Expected: `dev`

**Step 3: Push dev branch to remote**

```bash
git push -u origin dev
```

**Step 4: Return to main branch**

```bash
git checkout main
```

---

## Task 5: Update README with Automation Note

**Files:**
- Modify: `README.md:84-94`

**Step 1: Update README S3 upload section**

Find the section starting with:
```markdown
> **TODO:** Create a CI/CD pipeline in this repo to automatically publish `aws.yaml` to public S3 bucket
```

Replace with:
```markdown
> **Automated:** GitHub Actions automatically publishes `aws.yaml` to S3 on commits to `main`. Dev versions are uploaded as `aws-{branch}-{sha}.yaml` on PRs to `dev`.
```

**Step 2: Verify changes**

Run: `git diff README.md`
Expected: See the TODO replaced with automation note

**Step 3: Commit**

```bash
git add README.md
git commit -m "docs: update README to reflect automated S3 uploads"
```

---

## Task 6: Verification Checklist

**Manual Verification Steps:**

**Step 1: Verify GitHub secrets exist**

1. Go to GitHub repository settings
2. Navigate to Secrets and variables → Actions
3. Verify `DUPLO_PROD_TOKEN` secret exists
4. If missing, add the secret with value provided by user

**Step 2: Test dev workflow (requires GitHub UI)**

1. Create test branch from `dev`: `git checkout dev && git checkout -b test-workflow`
2. Make trivial change: `echo "# Test" >> README.md`
3. Commit and push: `git add README.md && git commit -m "test: trigger dev workflow" && git push -u origin test-workflow`
4. Open PR to `dev` branch on GitHub
5. Verify "Dev: Upload to S3" workflow runs in Actions tab
6. Check S3 bucket for file: `aws-test-workflow-{sha}.yaml`

**Step 3: Test publish workflow (requires GitHub UI)**

1. Merge a commit to `main` (can use the README update from Task 5)
2. Verify "Publish: Upload to S3" workflow runs in Actions tab
3. Check S3 bucket for file: `aws.yaml`
4. Verify CloudFormation quick-create URL works with new file

**Step 4: Document completion**

If all workflows run successfully and files appear in S3, mark implementation complete.

---

## Testing Notes

- **Dev workflow**: Triggers on push to `dev` and PRs targeting `dev`
- **Publish workflow**: Triggers only on push to `main` (after PR merge)
- **Filename format**: `aws-{branch}-{sha7}.yaml` for dev, `aws.yaml` for publish
- **S3 bucket**: `duploservices-ai-access-227120241369`
- **DuploCloud**: JIT credentials via `duplocloud/actions/setup@v1`

---

## Success Criteria

✅ `.github/workflows/dev-upload.yml` created and committed  
✅ `.github/workflows/publish.yml` created and committed  
✅ `dev` branch created and pushed to remote  
✅ README updated to reflect automation  
✅ Dev workflow successfully uploads versioned file to S3  
✅ Publish workflow successfully uploads canonical file to S3  
✅ CloudFormation quick-create URL works with published file

---

## Rollback Plan

If workflows fail or cause issues:

1. Delete workflow files: `git rm .github/workflows/*.yml`
2. Commit: `git commit -m "chore: remove workflows"`
3. Push: `git push origin main`
4. Workflows will stop running immediately
