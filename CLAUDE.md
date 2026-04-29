# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

BDTHPlugin ("Burning Down the House") is a Dalamud plugin for FFXIV that gives the player extended control over placing housing items (free placement, gizmo manipulation, snap, furnishing list with distance sort). User-facing entry point is the `/bdth` chat command (`/bdth list`, `/bdth debug`, `/bdth reset`, or `/bdth x y z [rot]` to write position/rotation directly).

## Build

The plugin uses the Dalamud .NET SDK and targets `net10.0-windows`. Building requires the Dalamud dev distribution to be present at `%AppData%\XIVLauncher\addon\Hooks\dev\` (the CI workflow downloads it from `https://goatcorp.github.io/dalamud-distrib/stg/latest.zip`).

```
dotnet restore
dotnet build --configuration Release
```

There are no tests — CI just builds the Release artifact, zips it, and on tag pushes creates a GitHub Release plus a `repository-dispatch` to `LeonBlade/DalamudPlugins` to trigger the third-party plugin repo update. To cut a release, bump `<Version>` in `BDTHPlugin/BDTHPlugin.csproj` and push a matching git tag.

## Architecture

The plugin is a single Dalamud `IDalamudPlugin` (`BDTHPlugin/Plugin.cs`) that wires together three long-lived singletons:

- **`Plugin`** — owns Dalamud `[PluginService]` references (all exposed as `public static` so the rest of the codebase reads them off the `Plugin` type rather than passing them around). Registers the `/bdth` command, subscribes to `Framework.Update` (drives `Memory.Update()` each frame) and `Condition.ConditionChange` (auto-opens the main window when entering housing mode if `Configuration.AutoVisible`). Also pre-loads `HousingFurniture` / `HousingYardObject` Lumina sheets into dictionaries used by `TryGetFurnishing` / `TryGetYardObject`.
- **`PluginMemory`** (`PluginMemory.cs`) — the unsafe game-interop layer. On construction it sig-scans the FFXIV client to locate the `LayoutWorld` / `HousingModule` static pointers, the `SelectItem` / `PlaceHousingItem` / `HousingLayoutModelUpdate` native functions, and the asm bytes that gate "place anywhere" / wall / wallmount placement (these get patched at runtime to lift restrictions). Exposes `HousingStructure*`, `HousingModule*`, `Camera*`, plus the editable `position` / `rotation` vectors that the UI binds to. Anything that touches the running game goes through here.
- **`PluginUI`** (`Interface/PluginUI.cs`) — owns a Dalamud `WindowSystem` containing `MainWindow`, `DebugWindow`, `FurnitureList` (under `Interface/Windows/`) and a `Gizmo` (ImGuizmo-based 3D manipulator). `Plugin.cs` calls `ImGuizmo.SetImGuiContext(ImGui.GetCurrentContext())` once at startup so the gizmo shares the Dalamud ImGui context. Shared item-edit controls live in `Interface/Components/ItemControls.cs`.

Cross-cutting:

- **`Configuration`** — Dalamud-serialized settings, fetched via `PluginInterface.GetPluginConfig()`.
- **`Structs.cs`** — manually-laid-out `[StructLayout]` mirrors of the in-game housing structures (`HousingStructure`, `HousingModule`, `LayoutWorld`, `Camera`, etc.) used by `PluginMemory`'s unsafe pointers. Update these together with sig changes when the game patches.
- **`AtkManager.cs`** — wraps Dalamud `GetAddonByName` lookups for the Housing/Inventory ATK addons, used by the furnishing-list flow.

The `/bdth x y z [rot]` command path is illustrative of the data flow: `Plugin.OnCommand` parses, writes into `Memory.position` / `Memory.rotation`, and calls `Memory.WritePosition` / `Memory.WriteRotation` which poke the live `HousingStructure->ActiveItem`. The same fields are bound to the ImGui inputs and the gizmo, so UI, command, and gizmo all mutate one shared piece of state. `Memory.CanEditItem()` gates writes to avoid touching memory when no item is selected or housing mode is inactive.

## Conventions

- 2-space indentation, Allman braces, `namespace BDTHPlugin{,.Interface,…}` (file-scoped namespaces are not used here).
- `AllowUnsafeBlocks` is on; pointer access to game memory is normal in `PluginMemory` / `Structs.cs`. Outside those, prefer the helpers (`Plugin.TryGetFurnishing`, `Plugin.IsOutdoors`, `AtkManager` accessors) over raw pointer chases.
- Dalamud services are accessed as `Plugin.Chat`, `Plugin.Log`, `Plugin.GameGui`, etc. — do not introduce new DI; add a new `[PluginService]` static on `Plugin` if a new service is needed.
