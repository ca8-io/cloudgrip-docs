# 📘 Management Group Schema Documentation  

```json
{
  "$schema": "http://schema.ca8.io/schemas/2024-01-01/managementgroup.json",
  "tenantId": "00000000-0000-0000-0000-000000000000",
  "parentManagementGroupId": "alz-platform",
  "managementGroupId": "alz-platform-management",
  "managementGroupName": "Platform",
  "managementGroups": [],
  "policy": {
    "initiative": [],
    "policy": []
  },
  "rbac": [],
  "pimAssignments": [],
  "subscriptions": []
}
```

**Schema URL:** `http://schema.ca8.io/schemas/2024-01-01/managementgroup.json`  
**Version:** 2024-01-01  
**Purpose:** Describes the structure and configuration of a CloudGrip Management Group.

---

## 🔗 Top-Level Structure

| Property                     | Type                       | Required | Description |
|------------------------------|----------------------------|----------|-------------|
| `$schema`                    | `string (uri)`             | ✅        | Fixed URI for this schema version. |
| `tenantId`                   | `uuid`                     | ✅        | Azure Active Directory (AAD) Tenant ID. |
| `parentManagementGroupId`   | `string`                   | ✅        | ID of the parent management group. |
| `managementGroupId`         | `string`                   | ✅        | Unique identifier of the management group. |
| `managementGroupName`       | `string`                   | ✅        | Display name of the management group. |
| `managementGroups`          | `array<ManagementGroup>`   | ✅        | Child management groups. |
| `policy`                    | `object`                   | ✅        | Policy and initiative assignments. |
| `rbac`                      | `array<RBAC>`              | ✅        | RBAC roles assigned at the group level. |
| `subscriptions`             | `array<Subscription>`      | ✅        | Subscriptions under this group. |
| `pimAssignments`            | `array<PIMAssignment>`     | ⬅️ optional | Optional PIM assignment entries. |

---

## 🧩 Management Group Definition

### Properties

| Property     | Type                  | Required | Notes |
|--------------|-----------------------|----------|-------|
| `name`       | `string`              | ✅        | Logical display name. |
| `id`         | `string`              | ✅        | Short identifier (e.g. `alz`). |
| `managedBy`  | `enum: parent/self`   | ✅        | Whether managed internally (`self`) or inherited. |
| `repoUrl`    | `uri`                 | ⚠️        | Required if `managedBy = self`. |
| `ownerEmail` | `email`               | ⚠️        | Required if `managedBy = self`. |

---

## 🔐 RBAC

Role-Based Access Control entries.

| Property               | Type            | Description |
|------------------------|-----------------|-------------|
| `name`                 | `string`        | Role name (e.g. Owner, Reader). |
| `assignedUsers`        | `array<email>`  | Direct user access. |
| `assignedGroups`       | `array<string>` | Azure AD group names. |
| `assignedSpnObjectIds` | `array<uuid>`   | SPN object IDs for automation. |

---

## 🛡️ Policy

### Structure

| Key        | Type       | Description                    |
|------------|------------|--------------------------------|
| `initiative`| `array`    | Policy initiatives to assign.  |
| `policy`    | `array`    | Individual policies to assign. |

---

## 📦 Subscription Definition

| Property     | Type      | Required | Notes |
|--------------|-----------|----------|-------|
| `name`       | `string`  | ✅        | Name of the subscription. |
| `id`         | `uuid`    | ✅        | Subscription GUID. |
| `managedBy`  | `enum`    | ✅        | Either `parent` or `self`. |
| `repoUrl`    | `uri`     | ⚠️        | Required if `managedBy = self`. |
| `ownerEmail` | `email`   | ⚠️        | Required if `managedBy = self`. |

---

## 🔐 PIM Assignment *(Optional)*

Structure not explicitly defined in schema. Required fields include:

- `name`
- `assignedUsers`
- `assignedGroups`
- `activationDurationHours`
- `activationRequirements`
- `activationAlertEmails`
- `activationApproverGroups`

---

**Note:** `⚠️` indicates conditional requirement based on another field.
