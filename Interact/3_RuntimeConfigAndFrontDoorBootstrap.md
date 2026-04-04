# Interact Runtime Config And Front Door Bootstrap

## Goal

Document the step that connects two missing pieces of the Interact bootstrap:
- runtime configuration loading inside the Angular app
- optional Front Door route deployment for `/interact`

This step does **not** finish authentication.

It only prepares the sample so that:
- the Angular app can read deployment-generated settings
- the deployment layer can generate a runtime config file
- the standalone host can later be exposed through Front Door with the correct path

## Phone Summary

```text
Runtime config:
Angular loads assets/configs/config.runtime.json first.
If it does not exist, it falls back to config.local.json.

Front Door:
optional Bicep route module creates /interact and /interact/*
and sends traffic to the Interact storage static website origin.

Auth status:
B2C values are prepared in config shape, but MSAL login flow is not implemented yet.
```

## Why this step was needed

Before this step, the sample had:
- a minimal Angular structure
- static deployment to storage

But it was missing:
- a config bridge between deployment and frontend runtime
- a deployment concept for `/interact` route exposure

Without these pieces, the sample was still too static to reflect the real Interact direction.

## Mental Model

This step connects **deployment-time values** with **browser runtime behavior**.

The important rule is:
the Angular code should not hardcode environment URLs or B2C tenant/client settings.

Instead:
- deployment generates a JSON config file
- Angular reads that config at startup
- services use that loaded config to build backend URLs and auth settings

## References Used

### FrontendService
Used as reference for runtime configuration loading:
- `C:\BobstSource\FrontendService\Client\apis\base-api\src\settings\ra-config.service.ts`
- `C:\BobstSource\FrontendService\Client\apps\remote-assistance\src\assets\configs\config.template.json`

Key idea reused:
- app loads a JSON config file from `assets/configs`
- runtime values are not hardcoded in the Angular source

### Andon
Used as reference for the Front Door route Bicep module:
- `C:\BobstSource\Andon\deployment\bicep\deploy-frontdoor-route.bicep`

Key idea reused:
- dedicated Front Door route for the standalone app
- path-based routing to the storage-hosted frontend origin

## Files Added

### Frontend runtime config assets
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\public\assets\configs\config.template.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\public\assets\configs\config.local.json`

### Front Door deployment module
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\deploy-frontdoor-route.bicep`

## Files Updated

### Frontend
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\core\config\runtime-config.model.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\core\config\runtime-config.service.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\core\http\interact-api.service.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\features\home\home-page.component.html`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\app.config.ts`

### Deployment
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\azuredeploy.bicep`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\deploy-output.bicep`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\deploy.ps1`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-development.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-feature.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-staging.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-production.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\README.md`

## Frontend Runtime Config Design

### Config files

Two config files were introduced:

1. `config.template.json`
- meant for deployment-time replacement
- contains placeholder tokens such as:
  - `%APP_NAME%`
  - `%ENVIRONMENT_NAME%`
  - `%FRONTEND_BASE_PATH%`
  - `%CONNECT_DOMAIN%`
  - `%BASE_SERVICE_URL%`

2. `config.local.json`
- local fallback used when no deployment-generated runtime file exists
- lets the sample load without a deployment pipeline

### Why we need both files
- `config.template.json` is for CI/CD and deployed environments.
- `config.local.json` is for local dev/demo when no deployment step generated `config.runtime.json`.

### Reference snippet
Template config shape:

```json
{
  "appName": "%APP_NAME%",
  "env": {
    "type": "%ENVIRONMENT_NAME%"
  },
  "frontendBasePath": "%FRONTEND_BASE_PATH%",
  "connectDomain": "%CONNECT_DOMAIN%",
  "baseServiceUrl": "%BASE_SERVICE_URL%",
  "serviceUrl": {
    "performanceManagement": "%PERFORMANCE_MANAGEMENT_URL%",
    "userManagement": "%USER_MANAGEMENT_URL%"
  }
}
```

### Expected deployed file
After deployment, the generated file should be available at:

```text
assets/configs/config.runtime.json
```

That file should contain real values, not `%PLACEHOLDER%` tokens.

### Loader behavior

The Angular app now uses `RuntimeConfigService.load()` during app bootstrap.

Load order:
1. try `assets/configs/config.runtime.json`
2. if missing, fall back to `assets/configs/config.local.json`

This gives the sample two useful behaviors:
- deployed environments can inject values dynamically
- local development still works without deployment replacement

### Why `config.runtime.json` is tried first
In deployed environments, runtime values should come from deployment.
The local fallback is only a safety net for development or sample usage.

### App bootstrap integration

The config load is triggered from:
- `app.config.ts`

Mechanism:
- `provideAppInitializer(() => inject(RuntimeConfigService).load())`

This ensures the app configuration is available before the routed UI starts using it.

### Reference snippet

```ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideHttpClient(),
    provideAppInitializer(() => inject(RuntimeConfigService).load()),
    provideRouter(routes),
  ],
};
```

### Current config shape

The current runtime config supports:
- `appName`
- `env.type`
- `frontendBasePath`
- `connectDomain`
- `baseServiceUrl`
- `serviceUrl.performanceManagement`
- `serviceUrl.userManagement`

This is intentionally smaller than FrontendService.

It is enough for the next steps:
- auth bootstrap
- PMS wiring
- User Management wiring

### Current auth-related config fields
The config model has already been extended to carry B2C/MSAL values such as:
- whether auth is enabled
- clientId
- authority / known authority domain
- scopes
- redirect URI

The code that **uses** these values for real MSAL login is still pending.

### Reference snippet
Current config accessors already expose B2C values:

```ts
readonly authenticationEnabled = computed(() => this.config().authentication.enabled);
readonly b2cClientId = computed(() => this.config().authentication.activeDirectoryB2CClientId);
readonly b2cDomainName = computed(() => this.config().authentication.activeDirectoryB2CDomainName);
readonly b2cCustomDomainName = computed(
  () => this.config().authentication.activeDirectoryB2CCustomDomainName,
);
readonly b2cTenantId = computed(() => this.config().authentication.activeDirectoryB2CTenantId);
readonly b2cPolicy = computed(() => this.config().authentication.activeDirectoryB2CPolicy);
readonly backendB2cApp = computed(() => this.config().authentication.backendADB2CApp);
```

## Deployment Runtime Config Bridge

### What deploy.ps1 now does

The deployment script now includes a `Write-RuntimeConfigFile` function.

Its responsibility is:
1. read `config.template.json` from the built artifact folder
2. replace placeholders using deployment parameters
3. generate `config.runtime.json`
4. upload that generated file together with the rest of the frontend assets

So the runtime config is no longer only a frontend concern.

It is now part of the deployment flow.

### Placeholder replacement concept
The deployment script reads `config.template.json`, replaces tokens like `%BASE_SERVICE_URL%`, then writes `config.runtime.json`.

So the relationship is:

```text
parameters-*.json -> deploy.ps1 -> config.template.json -> config.runtime.json -> RuntimeConfigService
```

### Reference snippet

```powershell
$replacementMap = @{
    '%APP_NAME%' = [string]$ConfigurationParameters['appName']
    '%ENVIRONMENT_NAME%' = [string]$ConfigurationParameters['environmentName']
    '%FRONTEND_BASE_PATH%' = [string]$ConfigurationParameters['frontendBasePath']
    '%CONNECT_DOMAIN%' = [string]$ConfigurationParameters['connectDomain']
    '%BASE_SERVICE_URL%' = [string]$ConfigurationParameters['baseServiceUrl']
    '%PERFORMANCE_MANAGEMENT_URL%' = [string]$ConfigurationParameters['performanceManagementUrl']
    '%USER_MANAGEMENT_URL%' = [string]$ConfigurationParameters['userManagementUrl']
}
```

### Parameters involved

The deployment parameter files now include values for:
- `frontendBasePath`
- `connectDomain`
- `baseServiceUrl`
- `performanceManagementUrl`
- `userManagementUrl`
- `enableFrontDoorRoute`

These values are used for:
- runtime config generation
- later route and backend integration

## Front Door Route Design

### New Bicep module

The new route module is:
- `deploy-frontdoor-route.bicep`

Its purpose is to create a dedicated route for:
- `/interact`
- `/interact/*`

The route forwards traffic to the static website origin created for Interact.

### Route pattern behavior
- `/interact` should load the SPA entry point.
- `/interact/*` should also be forwarded to the same origin so client-side routes can work.

If deep links are introduced later, verify whether a Front Door rule or static-host fallback is needed so Angular routes still resolve to `index.html`.

### Reference snippet

```bicep
param patternsToMatch array = [
  '/interact'
  '/interact/*'
]

resource route 'Microsoft.Cdn/profiles/afdEndpoints/routes@2021-06-01' = {
  name: '${frontDoorProfileName}/${frontDoorEndpointName}/${frontDoorRouteName}'
  properties: {
    originPath: '/'
    patternsToMatch: patternsToMatch
    forwardingProtocol: 'HttpsOnly'
    linkToDefaultDomain: 'Disabled'
    httpsRedirect: 'Enabled'
  }
}
```

### Why optional

Front Door is controlled by Main Interface shared infrastructure.

For the sample bootstrap, we do not want every deployment to require:
- existing Front Door values
- existing Main Interface outputs

So the route is controlled by:
- `enableFrontDoorRoute`

Default:
- `false`

This keeps local/sample deployment simple while still preparing the final architecture.

### New route-related deployment parameters

The main Bicep template now accepts:
- `enableFrontDoorRoute`
- `frontDoorProfileName`
- `frontDoorEndpointName`
- `frontDoorCustomDomain`
- `frontDoorRuleSetName`
- `frontDoorResourceGroupName`

When route deployment is enabled, the module deploys in the Main Interface resource group scope.

## New Outputs

The deployment now returns:
- storage-hosted artifact URL
- final Interact URL

Behavior:
- if Front Door route is disabled, `interactUrl` falls back to the storage web endpoint
- if Front Door route is enabled, `interactUrl` becomes `https://<custom-domain>/interact`

The deploy script also exposes:
- `DEPLOY_OUTPUT_INTERACT_STORAGE_URL`
- `DEPLOY_OUTPUT_INTERACT_URL`

## What this step does not yet do

Still out of scope:
- real B2C redirect URI registration
- token acquisition
- MSAL client wiring
- API interceptor wiring
- Main Interface output resolution for Front Door values
- automatic environment substitution from shared platform outputs

So this step is still infrastructure preparation, not end-to-end auth.

## What To Check Next When Continuing

### Frontend side
- Replace the placeholder auth service with a real MSAL-based service.
- Add a route guard.
- Add an HTTP interceptor that attaches `Authorization: Bearer <token>` when calling APIM APIs.

### MainInterfaceService side
- Add Interact as a dedicated SPA B2C app registration.
- Expose `azureADB2CInteractAppClientId` in deployment outputs.

### APIM/backend side
- Add Interact clientId as an accepted audience in UserManagementService and PMS APIM policies.

## Validation status

Validated:
- frontend lint passes after the runtime config changes
- file structure and deployment scaffolding were updated successfully

Not executed here:
- Angular build
- Azure deployment
- Front Door deployment

Reasons:
- local Angular build remains blocked by the existing Windows `esbuild` environment issue
- Azure context was not available in this environment

## Recommended next step

The next logical module is the auth bootstrap:
- add MSAL/B2C configuration loading from runtime config
- add auth service bootstrap
- prepare the route guard / redirect flow

At that point, the runtime config created here becomes directly useful.

## Short Explanation For Daily

```text
We now have a runtime-config bridge between deployment and Angular,
and a prepared Front Door route module for /interact.
The missing piece is to consume the B2C config with a real MSAL auth service
and then validate one protected API call through APIM.
```
