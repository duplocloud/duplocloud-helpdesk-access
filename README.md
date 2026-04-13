# DuploCloud Access

This repository provides cloud-provider-specific templates that grant DuploCloud access to your cloud account and managed Kubernetes cluster for analysis.

## Overview

Each template creates cross-account credentials and Kubernetes access bindings needed for DuploCloud to connect during a pre-sales engagement. Admin access is granted by default and can be scoped down to read-only if preferred. Deployment takes 2-3 minutes.

## Supported Platforms

| Cloud | Kubernetes | Status |
| --- | --- | --- |
| AWS | EKS | Available |
| Azure | AKS | _Coming soon_ |
| GCP | GKE | _Coming soon_ |

---

## AWS + EKS

### What Gets Created

A read-only role is always created. An admin role is created by default and can be disabled via the `AdminAccess` parameter.

| Role | IAM Policy | EKS Policy | When Created |
| --- | --- | --- | --- |
| `DuploCloud-ReadOnly-<cluster>` | `ReadOnlyAccess` | `AmazonEKSViewPolicy` | Always |
| `DuploCloud-Admin-<cluster>` | `AdministratorAccess` | `AmazonEKSClusterAdminPolicy` | `AdminAccess=true` (default) |

### Architecture

```text
Your AWS Account
  └── CloudFormation Stack
        ├── ReadOnlyRole          (IAM Role — ReadOnlyAccess)
        ├── ReadOnlyEKSAccessEntry (AmazonEKSViewPolicy)
        ├── AdminRole             (IAM Role — AdministratorAccess) [if AdminAccess=true]
        └── AdminEKSAccessEntry   (AmazonEKSClusterAdminPolicy)   [if AdminAccess=true]
```

### Tier 2 Access (Optional)

If your EKS public endpoint is CIDR-restricted, DuploCloud will provide a NAT IP to add to your cluster's public access CIDRs.

### Decision Matrix

| Scenario | Action |
| --- | --- |
| EKS public endpoint is open | Deploy the template; no extra steps |
| EKS public endpoint is CIDR-restricted | Deploy the template, then add the IP provided by DuploCloud |
| EKS endpoint is fully private | Contact DuploCloud for alternative access options |

### Parameters

| Parameter | Description | Default |
| --- | --- | --- |
| `DuploCloudAccountId` | DuploCloud AWS Account ID (provided by DuploCloud) | — |
| `EKSClusterName` | Name of the EKS cluster to analyze | — |
| `AdminAccess` | Grant admin access at AWS and EKS level (`true`/`false`) | `true` |

### Outputs

| Output | Description | When Present |
| --- | --- | --- |
| `ReadOnlyRoleArn` | Read-only Role ARN — provide to DuploCloud | Always |
| `AdminRoleArn` | Admin Role ARN — provide to DuploCloud | `AdminAccess=true` only |
| `ClusterName` | EKS Cluster Name | Always |

### CloudFormation Template

See [`aws.yaml`](aws.yaml).

### Deploy via Quick-Create URL

> **TODO:** Create a CI/CD pipeline in this repo to automatically publish `aws.yaml` to public S3 bucket

Use this link to open the CloudFormation console with the template pre-loaded. Replace `ACCOUNT_ID` with the DuploCloud account ID and fill in `EKSClusterName` in the console.

[Launch in CloudFormation](https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateURL=https%3A%2F%2Fs3.amazonaws.com%2Fduplocloud-public-cfn%2Fduplocloud-eks-readonly.yaml&stackName=DuploCloud-EKS-Access&param_DuploCloudAccountId=ACCOUNT_ID&param_EKSClusterName=)

> The template is hosted at `s3://duplocloud-public-cfn/duplocloud-eks-readonly.yaml`. After any template change, upload with:

```bash
aws s3 cp duplocloud-eks-readonly.yaml s3://duplocloud-public-cfn/duplocloud-eks-readonly.yaml --acl public-read
```

### Implementation Checklist

- [ ] Note your EKS cluster name
- [ ] Deploy the CloudFormation stack using the quick-create URL or directly in your AWS account
- [ ] Provide the `ReadOnlyRoleArn` (and `AdminRoleArn` if applicable) output values to DuploCloud
- [ ] (If CIDR-restricted) Add the NAT IP provided by DuploCloud to your EKS public access CIDRs

---

## Azure + AKS

> **Coming soon.** This section will cover granting DuploCloud access to an Azure subscription and AKS cluster.

_Planned approach:_

- Azure AD application registration with Owner/Contributor role on the subscription
- AKS `cluster-admin` ClusterRoleBinding for the service principal

---

## GCP + GKE

> **Coming soon.** This section will cover granting DuploCloud access to a GCP project and GKE cluster.

_Planned approach:_

- GCP service account with `roles/owner` on the project
- GKE `cluster-admin` ClusterRoleBinding for the service account
