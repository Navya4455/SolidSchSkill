# SOLIDWORKS Electrical skill — Phase 0 (spike) and Phase 1 (bridge) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prove that SOLIDWORKS Electrical 2026 SP4.1 can be driven through its API from inside the running application (Phase 0), then build the bridge — an Electrical add-in plus the `swe` command-line tool — that creates, reads, edits, renders, exports and backs up Electrical projects safely (Phase 1).

**Architecture:** A small C# add-in (`AddLife.SweBridge`, .NET Framework 4.8, x64) is loaded by Electrical at startup. It listens on `127.0.0.1` only, accepts JSON batches of operations and runs them one at a time on Electrical's UI thread through the EwAPI COM interface. A Python package `swe` talks to the add-in for every change, and reads Electrical's SQL Server directly — strictly `SELECT`-only, behind a schema-version guard — for bulk reads. Nothing in this repo holds project state; the Electrical database is the only master.

**Tech Stack:** C# / .NET Framework 4.8 (SDK-style project, `Microsoft.NETFramework.ReferenceAssemblies`), EwAPI interop embedded (`EmbedInteropTypes`), `System.Web.Extensions` JSON, xUnit 2; Python 3.14, `pyodbc` 5.3 (ODBC Driver 17 for SQL Server), `pytest`, `ezdxf` (spike only); PowerShell 7 scripts.

**Spec:** `docs/superpowers/specs/2026-10-09-solidworks-electrical-skill-design.md` (read it with this plan; §5, §6, §11 and §14 are the parts these phases implement).

---

## Summary for review

### What these two phases deliver

| Phase | What you get | How we know it is done |
|---|---|---|
| **0 — Spike** (about 2 working days) | Proof that our add-in loads inside Electrical with the licence code, and a written answer to ten open questions (S1–S10 in spec §11): can it place symbols, draw connected wires, write text, create library symbols and parts, PLC modules and macros, render sheets to PNG, export PDF, archive and restore projects, and does the SQL netlist match what the API sees. The add-in's communication core is built properly here and kept. | A findings document in `docs/superpowers/specs/`, with every answer backed by saved evidence; the spec and this plan amended where an answer was "no" |
| **1 — Bridge** (about 6–8 working days) | The working bridge: ~25 operations (projects, books, sheets, locations, functions, components, symbols, wires, text, wire numbering, PNG render, PDF export, archive/restore); `swe` commands for health, batches, read-only database queries, backups and a self-test; snapshots before every editing session; a write-only change log | `swe selftest` passes end to end on a scratch project `LIBTEST`; one motor-feed sheet is built from a batch, rendered and checked; a backup is restored and proven identical |

Estimates are working days of agent time, valid only once decisions D-A and D-B below are in hand. Your own time: closing and reopening Electrical when the add-in is updated (a few minutes, roughly once per task in Phase 1), and reviewing the findings and the selftest report. A firm estimate for Phases 2–4 (library pipeline, template, WetBlasting) is given after Phase 0.

### Decisions and approvals needed

| # | Decision | Needed by | Proposed |
|---|---|---|---|
| **D-A** | **Electrical API licence code from the reseller.** The official 2026 add-in sample states: *"License Code: Contact your reseller to get your license code."* The API also returns `EW_LICENSE_WITHOUT_API_OPTION` when the licence lacks the API option. **Without the code nothing in either phase can run.** | Task 0.5 | Request it from the reseller now; ask them to confirm the licence includes the API option |
| **D-B** | **Administrator rights on this PC, twice:** to install the .NET 8 SDK (Task 0.1) and to register the add-in once (`deploy.ps1 -Register`, Task 0.5). Later updates do not need admin. | Tasks 0.1, 0.5 | IT runs the two commands, or grants a one-time elevation |
| **D-C** | **Backup destination off this PC** (network share, OneDrive/SharePoint folder, or external drive). Today none of the 18 Electrical databases has ever been backed up. | Task 1.9 | A network share with daily copies, 30-day retention |
| **D-D** | **Rollout to colleagues' PCs** (library, template, add-in, skill on other machines) | Before Phase 5 | Phase 5, after WetBlasting is accepted |
| **D-E** | **Owner** who approves new library items and signs off the WetBlasting acceptance | Phase 2 | The requesting engineer |
| **D-F** | **Which library Claude's symbols and parts go into.** An `Addlife` library already exists with 196 symbols; a library named `ADDLIFE` would collide with it. | Phase 2 | A separate library `ADDLIFE_AUTO`, so Claude's items can be reviewed and rolled back without touching the existing 196 |

### Spec items deliberately left to later phases

| Spec item | Why later | Phase |
|---|---|---|
| Library operations (`create_symbol`, `update_symbol_graphics`, `create_part`, `create_macro`, `set_review_flag`), PLC I/O check against `.acbind`, review list | Shaped by Phase 0 answers S4–S6 | 2 |
| `title_block_grid`, grid/rail/routing layout helpers, per-sheet lint (spec §10.2) | Needed when real sheets are composed | 2 |
| `update_reports`, `generate_terminal_strips`, template v2, manual-edit test | Shaped by S7 | 3 |
| `move_sheet`, `move`, `clear_sheet`, `insert_macro`, reads of `wires`/`locations`/`functions` | Needed by the skill's workflows; the SQL netlist covers wires until then | 4 |
| The skill itself (`SKILL.md`, references), WetBlasting rebuild | After the bridge and library exist | 4 |

### What these phases will not touch

- **Your existing Electrical projects.** The add-in refuses any change to a project not on its allowlist (`LIBTEST*`, `SPIKE-*`). The existing project 25 "Wet Blasting Machine" is read-only to the bridge. The WetBlasting rebuild (Phase 4) goes into a new project.
- **The standard library and the existing `Addlife` library.** Phase 0 creates and then deletes one scratch library `ADDLIFE_SPIKE`; Phase 1 creates no library items.
- **The database.** Reads only; every change goes through Electrical's API.

---

## Global Constraints

- SOLIDWORKS Electrical **2026 SP4.1, 34.41.1011**; interop `2026.4.1.1011`; project-data schema **292**; app-project schema **32**. Anything else: stop and follow spec §12's upgrade routine.
- Add-in: .NET Framework **4.8**, **x64**, COM-visible class GUID **`6f3b8f0e-2c1d-4a7e-9b52-5d8a1c3e7f40`**, ProgID `AddLife.SweBridge`, EwAPI interop **embedded** (no runtime dependency on `Interop.EwAPI.dll`), no third-party NuGet packages in the add-in itself.
- Listener bound to **127.0.0.1 only**; every request carries `X-SWE-Token`; the token is random per Electrical start.
- **Writes only through the add-in. SQL is `SELECT`-only**, enforced in code, behind the schema guard.
- **Mutation allowlist** (`%LOCALAPPDATA%\AddLife\swe-bridge\config.json` → `mutation_allowlist`, default `["LIBTEST*", "SPIKE-*"]`). No other project is ever changed in these phases.
- A batch names its target project; an operation that needs a project is refused if Electrical's current project differs.
- First mutating operation of a session takes an Electrical **snapshot**.
- The ops log is **write-only**; no code reads it.
- **Secrets never in git:** the licence code lives in `%LOCALAPPDATA%\AddLife\swe-bridge\license.txt`; the session token in `session.json` next to it.
- Commits use the repo-local identity (Navya R) and end with the two trailer lines from the session's attribution rule.

## Review Focus

The five inputs most likely to bite a person using this, and where each is pinned:

1. **Electrical closed, crashed, or restarted since `session.json` was written** → `swe` says "bridge is not running / token rejected — restart Electrical" within a second, never hangs or prints a stack trace. Pinned in Task 0.6 (`test_dead_port_is_not_running`, `test_missing_session_file`, `test_wrong_token`).
2. **The user switches projects in Electrical between Claude's read and write** → the batch is refused with `project_mismatch`; nothing is applied to the wrong project. Pinned in Task 0.3 (`Refuses_op_when_current_project_differs`).
3. **A slow batch times out on the client and is sent again** → it runs once; the second send returns the cached result. Pinned in Task 0.3 (`Same_batch_id_returns_cached_result_without_rerunning`) and Task 0.6 (`test_batch_sends_given_batch_id`).
4. **Two components with the same tag, created by hand in Electrical** → upsert refuses with `ambiguous_identity` naming both ids instead of editing whichever comes first. Pinned in Task 1.4 (`IdentityTests`).
5. **Non-ASCII text** (Ω, µ, °, →) in descriptions, sheet titles and text → arrives in Electrical unchanged. Pinned in Task 0.3 (`Unicode_arguments_round_trip`), Task 0.4 (`Utf8_body_round_trips`) and Task 1.6 (selftest reads the component description and sheet title back from Electrical).

## File structure

```
SolidSchSkill/
├─ .gitignore                      work/, .venv/, bin/, obj/, spike/out/, *.png renders
├─ README.md                       what this repo is, how to build, where secrets live
├─ addin/
│  ├─ deploy.ps1                   build → copy to C:\ProgramData\AddLife\SweBridge → (once, admin) regasm
│  ├─ undeploy.ps1                 regasm /unregister, remove the folder
│  ├─ src/AddLife.SweBridge/
│  │  ├─ AddLife.SweBridge.csproj
│  │  ├─ Core/                     pure: no EwAPI types — unit-tested
│  │  │  ├─ OpException.cs  Json.cs  Args.cs  OpSpec.cs  IOperation.cs  OpRegistry.cs
│  │  │  ├─ BridgeConfig.cs  BridgePaths.cs  BridgeLog.cs  IBridgeHost.cs  Identity.cs
│  │  │  ├─ BatchRequest.cs  BatchResponse.cs  BatchCache.cs  BatchRunner.cs
│  │  │  └─ MiniHttpServer.cs  BridgeService.cs  SessionFile.cs
│  │  ├─ Electrical/               EwAPI-bound: verified live (selftest)
│  │  │  ├─ BridgeAddIn.cs  Registration.cs  LicenseCode.cs  ConnectEvidence.cs
│  │  │  ├─ WinFormsDispatcher.cs  EwContext.cs  Ew.cs  EwHost.cs  EwOperation.cs
│  │  │  ├─ ReadModel.cs  Operations.cs
│  │  │  └─ Ops/  CurrentProjectOp.cs  SessionOps.cs  StructureOps.cs  DrawingOps.cs  ReadOps.cs  ActionOps.cs
│  │  └─ Probes/                   Phase 0 only — deleted in Task 0.9
│  └─ tests/AddLife.SweBridge.Tests/   xUnit, net48
├─ swe/
│  ├─ pyproject.toml
│  ├─ src/swe/
│  │  ├─ errors.py  session.py  client.py  jsonout.py  config.py  opslog.py  backup.py  selftest.py  opsdoc.py  layout.py  cli.py
│  │  ├─ db/  connection.py  guard.py  queries.py
│  │  └─ commands/  health.py  batch.py  db.py  config.py  backup.py  selftest.py  opsdoc.py
│  └─ tests/
├─ spike/                          Phase 0 scripts; results/*.json committed as evidence
├─ examples/motor_feed.py          Phase 1 acceptance sheet
├─ tools/dump-ewapi.ps1
└─ docs/  reference/  superpowers/specs/  superpowers/plans/
```

## Conventions for every task

- **Repo root:** `C:\Automation\SolidSchSkill`. All commands run from there in PowerShell 7 unless stated.
- **C# build and test:** `dotnet test addin\tests\AddLife.SweBridge.Tests -c Release` (builds the add-in too).
- **Python:** the venv is `.venv`; run tools as `.venv\Scripts\python -m pytest swe\tests -q` and `.venv\Scripts\swe ...`.
- **Live steps** need Electrical running with the bridge loaded. Steps marked **[user]** need the human: closing/reopening Electrical, elevated commands, putting the licence code in place.
- **Deploying a changed add-in** (Phase 1, after the one-time registration): [user] close Electrical → `pwsh addin\deploy.ps1` → [user] start Electrical and open `LIBTEST`.
- **Commit** at the end of each task: `git add <files>` then `git commit` with a conventional message (`feat:`, `test:`, `docs:`, `chore:`) ending with the session's two trailer lines; `git push` at the end of each phase.

---

# Phase 0 — Spike

Phase 0 answers spec §11's questions S1–S10. The add-in's core (Tasks 0.3–0.6) is production code and is kept; the probe operations (Task 0.7) are throwaway and are deleted in Task 0.9.

### Task 0.1: Repository, tooling and the secrets folder

**Files:**
- Create: `.gitignore`, `README.md`
- Create (outside git): `%LOCALAPPDATA%\AddLife\swe-bridge\config.json`

**Interfaces:**
- Produces: the `.venv` Python environment; the bridge folder `%LOCALAPPDATA%\AddLife\swe-bridge\` that later tasks read (`config.json`, `license.txt`) and write (`session.json`, `bridge.log`).

- [ ] **Step 1: [user, admin] Install the .NET 8 SDK**

Run in an elevated PowerShell: `winget install --id Microsoft.DotNet.SDK.8 --exact --silent`

Then, in a normal PowerShell: `dotnet --list-sdks`
Expected: a line starting `8.0.`

- [ ] **Step 2: Create the Python environment**

```powershell
python -m venv .venv
.venv\Scripts\python -m pip install --upgrade pip
.venv\Scripts\python -m pip install "pyodbc==5.3.0" "pytest>=8.3" "ezdxf>=1.3"
.venv\Scripts\python -c "import pyodbc, pytest, ezdxf; print(pyodbc.version, pytest.__version__, ezdxf.__version__)"
```
Expected: three version numbers, `5.3.0` first.

- [ ] **Step 3: Write `.gitignore`**

```gitignore
# scratch and build output
work/
.venv/
**/bin/
**/obj/
*.user
.vs/
__pycache__/
*.egg-info/
.pytest_cache/
# spike artefacts (results/*.json are committed as evidence; images and PDFs are not)
spike/out/
# renders anywhere
*.png
!docs/**/*.png
```

- [ ] **Step 4: Write `README.md`**

```markdown
# SolidSchSkill

Lets Claude build and maintain native SOLIDWORKS Electrical projects.

- `addin/` — the AddLife SWE Bridge add-in loaded by SOLIDWORKS Electrical (C#, .NET Framework 4.8).
- `swe/` — the `swe` Python package and command-line tool that talks to the add-in and reads Electrical's database (read-only).
- `docs/superpowers/specs/` — the design (start with `2026-10-09-solidworks-electrical-skill-design.md`).
- `docs/superpowers/plans/` — implementation plans.

## Where secrets and machine state live (never in git)

`%LOCALAPPDATA%\AddLife\swe-bridge\`
- `license.txt` — the Electrical API licence code from the reseller.
- `config.json` — bridge settings (`mutation_allowlist`, `language`, `probes_enabled`, `port`).
- `session.json` — written by the add-in at every Electrical start: port and token.
- `bridge.log` — the add-in's log.
- `swe.json` — `swe` settings (`backup_dir`, `work_dir`, ...).

## Build

    dotnet test addin\tests\AddLife.SweBridge.Tests -c Release
    .venv\Scripts\python -m pytest swe\tests -q

Deploying the add-in: close Electrical, run `pwsh addin\deploy.ps1` (the first time: elevated, with `-Register`), start Electrical.
```

- [ ] **Step 5: Create the bridge folder and Phase 0 config**

```powershell
$dir = Join-Path $env:LOCALAPPDATA 'AddLife\swe-bridge'
New-Item -ItemType Directory -Force $dir | Out-Null
@'
{
  "mutation_allowlist": ["LIBTEST*", "SPIKE-*"],
  "probes_enabled": true,
  "language": "en",
  "port": 0
}
'@ | Set-Content -Encoding utf8NoBOM (Join-Path $dir 'config.json')
Get-Content (Join-Path $dir 'config.json')
```
Expected: the JSON above printed back.

- [ ] **Step 6: Commit**

```powershell
git add .gitignore README.md
git commit -m "chore: repository layout, ignore rules and README"
```

### Task 0.2: EwAPI interop surface dump

**Files:**
- Create: `tools/dump-ewapi.ps1`
- Create: `docs/reference/ewapi-2026.4.1.1011-types.txt` (generated)

**Interfaces:**
- Produces: `tools/dump-ewapi.ps1 [-Interop <dll>] [-OutDir <dir>]` writing `ewapi-<assembly version>-types.txt`. Spec §12's upgrade routine diffs two of these files.

- [ ] **Step 1: Write `tools/dump-ewapi.ps1`**

```powershell
<#
.SYNOPSIS
Writes a sorted, stable listing of every interface method and enum value in the SOLIDWORKS Electrical
EwAPI interop to docs/reference/ewapi-<version>-types.txt. After an Electrical update, run it again
and diff the two files to see exactly what the API gained or lost.
#>
param(
    [string]$Interop = 'C:\Program Files\SOLIDWORKS Corp\SOLIDWORKS Electrical\bin\Interop.x64.EwAPI.dll',
    [string]$OutDir = (Join-Path $PSScriptRoot '..\docs\reference')
)
$ErrorActionPreference = 'Stop'
$asm = [System.Reflection.Assembly]::LoadFile($Interop)
$version = $asm.GetName().Version.ToString()

function Format-Param($p) {
    $type = $p.ParameterType.Name.TrimEnd('&')
    $prefix = if ($p.IsOut) { 'out ' } elseif ($p.ParameterType.IsByRef) { 'ref ' } else { '' }
    "$prefix$type $($p.Name)"
}

$lines = New-Object System.Collections.Generic.List[string]
$lines.Add("# EwAPI interop $version  ($Interop)")
foreach ($t in ($asm.GetTypes() | Where-Object { $_.IsPublic } | Sort-Object FullName)) {
    if ($t.IsEnum) {
        $values = [Enum]::GetNames($t) | ForEach-Object { "$_=$([long][Enum]::Parse($t, $_))" }
        $lines.Add("enum $($t.Name): $($values -join ', ')")
    }
    elseif ($t.IsInterface -and -not $t.Name.StartsWith('_')) {
        $lines.Add("interface $($t.Name)")
        foreach ($m in ($t.GetMethods() | Sort-Object Name, { $_.GetParameters().Count })) {
            $params = ($m.GetParameters() | ForEach-Object { Format-Param $_ }) -join ', '
            $lines.Add("    $($m.ReturnType.Name) $($m.Name)($params)")
        }
    }
}
New-Item -ItemType Directory -Force $OutDir | Out-Null
$out = Join-Path $OutDir "ewapi-$version-types.txt"
[System.IO.File]::WriteAllLines($out, $lines, (New-Object System.Text.UTF8Encoding($false)))
Write-Output "$out ($($lines.Count) lines)"
```

- [ ] **Step 2: Run it**

Run: `pwsh tools\dump-ewapi.ps1`
Expected: `...docs\reference\ewapi-2026.4.1.1011-types.txt (N lines)` with N in the thousands.

- [ ] **Step 3: Check it is stable and contains what the design relies on**

```powershell
pwsh tools\dump-ewapi.ps1 | Out-Null
git diff --no-index --stat docs\reference\ewapi-2026.4.1.1011-types.txt docs\reference\ewapi-2026.4.1.1011-types.txt
Select-String docs\reference\ewapi-2026.4.1.1011-types.txt -Pattern 'connectToEwAPI|insertFromTemplate|newEwProjectSymbol|insertFromDwg|EW_LICENSE_WITHOUT_API_OPTION' | ForEach-Object Line
```
Expected: running twice produces the same file (no diff output); the five names are found.

- [ ] **Step 4: Commit**

```powershell
git add tools/dump-ewapi.ps1 docs/reference/ewapi-2026.4.1.1011-types.txt
git commit -m "chore: dump the EwAPI interop surface for upgrade diffs"
```

### Task 0.3: Add-in core — arguments, operations, configuration, batch runner

Pure C# with no EwAPI types, fully unit-tested. Everything Electrical-specific plugs into it later through `IBridgeHost`, `IOpContext` and `IOperation`.

**Files:**
- Create: `addin/src/AddLife.SweBridge/AddLife.SweBridge.csproj`
- Create: `addin/src/AddLife.SweBridge/Core/OpException.cs`, `Json.cs`, `Args.cs`, `OpSpec.cs`, `IOperation.cs`, `OpRegistry.cs`, `BridgeConfig.cs`, `BridgePaths.cs`, `BridgeLog.cs`, `IBridgeHost.cs`, `BatchRequest.cs`, `BatchResponse.cs`, `BatchCache.cs`, `BatchRunner.cs`
- Create: `addin/tests/AddLife.SweBridge.Tests/AddLife.SweBridge.Tests.csproj`
- Test: `addin/tests/AddLife.SweBridge.Tests/ArgsTests.cs`, `OpRegistryTests.cs`, `BridgeConfigTests.cs`, `BatchRequestTests.cs`, `BatchRunnerTests.cs`, `Fakes.cs`

**Interfaces:**
- Produces (namespace `AddLife.SweBridge.Core`):
  - `OpException(string code, string message, string lastError = null)` with `Code`, `LastError`
  - `Json.Write(object) : string`, `Json.ReadObject(string) : Dictionary<string, object>`
  - `Args(Dictionary<string, object>)`: `Has`, `Str`, `OptStr`, `Num`, `OptNum`, `Int`, `OptInt`, `OptBool`, `OptObj`, `OptArr`, `Keys`
  - `ArgSpec(name, type, required, doc)`, `OpSpec(name, summary, mutating, requiresProject, params ArgSpec[])`, both with `ToJson()`
  - `IOpContext` (marker), `IOperation { Spec; TargetProject(Args, string); Execute(IOpContext, Args) }`, abstract `Operation` with `Req(...)`/`Opt(...)`
  - `OpRegistry`: `Add`, `Find`, `Specs`, static `Validate(OpSpec, Args)`
  - `BridgeConfig`: `MutationAllowlist`, `ProbesEnabled`, `Language`, `Port`, static `Load(path)`, `AllowsMutation(project)`
  - `BridgePaths.Root` (settable for tests), `.Session`, `.Config`, `.License`, `.Log`; `BridgeLog.Info/Error`
  - `IBridgeHost { Config; CurrentProjectName(); EnsureSnapshot(session, project); Context(); LastApiError(); }`
  - `BatchOp(op, args)`, `BatchRequest(batchId, session, project, stopOnError, ops)` + `BatchRequest.Parse(Dictionary)`
  - `OpResult.Ok/Failed/Skipped`, `BatchResponse(...)` with `Refusal`, `Results`, `Snapshot`, `Cached`, `AsCached()`, `ToJson()`
  - `BatchCache(int capacity)`, `BatchRunner(OpRegistry, IBridgeHost, BatchCache).Run(BatchRequest) : BatchResponse`

- [ ] **Step 1: Create the add-in project file**

`addin/src/AddLife.SweBridge/AddLife.SweBridge.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
    <PlatformTarget>x64</PlatformTarget>
    <LangVersion>latest</LangVersion>
    <AssemblyName>AddLife.SweBridge</AssemblyName>
    <RootNamespace>AddLife.SweBridge</RootNamespace>
    <Version>0.1.0</Version>
    <ComVisible>false</ComVisible>
    <SweBin Condition="'$(SweBin)' == ''">C:\Program Files\SOLIDWORKS Corp\SOLIDWORKS Electrical\bin</SweBin>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NETFramework.ReferenceAssemblies" Version="1.0.3" PrivateAssets="all" />
  </ItemGroup>
  <ItemGroup>
    <!-- Embedded like Electrical's own add-ins do: no runtime dependency on the interop DLL. -->
    <Reference Include="Interop.EwAPI">
      <HintPath>$(SweBin)\Interop.EwAPI.dll</HintPath>
      <EmbedInteropTypes>true</EmbedInteropTypes>
      <Private>false</Private>
    </Reference>
    <Reference Include="System.Web.Extensions" />
    <Reference Include="System.Windows.Forms" />
  </ItemGroup>
</Project>
```

- [ ] **Step 2: Create the test project file**

`addin/tests/AddLife.SweBridge.Tests/AddLife.SweBridge.Tests.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
    <PlatformTarget>x64</PlatformTarget>
    <LangVersion>latest</LangVersion>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NETFramework.ReferenceAssemblies" Version="1.0.3" PrivateAssets="all" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.1" />
    <PackageReference Include="xunit" Version="2.9.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\src\AddLife.SweBridge\AddLife.SweBridge.csproj" />
    <Reference Include="System.Web.Extensions" />
    <Reference Include="System.Net.Http" />
  </ItemGroup>
</Project>
```

- [ ] **Step 3: Write the failing tests for arguments, specs and configuration**

`addin/tests/AddLife.SweBridge.Tests/ArgsTests.cs`:

```csharp
using System.Collections.Generic;
using AddLife.SweBridge.Core;
using Xunit;

public class ArgsTests
{
    static Args Of(string json) => new Args(Json.ReadObject(json));

    [Fact]
    public void Reads_strings_numbers_ints_and_bools()
    {
        var a = Of("{\"s\":\"x\",\"n\":1.5,\"i\":3,\"b\":true}");
        Assert.Equal("x", a.Str("s"));
        Assert.Equal(1.5, a.Num("n"));
        Assert.Equal(3, a.Int("i"));
        Assert.True(a.OptBool("b", false));
    }

    [Fact]
    public void Optional_values_fall_back_to_defaults()
    {
        var a = Of("{}");
        Assert.Null(a.OptStr("s"));
        Assert.Equal(7.0, a.OptNum("n", 7));
        Assert.Equal(2, a.OptInt("i", 2));
        Assert.True(a.OptBool("b", true));
        Assert.Null(a.OptObj("o"));
    }

    [Fact]
    public void Missing_required_argument_names_the_argument()
    {
        var e = Assert.Throws<OpException>(() => Of("{}").Str("tag"));
        Assert.Equal("bad_args", e.Code);
        Assert.Contains("'tag'", e.Message);
    }

    [Fact]
    public void Wrong_type_is_reported()
    {
        Assert.Throws<OpException>(() => Of("{\"n\":\"abc\"}").Num("n"));
        Assert.Throws<OpException>(() => Of("{\"i\":1.5}").Int("i"));
        Assert.Throws<OpException>(() => Of("{\"s\":\"\"}").Str("s"));
        Assert.Throws<OpException>(() => Of("{\"b\":\"yes\"}").OptBool("b", false));
    }

    [Fact]
    public void Unicode_survives_json_and_args()
    {
        var a = Of("{\"d\":\"Motor Ω µ → °\"}");
        Assert.Equal("Motor Ω µ → °", a.Str("d"));
        var again = Of(Json.Write(new Dictionary<string, object> { ["d"] = a.Str("d") }));
        Assert.Equal("Motor Ω µ → °", again.Str("d"));
    }

    [Fact]
    public void Invalid_json_is_a_bad_request()
    {
        Assert.Equal("bad_request", Assert.Throws<OpException>(() => Json.ReadObject("{not json")).Code);
        Assert.Equal("bad_request", Assert.Throws<OpException>(() => Json.ReadObject("[1,2]")).Code);
    }
}
```

`addin/tests/AddLife.SweBridge.Tests/OpRegistryTests.cs`:

```csharp
using AddLife.SweBridge.Core;
using Xunit;

public class OpRegistryTests
{
    static readonly OpSpec Spec = new OpSpec("place", "test", true, true,
        new ArgSpec("x", "number", true, "x"), new ArgSpec("label", "string", false, "label"));

    [Fact]
    public void Unknown_argument_is_refused_to_catch_typos()
    {
        var e = Assert.Throws<OpException>(() => OpRegistry.Validate(Spec, new Args(Json.ReadObject("{\"x\":1,\"lable\":\"a\"}"))));
        Assert.Contains("'lable'", e.Message);
    }

    [Fact]
    public void Missing_required_argument_is_refused()
    {
        Assert.Throws<OpException>(() => OpRegistry.Validate(Spec, new Args(Json.ReadObject("{\"label\":\"a\"}"))));
    }

    [Fact]
    public void Valid_arguments_pass()
    {
        OpRegistry.Validate(Spec, new Args(Json.ReadObject("{\"x\":1}")));
    }

    [Fact]
    public void Duplicate_operation_names_are_a_programming_error()
    {
        var r = new OpRegistry();
        r.Add(new FakeOp("a", false, false, _ => null));
        Assert.Throws<System.InvalidOperationException>(() => r.Add(new FakeOp("a", false, false, _ => null)));
    }
}
```

`addin/tests/AddLife.SweBridge.Tests/BridgeConfigTests.cs`:

```csharp
using System.IO;
using AddLife.SweBridge.Core;
using Xunit;

public class BridgeConfigTests
{
    [Fact]
    public void Defaults_allow_only_scratch_projects()
    {
        var c = BridgeConfig.Load(Path.Combine(Path.GetTempPath(), "does-not-exist-" + System.Guid.NewGuid() + ".json"));
        Assert.True(c.AllowsMutation("LIBTEST"));
        Assert.True(c.AllowsMutation("LIBTEST (1)"));
        Assert.True(c.AllowsMutation("SPIKE-20261012-101500"));
        Assert.False(c.AllowsMutation("Wet Blasting Machine"));
        Assert.False(c.AllowsMutation(null));
        Assert.Equal("en", c.Language);
        Assert.False(c.ProbesEnabled);
    }

    [Fact]
    public void Reads_the_file()
    {
        var path = Path.GetTempFileName();
        File.WriteAllText(path, "{\"mutation_allowlist\":[\"WB-REBUILD\"],\"probes_enabled\":true,\"language\":\"fr\",\"port\":51234}");
        var c = BridgeConfig.Load(path);
        Assert.True(c.AllowsMutation("wb-rebuild"));
        Assert.False(c.AllowsMutation("LIBTEST"));
        Assert.True(c.ProbesEnabled);
        Assert.Equal("fr", c.Language);
        Assert.Equal(51234, c.Port);
    }
}
```

`addin/tests/AddLife.SweBridge.Tests/Fakes.cs`:

```csharp
using System;
using System.Collections.Generic;
using AddLife.SweBridge.Core;

public sealed class FakeContext : IOpContext { }

public sealed class FakeHost : IBridgeHost
{
    public string Current = "LIBTEST";
    public readonly List<string> Snapshots = new List<string>();
    public BridgeConfig Config { get; } = new BridgeConfig();
    public string CurrentProjectName() => Current;
    public string EnsureSnapshot(string session, string project)
    {
        var name = "snap-" + session + "-" + project;
        if (!Snapshots.Contains(name)) Snapshots.Add(name);
        return name;
    }
    public IOpContext Context() => new FakeContext();
    public string LastApiError() => "last api error text";
}

public sealed class FakeOp : Operation
{
    readonly Func<Args, object> _run;
    readonly Func<Args, string> _target;
    public int Calls;

    public FakeOp(string name, bool mutating, bool requiresProject, Func<Args, object> run,
        Func<Args, string> target = null, params ArgSpec[] args)
    {
        Spec = new OpSpec(name, "fake " + name, mutating, requiresProject, args);
        _run = run;
        _target = target;
    }

    public override OpSpec Spec { get; }
    public override string TargetProject(Args args, string batchProject) => _target != null ? _target(args) : batchProject;
    public override object Execute(IOpContext context, Args args) { Calls++; return _run(args); }
}
```

`addin/tests/AddLife.SweBridge.Tests/BatchRequestTests.cs`:

```csharp
using AddLife.SweBridge.Core;
using Xunit;

public class BatchRequestTests
{
    [Fact]
    public void Parses_a_full_request()
    {
        var r = BatchRequest.Parse(Json.ReadObject(
            "{\"batch_id\":\"b-1\",\"session\":\"s.1\",\"project\":\"LIBTEST\",\"stop_on_error\":false," +
            "\"ops\":[{\"op\":\"a\",\"args\":{\"x\":1}},{\"op\":\"b\"}]}"));
        Assert.Equal("b-1", r.BatchId);
        Assert.Equal("s.1", r.Session);
        Assert.Equal("LIBTEST", r.Project);
        Assert.False(r.StopOnError);
        Assert.Equal(2, r.Ops.Count);
        Assert.Equal(1, r.Ops[0].Args.Int("x"));
        Assert.False(r.Ops[1].Args.Has("x"));
    }

    [Fact]
    public void Project_is_optional_and_stop_on_error_defaults_to_true()
    {
        var r = BatchRequest.Parse(Json.ReadObject("{\"batch_id\":\"b\",\"session\":\"s\",\"ops\":[{\"op\":\"a\"}]}"));
        Assert.Null(r.Project);
        Assert.True(r.StopOnError);
    }

    [Theory]
    [InlineData("{\"session\":\"s\",\"ops\":[{\"op\":\"a\"}]}")]
    [InlineData("{\"batch_id\":\"b\",\"ops\":[{\"op\":\"a\"}]}")]
    [InlineData("{\"batch_id\":\"has space\",\"session\":\"s\",\"ops\":[{\"op\":\"a\"}]}")]
    [InlineData("{\"batch_id\":\"b\",\"session\":\"s\",\"ops\":[]}")]
    [InlineData("{\"batch_id\":\"b\",\"session\":\"s\",\"ops\":[\"a\"]}")]
    public void Rejects_malformed_requests(string json)
    {
        Assert.Throws<OpException>(() => BatchRequest.Parse(Json.ReadObject(json)));
    }
}
```

`addin/tests/AddLife.SweBridge.Tests/BatchRunnerTests.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;
using AddLife.SweBridge.Core;
using Xunit;

public class BatchRunnerTests
{
    readonly FakeHost _host = new FakeHost();
    readonly OpRegistry _registry = new OpRegistry();
    readonly FakeOp _read, _write, _fail, _echo, _create;

    public BatchRunnerTests()
    {
        _read = new FakeOp("read", false, true, _ => "r");
        _write = new FakeOp("write", true, true, _ => "w");
        _fail = new FakeOp("fail", true, true, _ => throw new OpException("EW_BAD_INPUTS", "bad"));
        _echo = new FakeOp("echo", false, false, a => a.Str("text"), null, new ArgSpec("text", "string", true, ""));
        _create = new FakeOp("create", true, false, a => "c", a => a.Str("name"), new ArgSpec("name", "string", true, ""));
        foreach (var op in new[] { _read, _write, _fail, _echo, _create }) _registry.Add(op);
    }

    BatchResponse Run(string project, bool stop, string batchId, params (string op, string args)[] ops) =>
        new BatchRunner(_registry, _host, new BatchCache(50)).Run(Request(project, stop, batchId, ops));

    static BatchRequest Request(string project, bool stop, string batchId, params (string op, string args)[] ops) =>
        new BatchRequest(batchId, "s1", project, stop,
            ops.Select(o => new BatchOp(o.op, new Args(Json.ReadObject(o.args ?? "{}")))).ToList());

    static string[] Statuses(BatchResponse r) => r.Results.Select(x => x.Status).ToArray();

    [Fact]
    public void Runs_ops_in_order_and_reports_ok()
    {
        var r = Run("LIBTEST", true, "b1", ("read", null), ("write", null));
        Assert.Equal(new[] { "ok", "ok" }, Statuses(r));
        Assert.Equal("w", r.Results[1].Result);
        Assert.Null(r.Refusal);
    }

    [Fact]
    public void Stops_on_first_error_and_skips_the_rest()
    {
        var r = Run("LIBTEST", true, "b1", ("write", null), ("fail", null), ("write", null));
        Assert.Equal(new[] { "ok", "error", "skipped" }, Statuses(r));
        Assert.Equal("EW_BAD_INPUTS", r.Results[1].Error["code"]);
        Assert.Equal(1, _write.Calls);
    }

    [Fact]
    public void Continues_after_error_when_stop_on_error_is_false()
    {
        var r = Run("LIBTEST", false, "b1", ("fail", null), ("write", null));
        Assert.Equal(new[] { "error", "ok" }, Statuses(r));
    }

    [Fact]
    public void Attaches_last_api_error_to_EW_errors()
    {
        var r = Run("LIBTEST", true, "b1", ("fail", null));
        Assert.Equal("last api error text", r.Results[0].Error["last_error"]);
    }

    [Fact]
    public void Refuses_whole_batch_for_unknown_op_before_running_anything()
    {
        var r = Run("LIBTEST", true, "b1", ("write", null), ("nope", null));
        Assert.Equal("unknown_op", r.Refusal["code"]);
        Assert.Equal(new[] { "skipped", "skipped" }, Statuses(r));
        Assert.Equal(0, _write.Calls);
    }

    [Fact]
    public void Refuses_whole_batch_for_bad_arguments_before_running_anything()
    {
        var r = Run("LIBTEST", true, "b1", ("write", null), ("echo", "{\"txet\":\"a\"}"));
        Assert.Equal("bad_args", r.Refusal["code"]);
        Assert.Equal(0, _write.Calls);
    }

    [Fact]
    public void Refuses_mutation_on_project_not_in_allowlist()
    {
        _host.Current = "Wet Blasting Machine";
        var r = Run("Wet Blasting Machine", true, "b1", ("read", null), ("write", null));
        Assert.Equal("project_not_allowlisted", r.Refusal["code"]);
        Assert.Equal(0, _read.Calls);
        Assert.Empty(_host.Snapshots);
    }

    [Fact]
    public void Read_only_batch_on_any_project_is_allowed()
    {
        _host.Current = "Wet Blasting Machine";
        var r = Run("Wet Blasting Machine", true, "b1", ("read", null));
        Assert.Equal(new[] { "ok" }, Statuses(r));
    }

    [Fact]
    public void Project_creating_op_is_checked_against_its_own_target()
    {
        Assert.Equal("project_not_allowlisted", Run(null, true, "b1", ("create", "{\"name\":\"Customer X\"}")).Refusal["code"]);
        Assert.Equal(new[] { "ok" }, Statuses(Run(null, true, "b2", ("create", "{\"name\":\"LIBTEST\"}"))));
    }

    [Fact]
    public void Refuses_op_when_current_project_differs_from_batch_project()
    {
        _host.Current = "SPIKE-1";
        var r = Run("LIBTEST", true, "b1", ("write", null));
        Assert.Equal("project_mismatch", r.Results[0].Error["code"]);
        Assert.Equal(0, _write.Calls);
        Assert.Empty(_host.Snapshots);
    }

    [Fact]
    public void Takes_one_snapshot_before_first_mutation_only()
    {
        var r = Run("LIBTEST", true, "b1", ("read", null), ("write", null), ("write", null));
        Assert.Single(_host.Snapshots);
        Assert.Equal(_host.Snapshots[0], r.Snapshot);
    }

    [Fact]
    public void Read_only_batch_takes_no_snapshot()
    {
        var r = Run("LIBTEST", true, "b1", ("read", null));
        Assert.Empty(_host.Snapshots);
        Assert.Null(r.Snapshot);
    }

    [Fact]
    public void Same_batch_id_returns_cached_result_without_rerunning()
    {
        var runner = new BatchRunner(_registry, _host, new BatchCache(50));
        var first = runner.Run(Request("LIBTEST", true, "same", ("write", null)));
        var second = runner.Run(Request("LIBTEST", true, "same", ("write", null)));
        Assert.Equal(1, _write.Calls);
        Assert.False(first.Cached);
        Assert.True(second.Cached);
        Assert.Equal("ok", second.Results[0].Status);
    }

    [Fact]
    public void Unicode_arguments_round_trip()
    {
        var r = Run("LIBTEST", true, "b1", ("echo", "{\"text\":\"Ω µ → °\"}"));
        var json = Json.Write(r.ToJson());
        var back = Json.ReadObject(json);
        var result = (Dictionary<string, object>)((object[])back["results"])[0];
        Assert.Equal("Ω µ → °", result["result"]);
    }
}
```

- [ ] **Step 4: Run the tests to see them fail**

Run: `dotnet test addin\tests\AddLife.SweBridge.Tests -c Release`
Expected: build errors — `OpException`, `Json`, `Args`, `BatchRunner` and the other `Core` types do not exist yet.

- [ ] **Step 5: Implement the core types**

`addin/src/AddLife.SweBridge/Core/OpException.cs`:

```csharp
using System;

namespace AddLife.SweBridge.Core
{
    /// <summary>A failure the caller should see: a stable code plus a human sentence.</summary>
    public sealed class OpException : Exception
    {
        public OpException(string code, string message, string lastError = null) : base(message)
        {
            Code = code;
            LastError = lastError;
        }

        public string Code { get; }

        /// <summary>Electrical's own last-error text, when the failure came from the EwAPI.</summary>
        public string LastError { get; }
    }
}
```

`addin/src/AddLife.SweBridge/Core/Json.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Web.Script.Serialization;

namespace AddLife.SweBridge.Core
{
    /// <summary>JSON through the framework's own serializer, so the add-in ships no extra DLLs.</summary>
    public static class Json
    {
        static JavaScriptSerializer NewSerializer() =>
            new JavaScriptSerializer { MaxJsonLength = int.MaxValue, RecursionLimit = 256 };

        public static string Write(object value) => NewSerializer().Serialize(value);

        public static Dictionary<string, object> ReadObject(string text)
        {
            object parsed;
            try
            {
                parsed = NewSerializer().DeserializeObject(text ?? "");
            }
            catch (Exception e) when (!(e is OpException))
            {
                throw new OpException("bad_request", "body is not valid JSON: " + e.Message);
            }
            if (parsed is Dictionary<string, object> obj) return obj;
            throw new OpException("bad_request", "body must be a JSON object");
        }
    }
}
```

`addin/src/AddLife.SweBridge/Core/Args.cs`:

```csharp
using System;
using System.Collections.Generic;

namespace AddLife.SweBridge.Core
{
    /// <summary>Typed, validated access to one operation's JSON arguments.</summary>
    public sealed class Args
    {
        readonly Dictionary<string, object> _values;

        public Args(Dictionary<string, object> values)
        {
            _values = values ?? new Dictionary<string, object>();
        }

        public IEnumerable<string> Keys => _values.Keys;

        public bool Has(string key) => _values.TryGetValue(key, out var v) && v != null;

        public string Str(string key)
        {
            if (Raw(key) is string s && s.Length > 0) return s;
            throw Bad(key, "a non-empty string");
        }

        public string OptStr(string key, string fallback = null) => Has(key) ? Str(key) : fallback;

        public double Num(string key)
        {
            switch (Raw(key))
            {
                case int i: return i;
                case long l: return l;
                case decimal m: return (double)m;
                case double d when !double.IsNaN(d) && !double.IsInfinity(d): return d;
                default: throw Bad(key, "a number");
            }
        }

        public double OptNum(string key, double fallback) => Has(key) ? Num(key) : fallback;

        public int Int(string key)
        {
            var d = Num(key);
            if (d != Math.Floor(d) || d < int.MinValue || d > int.MaxValue) throw Bad(key, "an integer");
            return (int)d;
        }

        public int OptInt(string key, int fallback) => Has(key) ? Int(key) : fallback;

        public bool OptBool(string key, bool fallback)
        {
            if (!Has(key)) return fallback;
            if (Raw(key) is bool b) return b;
            throw Bad(key, "true or false");
        }

        public Dictionary<string, object> OptObj(string key)
        {
            if (!Has(key)) return null;
            return Raw(key) as Dictionary<string, object> ?? throw Bad(key, "an object");
        }

        public object[] OptArr(string key)
        {
            if (!Has(key)) return null;
            return Raw(key) as object[] ?? throw Bad(key, "an array");
        }

        object Raw(string key)
        {
            if (_values.TryGetValue(key, out var v) && v != null) return v;
            throw new OpException("bad_args", $"missing argument '{key}'");
        }

        static OpException Bad(string key, string wanted) =>
            new OpException("bad_args", $"argument '{key}' must be {wanted}");
    }
}
```

`addin/src/AddLife.SweBridge/Core/OpSpec.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;

namespace AddLife.SweBridge.Core
{
    public sealed class ArgSpec
    {
        public ArgSpec(string name, string type, bool required, string doc)
        {
            Name = name; Type = type; Required = required; Doc = doc;
        }

        public string Name { get; }
        public string Type { get; }
        public bool Required { get; }
        public string Doc { get; }

        public Dictionary<string, object> ToJson() => new Dictionary<string, object>
        {
            ["name"] = Name, ["type"] = Type, ["required"] = Required, ["doc"] = Doc,
        };
    }

    /// <summary>What an operation is called, what it changes, and which arguments it takes.</summary>
    public sealed class OpSpec
    {
        public OpSpec(string name, string summary, bool mutating, bool requiresProject, params ArgSpec[] args)
        {
            Name = name; Summary = summary; Mutating = mutating; RequiresProject = requiresProject;
            Args = args ?? new ArgSpec[0];
        }

        public string Name { get; }
        public string Summary { get; }
        /// <summary>Changes Electrical data: checked against the allowlist, covered by a snapshot.</summary>
        public bool Mutating { get; }
        /// <summary>Acts on the current project, which must be the batch's project.</summary>
        public bool RequiresProject { get; }
        public IReadOnlyList<ArgSpec> Args { get; }

        public Dictionary<string, object> ToJson() => new Dictionary<string, object>
        {
            ["name"] = Name,
            ["summary"] = Summary,
            ["mutating"] = Mutating,
            ["requires_project"] = RequiresProject,
            ["args"] = Args.Select(a => a.ToJson()).ToList(),
        };
    }
}
```

`addin/src/AddLife.SweBridge/Core/IOperation.cs`:

```csharp
namespace AddLife.SweBridge.Core
{
    /// <summary>Whatever an operation needs to reach Electrical; the live one is EwContext.</summary>
    public interface IOpContext { }

    public interface IOperation
    {
        OpSpec Spec { get; }

        /// <summary>The project a mutating operation would change; checked against the allowlist.</summary>
        string TargetProject(Args args, string batchProject);

        object Execute(IOpContext context, Args args);
    }

    public abstract class Operation : IOperation
    {
        public abstract OpSpec Spec { get; }
        public virtual string TargetProject(Args args, string batchProject) => batchProject;
        public abstract object Execute(IOpContext context, Args args);

        protected static ArgSpec Req(string name, string type, string doc) => new ArgSpec(name, type, true, doc);
        protected static ArgSpec Opt(string name, string type, string doc) => new ArgSpec(name, type, false, doc);
    }
}
```

`addin/src/AddLife.SweBridge/Core/OpRegistry.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace AddLife.SweBridge.Core
{
    public sealed class OpRegistry
    {
        readonly Dictionary<string, IOperation> _ops = new Dictionary<string, IOperation>(StringComparer.Ordinal);

        public void Add(IOperation op)
        {
            if (_ops.ContainsKey(op.Spec.Name)) throw new InvalidOperationException("duplicate operation " + op.Spec.Name);
            _ops.Add(op.Spec.Name, op);
        }

        public IOperation Find(string name) => _ops.TryGetValue(name, out var op) ? op : null;

        public IEnumerable<OpSpec> Specs => _ops.Values.Select(o => o.Spec).OrderBy(s => s.Name, StringComparer.Ordinal);

        /// <summary>Refuses unknown arguments (typos) and missing required ones before anything runs.</summary>
        public static void Validate(OpSpec spec, Args args)
        {
            foreach (var key in args.Keys)
                if (!spec.Args.Any(a => a.Name == key))
                    throw new OpException("bad_args", $"unknown argument '{key}' for {spec.Name}");
            foreach (var arg in spec.Args.Where(a => a.Required))
                if (!args.Has(arg.Name))
                    throw new OpException("bad_args", $"missing argument '{arg.Name}' for {spec.Name}");
        }
    }
}
```

`addin/src/AddLife.SweBridge/Core/BridgePaths.cs`:

```csharp
using System;
using System.IO;

namespace AddLife.SweBridge.Core
{
    /// <summary>%LOCALAPPDATA%\AddLife\swe-bridge — secrets and machine state, never in git.</summary>
    public static class BridgePaths
    {
        public static string Root { get; set; } = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "AddLife", "swe-bridge");

        public static string Session => Path.Combine(Root, "session.json");
        public static string Config => Path.Combine(Root, "config.json");
        public static string License => Path.Combine(Root, "license.txt");
        public static string Log => Path.Combine(Root, "bridge.log");
    }
}
```

`addin/src/AddLife.SweBridge/Core/BridgeLog.cs`:

```csharp
using System;
using System.IO;
using System.Text;

namespace AddLife.SweBridge.Core
{
    /// <summary>Append-only log; never throws, rotates at 5 MB.</summary>
    public static class BridgeLog
    {
        static readonly object Gate = new object();

        public static void Info(string message) => Write("INFO", message);
        public static void Error(string message) => Write("ERROR", message);

        static void Write(string level, string message)
        {
            try
            {
                lock (Gate)
                {
                    Directory.CreateDirectory(BridgePaths.Root);
                    var log = new FileInfo(BridgePaths.Log);
                    if (log.Exists && log.Length > 5_000_000)
                    {
                        var old = BridgePaths.Log + ".1";
                        if (File.Exists(old)) File.Delete(old);
                        File.Move(BridgePaths.Log, old);
                    }
                    File.AppendAllText(BridgePaths.Log,
                        $"{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff} {level} {message}\r\n", Encoding.UTF8);
                }
            }
            catch
            {
                // Logging must never take Electrical down.
            }
        }
    }
}
```

`addin/src/AddLife.SweBridge/Core/BridgeConfig.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text;

namespace AddLife.SweBridge.Core
{
    public sealed class BridgeConfig
    {
        /// <summary>Projects the bridge may change. "X*" matches names starting with X. Case-insensitive.</summary>
        public List<string> MutationAllowlist { get; set; } = new List<string> { "LIBTEST*", "SPIKE-*" };
        public bool ProbesEnabled { get; set; }
        /// <summary>Language code for descriptions and texts (confirmed in Phase 0, finding S2).</summary>
        public string Language { get; set; } = "en";
        /// <summary>0 = pick a free port at every start.</summary>
        public int Port { get; set; }

        public static BridgeConfig Load(string path)
        {
            var config = new BridgeConfig();
            if (!File.Exists(path)) return config;
            var values = Json.ReadObject(File.ReadAllText(path, Encoding.UTF8));
            var args = new Args(values);
            var list = args.OptArr("mutation_allowlist");
            if (list != null)
                config.MutationAllowlist = list.OfType<string>().Where(s => !string.IsNullOrWhiteSpace(s)).ToList();
            config.ProbesEnabled = args.OptBool("probes_enabled", false);
            config.Language = args.OptStr("language", "en");
            config.Port = args.OptInt("port", 0);
            return config;
        }

        public bool AllowsMutation(string project) =>
            !string.IsNullOrEmpty(project) && MutationAllowlist.Any(pattern => Matches(pattern, project));

        static bool Matches(string pattern, string name) =>
            pattern.EndsWith("*", StringComparison.Ordinal)
                ? name.StartsWith(pattern.Substring(0, pattern.Length - 1), StringComparison.OrdinalIgnoreCase)
                : string.Equals(pattern, name, StringComparison.OrdinalIgnoreCase);
    }
}
```

`addin/src/AddLife.SweBridge/Core/IBridgeHost.cs`:

```csharp
namespace AddLife.SweBridge.Core
{
    /// <summary>What the batch runner needs from Electrical; EwHost is the live one, FakeHost the test one.</summary>
    public interface IBridgeHost
    {
        BridgeConfig Config { get; }

        /// <summary>Name of the project open in Electrical, or null.</summary>
        string CurrentProjectName();

        /// <summary>Takes a snapshot of the current project once per (session, project); returns its name.</summary>
        string EnsureSnapshot(string session, string project);

        IOpContext Context();

        /// <summary>Electrical's last-error text, or null.</summary>
        string LastApiError();
    }
}
```

`addin/src/AddLife.SweBridge/Core/BatchRequest.cs`:

```csharp
using System.Collections.Generic;
using System.Text.RegularExpressions;

namespace AddLife.SweBridge.Core
{
    public sealed class BatchOp
    {
        public BatchOp(string op, Args args) { Op = op; Args = args; }
        public string Op { get; }
        public Args Args { get; }
    }

    public sealed class BatchRequest
    {
        const int MaxOps = 5000;
        static readonly Regex IdPattern = new Regex("^[A-Za-z0-9_.-]{1,64}$");

        public BatchRequest(string batchId, string session, string project, bool stopOnError, IReadOnlyList<BatchOp> ops)
        {
            BatchId = batchId; Session = session; Project = project; StopOnError = stopOnError; Ops = ops;
        }

        public string BatchId { get; }
        public string Session { get; }
        /// <summary>The project the batch is meant for; null when no operation needs one.</summary>
        public string Project { get; }
        public bool StopOnError { get; }
        public IReadOnlyList<BatchOp> Ops { get; }

        public static BatchRequest Parse(Dictionary<string, object> body)
        {
            var args = new Args(body);
            var batchId = args.Str("batch_id");
            var session = args.Str("session");
            if (!IdPattern.IsMatch(batchId)) throw new OpException("bad_request", "batch_id must be 1-64 characters of A-Z a-z 0-9 _ . -");
            if (!IdPattern.IsMatch(session)) throw new OpException("bad_request", "session must be 1-64 characters of A-Z a-z 0-9 _ . -");
            var items = args.OptArr("ops");
            if (items == null || items.Length == 0) throw new OpException("bad_request", "ops must be a non-empty array");
            if (items.Length > MaxOps) throw new OpException("bad_request", $"at most {MaxOps} operations per batch");

            var ops = new List<BatchOp>();
            for (var i = 0; i < items.Length; i++)
            {
                if (!(items[i] is Dictionary<string, object> item)) throw new OpException("bad_request", $"ops[{i}] must be an object");
                var itemArgs = new Args(item);
                ops.Add(new BatchOp(itemArgs.Str("op"), new Args(itemArgs.OptObj("args"))));
            }
            return new BatchRequest(batchId, session, args.OptStr("project"), args.OptBool("stop_on_error", true), ops);
        }
    }
}
```

`addin/src/AddLife.SweBridge/Core/BatchResponse.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;

namespace AddLife.SweBridge.Core
{
    public sealed class OpResult
    {
        OpResult(string op, string status, object result, Dictionary<string, object> error)
        {
            Op = op; Status = status; Result = result; Error = error;
        }

        public string Op { get; }
        /// <summary>"ok", "error" or "skipped".</summary>
        public string Status { get; }
        public object Result { get; }
        public Dictionary<string, object> Error { get; }

        public static OpResult Ok(string op, object result) => new OpResult(op, "ok", result, null);

        public static OpResult Failed(string op, string code, string message, string lastError) =>
            new OpResult(op, "error", null, new Dictionary<string, object>
            {
                ["code"] = code, ["message"] = message, ["last_error"] = lastError,
            });

        public static OpResult Skipped(string op) => new OpResult(op, "skipped", null, null);

        public Dictionary<string, object> ToJson()
        {
            var json = new Dictionary<string, object> { ["op"] = Op, ["status"] = Status };
            if (Result != null) json["result"] = Result;
            if (Error != null) json["error"] = Error;
            return json;
        }
    }

    public sealed class BatchResponse
    {
        public BatchResponse(string batchId, string project, string snapshot, Dictionary<string, object> refusal,
            IReadOnlyList<OpResult> results, long elapsedMs, bool cached = false)
        {
            BatchId = batchId; Project = project; Snapshot = snapshot; Refusal = refusal;
            Results = results; ElapsedMs = elapsedMs; Cached = cached;
        }

        public string BatchId { get; }
        public string Project { get; }
        /// <summary>Name of the snapshot covering this batch's changes, or null if nothing changed.</summary>
        public string Snapshot { get; }
        /// <summary>Set when the whole batch was refused before anything ran.</summary>
        public Dictionary<string, object> Refusal { get; }
        public IReadOnlyList<OpResult> Results { get; }
        public long ElapsedMs { get; }
        /// <summary>True when this is a repeat of an already-executed batch_id.</summary>
        public bool Cached { get; }

        public BatchResponse AsCached() => new BatchResponse(BatchId, Project, Snapshot, Refusal, Results, ElapsedMs, true);

        public Dictionary<string, object> ToJson() => new Dictionary<string, object>
        {
            ["batch_id"] = BatchId,
            ["cached"] = Cached,
            ["project"] = Project,
            ["snapshot"] = Snapshot,
            ["refusal"] = Refusal,
            ["results"] = Results.Select(r => r.ToJson()).ToList(),
            ["elapsed_ms"] = ElapsedMs,
        };
    }
}
```

`addin/src/AddLife.SweBridge/Core/BatchCache.cs`:

```csharp
using System.Collections.Generic;

namespace AddLife.SweBridge.Core
{
    /// <summary>Remembers the last N executed batches so a retried batch_id is answered, not re-run.</summary>
    public sealed class BatchCache
    {
        readonly int _capacity;
        readonly Queue<string> _order = new Queue<string>();
        readonly Dictionary<string, BatchResponse> _items = new Dictionary<string, BatchResponse>();
        readonly object _gate = new object();

        public BatchCache(int capacity) { _capacity = capacity; }

        public BatchResponse Get(string batchId)
        {
            lock (_gate) return _items.TryGetValue(batchId, out var r) ? r : null;
        }

        public void Put(string batchId, BatchResponse response)
        {
            lock (_gate)
            {
                if (_items.ContainsKey(batchId)) return;
                _items[batchId] = response;
                _order.Enqueue(batchId);
                while (_order.Count > _capacity) _items.Remove(_order.Dequeue());
            }
        }
    }
}
```

`addin/src/AddLife.SweBridge/Core/BatchRunner.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;

namespace AddLife.SweBridge.Core
{
    /// <summary>
    /// Runs a batch: refuses it whole if any operation is unknown, malformed or not allowed;
    /// otherwise runs the operations in order, snapshotting before the first change.
    /// </summary>
    public sealed class BatchRunner
    {
        readonly OpRegistry _registry;
        readonly IBridgeHost _host;
        readonly BatchCache _cache;

        public BatchRunner(OpRegistry registry, IBridgeHost host, BatchCache cache)
        {
            _registry = registry; _host = host; _cache = cache;
        }

        public BatchResponse Run(BatchRequest request)
        {
            var cached = _cache.Get(request.BatchId);
            if (cached != null) return cached.AsCached();

            var clock = Stopwatch.StartNew();
            var planned = new List<(IOperation Op, Args Args)>();
            foreach (var item in request.Ops)
            {
                var op = _registry.Find(item.Op);
                if (op == null) return Refuse(request, clock, "unknown_op", $"unknown operation '{item.Op}'");
                try
                {
                    OpRegistry.Validate(op.Spec, item.Args);
                    if (op.Spec.Mutating)
                    {
                        var target = op.TargetProject(item.Args, request.Project);
                        if (!_host.Config.AllowsMutation(target))
                            return Refuse(request, clock, "project_not_allowlisted",
                                $"{item.Op} would change '{target ?? "(no project)"}', which is not in mutation_allowlist");
                    }
                }
                catch (OpException e)
                {
                    return Refuse(request, clock, e.Code, e.Message);
                }
                planned.Add((op, item.Args));
            }

            var results = new List<OpResult>();
            string snapshot = null;
            var stopped = false;
            foreach (var (op, args) in planned)
            {
                var name = op.Spec.Name;
                if (stopped)
                {
                    results.Add(OpResult.Skipped(name));
                    continue;
                }
                try
                {
                    if (op.Spec.RequiresProject)
                    {
                        var current = _host.CurrentProjectName();
                        if (!string.Equals(current, request.Project, StringComparison.OrdinalIgnoreCase))
                            throw new OpException("project_mismatch",
                                $"batch targets '{request.Project ?? "(none)"}' but Electrical's current project is '{current ?? "(none)"}'");
                        if (op.Spec.Mutating && snapshot == null)
                            snapshot = _host.EnsureSnapshot(request.Session, request.Project);
                    }
                    results.Add(OpResult.Ok(name, op.Execute(_host.Context(), args)));
                }
                catch (OpException e)
                {
                    var lastError = e.LastError ?? (e.Code.StartsWith("EW_", StringComparison.Ordinal) ? _host.LastApiError() : null);
                    results.Add(OpResult.Failed(name, e.Code, e.Message, lastError));
                    stopped = request.StopOnError;
                }
                catch (Exception e)
                {
                    results.Add(OpResult.Failed(name, "exception", e.GetType().Name + ": " + e.Message, _host.LastApiError()));
                    stopped = request.StopOnError;
                }
            }

            var response = new BatchResponse(request.BatchId, request.Project, snapshot, null, results, clock.ElapsedMilliseconds);
            _cache.Put(request.BatchId, response);
            return response;
        }

        static BatchResponse Refuse(BatchRequest request, Stopwatch clock, string code, string message) =>
            new BatchResponse(request.BatchId, request.Project, null,
                new Dictionary<string, object> { ["code"] = code, ["message"] = message },
                request.Ops.Select(o => OpResult.Skipped(o.Op)).ToList(), clock.ElapsedMilliseconds);
    }
}
```

- [ ] **Step 6: Run the tests to see them pass**

Run: `dotnet test addin\tests\AddLife.SweBridge.Tests -c Release`
Expected: all tests in `ArgsTests`, `OpRegistryTests`, `BridgeConfigTests`, `BatchRequestTests`, `BatchRunnerTests` pass; 0 failed.

- [ ] **Step 7: Commit**

```powershell
git add addin/src addin/tests
git commit -m "feat(addin): operation core, configuration and batch runner"
```

### Task 0.4: Loopback HTTP server, bridge service and session file

A minimal HTTP/1.1 server on `TcpListener` (no `HttpListener`, so no URL ACL and no admin rights), the routing/auth layer, and the `session.json` writer.

**Files:**
- Create: `addin/src/AddLife.SweBridge/Core/MiniHttpServer.cs`, `BridgeService.cs`, `SessionFile.cs`
- Test: `addin/tests/AddLife.SweBridge.Tests/Http.cs`, `MiniHttpServerTests.cs`, `BridgeServiceTests.cs`, `SessionFileTests.cs`

**Interfaces:**
- Consumes: `Json`, `OpException`, `BatchRequest`, `BatchRunner`, `OpRegistry`, `BridgePaths`, `BridgeLog` (Task 0.3)
- Produces (namespace `AddLife.SweBridge.Core`):
  - `HttpReq { Method, Path, Headers, Body }`, `HttpResp(int status, string body)`
  - `MiniHttpServer(int port, Func<HttpReq, HttpResp> handler)`: `Start()`, `Port`, `Dispose()`
  - `IUiDispatcher { T Invoke<T>(Func<T>); int QueueDepth; }`
  - `BridgeService(string token, BatchRunner, OpRegistry, IUiDispatcher, Func<Dictionary<string, object>> health, string bridgeVersion).Handle(HttpReq) : HttpResp` — routes `GET /health`, `GET /schema`, `POST /batch`; 401 without the right `X-SWE-Token`
  - `SessionFile.Write(port, token, ewapiVersion, bridgeVersion)`, `SessionFile.Delete()`, `SessionFile.NewToken()`

- [ ] **Step 1: Write the failing tests**

`addin/tests/AddLife.SweBridge.Tests/Http.cs`:

```csharp
using System.IO;
using System.Net;
using System.Text;

public static class Http
{
    public static (int Status, string Body) Send(int port, string method, string path, string body = null, string token = null)
    {
        var request = (HttpWebRequest)WebRequest.Create($"http://127.0.0.1:{port}{path}");
        request.Method = method;
        request.Proxy = null;
        request.Timeout = 10000;
        if (token != null) request.Headers["X-SWE-Token"] = token;
        if (body != null)
        {
            var bytes = Encoding.UTF8.GetBytes(body);
            request.ContentType = "application/json; charset=utf-8";
            request.ContentLength = bytes.Length;
            using (var s = request.GetRequestStream()) s.Write(bytes, 0, bytes.Length);
        }
        try
        {
            using (var response = (HttpWebResponse)request.GetResponse())
            using (var reader = new StreamReader(response.GetResponseStream(), Encoding.UTF8))
                return ((int)response.StatusCode, reader.ReadToEnd());
        }
        catch (WebException e) when (e.Response is HttpWebResponse error)
        {
            using (error)
            using (var reader = new StreamReader(error.GetResponseStream(), Encoding.UTF8))
                return ((int)error.StatusCode, reader.ReadToEnd());
        }
    }
}
```

`addin/tests/AddLife.SweBridge.Tests/MiniHttpServerTests.cs`:

```csharp
using System.Net.Sockets;
using System.Text;
using AddLife.SweBridge.Core;
using Xunit;

public class MiniHttpServerTests
{
    [Fact]
    public void Utf8_body_round_trips()
    {
        using (var server = new MiniHttpServer(0, r => new HttpResp(200, r.Method + " " + r.Path + " " + r.Body)))
        {
            server.Start();
            var (status, body) = Http.Send(server.Port, "POST", "/echo", "Ω µ → °");
            Assert.Equal(200, status);
            Assert.Equal("POST /echo Ω µ → °", body);
        }
    }

    [Fact]
    public void Headers_are_case_insensitive()
    {
        using (var server = new MiniHttpServer(0, r => new HttpResp(200, r.Headers["x-swe-token"])))
        {
            server.Start();
            Assert.Equal("abc", Http.Send(server.Port, "GET", "/", token: "abc").Body);
        }
    }

    [Fact]
    public void Handler_exception_becomes_500_json()
    {
        using (var server = new MiniHttpServer(0, r => throw new System.InvalidOperationException("boom")))
        {
            server.Start();
            var (status, body) = Http.Send(server.Port, "GET", "/");
            Assert.Equal(500, status);
            Assert.Contains("boom", body);
        }
    }

    [Fact]
    public void Garbage_request_gets_400()
    {
        using (var server = new MiniHttpServer(0, r => new HttpResp(200, "{}")))
        {
            server.Start();
            using (var client = new TcpClient("127.0.0.1", server.Port))
            {
                var stream = client.GetStream();
                var bytes = Encoding.ASCII.GetBytes("NONSENSE\r\n\r\n");
                stream.Write(bytes, 0, bytes.Length);
                var buffer = new byte[256];
                var read = stream.Read(buffer, 0, buffer.Length);
                Assert.StartsWith("HTTP/1.1 400", Encoding.ASCII.GetString(buffer, 0, read));
            }
        }
    }

    [Fact]
    public void Listens_on_loopback_only()
    {
        using (var server = new MiniHttpServer(0, r => new HttpResp(200, "{}")))
        {
            server.Start();
            Assert.True(server.Port > 0);
            Assert.Equal(System.Net.IPAddress.Loopback, ((System.Net.IPEndPoint)server.LocalEndpoint).Address);
        }
    }
}
```

`addin/tests/AddLife.SweBridge.Tests/BridgeServiceTests.cs`:

```csharp
using System;
using System.Collections.Generic;
using AddLife.SweBridge.Core;
using Xunit;

public sealed class InlineDispatcher : IUiDispatcher
{
    public int Calls;
    public int QueueDepth => 0;
    public T Invoke<T>(Func<T> work) { Calls++; return work(); }
}

public class BridgeServiceTests
{
    const string Token = "secret-token";
    readonly InlineDispatcher _ui = new InlineDispatcher();
    readonly BridgeService _service;

    public BridgeServiceTests()
    {
        var registry = new OpRegistry();
        registry.Add(new FakeOp("read", false, true, _ => "r"));
        var runner = new BatchRunner(registry, new FakeHost(), new BatchCache(10));
        _service = new BridgeService(Token, runner, registry, _ui,
            () => new Dictionary<string, object> { ["current_project"] = "LIBTEST" }, "0.1.0");
    }

    HttpResp Call(string method, string path, string body = null, string token = Token) =>
        _service.Handle(new HttpReq
        {
            Method = method,
            Path = path,
            Body = body ?? "",
            Headers = token == null
                ? new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase)
                : new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase) { ["X-SWE-Token"] = token },
        });

    [Fact]
    public void Missing_or_wrong_token_is_401()
    {
        Assert.Equal(401, Call("GET", "/health", token: null).Status);
        Assert.Equal(401, Call("GET", "/health", token: "nope").Status);
        Assert.Equal(0, _ui.Calls);
    }

    [Fact]
    public void Health_runs_on_the_ui_thread_and_adds_versions()
    {
        var r = Call("GET", "/health");
        Assert.Equal(200, r.Status);
        var json = Json.ReadObject(r.Body);
        Assert.Equal("LIBTEST", json["current_project"]);
        Assert.Equal("0.1.0", json["bridge_version"]);
        Assert.Equal(1, _ui.Calls);
    }

    [Fact]
    public void Batch_runs_through_the_dispatcher()
    {
        var r = Call("POST", "/batch", "{\"batch_id\":\"b\",\"session\":\"s\",\"project\":\"LIBTEST\",\"ops\":[{\"op\":\"read\"}]}");
        Assert.Equal(200, r.Status);
        Assert.Contains("\"status\":\"ok\"", r.Body);
        Assert.Equal(1, _ui.Calls);
    }

    [Fact]
    public void Bad_json_is_400_and_never_reaches_electrical()
    {
        var r = Call("POST", "/batch", "{oops");
        Assert.Equal(400, r.Status);
        Assert.Equal(0, _ui.Calls);
    }

    [Fact]
    public void Schema_lists_operations()
    {
        var r = Call("GET", "/schema");
        Assert.Equal(200, r.Status);
        Assert.Contains("\"name\":\"read\"", r.Body);
    }

    [Fact]
    public void Unknown_route_is_404()
    {
        Assert.Equal(404, Call("GET", "/nope").Status);
    }
}
```

`addin/tests/AddLife.SweBridge.Tests/SessionFileTests.cs`:

```csharp
using System.IO;
using AddLife.SweBridge.Core;
using Xunit;

public class SessionFileTests
{
    [Fact]
    public void Writes_and_deletes_the_session_file()
    {
        BridgePaths.Root = Path.Combine(Path.GetTempPath(), "swe-bridge-test-" + System.Guid.NewGuid());
        SessionFile.Write(51234, "tok", "2026.4.1.1011", "0.1.0");
        var json = Json.ReadObject(File.ReadAllText(BridgePaths.Session));
        Assert.Equal(51234, json["port"]);
        Assert.Equal("tok", json["token"]);
        Assert.Equal("2026.4.1.1011", json["ewapi_version"]);
        SessionFile.Delete();
        Assert.False(File.Exists(BridgePaths.Session));
    }

    [Fact]
    public void Tokens_are_long_and_unique()
    {
        var a = SessionFile.NewToken();
        var b = SessionFile.NewToken();
        Assert.Equal(64, a.Length);
        Assert.NotEqual(a, b);
    }
}
```

- [ ] **Step 2: Run the tests to see them fail**

Run: `dotnet test addin\tests\AddLife.SweBridge.Tests -c Release`
Expected: build errors — `MiniHttpServer`, `BridgeService`, `SessionFile`, `HttpReq`, `HttpResp`, `IUiDispatcher` do not exist.

- [ ] **Step 3: Implement the server, service and session file**

`addin/src/AddLife.SweBridge/Core/MiniHttpServer.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading;

namespace AddLife.SweBridge.Core
{
    public sealed class HttpReq
    {
        public string Method { get; set; }
        public string Path { get; set; }
        public Dictionary<string, string> Headers { get; set; }
        public string Body { get; set; }
    }

    public sealed class HttpResp
    {
        public HttpResp(int status, string body) { Status = status; Body = body; }
        public int Status { get; }
        public string Body { get; }
    }

    /// <summary>
    /// One request per connection, JSON responses, 127.0.0.1 only. TcpListener rather than HttpListener
    /// so no URL reservation and no administrator rights are needed.
    /// </summary>
    public sealed class MiniHttpServer : IDisposable
    {
        const int MaxHeaderBytes = 64 * 1024;
        const int MaxBodyBytes = 64 * 1024 * 1024;

        readonly TcpListener _listener;
        readonly Func<HttpReq, HttpResp> _handler;
        volatile bool _stopping;

        public MiniHttpServer(int port, Func<HttpReq, HttpResp> handler)
        {
            _listener = new TcpListener(IPAddress.Loopback, port);
            _handler = handler;
        }

        public EndPoint LocalEndpoint => _listener.LocalEndpoint;
        public int Port => ((IPEndPoint)_listener.LocalEndpoint).Port;

        public void Start()
        {
            _listener.Start();
            new Thread(AcceptLoop) { IsBackground = true, Name = "swe-bridge-http" }.Start();
        }

        void AcceptLoop()
        {
            while (!_stopping)
            {
                TcpClient client;
                try { client = _listener.AcceptTcpClient(); }
                catch (ObjectDisposedException) { return; }
                catch (SocketException) { if (_stopping) return; continue; }
                ThreadPool.QueueUserWorkItem(_ => Serve(client));
            }
        }

        void Serve(TcpClient client)
        {
            using (client)
            {
                try
                {
                    client.ReceiveTimeout = 30000;
                    client.SendTimeout = 30000;
                    var stream = client.GetStream();
                    var request = ReadRequest(stream);
                    HttpResp response;
                    try
                    {
                        response = request == null
                            ? new HttpResp(400, Json.Write(Error("bad_request", "malformed HTTP request")))
                            : _handler(request);
                    }
                    catch (Exception e)
                    {
                        BridgeLog.Error("handler failed: " + e);
                        response = new HttpResp(500, Json.Write(Error("exception", e.Message)));
                    }
                    WriteResponse(stream, response);
                }
                catch (Exception e)
                {
                    BridgeLog.Error("connection failed: " + e.Message);
                }
            }
        }

        static Dictionary<string, object> Error(string code, string message) => new Dictionary<string, object>
        {
            ["error"] = new Dictionary<string, object> { ["code"] = code, ["message"] = message },
        };

        static HttpReq ReadRequest(NetworkStream stream)
        {
            var header = new List<byte>();
            while (true)
            {
                var b = stream.ReadByte();
                if (b < 0) return null;
                header.Add((byte)b);
                var n = header.Count;
                if (n >= 4 && header[n - 4] == '\r' && header[n - 3] == '\n' && header[n - 2] == '\r' && header[n - 1] == '\n') break;
                if (n > MaxHeaderBytes) return null;
            }
            var lines = Encoding.ASCII.GetString(header.ToArray()).Split(new[] { "\r\n" }, StringSplitOptions.None);
            var first = lines[0].Split(' ');
            if (first.Length != 3 || !first[2].StartsWith("HTTP/", StringComparison.Ordinal)) return null;

            var headers = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
            foreach (var line in lines.Skip(1))
            {
                var colon = line.IndexOf(':');
                if (colon > 0) headers[line.Substring(0, colon).Trim()] = line.Substring(colon + 1).Trim();
            }

            var length = 0;
            if (headers.TryGetValue("Content-Length", out var declared) &&
                (!int.TryParse(declared, out length) || length < 0 || length > MaxBodyBytes)) return null;
            var body = new byte[length];
            var read = 0;
            while (read < length)
            {
                var got = stream.Read(body, read, length - read);
                if (got <= 0) return null;
                read += got;
            }
            return new HttpReq { Method = first[0], Path = first[1], Headers = headers, Body = Encoding.UTF8.GetString(body) };
        }

        static void WriteResponse(NetworkStream stream, HttpResp response)
        {
            var body = Encoding.UTF8.GetBytes(response.Body ?? "");
            var head = Encoding.ASCII.GetBytes(
                $"HTTP/1.1 {response.Status} {Reason(response.Status)}\r\n" +
                "Content-Type: application/json; charset=utf-8\r\n" +
                $"Content-Length: {body.Length}\r\nConnection: close\r\n\r\n");
            stream.Write(head, 0, head.Length);
            stream.Write(body, 0, body.Length);
            stream.Flush();
        }

        static string Reason(int status)
        {
            switch (status)
            {
                case 200: return "OK";
                case 400: return "Bad Request";
                case 401: return "Unauthorized";
                case 404: return "Not Found";
                default: return "Internal Server Error";
            }
        }

        public void Dispose()
        {
            _stopping = true;
            try { _listener.Stop(); } catch { /* already stopped */ }
        }
    }
}
```

`addin/src/AddLife.SweBridge/Core/BridgeService.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace AddLife.SweBridge.Core
{
    /// <summary>Runs work on Electrical's UI thread; WinFormsDispatcher is the live one.</summary>
    public interface IUiDispatcher
    {
        T Invoke<T>(Func<T> work);
        int QueueDepth { get; }
    }

    /// <summary>Token check and routing. Everything that touches Electrical goes through the dispatcher.</summary>
    public sealed class BridgeService
    {
        readonly string _token;
        readonly BatchRunner _runner;
        readonly OpRegistry _registry;
        readonly IUiDispatcher _ui;
        readonly Func<Dictionary<string, object>> _health;
        readonly string _bridgeVersion;

        public BridgeService(string token, BatchRunner runner, OpRegistry registry, IUiDispatcher ui,
            Func<Dictionary<string, object>> health, string bridgeVersion)
        {
            _token = token; _runner = runner; _registry = registry; _ui = ui; _health = health; _bridgeVersion = bridgeVersion;
        }

        public HttpResp Handle(HttpReq request)
        {
            if (!request.Headers.TryGetValue("X-SWE-Token", out var token) || !TokenEquals(token, _token))
                return Reply(401, Error("unauthorized", "missing or wrong X-SWE-Token; session.json is rewritten at every Electrical start"));

            if (request.Method == "GET" && request.Path == "/health")
            {
                var health = _ui.Invoke(_health);
                health["bridge_version"] = _bridgeVersion;
                health["queue_depth"] = _ui.QueueDepth;
                return Reply(200, health);
            }
            if (request.Method == "GET" && request.Path == "/schema")
                return Reply(200, new Dictionary<string, object>
                {
                    ["bridge_version"] = _bridgeVersion,
                    ["ops"] = _registry.Specs.Select(s => s.ToJson()).ToList(),
                });
            if (request.Method == "POST" && request.Path == "/batch")
            {
                BatchRequest batch;
                try { batch = BatchRequest.Parse(Json.ReadObject(request.Body)); }
                catch (OpException e) { return Reply(400, Error(e.Code, e.Message)); }
                return Reply(200, _ui.Invoke(() => _runner.Run(batch)).ToJson());
            }
            return Reply(404, Error("not_found", request.Method + " " + request.Path));
        }

        static bool TokenEquals(string a, string b)
        {
            if (a == null || b == null || a.Length != b.Length) return false;
            var diff = 0;
            for (var i = 0; i < a.Length; i++) diff |= a[i] ^ b[i];
            return diff == 0;
        }

        static Dictionary<string, object> Error(string code, string message) => new Dictionary<string, object>
        {
            ["error"] = new Dictionary<string, object> { ["code"] = code, ["message"] = message },
        };

        static HttpResp Reply(int status, object body) => new HttpResp(status, Json.Write(body));
    }
}
```

`addin/src/AddLife.SweBridge/Core/SessionFile.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Security.Cryptography;
using System.Text;

namespace AddLife.SweBridge.Core
{
    /// <summary>session.json: how swe finds this Electrical session (port + token).</summary>
    public static class SessionFile
    {
        public static void Write(int port, string token, string ewapiVersion, string bridgeVersion)
        {
            Directory.CreateDirectory(BridgePaths.Root);
            var content = Json.Write(new Dictionary<string, object>
            {
                ["port"] = port,
                ["token"] = token,
                ["pid"] = Process.GetCurrentProcess().Id,
                ["started_at"] = DateTime.UtcNow.ToString("o"),
                ["ewapi_version"] = ewapiVersion,
                ["bridge_version"] = bridgeVersion,
            });
            File.WriteAllText(BridgePaths.Session, content, new UTF8Encoding(false));
        }

        public static void Delete()
        {
            try { File.Delete(BridgePaths.Session); } catch { /* best effort on shutdown */ }
        }

        public static string NewToken()
        {
            var bytes = new byte[32];
            using (var rng = RandomNumberGenerator.Create()) rng.GetBytes(bytes);
            return BitConverter.ToString(bytes).Replace("-", "").ToLowerInvariant();
        }
    }
}
```

- [ ] **Step 4: Run the tests to see them pass**

Run: `dotnet test addin\tests\AddLife.SweBridge.Tests -c Release`
Expected: all tests pass, including the new `MiniHttpServerTests`, `BridgeServiceTests`, `SessionFileTests`.

- [ ] **Step 5: Commit**

```powershell
git add addin/src addin/tests
git commit -m "feat(addin): loopback HTTP server, token-checked routing and session file"
```

### Task 0.5: The add-in inside Electrical — registration, licence, UI thread (S1, part of S8)

**Blocked until D-A (licence code) and D-B (admin for the one-time registration) are in hand.**

**Files:**
- Create: `addin/src/AddLife.SweBridge/Electrical/BridgeAddIn.cs`, `Registration.cs`, `LicenseCode.cs`, `ConnectEvidence.cs`, `WinFormsDispatcher.cs`, `Ew.cs`, `EwContext.cs`, `EwHost.cs`, `EwOperation.cs`, `ReadModel.cs`, `Operations.cs`, `Ops/CurrentProjectOp.cs`
- Create: `addin/deploy.ps1`, `addin/undeploy.ps1`

**Interfaces:**
- Consumes: everything in `AddLife.SweBridge.Core` (Tasks 0.3, 0.4)
- Produces (namespace `AddLife.SweBridge.Electrical`):
  - `Ew.Check(EwErrorCode, string what)` (throws `OpException` with the code name), `Ew.Array<T>(object safeArray) : T[]`
  - `EwContext(EwApplicationX, EwAPIX, BridgeConfig)` with `App`, `Api`, `Config`, `Lang`, `Env()`, `Projects()`, `AllProjects()`, `CurrentProject()`, `FindProject(name)`, `Sheet(project, id)`
  - `EwHost(EwContext) : IBridgeHost` plus `Health() : Dictionary<string, object>`
  - abstract `EwOperation : Operation` with `protected abstract object Run(EwContext ctx, Args args)`
  - `ReadModel.ProjectInfo(EwProjectX)`
  - `Operations.Build(BridgeConfig) : OpRegistry` — **the single place where every operation is registered**
  - operation `current_project` → `{id, name}` or `null`
  - `ConnectEvidence.LicenseResult`, `.ConnectThread`, `.ConnectApartment`

- [ ] **Step 1: Write the EwAPI helpers, context and host**

`addin/src/AddLife.SweBridge/Electrical/Ew.cs`:

```csharp
using System.Linq;
using AddLife.SweBridge.Core;
using EwAPI;

namespace AddLife.SweBridge.Electrical
{
    public static class Ew
    {
        /// <summary>Turns a non-zero EwErrorCode into an OpException whose code is the enum name.</summary>
        public static void Check(EwErrorCode code, string what)
        {
            if (code != EwErrorCode.EW_NO_ERROR) throw new OpException(code.ToString(), what + " returned " + code);
        }

        /// <summary>EwAPI returns collections as a SAFEARRAY of IDispatch, marshalled as object[].</summary>
        public static T[] Array<T>(object safeArray) => safeArray is object[] items ? items.Cast<T>().ToArray() : new T[0];
    }
}
```

`addin/src/AddLife.SweBridge/Electrical/EwContext.cs`:

```csharp
using System;
using System.Linq;
using AddLife.SweBridge.Core;
using EwAPI;

namespace AddLife.SweBridge.Electrical
{
    /// <summary>The live IOpContext: Electrical's application object plus lookups every operation needs.</summary>
    public sealed class EwContext : IOpContext
    {
        public EwContext(EwApplicationX app, EwAPIX api, BridgeConfig config)
        {
            App = app; Api = api; Config = config;
        }

        public EwApplicationX App { get; }
        public EwAPIX Api { get; }
        public BridgeConfig Config { get; }
        public string Lang => Config.Language;

        public EwEnvironmentX Env()
        {
            var env = App.getEwEnvironment(out var err);
            Ew.Check(err, "getEwEnvironment");
            return env;
        }

        public EwProjectManagerX Projects()
        {
            var manager = Env().getEwProjectManager(out var err);
            Ew.Check(err, "getEwProjectManager");
            return manager;
        }

        public EwProjectX[] AllProjects()
        {
            var array = Projects().getEwProjectArray(out var err);
            Ew.Check(err, "getEwProjectArray");
            return Ew.Array<EwProjectX>(array);
        }

        public EwProjectX CurrentProject()
        {
            var project = App.getEwProjectCurrent(out var err);
            if (err != EwErrorCode.EW_NO_ERROR || project == null)
                throw new OpException("EW_NO_ACTIVEPROJECT", "no project is open in Electrical");
            return project;
        }

        public EwProjectX FindProject(string name)
        {
            var hits = AllProjects().Where(p => string.Equals(p.getName(out _), name, StringComparison.Ordinal)).ToList();
            if (hits.Count == 0) throw new OpException("not_found", $"no project named '{name}'");
            if (hits.Count > 1)
                throw new OpException("ambiguous_identity",
                    $"{hits.Count} projects are named '{name}' (ids {string.Join(", ", hits.Select(p => p.getID()))})");
            return hits[0];
        }

        public EwProjectFileX Sheet(EwProjectX project, int id)
        {
            var manager = project.getEwProjectFileManager(out var err);
            Ew.Check(err, "getEwProjectFileManager");
            var sheet = manager.findEwProjectFileByID(id, out err);
            if (err != EwErrorCode.EW_NO_ERROR || sheet == null) throw new OpException("not_found", $"no sheet with id {id}");
            return sheet;
        }
    }
}
```

`addin/src/AddLife.SweBridge/Electrical/EwHost.cs`:

```csharp
using System;
using System.Collections.Generic;
using AddLife.SweBridge.Core;
using EwAPI;

namespace AddLife.SweBridge.Electrical
{
    public sealed class EwHost : IBridgeHost
    {
        readonly EwContext _ctx;
        readonly Dictionary<string, string> _snapshots = new Dictionary<string, string>();

        public EwHost(EwContext ctx) { _ctx = ctx; }

        public BridgeConfig Config => _ctx.Config;
        public IOpContext Context() => _ctx;

        public string CurrentProjectName()
        {
            var project = _ctx.App.getEwProjectCurrent(out var err);
            return err == EwErrorCode.EW_NO_ERROR && project != null ? project.getName(out _) : null;
        }

        public string LastApiError()
        {
            try { return _ctx.Api?.getUTLastError(); } catch { return null; }
        }

        public string EnsureSnapshot(string session, string project)
        {
            // Called before the first change of every batch: refuse to edit a project someone else has open (spec §5.5).
            if (_ctx.CurrentProject().isOpenByAnother(out _))
                throw new OpException("open_by_another", $"'{project}' is open for writing by another user; nothing was changed");
            var key = session + "|" + project;
            if (_snapshots.TryGetValue(key, out var existing)) return existing;

            var manager = _ctx.CurrentProject().getEwProjectSnapshotManager(out var err);
            Ew.Check(err, "getEwProjectSnapshotManager");
            var snapshot = manager.newEwProjectSnapshot(out err);
            Ew.Check(err, "newEwProjectSnapshot");
            var name = $"claude-{DateTime.Now:yyyyMMdd-HHmm}-{session}";
            Ew.Check(snapshot.setName(name), "snapshot setName");
            Ew.Check(snapshot.setDescription(_ctx.Lang, "Taken automatically before Claude's first change in session " + session),
                "snapshot setDescription");
            Ew.Check(snapshot.create(), "snapshot create");
            _snapshots[key] = name;
            BridgeLog.Info($"snapshot {name} taken on {project}");
            return name;
        }

        public Dictionary<string, object> Health()
        {
            var project = _ctx.App.getEwProjectCurrent(out var err);
            return new Dictionary<string, object>
            {
                ["app_version"] = _ctx.App.getApplicationVersion(),
                ["api_dll_version"] = _ctx.App.getApiDllVersion(),
                ["current_project"] = err == EwErrorCode.EW_NO_ERROR && project != null ? ReadModel.ProjectInfo(project) : null,
                ["language"] = _ctx.Lang,
                ["mutation_allowlist"] = _ctx.Config.MutationAllowlist,
            };
        }
    }
}
```

`addin/src/AddLife.SweBridge/Electrical/EwOperation.cs`:

```csharp
using AddLife.SweBridge.Core;

namespace AddLife.SweBridge.Electrical
{
    public abstract class EwOperation : Operation
    {
        public sealed override object Execute(IOpContext context, Args args) => Run((EwContext)context, args);
        protected abstract object Run(EwContext ctx, Args args);
    }
}
```

`addin/src/AddLife.SweBridge/Electrical/ReadModel.cs`:

```csharp
using System.Collections.Generic;
using EwAPI;

namespace AddLife.SweBridge.Electrical
{
    /// <summary>How Electrical objects are reported back as JSON. One shape per object kind.</summary>
    public static class ReadModel
    {
        public static Dictionary<string, object> ProjectInfo(EwProjectX project) => new Dictionary<string, object>
        {
            ["id"] = project.getID(),
            ["name"] = project.getName(out _),
        };
    }
}
```

`addin/src/AddLife.SweBridge/Electrical/Ops/CurrentProjectOp.cs`:

```csharp
using AddLife.SweBridge.Core;
using EwAPI;

namespace AddLife.SweBridge.Electrical.Ops
{
    public sealed class CurrentProjectOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("current_project",
            "The project open in Electrical, or null.", false, false);

        protected override object Run(EwContext ctx, Args args)
        {
            var project = ctx.App.getEwProjectCurrent(out var err);
            return err == EwErrorCode.EW_NO_ERROR && project != null ? ReadModel.ProjectInfo(project) : null;
        }
    }
}
```

`addin/src/AddLife.SweBridge/Electrical/Operations.cs`:

```csharp
using AddLife.SweBridge.Core;
using AddLife.SweBridge.Electrical.Ops;

namespace AddLife.SweBridge.Electrical
{
    /// <summary>The one place every operation is registered.</summary>
    public static class Operations
    {
        public static OpRegistry Build(BridgeConfig config)
        {
            var registry = new OpRegistry();
            registry.Add(new CurrentProjectOp());
            return registry;
        }
    }
}
```

(Task 0.7 adds the probe registration line here; Phase 1 tasks add one `registry.Add(...)` line per operation.)

- [ ] **Step 2: Write the add-in entry point, registration, licence and dispatcher**

`addin/src/AddLife.SweBridge/Electrical/ConnectEvidence.cs`:

```csharp
namespace AddLife.SweBridge.Electrical
{
    /// <summary>What happened at connect time, reported by probe_s1 (Phase 0) and the log.</summary>
    public static class ConnectEvidence
    {
        public static string LicenseResult { get; set; } = "not connected";
        public static int ConnectThread { get; set; }
        public static string ConnectApartment { get; set; }
    }
}
```

`addin/src/AddLife.SweBridge/Electrical/LicenseCode.cs`:

```csharp
using System;
using System.IO;
using System.Text;
using AddLife.SweBridge.Core;

namespace AddLife.SweBridge.Electrical
{
    /// <summary>The reseller's API licence code, kept outside git in license.txt.</summary>
    public static class LicenseCode
    {
        public static string Read()
        {
            try
            {
                return File.Exists(BridgePaths.License) ? File.ReadAllText(BridgePaths.License, Encoding.UTF8).Trim() : "";
            }
            catch (Exception e)
            {
                BridgeLog.Error("could not read license.txt: " + e.Message);
                return "";
            }
        }
    }
}
```

`addin/src/AddLife.SweBridge/Electrical/WinFormsDispatcher.cs`:

```csharp
using System;
using System.Threading;
using System.Windows.Forms;
using AddLife.SweBridge.Core;

namespace AddLife.SweBridge.Electrical
{
    /// <summary>
    /// Marshals work onto the thread that created it — Electrical's UI thread, because it is created
    /// inside connectToEwAPI. The EwAPI is single-threaded; this is also what serialises batches.
    /// </summary>
    public sealed class WinFormsDispatcher : IUiDispatcher, IDisposable
    {
        readonly Control _control;
        int _depth;

        public WinFormsDispatcher()
        {
            _control = new Control();
            _control.CreateControl();
            var _ = _control.Handle; // force the window handle on this thread
        }

        public int QueueDepth => _depth;

        public T Invoke<T>(Func<T> work)
        {
            Interlocked.Increment(ref _depth);
            try
            {
                return _control.InvokeRequired ? (T)_control.Invoke(work) : work();
            }
            finally
            {
                Interlocked.Decrement(ref _depth);
            }
        }

        public void Dispose() => _control.Dispose();
    }
}
```

`addin/src/AddLife.SweBridge/Electrical/Registration.cs`:

```csharp
using System;
using Microsoft.Win32;

namespace AddLife.SweBridge.Electrical
{
    /// <summary>The registry keys SOLIDWORKS Electrical reads to find and start add-ins (2026 API help, "Add-in sample in C#").</summary>
    public static class Registration
    {
        const string KeyRoot = @"SOFTWARE\SolidWorks\SOLIDWORKS Electrical\AddIns\";

        static string KeyFor(Type t) => KeyRoot + "{" + t.GUID.ToString() + "}";

        public static void Register(Type t, string title, string description)
        {
            using (var key = Registry.LocalMachine.CreateSubKey(KeyFor(t)))
            {
                key.SetValue(null, 0);
                key.SetValue("Description", description);
                key.SetValue("Title", title);
                key.SetValue("Path", t.Assembly.Location);
            }
            using (var key = Registry.CurrentUser.CreateSubKey(KeyFor(t)))
                key.SetValue("StartUp", 1, RegistryValueKind.DWord);
        }

        public static void Unregister(Type t)
        {
            Registry.LocalMachine.DeleteSubKeyTree(KeyFor(t), false);
            Registry.CurrentUser.DeleteSubKeyTree(KeyFor(t), false);
        }
    }
}
```

`addin/src/AddLife.SweBridge/Electrical/BridgeAddIn.cs`:

```csharp
using System;
using System.Runtime.InteropServices;
using System.Threading;
using AddLife.SweBridge.Core;
using EwAPI;

namespace AddLife.SweBridge.Electrical
{
    [ComVisible(true)]
    [Guid("6f3b8f0e-2c1d-4a7e-9b52-5d8a1c3e7f40")]
    [ProgId("AddLife.SweBridge")]
    [ClassInterface(ClassInterfaceType.None)]
    public class BridgeAddIn : EwAddIn
    {
        public const string Title = "AddLife SWE Bridge";
        static readonly string Version = typeof(BridgeAddIn).Assembly.GetName().Version.ToString();

        MiniHttpServer _server;
        WinFormsDispatcher _ui;

        public bool connectToEwAPI(object ewInteropFactory)
        {
            try
            {
                ConnectEvidence.ConnectThread = Thread.CurrentThread.ManagedThreadId;
                ConnectEvidence.ConnectApartment = Thread.CurrentThread.GetApartmentState().ToString();
                BridgeLog.Info($"connectToEwAPI on thread {ConnectEvidence.ConnectThread} ({ConnectEvidence.ConnectApartment}), bridge {Version}");

                var factory = (EwInteropFactoryX)ewInteropFactory;
                var app = factory.getEwApplication(LicenseCode.Read(), out var err);
                ConnectEvidence.LicenseResult = err.ToString();
                BridgeLog.Info("getEwApplication: " + err);
                if (err != EwErrorCode.EW_NO_ERROR || app == null) return false;

                var api = factory.getEwAPI(out err);
                if (err != EwErrorCode.EW_NO_ERROR) BridgeLog.Error("getEwAPI: " + err + " (last-error text unavailable)");

                var config = BridgeConfig.Load(BridgePaths.Config);
                var ctx = new EwContext(app, api, config);
                var host = new EwHost(ctx);
                var registry = Operations.Build(config);
                var runner = new BatchRunner(registry, host, new BatchCache(50));
                _ui = new WinFormsDispatcher();
                var token = SessionFile.NewToken();
                var service = new BridgeService(token, runner, registry, _ui, host.Health, Version);
                _server = new MiniHttpServer(config.Port, service.Handle);
                _server.Start();
                SessionFile.Write(_server.Port, token, app.getApiDllVersion(), Version);
                BridgeLog.Info($"listening on 127.0.0.1:{_server.Port}; {string.Join(", ", config.MutationAllowlist)} may be changed");
                return true;
            }
            catch (Exception e)
            {
                BridgeLog.Error("connect failed: " + e);
                return false;
            }
        }

        public bool disconnectFromEwApi()
        {
            try
            {
                _server?.Dispose();
                SessionFile.Delete();
                _ui?.Dispose();
                BridgeLog.Info("disconnected");
            }
            catch (Exception e)
            {
                BridgeLog.Error("disconnect: " + e);
            }
            return true;
        }

        [ComRegisterFunction]
        public static void RegisterFunction(Type t) =>
            Registration.Register(t, Title, "Lets Claude build and edit SOLIDWORKS Electrical projects through a local bridge.");

        [ComUnregisterFunction]
        public static void UnregisterFunction(Type t) => Registration.Unregister(t);
    }
}
```

- [ ] **Step 3: Build and run the unit tests**

Run: `dotnet test addin\tests\AddLife.SweBridge.Tests -c Release`
Expected: build succeeds (the Electrical files compile against the embedded interop); all unit tests still pass.

- [ ] **Step 4: Write the deploy scripts**

`addin/deploy.ps1`:

```powershell
<#
.SYNOPSIS
Builds the AddLife SWE Bridge add-in and installs it to C:\ProgramData\AddLife\SweBridge.
-Register (once, elevated, as the same Windows user who runs Electrical) also runs regasm /codebase,
whose ComRegisterFunction writes the SOLIDWORKS Electrical AddIns keys, and grants that user write
access to the folder so later deploys need no administrator rights.
Close SOLIDWORKS Electrical first: it locks the add-in DLL.
#>
param(
    [switch]$Register,
    [string]$Configuration = 'Release'
)
$ErrorActionPreference = 'Stop'
if (Get-Process -Name 'SOLIDWORKSElectrical' -ErrorAction SilentlyContinue) {
    throw 'SOLIDWORKS Electrical is running. Close it first: it locks the add-in DLL.'
}
$project = Join-Path $PSScriptRoot 'src\AddLife.SweBridge\AddLife.SweBridge.csproj'
dotnet build $project -c $Configuration
if ($LASTEXITCODE -ne 0) { throw 'build failed' }

$built = Join-Path $PSScriptRoot "src\AddLife.SweBridge\bin\$Configuration\net48"
$target = 'C:\ProgramData\AddLife\SweBridge'
New-Item -ItemType Directory -Force $target | Out-Null
Copy-Item (Join-Path $built 'AddLife.SweBridge.dll'), (Join-Path $built 'AddLife.SweBridge.pdb') $target -Force

if ($Register) {
    $principal = New-Object Security.Principal.WindowsPrincipal([Security.Principal.WindowsIdentity]::GetCurrent())
    if (-not $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) { throw '-Register needs an elevated PowerShell.' }
    icacls $target /grant "$($env:USERDOMAIN)\$($env:USERNAME):(OI)(CI)M" | Out-Null
    & "$env:WINDIR\Microsoft.NET\Framework64\v4.0.30319\RegAsm.exe" (Join-Path $target 'AddLife.SweBridge.dll') /codebase
    if ($LASTEXITCODE -ne 0) { throw 'regasm failed' }
}

$key = 'HKLM:\SOFTWARE\SolidWorks\SOLIDWORKS Electrical\AddIns\{6f3b8f0e-2c1d-4a7e-9b52-5d8a1c3e7f40}'
if (Test-Path $key) { Get-ItemProperty $key | Select-Object Title, Path | Format-List }
else { Write-Warning 'The add-in is not registered yet: run once, elevated, with -Register.' }
Write-Output "Deployed to $target. Start SOLIDWORKS Electrical, then check $env:LOCALAPPDATA\AddLife\swe-bridge\bridge.log"
```

`addin/undeploy.ps1`:

```powershell
<# Unregisters the add-in (elevated) and removes C:\ProgramData\AddLife\SweBridge. Close Electrical first. #>
$ErrorActionPreference = 'Stop'
if (Get-Process -Name 'SOLIDWORKSElectrical' -ErrorAction SilentlyContinue) { throw 'Close SOLIDWORKS Electrical first.' }
$dll = 'C:\ProgramData\AddLife\SweBridge\AddLife.SweBridge.dll'
if (Test-Path $dll) {
    & "$env:WINDIR\Microsoft.NET\Framework64\v4.0.30319\RegAsm.exe" $dll /unregister
    if ($LASTEXITCODE -ne 0) { throw 'regasm /unregister failed' }
}
Remove-Item 'C:\ProgramData\AddLife\SweBridge' -Recurse -Force -ErrorAction SilentlyContinue
Write-Output 'Unregistered and removed.'
```

- [ ] **Step 5: [user] Put the licence code in place and register (once, elevated)**

```powershell
# normal PowerShell: paste the reseller's code as the only line of the file
notepad "$env:LOCALAPPDATA\AddLife\swe-bridge\license.txt"
# close SOLIDWORKS Electrical, then in an ELEVATED PowerShell, same Windows user:
pwsh C:\Automation\SolidSchSkill\addin\deploy.ps1 -Register
```
Expected: `Title : AddLife SWE Bridge`, `Path : C:\ProgramData\AddLife\SweBridge\AddLife.SweBridge.dll`.

- [ ] **Step 6: [user] Start Electrical; check the add-in loaded (S1)**

```powershell
Get-Content "$env:LOCALAPPDATA\AddLife\swe-bridge\bridge.log" -Tail 5
$s = Get-Content "$env:LOCALAPPDATA\AddLife\swe-bridge\session.json" | ConvertFrom-Json
Invoke-RestMethod "http://127.0.0.1:$($s.port)/health" -Headers @{ 'X-SWE-Token' = $s.token } | ConvertTo-Json
```
Expected in the log: `getEwApplication: EW_NO_ERROR` and `listening on 127.0.0.1:<port>`. Expected from `/health`: `api_dll_version` `2026.4.1.1011`, `bridge_version` `0.1.0`.

If the log shows `EW_INVALID_LICENSE` or `EW_LICENSE_WITHOUT_API_OPTION`, S1 has failed: record the exact code and stop Phase 0 at Task 0.9 (findings). If there is no log at all, Electrical did not load the add-in: open Electrical's add-in manager, tick **AddLife SWE Bridge** with start-up on, restart Electrical, and record which of the two was needed.

- [ ] **Step 7: Prove a batch reaches the EwAPI on the right thread**

```powershell
$s = Get-Content "$env:LOCALAPPDATA\AddLife\swe-bridge\session.json" | ConvertFrom-Json
$body = '{"batch_id":"t1","session":"s1","ops":[{"op":"current_project"}]}'
Invoke-RestMethod "http://127.0.0.1:$($s.port)/batch" -Method Post -Body $body -ContentType 'application/json' -Headers @{ 'X-SWE-Token' = $s.token } | ConvertTo-Json -Depth 5
```
Expected: `results[0].status` is `ok`, `result` is the open project (or `null` with none open). Electrical stays responsive during and after the call.

- [ ] **Step 8: Commit**

```powershell
git add addin
git commit -m "feat(addin): Electrical entry point, registration, UI-thread dispatch, current_project"
```

### Task 0.6: The `swe` package — session discovery, client, `swe health` and `swe batch`

**Files:**
- Create: `swe/pyproject.toml`
- Create: `swe/src/swe/__init__.py`, `errors.py`, `session.py`, `client.py`, `jsonout.py`, `cli.py`, `commands/__init__.py`, `commands/health.py`, `commands/batch.py`
- Test: `swe/tests/conftest.py`, `swe/tests/fakebridge.py`, `swe/tests/test_client.py`, `swe/tests/test_cli.py`

**Interfaces:**
- Produces (Python):
  - `swe.errors`: `SweError`, `BridgeError(SweError)`, `BridgeNotRunning(BridgeError)`, `BridgeAuthError(BridgeError)`, `BatchFailed(BridgeError)` with `.response`
  - `swe.session`: `bridge_dir() -> Path` (env `SWE_BRIDGE_DIR` overrides), `Session(port, token, pid, started_at, ewapi_version, bridge_version)`, `read_session(directory=None) -> Session`
  - `swe.client`: `BridgeClient(session, timeout=600.0)`, `BridgeClient.connect(directory=None, timeout=600.0)`, `.health() -> dict`, `.schema() -> dict`, `.batch(ops, *, project, session, stop_on_error=True, batch_id=None) -> dict`; `all_ok(response) -> bool`; `require_ok(response) -> dict`; `new_session_id() -> str`
  - `swe.jsonout.print_json(value)`
  - `swe.cli.main(argv=None) -> int`; `COMMAND_MODULES` list — each module has `register(subparsers)` and sets `handler`

- [ ] **Step 1: Write the package metadata**

`swe/pyproject.toml`:

```toml
[build-system]
requires = ["setuptools>=75"]
build-backend = "setuptools.build_meta"

[project]
name = "swe"
version = "0.1.0"
description = "Drive SOLIDWORKS Electrical through the AddLife SWE Bridge add-in"
requires-python = ">=3.12"
dependencies = ["pyodbc>=5.3"]

[project.optional-dependencies]
dev = ["pytest>=8.3"]
spike = ["ezdxf>=1.3", "pypdf>=5.0"]

[project.scripts]
swe = "swe.cli:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
markers = ["live: needs SOLIDWORKS Electrical and its SQL Server running on this PC"]
addopts = "-m 'not live'"
```

Install it: `.venv\Scripts\python -m pip install -e "swe[dev,spike]"`
Expected: `Successfully installed swe-0.1.0`.

- [ ] **Step 2: Write the failing tests**

`swe/tests/fakebridge.py`:

```python
"""A stand-in for the add-in's HTTP endpoint, for tests."""
from __future__ import annotations

import json
import threading
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from pathlib import Path
from typing import Callable

Handler = Callable[[dict | None], tuple[int, dict]]


class FakeBridge:
    def __init__(self, token: str = "tok", routes: dict[tuple[str, str], Handler] | None = None):
        self.token = token
        self.routes = routes or {}
        self.requests: list[dict] = []
        bridge = self

        class _Handler(BaseHTTPRequestHandler):
            def _serve(self, method: str) -> None:
                length = int(self.headers.get("Content-Length") or 0)
                raw = self.rfile.read(length) if length else b""
                body = json.loads(raw.decode("utf-8")) if raw else None
                bridge.requests.append({"method": method, "path": self.path, "headers": dict(self.headers), "body": body})
                if self.headers.get("X-SWE-Token") != bridge.token:
                    status, reply = 401, {"error": {"code": "unauthorized", "message": "wrong token"}}
                elif (method, self.path) in bridge.routes:
                    status, reply = bridge.routes[(method, self.path)](body)
                else:
                    status, reply = 404, {"error": {"code": "not_found", "message": self.path}}
                data = json.dumps(reply, ensure_ascii=False).encode("utf-8")
                self.send_response(status)
                self.send_header("Content-Type", "application/json; charset=utf-8")
                self.send_header("Content-Length", str(len(data)))
                self.end_headers()
                self.wfile.write(data)

            def do_GET(self) -> None:  # noqa: N802
                self._serve("GET")

            def do_POST(self) -> None:  # noqa: N802
                self._serve("POST")

            def log_message(self, *args) -> None:
                pass

        self._server = ThreadingHTTPServer(("127.0.0.1", 0), _Handler)

    @property
    def port(self) -> int:
        return self._server.server_address[1]

    def __enter__(self) -> "FakeBridge":
        threading.Thread(target=self._server.serve_forever, daemon=True).start()
        return self

    def __exit__(self, *exc) -> None:
        self._server.shutdown()
        self._server.server_close()

    def write_session(self, directory: Path, token: str | None = None) -> None:
        directory.mkdir(parents=True, exist_ok=True)
        (directory / "session.json").write_text(json.dumps({
            "port": self.port, "token": token or self.token, "pid": 1, "started_at": "2026-10-12T08:00:00Z",
            "ewapi_version": "2026.4.1.1011", "bridge_version": "0.1.0",
        }), encoding="utf-8")
```

`swe/tests/conftest.py`:

```python
import pytest


@pytest.fixture
def bridge_dir(tmp_path, monkeypatch):
    """Every test gets its own %LOCALAPPDATA%\\AddLife\\swe-bridge."""
    directory = tmp_path / "swe-bridge"
    directory.mkdir()
    monkeypatch.setenv("SWE_BRIDGE_DIR", str(directory))
    return directory
```

`swe/tests/test_client.py`:

```python
import socket

import pytest

from fakebridge import FakeBridge
from swe.client import BridgeClient, all_ok, require_ok
from swe.errors import BatchFailed, BridgeAuthError, BridgeNotRunning


def ok_batch(body):
    return 200, {"batch_id": body["batch_id"], "cached": False, "refusal": None,
                 "results": [{"op": op["op"], "status": "ok", "result": op.get("args")} for op in body["ops"]]}


def test_missing_session_file(bridge_dir):
    with pytest.raises(BridgeNotRunning, match="not running"):
        BridgeClient.connect()


def test_dead_port_is_not_running(bridge_dir):
    with socket.socket() as s:
        s.bind(("127.0.0.1", 0))
        dead_port = s.getsockname()[1]
    (bridge_dir / "session.json").write_text(
        f'{{"port": {dead_port}, "token": "t", "pid": 1, "started_at": "x"}}', encoding="utf-8")
    with pytest.raises(BridgeNotRunning, match="nothing is listening"):
        BridgeClient.connect(timeout=2).health()


def test_wrong_token(bridge_dir):
    with FakeBridge(routes={("GET", "/health"): lambda _: (200, {})}) as fake:
        fake.write_session(bridge_dir, token="stale")
        with pytest.raises(BridgeAuthError, match="restart"):
            BridgeClient.connect().health()


def test_health_sends_token(bridge_dir):
    with FakeBridge(routes={("GET", "/health"): lambda _: (200, {"bridge_version": "0.1.0"})}) as fake:
        fake.write_session(bridge_dir)
        assert BridgeClient.connect().health()["bridge_version"] == "0.1.0"
        assert fake.requests[0]["headers"]["X-SWE-Token"] == "tok"


def test_batch_sends_given_batch_id_and_unicode(bridge_dir):
    with FakeBridge(routes={("POST", "/batch"): ok_batch}) as fake:
        fake.write_session(bridge_dir)
        client = BridgeClient.connect()
        response = client.batch([{"op": "echo", "args": {"text": "Ω µ → °"}}], project="LIBTEST",
                                session="s1", batch_id="fixed-id")
        sent = fake.requests[0]["body"]
        assert sent["batch_id"] == "fixed-id"
        assert sent["project"] == "LIBTEST"
        assert sent["stop_on_error"] is True
        assert response["results"][0]["result"]["text"] == "Ω µ → °"


def test_batch_generates_an_id_when_none_given(bridge_dir):
    with FakeBridge(routes={("POST", "/batch"): ok_batch}) as fake:
        fake.write_session(bridge_dir)
        BridgeClient.connect().batch([{"op": "a"}], project=None, session="s1")
        assert len(fake.requests[0]["body"]["batch_id"]) == 16


def test_require_ok_raises_with_the_failing_op():
    response = {"refusal": None, "results": [
        {"op": "a", "status": "ok"},
        {"op": "b", "status": "error", "error": {"code": "EW_BAD_INPUTS", "message": "bad", "last_error": None}}]}
    assert not all_ok(response)
    with pytest.raises(BatchFailed, match="b: EW_BAD_INPUTS bad"):
        require_ok(response)


def test_require_ok_raises_on_refusal():
    response = {"refusal": {"code": "project_not_allowlisted", "message": "not allowed"}, "results": []}
    with pytest.raises(BatchFailed, match="not allowed"):
        require_ok(response)
```

`swe/tests/test_cli.py`:

```python
import json

from fakebridge import FakeBridge
from swe.cli import main


def test_health_prints_json(bridge_dir, capsys):
    with FakeBridge(routes={("GET", "/health"): lambda _: (200, {"current_project": {"id": 1, "name": "LIBTEST Ω"}})}) as fake:
        fake.write_session(bridge_dir)
        assert main(["health"]) == 0
        assert json.loads(capsys.readouterr().out)["current_project"]["name"] == "LIBTEST Ω"


def test_health_without_electrical_exits_2(bridge_dir, capsys):
    assert main(["health"]) == 2
    assert "not running" in capsys.readouterr().err


def test_batch_file_with_project(bridge_dir, tmp_path, capsys):
    batch_file = tmp_path / "b.json"
    batch_file.write_text(json.dumps({"project": "LIBTEST", "ops": [{"op": "current_project"}]}), encoding="utf-8")
    reply = {"batch_id": "x", "refusal": None, "results": [{"op": "current_project", "status": "ok"}]}
    with FakeBridge(routes={("POST", "/batch"): lambda body: (200, reply)}) as fake:
        fake.write_session(bridge_dir)
        assert main(["batch", str(batch_file), "--session", "s1"]) == 0
        assert fake.requests[0]["body"]["project"] == "LIBTEST"
        assert fake.requests[0]["body"]["session"] == "s1"


def test_batch_exit_code_1_when_an_op_fails(bridge_dir, tmp_path):
    batch_file = tmp_path / "b.json"
    batch_file.write_text(json.dumps([{"op": "x"}]), encoding="utf-8")
    reply = {"refusal": {"code": "unknown_op", "message": "unknown operation 'x'"}, "results": [{"op": "x", "status": "skipped"}]}
    with FakeBridge(routes={("POST", "/batch"): lambda body: (200, reply)}) as fake:
        fake.write_session(bridge_dir)
        assert main(["batch", str(batch_file)]) == 1
```

- [ ] **Step 3: Run the tests to see them fail**

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: `ModuleNotFoundError: No module named 'swe.client'` (and the other new modules).

- [ ] **Step 4: Implement the package**

`swe/src/swe/__init__.py`:

```python
"""swe — drive SOLIDWORKS Electrical through the AddLife SWE Bridge add-in."""
__version__ = "0.1.0"
```

`swe/src/swe/errors.py`:

```python
"""Every error swe reports to a person derives from SweError and carries a sentence that says what to do."""
from __future__ import annotations


class SweError(Exception):
    """Base class: printed as 'swe: <message>' with exit code 2."""


class BridgeError(SweError):
    """Talking to the Electrical bridge failed."""


class BridgeNotRunning(BridgeError):
    """Electrical, and so the bridge add-in, is not running or not reachable."""


class BridgeAuthError(BridgeError):
    """The token in session.json was not accepted."""


class BatchFailed(BridgeError):
    """A batch was refused or one of its operations failed."""

    def __init__(self, response: dict):
        self.response = response
        refusal = response.get("refusal")
        if refusal:
            message = f"{refusal['code']}: {refusal['message']}"
        else:
            failed = [r for r in response.get("results", []) if r.get("status") == "error"]
            message = "; ".join(f"{r['op']}: {r['error']['code']} {r['error']['message']}" for r in failed)
        super().__init__(message or "batch failed")
```

`swe/src/swe/session.py`:

```python
"""Finds the running bridge through session.json, which the add-in rewrites at every Electrical start."""
from __future__ import annotations

import json
import os
from dataclasses import dataclass
from pathlib import Path

from .errors import BridgeNotRunning


def bridge_dir() -> Path:
    override = os.environ.get("SWE_BRIDGE_DIR")
    if override:
        return Path(override)
    return Path(os.environ["LOCALAPPDATA"]) / "AddLife" / "swe-bridge"


@dataclass(frozen=True)
class Session:
    port: int
    token: str
    pid: int
    started_at: str
    ewapi_version: str = ""
    bridge_version: str = ""


def read_session(directory: Path | None = None) -> Session:
    path = (directory or bridge_dir()) / "session.json"
    if not path.exists():
        raise BridgeNotRunning(
            f"SOLIDWORKS Electrical bridge is not running: {path} does not exist. "
            "Start SOLIDWORKS Electrical; the AddLife SWE Bridge add-in starts with it.")
    data = json.loads(path.read_text(encoding="utf-8-sig"))
    return Session(port=int(data["port"]), token=str(data["token"]), pid=int(data["pid"]),
                   started_at=str(data["started_at"]), ewapi_version=str(data.get("ewapi_version", "")),
                   bridge_version=str(data.get("bridge_version", "")))
```

`swe/src/swe/client.py`:

```python
"""HTTP client for the bridge add-in: GET /health, GET /schema, POST /batch."""
from __future__ import annotations

import datetime
import http.client
import json
import uuid
from pathlib import Path

from .errors import BatchFailed, BridgeAuthError, BridgeError, BridgeNotRunning
from .session import Session, read_session


def new_session_id() -> str:
    return "s-" + datetime.datetime.now().strftime("%Y%m%d-%H%M%S")


def all_ok(response: dict) -> bool:
    return not response.get("refusal") and all(r.get("status") == "ok" for r in response.get("results", []))


def require_ok(response: dict) -> dict:
    if not all_ok(response):
        raise BatchFailed(response)
    return response


class BridgeClient:
    def __init__(self, session: Session, timeout: float = 600.0):
        self.session = session
        self.timeout = timeout

    @classmethod
    def connect(cls, directory: Path | None = None, timeout: float = 600.0) -> "BridgeClient":
        return cls(read_session(directory), timeout)

    def health(self) -> dict:
        return self._request("GET", "/health")

    def schema(self) -> dict:
        return self._request("GET", "/schema")

    def batch(self, ops: list[dict], *, project: str | None, session: str, stop_on_error: bool = True,
              batch_id: str | None = None) -> dict:
        body = {"batch_id": batch_id or uuid.uuid4().hex[:16], "session": session, "project": project,
                "stop_on_error": stop_on_error, "ops": ops}
        return self._request("POST", "/batch", body)

    def _request(self, method: str, path: str, body: dict | None = None) -> dict:
        connection = http.client.HTTPConnection("127.0.0.1", self.session.port, timeout=self.timeout)
        headers = {"X-SWE-Token": self.session.token}
        payload = None
        if body is not None:
            payload = json.dumps(body, ensure_ascii=False).encode("utf-8")
            headers["Content-Type"] = "application/json; charset=utf-8"
        try:
            connection.request(method, path, body=payload, headers=headers)
            response = connection.getresponse()
            raw = response.read()
        except ConnectionRefusedError as e:
            raise BridgeNotRunning(
                f"nothing is listening on 127.0.0.1:{self.session.port} (session from {self.session.started_at}): "
                "SOLIDWORKS Electrical was closed or the add-in failed to load — see bridge.log") from e
        except TimeoutError as e:
            raise BridgeError(
                f"the bridge did not answer within {self.timeout:.0f}s; Electrical may be busy or showing a dialog. "
                "The batch may still finish: send it again with the same batch_id to get its result "
                "without running it twice.") from e
        except OSError as e:
            raise BridgeError(f"connection to the bridge on 127.0.0.1:{self.session.port} failed: {e}") from e
        finally:
            connection.close()
        if response.status == 401:
            raise BridgeAuthError("the bridge rejected the token in session.json; restart SOLIDWORKS Electrical "
                                  "if this persists")
        data = json.loads(raw.decode("utf-8")) if raw else {}
        if response.status >= 400:
            error = data.get("error", {})
            raise BridgeError(f"{error.get('code', response.status)}: {error.get('message', raw.decode('utf-8', 'replace'))}")
        return data
```

`swe/src/swe/jsonout.py`:

```python
from __future__ import annotations

import json
from typing import Any


def print_json(value: Any) -> None:
    print(json.dumps(value, indent=2, ensure_ascii=False, default=str))
```

`swe/src/swe/cli.py`:

```python
"""Entry point: `swe <command>`. Each command lives in swe.commands.<name> with register() and a handler."""
from __future__ import annotations

import argparse
import importlib
import sys

from .errors import SweError

COMMAND_MODULES = [
    "swe.commands.health",
    "swe.commands.batch",
]


def main(argv: list[str] | None = None) -> int:
    if hasattr(sys.stdout, "reconfigure"):
        sys.stdout.reconfigure(encoding="utf-8")
        sys.stderr.reconfigure(encoding="utf-8")
    parser = argparse.ArgumentParser(prog="swe", description="Drive SOLIDWORKS Electrical through the AddLife bridge.")
    subparsers = parser.add_subparsers(dest="command", required=True)
    for name in COMMAND_MODULES:
        importlib.import_module(name).register(subparsers)
    args = parser.parse_args(argv)
    try:
        return args.handler(args)
    except SweError as e:
        print(f"swe: {e}", file=sys.stderr)
        return 2


if __name__ == "__main__":
    sys.exit(main())
```

`swe/src/swe/commands/__init__.py`: empty file.

`swe/src/swe/commands/health.py`:

```python
from ..client import BridgeClient
from ..jsonout import print_json


def register(subparsers) -> None:
    parser = subparsers.add_parser("health", help="Show the bridge and Electrical status.")
    parser.set_defaults(handler=run)


def run(args) -> int:
    print_json(BridgeClient.connect().health())
    return 0
```

`swe/src/swe/commands/batch.py`:

```python
import json
import os
from pathlib import Path

from ..client import BridgeClient, all_ok, new_session_id
from ..jsonout import print_json


def register(subparsers) -> None:
    parser = subparsers.add_parser("batch", help="Send a batch of operations from a JSON file.")
    parser.add_argument("file", help="JSON: a list of {op, args}, or {project, ops}")
    parser.add_argument("--project", help="target project (overrides the file)")
    parser.add_argument("--session", default=os.environ.get("SWE_SESSION"),
                        help="session id; one snapshot per session (default: $SWE_SESSION or a new id)")
    parser.add_argument("--batch-id", help="re-send a batch that timed out, without running it twice")
    parser.add_argument("--continue-on-error", action="store_true")
    parser.set_defaults(handler=run)


def load_batch_file(path: Path) -> tuple[str | None, list[dict]]:
    data = json.loads(path.read_text(encoding="utf-8-sig"))
    if isinstance(data, list):
        return None, data
    return data.get("project"), data["ops"]


def run(args) -> int:
    file_project, ops = load_batch_file(Path(args.file))
    response = BridgeClient.connect().batch(
        ops, project=args.project or file_project, session=args.session or new_session_id(),
        stop_on_error=not args.continue_on_error, batch_id=args.batch_id)
    print_json(response)
    return 0 if all_ok(response) else 1
```

- [ ] **Step 5: Run the tests to see them pass**

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: all tests pass.

- [ ] **Step 6: Live check against Electrical**

Run: `.venv\Scripts\swe health`
Expected: the same JSON as Task 0.5 Step 6, printed with Ω-safe UTF-8.

- [ ] **Step 7: Commit**

```powershell
git add swe
git commit -m "feat(swe): session discovery, bridge client, health and batch commands"
```

### Task 0.7: Probe operations and the probe runner (S2–S8, S10) — throwaway

Each probe calls the EwAPI step by step and records every return code and read-back, whether it succeeds or not; it never stops at the first failure, because the failures are the findings. Probes only touch projects named `SPIKE-*` and a scratch library `ADDLIFE_SPIKE`. All of this code is deleted in Task 0.9.

**Files:**
- Create: `addin/src/AddLife.SweBridge/Probes/Evidence.cs`, `ProbeSet.cs`, `ProbeS1.cs`, `ProbeSheet.cs`, `ProbeLibrary.cs`, `ProbeTemplate.cs`, `ProbeArchive.cs`, `ProbeWires.cs`, `ProbeCleanup.cs`
- Modify: `addin/src/AddLife.SweBridge/Electrical/Operations.cs` (one line)
- Create: `spike/make_test_symbol.py`, `spike/run_probes.py`
- Create (evidence, committed): `spike/results/*.json`

**Interfaces:**
- Consumes: `EwContext`, `EwOperation`, `Ew`, `ConnectEvidence` (Task 0.5); `BridgeClient`, `require_ok` (Task 0.6)
- Produces: operations `probe_s1`, `probe_sheet`, `probe_library`, `probe_template`, `probe_archive`, `probe_wires`, `probe_cleanup` (registered only when `probes_enabled` is true); `spike/results/<probe>.json`

- [ ] **Step 1: Write the evidence recorder and the registration**

`addin/src/AddLife.SweBridge/Probes/Evidence.cs`:

```csharp
using System;
using System.Collections.Generic;
using EwAPI;

namespace AddLife.SweBridge.Probes
{
    /// <summary>Records each probe step — what was called, the result or the exception — and carries on.</summary>
    sealed class Evidence
    {
        public List<Dictionary<string, object>> Steps { get; } = new List<Dictionary<string, object>>();

        public T Step<T>(string name, Func<T> action)
        {
            try
            {
                var value = action();
                Steps.Add(new Dictionary<string, object> { ["step"] = name, ["ok"] = true, ["value"] = Describe(value) });
                return value;
            }
            catch (Exception e)
            {
                Steps.Add(new Dictionary<string, object> { ["step"] = name, ["ok"] = false, ["error"] = e.GetType().Name + ": " + e.Message });
                return default(T);
            }
        }

        /// <summary>For calls that return an EwErrorCode: ok means EW_NO_ERROR.</summary>
        public EwErrorCode Code(string name, Func<EwErrorCode> call)
        {
            try
            {
                var code = call();
                Steps.Add(new Dictionary<string, object> { ["step"] = name, ["ok"] = code == EwErrorCode.EW_NO_ERROR, ["value"] = code.ToString() });
                return code;
            }
            catch (Exception e)
            {
                Steps.Add(new Dictionary<string, object> { ["step"] = name, ["ok"] = false, ["error"] = e.GetType().Name + ": " + e.Message });
                return EwErrorCode.EW_UNDEFINED_ERROR;
            }
        }

        static object Describe(object value)
        {
            switch (value)
            {
                case null: return null;
                case Enum e: return e.ToString();
                case string _: case bool _: case int _: case long _: case double _: return value;
                case Dictionary<string, object> _: case List<object> _: case List<Dictionary<string, object>> _: return value;
                default: return "<" + value.GetType().Name + ">";
            }
        }
    }
}
```

`addin/src/AddLife.SweBridge/Probes/ProbeSet.cs`:

```csharp
using AddLife.SweBridge.Core;

namespace AddLife.SweBridge.Probes
{
    public static class ProbeSet
    {
        public static void Register(OpRegistry registry)
        {
            registry.Add(new ProbeS1());
            registry.Add(new ProbeSheet());
            registry.Add(new ProbeLibrary());
            registry.Add(new ProbeTemplate());
            registry.Add(new ProbeArchive());
            registry.Add(new ProbeWires());
            registry.Add(new ProbeCleanup());
        }
    }
}
```

`addin/src/AddLife.SweBridge/Probes/ProbeS1.cs`:

```csharp
using System.Collections.Generic;
using System.Threading;
using AddLife.SweBridge.Core;
using AddLife.SweBridge.Electrical;

namespace AddLife.SweBridge.Probes
{
    sealed class ProbeS1 : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("probe_s1",
            "Phase 0 (S1, S8): how the add-in connected and which thread runs operations.", false, false);

        protected override object Run(EwContext ctx, Args args) => new Dictionary<string, object>
        {
            ["license_result"] = ConnectEvidence.LicenseResult,
            ["connect_thread"] = ConnectEvidence.ConnectThread,
            ["connect_apartment"] = ConnectEvidence.ConnectApartment,
            ["op_thread"] = Thread.CurrentThread.ManagedThreadId,
            ["op_apartment"] = Thread.CurrentThread.GetApartmentState().ToString(),
            ["app_version"] = ctx.App.getApplicationVersion(),
            ["api_dll_version"] = ctx.App.getApiDllVersion(),
            ["win_app_name"] = ctx.App.getWinAppName(),
        };
    }
}
```

Modify `addin/src/AddLife.SweBridge/Electrical/Operations.cs` — insert before `return registry;`:

```csharp
            if (config.ProbesEnabled) Probes.ProbeSet.Register(registry);
```

- [ ] **Step 2: Write the sheet probe (S2, S3, S8)**

`addin/src/AddLife.SweBridge/Probes/ProbeSheet.cs`:

```csharp
using System.Collections.Generic;
using System.IO;
using System.Linq;
using AddLife.SweBridge.Core;
using AddLife.SweBridge.Electrical;
using EwAPI;

namespace AddLife.SweBridge.Probes
{
    /// <summary>
    /// S2: project from template, book, sheet, two symbols (one linked to a pre-made component), wires.
    /// S3: multilingual text and a CAD command. S8: render while open and closed, snapshot, PDF export.
    /// </summary>
    sealed class ProbeSheet : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("probe_sheet", "Phase 0 (S2, S3, S8).", true, false,
            Req("project_name", "string", "new project, must match SPIKE-*"),
            Req("template", "string", "project template name"),
            Req("symbol", "string", "library symbol to place twice"),
            Req("out_dir", "string", "folder for PNG/PDF evidence"));

        public override string TargetProject(Args args, string batchProject) => args.Str("project_name");

        protected override object Run(EwContext ctx, Args a)
        {
            var ev = new Evidence();
            var lang = ctx.Lang;
            var name = a.Str("project_name");
            var outDir = a.Str("out_dir");
            Directory.CreateDirectory(outDir);
            EwErrorCode err;

            var pm = ctx.Projects();
            var project = ev.Step("newEwProject", () => { var p = pm.newEwProject(out err); Ew.Check(err, "newEwProject"); return p; });
            if (project == null) return ev.Steps;
            ev.Code("project.setName", () => project.setName(name));
            ev.Code("project.insertFromTemplate", () => project.insertFromTemplate(a.Str("template")));
            var projectId = ev.Step("project.getID", () => project.getID());
            ev.Code("app.openEwProjectID", () => ctx.App.openEwProjectID(projectId));
            ev.Step("current project after open", () => ctx.CurrentProject().getName(out err));
            ev.Step("configuration current code language", () => project.getEwProjectConfiguration(out err).getCurrentCodeLanguage(out err));

            var books = project.getEwProjectBookManager(out err);
            var book = ev.Step("newEwProjectBook", () => { var b = books.newEwProjectBook(out err); Ew.Check(err, "newEwProjectBook"); return b; });
            ev.Code("book.setTagMode(manual)", () => book.setTagMode(EwTagMode.kManu));
            ev.Code("book.setTag", () => book.setTag("SPK"));
            ev.Code("book.setDescription", () => book.setDescription(lang, "Spike book"));
            ev.Code("book.insert", () => book.insert());
            var bookId = ev.Step("book.getID", () => book.getID());
            ev.Step("book.getTag", () => book.getTag(out err));
            ev.Step("book.getDescription(lang)", () => book.getDescription(lang, out err));

            var files = project.getEwProjectFileManager(out err);
            var sheet = ev.Step("newProjectFile", () => { var f = files.newProjectFile(out err); Ew.Check(err, "newProjectFile"); return f; });
            ev.Code("sheet.setFileType(folio)", () => sheet.setFileType(EwFileType.kFileFolio));
            ev.Code("sheet.setEwProjectBookID", () => sheet.setEwProjectBookID(bookId));
            ev.Code("sheet.setDescription", () => sheet.setDescription(lang, "Spike sheet Ω µ → °"));
            ev.Code("sheet.insert", () => sheet.insert());
            var sheetId = ev.Step("sheet.getID", () => sheet.getID());
            ev.Step("sheet.getPosition after insert", () => sheet.getPosition(out err));
            ev.Code("sheet.setPosition(1)", () => sheet.setPosition(1));
            ev.Code("sheet.update", () => sheet.update());
            ev.Step("sheet.getPosition after setPosition(1)", () => sheet.getPosition(out err));
            var sheetPath = ev.Step("sheet.getFilePath", () => sheet.getFilePath(out err));
            ev.Step("sheet.canInsertSymbol", () => sheet.canInsertSymbol(out err));

            var components = project.getEwProjectComponentManager(out err);
            var component = ev.Step("newEwProjectComponent", () => components.newEwProjectComponent());
            ev.Code("component.setTagMode(manual)", () => component.setTagMode(EwTagMode.kManu));
            ev.Code("component.setTag", () => component.setTag("-SPK2"));
            ev.Code("component.setDescription", () => component.setDescription(lang, "Spike component Ω"));
            ev.Code("component.insert", () => component.insert());
            var componentId = ev.Step("component.getID", () => component.getID());

            var first = Place(ev, "symbol 1", sheet, a.Str("symbol"), 100, 150, 0);
            var second = Place(ev, "symbol 2 (linked to -SPK2)", sheet, a.Str("symbol"), 160, 150, componentId);
            ev.Step("components after placing", () => Ew.Array<EwProjectComponentX>(components.getEwProjectComponentArray(out err))
                .Select(c => (object)(c.getID() + " " + c.getTag(out err))).ToList());

            var p1 = first?.Last();
            var p2 = second?.First();
            if (p1 != null) Line(ev, "free line from symbol 1", sheet, (double)p1["x"], (double)p1["y"], (double)p1["x"], (double)p1["y"] - 30);
            if (p1 != null && p2 != null) Line(ev, "line symbol 1 -> symbol 2", sheet, (double)p1["x"], (double)p1["y"], (double)p2["x"], (double)p2["y"]);
            ev.Step("number wires", () =>
            {
                var numbering = project.newEwProjectNumberWires(out err); Ew.Check(err, "newEwProjectNumberWires");
                Ew.Check(numbering.setSelectionType(EwSelectionType.kSelectionAll), "setSelectionType");
                Ew.Check(numbering.process(EwNumberWireAction.kNumberNewWiresAction), "process");
                return true;
            });
            ev.Step("wires in project", () => Ew.Array<EwProjectWireX>(project.getEwProjectWireManager(out err).getEwProjectWireArray(out err))
                .Select(w => (object)new Dictionary<string, object>
                {
                    ["id"] = w.getID(), ["tag"] = w.getTag(out err),
                    ["from_component"] = w.getOriginComponentID(out err), ["from_terminal"] = w.getOriginComponentTerminalNumber(out err),
                    ["to_component"] = w.getDestinationComponentID(out err), ["to_terminal"] = w.getDestinationComponentTerminalNumber(out err),
                    ["equipotential"] = w.getEquipotential(out err),
                }).ToList());

            var text = ev.Step("newEwProjectMultilingualText", () => { var t = sheet.newEwProjectMultilingualText(out err); Ew.Check(err, "newEwProjectMultilingualText"); return t; });
            ev.Code("text.setText", () => text.setText(lang, "Spike text Ω µ → °"));
            ev.Code("text.setXPosition", () => text.setXPosition(40));
            ev.Code("text.setYPosition", () => text.setYPosition(40));
            ev.Code("text.insert", () => text.insert());
            ev.Step("text.getText(lang)", () => text.getText(lang, out err));

            ev.Code("sheet.open", () => sheet.open());
            ev.Code("app.runCommand(_LINE)", () => ctx.App.runCommand("_LINE 20,20 60,20 "));
            ev.Step("current document's sheet id", () => ctx.App.getEwDocumentCurrent(out err)?.getEwProjectFileX(out err)?.getID());
            ev.Step("render while open", () => Render(ctx, sheetPath, Path.Combine(outDir, name + "-open.png")));
            ev.Code("sheet.close", () => sheet.close());
            ev.Step("render while closed", () => Render(ctx, sheetPath, Path.Combine(outDir, name + "-closed.png")));

            ev.Step("snapshot", () =>
            {
                var snapshots = project.getEwProjectSnapshotManager(out err); Ew.Check(err, "getEwProjectSnapshotManager");
                var s = snapshots.newEwProjectSnapshot(out err); Ew.Check(err, "newEwProjectSnapshot");
                Ew.Check(s.setName("spike"), "setName");
                Ew.Check(s.create(), "create");
                return snapshots.getCount(out err);
            });
            ev.Step("symbols by sheet id (objects or ids?)", () => (project.getEwProjectSymbolManager(out err)
                .getProjectSymbolsFromFileID(sheetId, out err) as object[])?.Select(o => (object)(o is EwProjectSymbolX s ? "symbol " + s.getID() : o?.ToString())).ToList());
            ev.Step("lines by sheet id (objects or ids?)", () => (project.getEwProjectLineManager(out err)
                .getEwProjectLineArrayFromFileID(sheetId, out err) as object[])?.Select(o => (object)(o is ewProjectLineX l ? "line " + l.getID() : o?.ToString())).ToList());
            ev.Step("texts by sheet id (objects or ids?)", () => (project.getEwProjectMultilingualTextManager(out err)
                .getEwProjectMultilingualTextByFileIDArray(sheetId, out err) as object[])?.Select(o => (object)(o is EwProjectMultilingualTextX t ? "text " + t.getID() : o?.ToString())).ToList());
            ev.Step("export pdf, all sheets", () => ExportPdf(project, Path.Combine(outDir, name + "-all.pdf"), null));
            ev.Step("export pdf, selection by id", () => ExportPdf(project, Path.Combine(outDir, name + "-one.pdf"), new object[] { sheetId }));

            return new Dictionary<string, object> { ["project_id"] = projectId, ["sheet_id"] = sheetId, ["steps"] = ev.Steps };
        }

        static List<Dictionary<string, object>> Place(Evidence ev, string label, EwProjectFileX sheet, string symbolName, double x, double y, int componentId)
        {
            EwErrorCode err;
            var symbol = ev.Step(label + ": newEwProjectSymbol", () => { var s = sheet.newEwProjectSymbol(out err); Ew.Check(err, "newEwProjectSymbol"); return s; });
            if (symbol == null) return null;
            ev.Code(label + ": setEwSymbolName", () => symbol.setEwSymbolName(symbolName));
            ev.Code(label + ": setXPosition", () => symbol.setXPosition(x));
            ev.Code(label + ": setYPosition", () => symbol.setYPosition(y));
            if (componentId != 0) ev.Code(label + ": setObjectID(component)", () => symbol.setObjectID(componentId));
            ev.Code(label + ": insert", () => symbol.insert());
            ev.Step(label + ": getID", () => symbol.getID());
            ev.Step(label + ": getObjectID", () => symbol.getObjectID(out err));
            return ev.Step(label + ": points", () => Ew.Array<EwProjectSymbolPointX>(symbol.getEwProjectSymbolPointArray(out err))
                .Select(p =>
                {
                    var position = p.getPointPosition(out err);
                    return new Dictionary<string, object>
                    {
                        ["n"] = p.getPointNumber(out err), ["circuit"] = p.getCircuitNumber(out err), ["terminal"] = p.getTerminalNumber(out err),
                        ["x"] = position.getXCoordinate(), ["y"] = position.getYCoordinate(),
                    };
                }).ToList());
        }

        static void Line(Evidence ev, string label, EwProjectFileX sheet, double x1, double y1, double x2, double y2)
        {
            EwErrorCode err;
            var line = ev.Step(label + ": newEwProjectLine", () => { var l = sheet.newEwProjectLine(EwLineType.kLineSchematic, out err); Ew.Check(err, "newEwProjectLine"); return l; });
            if (line == null) return;
            ev.Code(label + ": start x", () => line.setStartPointXPosition(x1));
            ev.Code(label + ": start y", () => line.setStartPointYPosition(y1));
            ev.Code(label + ": end x", () => line.setEndPointXPosition(x2));
            ev.Code(label + ": end y", () => line.setEndPointYPosition(y2));
            ev.Code(label + ": insert", () => line.insert());
            ev.Step(label + ": id", () => line.getID());
            ev.Step(label + ": equipotential id", () => line.getEquipotentialID(out err));
            ev.Step(label + ": wire mark", () => line.getWireMarkText(out err));
        }

        static long Render(EwContext ctx, string dwgPath, string pngPath)
        {
            var image = ctx.App.newEwSaveDWGImage(out var err); Ew.Check(err, "newEwSaveDWGImage");
            Ew.Check(image.setDWGFilePath(dwgPath), "setDWGFilePath");
            Ew.Check(image.setDestinationFilePath(pngPath), "setDestinationFilePath");
            Ew.Check(image.setSaveImageType(EwSaveImageType.kSaveImagePNG), "setSaveImageType");
            Ew.Check(image.setWidth(2480), "setWidth");
            Ew.Check(image.setHeight(1754), "setHeight");
            Ew.Check(image.setOverwriteDestination(true), "setOverwriteDestination");
            Ew.Check(image.save(), "save");
            return File.Exists(pngPath) ? new FileInfo(pngPath).Length : -1;
        }

        static long ExportPdf(EwProjectX project, string pdfPath, object[] sheetIds)
        {
            var export = project.newEwProjectExportPDF(out var err); Ew.Check(err, "newEwProjectExportPDF");
            Ew.Check(export.setExportToPDFFileName(pdfPath), "setExportToPDFFileName");
            Ew.Check(export.setAllProjectFiles(sheetIds == null), "setAllProjectFiles");
            if (sheetIds != null) Ew.Check(export.setSelectionFiles(sheetIds), "setSelectionFiles");
            Ew.Check(export.setSilentMode(true), "setSilentMode");
            Ew.Check(export.exportPDF(), "exportPDF");
            return File.Exists(pdfPath) ? new FileInfo(pdfPath).Length : -1;
        }
    }
}
```

- [ ] **Step 3: Write the library probe (S4, S5, S6)**

`addin/src/AddLife.SweBridge/Probes/ProbeLibrary.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;
using AddLife.SweBridge.Core;
using AddLife.SweBridge.Electrical;
using EwAPI;

namespace AddLife.SweBridge.Probes
{
    /// <summary>
    /// S4: scratch library, black-box symbol with two API-added connection points and DXF graphics.
    /// S5: a part linked to the symbol, and a PLC module part with channel addresses. S6: the macro manager.
    /// </summary>
    sealed class ProbeLibrary : EwOperation
    {
        const string Library = "ADDLIFE_SPIKE";
        const string SymbolName = "ADL_SPIKE_BOX";

        public override OpSpec Spec { get; } = new OpSpec("probe_library", "Phase 0 (S4, S5, S6).", true, false,
            Req("dxf_path", "string", "symbol graphics written by spike/make_test_symbol.py"));

        // Library objects belong to no project; this pseudo-name matches the SPIKE-* allowlist entry.
        public override string TargetProject(Args args, string batchProject) => "SPIKE-library";

        protected override object Run(EwContext ctx, Args a)
        {
            var ev = new Evidence();
            var lang = ctx.Lang;
            var env = ctx.Env();
            EwErrorCode err;

            var libraries = env.getEwLibraryManager(out err);
            ev.Step("existing libraries", () => Ew.Array<EwLibraryX>(libraries.getEwLibraryArray(out err)).Select(l => (object)l.getName(out err)).ToList());
            var library = ev.Step("newEwLibrary", () => libraries.newEwLibrary());
            ev.Code("library.setName", () => library.setName(Library));
            ev.Code("library.setEwLibContentType(all)", () => library.setEwLibContentType(EwLibContentType.kTypeAll));
            ev.Code("library.setDescription", () => library.setDescription(lang, "Phase 0 spike - delete me"));
            ev.Code("library.insert", () => library.insert());

            var circuitTypes = env.getEwCircuitTypeManagerX(out err);
            ev.Step("circuit types (first 80)", () => Ew.Array<ewCircuitTypeX>(circuitTypes.getEwCircuitTypeArray(out err))
                .Take(80).Select(c => (object)(c.getKeyCode(out err) + " = " + c.getDescription(lang, out err))).ToList());
            var circuitCode = ev.Step("first circuit key code", () => Ew.Array<ewCircuitTypeX>(circuitTypes.getEwCircuitTypeArray(out err)).First().getKeyCode(out err));

            var symbols = env.getEwSymbolManager(out err);
            var symbol = ev.Step("newEwSymbol", () => symbols.newEwSymbol());
            ev.Code("symbol.setName", () => symbol.setName(SymbolName));
            ev.Code("symbol.setEwSymbolType(blackbox)", () => symbol.setEwSymbolType(EwSymbolType.kSymbolBlackbox));
            ev.Code("symbol.setLibraryCode", () => symbol.setLibraryCode(Library));
            ev.Code("symbol.setRootMark", () => symbol.setRootMark("A"));
            ev.Code("symbol.setDescription", () => symbol.setDescription(lang, "Spike black box"));
            ev.Step("symbol.addEwSymbolCircuit", () => { symbol.addEwSymbolCircuit(circuitCode, out err); return err.ToString(); });
            foreach (var (x, number) in new[] { (10.0, 1), (30.0, 2) })
                ev.Step($"symbol connection point {number}", () =>
                {
                    var point = symbol.addEwSymbolPoint(out err); Ew.Check(err, "addEwSymbolPoint");
                    Ew.Check(point.setXPos(x), "setXPos");
                    Ew.Check(point.setYPos(-5), "setYPos");
                    Ew.Check(point.setEwSymbolCircuitNumber(1), "setEwSymbolCircuitNumber");
                    Ew.Check(point.setEwPointOrientation(EwPointOrientation.kPointOrientationInput), "setEwPointOrientation");
                    return true;
                });
            ev.Code("symbol.insert", () => symbol.insert());
            ev.Code("symbol.insertFromDwg", () => symbol.insertFromDwg(a.Str("dxf_path")));
            ev.Step("symbol read back", () =>
            {
                var found = symbols.findEwSymbolXByName(SymbolName, out err);
                return new Dictionary<string, object>
                {
                    ["found"] = found != null,
                    ["points"] = found?.getEwSymbolPointCount(),
                    ["circuits"] = found?.getEwSymbolCircuitCount(),
                    ["connection points visible"] = found?.isConnectionPointVisible(out err),
                };
            });
            ev.Step("symbol folder", () => env.getFolderPath(EwEnvironmentFolderPathValue.kFolderPathSymbol, out err));

            var parts = env.getEwManufacturerPartManager(out err);
            var part = ev.Step("newEwManufacturerPart", () => parts.newEwManufacturerPart());
            ev.Code("part.setManufacturer", () => part.setManufacturer("ADDLIFE"));
            ev.Code("part.setReference", () => part.setReference("ADL-SPIKE-BOX"));
            ev.Code("part.setLibraryCode", () => part.setLibraryCode(Library));
            ev.Code("part.setDescription", () => part.setDescription(lang, "Spike part Ω"));
            ev.Code("part.setSchemeSymbolName", () => part.setSchemeSymbolName(SymbolName));
            ev.Step("part: one circuit, two labelled terminals", () =>
            {
                var circuit = part.addEwManufacturerPartCircuit(circuitCode, out err); Ew.Check(err, "addEwManufacturerPartCircuit");
                foreach (var label in new[] { "L+", "M" })
                {
                    var terminal = circuit.addEwManufacturerPartTerminal(out err); Ew.Check(err, "addEwManufacturerPartTerminal");
                    Ew.Check(terminal.setText(label), "terminal setText");
                }
                Ew.Check(circuit.setSymbolName(SymbolName), "circuit setSymbolName");
                return circuit.getEwManufacturerPartTerminalCount();
            });
            ev.Code("part.insert", () => part.insert());
            ev.Code("part.setUserData(1, review flag)", () => part.setUserData(1, "ADL_REVIEW=pending"));
            ev.Code("part.update", () => part.update());

            var plc = ev.Step("newEwManufacturerPart (PLC module)", () => parts.newEwManufacturerPart());
            ev.Code("plc.setManufacturer", () => plc.setManufacturer("ADDLIFE"));
            ev.Code("plc.setReference", () => plc.setReference("ADL-SPIKE-DI2"));
            ev.Code("plc.setLibraryCode", () => plc.setLibraryCode(Library));
            ev.Code("plc.setEwManufacturerPartType(PLC module)", () => plc.setEwManufacturerPartType(EwManufacturerPartType.kManufacturerPartPlcModule));
            ev.Step("plc: two channels with addresses", () =>
            {
                for (var channel = 0; channel < 2; channel++)
                {
                    var circuit = plc.addEwManufacturerPartCircuit(circuitCode, out err); Ew.Check(err, "addEwManufacturerPartCircuit");
                    Ew.Check(circuit.setChannelAddress("I0." + channel), "setChannelAddress");
                    Ew.Check(circuit.setChannelGroup("IN"), "setChannelGroup");
                    var terminal = circuit.addEwManufacturerPartTerminal(out err); Ew.Check(err, "addEwManufacturerPartTerminal");
                    Ew.Check(terminal.setText("In " + (channel + 1)), "terminal setText");
                }
                return plc.getEwManufacturerPartCircuitCount();
            });
            ev.Code("plc.insert", () => plc.insert());

            ev.Step("macro manager contents", () =>
            {
                var macros = env.getEwMacroManager(out err); Ew.Check(err, "getEwMacroManager");
                var all = Ew.Array<EwProjectX>(macros.getEwProjectArray(out err));
                return new Dictionary<string, object>
                {
                    ["count"] = all.Length,
                    ["first ten"] = all.Take(10).Select(m => (object)(m.getName(out err) + " type=" + m.getProjectType(out err))).ToList(),
                };
            });
            ev.Step("macro create attempt", () =>
            {
                var macros = env.getEwMacroManager(out err); Ew.Check(err, "getEwMacroManager");
                var macro = macros.newEwProject(out err); Ew.Check(err, "newEwProject");
                Ew.Check(macro.setName("ADL_M_SPIKE"), "setName");
                var inserted = macro.insert();
                return inserted + " type=" + macro.getProjectType(out err) + " id=" + macro.getID();
            });

            return new Dictionary<string, object> { ["circuit_code"] = circuitCode, ["steps"] = ev.Steps };
        }
    }
}
```

- [ ] **Step 4: Write the template, archive, wires and clean-up probes (S7, S9, S10)**

`addin/src/AddLife.SweBridge/Probes/ProbeTemplate.cs`:

```csharp
using System.Collections.Generic;
using System.IO;
using System.Linq;
using AddLife.SweBridge.Core;
using AddLife.SweBridge.Electrical;
using EwAPI;

namespace AddLife.SweBridge.Probes
{
    /// <summary>S7: which title blocks the project uses, its report set, and whether export regenerates automated drawings.</summary>
    sealed class ProbeTemplate : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("probe_template", "Phase 0 (S7).", true, true,
            Req("sheet_id", "int", "a sheet of the current SPIKE project"),
            Req("out_dir", "string", "folder for evidence"));

        protected override object Run(EwContext ctx, Args a)
        {
            var ev = new Evidence();
            var lang = ctx.Lang;
            var project = ctx.CurrentProject();
            var outDir = a.Str("out_dir");
            var name = project.getName(out var err);

            ev.Step("title blocks in the environment", () => Ew.Array<EwTitleBlockX>(ctx.Env().getEwTitleBlockManager(out err).getEwTitleBlockArray(out err))
                .Select(t => (object)(t.getName(out err) + " | " + t.getDescription(lang, out err))).ToList());
            var config = project.getEwProjectConfiguration(out err);
            foreach (var value in new[] { EwProjectConfigValue.kCoverPageTitleBlock, EwProjectConfigValue.kSchematicTitleBlock,
                         EwProjectConfigValue.kBOMTitleBlock, EwProjectConfigValue.kTerminalTitleBlock, EwProjectConfigValue.kFormatDate,
                         EwProjectConfigValue.kFileFormula, EwProjectConfigValue.kBookFormula, EwProjectConfigValue.kCurrentCodeLg })
                ev.Step("configuration " + value, () => config.getEwProjectConfigValue(value, out err)?.ToString());

            ev.Step("reports configured in the project", () =>
            {
                var reports = project.getEwProjectReportManager(out err); Ew.Check(err, "getEwProjectReportManager");
                var list = new List<object>();
                for (var i = 0; i < reports.getCount(out err); i++)
                {
                    var report = reports.at(i, out err);
                    list.Add(report.getReportFileName(out err) + " | filter=" + report.getFilter(out err) + " | " + report.getEwProjectDataExportType(out err));
                }
                return list;
            });

            ev.Code("project.setCustomerName", () => project.setCustomerName("Spike Customer Ω"));
            ev.Code("project.setContractNumber", () => project.setContractNumber("SPK-0001"));
            ev.Code("project.update", () => project.update());
            var sheet = ctx.Sheet(project, a.Int("sheet_id"));
            ev.Step("render after setting project fields (look at the title block)", () =>
            {
                var png = Path.Combine(outDir, name + "-titleblock.png");
                var image = ctx.App.newEwSaveDWGImage(out err); Ew.Check(err, "newEwSaveDWGImage");
                Ew.Check(image.setDWGFilePath(sheet.getFilePath(out err)), "setDWGFilePath");
                Ew.Check(image.setDestinationFilePath(png), "setDestinationFilePath");
                Ew.Check(image.setSaveImageType(EwSaveImageType.kSaveImagePNG), "setSaveImageType");
                Ew.Check(image.setWidth(2480), "setWidth");
                Ew.Check(image.setHeight(1754), "setHeight");
                Ew.Check(image.setOverwriteDestination(true), "setOverwriteDestination");
                Ew.Check(image.save(), "save");
                return png;
            });
            foreach (var automated in new[] { false, true })
                ev.Step($"export pdf, generate automated drawings = {automated}", () =>
                {
                    var pdf = Path.Combine(outDir, $"{name}-automated-{automated}.pdf");
                    var export = project.newEwProjectExportPDF(out err); Ew.Check(err, "newEwProjectExportPDF");
                    Ew.Check(export.setExportToPDFFileName(pdf), "setExportToPDFFileName");
                    Ew.Check(export.setAllProjectFiles(true), "setAllProjectFiles");
                    Ew.Check(export.setGenerateAutomatedDrawings(automated), "setGenerateAutomatedDrawings");
                    Ew.Check(export.setSilentMode(true), "setSilentMode");
                    Ew.Check(export.exportPDF(), "exportPDF");
                    return pdf;
                });
            return new Dictionary<string, object> { ["steps"] = ev.Steps };
        }
    }
}
```

`addin/src/AddLife.SweBridge/Probes/ProbeArchive.cs`:

```csharp
using System.Collections.Generic;
using System.IO;
using System.Linq;
using AddLife.SweBridge.Core;
using AddLife.SweBridge.Electrical;
using EwAPI;

namespace AddLife.SweBridge.Probes
{
    /// <summary>S10: archive a SPIKE project, unarchive it, and archive the scratch library as an environment archive.</summary>
    sealed class ProbeArchive : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("probe_archive", "Phase 0 (S10).", true, false,
            Req("project_name", "string", "a SPIKE-* project"),
            Req("out_dir", "string", "folder for the archives"));

        public override string TargetProject(Args args, string batchProject) => args.Str("project_name");

        protected override object Run(EwContext ctx, Args a)
        {
            var ev = new Evidence();
            var name = a.Str("project_name");
            var outDir = a.Str("out_dir");
            Directory.CreateDirectory(outDir);
            var manager = ctx.Projects();
            var project = ctx.FindProject(name);
            var id = project.getID();
            EwErrorCode err;

            var zip = Path.Combine(outDir, name + ".proj.tewzip");
            var code = ev.Code("archive(object[] ids)", () => manager.archive(new object[] { id }, zip, true));
            if (code != EwErrorCode.EW_NO_ERROR) ev.Code("archive(int[] ids)", () => manager.archive(new[] { id }, zip, true));
            ev.Step("archive size", () => File.Exists(zip) ? new FileInfo(zip).Length : -1);

            var before = ctx.AllProjects().Select(p => p.getID()).ToList();
            ev.Step("unarchive", () =>
            {
                var result = manager.unarchive(zip, true, out err); Ew.Check(err, "unarchive");
                return result is object[] items ? items.Select(i => (object)i?.ToString()).ToList() : new List<object> { result?.ToString() };
            });
            ev.Step("projects created by unarchive", () => ctx.AllProjects().Where(p => !before.Contains(p.getID()))
                .Select(p => (object)(p.getID() + " " + p.getName(out err))).ToList());

            var envZip = Path.Combine(outDir, "env-ADDLIFE_SPIKE.tewzip");
            var archiver = ev.Step("getEwArchiveEnvironment", () => { var x = ctx.Env().getEwArchiveEnvironment(out err); Ew.Check(err, "getEwArchiveEnvironment"); return x; });
            ev.Code("env.setArchivePath", () => archiver.setArchivePath(envZip));
            ev.Code("env.setArchiveMode(from library)", () => archiver.setArchiveMode(EwArchiveMode.kArchiveModeObjectFromLibrary));
            ev.Code("env.setLibraries", () => archiver.setLibraries(new object[] { "ADDLIFE_SPIKE" }));
            ev.Code("env.setArchiveProject(false)", () => archiver.setArchiveProject(false));
            ev.Code("env.archive", () => archiver.archive());
            ev.Step("env last archive error", () => archiver.getLastArchiveError(out err));
            ev.Step("env archive size", () => File.Exists(envZip) ? new FileInfo(envZip).Length : -1);
            return new Dictionary<string, object> { ["archive"] = zip, ["steps"] = ev.Steps };
        }
    }
}
```

`addin/src/AddLife.SweBridge/Probes/ProbeWires.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;
using AddLife.SweBridge.Core;
using AddLife.SweBridge.Electrical;
using EwAPI;

namespace AddLife.SweBridge.Probes
{
    /// <summary>S9: the wires as the API sees them, for comparison with the SQL netlist.</summary>
    sealed class ProbeWires : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("probe_wires", "Phase 0 (S9).", false, true);

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.CurrentProject();
            var tags = Ew.Array<EwProjectComponentX>(project.getEwProjectComponentManager(out var err).getEwProjectComponentArray(out err))
                .ToDictionary(c => c.getID(), c => c.getTag(out err));
            return Ew.Array<EwProjectWireX>(project.getEwProjectWireManager(out err).getEwProjectWireArray(out err))
                .Select(w =>
                {
                    var from = w.getOriginComponentID(out err);
                    var to = w.getDestinationComponentID(out err);
                    return new Dictionary<string, object>
                    {
                        ["id"] = w.getID(),
                        ["tag"] = w.getTag(out err),
                        ["from_tag"] = tags.TryGetValue(from, out var f) ? f : null,
                        ["from_terminal_no"] = w.getOriginComponentTerminalNumber(out err),
                        ["to_tag"] = tags.TryGetValue(to, out var t) ? t : null,
                        ["to_terminal_no"] = w.getDestinationComponentTerminalNumber(out err),
                    };
                }).ToList();
        }
    }
}
```

`addin/src/AddLife.SweBridge/Probes/ProbeCleanup.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;
using AddLife.SweBridge.Core;
using AddLife.SweBridge.Electrical;
using EwAPI;

namespace AddLife.SweBridge.Probes
{
    /// <summary>Removes everything Phase 0 created: SPIKE-* projects, the scratch parts, symbol, macro and library.</summary>
    sealed class ProbeCleanup : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("probe_cleanup", "Phase 0: delete all spike objects.", true, false);
        public override string TargetProject(Args args, string batchProject) => "SPIKE-cleanup";

        protected override object Run(EwContext ctx, Args a)
        {
            var ev = new Evidence();
            var env = ctx.Env();
            EwErrorCode err;
            foreach (var project in ctx.AllProjects().Where(p => p.getName(out err).StartsWith("SPIKE-")))
            {
                var id = project.getID();
                if (ctx.App.isEwProjectOpened(id, out err)) ev.Code($"close project {id}", () => ctx.App.closeEwProjectID(id));
                ev.Code($"remove project {id} {project.getName(out err)}", () => project.remove());
            }
            var parts = env.getEwManufacturerPartManager(out err);
            foreach (var reference in new[] { "ADL-SPIKE-BOX", "ADL-SPIKE-DI2" })
                ev.Step("remove part " + reference, () => parts.findByManufacturerAndReference("ADDLIFE", reference, out err)?.remove().ToString());
            ev.Step("remove symbol ADL_SPIKE_BOX", () => env.getEwSymbolManager(out err).findEwSymbolXByName("ADL_SPIKE_BOX", out err)?.remove().ToString());
            ev.Step("remove macro ADL_M_SPIKE", () => Ew.Array<EwProjectX>(env.getEwMacroManager(out err).getEwProjectArray(out err))
                .Where(m => m.getName(out err) == "ADL_M_SPIKE").Select(m => (object)m.remove().ToString()).ToList());
            ev.Step("remove library ADDLIFE_SPIKE", () => Ew.Array<EwLibraryX>(env.getEwLibraryManager(out err).getEwLibraryArray(out err))
                .Where(l => l.getName(out err) == "ADDLIFE_SPIKE").Select(l => (object)l.remove().ToString()).ToList());
            return new Dictionary<string, object> { ["steps"] = ev.Steps };
        }
    }
}
```

- [ ] **Step 5: Write the spike scripts**

`spike/make_test_symbol.py`:

```python
"""Writes spike/out/adl_spike_box.dxf: a 40 x 20 mm box, two terminal stubs ending at (10,-5) and (30,-5),
and a TAG attribute definition — the graphics the library probe loads with insertFromDwg (S4)."""
from pathlib import Path

import ezdxf

OUT = Path(__file__).parent / "out"


def build(path: Path) -> Path:
    doc = ezdxf.new("R2010", units=4)  # 4 = millimetres
    msp = doc.modelspace()
    msp.add_lwpolyline([(0, 0), (40, 0), (40, 20), (0, 20)], close=True)
    for x in (10, 30):
        msp.add_line((x, 0), (x, -5))
        msp.add_circle((x, -5), radius=0.8)
    msp.add_attdef("TAG", insert=(0, 22), dxfattribs={"height": 2.5, "prompt": "Tag"})
    path.parent.mkdir(parents=True, exist_ok=True)
    doc.saveas(path)
    return path


if __name__ == "__main__":
    print(build(OUT / "adl_spike_box.dxf").resolve())
```

`spike/run_probes.py`:

```python
"""Runs one Phase 0 probe against the live bridge and saves the evidence to spike/results/<name>.json."""
import argparse
import datetime
import json
from pathlib import Path

from swe.client import BridgeClient

HERE = Path(__file__).parent
RESULTS = HERE / "results"
OUT = (HERE / "out").resolve()


def save(name: str, response: dict) -> Path:
    RESULTS.mkdir(exist_ok=True)
    path = RESULTS / f"{name}.json"
    path.write_text(json.dumps(response, indent=2, ensure_ascii=False), encoding="utf-8")
    return path


def summarise(response: dict) -> None:
    for result in response.get("results", []):
        print(result["op"], result["status"], result.get("error", ""))
        steps = (result.get("result") or {}).get("steps", []) if isinstance(result.get("result"), dict) else []
        for step in steps:
            mark = "ok " if step["ok"] else "NO "
            print(f"  {mark}{step['step']}: {step.get('value', step.get('error'))}")


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("probe", choices=["s1", "sheet", "library", "box", "template", "archive", "wires", "cleanup"])
    parser.add_argument("--project", help="SPIKE-* project for template, archive and wires")
    parser.add_argument("--sheet-id", type=int, help="sheet id for the template probe")
    parser.add_argument("--symbol", default="TR-AU01", help="symbol for the sheet probe")
    args = parser.parse_args()

    client = BridgeClient.connect()
    stamp = datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
    session = f"spike-{stamp}"
    OUT.mkdir(exist_ok=True)

    if args.probe == "s1":
        name, ops, project = "s1", [{"op": "probe_s1"}], None
    elif args.probe in ("sheet", "box"):
        symbol = args.symbol if args.probe == "sheet" else "ADL_SPIKE_BOX"
        project_name = f"SPIKE-{stamp}"
        name, project = f"{args.probe}-{stamp}", None
        ops = [{"op": "probe_sheet", "args": {"project_name": project_name, "template": "ADDLIFE Template",
                                              "symbol": symbol, "out_dir": str(OUT)}}]
    elif args.probe == "library":
        dxf = OUT / "adl_spike_box.dxf"
        name, project, ops = "library", None, [{"op": "probe_library", "args": {"dxf_path": str(dxf)}}]
    elif args.probe == "template":
        name, project = "template", args.project
        ops = [{"op": "probe_template", "args": {"sheet_id": args.sheet_id, "out_dir": str(OUT)}}]
    elif args.probe == "archive":
        name, project = "archive", None
        ops = [{"op": "probe_archive", "args": {"project_name": args.project, "out_dir": str(OUT)}}]
    elif args.probe == "wires":
        name, project, ops = "wires", args.project, [{"op": "probe_wires"}]
    else:
        name, project, ops = "cleanup", None, [{"op": "probe_cleanup"}]

    response = client.batch(ops, project=project, session=session, stop_on_error=False)
    print("saved", save(name, response))
    summarise(response)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 6: Build, deploy and run the probes in order**

```powershell
dotnet test addin\tests\AddLife.SweBridge.Tests -c Release   # unit tests still pass, probes compile
# [user] close SOLIDWORKS Electrical
pwsh addin\deploy.ps1
# [user] start SOLIDWORKS Electrical
.venv\Scripts\python spike\make_test_symbol.py
.venv\Scripts\python spike\run_probes.py s1
.venv\Scripts\python spike\run_probes.py sheet
```
Expected for `s1`: `license_result` `EW_NO_ERROR`; `op_thread` equals `connect_thread` (operations run on the UI thread).
Expected for `sheet`: a printed step list and `spike/results/sheet-<stamp>.json`. **View** `spike/out/SPIKE-<stamp>-open.png` and `-closed.png`: both symbols, the two lines, the text and the AddLife title block must be visible. Note in the findings which steps printed `NO`.

Then, using the project name and `sheet_id` printed by the sheet probe:

```powershell
.venv\Scripts\python spike\run_probes.py library
.venv\Scripts\python spike\run_probes.py box
.venv\Scripts\python spike\run_probes.py wires --project SPIKE-<stamp-of-sheet-run>
.venv\Scripts\python spike\run_probes.py template --project SPIKE-<stamp-of-sheet-run> --sheet-id <sheet_id>
.venv\Scripts\python spike\run_probes.py archive --project SPIKE-<stamp-of-sheet-run>
```
Expected: one results file per run. **View** the `box` run's PNG (the black-box symbol with its tag) and `SPIKE-<stamp>-titleblock.png` (does "Spike Customer Ω" and "SPK-0001" appear in the title block?). Open the two `-automated-*.pdf` files and compare their page counts with `.venv\Scripts\python -c "import pypdf,sys; print([len(pypdf.PdfReader(p).pages) for p in sys.argv[1:]])" spike\out\*automated*.pdf`.

Do **not** run `cleanup` yet — Task 0.8 needs the SPIKE project.

- [ ] **Step 7: Commit the probes and the evidence**

```powershell
git add addin/src spike/make_test_symbol.py spike/run_probes.py spike/results
git commit -m "spike: Phase 0 probe operations, runner and evidence"
```

### Task 0.8: SQL netlist against the API's wires (S9)

**Files:**
- Create: `spike/netlist_compare.py`
- Create (evidence): `spike/results/s9.json`

**Interfaces:**
- Consumes: `probe_wires` (Task 0.7), `BridgeClient`, `require_ok` (Task 0.6)
- Produces: the validated `NETLIST_SQL` text that Task 1.1 copies into `swe/db/queries.py`

- [ ] **Step 1: Write the comparison script**

`spike/netlist_compare.py`:

```python
"""S9: does a read-only SQL query reproduce the wires the EwAPI reports for the same project?"""
import argparse
import json
from pathlib import Path

import pyodbc

from swe.client import BridgeClient, require_ok

NETLIST_SQL = """
SELECT w.wir_id AS id, w.wir_tag AS tag,
       cf.com_tag AS from_tag, w.wir_cte_nofrom AS from_terminal_no, tf.cte_txt AS from_terminal,
       ct.com_tag AS to_tag, w.wir_cte_noto AS to_terminal_no, tt.cte_txt AS to_terminal
FROM dbo.tew_wire w
LEFT JOIN dbo.tew_component cf ON cf.com_id = w.wir_com_idfrom
LEFT JOIN dbo.tew_component ct ON ct.com_id = w.wir_com_idto
LEFT JOIN dbo.tew_componentterminal tf ON tf.cte_cel_id = w.wir_cel_idfrom AND tf.cte_no = w.wir_cte_nofrom
LEFT JOIN dbo.tew_componentterminal tt ON tt.cte_cel_id = w.wir_cel_idto AND tt.cte_no = w.wir_cte_noto
ORDER BY w.wir_id
"""


def connect(database: str):
    return pyodbc.connect("DRIVER={ODBC Driver 17 for SQL Server};SERVER=.\\TEW_SQLEXPRESS;"
                          f"DATABASE={database};Trusted_Connection=yes;", autocommit=True, readonly=True)


def rows(connection, sql: str, *params) -> list[dict]:
    cursor = connection.cursor()
    cursor.execute(sql, *params)
    columns = [c[0] for c in cursor.description]
    return [dict(zip(columns, r)) for r in cursor.fetchall()]


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("project")
    args = parser.parse_args()

    response = require_ok(BridgeClient.connect().batch([{"op": "probe_wires"}], project=args.project, session="spike-s9"))
    api = {w["id"]: w for w in response["results"][0]["result"]}

    directory = rows(connect("tew_app_project"), "SELECT pro_directory FROM tew.tew_project WHERE pro_name = ?", args.project)
    database = f"tew_project_data_{int(directory[0]['pro_directory'])}"
    sql = {r["id"]: r for r in rows(connect(database), NETLIST_SQL)}

    differences = []
    for wire_id in sorted(set(api) | set(sql)):
        a, s = api.get(wire_id), sql.get(wire_id)
        if a is None or s is None:
            differences.append({"id": wire_id, "only_in": "sql" if a is None else "api"})
            continue
        for field in ("tag", "from_tag", "from_terminal_no", "to_tag", "to_terminal_no"):
            if (a.get(field) or None) != (s.get(field) or None):
                differences.append({"id": wire_id, "field": field, "api": a.get(field), "sql": s.get(field)})

    report = {"project": args.project, "database": database, "api_wires": len(api), "sql_wires": len(sql),
              "differences": differences, "sql_rows": list(sql.values())}
    out = Path(__file__).parent / "results" / "s9.json"
    out.write_text(json.dumps(report, indent=2, ensure_ascii=False, default=str), encoding="utf-8")
    print(f"api {len(api)} wires, sql {len(sql)} wires, {len(differences)} differences -> {out}")
    return 0 if not differences and api else 1


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 2: Run it on the SPIKE project from Task 0.7**

Run: `.venv\Scripts\python spike\netlist_compare.py SPIKE-<stamp-of-sheet-run>`
Expected: `api N wires, sql N wires, 0 differences` with N ≥ 1, exit code 0. If there are differences, the report says which column disagrees; record it — Task 1.1 then either fixes the join or falls back to API-only reads (spec S9 fallback).

- [ ] **Step 3: Commit**

```powershell
git add spike/netlist_compare.py spike/results/s9.json
git commit -m "spike: S9 SQL netlist compared with the API's wires"
```

### Task 0.9: Findings, amendments, clean-up — the Phase 0 gate

**Files:**
- Create: `docs/superpowers/specs/YYYY-MM-DD-phase0-findings.md` (the date the findings are written)
- Modify: `docs/superpowers/specs/2026-10-09-solidworks-electrical-skill-design.md` (append a numbered amendment section only where a finding changes the design)
- Modify: this plan's Phase 1 tasks, where a finding contradicts an assumption listed below
- Delete: `addin/src/AddLife.SweBridge/Probes/`; modify `Electrical/Operations.cs` (remove the probe line)

**Interfaces:**
- Produces: the Phase 1 go/no-go decision, and the confirmed values Phase 1 relies on: language code, sheet-position meaning, symbol-to-component link, render and export behaviour, netlist SQL.

**Phase 1 assumes these answers.** Any "no" means the named Phase 1 task is amended before Phase 1 starts:

| # | Phase 1 assumes | Phase 1 task affected if not |
|---|---|---|
| S1 | The add-in loads with the licence code; operations run on the UI thread | All — Phase 1 does not start |
| S2a | `setName` → `insertFromTemplate("ADDLIFE Template")` → `openEwProjectID` creates and opens a project | 1.3 |
| S2b | Books take a manual tag; a sheet's `setPosition(n)` after `insert` gives position `n` within its book | 1.4 |
| S2c | `setEwSymbolName` + `setX/YPosition` + `insert` places a symbol; `setObjectID(componentId)` before `insert` links it to that component | 1.5 |
| S2d | A schematic line whose ends lie on two symbols' connection points becomes one wire between the two components | 1.5, 1.10 |
| S2e | The language code is `en` (`getCurrentCodeLanguage`) | config default |
| S2f | Symbols, lines and texts listed by sheet id come back as objects (not ids) | 1.6 |
| S3 | Multilingual text inserts and reads back unchanged, Ω included | 1.5 |
| S8 | A sheet renders to PNG from its `.ewg` path (open or closed — record which); PDF export works for all sheets and for a selection by id; snapshots create | 1.7 |
| S9 | The SQL netlist equals the API's wires | 1.1, 1.2 |
| S10 | `archive` + `unarchive` round-trips a project; record the restored project's name | 1.7, 1.9 |

S4–S7 do not block Phase 1; their answers shape the Phase 2 and Phase 3 plans.

- [ ] **Step 1: Write the findings document**

Create `docs/superpowers/specs/YYYY-MM-DD-phase0-findings.md` with this structure, filling every row from the files in `spike/results/` and the PNGs and PDFs viewed:

```markdown
# Phase 0 findings — SOLIDWORKS Electrical 2026 SP4.1

Evidence: `spike/results/*.json` (commit <hash>). Renders and PDFs were viewed, not committed.

| # | Question (spec §11) | Answer | Evidence (file → step names) | Consequence |
|---|---|---|---|---|
| S1 | … | yes / no / partly | s1.json → license_result, op_thread | … |
| …  one row per S1–S10 … |

## Values Phase 1 uses
- Language code: …
- Sheet position: …
- Symbol ↔ component link: …
- Render: works open / closed / both …
- Restored project name after unarchive: …

## Surprises
- … anything the API did that the design did not expect …

## Decision
Phase 1 starts as planned / starts with amendments A-1 … / does not start because …
```

- [ ] **Step 2: Amend the spec and this plan where needed**

For each "no" or "partly": append a numbered amendment section to the spec (e.g. `## 15. Amendment A-1 (Phase 0, S2b)`) stating the observed behaviour and the changed design; then edit the affected Phase 1 task in this plan so its code matches the observed behaviour. If every answer is "yes", write "No amendments" under Decision and change nothing.

- [ ] **Step 3: Clean up Electrical**

```powershell
.venv\Scripts\python spike\run_probes.py cleanup
```
Expected: every step `ok`; Electrical's project list has no `SPIKE-*` project; the `ADDLIFE_SPIKE` library is gone. Check the existing projects are untouched: `.venv\Scripts\python -c "import pyodbc; c=pyodbc.connect('DRIVER={ODBC Driver 17 for SQL Server};SERVER=.\\TEW_SQLEXPRESS;DATABASE=tew_app_project;Trusted_Connection=yes;'); print(c.execute('SELECT pro_id, pro_name, pro_modificationdate FROM tew.tew_project ORDER BY pro_id').fetchall())"` — the 13 projects listed on 2026-10-09 with their modification dates unchanged.

- [ ] **Step 4: Remove the probes**

Delete the folder `addin/src/AddLife.SweBridge/Probes/`. In `addin/src/AddLife.SweBridge/Electrical/Operations.cs` delete the line `if (config.ProbesEnabled) Probes.ProbeSet.Register(registry);`. In `%LOCALAPPDATA%\AddLife\swe-bridge\config.json` set `"probes_enabled": false`.

Run: `dotnet test addin\tests\AddLife.SweBridge.Tests -c Release`
Expected: builds; all unit tests pass.

- [ ] **Step 5: Commit and push**

```powershell
git add -A docs addin spike
git commit -m "docs: Phase 0 findings; remove spike probes"
git push
```

**Gate:** the findings go to the reviewer with the go/no-go decision. Phase 1 starts only on a "go".

---

# Phase 1 — Bridge

Starts only after the Phase 0 gate. Every operation task follows the same rhythm: write the selftest step that exercises it (fails: unknown operation), implement the operation, deploy, run the step (passes), commit.

### Task 1.1: Read-only SQL layer with schema guard

**Files:**
- Modify: `swe/src/swe/errors.py` (append three classes)
- Create: `swe/src/swe/db/__init__.py`, `swe/src/swe/db/connection.py`, `swe/src/swe/db/guard.py`, `swe/src/swe/db/queries.py`
- Test: `swe/tests/test_db_connection.py`, `swe/tests/test_db_guard.py`, `swe/tests/test_db_queries.py`, `swe/tests/fakedb.py`

**Interfaces:**
- Consumes: `NETLIST_SQL` as validated in Task 0.8
- Produces:
  - `swe.errors`: `SchemaMismatch(SweError)`, `ReadOnlyViolation(SweError)`, `ConfigError(SweError)`
  - `swe.db.connection`: `assert_select_only(sql)`, `check_database_name(name) -> str`, `Db(connector=odbc_connector)` with `.query(database, sql, params=()) -> list[dict]`, `.close()`
  - `swe.db.guard`: `PINNED_APP_PROJECT_SCHEMA = 32`, `PINNED_PROJECT_DATA_SCHEMA = 292`, `check_app(db)`, `check_project(db, database)`
  - `swe.db.queries`: `projects(db)`, `project_database(db, project) -> str`, `sheets(db, project)`, `components(db, project)`, `netlist(db, project)`, `search_symbols(db, text, limit=50)`, `netlist_diff(before, after) -> dict`

- [ ] **Step 1: Append the error classes**

Append to `swe/src/swe/errors.py`:

```python


class SchemaMismatch(SweError):
    """Electrical's database is not the schema this swe was verified against."""


class ReadOnlyViolation(SweError):
    """Something tried to send a non-SELECT statement to Electrical's database."""


class ConfigError(SweError):
    """A required swe setting is missing."""
```

- [ ] **Step 2: Write the failing tests**

`swe/tests/fakedb.py`:

```python
"""An in-memory stand-in for pyodbc: answers queries by matching a substring of the SQL."""
from __future__ import annotations


class FakeCursor:
    def __init__(self, answers: dict[str, list[dict]], log: list):
        self._answers, self._log = answers, log
        self.description, self._rows = None, []

    def execute(self, sql: str, *params):
        self._log.append((sql, params))
        for needle, rows in self._answers.items():
            if needle in sql:
                columns = list(rows[0].keys()) if rows else ["x"]
                self.description = [(c,) for c in columns]
                self._rows = [tuple(r[c] for c in columns) for r in rows]
                return self
        raise AssertionError(f"unexpected SQL: {sql}")

    def fetchall(self):
        return self._rows


class FakeConnection:
    def __init__(self, answers, log):
        self._answers, self._log = answers, log

    def cursor(self):
        return FakeCursor(self._answers, self._log)

    def close(self):
        pass


def connector(answers_by_database: dict[str, dict[str, list[dict]]], log: list | None = None):
    log = log if log is not None else []

    def connect(database: str):
        return FakeConnection(answers_by_database.get(database, {}), log)

    return connect
```

`swe/tests/test_db_connection.py`:

```python
import pytest

from fakedb import connector
from swe.db.connection import Db, assert_select_only, check_database_name
from swe.errors import ReadOnlyViolation, SweError


@pytest.mark.parametrize("sql", [
    "SELECT 1",
    "  select pro_name from tew.tew_project where pro_name = ?",
    "WITH x AS (SELECT 1 AS a) SELECT a FROM x",
    "SELECT fil_updatedate, pro_createdby, cre_update FROM t",  # words inside column names are fine
    "SELECT 'insert into me' AS s",                             # words inside string literals are fine
    "SELECT 1;",
])
def test_select_is_allowed(sql):
    assert_select_only(sql)


@pytest.mark.parametrize("sql", [
    "UPDATE tew_component SET com_tag = 'x'",
    "DELETE FROM tew_wire",
    "SELECT 1; DROP TABLE tew_wire",
    "SELECT * INTO copy FROM tew_wire",
    "EXEC sp_who",
    "/* SELECT */ INSERT INTO t VALUES (1)",
    "WITH x AS (SELECT 1 AS a) DELETE FROM t",
    "",
])
def test_anything_else_is_refused(sql):
    with pytest.raises(ReadOnlyViolation):
        assert_select_only(sql)


@pytest.mark.parametrize("name", ["tew_app_project", "tew_project_data_25", "tew_catalog", "tew_app_data"])
def test_electrical_database_names(name):
    assert check_database_name(name) == name


@pytest.mark.parametrize("name", ["master", "tew_project_data_25; DROP", "msdb", "tew_"])
def test_other_database_names_are_refused(name):
    with pytest.raises(SweError):
        check_database_name(name)


def test_query_returns_dicts_and_reuses_connections():
    log = []
    db = Db(connector({"tew_app_project": {"FROM t": [{"a": 1, "b": "x"}]}}, log))
    assert db.query("tew_app_project", "SELECT a, b FROM t") == [{"a": 1, "b": "x"}]
    assert db.query("tew_app_project", "SELECT a, b FROM t WHERE a = ?", [1]) == [{"a": 1, "b": "x"}]
    assert log[1][1] == (1,)


def test_query_refuses_a_write_before_connecting():
    db = Db(connector({}))
    with pytest.raises(ReadOnlyViolation):
        db.query("tew_app_project", "DELETE FROM t")
```

`swe/tests/test_db_guard.py`:

```python
import pytest

from fakedb import connector
from swe.db.connection import Db
from swe.db.guard import APP_COLUMNS, PROJECT_COLUMNS, SYMBOL_COLUMNS, check_app, check_project
from swe.errors import SchemaMismatch


def columns(spec: dict[str, list[str]]) -> list[dict]:
    return [{"TABLE_NAME": t, "COLUMN_NAME": c} for t, cs in spec.items() for c in cs]


def app_db(version=32, app_columns=None, symbol_columns=None):
    return Db(connector({
        "tew_app_project": {"tew_version": [{"ver_projects": version}],
                            "INFORMATION_SCHEMA": columns(app_columns or APP_COLUMNS)},
        "tew_app_data": {"INFORMATION_SCHEMA": columns(symbol_columns or SYMBOL_COLUMNS)},
    }))


def project_db(version=292, project_columns=None):
    return Db(connector({"tew_project_data_25": {
        "tew_version": [{"ver_projectdata": version}],
        "INFORMATION_SCHEMA": columns(project_columns or PROJECT_COLUMNS)}}))


def test_pinned_app_schema_passes():
    check_app(app_db())


def test_other_app_schema_version_is_refused_with_the_routine():
    with pytest.raises(SchemaMismatch, match="33.*32.*selftest"):
        check_app(app_db(version=33))


def test_missing_column_is_named():
    reduced = {**APP_COLUMNS, "tew_project": [c for c in APP_COLUMNS["tew_project"] if c != "pro_directory"]}
    with pytest.raises(SchemaMismatch, match="tew.tew_project.pro_directory"):
        check_app(app_db(app_columns=reduced))


def test_pinned_project_schema_passes():
    check_project(project_db(), "tew_project_data_25")


def test_older_project_schema_says_open_it_in_electrical():
    with pytest.raises(SchemaMismatch, match="open the project once in Electrical"):
        check_project(project_db(version=291), "tew_project_data_25")
```

`swe/tests/test_db_queries.py`:

```python
import datetime

import pytest

from fakedb import connector
from swe.db import queries
from swe.db.connection import Db
from swe.db.guard import APP_COLUMNS, PROJECT_COLUMNS, SYMBOL_COLUMNS
from swe.errors import SweError
from test_db_guard import columns

PROJECTS = [
    {"id": 1, "name": "Addlife Template", "directory": "1", "schema_version": 292, "modified": datetime.datetime(2025, 10, 6)},
    {"id": 26, "name": "LIBTEST", "directory": "26", "schema_version": 292, "modified": datetime.datetime(2026, 10, 12)},
]


def db_with(project_answers=None):
    return Db(connector({
        "tew_app_project": {"tew_version": [{"ver_projects": 32}], "INFORMATION_SCHEMA": columns(APP_COLUMNS),
                            "FROM tew.tew_project": PROJECTS},
        "tew_app_data": {"INFORMATION_SCHEMA": columns(SYMBOL_COLUMNS),
                         "vew_data_symbols": [{"name": "TR-AU01", "library": "IEC", "description": "NO switch"}]},
        "tew_project_data_26": {"tew_version": [{"ver_projectdata": 292}], "INFORMATION_SCHEMA": columns(PROJECT_COLUMNS),
                                **(project_answers or {})},
    }))


def test_project_database_by_name_and_by_id():
    assert queries.project_database(db_with(), "LIBTEST") == "tew_project_data_26"
    assert queries.project_database(db_with(), "26") == "tew_project_data_26"


def test_unknown_project_is_an_error():
    with pytest.raises(SweError, match="no Electrical project named 'Nope'"):
        queries.project_database(db_with(), "Nope")


def test_netlist_rows():
    wire = {"id": 7, "tag": "711", "from_tag": "-K1", "from_terminal_no": 1, "from_terminal": "13",
            "to_tag": "-K2", "to_terminal_no": 2, "to_terminal": "A1"}
    assert queries.netlist(db_with({"FROM dbo.tew_wire": [wire]}), "LIBTEST") == [wire]


def test_search_symbols_passes_the_pattern_as_a_parameter():
    log = []
    db = Db(connector({"tew_app_project": {"tew_version": [{"ver_projects": 32}], "INFORMATION_SCHEMA": columns(APP_COLUMNS)},
                       "tew_app_data": {"INFORMATION_SCHEMA": columns(SYMBOL_COLUMNS),
                                        "vew_data_symbols": [{"name": "TR-AU01"}]}}, log))
    assert queries.search_symbols(db, "switch'; DROP", limit=5) == [{"name": "TR-AU01"}]
    sql, params = log[-1]
    assert "DROP" not in sql
    assert params == (5, "%switch'; DROP%", "%switch'; DROP%")


def test_netlist_diff_ignores_order_and_reports_changes():
    before = [{"id": 1, "tag": "1", "from_tag": "-K1", "from_terminal": "13", "to_tag": "-K2", "to_terminal": "A1"},
              {"id": 2, "tag": "2", "from_tag": "-K2", "from_terminal": "A2", "to_tag": "-X1", "to_terminal": "1"}]
    after = [{"id": 9, "tag": "2", "from_tag": "-X1", "from_terminal": "1", "to_tag": "-K2", "to_terminal": "A2"},
             {"id": 3, "tag": "3", "from_tag": "-K1", "from_terminal": "14", "to_tag": "-H1", "to_terminal": "X1"}]
    diff = queries.netlist_diff(before, after)
    assert diff["added"] == [{"from": "-H1:X1", "to": "-K1:14", "tag": "3"}]
    assert diff["removed"] == [{"from": "-K1:13", "to": "-K2:A1", "tag": "1"}]
    assert diff["unchanged"] == 1
```

- [ ] **Step 3: Run the tests to see them fail**

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: `ModuleNotFoundError: No module named 'swe.db'`.

- [ ] **Step 4: Implement the SQL layer**

`swe/src/swe/db/__init__.py`:

```python
"""Read-only access to SOLIDWORKS Electrical's SQL Server. Every change goes through the add-in, never here."""
```

`swe/src/swe/db/connection.py`:

```python
from __future__ import annotations

import re
from typing import Any, Callable, Sequence

from ..errors import ReadOnlyViolation, SweError

SERVER = r".\TEW_SQLEXPRESS"
_DATABASE_NAME = re.compile(r"^tew_[a-z]+(_[a-z]+)*(_\d+)?$")
_COMMENTS = re.compile(r"--[^\n]*|/\*.*?\*/", re.DOTALL)
_STRINGS = re.compile(r"'(?:''|[^'])*'")
_FORBIDDEN = re.compile(
    r"\b(insert|update|delete|merge|drop|alter|create|truncate|exec|execute|grant|revoke|deny|backup|restore"
    r"|dbcc|into|openrowset|openquery|opendatasource|bulk|shutdown|waitfor)\b", re.IGNORECASE)


def assert_select_only(sql: str) -> None:
    """Refuse anything but one SELECT (or WITH … SELECT) statement. Comments and string literals are ignored."""
    bare = _STRINGS.sub("''", _COMMENTS.sub(" ", sql)).strip()
    if bare.endswith(";"):
        bare = bare[:-1].rstrip()
    if ";" in bare:
        raise ReadOnlyViolation("only one statement is allowed")
    first = bare.split(None, 1)[0].lower() if bare else ""
    if first not in ("select", "with"):
        raise ReadOnlyViolation(f"only SELECT is allowed, got {first.upper() or 'nothing'}")
    hit = _FORBIDDEN.search(bare)
    if hit:
        raise ReadOnlyViolation(f"'{hit.group(1).upper()}' is not allowed in a read-only query")


def check_database_name(name: str) -> str:
    if not _DATABASE_NAME.match(name):
        raise SweError(f"not a SOLIDWORKS Electrical database name: {name!r}")
    return name


def odbc_connector(database: str):
    import pyodbc

    return pyodbc.connect(
        f"DRIVER={{ODBC Driver 17 for SQL Server}};SERVER={SERVER};DATABASE={database};Trusted_Connection=yes;",
        autocommit=True, readonly=True, timeout=10)


class Db:
    def __init__(self, connector: Callable[[str], Any] = odbc_connector):
        self._connector = connector
        self._connections: dict[str, Any] = {}

    def query(self, database: str, sql: str, params: Sequence[Any] = ()) -> list[dict[str, Any]]:
        assert_select_only(sql)
        connection = self._connections.get(database)
        if connection is None:
            connection = self._connections[database] = self._connector(check_database_name(database))
        cursor = connection.cursor()
        if params:
            cursor.execute(sql, *params)
        else:
            cursor.execute(sql)
        names = [c[0] for c in cursor.description]
        return [dict(zip(names, row)) for row in cursor.fetchall()]

    def close(self) -> None:
        for connection in self._connections.values():
            connection.close()
        self._connections.clear()
```

`swe/src/swe/db/guard.py`:

```python
"""Refuse to read a database whose schema this swe was not verified against (spec §6)."""
from __future__ import annotations

from ..errors import SchemaMismatch

PINNED_APP_PROJECT_SCHEMA = 32     # tew_app_project.dbo.tew_version.ver_projects, Electrical 2026 SP4.1
PINNED_PROJECT_DATA_SCHEMA = 292   # tew_project_data_N.dbo.tew_version.ver_projectdata

APP_COLUMNS = {"tew_project": ["pro_id", "pro_name", "pro_directory", "pro_version", "pro_modificationdate"]}
SYMBOL_COLUMNS = {"vew_data_symbols": ["blo_filename", "blo_lib_name", "blo_standard", "blo_blocktype",
                                       "blo_circuitcount", "blo_pointcount", "blo_rootmark", "blo_tra_0"]}
PROJECT_COLUMNS = {
    "tew_file": ["fil_id", "fil_bun_id", "fil_fol_id", "fil_orderno", "fil_filetype", "fil_title", "fil_filename"],
    "tew_component": ["com_id", "com_tag", "com_loc_id", "com_fun_id", "com_type"],
    "tew_componentterminal": ["cte_cel_id", "cte_no", "cte_txt"],
    "tew_wire": ["wir_id", "wir_tag", "wir_com_idfrom", "wir_com_idto", "wir_cel_idfrom", "wir_cel_idto",
                 "wir_cte_nofrom", "wir_cte_noto"],
}

UPGRADE_ROUTINE = ("After an Electrical update: run tools/dump-ewapi.ps1 and diff docs/reference, run `swe selftest`, "
                   "then raise the pinned value in swe/db/guard.py.")


def _missing(db, database: str, schema: str, wanted: dict[str, list[str]]) -> list[str]:
    rows = db.query(database, "SELECT TABLE_NAME, COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_SCHEMA = ?",
                    [schema])
    have = {(r["TABLE_NAME"], r["COLUMN_NAME"]) for r in rows}
    return [f"{schema}.{table}.{column}" for table, cols in wanted.items() for column in cols if (table, column) not in have]


def check_app(db) -> None:
    rows = db.query("tew_app_project", "SELECT ver_projects FROM dbo.tew_version")
    found = rows[0]["ver_projects"] if rows else None
    if found != PINNED_APP_PROJECT_SCHEMA:
        raise SchemaMismatch(f"Electrical's application database schema is {found}, but swe was verified against "
                             f"{PINNED_APP_PROJECT_SCHEMA}. {UPGRADE_ROUTINE}")
    missing = _missing(db, "tew_app_project", "tew", APP_COLUMNS) + _missing(db, "tew_app_data", "tew", SYMBOL_COLUMNS)
    if missing:
        raise SchemaMismatch("Electrical's application database lacks " + ", ".join(missing) + ". " + UPGRADE_ROUTINE)


def check_project(db, database: str) -> None:
    rows = db.query(database, "SELECT ver_projectdata FROM dbo.tew_version")
    found = rows[0]["ver_projectdata"] if rows else None
    if found is not None and found < PINNED_PROJECT_DATA_SCHEMA:
        raise SchemaMismatch(f"{database} has schema {found}, older than {PINNED_PROJECT_DATA_SCHEMA}: "
                             "open the project once in Electrical so it is upgraded, then retry.")
    if found != PINNED_PROJECT_DATA_SCHEMA:
        raise SchemaMismatch(f"{database} has schema {found}, but swe was verified against "
                             f"{PINNED_PROJECT_DATA_SCHEMA}. {UPGRADE_ROUTINE}")
    missing = _missing(db, database, "dbo", PROJECT_COLUMNS)
    if missing:
        raise SchemaMismatch(f"{database} lacks " + ", ".join(missing) + ". " + UPGRADE_ROUTINE)
```

`swe/src/swe/db/queries.py`:

```python
"""The bulk reads swe needs. Each one checks the schema first."""
from __future__ import annotations

from ..errors import SweError
from .guard import check_app, check_project

NETLIST_SQL = """
SELECT w.wir_id AS id, w.wir_tag AS tag,
       cf.com_tag AS from_tag, w.wir_cte_nofrom AS from_terminal_no, tf.cte_txt AS from_terminal,
       ct.com_tag AS to_tag, w.wir_cte_noto AS to_terminal_no, tt.cte_txt AS to_terminal
FROM dbo.tew_wire w
LEFT JOIN dbo.tew_component cf ON cf.com_id = w.wir_com_idfrom
LEFT JOIN dbo.tew_component ct ON ct.com_id = w.wir_com_idto
LEFT JOIN dbo.tew_componentterminal tf ON tf.cte_cel_id = w.wir_cel_idfrom AND tf.cte_no = w.wir_cte_nofrom
LEFT JOIN dbo.tew_componentterminal tt ON tt.cte_cel_id = w.wir_cel_idto AND tt.cte_no = w.wir_cte_noto
ORDER BY w.wir_id
"""


def projects(db) -> list[dict]:
    check_app(db)
    return db.query("tew_app_project",
                    "SELECT pro_id AS id, pro_name AS name, pro_directory AS directory, pro_version AS schema_version, "
                    "pro_modificationdate AS modified FROM tew.tew_project ORDER BY pro_id")


def project_database(db, project: str) -> str:
    matches = [p for p in projects(db) if p["name"] == project or str(p["id"]) == project]
    if not matches:
        raise SweError(f"no Electrical project named '{project}'")
    if len(matches) > 1:
        raise SweError(f"{len(matches)} projects match '{project}' (ids {[m['id'] for m in matches]}); use the id")
    database = f"tew_project_data_{int(matches[0]['directory'])}"
    check_project(db, database)
    return database


def sheets(db, project: str) -> list[dict]:
    database = project_database(db, project)
    return db.query(database, "SELECT fil_id AS id, fil_bun_id AS book_id, fil_fol_id AS folder_id, fil_orderno AS position, "
                              "fil_filetype AS type, fil_title AS title, fil_filename AS file FROM dbo.tew_file "
                              "ORDER BY fil_bun_id, fil_orderno")


def components(db, project: str) -> list[dict]:
    database = project_database(db, project)
    return db.query(database, "SELECT com_id AS id, com_tag AS tag, com_loc_id AS location_id, com_fun_id AS function_id, "
                              "com_type AS type FROM dbo.tew_component ORDER BY com_tag")


def netlist(db, project: str) -> list[dict]:
    return db.query(project_database(db, project), NETLIST_SQL)


def search_symbols(db, text: str, limit: int = 50) -> list[dict]:
    check_app(db)
    pattern = f"%{text}%"
    return db.query("tew_app_data",
                    "SELECT TOP (?) blo_filename AS name, blo_lib_name AS library, blo_standard AS standard, "
                    "blo_blocktype AS type, blo_circuitcount AS circuits, blo_pointcount AS points, "
                    "blo_rootmark AS root_mark, MIN(blo_tra_0) AS description "
                    "FROM tew.vew_data_symbols WHERE blo_filename LIKE ? OR blo_tra_0 LIKE ? "
                    "GROUP BY blo_filename, blo_lib_name, blo_standard, blo_blocktype, blo_circuitcount, blo_pointcount, "
                    "blo_rootmark ORDER BY blo_lib_name, blo_filename",
                    [limit, pattern, pattern])


def _end(row: dict, side: str) -> str:
    terminal = row.get(f"{side}_terminal") or row.get(f"{side}_terminal_no")
    return f"{row.get(f'{side}_tag')}:{terminal}"


def netlist_diff(before: list[dict], after: list[dict]) -> dict:
    """Compares two netlists by their connection (unordered pair of tag:terminal), not by wire id."""
    def index(rows):
        return {tuple(sorted((_end(r, "from"), _end(r, "to")))): r for r in rows}

    old, new = index(before), index(after)

    def report(keys, rows):
        return [{"from": k[0], "to": k[1], "tag": rows[k].get("tag")} for k in sorted(keys)]

    return {"added": report(new.keys() - old.keys(), new),
            "removed": report(old.keys() - new.keys(), old),
            "unchanged": len(old.keys() & new.keys())}
```

- [ ] **Step 5: Run the tests to see them pass**

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: all pass.

- [ ] **Step 6: Live check (read-only) against this PC's SQL Server**

Run: `.venv\Scripts\python -c "from swe.db.connection import Db; from swe.db import queries; db=Db(); print([p['name'] for p in queries.projects(db)]); print(len(queries.search_symbols(db, 'TR-AU01')))"`
Expected: the 13 project names, then `1` or more. No schema error.

- [ ] **Step 7: Commit**

```powershell
git add swe
git commit -m "feat(swe): read-only SQL layer with select-only and schema guards"
```

### Task 1.2: `swe config` and `swe db` commands

**Files:**
- Create: `swe/src/swe/config.py`, `swe/src/swe/commands/config.py`, `swe/src/swe/commands/db.py`
- Modify: `swe/src/swe/cli.py` (`COMMAND_MODULES`)
- Test: `swe/tests/test_config.py`, `swe/tests/test_cli_db.py`

**Interfaces:**
- Consumes: `bridge_dir()` (Task 0.6), `Db`, `queries.*` (Task 1.1)
- Produces:
  - `swe.config`: `DEFAULTS`, `config_path()`, `load_config() -> dict`, `save_setting(key, value) -> dict`, `work_dir(config) -> Path`, `require(config, key, reason)`
  - commands `swe config show|set KEY VALUE`; `swe db projects | sheets P | components P | netlist P [--save FILE] | netlist-diff BEFORE AFTER | search-symbols TEXT [--limit N]`
  - `swe.commands.db.make_db()` — the one place a real `Db` is created (tests replace it)

- [ ] **Step 1: Write the failing tests**

`swe/tests/test_config.py`:

```python
import json

import pytest

from swe.cli import main
from swe.config import load_config, require, work_dir
from swe.errors import ConfigError


def test_defaults_without_a_file(bridge_dir):
    config = load_config()
    assert config["retention_days"] == 30
    assert config["selftest_symbol"] == "TR-AU01"
    assert config["template"] == "ADDLIFE Template"
    assert config["backup_dir"] is None


def test_set_and_show(bridge_dir, capsys):
    assert main(["config", "set", "backup_dir", r"\\nas\electrical-backups"]) == 0
    assert main(["config", "set", "retention_days", "45"]) == 0
    stored = json.loads((bridge_dir / "swe.json").read_text(encoding="utf-8"))
    assert stored == {"backup_dir": r"\\nas\electrical-backups", "retention_days": 45}
    capsys.readouterr()
    assert main(["config", "show"]) == 0
    assert json.loads(capsys.readouterr().out)["retention_days"] == 45


def test_unknown_key_is_refused(bridge_dir, capsys):
    assert main(["config", "set", "backupdir", "x"]) == 2
    assert "unknown setting" in capsys.readouterr().err


def test_require_explains_how_to_set_it(bridge_dir):
    with pytest.raises(ConfigError, match="swe config set backup_dir"):
        require(load_config(), "backup_dir", "decision D-C: where backups go")


def test_work_dir_defaults_to_cwd_work(bridge_dir, tmp_path, monkeypatch):
    monkeypatch.chdir(tmp_path)
    assert work_dir(load_config()) == tmp_path / "work"
```

`swe/tests/test_cli_db.py`:

```python
import json

import pytest

import swe.commands.db as dbcmd
from swe.cli import main
from test_db_queries import db_with


@pytest.fixture(autouse=True)
def fake_db(monkeypatch):
    wire = {"id": 7, "tag": "711", "from_tag": "-K1", "from_terminal_no": 1, "from_terminal": "13",
            "to_tag": "-K2", "to_terminal_no": 2, "to_terminal": "A1"}
    monkeypatch.setattr(dbcmd, "make_db", lambda: db_with({"FROM dbo.tew_wire": [wire]}))


def test_projects(capsys):
    assert main(["db", "projects"]) == 0
    assert [p["name"] for p in json.loads(capsys.readouterr().out)] == ["Addlife Template", "LIBTEST"]


def test_netlist_save_and_diff(tmp_path, capsys):
    saved = tmp_path / "before.json"
    assert main(["db", "netlist", "LIBTEST", "--save", str(saved)]) == 0
    after = tmp_path / "after.json"
    after.write_text("[]", encoding="utf-8")
    capsys.readouterr()
    assert main(["db", "netlist-diff", str(saved), str(after)]) == 0
    diff = json.loads(capsys.readouterr().out)
    assert diff["removed"] == [{"from": "-K1:13", "to": "-K2:A1", "tag": "711"}]


def test_search_symbols(capsys):
    assert main(["db", "search-symbols", "switch", "--limit", "5"]) == 0
    assert json.loads(capsys.readouterr().out)[0]["name"] == "TR-AU01"
```

- [ ] **Step 2: Run the tests to see them fail**

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: failures — `swe.config` and the `config`/`db` commands do not exist.

- [ ] **Step 3: Implement**

`swe/src/swe/config.py`:

```python
"""swe settings, stored beside the bridge's own files in %LOCALAPPDATA%\\AddLife\\swe-bridge\\swe.json."""
from __future__ import annotations

import json
from pathlib import Path

from .errors import ConfigError
from .session import bridge_dir

DEFAULTS = {
    "work_dir": None,          # scratch folder; default <current directory>\work
    "backup_dir": None,        # decision D-C: off this PC
    "retention_days": 30,
    "selftest_symbol": "TR-AU01",
    "template": "ADDLIFE Template",
}
INTEGER_KEYS = {"retention_days"}


def config_path() -> Path:
    return bridge_dir() / "swe.json"


def _stored() -> dict:
    path = config_path()
    return json.loads(path.read_text(encoding="utf-8-sig")) if path.exists() else {}


def load_config() -> dict:
    return {**DEFAULTS, **_stored()}


def save_setting(key: str, value: str) -> dict:
    if key not in DEFAULTS:
        raise ConfigError(f"unknown setting '{key}'; known: {', '.join(DEFAULTS)}")
    stored = _stored()
    stored[key] = int(value) if key in INTEGER_KEYS else value
    config_path().parent.mkdir(parents=True, exist_ok=True)
    config_path().write_text(json.dumps(stored, indent=2, ensure_ascii=False), encoding="utf-8")
    return stored


def work_dir(config: dict) -> Path:
    return Path(config["work_dir"]) if config.get("work_dir") else Path.cwd() / "work"


def require(config: dict, key: str, reason: str):
    value = config.get(key)
    if value in (None, ""):
        raise ConfigError(f"'{key}' is not set ({reason}). Set it with: swe config set {key} <value>")
    return value
```

`swe/src/swe/commands/config.py`:

```python
from ..config import load_config, save_setting
from ..jsonout import print_json


def register(subparsers) -> None:
    parser = subparsers.add_parser("config", help="Show or change swe settings.")
    actions = parser.add_subparsers(dest="action", required=True)
    actions.add_parser("show").set_defaults(handler=show)
    setter = actions.add_parser("set")
    setter.add_argument("key")
    setter.add_argument("value")
    setter.set_defaults(handler=set_value)


def show(args) -> int:
    print_json(load_config())
    return 0


def set_value(args) -> int:
    save_setting(args.key, args.value)
    print_json(load_config())
    return 0
```

`swe/src/swe/commands/db.py`:

```python
import json
from pathlib import Path

from ..db import queries
from ..db.connection import Db
from ..jsonout import print_json


def make_db() -> Db:
    return Db()


def register(subparsers) -> None:
    parser = subparsers.add_parser("db", help="Read-only queries on Electrical's database.")
    actions = parser.add_subparsers(dest="action", required=True)
    actions.add_parser("projects", help="All projects with schema version and modification date.").set_defaults(handler=projects)
    for name, handler, help_text in (("sheets", sheets, "Sheets of a project."),
                                     ("components", components, "Components of a project."),
                                     ("netlist", netlist, "Wires as from-tag:terminal / to-tag:terminal.")):
        sub = actions.add_parser(name, help=help_text)
        sub.add_argument("project", help="project name or id")
        if name == "netlist":
            sub.add_argument("--save", help="also write the netlist to this JSON file")
        sub.set_defaults(handler=handler)
    diff = actions.add_parser("netlist-diff", help="Compare two saved netlists by connection.")
    diff.add_argument("before")
    diff.add_argument("after")
    diff.set_defaults(handler=netlist_diff)
    search = actions.add_parser("search-symbols", help="Find library symbols by name or description.")
    search.add_argument("text")
    search.add_argument("--limit", type=int, default=50)
    search.set_defaults(handler=search_symbols)


def projects(args) -> int:
    print_json(queries.projects(make_db()))
    return 0


def sheets(args) -> int:
    print_json(queries.sheets(make_db(), args.project))
    return 0


def components(args) -> int:
    print_json(queries.components(make_db(), args.project))
    return 0


def netlist(args) -> int:
    rows = queries.netlist(make_db(), args.project)
    if args.save:
        Path(args.save).write_text(json.dumps(rows, indent=2, ensure_ascii=False, default=str), encoding="utf-8")
    print_json(rows)
    return 0


def netlist_diff(args) -> int:
    def load(path):
        return json.loads(Path(path).read_text(encoding="utf-8"))
    print_json(queries.netlist_diff(load(args.before), load(args.after)))
    return 0


def search_symbols(args) -> int:
    print_json(queries.search_symbols(make_db(), args.text, args.limit))
    return 0
```

Modify `swe/src/swe/cli.py` — `COMMAND_MODULES` becomes:

```python
COMMAND_MODULES = [
    "swe.commands.health",
    "swe.commands.batch",
    "swe.commands.config",
    "swe.commands.db",
]
```

- [ ] **Step 4: Run the tests to see them pass**

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: all pass.

- [ ] **Step 5: Live check**

Run: `.venv\Scripts\swe db projects` then `.venv\Scripts\swe db search-symbols "magneto-thermal" --limit 5`
Expected: the 13 projects; five circuit-breaker symbols including `TR-DI003`.

- [ ] **Step 6: Commit**

```powershell
git add swe
git commit -m "feat(swe): config and read-only db commands"
```

### Task 1.3: Selftest framework and session operations

**Files:**
- Create: `swe/src/swe/selftest.py`, `swe/src/swe/commands/selftest.py`
- Modify: `swe/src/swe/cli.py` (`COMMAND_MODULES`)
- Create: `addin/src/AddLife.SweBridge/Electrical/Ops/SessionOps.cs`
- Modify: `addin/src/AddLife.SweBridge/Electrical/Operations.cs`
- Test: `swe/tests/test_selftest.py`

**Interfaces:**
- Consumes: `BridgeClient`, `require_ok` (0.6); `Db`, `queries` (1.1); `load_config`, `work_dir` (1.2)
- Produces:
  - `swe.selftest`: `PROJECT = "LIBTEST"`, `PINNED_EWAPI = "2026.4.1.1011"`, `Run(client, db, config, out_dir)` with `.batch(*ops, project=..., batch_id=None, stop_on_error=True)`, `.ok(*ops, **kw) -> list` (results of a batch that must fully succeed), `.state: dict`; `op(name, **args) -> dict`; `@step(name)` and `@cleanup` decorators; `STEPS`, `CLEANUP`; `run_all(run, only=None, keep=False) -> list[StepResult]`; `expect(condition, message)`
  - command `swe selftest [--only TEXT] [--keep]`
  - operations `open_project {name}`, `create_project_from_template {name, template, description?}`, `delete_project {name, id?}`, `set_project_properties {customer_name?, customer_address1..3?, drawing_office_name?, drawing_office_address1..3?, contract_number?, description?, user_data?}`, `snapshot {name, description?}`

- [ ] **Step 1: Write the failing tests for the framework**

`swe/tests/test_selftest.py`:

```python
from pathlib import Path

import swe.selftest as st


def isolated(monkeypatch, steps, cleanups=()):
    monkeypatch.setattr(st, "STEPS", list(steps))
    monkeypatch.setattr(st, "CLEANUP", list(cleanups))


def fake_run(tmp_path):
    return st.Run(client=None, db=None, config={}, out_dir=Path(tmp_path))


def test_stops_at_first_failure_but_always_cleans_up(monkeypatch, tmp_path):
    calls = []
    isolated(monkeypatch,
             [("one", lambda r: calls.append("one")), ("two", lambda r: st.expect(False, "broken")),
              ("three", lambda r: calls.append("three"))],
             [("tidy", lambda r: calls.append("tidy"))])
    results = st.run_all(fake_run(tmp_path))
    assert [(r.name, r.ok) for r in results] == [("one", True), ("two", False), ("cleanup: tidy", True)]
    assert results[1].detail == "Check: broken"
    assert calls == ["one", "tidy"]


def test_keep_skips_cleanup(monkeypatch, tmp_path):
    calls = []
    isolated(monkeypatch, [("one", lambda r: None)], [("tidy", lambda r: calls.append("tidy"))])
    st.run_all(fake_run(tmp_path), keep=True)
    assert calls == []


def test_only_filters_steps(monkeypatch, tmp_path):
    calls = []
    isolated(monkeypatch, [("alpha", lambda r: calls.append("a")), ("beta", lambda r: calls.append("b"))])
    st.run_all(fake_run(tmp_path), only="bet")
    assert calls == ["b"]


def test_op_builds_an_operation():
    assert st.op("snapshot", name="x") == {"op": "snapshot", "args": {"name": "x"}}
```

- [ ] **Step 2: Run them to see them fail**

Run: `.venv\Scripts\python -m pytest swe\tests\test_selftest.py -q`
Expected: `ModuleNotFoundError: No module named 'swe.selftest'`.

- [ ] **Step 3: Write the selftest framework with the session steps**

`swe/src/swe/selftest.py`:

```python
"""swe selftest — drives every bridge operation against the scratch project LIBTEST and checks each result
through the API and through SQL. Each Phase 1 task appends its steps below, in order; later steps rely on
what earlier ones created (run.state). Cleanup steps always run unless --keep is given."""
from __future__ import annotations

import datetime
import time
from dataclasses import dataclass, field
from pathlib import Path

from .client import BridgeClient, require_ok
from .db import queries

PROJECT = "LIBTEST"
PINNED_EWAPI = "2026.4.1.1011"
TEXT = "Selftest Ω µ → °"


class Check(AssertionError):
    pass


def expect(condition, message: str) -> None:
    if not condition:
        raise Check(message)


def op(name: str, **args) -> dict:
    return {"op": name, "args": args}


@dataclass
class Run:
    client: BridgeClient
    db: object
    config: dict
    out_dir: Path
    stamp: str = field(default_factory=lambda: datetime.datetime.now().strftime("%Y%m%d%H%M%S"))
    state: dict = field(default_factory=dict)

    @property
    def session(self) -> str:
        return f"selftest-{self.stamp}"

    def batch(self, *ops, project: str | None = PROJECT, batch_id: str | None = None, stop_on_error: bool = True) -> dict:
        return self.client.batch(list(ops), project=project, session=self.session, batch_id=batch_id,
                                 stop_on_error=stop_on_error)

    def ok(self, *ops, **kw) -> list:
        return [r.get("result") for r in require_ok(self.batch(*ops, **kw))["results"]]


@dataclass
class StepResult:
    name: str
    ok: bool
    detail: str
    seconds: float


STEPS: list = []
CLEANUP: list = []


def step(name: str):
    def register(fn):
        STEPS.append((name, fn))
        return fn
    return register


def cleanup(name: str):
    def register(fn):
        CLEANUP.append((name, fn))
        return fn
    return register


def _timed(name: str, fn, run: Run) -> StepResult:
    start = time.monotonic()
    try:
        detail = fn(run) or ""
        return StepResult(name, True, str(detail), time.monotonic() - start)
    except Exception as e:  # a failed step is a result, not a crash
        return StepResult(name, False, f"{type(e).__name__}: {e}", time.monotonic() - start)


def run_all(run: Run, only: str | None = None, keep: bool = False) -> list[StepResult]:
    results = []
    for name, fn in STEPS:
        if only and only not in name:
            continue
        results.append(_timed(name, fn, run))
        if not results[-1].ok:
            break
    if not keep:
        for name, fn in CLEANUP:
            results.append(_timed("cleanup: " + name, fn, run))
    return results


# ---------------------------------------------------------------- Task 1.3: session

@step("bridge answers and is the pinned EwAPI")
def _bridge(run: Run):
    health = run.client.health()
    expect(health.get("api_dll_version") == PINNED_EWAPI,
           f"api_dll_version is {health.get('api_dll_version')}, expected {PINNED_EWAPI} (spec §12 upgrade routine)")
    return f"bridge {health.get('bridge_version')}, current project {health.get('current_project')}"


@step("database schema guard passes")
def _schema(run: Run):
    return f"{len(queries.projects(run.db))} projects readable"


@step("LIBTEST exists and is the current project")
def _libtest(run: Run):
    names = [p["name"] for p in queries.projects(run.db)]
    if PROJECT in names:
        run.ok(op("open_project", name=PROJECT), project=None)
    else:
        run.ok(op("create_project_from_template", name=PROJECT, template=run.config["template"],
                  description="Scratch project for swe selftest"), project=None)
    current = run.ok(op("current_project"), project=None)[0]
    expect(current and current["name"] == PROJECT, f"current project is {current}")
    return f"id {current['id']}"


@step("project properties are written (and a snapshot is taken first)")
def _properties(run: Run):
    response = require_ok(run.batch(op("set_project_properties", customer_name=TEXT, contract_number=run.stamp,
                                       user_data={"1": run.stamp})))
    expect(response["snapshot"], "no snapshot was taken before the first change")
    run.state["snapshot"] = response["snapshot"]
    return f"snapshot {response['snapshot']}"


@step("an explicit snapshot can be taken")
def _snapshot(run: Run):
    return run.ok(op("snapshot", name=f"selftest-{run.stamp}"))[0]


@step("a project outside the allowlist is refused before anything runs")
def _allowlist(run: Run):
    response = run.batch(op("set_project_properties", customer_name="must not be written"), project="Addlife Template")
    expect(response["refusal"] and response["refusal"]["code"] == "project_not_allowlisted", f"got {response}")


@step("a batch for a different project than the open one is refused")
def _mismatch(run: Run):
    response = run.batch(op("set_project_properties", customer_name="must not be written"), project="LIBTEST-other")
    result = response["results"][0]
    expect(result["status"] == "error" and result["error"]["code"] == "project_mismatch", f"got {result}")
```

`swe/src/swe/commands/selftest.py`:

```python
import sys

from ..client import BridgeClient
from ..config import load_config, work_dir
from ..db.connection import Db
from ..selftest import Run, run_all


def register(subparsers) -> None:
    parser = subparsers.add_parser("selftest", help="Exercise every bridge operation on the scratch project LIBTEST.")
    parser.add_argument("--only", help="run only steps whose name contains this text")
    parser.add_argument("--keep", action="store_true", help="skip clean-up, to inspect LIBTEST afterwards")
    parser.set_defaults(handler=run)


def run(args) -> int:
    config = load_config()
    out_dir = work_dir(config) / "selftest"
    out_dir.mkdir(parents=True, exist_ok=True)
    results = run_all(Run(client=BridgeClient.connect(), db=Db(), config=config, out_dir=out_dir),
                      only=args.only, keep=args.keep)
    for r in results:
        print(f"{'PASS' if r.ok else 'FAIL'}  {r.seconds:6.1f}s  {r.name}" + (f"  — {r.detail}" if r.detail else ""))
    failed = [r for r in results if not r.ok]
    print(f"\n{len(results) - len(failed)} passed, {len(failed)} failed", file=sys.stderr)
    return 0 if not failed else 1
```

Modify `swe/src/swe/cli.py` — append `"swe.commands.selftest",` to `COMMAND_MODULES`.

- [ ] **Step 4: Run the unit tests (pass) and the live selftest (fails: operations missing)**

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: all pass.

Run: `.venv\Scripts\swe selftest`
Expected: `PASS bridge answers…`, `PASS database schema guard…`, then `FAIL LIBTEST exists…` with `unknown_op: unknown operation 'create_project_from_template'` (or `open_project`).

- [ ] **Step 5: Implement the session operations**

`addin/src/AddLife.SweBridge/Electrical/Ops/SessionOps.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using AddLife.SweBridge.Core;
using EwAPI;

namespace AddLife.SweBridge.Electrical.Ops
{
    public sealed class OpenProjectOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("open_project",
            "Open an existing project by exact name and make it current.", false, false,
            Req("name", "string", "exact project name"));

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.FindProject(a.Str("name"));
            Ew.Check(ctx.App.openEwProjectID(project.getID()), "openEwProjectID");
            return ReadModel.ProjectInfo(ctx.CurrentProject());
        }
    }

    public sealed class CreateProjectFromTemplateOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("create_project_from_template",
            "Create a new project from a project template and open it.", true, false,
            Req("name", "string", "new project name; must be in mutation_allowlist"),
            Req("template", "string", "template name as Electrical lists it, e.g. 'ADDLIFE Template'"),
            Opt("description", "string", "project description"));

        public override string TargetProject(Args args, string batchProject) => args.Str("name");

        protected override object Run(EwContext ctx, Args a)
        {
            var name = a.Str("name");
            var existing = ctx.AllProjects().FirstOrDefault(p => string.Equals(p.getName(out _), name, StringComparison.Ordinal));
            if (existing != null) throw new OpException("already_exists", $"a project named '{name}' already exists (id {existing.getID()})");

            var project = ctx.Projects().newEwProject(out var err);
            Ew.Check(err, "newEwProject");
            Ew.Check(project.setName(name), "setName");
            if (a.Has("description")) Ew.Check(project.setDescription(ctx.Lang, a.Str("description")), "setDescription");
            Ew.Check(project.insertFromTemplate(a.Str("template")), "insertFromTemplate");
            Ew.Check(ctx.App.openEwProjectID(project.getID()), "openEwProjectID");
            return ReadModel.ProjectInfo(ctx.CurrentProject());
        }
    }

    public sealed class DeleteProjectOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("delete_project",
            "Delete a project, closing it first if it is open. Allowlisted projects only.", true, false,
            Req("name", "string", "exact project name; must be in mutation_allowlist"),
            Opt("id", "int", "required when several projects share the name"));

        public override string TargetProject(Args args, string batchProject) => args.Str("name");

        protected override object Run(EwContext ctx, Args a)
        {
            var name = a.Str("name");
            var matches = ctx.AllProjects().Where(p => string.Equals(p.getName(out _), name, StringComparison.Ordinal)).ToList();
            if (a.Has("id")) matches = matches.Where(p => p.getID() == a.Int("id")).ToList();
            if (matches.Count == 0) throw new OpException("not_found", $"no project named '{name}'" + (a.Has("id") ? $" with id {a.Int("id")}" : ""));
            if (matches.Count > 1)
                throw new OpException("ambiguous_identity",
                    $"{matches.Count} projects are named '{name}' (ids {string.Join(", ", matches.Select(p => p.getID()))}); pass id");

            var project = matches[0];
            var id = project.getID();
            if (ctx.App.isEwProjectOpened(id, out _)) Ew.Check(ctx.App.closeEwProjectID(id), "closeEwProjectID");
            Ew.Check(project.remove(), "remove");
            return new Dictionary<string, object> { ["deleted"] = id, ["name"] = name };
        }
    }

    public sealed class SetProjectPropertiesOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("set_project_properties",
            "Set project properties that title blocks show. Only the given ones change.", true, true,
            Opt("customer_name", "string", ""), Opt("customer_address1", "string", ""),
            Opt("customer_address2", "string", ""), Opt("customer_address3", "string", ""),
            Opt("drawing_office_name", "string", ""), Opt("drawing_office_address1", "string", ""),
            Opt("drawing_office_address2", "string", ""), Opt("drawing_office_address3", "string", ""),
            Opt("contract_number", "string", ""), Opt("description", "string", "in the configured language"),
            Opt("user_data", "object", "{\"1\": \"value\", ...} — project user data fields by number"));

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.CurrentProject();
            var updated = new List<object>();

            void Set(string key, Func<string, EwErrorCode> setter)
            {
                if (!a.Has(key)) return;
                Ew.Check(setter(a.Str(key)), key);
                updated.Add(key);
            }

            Set("customer_name", project.setCustomerName);
            Set("customer_address1", project.setCustomerAddress1);
            Set("customer_address2", project.setCustomerAddress2);
            Set("customer_address3", project.setCustomerAddress3);
            Set("drawing_office_name", project.setDrawingOfficeName);
            Set("drawing_office_address1", project.setDrawingOfficeAddress1);
            Set("drawing_office_address2", project.setDrawingOfficeAddress2);
            Set("drawing_office_address3", project.setDrawingOfficeAddress3);
            Set("contract_number", project.setContractNumber);
            Set("description", v => project.setDescription(ctx.Lang, v));

            var userData = a.OptObj("user_data");
            if (userData != null)
                foreach (var entry in userData)
                {
                    if (!int.TryParse(entry.Key, out var number) || number < 1)
                        throw new OpException("bad_args", $"user_data key '{entry.Key}' must be a field number from 1");
                    if (!(entry.Value is string value)) throw new OpException("bad_args", $"user_data '{entry.Key}' must be a string");
                    Ew.Check(project.setUserData(number, value), "setUserData " + number);
                    updated.Add("user_data." + number);
                }

            Ew.Check(project.update(), "update");
            return new Dictionary<string, object> { ["updated"] = updated };
        }
    }

    public sealed class SnapshotOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("snapshot",
            "Take a named Electrical snapshot of the current project.", false, true,
            Req("name", "string", "snapshot name"), Opt("description", "string", ""));

        protected override object Run(EwContext ctx, Args a)
        {
            var manager = ctx.CurrentProject().getEwProjectSnapshotManager(out var err);
            Ew.Check(err, "getEwProjectSnapshotManager");
            var snapshot = manager.newEwProjectSnapshot(out err);
            Ew.Check(err, "newEwProjectSnapshot");
            Ew.Check(snapshot.setName(a.Str("name")), "setName");
            if (a.Has("description")) Ew.Check(snapshot.setDescription(ctx.Lang, a.Str("description")), "setDescription");
            Ew.Check(snapshot.create(), "create");
            return new Dictionary<string, object> { ["id"] = snapshot.getID(), ["name"] = a.Str("name") };
        }
    }
}
```

Modify `addin/src/AddLife.SweBridge/Electrical/Operations.cs` — after `registry.Add(new CurrentProjectOp());` add:

```csharp
            registry.Add(new OpenProjectOp());
            registry.Add(new CreateProjectFromTemplateOp());
            registry.Add(new DeleteProjectOp());
            registry.Add(new SetProjectPropertiesOp());
            registry.Add(new SnapshotOp());
```

- [ ] **Step 6: Build, deploy, run the selftest**

```powershell
dotnet test addin\tests\AddLife.SweBridge.Tests -c Release
# [user] close Electrical
pwsh addin\deploy.ps1
# [user] start Electrical
.venv\Scripts\swe selftest
```
Expected: every step PASS — `LIBTEST` is created the first time (and opened on later runs), the snapshot name is printed, both refusals pass. Open Electrical's project properties for LIBTEST and check the customer name reads `Selftest Ω µ → °`.

- [ ] **Step 7: Commit**

```powershell
git add addin swe
git commit -m "feat: selftest framework and session operations"
```

### Task 1.4: Structure operations — books, sheets, locations, functions, components, delete

**Files:**
- Create: `addin/src/AddLife.SweBridge/Core/Identity.cs`
- Create: `addin/src/AddLife.SweBridge/Electrical/Ops/StructureOps.cs`
- Modify: `addin/src/AddLife.SweBridge/Electrical/EwContext.cs` (five lookups), `ReadModel.cs` (`Sheet`), `Operations.cs`
- Modify: `swe/src/swe/selftest.py` (append steps and the clean-up)
- Test: `addin/tests/AddLife.SweBridge.Tests/IdentityTests.cs`

**Interfaces:**
- Consumes: `EwContext`, `Ew`, `ReadModel` (0.5); selftest `step`, `cleanup`, `Run`, `op`, `expect` (1.3)
- Produces:
  - `Identity.Single<T>(IEnumerable<T> items, Func<T, string> key, string wanted, string kind, Func<T, int> id) : T` — `null` when absent, `OpException("ambiguous_identity")` naming every id when duplicated
  - `EwContext.Books(p)`, `.Sheets(p)`, `.Locations(p)`, `.Functions(p)`, `.Components(p)`; `ReadModel.Sheet(sheet, lang, created?)`
  - operations `create_book {tag, description}`, `create_sheet {book_tag, position, description, type?}`, `create_location {tag, description}`, `create_function {tag, description}`, `upsert_component {tag, description?, location_tag?, function_tag?, manufacturer?, reference?}`, `delete {kind, id?, tag?}` — creates return `{id, tag|…, created}`
  - selftest `run.state["sheet_id"]`

- [ ] **Step 1: Write the failing identity tests**

`addin/tests/AddLife.SweBridge.Tests/IdentityTests.cs`:

```csharp
using AddLife.SweBridge.Core;
using Xunit;

public class IdentityTests
{
    sealed class Item { public int Id; public string Tag; }

    static readonly Item[] Items = { new Item { Id = 1, Tag = "-K1" }, new Item { Id = 2, Tag = "-K2" }, new Item { Id = 3, Tag = "-K2" } };

    static Item Find(string tag) => Identity.Single(Items, i => i.Tag, tag, "component", i => i.Id);

    [Fact]
    public void Finds_the_single_match() => Assert.Equal(1, Find("-K1").Id);

    [Fact]
    public void Absent_is_null() => Assert.Null(Find("-K9"));

    [Fact]
    public void Tags_are_compared_exactly() => Assert.Null(Find("-k1"));

    [Fact]
    public void Duplicates_are_refused_naming_every_id()
    {
        var e = Assert.Throws<OpException>(() => Find("-K2"));
        Assert.Equal("ambiguous_identity", e.Code);
        Assert.Contains("2, 3", e.Message);
        Assert.Contains("-K2", e.Message);
    }
}
```

- [ ] **Step 2: Run them to see them fail**

Run: `dotnet test addin\tests\AddLife.SweBridge.Tests -c Release`
Expected: build error — `Identity` does not exist.

- [ ] **Step 3: Implement `Identity`**

`addin/src/AddLife.SweBridge/Core/Identity.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace AddLife.SweBridge.Core
{
    /// <summary>
    /// Finds an object by its natural key. Duplicates (e.g. two components tagged -K1 made by hand) are refused
    /// rather than resolved by picking one, so an update can never land on the wrong object.
    /// </summary>
    public static class Identity
    {
        public static T Single<T>(IEnumerable<T> items, Func<T, string> key, string wanted, string kind, Func<T, int> id)
            where T : class
        {
            var hits = items.Where(i => string.Equals(key(i), wanted, StringComparison.Ordinal)).ToList();
            if (hits.Count > 1)
                throw new OpException("ambiguous_identity",
                    $"{hits.Count} {kind}s have the tag '{wanted}' (ids {string.Join(", ", hits.Select(id))}); " +
                    "resolve the duplicate in Electrical first");
            return hits.Count == 1 ? hits[0] : null;
        }
    }
}
```

Run: `dotnet test addin\tests\AddLife.SweBridge.Tests -c Release`
Expected: `IdentityTests` pass.

- [ ] **Step 4: Append the selftest steps (they fail until the operations exist)**

Append to `swe/src/swe/selftest.py`:

```python


# ---------------------------------------------------------------- Task 1.4: structure

@step("book, sheet, location and function are created")
def _structure(run: Run):
    book, sheet, location, function = run.ok(
        op("create_book", tag="ST", description="Selftest book"),
        op("create_sheet", book_tag="ST", position=1, description=TEXT),
        op("create_location", tag="STLOC", description="Selftest location"),
        op("create_function", tag="STFUN", description="Selftest function"))
    run.state["sheet_id"] = sheet["id"]
    return f"book {book['id']}, sheet {sheet['id']}, location {location['id']}, function {function['id']}"


@step("upsert_component creates once, then updates")
def _component(run: Run):
    first, second = run.ok(
        op("upsert_component", tag="-STK1", description=TEXT, location_tag="STLOC", function_tag="STFUN"),
        op("upsert_component", tag="-STK1", description=TEXT))
    expect(first["created"] and not second["created"], f"created flags {first['created']}, {second['created']}")
    expect(first["id"] == second["id"], "the second upsert made a new component")
    matching = [c for c in queries.components(run.db, PROJECT) if c["tag"] == "-STK1"]
    expect(len(matching) == 1, f"SQL sees {len(matching)} components tagged -STK1")


@step("a repeated batch_id is answered from cache, not re-run")
def _idempotent(run: Run):
    batch_id = f"st{run.stamp}"
    first = run.batch(op("upsert_component", tag="-STK2", description="second"), batch_id=batch_id)
    second = run.batch(op("upsert_component", tag="-STK2", description="second"), batch_id=batch_id)
    expect(not first["cached"] and second["cached"], f"cached flags {first['cached']}, {second['cached']}")
    expect(first["results"][0]["result"]["created"], "-STK2 was not created by the first send")


@cleanup("delete everything the selftest created")
def _tidy(run: Run):
    ops = []
    if "sheet_id" in run.state:
        ops.append(op("delete", kind="sheet", id=run.state["sheet_id"]))
    ops += [op("delete", kind="component", tag="-STK1"), op("delete", kind="component", tag="-STK2"),
            op("delete", kind="location", tag="STLOC"), op("delete", kind="function", tag="STFUN"),
            op("delete", kind="book", tag="ST")]
    response = run.batch(*ops, stop_on_error=False)
    leftovers = [r for r in response["results"] if r["status"] == "error" and r["error"]["code"] != "not_found"]
    expect(not leftovers, f"clean-up failed: {leftovers}")
```

- [ ] **Step 5: Add the lookups and the sheet read model**

Modify `addin/src/AddLife.SweBridge/Electrical/EwContext.cs` — add these methods inside the class:

```csharp
        public EwProjectBookX[] Books(EwProjectX project)
        {
            var manager = project.getEwProjectBookManager(out var err);
            Ew.Check(err, "getEwProjectBookManager");
            var array = manager.getEwProjectBookArray(out err);
            Ew.Check(err, "getEwProjectBookArray");
            return Ew.Array<EwProjectBookX>(array);
        }

        public EwProjectFileX[] Sheets(EwProjectX project)
        {
            var manager = project.getEwProjectFileManager(out var err);
            Ew.Check(err, "getEwProjectFileManager");
            var array = manager.getEwProjectFileArray(out err);
            Ew.Check(err, "getEwProjectFileArray");
            return Ew.Array<EwProjectFileX>(array);
        }

        public EwProjectLocationX[] Locations(EwProjectX project)
        {
            var manager = project.getEwProjectLocationManager(out var err);
            Ew.Check(err, "getEwProjectLocationManager");
            var array = manager.getEwProjectLocationArray(out err);
            Ew.Check(err, "getEwProjectLocationArray");
            return Ew.Array<EwProjectLocationX>(array);
        }

        public EwProjectFunctionX[] Functions(EwProjectX project)
        {
            var manager = project.getEwProjectFunctionManager(out var err);
            Ew.Check(err, "getEwProjectFunctionManager");
            var array = manager.getEwProjectFunctionArray(out err);
            Ew.Check(err, "getEwProjectFunctionArray");
            return Ew.Array<EwProjectFunctionX>(array);
        }

        public EwProjectComponentX[] Components(EwProjectX project)
        {
            var manager = project.getEwProjectComponentManager(out var err);
            Ew.Check(err, "getEwProjectComponentManager");
            var array = manager.getEwProjectComponentArray(out err);
            Ew.Check(err, "getEwProjectComponentArray");
            return Ew.Array<EwProjectComponentX>(array);
        }
```

Modify `addin/src/AddLife.SweBridge/Electrical/ReadModel.cs` — add inside the class:

```csharp
        public static Dictionary<string, object> Sheet(EwProjectFileX sheet, string lang, bool? created = null)
        {
            var json = new Dictionary<string, object>
            {
                ["id"] = sheet.getID(),
                ["type"] = sheet.getFileType().ToString(),
                ["book_id"] = sheet.getEwProjectBookID(out _),
                ["folder_id"] = sheet.getEwProjectFolderID(out _),
                ["position"] = sheet.getPosition(out _),
                ["description"] = sheet.getDescription(lang, out _),
                ["file_path"] = sheet.getFilePath(out _),
            };
            if (created.HasValue) json["created"] = created.Value;
            return json;
        }
```

- [ ] **Step 6: Implement the structure operations**

`addin/src/AddLife.SweBridge/Electrical/Ops/StructureOps.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;
using AddLife.SweBridge.Core;
using EwAPI;

namespace AddLife.SweBridge.Electrical.Ops
{
    public sealed class CreateBookOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("create_book",
            "Create a book with a manual tag, or update its description if the tag exists.", true, true,
            Req("tag", "string", "book tag"), Req("description", "string", "book description"));

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.CurrentProject();
            var tag = a.Str("tag");
            var book = Identity.Single(ctx.Books(project), b => b.getTag(out _), tag, "book", b => b.getID());
            var created = book == null;
            if (created)
            {
                var manager = project.getEwProjectBookManager(out var err);
                Ew.Check(err, "getEwProjectBookManager");
                book = manager.newEwProjectBook(out err);
                Ew.Check(err, "newEwProjectBook");
                Ew.Check(book.setTagMode(EwTagMode.kManu), "setTagMode");
                Ew.Check(book.setTag(tag), "setTag");
                Ew.Check(book.setDescription(ctx.Lang, a.Str("description")), "setDescription");
                Ew.Check(book.insert(), "insert");
            }
            else
            {
                Ew.Check(book.setDescription(ctx.Lang, a.Str("description")), "setDescription");
                Ew.Check(book.update(), "update");
            }
            return new Dictionary<string, object> { ["id"] = book.getID(), ["tag"] = tag, ["created"] = created };
        }
    }

    public sealed class CreateSheetOp : EwOperation
    {
        static readonly Dictionary<string, EwFileType> Types = new Dictionary<string, EwFileType>
        {
            ["folio"] = EwFileType.kFileFolio,
            ["cover"] = EwFileType.kFileCoverPage,
            ["line_diagram"] = EwFileType.kFileLineDiagram,
            ["other"] = EwFileType.kFileOther,
        };

        public override OpSpec Spec { get; } = new OpSpec("create_sheet",
            "Create a sheet in a book at a position, or update the description of the sheet already there.", true, true,
            Req("book_tag", "string", "tag of an existing book"),
            Req("position", "int", "1-based position within the book"),
            Req("description", "string", "sheet title, shown in the title block and the contents"),
            Opt("type", "string", "folio (default) | cover | line_diagram | other"));

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.CurrentProject();
            var bookTag = a.Str("book_tag");
            var book = Identity.Single(ctx.Books(project), b => b.getTag(out _), bookTag, "book", b => b.getID())
                       ?? throw new OpException("not_found", $"no book tagged '{bookTag}'");
            var typeName = a.OptStr("type", "folio");
            if (!Types.TryGetValue(typeName, out var type))
                throw new OpException("bad_args", "type must be one of " + string.Join(", ", Types.Keys));
            var position = a.Int("position");
            if (position < 1) throw new OpException("bad_args", "position starts at 1");

            var bookId = book.getID();
            var there = ctx.Sheets(project)
                .Where(s => s.getEwProjectBookID(out _) == bookId && s.getPosition(out _) == position).ToList();
            if (there.Count > 1)
                throw new OpException("ambiguous_identity", $"{there.Count} sheets sit at position {position} of book '{bookTag}'");
            if (there.Count == 1)
            {
                Ew.Check(there[0].setDescription(ctx.Lang, a.Str("description")), "setDescription");
                Ew.Check(there[0].update(), "update");
                return ReadModel.Sheet(there[0], ctx.Lang, false);
            }

            var manager = project.getEwProjectFileManager(out var err);
            Ew.Check(err, "getEwProjectFileManager");
            var sheet = manager.newProjectFile(out err);
            Ew.Check(err, "newProjectFile");
            Ew.Check(sheet.setFileType(type), "setFileType");
            Ew.Check(sheet.setEwProjectBookID(bookId), "setEwProjectBookID");
            Ew.Check(sheet.setDescription(ctx.Lang, a.Str("description")), "setDescription");
            Ew.Check(sheet.insert(), "insert");
            Ew.Check(sheet.setPosition(position), "setPosition");
            Ew.Check(sheet.update(), "update");
            return ReadModel.Sheet(sheet, ctx.Lang, true);
        }
    }

    public sealed class CreateLocationOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("create_location",
            "Create a location (e.g. CAB for +CAB) with a manual tag, or update its description.", true, true,
            Req("tag", "string", "location tag without the + prefix"), Req("description", "string", ""));

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.CurrentProject();
            var tag = a.Str("tag");
            var location = Identity.Single(ctx.Locations(project), l => l.getTag(out _), tag, "location", l => l.getID());
            var created = location == null;
            if (created)
            {
                var manager = project.getEwProjectLocationManager(out var err);
                Ew.Check(err, "getEwProjectLocationManager");
                location = manager.newEwProjectLocation();
                Ew.Check(location.setTagMode(EwTagMode.kManu), "setTagMode");
                Ew.Check(location.setTag(tag), "setTag");
                Ew.Check(location.setDescription(ctx.Lang, a.Str("description")), "setDescription");
                Ew.Check(location.insert(), "insert");
            }
            else
            {
                Ew.Check(location.setDescription(ctx.Lang, a.Str("description")), "setDescription");
                Ew.Check(location.update(), "update");
            }
            return new Dictionary<string, object> { ["id"] = location.getID(), ["tag"] = tag, ["created"] = created };
        }
    }

    public sealed class CreateFunctionOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("create_function",
            "Create a function (e.g. WB for =WB) with a manual tag, or update its description.", true, true,
            Req("tag", "string", "function tag without the = prefix"), Req("description", "string", ""));

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.CurrentProject();
            var tag = a.Str("tag");
            var function = Identity.Single(ctx.Functions(project), f => f.getTag(out _), tag, "function", f => f.getID());
            var created = function == null;
            if (created)
            {
                var manager = project.getEwProjectFunctionManager(out var err);
                Ew.Check(err, "getEwProjectFunctionManager");
                function = manager.newEwProjectFunction(out err);
                Ew.Check(err, "newEwProjectFunction");
                Ew.Check(function.setTagMode(EwTagMode.kManu), "setTagMode");
                Ew.Check(function.setTag(tag), "setTag");
                Ew.Check(function.setDescription(ctx.Lang, a.Str("description")), "setDescription");
                Ew.Check(function.insert(), "insert");
            }
            else
            {
                Ew.Check(function.setDescription(ctx.Lang, a.Str("description")), "setDescription");
                Ew.Check(function.update(), "update");
            }
            return new Dictionary<string, object> { ["id"] = function.getID(), ["tag"] = tag, ["created"] = created };
        }
    }

    public sealed class UpsertComponentOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("upsert_component",
            "Create a component with a manual tag, or update the one with that tag.", true, true,
            Req("tag", "string", "component tag, e.g. -13A1"),
            Opt("description", "string", ""),
            Opt("location_tag", "string", "tag of an existing location"),
            Opt("function_tag", "string", "tag of an existing function"),
            Opt("manufacturer", "string", "with reference: assign this catalog part"),
            Opt("reference", "string", "with manufacturer: assign this catalog part"));

        protected override object Run(EwContext ctx, Args a)
        {
            if (a.Has("manufacturer") != a.Has("reference"))
                throw new OpException("bad_args", "manufacturer and reference are given together or not at all");
            var project = ctx.CurrentProject();
            var tag = a.Str("tag");
            var component = Identity.Single(ctx.Components(project), c => c.getTag(out _), tag, "component", c => c.getID());
            var created = component == null;
            if (created)
            {
                var manager = project.getEwProjectComponentManager(out var err);
                Ew.Check(err, "getEwProjectComponentManager");
                component = manager.newEwProjectComponent();
                Ew.Check(component.setTagMode(EwTagMode.kManu), "setTagMode");
                Ew.Check(component.setTag(tag), "setTag");
            }
            if (a.Has("description")) Ew.Check(component.setDescription(ctx.Lang, a.Str("description")), "setDescription");
            if (a.Has("location_tag"))
            {
                var location = Identity.Single(ctx.Locations(project), l => l.getTag(out _), a.Str("location_tag"), "location", l => l.getID())
                               ?? throw new OpException("not_found", $"no location tagged '{a.Str("location_tag")}'");
                Ew.Check(component.setLocationID(location.getID()), "setLocationID");
            }
            if (a.Has("function_tag"))
            {
                var function = Identity.Single(ctx.Functions(project), f => f.getTag(out _), a.Str("function_tag"), "function", f => f.getID())
                               ?? throw new OpException("not_found", $"no function tagged '{a.Str("function_tag")}'");
                Ew.Check(component.setFunctionID(function.getID()), "setFunctionID");
            }
            Ew.Check(created ? component.insert() : component.update(), created ? "insert" : "update");
            if (a.Has("manufacturer"))
                Ew.Check(component.assignManufacturerPart(a.Str("manufacturer"), a.Str("reference")), "assignManufacturerPart");
            return new Dictionary<string, object> { ["id"] = component.getID(), ["tag"] = tag, ["created"] = created };
        }
    }

    public sealed class DeleteOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("delete",
            "Delete one object: symbol, line, text or sheet by id; component, location, function or book by tag.", true, true,
            Req("kind", "string", "symbol | line | text | sheet | component | location | function | book"),
            Opt("id", "int", "for symbol, line, text, sheet"),
            Opt("tag", "string", "for component, location, function, book"));

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.CurrentProject();
            var kind = a.Str("kind");
            EwErrorCode err;
            switch (kind)
            {
                case "symbol":
                {
                    var manager = project.getEwProjectSymbolManager(out err);
                    Ew.Check(err, "getEwProjectSymbolManager");
                    Remove(manager.getProjectSymbolByID(a.Int("id"), out err)?.remove(), kind, a.Int("id"));
                    break;
                }
                case "line":
                {
                    var manager = project.getEwProjectLineManager(out err);
                    Ew.Check(err, "getEwProjectLineManager");
                    Remove(manager.findEwProjectLineByID(a.Int("id"), out err)?.remove(), kind, a.Int("id"));
                    break;
                }
                case "text":
                {
                    var manager = project.getEwProjectMultilingualTextManager(out err);
                    Ew.Check(err, "getEwProjectMultilingualTextManager");
                    Remove(manager.findEwProjectMultilingualTextByID(a.Int("id"), out err)?.remove(), kind, a.Int("id"));
                    break;
                }
                case "sheet":
                    Ew.Check(ctx.Sheet(project, a.Int("id")).remove(), "remove sheet");
                    break;
                case "component":
                    Remove(Identity.Single(ctx.Components(project), c => c.getTag(out _), a.Str("tag"), kind, c => c.getID())?.remove(), kind, a.Str("tag"));
                    break;
                case "location":
                    Remove(Identity.Single(ctx.Locations(project), l => l.getTag(out _), a.Str("tag"), kind, l => l.getID())?.remove(), kind, a.Str("tag"));
                    break;
                case "function":
                    Remove(Identity.Single(ctx.Functions(project), f => f.getTag(out _), a.Str("tag"), kind, f => f.getID())?.remove(), kind, a.Str("tag"));
                    break;
                case "book":
                    Remove(Identity.Single(ctx.Books(project), b => b.getTag(out _), a.Str("tag"), kind, b => b.getID())?.remove(), kind, a.Str("tag"));
                    break;
                default:
                    throw new OpException("bad_args", "kind must be symbol, line, text, sheet, component, location, function or book");
            }
            return new Dictionary<string, object> { ["kind"] = kind, ["deleted"] = a.Has("id") ? (object)a.Int("id") : a.Str("tag") };
        }

        static void Remove(EwErrorCode? removed, string kind, object key)
        {
            if (removed == null) throw new OpException("not_found", $"no {kind} '{key}'");
            Ew.Check(removed.Value, "remove " + kind);
        }
    }
}
```

Modify `addin/src/AddLife.SweBridge/Electrical/Operations.cs` — add after the session operations:

```csharp
            registry.Add(new CreateBookOp());
            registry.Add(new CreateSheetOp());
            registry.Add(new CreateLocationOp());
            registry.Add(new CreateFunctionOp());
            registry.Add(new UpsertComponentOp());
            registry.Add(new DeleteOp());
```

- [ ] **Step 7: Build, deploy, run the selftest**

```powershell
dotnet test addin\tests\AddLife.SweBridge.Tests -c Release
# [user] close Electrical
pwsh addin\deploy.ps1
# [user] start Electrical
.venv\Scripts\swe selftest
```
Expected: all steps PASS, including `cleanup: delete everything the selftest created`. Then `.venv\Scripts\swe selftest --keep` and look at LIBTEST in Electrical: book `ST` with one sheet titled `Selftest Ω µ → °`, components `-STK1` and `-STK2`. Remove them with `.venv\Scripts\swe selftest --only cleanup-only` (no step matches, so only the clean-up runs).

- [ ] **Step 8: Commit**

```powershell
git add addin swe
git commit -m "feat: book, sheet, location, function, component and delete operations"
```

### Task 1.5: Drawing operations — symbols, wires, text

**Files:**
- Create: `addin/src/AddLife.SweBridge/Electrical/Ops/DrawingOps.cs`
- Modify: `addin/src/AddLife.SweBridge/Electrical/ReadModel.cs` (`Symbol`, `Points`, `Line`, `Text`), `Operations.cs`
- Modify: `swe/src/swe/selftest.py` (append steps)

**Interfaces:**
- Consumes: `EwContext.Sheet`, `.Components`, `Identity.Single` (1.4)
- Produces:
  - `ReadModel.Symbol(EwProjectSymbolX)` → `{id, name, x, y, angle, component_id, points:[{n, circuit, terminal, x, y}]}`; `ReadModel.Line(ewProjectLineX)` → `{id, x1, y1, x2, y2, equipotential_id, wire_mark}`; `ReadModel.Text(EwProjectMultilingualTextX, lang)` → `{id, text, x, y, angle}`
  - operations `place_symbol {sheet_id, symbol, x, y, angle?, component_tag?}` → symbol JSON; `draw_wire {sheet_id, x1, y1, x2, y2, wire_style_id?}` → line JSON; `place_text {sheet_id, text, x, y, angle?, align?}` → text JSON
  - selftest `run.state["symbols"]`, `run.state["line_id"]`, `run.state["text_id"]`

- [ ] **Step 1: Append the selftest steps**

Append to `swe/src/swe/selftest.py`:

```python


# ---------------------------------------------------------------- Task 1.5: drawing

@step("two symbols are placed, the second linked to -STK2")
def _symbols(run: Run):
    symbol = run.config["selftest_symbol"]
    first, second = run.ok(
        op("place_symbol", sheet_id=run.state["sheet_id"], symbol=symbol, x=100, y=150, component_tag="-STK1"),
        op("place_symbol", sheet_id=run.state["sheet_id"], symbol=symbol, x=160, y=150, component_tag="-STK2"))
    expect(len(first["points"]) >= 2 and len(second["points"]) >= 2, "a symbol came back without its connection points")
    run.state["symbols"] = [first, second]
    return f"symbols {first['id']}, {second['id']}"


@step("a wire joins the two symbols' connection points")
def _wire(run: Run):
    a = run.state["symbols"][0]["points"][-1]
    b = run.state["symbols"][1]["points"][0]
    line = run.ok(op("draw_wire", sheet_id=run.state["sheet_id"], x1=a["x"], y1=a["y"], x2=b["x"], y2=b["y"]))[0]
    run.state["line_id"] = line["id"]
    return f"line {line['id']}, equipotential {line['equipotential_id']}"


@step("text with non-ASCII characters is placed")
def _text(run: Run):
    text = run.ok(op("place_text", sheet_id=run.state["sheet_id"], text=TEXT, x=40, y=40))[0]
    expect(text["text"] == TEXT, f"Electrical returned {text['text']!r}")
    run.state["text_id"] = text["id"]


@step("a zero-length wire is refused")
def _zero_wire(run: Run):
    response = run.batch(op("draw_wire", sheet_id=run.state["sheet_id"], x1=10, y1=10, x2=10, y2=10))
    result = response["results"][0]
    expect(result["status"] == "error" and result["error"]["code"] == "bad_args", f"got {result}")
```

- [ ] **Step 2: Extend the read model**

Modify `addin/src/AddLife.SweBridge/Electrical/ReadModel.cs` — add `using System.Linq;` at the top and these methods inside the class:

```csharp
        public static List<Dictionary<string, object>> Points(EwProjectSymbolX symbol) =>
            Ew.Array<EwProjectSymbolPointX>(symbol.getEwProjectSymbolPointArray(out _))
                .Select(p =>
                {
                    var position = p.getPointPosition(out _);
                    return new Dictionary<string, object>
                    {
                        ["n"] = p.getPointNumber(out _),
                        ["circuit"] = p.getCircuitNumber(out _),
                        ["terminal"] = p.getTerminalNumber(out _),
                        ["x"] = position.getXCoordinate(),
                        ["y"] = position.getYCoordinate(),
                    };
                }).ToList();

        public static Dictionary<string, object> Symbol(EwProjectSymbolX symbol) => new Dictionary<string, object>
        {
            ["id"] = symbol.getID(),
            ["name"] = symbol.getEwSymbolName(out _),
            ["x"] = symbol.getXPosition(out _),
            ["y"] = symbol.getYPosition(out _),
            ["angle"] = symbol.getRotationAngle(out _),
            ["component_id"] = symbol.getObjectID(out _),
            ["points"] = Points(symbol),
        };

        public static Dictionary<string, object> Line(ewProjectLineX line) => new Dictionary<string, object>
        {
            ["id"] = line.getID(),
            ["x1"] = line.getStartPointXPosition(out _),
            ["y1"] = line.getStartPointYPosition(out _),
            ["x2"] = line.getEndPointXPosition(out _),
            ["y2"] = line.getEndPointYPosition(out _),
            ["equipotential_id"] = line.getEquipotentialID(out _),
            ["wire_mark"] = line.getWireMarkText(out _),
        };

        public static Dictionary<string, object> Text(EwProjectMultilingualTextX text, string lang) => new Dictionary<string, object>
        {
            ["id"] = text.getID(),
            ["text"] = text.getText(lang, out _),
            ["x"] = text.getXPosition(out _),
            ["y"] = text.getYPosition(out _),
            ["angle"] = text.getRotationAngle(out _),
        };
```

- [ ] **Step 3: Implement the drawing operations**

`addin/src/AddLife.SweBridge/Electrical/Ops/DrawingOps.cs`:

```csharp
using System.Collections.Generic;
using AddLife.SweBridge.Core;
using EwAPI;

namespace AddLife.SweBridge.Electrical.Ops
{
    public sealed class PlaceSymbolOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("place_symbol",
            "Place a library symbol on a sheet, optionally as part of an existing component.", true, true,
            Req("sheet_id", "int", "sheet id from create_sheet or project_tree"),
            Req("symbol", "string", "library symbol name, e.g. TR-DI003"),
            Req("x", "number", "insertion point, mm"), Req("y", "number", "insertion point, mm"),
            Opt("angle", "number", "rotation in degrees (default 0)"),
            Opt("component_tag", "string", "tag of an existing component this symbol belongs to"));

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.CurrentProject();
            var sheet = ctx.Sheet(project, a.Int("sheet_id"));
            if (!sheet.canInsertSymbol(out var err)) throw new OpException("cannot_insert_symbol", $"sheet {a.Int("sheet_id")} does not accept symbols");
            var symbol = sheet.newEwProjectSymbol(out err);
            Ew.Check(err, "newEwProjectSymbol");
            Ew.Check(symbol.setEwSymbolName(a.Str("symbol")), "setEwSymbolName");
            Ew.Check(symbol.setXPosition(a.Num("x")), "setXPosition");
            Ew.Check(symbol.setYPosition(a.Num("y")), "setYPosition");
            Ew.Check(symbol.setRotationAngle(a.OptNum("angle", 0)), "setRotationAngle");
            if (a.Has("component_tag"))
            {
                var tag = a.Str("component_tag");
                var component = Identity.Single(ctx.Components(project), c => c.getTag(out _), tag, "component", c => c.getID())
                                ?? throw new OpException("not_found", $"no component tagged '{tag}'");
                Ew.Check(symbol.setObjectID(component.getID()), "setObjectID");
            }
            Ew.Check(symbol.insert(), "insert");
            return ReadModel.Symbol(symbol);
        }
    }

    public sealed class DrawWireOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("draw_wire",
            "Draw a schematic wire line. Ends that lie on connection points connect to them.", true, true,
            Req("sheet_id", "int", ""),
            Req("x1", "number", "mm"), Req("y1", "number", "mm"), Req("x2", "number", "mm"), Req("y2", "number", "mm"),
            Opt("wire_style_id", "int", "wire style; default is the project's"));

        protected override object Run(EwContext ctx, Args a)
        {
            double x1 = a.Num("x1"), y1 = a.Num("y1"), x2 = a.Num("x2"), y2 = a.Num("y2");
            if (x1 == x2 && y1 == y2) throw new OpException("bad_args", "a wire needs two different end points");
            var sheet = ctx.Sheet(ctx.CurrentProject(), a.Int("sheet_id"));
            var line = sheet.newEwProjectLine(EwLineType.kLineSchematic, out var err);
            Ew.Check(err, "newEwProjectLine");
            Ew.Check(line.setStartPointXPosition(x1), "setStartPointXPosition");
            Ew.Check(line.setStartPointYPosition(y1), "setStartPointYPosition");
            Ew.Check(line.setEndPointXPosition(x2), "setEndPointXPosition");
            Ew.Check(line.setEndPointYPosition(y2), "setEndPointYPosition");
            if (a.Has("wire_style_id")) Ew.Check(line.setWireStyleID(a.Int("wire_style_id")), "setWireStyleID");
            Ew.Check(line.insert(), "insert");
            return ReadModel.Line(line);
        }
    }

    public sealed class PlaceTextOp : EwOperation
    {
        static readonly Dictionary<string, EwAlignmentType> Alignments = new Dictionary<string, EwAlignmentType>
        {
            ["bottom_left"] = EwAlignmentType.kAlignmentTypeBottomLeft,
            ["middle_left"] = EwAlignmentType.kAlignmentTypeMiddleLeft,
            ["top_left"] = EwAlignmentType.kAlignmentTypeTopLeft,
            ["middle_center"] = EwAlignmentType.kAlignmentTypeMiddleCenter,
        };

        public override OpSpec Spec { get; } = new OpSpec("place_text",
            "Place a free text (notes, headings). Never for values Electrical can derive (spec §8).", true, true,
            Req("sheet_id", "int", ""), Req("text", "string", ""),
            Req("x", "number", "mm"), Req("y", "number", "mm"),
            Opt("angle", "number", "degrees, default 0"),
            Opt("align", "string", "bottom_left (default) | middle_left | top_left | middle_center"));

        protected override object Run(EwContext ctx, Args a)
        {
            var alignName = a.OptStr("align", "bottom_left");
            if (!Alignments.TryGetValue(alignName, out var align))
                throw new OpException("bad_args", "align must be one of " + string.Join(", ", Alignments.Keys));
            var sheet = ctx.Sheet(ctx.CurrentProject(), a.Int("sheet_id"));
            var text = sheet.newEwProjectMultilingualText(out var err);
            Ew.Check(err, "newEwProjectMultilingualText");
            Ew.Check(text.setText(ctx.Lang, a.Str("text")), "setText");
            Ew.Check(text.setXPosition(a.Num("x")), "setXPosition");
            Ew.Check(text.setYPosition(a.Num("y")), "setYPosition");
            Ew.Check(text.setRotationAngle(a.OptNum("angle", 0)), "setRotationAngle");
            Ew.Check(text.setEwAlignmentType(align), "setEwAlignmentType");
            Ew.Check(text.insert(), "insert");
            return ReadModel.Text(text, ctx.Lang);
        }
    }
}
```

Modify `addin/src/AddLife.SweBridge/Electrical/Operations.cs` — add:

```csharp
            registry.Add(new PlaceSymbolOp());
            registry.Add(new DrawWireOp());
            registry.Add(new PlaceTextOp());
```

- [ ] **Step 4: Build, deploy, run the selftest**

```powershell
dotnet test addin\tests\AddLife.SweBridge.Tests -c Release
# [user] close Electrical
pwsh addin\deploy.ps1
# [user] start Electrical
.venv\Scripts\swe selftest --keep
```
Expected: all steps PASS. In Electrical, LIBTEST sheet 1 of book `ST` shows two `TR-AU01` switches tagged `-STK1` and `-STK2`, a wire between them and the text `Selftest Ω µ → °`. Then `.venv\Scripts\swe selftest --only cleanup-only`.

- [ ] **Step 5: Commit**

```powershell
git add addin swe
git commit -m "feat: place_symbol, draw_wire and place_text operations"
```

### Task 1.6: Read operations — project tree, sheet contents, components

**Files:**
- Create: `addin/src/AddLife.SweBridge/Electrical/Ops/ReadOps.cs`
- Modify: `addin/src/AddLife.SweBridge/Electrical/ReadModel.cs` (`Component`), `Operations.cs`
- Modify: `swe/src/swe/selftest.py` (append steps)

**Interfaces:**
- Consumes: `ReadModel.Sheet/Symbol/Line/Text` (1.4, 1.5); `EwContext.Books/Sheets/Components` (1.4)
- Produces: operations `project_tree {}` → `{project, books:[{id, tag, description}], sheets:[sheet JSON]}`; `sheet_contents {sheet_id}` → `{sheet, symbols:[…], lines:[…], texts:[…]}` with each symbol's `component_tag`; `components {}` → `[{id, tag, description, location_id, function_id, type}]`

- [ ] **Step 1: Append the selftest steps**

Append to `swe/src/swe/selftest.py`:

```python


# ---------------------------------------------------------------- Task 1.6: read back

@step("sheet_contents reads back what was placed")
def _contents(run: Run):
    contents = run.ok(op("sheet_contents", sheet_id=run.state["sheet_id"]))[0]
    placed = {s["id"]: s for s in run.state["symbols"]}
    found = {s["id"]: s for s in contents["symbols"]}
    expect(placed.keys() <= found.keys(), f"symbols {placed.keys() - found.keys()} missing")
    for sid, s in placed.items():
        expect(abs(found[sid]["x"] - s["x"]) < 0.01 and abs(found[sid]["y"] - s["y"]) < 0.01, f"symbol {sid} moved")
    expect({found[sid]["component_tag"] for sid in placed} == {"-STK1", "-STK2"},
           f"component tags {[found[sid]['component_tag'] for sid in placed]}")
    expect(any(l["id"] == run.state["line_id"] for l in contents["lines"]), "the wire line is missing")
    expect(any(t["text"] == TEXT for t in contents["texts"]), "the text is missing or changed")


@step("components and project_tree read back from Electrical")
def _tree(run: Run):
    tree = run.ok(op("project_tree"))[0]
    expect(any(b["tag"] == "ST" for b in tree["books"]), "book ST missing from project_tree")
    expect(any(s["id"] == run.state["sheet_id"] and s["description"] == TEXT for s in tree["sheets"]),
           "sheet missing or its title changed")
    components = {c["tag"]: c for c in run.ok(op("components"))[0]}
    expect(components.get("-STK1", {}).get("description") == TEXT,
           f"-STK1 description reads {components.get('-STK1', {}).get('description')!r}")


@step("a text is deleted by id and is gone")
def _delete_text(run: Run):
    run.ok(op("delete", kind="text", id=run.state["text_id"]))
    contents = run.ok(op("sheet_contents", sheet_id=run.state["sheet_id"]))[0]
    expect(all(t["id"] != run.state["text_id"] for t in contents["texts"]), "the deleted text is still there")
```

- [ ] **Step 2: Add the component read model**

Modify `addin/src/AddLife.SweBridge/Electrical/ReadModel.cs` — add inside the class:

```csharp
        public static Dictionary<string, object> Component(EwProjectComponentX component, string lang) => new Dictionary<string, object>
        {
            ["id"] = component.getID(),
            ["tag"] = component.getTag(out _),
            ["description"] = component.getDescription(lang, out _),
            ["location_id"] = component.getLocationID(out _),
            ["function_id"] = component.getFunctionID(out _),
            ["type"] = component.getType(out _).ToString(),
        };
```

- [ ] **Step 3: Implement the read operations**

`addin/src/AddLife.SweBridge/Electrical/Ops/ReadOps.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;
using AddLife.SweBridge.Core;
using EwAPI;

namespace AddLife.SweBridge.Electrical.Ops
{
    public sealed class ProjectTreeOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("project_tree",
            "Books and sheets of the current project, in book and position order.", false, true);

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.CurrentProject();
            return new Dictionary<string, object>
            {
                ["project"] = ReadModel.ProjectInfo(project),
                ["books"] = ctx.Books(project).Select(b => (object)new Dictionary<string, object>
                {
                    ["id"] = b.getID(), ["tag"] = b.getTag(out _), ["description"] = b.getDescription(ctx.Lang, out _),
                }).ToList(),
                ["sheets"] = ctx.Sheets(project)
                    .OrderBy(s => s.getEwProjectBookID(out _)).ThenBy(s => s.getPosition(out _))
                    .Select(s => (object)ReadModel.Sheet(s, ctx.Lang)).ToList(),
            };
        }
    }

    public sealed class SheetContentsOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("sheet_contents",
            "Everything on one sheet: symbols with connection points and component tags, wire lines, texts.", false, true,
            Req("sheet_id", "int", ""));

        protected override object Run(EwContext ctx, Args a)
        {
            var project = ctx.CurrentProject();
            var sheetId = a.Int("sheet_id");
            var sheet = ctx.Sheet(project, sheetId);
            var tags = ctx.Components(project).ToDictionary(c => c.getID(), c => c.getTag(out _));

            var symbolManager = project.getEwProjectSymbolManager(out var err);
            Ew.Check(err, "getEwProjectSymbolManager");
            var symbols = Ew.Array<EwProjectSymbolX>(symbolManager.getProjectSymbolsFromFileID(sheetId, out err)).Select(s =>
            {
                var json = ReadModel.Symbol(s);
                json["component_tag"] = tags.TryGetValue((int)json["component_id"], out var tag) ? tag : null;
                return (object)json;
            }).ToList();

            var lineManager = project.getEwProjectLineManager(out err);
            Ew.Check(err, "getEwProjectLineManager");
            var lines = Ew.Array<ewProjectLineX>(lineManager.getEwProjectLineArrayFromFileID(sheetId, out err))
                .Select(l => (object)ReadModel.Line(l)).ToList();

            var textManager = project.getEwProjectMultilingualTextManager(out err);
            Ew.Check(err, "getEwProjectMultilingualTextManager");
            var texts = Ew.Array<EwProjectMultilingualTextX>(textManager.getEwProjectMultilingualTextByFileIDArray(sheetId, out err))
                .Select(t => (object)ReadModel.Text(t, ctx.Lang)).ToList();

            return new Dictionary<string, object>
            {
                ["sheet"] = ReadModel.Sheet(sheet, ctx.Lang), ["symbols"] = symbols, ["lines"] = lines, ["texts"] = texts,
            };
        }
    }

    public sealed class ComponentsOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("components",
            "All components of the current project with descriptions in the configured language.", false, true);

        protected override object Run(EwContext ctx, Args a) =>
            ctx.Components(ctx.CurrentProject())
                .OrderBy(c => c.getTag(out _))
                .Select(c => (object)ReadModel.Component(c, ctx.Lang)).ToList();
    }
}
```

Modify `addin/src/AddLife.SweBridge/Electrical/Operations.cs` — add:

```csharp
            registry.Add(new ProjectTreeOp());
            registry.Add(new SheetContentsOp());
            registry.Add(new ComponentsOp());
```

- [ ] **Step 4: Build, deploy, run the selftest**

```powershell
dotnet test addin\tests\AddLife.SweBridge.Tests -c Release
# [user] close Electrical
pwsh addin\deploy.ps1
# [user] start Electrical
.venv\Scripts\swe selftest
```
Expected: all steps PASS, including the three new ones and the clean-up.

- [ ] **Step 5: Commit**

```powershell
git add addin swe
git commit -m "feat: project_tree, sheet_contents and components read operations"
```

### Task 1.7: Action operations — wire numbering, render, PDF, archive and restore

**Files:**
- Create: `addin/src/AddLife.SweBridge/Electrical/Ops/ActionOps.cs`
- Modify: `addin/src/AddLife.SweBridge/Electrical/Operations.cs`
- Modify: `swe/src/swe/selftest.py` (append steps)

**Interfaces:**
- Consumes: `EwContext` (0.5, 1.4); `queries.netlist` (1.1)
- Produces: operations
  - `number_wires {action?}` (`new` default | `new_and_recalculate` | `renumber`) — mutating
  - `render_sheet_png {sheet_id, path, width?, height?}` → `{path, bytes}` — read-only
  - `export_pdf {path, sheet_ids?}` → `{path, bytes}` — read-only
  - `archive_project {name, path}` → `{project_id, path, bytes}` — read-only, any project (backups cover everything)
  - `unarchive_project {path, expect_name}` → `{project_ids, names}` — mutating, target `expect_name`
  - `archive_environment {path, libraries?}` → `{path, bytes}` — read-only

- [ ] **Step 1: Append the selftest steps**

Append to `swe/src/swe/selftest.py`:

```python


# ---------------------------------------------------------------- Task 1.7: actions

@step("wires are numbered and the SQL netlist sees -STK1 to -STK2")
def _numbering(run: Run):
    run.ok(op("number_wires"))
    wires = queries.netlist(run.db, PROJECT)
    joined = [w for w in wires if {w["from_tag"], w["to_tag"]} == {"-STK1", "-STK2"}]
    expect(joined, f"no wire between -STK1 and -STK2 in the SQL netlist ({len(wires)} wires)")
    expect(all(w["tag"] for w in joined), "the wire has no number after number_wires")
    return f"wire {joined[0]['tag']}: -STK1:{joined[0]['from_terminal']} - -STK2:{joined[0]['to_terminal']}"


@step("the sheet renders to PNG")
def _render(run: Run):
    path = run.out_dir / f"selftest-{run.stamp}.png"
    result = run.ok(op("render_sheet_png", sheet_id=run.state["sheet_id"], path=str(path.resolve())))[0]
    expect(result["bytes"] > 10_000, f"PNG is only {result['bytes']} bytes")
    run.state["png"] = str(path)
    return str(path)


@step("the sheet exports to PDF")
def _pdf(run: Run):
    path = run.out_dir / f"selftest-{run.stamp}.pdf"
    result = run.ok(op("export_pdf", path=str(path.resolve()), sheet_ids=[run.state["sheet_id"]]))[0]
    expect(result["bytes"] > 1_000, f"PDF is only {result['bytes']} bytes")
    return str(path)


@step("LIBTEST archives to a .proj.tewzip")
def _archive(run: Run):
    path = run.out_dir / f"LIBTEST-{run.stamp}.proj.tewzip"
    result = run.ok(op("archive_project", name=PROJECT, path=str(path.resolve())), project=None)[0]
    expect(result["bytes"] > 10_000, f"archive is only {result['bytes']} bytes")
    return f"{result['bytes']} bytes"
```

- [ ] **Step 2: Implement the action operations**

`addin/src/AddLife.SweBridge/Electrical/Ops/ActionOps.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using AddLife.SweBridge.Core;
using EwAPI;

namespace AddLife.SweBridge.Electrical.Ops
{
    static class Files
    {
        /// <summary>Output paths must be absolute; the folder is created; the result reports the file size.</summary>
        public static string Prepare(string path)
        {
            if (!Path.IsPathRooted(path)) throw new OpException("bad_args", $"path must be absolute: {path}");
            Directory.CreateDirectory(Path.GetDirectoryName(path));
            return path;
        }

        public static Dictionary<string, object> Written(string path, string what)
        {
            if (!File.Exists(path)) throw new OpException("not_written", $"{what} reported success but {path} does not exist");
            return new Dictionary<string, object> { ["path"] = path, ["bytes"] = new FileInfo(path).Length };
        }
    }

    public sealed class NumberWiresOp : EwOperation
    {
        static readonly Dictionary<string, EwNumberWireAction> Actions = new Dictionary<string, EwNumberWireAction>
        {
            ["new"] = EwNumberWireAction.kNumberNewWiresAction,
            ["new_and_recalculate"] = EwNumberWireAction.kNumberNewWiresAndRecalculateMarksAction,
            ["renumber"] = EwNumberWireAction.kReNumberWiresAction,
        };

        public override OpSpec Spec { get; } = new OpSpec("number_wires",
            "Number wires with the project formula. 'new' keeps existing (including fixed) numbers.", true, true,
            Opt("action", "string", "new (default) | new_and_recalculate | renumber"));

        protected override object Run(EwContext ctx, Args a)
        {
            var name = a.OptStr("action", "new");
            if (!Actions.TryGetValue(name, out var action)) throw new OpException("bad_args", "action must be one of " + string.Join(", ", Actions.Keys));
            var numbering = ctx.CurrentProject().newEwProjectNumberWires(out var err);
            Ew.Check(err, "newEwProjectNumberWires");
            Ew.Check(numbering.setSelectionType(EwSelectionType.kSelectionAll), "setSelectionType");
            Ew.Check(numbering.process(action), "process");
            return new Dictionary<string, object> { ["action"] = name };
        }
    }

    public sealed class RenderSheetPngOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("render_sheet_png",
            "Render one sheet to a PNG file (A3 at 150 dpi by default).", false, true,
            Req("sheet_id", "int", ""), Req("path", "string", "absolute .png path"),
            Opt("width", "int", "pixels, default 2480"), Opt("height", "int", "pixels, default 1754"));

        protected override object Run(EwContext ctx, Args a)
        {
            var path = Files.Prepare(a.Str("path"));
            var sheet = ctx.Sheet(ctx.CurrentProject(), a.Int("sheet_id"));
            var image = ctx.App.newEwSaveDWGImage(out var err);
            Ew.Check(err, "newEwSaveDWGImage");
            Ew.Check(image.setDWGFilePath(sheet.getFilePath(out err)), "setDWGFilePath");
            Ew.Check(image.setDestinationFilePath(path), "setDestinationFilePath");
            Ew.Check(image.setSaveImageType(EwSaveImageType.kSaveImagePNG), "setSaveImageType");
            Ew.Check(image.setWidth(a.OptInt("width", 2480)), "setWidth");
            Ew.Check(image.setHeight(a.OptInt("height", 1754)), "setHeight");
            Ew.Check(image.setOverwriteDestination(true), "setOverwriteDestination");
            Ew.Check(image.save(), "save");
            return Files.Written(path, "render");
        }
    }

    public sealed class ExportPdfOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("export_pdf",
            "Export the whole project, or the given sheets, to one A3 PDF.", false, true,
            Req("path", "string", "absolute .pdf path"), Opt("sheet_ids", "array", "sheet ids; default all sheets"));

        protected override object Run(EwContext ctx, Args a)
        {
            var path = Files.Prepare(a.Str("path"));
            var ids = a.OptArr("sheet_ids");
            var export = ctx.CurrentProject().newEwProjectExportPDF(out var err);
            Ew.Check(err, "newEwProjectExportPDF");
            Ew.Check(export.setExportToPDFFileName(path), "setExportToPDFFileName");
            Ew.Check(export.setPaperFormat(EwPDFPaperFormat.kPDFISO_A3_297_x_420_MM), "setPaperFormat");
            Ew.Check(export.setAllProjectFiles(ids == null), "setAllProjectFiles");
            if (ids != null) Ew.Check(export.setSelectionFiles(ids.Select(i => (object)Convert.ToInt32(i)).ToArray()), "setSelectionFiles");
            Ew.Check(export.setSilentMode(true), "setSilentMode");
            Ew.Check(export.exportPDF(), "exportPDF");
            return Files.Written(path, "export_pdf");
        }
    }

    public sealed class ArchiveProjectOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("archive_project",
            "Archive a project (database + drawings) to a .proj.tewzip. Read-only; works for every project.", false, false,
            Req("name", "string", "exact project name"), Req("path", "string", "absolute .proj.tewzip path"));

        protected override object Run(EwContext ctx, Args a)
        {
            var path = Files.Prepare(a.Str("path"));
            var project = ctx.FindProject(a.Str("name"));
            Ew.Check(ctx.Projects().archive(new object[] { project.getID() }, path, true), "archive");
            var written = Files.Written(path, "archive");
            written["project_id"] = project.getID();
            return written;
        }
    }

    public sealed class UnarchiveProjectOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("unarchive_project",
            "Restore a project from a .proj.tewzip. expect_name must be allowlisted.", true, false,
            Req("path", "string", "absolute .proj.tewzip path"),
            Req("expect_name", "string", "name of the project inside the archive"));

        public override string TargetProject(Args args, string batchProject) => args.Str("expect_name");

        protected override object Run(EwContext ctx, Args a)
        {
            var path = a.Str("path");
            if (!File.Exists(path)) throw new OpException("not_found", $"no archive at {path}");
            var before = new HashSet<int>(ctx.AllProjects().Select(p => p.getID()));
            ctx.Projects().unarchive(path, true, out var err);
            Ew.Check(err, "unarchive");
            var restored = ctx.AllProjects().Where(p => !before.Contains(p.getID())).ToList();
            return new Dictionary<string, object>
            {
                ["project_ids"] = restored.Select(p => (object)p.getID()).ToList(),
                ["names"] = restored.Select(p => (object)p.getName(out _)).ToList(),
            };
        }
    }

    public sealed class ArchiveEnvironmentOp : EwOperation
    {
        public override OpSpec Spec { get; } = new OpSpec("archive_environment",
            "Archive libraries and templates (no projects) to a .tewzip.", false, false,
            Req("path", "string", "absolute .tewzip path"),
            Opt("libraries", "array", "library names; default every object"));

        protected override object Run(EwContext ctx, Args a)
        {
            var path = Files.Prepare(a.Str("path"));
            var archiver = ctx.Env().getEwArchiveEnvironment(out var err);
            Ew.Check(err, "getEwArchiveEnvironment");
            Ew.Check(archiver.setArchivePath(path), "setArchivePath");
            var libraries = a.OptArr("libraries");
            if (libraries != null)
            {
                Ew.Check(archiver.setArchiveMode(EwArchiveMode.kArchiveModeObjectFromLibrary), "setArchiveMode");
                Ew.Check(archiver.setLibraries(libraries), "setLibraries");
            }
            else
            {
                Ew.Check(archiver.setArchiveMode(EwArchiveMode.kArchiveModeAllObjects), "setArchiveMode");
            }
            Ew.Check(archiver.setArchiveProject(false), "setArchiveProject");
            var code = archiver.archive();
            if (code != EwErrorCode.EW_NO_ERROR)
                throw new OpException(code.ToString(), "environment archive failed: " + archiver.getLastArchiveError(out _));
            return Files.Written(path, "archive_environment");
        }
    }
}
```

Modify `addin/src/AddLife.SweBridge/Electrical/Operations.cs` — add:

```csharp
            registry.Add(new NumberWiresOp());
            registry.Add(new RenderSheetPngOp());
            registry.Add(new ExportPdfOp());
            registry.Add(new ArchiveProjectOp());
            registry.Add(new UnarchiveProjectOp());
            registry.Add(new ArchiveEnvironmentOp());
```

If finding S8 recorded that rendering needs the sheet closed (or open), add the matching `sheet.close()` / `sheet.open()` call before `image.save()` in `RenderSheetPngOp`; if S10 recorded that `archive` needs `int[]` ids, change `new object[] { project.getID() }` to `new[] { project.getID() }`.

- [ ] **Step 3: Build, deploy, run the selftest, look at the render**

```powershell
dotnet test addin\tests\AddLife.SweBridge.Tests -c Release
# [user] close Electrical
pwsh addin\deploy.ps1
# [user] start Electrical
.venv\Scripts\swe selftest
```
Expected: all steps PASS; the numbering step prints a wire number and the two terminal labels. **View** the PNG printed by `the sheet renders to PNG`: the AddLife title block, both switches, the numbered wire.

- [ ] **Step 4: Commit**

```powershell
git add addin swe
git commit -m "feat: wire numbering, PNG render, PDF export, archive and restore operations"
```

### Task 1.8: Operations reference from the schema, and the write-only ops log

**Files:**
- Create: `swe/src/swe/opsdoc.py`, `swe/src/swe/opslog.py`, `swe/src/swe/commands/opsdoc.py`
- Modify: `swe/src/swe/commands/batch.py` (log every batch), `swe/src/swe/cli.py` (`COMMAND_MODULES`)
- Create (generated): `docs/reference/ops.md`
- Test: `swe/tests/test_opsdoc.py`, `swe/tests/test_opslog.py`

**Interfaces:**
- Consumes: `BridgeClient.schema()` (0.6), `load_config`, `work_dir` (1.2)
- Produces: `swe.opsdoc.render(schema) -> str`; `swe.opslog.append(work, request, response) -> Path` (there is deliberately no function that reads the log — spec D8); command `swe ops-doc [--out PATH]`

- [ ] **Step 1: Write the failing tests**

`swe/tests/test_opsdoc.py`:

```python
from swe.opsdoc import render

SCHEMA = {"bridge_version": "0.1.0", "ops": [
    {"name": "draw_wire", "summary": "Draw a wire | line", "mutating": True, "requires_project": True,
     "args": [{"name": "x1", "type": "number", "required": True, "doc": "mm"}]},
    {"name": "current_project", "summary": "The open project.", "mutating": False, "requires_project": False, "args": []},
]}


def test_overview_table_and_sections():
    text = render(SCHEMA)
    assert "| `draw_wire` | yes | yes | Draw a wire \\| line |" in text
    assert "| `current_project` | no | no | The open project. |" in text
    assert "## `draw_wire`" in text
    assert "| `x1` | number | yes | mm |" in text
    assert "No arguments." in text
    assert "Do not edit by hand" in text
```

`swe/tests/test_opslog.py`:

```python
import json

import swe.opslog as opslog
from fakebridge import FakeBridge
from swe.cli import main


def test_append_writes_one_line_per_batch(tmp_path):
    request = {"project": "LIBTEST", "session": "s1", "ops": [{"op": "delete", "args": {"kind": "text", "id": 4}}]}
    response = {"batch_id": "b1", "snapshot": "claude-x", "cached": False, "refusal": None,
                "results": [{"op": "delete", "status": "ok"}]}
    path = opslog.append(tmp_path, request, response)
    opslog.append(tmp_path, request, response)
    lines = path.read_text(encoding="utf-8").splitlines()
    assert path == tmp_path / "LIBTEST" / "ops-log.jsonl"
    assert len(lines) == 2
    record = json.loads(lines[0])
    assert record["snapshot"] == "claude-x"
    assert record["ops"] == [{"op": "delete", "args": {"kind": "text", "id": 4}, "status": "ok", "error": None}]


def test_the_log_has_no_reader():
    assert [n for n in dir(opslog) if n.startswith(("read", "load", "last"))] == []


def test_swe_batch_logs_to_the_work_dir(bridge_dir, tmp_path):
    (bridge_dir / "swe.json").write_text(json.dumps({"work_dir": str(tmp_path / "work")}), encoding="utf-8")
    batch_file = tmp_path / "b.json"
    batch_file.write_text(json.dumps({"project": "LIBTEST", "ops": [{"op": "current_project"}]}), encoding="utf-8")
    reply = {"batch_id": "x", "refusal": None, "results": [{"op": "current_project", "status": "ok"}]}
    with FakeBridge(routes={("POST", "/batch"): lambda body: (200, reply)}) as fake:
        fake.write_session(bridge_dir)
        assert main(["batch", str(batch_file), "--session", "s1"]) == 0
    assert (tmp_path / "work" / "LIBTEST" / "ops-log.jsonl").exists()
```

- [ ] **Step 2: Run them to see them fail**

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: `ModuleNotFoundError: No module named 'swe.opsdoc'` / `'swe.opslog'`.

- [ ] **Step 3: Implement**

`swe/src/swe/opsdoc.py`:

```python
"""Renders the add-in's GET /schema as Markdown, so the operations reference cannot drift from the code."""
from __future__ import annotations


def _cell(text: str) -> str:
    return (text or "").replace("|", "\\|").replace("\n", " ")


def render(schema: dict) -> str:
    ops = sorted(schema["ops"], key=lambda o: o["name"])
    lines = ["# Bridge operations", "",
             f"Generated by `swe ops-doc` from the add-in's schema (bridge {schema['bridge_version']}). Do not edit by hand.",
             "", "| Operation | Changes data | Needs the batch's project open | Summary |", "|---|---|---|---|"]
    for o in ops:
        lines.append(f"| `{o['name']}` | {'yes' if o['mutating'] else 'no'} | "
                     f"{'yes' if o['requires_project'] else 'no'} | {_cell(o['summary'])} |")
    for o in ops:
        lines += ["", f"## `{o['name']}`", "", _cell(o["summary"]), ""]
        if o["args"]:
            lines += ["| Argument | Type | Required | Notes |", "|---|---|---|---|"]
            lines += [f"| `{a['name']}` | {a['type']} | {'yes' if a['required'] else 'no'} | {_cell(a['doc'])} |"
                      for a in o["args"]]
        else:
            lines.append("No arguments.")
    return "\n".join(lines) + "\n"
```

`swe/src/swe/opslog.py`:

```python
"""Write-only history of every batch sent with `swe batch` (spec D8).

Nothing in swe or the skill reads this file to decide anything: the Electrical database is the only state.
It exists so a person can see what Claude changed and when."""
from __future__ import annotations

import datetime
import json
import re
from pathlib import Path


def append(work: Path, request: dict, response: dict) -> Path:
    project = request.get("project") or "_no_project"
    path = Path(work) / re.sub(r'[<>:"/\\|?*]', "_", project) / "ops-log.jsonl"
    path.parent.mkdir(parents=True, exist_ok=True)
    results = response.get("results", [])
    record = {
        "at": datetime.datetime.now(datetime.timezone.utc).isoformat(timespec="seconds"),
        "batch_id": response.get("batch_id"),
        "session": request.get("session"),
        "project": request.get("project"),
        "snapshot": response.get("snapshot"),
        "cached": response.get("cached"),
        "refusal": response.get("refusal"),
        "ops": [{"op": o["op"], "args": o.get("args", {}), "status": r.get("status"), "error": r.get("error")}
                for o, r in zip(request["ops"], results)],
    }
    with path.open("a", encoding="utf-8") as log:
        log.write(json.dumps(record, ensure_ascii=False, default=str) + "\n")
    return path
```

`swe/src/swe/commands/opsdoc.py`:

```python
from pathlib import Path

from ..client import BridgeClient
from ..opsdoc import render


def register(subparsers) -> None:
    parser = subparsers.add_parser("ops-doc", help="Write the operations reference from the running add-in's schema.")
    parser.add_argument("--out", default="docs/reference/ops.md")
    parser.set_defaults(handler=run)


def run(args) -> int:
    out = Path(args.out)
    out.parent.mkdir(parents=True, exist_ok=True)
    out.write_text(render(BridgeClient.connect().schema()), encoding="utf-8")
    print(out)
    return 0
```

Modify `swe/src/swe/commands/batch.py` — replace the `run` function with:

```python
def run(args) -> int:
    file_project, ops = load_batch_file(Path(args.file))
    project = args.project or file_project
    session = args.session or new_session_id()
    response = BridgeClient.connect().batch(ops, project=project, session=session,
                                            stop_on_error=not args.continue_on_error, batch_id=args.batch_id)
    opslog.append(work_dir(load_config()), {"project": project, "session": session, "ops": ops}, response)
    print_json(response)
    return 0 if all_ok(response) else 1
```

and add to its imports:

```python
from .. import opslog
from ..config import load_config, work_dir
```

Modify `swe/src/swe/cli.py` — append `"swe.commands.opsdoc",` to `COMMAND_MODULES`.

- [ ] **Step 4: Run the tests to see them pass**

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: all pass.

- [ ] **Step 5: Generate the reference from the live add-in**

Run: `.venv\Scripts\swe ops-doc`
Expected: `docs\reference\ops.md` listing the 24 operations (1 from Phase 0 + 23 from Tasks 1.3–1.7), each with its arguments.

- [ ] **Step 6: Commit**

```powershell
git add swe docs/reference/ops.md
git commit -m "feat(swe): operations reference from the add-in schema; write-only ops log"
```

### Task 1.9: `swe backup` — archives off this PC, retention, status, restore proof

**Needs decision D-C** (backup destination) for the live steps; the code and tests do not.

**Files:**
- Create: `swe/src/swe/backup.py`, `swe/src/swe/commands/backup.py`
- Modify: `swe/src/swe/commands/health.py` (report backup status), `swe/src/swe/cli.py` (`COMMAND_MODULES`)
- Test: `swe/tests/test_backup.py`

**Interfaces:**
- Consumes: `archive_project`, `archive_environment`, `unarchive_project`, `delete_project` (1.3, 1.7); `queries.projects`, `netlist`, `netlist_diff` (1.1); `require`, `load_config`, `work_dir` (1.2)
- Produces: `swe.backup`: `plan(projects, manifest, only=None) -> list[Planned]`, `archive_path(backup_dir, project, now) -> Path`, `expired(backup_dir, now, retention_days) -> list[Path]`, `status(backup_dir, now) -> dict`, `run_backup(client, db, config, now, full=False, only=None) -> dict`, `restore_test(client, db, project, scratch) -> dict`; command `swe backup [--full] [--project NAME] [--status] [--restore-test PROJECT]`; `swe health` gains a `backup` field

- [ ] **Step 1: Write the failing tests**

`swe/tests/test_backup.py`:

```python
import datetime
import os

import pytest

from fakedb import connector
from swe import backup
from swe.db.connection import Db
from swe.db.guard import APP_COLUMNS, PROJECT_COLUMNS, SYMBOL_COLUMNS
from swe.errors import ConfigError
from test_db_guard import columns
from test_db_queries import PROJECTS, db_with

NOW = datetime.datetime(2026, 11, 2, 18, 30, 0)


class FakeClient:
    def __init__(self, fail=()):
        self.calls, self.fail = [], set(fail)

    def batch(self, ops, *, project, session, stop_on_error=True, batch_id=None):
        self.calls.append(ops[0])
        name, args = ops[0]["op"], ops[0]["args"]
        if args.get("name") in self.fail:
            return {"refusal": None, "results": [{"op": name, "status": "error", "error": {"code": "EW_CAN_NOT_WRITE", "message": "x"}}]}
        if name in ("archive_project", "archive_environment"):
            return {"refusal": None, "results": [{"op": name, "status": "ok", "result": {"path": args["path"], "bytes": 1234}}]}
        if name == "unarchive_project":
            return {"refusal": None, "results": [{"op": name, "status": "ok", "result": {"project_ids": [27], "names": ["LIBTEST (1)"]}}]}
        return {"refusal": None, "results": [{"op": name, "status": "ok", "result": {}}]}


def test_plan_backs_up_new_and_modified_projects_only():
    manifest = {"projects": {"Addlife Template": {"modified": "2025-10-06T00:00:00", "backed_up": "2026-11-01T00:00:00"},
                             "LIBTEST": {"modified": "2026-10-01T00:00:00", "backed_up": "2026-10-01T00:00:00"}}}
    assert [(p.name, p.reason) for p in backup.plan(PROJECTS, manifest)] == [("LIBTEST", "modified since the last backup")]
    assert [p.name for p in backup.plan(PROJECTS, {"projects": {}})] == ["Addlife Template", "LIBTEST"]
    assert [p.name for p in backup.plan(PROJECTS, {"projects": {}}, only="LIBTEST")] == ["LIBTEST"]


def test_archive_path_is_per_project_and_timestamped(tmp_path):
    path = backup.archive_path(tmp_path, "Wet Blasting: v2", NOW)
    assert path == tmp_path / "projects" / "Wet Blasting_ v2" / "Wet Blasting_ v2_20261102-183000.proj.tewzip"


def test_expired_keeps_the_newest_archive_of_each_project(tmp_path):
    folder = tmp_path / "projects" / "LIBTEST"
    folder.mkdir(parents=True)
    files = []
    for days_old in (90, 60, 40):
        f = folder / f"LIBTEST_{days_old}.proj.tewzip"
        f.write_bytes(b"x")
        stamp = (NOW - datetime.timedelta(days=days_old)).timestamp()
        os.utime(f, (stamp, stamp))
        files.append(f)
    assert sorted(backup.expired(tmp_path, NOW, 30)) == sorted(files[:2])


def test_status_reports_unset_empty_and_stale(tmp_path):
    assert backup.status(None, NOW)["configured"] is False
    assert backup.status(tmp_path, NOW)["stale"] is True
    (tmp_path / "manifest.json").write_text('{"projects": {"LIBTEST": {"backed_up": "2026-10-20T00:00:00"}}}', encoding="utf-8")
    assert backup.status(tmp_path, NOW)["stale"] is True
    (tmp_path / "manifest.json").write_text('{"projects": {"LIBTEST": {"backed_up": "2026-11-01T00:00:00"}}}', encoding="utf-8")
    assert backup.status(tmp_path, NOW) | {"age_days": 1} == backup.status(tmp_path, NOW)
    assert backup.status(tmp_path, NOW)["stale"] is False


def test_run_backup_refuses_without_a_destination():
    with pytest.raises(ConfigError, match="D-C"):
        backup.run_backup(FakeClient(), db_with(), {"backup_dir": None, "retention_days": 30}, NOW)


def test_run_backup_archives_then_skips_unchanged_projects(tmp_path):
    config = {"backup_dir": str(tmp_path), "retention_days": 30}
    first = backup.run_backup(FakeClient(), db_with(), config, NOW, full=True)
    assert [d.get("project") for d in first["backed_up"][:2]] == ["Addlife Template", "LIBTEST"]
    assert "environment" in first["backed_up"][2]
    second = backup.run_backup(FakeClient(), db_with(), config, NOW + datetime.timedelta(hours=1))
    assert second["backed_up"] == []


def test_a_failed_archive_is_reported_and_not_recorded(tmp_path):
    config = {"backup_dir": str(tmp_path), "retention_days": 30}
    report = backup.run_backup(FakeClient(fail={"LIBTEST"}), db_with(), config, NOW)
    assert report["failed"][0]["project"] == "LIBTEST"
    assert "LIBTEST" not in backup.load_manifest(tmp_path)["projects"]


def test_restore_test_compares_netlists_and_deletes_the_copy(tmp_path):
    wire = {"id": 1, "tag": "1", "from_tag": "-K1", "from_terminal": "13", "to_tag": "-K2", "to_terminal": "A1"}
    restored = {**PROJECTS[1], "id": 27, "name": "LIBTEST (1)", "directory": "27"}
    project_db = {"tew_version": [{"ver_projectdata": 292}], "INFORMATION_SCHEMA": columns(PROJECT_COLUMNS),
                  "FROM dbo.tew_wire": [wire]}
    db = Db(connector({
        "tew_app_project": {"tew_version": [{"ver_projects": 32}], "INFORMATION_SCHEMA": columns(APP_COLUMNS),
                            "FROM tew.tew_project": PROJECTS + [restored]},
        "tew_app_data": {"INFORMATION_SCHEMA": columns(SYMBOL_COLUMNS)},
        "tew_project_data_26": project_db, "tew_project_data_27": project_db}))
    client = FakeClient()
    result = backup.restore_test(client, db, "LIBTEST", tmp_path)
    assert result["identical"] is True
    assert client.calls[-1] == {"op": "delete_project", "args": {"name": "LIBTEST (1)", "id": 27}}
```

- [ ] **Step 2: Run them to see them fail**

Run: `.venv\Scripts\python -m pytest swe\tests\test_backup.py -q`
Expected: `ImportError: cannot import name 'backup' from 'swe'`.

- [ ] **Step 3: Implement the backup module**

`swe/src/swe/backup.py`:

```python
"""Backups of every Electrical project, and of the environment, to a folder off this PC (spec §14).

A project is archived when it has never been backed up or was modified since its last backup. Archives
older than retention_days are pruned, but the newest archive of each project is always kept. A backup only
counts once restore_test has proven an archive restores to an identical netlist."""
from __future__ import annotations

import datetime
import json
import re
from dataclasses import dataclass
from pathlib import Path

from .client import require_ok
from .config import require
from .db import queries
from .errors import SweError

STALE_AFTER_DAYS = 7
MANIFEST = "manifest.json"


def _safe(name: str) -> str:
    return re.sub(r'[<>:"/\\|?*]', "_", name)


def _iso(value) -> str:
    return value.isoformat(timespec="seconds") if isinstance(value, datetime.datetime) else str(value)


def load_manifest(backup_dir: Path) -> dict:
    path = Path(backup_dir) / MANIFEST
    return json.loads(path.read_text(encoding="utf-8")) if path.exists() else {"projects": {}, "environment": None}


def save_manifest(backup_dir: Path, manifest: dict) -> None:
    (Path(backup_dir) / MANIFEST).write_text(json.dumps(manifest, indent=2, ensure_ascii=False), encoding="utf-8")


@dataclass(frozen=True)
class Planned:
    name: str
    modified: str
    reason: str


def plan(projects: list[dict], manifest: dict, only: str | None = None) -> list[Planned]:
    planned = []
    for project in projects:
        if only and project["name"] != only:
            continue
        modified = _iso(project["modified"])
        last = manifest["projects"].get(project["name"])
        if last is None:
            planned.append(Planned(project["name"], modified, "never backed up"))
        elif modified > last["modified"]:
            planned.append(Planned(project["name"], modified, "modified since the last backup"))
    return planned


def archive_path(backup_dir: Path, project: str, now: datetime.datetime) -> Path:
    safe = _safe(project)
    return Path(backup_dir) / "projects" / safe / f"{safe}_{now:%Y%m%d-%H%M%S}.proj.tewzip"


def expired(backup_dir: Path, now: datetime.datetime, retention_days: int) -> list[Path]:
    cutoff = now - datetime.timedelta(days=retention_days)
    folders = list((Path(backup_dir) / "projects").glob("*")) + [Path(backup_dir) / "environment"]
    old = []
    for folder in folders:
        archives = sorted(folder.glob("*.tewzip"), key=lambda f: f.stat().st_mtime)
        old += [f for f in archives[:-1] if datetime.datetime.fromtimestamp(f.stat().st_mtime) < cutoff]
    return old


def status(backup_dir: Path | None, now: datetime.datetime) -> dict:
    if not backup_dir:
        return {"configured": False, "message": "backup_dir is not set (decision D-C): nothing is being backed up"}
    stamps = [p["backed_up"] for p in load_manifest(Path(backup_dir))["projects"].values()]
    if not stamps:
        return {"configured": True, "last_backup": None, "stale": True, "message": "no backup has run yet"}
    last = max(stamps)
    age = (now - datetime.datetime.fromisoformat(last)).days
    return {"configured": True, "last_backup": last, "age_days": age, "stale": age > STALE_AFTER_DAYS,
            "projects": len(stamps)}


def run_backup(client, db, config: dict, now: datetime.datetime, full: bool = False, only: str | None = None) -> dict:
    backup_dir = Path(require(config, "backup_dir", "decision D-C: where backups go, off this PC"))
    backup_dir.mkdir(parents=True, exist_ok=True)
    manifest = load_manifest(backup_dir)
    session = f"backup-{now:%Y%m%d-%H%M%S}"
    done, failed = [], []
    for item in plan(queries.projects(db), manifest, only):
        path = archive_path(backup_dir, item.name, now)
        response = client.batch([{"op": "archive_project", "args": {"name": item.name, "path": str(path)}}],
                                project=None, session=session)
        result = response["results"][0]
        if result["status"] == "ok":
            manifest["projects"][item.name] = {"file": str(path), "modified": item.modified, "backed_up": _iso(now)}
            done.append({"project": item.name, "reason": item.reason, "bytes": result["result"]["bytes"]})
        else:
            failed.append({"project": item.name, "error": result.get("error") or response.get("refusal")})
    if full:
        env_path = Path(backup_dir) / "environment" / f"environment_{now:%Y%m%d-%H%M%S}.tewzip"
        response = require_ok(client.batch([{"op": "archive_environment", "args": {"path": str(env_path)}}],
                                           project=None, session=session))
        manifest["environment"] = {"file": str(env_path), "backed_up": _iso(now)}
        done.append({"environment": str(env_path), "bytes": response["results"][0]["result"]["bytes"]})
    save_manifest(backup_dir, manifest)
    pruned = expired(backup_dir, now, int(config["retention_days"]))
    for archive in pruned:
        archive.unlink()
    return {"backed_up": done, "failed": failed, "pruned": [str(p) for p in pruned]}


def restore_test(client, db, project: str, scratch: Path) -> dict:
    """Archive, restore as a second project, compare netlists, delete the copy."""
    session = "restore-test-" + datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
    path = (Path(scratch) / f"{_safe(project)}-restore-test.proj.tewzip").resolve()

    def one(op_name: str, **args):
        return require_ok(client.batch([{"op": op_name, "args": args}], project=None, session=session))["results"][0]["result"]

    one("archive_project", name=project, path=str(path))
    restored = one("unarchive_project", path=str(path), expect_name=project)
    if len(restored["project_ids"]) != 1:
        raise SweError(f"unarchive created {restored}; expected exactly one project")
    new_id, new_name = restored["project_ids"][0], restored["names"][0]
    try:
        diff = queries.netlist_diff(queries.netlist(db, project), queries.netlist(db, str(new_id)))
    finally:
        one("delete_project", name=new_name, id=new_id)
    return {"restored_as": new_name, "restored_id": new_id,
            "identical": not diff["added"] and not diff["removed"], "diff": diff}
```

`swe/src/swe/commands/backup.py`:

```python
import datetime
from pathlib import Path

from ..backup import restore_test, run_backup, status
from ..client import BridgeClient
from ..config import load_config, work_dir
from ..db.connection import Db
from ..jsonout import print_json


def register(subparsers) -> None:
    parser = subparsers.add_parser("backup", help="Archive changed projects (and with --full the environment) off this PC.")
    parser.add_argument("--full", action="store_true", help="also archive libraries and templates")
    parser.add_argument("--project", help="only this project")
    parser.add_argument("--status", action="store_true", help="only report the age of the last backup")
    parser.add_argument("--restore-test", metavar="PROJECT", help="prove an archive of PROJECT restores identically")
    parser.set_defaults(handler=run)


def run(args) -> int:
    config = load_config()
    now = datetime.datetime.now().replace(microsecond=0)
    if args.status:
        print_json(status(config.get("backup_dir"), now))
        return 0
    if args.restore_test:
        scratch = work_dir(config) / "restore-test"
        scratch.mkdir(parents=True, exist_ok=True)
        result = restore_test(BridgeClient.connect(), Db(), args.restore_test, scratch)
        print_json(result)
        return 0 if result["identical"] else 1
    report = run_backup(BridgeClient.connect(), Db(), config, now, full=args.full, only=args.project)
    print_json(report)
    return 0 if not report["failed"] else 1
```

Modify `swe/src/swe/commands/health.py` — replace `run` with:

```python
def run(args) -> int:
    health = BridgeClient.connect().health()
    health["backup"] = status(load_config().get("backup_dir"), datetime.datetime.now())
    print_json(health)
    return 0
```

and add the imports:

```python
import datetime

from ..backup import status
from ..config import load_config
```

Modify `swe/src/swe/cli.py` — append `"swe.commands.backup",` to `COMMAND_MODULES`.

- [ ] **Step 4: Run all tests**

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: all pass (the existing `test_health_prints_json` still passes: it only reads `current_project`).

- [ ] **Step 5: [user, after D-C] Set the destination and take the first backup**

```powershell
.venv\Scripts\swe config set backup_dir "<the agreed off-PC folder>"
.venv\Scripts\swe backup --full
.venv\Scripts\swe backup --status
```
Expected: `backed_up` lists every project once (all 13 plus LIBTEST) and the environment; `failed` is empty; `--status` shows `stale: false`. A second `swe backup` immediately after backs up nothing.

If the backup folder is not reachable from Electrical's process (UNC rights), `failed` names the projects with `EW_CAN_NOT_WRITE` or `not_written`: fix the share permissions; do not change the code.

- [ ] **Step 6: Commit**

```powershell
git add swe
git commit -m "feat(swe): backups with manifest, retention, status and restore test"
```

### Task 1.10: Phase 1 acceptance — a motor-feed sheet from a batch, a proven restore, handover

**Files:**
- Create: `swe/src/swe/layout.py`, `examples/motor_feed.py`, `PROGRESS.md`
- Modify: `README.md` (commands section), `docs/reference/ops.md` (regenerated)
- Test: `swe/tests/test_layout.py`

**Interfaces:**
- Consumes: every operation of Tasks 1.3–1.7; `queries.netlist` (1.1); `restore_test` (1.9)
- Produces: `swe.layout.end_points(points, side) -> list[dict]` and `swe.layout.pair_vertical(upstream_points, downstream_points) -> list[tuple[dict, dict]]` — the first layout helper, extended in Phase 2

- [ ] **Step 1: Write the failing layout tests**

`swe/tests/test_layout.py`:

```python
import pytest

from swe.layout import end_points, pair_vertical

BREAKER = [{"x": 95, "y": 240}, {"x": 100, "y": 240}, {"x": 105, "y": 240},
           {"x": 95, "y": 220}, {"x": 100, "y": 220}, {"x": 105, "y": 220}]
RELAY = [{"x": 96, "y": 175}, {"x": 101, "y": 175}, {"x": 106, "y": 175},
         {"x": 96, "y": 150}, {"x": 101, "y": 150}, {"x": 106, "y": 150}]


def test_end_points_by_side():
    assert [p["x"] for p in end_points(BREAKER, "bottom")] == [95, 100, 105]
    assert all(p["y"] == 240 for p in end_points(BREAKER, "top"))


def test_pairs_bottom_of_upstream_with_top_of_downstream_in_x_order():
    pairs = pair_vertical(BREAKER, RELAY)
    assert [(a["x"], b["x"]) for a, b in pairs] == [(95, 96), (100, 101), (105, 106)]
    assert all(a["y"] == 220 and b["y"] == 175 for a, b in pairs)


def test_mismatched_pole_counts_are_refused():
    with pytest.raises(ValueError, match="3 .* 2"):
        pair_vertical(BREAKER, [{"x": 1, "y": 5}, {"x": 2, "y": 5}])
```

Run: `.venv\Scripts\python -m pytest swe\tests\test_layout.py -q`
Expected: `ModuleNotFoundError: No module named 'swe.layout'`.

- [ ] **Step 2: Implement the layout helper**

`swe/src/swe/layout.py`:

```python
"""Layout helpers: geometry decisions computed from the connection points Electrical reports back.
Phase 1 has only vertical pole-to-pole pairing; Phase 2 adds the grid, rails and routing."""
from __future__ import annotations

TOLERANCE_MM = 0.01


def end_points(points: list[dict], side: str) -> list[dict]:
    """The connection points on the top or bottom edge of a symbol, left to right."""
    edge = max(p["y"] for p in points) if side == "top" else min(p["y"] for p in points)
    return sorted((p for p in points if abs(p["y"] - edge) < TOLERANCE_MM), key=lambda p: p["x"])


def pair_vertical(upstream: list[dict], downstream: list[dict]) -> list[tuple[dict, dict]]:
    """Pairs the upstream symbol's bottom points with the downstream symbol's top points, pole by pole."""
    lower, upper = end_points(upstream, "bottom"), end_points(downstream, "top")
    if len(lower) != len(upper):
        raise ValueError(f"cannot pair {len(lower)} upstream poles with {len(upper)} downstream poles")
    return list(zip(lower, upper))
```

Run: `.venv\Scripts\python -m pytest swe\tests -q`
Expected: all pass.

- [ ] **Step 3: Write the acceptance script**

`examples/motor_feed.py`:

```python
"""Phase 1 acceptance: build a 3-phase motor feed (-Q1 breaker -> -F1 thermal relay -> -M1 motor) on one
sheet of LIBTEST through the bridge, number the wires, render the sheet, and check the SQL netlist."""
import datetime
import sys
from pathlib import Path

from swe.client import BridgeClient, require_ok
from swe.config import load_config, work_dir
from swe.db import queries
from swe.db.connection import Db
from swe.layout import pair_vertical

PROJECT = "LIBTEST"
DEVICES = [  # tag, symbol, description, insertion point (mm)
    ("-Q1", "TR-DI003", "Motor circuit breaker", (150, 230)),
    ("-F1", "TR-EL039", "Thermal overload relay", (150, 170)),
    ("-M1", "TR-EL092", "Pump motor 1.5 kW", (150, 100)),
]


def main() -> int:
    client = BridgeClient.connect()
    session = "acceptance-" + datetime.datetime.now().strftime("%Y%m%d-%H%M%S")

    def ok(*ops):
        return [r.get("result") for r in require_ok(client.batch(list(ops), project=PROJECT, session=session))["results"]]

    def op(name, **args):
        return {"op": name, "args": args}

    sheet = ok(op("create_book", tag="ACC", description="Phase 1 acceptance"),
               op("create_sheet", book_tag="ACC", position=1, description="Motor feed -M1"),
               op("create_location", tag="CAB", description="Control cabinet"),
               op("create_function", tag="WB", description="Wet blasting"))[1]
    ok(*[op("upsert_component", tag=tag, description=text, location_tag="CAB", function_tag="WB")
         for tag, _, text, _ in DEVICES])
    placed = ok(*[op("place_symbol", sheet_id=sheet["id"], symbol=symbol, x=x, y=y, component_tag=tag)
                  for tag, symbol, _, (x, y) in DEVICES])

    wires = []
    for upstream, downstream in zip(placed, placed[1:]):
        for a, b in pair_vertical(upstream["points"], downstream["points"]):
            wires.append(op("draw_wire", sheet_id=sheet["id"], x1=a["x"], y1=a["y"], x2=b["x"], y2=b["y"]))
    ok(*wires)
    ok(op("number_wires"))

    png = (work_dir(load_config()) / "acceptance" / "motor_feed.png").resolve()
    ok(op("render_sheet_png", sheet_id=sheet["id"], path=str(png)))

    netlist = queries.netlist(Db(), PROJECT)
    links = {tuple(sorted((w["from_tag"], w["to_tag"]))) for w in netlist}
    q1_f1 = [w for w in netlist if {w["from_tag"], w["to_tag"]} == {"-Q1", "-F1"}]
    f1_m1 = [w for w in netlist if {w["from_tag"], w["to_tag"]} == {"-F1", "-M1"}]
    print(f"sheet {sheet['id']}, render {png}")
    print(f"-Q1 -> -F1: {len(q1_f1)} wires, -F1 -> -M1: {len(f1_m1)} wires, links {sorted(links)}")
    return 0 if len(q1_f1) == 3 and len(f1_m1) == 3 else 1


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Run the acceptance**

```powershell
.venv\Scripts\swe selftest
.venv\Scripts\python examples\motor_feed.py
```
Expected: selftest all PASS; the script prints `-Q1 -> -F1: 3 wires, -F1 -> -M1: 3 wires` and exits 0. **View** `work\acceptance\motor_feed.png`: three devices stacked under the AddLife title block, six numbered wires, tags `-Q1`, `-F1`, `-M1`. Open the sheet in Electrical and check the cross-references and the component list behave as for a hand-drawn sheet.

If a device's pole count does not pair (`ValueError`), choose a matching symbol with `.venv\Scripts\swe db search-symbols "three poles"` and update `DEVICES`.

- [ ] **Step 5: Prove a restore**

Run: `.venv\Scripts\swe backup --restore-test LIBTEST`
Expected: `"identical": true`, the restored copy's name, and the copy deleted again (it no longer appears in `swe db projects`).

- [ ] **Step 6: Regenerate the operations reference and update the README**

Run: `.venv\Scripts\swe ops-doc`

Append to `README.md`:

```markdown
## Everyday commands

    .venv\Scripts\swe health                      # bridge, Electrical, current project, backup age
    .venv\Scripts\swe batch ops.json              # send operations; logged to work\<project>\ops-log.jsonl
    .venv\Scripts\swe db projects                 # read-only database queries (also: sheets, components, netlist, search-symbols)
    .venv\Scripts\swe selftest                    # exercise every operation on LIBTEST
    .venv\Scripts\swe backup [--full]             # archive changed projects off this PC
    .venv\Scripts\swe backup --restore-test LIBTEST
    .venv\Scripts\swe ops-doc                     # regenerate docs\reference\ops.md

The operations themselves are documented in `docs/reference/ops.md`.
```

- [ ] **Step 7: Write the handover**

Create `PROGRESS.md` with: date; what Phase 0 found (link the findings document); what Phase 1 built (operations count, `swe` commands); the selftest result and the acceptance result (wire counts, render path); the restore-test result; backup status (`swe backup --status`); the open decisions D-D, D-E, D-F; known limitations found while building; the next step — the Phase 2 plan (library pipeline), to be written from the Phase 0 findings on S4–S6.

- [ ] **Step 8: Commit and push**

```powershell
git add swe examples README.md PROGRESS.md docs/reference/ops.md
git commit -m "feat: Phase 1 acceptance — motor feed from a batch, restore proven; handover"
git push
```

**Phase 1 is done when:** `swe selftest` passes end to end; `examples/motor_feed.py` exits 0 and its render has been viewed; `swe backup --restore-test LIBTEST` reports `identical: true`; `swe backup --status` is not stale; the existing 13 projects are unchanged except for being backed up.

