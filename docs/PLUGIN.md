# Blueprint Review plugin — user guide

**Blueprint Review** is one Unreal Editor plugin (`plugins/BlueprintReview`) with several modules. This guide starts with its exporter, the *BlueprintExport* module. It writes Blueprints to text files that the BlueprintReview tools (`blueprintreview`, `blueprintreview-app`) read, and that can be diffed, reviewed or archived outside the editor. For each Blueprint it writes:

- **`<Blueprint>.json`**: everything in the Blueprint, in the same format as `blueprintreview dump`:
  - variables, components and class defaults;
  - the enums the Blueprint uses, with each entry's display name;
  - every graph with its nodes, pins and wires;
  - the node titles, colours and pin labels exactly as the editor shows them.
- **`<Blueprint>.<Graph>.t3d`**: each graph in the editor's own copy/paste format. Paste it back into a Blueprint graph, or into [blueprintue.com](https://blueprintue.com).

You don't need the plugin to *view* Blueprints: `blueprintreview` reads `.uasset` files directly. The plugin is for:
- **Exact editor output:** node titles, colours and pin labels as the editor computes them. `blueprintreview` reads graphs, nodes and pins, but not that display text.
- **Batch export:** exporting a whole project from the command line, e.g. on a build machine.
- **Checking `blueprintreview`:** comparing its reading of an asset against the editor's (`blueprintreview check`).

The plugin's other modules, all in the same plugin and enabled together:

| Module | What it does |
|---|---|
| `BlueprintExport` | The exporter described here: canonical JSON and T3D, the `BlueprintExport` commandlet and the Content Browser entry. The base the other modules build on. |
| `BlueprintApiDump` | The engine metadata and API tables behind `blueprintreview cpp` (`-run=BlueprintMetadataDump`). |
| `BlueprintEditAutomation` | **Editing Blueprints** (by hand, by script or by an AI agent) with edit plans: see [EDITING.md](EDITING.md). It's **experimental**. |
| `BlueprintReviewLink` | The diff hand-over and view sync with the BlueprintReview app. |
| `BlueprintConvert` | *Convert to C++…* and the migration and behaviour commandlets: see [CONVERTING.md](CONVERTING.md). |
| `BlueprintTrace` | Records converted classes' actors in a game run, for the converter's behaviour checks. |

Before 1.2 this was two plugins, *Blueprint Exporter* and *Blueprint Edit Automation*. **Upgrading:** delete `Plugins/BlueprintExporter` and `Plugins/BlueprintEditAutomation` from the project and enable *Blueprint Review* instead (`scripts/deploy_plugin.py` does all of that). A Converter Path saved in the project's user settings is carried over the first time the editor starts with the new plugin.

Supported engines: **UE 5.7 and 5.8**. Other versions haven't been tried; the editing module (`BlueprintEditAutomation`) has no code for older engines. The plugin depends on the engine's **Enhanced Input** plugin (enabled by default in UE 5 projects) and enables it.

---

## 1. Install it into a project

> **Not in the beta's downloads:** the Unreal plugin (*Blueprint Review*) isn't
> distributed with the beta yet. Write to [support@efenggamedev.com](mailto:support@efenggamedev.com) if you'd like
> to try them.

The plugin source is in the BlueprintReview source repository at `plugins/BlueprintReview`.

### C++ projects (the project has a `Source/` folder)

1. Close the editor.
2. Copy `plugins/BlueprintReview` to `<YourProject>/Plugins/BlueprintReview`.
3. Open the project. The editor offers to build the missing module; click **Yes**. You can also build from your IDE.

### Blueprint-only projects

The editor can't compile plugins for a project without C++ code, so use a prebuilt copy for your engine version. Someone with Visual Studio installed builds it once:

```bat
"C:\Program Files\Epic Games\UE_5.7\Engine\Build\BatchFiles\RunUAT.bat" BuildPlugin ^
    -Plugin="<repo>\plugins\BlueprintReview\BlueprintReview.uplugin" ^
    -Package="D:\Build\BlueprintReview_5.7" -TargetPlatforms=Win64 -Rocket
```

Then copy `D:\Build\BlueprintReview_5.7` to `<YourProject>/Plugins/BlueprintReview`, leaving out its `HostProject` and `Intermediate` folders. A build only works with the engine version it was built for.

### With the deploy script (from a checkout of this repo)

`scripts/deploy_plugin.py` installs the plugin and the converter from your checkout in one go. Use it to try the current code in a real project, or to update a project after pulling. Close the editor for that project first, then run:

```sh
python scripts/deploy_plugin.py "D:/Documents/Unreal Projects/StackOBot"
```

It needs Rust (`cargo`) and, for a Blueprint-only project, Visual Studio. It takes the project folder or its `.uproject`, and it:

1. builds `blueprintreview` and `blueprintreview-app` (`cargo build --release -p blueprintreview-cli -p blueprintreview-app`);
2. installs the plugin to `<Project>/Plugins/BlueprintReview` (and removes the old `BlueprintExporter` and `BlueprintEditAutomation` folders, if the project has them). A C++ project gets the source, and the editor builds it when the project opens. A Blueprint-only project gets a `RunUAT BuildPlugin` build for the engine in the `.uproject`'s `EngineAssociation` (`C:/Program Files/Epic Games/UE_<version>`). If a copy is already installed, it's replaced;
3. copies both executables into the plugin's `Tools/` folder: the converter, and the app the editor's diff hand-over starts;
4. enables `BlueprintReview` in the `.uproject` if it isn't enabled yet, and drops the old plugins' entries;
5. points **Editor Preferences → Plugins → Blueprint Review → Converter Path** at the copied executable. This is a per-user setting, written to the project's `Saved/Config/WindowsEditor/EditorPerProjectUserSettings.ini`, so nothing machine-specific goes into the project's `Config/`.

The project then has its own copy of everything. Rebuilding in the repo doesn't affect it until you deploy again.

| Option | Effect |
|---|---|
| `--engine DIR` | Use this engine install, not the one named by `EngineAssociation` (needed for a source-built engine). |
| `--prebuilt` | Build the plugin with RunUAT even for a C++ project, so the editor doesn't have to compile it. |
| `--build-dir DIR` | Where RunUAT builds (default `E:/tmp/bpexporter/deploy/UE<version>`; the build log goes next to it). |
| `--skip-cargo` | Copy the existing release executables in `target/release/` without rebuilding them. |
| `--no-cli` | Install only the plugin. The converter, the app and the Converter Path are left as they are. |

If the editor still has the plugin loaded, the script stops with *"…is in use; close the editor and run again"*, and the installed copy is left as it was. A project that has gained a `Source/` folder since the last deploy (after a *Convert to C++…*, for example) gets the source copy next time.

### Check it's enabled

In the editor, open **Edit → Plugins** and search for "Blueprint Review". It's under the *Editor* category. Tick **Enabled** if it isn't already, and restart when asked.

The plugin is editor-only. It isn't included in packaged games.

---

## 2. Export from the editor

1. In the **Content Browser**, select one or more Blueprints. Actor, Widget, Anim and other Blueprint types all work, and you can select across folders.
2. Right-click, then choose **Export Blueprint (JSON + T3D)…**.
3. Pick an output folder.

A message lists how many were exported and any that failed. Files are placed under the output folder by their content path:

```
<output>/Game/Blueprints/BP_Door.json
<output>/Game/Blueprints/BP_Door.EventGraph.t3d
<output>/Game/Blueprints/BP_Door.UserConstructionScript.t3d
<output>/Game/Blueprints/BP_Door.OpenDoor.t3d          (one per function/macro graph with nodes)
```

Unsaved changes in the editor *are* included, because the plugin exports what's currently loaded, not the file on disk.

---

## 3. Export from the command line

Use the editor's command-line executable with the `BlueprintExport` commandlet. No editor window opens. This suits build machines, scripts and large projects.

```bat
"C:\Program Files\Epic Games\UE_5.7\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" ^
    "D:\Projects\MyGame\MyGame.uproject" ^
    -run=BlueprintExport -Out="D:\Exports" -Paths=/Game/Blueprints ^
    -unattended -nullrhi -nosplash
```

| Argument | Meaning |
|---|---|
| `-Out=<folder>` | Where to write the files (required). |
| `-Paths=/Game/A,/Game/B` | Export every Blueprint under these content folders, including subfolders. `-Paths=/Game` exports the whole project. |
| `-Assets=/Game/A/BP_X,/Game/B/BP_Y` | Export specific Blueprints, by content path. |
| `-NoT3D` | Only write the `.json` files. |
| `-SaveBeforeExport` | Re-save each Blueprint before exporting it. **This overwrites the `.uasset` files.** Use it only to verify `blueprintreview`, so the file and the export describe exactly the same state (see section 5). |

`-Paths` and `-Assets` can be combined. `-unattended -nullrhi -nosplash` are standard UE flags: no dialogs, no GPU, no splash screen.

The log ends with a summary like:

```
LogBlueprintExport: Display: BlueprintExport: 48 exported, 0 failed -> D:/Exports
```

> **The exit code isn't a reliable success signal.** UE returns a non-zero exit code whenever *any* error was logged during the run. Blueprint compile errors count, and they often appear for assets that are broken or missing references, even though they still export fine. In scripts, check the `0 failed` part of the summary line. A Blueprint that can't be exported at all is logged as `LogBlueprintExport: Error: <path>: ...`.

---

## 4. What's in the files

### JSON

The format is described by `schema/blueprint.schema.json`, and its fields are explained in [docs/CLI.md](CLI.md#blueprintreview-dump--blueprint-as-json). A node from an export looks like this:

```json
{
  "guid": "0ECDACC94AFCBF069FBCF3A860A8E683",
  "class": "K2Node_CallFunction",
  "pos_x": -128,
  "pos_y": 1520,
  "title": "Get Actor Up Vector\nTarget is Actor",
  "title_color": [0.4, 0.85, 0.35, 1],
  "compact_title": "×",
  "properties": [
    { "name": "FunctionReference", "type": "StructProperty(MemberReference)", "value": { "Struct": { "...": "..." } } }
  ],
  "pins": [
    {
      "id": "5983533840253B1DBC9250803918DAED",
      "name": "self",
      "direction": "input",
      "friendly_name": "Target",
      "type": { "category": "object", "sub_category_object": "/Script/Engine.Actor" },
      "display_name": "Target"
    }
  ]
}
```

The editor-only fields:
- **`title` and `title_color`:** what the editor draws in the node header.
- **`compact_title`:** the text drawn in the body of nodes the editor draws compact (`×`, `+`, `GET`). Only those nodes have one.
- **`display_name`:** the pin label the editor shows, when it differs from the pin's name. `""` means the pin is drawn without a label, like exec pins.
- **`compiled_flags`** (on variables): the `PropertyFlags` of the compiled skeleton class property, which the Details panel lists as *Defined Property Flags*. These are the stored `property_flags` plus the flags the compiler and the type add. A `.uasset` doesn't save the type-implied ones (`FProperty::Serialize` drops `CPF_ComputedFlags`), so the viewer derives them when this field is missing.

`friendly_name` is the pin's saved label, so the standalone parser reads it too.

A Widget Blueprint's export also has `widgets`, its designer hierarchy as the editor holds it after loading. `is_variable` there is the widget's own *Is Variable*; the standalone parser reads it when saved and otherwise takes the widget class's default, which it knows for the engine's UMG and CommonUI widgets.

The files are designed to **diff cleanly** in source control:
- Nodes are sorted by their GUID.
- Wires are written as `"linked_to": ["<nodeGuid>:<pinId>"]`.
- Nothing changes between exports unless the Blueprint changed.
- Only values that differ from their defaults are listed (class defaults, class settings, component settings, node settings), the same as what UE saves in the `.uasset`. `class_settings` holds the `UBlueprint`'s own saved properties (Display Name, Hide Categories, Deprecate, ...) except those other fields carry or that are generated (`bp_model::BLUEPRINT_PROPERTIES_EXCLUDED`).

### T3D

Each `.t3d` file is exactly what **Ctrl+C** on that graph's nodes would put on the clipboard. `blueprintreview copy` and the viewer's Ctrl+C write the same format without the editor. To use one:
- **Back into the editor:** open the file, copy all of it, and press Ctrl+V in any Blueprint graph.
- **Sharing:** paste it into blueprintue.com.

---

## 5. Checking `blueprintreview` against the editor

`blueprintreview check` compares `blueprintreview`'s own reading of a `.uasset` with the plugin's export of it:

```sh
blueprintreview check Content/Blueprints/BP_Door.uasset D:/Exports/Game/Blueprints/BP_Door.json
```

For the result to be meaningful:
- Export **in the asset's own project**, so every class and asset it references can load. Missing references load as `None`.
- Export **with the engine version that last saved the asset**. A newer engine may consider different values "default". It also upgrades data when loading, so its view differs from the saved file.

Export assets checked this way can be added to the test fixtures as golden files (see [fixtures/README.md](../fixtures/README.md)).

### Checking `blueprintreview copy` against the editor

`-ImportT3D=<list>` pastes copy text into Blueprints and exports what was pasted. Each line of the list file has four tab-separated fields: the Blueprint (`/Game/A/BP_X`), the graph name, the copy text file, and the JSON file to write. The text is pasted with `FEdGraphUtilities::ImportNodesFromText`, the editor's paste, into a new graph that temporarily takes that graph's name and place. Pasted nodes keep their GUIDs.

```sh
UnrealEditor-Cmd.exe MyGame.uproject -run=BlueprintExport -ImportT3D=D:/paste.txt -unattended -nullrhi
```

`scripts/plugin_roundtrip.py --t3d` uses it to check `blueprintreview copy`. For every graph it pastes `blueprintreview`'s copy text and the editor's own (the plugin's `.t3d`), each in a fresh editor session, and compares the two pasted graphs with `blueprintreview check --graph`. Pasting changes nodes (they're rebuilt, and names made unique), so the two texts are compared by what they paste to, not by what they say.

### Screenshots of the editor's views

The `BlueprintExport.Shots <list file>` console command takes screenshots of the editor itself, so the viewer can be compared with it (`scripts/parity_audit.py shots` writes the list and the viewer's matching storyboard). It needs a real editor session (no `-nullrhi`). Each line of the list is tab-separated:

| Line | Writes |
|---|---|
| `graph <asset> <graph name> <out.png>` | the Blueprint editor on that graph, zoomed to fit (`OpenGraphAndBringToFront`, `ZoomToFit`) |
| `defaults <asset> <out.png>` | the Blueprint editor showing *Class Defaults* |
| `diff <old file> <new file or -> <asset> <out>` | the editor's Blueprint diff window (`SBlueprintDiff`) for two versions of the asset: `<out>.png` and `<out>.json`, the differences tree's rows as shown. Each file is loaded as a revision (`DiffUtils::LoadPackageForDiff`), so it has to be under the project's `Saved/` folder; `-` is the asset itself |
| `quit` | exits the editor |

`<asset>` is a package path (`/Game/A/BP_X`).

```sh
UnrealEditor.exe MyGame.uproject -ExecCmds="t.IdleWhenNotForeground 0,BlueprintExport.Shots D:/shots.txt"
```

---

## 6. Engine metadata for the viewer

A `.uasset` doesn't contain the display names of the engine functions it calls. For example, `K2_GetActorLocation` is shown as "Get Actor Location". It doesn't contain a variable's type-implied property flags, engine enums' display names, or the values a Blueprint inherits from its native parent classes either. The viewer gets them from two files built from the engine: a table, `crates/bp-model/data/engine_table.tsv`, and the class defaults, `crates/bp-model/data/engine_defaults.json`.

To regenerate the table after an engine upgrade:
1. Build the plugin for each engine you want covered.
2. Run the `BlueprintMetadataDump` commandlet once per engine, oldest first, and merge the results:

```sh
python scripts/gen_engine_table.py \
    --all-plugins "C:/Program Files/Epic Games/UE_5.7" \
    --all-plugins "C:/Program Files/Epic Games/UE_5.8"
```

`--all-plugins` is how the shipped files are made: it builds the plugin for that engine, makes a scratch project with every engine plugin that supports Win64 enabled (Mover, CommonUI, SmartObjects, Chooser, Control Rig…), and drops any plugin the editor fails to load, so the table covers what the engine and its plugins declare. `--reuse-plugin` skips the build when the plugin is already in the work folder. Your project's own C++ isn't in it (see the known limitation in `docs/ROADMAP.md`). `--run ENGINE PROJECT` dumps from a project of yours instead, with the plugin enabled. To run the commandlet by hand:

```bat
UnrealEditor-Cmd.exe MyGame.uproject -run=BlueprintMetadataDump -Out=D:\engine_table.tsv -Defaults=D:\engine_defaults.json
```

The table covers every Blueprint-visible function, class (with its `DefaultToInstanced` / `Interface` flags and the interfaces it implements), struct display name, native enum entry display name (where it isn't the entry's own name), StandardMacros entry and input key name in the modules the editor loads, each Blueprint-callable function's parameters (direction, pin type, `CPP_Default_` value, advanced and hidden pins: the pins the editor adds when it rebuilds a node) and the display names of `BlueprintAssignable` delegates. It also has the compiled flags of a variable of each struct and each other type kind. The commandlet gets them by creating the variable's property with the Kismet compiler's own code (`FKismetCompilerUtilities::CreatePropertyOnScope`) and linking it. It's sorted and has no timestamps, so regenerating it with the same engine gives the same file.

`-Defaults=` also writes the class defaults: for every class a Blueprint can derive from and every component class (with their ancestors), its parent, its editable properties and its default subobjects (Character's capsule, mesh and movement component). A class lists the properties it declares with their whole value, `Category`, `DisplayName` and whether they're advanced (`CPF_AdvancedDisplay`), and the inherited ones its constructor changed as a delta from its parent's, so the file stays small (about 4.4 MB with every engine plugin). The animation graph and state machine node classes (editor-only, `AnimGraphNode_*`, `AnimState*`) are in it too, for an anim node's *Details*, and a `structs` map lists, per `FAnimNode_*` struct those nodes edit, its editable fields with their `Category`, `DisplayName`, advanced flag and pin metadata (`PinShownByDefault`, `PinHiddenByDefault`, `NeverAsPin`, `AlwaysAsPin`). `gen_engine_table.py` keeps the newest engine's; the viewer's *Class Defaults* says which engine that is when the asset was saved by another.

---

## 7. The engine API table for the C++ converter

The Blueprint → C++ converter (`blueprintreview cpp`) generates code against an API table: for every native class, struct and enum its C++ name, header and module, for every Blueprint-visible function its native signature, default arguments and metadata, for every Blueprint-visible property its C++ type and access. One table per engine version is committed as `crates/bp-cpp/data/engine_api_<version>.tsv`.

To regenerate them (this builds the plugin for each engine and runs the commandlet in a scratch project with the engine plugins the fixtures use enabled):

```sh
python scripts/gen_engine_api.py --engine "C:/Program Files/Epic Games/UE_5.7" --engine "C:/Program Files/Epic Games/UE_5.8"
```

A project's own table, for its C++ and its project plugins (`blueprintreview cpp --project-api`), from a project that has the plugin installed:

```sh
python scripts/gen_engine_api.py --engine "C:/Program Files/Epic Games/UE_5.7" --project D:/Work/MyGame/MyGame.uproject --out MyGame.api.tsv
```

By hand: `UnrealEditor-Cmd.exe MyGame.uproject -run=BlueprintMetadataDump -Api=D:\engine_api.tsv [-ProjectOnly]`.

Reflection doesn't record whether a function is defined inline, which makes it callable from other modules without being exported. The commandlet reads the class's header (the engine's installed source) to find out, and the same way finds the inline `GetFoo()` getters of private members such as `ACharacter::GetCharacterMovement`. The table also lists the `BlueprintAssignable` dispatchers (`M`) with their signatures (`Y` rows), so the converter can declare bound functions exactly as `AddDynamic` checks them. It also marks the members a class's header declares under `UE_DEPRECATED` (`C`, as `USkinnedMeshComponent::ForcedLodModel`): Blueprints still use them directly, but C++ has to go through their accessors. It also flags a `MinimalAPI` class whose header declares a virtual function that is neither exported nor inline (`UEnvQueryContext_BlueprintBase::ProvideContext`): a Blueprint can subclass it, but a C++ class in another module can't link. Editor-only properties (`WITH_EDITORONLY_DATA`, such as `USplineComponent::EditorTangentColor`) are marked `W`, so the converter sets them under the same guard and a game build still compiles. A native setter must take the value: `UShapeComponent::SetUseSystemDefaultObstacleAreaClass()` takes none, so it isn't recorded as `bUseSystemDefaultObstacleAreaClass`'s setter. The committed tables predate both rules; the converter keeps a short list of the affected members until they're regenerated.

A Blueprint-only project that enables engine plugins the committed tables don't include can get a full table without installing the plugin into it: `--plugins-of MyGame.uproject --out MyGame.api.tsv` makes a scratch project with the same engine plugins enabled.

### Test Blueprints for the converter

`-run=BlueprintTestContent` builds the converter's verification Blueprints (`tests/cpp-project/Content/CppTests`) from code: a user enum and struct, a Blueprint Interface, an actor with a variable of every kind, one with a component tree, one with function signatures, a dispatcher and the interface, a function library, a Blueprint child, a UObject and an actor component Blueprint, `BP_Logic`, whose functions have bodies (arithmetic, branches, a sequence, loops, a switch, a cast, `IsValid`, calls), `BP_Events`, with an event graph (BeginPlay, Tick, custom events, a Server RPC, an event parameter another event reads, FlipFlop and DoOnce), `BP_Delegates` (a dispatcher bound in BeginPlay, broadcast and unbound by custom events, and a component's overlap event), `BP_Latent` (a delay whose continuation reads a value written before it, a next-tick delay, and a retriggerable delay a custom event restarts), `BP_Timeline` (a timeline with a float and an event track, played from BeginPlay), and `BP_Input` (a pawn with two Enhanced Input actions, `IA_CppTestThrottle` and `IA_CppTestFire`, made as assets too: Triggered, Started and Completed events, a value Tick reads, and *Get Input Action Value*). `-Only=BP_Variables,...` makes some of them.

`-run=BlueprintBehaviour -Map=<folder> -Module=<module> [-Actors]` compares every pure function of the Blueprints converted into that folder with its C++ version: both are called on their class default objects with the same sampled inputs, and every output is compared. With `-Actors`, each actor class also runs in both forms in a game world. Both forms are spawned, ticked 5 frames, have each custom event called with the same sampled arguments, and are ticked twice more. Then every variable is compared. The C++ form is spawned deferred, so the members the `.bpmap.json` lists as `kept` can be set to their assets before its construction and BeginPlay, as the reparented Blueprint's defaults would set them. Pawns and player controllers also get an Enhanced Input component through the engine's own setup path (`SetupPlayerInputComponent` then the Blueprint's input bindings, or `SetupInputComponent`). Both forms must bind the same actions, trigger events and values. Each event binding then runs in both forms with the same sampled action instances, before the last ticks. `scripts/cpp_compile_check.py --behaviour` runs it after the build. `scripts/cpp_compile_check.py --make-content` runs it and copies the assets into the repo; see [tests/cpp-project/README.md](../tests/cpp-project/README.md).

Each actor runs in a test world of its own, after a twin of the same form, so overlap and actor-query logic finds one. The values are compared in a form both have: enums by value, structs field by field in order, object references relative to the actor, numbers within a relative 1e-5. Across a migration, the two forms are in two projects (or one project before and after):

- `-Record=<file.json>` runs the original Blueprints and writes what they did, under the Blueprints' names.
- `-Compare=<file.json>` runs the migrated forms against that file. Those are the reparented Blueprint (same class, the C++ names inherited) or the class that replaced it.

### Migrating a project: `BlueprintNativize`

`-run=BlueprintNativize -Map=<folder of .bpmap.json> -Step=Types|Classes [-KeepReplaced] [-DryRun]` applies the converter's output to the project. See [CLI.md](CLI.md#migrating-a-project) for what each step does. Run it on a copy or under source control: it deletes the replaced assets and resaves the Blueprints and maps that reference them. Its summary lines are:

- `Nativize types: N asset(s) replaced, M error(s)`;
- `Nativized: <asset>: …` per Blueprint, each with its compiler messages;
- `Nativize: N asset(s), R referencer(s), S saved, E error(s), W warning(s)`.

A referencer's message counts only when its baseline (`Nativize.baseline.json`, written by the `Types` step) didn't have it.

With `-KeepReplaced` (pass it to both steps), the replaced assets aren't deleted: they stay at their path, unused. Their redirects send every reference to the C++ type, and after the resave `Classes` checks the asset registry: each kept asset that a project package (other than another kept one) still references is a warning, `Kept: <asset> is still referenced by …`, and the exit code is 1. A kept asset is a copy for reference only. Loading it by its path gives the C++ type (the redirect), so don't place, use or edit it. Delete it once you no longer need it.

## 8. Convert to C++ from the editor

Right-click Blueprints, user-defined structs or enums in the Content Browser and choose **Convert to C++…**. This does what [CONVERTING.md](CONVERTING.md) describes, from one dialog:

| Option | What it does |
|---|---|
| **Mode** | *Native base* reparents each Blueprint onto its C++ class. *Replace* lets the C++ class take over. In *Replace*, a class only its Blueprint can keep parts of takes a native base instead, by itself. |
| **Replaced Blueprints** | *Keep at their path* (the default) or *Delete*. This applies to what's replaced: the structs, enums, Blueprint Interfaces and function libraries in either mode, and every class in *Replace*. A kept asset is unused afterwards (see `-KeepReplaced` above). |
| **Runtime errors** | *As Blueprint* (`--safety bp`) or *Idiomatic C++*. |
| **Module** and **Path** | Where the code goes, as in *New C++ Class*. Pick one of the project's modules (or a project plugin's), and a folder inside it, by default `<Module>/Converted`. The header and source of each type go into that one folder, since the generated files include each other by name. For a Blueprint-only project, the list shows the project's name as a new module, which is created. |
| **Move only part of the Blueprint** | The partial migration's `--functions`, `--variables`, `--events`, `--keep-in-bp` and `--pull-dependencies`, native base only. |

The dialog checks when it opens: it lists each asset as ready or blocked, with the blockers. Blueprints the selection depends on are listed as *(dependency)* and converted with it. Tick that the project is committed or backed up, then **Convert**:

1. **Check, then generate.** It checks again with the chosen options, then generates the code with `blueprintreview cpp` against an API table of everything the editor has loaded (`Saved/BlueprintReview/api.tsv`), which covers the project's C++ and its plugins. Writing that table reads the header of every native class, so it shows its progress and can be cancelled. It's kept with what it was made from (`api.tsv.key`: the engine version and every loaded module's binary) and reused until a module is rebuilt, added or removed. While the converter runs, the dialog shows what it's doing (`--progress`) with a progress bar: reading the assets, loading what they use, checking, generating each asset, writing the files, and in replace mode reading every package of the project to find what references the replaced assets.
2. **Confirm.** It shows what the converter printed (the values left to the Blueprints, the asset members) and asks before going on. If you stop there, the code is written and nothing else has changed.
3. **Set up the project:**
   - The module's `Build.cs` gets a marked block (`// BEGIN blueprintreview: Converted` … `// END …`) with the folder's include path and its `Modules.txt`. Converting into the same folder again replaces the block.
   - The `.uproject` enables the plugins in `Plugins.txt`. In a Blueprint-only project it also gets the new module, created with its `Build.cs`, its `.cpp` and the game and editor targets.
4. **Save and close.** It saves the open assets (asking first), starts the runner (`Saved/BlueprintReview/Convert/<time>/run.bat`) and closes the editor.
5. **The runner,** in a console window:
   1. waits for the editor to exit;
   2. builds the editor target (`build.log`);
   3. runs `BlueprintNativize -Step=Types` (`types.log`), then `-Step=Classes` (`classes.log`), with `-KeepReplaced` unless you chose *Delete*;
   4. opens the editor again, which shows the outcome in a notification with an *Open log* link.

   If the build fails, nothing is migrated. The runner shows `build.log` instead of opening the editor, since the project's module doesn't build. To go back, delete the output folder and its block in `Build.cs`.

The converter is found through **Editor Preferences → Plugins → Blueprint Review → Converter Path**, or as `blueprintreview` on `PATH` when that's empty. `scripts/deploy_plugin.py` sets it ([section 1](#with-the-deploy-script-from-a-checkout-of-this-repo)). The dialog remembers its choices per user (`EditorPerProjectUserSettings`). Windows only, like the rest of the plugin's build steps.

### The game trace: `BlueprintTrace`

The plugin's `BlueprintTrace` module (uncooked only: it loads in `-game` from the editor binary and in PIE), records the actors of the converted classes while the game runs. `scripts/trace_diff.py` diffs two recordings:

```sh
UnrealEditor.exe MyGame.uproject /Game/Maps/Level1 -game -nullrhi -deterministic -benchmark -fps=30     -BlueprintTrace=E:/tmp/trace.json -TraceMap=Source/MyGame/Converted [-TraceFrames=600] [-TraceEvery=15] [-TraceInput=input.json]
```

- **What is recorded:** every `-TraceEvery` frames, each actor whose class is a converted asset in any of its forms. Each sample has its location, rotation, `hidden`, and its variables under their Blueprint names (`-TraceMap` holds the manifests). After `-TraceFrames` frames the file is written and the game exits.
- **Keys:** level actors are keyed by level and name, spawned ones by asset and spawn order.
- **Input:** `-TraceInput` injects Enhanced Input actions on given frames, `{ "events": [ { "frame": 30, "frames": 5, "action": "/Game/Input/IA_Move.IA_Move", "value": [1, 0, 0] } ] }`.
- **`-TraceAll`:** traces every other Blueprint actor too, with the variables its Blueprints declare. That covers what uses the converted types without being converted itself.
- **`-TraceFlushLoads`:** finishes asynchronous loads at the start of every frame, so a load completes in the same frame in every run. Without it, disk and cache timing decide.
- **In PIE:** `BlueprintTrace.Start <out.json>` on the console traces the session (the frame settings come from the command line).

---

## 9. Diffs in the BlueprintReview app, and view sync

The `BlueprintReviewLink` module (editor only) does two things:

- **Diff hand-over.** Every Blueprint diff the editor starts (Content Browser *Diff Selected*, Revision Control *Diff Against Depot*, the History window, the Blueprint editor's *Diff* menu) opens `blueprintreview-app --diff` instead of the built-in window, with the revision labels (git hash, or `#3 (CL 120)` for Perforce). It registers a diff action for `UBlueprint` and each Blueprint class (`IAssetTools::RegisterAssetTypeActions`); opening, thumbnails and menus stay with the class's asset definition. The built-in window is used when the app is not found, a side has unsaved changes (the app reads files, not the editor's memory) or the app fails to start. The app is looked up in *Editor Preferences > Plugins > Blueprint Review Link > App Path*, the plugin's `Tools` folder, then `PATH`. `BlueprintReview.Diff /Game/A /Game/B` in the console starts the same diff.
- **View sync.** A server on `127.0.0.1` (first free port from 17630) writes `Saved/BlueprintReview/link.json` (port, token). The app's *Sync* toggle in the diff view uses it: `GET /view?asset=/Game/X` is the active graph of the editor that has that Blueprint open, never another editor (without `asset`: the editor in the active window) (asset, graph names, view centre and zoom in graph coordinates, selected node GUIDs), `POST /view` opens the Blueprint if needed, shows the graph, centres the view and selects nodes. Every request needs `Authorization: Bearer <token>`. Switch it off with *Sync Server* in the same preferences; *Poll Interval Ms* and *Send Interval Ms* there set how often the app asks for the editor's view and sends its own (`GET /status` returns them as `poll_ms` and `send_ms`; the app reads them when Sync is switched on). An editor in the background slows to 3 frames a second (*Editor Preferences > Performance > Use Less CPU when in Background*), and the server only answers on a frame, so each request would wait up to a third of a second; *Keep Editor Awake* (on by default) keeps the editor at its normal rate while the app is asking, and lets it slow down two seconds after the app stops (`UEditorEngine::ShouldDisableCPUThrottlingDelegates`).

## Troubleshooting

| Problem | Fix |
|---|---|
| The menu entry is missing | Check the plugin is enabled (**Edit → Plugins**). The entry only appears when the selection includes a Blueprint. |
| "The following modules are missing or built with a different engine version" | A prebuilt copy for another engine version is installed. Rebuild for your engine, or let a C++ project compile it. |
| Blueprint-only project can't build the plugin | Use a prebuilt copy (section 1). |
| `LogBlueprintExport: Error: ...: not found or not a Blueprint` | Check the path in `-Assets`. It's the content path (`/Game/Folder/BP_Name`), not a file path. |
| Exit code is 1 but everything exported | Expected when other errors were logged; see section 3. |
| Some values show as `null` / `None` | The referenced asset or class didn't load. It's missing from the project, or provided by a plugin that isn't enabled. |
