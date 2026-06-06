# ArgGen — Inspection & Smoke-Test Guide

> Temporary file. Delete after acceptance.

---

## 1. Source Inspection

**`josyn-sandbox\tools\arg-gen\`** — read these four files:

- `JobLoader.cs` — verify: loads `.dll` not `.exe`, finds `[JobEntryPoint]` by attribute *name*
  (string comparison, no hard dep on josyn-job-host), `<Clone>$` record detection,
  `JobAssemblyLoadContext` resolves deps from job folder
- `ScaffoldGenerator.cs` — verify: `; DemoArguments` comment, placeholder table matches ADR-014,
  `_launcher.cmd` template matches the original hand-authored file structurally
- `Program.cs` — verify: exit codes, `--cli-path` required check, no-parameters path exits 0
  with stderr note

**`josyn-sandbox\tools\deploy\deploy-maintainer.ps1`** — verify:
- `$SandboxRepoRoot` is derived from `$DevRoot` (same pattern as `$BackendRepoRoot`)
- Step 7a is positioned *after* job publish, *before* bootstrap.ini
- `dotnet run ... --configuration Release` (not debug)
- Error throw on non-zero exit code

**`josyn-contoso\contoso-demo-job\Contoso.DemoProduct.DemoJob\`** — verify:
- `local-arguments\` folder is gone
- Nothing else was touched

---

## 2. ArgGen Standalone — Edge Cases

```powershell
# Setup
$argGen = "C:\DevGit\josyn-sandbox\tools\arg-gen"
$jobExe = "C:\ProgramData\JOSYN\JobRepository\Contoso.DemoProduct.DemoJob\Contoso.DemoProduct.DemoJob.exe"
$cliExe = "C:\ProgramData\JOSYN\CLI\JOSYN.Backend.CLI.exe"
```

**a) Happy path — inspect output**
```powershell
dotnet run --project $argGen --configuration Release --no-build -- `
    $jobExe --cli-path $cliExe --output-dir C:\Temp\arggen-inspect

Get-Content C:\Temp\arggen-inspect\arguments-default.ini
Get-Content C:\Temp\arggen-inspect\_launcher.cmd
Get-Content C:\Temp\arggen-inspect\arguments-default.cmd
```
✅ Expect: INI has `; DemoArguments`, 5 properties, de-DE values. Launcher has correct CLI path and job name.

**b) Missing `--cli-path` → exit 1**
```powershell
dotnet run --project $argGen --configuration Release --no-build -- $jobExe
echo "Exit: $LASTEXITCODE"
```
✅ Expect: exit code 1, error to stderr.

**c) Non-existent job exe → exit 1**
```powershell
dotnet run --project $argGen --configuration Release --no-build -- `
    "C:\does-not-exist.exe" --cli-path $cliExe
echo "Exit: $LASTEXITCODE"
```
✅ Expect: exit code 1, clear error message.

**d) Verify INI is valid PropertyBag** — the critical functional check.
Edit `C:\Temp\arggen-inspect\arguments-default.ini` to fill in real values:
```ini
; DemoArguments
Message=Test via ArgGen
RepeatCount=1
ScheduledFor=06.06.2026
IsHighPriority=False
Budget=0,00
```
Then trigger a real job run:
```powershell
& $cliExe run-job "Contoso.DemoProduct.DemoJob" "C:\Temp\arggen-inspect\arguments-default.ini"
```
✅ Expect: exit 0, session starts, job completes with no deserialisation errors in ErrorStore/LocalLog.

---

## 3. Deploy Integration

```powershell
powershell -ExecutionPolicy Bypass -File `
    "C:\DevGit\josyn-sandbox\tools\deploy\deploy-maintainer.ps1" -SkipNugets
```

✅ Expect:
- Step `=== ArgGen: Contoso.DemoProduct.DemoJob ===` appears after job publish
- Exit 0, summary shows `local-arguments: generiert von ArgGen`
- `C:\ProgramData\JOSYN\JobRepository\Contoso.DemoProduct.DemoJob\local-arguments\` contains exactly 3 files

```powershell
Get-ChildItem "C:\ProgramData\JOSYN\JobRepository\Contoso.DemoProduct.DemoJob\local-arguments"
```

---

## 4. Live Launcher Test

Double-click `arguments-default.cmd` in Explorer, or run from cmd:
```
C:\ProgramData\JOSYN\JobRepository\Contoso.DemoProduct.DemoJob\local-arguments\arguments-default.cmd
```
✅ Expect: console shows job name + args filename, waits for ENTER. Ctrl-C aborts cleanly.

---

## Acceptance Criteria

| Check | Pass condition |
|-------|----------------|
| Source inspection | No surprises, matches design |
| Happy path output | 3 files, correct content |
| Edge cases b, c | Exit 1 + readable error |
| INI valid for PropertyBag | Job runs, no deserialisation error |
| Deploy integration | Step 7a runs, 3 files in deployed folder |
| Live launcher | Confirmation gate works |
| `josyn-contoso` | `local-arguments\` gone, nothing else changed |

If all seven pass — ship it.
