# Check Flux Reconciliation

A GitHub composite action that verifies Flux CD has successfully reconciled your Kubernetes deployment.

## What it does

This action waits for Flux to reconcile your deployment and verifies that:
1. The Kustomization is in a "Ready" state
2. Flux is using the correct Git SHA from your flux-main branch

If Flux reconciliation fails or times out (default 5 minutes, configurable via `timeout`), the action will fail and provide detailed diagnostic information including the specific failure reason. A job summary is written with reconciliation status, expected and actual SHAs, and duration.

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
| `timeout` | Seconds to keep polling for the deploy SHA to reconcile before giving up. Raise for services whose rollout (migrations + serial pod rollout) routinely outlasts the Kustomization's own healthCheck timeout | No | `300` |

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
4. Waits 60 seconds before polling to help prevent false negatives
5. Polls every 10 seconds (up to `timeout`, default 5 minutes) to check if:
   - The Kustomization status is "Ready"
   - Flux is using the same Git SHA as the flux-main branch
6. Handles intermediate states gracefully — transient reasons (`DependencyNotReady`, `Progressing`, `ReconciliationSucceeded`, and `HealthCheckFailed`) keep waiting instead of failing immediately. `HealthCheckFailed` is transient during a normal deploy: Flux's Kustomization gives up its own healthCheck while the HelmRelease is still rolling, then recovers on its next reconcile. A genuinely stuck deploy stays failed and is caught when `timeout` expires. Terminal reasons (e.g. apply/build errors) still fail fast.
7. On failure, reports the specific failure reason and full Kustomization details
8. Writes a GitHub Actions job summary with reconciliation status, SHAs, and duration

## Versioning

This action uses rolling major version tags. You can pin to:

- A specific version: `@v3.1.0` (exact, never changes)
- A major version: `@v3` (recommended, gets bug fixes and new features)

When a new semantic version tag (e.g., `v3.2.0`) is pushed, a GitHub Actions workflow automatically updates the corresponding major version tag (`v3`) to point to the new release.