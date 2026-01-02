# Check Flux Reconciliation

A GitHub composite action that verifies Flux CD has successfully reconciled your Kubernetes deployment.

## What it does

This action waits for Flux to reconcile your deployment and verifies that:
1. The Kustomization is in a "Ready" state
2. Flux is using the correct Git SHA from your flux-main branch

If Flux reconciliation fails or times out after 5 minutes, the action will fail and provide detailed diagnostic information.

## Usage

```yaml
- name: Check Flux reconciliation
  uses: bisnow/github-actions-check-flux-reconciliation@main
  with:
    service-name: dev-biscred-api
    cluster: my-eks-cluster
    region: us-east-1  # optional, defaults to us-east-1
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `service-name` | Name of the service being deployed (e.g., `dev-biscred-api`) | Yes | - |
| `cluster` | Name of the Kubernetes cluster being deployed to | Yes | - |
| `region` | AWS region where the cluster is located | No | `us-east-1` |

## Prerequisites

- Flux CD must be installed in your cluster
- The action assumes you have a `flux-main` branch in your repository
- AWS credentials must be configured (this action uses `bisnow/github-actions-assume-role-for-environment@main`)

## Example workflow

```yaml
name: Deploy and verify

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to cluster
        # Your deployment steps here

      - name: Verify Flux reconciliation
        uses: bisnow/github-actions-check-flux-reconciliation@main
        with:
          service-name: my-service
          cluster: production-cluster
```

## How it works

1. Checks out the `flux-main` branch
2. Assumes AWS role for the Bisnow account
3. Configures kubectl for your EKS cluster
4. Waits up to 5 minutes for Flux to reconcile
5. Checks every 10 seconds to see if:
   - The Kustomization status is "Ready"
   - Flux is using the same Git SHA as the flux-main branch
6. Exits successfully when reconciliation is complete, or fails with diagnostics if there's an issue
