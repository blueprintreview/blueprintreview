# `blueprintreview` command-line guide

`blueprintreview` reads Unreal Engine Blueprint `.uasset` files directly. You don't need the UE editor. It works with **uncooked** editor assets saved by UE 4.25 through 5.8. Cooked (packaged-game) assets have their graphs removed, so `blueprintreview` rejects them.

| Command | What it does |
|---|---|
| [`blueprintreview dump`](#blueprintreview-dump--blueprint-as-json) | Prints the Blueprint as JSON: variables, graphs, components and class defaults |
| [`blueprintreview diff`](#blueprintreview-diff--what-changed-between-two-versions) | Shows what changed between two versions of a Blueprint, as Markdown for a PR comment or as JSON |
| [`blueprintreview render`](#blueprintreview-render--graphs-as-svg-images) | Draws graphs as SVG images; with `--against`, the changed graphs' old and new versions side by side |
| [`blueprintreview info`](#blueprintreview-info--package-summary) | Prints package facts: which engine saved it, and what it contains |
| [`blueprintreview props`](#blueprintreview-props--raw-property-tree) | Prints every stored property of the objects in the file, for low-level inspection |
| [`blueprintreview check`](#blueprintreview-check--compare-with-a-json-export) | Compares `blueprintreview`'s reading of an asset with a JSON export of it, e.g. from the UE plugin |
| [`blueprintreview copy`](#blueprintreview-copy--nodes-as-copy-text) | Prints graph nodes as the editor's copy text, to paste into the UE Blueprint editor or blueprintue.com |
| [`blueprintreview schema`](#blueprintreview-schema--json-schema) | Prints the JSON Schema of the `dump` format |
| [`blueprintreview git-setup`](#blueprintreview-git-setup--set-git-up-for-blueprints) | Sets git up for Blueprints: `git diff` shows them as text, `git difftool` opens the viewer's diff |
| [`blueprintreview p4-setup`](#with-perforce-p4v) | Prints how to make P4V open `.uasset` diffs in the viewer |
| [`blueprintreview textconv`](#blueprintreview-textconv--blueprints-in-git-diff) | Prints a Blueprint as plain text, so `git diff` shows what changed in a `.uasset` |
| [`blueprintreview pr-report`](#blueprintreview-pr-report--a-pull-requests-blueprint-changes) | Reports the Blueprints a branch changes as one Markdown PR comment, with links to the web viewer's diff (the [GitHub Action](ACTION.md) runs it) |

Every command takes a path to one `.uasset` file (`copy` also takes a JSON export), except `diff`, which takes two `.uasset` or JSON files (`render --against` too), `pr-report`, which reads both versions from git, and `git-setup`, which takes none. Use `blueprintreview <command> --help` for the options.

## Getting `blueprintreview`

Each [release](https://github.com/blueprintreview/blueprintreview/releases) has prebuilt binaries for Linux (x64, arm64), Windows (x64) and macOS (arm64): `blueprintreview-<target>.tar.gz` holds the CLI alone, and `blueprintreview-app-<target>` (`.zip` on Windows, `.tar.gz` elsewhere) holds the [viewer app](#the-viewer-app-blueprintreview-app) and the CLI together. Unpack it into a folder of its own; `SHA256SUMS` has the checksums.

To build from source instead, you need the Rust toolchain. The repo's `rust-toolchain.toml` picks the right version automatically.

```sh
# Run straight from the repo (builds on first use):
cargo run -p blueprintreview-cli -- <command> <args>

# Or build a standalone binary:
cargo build --release -p blueprintreview-cli
#   -> target/release/blueprintreview(.exe)

# Or install it onto your PATH (~/.cargo/bin):
cargo install --path crates/blueprintreview-cli
```

The examples below assume `blueprintreview` is on your PATH. If it isn't, replace `blueprintreview` with `cargo run -q -p blueprintreview-cli --`.

---

## `blueprintreview dump` — Blueprint as JSON

```sh
blueprintreview dump <PATH>
```

Prints a Blueprint summary as JSON. This is the format the viewer and the diff tools use.

Enum, user-defined struct and data table assets (`UserDefinedEnum`, `UserDefinedStruct`, `DataTable`) work too: the JSON's `blueprint_class` names the asset class, and the asset is in `enums` (its own entry), `structs` (the one at its own path, plus the structs it uses) or `data_table` (`row_struct` and the `rows`, each row's values by the struct's property names). `diff`, `textconv`, `pr-report` and the viewer handle them like Blueprints: a data table's rows are compared by name, field by field.

```sh
blueprintreview dump fixtures/4.26/BP_gem.uasset
```

```json
{
  "schema_version": 4,
  "path": "/Game/Hour_of_Code/Blueprints/BP_gem",
  "blueprint_class": "Blueprint",
  "parent_class": "/Script/Engine.Actor",
  "blueprint_type": "BPTYPE_Normal",
  "variables": [
    {
      "name": "Vis_on",
      "guid": "5217527F41AB0DCE9AA8C78F2EFB40DE",
      "type": { "category": "bool" },
      "friendly_name": "Vis On",
      "category": "Default",
      "property_flags": 65541,
      "metadata": [["MultiLine", "true"]]
    }
  ],
  "graphs": [
    { "name": "EventGraph", "kind": "event_graph" },
    { "name": "UserConstructionScript", "kind": "function" }
  ],
  "components": [
    {
      "name": "DefaultSceneRoot",
      "class": "/Script/Engine.SceneComponent",
      "properties": [ ... ]
    }
  ],
  "defaults": [ ... ]
}
```

What the fields mean:

| Field | Contents |
|---|---|
| `path`, `blueprint_class`, `parent_class` | The asset path, the asset type (`Blueprint`, `WidgetBlueprint`, `AnimBlueprint`, ...), and the class this Blueprint extends. |
| `description`, `interfaces` | The Blueprint's description and the interfaces it implements. Interface graphs (`"kind": "interface"`) name the interface they implement in `interface`. |
| `class_settings` | The rest of the editor's *Class Settings* that this Blueprint changed: `BlueprintDisplayName`, `BlueprintCategory`, `HideCategories`, `bDeprecate`, `bGenerateAbstractClass`, ... Like `defaults`, only saved values are listed. Generated and editor bookkeeping properties (`BlueprintGuid`, `LastEditedDocuments`, ...) are left out. |
| `variables` | Member variables in the order the editor lists them. Each has its type (`category` such as `bool`, `real`, `object`, `struct`, plus the class/struct path and the container kind), category, flags and metadata. |
| `graphs` | Every graph by name and kind (`event_graph`, `function`, `macro`, `delegate_signature`, `interface`), with its **nodes**. Each node lists its class, position, comment, saved settings (e.g. which function it calls) and **pins**: name, direction, type, default value, and connections (`"linked_to": ["<nodeGuid>:<pinId>"]`). Nodes are sorted by GUID so the output is stable. Editor-only display data (node titles, pin labels) is only in plugin exports; see [PLUGIN.md](PLUGIN.md). Collapsed sub-graphs aren't included yet. |
| `components` | Components added in the Components panel. Each has its class, its parent component (`parent`), its attach socket, and the values that differ from the component class defaults. |
| `defaults` | Class Defaults values that **differ from the parent class**. UE doesn't store unchanged values, so a missing entry means "same as the parent". |
| `enums` | The enums the Blueprint uses (variable and pin types, `BitmaskEnum` metadata), each with its entries: the stored name (`E_Gait::NewEnumerator1`), the display name (`Run`) and the value. A Blueprint only stores entry names, so `blueprintreview` reads the enum assets themselves (see [Enum assets](#enum-assets)). Engine enums (`/Script/...`) are only in plugin exports. |
| `warnings` | Problems `blueprintreview` worked around, such as an enum asset it couldn't find. |

### Enum assets

`dump`, `check` and `blueprintreview-app` look for each enum asset in the project the Blueprint came from. `/Game/Blueprints/BP_Hero` opened from `MyGame/Content/Blueprints/BP_Hero.uasset` means `/Game/Data/E_Gait` is `MyGame/Content/Data/E_Gait.uasset`. If the asset isn't there, they look next to the Blueprint file (`E_Gait.uasset`). An enum that isn't found is left out of `enums` and listed in `warnings`. When diffing (`blueprintreview diff`, `render --against`, the viewer's diff view), the old version also looks in the new version's project, because a difftool hands over the old version as a temp copy outside the project.

Property values keep their UE type, e.g. `{"Bool": true}`, `{"Float": 0.5}`, `{"Object": "/Game/Meshes/SM_Fan.SM_Fan"}` (a full object path, or `null` for "none"), or `{"Raw": "0a1b..."}` for the few value types that `blueprintreview` can't decode yet.

Useful patterns:

```sh
blueprintreview dump BP_Hero.uasset > BP_Hero.json            # save to a file
blueprintreview dump BP_Hero.uasset | jq '.variables[].name'  # list variable names (needs jq)
```

To see what changed between two versions, use [`blueprintreview diff`](#blueprintreview-diff--what-changed-between-two-versions) rather than diffing two dumps.

`dump` fails if the file is a valid asset but not a Blueprint, e.g. a user-defined struct:

```
Error: fixtures/5.7/CollectableObjectData.uasset

Caused by:
    a UserDefinedStruct asset, not a Blueprint
```

---

## `blueprintreview diff` — what changed between two versions

```sh
blueprintreview diff <OLD> <NEW> [--format md|json] [--cosmetic] [--exit-code]
```

Compares two versions of a Blueprint and lists what changed: class settings, variables, components, class defaults, timelines, enums, user-defined structs, graphs, and in each graph the nodes, pins and links. `OLD` and `NEW` are `.uasset` files or JSON exports (from `blueprintreview dump` or the UE plugin), in any combination.

Items are matched by identity, not by position, so reordering isn't a change:
- variables by their GUID, so a renamed variable shows as one modified variable (`HP (was Health)`);
- graphs by their `GraphGuid` (then name and kind), so renamed functions are recognised too;
- nodes by `NodeGuid`, pins by `PinId` (then by name and direction, for nodes that regenerate pin IDs);
- components, properties and struct fields by name. Equal-length arrays are compared element by element (`ResponseArray[3].Response`);
- links as wires from an output pin to an input pin, each reported once;
- the changes a Blueprint makes to components it inherits from a Blueprint parent (*Inherited components*), by the parent's component GUID, then by name; each property that changed is listed;
- timelines by their GUID (so a renamed timeline is one change), then by name; their tracks by kind and name, and each curve's keys by time. A changed key reads `1 s: 0.5 (cubic, auto)`: time, value, interpolation and tangent mode. The editor's diff doesn't compare timelines;
- user-defined structs' fields by GUID, then name: a field renamed, retyped, added, removed or moved is listed. Like enums, a struct on one side only isn't reported (the other side couldn't load its asset).

Moving or resizing a node, a comment's colour or bubble, a pin's *Advanced* flag and node object names are **cosmetic**. So is **re-routing**: a wire moved onto reroute nodes (or off them, or onto other ones), and the reroute nodes added or removed for it, when every connection from a real output to a real input is the same before and after. The editor reports those as added nodes and links; here they're listed (with `--cosmetic`) as `re-routed`. A reroute node that changes what connects to what is a real change. They're left out of the Markdown unless you pass `--cosmetic`, and they don't count for `--exit-code`. Fields only the editor computes (titles, colours, pin display names), `generator`, `warnings` and property type strings are ignored. Enums listed on one side only are ignored too: enums are separate assets, so this means one side couldn't load them (or, for engine `/Script/...` enums, that only the UE plugin exports them). Changed entries of an enum on both sides are reported. Values that loading recomputes don't count either, e.g. a cast node's `PureState` or an *Evaluate Chooser* node's `bShowModeSetting`. Texts compare by their string, as in the editor, so a regenerated localization key isn't a change.

Two saves by different engine versions (say the same asset before and after a project upgrade) differ in ways the editor never shows, since it migrates old data and rebuilds every node on load. These are ignored too:
- properties newer engines migrate or dropped: `bIsPureFunc` (now `bDefaultsToPureFunc`), `bIsConstFunc`, `bIsInterfaceCall`, `bIsPureGet` (now `CurrentVariation`), a promotable operator's `OperationName`, empty structs;
- renamed engine functions, through the engine's function redirects (`BreakVector3f` → `BreakVector`);
- on calls of engine functions and on Make / Break nodes of engine structs, pins the newer signature or struct added and pin types it changed (`float` → `double`), as well as a struct default value's new fields. These count as cosmetic. The pins of the Blueprint's own functions and structs are compared as saved;
- `ShowPinForProperties` metadata (tooltips, friendly names) and entries for struct members only one version has.

### `--format md` (default)

GitHub-flavoured Markdown, made for a PR comment. A summary line is followed by tables; each graph is a collapsed `<details>` block, with its added and removed links in a nested one. Long values are cut at 80 characters and tables at 60 rows.

```sh
blueprintreview diff old/BP_MovingPlatform.uasset BP_MovingPlatform.uasset
```
```
### `BP_MovingPlatform`

variables: 5 added, 2 removed · components: 7 added, 2 removed, 1 modified · graphs: 2 modified · nodes: 70 added, 37 removed, 6 modified · links: 78 added, 38 removed · 5 class setting/defaults changed

**Variables**

| | Variable | Changes |
|---|---|---|
| ➕ | `Speed` | real:double |
| ➖ | `Trigger` | object<Actor> |

**Graphs**

<details><summary>✏️ Event graph <b>EventGraph</b>: nodes 57 added, 37 removed, 5 modified, links +61 −38</summary>

| | Node | Changes |
|---|---|---|
| ➖ | Get Bone Transform |  |
| ➕ | Delay |  |
| ✏️ | Toggle Platform | pin `bEnabled` default_value: `false` → `true` |
...
```

A file compared with itself prints ``### `BP_Foo`: no differences``. If only cosmetic things changed, it prints ``### `BP_Foo`: only cosmetic changes (3 nodes moved or resized)``.

### `--format json`

The structured diff, for scripts and other tools: `summary` (added/removed/modified/cosmetic counts per kind), `fields`, `variables`, `components`, `defaults`, `timelines`, `enums`, `structs` and `graphs`. Every graph of either side is listed, unchanged ones too, with `old_index` / `new_index` into each side's `graphs`. Each graph lists its changed `nodes` (by `guid`, with `status`, `changes`, `pins` and owned `sub_graphs`) and its added and removed `links`. A change is `{"path", "old", "new", "cosmetic"}`, with values as UE's export text (`(X=1.000000,Y=2.000000)`); a missing `old` or `new` means the value was absent or empty on that side.

### Comparing with an earlier commit

```sh
git show HEAD~1:Content/BP_Hero.uasset > old.uasset
blueprintreview diff old.uasset Content/BP_Hero.uasset
```

(If the repo stores assets in Git LFS, `git show` prints the LFS pointer instead of the asset. Use `git show HEAD~1:Content/BP_Hero.uasset | git lfs smudge > old.uasset` instead.)

`--exit-code` exits with `1` when there are changes other than cosmetic ones, like `git diff --exit-code`. Without it, `diff` exits with `0` whatever it finds.

To make plain `git diff` and `git difftool` understand `.uasset` files, see [`blueprintreview textconv`](#blueprintreview-textconv--blueprints-in-git-diff) and [the diff view as a difftool](#as-a-git-difftool).

---

## `blueprintreview render` — graphs as SVG images

```sh
blueprintreview render <PATH> [--graph <NAME>] [--against <OLD>] [-o <DIR>]
```

Draws graphs as SVG files, `<asset>.<graph>.svg` in `DIR` (default: the current folder), and prints each file's path. `PATH` is a `.uasset` or a JSON export. `--graph` picks one graph by name; `-o -` writes that one graph to stdout.

With `--against <OLD>`, it draws each graph that changed (see [`blueprintreview diff`](#blueprintreview-diff--what-changed-between-two-versions)) with the old version on the left and the new one on the right. Both sides cover the same area, so unchanged nodes line up. Changed nodes get an outline in the editor's diff colours (green added, red removed, cyan modified, grey-violet moved), and added and removed wires get a halo. A graph that exists on only one side shows "Not in the old/new version" on the other. `--graph` picks a graph by its new or old name, even an unchanged one.

```sh
blueprintreview render BP_Spawner.uasset -o renders
blueprintreview render BP_MovingPlatform.uasset --against old/BP_MovingPlatform.uasset -o renders
```
```
renders/BP_MovingPlatform.EventGraph.svg
renders/BP_MovingPlatform.UserConstructionScript.svg
```

The layout is the viewer's: node sizes come from the same text measurement (Roboto, shaped as egui shapes it). The drawing is a flat vector version of the editor's look: rounded node bodies, title colour bands, vector pin icons, wires coloured by type, comment boxes. It uses no editor art, so header and corner icons are left out. Text names the Roboto font and falls back to Helvetica/Arial where Roboto isn't installed. Graphs owned by nodes (collapsed graphs) aren't drawn separately yet.

---

## `blueprintreview info` — package summary

```sh
blueprintreview info <PATH> [--imports] [--exports]
```

Prints which engine saved the file, its format versions, and how many objects it contains. Use it to check an asset's real engine version. That may be older than the project's, because assets aren't re-saved when a project is upgraded.

```sh
blueprintreview info fixtures/4.26/BP_gem.uasset
```

```
package:      /Game/Hour_of_Code/Blueprints/BP_gem
saved by:     4.25.4-14469661+++UE4+Release-4.25
file version: legacy -7 / UE4 518 / licensee 0
custom vers:  8
names:        251
imports:      59
exports:      34
```

- **saved by**: the engine version and changelist that last saved the asset.
- **file version**: the package format versions. These decide how the rest of the file is read.
- **imports**: objects the asset references from elsewhere, such as engine classes or other assets.
- **exports**: objects stored inside this asset, such as the Blueprint, its graphs, nodes and components.

### `--exports`

Lists every object stored in the file. The columns are: index, class, size in bytes, and full path.

```sh
blueprintreview info fixtures/4.26/BP_gem.uasset --exports
```

```
exports:
      1  Function                         1247  /Game/Hour_of_Code/Blueprints/BP_gem.BP_gem_C:ExecuteUbergraph_BP_gem
      3  BlueprintGeneratedClass           851  /Game/Hour_of_Code/Blueprints/BP_gem.BP_gem_C
      4  BP_gem_C                          149  /Game/Hour_of_Code/Blueprints/BP_gem.Default__BP_gem_C
      5  Blueprint                        1097  /Game/Hour_of_Code/Blueprints/BP_gem.BP_gem
      7  EdGraph                           217  /Game/Hour_of_Code/Blueprints/BP_gem.BP_gem:EventGraph
      9  K2Node_CallFunction               884  /Game/Hour_of_Code/Blueprints/BP_gem.BP_gem:EventGraph.K2Node_CallFunction_0
    ...
```

The index in the first column is the number to pass to `blueprintreview props --export`.

### `--imports`

Lists the external objects the asset depends on. The index is negative, as UE stores it.

```
imports:
     -6  Class                        /Script/Engine.Actor
     -7  Class                        /Script/Engine.Blueprint
    ...
```

Both flags can be combined.

---

## `blueprintreview props` — raw property tree

```sh
blueprintreview props <PATH> [--export <N>]
```

Prints every stored property of each object in the file, with its UE type. This is a debugging view, much lower-level than `dump`. Use it to see what a particular node or component actually stores. Without `--export` it prints every object, which can be long.

```sh
blueprintreview props fixtures/4.26/BP_gem.uasset --export 21
```

```
[21] K2Node_VariableSet /Game/Hour_of_Code/Blueprints/BP_gem.BP_gem:EventGraph.K2Node_VariableSet_0
  VariableReference: StructProperty(MemberReference) = MemberReference {
    MemberParent: ObjectProperty = /Game/ThirdPersonBP/Blueprints/ThirdPersonGameMode.ThirdPersonGameMode_C
    MemberName: NameProperty = Name("gemCount")
    MemberGuid: StructProperty(Guid) = Str("B415C9B04BB71E002B953FA5E731CFF4")
  }
  SelfContextInfo: ByteProperty(ESelfContextInfo) = Enum("ESelfContextInfo::NotSelfContext")
  NodePosX: IntProperty = Int(1104)
  NodePosY: IntProperty = Int(240)
  NodeGuid: StructProperty(Guid) = Str("321A71594F9B06E9C1CF7D8D2311025D")
```

Values that `blueprintreview` can't decode are shown as `<raw N bytes>`. A node's pins aren't listed here, because they're stored outside the property list. `blueprintreview dump` shows them.

---

## `blueprintreview check` — compare with a JSON export

```sh
blueprintreview check <PATH> <EXPECTED.json> [--nodes]
```

Reads the asset and compares the result with a JSON file in the same format. The usual source is the **BlueprintExporter** UE plugin, which exports what the editor itself sees (`plugins/BlueprintExporter/README.md`). It prints every difference with the place where it occurs, and exits with `1` if there are any.

```sh
blueprintreview check BP_Fan.uasset Exports/Game/StackOBot/Blueprints/GameElements/BP_Fan.json
```
```
BP_Fan.uasset: matches Exports/Game/StackOBot/Blueprints/GameElements/BP_Fan.json
```
```
defaults[Affect Distance]: Float(500.0) != Float(750.0)
components[Mesh]: only in second
2 difference(s)
```

The comparison ignores:
- fields only the editor can compute (node titles, colours, pin display names);
- the `generator` field;
- property type strings;
- enums present in only one of the two files (each tool can only include the enum assets it can load).

Floats compare with a small tolerance, and values `blueprintreview` couldn't decode match anything. `--nodes` also compares graph nodes and pins: classes, positions, settings, pin types, defaults and links.

For node comparisons, export with `-SaveBeforeExport` (see [PLUGIN.md](PLUGIN.md)). Some nodes rebuild themselves each time the editor loads them, and get fresh pin IDs. Saving first makes the file and the export describe the same state.

`--graph <NAME>` compares one graph's nodes and pins only. `EXPECTED.json` is then a single graph, as the plugin's `-ImportT3D` mode writes it. `PATH` may be a single graph's JSON too, to compare two pasted graphs.

---

## `blueprintreview copy` — nodes as copy text

```sh
blueprintreview copy <PATH> --graph <NAME> [--node <GUID>]...
```

Prints the nodes of a graph as the text the Blueprint editor puts on the clipboard when you copy nodes (T3D). Paste it into a graph in the UE editor, or into blueprintue.com. `PATH` is a `.uasset` or a JSON export. Without `--node` it copies every node of the graph. Each `--node` names one by its `NodeGuid` (as in `blueprintreview dump`).

```sh
blueprintreview copy BP_Door.uasset --graph EventGraph > door.t3d
blueprintreview copy BP_Door.uasset --graph EventGraph --node 1A2B... --node 3C4D...
```

What gets copied:
- **Nodes** with their saved settings, pins, pin defaults and user-added pins (event and function parameters, collapsed-graph inputs and outputs).
- **Links** between the copied nodes. Links to nodes you didn't copy are left out, as the editor drops them on paste.
- **Collapsed graphs** (and other graphs a node owns) with all their nodes.

The editor rebuilds pasted nodes, so values the files don't store come back on paste: pin tooltips, and the localization keys of pin labels. A node whose pins `blueprintreview` couldn't read is copied without pins; the editor gives it its default pins, without links. Those are listed as warnings on stderr.

---

## `blueprintreview schema` — JSON Schema

```sh
blueprintreview schema > blueprint.schema.json
```

Prints the JSON Schema of the format `dump` writes and `check` reads. The repo keeps a copy in `schema/blueprint.schema.json`, and a test fails if it goes stale.

---

## `blueprintreview edit` — Blueprint edit plans (experimental)

> **Experimental:** Blueprint editing (the *Blueprint Edit Automation* plugin, edit plans and
> `blueprintreview edit`) is experimental. The plan format and the ops may still change between releases, and
> an edit can fail or need finishing by hand in the editor. Keep the project under source control and review
> every change before you commit it.

```sh
blueprintreview edit validate plan.json --project MyGame.uproject          # offline checks; exit 1 on errors
blueprintreview edit apply plan.json --project MyGame.uproject --dry-run   # through the editor's edit server, or a commandlet
blueprintreview edit apply plan.json --project MyGame.uproject             # applies (commandlet: saves; live: leaves it for review, --save saves)
blueprintreview edit ops                                                   # every op and its keys (--json: the plugin's -ListOps form)
blueprintreview edit schema > edit-plan.schema.json                        # the plan's JSON Schema
```

These drive the BlueprintEditAutomation plugin: `BlueprintEditPlan/2` plans that edit Blueprints, user enums,
structs and data tables. A plan uses the same types, node GUIDs and `nodeGuid:pinId` pins that `dump` prints, so
read an asset with `dump`, write the plan, then `validate` and `apply` it. The plan format, the ops and the hosts
are in [EDITING.md](EDITING.md).

- **`validate`** repeats the plugin's checks: unknown ops and keys, the kind of each key, PinType shapes, the rules
  between keys. With `--project` (or `--content`, the folder `/Game` maps to), it also loads each edited asset and
  checks that the node GUIDs, pin ids, graphs, variables and aliases the plan names exist when each op runs. A pin
  name the model can't place is a warning: the editor also matches display names.
- **`apply`**:
  - validates first;
  - uses the open editor's edit server when `Saved/BlueprintEdit/server.json` says one runs, and a headless
    `-run=BlueprintEdit` commandlet otherwise (`--live` / `--commandlet` force one, `--engine` picks the engine);
  - checks that the report carries this run's nonce and succeeded, and exits with 1 otherwise;
  - prints each op's result, the GUIDs of what it created, the compile messages, and a `blueprintreview diff`
    command line for each Blueprint's before/after JSON (`--json` prints the report itself).

`edit` needs no license.

The repo keeps the schema in `schema/edit-plan.schema.json` and the plugin's op table in `test-data/edit-ops.json`;
tests fail if either goes stale.

---

## `blueprintreview git-setup` — set git up for Blueprints

```sh
blueprintreview git-setup [--repo DIR] [--dry-run]   # one repository (default: the current one)
blueprintreview git-setup --global [--dry-run]       # all your repositories
```

Writes everything `git diff` and `git difftool` need for Blueprints:
- `diff.blueprintreview.textconv` and `cachetextconv`, so `git diff`, `git log -p` and `git show` print Blueprint changes as text ([`textconv`](#blueprintreview-textconv--blueprints-in-git-diff));
- `difftool.blueprintreview.cmd`, so `git difftool -t blueprintreview` opens the viewer's [side-by-side diff](#as-a-git-difftool);
- the line `*.uasset diff=blueprintreview` at the end of `.gitattributes`. It goes at the end because the usual LFS line (`*.uasset filter=lfs diff=lfs merge=lfs -text`) also sets `diff`, and later lines win. LFS still works: git smudges each version before the text conversion. With `--global`, the line goes to your global attributes file instead (`core.attributesFile`, else `~/.config/git/attributes`).

The commands run `blueprintreview-app` from the CLI's own folder (an install, or `target/release`) and the CLI itself by its full path, so neither has to be on `PATH`. Running the setup again changes nothing, and `--dry-run` shows what it would do:

```
set difftool.blueprintreview.cmd = "C:/Program Files/BlueprintReview/blueprintreview-app.exe" --diff "$LOCAL" "$REMOTE" --path "$MERGED"
set diff.blueprintreview.textconv = "C:/Program Files/BlueprintReview/blueprintreview.exe" textconv
set diff.blueprintreview.cachetextconv = true
added `*.uasset diff=blueprintreview` to E:\Projects\MyGame\.gitattributes
```

Commit `.gitattributes`, so the rest of the team only needs to run `blueprintreview git-setup --global` once.

## `blueprintreview textconv` — Blueprints in `git diff`

```sh
blueprintreview textconv <PATH>
```

Prints a Blueprint as plain text, one fact per line. It's meant as a git `textconv` driver: git runs it on both versions of a `.uasset` and diffs the text, so `git diff`, `git log -p` and `git show` print what changed instead of `Binary files differ`.

[`blueprintreview git-setup`](#blueprintreview-git-setup--set-git-up-for-blueprints) sets this up. By hand, once per clone (or with `--global`), plus the line in `.gitattributes`:

```sh
git config diff.blueprintreview.textconv "blueprintreview textconv"
git config diff.blueprintreview.cachetextconv true      # optional: cache the text per blob
echo '*.uasset diff=blueprintreview' >> .gitattributes
```

```diff
@@ -47,7 +47,7 @@ float + float {0BFDF466}
   OperationName = "Add"
   bDefaultsToPureFunc = True
   FunctionReference = (MemberParent="/Script/Engine.KismetMathLibrary",MemberName="Add_DoubleDouble")
-  in A: real:double = 172.000000
+  in A: real:double = 999.0
   in B: real:double
   out ReturnValue: real:double
     -> Set Relative Location {3F81561B}.NewLocation_Y
```

The text has a header (path, class, parent, type, description, interfaces, and one `Setting <name> = <value>` line per changed class setting), then `[Variables]`, `[Components]` (as a tree), `[Class defaults]` and one `[Graph <name> (<kind>)]` block per graph. Each node is its title and the first 8 characters of its GUID, then its saved properties, its pins (`in` / `out`, type, and the default value of an unconnected input) and its links, listed once under the output pin (`-> Print String {5E6F7A8B}.execute`). Owned graphs (collapsed graphs, ...) are nested under their node. Cosmetic data is left out, as in [`blueprintreview diff`](#blueprintreview-diff--what-changed-between-two-versions): moving or resizing a node, or recolouring a comment, changes nothing in the text. Hidden pins without links are left out too.

It never fails, so it can't break `git diff`: a file that isn't a Blueprint prints its package name and exports, a broken file prints the error, and an LFS pointer (an asset that `git lfs pull` hasn't fetched) prints a note and the pointer. It also reads JSON exports. git runs the LFS smudge filter before `textconv`, so assets stored in Git LFS work when `git lfs` is installed.

---

## `blueprintreview pr-report` — a pull request's Blueprint changes

Reports every Blueprint that changed between two commits as one Markdown document, sized for a PR comment. The [GitHub Action](ACTION.md) runs it on each pull request; you can run it locally too:

```sh
blueprintreview pr-report --base origin/main --head HEAD
blueprintreview pr-report --base origin/main --head HEAD --repo octo/game --pr 12 \
    --viewer-url https://example.com/view/ -o comment.md --full-output report.md --svg-dir svgs
```

- The changes are the head's since its **merge base** with `--base` (`git diff --name-status -M`), so commits that landed on the base branch afterwards don't show. The repository needs that history (in GitHub Actions, check out with `fetch-depth: 0`).
- Both versions are read from git (`git cat-file`); Git LFS pointers are replaced by their objects through `git lfs smudge`, which downloads them when needed. Enum assets and Blueprint parents are looked for in the working tree (`--dir`, default `.`), as `diff` does.
- `--pathspec` picks the files (git pathspecs, repeatable; default `*.uasset`).
- The report starts with the line `<!-- blueprintreview-report assets=N -->`, which the Action uses to find its comment again. Then a table: each changed Blueprint with its path, whether it was added, modified, deleted or renamed, a one-line summary and, given `--viewer-url` and `--repo`, a link to it in the [PR viewer](#pr-viewer-view). Then each modified Blueprint's `diff --format md`, collapsed. Changed assets that aren't Blueprints (textures, enums, levels) are listed in one line; a file that can't be read gets a line with the reason, and the report still succeeds.
- `--max-bytes` (default 65000, under GitHub's 65,536-character comment limit) caps the comment: the largest diffs are left out first, with a note, then table rows. `--full-output` writes the whole report too, e.g. for the job summary. `-o` writes the comment to a file instead of stdout.
- `--svg-dir` writes the changed graphs of each modified Blueprint side by side (as `render --against`), in folders that mirror the assets'.
- `--cosmetic` lists cosmetic changes too.

The viewer links carry `repo`, the commits (`base` for the old side, `head` for the new one, only those the file exists on), `path` (the new path, or the old one of a deleted file), `old_path` for a rename and `pr`, e.g. `https://example.com/view/?repo=octo/game&base=<sha>&head=<sha>&path=Content/BP_Cog.uasset&pr=12`.

## The viewer app (`blueprintreview-app`)

`blueprintreview-app` is the desktop viewer. It opens a `.uasset` or a JSON export (from `blueprintreview dump` or the [UE plugin](PLUGIN.md)), or a whole folder of them (see [Folders](#folders)), and lays it out like the Blueprint editor:
- *Components* and *My Blueprint* on the left;
- the graphs as tabs in the middle;
- *Details* on the right, for whatever is selected.

It comes with the CLI in each release's `blueprintreview-app-<target>` archive (see [Getting `blueprintreview`](#getting-blueprintreview)), or build it with `cargo build --release -p blueprintreview-app` (`target/release/blueprintreview-app(.exe)`). Keep it in the same folder as `blueprintreview`: `git-setup` looks for it there.

### Enums, structs and data tables

Opening an enum, user-defined struct or data table asset shows it as a table, as the editor's asset editors list them: an enum's entries (name, display name, value), a struct's fields (name, type, default value, tooltip), a data table's rows with a column per field of the row struct (unsaved values show the field's default). The row struct and the enums are looked up in the project, as for Blueprints. In the diff view the two versions' tables are side by side, each row coloured by what happened to it (added, removed, changed), and *Changes* lists the changed entries, fields or rows.

An enum asset dropped onto the window while a Blueprint is open still only supplies its display names to that Blueprint.

### The graph canvas

Graphs are drawn like the UE5 editor's, read-only:
- **Nodes:** headers coloured by node kind, with header icons; compact operator nodes (`×`, `+`, `==`); variable getters as pills and setters with a `SET` header; reroute knots; comment boxes.
- **Pins:** pin-type icons and colours, filled when connected. Unconnected inputs show their default value. Unconnected advanced pins are collapsed behind the ▾ expander.
- **Wires:** coloured by type. Exec wires are white and thicker.
- **Zoom:** shows less detail as you zoom out, like the editor (no pin labels, then title boxes only).

Graph controls:
- **Pan:** drag with the right or middle mouse button. A left-button drag that starts on a node pans too, since nodes can't be moved.
- **Zoom:** use the mouse wheel. Zooming stops at 1:1 unless you hold Ctrl.
- **Fit the graph:** press F (or Home) while the pointer is over the graph. A graph also fits itself when its tab opens.
- **Select nodes**, as in the editor:
  - click a node to select it; Shift+click adds a node, Ctrl+click toggles one, and clicking empty space clears the selection;
  - drag a box over empty space with the left button to select every node it touches (comments only by their title bar); hold Shift to add to the selection, Ctrl to toggle, Alt to remove;
  - Ctrl+A selects every node.
- **Copy nodes:** Ctrl+C (or *Copy* in the right-click menu) puts the selected nodes on the clipboard as the editor's copy text, the same as [`blueprintreview copy`](#blueprintreview-copy--nodes-as-copy-text). Paste it into a graph in the UE editor, or into blueprintue.com. The toolbar confirms how many nodes were copied, with any warnings.
- **Node details:** with one node selected, *Details* shows its class, pins and saved properties. As in the editor, some nodes show something else first, with the node's own details under the collapsed *Graph Node* categories:
  - a get or set of one of the Blueprint's variables shows that variable;
  - a function's entry or result node shows the function's details;
  - a custom event shows the event's details.
- **Pin details:** hover a pin to see its type and default value.
- **Jump to a definition:** double-click a node that uses something the Blueprint defines, as in the editor. A call of one of its functions (or interface functions) opens that function's graph, a call of a custom event selects the event, a macro opens the macro graph, and a collapsed graph (or math expression) opens in its own tab. Calls of engine functions and of parent Blueprints' functions stay put.
- **Graph tabs:** clicking a graph in *My Blueprint* opens it as a tab. The event graph opens with the file. Clicking an event under a graph opens that graph, then selects and centres the event's node.

### Diff view

`blueprintreview-app --diff <OLD> <NEW>` opens `NEW` compared with `OLD`, like the editor's *Blueprint Diff* window. With a file already open, **Diff against…** in the toolbar picks the old version to compare it with. The comparison is the same as [`blueprintreview diff`](#blueprintreview-diff--what-changed-between-two-versions).
- **Graph tabs** show the old graph on the left and the new one on the right. Both sides pan and zoom together, and selecting nodes on one side selects the same nodes on the other. A graph that exists in only one version shows "Not in the old/new version" on the other side.
- **Lock / Unlock** (the padlock in the toolbar) lets the two sides pan and zoom independently, as the editor's *Lock/Unlock* button does. Clicking a difference then frames the node on each side that has it. Locking again lines the other side up with the one you last used.
- **Split** (the button next to the padlock) shows the old graph above the new one instead of beside it, as the editor's *Vertical/Horizontal* button does. Both settings carry over when you *Swap*.
- **Highlights** use the editor's diff colours: green for added, red for removed, cyan for modified, and grey-violet for nodes that only moved. A node difference (added, removed, moved, a changed property) outlines the node. A pin difference (a changed default value, an added or removed link, a changed type) outlines just the pin, as in the editor; the pin of the difference selected in *Changes* also gets the pin hover highlight and a red icon. Added and removed wires get a halo in the same colours. Zoomed out to 0.25 or less, nodes with pin differences are outlined too, and unchanged nodes, comments and the wires between them are faded, as in the editor.
- **Changes** replaces *My Blueprint* and is laid out like the editor's differences tree: *My Blueprint* (variables), *Defaults*, *Class Settings*, *Components*, then every graph (functions, macros, event graphs, ...). A category with differences is yellow; one without shows "No differences detected...". Each graph has two boxes for whether it exists in the old and the new version (blue when it's missing from one).
- Under each graph, one row per difference, with the editor's wording and colours, e.g. `Moved Node 'Play Montage'`, `Added Pins to 'Play Montage'`, `Pin IsReference 'S_Out' ['true' -> 'false']`, `Added Link to 'Then'`, `Pin Default 'Duration' ['1.0' -> '2.0']`, `Property Changed: bHasDefaultPin`. Hover a row for details. Click it to open the graph and centre its node; click a graph to open it.
- **Variables, defaults, class settings and components:** click a row under *My Blueprint*, *Defaults*, *Class Settings* or *Components* (or the category's name) and the graphs make way for both versions' details side by side, old on the left, as the editor's diff modes show them. Rows are lined up, so a property is on the same line in both. Differing rows are highlighted and joined across the gap, and the selected difference is outlined. *Defaults* and *Class Settings* list only what differs (a property one version doesn't set shows `Inherited` there); a variable or component shows its full details. A variable or component in one version only shows "Not in the old/new version" on the other side. Click a graph, or a graph's difference, to go back to the graphs.
- **Filter:** the box under the summary keeps the rows whose text, category or graph name contains what you type (`delay`, `Health`, a graph's name), and **Hide moves** hides the cosmetic rows (moved nodes, re-routed wires, comment colours). Graphs with nothing left to show are hidden while a filter is on.
- **Prev** / **Next** step through every difference in the tree, categories first, then the graphs; with a filter on, only through the rows it shows.
- **Timelines**, **Enums** and **Structs** categories appear when the Blueprint's timelines, or the enums and user-defined structs it uses, changed. Changes the Blueprint makes to inherited components are listed under *Components* (`Inherited Component Override`).
- **Details** shows the selected node's status and changes (old and new value of each changed field and pin), then the node's details as in the normal view (from the side it was selected on; unchanged nodes too), or a summary of the shown graph.
- **Revision labels:** the bottom-right corner of each side names the asset and the revision it comes from, as in the editor: `Revision 45dab25b - 19 Jun 2026, 12:33:25 pm` for a version from a git commit, `Local Revision` for the working copy or a file git doesn't know. git runs in the background, so the window opens straight away; until it answers, the labels say `Looking up revision…`.
- **Swap** exchanges the old and new versions; **Exit diff** goes back to the normal view of the new version.
- **Report** (in the toolbar) shows the diff as a report instead: every change as a list, in the same wording and colours as *Changes*, then each changed graph's old and new version side by side as one image (the drawing [`blueprintreview render --against`](#blueprintreview-render--graphs-as-svg-images) writes). **Fit width**, 50%, 100% and 200% size the images. **Save as HTML…** writes the report as one file with the images inline, which opens in any browser (on the web it downloads). **Interactive diff** goes back.
- **Editions:** the free Lite shows every diff as its report; the interactive view, and the Time-lapse, are part of Pro (see `docs/RELEASE_PLAN.md`, "Pricing and tiers"). The window title and a toolbar badge (`LITE` or `PRO`) show which edition runs. The license decides which (see [GUIDE.md](GUIDE.md), "Editions, trial and license"). A license, the 14-day trial from the first launch, or the beta (which unlocks everything until the end of 30 November 2026, `bp_license::BETA_UNTIL`; a first launch in the beta gets its trial after that date) runs as Pro. The toolbar's licence chip names the state (`PRO TRIAL · 12 DAYS`, `BETA`, `LITE`, `PRO`), and its **Activate…** (Lite: **Upgrade…**) button opens the *License* window, which activates a key or links to the pricing; the window title names the state too, and a dismissable bar reminds in the trial's last 3 days and once after. In Lite, *Interactive diff · PRO* and *History · PRO* open that window. `blueprintreview diff` and `render` check the same license. `BLUEPRINTREVIEW_EDITION=lite` or `pro` (on the web `?edition=lite`) overrides it, to try an edition.
- **Updates:** once a day at start-up, a release build reads `https://blueprintreview.efenggamedev.com/latest.json` (written by the release workflow). When there's a newer release, *Version X available* appears next to the version label. The label's menu runs the check now, or turns it off (*Check for updates daily*, saved in `%LOCALAPPDATA%\BlueprintReview\Updates.json`). Debug builds don't check unless `BLUEPRINTREVIEW_UPDATE_URL` names a `latest.json`, and the Microsoft Store package never does (the Store updates it).

#### As a git difftool

[`blueprintreview git-setup`](#blueprintreview-git-setup--set-git-up-for-blueprints) writes the config line below; add `-y` to skip git's prompt before each file.

```sh
git config difftool.blueprintreview.cmd 'blueprintreview-app --diff "$LOCAL" "$REMOTE" --path "$MERGED"'
git difftool -t blueprintreview -- Content/BP_Hero.uasset             # working copy vs the index
git difftool -t blueprintreview HEAD~3 -- Content/BP_Hero.uasset      # working copy vs 3 commits ago
git difftool -t blueprintreview main feature -- Content/BP_Hero.uasset
git difftool -d -t blueprintreview HEAD~1                             # every Blueprint a commit changed, in one window
```

With `-d` (`--dir-diff`), git hands over a folder per side instead of one file at a time. The viewer lists the changed assets in a *Files* panel on the left, marked `M` (modified), `A` (added) or `D` (deleted), and opens the first one. Click another to open it: a modified asset opens in the diff view, an added or deleted one on its own. Each file's path in the folders is its path in the repository, so the revision labels work as for a single file.

git hands the viewer a temporary copy of each committed version. The viewer finds which commit a copy comes from by asking git for the latest commit that wrote that content to the file (`git hash-object`, `git log --find-object`), and shows its short hash and date. `--path` tells it where the file lives in the repo, for when neither side is the working-tree file (two commits compared). Without git, or for content no commit wrote, the label says `Local Revision`. Assets in Git LFS work: git smudges the temporary copies, and the content is hashed through the LFS filter. Enum assets are also looked up in the project of the other version, as a temporary copy has no project around it.

`--old-rev <REV>` / `--new-rev <REV>` set a side's revision instead, e.g. from a script: a commit-ish that git resolves (`HEAD~1`, a tag), else the text as given.

```sh
blueprintreview-app --diff old/BP_Hero.uasset Content/BP_Hero.uasset --old-rev v1.2
```

Where node titles come from:
- **JSON exports from the plugin** carry the editor's own node titles, colours and pin labels.
- **`.uasset` files** don't store them, so the viewer computes them the way the editor does. It uses a table of engine function metadata built into the viewer (see `crates/bp-model/data/README.md`). On plugin exports this matches the editor's titles for 99.8% of nodes. When a node's title was computed, its *Details* say so.

The *Components* panel sits above *My Blueprint* for actor Blueprints (or any Blueprint with components), as in the editor:
- **Self:** the first row is `<Blueprint> (Self)`, with the parent class's icon. Clicking it shows the class defaults, as the toolbar's *Class Defaults* does.
- **Hierarchy:** components are nested under the component they're attached to. A parent the Blueprint doesn't define (inherited from the parent class) shows as a dimmed row with its children under it.
- **Icons:** each component shows the editor's class icon (Static Mesh, Sphere Collision, Camera, …), found by walking up its class hierarchy. A Blueprint component class falls back to the generic scene or actor component icon.
- **Expand and collapse:** click a row's arrow, or double-click the row.

The *My Blueprint* panel works like the editor's:
- **Sections:** Graphs, Functions, Interfaces, Macros, Variables and Event Dispatchers.
- **Events:** each event graph lists its events under it: events, custom events, bound events and input events, named as the editor names them (`Event BeginPlay`, `HidePickup`). They are sorted by name, since the file's node order isn't kept. Click the arrow or double-click the graph to collapse the list. Search matches events too.
- **Categories:** functions, macros, variables and dispatchers are grouped by their category. `Movement | Simulated` is a sub-category of `Movement`, and items in the `Default` category sit directly under the section.
- **Search:** the box at the top filters every section by item name or category.
- **Tooltips:** hovering a name shows its description (the variable's or function's *Tooltip*).
- **Variable rows:** three columns, lined up across the list: the name, then the type (its icon and name), then an eye that is open when the variable is *Instance Editable*. The type icon shows the container: a pill for a single value, a grid for an array, the set icon, or the map's key and value icons. When a file opens, the panel widens to fit the longest name and type, up to 40% of the window. It never shrinks a panel you widened yourself.
- **Components:** the Variables section lists the component variables under *Components*, with only their type icon.
- **Copy:** right-click a variable, function or macro and pick *Copy*, or select it and press **Ctrl+C** over the panel. The clipboard gets what the editor's *My Blueprint* copies (`BPVar(...)` for a variable, with its default value; `BPGraph(...)` for a function or macro, with all its nodes), so it pastes into *My Blueprint* in the editor.

The toolbar's **Class Settings** and **Class Defaults** buttons show the Blueprint's own settings in *Details*, as the editor's buttons do; the one shown is highlighted. A file opens on *Class Defaults*.
- **Class Settings:** *Class Options* has the Parent Class, and under *Advanced* Deprecate, Generate Const Class, Generate Abstract Class and Should Cook Property Guids?. *Interfaces* lists the implemented interfaces, each expanding to the graphs that implement its functions (not for interfaces, macro libraries and function libraries, as in the editor). The inherited interfaces of a native parent class aren't listed yet. *Blueprint Options* has Run Construction Script on Drag / in Sequencer, Blueprint Display Name, Description, Namespace, Category and Hide Categories. A setting the Blueprint didn't change shows the engine's default value. *Info*, collapsed, has the asset path and class, the Blueprint type, the enums it uses and what exported the file.
- **Class Defaults:** every editable property, by category as the editor groups them: the Blueprint's variables first, then the native classes' categories from the most derived up. Values the Blueprint inherits are dimmed, with an "Inherited from …" tooltip: native parent classes' from the viewer's engine data, Blueprint parents' from their assets. A value the Blueprint changed from the inherited one has a reset arrow, and the ✏ button next to the search box shows only those (*Show Only Modified Properties*). The viewer loads Blueprint parents from the project (or next to the file); when one isn't found, a note says its values aren't shown, and dropping its `.uasset` onto the window fills them in (opening it with Open… opens it instead). A note also says when the engine data comes from another engine version than the asset's. Engine enums show their display names.
- **Components:** a component's *Properties* show its class defaults the same way. An inherited component (a dimmed row) shows the values it has in the parent class, with the changes the Blueprint (and any parent between) made to it on top, marked with the reset arrow. Inherited components the Blueprint changes are listed even when none of its own components attach to them (as roots, since their place in the parent's tree needs the parent).

The *Details* panel works like the editor's too: a search box, then collapsible categories of name/value rows with read-only check boxes, text boxes and combo buttons (without their dropdown arrows, since nothing can be picked).
- **Functions:** the *Graph* category has Description, Category, Keywords, Compact Node Title, Pure, Call In Editor and Access Specifier. Under *Advanced* are Const, Deprecated and Deprecation Message. *Inputs* and *Outputs* list the parameters: each has its name and type selector, and expands to its Default Value (inputs) and Pass-by-Reference. With no parameters, a "No parameters" note is shown in place of the editor's prompt to add some. *Info* has the graph's name, kind and node count.
- **Custom events:** *Graph* has Keywords, Replicates, Reliable, Call In Editor and Access Specifier, plus Deprecated and Deprecation Message under *Advanced*. *Inputs* lists the event's parameters.
- **Macros:** *Graph* has Description and Category, and *Inputs* / *Outputs* list the macro's tunnel pins.
- **Variables:** the *Variable* category has the editor's rows in its order: Variable Name, Variable Type, Description, Instance Editable, Blueprint Read Only, Expose on Spawn, Private, Expose to Cinematics, Category, the range, units and bitmask rows where the type has them, Replication and Replication Condition. Under *Advanced*, collapsed until you open it as in the editor, are Config Variable, Transient, SaveGame, Advanced Display, Deprecated and the *Defined Property Flags*. *Default Value* shows the class default, and *Info* has the GUID and any other metadata.
- **Defined Property Flags** lists the compiled property's flags, as the editor does: the variable's stored flags (`CPF_Edit`, `CPF_BlueprintVisible`, …) plus the ones its type implies (`CPF_ZeroConstructor`, `CPF_HasGetValueTypeHash`, `CPF_InstancedReference`, …). A plugin export carries the editor's own list. For a `.uasset` the viewer works them out from the type and the engine table. When that depends on a type it doesn't know, such as a user-defined struct or a Blueprint class, the heading's tooltip says which flags are approximate and why. *Info* shows both raw values (*Property Flags* and *Compiled Flags*).
- **Values:** structs, arrays, sets and maps expand into their fields and elements, under a `3 Array elements` header as in the editor. A container variable with nothing stored shows `0 Array elements`. Enum values show their display names, in struct fields, container elements and map keys too. Colours show a swatch.
- **Search:** the box keeps only the rows whose name matches, plus the parents of matching fields.
- **Copy:** right-click a row for *Copy* (the value as the editor copies it: `True`, `1.5`, a struct as `(X=1.0,Y=2.0)`, an array as `(A,B)`), *Copy Display Name* and *Copy Internal Name*. There's no *Paste*, as nothing in the viewer is editable; the copied value pastes into the property in the editor.

#### With Perforce (P4V)

`blueprintreview p4-setup` prints the steps with the app's path filled in. In P4V, open *Edit → Preferences → Diff* and, under *Specify diff application by extension*, add:

| Field | Value |
|---|---|
| Extension | `.uasset` |
| Application | the full path of `blueprintreview-app.exe` |
| Arguments | `--diff %1 %2` |

P4V's diffs of `.uasset` files then open in the viewer: *Diff Against Have Revision*, *Diff Against Previous*, and the diffs in the *History* tab. P4V hands over temporary copies named for their revision (`BP_Hero#3.uasset`), so each side is labelled `Revision 3`, or `Revision shelved in CL 1234` for a shelved file (`@=1234`). Your workspace file is `Local Revision`. P4V's names carry no dates, so the labels have none.

### Source control and the Time-lapse view

The viewer finds out on its own whether the open asset is in a git work tree or a Perforce workspace. The toolbar's source control button, at the top right, says which (`✔ Git`, `✔ Perforce`, `Not in source control`) and opens the settings:
- **Provider:** *Auto* (git, then Perforce), *Git*, *Perforce* or *None*.
- **Perforce:** *Server* (`P4PORT`), *User Name* (`P4USER`) and *Workspace* (`P4CLIENT`), as in the editor's *Revision Control Login*. Empty fields come from `P4CONFIG` files and your Perforce environment (`p4 set`), so a workspace P4V already knows usually needs nothing here. When the server wants a login, the window asks for your password and runs `p4 login`.
- **Accept Settings** saves them to `%LOCALAPPDATA%\BlueprintReview\SourceControl.json` and looks the asset up again.

The viewer runs the `git` and `p4` command-line programs. Git for Windows and P4V's installer put them on `PATH`.

**History** (the clock button, next to *Diff against…*) opens the Time-lapse view of the asset, like P4V's:
- **The bar** has a tick per revision, oldest on the left: the commit's short hash, or the Perforce revision number. When your working copy has changes, it is the last tick, *Local*. Hover a tick for its changelist, date, author, action and description. Deleted revisions have no content and get no tick. git history follows the file through renames, and on the first-parent line only, so it reads like a Perforce file log; Perforce history follows moves and branches (`filelog -i`).
- **Single revision** (the default): pick a tick and the diff view shows that revision against the one before it, i.e. what that change did. ◀ *Older* / *Newer* ▶, or Left/Right over the bar, step through the history. The first revision is shown against nothing, so everything in it is added.
- **Range:** two handles, old (orange) and new (blue). Drag either, or click near one to move it, to diff any two revisions.
- Under the bar, *Old:* and *New:* describe the two revisions shown. Each side of the diff carries the editor's revision label (`Revision 3 - CL 1042 - 19 Jun 2026, 12:33:25 pm` for Perforce, `Revision 45dab25b - …` for git, `Local Revision`).
- Everything in the diff view works as usual, and the open graph stays open as you step. Revisions are fetched in the background and kept, so stepping back is instant.
- **Exit time-lapse** goes back to your working copy, and to the diff you had open before, if any.

### Output Log and the log file

The app writes what it does to a log: the files it opens, their warnings, assets it couldn't load, and why a file failed to open. Click **Output Log** on the right of the toolbar, or press ``Ctrl+` `` (backtick), to show the log under the window, like the editor's *Output Log*. You can filter it by text or verbosity (*All*, *Warnings and errors*, *Errors*), clear it, copy it, and open the log folder. The button counts the warnings (yellow) and errors (red) logged while the panel was closed.

The same lines go to `%LOCALAPPDATA%\BlueprintReview\Logs\BlueprintReview.log` (on Linux, `~/.local/state/BlueprintReview/Logs`). Each run starts a new file and keeps the last five as `BlueprintReview-backup-<time>.log`. Lines are stamped in UTC, as in the editor's logs:

```
[2026.10.19-14.03.11:204][info] blueprintreview_app::app::loading: Opened BP_Hero.uasset: 4 graphs, 7 variables
[2026.10.19-14.03.11:290][warn] blueprintreview_app::app::loading: BP_Hero.uasset: E_Mood: enum asset not found
```

Set `BLUEPRINTREVIEW_LOG=debug` (or `trace`) for more detail, such as the git commands behind the diff view's revision labels.

A release build opens no console window. Started from a terminal, it still writes the log and its usage errors there. If the app crashes, a dialog says so and gives the log's path. The log then holds the panic message and a backtrace, so please attach it to bug reports.

### Find in Blueprint

Press **Ctrl+F**, or click **🔍 Find** in the toolbar, to open *Find Results* under the graph. It works like the editor's *Find in current Blueprint*. Type a search and press Enter.

What it searches:
- **variables:** name, tooltip and default value;
- **graphs:** event graphs, functions, interface graphs and macros, by name;
- **nodes:** title and comment;
- **pins:** label and default value, for visible pins and the `self` pin;
- **components:** by name.

How a search matches:
- Matching ignores case and spaces, so `spawntypes` finds "Spawn Types".
- Words are combined with AND, and `"quotes"` keep a phrase together.
- All the words must match the same item. A node doesn't match if one word is in its title and another is in a pin.
- The advanced syntax (`Nodes(Name=…)`, `OR`, `+exact`) isn't supported. To search every Blueprint of a project, open its folder and use *Find in Blueprints* (see [Folders](#folders)).

How the results are shown:
- Matching variables come first. Then come the graphs, each listing its matching nodes and, under each node, its matching pins. Matching components come last.
- When a match is in a tooltip or default value, a `Tooltip: …` or `DefaultValue: …` row shows where.
- The search text is highlighted in green.

Clicking a result selects it. **Double-clicking** a result acts on it:
- a node or pin opens its graph's tab, then selects and centres the node (at 1:1, or zoomed out just enough to fit it);
- a variable, graph or component shows it, as clicking it in *My Blueprint* does.

A variable's details show:
- **Property flags** by name (`Edit`, `BlueprintVisible`, ...). Hover for the raw value.
- **Default value.** If the asset doesn't store one, the type's default is shown in grey italics: `false`, `0`, `0.0`, `None`, `""`, `nullptr`, `[]`, the first enum entry, or the zero value for common structs such as `Vector`.
- **Enums** by their display names (`Run` rather than `NewEnumerator1`), and bitmasks as their set entries (`A | C`).
- **Colours** (`Color`, `LinearColor`) as a swatch. Class defaults and component properties show swatches too.

Enum display names come from the enum assets (see [Enum assets](#enum-assets)). In the web build, which can't read the project folder, drop the enum `.uasset` files onto the window along with the Blueprint. Dropping or opening an enum asset adds it to the ones the viewer knows, and refreshes the open Blueprint. The web demo loads a sample's enum assets by itself when they're staged next to it (`<Enum>.uasset` in the sample's folder). Enums that can't be found show their entry names without the `EEnum::` prefix. Hovering such a value names the enum asset it needs, and a ⚠ in the toolbar lists them all.

### Folders

Open a project's `Content` folder, or any folder in it, to browse all its Blueprints, as VS Code's *Open Folder* does:

```sh
cargo run -p blueprintreview-app -- "D:/Unreal Projects/StackOBot/Content"   # or click "Open Folder…"
```

- **Explorer** (left, **Ctrl+Shift+E** shows or hides it): the folder's Blueprint-type assets as a tree under the opened folder's own row, as in VS Code, folders first, each with a › when collapsed and a ⌄ when open. That means Blueprints, Widget and Animation Blueprints, Blueprint Interfaces, macro and function libraries, enums, structs and data tables, and plugin `.json` exports. Every one of them opens in a tab: enums, structs and data tables as their [tables](#enums-structs-and-data-tables). Textures, meshes, levels and other assets aren't listed, and neither are folders without any of these. Each asset has its class's icon. Type in the box to filter by name. **⟳** picks up files changed on disk, and **⊟** collapses every folder. Hovering an asset gives its package, kind and, for an enum, its entries (a struct's field count, a data table's row count). An asset that couldn't be read has a ⚠, and hovering it says why. Files whose header can't be read at all (often textures or meshes saved by a pre-4.25 engine) aren't listed; the *Output Log* names them.
- **Asset tabs:** clicking an asset opens it in a tab of its own above the window, or switches to its tab if it's already open. Each tab keeps its own graph tabs, selection and panels. ✕ or a middle click closes it.
- **Find in Blueprints** (**Ctrl+Shift+F**, or **🔍 Find in Blueprints** in the toolbar): the [Find in Blueprint](#find-in-blueprint) search, run over every Blueprint in the folder. The results come grouped by asset, and update as you type. Double-clicking a result opens its asset and shows the node, graph, variable or component. **Ctrl+F** still searches only the open Blueprint.
- **Double-clicking a node goes to its definition in any Blueprint of the folder:**
  - a function call goes to the function, or to the custom event it calls. This works across Blueprints and up the Blueprint parents, as `GetFunctionGraph` does.
  - a *Parent:* call goes to the parent's function or event (for example *Parent: BeginPlay* goes to the parent's *Event BeginPlay*).
  - an interface message goes to the interface's function, found by its GUID.
  - a macro instance goes to its macro, in this Blueprint or in a macro library.
  - an overridden event goes to the parent Blueprint's graph of that name.
  - a *Create Event* node goes to the function or event it binds.
  - a variable with a RepNotify goes to its notify function.

  If the definition isn't in the folder, the toolbar says so. Native functions can't be opened.
- **Back and Forward** (**Alt+Left** / **Alt+Right**, the mouse's back and forward buttons, or **←** / **→** at the toolbar's left): return to where a jump left, and go to it again. Jumps are double-clicks to a definition (in the same Blueprint or another), *Find* and *Find in Blueprints* results, and assets opened from the Explorer. A place is its asset tab, graph tab, view and selection. An asset whose tab was closed meanwhile opens again. The last 100 places are kept, and a new jump drops the way forward.

Nothing slow runs on the UI thread, so the window stays responsive:
- **Opening:** on Windows the file and folder dialogs run on a thread of their own, so the window keeps drawing while Windows closes one, which can take seconds for a large folder. The folder dialog hands its folder over the moment *Select Folder* is clicked, so the folder starts opening while Windows is still closing the dialog. Elsewhere the dialogs are modal. What a dialog picks is opened in the background. The log's first lines name the GPU and graphics backend. The Explorer appears at once and fills in as files are found. Its header shows each phase with a spinner: *Scanning… N files*, *Reading headers n / N*, then *Indexing Blueprints n / N*. Frames over 50 ms are logged with where the time went (*Slow frame: …* in the *Output Log*).
- **Indexing:** the folder is walked once, on a thread of its own. Each file's header is read first (16 KB for most files; textures and meshes are never read further), by 4 threads running below the UI's priority. The Blueprints are then extracted in parallel, smallest first. Their search data is kept, not the Blueprints.
- **Opening** an asset, with its enums and parents, runs on a worker thread. The tab shows a spinner until the asset is ready. This applies to every open in the desktop app, folder or not: *Open…*, a file dropped on the window or given on the command line, and both sides of `--diff` or a `git difftool -d` pair, whose diff is computed there too. *Diff against…* computes its diff in the background as well. The exceptions are an enum asset or a missing Blueprint parent dropped onto the window, which are read on the spot, and the web build, which reads files asynchronously but extracts them on the page.
- **Searches** run on worker threads in batches of 16 assets. A new search cancels the running one. A search started while indexing is still going is run again once indexing is done.

On the projects in `d:/Documents/Unreal Projects` (release build, files in the OS cache):

| Project | Files | Blueprint-type assets | Indexed in | Find in Blueprints (one thread) |
|---|---|---|---|---|
| StackOBot | 450 | 88 | 0.6 s | 2 ms |
| CropoutSampleProject | 475 | 79 | 0.55 s | 2 ms |
| GameAnimationSample | 3159 | 179 | 0.25 s | 12–25 ms |
| GASVanilla | 3890 | 240 | 0.29 s | 15–35 ms |

The first index of a project that isn't in the OS cache yet takes longer, because the disk reads dominate (about 4 s for GASVanilla's 5.5 GB). The largest Blueprint there, a 20 MB Control Rig with 1871 nodes, opens in about 30 ms.

The web build has *Open Folder…* too, through the browser's folder picker (`showDirectoryPicker`, read-only, where the browser has it; else a `webkitdirectory` input). Nothing is uploaded: files are read in the browser a slice at a time, and the work is done between frames. The index there doesn't resolve enums (a search finds their entry names), but opening an asset reads its enums, structs and parents first. To pick up changed files on the web, open the folder again.

There are three ways to open a file:

```sh
cargo run -p blueprintreview-app -- fixtures/5.7/BP_Fan.uasset   # open a file at startup (.uasset or .json)
cargo run -p blueprintreview-app                                 # then click "Open…", or drag a file onto the window
```

The web build (`cd crates/blueprintreview-app && trunk serve`) supports **Open…** and drag-and-drop.

#### Web demo

The [live demo](https://blueprintreview.efenggamedev.com/demo/) is the web build with `fixtures/` staged next to it. `.github/workflows/web-demo.yml` builds it on every push to `main` and deploys `blueprintreview.efenggamedev.com` to the Cloudflare Worker `ue-blueprint-viewer` (`site/wrangler.toml`, `wrangler deploy`): as static assets, the page in `site/public/` at `/`, the user guide at `/docs/` (`scripts/build_docs.py`, from `docs/guide/` and the reference pages), the release notes at `/changelog/`, the [public demo](#public-demo-try) at `/try/`, the demo at `/demo/` and the [PR viewer](#pr-viewer-view) at `/view/`, plus the Worker script for the PR viewer's GitHub sign-in. The landing page (`/`, with `style.css`, `privacy.html`, `img/` and `legal/`, where the deploy copies `EULA.md` and `THIRD-PARTY-NOTICES.md` as `.txt`), `/docs/`, `/changelog/` and `/try/` are public. Everything else sits behind Cloudflare Access, because the demo ships internal fixtures (see `RELEASE_CHECKLIST.md`). After each deploy the workflow checks that `/demo/`, `/demo/fixtures/index.json`, `/view/`, `/auth/login` and `/api/blob` redirect to the Access login, and fails otherwise, and fails unless `/try/` and `/try/fixtures/index.json` answer 200 with only the sample project's files in the manifest; it warns if `/`, `/style.css`, `/privacy.html`, `/changelog/`, `/docs/` or `/legal/EULA.txt` don't answer 200.
- It opens `5.7/BP_Spawner.uasset` by default. `?file=<folder>/<file>` opens another fixture, e.g. `?file=5.3/BP_Door.uasset`. Adding `&old=<folder>/<file>` opens the diff view against another fixture, e.g. `?file=diff/BP_Cog/new.uasset&old=diff/BP_Cog/old.uasset`.
- The toolbar's **Samples** menu lists every fixture by folder. Picking one opens it and updates the URL, so you can share the link.
- **Samples → Diff samples** opens the edited pairs of `fixtures/diff/` in the diff view: two versions of one Blueprint, each saved by the editor.
- **Samples → Diff against sample** diffs the open Blueprint against any fixture, shown as the old version. Fixtures of the same asset (e.g. `BP_Bot` from another project) are listed first. **Swap** and **Exit diff** keep the URL in step.
- In the diff view, a sample's corner label shows its path in place of a git revision.
- A sample's enum assets are loaded when they're staged next to it (`<Enum>.uasset` in the same folder). The manifest lists them under `enums`, so the *Samples* menu doesn't offer them. A sample's Blueprint parent is loaded the same way, when it's a sample in the same folder.
- **Open…** and drag-and-drop work as in the plain web build.

#### Public demo (`/try/`)

`/try/` is the landing page's *Try it in your browser*: the same web build, with two public projects as its samples instead of `fixtures/` (Epic content, which may not be published): the [demo project](https://github.com/blueprintreview/demo-project), a player character whose history is ten larger changes, and the guide's [sample project](https://github.com/blueprintreview/sample-project). Both are made by `scripts/make_sample_project.py` from edit plans (`--set demo`: `site/demo-project/plans/` and `files/`; the guide's: `docs/guide/sample/`). The workflow clones them with their whole history and LFS objects, and `scripts/stage_sample.py` stages, per `--sample PATH=Folder`, the assets as they are now under `<Folder>/` (one folder, so enum assets are found next to the Blueprints), and a diff pair per asset each commit modified, `diff/<Folder>-<NN>-<Asset>/old.uasset` and `new.uasset` (at the commit's parent and at the commit, with the commit's enum assets beside them), named `<Asset>: <commit subject>` in **Samples → Diff samples**, the projects in the order given. The manifest's `default` is the pair the page opens without `?file=` (`--default`, a commit subject); `?file=` and `&old=` work as in the demo.

```sh
cd crates/blueprintreview-app && trunk build --release --public-url ./ && cd ../..
cp -r crates/blueprintreview-app/dist E:/tmp/try && rm -rf E:/tmp/try/fixtures
python scripts/stage_sample.py --sample E:/dev/projects/BlueprintReviewDemo=Demo --sample E:/dev/projects/BlueprintReviewSample=Sample --default "Player: stamina for sprinting, drained on Tick" E:/tmp/try
python -m http.server -d E:/tmp/try 8080                    # then open http://localhost:8080/
```

`stage_sample.py` reads each revision with `git cat-file` and `git lfs smudge`, so a clone needs every revision's LFS objects (`git lfs fetch --all`); it fails on a blob that isn't a package.

To change the demo project, edit or add a plan in `site/demo-project/plans/` (one commit per plan, its `meta.commit`), rebuild it with `python scripts/make_sample_project.py --set demo --out E:/dev/projects/BlueprintReviewDemo --engine "C:/Program Files/Epic Games/UE_5.8" --work E:/tmp/bpedit/host58 --force` (or `--from N`), and force-push it to `blueprintreview/demo-project`. Its plans name the nodes earlier plans made by object name (`{"name": "K2Node_VariableSet_3", "graph": "EventGraph"}`), which every build gives the same, so a plan changed in the middle means checking the ones after it.

#### PR viewer (`/view/`)

`/view/` is the same web build without the fixtures. It opens the links in `pr-report`'s PR comments: the head version of the file, diffed against the base version (an added or deleted file opens alone), with each side's corner label showing its commit. The files come from GitHub through the site's Worker (`site/worker/index.js`): `/api/blob?repo=…&ref=<sha>&path=…` reads the file at that commit as the signed-in user and resolves Git LFS objects, so the viewer only shows what that user can read on GitHub. Without a session, the toolbar shows **Sign in with GitHub** (`/auth/login`, back to the same diff afterwards). Enum assets and the Blueprint parent are fetched from the repository at the same commit, where `DirSource` would find them on disk. Setting the Worker up is described in [ACTION.md](ACTION.md#hosting-the-viewer).

The Worker's tests: `node --test "site/worker/*.test.js"`.

To build it locally:

```sh
cd crates/blueprintreview-app && trunk build --release --public-url ./ && cd ../..
python scripts/stage_fixtures.py crates/blueprintreview-app/dist   # copies fixtures/ and writes fixtures/index.json (files, diff pairs, enum assets)
rm -rf site/dist && cp -r site/public site/dist && cp -r crates/blueprintreview-app/dist site/dist/demo
mkdir -p site/dist/legal && cp EULA.md site/dist/legal/EULA.txt && cp THIRD-PARTY-NOTICES.md site/dist/legal/THIRD-PARTY-NOTICES.txt
pip install -r scripts/requirements-docs.txt && python scripts/build_docs.py site/dist   # /docs/
python -m http.server -d site/dist 8080                      # then open http://localhost:8080/demo/
```

`stage_fixtures.py` fails on Git LFS pointer files, so run `git lfs pull` first. Without staged fixtures (e.g. under `trunk serve`), there's no **Samples** menu.

One-time hosting setup:
1. Restrict access **first**: in Cloudflare Zero Trust, *Access → Applications → Add an application → Self-hosted*, with the domain `blueprintreview.efenggamedev.com` and the paths `demo`, `view`, `auth` and `api` (one destination each; every path under them is covered), and an Allow policy for your email addresses. A deploy attaches the custom domain, so without this the demo would go live unprotected. (Until 2026-10-30 the application had no path, covering the whole hostname; narrowing it to these four paths is what makes the landing page public.)
2. Create an API token with the *Edit Cloudflare Workers* template, covering your account and the `efenggamedev.com` zone (*Zone → Workers Routes → Edit* attaches the custom domain). Add it, and your account ID, as the repo secrets `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID`. Until they exist, the workflow builds but skips the deploy.
3. The first deploy creates the Worker `ue-blueprint-viewer` if needed, the DNS record and the certificate for `blueprintreview.efenggamedev.com`. The `workers.dev` and preview URLs are off (`site/wrangler.toml`).

---

## `blueprintreview cpp` — Blueprints to C++

```sh
blueprintreview cpp <PATH>... --module NAME -o DIR [--mode native-base|replace] [--engine 5.7|5.8] [--project-api FILE] [--class-name ASSET=NAME]... [--api-macro M] [--copyright TEXT] [--safety bp|idiomatic] [--allow-partial] [--progress]
                    [--functions A,B] [--variables X,Y] [--events E] [--pull-dependencies] [--keep-in-bp F]
blueprintreview cpp <PATH>... --check [--format text|md|json] [--engine 5.7|5.8] [--project-api FILE] [--class-name ASSET=NAME]... [--mode native-base|replace] [--functions A,B] [--variables X,Y] [--events E] [--pull-dependencies] [--keep-in-bp F] [--exit-code]
```

Converts Blueprints, user-defined structs and user-defined enums to UE C++ (milestones C1–C6 of the [converter design](superpowers/specs/2026-10-19-bp-to-cpp-design.md)). By default it's a *native base* (`--mode native-base`): the Blueprint is reparented onto the generated class and keeps its asset references. With `--mode replace` the generated class replaces the Blueprint, which is deleted (see [Migrating a project](#migrating-a-project)).

### Generating code

```sh
blueprintreview cpp Content/Blueprints/*.uasset --module MyGame -o Source/MyGame/Converted
```

writes, per asset, a header and a source file named after the C++ type without its prefix (`BP_Door` → `ADoor` in `Door.h` and `Door.cpp`; structs and enums are headers only), a `.bpmap.json` sidecar per asset that maps line ranges to the variables, functions and nodes they come from, and `Modules.txt`: the modules to add to the module's `PublicDependencyModuleNames`. When some of those modules come from engine plugins (`SmartObjects`, `EnhancedInput`, …), `Plugins.txt` names the plugins.

What a class gets:
- **Variables** as `UPROPERTY` members, their specifiers from the variable's flags (`EditAnywhere` / `EditDefaultsOnly`, `BlueprintReadWrite` / `BlueprintReadOnly`, `Transient`, `SaveGame`, `Replicated` / `ReplicatedUsing`, …), `Category`, the tooltip as a doc comment, other metadata in `meta`, and the default value as an initializer. A name C++ can't use as it is (spaces, a clash with an inherited member) is changed, and the original kept as `DisplayName`.
- **Components** created in the constructor, with their attachment (and socket) and the template values C++ can set.
- **Class defaults** the Blueprint changed on inherited native properties, in the constructor.
- **Functions** as `BlueprintNativeEvent`s (so child Blueprints can still override them), one output as the return value, several as out parameters; overrides of native events and of a Blueprint parent's functions as `_Implementation` overrides; implemented interfaces as C++ bases; **event dispatchers** as dynamic multicast delegates.
- **Function bodies** and the construction script (`OnConstruction`): calls, variables, branches, switches, sequences, casts, the standard macros' loops and `IsValid`, collapsed graphs and macros inlined, the Kismet math and container functions as operators and methods (`A + B`, `Values.Add(5)`), struct makes and breaks, and an engine struct's pin default set field by field (`(bIgnoreAllPressedKeysUntilRelease=True,…)`). Locals are declared where their node runs when that's possible.
- **Event graphs**:
  - **Engine events** become the native virtual the engine calls them from, with the logic before or after `Super::` where the engine's call is: `ReceiveBeginPlay` → `BeginPlay()`, `ReceiveTick` → `Tick()` (plus `bCanEverTick`, as the Blueprint compiler sets it), overlaps, hits, `EndPlay`, `Destroyed`, `PossessedBy`, `Landed`, and the component events.
  - **Custom events** become `BlueprintNativeEvent` `UFUNCTION`s, or RPCs (`Server`/`Client`/`NetMulticast`, `Reliable`/`Unreliable`).
  - **Blueprint interface events** and overrides of a Blueprint parent's events become their `_Implementation`.
  - **Delegates.** Binding a dispatcher becomes `AddDynamic` / `RemoveDynamic` / `Clear()` (Bind / Assign / Unbind / Unbind All), and calling one becomes `Broadcast`. This works for the class's own dispatchers and for native ones such as `OnComponentBeginOverlap`, on this object or another. A component's event (*On Component Begin Overlap (Box)*) becomes a `UFUNCTION()` with the dispatcher's exact signature, bound in `PostInitializeComponents`. Blueprint binds it while the actor is constructed. A custom event bound to a native dispatcher takes that dispatcher's signature.
  - **Delays.** `Delay`, `RetriggerableDelay`, `DelayUntilNextTick` and `DelayUntilNextFrame` call the same latent function from C++, with an `FLatentActionInfo` naming a private `UFUNCTION()` continuation that runs what follows the delay. The engine's latent actions then keep Blueprint's timing, re-entry rules (`Delay` ignores a call while it's pending, `RetriggerableDelay` restarts), pause and time dilation exactly. Values written before a delay and read after it become members. A delay inside a loop's body is a blocker, and so are the other latent actions (async nodes, `MoveComponentTo`, …).
  - **Timelines** become a `UTimelineComponent` created in `PostInitializeComponents`, the way the engine creates a Blueprint's timeline (`CreateTimelineComponent`). That covers its length, length mode, autoplay, looping, replication and time dilation. Its tracks drive members by name, and internal curves are rebuilt in C++ from their saved keys. The `Update`, `Finished` and event-track outputs become `UFUNCTION()` continuations. Play, Reverse, Stop and Set New Time become calls on the component. A track with an external curve asset reads it from an asset member (below), which the Blueprint keeps assigned.
  - **Input.** Enhanced Input action events (*IA_Jump* with its Triggered, Started, Ongoing, Canceled and Completed pins) become private handlers taking the `FInputActionInstance`. Its value, elapsed and triggered times are the node's outputs. The handlers are bound with `UEnhancedInputComponent::BindAction` at the point where the engine binds the Blueprint's: a pawn's `SetupPlayerInputComponent`, a player controller's `SetupInputComponent`, or another actor's `EnableInput` (which `Auto Receive Input` calls too). *Get Input Action Value* reads `UEnhancedInputLibrary::GetBoundActionValue`, and its action is bound for its value there as well. The subsystem nodes (*Enhanced Input Local Player Subsystem*, …) become the `USubsystemBlueprintLibrary` call they compile to, so the mapping-context setup converts as the Blueprint does it. Input outside an actor (a component's), and the legacy input events (*InputAction*, *InputAxis*, key and touch events), are blockers.
  - **Shared values:** a value one event writes and another reads, a value a later run of the same event reads, an event parameter read elsewhere (`LastAmount`), and the state of `DoOnce`, `FlipFlop`, `Do N` and `Gate` become private `Transient` members, since Blueprint keeps them in the event graph's persistent frame.
  - **Not converted:** an engine event no native virtual calls (an `AnimInstance` or `UserWidget` event, a StateTree task) is a blocker.
  - **One difference from Blueprint:** a child Blueprint's `BeginPlay` used to replace the parent's, but now the C++ override always runs.
- **Replication**: `GetLifetimeReplicatedProps` with each replicated variable and its condition.

Asset references (meshes, materials, Blueprint classes) and values C++ can't reach (a private engine member without a setter) aren't hard-coded: the Blueprint keeps them after reparenting. They're listed on stderr as *left to the Blueprints*. An asset that a node uses (an input action, a mapping context, the sound of a *Play Sound* node) becomes a protected `EditDefaultsOnly` member under *Assets*, named after the asset (`IA_Jump`), for the Blueprint to assign. These members are listed on stderr and in the `.bpmap.json`'s `kept` list (`member` and `asset`). A Blueprint class used as a value (*Get Component by Class*'s class) is still a blocker.

`--progress` reports each step on stderr as it runs, one line per step: `progress <stage> <done> <total> <item>`, where the stage is `read` (the requested packages), `load` (the assets they depend on; the total grows as more are found), `check` (a dry run: the check's, and replace mode's choice of the classes that keep a native base), `generate` (each asset), `write` (each file) or `scan` (each package of the project, read for the referencers of what's replaced), and the item is the package path or file. The editor's *Convert to C++…* dialog reads these lines. They work with `--check` too.

An asset with something that doesn't convert yet is blocked, and nothing is written: the command prints the report instead. `--allow-partial` writes what converts anyway, with a `// TODO(blueprintreview): …` comment per blocker; such code is expected not to compile, or to behave differently, and the command says so.

### Using the output in a project

[CONVERTING.md](CONVERTING.md) walks through the whole process for a project, from installing the plugin to checking behaviour after the migration. In short:

1. **Generate into the module's source folder**, e.g. `-o Source/MyGame/Converted`, with `--module` set to that module's name. Regenerating overwrites the files, so don't edit them by hand.
2. **Set up `MyGame.Build.cs`.** Add the output folder to the include paths, because the generated files include each other by file name (`#include "Door.h"`). Then add every line of `Modules.txt` to `PublicDependencyModuleNames`. You can also have the `Build.cs` read the file, so it never needs editing when the list changes. This is what [`tests/cpp-project`](../tests/cpp-project/Source/CppTest/CppTest.Build.cs) does (with `using System.IO;`):

   ```csharp
   string Converted = Path.Combine(ModuleDirectory, "Converted");
   PublicIncludePaths.Add(Converted);
   string Modules = Path.Combine(Converted, "Modules.txt");
   if (File.Exists(Modules))
   {
       foreach (string Line in File.ReadAllLines(Modules))
       {
           string Module = Line.Trim();
           if (Module.Length > 0 && !PublicDependencyModuleNames.Contains(Module))
           {
               PublicDependencyModuleNames.Add(Module);
           }
       }
   }
   ```

3. **Enable the plugins.** If there's a `Plugins.txt`, add each plugin it names to the `.uproject`'s `Plugins` list with `"Enabled": true`.
4. **Build** the editor target (from your IDE, or `Build.bat MyGameEditor Win64 Development -Project=…`). The generated code is written to build with no warnings at `/W4`. A warning means the converter has a bug, so please report it.
5. **Migrate the Blueprints** with the plugin's `BlueprintNativize` commandlet (see [Migrating a project](#migrating-a-project)). It reparents each Blueprint onto its generated class, removes the members the class took over, keeps the values left to it, and replaces the user structs, enums, Blueprint Interfaces and function libraries.
6. **Check the behaviour.** `scripts/nativize_project.py` records the original Blueprints and compares the migrated ones, function by function, actor by actor and over a game run; for the repo's own test content, `scripts/cpp_compile_check.py --behaviour --nativize` does the same.

Convert the batches in the order the report lists them. A batch's C++ includes the headers of the batches before it, so they all have to be in the same module (or in modules that depend on each other).

### Migrating a project

The `migrate` object of each `.bpmap.json` tells the plugin what the generated code takes over:

- each variable, dispatcher, component, timeline, function and event, by its Blueprint name and its C++ name, with renamed parameters;
- the construction script;
- the interfaces the class now inherits.

What happens to the asset depends on its kind:

- A **class** in native-base mode is *reparented*. The Blueprint stays, minus those members: its own nodes and other Blueprints' nodes now use the inherited C++ ones.
- A **user-defined struct or enum, Blueprint Interface or function library** has nothing to reparent, so it is *replaced* in either mode. Its asset is deleted, and its references are redirected to the C++ type.
- In **`--mode replace`**, classes are replaced too. Asset references are found by path in the constructor (`ConstructorHelpers::FObjectFinder`, or `FSoftObjectPath` for soft ones). Replace takes priority: a class with something only its Blueprint can keep (a private engine member without a setter, a component setting C++ can't make, an override of a Blueprint-implementable event) takes a native base instead, by itself. The report marks it `(native base)` and lists why, and its `.bpmap.json` says `reparent`, so one batch replaces what it can and reparents the rest.

Two more files are written next to the code:

- `Redirects.ini`: the `[CoreRedirects]` for what is replaced. There's one `Struct`/`Enum`/`ClassRedirects` line per type, with `OverrideClassName`, plus `PropertyRedirects`, `FunctionRedirects` and enum `ValueChanges` for every name that changed (`Count_7_7E42…` → `Count`, `NewEnumerator1` → `Walk`).
- `Referencers.txt`: the project's packages that import each replaced asset (Blueprints, maps, data assets). They are resaved after the migration.

The plugin applies it, in a copy or under source control, in two runs (the redirects only apply to what loads after them):

```sh
UnrealEditor-Cmd.exe MyGame.uproject -run=BlueprintNativize -Map=Source/MyGame/Converted -Step=Types
UnrealEditor-Cmd.exe MyGame.uproject -run=BlueprintNativize -Map=Source/MyGame/Converted -Step=Classes
```

`Types` records each referencing Blueprint's compiler messages as a baseline, merges `Redirects.ini` into `Config/DefaultEngine.ini`, and deletes the replaced assets. `Classes` does the following and saves every Blueprint it touched:

1. It reparents the Blueprints and removes what moved.
2. It renames the pins of renamed parameters and fields.
3. It puts back the values the Blueprint keeps (its defaults the C++ doesn't set, the asset members' assets).
4. It compiles each migrated Blueprint and every referencer.

It exits with 1 when a migrated Blueprint has a compiler error or warning, or a referencer has one it didn't have before. `-DryRun` changes and saves nothing.

**Partial migration.** `--functions`, `--variables` and `--events` move only part of a Blueprint, by Blueprint names (`UserConstructionScript` for the construction script, `ReceiveBeginPlay` for an engine event). The rest stays in the reparented Blueprint:

- **Blocked by default:** moved code that uses something that stays (C++ can't name a Blueprint's members). The report names each such member (*Partial migration*).
- **`--pull-dependencies`** adds those members to the selection instead.
- **`--keep-in-bp F`** declares function `F` as a `BlueprintImplementableEvent` of the class: moved code calls it, and the Blueprint's graph becomes its implementation.
- **Moving one event alone** requires that it reads nothing a staying event's code writes, because all events share the event graph's state.

**A whole project.** `scripts/nativize_project.py` does all of the above on copies of a project and checks the result against the original:

```sh
python scripts/nativize_project.py --engine "C:/Program Files/Epic Games/UE_5.7" \
    --project "D:/Unreal Projects/MyGame" --work E:/tmp/mygame --mode native-base --map /Game/Maps/Level1
```

It copies the project twice (`original/`, `<mode>/`), adds a C++ module to a Blueprint-only project, converts what `--check` reports ready, and builds it with warnings as errors. Then:

1. **Before the migration**, in the original copy: it records the Blueprints with `BlueprintBehaviour -Record` and plays each `--map` headless twice with the `BlueprintTrace` module (fixed time step and seed), which gives the noise floor.
2. **The migration**, in the converted copy: both `BlueprintNativize` steps.
3. **After it**, in the converted copy: `BlueprintBehaviour -Compare`, and the same game runs diffed against the original's (`scripts/trace_diff.py`).

Each game run starts from the same state. The copy's `Saved/SaveGames` is cleared first, since a game that autosaves would load the last run's save. Every Blueprint actor is traced (`-TraceAll`), and asynchronous loads finish within their frame (`-TraceFlushLoads`).

A divergence counts only when the two original runs agree on that actor and field. For spawned actors this is judged per asset, since spawn order is as random as the spawning.

Two effects of the migration show up there:

- **Placed instances.** A placed instance of a migrated class is resaved, because on disk its components are still the Blueprint's, which the next load would drop.
- **Resaving runs editor-time logic.** A map is saved after the editor loaded it, so editor-time logic in it runs. Cropout's island generator nudges itself 0.05 up on each generation, and that shows as a divergence.

`report.md` sums it up. The project itself is only read. Open `<mode>/<Project>.uproject` to play the converted game in PIE. In Git Bash, set `MSYS_NO_PATHCONV=1` so `/Game/...` isn't turned into a file path.

### The convertibility report

`--check` writes no code. It reports what would convert:

- each asset **ready** or **blocked**, and the C++ type it becomes (`BP_Hero` → `AHero`, `S_Item` → `FItem`, `E_Gait` → `EGait`);
- every blocker, grouped by cause, naming the asset, graph, node title and `NodeGuid`, and the reason;
- the **conversion order**: batches, each converted and compiled after the ones it depends on.

`PATH` is an uncooked `.uasset`, a Blueprint's JSON export, or a folder. A folder stands for every `.uasset` under it, except the per-actor packages in `__ExternalActors__` / `__ExternalObjects__`. Packages that aren't Blueprints, user-defined structs or enums (materials, meshes, …) are skipped with a count on stderr. In a folder, so are packages that can't be read; a file you name that can't be read is an error. The assets' dependencies (Blueprint parents, user-defined structs and enums, Blueprints they reference, macro libraries) are loaded from their project's `Content` folder, or from the folder they're in, and added to the report as `(dependency)`. Pass a whole project to see what converts:

```sh
blueprintreview cpp --check Content
blueprintreview cpp --check Content/Blueprints/*.uasset
```

```text
7 of 23 assets ready (UE 5.7.4)

ready   /Game/StackOBot/Blueprints/SaveGame/PlayerSaveObject -> UPlayerSaveObject
blocked /Game/StackOBot/Blueprints/GameElements/BP_Coin -> ACoin
...

conversion order:
  batch 1: /Game/StackOBot/Blueprints/GameElements/BP_PickupSpline, ...
  batch 2: ...

Events (milestone C4) (9):
  /Game/StackOBot/Blueprints/GameElements/BP_Coin: EventGraph: Event BeginPlay (E4F7E741…): K2Node_Event is an event node; events convert in milestone C4
```

The causes:

| Cause | What it means |
|---|---|
| Asset kind out of scope | Editor Utility, Anim, Widget and Level Blueprints and macro libraries aren't converted. |
| Missing dependency | An asset it needs couldn't be loaded, or depends on one that's blocked. |
| Dependency cycle | Parents, interfaces or structs used by value form a cycle. A cycle through pointers or calls is fine. |
| Native API not in the API table | A native class, struct, function or property the engine API table doesn't have. A type from your project's C++ needs `--project-api`. |
| Native API not usable from another module | The function isn't exported (`*_API`) or inline, the member is private with no getter, a protected member is used on another object, or the header is private. |
| Deprecated / Editor-only native API | As named. |
| Unsupported node / type | No lowering for that node class or pin type yet. |
| Events, Latent actions, Timelines | What event graphs can't convert yet: the legacy input events (*InputAction*, *InputAxis*, key, touch) and input in a component, latent actions other than the delays (and a delay inside a loop), a timeline on a class that isn't an actor. An engine event with no native override (Anim, Widget, StateTree events) is one too. The report still labels these *(milestone C4)*. |
| Macro unavailable | A macro instance whose macro library couldn't be loaded. |
| Unstructured control flow | Exec wires that jump into the middle of a loop. |

Options:
- `--engine` picks the engine API table the code targets: UE 5.7 or 5.8 (5.4 and later as tables are generated, see [PLUGIN.md](PLUGIN.md#7-the-engine-api-table-for-the-c-converter)).
- `--project-api FILE` adds your project's API table, for its C++ classes and its plugins'. Generate it with `scripts/gen_engine_api.py --project` (the project needs the BlueprintExporter plugin). A Blueprint-only project that enables engine plugins the built-in table doesn't cover (the report says *module X isn't in the API table*) can use `scripts/gen_engine_api.py --plugins-of MyGame.uproject --out MyGame.api.tsv`. It enables the same engine plugins in a scratch project and leaves yours untouched.
- `--class-name BP_Hero=AHeroCharacter` overrides a type's name. Without it, a name that clashes with an engine type or with another asset of the batch gets the lowest free suffix from 2 (`APawn_2`), with a note. Names are compared as UHT compares them: without the prefix and without case, against engine header names too, since each type is written to `<name without prefix>.h`. So `ST_Objective` and `BP_Objective` can't both be `Objective`. A Blueprint Interface's name ends in `Interface`, as UE's own do: `BPI_CamManager` becomes `UCamManagerInterface` / `ICamManagerInterface` (in `CamManagerInterface.h`), apart from `BP_CamManager` → `ACamManager`. A class an earlier conversion generated from the same asset isn't a clash: the converter reads the `.bpmap.json` sidecars under the project's `Source` folders (and its plugins'), so regenerating `BP_Bot` writes `ABot` again, not `ABot_2`. Generate it into the folder it was generated into before (the command refuses another folder, which would declare the class twice). A Blueprint already migrated onto its generated class (its parent is that class) is blocked: its C++ is the source now.
- `--format md` writes the report as Markdown for a PR comment, `--format json` as JSON.
- `--exit-code` exits with `1` when an asset you passed is blocked.
- `--module` names the target module; its export macro is `<MODULE>_API` unless `--api-macro` says otherwise. `--copyright` puts a line first in every file.
- `--safety bp` (the default) keeps Blueprint's results on its runtime errors: a call or property read on `None` does nothing and gives the default, an out-of-range array read gives a default element, an integer division by zero gives 0. `--safety idiomatic` writes only the checks a person would, and differs from the Blueprint on those error paths.
- `--mode native-base` (the default) or `--mode replace`: see [Migrating a project](#migrating-a-project). `--functions`, `--variables`, `--events`, `--pull-dependencies` and `--keep-in-bp` make it a partial migration (native base only). `--check` takes them too, so the report shows what blocks that mode and that selection (*Needs the Blueprint*, *Partial migration*).

---

## Errors and exit codes

Every command exits with code `0` on success and `1` on error, printing the reason to stderr. `check` also exits with `1` when it finds differences, and so does `diff --exit-code` when there are non-cosmetic changes. Common messages:

| Message | Meaning |
|---|---|
| `not an Unreal package` | The file isn't a `.uasset`. |
| `a Git LFS pointer, not the asset` | The clone has the LFS pointer file but not the asset itself: run `git lfs pull`. |
| `a cooked asset` | A packaged-game asset, whose graphs cooking stripped. Use the editor asset from the project's `Content/` folder. |
| `saved by an engine older than UE 4.25` | Re-save the asset in a newer editor. |
| `saved by a newer engine than this reader supports` | The asset comes from an engine newer than this version of BlueprintReview knows, and its layout changed. Look for an update. |
| `a Texture2D asset, not a Blueprint` | `dump` only: the asset isn't a Blueprint (the class says what it is). `info` and `props` still work on it. |

The viewer app shows the same messages when a file can't be opened, and writes them to its [log](#output-log-and-the-log-file).
