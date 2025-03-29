# 📘 Subscription Schema Documentation  

```json
{
  "$schema": "http://schema.ca8.io/schemas/2024-01-01/subscription.json",
  "tenantId": "00000000-0000-0000-0000-000000000000",
  "parentManagementGroupId": "alz-platform-management",
  "subscriptionName": "management",
  "subscriptionId": "00000000-0000-0000-0000-000000000000",
  "location": "westeurope",
  "centralConfig": {},
  "deployments": [],
  "deploymentStacks": [],
  "tags": {},
  "rbac": [],
  "marketPlaceFeatures": []
}
```

**Schema URL:** `http://schema.ca8.io/schemas/2024-01-01/subscription.json`  
**Version:** 2024-01-01  
**Purpose:** Defines the structure and configuration of a CloudGrip subscription object.

---

## 🔗 Top-Level Structure

| Property                  | Type                         | Required | Description |
|---------------------------|------------------------------|----------|-------------|
| `$schema`                 | `string (uri)`               | ✅        | Fixed URI to schema version. |
| `tenantId`                | `uuid`                       | ✅        | Tenant ID this subscription belongs to. |
| `billingAccountId`        | `string`                     | ❌        | Azure billing account ID. |
| `partnerId`               | `string`                     | ❌        | Partner/Reseller ID. |
| `parentManagementGroupId`| `string`                     | ✅        | Management group under which this subscription resides. |
| `subscriptionName`        | `string`                    | ✅        | Friendly name (e.g., "Corp", "Sandbox"). |
| `subscriptionId`          | `uuid`                      | ✅        | Azure subscription ID. |
| `location`                | `enum`                      | ✅        | Azure region or logical location. |
| `centralConfig`           | `object`                    | ✅        | Optional shared services configuration. |
| `deployments`             | `array<Deployment>`         | ✅        | Bicep/ARM/Terraform deployments. |
| `deploymentStacks`        | `array`                     | ✅        | Placeholder for deployment stacks. |
| `tags`                    | `object`                    | ✅        | Custom tags (e.g., env, cost center). |
| `rbac`                    | `array<RBAC>`               | ✅        | Role assignments for users, groups, SPNs. |
| `marketPlaceFeatures`     | `array<MarketplaceItem>`    | ✅        | Marketplace integrations to enable. |

---

## 🔐 RBAC

Role-Based Access Control entries.

| Property               | Type            | Description |
|------------------------|-----------------|-------------|
| `name`                 | `string`        | Role name (e.g. Owner, Reader). |
| `assignedUsers`        | `array<email>`  | Direct user access. |
| `assignedGroups`       | `array<string>` | Group names (e.g. BU-XXX). |
| `assignedSpnObjectIds` | `array<uuid>`   | SPN object IDs for automation. |

---

## 🚀 Deployment

Defines infrastructure deployments per environment or service.

| Property                    | Type      | Required | Description |
|-----------------------------|-----------|----------|-------------|
| `name`                      | `string`  | ✅        | Logical deployment name. |
| `type`                      | `enum`    | ✅        | `bicep`, `arm`, or `terraform`. |
| `templateFilePath`          | `string`  | ⚠️        | Required for `bicep` or `arm`. |
| `templateParameterFilePath`| `string`  | ❌        | Optional parameters file. |
| `preScriptPath`             | `string`  | ❌        | Optional pre-deployment script. |
| `postScriptPath`            | `string`  | ❌        | Optional post-deployment script. |
| `deploymentGuid`            | `string`  | ✅        | Unique deployment identifier. |
| `resourceGroupName`         | `string`  | ❌        | Target resource group name. |

---

## 🌐 Central Config

Optional shared services integration.

| Property           | Type     | Description |
|--------------------|----------|-------------|
| `vhubResourceId`   | `string` | ID of connected Virtual Hub. |
| `vnetResourceId`   | `string` | ID of Virtual Network. |
| `lawsResourceId`   | `string` | ID of Log Analytics workspace. |

All properties must match the Azure resource ID pattern.

---

## 🏷 Tags

Custom metadata as key-value pairs.

**Example:**

```json
{
  "environment": "dev",
  "costCenter": "12345"
}
```

## 🛒 Marketplace Feature

Enables CA8 Marketplace integrations.

| Property    | Type     | Required | Description                                        |
|-------------|----------|----------|----------------------------------------------------|
| `name`      | `string` | ✅        | Feature display name (e.g., CloudGrip).            |
| `offerId`   | `string` | ✅        | Internal marketplace ID (`ca8-cloudgrip`, etc.).  |
| `properties`| `object` | ✅        | Custom configuration options per feature.          |

---

**Note:** `⚠️*` indicates conditional requirements based on the value of other fields.
