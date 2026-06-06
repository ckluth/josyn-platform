# ADR-012 — Maintainer Deployment (Erste Iteration)

**Date:** 2026-06-06
**Status:** Proposal — unter Konstruktion; dient als Basis für späteres Refinement

---

## Context

Die JOSYN-Plattform befindet sich in der PoC-Phase. Es gibt noch kein CI/CD-System,
keinen Installer und keine automatisierte Staging-Pipeline. Dennoch ist es notwendig,
die aufeinander abgestimmten Artefakte (Backend-EXEs, Adapter-DLLs, Job-EXEs,
bootstrap.ini) reproduzierbar und ohne Handarbeit auf einem Entwicklungsrechner zu
deployen — sowohl für Integrationstests als auch zur manuellen Demonstration.

Dieses ADR beschreibt die erste, bewusst einfach gehaltene Iteration: ein einzelnes
PowerShell-Skript für den Maintainer-Einsatz auf demselben Rechner, auf dem die
lokalen Repos liegen.

---

## Decision

### Deployment-Skript

Ein PowerShell-Skript `deploy-maintainer.ps1` liegt in:

```
josyn-sandbox\tools\deploy\deploy-maintainer.ps1
```

Das Skript ist in `josyn-sandbox` abgelegt, weil es:
- keine Plattform-eigene Infrastruktur ist (kein CI, kein Service, kein Protokoll)
- Maintainer-Tooling ist — bewusst außerhalb der produzierten Artefakte
- Abhängigkeit auf mehrere Repos hat (`josyn-backend`, `josyn-contoso`) und deshalb
  nicht sinnvoll in einem einzelnen Repo beheimatet ist

Das Skript verwendet **`dotnet publish --no-self-contained`** statt `dotnet build + Copy-Item`.
Damit werden alle Laufzeit-Abhängigkeiten vollständig und ohne manuelle Selektion in
den Zielordner publiziert.

### Zielstruktur

```
$BackendRoot\                            (default: C:\ProgramData\JOSYN)
    josyn.bootstrap.ini                  <- gemeinsame Bootstrap-Konfiguration
    CLI\
        JOSYN.Backend.CLI.exe            <- Backend-CLI-Executable
    JAPServer\
        JOSYN.Jap.JAPServer.exe          <- JAP-Server-Executable
        Adapters\
            Contoso.Josyn.Adapter.dll    <- Adapter-Assembly (ADR-009)

$JobRepositoryRoot\                      (default: C:\ProgramData\JOSYN\JobRepository)
    Contoso.DemoProduct.DemoJob\
        Contoso.DemoProduct.DemoJob.exe  <- Erster Demojob
```

### bootstrap.ini-Konvention

`josyn.bootstrap.ini` liegt auf `$BackendRoot`-Ebene — eine Ebene **über** dem
Verzeichnis des jeweiligen Trigger-Executables. Jedes Trigger-Exe (CLI, künftig
Listener, Ticker, …) lädt sie über:

```csharp
Path.Combine(AppContext.BaseDirectory, "..", "josyn.bootstrap.ini")
```

**Begründung:** Die ini ist keine CLI-eigene Konfiguration, sondern eine
Installations-weite Bootstrap-Ressource. Sie einmal auf `$BackendRoot`-Ebene
abzulegen vermeidet Duplizierung und stellt sicher, dass alle Trigger-Executables
dieselbe Konfiguration sehen.

### BackendRoot-Konvention

Das Verzeichnis der geladenen `josyn.bootstrap.ini` **ist** der BackendRoot.
Alle weiteren Deployment-Pfade werden daraus per Konvention abgeleitet —
kein einziger Pfad steht explizit in der ini:

| Ressource | Konventionspfad |
|-----------|----------------|
| JAPServer-Executable | `BackendRoot\JAPServer\JOSYN.Jap.JAPServer.exe` |
| Job-Repository | `BackendRoot\JobRepository\{JobTypeName}\{JobTypeName}.exe` |
| Adapters | `BackendRoot\JAPServer\Adapters\` |

`BackendRoot` selbst wird in `FileBootstrapConfig` als `Path.GetDirectoryName(iniPath)`
berechnet und über `IBootstrapConfig.BackendRoot` bereitgestellt.

### JAPServer-Pfad-Konvention

Der Pfad zu `JOSYN.Jap.JAPServer.exe` wird **nicht** in der `josyn.bootstrap.ini`
konfiguriert. Stattdessen berechnet `SessionStarter` ihn zur Laufzeit per Konvention:

```
<Verzeichnis des aufrufenden Trigger-Executables>/../JAPServer/JOSYN.Jap.JAPServer.exe
```

**Begründung:**  
`SessionStarter` ist der einzige Ort, der `JAPServer.exe` kennen muss. Neben dem
aktuellen CLI-Executable wird es künftig weitere Trigger-Executables geben
(Scheduler-Service, Workflow-Adapter). Alle werden als Geschwister-Unterordner neben
`JAPServer\` deployed. Die Konvention hält diese Kopplung ohne Konfigurationsaufwand
konsistent — auf Kosten einer festen Verzeichnisstruktur, die durch dieses ADR
verbindlich festgelegt wird.

**Konsequenz für Deployment:**  
Der `JapServerExePath`-Schlüssel entfällt aus `josyn.bootstrap.ini` vollständig.
`deploy-maintainer.ps1` schreibt ihn nicht mehr.

### bootstrap.ini-Anpassung

Die `josyn.bootstrap.ini` aus dem `josyn-backend`-Repo-Wurzel wird kopiert und dabei
zwei Schlüssel auf Deployment-Pfade umgeschrieben:

| Schlüssel | Ursprungswert (Repo) | Deployment-Wert |
|-----------|----------------------|-----------------|
| — | — | — |

Keine Schlüssel müssen umgeschrieben werden. `SessionStoreConnectionString` und
`ConfigSourceType` werden unverändert übernommen. Alle Pfad-Schlüssel entfallen —
sie werden per BackendRoot-Konvention berechnet (siehe Abschnitt oben).

### Reihenfolge der Schritte

1. Zielordner bereinigen (vollständiges Delete + Recreate)
2. `JOSYN.Jap.JAPServer` publizieren → `$BackendRoot\JAPServer\`
3. `JOSYN.Backend.CLI` publizieren → `$BackendRoot\CLI\`
4. `Contoso.Josyn.Adapter` publizieren → `$BackendRoot\JAPServer\Adapters\`
5. `Contoso.DemoProduct.DemoJob` publizieren → `$JobRepositoryRoot\Contoso.DemoProduct.DemoJob\`
6. `josyn.bootstrap.ini` kopieren und anpassen (`JobRepositoryRoot`) → `$BackendRoot\`

---

## Rationale

**Warum `josyn-sandbox\tools\deploy\`?**
Das Skript ist Maintainer-Tooling. Es hat keine Eigenschaft eines Plattform-Artefakts
(kein Protokoll, kein NuGet-Paket, keine Assembly). `josyn-sandbox` ist der designierte
Ort für solches Tooling. Der Sandbox-Constraint gilt in dieser Richtung nicht: das Skript
*konsumiert* Plattform-Repos — es wird nicht von ihnen referenziert.

**Warum vollständiges Löschen statt inkrementelles Update?**
In der PoC-Phase ist das Zielverzeichnis kein produktiver Serviceordner — ein sauberer
Zustand ist wichtiger als minimale Downtime. Der Maintainer deployt manuell; ein kurzer
Ausfall ist akzeptabel.

**Warum `--no-self-contained`?**
Der Entwicklungsrechner hat .NET 10 installiert. Self-contained würde die Deployment-
Größe signifikant erhöhen ohne Mehrwert für diese Iteration.

---

## Offene Fragen (Refinement-Basis)

Diese Fragen sind explizit offen — dieses ADR schließt sie nicht ab, dokumentiert
sie aber als Input für eine spätere Deployment-Architektur:

1. **Staging-Umgebungen:** Wie wird zwischen DEV, INT, PROD unterschieden?
   Separate bootstrap.ini pro Umgebung? Environment-Parameter im Skript?
   (Vgl. ADR-010)

2. **Job-Repository-Konvention:** Das `JobRepositoryRoot`-Konzept ist in
   `IBootstrapConfig` als einfacher Stammpfad modelliert. Wenn Jobs mehrere
   Dateien (Konfiguration, Icons, Manifest) benötigen, braucht jeder Job-Subfolder
   eine definierte Struktur. Diese Struktur ist noch nicht entschieden.

3. **Adapter-Ladekonvention:** Das `Adapters\`-Unterverzeichnis neben dem Backend-EXE
   ist eine Konvention aus ADR-009. Sie ist hier erstmals physisch umgesetzt.
   Ob mehrere Adapter nebeneinander liegen können, ist noch nicht spezifiziert.

4. **CI/CD:** Dieses Skript ist explizit kein CI-Build-Step. Wenn eine Pipeline
   eingeführt wird, muss das Deployment-Modell neu entworfen werden (Staging-Slots,
   Artifact-Upload, Rollback-Strategie).

5. **Installer vs. Script:** Für eine echte Produktionsumgebung ist ein Windows-Installer
   (MSI/WiX) oder ein Paketierungsformat naheliegend. Das ist außerhalb des aktuellen
   Scopes.

6. **Mehrere Jobs:** Derzeit ist `Contoso.DemoProduct.DemoJob` der einzige Job.
   Wenn weitere Jobs hinzukommen, muss das Skript entweder erweitert oder
   parametrisiert werden (z. B. Schleife über eine Liste von Job-Solutions).

---

## Consequences

- Maintainer kann mit einem einzigen `pwsh .\deploy-maintainer.ps1`-Aufruf eine
  vollständige, konsistente Deployment-Umgebung auf dem lokalen Rechner erstellen.
- Die genaue Zielstruktur ist dokumentiert und reproducierbar.
- Offene Fragen sind explizit festgehalten — späteres Refinement kann direkt
  hier anknüpfen.

---

## Relation zu anderen ADRs

- **ADR-009** (Runtime Context Provider / Adapter-Pattern): Das `Adapters\`-Verzeichnis
  ist die physische Umsetzung der dort beschriebenen Adapter-Lademechanik.
- **ADR-010** (Environment Separation): Dieses ADR adressiert nur DEV.
  INT und PROD sind explizit offene Fragen (s. o.).
