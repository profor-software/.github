# AL-Go Workflows - Technische Detaildokumentation

Detaillierte technische Dokumentation aller GitHub Workflows im Repository mit vollständiger Auflistung von Jobs, Steps und Konfigurationsmöglichkeiten.

**AL-Go Version:** 6.2  
**Letzte Aktualisierung:** April 2026  
**Business Central Version:** 26.x

---

## Inhaltsverzeichnis

1. [CI/CD](#1-cicd)
2. [Pull Request Handler](#2-pull-request-handler)
3. [Create Release](#3-create-release)
4. [Increment Version Number](#4-increment-version-number)
5. [Publish To Environment](#5-publish-to-environment)
6. [Create App](#6-create-app)
7. [Create Test App](#7-create-test-app)
8. [Create Performance Test App](#8-create-performance-test-app)
9. [Add Existing App or Test App](#9-add-existing-app-or-test-app)
10. [Create Online Development Environment](#10-create-online-development-environment)
11. [Current / Next Minor / Next Major](#11-current--next-minor--next-major)
12. [Update AL-Go System Files](#12-update-al-go-system-files)
13. [Deploy Reference Documentation](#13-deploy-reference-documentation)
14. [Troubleshooting](#14-troubleshooting)
15. [_Build AL-Go Project (Interner Workflow)](#15-_build-al-go-project-interner-workflow)

---

## 1. CI/CD

### Workflow-Name
`CI/CD` (`.github/workflows/CICD.yaml`)

### Beschreibung
Haupt-Workflow für Continuous Integration und Continuous Deployment. Kompiliert alle Apps, führt Tests aus, erstellt Artifacts und deployt automatisch auf konfigurierte Umgebungen.

### Auslösemöglichkeiten

**Aktuell:**
- Manuell über `workflow_dispatch`

**Konfigurierbar:**
- Automatisch bei Push auf definierte Branches (über `CICDPushBranches` Setting)
- Zeitgesteuert über CRON (über `workflowSchedule` Setting)

### Input-Parameter

Keine (Workflow wird über Settings gesteuert)

### Notwendige Settings

#### GitHub Secrets (Optional):
- `licenseFileUrl`: URL zur BC-Lizenz (für ältere BC-Versionen)
- `codeSignCertificateUrl`: URL zum Code-Signing-Zertifikat
- `codeSignCertificatePassword`: Passwort für Zertifikat
- `keyVaultCertificateUrl`: Azure KeyVault Zertifikat-URL
- `keyVaultCertificatePassword`: KeyVault Zertifikat-Passwort
- `keyVaultClientId`: KeyVault Client-ID
- `gitHubPackagesContext`: Context für GitHub Packages
- `applicationInsightsConnectionString`: Application Insights Connection String
- `gitSubmodulesToken`: Token für Git Submodules (bei Verwendung)
- `<EnvironmentName>_AuthContext`: Auth-Context pro Environment

#### AL-Go-Settings.json:
```json
{
  "country": "de",
  "appFolders": ["src"],
  "testFolders": ["test"],
  "environments": ["QA", "Production"],
  "CICDPushBranches": ["main", "release/*", "feature/*"],
  "buildModes": ["Default", "Clean"],
  "versioningStrategy": 0
}
```

### Anpassungsmöglichkeiten

#### Automatischer Push-Trigger aktivieren:
```json
{
  "CICDPushBranches": ["main", "release/*", "feature/*"]
}
```

#### Zeitgesteuerte Ausführung (täglich um 2 Uhr):
Datei: `.github/CICD.settings.json`
```json
{
  "workflowSchedule": {
    "cron": "0 2 * * *",
    "includeBranches": ["main"]
  }
}
```

#### Inkrementelle Builds aktivieren:
```json
{
  "incrementalBuilds": {
    "onPush": true,
    "onPull_Request": true,
    "onSchedule": false,
    "retentionDays": 30,
    "mode": "modifiedProjects"
  },
  "workflowConcurrency": [
    "group: ${{ github.workflow }}-${{ github.ref }}",
    "cancel-in-progress: true"
  ]
}
```

#### Build-Modi konfigurieren:
```json
{
  "buildModes": ["Default", "Clean", "Translated"]
}
```

#### Continuous Deployment konfigurieren:
```json
{
  "DeployToQA": {
    "EnvironmentType": "SaaS",
    "Branches": ["main"],
    "ContinuousDeployment": true,
    "SyncMode": "Add"
  }
}
```

### Permissions
```yaml
permissions:
  actions: read
  contents: read
  id-token: write
  pages: read
```

### Environment Variables
```yaml
env:
  workflowDepth: 1
  ALGoOrgSettings: ${{ vars.ALGoOrgSettings }}
  ALGoRepoSettings: ${{ vars.ALGoRepoSettings }}
```

### Jobs und Steps

#### Job 1: Initialization
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** Keine

**Outputs:**
- `telemetryScopeJson`: Telemetrie-Scope
- `environmentsMatrixJson`: Deployment-Environments als Matrix
- `environmentCount`: Anzahl Environments
- `deploymentEnvironmentsJson`: Deployment-Environment-Details
- `generateALDocArtifact`: Flag für AL-Doc-Generierung
- `deployALDocArtifact`: Flag für AL-Doc-Deployment
- `deliveryTargetsJson`: Delivery-Targets
- `githubRunner`: GitHub Runner-Konfiguration
- `githubRunnerShell`: Shell für Runner
- `projects`: Projekte zum Bauen
- `projectDependenciesJson`: Projekt-Dependencies
- `buildOrderJson`: Build-Reihenfolge
- `powerPlatformSolutionFolder`: Power Platform Ordner
- `workflowDepth`: Workflow-Tiefe

**Steps:**
1. **Dump Workflow Information**
   - Action: `microsoft/AL-Go-Actions/DumpWorkflowInfo@v6.2`
   - Funktion: Gibt Workflow-Informationen aus

2. **Checkout**
   - Action: `actions/checkout@v4.2.2`
   - Funktion: Checked Repository aus
   - Parameter: `lfs: true` (Git LFS aktiviert)

3. **Initialize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowInitialize@v6.2`
   - Funktion: Initialisiert Workflow, setzt Telemetrie

4. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`
   - Funktion: Liest AL-Go Settings
   - Parameter: `get: type,powerPlatformSolutionFolder,useGitSubmodules`

5. **Read submodules token** (Conditional)
   - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
   - Bedingung: `if: env.useGitSubmodules != 'false' && env.useGitSubmodules != ''`
   - Funktion: Liest Token für Git Submodules
   - Parameter: `getSecrets: '-gitSubmodulesToken'`

6. **Checkout Submodules** (Conditional)
   - Action: `actions/checkout@v4.2.2`
   - Bedingung: `if: env.useGitSubmodules != 'false' && env.useGitSubmodules != ''`
   - Funktion: Checked Submodules aus
   - Parameter: `lfs: true`, `submodules: ${{ env.useGitSubmodules }}`

7. **Determine Workflow Depth**
   - Funktion: Setzt Workflow-Tiefe aus Environment Variable

8. **Determine Projects To Build**
   - Action: `microsoft/AL-Go-Actions/DetermineProjectsToBuild@v6.2`
   - Funktion: Ermittelt Projekte, Dependencies und Build-Reihenfolge
   - Parameter: `maxBuildDepth: ${{ env.workflowDepth }}`

9. **Determine PowerPlatform Solution Folder** (Conditional)
   - Bedingung: `if: env.type == 'PTE'`
   - Funktion: Ermittelt Power Platform Ordner

10. **Determine Delivery Target Secrets**
    - Action: `microsoft/AL-Go-Actions/DetermineDeliveryTargets@v6.2`
    - Funktion: Ermittelt benötigte Secrets für Delivery
    - Parameter: `checkContextSecrets: 'false'`

11. **Read secrets**
    - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
    - Funktion: Liest benötigte Secrets aus GitHub Secrets oder KeyVault

12. **Determine Delivery Targets**
    - Action: `microsoft/AL-Go-Actions/DetermineDeliveryTargets@v6.2`
    - Funktion: Ermittelt finale Delivery-Targets
    - Parameter: `checkContextSecrets: 'true'`

13. **Determine Deployment Environments**
    - Action: `microsoft/AL-Go-Actions/DetermineDeploymentEnvironments@v6.2`
    - Funktion: Ermittelt Deployment-Environments
    - Parameter: `getEnvironments: '*'`, `type: 'CD'`

#### Job 2: CheckForUpdates
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** `Initialization`

**Steps:**
1. **Checkout**
   - Action: `actions/checkout@v4.2.2`
   - Funktion: Checked Repository aus

2. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`
   - Funktion: Liest templateUrl Setting

3. **Check for updates to AL-Go system files**
   - Action: `microsoft/AL-Go-Actions/CheckForUpdates@v6.2`
   - Funktion: Prüft auf verfügbare AL-Go System File Updates
   - Parameter: `downloadLatest: true`

#### Job 3: Build
**Läuft auf:** Matrix-basiert (siehe `buildOrderJson`)  
**Abhängigkeiten:** `Initialization`  
**Bedingung:** `if: (!failure()) && (!cancelled()) && fromJson(needs.Initialization.outputs.buildOrderJson)[0].projectsCount > 0`  
**Strategy:** `matrix` mit `buildDimensions` aus Initialization

**Matrix-Dimensionen:**
- `project`: Projekt-Pfad
- `projectName`: Projekt-Name
- `buildMode`: Build-Modus (Default, Clean, etc.)
- `githubRunner`: Runner-Typ
- `githubRunnerShell`: Shell-Typ

**Aufruf:**
- Verwendet wiederverwendbaren Workflow: `./.github/workflows/_BuildALGoProject.yaml`
- Secrets: `inherit`

**Parameter:**
- `shell`: ${{ matrix.githubRunnerShell }}
- `runsOn`: ${{ matrix.githubRunner }}
- `project`: ${{ matrix.project }}
- `projectName`: ${{ matrix.projectName }}
- `buildMode`: ${{ matrix.buildMode }}
- `projectDependenciesJson`: Dependencies
- `secrets`: 'licenseFileUrl,codeSignCertificateUrl,*codeSignCertificatePassword,...'
- `publishThisBuildArtifacts`: Basierend auf workflowDepth
- `publishArtifacts`: Bei main/release Branches oder Delivery-Targets
- `signArtifacts`: true
- `useArtifactCache`: true

#### Job 4: DeployALDoc (Conditional)
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** `Initialization`, `Build`  
**Bedingung:** `if: (!cancelled()) && needs.Build.result == 'Success' && needs.Initialization.outputs.generateALDocArtifact == 1 && github.ref_name == 'main'`

**Environment:**
- name: github-pages
- url: ${{ steps.deployment.outputs.page_url }}

**Permissions:**
```yaml
permissions:
  contents: read
  actions: read
  pages: write
  id-token: write
```

**Steps:**
1. **Checkout**
   - Action: `actions/checkout@v4.2.2`

2. **Download artifacts**
   - Action: `actions/download-artifact@v4.1.8`
   - Funktion: Lädt Build-Artifacts herunter
   - Parameter: `path: '.artifacts'`

3. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`
   - Funktion: Liest Einstellungen

4. **Build Reference Documentation**
   - Action: `microsoft/AL-Go-Actions/BuildReferenceDocumentation@v6.2`
   - Funktion: Generiert AL-Referenzdokumentation

5. **Upload pages artifact**
   - Action: `actions/upload-pages-artifact@v3`
   - Funktion: Lädt Dokumentation als Pages-Artifact hoch

6. **Deploy to GitHub Pages**
   - Action: `actions/deploy-pages@v4`
   - Funktion: Deployed Dokumentation auf GitHub Pages

#### Job 5: Deploy (Conditional, Matrix)
**Läuft auf:** Matrix-basiert  
**Abhängigkeiten:** `Initialization`, `Build`  
**Bedingung:** `if: (!cancelled()) && needs.Build.result == 'Success' && needs.Initialization.outputs.environmentCount > 0`  
**Strategy:** Matrix aus `environmentsMatrixJson`

**Environment:**
- name: ${{ matrix.environment }}
- url: ${{ steps.Deploy.outputs.environmentUrl }}

**Steps:**
1. **Checkout**
2. **EnvName** - Extrahiert Environment-Name
3. **Read settings** - Liest Environment-Settings
4. **Read secrets** - Liest Auth-Context für Environment
5. **Deploy** - Führt Deployment auf Environment aus
   - Action: `microsoft/AL-Go-Actions/Deploy@v6.2`
   - Parameter: Apps, Environment-Details, Auth-Context

#### Job 6: Deliver (Conditional, Matrix)
**Läuft auf:** Matrix-basiert  
**Abhängigkeiten:** `Initialization`, `Build`  
**Bedingung:** Wenn Delivery-Targets konfiguriert

**Steps:**
1. **Checkout**
2. **Read settings**
3. **Read secrets**
4. **Deliver** - Liefert Apps an konfigurierte Delivery-Targets
   - Action: `microsoft/AL-Go-Actions/Deliver@v6.2`

#### Job 7: PostProcess
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** Alle vorherigen Jobs  
**Bedingung:** `if: (!cancelled())`

**Steps:**
1. **Finalize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowPostProcess@v6.2`
   - Funktion: Finalisiert Workflow, sammelt Telemetrie

---

## 2. Pull Request Handler

### Workflow-Name
`Pull Request Build` (`.github/workflows/PullRequestHandler.yaml`)

### Beschreibung
Automatischer Workflow, der bei Pull Requests auf den Main-Branch getriggert wird. Validiert Code-Änderungen durch Build und Tests, bevor sie gemergt werden.

### Auslösemöglichkeiten

**Automatisch:**
- Pull Request auf Branch `main` (konfigurierbar über `CICDPullRequestBranches`)

**Trigger-Typ:**
- `pull_request_target` (Standard, sicherer für Forks)
- Konfigurierbar auf `pull_request` über Setting

### Input-Parameter

Keine (automatischer Trigger)

### Notwendige Settings

#### GitHub Secrets (Optional):
- `licenseFileUrl`: URL zur BC-Lizenz
- `keyVaultCertificateUrl`: Azure KeyVault Zertifikat-URL
- `keyVaultCertificatePassword`: KeyVault Zertifikat-Passwort
- `keyVaultClientId`: KeyVault Client-ID
- `gitHubPackagesContext`: Context für GitHub Packages
- `applicationInsightsConnectionString`: Application Insights

#### AL-Go-Settings.json:
```json
{
  "CICDPullRequestBranches": ["main", "develop"],
  "pullRequestTrigger": "pull_request_target",
  "fullBuildPatterns": ["Build/*", ".AL-Go/*"],
  "incrementalBuilds": {
    "onPull_Request": true
  }
}
```

### Anpassungsmöglichkeiten

#### Pull Request Branches konfigurieren:
```json
{
  "CICDPullRequestBranches": ["main", "develop", "release/*"]
}
```

#### Full Build bei bestimmten Datei-Änderungen:
```json
{
  "fullBuildPatterns": [
    "Build/*",
    ".AL-Go/*",
    ".github/workflows/*",
    "app.json"
  ]
}
```

#### Trigger-Typ ändern (für Fork-PRs mit Secrets):
```json
{
  "pullRequestTrigger": "pull_request_target"
}
```

#### Immer alle Projekte bauen:
```json
{
  "alwaysBuildAllProjects": true
}
```

#### Inkrementelle Builds deaktivieren:
```json
{
  "incrementalBuilds": {
    "onPull_Request": false
  }
}
```

### Concurrency
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```
Stellt sicher, dass nur ein Build pro PR gleichzeitig läuft.

### Permissions
```yaml
permissions:
  actions: read
  contents: read
  id-token: write
  pull-requests: read
```

### Environment Variables
```yaml
env:
  workflowDepth: 1
  ALGoOrgSettings: ${{ vars.ALGoOrgSettings }}
  ALGoRepoSettings: ${{ vars.ALGoRepoSettings }}
```

### Jobs und Steps

#### Job 1: PregateCheck (Conditional)
**Läuft auf:** `windows-latest`  
**Bedingung:** `if: (github.event.pull_request.base.repo.full_name != github.event.pull_request.head.repo.full_name) && (github.event_name != 'pull_request')`  
**Funktion:** Sicherheitsprüfung für Fork-PRs

**Steps:**
1. **Verify PR Changes**
   - Action: `microsoft/AL-Go-Actions/VerifyPRChanges@v6.2`
   - Funktion: Verifiziert, dass PR keine gefährlichen Änderungen enthält

#### Job 2: Initialization
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** `PregateCheck`  
**Bedingung:** `if: (!failure() && !cancelled())`

**Outputs:**
- `projects`: Zu bauende Projekte
- `projectDependenciesJson`: Projekt-Dependencies
- `buildOrderJson`: Build-Reihenfolge
- `baselineWorkflowRunId`: Baseline für inkrementelle Builds
- `workflowDepth`: Workflow-Tiefe
- `telemetryScopeJson`: Telemetrie-Scope

**Steps:**
1. **Dump Workflow Information**
   - Action: `microsoft/AL-Go-Actions/DumpWorkflowInfo@v6.2`

2. **Checkout**
   - Action: `actions/checkout@v4.2.2`
   - Parameter: `lfs: true`, `ref: ${{ github.event_name == 'pull_request' && github.sha || format('refs/pull/{0}/merge', github.event.pull_request.number) }}`
   - Funktion: Checked PR-Merge-Commit aus

3. **Initialize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowInitialize@v6.2`

4. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`

5. **Determine Workflow Depth**
   - Funktion: Setzt Workflow-Tiefe

6. **Determine Projects To Build**
   - Action: `microsoft/AL-Go-Actions/DetermineProjectsToBuild@v6.2`
   - Funktion: Ermittelt geänderte Projekte (bei inkrementellen Builds)
   - Parameter: `maxBuildDepth: ${{ env.workflowDepth }}`

#### Job 3: Build
**Läuft auf:** Matrix-basiert  
**Abhängigkeiten:** `Initialization`  
**Bedingung:** `if: (!failure()) && (!cancelled()) && fromJson(needs.Initialization.outputs.buildOrderJson)[0].projectsCount > 0`  
**Strategy:** Matrix mit `buildDimensions`

**Aufruf:**
- Wiederverwendbarer Workflow: `./.github/workflows/_BuildALGoProject.yaml`
- Secrets: `inherit`

**Parameter:**
- `shell`: ${{ matrix.githubRunnerShell }}
- `runsOn`: ${{ matrix.githubRunner }}
- `checkoutRef`: PR-Merge-Commit
- `project`: ${{ matrix.project }}
- `projectName`: ${{ matrix.projectName }}
- `buildMode`: ${{ matrix.buildMode }}
- `projectDependenciesJson`: Dependencies
- `baselineWorkflowRunId`: Für inkrementelle Builds
- `secrets`: Required secrets
- `publishThisBuildArtifacts`: Basierend auf workflowDepth
- `artifactsNameSuffix`: 'PR${{ github.event.number }}'

#### Job 4: StatusCheck
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** `Initialization`, `Build`  
**Bedingung:** `if: (!cancelled())`

**Steps:**
1. **Pull Request Status Check**
   - Action: `microsoft/AL-Go-Actions/PullRequestStatusCheck@v6.2`
   - Funktion: Prüft Build-Status und postet Ergebnis in PR

2. **Finalize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowPostProcess@v6.2`
   - Funktion: Finalisiert Workflow

---

## 3. Create Release

### Workflow-Name
`Create release` (`.github/workflows/CreateRelease.yaml`)

### Beschreibung
Erstellt einen formalen GitHub Release mit Versionierung, Release Notes, optional Release-Branch und automatischem Version-Increment im Main-Branch.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `appVersion` | string | Nein | 'latest' | App-Version für Release (current, prerelease, draft, latest, oder Versionsnummer) |
| `name` | string | Ja | '' | Name des Release (z.B. "v1.0") |
| `tag` | string | Ja | '' | Semantic Version Tag (z.B. "1.0.0") |
| `prerelease` | boolean | Nein | false | Als Pre-Release markieren |
| `draft` | boolean | Nein | false | Als Draft erstellen |
| `createReleaseBranch` | boolean | Nein | false | Release-Branch erstellen |
| `releaseBranchPrefix` | string | Nein | 'release/' | Präfix für Release-Branch |
| `updateVersionNumber` | string | Nein | '' | Neue Version im Main (z.B. "+0.1" oder "2.0") |
| `directCommit` | boolean | Nein | false | Direkter Commit statt PR |
| `useGhTokenWorkflow` | boolean | Nein | false | GhTokenWorkflow Secret verwenden |

### Notwendige Settings

#### GitHub Secrets:
- `GhTokenWorkflow` (oder `TokenForPush`): PAT für Version-Update-PR
- `<DeliveryTarget>Context`: Für Delivery (z.B. `StorageContext`, `AppSourceContext`)

#### AL-Go-Settings.json:
```json
{
  "repoVersion": "1.0",
  "versioningStrategy": 0,
  "deliveryTargets": ["Storage", "AppSource"],
  "DeliverToStorage": {
    "Branches": ["main", "release/*"]
  },
  "deliverToAppSource": {
    "branches": ["main"],
    "productId": "your-product-id",
    "continuousDelivery": false
  }
}
```

### Anpassungsmöglichkeiten

#### Automatisches Increment konfigurieren:
```json
{
  "versioningStrategy": 0
}
```
Bei `+0.1` Input wird Minor-Version erhöht.

#### Delivery Targets konfigurieren:
```json
{
  "deliveryTargets": ["Storage", "NuGet", "GitHub Packages"],
  "DeliverToStorage": {
    "Branches": ["main", "release/*"],
    "CreateContainerIfNotExist": true
  }
}
```

#### AppSource Delivery:
```json
{
  "deliverToAppSource": {
    "branches": ["main"],
    "productId": "12345678-1234-1234-1234-123456789012",
    "mainAppFolder": "src/MyApp",
    "continuousDelivery": false,
    "includeDependencies": ["*.app"]
  }
}
```

#### Release Branch Naming:
Input: `releaseBranchPrefix: "releases/"`  
Ergebnis: `releases/1.0.0`

#### Direct Commit für Version Update:
Input: `directCommit: true`  
Resultat: Keine PR, direkter Commit auf main

### Concurrency
```yaml
concurrency: release
```
Nur ein Release-Prozess gleichzeitig.

### Permissions
```yaml
permissions:
  actions: read
  contents: write
  id-token: write
  pull-requests: write
```

### Environment Variables
```yaml
env:
  ALGoOrgSettings: ${{ vars.ALGoOrgSettings }}
  ALGoRepoSettings: ${{ vars.ALGoRepoSettings }}
```

### Jobs und Steps

#### Job 1: CreateRelease
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** Keine

**Outputs:**
- `artifacts`: Release-Artifacts
- `releaseId`: GitHub Release ID
- `commitish`: Commit SHA für Release
- `releaseVersion`: Release-Version
- `telemetryScopeJson`: Telemetrie-Scope

**Steps:**
1. **Dump Workflow Information**
   - Action: `microsoft/AL-Go-Actions/DumpWorkflowInfo@v6.2`

2. **Checkout**
   - Action: `actions/checkout@v4.2.2`

3. **Initialize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowInitialize@v6.2`

4. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`
   - Parameter: `get: templateUrl,repoName,type,powerPlatformSolutionFolder`

5. **Read secrets**
   - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
   - Parameter: `getSecrets: 'TokenForPush'`
   - Funktion: Liest Token für Version-Update-PR

6. **Determine Projects**
   - Action: `microsoft/AL-Go-Actions/DetermineProjectsToBuild@v6.2`
   - Funktion: Ermittelt Projekte für Release

7. **Check for updates to AL-Go system files**
   - Action: `microsoft/AL-Go-Actions/CheckForUpdates@v6.2`
   - Funktion: Warnt bei verfügbaren Updates

8. **Analyze Artifacts**
   - Funktion: Sucht und validiert Build-Artifacts für Release
   - Verwendet GitHub API um Artifacts zu finden
   - Filter: Nicht abgelaufene Artifacts
   - Matching: Basierend auf `appVersion` Input

9. **Prepare Release Notes**
   - Action: `microsoft/AL-Go-Actions/CreateReleaseNotes@v6.2`
   - Funktion: Generiert Release Notes aus Commits

10. **Create Release**
    - Action: `microsoft/AL-Go-Actions/CreateRelease@v6.2`
    - Funktion: Erstellt GitHub Release
    - Parameter:
      - `tag_name`: Input tag
      - `name`: Input name
      - `body`: Generated Release Notes
      - `draft`: Input draft
      - `prerelease`: Input prerelease
      - `commitish`: Commit SHA
      - `artifacts`: Ausgewählte Artifacts

11. **Create Release Branch** (Conditional)
    - Bedingung: `if: github.event.inputs.createReleaseBranch == 'true'`
    - Funktion: Erstellt Branch mit Präfix + Tag
    - Beispiel: `release/1.0.0`

#### Job 2: UpgradeTestCurrent (Conditional, Matrix)
**Läuft auf:** Matrix-basiert  
**Abhängigkeiten:** `CreateRelease`  
**Bedingung:** Wenn Projekte vorhanden

**Funktion:** Testet Upgrade von vorheriger Version auf neue Release-Version

**Steps:**
1. **Checkout**
2. **Download Current Artifacts** - Neue Release-Artifacts
3. **Download Previous Artifacts** - Vorherige Release-Artifacts
4. **Run Upgrade Tests** - Führt Upgrade-Tests aus

#### Job 3: UpgradeTestNextMinor (Conditional, Matrix)
**Läuft auf:** Matrix-basiert  
**Abhängigkeiten:** `CreateRelease`

**Funktion:** Testet Upgrade mit Next Minor BC-Version

#### Job 4: UpgradeTestNextMajor (Conditional, Matrix)
**Läuft auf:** Matrix-basiert  
**Abhängigkeiten:** `CreateRelease`

**Funktion:** Testet Upgrade mit Next Major BC-Version

#### Job 5: CreateReleaseBranch (Conditional)
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** `CreateRelease`, Upgrade-Test-Jobs  
**Bedingung:** `if: github.event.inputs.createReleaseBranch == 'true'`

**Steps:**
1. **Checkout**
2. **Create Release Branch**
   - Funktion: `git checkout -b ${{ inputs.releaseBranchPrefix }}${{ inputs.tag }}`
   - Push: `git push origin ${{ inputs.releaseBranchPrefix }}${{ inputs.tag }}`

#### Job 6: UpdateVersionNumber (Conditional)
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** `CreateRelease`, alle Test-Jobs  
**Bedingung:** `if: github.event.inputs.updateVersionNumber != ''`

**Steps:**
1. **Checkout**
2. **Read settings**
3. **Read secrets**
4. **Update Version Number**
   - Action: `microsoft/AL-Go-Actions/IncrementVersionNumber@v6.2`
   - Parameter:
     - `versionNumber`: Input updateVersionNumber
     - `directCommit`: Input directCommit
   - Funktion: Erhöht Version in app.json im main-Branch

5. **Create Pull Request** (Conditional)
   - Bedingung: `if: !inputs.directCommit`
   - Funktion: Erstellt PR für Version-Update
   - Title: "Update version to ${{ inputs.updateVersionNumber }}"

#### Job 7: PostProcess
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** Alle vorherigen Jobs  
**Bedingung:** `if: (!cancelled())`

**Steps:**
1. **Finalize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowPostProcess@v6.2`

---

## 4. Increment Version Number

### Workflow-Name
`Increment Version Number` (`.github/workflows/IncrementVersionNumber.yaml`)

### Beschreibung
Aktualisiert die Major.Minor Version in app.json Dateien aller oder ausgewählter Projekte. Kann absolute oder inkrementelle Änderungen durchführen.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `projects` | string | Nein | '*' | Komma-separierte Liste von Projekt-Patterns |
| `versionNumber` | string | Ja | - | Neue Version (z.B. "2.0" oder "+0.1") |
| `directCommit` | boolean | Nein | false | Direkter Commit statt PR |
| `useGhTokenWorkflow` | boolean | Nein | false | GhTokenWorkflow Secret verwenden |

### Notwendige Settings

#### GitHub Secrets:
- `GhTokenWorkflow` (oder `TokenForPush`): PAT für Commit/PR

#### AL-Go-Settings.json:
```json
{
  "versioningStrategy": 0,
  "repoVersion": "1.0"
}
```

### Anpassungsmöglichkeiten

#### Absolute Version-Änderung:
Input: `versionNumber: "2.0"`  
Resultat: Setzt Major=2, Minor=0 in allen app.json

#### Inkrementelle Version-Änderung:
Input: `versionNumber: "+0.1"`  
Resultat: Erhöht Minor um 1

Input: `versionNumber: "+1.0"`  
Resultat: Erhöht Major um 1, setzt Minor auf 0

#### Spezifische Projekte:
Input: `projects: "App1,App2"`  
Resultat: Nur App1 und App2 werden aktualisiert

#### Pattern Matching:
Input: `projects: "Test*"`  
Resultat: Alle Projekte mit "Test" Präfix

#### Direct Commit:
Input: `directCommit: true`  
Resultat: Keine PR, direkter Commit

### Permissions
```yaml
permissions:
  actions: read
  contents: write
  id-token: write
  pull-requests: write
```

### Environment Variables
```yaml
env:
  ALGoOrgSettings: ${{ vars.ALGoOrgSettings }}
  ALGoRepoSettings: ${{ vars.ALGoRepoSettings }}
```

### Jobs und Steps

#### Job 1: IncrementVersionNumber
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** Keine

**Steps:**
1. **Dump Workflow Information**
   - Action: `microsoft/AL-Go-Actions/DumpWorkflowInfo@v6.2`

2. **Checkout**
   - Action: `actions/checkout@v4.2.2`

3. **Initialize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowInitialize@v6.2`

4. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`

5. **Read secrets**
   - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
   - Parameter: `getSecrets: 'TokenForPush'`

6. **Increment Version Number**
   - Action: `microsoft/AL-Go-Actions/IncrementVersionNumber@v6.2`
   - Parameter:
     - `token`: Token für Commit/PR
     - `projects`: Input projects
     - `versionNumber`: Input versionNumber
     - `directCommit`: Input directCommit
   - Funktion:
     - Parsed Projekt-Pattern
     - Findet alle matching Projekte
     - Liest app.json aus jedem Projekt
     - Berechnet neue Version (absolut oder inkrementell)
     - Schreibt aktualisierte app.json
     - Erstellt Commit oder PR

7. **Finalize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowPostProcess@v6.2`
   - Bedingung: `if: always()`

---

## 5. Publish To Environment

### Workflow-Name
`Publish To Environment` (`.github/workflows/PublishToEnvironment.yaml`)

### Beschreibung
Deployed eine spezifische App-Version auf eine oder mehrere Business Central Umgebungen. Unterstützt Wildcard-Matching für Environment-Namen.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `appVersion` | string | Nein | 'current' | Version zum Deployen (current, prerelease, draft, latest, oder Versionsnummer) |
| `environmentName` | string | Ja | - | Environment-Maske (* für alle, PROD* für alle PROD-Envs) |

### Notwendige Settings

#### GitHub Secrets:
- `<EnvironmentName>_AuthContext` (oder `<EnvironmentName>-AuthContext`, `AuthContext`): Auth-Context für Environment
  - Service-to-Service (S2S): `{"clientId":"...","clientSecret":"...","tenantId":"..."}`
  - RefreshToken: `{"refreshToken":"..."}`

#### AL-Go-Settings.json:
```json
{
  "environments": ["QA", "Production"],
  "DeployToQA": {
    "EnvironmentType": "SaaS",
    "EnvironmentName": "My QA Environment",
    "Branches": ["main"],
    "SyncMode": "Add",
    "Scope": "PTE",
    "DependencyInstallMode": "install"
  },
  "DeployToProduction": {
    "EnvironmentType": "SaaS",
    "Branches": ["main"],
    "SyncMode": "ForceSync",
    "ContinuousDeployment": false
  }
}
```

### Anpassungsmöglichkeiten

#### Wildcard Environment Matching:
Input: `environmentName: "PROD*"`  
Resultat: Deployt auf alle Environments, die mit "PROD" beginnen

Input: `environmentName: "*"`  
Resultat: Deployt auf alle konfigurierten Environments

#### Version Selection:
- `current`: Neueste Version vom main-Branch
- `latest`: Neueste GitHub Release
- `prerelease`: Neueste Pre-Release
- `draft`: Neueste Draft-Release
- `1.0.0`: Spezifische Version

#### Environment-Konfiguration:
```json
{
  "DeployToQA": {
    "EnvironmentType": "SaaS",
    "EnvironmentName": "My QA",
    "Branches": ["main", "develop"],
    "Projects": "*",
    "SyncMode": "Add",
    "Scope": "PTE",
    "BuildMode": "Default",
    "ContinuousDeployment": false,
    "DependencyInstallMode": "install",
    "includeTestAppsInSandboxEnvironment": false,
    "excludeAppIds": [],
    "runs-on": "windows-latest",
    "shell": "powershell"
  }
}
```

#### Custom Deployment (OnPrem):
```json
{
  "DeployToOnPrem": {
    "EnvironmentType": "OnPrem",
    "Branches": ["main"]
  }
}
```
Benötigt: `.github/DeployToOnPrem.ps1` Script

### Permissions
```yaml
permissions:
  actions: read
  contents: read
  id-token: write
```

### Environment Variables
```yaml
env:
  ALGoOrgSettings: ${{ vars.ALGoOrgSettings }}
  ALGoRepoSettings: ${{ vars.ALGoRepoSettings }}
```

### Jobs und Steps

#### Job 1: Initialization
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** Keine

**Outputs:**
- `environmentsMatrixJson`: Environments als Matrix
- `environmentCount`: Anzahl Environments
- `deploymentEnvironmentsJson`: Environment-Details
- `deviceCode`: Device Code für Auth (falls benötigt)
- `telemetryScopeJson`: Telemetrie-Scope

**Steps:**
1. **Dump Workflow Information**
   - Action: `microsoft/AL-Go-Actions/DumpWorkflowInfo@v6.2`

2. **Checkout**
   - Action: `actions/checkout@v4.2.2`

3. **Initialize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowInitialize@v6.2`

4. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`

5. **Determine Deployment Environments**
   - Action: `microsoft/AL-Go-Actions/DetermineDeploymentEnvironments@v6.2`
   - Parameter:
     - `getEnvironments`: Input environmentName
     - `type`: 'Publish'
   - Funktion:
     - Matched Environment-Namen gegen Pattern
     - Filtert nach erlaubten Branches
     - Erstellt Deployment-Matrix

6. **EnvName** (Conditional)
   - Bedingung: `if: steps.DetermineDeploymentEnvironments.outputs.UnknownEnvironment == 1`
   - Funktion: Extrahiert Environment-Namen für unbekannte Environments

7. **Read secrets** (Conditional)
   - Bedingung: `if: steps.DetermineDeploymentEnvironments.outputs.UnknownEnvironment == 1`
   - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
   - Parameter: `getSecrets: '${{ steps.envName.outputs.envName }}-AuthContext,${{ steps.envName.outputs.envName }}_AuthContext,AuthContext'`
   - Funktion: Sucht AuthContext Secret mit verschiedenen Namenskonventionen

8. **Authenticate** (Conditional)
   - Bedingung: `if: steps.DetermineDeploymentEnvironments.outputs.UnknownEnvironment == 1`
   - Funktion:
     - Prüft ob AuthContext Secret existiert
     - Falls nicht: Initiiert Device Code Flow
     - Output: Device Code für manuelle Authentifizierung

#### Job 2: Deploy (Matrix)
**Läuft auf:** ${{ matrix.os }} (aus Matrix)  
**Abhängigkeiten:** `Initialization`  
**Bedingung:** `if: needs.Initialization.outputs.environmentCount > 0`  
**Strategy:** Matrix aus `environmentsMatrixJson`

**Environment:**
- name: ${{ matrix.environment }}
- url: ${{ steps.Deploy.outputs.environmentUrl }}

**Matrix-Dimensionen:**
- `environment`: Environment-Name
- `os`: Runner OS
- `shell`: Shell-Typ
- Plus alle DeployTo<Env> Settings

**Steps:**
1. **Checkout**
   - Action: `actions/checkout@v4.2.2`

2. **EnvName**
   - Funktion: Extrahiert Environment-Namen

3. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`
   - Parameter: `get: type,powerPlatformSolutionFolder`

4. **Read secrets**
   - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
   - Parameter: `getSecrets: '${{ steps.envName.outputs.envName }}-AuthContext,${{ steps.envName.outputs.envName }}_AuthContext,AuthContext'`

5. **Determine Projects**
   - Action: `microsoft/AL-Go-Actions/DetermineProjectsToBuild@v6.2`
   - Funktion: Filtert Projekte basierend auf Environment-Konfiguration

6. **Download Artifacts**
   - Funktion:
     - Sucht Artifacts basierend auf `appVersion` Input
     - Lädt Apps und Dependencies herunter
     - Entpackt Artifacts

7. **Deploy to Environment**
   - Action: `microsoft/AL-Go-Actions/Deploy@v6.2`
   - Parameter:
     - `environmentName`: Matrix environment
     - `artifacts`: Heruntergeladene Apps
     - `type`: 'Publish'
     - `deploymentEnvironmentsJson`: Environment-Details
   - Funktion:
     - Authentifiziert gegen BC Environment
     - Installiert/Upgraded Dependencies
     - Published/Upgrades Apps
     - Führt Sync durch (Add/ForceSync/Development/Clean)
     - Gibt Environment-URL aus

8. **Deploy Power Platform Solution** (Conditional)
   - Bedingung: `if: env.type == 'PTE' && env.powerPlatformSolutionFolder != ''`
   - Action: `microsoft/AL-Go-Actions/DeployPowerPlatform@v6.2`
   - Funktion: Deployed Power Platform Solution auf konfiguriertes Environment

#### Job 3: PostProcess
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** `Initialization`, `Deploy`  
**Bedingung:** `if: (!cancelled())`

**Steps:**
1. **Finalize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowPostProcess@v6.2`

---

## 6. Create App

### Workflow-Name
`Create a new app` (`.github/workflows/CreateApp.yaml`)

### Beschreibung
Erstellt eine neue AL App im Repository mit korrekter Ordnerstruktur, app.json und optional Sample-Code.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `project` | string | Nein | '.' | Projektname (bei Multi-Project Repos) |
| `name` | string | Ja | - | App-Name |
| `publisher` | string | Ja | - | Publisher-Name |
| `idrange` | string | Ja | - | ID-Range (z.B. "50000..50099") |
| `sampleCode` | boolean | Nein | true | Sample Code inkludieren |
| `directCommit` | boolean | Nein | false | Direkter Commit statt PR |
| `useGhTokenWorkflow` | boolean | Nein | false | GhTokenWorkflow Secret verwenden |

### Notwendige Settings

#### GitHub Secrets:
- `GhTokenWorkflow` (oder `TokenForPush`): PAT für Commit/PR

#### AL-Go-Settings.json:
```json
{
  "type": "PTE",
  "country": "de",
  "appFolders": ["src"]
}
```

### Anpassungsmöglichkeiten

#### PSG Naming Conventions (aus copilot-instructions.md):
```
Namespace: PSG.Customization.<CustomerName>
Object Prefix: PSG_
ID Range: 50900..50999 (für Kunden-Entwicklungen)
```

#### App-Struktur anpassen:
Nach Erstellung manuell anpassen:
- app.json: Logo, Description, etc.
- Ordnerstruktur nach Bedarf organisieren

#### Sample Code deaktivieren:
Input: `sampleCode: false`  
Resultat: Leere App-Struktur ohne Beispiel-Code

#### Direct Commit:
Input: `directCommit: true`  
Resultat: Keine PR, direkter Commit

### Permissions
```yaml
permissions:
  actions: read
  contents: write
  id-token: write
  pull-requests: write
```

### Environment Variables
```yaml
env:
  ALGoOrgSettings: ${{ vars.ALGoOrgSettings }}
  ALGoRepoSettings: ${{ vars.ALGoRepoSettings }}
```

### Jobs und Steps

#### Job 1: CreateApp
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** Keine

**Steps:**
1. **Dump Workflow Information**
   - Action: `microsoft/AL-Go-Actions/DumpWorkflowInfo@v6.2`

2. **Checkout**
   - Action: `actions/checkout@v4.2.2`

3. **Initialize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowInitialize@v6.2`

4. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`
   - Parameter: `get: type`

5. **Read secrets**
   - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
   - Parameter: `getSecrets: 'TokenForPush'`

6. **Creating a new app**
   - Action: `microsoft/AL-Go-Actions/CreateApp@v6.2`
   - Parameter:
     - `token`: Token für Commit/PR
     - `project`: Input project
     - `type`: env.type (PTE oder AppSource App)
     - `name`: Input name
     - `publisher`: Input publisher
     - `idrange`: Input idrange
     - `sampleCode`: Input sampleCode
     - `directCommit`: Input directCommit
   - Funktion:
     - Erstellt neuen Ordner in appFolders
     - Generiert app.json mit Metadaten
     - Erstellt HelloWorld.al (bei sampleCode=true)
     - Erstellt .alpackages Ordner
     - Committed Changes oder erstellt PR

7. **Finalize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowPostProcess@v6.2`
   - Bedingung: `if: always()`

**Generierte Struktur:**
```
<project>/
  <appFolders>/
    <name>/
      app.json
      HelloWorld.al (optional)
      .alpackages/
```

**app.json Inhalt:**
```json
{
  "id": "<generated-guid>",
  "name": "<input-name>",
  "publisher": "<input-publisher>",
  "version": "1.0.0.0",
  "brief": "",
  "description": "",
  "privacyStatement": "",
  "EULA": "",
  "help": "",
  "url": "",
  "logo": "",
  "dependencies": [],
  "screenshots": [],
  "platform": "<from-artifact>",
  "application": "<from-artifact>",
  "idRanges": [
    {
      "from": <input-idrange-from>,
      "to": <input-idrange-to>
    }
  ],
  "resourceExposurePolicy": {
    "allowDebugging": true,
    "allowDownloadingSource": false,
    "includeSourceInSymbolFile": false
  },
  "runtime": "<from-artifact>",
  "features": ["NoImplicitWith"]
}
```

---

## 7. Create Test App

### Workflow-Name
`Create a new test app` (`.github/workflows/CreateTestApp.yaml`)

### Beschreibung
Erstellt eine neue Test App für automatisierte Tests mit Test Framework Dependencies.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `project` | string | Nein | '.' | Projektname |
| `name` | string | Ja | '\<YourAppName>.Test' | Test-App-Name |
| `publisher` | string | Ja | - | Publisher-Name |
| `idrange` | string | Ja | '50000..99999' | ID-Range |
| `sampleCode` | boolean | Nein | true | Sample Test Code inkludieren |
| `directCommit` | boolean | Nein | false | Direkter Commit statt PR |
| `useGhTokenWorkflow` | boolean | Nein | false | GhTokenWorkflow Secret verwenden |

### Notwendige Settings

#### GitHub Secrets:
- `GhTokenWorkflow` (oder `TokenForPush`): PAT für Commit/PR

#### AL-Go-Settings.json:
```json
{
  "testFolders": ["test"],
  "installTestFramework": true,
  "installTestLibraries": true
}
```

### Anpassungsmöglichkeiten

#### Test Framework automatisch erkennen:
Settings nicht setzen - AL-Go erkennt automatisch Dependencies

#### Spezifische Test Framework Version:
```json
{
  "installTestFramework": true,
  "installTestLibraries": true,
  "installApps": [],
  "installTestApps": [
    "https://url-to-specific-test-framework-version.app"
  ]
}
```

#### Test-Ausführung kontrollieren:
```json
{
  "doNotBuildTests": false,
  "doNotRunTests": false
}
```

### Permissions
```yaml
permissions:
  actions: read
  contents: write
  id-token: write
  pull-requests: write
```

### Jobs und Steps

#### Job 1: CreateTestApp
**Läuft auf:** `windows-latest`

**Steps:**
1. **Dump Workflow Information**
2. **Checkout**
3. **Initialize the workflow**
4. **Read settings**
5. **Read secrets**

6. **Creating a new test app**
   - Action: `microsoft/AL-Go-Actions/CreateApp@v6.2`
   - Parameter:
     - `type`: 'Test App'
     - Andere Parameter wie bei CreateApp
   - Funktion:
     - Erstellt Test-App in testFolders
     - Fügt Test Framework Dependencies zu app.json hinzu
     - Erstellt Sample-Test (bei sampleCode=true)

7. **Finalize the workflow**

**Generierte app.json Dependencies:**
```json
{
  "dependencies": [
    {
      "id": "...",
      "name": "Library Assert",
      "publisher": "Microsoft",
      "version": "..."
    },
    {
      "id": "...",
      "name": "Test Runner",
      "publisher": "Microsoft",
      "version": "..."
    }
  ]
}
```

**Sample Test Code:**
```al
codeunit 50000 "My Test"
{
    Subtype = Test;

    [Test]
    procedure TestExample()
    var
        Assert: Codeunit "Library Assert";
    begin
        Assert.AreEqual(1, 1, 'Test failed');
    end;
}
```

---

## 8. Create Performance Test App

### Workflow-Name
`Create a new performance test app` (`.github/workflows/CreatePerformanceTestApp.yaml`)

### Beschreibung
Erstellt eine BCPT (Business Central Performance Toolkit) App für Performance-Tests.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `project` | string | Nein | '.' | Projektname |
| `name` | string | Ja | '\<YourAppName>.PerformanceTest' | BCPT App-Name |
| `publisher` | string | Ja | - | Publisher-Name |
| `idrange` | string | Ja | '50000..99999' | ID-Range |
| `sampleCode` | boolean | Nein | true | Sample Code inkludieren |
| `sampleSuite` | boolean | Nein | true | Sample BCPT Suite inkludieren |
| `directCommit` | boolean | Nein | false | Direkter Commit statt PR |
| `useGhTokenWorkflow` | boolean | Nein | false | GhTokenWorkflow Secret verwenden |

### Notwendige Settings

#### GitHub Secrets:
- `GhTokenWorkflow` (oder `TokenForPush`): PAT

#### AL-Go-Settings.json:
```json
{
  "bcptTestFolders": ["bcptTest"],
  "bcptThresholds": {
    "DurationWarning": 10,
    "DurationError": 25,
    "NumberOfSqlStmtsWarning": 5,
    "NumberOfSqlStmtsError": 10
  },
  "installPerformanceToolkit": true
}
```

### Anpassungsmöglichkeiten

#### BCPT Thresholds anpassen:
```json
{
  "bcptThresholds": {
    "DurationWarning": 5,
    "DurationError": 15,
    "NumberOfSqlStmtsWarning": 3,
    "NumberOfSqlStmtsError": 8
  }
}
```

#### BCPT Tests deaktivieren:
```json
{
  "doNotRunBcptTests": true
}
```

### Permissions
```yaml
permissions:
  actions: read
  contents: write
  id-token: write
  pull-requests: write
```

### Jobs und Steps

#### Job 1: CreatePerformanceTestApp
**Läuft auf:** `windows-latest`

**Steps:**
1-5: Wie bei CreateTestApp

6. **Creating a new performance test app**
   - Action: `microsoft/AL-Go-Actions/CreateApp@v6.2`
   - Parameter:
     - `type`: 'Performance Test App'
     - `sampleSuite`: Input sampleSuite
   - Funktion:
     - Erstellt BCPT App in bcptTestFolders
     - Fügt Performance Toolkit Dependencies hinzu
     - Erstellt Sample BCPT Codeunit und Suite

7. **Finalize the workflow**

**Sample BCPT Codeunit:**
```al
codeunit 50000 "BCPT Create Sales Order"
{
    SingleInstance = true;

    trigger OnRun()
    begin
        // Performance test logic
    end;
}
```

**Sample BCPT Suite:**
```al
codeunit 50001 "BCPT Suite"
{
    Subtype = TestRunner;

    trigger OnRun()
    var
        BCPTSuiteHdr: Record "BCPT Header";
    begin
        BCPTSuiteHdr.Init();
        BCPTSuiteHdr."Code" := 'MYSUITE';
        BCPTSuiteHdr.Description := 'My Performance Test Suite';
        BCPTSuiteHdr.Insert();
    end;
}
```

---

## 9. Add Existing App or Test App

### Workflow-Name
`Add existing app or test app` (`.github/workflows/AddExistingAppOrTestApp.yaml`)

### Beschreibung
Importiert eine existierende App (.app oder .zip) aus einer Direct Download URL ins Repository.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `project` | string | Nein | '.' | Projektname |
| `url` | string | Ja | - | Direct Download URL (.app oder .zip) |
| `directCommit` | boolean | Nein | false | Direkter Commit statt PR |
| `useGhTokenWorkflow` | boolean | Nein | false | GhTokenWorkflow Secret verwenden |

### Notwendige Settings

#### GitHub Secrets:
- `GhTokenWorkflow` (oder `TokenForPush`): PAT

#### AL-Go-Settings.json:
```json
{
  "appFolders": ["src"],
  "testFolders": ["test"]
}
```

### Anpassungsmöglichkeiten

#### Aus GitHub Release importieren:
```
url: https://github.com/owner/repo/releases/download/v1.0.0/MyApp.app
```

#### Aus Azure Blob Storage:
```
url: https://mystorageaccount.blob.core.windows.net/mycontainer/MyApp.app?SASToken
```

#### Aus privater URL mit Secret:
Settings:
```json
{
  "installApps": [
    "${{MyAppUrl}}"
  ]
}
```
Secret: `MyAppUrl` = Private URL

### Permissions
```yaml
permissions:
  actions: read
  contents: write
  id-token: write
  pull-requests: write
```

### Jobs und Steps

#### Job 1: AddExistingAppOrTestApp
**Läuft auf:** `windows-latest`

**Steps:**
1. **Dump Workflow Information**
2. **Checkout**
3. **Initialize the workflow**
4. **Read settings**
5. **Read secrets**

6. **Download and Add Existing App**
   - Action: `microsoft/AL-Go-Actions/AddExistingApp@v6.2`
   - Parameter:
     - `token`: Token für Commit/PR
     - `project`: Input project
     - `url`: Input url
     - `directCommit`: Input directCommit
   - Funktion:
     - Lädt .app oder .zip von URL herunter
     - Entpackt .zip falls notwendig
     - Extrahiert App-Metadaten aus .app
     - Bestimmt ob Test App (basierend auf Dependencies)
     - Erstellt Ordner in appFolders oder testFolders
     - Extrahiert Source-Code aus .app (wenn verfügbar)
     - Erstellt app.json
     - Committed oder erstellt PR

7. **Finalize the workflow**

**Prozess-Details:**
1. Download von URL
2. Validierung der .app Datei
3. Extraktion der Metadaten
4. Erkennung App-Typ (App/Test App)
5. Ordner-Erstellung
6. Source-Code-Extraktion (falls verfügbar in .app)
7. Commit/PR-Erstellung

---

## 10. Create Online Development Environment

### Workflow-Name
`Create Online Dev. Environment` (`.github/workflows/CreateOnlineDevelopmentEnvironment.yaml`)

### Beschreibung
Erstellt eine BC Online Development Environment und konfiguriert VS Code für Remote-Entwicklung.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `project` | string | Nein | '.' | Projektname |
| `environmentName` | string | Ja | - | Name der Online-Umgebung |
| `reUseExistingEnvironment` | boolean | Nein | false | Existierende Umgebung wiederverwenden |
| `directCommit` | boolean | Nein | false | Direkter Commit statt PR |
| `useGhTokenWorkflow` | boolean | Nein | false | GhTokenWorkflow Secret verwenden |

### Notwendige Settings

#### GitHub Secrets:
- `AdminCenterApiCredentials`: Admin Center API Credentials
  ```json
  {
    "clientId": "...",
    "clientSecret": "...",
    "tenantId": "..."
  }
  ```
  Oder:
  ```json
  {
    "refreshToken": "..."
  }
  ```
- `GhTokenWorkflow`: Für launch.json Update

#### AL-Go-Settings.json:
```json
{
  "adminCenterApiCredentialsSecretName": "AdminCenterApiCredentials",
  "country": "de"
}
```

### Anpassungsmöglichkeiten

#### Secret-Name anpassen:
```json
{
  "adminCenterApiCredentialsSecretName": "MyAdminCenterCreds"
}
```

#### Environment automatisch wiederverwenden:
Input: `reUseExistingEnvironment: true`  
Resultat: Verwendet existierende Environment, falls vorhanden

#### Spezifische BC-Version:
```json
{
  "artifact": "*/20.0/*/latest"
}
```

### Permissions
```yaml
permissions:
  actions: read
  contents: write
  id-token: write
  pull-requests: write
```

### Jobs und Steps

#### Job 1: Initialization
**Läuft auf:** `windows-latest`

**Outputs:**
- `deviceCode`: Device Code für manuelle Auth (falls nötig)
- `githubRunner`: Runner-Konfiguration
- `githubRunnerShell`: Shell-Typ
- `telemetryScopeJson`: Telemetrie

**Steps:**
1. **Dump Workflow Information**
2. **Checkout**
3. **Initialize the workflow**
4. **Read settings**

5. **Read secrets**
   - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
   - Parameter: `getSecrets: 'adminCenterApiCredentials,TokenForPush'`

6. **Authenticate** (Conditional)
   - Funktion:
     - Prüft adminCenterApiCredentials
     - Falls nicht vorhanden: Device Code Flow
     - Output: Device Code für User

#### Job 2: CreateEnvironment
**Läuft auf:** ${{ needs.Initialization.outputs.githubRunner }}  
**Abhängigkeiten:** `Initialization`  
**Environment:** deviceCode aus Initialization

**Steps:**
1. **Checkout**

2. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`
   - Parameter: `get: type`

3. **Read secrets**
   - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
   - Parameter: `getSecrets: 'adminCenterApiCredentials,TokenForPush'`

4. **Create Development Environment**
   - Action: `microsoft/AL-Go-Actions/CreateDevelopmentEnvironment@v6.2`
   - Parameter:
     - `type`: 'Online'
     - `environmentName`: Input environmentName
     - `reUseExistingEnvironment`: Input reUseExistingEnvironment
     - `project`: Input project
     - `adminCenterApiCredentials`: Secret
   - Funktion:
     - Authentifiziert gegen Admin Center API
     - Prüft existierende Environment (bei reUse=true)
     - Erstellt neue Environment (bei reUse=false oder nicht vorhanden)
     - Wartet auf Environment-Bereitschaft
     - Installed Apps auf Environment
     - Konfiguriert VS Code launch.json
     - Committed launch.json Update

5. **Finalize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowPostProcess@v6.2`

**Environment-Konfiguration:**
- Name: Input environmentName
- Type: Sandbox
- Country: Aus Settings
- Application Version: Aus artifact Setting
- Apps: Alle Apps aus Projekt deployed

**VS Code launch.json Update:**
```json
{
  "type": "al",
  "request": "launch",
  "name": "<environmentName>",
  "server": "https://businesscentral.dynamics.com",
  "serverInstance": "<tenant>/<environmentName>",
  "authentication": "AAD",
  "startupObjectId": 22,
  "startupObjectType": "Page",
  "breakOnError": true,
  "launchBrowser": true,
  "enableLongRunningSqlStatements": true,
  "enableSqlInformationDebugger": true
}
```

---

## 11. Current / Next Minor / Next Major

### Workflow-Name
- `Test Current` (`.github/workflows/Current.yaml`)
- `Test Next Minor` (`.github/workflows/NextMinor.yaml`)
- `Test Next Major` (`.github/workflows/NextMajor.yaml`)

### Beschreibung
Test-Workflows für verschiedene BC-Versionen. Ermöglichen Kompatibilitätstests gegen aktuelle und zukünftige BC-Releases.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

**Konfigurierbar:**
- Zeitgesteuert über `workflowSchedule`

### Input-Parameter

Keine

### Notwendige Settings

#### GitHub Secrets:
- Wie bei CI/CD Workflow

#### Workflow-spezifische Settings:

**Current.settings.json:**
```json
{
  "artifact": "*/current/*/*/latest",
  "skipUpgrade": false
}
```

**NextMinor.settings.json:**
```json
{
  "artifact": "*/nextminor/*/*/latest",
  "skipUpgrade": true
}
```

**NextMajor.settings.json:**
```json
{
  "artifact": "*/nextmajor/*/*/latest",
  "skipUpgrade": true
}
```

### Anpassungsmöglichkeiten

#### Zeitgesteuerte Ausführung (wöchentlich):
```json
{
  "workflowSchedule": {
    "cron": "0 2 * * 1"
  }
}
```

#### Spezifische Version testen:
```json
{
  "artifact": "*/22.0/*/*/latest"
}
```

#### Upgrade-Tests aktivieren/deaktivieren:
```json
{
  "skipUpgrade": false
}
```

#### Zusätzliche Länder testen:
```json
{
  "additionalCountries": ["us", "fr", "de"]
}
```

### Permissions
```yaml
permissions:
  actions: read
  contents: read
  id-token: write
```

### Environment Variables
```yaml
env:
  workflowDepth: 1
  ALGoOrgSettings: ${{ vars.ALGoOrgSettings }}
  ALGoRepoSettings: ${{ vars.ALGoRepoSettings }}
```

### Jobs und Steps

#### Job 1: Initialization
**Läuft auf:** `windows-latest`

**Outputs:**
- `projects`: Zu bauende Projekte
- `projectDependenciesJson`: Dependencies
- `buildOrderJson`: Build-Reihenfolge
- `workflowDepth`: Workflow-Tiefe
- `telemetryScopeJson`: Telemetrie

**Steps:**
1. **Dump Workflow Information**
2. **Checkout** (mit LFS)
3. **Initialize the workflow**

4. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`
   - Parameter: `get: useGitSubmodules`
   - Funktion: Liest workflow-spezifische Settings aus `<Workflow>.settings.json`

5. **Read submodules token** (Conditional)
6. **Checkout Submodules** (Conditional)
7. **Determine Workflow Depth**
8. **Determine Projects To Build**

#### Job 2: Build (Matrix)
**Läuft auf:** Matrix-basiert  
**Abhängigkeiten:** `Initialization`  
**Bedingung:** Wenn Projekte vorhanden

**Aufruf:**
- Wiederverwendbarer Workflow: `./.github/workflows/_BuildALGoProject.yaml`

**Parameter:**
- Gleich wie CI/CD Build-Job
- Verwendet workflow-spezifische Artifacts (Current/NextMinor/NextMajor)

**Besonderheiten:**
- **Current:** Testet mit aktueller BC-Version
  - Führt Upgrade-Tests durch (falls `skipUpgrade: false`)
  - Verwendet production-ähnliche Artifacts
  
- **NextMinor:** Testet mit nächster Minor-Version
  - Identifiziert Breaking Changes frühzeitig
  - Upgrade-Tests meist deaktiviert
  
- **NextMajor:** Testet mit nächster Major-Version
  - Vorbereitung auf große Updates
  - Upgrade-Tests meist deaktiviert

#### Job 3: PostProcess
**Läuft auf:** `windows-latest`  
**Abhängigkeiten:** `Initialization`, `Build`

**Steps:**
1. **Finalize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowPostProcess@v6.2`

---

## 12. Update AL-Go System Files

### Workflow-Name
`Update AL-Go System Files` (`.github/workflows/UpdateGitHubGoSystemFiles.yaml`)

### Beschreibung
Aktualisiert AL-Go System-Dateien (.github/workflows und .AL-Go Ordner) auf neueste Version vom Template Repository.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

**Konfigurierbar:**
- Zeitgesteuert über `workflowSchedule` (führt dann Direct Commit durch)

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `templateUrl` | string | Nein | '' | Template Repository URL (leer = aktuelles Template) |
| `downloadLatest` | boolean | Nein | true | Neueste Version vom Template laden |
| `directCommit` | boolean | Nein | false | Direkter Commit statt PR |

### Notwendige Settings

#### GitHub Secrets:
- `GhTokenWorkflow`: Personal Access Token mit Workflow-Permissions
  - Scopes: `workflow`, `repo`
  - Benötigt für Änderungen an .github/workflows

#### AL-Go-Settings.json:
```json
{
  "templateUrl": "https://github.com/microsoft/AL-Go-PTE@main",
  "updateALGoSystemFilesEnvironment": "",
  "unusedALGoSystemFiles": [],
  "customALGoFiles": {
    "filesToInclude": [],
    "filesToExclude": []
  }
}
```

### Anpassungsmöglichkeiten

#### Custom Template Repository:
```json
{
  "templateUrl": "https://github.com/myorg/my-al-go-template@main"
}
```

#### Zeitgesteuerte Updates (monatlich, Direct Commit):
Datei: `.github/UpdateGitHubGoSystemFiles.settings.json`
```json
{
  "workflowSchedule": {
    "cron": "0 3 1 * *"
  }
}
```

#### Approval Environment:
```json
{
  "updateALGoSystemFilesEnvironment": "SystemUpdate"
}
```
Erfordert: GitHub Environment "SystemUpdate" mit Approvers

#### Custom Files konfigurieren:
```json
{
  "customALGoFiles": {
    "filesToInclude": [
      ".github/MyCustomWorkflow.yaml",
      ".AL-Go/MyCustomScript.ps1"
    ],
    "filesToExclude": [
      ".github/workflows/Troubleshooting.yaml"
    ]
  }
}
```

#### Unused Files entfernen:
```json
{
  "unusedALGoSystemFiles": [
    ".github/workflows/DeployReferenceDocumentation.yaml"
  ]
}
```

### Permissions
```yaml
permissions:
  actions: read
  contents: read
  id-token: write
```

### Environment Variables
```yaml
env:
  ALGoOrgSettings: ${{ vars.ALGoOrgSettings }}
  ALGoRepoSettings: ${{ vars.ALGoRepoSettings }}
```

### Jobs und Steps

#### Job 1: UpdateALGoSystemFiles
**Läuft auf:** `windows-latest`

**Steps:**
1. **Dump Workflow Information**
   - Action: `microsoft/AL-Go-Actions/DumpWorkflowInfo@v6.2`

2. **Checkout**
   - Action: `actions/checkout@v4.2.2`

3. **Initialize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowInitialize@v6.2`

4. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`
   - Parameter: `get: templateUrl`

5. **Read secrets**
   - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
   - Parameter: `getSecrets: 'ghTokenWorkflow'`
   - Funktion: Liest GhTokenWorkflow Secret

6. **Override templateUrl** (Conditional)
   - Bedingung: `if: github.event.inputs.templateUrl != ''`
   - Funktion: Überschreibt templateUrl aus Settings mit Input

7. **Calculate Input**
   - Funktion:
     - Bei Schedule-Trigger: Setzt directCommit und downloadLatest auf true
     - Bei Manual: Verwendet Input-Werte

8. **Update AL-Go system files**
   - Action: `microsoft/AL-Go-Actions/CheckForUpdates@v6.2`
   - Parameter:
     - `token`: GhTokenWorkflow Secret
     - `downloadLatest`: Berechneter Wert
     - `update`: 'Y'
     - `templateUrl`: Berechneter Wert
     - `directCommit`: Berechneter Wert
   - Funktion:
     - Lädt Template Repository herunter
     - Vergleicht System-Dateien
     - Identifiziert Änderungen
     - Preserved Custom Jobs und Modifications
     - Updated System-Dateien
     - Erstellt PR oder Commit

9. **Finalize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowPostProcess@v6.2`
   - Bedingung: `if: always()`

**Update-Prozess Details:**

1. **Download Template:**
   - Cloned Template Repository
   - Extrahiert System-Dateien

2. **Vergleich:**
   - Vergleicht .github/workflows/*.yaml
   - Vergleicht .AL-Go/*.ps1
   - Identifiziert Custom Jobs (CustomJob-*)
   - Identifiziert Custom Modifications

3. **Merge:**
   - Behält Custom Jobs bei
   - Merged neue Features
   - Preserved Custom Settings in Workflows

4. **Custom Files:**
   - Kopiert filesToInclude aus Custom Template
   - Ignoriert filesToExclude

5. **Commit/PR:**
   - Bei directCommit: Direkter Commit
   - Sonst: Pull Request "Update AL-Go System Files"

**PR-Inhalt:**
```
Title: Update AL-Go System Files

Body:
Updated files:
- .github/workflows/CICD.yaml
- .github/workflows/CreateRelease.yaml
- .AL-Go/cloudDevEnv.ps1

Preserved custom jobs:
- CustomJob-DeployDocs in CICD.yaml

Breaking changes:
- None

Release notes: [Link to AL-Go Release Notes]
```

---

## 13. Deploy Reference Documentation

### Workflow-Name
`Deploy Reference Documentation` (`.github/workflows/DeployReferenceDocumentation.yaml`)

### Beschreibung
Generiert AL-Referenzdokumentation mit AL-Doc und deployed auf GitHub Pages.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

**Automatisch:**
- Nach erfolgreichem CI/CD auf main (bei `continuousDeployment: true`)

### Input-Parameter

Keine

### Notwendige Settings

#### GitHub Repository Settings:
- GitHub Pages aktiviert
- Source: GitHub Actions

#### AL-Go-Settings.json:
```json
{
  "alDoc": {
    "continuousDeployment": false,
    "deployToGitHubPages": true,
    "maxReleases": 3,
    "groupByProject": true,
    "includeProjects": ["*"],
    "excludeProjects": [],
    "header": "Documentation for {REPOSITORY}",
    "footer": "Made with AL-Go for GitHub",
    "defaultIndexMD": "# Welcome to {REPOSITORY} Documentation",
    "defaultReleaseMD": "# {REPOSITORY} {VERSION} Documentation"
  }
}
```

### Anpassungsmöglichkeiten

#### Continuous Deployment aktivieren:
```json
{
  "alDoc": {
    "continuousDeployment": true
  }
}
```

#### Mehr Releases inkludieren:
```json
{
  "alDoc": {
    "maxReleases": 10
  }
}
```

#### Spezifische Projekte:
```json
{
  "alDoc": {
    "includeProjects": ["MainApp", "IntegrationApp"],
    "excludeProjects": ["TestApp"]
  }
}
```

#### Custom Header/Footer:
```json
{
  "alDoc": {
    "header": "# {REPOSITORY} v{VERSION} - API Reference",
    "footer": "© 2024 My Company | Generated {DATE}",
    "defaultIndexMD": "# My Company AL Documentation\n\nWelcome to our API reference."
  }
}
```

**Verfügbare Platzhalter:**
- `{REPOSITORY}`: Repository-Name
- `{VERSION}`: Aktuelle Version
- `{RELEASENOTES}`: Link zu Release Notes
- `{INDEXTEMPLATERELATIVEPATH}`: Relativer Pfad

### Permissions
```yaml
permissions:
  actions: read
  contents: read
  id-token: write
  pages: write
```

### Environment Variables
```yaml
env:
  ALGoOrgSettings: ${{ vars.ALGoOrgSettings }}
  ALGoRepoSettings: ${{ vars.ALGoRepoSettings }}
```

### Jobs und Steps

#### Job 1: DeployALDoc
**Läuft auf:** `windows-latest`

**Environment:**
- name: github-pages
- url: ${{ steps.deployment.outputs.page_url }}

**Steps:**
1. **Checkout**
   - Action: `actions/checkout@v4.2.2`

2. **Initialize the workflow**
   - Action: `microsoft/AL-Go-Actions/WorkflowInitialize@v6.2`

3. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`

4. **Determine Deployment Environments**
   - Action: `microsoft/AL-Go-Actions/DetermineDeploymentEnvironments@v6.2`
   - Parameter: `getEnvironments: 'github-pages'`, `type: 'Publish'`

5. **Determine Projects**
   - Action: `microsoft/AL-Go-Actions/DetermineProjectsToBuild@v6.2`

6. **Download Latest Release Artifacts**
   - Funktion:
     - Lädt Artifacts der letzten X Releases (maxReleases)
     - Entpackt Apps

7. **Build Reference Documentation**
   - Action: `microsoft/AL-Go-Actions/BuildReferenceDocumentation@v6.2`
   - Parameter: alDoc Settings
   - Funktion:
     - Extrahiert AL-Symbole aus Apps
     - Generiert Markdown-Dokumentation
     - Erstellt HTML mit Jekyll
     - Strukturiert nach Projekten/Releases
     - Fügt Header/Footer hinzu

8. **Upload pages artifact**
   - Action: `actions/upload-pages-artifact@v3`
   - Parameter: `path: '_site'`

9. **Deploy to GitHub Pages**
   - Action: `actions/deploy-pages@v4`
   - Funktion: Deployed auf GitHub Pages

10. **Finalize the workflow**
    - Action: `microsoft/AL-Go-Actions/WorkflowPostProcess@v6.2`

**Generierte Dokumentationsstruktur:**
```
/
  index.html (Landing Page)
  /v1.0.0/
    index.html
    /Project1/
      Objects.html
      Codeunits/
        MyCodeunit.html
      Tables/
        MyTable.html
      Pages/
        MyPage.html
    /Project2/
      ...
  /v1.1.0/
    ...
  /v2.0.0/
    ...
```

---

## 14. Troubleshooting

### Workflow-Name
`Troubleshooting` (`.github/workflows/Troubleshooting.yaml`)

### Beschreibung
Diagnose-Workflow zur Fehlersuche und Validierung der AL-Go Konfiguration.

### Auslösemöglichkeiten

**Manuell:**
- Via GitHub Actions UI: `workflow_dispatch`

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `displayNameOfSecrets` | boolean | Nein | false | Zeigt Secret-Namen (nicht Werte) |

### Notwendige Settings

Keine spezifischen Settings erforderlich.

### Anpassungsmöglichkeiten

#### Secret-Namen anzeigen:
Input: `displayNameOfSecrets: true`  
Resultat: Listet alle verfügbaren Secret-Namen auf (ohne Werte zu zeigen)

### Permissions
```yaml
permissions:
  actions: read
  contents: read
```

### Environment Variables
```yaml
env:
  ALGoOrgSettings: ${{ vars.ALGoOrgSettings }}
  ALGoRepoSettings: ${{ vars.ALGoRepoSettings }}
```

### Jobs und Steps

#### Job 1: Troubleshooting
**Läuft auf:** `windows-latest`

**Steps:**
1. **Checkout**
   - Action: `actions/checkout@v4.2.2`
   - Parameter: `lfs: true`

2. **Troubleshooting**
   - Action: `microsoft/AL-Go-Actions/Troubleshooting@v6.2`
   - Parameter:
     - `gitHubSecrets`: ${{ toJson(secrets) }}
     - `displayNameOfSecrets`: Input displayNameOfSecrets
   - Funktion: Führt umfassende Diagnose durch

**Diagnose-Ausgabe:**

1. **System-Information:**
   ```
   AL-Go Version: 6.2
   Template URL: https://github.com/microsoft/AL-Go-PTE@main
   Repository Type: PTE
   ```

2. **Settings-Übersicht:**
   ```
   Merged Settings:
   - country: de
   - appFolders: ["src"]
   - testFolders: ["test"]
   - environments: ["QA", "Production"]
   - buildModes: ["Default", "Clean"]
   ...
   ```

3. **Secrets-Verfügbarkeit:**
   ```
   Available Secrets (bei displayNameOfSecrets=true):
   - GhTokenWorkflow: Available
   - QA_AuthContext: Available
   - Production_AuthContext: Available
   - licenseFileUrl: Not found
   - StorageContext: Available
   ```

4. **Environments:**
   ```
   GitHub Environments:
   - QA:
       Protection Rules: None
       Deployment Branches: main
   - Production:
       Protection Rules: Required reviewers (2)
       Deployment Branches: main
   ```

5. **Projects:**
   ```
   Detected Projects:
   - Project: .
     Apps: 3
     Test Apps: 2
     BCPT Apps: 0
   ```

6. **Build-Konfiguration:**
   ```
   Build Settings:
   - Versioning Strategy: 0
   - Repo Version: 1.0
   - GitHub Runner: windows-latest
   - Shell: powershell
   ```

7. **Artifact-Konfiguration:**
   ```
   Artifacts:
   - Latest: myrepo-main-Apps-1.0.25.0
   - BC Version: 22.0.54157.55210
   - Country: de
   ```

8. **Validierung:**
   ```
   Configuration Validation:
   ✓ Template URL is valid
   ✓ All required secrets are configured
   ✓ Environments are properly configured
   ✓ Project structure is valid
   ⚠ Warning: licenseFileUrl secret not found (required for BC < 22)
   ✗ Error: None
   ```

9. **Empfehlungen:**
   ```
   Recommendations:
   - Consider enabling incremental builds for faster PR builds
   - Update AL-Go system files (v6.1 -> v6.2)
   - Configure alDoc for automatic documentation
   ```

---

## 15. _Build AL-Go Project (Interner Workflow)

### Workflow-Name
`_Build AL-Go project` (`.github/workflows/_BuildALGoProject.yaml`)

### Beschreibung
Wiederverwendbarer interner Workflow für Build-Prozesse. Wird von CI/CD, PR-Handler und Test-Workflows aufgerufen.

### Auslösemöglichkeiten

**Intern:**
- Via `workflow_call` von anderen Workflows

### Input-Parameter

| Parameter | Typ | Required | Default | Beschreibung |
|-----------|-----|----------|---------|--------------|
| `shell` | string | Nein | 'powershell' | Shell (powershell oder pwsh) |
| `runsOn` | string | Ja | - | Runner-Typ (JSON-formatted) |
| `checkoutRef` | string | Nein | ${{ github.sha }} | Git Ref zum Checkout |
| `project` | string | Ja | - | Projekt-Pfad |
| `projectName` | string | Ja | - | Friendly Name |
| `projectDependenciesJson` | string | Nein | '{}' | Dependencies (JSON) |
| `buildMode` | string | Ja | - | Build-Modus |
| `baselineWorkflowRunId` | string | Nein | '0' | Baseline für inkrementelle Builds |
| `secrets` | string | Nein | '' | Komma-separierte Secret-Namen |
| `publishThisBuildArtifacts` | boolean | Ja | - | Diesen Build publishen |
| `publishArtifacts` | boolean | Nein | false | Artifacts generell publishen |
| `signArtifacts` | boolean | Nein | false | Artifacts signieren |
| `useArtifactCache` | boolean | Nein | false | Artifact-Cache verwenden |
| `artifactsNameSuffix` | string | Nein | '' | Suffix für Artifact-Namen |

### Notwendige Settings

Abhängig vom aufrufenden Workflow.

### Permissions
```yaml
permissions:
  actions: read
  contents: read
  id-token: write
```

### Jobs und Steps

#### Job 1: Build
**Läuft auf:** ${{ inputs.runsOn }} (Matrix-basiert)

**Steps:**
1. **Checkout**
   - Action: `actions/checkout@v4.2.2`
   - Parameter:
     - `ref`: inputs.checkoutRef
     - `lfs`: true
     - `submodules`: Bei Bedarf

2. **Read settings**
   - Action: `microsoft/AL-Go-Actions/ReadSettings@v6.2`
   - Parameter: `project: ${{ inputs.project }}`
   - Funktion: Liest projekt-spezifische Settings

3. **Read secrets**
   - Action: `microsoft/AL-Go-Actions/ReadSecrets@v6.2`
   - Parameter: `getSecrets: ${{ inputs.secrets }}`

4. **Determine Artifacts to use** (Conditional)
   - Bedingung: `if: inputs.baselineWorkflowRunId != '0'`
   - Funktion:
     - Bei inkrementellen Builds
     - Lädt Baseline-Artifacts
     - Ermittelt geänderte Apps

5. **Cache Business Central Artifacts** (Conditional)
   - Bedingung: `if: inputs.useArtifactCache`
   - Action: `actions/cache@v3`
   - Funktion: Cached BC-Artifacts für schnellere Builds

6. **Download Project Dependencies** (Conditional)
   - Bedingung: `if: inputs.projectDependenciesJson != '{}'`
   - Funktion:
     - Lädt Dependencies von anderen Projekten
     - Aus vorherigen Build-Jobs oder Baseline

7. **Run Pipeline**
   - Action: `microsoft/AL-Go-Actions/RunPipeline@v6.2`
   - Parameter:
     - `project`: inputs.project
     - `projectName`: inputs.projectName
     - `buildMode`: inputs.buildMode
     - `secrets`: Secrets JSON
   - Funktion: Führt Haupt-Build-Pipeline aus

**Run Pipeline Details:**

1. **Setup:**
   - Lädt BcContainerHelper
   - Liest Settings (merged from all levels)
   - Validiert Konfiguration

2. **Download BC Artifacts:**
   - Ermittelt benötigte BC-Version
   - Lädt BC-Artifacts von Storage
   - Cached für wiederverwendung

3. **Container/Compiler Setup:**
   - Bei useCompilerFolder=false: Erstellt BC-Container
   - Bei useCompilerFolder=true: Erstellt Compiler-Folder
   - Installiert Extensions

4. **Download Dependencies:**
   - Aus appDependencyProbingPaths
   - Aus NuGet Feeds
   - Aus GitHub Packages
   - InstallApps / InstallTestApps

5. **Compile Apps:**
   - Sortiert Apps nach Dependencies
   - Kompiliert in Reihenfolge
   - Für jeden buildMode
   - Erstellt .app Dateien

6. **Publish Apps:**
   - Published Apps auf Container/CompilerFolder
   - Installiert in Dependency-Reihenfolge

7. **Run Tests:**
   - Installiert Test Framework
   - Published Test Apps
   - Führt Tests aus
   - Sammelt Test-Ergebnisse

8. **Run BCPT Tests:**
   - Installiert Performance Toolkit
   - Published BCPT Apps
   - Führt Performance-Tests aus
   - Validiert gegen Thresholds

9. **Code Signing:**
   - Bei signArtifacts=true
   - Signiert .app Dateien
   - Mit Certificate oder Trusted Signing

10. **Artifact Creation:**
    - Packt Apps als .zip
    - Erstellt Dependency-Artifacts
    - Generiert Build-Output

11. **Cleanup:**
    - Entfernt Container/CompilerFolder
    - Cleaned temporäre Dateien

8. **Sign Artifacts** (Conditional)
   - Bedingung: `if: inputs.signArtifacts`
   - Action: `microsoft/AL-Go-Actions/Sign@v6.2`
   - Funktion: Signiert alle .app Dateien

9. **Publish Test Results** (Conditional)
   - Action: `EnricoMi/publish-unit-test-result-action@v2`
   - Funktion: Published Test-Resultate in GitHub

10. **Publish Build Output**
    - Action: `actions/upload-artifact@v4`
    - Funktion:
      - Published Compile-Log
      - Published Test-Resultate
      - Published Error-Log

11. **Publish Artifacts** (Conditional)
    - Bedingung: `if: inputs.publishArtifacts`
    - Action: `actions/upload-artifact@v4`
    - Funktion:
      - Published Apps.zip
      - Published TestApps.zip (optional)
      - Published Dependencies.zip (optional)
      - Name: `${{ inputs.projectName }}-Apps-${{ version }}${{ inputs.artifactsNameSuffix }}`

12. **Cleanup**
    - Action: `microsoft/AL-Go-Actions/Cleanup@v6.2`
    - Funktion: Cleaned Build-Artifacts und Container

---

## Best Practices

### Workflow-Ausführung optimieren

#### 1. Inkrementelle Builds aktivieren
```json
{
  "incrementalBuilds": {
    "onPush": true,
    "onPull_Request": true,
    "retentionDays": 30,
    "mode": "modifiedProjects"
  }
}
```

#### 2. Artifact-Caching nutzen
- Bei Self-Hosted Runners: `cacheImageName` und `cacheKeepDays` setzen
- GitHub-Hosted: Automatisches Caching von BC-Artifacts

#### 3. Workflow-Concurrency konfigurieren
```json
{
  "workflowConcurrency": [
    "group: ${{ github.workflow }}-${{ github.ref }}",
    "cancel-in-progress: true"
  ]
}
```

#### 4. Parallele Builds mit Dependencies
```json
{
  "useProjectDependencies": true
}
```

#### 5. Containerless Compilation
```json
{
  "useCompilerFolder": true,
  "doNotPublishApps": true
}
```

### Security Best Practices

#### 1. Secrets in Azure KeyVault
- Zentrale Secret-Verwaltung
- Automatische Rotation
- Audit-Logs

#### 2. Environment Protection Rules
- Required Reviewers für Production
- Branch Protection
- Deployment Delays

#### 3. Pull Request Security
- `pull_request_target` für Fork-PRs
- Pregate Checks aktiviert
- Secret-Access beschränkt

#### 4. Code Signing
- Alle Production-Apps signieren
- Trusted Signing für AppSource
- Certificate Management in KeyVault

### Troubleshooting-Tipps

#### Workflow schlägt fehl

1. **Troubleshooting Workflow ausführen**
   - Zeigt vollständige Konfiguration
   - Identifiziert fehlende Secrets
   - Validiert Settings

2. **Logs prüfen**
   - Build-Logs in Artifact "Build Output"
   - Error-Logs in "Build Output"
   - Test-Resultate in "Test Results"

3. **Häufige Probleme**
   - Fehlende Secrets
   - Falsche ID-Ranges
   - Dependency-Konflikte
   - Artifact-Expiration

#### Performance-Probleme

1. **Build-Zeit reduzieren**
   - Inkrementelle Builds aktivieren
   - Containerless Compilation
   - Self-Hosted Runner

2. **Artifact-Größe reduzieren**
   - Dependencies nicht inkludieren
   - Test-Apps nur bei Bedarf

3. **Parallel-Builds**
   - useProjectDependencies aktivieren
   - Projekte optimal strukturieren

---

**Letzte Aktualisierung:** April 2026  
**AL-Go Version:** 6.2  
**Dokumentationsversion:** 1.0
