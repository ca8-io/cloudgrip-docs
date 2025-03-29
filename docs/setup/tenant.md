# 📘 Tenant Schema Documentation  

The tenant file contains all properties and settings that are defined at tenant root level. Below is a empty base

```json
{
  "$schema": "http://schema.ca8.io/schemas/2024-01-01/tenant.json",
  "tenantName": "CLOUDGRIP",
  "tenantId": "00000000-0000-0000-0000-000000000000",
  "primaryDomain": "ca8.io",
  "rootManagementGroup": {
    "managementGroupId": "00000000-0000-0000-0000-000000000000",
    "policy": {
      "initiative": [],
      "policy": []
    },
    "rbac": [],
    "pimAssignments": [],
    "subscriptions": []
  },
  "managementGroups": [],
  "marketPlaceFeatures": []
}
```

**Schema URL:** `http://schema.ca8.io/schemas/2024-01-01/tenant.json`  
**Version:** 2024-01-01  
**Purpose:** Define and manage a CloudGrip tenant and its associated Azure resources.

---

## 🔗 Top-Level Structure

| Property              | Type                     | Required | Description                                         |
|-----------------------|--------------------------|----------|-----------------------------------------------------|
| `$schema`             | `string` (uri)           | ✅        | URI to the schema definition.                      |
| `tenantName`          | `string`                 | ✅        | Display name of the tenant.                        |
| `tenantId`            | `string` (uuid)          | ✅        | Azure Active Directory (AAD) Tenant ID.            |
| `primaryDomain`       | `string`                 | ✅        | Primary verified domain (e.g., `example.com`).     |
| `rootManagementGroup` | `object`                 | ✅        | Root management group definition.                  |
| `managementGroups`    | `array<ManagementGroup>` | ✅        | Additional management groups under the root.       |
| `marketPlaceFeatures` | `array<MarketplaceItem>` | ✅        | Marketplace features to enable.                    |

---

## 🗂 Root Management Group

Defines the foundational structure of the Azure environment.

### Properties

| Property           | Type                    | Required | Description                                            |
|--------------------|-------------------------|----------|--------------------------------------------------------|
| `managementGroupId`| `string`                | ✅        | ID for the root management group.                      |
| `policy`           | `object`                | ✅        | Policy assignments: includes `initiative` & `policy`.  |
| `rbac`             | `array<RBAC>`           | ✅        | Role assignments for this group.                       |
| `pimAssignments`   | `array<PIMAssignment>`  | ✅        | Privileged Identity Management (PIM) roles.            |
| `subscriptions`    | `array<Subscription>`   | ✅        | Subscriptions under this group.                        |

---

## 🔒 RBAC Definition

Role-Based Access Control for users, groups, and SPNs.

| Property             | Type              | Required | Notes                                     |
|----------------------|-------------------|----------|-------------------------------------------|
| `name`               | `string`          | ✅        | Role name (`Owner`, `Reader`, etc.)       |
| `assignedUsers`      | `array<email>`    | ✅        | List of user emails.                      |
| `assignedGroups`     | `array<string>`   | ✅        | AAD group names.                          |
| `assignedSpnObjectIds`| `array<uuid>`    | ✅        | List of service principal object IDs.     |

---

## 🧑‍💼 Management Group

Represents an Azure management group.

| Property      | Type     | Required | Notes                                                     |
|---------------|----------|----------|-----------------------------------------------------------|
| `name`        | `string` | ✅        | Friendly name of the management group.                    |
| `id`          | `string` | ✅        | Unique identifier.                                        |
| `managedBy`   | `enum` (`parent` \| `self`) | ✅ | Specifies if it's self-managed or inherited.             |
| `repoUrl`     | `uri`    | ⚠️*       | Required when `managedBy` = `self`.                      |
| `ownerEmail`  | `email`  | ⚠️*       | Required when `managedBy` = `self`.                      |

---

## 📦 Subscription

An Azure subscription managed under a group.

| Property      | Type     | Required | Notes                                                     |
|---------------|----------|----------|-----------------------------------------------------------|
| `name`        | `string` | ✅        | Subscription name.                                        |
| `id`          | `uuid`   | ✅        | Subscription GUID.                                        |
| `managedBy`   | `enum` (`parent` \| `self`) | ✅ | Specifies management source.                             |
| `repoUrl`     | `uri`    | ⚠️*       | Required when `managedBy` = `self`.                      |
| `ownerEmail`  | `email`  | ⚠️*       | Required when `managedBy` = `self`.                      |

---

## 🔐 PIM Assignment

Privileged Identity Management assignment (structure is defined externally).

### Required Properties

- `name`
- `assignedUsers`
- `assignedGroups`
- `activationDurationHours`
- `activationRequirements`
- `activationAlertEmails`
- `activationApproverGroups`

---

## 🛒 Marketplace Feature

Enables CA8 Marketplace integrations.

| Property    | Type     | Required | Description                                        |
|-------------|----------|----------|----------------------------------------------------|
| `name`      | `string` | ✅        | Feature display name (e.g., CloudGrip).            |
| `offerId`   | `string` | ✅        | Internal marketplace ID (`ca8-cloudgrip`, etc.).  |
| `properties`| `object` | ✅        | Custom configuration options per feature.          |

---

**Note:** `⚠️*` indicates conditional requirements based on the value of other fields.
