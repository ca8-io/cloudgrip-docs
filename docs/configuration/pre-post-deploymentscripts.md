# Pre and Post Deployment Scripts

CloudGrip supports PowerShell scripts that run before and after deployment execution, providing hooks for custom logic, validation, and post-deployment configuration.

## Table of Contents

- [Overview](#overview)
- [Script Configuration](#script-configuration)
- [Script Parameters](#script-parameters)
- [Parameter Objects Reference](#parameter-objects-reference)
- [Pre-Deployment Scripts](#pre-deployment-scripts)
- [Post-Deployment Scripts](#post-deployment-scripts)
- [Best Practices](#best-practices)
- [Common Use Cases](#common-use-cases)
- [Examples](#examples)

## Overview

Pre and post deployment scripts are **PowerShell scripts** that execute as part of the CloudGrip deployment pipeline:

- **Pre-Deployment Scripts** (`preScriptPath`): Run **before** the template deployment
- **Post-Deployment Scripts** (`postScriptPath`): Run **after** successful template deployment

These scripts receive structured objects from the pipeline containing deployment configuration, subscription details, and file paths.

## Script Configuration

Scripts are configured in the deployment object within your subscription configuration:

```json
{
  "deployments": [
    {
      "name": "storage-infrastructure",
      "type": "bicep",
      "templateFilePath": "storage/main.bicep",
      "templateParameterFilePath": "storage/main.prod.bicepparam",
      "deploymentGuid": "a1b2c3d4",
      "resourceGroupName": "rg-storage-prod",
      "preScriptPath": "storage/pre.ps1",
      "postScriptPath": "storage/post.ps1"
    }
  ]
}
```

**Path Resolution**:

- Paths are relative to the deployment's folder location
- Can be absolute paths or relative to repository root
- Scripts must be PowerShell (`.ps1`) files

## Script Parameters

All CloudGrip deployment scripts receive **four mandatory parameters** from the pipeline:

```powershell
param (
  [Parameter(Mandatory)]
  [pscustomobject] $Deployment,

  [Parameter(Mandatory)]
  [pscustomobject] $Subscription,

  [Parameter(Mandatory)]
  [pscustomobject] $TemplateFile,

  [Parameter(Mandatory)]
  [pscustomobject] $TemplateParameterFile,

  [Parameter(ValueFromRemainingArguments)]
  $RemainingArgs
)
```

### Parameter Overview

| Parameter | Type | Description | Source |
|-----------|------|-------------|--------|
| `$Deployment` | PSCustomObject | The deployment configuration object | Parsed from subscription JSON |
| `$Subscription` | PSCustomObject | The subscription configuration object | Parsed from subscription JSON |
| `$TemplateFile` | String | Path to the template file | Deployment configuration |
| `$TemplateParameterFile` | String | Path to the parameter file | Deployment configuration |
| `$RemainingArgs` | Array | Catches any additional arguments | Pipeline extensibility |

## Parameter Objects Reference

### `$Deployment` Parameter

The `$Deployment` object contains the complete deployment configuration from your subscription JSON file.

**Object Structure**:

```powershell
$Deployment = @{
    name                      = "storage-infrastructure"
    type                      = "bicep"
    templateFilePath          = "storage/main.bicep"
    templateParameterFilePath = "storage/main.prod.bicepparam"
    deploymentGuid            = "a1b2c3d4"
    resourceGroupName         = "rg-storage-prod"
    preScriptPath             = "storage/pre.ps1"
    postScriptPath            = "storage/post.ps1"
}
```

**Available Properties**:

| Property | Type | Description | Always Present |
|----------|------|-------------|----------------|
| `name` | String | Deployment name | ✅ Yes |
| `type` | String | Deployment type (`bicep`, `arm`, `terraform`) | ✅ Yes |
| `deploymentGuid` | String | 8-character deployment version GUID | ✅ Yes |
| `templateFilePath` | String | Path to template file | ✅ (except terraform) |
| `templateParameterFilePath` | String | Path to parameter file | ⚠️ Optional |
| `resourceGroupName` | String | Target resource group | ⚠️ Optional |
| `preScriptPath` | String | Path to pre-deployment script | ⚠️ Optional |
| `postScriptPath` | String | Path to post-deployment script | ⚠️ Optional |

**Usage Examples**:

```powershell
# Access deployment name
Write-Information "Deploying: $($Deployment.name)"

# Check deployment type
if ($Deployment.type -eq "bicep") {
    # Bicep-specific logic
}

# Get resource group
$resourceGroup = $Deployment.resourceGroupName

# Access deployment GUID for logging
Write-Information "Deployment version: $($Deployment.deploymentGuid)"
```

**In Post-Deployment Scripts**, the `$Deployment` object also includes **deployment outputs**:

```powershell
# Access deployment outputs (post-script only)
$Deployment.Outputs.outStorageAccountName.value
$Deployment.Outputs.outStorageAccountId.value
$Deployment.Outputs.outPrimaryEndpoints.value
```

### `$Subscription` Parameter

The `$Subscription` object contains the complete subscription configuration.

**Object Structure**:

```powershell
$Subscription = @{
    '$schema'                = "http://schema.ca8.io/schemas/2024-01-01/subscription.json"
    tenantId                 = "12345678-1234-1234-1234-123456789012"
    parentManagementGroupId  = "mg-production"
    subscriptionName         = "prod-workloads"
    subscriptionId           = "87654321-4321-4321-4321-210987654321"
    location                 = "westeurope"
    centralConfig            = @{
        vnetResourceId = "/subscriptions/.../virtualNetworks/vnet-prod"
        lawsResourceId = "/subscriptions/.../workspaces/laws-prod"
    }
    deployments              = @(...)  # Array of all deployments
    tags                     = @{
        environment = "production"
        costCenter  = "IT-001"
    }
    rbac                     = @(...)  # Array of RBAC assignments
    marketPlaceFeatures      = @()
}
```

**Available Properties**:

| Property | Type | Description |
|----------|------|-------------|
| `$schema` | String | Schema URI for validation |
| `tenantId` | String (GUID) | Azure AD tenant ID |
| `parentManagementGroupId` | String | Parent management group ID |
| `subscriptionName` | String | Subscription display name |
| `subscriptionId` | String (GUID) | Azure subscription ID |
| `location` | String | Default Azure region |
| `centralConfig` | Object | Shared infrastructure references |
| `deployments` | Array | All deployment configurations |
| `tags` | Object | Subscription-level tags |
| `rbac` | Array | RBAC assignments |
| `marketPlaceFeatures` | Array | Marketplace features |

**Usage Examples**:

```powershell
# Get subscription details
$subscriptionId = $Subscription.subscriptionId
$location = $Subscription.location

Write-Information "Deploying to subscription: $($Subscription.subscriptionName)"
Write-Information "Target region: $location"

# Access central configuration
$vnetId = $Subscription.centralConfig.vnetResourceId
$lawsId = $Subscription.centralConfig.lawsResourceId

# Use subscription tags
$environment = $Subscription.tags.environment
if ($environment -eq "production") {
    # Production-specific logic
}

# Find other deployments
$otherDeployments = $Subscription.deployments | Where-Object { $_.name -ne $Deployment.name }
```

### `$TemplateFile` Parameter

A **string** containing the absolute or relative path to the template file.

**Value Examples**:

```powershell
# Bicep template
$TemplateFile = "storage/main.bicep"

# ARM template
$TemplateFile = "network/vnet.json"

# Absolute path
$TemplateFile = "/Users/mark-s/Repos/cloudautom8/cloudgrip/config/Azure/storage/main.bicep"
```

**Usage**:

```powershell
# Check if file exists
if (Test-Path $TemplateFile) {
    Write-Information "Template found: $TemplateFile"
}

# Get file details
$templateInfo = Get-Item $TemplateFile
Write-Information "Template size: $($templateInfo.Length) bytes"

# Read template content (for validation)
$templateContent = Get-Content $TemplateFile -Raw
```

### `$TemplateParameterFile` Parameter

A **string** containing the path to the parameter file (Bicep `.bicepparam` or ARM `.parameters.json`).

**Value Examples**:

```powershell
# Bicep parameter file
$TemplateParameterFile = "storage/main.prod.bicepparam"

# ARM parameter file
$TemplateParameterFile = "network/vnet.parameters.json"
```

**Usage**:

```powershell
# Check parameter file type
if ($TemplateParameterFile -like "*.bicepparam") {
    Write-Information "Using Bicep parameter file"
} elseif ($TemplateParameterFile -like "*.parameters.json") {
    Write-Information "Using ARM parameter file"
}

# Read parameters (see examples below)
```

## Pre-Deployment Scripts

Pre-deployment scripts run **before** the template deployment executes.

### Purpose

- **Validation**: Check prerequisites and environment readiness
- **Preparation**: Set up resources or configuration needed by the deployment
- **Parameter Extraction**: Read and validate template parameters
- **Resource Checks**: Verify required resources exist
- **Secret Retrieval**: Get secrets from Key Vault to pass to deployment

### Script Template

```powershell
# filepath: pre.ps1
<#
  .SYNOPSIS
  Pre-deployment script for CloudGrip deployment

  .DESCRIPTION
  Runs before template deployment to validate and prepare environment

  .PARAMETER Deployment
  The deployment configuration object from subscription JSON

  .PARAMETER Subscription
  The subscription configuration object from subscription JSON

  .PARAMETER TemplateFile
  Path to the ARM/Bicep template file

  .PARAMETER TemplateParameterFile
  Path to the parameter file

  .PARAMETER RemainingArgs
  Additional arguments from pipeline
#>

param (
  [Parameter(Mandatory)]
  [pscustomobject] $Deployment,

  [Parameter(Mandatory)]
  [pscustomobject] $Subscription,

  [Parameter(Mandatory)]
  [string] $TemplateFile,

  [Parameter(Mandatory)]
  [string] $TemplateParameterFile,

  [Parameter(ValueFromRemainingArguments)]
  $RemainingArgs
)

begin {
  Write-Information -MessageData "Started pre-deployment script for: $($Deployment.name)"
}

process {
  try {
    # Your pre-deployment logic here
    
    # Return data to post-deployment script if needed
    return @{
      PreDeploymentTimestamp = Get-Date
      ValidatedParameters = $true
    }
  }
  catch {
    Write-Error -Message "Pre-deployment failed: $_" -ErrorAction Stop
  }
}

end {
  Write-Information -MessageData "Finished pre-deployment script"
}
```

### Return Values

Pre-deployment scripts can **return data** that will be available to post-deployment scripts:

```powershell
# In pre-deployment script
return @{
    resourcesCreated = @("resource1", "resource2")
    validationPassed = $true
    timestamp = Get-Date
}

# This data becomes available in post-deployment context
```

## Post-Deployment Scripts

Post-deployment scripts run **after** successful template deployment.

### Purpose

- **Configuration**: Apply settings not possible in templates
- **Integration**: Connect deployed resources to external systems
- **Validation**: Verify deployment success and resource health
- **Notification**: Send deployment completion alerts
- **Cleanup**: Remove temporary resources or configurations
- **Documentation**: Extract and store deployment outputs

### Script Template

```powershell
# filepath: post.ps1
<#
  .SYNOPSIS
  Post-deployment script for CloudGrip deployment

  .DESCRIPTION
  Runs after successful template deployment for configuration and validation

  .PARAMETER Deployment
  The deployment configuration object including deployment outputs

  .PARAMETER Subscription
  The subscription configuration object

  .PARAMETER TemplateFile
  Path to the deployed template file

  .PARAMETER TemplateParameterFile
  Path to the parameter file used

  .PARAMETER RemainingArgs
  Additional arguments from pipeline
#>

param (
  [Parameter(Mandatory)]
  [pscustomobject] $Deployment,

  [Parameter(Mandatory)]
  [pscustomobject] $Subscription,

  [Parameter(Mandatory)]
  [string] $TemplateFile,

  [Parameter(Mandatory)]
  [string] $TemplateParameterFile,

  [Parameter(ValueFromRemainingArguments)]
  $RemainingArgs
)

begin {
  Write-Information -MessageData "Started post-deployment script for: $($Deployment.name)"
}

process {
  try {
    # Access deployment outputs
    $storageAccountName = $Deployment.Outputs.outStorageAccountName.value
    
    Write-Information "Deployed storage account: $storageAccountName"
    
    # Your post-deployment logic here
  }
  catch {
    Write-Error -Message "Post-deployment failed: $_" -ErrorAction Stop
  }
}

end {
  Write-Information -MessageData "Finished post-deployment script"
}
```

### Accessing Deployment Outputs

In post-deployment scripts, the `$Deployment.Outputs` property contains all template outputs:

```powershell
# Access outputs defined in Bicep/ARM template
$storageAccountName = $Deployment.Outputs.outStorageAccountName.value
$storageAccountId = $Deployment.Outputs.outStorageAccountId.value
$primaryEndpoint = $Deployment.Outputs.outPrimaryBlobEndpoint.value

# Check if output exists
if ($Deployment.Outputs.PSObject.Properties.Name -contains "outStorageAccountName") {
    $name = $Deployment.Outputs.outStorageAccountName.value
}

# Iterate all outputs
foreach ($output in $Deployment.Outputs.PSObject.Properties) {
    Write-Information "$($output.Name): $($output.Value.value)"
}
```

## Best Practices

### Script Structure

1. **Use Proper Parameter Blocks**

```powershell
# Always use typed parameters with [Parameter(Mandatory)]
param (
  [Parameter(Mandatory)]
  [pscustomobject] $Deployment,
  # ...
)
```

2. **Include Begin/Process/End Blocks**

```powershell
begin {
  # Initialization
}
process {
  # Main logic wrapped in try/catch
}
end {
  # Cleanup
}
```

3. **Implement Error Handling**

```powershell
try {
  # Your logic
}
catch {
  Write-Error -Message "Error: $_" -ErrorAction Stop
}
```

### Logging

Use `Write-Information` for logging (captured by pipeline):

```powershell
Write-Information -MessageData "Processing deployment: $($Deployment.name)" -InformationAction Continue

# Log object details
Write-Information -MessageData (ConvertTo-Json $Deployment -Depth 10)
```

### Parameter Validation

```powershell
# Validate required properties exist
if (-not $Deployment.resourceGroupName) {
    Write-Error "Resource group name is required" -ErrorAction Stop
}

# Validate subscription context
if ($Subscription.subscriptionId -ne (Get-AzContext).Subscription.Id) {
    Write-Error "Wrong subscription context" -ErrorAction Stop
}
```

### Idempotency

Make scripts idempotent (safe to run multiple times):

```powershell
# Check if resource exists before creating
$resource = Get-AzResource -Name $name -ResourceGroupName $rg -ErrorAction SilentlyContinue
if (-not $resource) {
    # Create resource
}
```

## Common Use Cases

### 1. Reading Bicep Parameters

Extract parameters from Bicep parameter file in pre-deployment:

```powershell
# Build Bicep parameters to JSON
bicep build-params $TemplateParameterFile --outfile ./temp-params.json

# Read parameters
$paramsObject = Get-Content ./temp-params.json -Raw | ConvertFrom-Json
$storageAccountName = $paramsObject.parameters.parStorageAccountName.value

Write-Information "Storage account name: $storageAccountName"

# Cleanup temp file
Remove-Item ./temp-params.json -Force
```

### 2. Reading ARM Parameters

```powershell
# Read ARM parameter file
$paramsObject = Get-Content $TemplateParameterFile -Raw | ConvertFrom-Json
$parameters = $paramsObject.parameters

# Access specific parameter
$sku = $parameters.storageAccountSku.value
```

### 3. Validating Resource Group Exists

```powershell
# Pre-deployment: Ensure resource group exists
$rgName = $Deployment.resourceGroupName
$rg = Get-AzResourceGroup -Name $rgName -ErrorAction SilentlyContinue
    
if (-not $rg) {
    Write-Information "Creating resource group: $rgName" -InformationAction Continue
    New-AzResourceGroup -Name $rgName -Location $Subscription.location -Tag $Subscription.tags
}
```

### 4. Retrieving Secrets from Key Vault

```powershell
# Get Key Vault reference from central config
$keyVaultId = $Subscription.centralConfig.keyVaultResourceId

# Parse Key Vault name from resource ID
$kvName = ($keyVaultId -split '/')[-1]

# Retrieve secret
$secret = Get-AzKeyVaultSecret -VaultName $kvName -Name "DatabasePassword" -AsPlainText

# Use secret (ensure not logged)
# Pass to deployment or use in post-configuration
```

### 5. Configuring Deployed Resources

```powershell
# Post-deployment: Configure storage account settings
$storageAccountName = $Deployment.Outputs.outStorageAccountName.value
$rgName = $Deployment.resourceGroupName

# Enable blob versioning (not available in ARM/Bicep at time of writing)
$storageAccount = Get-AzStorageAccount -ResourceGroupName $rgName -Name $storageAccountName
Enable-AzStorageBlobDeleteRetentionPolicy -ResourceGroupName $rgName `
    -StorageAccountName $storageAccountName `
    -RetentionDays 30

Write-Information "Configured retention policy for: $storageAccountName"
```

### 6. Sending Notifications

```powershell
# Post-deployment: Send Teams notification
$webhook = $env:TEAMS_WEBHOOK_URL

$message = @{
    text = "Deployment completed: $($Deployment.name) in $($Subscription.subscriptionName)"
} | ConvertTo-Json

Invoke-RestMethod -Uri $webhook -Method Post -Body $message -ContentType 'application/json'
```

### 7. Creating Custom Monitoring

```powershell
# Post-deployment: Create custom metric alert
$storageAccountId = $Deployment.Outputs.outStorageAccountId.value

$actionGroup = Get-AzActionGroup -ResourceGroupName "rg-monitoring" -Name "ag-alerts"

$criteria = New-AzMetricAlertRuleV2Criteria `
    -MetricName "UsedCapacity" `
    -TimeAggregation Average `
    -Operator GreaterThan `
    -Threshold 1000000000

Add-AzMetricAlertRuleV2 `
    -Name "alert-storage-capacity" `
    -ResourceGroupName $Deployment.resourceGroupName `
    -TargetResourceId $storageAccountId `
    -Condition $criteria `
    -ActionGroupId $actionGroup.Id `
    -Severity 3
```

## Examples

### Complete Pre-Deployment Script Example

```powershell
# filepath: storage/pre.ps1
param (
  [Parameter(Mandatory)]
  [pscustomobject] $Deployment,

  [Parameter(Mandatory)]
  [pscustomobject] $Subscription,

  [Parameter(Mandatory)]
  [string] $TemplateFile,

  [Parameter(Mandatory)]
  [string] $TemplateParameterFile,

  [Parameter(ValueFromRemainingArguments)]
  $RemainingArgs
)

begin {
  Write-Information -MessageData "=== Pre-Deployment: $($Deployment.name) ===" -InformationAction Continue
}

process {
  try {
    #region Validate Context
    $currentContext = Get-AzContext
    if ($currentContext.Subscription.Id -ne $Subscription.subscriptionId) {
        Write-Error "Wrong subscription context. Expected: $($Subscription.subscriptionId)" -ErrorAction Stop
    }
    #endregion

    #region Extract Parameters
    Write-Information "Extracting parameters from: $TemplateParameterFile"
    
    bicep build-params $TemplateParameterFile --outfile ./temp-params.json
    $paramsObject = Get-Content ./temp-params.json -Raw | ConvertFrom-Json
    $parameters = $paramsObject.parameters
    
    $storageAccountName = $parameters.parStorageAccountName.value
    Write-Information "Storage Account Name: $storageAccountName" -InformationAction Continue
    
    Remove-Item ./temp-params.json -Force
    #endregion

    #region Validate Resource Group
    $rgName = $Deployment.resourceGroupName
    $rg = Get-AzResourceGroup -Name $rgName -ErrorAction SilentlyContinue
    
    if (-not $rg) {
        Write-Information "Creating resource group: $rgName" -InformationAction Continue
        New-AzResourceGroup -Name $rgName -Location $Subscription.location -Tag $Subscription.tags
    }
    #endregion

    #region Check Storage Account Name Availability
    $nameAvailable = Get-AzStorageAccountNameAvailability -Name $storageAccountName
    if (-not $nameAvailable.NameAvailable) {
        Write-Error "Storage account name '$storageAccountName' is not available: $($nameAvailable.Reason)" -ErrorAction Stop
    }
    #endregion

    # Return validation results
    return @{
        ValidationPassed = $true
        StorageAccountName = $storageAccountName
        ResourceGroupCreated = ($null -eq $rg)
        Timestamp = Get-Date
    }
  }
  catch {
    Write-Error -Message "Pre-deployment validation failed: $_" -ErrorAction Stop
  }
}

end {
  Write-Information -MessageData "=== Pre-Deployment Complete ===" -InformationAction Continue
}
```

### Complete Post-Deployment Script Example

```powershell
# filepath: storage/post.ps1
param (
  [Parameter(Mandatory)]
  [pscustomobject] $Deployment,

  [Parameter(Mandatory)]
  [pscustomobject] $Subscription,

  [Parameter(Mandatory)]
  [string] $TemplateFile,

  [Parameter(Mandatory)]
  [string] $TemplateParameterFile,

  [Parameter(ValueFromRemainingArguments)]
  $RemainingArgs
)

begin {
  Write-Information -MessageData "=== Post-Deployment: $($Deployment.name) ===" -InformationAction Continue
}

process {
  try {
    #region Extract Deployment Outputs
    if (-not $Deployment.Outputs) {
        Write-Error "No deployment outputs available" -ErrorAction Stop
    }

    $storageAccountName = $Deployment.Outputs.outStorageAccountName.value
    $storageAccountId = $Deployment.Outputs.outStorageAccountId.value
    $primaryEndpoint = $Deployment.Outputs.outPrimaryBlobEndpoint.value

    Write-Information "Deployed Storage Account: $storageAccountName" -InformationAction Continue
    Write-Information "Resource ID: $storageAccountId" -InformationAction Continue
    Write-Information "Primary Endpoint: $primaryEndpoint" -InformationAction Continue
    #endregion

    #region Configure Advanced Settings
    Write-Information "Configuring blob retention policy..." -InformationAction Continue
    
    $storageAccount = Get-AzStorageAccount `
        -ResourceGroupName $Deployment.resourceGroupName `
        -Name $storageAccountName

    # Enable soft delete for blobs
    Enable-AzStorageBlobDeleteRetentionPolicy `
        -ResourceGroupName $Deployment.resourceGroupName `
        -StorageAccountName $storageAccountName `
        -RetentionDays 7

    Write-Information "Retention policy configured" -InformationAction Continue
    #endregion

    #region Create Monitoring Alert
    if ($Subscription.centralConfig.PSObject.Properties.Name -contains 'actionGroupId') {
        Write-Information "Creating capacity alert..." -InformationAction Continue
        
        $criteria = New-AzMetricAlertRuleV2Criteria `
            -MetricName "UsedCapacity" `
            -TimeAggregation Total `
            -Operator GreaterThan `
            -Threshold 900000000000  # 900 GB

        Add-AzMetricAlertRuleV2 `
            -Name "alert-$storageAccountName-capacity" `
            -ResourceGroupName $Deployment.resourceGroupName `
            -TargetResourceId $storageAccountId `
            -Condition $criteria `
            -ActionGroupId $Subscription.centralConfig.actionGroupId `
            -Severity 2 `
            -WindowSize (New-TimeSpan -Hours 1) `
            -Frequency (New-TimeSpan -Minutes 15)

        Write-Information "Alert created successfully" -InformationAction Continue
    }
    #endregion

    #region Send Completion Notification
    $message = @"
Deployment Completed Successfully

Deployment: $($Deployment.name)
Subscription: $($Subscription.subscriptionName)
Resource Group: $($Deployment.resourceGroupName)
Storage Account: $storageAccountName
Deployment GUID: $($Deployment.deploymentGuid)
Timestamp: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')
"@

    Write-Information $message -InformationAction Continue
    #endregion
  }
  catch {
    Write-Error -Message "Post-deployment configuration failed: $_" -ErrorAction Stop
  }
}

end {
  Write-Information -MessageData "=== Post-Deployment Complete ===" -InformationAction Continue
}
```

## Summary

CloudGrip pre and post deployment scripts provide powerful extension points in the deployment pipeline:

- **Four Standard Parameters**: `$Deployment`, `$Subscription`, `$TemplateFile`, `$TemplateParameterFile`
- **Pipeline-Provided**: All parameters automatically passed by CloudGrip pipeline
- **Full Configuration Access**: Complete deployment and subscription configuration available
- **Deployment Outputs**: Post-scripts can access all template outputs
- **Flexible Logic**: Implement validation, configuration, integration, and monitoring
- **Error Handling**: Use PowerShell best practices with try/catch blocks
- **Logging**: Use `Write-Information` for pipeline-captured logs

These scripts enable complex deployment scenarios while maintaining the declarative nature of CloudGrip's infrastructure-as-code approach.
