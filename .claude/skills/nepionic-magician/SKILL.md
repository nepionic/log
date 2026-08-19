---
name: Nepionic-Magician-RestApi
description: Drive TwinCAT 3 XAE end-to-end via the Nepionic_Magician REST API (localhost, /api/v1, auto-incrementing port from 0x4B1D/19229) instead of the PowerShell TCAI module - launch devenv /rootsuffix Exp, create a solution/PLC project/POU/GVL/DUT from scratch, provision and start a usermode runtime, activate/build/login/run, and read/write PLC variables over ADS. Use this when automating TwinCAT PLC project creation and testing programmatically rather than interactively.
---

# Nepionic Magician REST API Skill

A Visual Studio extension (VSIX) that runs inside TwinCAT XAE and exposes a loopback-only REST API over the TwinCAT Automation Interface (TCAI) + ADS. Unlike the `twincat-automation` PowerShell module (external COM/ROT connection, known-buggy), this runs in-process inside devenv.exe and has been independently verified against real hardware, TCAI call by TCAI call.

## Setup

The extension must be built and deployed into the experimental VS hive before any of this works:

```powershell
& "C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\MSBuild.exe" `
  ".\Nepionic_Magician\Nepionic_Magician.csproj" `
  /p:Configuration=Debug /t:Rebuild /nologo /v:minimal
```

Then launch VS with (or without, to create a project from scratch) a target solution:

```powershell
& "C:\Program Files\Microsoft Visual Studio\2022\Common7\IDE\devenv.exe" /rootsuffix Exp [<path-to-.sln>]
```

Poll until the REST server responds (port may be incremented past 19229 if another Exp instance is already running one):

```powershell
Invoke-RestMethod http://127.0.0.1:19229/api/v1/solution -TimeoutSec 2
```

All requests use the envelope `{"success":true,"data":{...}}` or `{"success":false,"error":{"message":"...","code":"..."}}`.

## Full Lifecycle (from nothing to a running, interrogable PLC)

```powershell
$base = "http://127.0.0.1:19229/api/v1"

# 1. Create a brand-new solution + TwinCAT XAE project + PLC project
Invoke-RestMethod "$base/solution" -Method Post -ContentType 'application/json' -Body (@{
    name = "MyProject"; path = "<directory-to-create-the-project-in>"
} | ConvertTo-Json)
# -> plcProject: "MyProject_Plc" - this is the {plcName} used below

$plc = "$base/solution/projects/plc/MyProject_Plc"

# 2. Add a GVL with some variables
Invoke-RestMethod "$plc/gvls" -Method Post -ContentType 'application/json' -Body (@{
    name = "GVL_Test"; declaration = "VAR_GLOBAL`n`tnCounter : INT := 0;`nEND_VAR"
} | ConvertTo-Json)

# 3. Add a POU with an implementation (Declaration is optional - auto-generated if omitted)
Invoke-RestMethod "$plc/pous" -Method Post -ContentType 'application/json' -Body (@{
    name = "PRG_Test"; type = "Program"; implementation = "GVL_Test.nCounter := GVL_Test.nCounter + 1;"
} | ConvertTo-Json)

# 4. Build the whole solution - MUST show errors:0 (also the real check that POU/GVL subtype integers were correct)
Invoke-RestMethod "$base/solution/build" -Method Post
# (or build just this PLC project: Invoke-RestMethod "$plc/build" -Method Post)

# 5. Provision + start a usermode runtime (bootstraps its config folder from UmRT_Template on first use)
Invoke-RestMethod "$base/runtimes" -Method Post -ContentType 'application/json' -Body (@{
    name = "UmRT_MyProject"; amsNetId = "10.0.0.1.1.1"
} | ConvertTo-Json)

# 6. Point the project at that runtime
Invoke-RestMethod "$base/solution/target" -Method Put -ContentType 'application/json' -Body (@{
    amsNetId = "10.0.0.1.1.1"
} | ConvertTo-Json)

# 7. Activate configuration and restart into Run (waits for TwinCAT/devenv restart - can take ~30-90s)
Invoke-RestMethod "$base/solution/activate" -Method Post -ContentType 'application/json' -Body (@{ force = $true } | ConvertTo-Json)

# 8. Login (download + login, no dialogs)
Invoke-RestMethod "$plc/session" -Method Post -ContentType 'application/json' -Body '{}'

# 9. Open ADS connection - AFTER login, not before (port 851 only exists post-download)
Invoke-RestMethod "$base/ads/connection" -Method Post -ContentType 'application/json' -Body (@{ amsNetId = "10.0.0.1.1.1" } | ConvertTo-Json)

# 10. Start the PLC task
Invoke-RestMethod "$plc/state" -Method Put -ContentType 'application/json' -Body (@{ state = "Run" } | ConvertTo-Json)

# 11. Interrogate: list symbols, read/write variables
Invoke-RestMethod "$base/ads/symbols?filter=GVL_Test.*"
Invoke-RestMethod "$base/ads/variables?path=GVL_Test.nCounter"
Invoke-RestMethod "$base/ads/variables" -Method Put -ContentType 'application/json' -Body (@{ path = "GVL_Test.nCounter"; value = 100 } | ConvertTo-Json)

# 12. Teardown
Invoke-RestMethod "$plc/state" -Method Put -ContentType 'application/json' -Body (@{ state = "Stop" } | ConvertTo-Json)
Invoke-RestMethod "$base/ads/connection" -Method Delete
Invoke-RestMethod "$plc/session" -Method Delete
Invoke-RestMethod "$base/runtimes/UmRT_MyProject" -Method Delete
```

## Library Projects (verified working end-to-end)

```powershell
# 1. Create a standalone PLC library project (its own top-level VS project)
Invoke-RestMethod "$base/solution/projects/plc" -Method Post -ContentType 'application/json' -Body (@{
    name = "MyLibrary"; plcName = "MyFunkyLibrary"
} | ConvertTo-Json)

$lib = "$base/solution/projects/plc/MyFunkyLibrary"

# 2. Set its metadata (Company/Title/Version/Placeholder/etc - the "Project Information" fields)
Invoke-RestMethod "$lib/metadata" -Method Put -ContentType 'application/json' -Body (@{
    company = "MyCompany"; title = "MyCompany_MyFunkyLibrary"; version = "0.1.0"; released = $false
    defaultNamespace = "MyCompany_MyFunkyLibrary"; placeholder = "MyCompany_MyFunkyLibrary"
    author = "MyCompany"; referencedLibrary = $true
} | ConvertTo-Json)

# 3. Add a callable Function
Invoke-RestMethod "$lib/pous" -Method Post -ContentType 'application/json' -Body (@{
    name = "FUN_Add"; type = "Function"; returnType = "INT"
    declaration = "FUNCTION FUN_Add : INT`nVAR_INPUT`n`ta : INT;`n`tb : INT;`nEND_VAR"
    implementation = "FUN_Add := a + b;"
} | ConvertTo-Json)

# 4. Reference it from another PLC project - name is the target's Placeholder, SQUARE-BRACKETED
#    for a same-solution project reference (see Critical Rule 16)
Invoke-RestMethod "$base/solution/projects/plc/MyProject_Plc/libraries" -Method Post -ContentType 'application/json' -Body (@{
    name = "[MyCompany_MyFunkyLibrary]"; version = "0.1.0"; company = "MyCompany"
} | ConvertTo-Json)

# 5. Call it from a POU in the consuming project (unqualified call worked - no namespace prefix needed)
Invoke-RestMethod "$base/solution/projects/plc/MyProject_Plc/pous" -Method Post -ContentType 'application/json' -Body (@{
    name = "PRG_LibTest"; type = "Program"
    declaration = "PROGRAM PRG_LibTest`nVAR`n`tnResult : INT;`nEND_VAR"
    implementation = "nResult := FUN_Add(3, 4);"
} | ConvertTo-Json)

# 6. Build the WHOLE solution (not just one project) to compile the cross-project reference
Invoke-RestMethod "$base/solution/build" -Method Post
```

## Route Reference

**Solution** (`ITcSysManager3`-backed)
| Method | Path | Body/Query | Notes |
|---|---|---|---|
| POST | `/solution` | `{name, path}` | Create solution+project+PLC project, bundled |
| GET | `/solution` | - | Status: `{solutionLoaded, solutionPath, tcProjectFound, amsNetId, ideVersion}` |
| GET | `/solution/tree` | `?path=TIPC^...&depth=2` | Browse the raw tree (`{name, pathName, itemType, itemSubTypeName, children}`) - useful for discovering real structure/subtypes empirically instead of guessing. Cannot resolve hidden subitems (e.g. anything under a PLC project's `"<name> Project"` node) - use the PLC-scoped tree endpoints below for those instead |
| GET | `/solution/tree/xml` | `?path=...` | Raw `ProduceXml(false)` dump for any resolvable path - same hidden-subitem limitation as above |
| PUT | `/solution/tree/rename` | `?path=...` `{name}` | Rename any resolvable (non-hidden) tree item |
| POST | `/solution/tree/move` | `?path=...` `{destPath}` | Move any resolvable (non-hidden) tree item |
| GET | `/solution/state` | - | `{state, adsState}` (Config/Run) |
| PUT | `/solution/state` | `{state}` | `"Run"` or `"Config"` |
| PUT | `/solution/target` | `{amsNetId}` | Set the project's target AmsNetId |
| POST | `/solution/activate` | `{force}` | Activate configuration; `force:true` also restarts into Run. **Known risk**: can crash a running UmRT and wedge the shared AMS router system-wide - see Critical Rule 19 |
| POST | `/solution/build` | `{target?}` | Whole-solution build. `{target, errors, warnings, messages}`. `target`: `"build"` (default), `"rebuild"`, or `"clean"`. Builds the entire VS/TcXaeShell project (I/O config + every nested PLC project) - use `/solution/projects/plc/{name}/build` (below) to build just one PLC project. **The `errors` count only reflects VS's Error List, which can miss errors from a non-active build platform - always cross-check `GET /solution/output/Build`'s raw text for anything downstream will rely on** (see Critical Rule 20). Also: `target:"build"` can leave a stale incremental type-check cache for an unused platform - use `"rebuild"` if in doubt |
| GET | `/solution/projects` | - | Raw VS-project diagnostic: `{projects: [{name, uniqueName, kind, objectTypeName}]}` - `objectTypeName` lists which `TCatSysManagerLib` interfaces `.Object` implements (e.g. `"ITcSysManager3"`). Useful for discovering a new project type's real shape empirically |
| GET | `/solution/output` | - | `{names: [...]}` - every VS Output window pane (Build, Debug, General, and a TwinCAT-specific "TwinCAT" pane with runtime/deploy diagnostics not visible anywhere else) |
| GET | `/solution/output/{name}` | - | `{name, text}` - full text content of that pane, e.g. `/solution/output/Build` or `/solution/output/TwinCAT` |
| GET | `/solution/projects/plc` | - | `{names: [...]}` - every PLC project instance name, both nested (under a TcXaeShell project's TIPC) and standalone (their own top-level VS project, e.g. a library) - see Critical Rule 13 |
| POST | `/solution/projects/plc` | `{name, plcName?}` | Creates a standalone PLC library project as a new top-level VS project (`File > New Project > TwinCAT PLC Project` equivalent) - `name` is the VS project name, `plcName` is the PLC project's own name if it should differ (defaults to `name`). Returns `{project, plcName, plcProject, plcProjectSubTypeName}` |
| POST | `/solution/tasks` | `{name, priority?, cycleTicks?}` | Creates a REAL real-time task under the solution-scoped `TIRT` root (System > Real-Time > Tasks) - see Critical Rule 21 for the two-kinds-of-task model. `priority` 1-61 (default 20), `cycleTicks` raw ticks 1-65535 (default 1000) - NOT a time value, see Critical Rule 22 |
| GET | `/solution/tasks` | - | `{names: [...]}` - every real-time task under `TIRT` |
| GET/PUT/DELETE | `/solution/tasks/{name}` | `{priority?, cycleTicks?}` on PUT | Get/update/delete a real-time task |

**PLC project** (`/solution/projects/plc/{plcName}/...` - a solution can contain more than one PLC project, addressed by name; other project types, e.g. Safety/NC, would get their own `/solution/projects/{type}/{name}` sibling in future)
| Method | Path | Body/Query | Notes |
|---|---|---|---|
| POST | `/solution/projects/plc/{plcName}/session` | `{}` | Login (download + login, no dialogs) |
| DELETE | `/solution/projects/plc/{plcName}/session` | - | Logout |
| GET | `/solution/projects/plc/{plcName}/state` | - | `{appState, opState, isLoggedIn}` |
| PUT | `/solution/projects/plc/{plcName}/state` | `{state}` | `"Run"`, `"Stop"`, or `"Reset"` |
| POST | `/solution/projects/plc/{plcName}/build` | `{target?}` | Builds only this PLC project via `ITcPlcProject.CompileProject()` (distinct COM call from solution-level build - see Critical Rule 12, VERIFIED working). Only `target:"build"` is supported (no per-project clean/rebuild exists in TCatSysManagerLib) |
| POST | `/solution/projects/plc/{plcName}/pous` | `{name, type, returnType?, declaration?, implementation?}` | `type`: `Program`/`FunctionBlock`/`Function`. Declaration auto-generated if omitted. Response includes `linkedToTask` - for `Program` type, automatically wires the POU into the PLC Task's call list (see Critical Rule 8). See Critical Rule 14 for `Function`-specific `vInfo` handling |
| GET | `/solution/projects/plc/{plcName}/pous/{name}` | - | `{path, declaration, implementation}` |
| PUT | `/solution/projects/plc/{plcName}/pous/{name}` | `{declaration?, implementation?}` | Partial update - only non-null fields written, `updated` reports which |
| GET | `/solution/projects/plc/{plcName}/pous/{name}/source` | `?format=text` | Plain-text "single file" view of a POU: Declaration + Implementation, plus (for a FunctionBlock) each Method's/Property's own, with proper IEC closing keywords (`END_METHOD`, `END_PROPERTY`, nested `GET`/`SET`, etc.) |
| PUT | `/solution/projects/plc/{plcName}/pous/{name}/source` | `{source}` | Reverse of the above - parses the same shape back apart; matches Methods/Properties/accessors against EXISTING children by name, does not create new ones |
| POST | `/solution/projects/plc/{plcName}/pous/{name}/methods` | `{name, returnType?, declaration?, implementation?}` | Adds a Method to a FunctionBlock POU |
| POST | `/solution/projects/plc/{plcName}/pous/{name}/properties` | `{name, type}` | Adds a Property to a FunctionBlock POU (container only) |
| POST | `/solution/projects/plc/{plcName}/pous/{name}/properties/{propName}/get` `/set` | - | Adds the Get/Set accessor under an existing Property - does NOT auto-create, needs its own call. Accessor body is set via the POU-level `source` endpoint above, not a dedicated route |
| POST | `/solution/projects/plc/{plcName}/gvls` | `{name, declaration}` | `declaration` required (raw `VAR_GLOBAL ... END_VAR`) |
| GET/PUT | `/solution/projects/plc/{plcName}/gvls/{name}` | `{declaration}` | Read/update declaration |
| POST | `/solution/projects/plc/{plcName}/duts` | `{name, dutType, declaration}` | `dutType`: `Struct`/`Enum`/`Alias`/`Union` |
| GET/PUT | `/solution/projects/plc/{plcName}/duts/{name}` | `{declaration}` | Read/update declaration |
| POST | `/solution/projects/plc/{plcName}/interfaces` | `{name, declaration}` | Container only (`INTERFACE X [EXTENDS Y]`) - use the interface-scoped method/property routes below to add real members |
| GET/PUT | `/solution/projects/plc/{plcName}/interfaces/{name}` | `{declaration}` | Read/update the interface's own header declaration |
| POST | `/solution/projects/plc/{plcName}/interfaces/{name}/methods` | `{name, returnType?, declaration?, implementation?}` | Adds a Method signature to an Interface (mirrors the POU-scoped route, different underlying subtype - see Critical Rule 23) |
| POST | `/solution/projects/plc/{plcName}/interfaces/{name}/properties` | `{name, type}` | Adds a Property to an Interface (container only) |
| POST | `/solution/projects/plc/{plcName}/interfaces/{name}/properties/{propName}/get` `/set` | - | Adds the Get/Set accessor declaration under an Interface Property |
| POST | `/solution/projects/plc/{plcName}/folders` | `{name, folderPath?}` | Creates a folder |
| GET/PUT/DELETE | `/solution/projects/plc/{plcName}/tree/{name}` | `?folderPath=...` | Generic read raw XML / rename / delete for any named child under a PLC project (folders, POUs, tasks, etc) - resolves via `LookupChild`, so (unlike the solution-level `/solution/tree/*`) it CAN reach hidden subitems |
| POST | `/solution/projects/plc/{plcName}/tree/{name}/move` | `?folderPath=...` `{destPath}` | Move any named child (empty/omitted `destPath` means move to the PLC project root) - see Critical Rule 24 for a real caveat when moving a Task specifically |
| GET/PUT | `/solution/projects/plc/{plcName}/metadata` | `{company?, title?, version?, released?, defaultNamespace?, placeholder?, author?, referencedLibrary?}` | Company/Title/Version/Released/DefaultNamespace/Author round-trip via COM (`ITcSmTreeItem.ProduceXml`/`ConsumeXml`); Placeholder/ReferencedLibrary have no COM path - see Critical Rule 15. **A freshly-created library project's empty `<IECProjectDef />` needs the `ProjectInfo` wrapper synthesized on first write** - already handled, but worth knowing if you see `"References node not found!"` |
| GET | `/solution/projects/plc/{plcName}/xml` | - | `{xml}` - raw `ProduceXml(false)` dump of the PLC project node, diagnostic for discovering the real XML shape empirically (same role as `/solution/tree` but for XML content, not just tree structure) |
| POST | `/solution/projects/plc/{plcName}/libraries` | `{name, version?, company?}` | Adds a library reference (wires e.g. a referenced-library project into this PLC project's References) via `ITcPlcLibraryManager.AddLibrary` - see Critical Rule 16 for the required `name` format |
| GET | `/solution/projects/plc/{plcName}/tasks/{taskName}/programs` | - | List a Task's program call-list |
| POST | `/solution/projects/plc/{plcName}/tasks/{taskName}/programs` | `{programName}` | Link an existing Program POU into an existing Task's call list |
| DELETE | `/solution/projects/plc/{plcName}/tasks/{taskName}/programs/{programName}` | - | Unlink |
| POST | `/solution/projects/plc/{plcName}/tasks` | `{name, realTimeTaskName, folderPath?}` | Creates a PLC-side "referenced task" pointer bound to an existing real-time task by name - see Critical Rule 21 |
| GET | `/solution/projects/plc/{plcName}/tasks` | `?folderPath=...` | List referenced tasks (names only) |
| GET/DELETE | `/solution/projects/plc/{plcName}/tasks/{name}` | `?folderPath=...` | Get details (`{name, path, linkedTask}`) / delete a referenced task |
| PUT | `/solution/projects/plc/{plcName}/tasks/{name}/assignment` | `{realTimeTaskName}` | "Assign to Task" - (re)links a referenced task to a real-time task by name |

**ADS**
| Method | Path | Body/Query | Notes |
|---|---|---|---|
| POST | `/ads/connection` | `{amsNetId?, port?}` | Connect (amsNetId auto-detected from the open project if omitted) |
| GET | `/ads/connection` | - | Non-throwing status: `{connected, amsNetId, port}` |
| DELETE | `/ads/connection` | - | Disconnect |
| GET | `/ads/symbols` | `?filter=MAIN.*` | Recursively flattened leaf variables (not just top-level instances) |
| GET | `/ads/variables` | `?path=GVL.var` | Read one variable |
| PUT | `/ads/variables` | `{path, value}` | Write one variable |

**Runtimes** (plain OS process management, no TCAI/COM)
| Method | Path | Body/Query | Notes |
|---|---|---|---|
| POST | `/runtimes` | `{name, amsNetId}` | Start (idempotent) - `amsNetId` is required, not auto-derived |
| GET | `/runtimes/{name}` | - | `{running, pid, commandLine}` |
| DELETE | `/runtimes/{name}` | - | Stop (hard kill in this version) |

**Debugger** (`/debugger/*` - EnvDTE.Debugger automation, plus in-process command invocation for the embedded CODESYS PLC editor; real IEC breakpoints/stepping are NOT reachable this way, see Critical Rule 18)
| Method | Path | Body/Query | Notes |
|---|---|---|---|
| GET | `/debugger` | - | `{currentMode, lastBreakReason, currentProcessName, currentProgramName, breakpointCount, hasCurrentStackFrame}` |
| POST | `/debugger/action` | `?action=go\|stepover\|stepinto\|stepout\|break` | Generic `EnvDTE.Debugger` control - unverified against a real halted TC PLC session |
| GET | `/debugger/cds-commands` | - | Reflects `Beckhoff.TwinCAT.VS.CDSCmdGuids`' full field list live (no decompilation needed) - every named command the embedded CODESYS PLC editor knows, with its GUID |
| POST | `/debugger/cds-command` | `?name=<CommandName>` | Invokes a CODESYS command in-process via `CDSCmdTools.Exec(Guid)` - the same path a gutter/menu click takes. Names come from the hand-curated `CdsCommandGuids` dict in `TcDebuggerService.cs` (breakpoints, login/logout, stepping) - see `/debugger/cds-commands` for the FULL list if you need one not yet added there |
| GET | `/debugger/dte-commands` | `?filter=<substring>` | Lists standard `EnvDTE.Commands` (VS-level menu/toolbar commands) - a DIFFERENT surface from CDS commands above, e.g. File-menu items like `File.OpenTcTaskDump`/`File.OpenProjectfromTarget`/`PLC.LoadCoreDump` live here, not in CDSCmdGuids |
| POST | `/debugger/dte-command` | `?name=<Name>&args=<optional>` | Invokes a standard DTE command by Name. **Some commands pop a modal dialog and the REST call will hang until it's dismissed** - see Critical Rule 25 for a safe recovery technique |
| GET | `/debugger/open-document` | `?path=<file path>` | Opens a file via `DTE.ItemOperations.OpenFile` and reports document/window type info - diagnostic for "can this kind of content be addressed by EnvDTE.Debugger.Breakpoints". **Do not use on a `.core` dump file** - `.core` isn't a registered type and this pops a blocking "Open With" picker (see Critical Rule 25) |
| GET | `/debugger/open-document/assembly` | - | Type/assembly info for whatever `dte.ActiveWindow.Object` currently is - lets you find and reflect over the exact DLL implementing a custom editor without guessing paths |

## Critical Rules

1. **AmsNetId must be explicit for `/runtimes`** - the `-i path` default-derivation semantics used by TwinCAT's own template `Start.bat` are undocumented and unverified; always pass a chosen AmsNetId.
2. **Lifecycle order matters**: build → activate → login (`/plc/session`) → ADS connect (`/ads/connection`) → run. ADS port 851 only exists after login+download.
3. **Subtype integers for POU/GVL/DUT/PLC-project creation have no backing enum anywhere in TCatSysManagerLib** (confirmed via reflection) - trust the `subTypeName` field every create response returns over any assumption; if it looks wrong, `POST /solution/build` will very likely fail with a real IEC compiler error. **This does NOT hold universally** - the Interface method/property family (`TREEITEMTYPE_PLCITFMETH`/`PLCITFPROP`/etc, Critical Rule 23) and the Task family (`TREEITEMTYPE_TASK`/`TREEITEMTYPE_PLCTASK`, Critical Rule 21) DO have real, findable enum values - always check `TCatSysManagerLib.TREEITEMTYPES` via direct reflection first (see Critical Rule 26) before assuming you have to guess.
4. **A routine, harmless double package-load** happens on every TwinCAT XAE launch (not just restarts) - VS logs a "SetSite failed for package [NepionicMagicianPackage]" entry every time. This is a TwinCAT-XAE-specific VS quirk, mitigated by a process-wide singleton guard - it is not a sign anything is actually broken as long as the REST server responds.
5. **`/system/*` no longer exists** - the whole surface was renamed to `/solution/*` for a RESTful redesign (resource nouns, correct verbs). `/ads/connect`→`/ads/connection`, `/ads/variable`→`/ads/variables`, `/plc/login`+`/plc/logout`→`POST`/`DELETE /plc/session`, `/runtime/stop`→`DELETE /runtimes/{name}`.
6. **`/solution/activate {force:true}` can restart the whole devenv.exe process**, not just the TwinCAT runtime, for local/UM targets - poll `GET /solution` until it responds again rather than assuming the same process/port survives.
7. **ADS variable "force" (override until released) is not implemented** - no Force-named API exists anywhere in the `Beckhoff.TwinCAT.Ads` managed library; this needs dedicated research before it can be added, not a guessed raw ADS index group.
8. **A `Program`-type POU that's never referenced by a PLC Task compiles clean but is dead code** - TwinCAT's dead-code elimination silently strips it (and anything only it references) from the actual downloaded boot project, even though the build reports 0 errors. `POST /plc/pous` auto-wires `Program` POUs into the `PlcTask` node's call list for exactly this reason (`ITcSmTreeItem.CreateChild(name, 650, "", "")` under the `PlcTask` node, matching the `ITcPlcTaskReference`/`TREEITEMTYPE_PLCPROGREF` pattern the default template's own `MAIN` program uses - confirmed via `GET /solution/tree`). If a POU seems to compile but its variables never show up in `/ads/symbols`, check `linkedToTask` in its create response. **Also check the auto-link actually found the task** - if `PlcTask` has been moved out of its default location (e.g. into a user-renamed folder), the auto-link can silently fail to find it; link explicitly via `POST .../tasks/{taskName}/programs` instead.
9. **Every mutating POU/GVL/DUT call saves the project to disk** (`Project.Save()` via DTE) - without this, in-memory-only changes are silently lost on the next devenv restart. If a previously-created item is missing after a restart, this is the first thing to suspect.
10. **New usermode runtimes need a valid license before they can enter Run state** - `POST /runtimes` only starts the process; a freshly bootstrapped instance has no trial license and will stay stuck in `Config` state (`PUT /solution/state {state:"Run"}` reports success but `currentState` stays `Config`) until one is added. Getting a genuine 7-day trial requires a human to read and respond to a security-code dialog in the TwinCAT IDE (Beckhoff deliberately gates this against automation - confirmed via official docs and the `TcLicenseServices.h` SDK header's `ApplicationChallenge` mechanism). **Workaround confirmed working**: trial licenses appear to be bound to the machine, not the specific runtime instance - copying an already-licensed instance's `Target\License\TrialLicense.tclrs` file into the new instance's `Target\License\` folder (creating the folder if needed) and restarting the runtime process picks it up. No REST endpoint for this yet (manual file copy + `DELETE`/`POST /runtimes/{name}` to restart).
11. **POU/GVL/DUT declaration content is set via `ITcPlcDeclaration.DeclarationText` *after* `CreateChild`, never via the `vInfo` parameter** - passing a `<Declaration><![CDATA[...]]></Declaration>`-wrapped string as `vInfo` does NOT get unwrapped by TwinCAT; it gets stored verbatim as literal text (confirmed empirically - `GET` on the created item echoed the literal XML back), which then fails to parse as ST/IEC. `vInfo` at creation time should be an `IECLANGUAGETYPES` value (e.g. `IECLANGUAGE_ST`) for POUs/GVLs/DUTs.
12. **`/plc/*` was restructured to `/solution/projects/plc/{plcName}/*`** (formerly a flat `/plc/*` with an optional `plcProjectPath` query-string escape hatch for the multi-PLC-project case). The old `plcProjectPath` raw-tree-path parameter is gone entirely - `{plcName}` is a plain PLC project name, resolved by matching `ITcSmTreeItem.Name` under `TIPC`, not a serialized tree path. Solution-level build (`POST /solution/build`) and PLC-project-level build (`POST /solution/projects/plc/{plcName}/build`) are two genuinely different COM calls - `EnvDTE.SolutionBuild.Build()`/`.Clean()` for the former, `ITcPlcProject.CompileProject()` for the latter, because a TwinCAT solution normally has exactly one VS-level project (the `.tsproj`) with multiple PLC projects nested inside as tree children, so `SolutionBuild.BuildProject` can't distinguish between them. **VERIFIED working against real hardware** - but `ITcPlcProject` casts from the shallow PLC *instance* node (the raw `TIPC` child), NOT the deeper `"<name> Project"` node that `ITcPlcOnline`/POUs/GVLs/DUTs all resolve through (that cast fails) - the two nodes support different COM interfaces despite looking like "the same PLC project" from the tree.
13. **A single solution can contain more than one `ITcSysManager3`-backed project** - not just the main TwinCAT XAE project. A standalone PLC library project (`File > New Project > TwinCAT PLC Project`, e.g. the user's "SomeLibrary" containing "SomeFunkyLibrary") is ALSO `ITcSysManager3`-backed (a minimal one, PLC-only, no I/O config, `.tspproj` not `.tsproj`) with its own `TIPC` - it is NOT a bare `ITcSmTreeItem` as an earlier draft of this code assumed. PLC-project name resolution (`LookupPlcInstanceAsync`) searches `TIPC` across every `ITcSysManager3`-backed project in the solution, not just the first one found - a single cached "primary sysManager" model breaks as soon as a second one exists. `GET /solution/projects` is the diagnostic that revealed this (both projects reported `objectTypeName: "ITcSysManager3"`). **Note**: a standalone library project does NOT have its own `TIRT` (real-time Tasks) - that's solution/main-project-scoped only.
14. **Creating a `Function`-type POU needs `vInfo = null` in `CreateChild`** - a THIRD distinct case, different from both Program/FunctionBlock (`vInfo = IECLANGUAGETYPES.IECLANGUAGE_ST`) and GVLs (same enum). Passing the language enum (boxed Int32) fails with `"the specified vInfo (Type: Int32) is not supported for creating TreeItem type 'TREEITEMTYPE_PLCPOUFUNC'"`; passing the return type as a string also fails (`"vInfo (Type: String) is not supported"`). Only `vInfo = null` succeeds - the return type is fully carried by the `FUNCTION x : <ReturnType>` declaration text set afterward via `ITcPlcDeclaration`, so `vInfo` needs nothing at all for this subtype.
15. **PLC project metadata splits across two different mechanisms, confirmed via `GET .../xml`**: Company/Title/Version/Released/DefaultNamespace/Author live in `ITcSmTreeItem.ProduceXml`'s `IECProjectDef/ProjectInfo` element and round-trip cleanly through `ProduceXml`/`ConsumeXml`. **A freshly-created library project's `ProduceXml` can come back with a completely empty `<IECProjectDef />` (no `ProjectInfo` child at all, not just blank fields)** - the metadata write path builds the `ProjectInfo` wrapper itself when missing; if you ever see `"References node not found!"` from `ConsumeXml`, that's this case not being handled somewhere. **Placeholder and ReferencedLibrary (`VirtualLibrary` in the file) do NOT appear anywhere in that XML schema** - `ProduceXml`'s `DevicePlaceholders` section only lists library *references this project consumes*, not this project's own placeholder identity when compiled as a library, and no COM interface anywhere in `TCatSysManagerLib` exposes either field (confirmed via reflection). Those two are read/written by locating the `.plcproj` file directly (path derived as `<owning VS project dir>\<plcName>\<plcName>.plcproj`) and editing its `<PropertyGroup>`. **This file write is NOT visible to VS's in-memory project state** - if anything else mutates and saves the same PLC project afterward (another POU/GVL/DUT call, another metadata PUT) before a devenv/solution reload, VS will silently overwrite the file edit back to its prior in-memory value. Only safe as the last write before a reload. `PUT .../metadata` calls `SaveActiveProjectAsync()` immediately before writing the file for exactly this reason - to land on top of current state, not stale content.
16. **Adding a library reference to a PLC project (`POST .../libraries`, `ITcPlcLibraryManager.AddLibrary`) requires the target's Placeholder name wrapped in square brackets** (e.g. `"[ClaudeLabs_ClaudeFunkyLibrary]"`, not `"ClaudeLabs_ClaudeFunkyLibrary"`) when referencing another project *in the same solution* rather than an installed/managed library - confirmed via the actual error (`"Managed Library '...' not found!"` without brackets) and by inspecting the resulting `.plcproj`, which shows `<LibraryReference Include="[Company_LibName],Version,Company">` for the bracketed form vs. the `<PlaceholderReference>` form VS's UI produces manually - both are valid and TC resolves either at build/run time. **Full round-trip VERIFIED live**: created a new library project from scratch via `POST /solution/projects/plc`, set all 8 metadata fields via `PUT .../metadata`, added a `Function` POU (`FUN_ClaudeAdd`), referenced it from the main PLC project via `POST .../libraries`, wrote a `Program` POU in the main project calling `FUN_ClaudeAdd(3, 4)`, built the whole solution (0 errors), provisioned+licensed+activated a runtime, and read the live result over ADS - got `7`, confirming the library call actually executed on-target, not just compiled.
17. **`Solution.AddFromTemplate` (used by `POST /solution/projects/plc` to add a new top-level VS project to an existing solution) does NOT save the `.sln` file's reference to the new project** - calling `.Save()` on the new project itself is not enough (mirrors the earlier POU/GVL/DUT disk-save gotcha, but at the solution level). Without an explicit `dte.Solution.SaveAs(dte.Solution.FullName)` afterward, the new project's own files are written to disk correctly but the solution silently "forgets" it exists on the next devenv restart (confirmed empirically: `.tspproj` file present on disk, zero references to it in `.sln`). `AddLibraryProjectAsync` now calls `Solution.SaveAs` immediately after `tcProject.Save()` for exactly this reason. **This same call can also crash devenv outright** (confirmed twice via a real `System.AccessViolationException`/`0xc0000005` in `ITcSmTreeItem.CreateChild`, logged to the Windows Application event log) - not yet root-caused, but a clean retry after devenv's own crash-auto-restart has worked both times observed so far.
18. **TwinCAT PLC debugging (breakpoints, stepping, watch expressions) does NOT integrate with `EnvDTE.Debugger` at all** - confirmed empirically: `Debugger.CurrentMode` stayed `dbgDesignMode` even with a PLC project fully online, downloaded, and running (`PLC_APPSTATE_RUN`). No login flag changes this (`PLC_LOGIN_FLAGS` has no debug-related member), and reflection over the entire `TCatSysManagerLib` assembly found zero interfaces with "Debug"/"Breakpoint"/"Monitor" in the name. This means real IEC breakpoints/stepping are not reachable through any COM surface found so far - same category of gap as ADS variable "force" (Critical Rule 7). **What IS reachable**: the embedded CODESYS PLC editor's own command dispatch, `Beckhoff.TwinCAT.VS.CDSCmdTools.Exec(Guid)` (public static method, invokable via reflection, in `TwinCATPlcControlx64.dll`) - this is the real path a gutter-click toggle-breakpoint or menu Login/Logout/Step takes, and `/debugger/cds-command` drives it directly, in-process, no GUI needed. Full wire-protocol-level online-debug research (breakpoint addressing, session GUIDs) is documented separately in `docs/TC_ONLINE_DEBUG_PROTOCOL.md` - not superseded by this, but this CDS-command path is far more practical for anything the IDE itself already knows how to do (login/logout/step/toggle-breakpoint) without needing the raw wire protocol at all. **What already covers the watch/output need**: `GET/PUT /ads/variables` already gives live read/write of any variable by path (poll it repeatedly for a "watch"-like view, no formal Watch-window UI needed); `GET /solution/output`/`GET /solution/output/{name}` reads any VS Output window pane as plain text, including a dedicated "TwinCAT" pane carrying runtime diagnostics (e.g. PLC login timeouts, TMC file warnings, and PLC crash exception details - see Critical Rule 27) not visible via the Error List (`/solution/build`) or anywhere else in this API.
19. **A running UmRT usermode runtime crash (or, separately, repeated forceful `devenv` process kills mid-session) can wedge the shared AMS router system-wide** - confirmed live: `POST /ads/connection` starts failing with `AdsErrorCode 6` ("Target port could not be found") or `AdsErrorCode 4`/`7` for EVERY AmsNetId tried, including the local engineering machine's own, not just the affected runtime. **Restarting the runtime process alone does NOT fix it** - this needs a TwinCAT system service restart or a full reboot; do not attempt that autonomously without the user's say-so, since it's shared machine infrastructure. If `POST /ads/connection` fails for both your target AND the local machine's own AmsNetId, stop retrying and treat it as a router-level problem, not a project-level one. Before concluding the router is wedged, though, rule out mundane causes first (Critical Rule 21's Priority=0 gotcha caused what looked exactly like this symptom at least once) - and prefer closing devenv gracefully over force-killing it when you have the choice, since force-kills are themselves a suspected trigger.
20. **`/solution/build`'s `errors` count is not fully trustworthy on its own** - it only reads `EnvDTE.ToolWindows.ErrorList.ErrorItems`, which can fail to capture errors from a build platform that isn't the "active" one (e.g. a project configured to also build for `TwinCAT OS (x64)` alongside the intended `TwinCAT RT (x64)`) even though those errors DO print to the raw Build output pane. **Always cross-check `GET /solution/output/Build`'s raw text** before trusting a build result anything downstream depends on (a redeploy, a "let's move on" decision) - this was caught by the user explicitly asking to check the pane, not by the API's own reported count.
21. **TwinCAT has TWO distinct kinds of task, reverse-engineered live this session (not documented anywhere obvious) - conflating them is the single most likely reason a PLC application will silently never execute**:
    - The **real real-time task** (`TREEITEMTYPE_TASK=1`) lives under the solution/main-project-scoped `TIRT` root (System > Real-Time > Tasks) - THIS is what TwinCAT actually schedules. `Priority` genuinely is constrained to 1-61 - **0 is accepted silently at creation and the task then never cycles, with no error anywhere in the pipeline** (build succeeds, download succeeds, login reports success, `isLoggedIn` even reads `true`) - this was the actual root cause of an extended debugging session initially misattributed to AMS router wedging (Critical Rule 19) and to a Task-move corrupting its execution binding.
    - The **PLC-side "referenced task"** (`TREEITEMTYPE_PLCTASK=621`, lives under a specific PLC project like any POU/GVL, what `POST .../tasks` on a `{plcName}` creates) is just a named pointer. Its own `Priority`/`CycleTime` are cosmetic/unused for scheduling - what actually matters is being linked to a real-time task via the real typed COM property `ITcPlcTaskReference.LinkedTask` (`string`, e.g. `"TIRT^PlcTask"`) - set by `PUT .../tasks/{name}/assignment`. **A referenced task with no assignment (or pointing at nothing) downloads and logs in without any error and just never cycles** - the same class of silent failure as Priority=0 above, and just as easy to misdiagnose as an infrastructure problem instead of a config one.
    - `CreateChild(name, 621, "", vInfo)` for a referenced task validates the new item's OWN NAME against `TIRT`'s children regardless of `vInfo` (`"No task 'X' found in Realtime-Settings!"` if it doesn't match) - `vInfo` does NOT let you decouple the reference's display name from its target the way template-name `vInfo` works elsewhere in this codebase. `POST .../tasks` works around this by creating with the target's name first (auto-populates `LinkedTask`), then renaming to the requested name - confirmed live that renaming preserves the `LinkedTask` assignment.
22. **The real-time task's `CycleTime` (`_ITcTaskDef.CycleTime`, a `uint`) is raw ticks, not a time value** - confirmed live by cross-checking the IDE's own "Cycle ticks"/ms display against `ProduceXml` output on a hand-created task (47 UI "Cycle ticks" = "47.000 ms" = raw property value 470000, i.e. 1 raw tick = 100ns). **The actual real-world duration is `CycleTicks x BaseTime`, where BaseTime is a separate, system-wide setting (System > Real-Time > "Base Time" dropdown, e.g. 1ms) this API has no visibility into** - the REST DTOs deliberately expose `cycleTicks` directly rather than computing/reporting a misleading time value. Valid range is 1-65535 ticks (confirmed via the IDE's own validation, NOT the ~6.5ms-if-100ns-per-tick range an earlier draft of this code incorrectly assumed and enforced).
23. **Interface Method/Property children use a DIFFERENT subtype family than the equivalent POU children**, despite looking identical from the outside - confirmed via direct `TCatSysManagerLib.TREEITEMTYPES` enum reflection (not guessing): `TREEITEMTYPE_PLCITFMETH=610` (vs `PLCMETHOD=609` for a POU), `PLCITFPROP=612` (vs `PLCPROP=611`), `PLCITFPROPGET=654`/`PLCITFPROPSET=655` (vs `PLCPROPGET=613`/`PLCPROPSET=614` for a POU - note these aren't even sequential/nearby, so don't assume a pattern). Reusing the POU-context values against an Interface container fails with `"Cannot insert 'X' below 'Y'"`.
24. **Relocating a Task out of its canonical position via the generic Export/Import-based `.../tree/{name}/move` can silently break its real-time execution binding**, even though the move looks entirely successful (correct call-list, clean compile, correct `ItemSubTypeName`, ADS symbols readable/writable) - diagnosed by writing a value directly to a running FB instance's field and finding it unchanged after thousands of expected cycles, despite `appState: PLC_APPSTATE_RUN`. `ImportChild`'s explicit-name parameter is REJECTED specifically for Task-type items ("Cannot change imported child name. Please specify without childName!"), confirming Tasks have special container/identity semantics ordinary tree items don't - and importing into a plain user folder can add an unwanted wrapper folder level too (task lands one level deeper than intended). **Recommendation: don't move the default `PlcTask` out of the PLC project root at all**, even when applying a folder-reorganization convention to everything else in the project.
25. **Some `/debugger/dte-command` invocations pop a modal dialog and hang the REST call until it's dismissed** - confirmed for at least one File-menu Open variant (opening an unregistered file type triggers an "Open With" picker; `PLC.LoadCoreDump`/`File.OpenTcTaskDump` pop their own multi-step dialog sequence - a "application changed since last download" Yes/No confirm, then a real file-open dialog). **Safe, general recovery technique for a devenv hang like this**: enumerate top-level windows for the devenv PID via `user32.dll` `EnumWindows`/`GetWindowText` (P/Invoke from PowerShell via inline `Add-Type`), look for a dialog-like title distinct from the main IDE window's fuller title, and `PostMessage(hwnd, WM_CLOSE=0x0010, 0, 0)` to dismiss it (equivalent to clicking X/Cancel, safe/non-destructive). **For WPF-hosted dialogs** (class name like `HwndWrapper[DefaultDomain;...]`, common for modern VS message boxes) plain Win32 child-window enumeration finds nothing useful - use `System.Windows.Automation` (UI Automation, `Add-Type -AssemblyName UIAutomationClient,UIAutomationTypes`) instead to read the actual text content and invoke buttons (`AutomationElement.FromHandle`, `TreeWalker`, `InvokePattern`) - this works even when raw `EnumChildWindows` returns nothing.
26. **When a `CreateChild` subtype or `vInfo` value is unknown, check `TCatSysManagerLib.TREEITEMTYPES`/other relevant types directly via runtime reflection BEFORE guessing or reaching for `ilspycmd` decompilation** - fastest method found: `Add-Type -Path <TCatSysManagerLib.dll path>; $asm = [Reflection.Assembly]::LoadFrom(<path>); $enumType = $asm.GetTypes() | Where-Object Name -eq 'TREEITEMTYPES'; [Enum]::GetNames($enumType)` (find the DLL via `Get-ChildItem C:\Windows\Microsoft.NET\assembly -Recurse -Filter TCatSysManagerLib.dll` - it lives in the GAC, not this extension's own `bin\Debug`). This resolved the Interface-method subtype question (Critical Rule 23) and the Task subtype question (Critical Rule 21) correctly on the first try, in both cases correcting an initial wrong guess that had been carried over by analogy from a different subtype family. The same technique works for any other TwinCAT-shipped managed assembly loaded in-process (e.g. `TwinCATPlcControlx64.dll` for `Beckhoff.TwinCAT.VS.CDSCmdGuids`, Critical Rule 18) - always cheaper than a decompile-and-read cycle.
27. **A PLC application crash leaves a genuine forensic artifact on disk**: `<runtime folder>\3.1\Boot\Plc\CoreDump\Port_851.<id>\Plc\Port_851.<id>.core` (small, proprietary binary format, no readable header) alongside a `.tizip`/`.tszip` archive bundling the matching compiled boot project for symbol correlation. The crash itself is also logged with useful detail to `GET /solution/output/TwinCAT` (exception code, faulting task name, `RBP`/`RIP`/`RSP`, `Area`/`Offset`) immediately after it happens - check there first, it's often enough on its own. Loading the `.core` file for full IDE-side symbol correlation (matching what `File > Open > Open Project from Target...`/`Open TcTaskDump` do manually) is possible via `/debugger/dte-command` (`PLC.LoadCoreDump` with no `args` - see Critical Rule 25 for the dialog sequence it triggers) but **only works if the project hasn't been rebuilt since the crash** - correlating against a rebuilt project fails with `"There is no compile information available for the application '<name>'"`. If you need the crash's exact source line and the project has since changed, the dump is effectively expired - work from the `GET /solution/output/TwinCAT` message and manual code review instead of continuing to retry dump-loading. **Also observed once, not fully explained**: after a dump-loading attempt, one PLC project's build stopped resolving entirely (`"Unspecific error '<name>' - 0x98510001"`, all its types reported "Unknown type" from a referencing project) and persisted across multiple devenv restarts, a `.vs` cache clear, and removing `_Boot`/`_CompileInfo` build artifacts - root cause not found before the session ended. If this happens, suspect the dump-loading flow specifically as the trigger rather than anything else recently changed.
