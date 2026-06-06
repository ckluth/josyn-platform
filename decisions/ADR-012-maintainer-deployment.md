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
$BackendRoot\                            (default: C:\Programme\JOSYN)
    JOSYN.Jap.JAPServer.exe              <- JAP-Server-Executable
    JOSYN.Backend.CLI.exe                <- Backend-CLI-Executable
    josyn.bootstrap.ini                  <- angepasste Bootstrap-Konfiguration
    adapters\
        Contoso.Josyn.Adapter.dll        <- Adapter-Assembly (ADR-009)

$JobRepositoryRoot\                      (default: C:\Programme\JOSYN\JobRepository)
    Contoso.DemoProduct.DemoJob\
        Contoso.DemoProduct.DemoJob.exe  <- Erster Demojob
```

### bootstrap.ini-Anpassung

Die `josyn.bootstrap.ini` aus dem `josyn-backend`-Repo-Wurzel wird kopiert und dabei
zwei Schlüssel auf Deployment-Pfade umgeschrieben:

| Schlüssel | Ursprungswert (Repo) | Deployment-Wert |
|-----------|----------------------|-----------------|
| `JapServerExePath` | Entwickler-lokaler `C:\Temp\VS.OUT\...`-Pfad | `$BackendRoot\JOSYN.Jap.JAPServer.exe` |
| `JobRepositoryRoot` | Entwickler-lokaler `C:\Temp\VS.OUT\...`-Pfad | `$JobRepositoryRoot` |

`SessionStoreConnectionString` und `ConfigSourceType` werden unverändert übernommen —
sie sind deployment-unabhängig (Datenbankverbindung und Adapter-Typname bleiben gleich).

### Reihenfolge der Schritte

1. Zielordner bereinigen (vollständiges Delete + Recreate)
2. `JOSYN.Jap.JAPServer` publizieren → `$BackendRoot\`
3. `JOSYN.Backend.CLI` publizieren → `$BackendRoot\`
4. `Contoso.Josyn.Adapter` publizieren → `$BackendRoot\adapters\`
5. `Contoso.DemoProduct.DemoJob` publizieren → `$JobRepositoryRoot\Contoso.DemoProduct.DemoJob\`
6. `josyn.bootstrap.ini` kopieren und anpassen

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

3. **Adapter-Ladekonvention:** Das `adapters\`-Unterverzeichnis neben dem Backend-EXE
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

- **ADR-009** (Runtime Context Provider / Adapter-Pattern): Das `adapters\`-Verzeichnis
  ist die physische Umsetzung der dort beschriebenen Adapter-Lademechanik.
- **ADR-010** (Environment Separation): Dieses ADR adressiert nur DEV.
  INT und PROD sind explizit offene Fragen (s. o.).
