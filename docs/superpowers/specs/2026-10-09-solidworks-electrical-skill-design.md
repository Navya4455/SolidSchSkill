# SOLIDWORKS Electrical drawing skill — design

Date: 2026-10-09. Status: **draft for review**. Repo: `C:\Automation\SolidSchSkill`.

## 1. Goal

Let Claude produce and maintain electrical drawings as **native SOLIDWORKS Electrical projects** —
real components, wires, cross-references, terminal strips and reports — through an agent + skill +
prompts, instead of the one-off PDF that the WetBlasting schematic was (`C:\Automation\WetBlasting`:
Python → HTML → PDF/DXF, 31 pages, which nobody can edit as a schematic).

The output keeps the WetBlasting **document format** — AddLife title block, A3 sheets with columns
0–9, cover and front matter, `=WB +CAB` designation, the same sheet order — but lives in
SOLIDWORKS Electrical, where both the user and Claude keep editing it.

**First target:** rebuild the **full 31-sheet WetBlasting set** as a native project (§10.4).

## 2. Decisions taken in brainstorming

| # | Decision | Consequence |
|---|---|---|
| D1 | **The Electrical project is the master.** No design file is the source of truth. | Every edit starts by reading the live project. Claude edits incrementally; the user edits in Electrical freely. |
| D2 | **Live session.** Electrical is open while Claude works. | The bridge is an **add-in inside Electrical**, not a background process. |
| D3 | Inputs: **prompts, existing schematic PDFs, controller/I-O config (`.acbind`, terminal xlsx), datasheets/manuals.** | The skill has an intake stage per input type, all converging on one design brief (§9). |
| D4 | First target: the **full WetBlasting set**. | Acceptance in §10.4. Built in phases (§11) so problems surface early. |
| D5 | **The skill builds the library** — device schematic symbols and macros; the 2D design is much of the work. | §7. Library items go into one `ADDLIFE` library. |
| D6 | **Part rule:** every device gets a part, always linked to its symbol and its terminal labels; **manufacturer data only when required**. | §7.3. Generic placeholder manufacturer `ADDLIFE` otherwise. |
| D7 | **Approach 1:** add-in + JSON operation batches + `swe` CLI. | §4–§5. An MCP wrapper can be added later on the same endpoint. |
| D8 | **The audit log is not state.** State is always the Electrical database. | §5.6. The log is write-only history for humans. |
| D9 | **Read two ways, write one way.** Writes only through the add-in; bulk reads may use read-only SQL. | §6. |
| D10 | **EtherCAT I/O terminals are Electrical PLC modules** (channels with addresses), drives/PSU/HMI/switch are black boxes. | I/O is checkable against `.acbind`, channel by channel (§10.3). |
| D11 | **Automation lives in the template.** Claude writes source data only; dates, page counts, contents, component/wire lists are derived by Electrical. | §8. A template upgrade precedes the WetBlasting build. Manual edits by the user keep every derived page correct. |
| D12 | **Pinned to Electrical 2026 SP4.1** (upgraded 2026-10-09). 2027 not until generally available with published API notes. | §3, §12. |

## 3. Environment (pinned)

| Item | Value (verified 2026-10-09) |
|---|---|
| SOLIDWORKS Electrical | **2026 SP4.1, 34.41.1011** — `C:\Program Files\SOLIDWORKS Corp\SOLIDWORKS Electrical` |
| SOLIDWORKS | 2026 SP4.1, 34.141.1011 (must stay on the same version as Electrical) |
| API | COM, `ewapix.dll` 2026.4.1.1011; interop `bin\Interop.x64.EwAPI.dll` (718 types; SP0 had 693). Use the **version-independent ProgID** `EwAPI.EwInteropFactoryX` (both `…2026.0` and `…2026.4` are registered). |
| API licence | **The API needs a licence code from the reseller.** The official 2026 add-in sample calls `getEwApplication("License Code(*)")` with the note *"License Code(*): Contact your reseller to get your license code."* `EwErrorCode` has `EW_INVALID_LICENSE` (39) and `EW_LICENSE_WITHOUT_API_OPTION` (40), so the Electrical licence must also include the API option. The code is a secret: it lives in `%LOCALAPPDATA%\AddLife\swe-bridge\license.txt`, never in git. |
| Database | SQL Server 2022 Express RTM 16.0.1000.6, instance `.\TEW_SQLEXPRESS`, Windows auth. App DBs `tew_app_data`, `tew_app_macro`, `tew_app_project`, `tew_catalog`, `tew_classification` keep their tables in schema **`tew`**; each project DB `tew_project_data_<pro_directory>` keeps its tables in schema **`dbo`**; `tew_version` is in `dbo` everywhere. **Project-data schema 292 (`dbo.tew_version.ver_projectdata`), app-project schema 32 (`tew_app_project.dbo.tew_version.ver_projects`).** |
| Data folder | `C:\ProgramData\SOLIDWORKS Electrical` — `Projects\<id>\Drawings\*.ewg`, `ProjectTemplate\ADDLIFE Template.proj.tewzip`, `TitleBlock`, `XmlConfig\AutomateDrawing` |
| Library (counted 2026-10-05) | ~22,200 symbols, 4,233 manufacturer parts (3,781 EATON, 23 Beckhoff, **0 Inovance**), 66 title blocks |
| Licence | Standalone serial-number activation (not SolidNetWork, not 3DEXPERIENCE) |
| Tooling present | Python 3.14 (`ezdxf`, PyMuPDF, `pypdf`), VS 2022 Build Tools (MSBuild), git, ODBC Driver 17 for SQL Server |
| Tooling to install | **.NET SDK** (SDK-style project targeting `net48` via `Microsoft.NETFramework.ReferenceAssemblies`, so no developer pack), Python `pyodbc`, `pytest` |

## 4. Architecture

```
┌───────────── Claude Code ─────────────┐
│  skill: SKILL.md + references         │
└──────┬────────────────────────────────┘
       │ JSON op batches · DXF symbol geometry
       ▼
  swe  (Python package + CLI)        ── read-only SQL ──►  TEW_SQLEXPRESS
       │  HTTP 127.0.0.1 + token
       ▼
┌── SOLIDWORKS Electrical (user's open session) ─┐
│  AddLife bridge add-in (C#, net48)              │
│   queue → Electrical UI thread → EwAPI (COM)    │
│        ▼                                        │
│  project DB + .ewg drawings + library           │
└─────────────────────────────────────────────────┘
       ▲ results JSON · sheet PNGs · PDF
```

| Unit | Responsibility | Depends on |
|---|---|---|
| `addin/` — **bridge add-in** | Translate JSON operations into EwAPI calls on Electrical's UI thread; snapshots; session file. **No design logic.** | EwAPI interop |
| `swe/` — **Python package + CLI** | Batch client; read-only SQL layer with schema guard; **layout helpers** (grid, rails, device placement, wire routes between named terminals); **symbol builder** (ezdxf); checks (lint, netlist, I/O); renders | add-in endpoint, SQL |
| `skill/` — **the skill** | `SKILL.md` workflows and hard rules; references (conventions, symbol style, ops, per-input guides, gotchas). Where the design judgement lives. | `swe` CLI |
| `work/<project>/` | Scratch: extracted inputs, design brief, per-sheet scripts, renders, the ops log. **Never state.** Git-ignored. | — |

The add-in is kept small and stable because rebuilding it means restarting Electrical; everything that
iterates (layout, symbol drawing, conventions, checks) is Python and Markdown.

The skill is installed for Claude Code as `solidworks-electrical` by a directory junction from
`%USERPROFILE%\.claude\skills\solidworks-electrical` to `skill/`.

## 5. The bridge add-in

### 5.1 Loading and connection

- A COM-visible, x64 class implementing `EwAPI.EwAddIn`. In `connectToEwAPI(factory)` it obtains the
  application through `IEwInteropFactoryX.getEwApplication(<licence code>)`.
- Registration follows the documented procedure: `regasm /codebase` (administrator) runs the class's
  `[ComRegisterFunction]`, which writes `HKLM\SOFTWARE\SolidWorks\SOLIDWORKS Electrical\AddIns\{GUID}`
  (`Title`, `Description`, `Path`) and `HKCU\SOFTWARE\SolidWorks\SOLIDWORKS Electrical\AddIns\{GUID}`
  (`StartUp=1`). Electrical loads it at startup. Whether this works with our licence code is spike S1.
- It starts an HTTP listener bound to **127.0.0.1 only** on a free port, and writes
  `%LOCALAPPDATA%\AddLife\swe-bridge\session.json` = `{port, token, pid, started_at, ewapi_version}`.
  The token is random per start and must be sent as `X-SWE-Token`. `swe` reads the file.
- Requests are queued and executed **one at a time on Electrical's UI thread** (captured
  `SynchronizationContext` from a hidden control created at connect). The EwAPI is single-threaded;
  this also keeps the add-in from colliding with the user's own actions. Long batches show Electrical's
  progress dialog.

### 5.2 Protocol

- `GET /health` → versions, open project, queue depth.
- `POST /batch` with `{session, ops: [{op, args}], stop_on_error: true}` → `{results: [{ok, ids, warnings,
  error: {code, name, message, last_error}}], elapsed_ms}`. `code`/`name` are the `EwErrorCode`;
  `last_error` is `IEwAPIX.getUTLastError()`.
- `session` is a client-chosen id for one editing session (one CLI invocation chain).

### 5.3 Identity — updates, not duplicates

Named objects carry an identity; a create on an existing identity **updates** it:

| Object | Identity |
|---|---|
| Component | tag (project-wide) |
| Location / function | tag |
| Book | tag |
| Sheet | book + sheet position |
| Library symbol | name |
| Part | manufacturer + reference |
| Macro | name |

On-sheet geometry (placed symbols, wire lines, texts) has no natural identity: edits to it address the
**ids read back with `sheet_contents`**. `clear_sheet` exists only for workflow A before a sheet has been
handed over (§9.1).

### 5.4 Operations

| Group | Operations |
|---|---|
| Session | `health`, `open_project`, `current_project`, `create_project_from_template`, `set_project_properties`, `snapshot`, `archive` (`.tewzip`) |
| Read | `project_tree` (books/folders/sheets with type, title, position), `sheet_contents` (symbols, connection points, lines, texts with ids and coordinates), `components`, `wires`, `locations`, `functions`, `title_block_grid` (columns/rows from `IEwTitleBlockX`). Library search (symbols, parts, macros, title blocks) is a bulk read and is served by `swe db search-*` over SQL (§6), not by the add-in. |
| Edit project | `create_book`, `create_sheet` / `move_sheet` / `delete_sheet` (type, title block, description), `create_location`, `create_function`, `upsert_component` (tag, description, location, function, part, user data), `place_symbol`, `draw_wire`, `place_text`, `insert_macro`, `move`, `delete`, `clear_sheet` |
| Library | `create_symbol` (DXF graphics, connection points, circuits, root mark, class, library), `update_symbol_graphics`, `create_part` (circuits, terminal labels, symbol link, manufacturer data, PLC channels), `create_macro`, `set_review_flag` |
| Actions | `number_wires`, `generate_terminal_strips`, `update_reports`, `render_sheet_png` (`IEwSaveDWGImageX`), `render_symbol_png`, `export_pdf` |

`skill/references/ops.md` is **generated from the add-in's operation schema** (the add-in serves it at
`GET /schema`) so the documentation cannot drift from the code.

### 5.5 Safety

- The **first mutating batch of a session takes an Electrical project snapshot** named
  `claude-<yyyymmdd-hhmm>-<summary>`; the change report names it so the user can roll back from inside
  Electrical.
- Library writes go only to the `ADDLIFE` library. **Standard library items are never modified.**
- The add-in refuses mutating operations while the target project is open for write by another user
  (`IEwProjectX.isOpenByAnother`).
- **Mutation allowlist:** mutating operations run only on projects named in
  `%LOCALAPPDATA%\AddLife\swe-bridge\config.json` → `mutation_allowlist` (Phase 0/1: `SPIKE-*`, `LIBTEST*`).
  Every other project — including the user's existing ones — is read-only to the bridge until it is
  added on purpose.
- Every batch names its target project; the add-in refuses the batch if Electrical's current project is
  a different one, so a project switched in the UI mid-session cannot receive another project's edits.

### 5.6 Ops log (D8)

Every batch and its result are appended to `work/<project>/ops-log.jsonl`. It is a **write-only history
for humans** ("what did Claude change on Tuesday", debugging). **Nothing reads it to decide anything** —
not the skill, not `swe`. Current state is always read from Electrical.

## 6. Read paths (D9)

| Purpose | Path |
|---|---|
| Any change | **Add-in only.** Never write SQL — it bypasses cross-references, equipotential recalculation, numbering, locks and `.ewg` sync, and corrupts the project. |
| Reading the sheet/component about to be changed | Add-in (`sheet_contents`, `components`) — sees exactly what Electrical shows, including unsaved drawing state. |
| Bulk reads: whole-project netlist, verification, library search, report cross-checks, review list | **Read-only SQL** through `swe db …` |

The SQL layer:

- issues `SELECT` only (enforced in code; any other statement is a programming error), default
  `READ COMMITTED`, no `NOLOCK`, short queries;
- **refuses to run on an unknown schema**: on connect it checks `dbo.tew_version` (project-data 292,
  app-project 32) and the presence of every table and column it uses; a mismatch fails loudly with the
  observed versions, it never guesses;
- knows the limit of the database: the logical model (components, terminals, symbols and their points,
  wires, equipotentials, cables, sheets, texts) is in SQL; free graphics, title-block fill and drawn
  appearance are only in the `.ewg` files.

## 7. Library building

### 7.1 Where things go

- One library, **`ADDLIFE`**, filed under Electrical's standard classification nodes. *Name pending decision
  D-F:* an `Addlife` library with 196 human-made symbols already exists (2026-10-09) and the names would
  collide; the proposal is a separate `ADDLIFE_AUTO` library so Claude's items can be reviewed and rolled
  back on their own. Wherever this spec says `ADDLIFE` library, read the name D-F settles on.
- **Reuse first:** before creating anything Claude searches the existing library (`swe db search-symbols`,
  `search-parts`). Generic IEC symbols (contacts, coils, MCBs, motors, fuses, lamps…) are always reused.

### 7.2 Device pipeline

1. **Device sheet** (in `work/library/<model>.yaml`, a build input, not state): terminal groups (`R S T
   PE`, `U V W PE`, `J14 STO`, `CN1 relay`, `CN2 control`…), each terminal's label, circuit type and
   mnemonic, the side of the symbol each group faces, and the datasheet page each fact came from. To
   change an existing item later, Claude reads it back from Electrical first (connection points,
   circuits, part terminals, a render) and does not trust an old device sheet.
2. **Symbol graphics:** the `swe` symbol builder draws the 2D by the **AddLife symbol style guide**
   (`skill/references/symbol-style.md`, written in phase 2): solid device outline, a dashed sub-box with a
   heading per terminal group, terminal circles on the edge with labels — the visual language of the
   WetBlasting drive sheets. Grid, pitch, text heights and layers are **measured from existing standard
   symbols in the library**, not guessed. Connection-point coordinates are produced by the **same code
   that draws the geometry**, so they cannot drift. Electrical's attribute definitions are taken from an
   existing standard black-box symbol and merged into the generated DXF.
3. **Create:** `create_symbol` (name, type, `ADDLIFE`, class, root mark, circuits, connection points →
   `insertFromDwg`), then `create_part` (§7.3).
4. **Prove:** render the symbol to PNG and view it; read the points back and compare with the device
   sheet; place it in the scratch project **`LIBTEST`**, attach the part, draw a wire to a terminal. Pass
   = terminal labels shown on the drawing come from the part, and Electrical detects the wire on the
   point.
5. **Review flag in Electrical, not in a file:** each new symbol and part carries user data
   `ADL_REVIEW=pending`; `swe db review-list` lists them; the user approves by clearing the flag.

### 7.3 Part rule (D6)

| Layer | Rule |
|---|---|
| **Part** in `ADDLIFE` | Always created for every device. Linked to its schematic symbol; carries its circuits and **terminal labels**, so labels on the drawing always come from the part, never from text typed on the symbol. |
| **Manufacturer data** (manufacturer, order reference, article no., supplier, datasheet) | Electrical requires manufacturer + reference on every part, so by default a generic placeholder: manufacturer `ADDLIFE`, reference `ADL-<CLASS>-<KEY>` (e.g. `ADL-VFD-3PH-1.5KW`). The real manufacturer data is filled **only when required**: the source names a specific model (datasheet, source drawing, `.acbind` slave list), or the user asks for a procurement BOM. |
| **Generic symbols** (single contacts, arrows, potentials, notes) | No part of their own; a contact belongs to its relay's component, which has the part. |

### 7.4 PLC modules (D10)

EtherCAT I/O terminals (EK1100 coupler, EL1008/EL2004/EL5151…) are created as Electrical **PLC parts**
(`kManufacturerPartPlcModule` with per-channel circuits, channel group and channel address), so each
channel exists in Electrical's input/output table (`IEwProjectInputOutputManagerX`) with its address
and function. Drives, PSU, HMI and the Ethernet switch are black-box devices.

### 7.5 Macros

A circuit is saved as an `ADDLIFE` macro **only after it has been drawn and verified on a real sheet**
(e.g. the MD520 drive page, an 8-channel DI card page) — with tag variables and a rendered preview.
Never designed speculatively.

### 7.6 Naming

| Item | Pattern | Example |
|---|---|---|
| Symbol | `ADL_<CLASS>_<MODEL>` | `ADL_VFD_MD520-4T1.5B` |
| Part | real model when manufacturer data is required, else `ADL-<CLASS>-<KEY>` | `MD520-4T1.5B` / `ADL-PSU-24V-10A` |
| Macro | `ADL_M_<CIRCUIT>` | `ADL_M_VFD_STO_BRAKE` |

## 8. Automated template (D11)

**Hard rule:** Claude never types a value Electrical can derive. It writes **source data only** —
project properties, sheet descriptions, components with descriptions and parts, locations, functions,
user data. Everything else is an attribute, a formula or a report. Lint (§10.2) flags static text that
duplicates a project property, a date or a page number.

**`ADDLIFE Template` v2** — upgraded before the WetBlasting build (phase 3):

| Item | Driven by |
|---|---|
| Cover page: company, project name/description, drawing no., type, installation site, responsible, created on, **edited on**, number of sheets | Cover-page title-block attributes bound to project properties and dates |
| Title block on every sheet: sheet title, `=WB +CAB`, page / of, customer, machine type, order no., dates, creator | Title-block attributes bound to project, book and sheet variables |
| Contents | Drawing-list report |
| Function group overview | Function-list report |
| Component list | BOM / component report in the AddLife column layout |
| Wire list | Wire report |
| Terminal schedule | Terminal-strip drawings or terminal report |
| Cross-references (`→ 3.8`) | Electrical cross-references |
| Tags, new wire numbers | Project formulas |

**Static by nature (stated, not hidden):** the Conventions sheet (prose); the **Symbol table** (reports
cannot show graphics — Claude generates it from the symbols actually used; lint warns when a used symbol
is missing from it); VFD parameter settings (source data).

**Refresh:** reports and terminal drawings refresh with Electrical's update command, and on PDF export
where the "generate automated drawings" option allows (spike S7). They do not refresh on every edit —
that is Electrical's model, not ours to change.

**Wire numbers:** on an existing machine the numbers printed on the physical wires are **source data**,
entered as fixed numbers. Every wire added later — by the user or by Claude — is numbered by the
project formula; Electrical keeps fixed and formula numbers side by side.

## 9. The skill

### 9.1 Workflows (`SKILL.md`)

**A. New project**

1. **Intake** — read every input; write the **design brief** in `work/<project>/`: device list (tag,
   function, model, source), sheet plan (order, titles), connections per sheet, I/O map from the
   controller config, and **open questions** (Claude asks; it does not write `TBC` and move on).
   ⏸ **The user approves the brief and sheet plan before anything is drawn.**
2. **Library pass** — reuse or build every device (§7). Pending review does not block drawing.
3. **Skeleton** — project from `ADDLIFE Template` v2; project properties; books and sheets in AddLife
   order; locations (`=WB`, `+CAB`) and functions.
4. **Sheets, one at a time** — a short disposable per-sheet script using the layout helpers emits the
   batch → apply → **render to PNG and view** → fix → next. Order: power → control/safety → I/O →
   drives → the rest.
5. **Data passes** — number wires, generate terminal strips, update reports.
6. **Verify** (§10) → export PDF, archive `.tewzip`.

**B. Change an existing project** (the everyday case) — read the live project → snapshot (automatic,
§5.5) → plan the smallest change → apply → re-render affected sheets → re-run numbering/reports if wiring
changed → verify (netlist diff) → report what changed and the snapshot name.

**C. Library only** — e.g. "add the EL3062 from this datasheet": §7.2 on its own.

### 9.2 Hard rules (top of `SKILL.md`)

1. State comes from the Electrical database. Read before you write. Never trust `work/` for state.
2. Never write SQL.
3. Every editing session is covered by a snapshot.
4. Never type a derivable value (§8).
5. A sheet is not done until its render has been viewed.
6. Library writes only to `ADDLIFE`; standard items are never modified.
7. Unknowns become questions to the user, not placeholders.

### 9.3 References

| File | Content |
|---|---|
| `references/conventions.md` | Taken from the WetBlasting set and the Ecoclean reference: sheet structure (front matter, schematics, generated reports, appendix), A3 grid 0–9 and `sheet.column` cross-references, tags carrying the sheet number (`-13A1`, `-13F1`, `-13M1`, `-X1:8`), `=WB +CAB`, potentials (`2L1–2L3`, `24 V P4`, `0 V`), wire-numbering scheme, notes block at the sheet foot |
| `references/symbol-style.md` | AddLife symbol style guide (§7.2) |
| `references/ops.md` | Generated from the add-in schema |
| `references/inputs-*.md` | One guide each: schematic PDFs, `.acbind`/xlsx, datasheets, prompts |
| `references/gotchas.md` | Grows as problems are found |

## 10. Verification

### 10.1 Code tests (every change)

- `swe` (pytest): grid and coordinate maths; wire-route geometry; symbol builder — every connection point
  lies on a drawn terminal and the DXF passes `ezdxf` audit; operation batches validate against the add-in
  schema.
- Add-in: pure-logic tests (parsing, validation, identity matching) plus **`swe selftest`**, which runs
  every operation against `LIBTEST`, reads each result back, compares and cleans up. Must pass after every
  add-in build and after every Electrical update.

### 10.2 Per sheet (every time a sheet changes)

- Render to PNG and view it — mandatory.
- Lint from `sheet_contents`: nothing outside the frame; no overlapping symbols; no wire end that is not on
  a connection point; no unconnected point that should be connected; no text over a symbol; no static
  text duplicating a derivable value (§8).

### 10.3 Project (read-only SQL; after data passes and after every edit session)

- **Integrity:** no duplicate tags; no component without a symbol or symbol without a component; no wire
  without a number; every `ADDLIFE` part linked to its symbol with terminal-label count = connection
  points.
- **I/O vs controller:** each PLC channel's address and function compared with the `.acbind` binding
  **one channel at a time, by position** — a multiset comparison cannot see a swap (the WetBlasting
  `-7A1` lesson).
- **Netlist diff for edits:** netlist at session start vs end, read live from Electrical (the "before" is
  a throwaway temp file). The change report must contain only intended changes.

### 10.4 WetBlasting acceptance

Claude extracts a **truth set** from the existing WetBlasting sources (sheet titles; tags with
description and model; terminal-to-terminal connections; wire numbers; potentials; I/O channels) and
compares it with the generated project:

| Check | Pass |
|---|---|
| Sheets | All 31 in the same order with the same titles. Generated reports (wire list, component list, terminal schedule) may have a **different page count**, **same content**. |
| Components | Every tag present, description and model match |
| Connections | Every terminal-to-terminal connection present, none extra |
| Wire numbers | Match exactly (fixed numbers from the source, §8) |
| I/O | Matches the corrected schematic channel by channel; the `.acbind` channel 6/8 mismatch is **reported as a finding**, not silently resolved |
| Open items | WetBlasting's `TBC`s and the `-13M1` rating conflict carried over as flagged items |
| Derived pages | Cover, title blocks, contents, component list, wire list, terminal schedule all derived (§8), none typed |
| Visual | Every sheet rendered and reviewed by Claude, then **user sign-off side by side with the original PDF** (PyMuPDF renders the original pages). Layout may differ; content may not. |
| **Manual-edit test** | The **user**, with no agent involved, adds a component, renames a sheet, deletes a wire and inserts a sheet, then runs update. Pass = cover edited date and sheet count follow, contents shows the new title, component list has the new component, wire list drops the deleted wire, page numbering shifts. |

## 11. Phases

Each phase ends usable and gets **its own implementation plan, written after the previous phase**, since
the spike can change what follows.

| Phase | Delivers | Done when |
|---|---|---|
| **0. Spike** | Answers to S1–S10 with a chosen fallback where one fails; interop surface dump `docs/reference/ewapi-2026.4.1.1011-types.txt`. The add-in **transport skeleton** (registration, session file, listener, UI dispatcher, batch runner, `health`) is built properly here because every probe needs it, and is kept; only the probe operations are throwaway. | Findings written to `docs/superpowers/specs/` as an amendment; this design and the Phase 1 plan updated where needed |
| **1. Bridge** | Add-in operations, `swe` CLI, read-only SQL layer with schema guard, `swe selftest`, snapshots, renders, **backups (§14)** | Selftest passes on `LIBTEST`; one sheet built from a batch and rendered; a backup restored |
| **2. Library pipeline** | Symbol builder, symbol style guide, part creation, PLC modules, review flags, macro capture | MD520, EK1100 + one EL1008, and the PSU built and proven in `LIBTEST` |
| **3. Template v2** | `ADDLIFE Template` v2 (§8) | Manual-edit test passes on a three-sheet test project |
| **4. Skill + WetBlasting** | `SKILL.md` + references; the full 31-sheet project | §10.4 passes; user signs off |

### Spike questions (phase 0)

| # | Question | Fallback |
|---|---|---|
| S1 | Does the documented registration (§5.1) load our add-in in 2026 SP4.1, and does `getEwApplication` succeed with the reseller's licence code? | **None without a valid code** — background mode also goes through `getEwApplication(key)`. Without the code the project is blocked; this is decision D-A in the Phase 0/1 plan. |
| S2 | Does a placed symbol plus a drawn line create a connected wire (equipotential, wire, terminal)? | Small connection macros via `insertMacroAt`, or `runCommand` |
| S3 | Can free text and graphics be placed on a sheet (multilingual text, `runCommand`)? | Text-only notes symbols |
| S4 | Does `insertFromDwg` keep attributes; do API-added connection points work? | Duplicate a standard black-box symbol and swap its graphics |
| S5 | Can PLC-module parts with channel addresses be created and populated through the API? | Black boxes, I/O checked through user data |
| S6 | Can macros be created through the API? | Claude prepares the sheet, the user saves the macro with one click |
| S7 | Can title-block attributes cover edited date and sheet count? Can reports be regenerated through the API or on export? | `update_reports` operation run by the skill and on demand |
| S8 | Do PNG render of an open drawing, snapshots via API, a localhost listener without admin, and execution on the UI thread all work? | Local workaround per item |
| S9 | Does a SQL netlist query reproduce the API's view of the same project? | API-only reads, accepting the frozen window |
| S10 | Does a project archived through the API (`IEwProjectManagerX.archive`) unarchive into an identical project (same netlist), and does an environment archive (`IEwArchiveEnvironmentX`) capture libraries and templates? | Manual archive from Electrical's UI on a schedule; backups stay a documented manual step |

## 12. Risks

| Risk | Mitigation |
|---|---|
| A service pack or 2027 changes the SQL schema or the interop | Schema guard (§6); `swe selftest`; on every Electrical update: dump the interop surface and diff against `docs/reference/`, run selftest, re-check the netlist query (S9) before work continues |
| Shared-library side effects for other users of this Electrical server | Everything in `ADDLIFE`; review flags make new items visible; standard items untouched |
| Layout quality from computed coordinates | Layout helpers, grid taken from the title block (`title_block_grid`), mandatory render review. Worst case is slower progress, not wrong data |
| Electrical UI pauses during large batches | Batch per sheet; progress dialog |
| User and Claude editing the same sheet at once | Claude reads the sheet immediately before each batch; the add-in executes on the UI thread between user actions; snapshot covers mistakes |

## 13. Out of scope

- 2D cabinet footprints and panel layouts; 3D.
- An MCP server (possible later on the same endpoint).
- A design file as master, or syncing manual edits back into one (D1).
- Excel-automation-driven generation.
- Writing to the database.
- SOLIDWORKS PDM.
- Fixing WetBlasting's controller-side open items (`.acbind` channels 6/8, `ioPoints.ts`) — reported only.

## 14. Operations: backups, rollout, ownership

**Backups.** With the Electrical project as the only master (D1), its loss is the loss of the work.
Checked 2026-10-09: none of the 18 `tew_*` databases had ever been backed up and no project archive
existed on disk. Therefore:

- `swe backup` archives every project modified since its last successful backup through the add-in
  (`IEwProjectManagerX.archive`, `.proj.tewzip` = database + drawings) to a **destination off this PC**,
  and `swe backup --full` adds an environment archive (libraries, templates; `IEwArchiveEnvironmentX`).
- Retention 30 days. `swe backup --status` reports the age of the last good backup; `swe health` warns
  when it is older than 7 days.
- A backup counts only once a **restore has been proven**: phase 1 restores a `LIBTEST` archive into a
  new project and compares netlists (S10).
- The skill's workflow B ends with `swe backup` (phase 4).
- The destination is decision D-C; until it is set, `swe backup` refuses with that message.

**Rollout.** The first version runs on this workstation only. Electrical libraries and templates live
in each machine's own SQL Server, so colleagues on other PCs need the `ADDLIFE` library and template
(environment archive), the add-in (installer) and the skill. Proposed as **phase 5**, decision D-D.

**Ownership.** Someone approves new library items (clears `ADL_REVIEW`) and signs off the WetBlasting
acceptance. Proposed: the requesting engineer; decision D-E.
