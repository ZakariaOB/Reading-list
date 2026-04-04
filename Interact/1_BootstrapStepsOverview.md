# Interact Bootstrap Steps Overview

This note explains **what we already did**, **why we did it**, and **which files were created/changed**.

The goal is to make the current prototype easy to present and easy to continue without losing the rationale.

## Phone Summary

If you read only one section before a daily, read this one.

```text
Step 1  repo baseline
Step 2  minimal Angular app
Step 3  lint / tests / Sonar baseline
Step 4  CI skeleton
Step 5  core/config + core/auth placeholder + core/http + home page
Step 6  Bicep + storage static hosting deployment
Step 7  runtime config + optional Front Door /interact route

Current blocker before feature work:
real B2C/MSAL auth bootstrap is not implemented yet.
```

## Mental Model

The current sample is **not** trying to be the final full app yet.

It is a **technical seed** with just enough structure to answer these questions:
- can Interact exist as a separate Angular SPA repo?
- can it stay lightweight while following FrontendService-style structure?
- can it be deployed independently?
- can runtime environment values be injected without rebuilding the app?
- where will auth and backend access live when we wire them?

## What To Open First

If you want to understand the current prototype quickly, open these files in this order:

1. `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\README.md`
2. `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\angular.json`
3. `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\app.routes.ts`
4. `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\core\config\runtime-config.service.ts`
5. `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\deploy.ps1`

## Current Working Folder

```text
C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact
```

The current frontend sample is here:

```text
C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client
```

## Step 1 - Repository Bootstrap

### What we did
- Created a minimal repository baseline with a root `README.md`.
- Added a root `.gitignore` for Node/Angular, local caches, coverage, build outputs, and IDE files.

### Why
- We need a clean new Interact repo before adding the Angular app and pipelines.
- The repo must not commit generated folders such as `node_modules`, build output, or local npm cache.

### Files
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\README.md`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\.gitignore`

### What to verify
- The new Azure DevOps repo exists and local files can be pushed.
- The repo default branch and PR policy strategy are agreed by the team.
- The root `.gitignore` still allows committing source and pipeline files but excludes generated outputs.

## Step 2 - Minimal Angular App Creation

### What we did
- Created a minimal Angular workspace under `Samples\Client`.
- Restructured it to follow the **FrontendService-style** layout with `Client/apps/interact`.
- Replaced the default Angular starter page with a minimal Interact shell.

### Why
- Interact should be a **lightweight standalone SPA**, but still look structurally close to `FrontendService`.
- We want the same app shape and runtime direction as the frontend, without copying the full monorepo complexity.
- This is a deliberate difference from Andon: Andon is Vue/Vite and API-key-oriented, while Interact should be Angular + B2C-user-oriented.

### Files and folders
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\angular.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\package.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\tsconfig.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact`

### Current app structure
```text
Client/
├── angular.json
├── package.json
├── tsconfig.json
└── apps/
    └── interact/
        ├── public/
        ├── src/
        ├── tsconfig.app.json
        └── tsconfig.spec.json
```

### What to look for in `angular.json`
- The project name is `interact`.
- The app root is `apps/interact`.
- Build/test targets are already scoped to that app.

### Reference snippet
This is the key part that makes the workspace a **small frontend-like app under `apps/interact`**:

```json
{
  "newProjectRoot": "apps",
  "projects": {
    "interact": {
      "projectType": "application",
      "root": "apps/interact",
      "sourceRoot": "apps/interact/src",
      "architect": {
        "build": {
          "options": {
            "browser": "apps/interact/src/main.ts",
            "tsConfig": "apps/interact/tsconfig.app.json",
            "assets": [{ "glob": "**/*", "input": "apps/interact/public" }],
            "styles": ["apps/interact/src/styles.scss"]
          }
        }
      }
    }
  }
}
```

### What is intentionally not copied from FrontendService
- full Nx monorepo setup
- multiple app/libs structure
- Storybook/docs viewer
- i18n extraction/release workflows
- feature flags and larger platform-specific pipeline complexity

## Step 3 - Quality Baseline

### What we did
- Added ESLint configuration.
- Added Prettier ignore rules.
- Added Sonar configuration.
- Added npm scripts for lint, test, coverage, formatting, and Sonar.

### Why
- We need a baseline engineering quality setup before the app grows.
- This also prepares the repo for CI validation and later team development.
- The goal is to make quality checks visible from day 1, not retrofit them after business screens are added.

### Files
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\eslint.config.js`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\.prettierignore`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\sonar-project.properties`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\package.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\package-lock.json`

### Validation status
- `lint` / `lint:check` passes locally.
- `build` and `test:coverage` are blocked in the current local Codex/Windows environment by an `esbuild spawn EPERM` issue.
- That failure is considered **environmental**, because it happens when `esbuild` starts its child process, not in application code.

### Practical command intent
- `npm run lint:check`: verify TypeScript/Angular lint rules
- `npm run lint:fix`: auto-fix lint issues where safe
- `npm run test:coverage`: run unit tests with coverage output
- `npm run sonar`: run Sonar scanner using `sonar-project.properties`

### Reference snippet
Current `package.json` scripts:

```json
{
  "scripts": {
    "start": "ng serve interact",
    "build": "ng build interact",
    "test": "ng test interact --watch=false",
    "test:coverage": "ng test interact --watch=false --code-coverage",
    "lint": "eslint . --ext .ts,.html",
    "lint:check": "npm run lint",
    "lint:fix": "eslint . --ext .ts,.html --fix",
    "prettier:check": "prettier --check .",
    "prettier:fix": "prettier --write .",
    "sonar": "sonar-scanner"
  }
}
```

## Step 4 - CI/CD Skeleton

### What we did
- Added a minimal Azure DevOps pipeline folder under `deployment/ci`.
- Added PR and feature pipeline entry files.
- Added reusable task templates for install, lint, tests with coverage, and build.
- Added a shared pipeline variables file.

### Why
- We need a small but clear CI/CD structure early.
- The goal is to keep the repo lightweight, but still ready for PR validation and feature validation.
- We used the **Andon pipeline folder pattern** as a compact reference, but the frontend app itself remains **FrontendService-style**.

### How to read this folder
- `deployment/ci/feature.yml` and `deployment/ci/pull-request.yml` are the pipeline entry points.
- `deployment/ci/templates/variables.yml` holds shared pipeline variables.
- `deployment/ci/templates/tasks/*.yml` contains reusable task steps.

### Important design point
The CI folder shape came from Andon because it is compact.
That does **not** mean the Interact app runtime should follow Andon.
The runtime app should stay frontend-like Angular + B2C, not Vue + API key.

### Reference snippet
Current feature pipeline entry point:

```yaml
trigger: none

pool:
  vmImage: 'ubuntu-latest'

variables:
  - template: templates/variables.yml

stages:
  - stage: BuildAndValidate
    jobs:
      - job: ClientValidation
        steps:
          - template: templates/tasks/install-packages.yml
            parameters:
              workingDirectory: $(ClientWorkingDirectory)

          - template: templates/tasks/run-lint-check.yml
            parameters:
              workingDirectory: $(ClientWorkingDirectory)

          - template: templates/tasks/run-tests-with-coverage.yml
            parameters:
              workingDirectory: $(ClientWorkingDirectory)

          - template: templates/tasks/build-app.yml
            parameters:
              workingDirectory: $(ClientWorkingDirectory)
```

### Files
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\ci\feature.yml`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\ci\pull-request.yml`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\ci\README.md`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\ci\templates\variables.yml`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\ci\templates\tasks\install-packages.yml`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\ci\templates\tasks\run-lint-check.yml`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\ci\templates\tasks\run-tests-with-coverage.yml`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\ci\templates\tasks\build-app.yml`

## Step 5 - Minimal Frontend-Like Internal App Structure

### What we did
- Added a first `core` area with config/auth/http slices.
- Added a first `features/home` page.
- Updated routing so the app shell renders the `HomePageComponent`.
- Enabled `provideHttpClient()` in app config.

### Why
- A raw Angular starter is not enough to explain the target architecture.
- We want a minimal but real structure that already shows where config, auth, and API integration will live.

### Folder meaning
- `core/config`: load deployment/runtime config before app starts
- `core/auth`: future MSAL/B2C login/session logic
- `core/http`: future API access layer for PMS/UserManagement calls
- `features/home`: temporary landing page to display bootstrap status and config values

### Files
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\app.routes.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\app.config.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\core\config\runtime-config.service.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\core\auth\interact-auth.service.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\core\http\interact-api.service.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\features\home\home-page.component.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\features\home\home-page.component.html`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\features\home\home-page.component.scss`

### Important status
- `core/auth/interact-auth.service.ts` is still a **placeholder**.
- The real MSAL/B2C implementation, auth guard, and bearer token interceptor are **not finished yet**.

### Why this placeholder exists
We intentionally created the target folder/file location before implementing MSAL.
That makes the architecture visible and lets the team agree on where auth code should live.

### Reference snippets
Current route wiring:

```ts
import { Routes } from '@angular/router';

import { HomePageComponent } from './features/home/home-page.component';

export const routes: Routes = [{ path: '', component: HomePageComponent }];
```

Current home page dependency injection:

```ts
export class HomePageComponent {
  protected readonly runtimeConfig = inject(RuntimeConfigService);
  protected readonly auth = inject(InteractAuthService);
  protected readonly api = inject(InteractApiService);
}
```

Current auth service placeholder:

```ts
@Injectable({ providedIn: 'root' })
export class InteractAuthService {
  private readonly authenticated = signal(false);

  readonly isAuthenticated = this.authenticated.asReadonly();
  readonly provider = 'Bobst Connect B2C';
  readonly status = computed(() =>
    this.authenticated()
      ? 'Authenticated session available.'
      : 'Authentication bootstrap is not wired yet.',
  );
}
```

This placeholder is exactly what must be replaced in the next auth step.

## Step 6 - Deployment Bootstrap

### What we did
- Added a `deployment` folder with a small Bicep baseline.
- Added storage static hosting resources.
- Added environment parameter files for feature/development/staging/production.
- Added a local `deploy.ps1` script for deployment and artifact upload.
- Renamed `deploy-storage-account-gen2.bicep` to `deploy-storage-account.bicep` because Interact does not need Data Lake Gen2 terminology/capability for this frontend hosting use case.

### Why
- Interact should be independently deployable as a standalone frontend app.
- A small Bicep baseline is needed before connecting Front Door and B2C runtime configuration.

### Deployment model in one sentence
`ng build` creates static browser files, Bicep creates an Azure Storage static website host, and `deploy.ps1` uploads the generated files to `$web`.

### Reference snippet
Main Bicep composition:

```bicep
module interactStorage 'deploy-storage-account.bicep' = {
  name: names.storageDeployment
  params: {
    location: location
    storageAccountsName: names.storageAccountName
    sensitiveResourcesLock: sensitiveResourcesLock
  }
}

module interactWebContainer 'deploy-storage-account-container.bicep' = {
  name: names.webContainerDeployment
  params: {
    storageAccountsName: interactStorage.outputs.storageAccountName
    containerName: names.webContainerName
    sensitiveResourcesLock: sensitiveResourcesLock
  }
}
```

### Files
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\deploy.ps1`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\README.md`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-feature.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-development.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-staging.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\parameters-production.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\azuredeploy.bicep`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\deploy-storage-account.bicep`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\deploy-storage-account-container.bicep`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\deploy-output.bicep`

### More details
- See `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\Notes\HowTo\2_DeploymentBootstrap.md`

## Step 7 - Runtime Config And Front Door Route Bootstrap

### What we did
- Added `config.template.json` and `config.local.json`.
- Updated `RuntimeConfigService` to load `config.runtime.json` first, then fallback to `config.local.json`.
- Updated app bootstrap to initialize runtime config before the app starts.
- Added an optional Bicep module for Front Door route `/interact` and `/interact/*`.
- Updated `deploy.ps1` to generate `config.runtime.json` from `config.template.json`.

### Why
- Environment-specific values such as backend URL, B2C tenant/client config, and Connect domain should come from deployment/runtime config, not hardcoded source code.
- Front Door routing is needed so the standalone app can be exposed as `https://<prefix>.connect.bobst.com/interact`.

### Runtime config flow in one sentence
`deploy.ps1` generates `assets/configs/config.runtime.json`, and `RuntimeConfigService` loads that file at Angular startup; if it is missing, local fallback config is used.

### Reference snippets
App bootstrap loads runtime config before route rendering:

```ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),
    provideAppInitializer(() => inject(RuntimeConfigService).load()),
    provideRouter(routes),
  ],
};
```

`RuntimeConfigService` load order:

```ts
async load(): Promise<void> {
  const runtimeConfigPath = 'assets/configs/config.runtime.json';
  const localConfigPath = 'assets/configs/config.local.json';

  try {
    const config = await firstValueFrom(
      this.http.get<InteractRuntimeConfig>(runtimeConfigPath, {
        headers: { 'Cache-Control': 'no-cache' },
      }),
    );

    this.config.set(config);
    this.loadedFromPath.set(runtimeConfigPath);
    return;
  } catch {
    const config = await firstValueFrom(this.http.get<InteractRuntimeConfig>(localConfigPath));
    this.config.set(config);
    this.loadedFromPath.set(localConfigPath);
  }
}
```

### Files
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\public\assets\configs\config.template.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\public\assets\configs\config.local.json`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\core\config\runtime-config.model.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\core\config\runtime-config.service.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\apps\interact\src\app\app.config.ts`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\deploy.ps1`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\deploy-frontdoor-route.bicep`
- `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\0_Interact\Samples\Client\deployment\bicep\azuredeploy.bicep`

### More details
- See `C:\Users\zboukhris\Desktop\Bobst_DAILY\Sprints\187_Sprint\Interact\Notes\HowTo\3_RuntimeConfigAndFrontDoorBootstrap.md`

## Current State Before Continuing

### Done
- Repo baseline
- Minimal Angular app structure
- Quality baseline
- CI skeleton
- Deployment scaffold
- Runtime config + optional Front Door route
- Architecture notes and Excalidraw sketch

### Not done yet
- Real MSAL/B2C auth service implementation
- Route guard
- Bearer token HTTP interceptor
- Dedicated Interact B2C SPA client registration changes in `MainInterfaceService`
- APIM audience changes in `UserManagementService` and `PerformanceManagementService`
- First protected API call validation through APIM

## Next Recommended Step

Continue with **Auth Bootstrap** in the sample app:

1. Implement `core/auth/interact-auth.service.ts` with MSAL browser client initialization.
2. Add an auth guard for the protected Interact route.
3. Add an HTTP interceptor that injects `Authorization: Bearer <token>`.
4. Extend deployment runtime config values for Interact B2C settings.
5. Document the MainInterfaceService and APIM changes needed for the dedicated Interact clientId.

## Suggested Daily Explanation

If you need to explain this in one short daily update:

```text
We now have a lightweight Interact Angular seed repo with basic quality checks,
CI skeleton, standalone Bicep static hosting, runtime config loading,
and an optional Front Door /interact route module.
The next implementation step is the real MSAL/B2C auth bootstrap and
the platform-side clientId/APIM audience plumbing.
```
