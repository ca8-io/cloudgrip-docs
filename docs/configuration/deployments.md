# CloudGrip Configuration Guide: Deployments & RBAC

## Table of Contents

- [Overview](#overview)
- [Understanding CloudGrip Architecture](#understanding-cloudgrip-architecture)
- [Configuration Levels](#configuration-levels)
- [Deployment Configuration](#deployment-configuration)
- [Deployment Properties Reference](#deployment-properties-reference)
- [RBAC Configuration](#rbac-configuration)
- [RBAC Properties Reference](#rbac-properties-reference)
- [Best Practices](#best-practices)
- [Pipeline Integration](#pipeline-integration)

## Overview

CloudGrip is a hierarchical infrastructure-as-code platform that manages Azure resources at three levels: **Tenant**, **Management Group**, and **Subscription**. It uses JSON configuration files with defined schemas to declare infrastructure state, RBAC assignments, and deployment specifications.

The platform supports multiple deployment technologies:

- **Bicep** templates
- **ARM** (Azure Resource Manager) templates  
- **Terraform** configurations

## Understanding CloudGrip Architecture

CloudGrip follows a **hierarchical, declarative configuration model** based on Azure's management structure:

```
Tenant (Root)
├── Root Management Group
│   ├── RBAC Assignments
│   ├── Policy Assignments
│   └── Subscriptions
└── Child Management Groups
    ├── RBAC Assignments
    ├── Policy Assignments
    ├── Subscriptions
    │   ├── Deployments (Bicep/ARM/Terraform)
    │   └── RBAC Assignments
    └── Child Management Groups
```

### Key Concepts

1. **Hierarchical Structure**: Mirrors Azure's tenant → management group → subscription hierarchy
2. **Schema-Driven**: All configurations validated against JSON schemas
3. **Declarative State**: Define desired state; pipelines handle execution
4. **Deployment Versioning**: `deploymentGuid` controls when redeployments occur
5. **Automated Pipelines**: CI/CD pipelines detect changes and execute deployments
6. **Multi-Technology Support**: Use Bicep, ARM, or Terraform within the same platform

## Configuration Levels

### Tenant Level (`tenant.json`)

The tenant configuration defines the root-level Azure tenant settings.

**Schema**: `http://schema.ca8.io/schemas/2024-01-01/tenant.json`

**Key Properties**:

- `tenantName`: Human-readable tenant name
- `tenantId`: Azure AD tenant GUID
- `primaryDomain`: Primary domain (e.g., `company.onmicrosoft.com`)
- `rootManagementGroup`: Root management group configuration
- `managementGroups`: Array of child management groups
- `marketPlaceFeatures`: Marketplace offerings to enable

### Management Group Level (`managementgroup.json`)

Management groups provide organizational structure and inheritance of policies and RBAC.

**Schema**: `http://schema.ca8.io/schemas/2024-01-01/managementgroup.json`

**Key Properties**:

- `tenantId`: Parent tenant GUID
- `parentManagementGroupId`: Parent management group ID
- `managementGroupId`: This management group's unique ID
- `managementGroupName`: Display name
- `managementGroups`: Child management groups
- `subscriptions`: Subscriptions under this management group
- `policy`: Policy and initiative assignments
- `rbac`: RBAC role assignments at MG scope

### Subscription Level (`subscription.json`)

Subscriptions contain the actual resource deployments and detailed configurations.

**Schema**: `http://schema.ca8.io/schemas/2024-01-01/subscription.json`

**Key Properties**:

- `subscriptionId`: Azure subscription GUID
- `subscriptionName`: Display name
- `location`: Default Azure region
- `deployments`: Array of deployment objects **(main focus)**
- `rbac`: RBAC assignments at subscription scope
- `tags`: Subscription-level tags
- `centralConfig`: Shared infrastructure references (VNet, Log Analytics, etc.)

## Deployment Configuration

Deployments are defined in the `deployments` array within a **subscription configuration file**.

### Deployment Object Structure

```json
{
  "name": "deployment-name",
  "type": "bicep|arm|terraform",
  "templateFilePath": "path/to/template.bicep",
  "templateParameterFilePath": "path/to/parameters.bicepparam",
  "deploymentGuid": "803de292",
  "resourceGroupName": "rg-resource-group",
  "preScriptPath": "scripts/pre-deploy.ps1",
  "postScriptPath": "scripts/post-deploy.ps1"
}
```

### Complete Example

```json
{
  "deployments": [
    {
      "name": "storage",
      "type": "bicep",
      "templateFilePath": "main.bicep",
      "templateParameterFilePath": "main.prod.bicepparam",
      "deploymentGuid": "a1b2c3d4",
      "resourceGroupName": "rg-storage-prod",
      "preScriptPath": "pre-deploy.ps1",
      "postScriptPath": "post-deploy.ps1"
    },
    {
      "name": "network",
      "type": "arm",
      "templateFilePath": "vnet.json",
      "templateParameterFilePath": "vnet.parameters.json",
      "deploymentGuid": "5e6f7g8h",
      "resourceGroupName": "rg-network-prod"
    },
    {
      "name": "monitoring",
      "type": "terraform",
      "deploymentGuid": "9i0j1k2l",
      "resourceGroupName": "rg-monitoring"
    }
  ]
}
```

## Deployment Properties Reference

### `name` (required)

- **Type**: String
- **Description**: Unique identifier for this deployment
- **Example**: `"devopsagents"`, `"storage-infrastructure"`
- **Constraints**: Must be unique within the subscription configuration

### `type` (required)

- **Type**: String (enum)
- **Description**: Deployment technology to use
- **Valid Values**:
  - `"bicep"` - Bicep template deployment
  - `"arm"` - ARM JSON template deployment
  - `"terraform"` - Terraform configuration
- **Example**: `"bicep"`
- **Pattern**: `^(bicep|arm|terraform)$`

### `templateFilePath` (required for bicep/arm)

- **Type**: String
- **Description**: Path to the template file relative to repository root
- **Required When**: `type` is `"bicep"` or `"arm"`
- **Not Required When**: `type` is `"terraform"`
- **Examples**:
  - `"main.bicep"`
  - `"templates/storage.json"`
  - `"infrastructure/network/vnet.bicep"`

### `templateParameterFilePath` (optional)

- **Type**: String
- **Description**: Path to parameter file for the template
- **Examples**:
  - `"main.dev.bicepparam"` (Bicep parameters)
  - `"storage.parameters.json"` (ARM parameters)
- **Note**: If omitted, template must have default values or use inline parameters

### `deploymentGuid` (required)

- **Type**: String (8-character hex)
- **Description**: **Version control identifier** - changing this triggers redeployment
- **Format**: 8 lowercase hexadecimal characters
- **Pattern**: `^([a-f0-9]{8})$`
- **Examples**: `"803de292"`, `"a1b2c3d4"`, `"5e6f7g8h"`

**Critical Behavior**:

- This is the **deployment version fingerprint**
- Pipeline compares current `deploymentGuid` with last deployed value
- **Same GUID** → Configuration unchanged → **No deployment**
- **Different GUID** → New version → **Deployment executes**
- This prevents unnecessary redeployments and gives explicit control

**How to Use**:

1. Initial deployment: Set any 8-char hex value (e.g., `"00000001"`)
2. Make template/parameter changes
3. To deploy changes: **Change the GUID** (e.g., `"00000002"`)
4. Commit and push
5. Pipeline detects GUID change and deploys

**Example Workflow**:

```
Initial State:
"deploymentGuid": "803de292"
→ Deploy storage account

Update template, don't deploy yet
"deploymentGuid": "803de292"  // SAME - no deployment
→ Pipeline skips deployment

Change GUID to deploy update:
"deploymentGuid": "804de292"  // CHANGED
→ Pipeline deploys updated template
```

### `resourceGroupName` (optional)

- **Type**: String
- **Description**: Target resource group for deployment
- **Example**: `"rg-devopsagents"`, `"rg-storage-prod"`
- **Note**: If resource group doesn't exist, deployment may create it or fail depending on template

### `preScriptPath` (optional)

- **Type**: String
- **Description**: PowerShell script to run **before** deployment
- **Example**: `"pre.ps1"`, `"scripts/pre-deploy.ps1"`
- **Use Cases**:
  - Environment validation
  - Prerequisite resource checks
  - Configuration preparation
  - Secret retrieval from Key Vault

### `postScriptPath` (optional)

- **Type**: String
- **Description**: PowerShell script to run **after** successful deployment
- **Example**: `"post.ps1"`, `"scripts/configure-app.ps1"`
- **Use Cases**:
  - Post-deployment configuration
  - Application setup
  - Integration tests
  - Notification sending

## RBAC Configuration

RBAC (Role-Based Access Control) can be configured at three levels in CloudGrip:

1. **Tenant Root Management Group** - Inherited by all child resources
2. **Management Group Level** - Inherited by subscriptions and child MGs
3. **Subscription Level** - Applies to subscription and its resource groups

### RBAC Object Structure

```json
{
  "name": "Contributor",
  "assignedUsers": [
    "user@example.com"
  ],
  "assignedGroups": [
    "DevOps Team",
    "ENGINEERING/CLOUD-PLATFORM"
  ],
  "assignedSpnObjectIds": [
    "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  ]
}
```

### Complete Examples

#### Subscription-Level RBAC

```json
{
  "rbac": [
    {
      "name": "Owner",
      "assignedUsers": [
        "admin@company.com"
      ],
      "assignedGroups": [
        "Platform Admins"
      ],
      "assignedSpnObjectIds": []
    },
    {
      "name": "Contributor",
      "assignedUsers": [],
      "assignedGroups": [
        "DevOps Engineers",
        "ENGINEERING/DEVELOPERS"
      ],
      "assignedSpnObjectIds": [
        "12345678-1234-1234-1234-123456789012"
      ]
    },
    {
      "name": "Reader",
      "assignedUsers": [
        "auditor@company.com"
      ],
      "assignedGroups": [
        "Security Team"
      ],
      "assignedSpnObjectIds": []
    }
  ]
}
```

#### Management Group RBAC with Inheritance

```json
{
  "managementGroupId": "mg-production",
  "rbac": [
    {
      "name": "Security Admin",
      "assignedUsers": [],
      "assignedGroups": [
        "Security Operations Center"
      ],
      "assignedSpnObjectIds": []
    }
  ]
}
```

## RBAC Properties Reference

### `name` (required)

- **Type**: String
- **Description**: Azure built-in or custom role name
- **Common Built-in Roles**:
  - `"Owner"` - Full access including access management
  - `"Contributor"` - Manage resources, cannot grant access
  - `"Reader"` - View resources only
  - `"User Access Administrator"` - Manage user access only
  - Resource-specific roles (e.g., `"Storage Blob Data Contributor"`)
- **Example**: `"Contributor"`

### `assignedUsers` (required)

- **Type**: Array of strings (email addresses)
- **Description**: Email addresses of Azure AD users to assign this role
- **Format**: Valid email addresses
- **Example**: `["user@example.com", "admin@company.com"]`
- **Note**: Can be empty array `[]` if using only groups/service principals

### `assignedGroups` (required)

- **Type**: Array of strings
- **Description**: Names of Azure AD groups to assign this role
- **Examples**:
  - `["DevOps Team"]`
  - `["ENGINEERING/PLATFORM", "BU-CLOUDTEAM"]`
- **Note**: Group names are case-sensitive; can be empty array `[]`

### `assignedSpnObjectIds` (required)

- **Type**: Array of strings (GUIDs)
- **Description**: Object IDs of service principals (managed identities, app registrations)
- **Format**: Valid UUID/GUID format
- **Pattern**: `^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`
- **Example**: `["a1b2c3d4-e5f6-7890-abcd-ef1234567890"]`
- **Note**: Can be empty array `[]`; find in Azure Portal → Enterprise Applications → Object ID

### Scope and Inheritance

RBAC assignments automatically inherit down the hierarchy:

```
Root Management Group (Owner: Platform Admins)
├── Management Group: Production (Contributor: DevOps Team)
│   ├── Subscription: Prod-East (Reader: Auditors)
│   │   └── Resource Group: rg-app
│   │       └── Storage Account
│   │           ↓
│   │       Effective Permissions:
│   │       - Platform Admins: Owner (inherited from root)
│   │       - DevOps Team: Contributor (inherited from MG)
│   │       - Auditors: Reader (from subscription)
```

## Best Practices

### Deployment Configuration Best Practices

1. **Semantic Deployment Names**

   ```json
   // Good - descriptive, scoped
   "name": "prod-eastus-storage-accounts"
   
   // Bad - vague
   "name": "deployment1"
   ```

2. **Deployment GUID Management**
   - Use sequential or timestamped hex values for traceability
   - Document GUID changes in commit messages
   - Example pattern: `YYYYMMDD` in hex: `"20240315"` → `"013c9f43"`

   ```pwsh
   # Generate timestamp-based GUID
   echo "obase=16; $(date +%Y%m%d)" | bc | tr '[:upper:]' '[:lower:]'

   # Use first part of GUID
   ((New-Guid) -replace '-').SubString(0, 8) | clip
   ```

3. **Organize by Environment and Function**

   ```
   lz-name-app-x/
   ├── networking/
   │   ├── main.bicep
   │   ├── main.dev.bicepparam
   │   └── main.prod.bicepparam
   ├── storage/
   │   └── ...
   └── compute/
       └── ...
   ```

4. **Parameter File Naming Convention**
   - `main.{environment}.bicepparam` for Bicep
   - `template.{environment}.parameters.json` for ARM
   - Examples: `main.dev.bicepparam`, `storage.prod.parameters.json`

5. **Use Pre/Post Scripts Wisely**
   - Keep scripts idempotent
   - Log all actions
   - Handle errors gracefully
   - Test scripts independently

6. **Template Type Selection**
   - **Bicep**: Modern, concise, type-safe (recommended for new deployments)
   - **ARM**: Mature, widely supported, verbose
   - **Terraform**: Multi-cloud, state management

### RBAC Configuration Best Practices

1. **Principle of Least Privilege**

   ```json
   // Prefer specific roles
   {
     "name": "Storage Blob Data Reader",  // Specific
     "assignedGroups": ["App Service Team"]
   }
   
   // Avoid overly broad access
   {
     "name": "Owner",  // Too broad - use sparingly
     "assignedGroups": ["Everyone"]
   }
   ```

2. **Prefer Groups Over Individual Users**

   ```json
   // Good - use groups
   {
     "name": "Contributor",
     "assignedUsers": [],
     "assignedGroups": ["Platform Engineering"],
     "assignedSpnObjectIds": []
   }
   
   // Bad - individual users (harder to maintain)
   {
     "name": "Contributor",
     "assignedUsers": ["user1@company.com", "user2@company.com", ...],
     "assignedGroups": [],
     "assignedSpnObjectIds": []
   }
   ```

3. **Document Role Assignments**
   - Use clear group names
   - Maintain group membership documentation
   - Regular access reviews

4. **Service Principal Management**
   - Use managed identities when possible
   - Document SPN purpose
   - Rotate credentials regularly

5. **Hierarchical Assignment Strategy**
   - Assign broad roles at management group level
   - Specific roles at subscription/resource group level
   - Review inheritance impacts

### Security Best Practices

1. **Secret Management**
   - Never commit secrets to configuration files
   - Use Azure Key Vault references in templates
   - Leverage Key Vault integration in Bicep/ARM

2. **Template Validation**
   - Use schema validation before committing
   - Run what-if analysis for ARM/Bicep
   - Test in non-production first

3. **Change Control**
   - Require pull request reviews
   - Mandate GUID changes for production deployments
   - Use branch protection rules

## Pipeline Integration

CloudGrip deployments are executed through automated CI/CD pipelines that monitor configuration changes.

### How Pipeline Deployment Works

1. **Configuration Change**: Developer modifies subscription JSON config
2. **Git Commit**: Changes committed and pushed to repository
3. **Pipeline Trigger**: Git push triggers CloudGrip deployment pipeline
4. **Configuration Parsing**: Pipeline loads and validates JSON against schema
5. **Deployment Detection**: For each deployment in `deployments` array:
   - Compare `deploymentGuid` with last deployed GUID (stored in state)
   - **If GUID unchanged**: Skip deployment, log "No version change"
   - **If GUID changed**: Queue deployment for execution
6. **RBAC Processing**: RBAC changes always processed (no GUID check)
7. **Deployment Execution**:
   - Run `preScriptPath` if specified
   - Execute deployment based on `type`:
     - **Bicep**: `az deployment group create --template-file ... --parameters ...`
     - **ARM**: `az deployment group create --template-file ...`
     - **Terraform**: `terraform plan && terraform apply`
   - Run `postScriptPath` if specified
8. **State Update**: Record deployed `deploymentGuid` in state storage
9. **Validation**: Post-deployment health checks and validation

### Pipeline Workflow Diagram

```
┌─────────────────────┐
│   Developer Edits   │
│  subscription.json  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Git Commit/Push   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Pipeline Triggered │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Validate Schema    │
│  Load Config JSON   │
└──────────┬──────────┘
           │
           ▼
┌────────────────────────────────────┐
│  For Each Deployment:              │
│  ┌─────────────────────────────┐   │
│  │ Compare deploymentGuid      │   │
│  │ with Last Deployed GUID     │   │
│  └──────────┬──────────────────┘   │
│             │                      │
│    ┌────────┴────────┐             │
│    │                 │             │
│    ▼                 ▼             │
│ ┌─────┐         ┌────────┐         │
│ │Same │         │Changed │         │
│ │GUID │         │  GUID  │         │
│ └──┬──┘         └───┬────┘         │
│    │                │              │
│    ▼                ▼              │
│ ┌─────┐      ┌──────────────┐      │
│ │Skip │      │Queue Deploy  │      │
│ └─────┘      └──────┬───────┘      │
│                     │              │
└─────────────────────┼──────────────┘
                      │
                      ▼
            ┌──────────────────┐
            │  Execute Queued  │
            │   Deployments    │
            └────────┬─────────┘
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
    ┌──────────┐         ┌──────────┐
    │  Pre-    │         │   RBAC   │
    │ Script   │         │  Updates │
    └────┬─────┘         └──────────┘
         │
         ▼
    ┌──────────────┐
    │   Deploy     │
    │   Template   │
    │ (Bicep/ARM/  │
    │  Terraform)  │
    └─────┬────────┘
          │
          ▼
    ┌──────────┐
    │  Post-   │
    │ Script   │
    └────┬─────┘
         │
         ▼
    ┌──────────────┐
    │  Update      │
    │  State with  │
    │  New GUID    │
    └──────────────┘
```

### The deploymentGuid Mechanism Explained

The `deploymentGuid` is CloudGrip's **deployment version control mechanism**:

**State Tracking**:

- Pipeline maintains a state file/database
- After each successful deployment, stores: `{ deploymentName: guid }`
- Example state: `{ "storage-infrastructure": "a1b2c3d4" }`

**Deployment Decision Logic**:

```
FOR EACH deployment in deployments:
  current_guid = deployment.deploymentGuid
  last_deployed_guid = state.get(deployment.name)
  
  IF current_guid == last_deployed_guid:
    LOG "Deployment {deployment.name}: GUID unchanged, skipping"
    CONTINUE
  
  IF current_guid != last_deployed_guid:
    LOG "Deployment {deployment.name}: GUID changed from {last_deployed_guid} to {current_guid}"
    EXECUTE_DEPLOYMENT(deployment)
    state.set(deployment.name, current_guid)
```

**Why This Design**:

1. **Explicit Versioning**: You control when deployments happen
2. **Prevents Unnecessary Work**: Saves time and Azure API calls
3. **Predictable Behavior**: No surprises from automatic redeployments
4. **Cost Optimization**: Reduces deployment frequency
5. **Audit Trail**: GUID changes visible in Git history

**Practical Example**:

```json
// Initial deployment
{
  "name": "app-infrastructure",
  "deploymentGuid": "00000001",
  "templateFilePath": "app.bicep"
}
// Deployment runs → State: { "app-infrastructure": "00000001" }

// Update template, don't deploy yet
{
  "name": "app-infrastructure",
  "deploymentGuid": "00000001",  // SAME
  "templateFilePath": "app.bicep"  // Modified template
}
// Pipeline skips → No deployment

// Ready to deploy changes
{
  "name": "app-infrastructure",
  "deploymentGuid": "00000002",  // CHANGED
  "templateFilePath": "app.bicep"
}
// Pipeline deploys → State: { "app-infrastructure": "00000002" }
```

### Monitoring and Troubleshooting

**Pipeline Logs**: Available in your CI/CD platform

- Detailed deployment execution logs
- GUID comparison results
- Pre/post script output
- Deployment success/failure status

**Common Issues**:

| Issue | Cause | Solution |
|-------|-------|----------|
| Deployment skipped unexpectedly | `deploymentGuid` not changed | Generate new 8-char hex GUID |
| Template validation failed | Invalid Bicep/ARM syntax | Run `az bicep build` or ARM validation locally |
| Pre-script failed | Script error or missing dependencies | Test script locally, check permissions |
| RBAC assignment failed | Invalid email/group name or insufficient pipeline permissions | Verify Azure AD objects exist, check pipeline SPN permissions |
| Resource group not found | `resourceGroupName` doesn't exist | Create RG first or use template that creates it |

**Generating deploymentGuid Values**:

```bash
# Random 8-char hex
openssl rand -hex 4

# Timestamp-based (YYYYMMDD in hex)
echo "obase=16; $(date +%Y%m%d)" | bc | tr '[:upper:]' '[:lower:]'

# Sequential
# Manually increment: 00000001 → 00000002 → 00000003...
```

```powershell
# Random 8-char hex
-join ((0..7) | ForEach-Object { '{0:x}' -f (Get-Random -Max 16) })

# Timestamp-based
[Convert]::ToString((Get-Date -Format 'yyyyMMdd'), 16).ToLower()

#Using GUID
((New-Guid) -replace '-').SubString(0, 8)
```

## Summary

CloudGrip provides enterprise-grade Azure infrastructure management through:

- **Hierarchical Configuration**: Tenant → Management Group → Subscription structure
- **Schema Validation**: JSON schemas ensure configuration correctness
- **Multi-Technology Support**: Deploy using Bicep, ARM, or Terraform
- **Deployment Versioning**: `deploymentGuid` provides explicit deployment control
- **RBAC Integration**: Manage access at all hierarchy levels
- **Pipeline Automation**: Git-driven deployments with change detection

### Key Takeaways

1. **deploymentGuid is Critical**: Always change it when you want to deploy template changes
2. **Schema Compliance**: Validate configurations against CloudGrip schemas
3. **Hierarchy Matters**: Understand RBAC and policy inheritance
4. **Use Appropriate Scope**: Deploy resources at subscription level via deployments array
5. **Version Control Everything**: All configuration in Git for audit trail
6. **Test First**: Use non-production environments before production changes

### Configuration File Locations

**Remember**: When you want infrastructure changes deployed, always update the `deploymentGuid` in your deployment configuration. This simple 8-character hex value is what triggers the pipeline to execute your deployment.

For detailed schema references, see:

- Tenant schema: `http://schema.ca8.io/schemas/2024-01-01/tenant.json`
- Management Group schema: `http://schema.ca8.io/schemas/2024-01-01/managementgroup.json`
- Subscription schema: `http://schema.ca8.io/schemas/2024-01-01/subscription.json`
