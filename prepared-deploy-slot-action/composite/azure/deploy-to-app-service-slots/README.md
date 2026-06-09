# Deploy to App Service with Slot Management

Zero-downtime deployment to Azure App Service using deployment slots, with health
checks and automatic rollback.

This action does **not** log in to Azure itself — it only runs `az ...` commands.
Azure authentication must happen **before** this action is called (see prerequisites).

---

## How it works

1. Stops the deployment slot and (optionally) creates it if it does not exist.
2. Sets the container image on the slot and starts the slot.
3. Waits for application startup and polls the health-check endpoint (retries with timeout).
4. If the health check passes → **swap** the slot into production and stop the old slot.
5. If the health check fails → **rollback** (stop the slot) and fail the job.

---

## Prerequisites (on the consumer repository side)

- A runner with Azure CLI installed (`ubuntu-latest` ships with `az` preinstalled).
- **Azure login before this action** (e.g. via `Azure/login@v2` or the shared `azurelogin` composite action).
- For OIDC login, the job requires the following permissions:
  ```yaml
  permissions:
    id-token: write
    contents: read
  ```

---

## Usage (cross-repo, Internal repository → `@main`)

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Azure Login (OIDC)
        uses: <owner>/<repo>/composite/azure/azurelogin@main
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to App Service Slot
        id: deploy
        uses: <owner>/<repo>/composite/azure/deploy-to-app-service-slots@main
        with:
          resource-group: my-resource-group
          app-service-name: my-app-service
          image-uri: myregistry.azurecr.io/myapp:1.0.0
          deployment-slot-name: deploy
          health-check-path: /health

      - name: Show result
        run: |
          echo "Status: ${{ steps.deploy.outputs.deployment-status }}"
          echo "URL:    ${{ steps.deploy.outputs.slot-url }}"
```

> **Note on versioning:** the repository is **Internal**, so we pin to `@main`.
> If the repo becomes public or you start tagging releases in the future,
> pinning to a tag (e.g. `@v1`) is recommended instead of the moving `@main` branch.

---

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `resource-group` | yes | – | Azure Resource Group name |
| `app-service-name` | yes | – | Azure App Service name (without slot suffix) |
| `image-uri` | yes | – | Full Docker image URI (`registry/repository:tag`) |
| `deployment-slot-name` | no | `deploy` | Deployment slot name |
| `subscription-id` | no | – | Azure subscription ID; if provided, it is set via `az account set` |
| `create-slot-if-missing` | no | `true` | Create the slot if it does not exist |
| `health-check-path` | no | `/` | Health-check endpoint path |
| `health-check-timeout-seconds` | no | `300` | Total health-check timeout (s) |
| `health-check-retries` | no | `10` | Number of health-check attempts |
| `health-check-interval-seconds` | no | `15` | Wait time between attempts (s) |
| `startup-wait-seconds` | no | `30` | Wait time for app startup before the first attempt (s) |
| `swap-timeout-seconds` | no | `600` | Slot swap operation timeout (s) |
| `rollback-on-health-failure` | no | `true` | Automatically roll back on a failed health check |
| `enable-debug` | no | `false` | Extra diagnostic logging |

## Outputs

| Name | Description |
|---|---|
| `deployment-status` | `success`, `failure`, `health-check-failed`, or `swap-failed` |
| `slot-url` | URL of the deployment slot |
| `swap-timestamp` | Slot swap timestamp (ISO 8601), if it succeeded |
| `deployment-duration` | Total deployment duration (s) |
