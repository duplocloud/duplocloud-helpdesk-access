# DuploCloud Access

This repository provides cloud-provider-specific templates that grant DuploCloud access to your cloud account and managed Kubernetes cluster for analysis.

## Overview

Each template creates cross-account credentials and Kubernetes access bindings needed for DuploCloud to connect during a pre-sales engagement. Four independent roles are available, with read-only access enabled by default and admin access disabled by default. Deployment takes 2-3 minutes.

## Supported Platforms

| Cloud | Kubernetes | Status |
| --- | --- | --- |
| AWS | EKS | Available |
| Azure | AKS | _Coming soon_ |
| GCP | GKE | _Coming soon_ |

---

## AWS + EKS

### What Gets Created

Four independent IAM roles, each enabled/disabled via parameters:

| Role | IAM Policy | EKS Policy | Default |
| --- | --- | --- | --- |
| `DuploCloud-AWS-Admin-<cluster>` | `AdministratorAccess` | None | Disabled |
| `DuploCloud-AWS-ReadOnly-<cluster>` | `ReadOnlyAccess` | None | Enabled |
| `DuploCloud-EKS-Admin-<cluster>` | None | `AmazonEKSClusterAdminPolicy` | Disabled |
| `DuploCloud-EKS-ReadOnly-<cluster>` | None | `AmazonEKSViewPolicy` | Enabled |

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
| `HelpdeskAccountId` | Helpdesk AWS Account ID (provided by helpdesk team) | — |
| `EKSClusterName` | Name of the EKS cluster to analyze | — |
| `EnableAWSAdmin` | Create AWS Admin role (`true`/`false`) | `false` |
| `EnableAWSReadOnly` | Create AWS Read-Only role (`true`/`false`) | `true` |
| `EnableEKSAdmin` | Create EKS Admin role (`true`/`false`) | `false` |
| `EnableEKSReadOnly` | Create EKS Read-Only role (`true`/`false`) | `true` |

### Outputs

| Output | Description | When Present |
| --- | --- | --- |
| `AWSAdminRoleArn` | AWS Admin Role ARN — provide to helpdesk | `EnableAWSAdmin=true` |
| `AWSReadOnlyRoleArn` | AWS Read-Only Role ARN — provide to helpdesk | `EnableAWSReadOnly=true` |
| `EKSAdminRoleArn` | EKS Admin Role ARN — provide to helpdesk | `EnableEKSAdmin=true` |
| `EKSReadOnlyRoleArn` | EKS Read-Only Role ARN — provide to helpdesk | `EnableEKSReadOnly=true` |
| `ClusterName` | EKS Cluster Name | Always |

### CloudFormation Template

See [`aws.yaml`](aws.yaml).

### Deploy via Quick-Create URL

> **TODO:** Create a CI/CD pipeline in this repo to automatically publish `aws.yaml` to public S3 bucket

Use this link to open the CloudFormation console with the template pre-loaded. Replace `ACCOUNT_ID` with the DuploCloud account ID and fill in `EKSClusterName` in the console.

[Launch in CloudFormation](https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateURL=https%3A%2F%2Fs3.amazonaws.com%2Fduplocloud-public-cfn%2Fduplocloud-eks-access.yaml&stackName=DuploCloud-EKS-Access&param_HelpdeskAccountId=ACCOUNT_ID&param_EKSClusterName=)

> The template is hosted at `s3://duplocloud-public-cfn/duplocloud-eks-access.yaml`. After any template change, upload with:

```bash
aws s3 cp duplocloud-eks-access.yaml s3://duplocloud-public-cfn/duplocloud-eks-access.yaml --acl public-read
```

### Implementation Checklist

- [ ] Note your EKS cluster name
- [ ] Deploy the CloudFormation stack using the quick-create URL or directly in your AWS account
- [ ] Choose which roles to enable (defaults: read-only enabled, admin disabled)
- [ ] Provide the enabled role ARN output values to the helpdesk team
- [ ] (If CIDR-restricted) Add the NAT IP provided by helpdesk to your EKS public access CIDRs

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
