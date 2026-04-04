# Interact HowTo Index

This folder is the **phone-readable technical notebook** for the Interact bootstrap.

## Recommended Reading Order

1. `1_BootstrapStepsOverview.md`
   - global summary of everything done so far
   - what each step created
   - what is still missing

2. `2_DeploymentBootstrap.md`
   - how the standalone static hosting deployment is structured
   - what each Bicep module does
   - how `deploy.ps1` works

3. `3_RuntimeConfigAndFrontDoorBootstrap.md`
   - how runtime config is loaded by Angular
   - how deployment generates `config.runtime.json`
   - how the `/interact` Front Door route is prepared

## One-Line Architecture Summary

```text
Browser -> Front Door /interact -> Interact Angular SPA -> Azure AD B2C -> APIM -> UserManagementService + PMS
```

## Current Implementation State

### Done
- repo bootstrap files
- minimal Angular workspace
- lint / Sonar / CI skeleton
- storage-based Bicep deployment scaffold
- runtime config loading
- optional `/interact` Front Door route module
- architecture sketch for Excalidraw

### Not done yet
- real MSAL auth service implementation
- auth route guard
- bearer token HTTP interceptor
- dedicated Interact B2C SPA client registration in `MainInterfaceService`
- APIM audience acceptance for Interact in `UserManagementService` and `PerformanceManagementService`

## Main Local Paths

### Sample app
```text
C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client
```

### HowTo notes
```text
C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\Notes\HowTo
```

### Excalidraw sketch
```text
C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\Notes\Interact_Main_Components.excalidraw
```

## Key Design Decisions To Keep In Mind

### Interact frontend shape
Interact should be a **separate lightweight Angular SPA**, but structurally inspired by `FrontendService`.

Keep:
- `Client/apps/interact`
- runtime config
- MSAL/B2C auth bootstrap
- route guard + HTTP interceptor
- standalone CI/CD + Bicep deployment

Avoid at this stage:
- full FrontendService monorepo complexity
- Andon API-key auth model
- Andon Vue runtime structure

### Auth model
Interact should use:
- **same B2C custom policy** as Bobst Connect
- **dedicated Interact SPA clientId**
- **APIM + UserManagementService authorization flow**

So no custom policy XML change is expected initially in `AzureADB2C`.
The B2C app registration change belongs in `MainInterfaceService`.

## Files To Reopen Before Coding Auth

- `C:\BobstSource\FrontendService\Client\apis\base-api\src\settings\ra-config.service.ts`
- `C:\BobstSource\FrontendService\Client\apps\remote-assistance\src\app\ra-app.module.ts`
- `C:\BobstSource\FrontendService\Client\libs\login\src\services\ra-auth.service.ts`
- `C:\BobstSource\MainInterfaceService\deployment\lib\manageAzureADB2C.ps1`
- `C:\BobstSource\UserManagementService\deployment\templates\policies\backendzr\policy-api-template.xml`
- `C:\BobstSource\PerformanceManagementService\deployment\templates\policies\backendzr\policy-api-template.xml`
