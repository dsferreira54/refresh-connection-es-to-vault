# CronJob — Refresh Connection — External Secrets to Hashicorp Vault

Helm chart that runs a scheduled job on OpenShift or Kubernetes to keep HashiCorp Vault reachable from [External Secrets Operator](https://external-secrets.io/) after restarts or drift: it unseals Vault pods when needed and annotates External Secrets custom resources so their connection to Vault is refreshed.

## What it does

On each run the job:

1. **Unseals Vault** — Finds pods in the configured namespace whose owner matches the configured workload (for example a Vault `StatefulSet`) and whose main container is not ready. For each such pod it runs `vault operator unseal` using keys supplied via environment variables named `unseal_key_*` (from the credentials secret).

2. **Refreshes External Secrets** — Discovers Custom Resource Definitions whose API group matches `external-secrets.io`, then annotates every instance of those resource kinds cluster-wide. Namespaced resources get `refresh-connection-es-to-vault`; cluster-scoped resources get `external-secrets-vault-refresh`, both set to the current UTC timestamp. That pattern triggers reconciliation against Vault according to how your External Secrets setup is configured.

The workload uses the OpenShift CLI (`oc`) from `quay.io/openshift/origin-cli:latest`.

## Prerequisites

- Helm 3
- Cluster where you can install chart resources (CronJob, ConfigMaps, ServiceAccount, RBAC)
- A namespace where Vault runs (default in values: `vault`)
- A **Secret** referenced by `cronjobs.refreshConnectionEsToVault.hashicorpVault.credentialsSecret.name` that provides at least:
  - Variables expected by your Vault usage (for example token-related settings if required by your script or tooling)
  - `unseal_key_*` variables whose values are the unseal keys used by `vault operator unseal`
- Appropriate permissions for the job’s ServiceAccount (the chart binds it to the `admin` `ClusterRole` in the Vault namespace and adds a `ClusterRole` for CRDs and `external-secrets.io` resources)

Review and tighten RBAC for your environment before production use.

## Install

From the chart directory:

```bash
helm install refresh-connection-es-to-vault . -n <target-namespace> --create-namespace
```

Override defaults with `-f` or `--set` (see [Configuration](#configuration)).

## Configuration

| Value | Description | Default |
|--------|-------------|---------|
| `cronjobs.refreshConnectionEsToVault.namespace` | Namespace where Vault pods are listed and where the ServiceAccount is bound | `vault` |
| `cronjobs.refreshConnectionEsToVault.schedule` | Cron schedule for the job | `*/5 * * * *` (every 5 minutes) |
| `cronjobs.refreshConnectionEsToVault.hashicorpVault.credentialsSecret.name` | Secret name for Vault credentials and `unseal_key_*` env vars | `vault-token` |
| `cronjobs.refreshConnectionEsToVault.hashicorpVault.pods.ownerReferences` | Identifies Vault pods: `apiVersion`, `type` (e.g. `StatefulSet`), `name` | `apps/v1`, `StatefulSet`, `vault` |
| `cronjobs.refreshConnectionEsToVault.externalSecrets.apiGroup` | API group used to find External Secrets CRDs | `external-secrets.io` |

## Example `credentialsSecret`

The CronJob loads this secret with `envFrom` and the script runs `vault operator unseal` once per environment variable whose name matches `unseal_key_*`. Use one key per shard of your unseal configuration (order does not need to match Vault’s key order; each key is applied in shell iteration order).

Create the secret in the **same namespace as the Helm release** (where the CronJob runs). That may or may not match `cronjobs.refreshConnectionEsToVault.namespace`, which is only where the chart looks for Vault pods. The secret name must match `cronjobs.refreshConnectionEsToVault.hashicorpVault.credentialsSecret.name` (default `vault-token`).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: my-release-namespace
type: Opaque
stringData:
  unseal_key_1: "replace-with-first-unseal-key"
  unseal_key_2: "replace-with-second-unseal-key"
  unseal_key_3: "replace-with-third-unseal-key"
```
