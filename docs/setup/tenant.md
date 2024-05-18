# Tenant file structure

The tenant file contains all properties and settings that are defined at tenant root level. Below is a empty base

```json
{
  "$schema": "http://schema.ca8.io/schemas/2024-01-01/tenant.json",
  "tenantName": "CLOUDGRIP",
  "tenantId": "00000000-0000-0000-0000-000000000000",
  "primaryDomain": "ca8.io",
  "partnerId": "0000000",
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
