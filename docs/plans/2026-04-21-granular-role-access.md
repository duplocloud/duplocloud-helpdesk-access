# Granular IAM Role Access Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Refactor CloudFormation template to support 4 independent IAM roles (AWS admin/read-only, EKS admin/read-only) with individual enable/disable parameters.

**Architecture:** Replace the current 2-role structure (1 read-only always + 1 admin conditional) with 4 independent roles, each controlled by its own boolean parameter. Update parameters, conditions, resources, and outputs to support granular access control and multi-stack deployments.

**Tech Stack:** AWS CloudFormation (YAML), IAM roles, EKS access entries

---

## Task 1: Update Parameters Section

**Files:**
- Modify: `aws.yaml:6-28`

**Step 1: Rename DuploCloudAccountId parameter**

Replace lines 7-11 with:

```yaml
  HelpdeskAccountId:
    Type: String
    Description: "Helpdesk AWS Account ID (provided by helpdesk team)"
    AllowedPattern: "[0-9]{12}"
    ConstraintDescription: "Must be a 12-digit AWS account ID"
```

**Step 2: Replace AdminAccess parameter with 4 enable parameters**

Replace lines 19-28 with:

```yaml
  EnableAWSAdmin:
    Type: String
    Default: "false"
    AllowedValues:
      - "true"
      - "false"
    Description: "Create AWS Admin role with AdministratorAccess policy"

  EnableAWSReadOnly:
    Type: String
    Default: "true"
    AllowedValues:
      - "true"
      - "false"
    Description: "Create AWS Read-Only role with ReadOnlyAccess policy"

  EnableEKSAdmin:
    Type: String
    Default: "false"
    AllowedValues:
      - "true"
      - "false"
    Description: "Create EKS Admin role with AmazonEKSClusterAdminPolicy"

  EnableEKSReadOnly:
    Type: String
    Default: "true"
    AllowedValues:
      - "true"
      - "false"
    Description: "Create EKS Read-Only role with AmazonEKSViewPolicy"
```

**Step 3: Verify parameter section**

Expected result: 6 parameters total (HelpdeskAccountId, EKSClusterName, 4 Enable* parameters)

**Step 4: Commit parameter updates**

```bash
git add aws.yaml
git commit -m "refactor: update parameters for granular role access

Replace DuploCloudAccountId with HelpdeskAccountId and AdminAccess with
4 independent enable parameters (EnableAWSAdmin, EnableAWSReadOnly,
EnableEKSAdmin, EnableEKSReadOnly).

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 2: Update Conditions Section

**Files:**
- Modify: `aws.yaml:30-31`

**Step 1: Replace single condition with 4 conditions**

Replace lines 30-31 with:

```yaml
Conditions:
  CreateAWSAdminRole: !Equals [!Ref EnableAWSAdmin, "true"]
  CreateAWSReadOnlyRole: !Equals [!Ref EnableAWSReadOnly, "true"]
  CreateEKSAdminRole: !Equals [!Ref EnableEKSAdmin, "true"]
  CreateEKSReadOnlyRole: !Equals [!Ref EnableEKSReadOnly, "true"]
```

**Step 2: Verify conditions section**

Expected result: 4 conditions, each mapping to its enable parameter

**Step 3: Commit condition updates**

```bash
git add aws.yaml
git commit -m "refactor: add conditions for 4 independent roles

Add CreateAWSAdminRole, CreateAWSReadOnlyRole, CreateEKSAdminRole, and
CreateEKSReadOnlyRole conditions.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 3: Create AWS Admin Role

**Files:**
- Modify: `aws.yaml:33-92` (Resources section)

**Step 1: Add AWS Admin role resource**

After the `Resources:` line (line 33), add:

```yaml
  # -------------------------------------------------------
  # AWS Admin IAM Role (no EKS access)
  # -------------------------------------------------------
  AWSAdminRole:
    Type: AWS::IAM::Role
    Condition: CreateAWSAdminRole
    Properties:
      RoleName: !Sub "DuploCloud-AWS-Admin-${EKSClusterName}"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS:
                - !Sub "arn:aws:iam::${HelpdeskAccountId}:root"
                - !Sub "arn:aws:iam::${AWS::AccountId}:root"
            Action: "sts:AssumeRole"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AdministratorAccess
```

**Step 2: Verify AWS Admin role**

Check that:
- Condition references `CreateAWSAdminRole`
- Trust policy includes both `HelpdeskAccountId` and `AWS::AccountId`
- ManagedPolicyArns includes AdministratorAccess
- No EKS access entry

**Step 3: Commit AWS Admin role**

```bash
git add aws.yaml
git commit -m "feat: add AWS Admin role with dual-account trust policy

Create AWS Admin role (AdministratorAccess) conditional on
EnableAWSAdmin parameter. Trust policy allows both helpdesk account and
current account principals.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 4: Create AWS Read-Only Role

**Files:**
- Modify: `aws.yaml` (Resources section)

**Step 1: Add AWS Read-Only role resource**

After the AWS Admin role, add:

```yaml
  # -------------------------------------------------------
  # AWS Read-Only IAM Role (no EKS access)
  # -------------------------------------------------------
  AWSReadOnlyRole:
    Type: AWS::IAM::Role
    Condition: CreateAWSReadOnlyRole
    Properties:
      RoleName: !Sub "DuploCloud-AWS-ReadOnly-${EKSClusterName}"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS:
                - !Sub "arn:aws:iam::${HelpdeskAccountId}:root"
                - !Sub "arn:aws:iam::${AWS::AccountId}:root"
            Action: "sts:AssumeRole"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/ReadOnlyAccess
```

**Step 2: Verify AWS Read-Only role**

Check that:
- Condition references `CreateAWSReadOnlyRole`
- Trust policy includes both accounts
- ManagedPolicyArns includes ReadOnlyAccess
- No EKS access entry

**Step 3: Commit AWS Read-Only role**

```bash
git add aws.yaml
git commit -m "feat: add AWS Read-Only role with dual-account trust policy

Create AWS Read-Only role (ReadOnlyAccess) conditional on
EnableAWSReadOnly parameter.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 5: Create EKS Admin Role and Access Entry

**Files:**
- Modify: `aws.yaml` (Resources section)

**Step 1: Add EKS Admin role resource**

After the AWS Read-Only role, add:

```yaml
  # -------------------------------------------------------
  # EKS Admin IAM Role (no AWS IAM policies)
  # -------------------------------------------------------
  EKSAdminRole:
    Type: AWS::IAM::Role
    Condition: CreateEKSAdminRole
    Properties:
      RoleName: !Sub "DuploCloud-EKS-Admin-${EKSClusterName}"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS:
                - !Sub "arn:aws:iam::${HelpdeskAccountId}:root"
                - !Sub "arn:aws:iam::${AWS::AccountId}:root"
            Action: "sts:AssumeRole"

  EKSAdminAccessEntry:
    Type: AWS::EKS::AccessEntry
    Condition: CreateEKSAdminRole
    Properties:
      ClusterName: !Ref EKSClusterName
      PrincipalArn: !GetAtt EKSAdminRole.Arn
      Type: STANDARD
      AccessPolicies:
        - PolicyArn: "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
          AccessScope:
            Type: cluster
```

**Step 2: Verify EKS Admin role**

Check that:
- Role has no ManagedPolicyArns (EKS access only)
- AccessEntry references EKSAdminRole.Arn
- Both use CreateEKSAdminRole condition
- AccessPolicies uses AmazonEKSClusterAdminPolicy

**Step 3: Commit EKS Admin role**

```bash
git add aws.yaml
git commit -m "feat: add EKS Admin role with cluster admin access

Create EKS Admin role with AmazonEKSClusterAdminPolicy access entry,
conditional on EnableEKSAdmin parameter. Role has no AWS IAM policies.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 6: Create EKS Read-Only Role and Access Entry

**Files:**
- Modify: `aws.yaml` (Resources section)

**Step 1: Add EKS Read-Only role resource**

After the EKS Admin resources, add:

```yaml
  # -------------------------------------------------------
  # EKS Read-Only IAM Role (no AWS IAM policies)
  # -------------------------------------------------------
  EKSReadOnlyRole:
    Type: AWS::IAM::Role
    Condition: CreateEKSReadOnlyRole
    Properties:
      RoleName: !Sub "DuploCloud-EKS-ReadOnly-${EKSClusterName}"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS:
                - !Sub "arn:aws:iam::${HelpdeskAccountId}:root"
                - !Sub "arn:aws:iam::${AWS::AccountId}:root"
            Action: "sts:AssumeRole"

  EKSReadOnlyAccessEntry:
    Type: AWS::EKS::AccessEntry
    Condition: CreateEKSReadOnlyRole
    Properties:
      ClusterName: !Ref EKSClusterName
      PrincipalArn: !GetAtt EKSReadOnlyRole.Arn
      Type: STANDARD
      AccessPolicies:
        - PolicyArn: "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"
          AccessScope:
            Type: cluster
```

**Step 2: Verify EKS Read-Only role**

Check that:
- Role has no ManagedPolicyArns (EKS access only)
- AccessEntry references EKSReadOnlyRole.Arn
- Both use CreateEKSReadOnlyRole condition
- AccessPolicies uses AmazonEKSViewPolicy

**Step 3: Commit EKS Read-Only role**

```bash
git add aws.yaml
git commit -m "feat: add EKS Read-Only role with view access

Create EKS Read-Only role with AmazonEKSViewPolicy access entry,
conditional on EnableEKSReadOnly parameter. Role has no AWS IAM policies.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 7: Remove Old Roles

**Files:**
- Modify: `aws.yaml` (Resources section)

**Step 1: Delete ReadOnlyRole resource**

Remove the old `ReadOnlyRole` resource (originally lines 38-50).

**Step 2: Delete ReadOnlyEKSAccessEntry resource**

Remove the old `ReadOnlyEKSAccessEntry` resource (originally lines 52-61).

**Step 3: Delete AdminRole resource**

Remove the old `AdminRole` resource (originally lines 66-79).

**Step 4: Delete AdminEKSAccessEntry resource**

Remove the old `AdminEKSAccessEntry` resource (originally lines 81-91).

**Step 5: Verify Resources section**

Expected result: Only 6 new resources remain:
- AWSAdminRole
- AWSReadOnlyRole
- EKSAdminRole
- EKSAdminAccessEntry
- EKSReadOnlyRole
- EKSReadOnlyAccessEntry

**Step 6: Commit removal of old roles**

```bash
git add aws.yaml
git commit -m "refactor: remove old 2-role structure

Delete ReadOnlyRole, ReadOnlyEKSAccessEntry, AdminRole, and
AdminEKSAccessEntry in favor of new 4-role structure.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 8: Update Outputs Section

**Files:**
- Modify: `aws.yaml:93-105` (Outputs section)

**Step 1: Replace outputs with 4 conditional role ARNs**

Replace lines 94-101 with:

```yaml
  AWSAdminRoleArn:
    Description: "AWS Admin Role ARN - provide to helpdesk"
    Condition: CreateAWSAdminRole
    Value: !GetAtt AWSAdminRole.Arn

  AWSReadOnlyRoleArn:
    Description: "AWS Read-Only Role ARN - provide to helpdesk"
    Condition: CreateAWSReadOnlyRole
    Value: !GetAtt AWSReadOnlyRole.Arn

  EKSAdminRoleArn:
    Description: "EKS Admin Role ARN - provide to helpdesk"
    Condition: CreateEKSAdminRole
    Value: !GetAtt EKSAdminRole.Arn

  EKSReadOnlyRoleArn:
    Description: "EKS Read-Only Role ARN - provide to helpdesk"
    Condition: CreateEKSReadOnlyRole
    Value: !GetAtt EKSReadOnlyRole.Arn
```

**Step 2: Verify outputs section**

Expected result: 5 outputs total (4 conditional role ARNs + ClusterName)

**Step 3: Commit output updates**

```bash
git add aws.yaml
git commit -m "refactor: update outputs for 4 role ARNs

Replace ReadOnlyRoleArn and AdminRoleArn with AWSAdminRoleArn,
AWSReadOnlyRoleArn, EKSAdminRoleArn, and EKSReadOnlyRoleArn.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 9: Update Template Description

**Files:**
- Modify: `aws.yaml:2-4`

**Step 1: Update description to reflect new functionality**

Replace lines 2-4 with:

```yaml
Description: >
  Grants helpdesk granular access to your AWS account and EKS cluster.
  Create 4 independent roles (AWS admin/read-only, EKS admin/read-only)
  with individual enable/disable controls. Takes 2-3 minutes to deploy.
```

**Step 2: Commit description update**

```bash
git add aws.yaml
git commit -m "docs: update template description for granular roles

Update CloudFormation description to reflect 4 independent roles with
enable/disable controls.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 10: Update README Documentation

**Files:**
- Modify: `README.md:21-80`

**Step 1: Update "What Gets Created" section**

Replace lines 21-28 with:

```markdown
### What Gets Created

Four independent IAM roles, each enabled/disabled via parameters:

| Role | IAM Policy | EKS Policy | Default |
| --- | --- | --- | --- |
| `DuploCloud-AWS-Admin-<cluster>` | `AdministratorAccess` | None | Disabled |
| `DuploCloud-AWS-ReadOnly-<cluster>` | `ReadOnlyAccess` | None | Enabled |
| `DuploCloud-EKS-Admin-<cluster>` | None | `AmazonEKSClusterAdminPolicy` | Disabled |
| `DuploCloud-EKS-ReadOnly-<cluster>` | None | `AmazonEKSViewPolicy` | Enabled |
```

**Step 2: Update Architecture section**

Replace lines 30-39 with:

```markdown
### Architecture

```text
Your AWS Account
  └── CloudFormation Stack
        ├── AWSAdminRole           (IAM — AdministratorAccess) [if EnableAWSAdmin=true]
        ├── AWSReadOnlyRole        (IAM — ReadOnlyAccess) [if EnableAWSReadOnly=true]
        ├── EKSAdminRole           (IAM — no policies) [if EnableEKSAdmin=true]
        ├── EKSAdminAccessEntry    (AmazonEKSClusterAdminPolicy) [if EnableEKSAdmin=true]
        ├── EKSReadOnlyRole        (IAM — no policies) [if EnableEKSReadOnly=true]
        └── EKSReadOnlyAccessEntry (AmazonEKSViewPolicy) [if EnableEKSReadOnly=true]
```
```

**Step 3: Update Parameters section**

Replace lines 53-59 with:

```markdown
### Parameters

| Parameter | Description | Default |
| --- | --- | --- |
| `HelpdeskAccountId` | Helpdesk AWS Account ID (provided by helpdesk team) | — |
| `EKSClusterName` | Name of the EKS cluster to analyze | — |
| `EnableAWSAdmin` | Create AWS Admin role (`true`/`false`) | `false` |
| `EnableAWSReadOnly` | Create AWS Read-Only role (`true`/`false`) | `true` |
| `EnableEKSAdmin` | Create EKS Admin role (`true`/`false`) | `false` |
| `EnableEKSReadOnly` | Create EKS Read-Only role (`true`/`false`) | `true` |
```

**Step 4: Update Outputs section**

Replace lines 61-66 with:

```markdown
### Outputs

| Output | Description | When Present |
| --- | --- | --- |
| `AWSAdminRoleArn` | AWS Admin Role ARN — provide to helpdesk | `EnableAWSAdmin=true` |
| `AWSReadOnlyRoleArn` | AWS Read-Only Role ARN — provide to helpdesk | `EnableAWSReadOnly=true` |
| `EKSAdminRoleArn` | EKS Admin Role ARN — provide to helpdesk | `EnableEKSAdmin=true` |
| `EKSReadOnlyRoleArn` | EKS Read-Only Role ARN — provide to helpdesk | `EnableEKSReadOnly=true` |
| `ClusterName` | EKS Cluster Name | Always |
```

**Step 5: Update Deploy URL parameters**

Replace line 79 with:

```markdown
[Launch in CloudFormation](https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateURL=https%3A%2F%2Fs3.amazonaws.com%2Fduplocloud-public-cfn%2Fduplocloud-eks-access.yaml&stackName=DuploCloud-EKS-Access&param_HelpdeskAccountId=ACCOUNT_ID&param_EKSClusterName=)
```

**Step 6: Update Implementation Checklist**

Replace lines 87-92 with:

```markdown
### Implementation Checklist

- [ ] Note your EKS cluster name
- [ ] Deploy the CloudFormation stack using the quick-create URL or directly in your AWS account
- [ ] Choose which roles to enable (defaults: read-only enabled, admin disabled)
- [ ] Provide the enabled role ARN output values to the helpdesk team
- [ ] (If CIDR-restricted) Add the NAT IP provided by helpdesk to your EKS public access CIDRs
```

**Step 7: Commit README updates**

```bash
git add README.md
git commit -m "docs: update README for granular role access

Update documentation to reflect 4 independent roles with enable/disable
parameters. Update architecture diagram, parameters table, and outputs.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Task 11: Manual Validation

**Files:**
- Read: `aws.yaml` (entire file)

**Step 1: Validate CloudFormation syntax**

Run: `aws cloudformation validate-template --template-body file://aws.yaml` (if AWS CLI available)

Or use online validator: https://yaml.org/ to check YAML syntax

Expected: No syntax errors

**Step 2: Review checklist**

Verify:
- [ ] 6 parameters (HelpdeskAccountId, EKSClusterName, 4 Enable*)
- [ ] 4 conditions (one per role)
- [ ] 6 resources (4 roles + 2 EKS access entries)
- [ ] 5 outputs (4 conditional role ARNs + ClusterName)
- [ ] All references to DuploCloudAccountId replaced with HelpdeskAccountId
- [ ] All roles have dual-account trust policy
- [ ] Admin defaults to false, read-only defaults to true
- [ ] EKS roles have no AWS IAM policies

**Step 3: Document completion**

All tasks complete. Template ready for deployment testing.

---

## Post-Implementation Notes

**Testing Strategy:**
- Deploy stack with default parameters (both read-only roles enabled)
- Verify 2 roles created: AWSReadOnlyRole, EKSReadOnlyRole
- Verify 2 outputs present: AWSReadOnlyRoleArn, EKSReadOnlyRoleArn
- Update stack with all roles enabled
- Verify 4 roles created
- Verify 4 outputs present

**Multi-Stack Testing:**
- Deploy stack 1 with cluster name "cluster-a"
- Deploy stack 2 with cluster name "cluster-b" 
- Verify no naming conflicts
- Verify both stacks show correct outputs

**Rollback Plan:**
- Previous template is in git history (commit: 16a7c28)
- Run: `git checkout 16a7c28 aws.yaml` to restore old template if needed
