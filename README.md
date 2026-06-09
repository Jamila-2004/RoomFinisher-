# Room Finisher — Revit Add-In

A production-quality Revit 2025/2026 add-in (WPF + MVVM) that loads every room from the active
model into an Excel-like editable schedule, lets architects bulk-edit finishes, save/load templates,
run validation, quality checks and analytics, and push all changes back to Revit **in a single
transaction**.

---

## 1. Requirements

| Tool | Version |
|------|---------|
| Revit | **2025** (or 2026 — see retargeting below) |
| .NET SDK | **.NET 8 SDK** (`net8.0-windows`) |
| OS | Windows x64 |
| NuGet packages | `ClosedXML` 0.105.0, `OxyPlot.Wpf` 2.2.0 (restored automatically) |

> JSON uses the in-box **System.Text.Json** — Newtonsoft is deliberately avoided because Revit ships
> its own copy and a version clash inside Revit's shared AppDomain is a classic add-in failure.

---

## 2. Architecture

Clean separation; **no Revit API calls in the ViewModels**, and because the window is **modeless**,
every model read/write after the first load is marshalled onto Revit's thread through an
`ExternalEvent`.

```
Commands/        OpenRoomFinisherCommand        — IExternalCommand: first room read + show window
App/             RoomFinisherApplication        — IExternalApplication: ribbon tab + button
Revit/           RevitRequestRouter             — generic IExternalEventHandler (UI <-> Revit bridge)
                 RevitWorker                    — IRevitWorker impl (Collect / ApplyChanges)
                 RevitContext                   — singleton window + router + WPF Application guard
RevitServices/   RoomCollectorService           — FilteredElementCollector -> RoomData
                 RoomParameterService           — BuiltInParameter map, units, read-only handling
                 RevitTransactionService        — the single commit transaction
Models/          RoomData, RoomChangeSet, FinishTemplate, ChangeRecord,
                 AnalyticsData, QualityIssue, CommitResult, FinishOptions, AppSettings, RoomField
ViewModels/      MainViewModel (orchestrator + CollectionViewSource), RoomViewModel (dirty/undo),
                 BulkEditViewModel, ReviewChangesViewModel, AnalyticsViewModel,
                 QualityReportViewModel, ApplyTemplateViewModel
Views/           MainWindow, BulkEditWindow, ReviewChangesWindow, AnalyticsWindow,
                 QualityReportWindow, ApplyTemplateWindow, DialogService
Services/        SearchService, TemplateService, BulkEditService, UndoService,
                 ChangeTrackingService, ExcelExportService, AnalyticsService,
                 QualityCheckService, AuditLogService, IRevitWorker, IDialogService
Helpers/         RelayCommand, ValidationHelper, LevenshteinHelper
Resources/       Styles.xaml (flat BIM theme), Converters.cs
```

### Why these design choices
- **Modeless + ExternalEvent.** A WPF window cannot call the Revit API from its own thread. The
  window stays open while you keep working in Revit; *Refresh* and *Apply* route through
  `RevitRequestRouter.Enqueue(...)` → `ExternalEvent.Raise()` → result posted back via the dispatcher.
- **Native parameters** are used for Number, Name, Department, Wall/Floor/Ceiling/Base Finish and
  Level. **Ceiling Height is not a native room parameter**, so it is resolved by a *configurable
  parameter name* (default `"Ceiling Height"`). If the model has no such parameter the cell is shown
  read-only/blank — values are never fabricated.
- **Units.** Ceiling height round-trips through Revit's `AsValueString()` / `SetValueString()`, which
  respects the document's display units (mm, m, feet-inches).
- **Charts use OxyPlot** (not LiveCharts2): OxyPlot has no native SkiaSharp dependency, so there is no
  risk of a native-binary version clash inside Revit.

---

## 3. Build

```powershell
# from the solution folder
dotnet build "RoomFinisher\RoomFinisher.csproj" -c Release -p:Platform=x64
```

Output (with all deployable dependencies) lands in:
```
RoomFinisher\bin\x64\Release\
    RoomFinisher.dll
    ClosedXML.dll, DocumentFormat.OpenXml*.dll, ExcelNumberFormat.dll, RBush.dll,
    SixLabors.Fonts.dll, System.IO.Packaging.dll, ClosedXML.Parser.dll
    OxyPlot.dll, OxyPlot.Wpf.dll, OxyPlot.Wpf.Shared.dll
```
`RevitAPI.dll` / `RevitAPIUI.dll` are intentionally **not** copied (referenced with `Private=false`).

### Targeting Revit 2026
Both 2025 and 2026 use `net8.0-windows`; only the API path differs:
```powershell
dotnet build "RoomFinisher\RoomFinisher.csproj" -c Release -p:Platform=x64 -p:RevitVersion=2026
```

---

## 4. Deployment

1. Build (Release).
2. Create a folder for the add-in DLLs, e.g.:
   `C:\ProgramData\Autodesk\Revit\Addins\2025\RoomFinisher\`
   and copy the **entire** contents of `bin\x64\Release\` into it.
3. Copy `RoomFinisher\RoomFinisher.addin` to:
   `C:\ProgramData\Autodesk\Revit\Addins\2025\`
4. Confirm the manifest's `<Assembly>` points to the DLL (relative to the Addins\2025 folder):
   ```xml
   <Assembly>RoomFinisher\RoomFinisher.dll</Assembly>
   ```
5. Start Revit → a **Room Finisher** ribbon tab with an **Open Room Finisher** button appears.

---

## 5. Usage

Open a project, click **Open Room Finisher**:

- **Grid** lists all rooms, ordered by level then number. Edit cells inline; finish columns are
  editable combo-boxes (pick a default or type a custom value — custom values are remembered for the
  session). Edited cells turn **yellow**, the row shows a ● indicator, and the header shows
  *"N modified rooms"*.
- **Search** by Name / Number / Department (real-time) combined with the **Level** filter.
- **Bulk Edit** — select multiple rows, choose a target field + value, apply to all at once.
- **Save / Load Template** — JSON finish templates (e.g. `Office_Template.json`). On load, choose a
  scope (All / Selected / By Department) and **preview** before applying.
- **Detect Issues** — quality engine: duplicate numbers (error), duplicate names, empty finishes,
  case/spacing inconsistencies, and fuzzy near-duplicates (e.g. *Marble* / *Marbel*). Shows a
  **Quality Score** = `100 − errors×5 − warnings×1` (min 0). Export to CSV/XLSX, or *Fix Selected* to
  jump to the room.
- **Analytics** — finish distributions (counts + %), most-common summaries, pie + bar charts; updates
  live as you edit.
- **Export to Excel** — exports the **currently visible** (filtered) rows to a formatted `.xlsx`
  (bold/frozen header, auto-fit columns).
- **Undo Changes** — reverts local edits (pre-commit; cleared after a successful commit).
- **Apply Changes** — validates → **Review Changes** window (before/after per parameter) → on Confirm,
  writes everything in **one transaction**, then reports *Updated: N / Failed: M* with details, and
  reloads the grid from the model. Unplaced rooms and read-only parameters are reported, never crash.
- **Show in Revit** — selects the room(s) and zooms the active Revit view to them (toolbar button or right-click a row).
- **Refresh Rooms** — reloads from Revit; warns if you have unsaved local edits.

Audit log of every commit: `%APPDATA%\RoomFinisher\audit.log`.

### AI Finish Assistant (Anthropic Claude)

**AI Assist** analyzes the selected rooms (or all rooms) and proposes Wall/Floor/Ceiling/Base
finishes — filling empty ones *and* flagging existing finishes that are unsuitable for a room's
function (hygiene, moisture, durability, traffic, acoustics, accessibility), reusing your project's
existing finish palette for consistency. Each suggestion carries a short reason.

- **It always asks before changing anything.** Suggestions appear in an approval window with a
  checkbox per change (current → suggested + reason). You untick what you don't want, click
  **Apply Selected**, and the accepted changes land in the grid as normal edits — which then go
  through the usual validation → Review → single-transaction commit. Nothing is written to Revit by
  the AI directly.
- **Providers:** works with **Google Gemini** (default — free tier available at
  `aistudio.google.com`) **or** **Anthropic Claude**. Pick the provider in **AI Settings**, paste the
  matching key, and choose a model (e.g. `gemini-2.0-flash`, or `claude-haiku-4-5`).
- **Setup:** click the gear (**AI Settings**) → choose provider → paste the API key (must match the
  provider — a Google key for Gemini, an `sk-ant-…` key for Anthropic) → pick a model → Save.
- **Implementation:** direct HTTPS calls to the Gemini (`generativelanguage.googleapis.com`) or
  Anthropic (`api.anthropic.com`) REST APIs via `HttpClient` — **no SDK and no extra NuGet dependency**
  (deliberately, to avoid assembly conflicts inside Revit). Output is constrained to JSON
  (`responseMimeType` for Gemini, `output_config.format` for Anthropic) so it's always parseable. Up to
  80 rooms per run.
- **Security note:** the API key is stored in plain text in `%APPDATA%\RoomFinisher\settings.json` on
  your machine. Calls require internet access; if no key is set, AI Assist prompts for one.

---

## 6. Configuration

`%APPDATA%\RoomFinisher\settings.json`:
```json
{
  "CeilingHeightParameterName": "Ceiling Height",
  "LastTemplateFolder": "",
  "AiProvider": "Google",
  "GoogleApiKey": "",
  "GoogleModel": "gemini-2.0-flash",
  "AnthropicApiKey": "",
  "AiModel": "claude-opus-4-8"
}
```
Set `CeilingHeightParameterName` to whatever project/shared parameter your office uses for ceiling
height. If it doesn't exist on a room, that cell is read-only/blank.

---

## 7. Known simplifications & caveats

- **Multi-level filter** is exposed as single-level + *All Levels* (multi-select level filtering was
  intentionally simplified — use search to narrow further).
- **"Apply by room type"** maps to **by Department**, because Revit rooms have no family/type concept.
- **Pre-commit ceiling-height validation** is lenient (digit/sign check only); the authoritative parse
  happens at commit via Revit's unit engine, and unparseable values are reported as per-room failures.
- **Dependency note:** ClosedXML and OxyPlot are copied locally. They build and run cleanly against
  Revit 2025; if another add-in loads conflicting versions of a shared dependency you may need to align
  versions. OxyPlot was chosen specifically to avoid native (SkiaSharp) conflicts.
- The add-in cannot be run headless — it must be loaded inside Revit. The build here is verified
  clean; in-Revit behaviour is verified via the checklist below.

---

## 8. Verification checklist (inside Revit)

1. Ribbon tab **Room Finisher** + **Open Room Finisher** button appear.
2. Open a model with rooms → grid populates and groups by level.
3. Edit a finish → cell turns yellow, modified count updates.
4. Select several rooms → **Bulk Edit** applies to all.
5. **Save Template**, then **Load Template** → scope + preview → apply.
6. **Detect Issues** → score + issues; **Export Report** (CSV/XLSX).
7. **Analytics** → charts render; edit a room → charts update live.
8. **Export to Excel** → filtered rows, bold + frozen header.
9. **Apply Changes** → validation → Review → Confirm → values written in Revit (verify in Properties).
10. **Undo** reverts local edits; **Refresh** warns on unsaved changes.
11. Edge cases: 500-room model (no lag), unplaced room (excluded), read-only param (reported), missing
    Ceiling Height parameter (blank/read-only cell).
