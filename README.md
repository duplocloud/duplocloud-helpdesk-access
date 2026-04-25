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


### Parameters

| Parameter | Description | Default |
| --- | --- | --- |
| `HelpdeskAccountId` | Optional: Helpdesk AWS Account ID for cross-account access. Leave empty for same-account only. | Empty |
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
