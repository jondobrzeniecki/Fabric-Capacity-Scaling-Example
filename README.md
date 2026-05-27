# Fabric Capacity Scaling Example

> **Disclaimer:** Provided diagrams, documents, and code are provided AS IS without warranty of any kind and should not be interpreted as an offer or commitment on the part of Microsoft, and Microsoft cannot guarantee the accuracy of any information presented. MICROSOFT MAKES NO WARRANTIES, EXPRESS OR IMPLIED, IN THIS DIAGRAM(s) DOCUMENT(s) CODE SAMPLE(s).

## Overview

[scheduled_scaling_shareable.ipynb](scheduled_scaling_shareable.ipynb) is a working example of how to programmatically scale a Microsoft Fabric capacity SKU from a notebook running inside a Fabric workspace. The intended use case is running it on a schedule so a capacity can, for example, scale up before business hours and scale back down at the end of the day to control cost.

This notebook is a simpler, schedule-driven variant of the more comprehensive autoscale example in the [FabricTools / CapacityAutoScale](https://github.com/bretamyers/FabricTools/blob/main/CapacityAutoScale/NB_CapacityAutoScale.ipynb) repository. This example focuses on the simpler case of scaling on a fixed schedule (for example, up in the morning and down in the evening) and trims the code down to the minimum needed to do that.

The notebook:

1. Reads service principal credentials from Azure Key Vault.
2. Acquires Microsoft Entra tokens for the Power BI and Azure Resource Manager audiences.
3. Calls the `Microsoft.Fabric/capacities` REST API to PATCH the capacity to a target SKU.
4. Polls the capacity until the provisioning state reports `Succeeded`.

> **Note on scaling between F256 and F512:** Per the official documentation, scaling up or down between SKUs at or below F256 and SKUs at or above F512 might result in a slower experience. Plan these transitions accordingly. See [Scale your Fabric capacity](https://learn.microsoft.com/fabric/enterprise/scale-capacity) for details.

See the [Fabric Capacities REST API](https://learn.microsoft.com/rest/api/microsoftfabric/fabric-capacities?view=rest-microsoftfabric-2023-11-01) documentation for the full list of REST operations available on `Microsoft.Fabric/capacities` (Get, Update, List, Suspend, Resume, etc.) including the ones this example calls to read and update the capacity SKU.

## Prerequisites

- A Microsoft Fabric workspace where you can upload and schedule notebooks.
- A Microsoft Fabric capacity (F SKU) in an Azure subscription.
- An Azure Key Vault.
- A Microsoft Entra app registration (service principal) with a client secret. The `tenantId`, `clientId`, and client `secret` should be stored as secrets in your Key Vault.

## Authentication setup

There are two pieces of authentication to configure: reading the service principal credentials from Key Vault, and granting the service principal permission to scale the capacity.

### 1. Reading secrets from Azure Key Vault

The notebook uses `notebookutils.credentials.getSecret(keyVaultEndpoint, secretName)` to retrieve the service principal's `tenantId`, `clientId`, and `secret` from Key Vault. This call runs as the identity executing the notebook (the interactive user, or for scheduled runs the user who created/last updated the schedule), and that identity must have `Get` permission on the secrets.

For full details on how `getSecret` authenticates and the permissions it requires, see the official documentation: [NotebookUtils credentials utilities for Fabric](https://learn.microsoft.com/fabric/data-engineering/notebookutils/notebookutils-credentials).

### 2. Granting the service principal permission to scale the capacity

Add the service principal as a **Capacity admin** on the target Fabric capacity. This gives it the rights needed to manage the capacity's SKU through the REST API.

For step-by-step instructions, see: [Configure and manage capacity settings — Capacity admins](https://learn.microsoft.com/en-us/fabric/admin/capacity-settings?tabs=fabric-capacity#add-and-remove-admins).

## Configuration

Open the notebook and update the placeholder values in the first two configuration cells.

**Cell 1 — target capacity:**

| Variable | Description |
| --- | --- |
| `targetSku` | The SKU to scale to, e.g. `F4`, `F8`, `F64`. |
| `capacityName` | Name of the Fabric capacity to scale. |
| `subscriptionId` | Azure subscription ID containing the capacity. |
| `resourceGroup` | Resource group containing the capacity. |

**Cell 3 — Key Vault and secret names:**

| Variable | Description |
| --- | --- |
| `keyVaultEndpoint` | Your Key Vault URL, e.g. `https://<key-vault-name>.vault.azure.net/`. |
| `tenantId` secret name | Name of the AKV secret holding the service principal's tenant ID. |
| `clientId` secret name | Name of the AKV secret holding the service principal's client (application) ID. |
| `secret` secret name | Name of the AKV secret holding the service principal's client secret. |

For testing you can hard-code these values directly in the notebook instead of pulling from Key Vault, but do not commit a notebook with hard-coded credentials.

## Usage

1. Upload `scheduled_scaling_shareable.ipynb` to a Fabric workspace.
2. Update the configuration cells described above.
3. Run the notebook interactively to confirm the scaling works end-to-end.
4. Create a Fabric schedule for the notebook. A common pattern is to maintain two copies of the notebook with different `targetSku` values — one scheduled to scale up at the start of the working day and one scheduled to scale down at the end.

## License

See [LICENSE](LICENSE).
