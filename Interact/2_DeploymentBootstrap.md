# Interact Deployment Bootstrap

## Goal

Document the first deployment step for the Interact sample application.

This step does **not** deliver the final deployment model. It creates the first standalone deployment baseline so the app can later be connected to:
- Front Door
- Bobst Connect auth
- runtime environment configuration
- APIM-facing backend URLs

At this stage, the deployment scope is limited to:
- Azure resource group deployment
- storage account creation
- `$web` container creation
- static website enablement
- artifact upload for the Angular build output

## Phone Summary

```text
Goal:
host the Interact Angular build as static files in a dedicated storage account.

Main files:
deployment/deploy.ps1
deployment/bicep/azuredeploy.bicep
deployment/bicep/deploy-storage-account.bicep
deployment/bicep/deploy-storage-account-container.bicep
deployment/bicep/deploy-output.bicep

Not covered yet in this step:
real auth wiring, APIM config, monitoring.
```

## Why this step exists

Before wiring `/interact` into Bobst Connect, we need a minimal standalone hosting model for the frontend.

This is the same overall direction as Andon:
- standalone frontend deployment
- its own resource group
- static artifact hosting

But it is intentionally smaller than both:
- Andon deployment
- FrontendService deployment

Reason:
- Interact should behave like a Bobst Connect frontend app
- but its repository and first implementation should stay lightweight

## Mental Model

Think of this deployment step as **two layers**:

1. **Infrastructure layer**
   - Bicep creates the Azure resources needed to host static frontend files.

2. **Artifact publication layer**
   - `deploy.ps1` takes the Angular build output and uploads it to the storage account `$web` container.

This means the Angular app itself is still just static files.
There is no app server in this baseline.

## References Used

### FrontendService
Used as reference for the general deployment folder shape:
- `deployment/`
- `deployment/bicep/`
- `deploy.ps1`
- environment parameter files

Reference location:
- `C:\BobstSource\FrontendService\deployment`

### Andon
Used as reference for the standalone static-host deployment model:
- storage account hosting
- static website enablement
- simple Bicep modules for storage resources

Reference location:
- `C:\BobstSource\Andon\deployment`

## Files Added

### Deployment root
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\README.md`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\.gitignore`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\deploy.ps1`

### Parameter files
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-development.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-feature.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-staging.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-production.json`

### Bicep templates
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\azuredeploy.bicep`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\deploy-storage-account.bicep`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\deploy-storage-account-container.bicep`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\deploy-output.bicep`

## Design Choice

The deployment baseline is:
- **frontend-like in folder organization**
- **Andon-like in standalone hosting**
- **minimal in resource scope**

That means:
- we keep `deployment/` local to the app repo
- we avoid the full FrontendService deployment complexity
- we avoid adding Front Door and B2C registration too early

## What Each File Does

### `deploy.ps1`

This is the local deployment entry point.

Current responsibilities:
1. Read the chosen parameter file.
2. Inject `deploymentPrefix`.
3. Create the target resource group.
4. Run the Bicep deployment.
5. Enable static website hosting on the storage account.
6. Upload Angular build artifacts to `$web`.
7. Expose the deployed storage URL as a pipeline output variable.

### Why `index.html` is uploaded last
For SPA hosting, `index.html` is the entry point.
Uploading it last reduces the chance that users briefly load a new `index.html` that references JS/CSS files not uploaded yet.

### Reference snippets
Script parameters:

```powershell
param(
    [Parameter(Mandatory = $true)]
    [string]
    $DeploymentPrefix,

    [Parameter(Mandatory = $false)]
    [string]
    $ArtifactsFolder = "$PSScriptRoot\..\dist\interact\browser",

    [Parameter(Mandatory = $false)]
    [string]
    $ParametersFilePath = "parameters-feature.json"
)
```

Upload all generated files to `$web`, but push `index.html` last:

```powershell
foreach ($file in $files | Where-Object { $_.Name -ne 'index.html' }) {
    $relativePath = $file.FullName.Substring($artifactRoot.Length).TrimStart('\').TrimStart('/') -replace '\\', '/'
    Set-AzStorageBlobContent -File $file.FullName -Container $webContainerName -Blob $relativePath -Context $storageContext -Force | Out-Null
}

$indexHtmlFile = $files | Where-Object { $_.Name -eq 'index.html' } | Select-Object -First 1

if ($null -ne $indexHtmlFile) {
    Set-AzStorageBlobContent -File $indexHtmlFile.FullName -Container $webContainerName -Blob 'index.html' -Context $storageContext -Force | Out-Null
}
```

Important defaults:
- `ArtifactsFolder` defaults to `..\dist\interact\browser`
- `ParametersFilePath` defaults to `parameters-feature.json`
- `ResourceGroupLocation` defaults to `West Europe`

### `azuredeploy.bicep`

This is the main deployment template.

Current responsibilities:
- define naming for the Interact storage resources
- deploy the storage account
- deploy the `$web` container
- return output values needed by the deploy script

It currently does **not** deploy:
- Front Door route
- monitoring
- App Insights
- auth-related resources

### How it is wired
`azuredeploy.bicep` calls:
- `deploy-storage-account.bicep` to create the storage account
- `deploy-storage-account-container.bicep` to create the `$web` container
- `deploy-output.bicep` to normalize deployment outputs

### Reference snippet

```bicep
var names = {
  service: '${appName}-service'
  storageDeployment: '${deploymentPrefix}-${appName}-storage'
  storageAccountName: take(toLower('itr${uniqueId}'), 24)
  webContainerDeployment: '${deploymentPrefix}-${appName}-web-container'
  webContainerName: '$web'
}

module interactStorage 'deploy-storage-account.bicep' = {
  name: names.storageDeployment
  params: {
    location: location
    storageAccountsName: names.storageAccountName
    sensitiveResourcesLock: sensitiveResourcesLock
  }
}
```

### `deploy-storage-account.bicep`

Creates the storage account used to host the static frontend files.

Important characteristics:
- `StorageV2`
- HTTPS only
- blob/file encryption enabled
- optional delete lock through `sensitiveResourcesLock`

This module was intentionally renamed from the Andon/FrontendService-style `gen2` name because Interact does not currently use Data Lake Gen2 capability.

### Why not keep the `gen2` name
Gen2 refers to Data Lake Storage hierarchical namespace.
For this frontend hosting use case we only need standard static website hosting, so the `gen2` wording was misleading.

### `deploy-storage-account-container.bicep`

Creates the blob container used for static hosting.

Current use:
- create `$web`

### `deploy-output.bicep`

Collects and returns deployment outputs in a clean shape.

Current outputs:
- app name
- environment name
- artifact host URL
- storage account name
- web endpoint host name

### Parameter files

These define the minimal environment-specific values.

Current parameters:
- `appName`
- `environmentName`
- `sensitiveResourcesLock`

### Why multiple parameter files exist
The goal is to keep one shared Bicep template, and switch environment-specific values through JSON files instead of editing the template per environment.

### Reference snippet

```json
{
  "parameters": {
    "appName": { "value": "interact" },
    "environmentName": { "value": "feature" },
    "sensitiveResourcesLock": { "value": false }
  }
}
```

## Script Flow

The current `deploy.ps1` flow is:

1. Validate input parameters.
2. Read the selected JSON parameter file.
3. Resolve deployment values such as:
   - deployment prefix
   - resource group name
   - template path
4. Check Azure login context with `Get-AzContext`.
5. Create the resource group if needed.
6. If `ValidationOnly` is enabled:
   - run `Test-AzResourceGroupDeployment`
   - stop there
7. Otherwise:
   - run `New-AzResourceGroupDeployment`
   - collect outputs
8. Enable static website hosting on the deployed storage account.
9. Upload all build artifacts to `$web`.
10. Upload `index.html` last.
11. Emit the final storage-hosted URL.

## Read This Before Running The Script

### Prerequisites
- Azure PowerShell modules must be installed.
- You must already be logged into the correct Azure subscription.
- The Angular build output must exist in the folder expected by `ArtifactsFolder`.

### Important local limitation
In the current Codex/Windows environment, Angular `build` is blocked by an `esbuild spawn EPERM` issue.
So actual deployment validation should be done in a normal developer terminal or in Azure DevOps CI.

## Why storage static hosting first

This is enough to prove:
- Interact can be deployed independently
- the Angular output can be hosted as static artifacts
- the app can have its own deployment lifecycle

It also keeps the deployment step small enough to review before introducing:
- Front Door coupling
- auth redirect concerns
- APIM/runtime config concerns

## What is intentionally out of scope

This step does **not** yet handle:
- `/interact` route creation in Front Door
- custom domain binding
- Bobst Connect redirect URI registration
- runtime config file replacement
- API endpoint substitution
- Application Insights
- availability tests
- pipeline deployment stages
- environment-specific auth settings

These will be added in later steps.

## What To Check Next When Continuing

Before moving to auth implementation, verify these deployment questions:
- Who owns the `/interact` Front Door route long term: Interact repo or MainInterfaceService?
- Which Front Door profile/endpoint/custom domain values will be passed into Interact deployment?
- Which runtime config values should come from MainInterface outputs vs Interact parameter files?
- Do we need App Insights now, or can it stay in a later hardening step?

## Expected follow-up steps

### Next deployment step
Add Front Door integration so the app becomes reachable through:
- `https://<prefix>.connect.bobst.com/interact`

That future step will likely require:
- Front Door route Bicep
- dependency on Main Interface outputs
- path patterns:
  - `/interact`
  - `/interact/*`

### After Front Door
Add runtime configuration and auth-aware deployment:
- environment config file generation or replacement
- B2C redirect URI alignment
- API base URL wiring for PMS and User Management

## Validation status

What was validated:
- file structure created successfully
- deployment scaffold is present in the sample repo

What was **not** executed here:
- actual Azure deployment

Reason:
- no Azure session/context was available in the current environment

## Command Example

Illustrative usage of the current deployment script:

```powershell
.\deploy.ps1 `
  -DeploymentPrefix "my-prefix" `
  -ParametersFilePath "parameters-feature.json"
```

Validation-only mode:

```powershell
.\deploy.ps1 `
  -DeploymentPrefix "my-prefix" `
  -ParametersFilePath "parameters-feature.json" `
  -ValidationOnly $true
```

## Recommendation

Keep this deployment layer minimal until the Front Door step.

Do not add:
- auth resources
- APIM substitutions
- monitoring extras
- complex release logic

before the route and runtime config decisions are finalized.

## Quick Troubleshooting Notes

### If static site URL returns 404
- Check that `$web` exists.
- Check that `index.html` was uploaded.
- Check that static website hosting is enabled on the storage account.

### If deployment succeeds but route still points nowhere
- That is expected at this stage if Front Door is not enabled yet.
- Use the storage website endpoint from deployment outputs.

### If the script fails at `Get-AzContext`
- You are not logged in to Azure from that shell session.

### Reference snippet

```powershell
$currentAzureContext = Get-AzContext
if (-not $currentAzureContext) {
    throw "Please connect to Azure with Connect-AzAccount before executing this script."
}
```
