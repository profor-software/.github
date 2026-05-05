# AL-Go Workflows Zusammenfassung

Diese Zusammenfassung beschreibt alle Workflows in diesem Repository, ihre Auslöser (manuell oder automatisch), Anpassungsmöglichkeiten und Funktionen. Die Informationen basieren auf der offiziellen [AL-Go for GitHub Dokumentation](https://github.com/microsoft/AL-Go).

## Inhaltsverzeichnis

- [Automatisch getriggerte Workflows](#automatisch-getriggerte-workflows)
- [Manuell getriggerte Workflows](#manuell-getriggerte-workflows)
- [Interne Workflows](#interne-workflows)
- [Allgemeine Anpassungsmöglichkeiten](#allgemeine-anpassungsmöglichkeiten)
- [Wichtige Settings](#wichtige-settings)
- [Weiterführende Dokumentation](#weiterführende-dokumentation)

---

## Automatisch getriggerte Workflows

### CI/CD (`CICD.yaml`)

**Trigger:** Automatisch durch `workflow_dispatch` (kann auch auf Push/Schedule konfiguriert werden)

**Beschreibung:**
Der Haupt-CI/CD-Workflow führt Build, Test und Deployment-Prozesse für das AL-Go Projekt durch. Er kompiliert Apps, führt Tests aus, erstellt Artifacts und deployt automatisch auf konfigurierte Umgebungen.

**Hauptfunktionen:**
- **Initialization:** Ermittelt Projekte, Dependencies, Build-Reihenfolge, Deployment-Umgebungen
- **Build:** Kompiliert alle Apps und Test-Apps im Projekt
- **Test:** Führt automatische Tests aus (wenn konfiguriert)
- **Deploy:** Automatisches Deployment auf definierte Umgebungen (z.B. Sandbox)
- **Artifact-Erstellung:** Erstellt Build-Artifacts für Apps und Test-Apps
- **Reference Documentation:** Generiert AL-Dokumentation (wenn aktiviert)

**Anpassungsmöglichkeiten:**
- `workflowDepth`: Steuert die Workflow-Tiefe (Standard: 1)
- `CICDPushBranches`: Definiert Branches für automatische CI/CD-Trigger (Standard: `["main", "release/*", "feature/*"]`)
- `buildModes`: Verschiedene Build-Modi (Default, Clean, Translated)
- `environments`: Deployment-Ziele konfigurieren
- `incrementalBuilds`: Aktiviert inkrementelle Builds für schnellere Builds
- `workflowSchedule`: CRON-basierte Zeitplanung für automatische Ausführung

**Workflow-spezifische Settings:**
```json
{
  "CICDPushBranches": ["main", "release/*", "feature/*"],
  "workflowSchedule": {
    "cron": "0 2 * * *"
  }
}
```

**Weitere Informationen:**
- [CI/CD Settings](https://aka.ms/algosettings)
- [Continuous Deployment](https://freddysblog.com/2022/05/06/deployment-strategies-and-al-go-for-github/)

---

### Pull Request Handler (`PullRequestHandler.yaml`)

**Trigger:** Automatisch bei Pull Requests auf Branch `main`

**Beschreibung:**
Führt automatische Build- und Test-Validierungen für Pull Requests durch. Stellt sicher, dass Änderungen keine Build-Fehler verursachen, bevor sie in den Hauptbranch gemergt werden.

**Hauptfunktionen:**
- **PregateCheck:** Überprüft PR-Änderungen auf Sicherheitsrisiken
- **Initialization:** Ermittelt betroffene Projekte
- **Build:** Kompiliert nur geänderte Projekte (oder alle, je nach Konfiguration)
- **Test:** Führt Tests auf geänderten Apps aus
- **Concurrency Control:** Verhindert parallele Builds für denselben PR

**Anpassungsmöglichkeiten:**
- `CICDPullRequestBranches`: Definiert Branches, die PR-Builds triggern (Standard: `["main"]`)
- `pullRequestTrigger`: Definiert den Trigger-Typ (`pull_request` oder `pull_request_target`)
- `alwaysBuildAllProjects`: Erzwingt Build aller Projekte bei jedem PR (deprecated)
- `fullBuildPatterns`: Definiert Dateien/Ordner, die Full-Build bei Änderung triggern
- `incrementalBuilds.onPull_Request`: Aktiviert inkrementelle Builds für PRs (Standard: true)

**Workflow-spezifische Settings:**
```json
{
  "CICDPullRequestBranches": ["main", "develop"],
  "pullRequestTrigger": "pull_request",
  "fullBuildPatterns": ["Build/*", ".AL-Go/*"]
}
```

**Sicherheitshinweis:**
Bei `pull_request_target` sind Secrets für Fork-PRs verfügbar, aber es gelten besondere [Sicherheitsüberlegungen](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#pull_request_target).

---

## Manuell getriggerte Workflows

### Create Release (`CreateRelease.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Erstellt einen formalen Release der App mit Versionierung, Release Notes und optionalem Release-Branch. Der Workflow promoted vorhandene Build-Artifacts zu einem Release.

**Hauptfunktionen:**
- **Release-Erstellung:** Erstellt GitHub Release mit Tag
- **Versioning:** Setzt Release-Version und Tag
- **Release Notes:** Auto-generiert Release Notes aus Commits
- **Release Branch:** Optional Release-Branch erstellen
- **Version Increment:** Automatisches Inkrementieren der Version im Hauptbranch nach Release
- **Artifacts:** Publiziert Apps und Source Code als Release-Artifacts

**Input-Parameter:**
- `appVersion`: App-Version für Release (latest, prerelease, draft, oder spezifische Version)
- `name`: Name des Release
- `tag`: Semantic Version Tag (z.B. "1.0.0")
- `prerelease`: Als Pre-Release markieren
- `draft`: Als Draft erstellen
- `createReleaseBranch`: Release-Branch erstellen
- `releaseBranchPrefix`: Präfix für Release-Branch (Standard: "release/")
- `updateVersionNumber`: Neue Versionsnummer im Main-Branch (z.B. "+0.1")
- `directCommit`: Direkter Commit ohne PR
- `useGhTokenWorkflow`: Verwendet GhTokenWorkflow Secret für PR/Commit

**Anpassungsmöglichkeiten:**
- `deliveryTargets`: Definiert Delivery-Ziele für Release
- `appSourceContextSecretName`: Secret für AppSource-Upload
- `deliverToAppSource`: Konfiguration für AppSource-Delivery
- `generateDependencyArtifact`: Inkludiert Dependencies im Release

**Workflow-Beispiel:**
```bash
Name: "v1.0"
Tag: "1.0.0"
Update Version Number: "+0.1"
```

**Weitere Informationen:**
- [Create Release Scenario](https://aka.ms/algoscenario/createrelease)
- [Versioning Strategy](https://aka.ms/algosettings#versioningStrategy)

---

### Increment Version Number (`IncrementVersionNumber.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Inkrementiert die Versionsnummer in den app.json Dateien aller oder ausgewählter Projekte.

**Hauptfunktionen:**
- **Version Update:** Aktualisiert Major.Minor Version in app.json
- **Selective Update:** Kann auf spezifische Projekte beschränkt werden
- **Commit Options:** Direct Commit oder Pull Request

**Input-Parameter:**
- `projects`: Komma-separierte Liste von Projekt-Patterns (Standard: "*" für alle)
- `versionNumber`: Neue Versionsnummer (z.B. "2.0" absolut oder "+0.1" inkrementell)
- `directCommit`: Direkter Commit ohne PR
- `useGhTokenWorkflow`: Verwendet GhTokenWorkflow Secret

**Anpassungsmöglichkeiten:**
- `versioningStrategy`: Definiert die Versioning-Strategie (0, 2, 3, 15, +16)
- `repoVersion`: Repository Version für Artifacts
- `commitOptions`: Steuert PR- und Commit-Verhalten

**Versioning Strategies:**
- **0:** Build = GitHub run_number, Revision = run_attempt
- **2:** Build = yyyyMMdd, Revision = hhmmss (UTC)
- **3:** Build aus app.json, Revision = run_number
- **15:** Build = max value, Revision = run_number (nur für non-official builds)
- **+16:** Verwendet repoVersion statt app.json Major.Minor

**Weitere Informationen:**
- [Versioning Strategy](https://aka.ms/algosettings#versioningStrategy)

---

### Publish To Environment (`PublishToEnvironment.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Deployed eine spezifische App-Version auf eine oder mehrere Business Central Umgebungen.

**Hauptfunktionen:**
- **Environment Selection:** Deployment auf spezifische Umgebungen (mit Wildcard-Support)
- **Version Selection:** Wählt spezifische App-Version (current, prerelease, draft, latest, oder Version)
- **Authentication:** Unterstützt verschiedene Auth-Mechanismen (S2S, RefreshToken)
- **Deployment Validation:** Validiert Deployment vor Ausführung

**Input-Parameter:**
- `appVersion`: Version zum Deployen (current, prerelease, draft, latest, oder Versionsnummer)
- `environmentName`: Environment-Maske (z.B. "PROD*" für alle PROD-Umgebungen)

**Anpassungsmöglichkeiten:**
- `environments`: Array von Environment-Namen
- `DeployTo<environmentname>`: Spezifische Environment-Konfiguration
  - `EnvironmentType`: SaaS, OnPrem, oder Custom
  - `Branches`: Erlaubte Branches für Deployment
  - `Projects`: Projekte für Deployment
  - `SyncMode`: ForceSync, Add, Development, Clean
  - `Scope`: Dev oder PTE
  - `ContinuousDeployment`: Automatisches Deployment aktivieren
  - `DependencyInstallMode`: install, ignore, upgrade, forceUpgrade

**Environment-Konfiguration Beispiel:**
```json
{
  "environments": ["QA", "PRODUCTION"],
  "DeployToQA": {
    "EnvironmentType": "SaaS",
    "Branches": ["main", "release/*"],
    "SyncMode": "Add",
    "ContinuousDeployment": true
  },
  "DeployToPRODUCTION": {
    "EnvironmentType": "SaaS",
    "Branches": ["main"],
    "SyncMode": "ForceSync",
    "ContinuousDeployment": false
  }
}
```

**Weitere Informationen:**
- [Register Sandbox Environment](https://aka.ms/algoscenario/registersandboxenvironment)
- [Register Production Environment](https://aka.ms/algoscenario/registerproductionenvironment)

---

### Create App (`CreateApp.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Erstellt eine neue AL App im Repository mit der korrekten Ordnerstruktur und app.json.

**Hauptfunktionen:**
- **App Generation:** Erstellt neue App-Struktur
- **Configuration:** Setzt Name, Publisher, ID Range
- **Sample Code:** Optional Beispiel-Code hinzufügen
- **Commit/PR:** Direkter Commit oder Pull Request

**Input-Parameter:**
- `project`: Projektname (bei Multi-Project Repos)
- `name`: App-Name
- `publisher`: Publisher-Name
- `idrange`: ID-Range (z.B. "50000..50099")
- `sampleCode`: Sample Code inkludieren
- `directCommit`: Direkter Commit ohne PR
- `useGhTokenWorkflow`: Verwendet GhTokenWorkflow Secret

**Anpassungsmöglichkeiten:**
- `appFolders`: Definiert App-Ordner im Projekt
- `idrange`: Wird validiert gegen verfügbare Ranges
- Namenskonventionen gemäß `.github/copilot-instructions.md`

**Naming Conventions (PSG):**
- Object Prefix: "PSG_" (z.B. PSG_Customer)
- Namespace: "PSG.Customization." für Kunden-Projekte
- File Names: Ohne Underscore

**Weitere Informationen:**
- [Add Test App Scenario](https://aka.ms/algoscenario/addatestapp)

---

### Create Test App (`CreateTestApp.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Erstellt eine neue Test App für automatisierte Tests der Business Central Extension.

**Hauptfunktionen:**
- **Test App Generation:** Erstellt Test-App-Struktur
- **Test Framework:** Konfiguriert Test-Framework Dependencies
- **Sample Tests:** Optional Beispiel-Tests hinzufügen

**Input-Parameter:**
- `project`: Projektname (bei Multi-Project Repos)
- `name`: Test-App-Name (Standard: "\<YourAppName>.Test")
- `publisher`: Publisher-Name
- `idrange`: ID-Range (Standard: "50000..99999")
- `sampleCode`: Sample Test Code inkludieren
- `directCommit`: Direkter Commit ohne PR
- `useGhTokenWorkflow`: Verwendet GhTokenWorkflow Secret

**Anpassungsmöglichkeiten:**
- `testFolders`: Definiert Test-Ordner im Projekt
- `installTestFramework`: Test Framework installieren (auto-detected)
- `installTestLibraries`: Test Libraries installieren (auto-detected)
- `doNotBuildTests`: Tests nicht bauen
- `doNotRunTests`: Tests nicht ausführen

**Test Framework:**
- Automatische Erkennung von Test Framework Dependencies
- Unterstützt Business Central Test Toolkit
- Integration mit CI/CD für automatische Test-Ausführung

**Weitere Informationen:**
- [Add Test App Scenario](https://aka.ms/algoscenario/addatestapp)

---

### Create Performance Test App (`CreatePerformanceTestApp.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Erstellt eine Performance Test App basierend auf dem Business Central Performance Toolkit (BCPT).

**Hauptfunktionen:**
- **BCPT App Generation:** Erstellt Performance Test App
- **BCPT Suite:** Optional BCPT Test Suite hinzufügen
- **Performance Benchmarks:** Konfiguriert Performance-Schwellenwerte

**Input-Parameter:**
- `project`: Projektname
- `name`: Performance Test App Name (Standard: "\<YourAppName>.PerformanceTest")
- `publisher`: Publisher-Name
- `idrange`: ID-Range (Standard: "50000..99999")
- `sampleCode`: Sample Code inkludieren
- `sampleSuite`: Sample BCPT Suite inkludieren
- `directCommit`: Direkter Commit ohne PR
- `useGhTokenWorkflow`: Verwendet GhTokenWorkflow Secret

**Anpassungsmöglichkeiten:**
- `bcptTestFolders`: Definiert BCPT-Test-Ordner
- `bcptThresholds`: Performance-Schwellenwerte konfigurieren
  - `DurationWarning`: Warnung bei Performance-Degradation (Standard: 10%)
  - `DurationError`: Fehler bei Performance-Degradation (Standard: 25%)
  - `NumberOfSqlStmtsWarning`: Warnung bei SQL-Statement-Anstieg (Standard: 5%)
  - `NumberOfSqlStmtsError`: Fehler bei SQL-Statement-Anstieg (Standard: 10%)
- `doNotRunBcptTests`: BCPT Tests nicht ausführen

**BCPT Thresholds Beispiel:**
```json
{
  "bcptThresholds": {
    "DurationWarning": 10,
    "DurationError": 25,
    "NumberOfSqlStmtsWarning": 5,
    "NumberOfSqlStmtsError": 10
  }
}
```

**Weitere Informationen:**
- [Add Performance Test App](https://aka.ms/algoscenario/addaperformancetestapp)
- [Business Central Performance Toolkit](https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/devenv-performance-toolkit)

---

### Add Existing App or Test App (`AddExistingAppOrTestApp.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Fügt eine existierende App (.app oder .zip) aus einer URL zum Repository hinzu.

**Hauptfunktionen:**
- **App Download:** Lädt App von Direct Download URL
- **Integration:** Integriert App in Projekt-Struktur
- **Dependency Handling:** Behandelt App-Dependencies

**Input-Parameter:**
- `project`: Projektname
- `url`: Direct Download URL der .app oder .zip Datei
- `directCommit`: Direkter Commit ohne PR
- `useGhTokenWorkflow`: Verwendet GhTokenWorkflow Secret

**Anpassungsmöglichkeiten:**
- `installApps`: 3rd-Party Dependency Apps
- `installTestApps`: 3rd-Party Test Dependency Apps
- `appDependencyProbingPaths`: Dependency-Pfade konfigurieren

**Use Cases:**
- Import von Apps aus anderem Repository
- Hinzufügen von 3rd-Party Apps
- Migration von Apps aus anderen Systemen

---

### Create Online Development Environment (`CreateOnlineDevelopmentEnvironment.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Erstellt eine Online Business Central Entwicklungsumgebung direkt aus GitHub.

**Hauptfunktionen:**
- **Environment Creation:** Erstellt BC Online Dev Environment
- **App Deployment:** Deployed aktuelle Apps auf Environment
- **VS Code Integration:** Konfiguriert VS Code für Remote-Entwicklung
- **Reuse:** Kann existierende Environment wiederverwenden

**Input-Parameter:**
- `project`: Projektname
- `environmentName`: Name der Online-Umgebung
- `reUseExistingEnvironment`: Existierende Environment wiederverwenden
- `directCommit`: Direkter Commit ohne PR
- `useGhTokenWorkflow`: Verwendet GhTokenWorkflow Secret

**Anpassungsmöglichkeiten:**
- `adminCenterApiCredentialsSecretName`: Secret für Admin Center API
- Authentication über Service-to-Service oder User-Credentials

**Prerequisites:**
- Business Central Admin Center API Credentials
- Gültige Business Central Lizenz
- Secret `AdminCenterApiCredentials` konfiguriert

**Weitere Informationen:**
- [Create Online Dev Environment from GitHub](https://aka.ms/algoscenario/createonlinedevenv2)
- [Create Online Dev Environment from VS Code](https://aka.ms/algoscenario/createonlinedevenv)

---

### Current / Next Minor / Next Major (`Current.yaml`, `NextMinor.yaml`, `NextMajor.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Test-Workflows für verschiedene Business Central Versionen. Ermöglichen Tests gegen aktuelle, nächste Minor- oder nächste Major-Version von Business Central.

**Hauptfunktionen:**
- **Version Testing:** Testet Apps gegen verschiedene BC-Versionen
- **Future Compatibility:** Validiert Kompatibilität mit zukünftigen Releases
- **Artifact Selection:** Wählt entsprechende BC-Artifacts
- **Upgrade Testing:** Testet Upgrade-Szenarien

**Anpassungsmöglichkeiten:**
- Workflow-spezifische Settings in `.github/<workflowName>.settings.json`
- `artifact`: Definiert BC-Artifacts für Testing
- `additionalCountries`: Zusätzliche Länder für Tests

**Workflow-spezifische Settings Beispiele:**

**Current.settings.json:**
```json
{
  "artifact": "*/current/*/*/latest"
}
```

**NextMinor.settings.json:**
```json
{
  "artifact": "*/nextminor/*/*/latest"
}
```

**NextMajor.settings.json:**
```json
{
  "artifact": "*/nextmajor/*/*/latest"
}
```

**Use Cases:**
- Testen gegen aktuelle BC-Version (Current)
- Vorbereitung auf nächstes Minor Update (NextMinor)
- Vorbereitung auf nächstes Major Update (NextMajor)

**Weitere Informationen:**
- [Artifacts](https://freddysblog.com/2023/04/21/working-with-artifacts/)

---

### Update AL-Go System Files (`UpdateGitHubGoSystemFiles.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Aktualisiert AL-Go System-Dateien (Workflows und Scripts) auf die neueste Version vom Template Repository.

**Hauptfunktionen:**
- **System Update:** Aktualisiert .github/workflows und .AL-Go Ordner
- **Template Sync:** Synchronisiert mit AL-Go Template Repository
- **Custom Template:** Unterstützt Custom Template Repositories
- **Pull Request:** Erstellt PR mit Updates (oder Direct Commit)
- **Compatibility:** Behält Custom Jobs und Overrides bei

**Input-Parameter:**
- `templateUrl`: Template Repository URL (Standard: aktuelles Template)
- `downloadLatest`: Latest Version vom Template herunterladen
- `directCommit`: Direkter Commit ohne PR

**Anpassungsmöglichkeiten:**
- `templateUrl`: Custom Template Repository verwenden
- `unusedALGoSystemFiles`: Dateien zum Entfernen markieren (deprecated)
- `customALGoFiles`: Custom Dateien konfigurieren
- `updateALGoSystemFilesEnvironment`: Environment für GhTokenWorkflow

**Custom Template Repository:**
Ermöglicht Verwendung eines eigenen Template Repositories mit:
- Custom Workflows
- Script Overrides
- Custom Jobs
- Organisationsspezifische Settings

**Prerequisites:**
- Secret `GhTokenWorkflow` für PR-Erstellung

**Weitere Informationen:**
- [Update AL-Go System Files](https://aka.ms/algoscenario/updatealgosystemfiles)
- [Custom Template Repositories](https://aka.ms/algosettings#customtemplate)

---

### Deploy Reference Documentation (`DeployReferenceDocumentation.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Generiert und deployed AL-Referenzdokumentation auf GitHub Pages.

**Hauptfunktionen:**
- **Documentation Generation:** Generiert AL-Dokumentation mit AL-Doc
- **GitHub Pages:** Deployed auf GitHub Pages
- **Multi-Release:** Unterstützt Dokumentation mehrerer Releases
- **Customization:** Anpassbare Header, Footer, und Landing Pages

**Anpassungsmöglichkeiten:**
- `alDoc`: Dokumentations-Konfiguration
  - `continuousDeployment`: Auto-Deploy bei CI/CD
  - `deployToGitHubPages`: GitHub Pages Deployment
  - `maxReleases`: Max. Anzahl Releases in Doku (Standard: 3)
  - `groupByProject`: Gruppierung nach Projekten
  - `includeProjects`: Projekte einschließen
  - `excludeProjects`: Projekte ausschließen
  - `header`: Custom Header
  - `footer`: Custom Footer
  - `defaultIndexMD`: Custom Landing Page
  - `defaultReleaseMD`: Custom Release Page

**AL-Doc Konfiguration Beispiel:**
```json
{
  "alDoc": {
    "continuousDeployment": true,
    "deployToGitHubPages": true,
    "maxReleases": 5,
    "header": "# My App Documentation",
    "footer": "© 2024 My Company"
  }
}
```

**Prerequisites:**
- GitHub Pages aktiviert
- GitHub Pages auf GitHub Actions konfiguriert

**Weitere Informationen:**
- [AL-Doc Settings](https://aka.ms/algosettings#aldoc)

---

### Troubleshooting (`Troubleshooting.yaml`)

**Trigger:** Manuell über `workflow_dispatch`

**Beschreibung:**
Diagnose-Workflow zur Fehlersuche und Analyse der AL-Go Konfiguration.

**Hauptfunktionen:**
- **Configuration Check:** Überprüft AL-Go Settings
- **Secret Validation:** Validiert Secret-Konfiguration
- **Environment Check:** Überprüft Environments
- **Diagnostic Output:** Detaillierte Diagnose-Informationen

**Input-Parameter:**
- `displayNameOfSecrets`: Zeigt Secret-Namen (nicht Werte)

**Use Cases:**
- Debugging von Workflow-Problemen
- Validierung der Konfiguration
- Überprüfung von Secrets und Settings
- Diagnose von Deployment-Problemen

**Weitere Informationen:**
- [Troubleshooting](https://aka.ms/algoscenario/troubleshooting)

---

## Interne Workflows

### _Build AL-Go Project (`_BuildALGoProject.yaml`)

**Trigger:** Wird von anderen Workflows aufgerufen (`workflow_call`)

**Beschreibung:**
Interner wiederverwendbarer Workflow für Build-Prozesse. Wird von CI/CD, PR-Handler und Test-Workflows verwendet.

**Hauptfunktionen:**
- **Project Build:** Kompiliert spezifisches AL-Go Projekt
- **Dependency Handling:** Resolved und installiert Dependencies
- **Artifact Publishing:** Published Build-Artifacts
- **Multi-Build-Mode:** Unterstützt verschiedene Build-Modi

**Input-Parameter:**
- `shell`: PowerShell oder pwsh
- `runsOn`: Runner-Typ (JSON-formatted)
- `checkoutRef`: Git Ref zum Checkout
- `project`: Projekt-Name
- `projectName`: Friendly Name
- `projectDependenciesJson`: Dependencies als JSON
- `buildMode`: Build-Modus (Default, Clean, Translated)
- `baselineWorkflowRunId`: Baseline für inkrementelle Builds
- `secrets`: Required Secrets
- `publishThisBuildArtifacts`: Artifact Publishing Flag

**Anpassungsmöglichkeiten:**
- Wird durch aufrufende Workflows konfiguriert
- Script Overrides in .AL-Go Ordner möglich
- Custom Jobs können hinzugefügt werden

**Verwendung:**
Wird intern von folgenden Workflows verwendet:
- CI/CD
- Pull Request Handler
- Current/NextMinor/NextMajor
- Publish To Environment
- Create Release

---

## Allgemeine Anpassungsmöglichkeiten

### Settings-Hierarchie

AL-Go Settings werden in einer definierten Hierarchie geladen (letzte Datei überschreibt vorherige):

1. **ALGoOrgSettings** (GitHub Variable, Organization-Level)
2. `.github/AL-Go-TemplateRepoSettings.doNotEdit.json` (Template Settings)
3. `.github/AL-Go-settings.json` (Repository Settings)
4. **ALGoRepoSettings** (GitHub Variable, Repository-Level)
5. `.github/AL-Go-TemplateProjectSettings.doNotEdit.json` (Template Project Settings)
6. `.AL-Go/settings.json` (Project Settings)
7. `.github/<workflow>.settings.json` (Workflow-spezifische Settings für alle Projekte)
8. `.AL-Go/<workflow>.settings.json` (Workflow-spezifische Settings für Projekt)
9. `.AL-Go/<username>.settings.json` (User-spezifische Settings)

### Conditional Settings

Settings können bedingt angewendet werden basierend auf:
- **repositories**: Repository-Patterns
- **projects**: Projekt-Patterns
- **buildModes**: Build-Modi
- **branches**: Branch-Patterns
- **workflows**: Workflow-Patterns
- **users**: User-Patterns

**Beispiel:**
```json
{
  "ConditionalSettings": [
    {
      "branches": ["feature/*"],
      "settings": {
        "doNotPublishApps": true,
        "doNotSignApps": true
      }
    }
  ]
}
```

### Commit Options

Steuert PR- und Commit-Verhalten für automatisierte Änderungen:

```json
{
  "commitOptions": {
    "messageSuffix": "AB#1234",
    "createPullRequest": true,
    "pullRequestAutoMerge": true,
    "pullRequestMergeMethod": "squash",
    "pullRequestLabels": ["AL-Go", "automated"]
  }
}
```

### Script Overrides

AL-Go unterstützt Overrides für verschiedene Pipeline-Schritte:

**Repository-Level (`.github/` Ordner):**
- `DeliverTo<DeliveryTarget>.ps1`: Custom Delivery
- `DeployTo<EnvironmentType>.ps1`: Custom Deployment

**Project-Level (`.AL-Go/` Ordner):**
- `PipelineInitialize.ps1`: Pipeline-Initialisierung
- `NewBcContainer.ps1`: Container-Erstellung
- `CompileAppInBcContainer.ps1`: App-Kompilierung
- `PublishBcContainerApp.ps1`: App-Publishing
- `RunTestsInBcContainer.ps1`: Test-Ausführung
- `SignBcContainerApp.ps1`: App-Signierung
- Und viele mehr...

**Weitere Informationen:**
- [Script Overrides](https://aka.ms/algosettings#scriptoverrides)
- [Custom Delivery](https://aka.ms/algosettings#customdelivery)
- [Custom Deployment](https://aka.ms/algosettings#customdeployment)

### Custom Jobs

Custom Jobs können zu Workflows hinzugefügt werden:

```yaml
CustomJob-PrepareDeploy:
  name: My Custom Job
  needs: [Build]
  runs-on: [ubuntu-latest]
  steps:
    - name: Custom Step
      run: |
        Write-Host "Custom Logic"
```

Custom Jobs bleiben bei "Update AL-Go System Files" erhalten.

**Weitere Informationen:**
- [Custom Jobs](https://aka.ms/algosettings#customjobs)

---

## Wichtige Settings

### Build & Compilation

| Setting | Beschreibung | Default |
|---------|--------------|---------|
| `country` | Zielland für App-Build | us |
| `repoVersion` | Repository Version (Major.Minor) | 1.0 |
| `appFolders` | Ordner mit Apps | [ ] (auto-detect) |
| `testFolders` | Ordner mit Test-Apps | [ ] (auto-detect) |
| `bcptTestFolders` | Ordner mit Performance-Test-Apps | [ ] (auto-detect) |
| `versioningStrategy` | Versioning-Strategie (0, 2, 3, 15, +16) | 0 |
| `buildModes` | Build-Modi (Default, Clean, Translated, Custom) | ["Default"] |
| `artifact` | BC Artifacts für Build | Latest Sandbox |

### Code Analysis

| Setting | Beschreibung | Default |
|---------|--------------|---------|
| `enableCodeCop` | CodeCop Analyzer aktivieren | false |
| `enableUICop` | UICop Analyzer aktivieren | false |
| `enableAppSourceCop` | AppSourceCop aktivieren | true (für AppSource Apps) |
| `enablePerTenantExtensionCop` | PTE Cop aktivieren | true (für PTEs) |
| `customCodeCops` | Custom Code Cop DLLs | [ ] |
| `failOn` | Pipeline Fail Level (none, warning, newWarning, error) | error |
| `trackALAlertsInGitHub` | AL Alerts in GitHub Security Tab | false |

### Testing

| Setting | Beschreibung | Default |
|---------|--------------|---------|
| `doNotBuildTests` | Tests nicht bauen | false |
| `doNotRunTests` | Tests nicht ausführen | false |
| `doNotRunBcptTests` | BCPT Tests nicht ausführen | false |
| `installTestFramework` | Test Framework installieren | auto-detected |
| `installTestLibraries` | Test Libraries installieren | auto-detected |
| `restoreDatabases` | Datenbank-Restore Events | [ ] |

### Deployment

| Setting | Beschreibung | Default |
|---------|--------------|---------|
| `environments` | Deployment-Umgebungen | [ ] |
| `DeployTo<EnvironmentName>` | Environment-Konfiguration | - |
| `excludeEnvironments` | Ausgeschlossene Environments | [ ] |

### Dependencies

| Setting | Beschreibung | Default |
|---------|--------------|---------|
| `appDependencyProbingPaths` | Dependency-Quellen | [ ] |
| `installApps` | 3rd-Party Dependencies | [ ] |
| `installTestApps` | 3rd-Party Test Dependencies | [ ] |
| `updateDependencies` | Auto-Update Dependencies | false |
| `generateDependencyArtifact` | Dependency Artifact generieren | false |

### Performance

| Setting | Beschreibung | Default |
|---------|--------------|---------|
| `useCompilerFolder` | Containerless Compilation | false |
| `workspaceCompilation` | Workspace Compilation (Preview) | { "enabled": false } |
| `incrementalBuilds` | Inkrementelle Builds | siehe Details |
| `cacheImageName` | Docker Image Cache (Self-Hosted) | my |
| `cacheKeepDays` | Cache Retention Days | 3 |

### Security & Signing

| Setting | Beschreibung | Default |
|---------|--------------|---------|
| `doNotSignApps` | Apps nicht signieren | false |
| `codeSignCertificateUrlSecretName` | Code-Signing Certificate URL Secret | CodeSignCertificateUrl |
| `codeSignCertificatePasswordSecretName` | Code-Signing Certificate Password Secret | CodeSignCertificatePassword |
| `keyVaultCodesignCertificateName` | KeyVault Certificate für Signing | - |
| `trustedSigning` | Azure Trusted Signing | - |

### Workflows

| Setting | Beschreibung | Default |
|---------|--------------|---------|
| `CICDPushBranches` | Branches für CI/CD bei Push | ["main", "release/*", "feature/*"] |
| `CICDPullRequestBranches` | Branches für PR Builds | ["main"] |
| `pullRequestTrigger` | PR Trigger Typ | pull_request |
| `workflowSchedule` | CRON Schedule für Workflows | - |
| `workflowConcurrency` | Workflow Concurrency Control | - |
| `workflowDefaultInputs` | Default Input Werte | [ ] |

### GitHub Runner

| Setting | Beschreibung | Default |
|---------|--------------|---------|
| `runs-on` | Standard GitHub Runner | windows-latest |
| `githubRunner` | Build/Test Runner | windows-latest |
| `shell` | Standard Shell | powershell |
| `githubRunnerShell` | Build Shell | powershell |

---

## Weiterführende Dokumentation

### AL-Go Hauptdokumentation

- [AL-Go for GitHub Repository](https://github.com/microsoft/AL-Go)
- [AL-Go Roadmap](https://aka.ms/ALGoRoadmap)
- [AL-Go Release Notes](https://github.com/microsoft/AL-Go/blob/main/RELEASENOTES.md)
- [AL-Go Deprecations](https://github.com/microsoft/AL-Go/blob/main/DEPRECATIONS.md)

### Getting Started

- [AL-Go Workshop](https://aka.ms/algoworkshop)
- [Get Started with AL-Go](https://aka.ms/algoscenario/getstarted)
- [Create New App](https://aka.ms/algoscenario/addatestapp)

### Configuration & Settings

- [Settings Overview](https://aka.ms/algosettings)
- [Secrets Documentation](https://freddysblog.com/2022/05/14/secrets-in-al-go-for-github/)
- [Conditional Settings](https://aka.ms/algosettings#conditional-settings)

### Deployment

- [Register Sandbox Environment](https://aka.ms/algoscenario/registersandboxenvironment)
- [Register Production Environment](https://aka.ms/algoscenario/registerproductionenvironment)
- [Deployment Strategies](https://freddysblog.com/2022/05/06/deployment-strategies-and-al-go-for-github/)

### Advanced Topics

- [Use Azure KeyVault](https://aka.ms/algoscenario/useazurekeyvault)
- [Self-Hosted GitHub Runner](https://aka.ms/algoscenario/selfhostedgithubrunner)
- [App Dependencies](https://aka.ms/algoscenario/appdependencies)
- [Delivery Targets & NuGet](https://aka.ms/algoscenario/deliverytargets)
- [Custom Template Repositories](https://aka.ms/algosettings#customtemplate)
- [Script Overrides](https://aka.ms/algosettings#scriptoverrides)

### Branching & Versioning

- [Branching Strategies](https://freddysblog.com/2022/05/03/branching-strategies-for-your-al-go-for-github-repo/)
- [Structuring Repositories](https://freddysblog.com/2022/04/28/structuring-your-github-repositories/)

### AppSource

- [Setup CI/CD for AppSource App](https://aka.ms/algoscenario/setupcicdforappourceapp)
- [Publish to AppSource](https://aka.ms/algoscenario/publishtoappource)
- [Enable KeyVault for AppSource](https://aka.ms/algoscenario/enablekeyvaultforappourceapp)

### Migration

- [Migrate from Azure DevOps (without history)](https://aka.ms/algoscenario/migratefromazuredevopswithouthistory)
- [Migrate from Azure DevOps (with history)](https://aka.ms/algoscenario/migratefromazuredevopswithhistory)

### Power Platform

- [Connect to Power Platform](https://aka.ms/algoscenario/setupowerplatform)
- [Setup Service Principal for Power Platform](https://aka.ms/algoscenario/setupserviceprincipalforowerplatform)
- [Try Power Platform Samples](https://aka.ms/algoscenario/trypowerplatformsamples)

### Community & Support

- [AL-Go for GitHub Blog Series](https://freddysblog.com/2022/04/26/al-go-for-github/)
- [Contributing to AL-Go](https://github.com/microsoft/AL-Go/blob/main/Scenarios/Contribute.md)
- [GitHub Issues](https://github.com/microsoft/AL-Go/issues)

---

## Zusammenfassung

Dieses Repository nutzt **AL-Go for GitHub** Version 6.2 für automatisierte DevOps-Prozesse. Die Workflows sind in drei Kategorien unterteilt:

### Automatische Workflows (2)
1. **CI/CD** - Automatischer Build/Test/Deploy bei Code-Änderungen
2. **Pull Request Handler** - Automatische Validierung bei Pull Requests

### Manuelle Workflows (14)
3. **Create Release** - Release-Erstellung mit Versioning
4. **Increment Version Number** - Versionsnummer aktualisieren
5. **Publish To Environment** - Deployment auf spezifische Umgebungen
6. **Create App** - Neue App erstellen
7. **Create Test App** - Neue Test App erstellen
8. **Create Performance Test App** - Neue BCPT App erstellen
9. **Add Existing App** - Existierende App hinzufügen
10. **Create Online Dev Environment** - BC Online Umgebung erstellen
11. **Current/NextMinor/NextMajor** - Version-Tests
12. **Update AL-Go System Files** - System-Update
13. **Deploy Reference Documentation** - Dokumentation deployen
14. **Troubleshooting** - Diagnose & Fehlersuche

### Interne Workflows (1)
15. **_Build AL-Go Project** - Wiederverwendbarer Build-Workflow

Alle Workflows sind hochgradig konfigurierbar über Settings-Dateien, Conditional Settings, Script Overrides und Custom Jobs. Die detaillierte Dokumentation von Microsoft bietet umfassende Informationen für erweiterte Anpassungen.

---

**Letzte Aktualisierung:** April 2026  
**AL-Go Version:** 6.2  
**Business Central Version:** 26.x
