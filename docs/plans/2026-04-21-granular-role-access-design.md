# Granular IAM Role Access Design

**Date:** 2026-04-21  
**Status:** Approved

## Overview

Update the CloudFormation template to support 4 independent IAM roles with granular access control. Each role can be individually enabled or disabled, allowing flexible combinations (e.g., AWS read-only + EKS admin). The template must support multiple deployments in the same AWS account and region by using the EKS cluster name as a unique identifier.

## Requirements

1. **4 Independent Roles:**
   - AWS Admin Access (AdministratorAccess policy, no EKS access)
   - AWS Read-Only Access (ReadOnlyAccess policy, no EKS access)
   - EKS Admin Access (AmazonEKSClusterAdminPolicy, no AWS IAM policy)
   - EKS Read-Only Access (AmazonEKSViewPolicy, no AWS IAM policy)

2. **Parameter-Driven Creation:**
   - Each role has a boolean enable/disable parameter
   - Admin roles default to `false`
   - Read-only roles default to `true`

3. **Multi-Stack Support:**
   - Use EKS cluster name in role names for uniqueness
   - Same account/region can have multiple stacks for different clusters

4. **Flexible Trust Policy:**
   - Support cross-account access (from helpdesk account)
   - Support same-account access
   - Trust policy allows principals from both accounts

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `HelpdeskAccountId` | String (12 digits) | - | Helpdesk AWS Account ID (provided by helpdesk team) |
| `EKSClusterName` | String (1-100 chars) | - | EKS cluster name - used for role naming uniqueness |
| `EnableAWSAdmin` | String (true/false) | `false` | Create AWS Admin role with AdministratorAccess |
| `EnableAWSReadOnly` | String (true/false) | `true` | Create AWS Read-Only role with ReadOnlyAccess |
| `EnableEKSAdmin` | String (true/false) | `false` | Create EKS Admin role with cluster admin policy |
| `EnableEKSReadOnly` | String (true/false) | `true` | Create EKS Read-Only role with view policy |

## Resources

### IAM Roles (Conditional)

Each role is created only when its corresponding `Enable*` parameter is `true`. All roles share the same trust policy structure but differ in their names, managed policies, and EKS access entries.

**Common Trust Policy (all roles):**
```yaml
AssumeRolePolicyDocument:
  Version: "2012-10-17"
  Statement:
    - Effect: Allow
      Principal:
        AWS:
          - !Sub "arn:aws:iam::${HelpdeskAccountId}:root"
          - !Sub "arn:aws:iam::${AWS::AccountId}:root"
      Action: "sts:AssumeRole"
```

### Role Configurations

| Role | Name Pattern | IAM Policy | EKS Access Policy | Condition |
|------|-------------|------------|-------------------|-----------|
| AWS Admin | `DuploCloud-AWS-Admin-<cluster>` | AdministratorAccess | None | `EnableAWSAdmin=true` |
| AWS Read-Only | `DuploCloud-AWS-ReadOnly-<cluster>` | ReadOnlyAccess | None | `EnableAWSReadOnly=true` |
| EKS Admin | `DuploCloud-EKS-Admin-<cluster>` | None | AmazonEKSClusterAdminPolicy | `EnableEKSAdmin=true` |
| EKS Read-Only | `DuploCloud-EKS-ReadOnly-<cluster>` | None | AmazonEKSViewPolicy | `EnableEKSReadOnly=true` |

### EKS Access Entries

EKS roles (admin and read-only) will have corresponding `AWS::EKS::AccessEntry` resources that bind the role ARN to the EKS cluster with the appropriate access policy. These are also conditional based on their enable parameters.

## Outputs

All outputs are conditional - they only appear when their corresponding role is created:

| Output Name | Description | Condition |
|-------------|-------------|-----------|
| `AWSAdminRoleArn` | AWS Admin Role ARN - provide to helpdesk | `EnableAWSAdmin=true` |
| `AWSReadOnlyRoleArn` | AWS Read-Only Role ARN - provide to helpdesk | `EnableAWSReadOnly=true` |
| `EKSAdminRoleArn` | EKS Admin Role ARN - provide to helpdesk | `EnableEKSAdmin=true` |
| `EKSReadOnlyRoleArn` | EKS Read-Only Role ARN - provide to helpdesk | `EnableEKSReadOnly=true` |
| `ClusterName` | EKS Cluster Name | Always present |

## Design Decisions

### 4 Separate Roles vs Combined Roles

**Decision:** Use 4 separate roles with individual enable/disable parameters.

**Rationale:** Maximum flexibility for customers. They can enable any combination:
- AWS read-only + EKS admin (e.g., view AWS resources, manage K8s workloads)
- Only EKS read-only (e.g., troubleshoot pods without AWS access)
- All 4 roles (full flexibility for helpdesk team)

### Trust Policy: Both Accounts

**Decision:** Trust policy always allows both helpdesk account AND current account.

**Rationale:** Supports both cross-account and same-account scenarios without requiring users to configure which model they're using. No security downside since the current account already has full control over its own resources.

### Role Naming: EKS Cluster Name

**Decision:** Include EKS cluster name in role names (not stack name).

**Rationale:** 
- Descriptive: Role name indicates which cluster it accesses
- Multi-stack support: Different clusters = different role names = no conflicts
- Consistent with existing template pattern

### Defaults: Read-Only On, Admin Off

**Decision:** Read-only roles default to `true`, admin roles default to `false`.

**Rationale:**
- Security best practice: least privilege by default
- Common use case: customers comfortable with read-only access first
- Easy to enable admin access if needed

## Multi-Stack Deployment Example

Customer with 2 EKS clusters:

**Stack 1:** Cluster `prod-us-east-1`
- Parameters: `EnableEKSReadOnly=true`, `EnableEKSAdmin=false`
- Creates: `DuploCloud-EKS-ReadOnly-prod-us-east-1`

**Stack 2:** Cluster `prod-us-west-2`
- Parameters: `EnableEKSReadOnly=true`, `EnableEKSAdmin=true`
- Creates: 
  - `DuploCloud-EKS-ReadOnly-prod-us-west-2`
  - `DuploCloud-EKS-Admin-prod-us-west-2`

Both stacks coexist in the same account/region with no naming conflicts.

## Migration from Current Template

The current template has:
- Parameter: `DuploCloudAccountId` → Renamed to `HelpdeskAccountId`
- Parameter: `AdminAccess` (true/false) → Replaced with 4 granular parameters
- 2 roles (read-only always, admin conditional) → 4 independent roles

Existing users will need to update their parameter values when migrating to the new template.
