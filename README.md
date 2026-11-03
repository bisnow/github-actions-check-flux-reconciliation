# Check Flux Reconciliation

A GitHub composite action that verifies Flux CD has successfully reconciled your Kubernetes deployment.

## What it does

This action waits for Flux to reconcile your deployment and verifies that:
1. The Kustomization is in a "Ready" state
2. Flux is using the correct Git SHA from your flux-main branch, or a newer flux-main revision that contains it (when another deploy lands on flux-main right after yours, Flux applies that newer commit instead)

If Flux reconciliation fails or times out (default 5 minutes, configurable via `timeout`, plus up to another 5 minutes while waiting on a dependency, configurable via `dependency-timeout`), the action will fail and provide detailed diagnostic information including the specific failure reason. A job summary is written with reconciliation status, expected and actual SHAs, and duration.

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
| `dependency-timeout` | Extra seconds, on top of `timeout`, to keep waiting while the Kustomization reports `DependencyNotReady` (a Kustomization it `dependsOn`, e.g. `kyverno-policies`, is not Ready yet). Time spent `DependencyNotReady` doesn't count against `timeout`, up to this many seconds. `0` disables it | No | `300` |

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
   - The Kustomization's last reconcile succeeded (`ReconciliationSucceeded`)
   - Flux applied the same Git SHA as the flux-main branch, or a newer one that contains it
6. Accepts a newer revision: if another deploy lands on flux-main after this checkout, Flux applies that commit and never reports this run's SHA. When the applied SHA differs and the reconcile succeeded, the action fetches more flux-main history (the checkout is shallow; once per new Flux SHA) and succeeds if this run's SHA is an ancestor of the applied one. If the applied SHA isn't in the fetched history, or doesn't contain this run's SHA, it keeps waiting as before.
7. Handles intermediate states gracefully — transient reasons (`DependencyNotReady`, `Progressing`, `ReconciliationSucceeded`, and `HealthCheckFailed`) keep waiting instead of failing immediately. `HealthCheckFailed` is transient during a normal deploy: Flux's Kustomization gives up its own healthCheck while the HelmRelease is still rolling, then recovers on its next reconcile. A genuinely stuck deploy stays failed and is caught when `timeout` expires. Terminal reasons (e.g. apply/build errors) still fail fast.
8. Waits longer on dependencies: time spent `DependencyNotReady` (e.g. `dependency 'flux-system/kyverno-policies' is not ready`) is added to the deadline, up to `dependency-timeout` (default 5 minutes). If it is still `DependencyNotReady` at the deadline, the timeout message and job summary name the dependency that held it up.
9. On failure, reports the specific failure reason and full Kustomization details
10. Writes a GitHub Actions job summary with reconciliation status, SHAs, and duration

## Versioning

This action uses rolling major version tags. You can pin to:

- A specific version: `@v3.1.0` (exact, never changes)
- A major version: `@v3` (recommended, gets bug fixes and new features)

When a new semantic version tag (e.g., `v3.2.0`) is pushed, a GitHub Actions workflow automatically updates the corresponding major version tag (`v3`) to point to the new release.