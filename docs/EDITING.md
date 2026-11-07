# Editing Blueprints with edit plans

> **Experimental:** Blueprint editing (the editing part of the *Blueprint Review* plugin, edit plans and
> `blueprintreview edit`) is experimental. The plan format and the ops may still change between releases, and
> an edit can fail or need finishing by hand in the editor. Keep the project under source control and review
> every change before you commit it.

The **BlueprintEditAutomation** module of the *Blueprint Review* plugin applies *edit plans* to Blueprints, user enums, user structs and data
tables: JSON files that say what to change, op by op. A person, a script or an AI agent writes the plan, and the
plugin makes the edits through the same engine calls the Blueprint editor uses.

It doesn't change anything by halves:
- every reference must resolve to exactly one node or pin;
- every edit is checked;
- nodes are laid out without overlaps, along their wires (section 6);
- every edited Blueprint is compiled;
- what departs from the Blueprint style guides is reported (section 7).

If any op or any compile fails, nothing is saved.

A plan speaks the language of `blueprintreview dump`: types are its `PinType` objects, values its `Value`s, nodes
their `NodeGuid`s and pins `nodeGuid:pinId`. So the loop is:
1. **read** the asset with `blueprintreview dump`;
2. **write** a plan in the same terms;
3. **check** it with `blueprintreview edit validate`;
4. **apply** it with `blueprintreview edit apply`;
5. **review** the change with `blueprintreview diff` on the before and after JSON.

- Install: [PLUGIN.md](PLUGIN.md). It is one module of the *Blueprint Review* plugin, which isn't in the beta's downloads yet; write to [support@efenggamedev.com](mailto:support@efenggamedev.com) to try it.
- Engines: UE 5.7 and 5.8.
- Supported assets: uncooked editor assets.

## Contents

1. [Running a plan](#1-running-a-plan)
2. [Plan format](#2-plan-format)
3. [References](#3-references)
4. [Types and values](#4-types-and-values)
5. [Ops](#5-ops)
6. [Layout](#6-layout)
7. [Style](#7-style)
8. [The report](#8-the-report)
9. [The edit server](#9-the-edit-server)
10. [Not covered](#10-not-covered)

## 1. Running a plan

```sh
blueprintreview edit validate plan.json --project MyGame.uproject   # offline: shape, the GUIDs/pins/graphs it names, style
blueprintreview edit apply plan.json --project MyGame.uproject --dry-run
blueprintreview edit apply plan.json --project MyGame.uproject
```

`apply` validates first. It then uses the open editor's edit server when one runs (section 9), and a headless
commandlet otherwise (`--live` / `--commandlet` force one or the other).
- **Commandlet:** it saves, unless `--dry-run`. Close the editor first.
- **Live:** it leaves the packages dirty for you to review and save, and Ctrl+Z undoes the whole plan. `--save`
  saves instead.

The command exits with 1 unless the plugin's report, carrying this run's nonce, says it succeeded.

The plugin can also be driven directly:

| Host | Command |
|---|---|
| Commandlet | `UnrealEditor-Cmd.exe MyGame.uproject -run=BlueprintEdit -Plan=<abs.json>[+<abs.json>...] -Nonce=<hex> [-DryRun] [-Report=<abs.json>] -unattended -nopause` |
| Snapshot only | `... -run=BlueprintEdit -Snapshot=/Game/A[+/Game/B] -Nonce=<hex>`: each asset's canonical JSON |
| Op table | `... -run=BlueprintEdit -ListOps=<abs.json>` |
| Editor console | `BlueprintEdit.Apply <abs.json> [-DryRun] [-Save] [-Nonce=<hex>]` |

Several `+`-joined plans share one editor start-up, which is the slow part, and each gets its own report.

**Don't trust the exit code.** Editor commandlets have exited 0 without running. A run counts only when its report
exists, carries your nonce, and has a `status` of `applied`, `dry_run_ok` or `snapshot_ok`. `started` means the
process died part-way. `blueprintreview edit apply` checks all of this for you.

**Preflight refuses an asset when:**
- its file is read-only (check it out first; `--dry-run` skips this check);
- it has unsaved changes in the editor;
- it already fails to compile;
- its parent isn't `expect_parent`;
- it's listed with `create` but already exists.

**Saving.** Every existing file is copied to `Saved/BlueprintEdit/backup/<plan>.<nonce>/` first. If a save fails
part-way, the files already saved are restored from those copies. New files still need adding to source control.

## 2. Plan format

```json
{
  "schema": "BlueprintEditPlan/2",
  "meta": { "ticket": "GAME-123", "agent_run": "..." },
  "check_referencers": false,
  "layout": "conventions",
  "style": "warn",
  "assets": [
    { "asset": "/Game/Characters/BP_Hero", "expect_parent": "/Script/Engine.Character", "ops": [ ... ] },
    { "asset": "/Game/Data/E_Mood", "create": { "kind": "enum", "enumerators": ["Calm", "Angry"] } }
  ],
  "asset_ops": [ { "op": "rename_asset", "asset": "/Game/Old/BP_Thing", "to": "/Game/New/BP_Thing" } ]
}
```

- **Unknown keys** are errors everywhere. The whole plan is validated before anything loads.
- **`meta`** is free-form, and is echoed into the report.
- **Order:** assets apply and compile in array order. Something that uses what this plan adds to another asset must
  come after it.
- **`check_referencers`** (commandlet only): after the apply, it loads every Blueprint that references an edited
  asset, refreshes its nodes and compiles it. If any of them fails, nothing is saved.
- **`layout`**: `conventions` (the default) lays new nodes out along their wires; `none` keeps only the no-overlap
  rule (section 6).
- **`style`**: `warn` (the default) reports style findings; `strict` fails the plan on any of them (section 7).
- **`asset_ops`** rename, duplicate or delete whole assets. They run after the edits are saved, and only in a saving
  run. A dry run only checks them.
- **Common op keys:**
  - `op` (required);
  - `id`: an alias, `@id`, for what the op creates;
  - `step`: a cross-reference to your own notes;
  - `note`.

**Creating assets** with an asset's `create` block. A created asset may take ops too.

| `kind` | Keys |
|---|---|
| `blueprint` | `parent`, `type?`: `normal`, `function_library`, `macro_library` or `interface`. Widget and Anim Blueprints are refused |
| `enum` | `enumerators`: display names |
| `struct` | `fields: [{name, type, default_value?}]` |
| `data_table` | `row_struct`, `rows: [{ "Name": "Row1", ... }]` (the DataTable editor's JSON import form) |

The full grammar is [schema/edit-plan.schema.json](../schema/edit-plan.schema.json), and
`blueprintreview edit ops` lists every op with its keys.

## 3. References

**Nodes** are a string or an object:

| Form | Matches |
|---|---|
| `"8F3A…C2"` | a `NodeGuid`: 32 hex digits, as `dump` prints it |
| `"@print"` | a node an earlier op created with `"id": "print"`. A function's `id` also gives `@id.entry` and `@id.result`, an override's `@id.parent`, a collapse's `@id.entry` and `@id.result` (and graph `id.graph`), and a paste's `@id.1`, `@id.2`, … |
| `{"guid": …}` / `{"alias": …}` | the same, as objects |
| `{"name": "K2Node_CallFunction_3", "graph": "EventGraph"}` | the object name. Names are unique only per graph |
| `{"class": "K2Node_CustomEvent", "member": "OnHit"}` | a class (or any base class), plus a function / event / variable / macro name |
| `{"class": "K2Node_CallFunction", "title": "Print String"}` | a class plus its list-view title; a last resort |

Zero or several matches is an error, and the error lists every candidate. An op's `id` also names the graph it
creates, anywhere a graph name goes.

**Pins** are `"<node>:<pin>"`, where `<node>` is a GUID or `@alias` and `<pin>` is a `PinId` or a pin name. So a
`dump` link, `nodeGuid:pinId`, is a pin reference as it stands. The object form is
`{"node": <node>, "pin": "<name>" | "id": "<PinId>", "direction": "input" | "output"}`.

A pin name matches exactly, then case-insensitively, then by the label the editor shows. `exec` means `execute` and
`return` means `ReturnValue`, unless a pin really has that name. Some nodes get new pin ids when they're rebuilt,
so a pin name is the safer reference to a node an earlier op changed.

## 4. Types and values

**Types** are bp-model `PinType` objects, exactly as `dump` prints them:

```json
{ "category": "real", "sub_category": "double" }
{ "category": "object", "sub_category_object": "/Script/Engine.Actor", "container": "array" }
{ "category": "byte", "sub_category_object": "/Game/Data/E_Mood.E_Mood" }
{ "category": "name", "container": "map", "value_type": { "category": "int" } }
```

- **Categories** are UEdGraphSchema_K2's names: `bool int int64 real byte name string text struct enum object
  class interface softobject softclass`.
- **`real`** needs `"sub_category": "float"` or `"double"`. UE5's Float variables are `double`.
- **Paths** are full object paths.
- **`exec` and `wildcard`** only go in macro pins and `assert_pin`.

**Pin defaults** use the three fields `dump` prints: `default_value` (most pins), `default_object` (object and
class pins, an asset path) and `default_text` (text pins).

**Property values** take one of two keys:
- `"value"`: a bp-model `Value` (`{"Float": 2.5}`, `{"Struct": {"ty": "Vector", "fields": [...]}}`,
  `{"Object": "/Game/A.A"}`, ...);
- `"text"`: the property's ExportText form (`"(X=1,Y=2,Z=3)"`, `"True"`).

## 5. Ops

Every op checks what the editor would refuse (a name that's taken, a pin type a function can't have, a node a user
can't delete), and fails rather than half-doing it. Ops that would silently change something else need saying so
(`link` over an existing exec link: `"replace": true`; a conversion node: `"allow_conversion": true`). The keys
of each op are in `blueprintreview edit ops`.

| Area | Ops |
|---|---|
| Variables | `add_variable`, `set_variable_flags` (every Details setting: instance editable, expose on spawn, private, read only, replication + condition, cinematics, config, transient, save game, advanced, deprecated, tooltip, category, friendly name), `rename_variable`, `remove_variable`, `set_variable_type`, `set_variable_default`, `set_variable_metadata`, `move_variable`, `duplicate_variable`, `replace_variable_references`, and the local versions: `add_local_variable`, `remove_local_variable`, `rename_local_variable`, `set_local_variable_type`, `set_local_variable_default` |
| Functions and macros | `add_function`, `add_function_override`, `add_macro`, `add_function_inputs` / `add_function_outputs` and `remove_function_inputs` / `remove_function_outputs` (functions, macros and dispatchers), `set_param` (rename, type, by reference, default), `move_param`, `set_function_flags`, `set_function_category`, `set_function_metadata` (tooltip, keywords, compact title, call in editor, deprecated, thread safe, access, const, pure), `rename_function` |
| Graphs | `add_event_graph`, `remove_graph`, `rename_graph`, `collapse_nodes` (into a collapsed graph, a function or a macro), `expand_node`, `convert_function_to_event`, `convert_event_to_function` |
| Events and dispatchers | `add_event_inputs`, `remove_event_inputs`, `rename_event`, `add_dispatcher`, `remove_dispatcher`, `rename_dispatcher`; custom-event settings through `set_param`, `move_param`, `set_function_flags` and `set_function_metadata` with `"node"` |
| Class | `implement_interface`, `remove_interface`, `reparent`, `set_class_default`, `set_class_setting` (Class Settings), `compile` |
| Components | `add_component`, `remove_component`, `rename_component`, `attach_component` (with socket), `set_root_component`, `set_component_property` (inherited components through their override template) |
| Nodes | `spawn_node`, `remove_node`, `paste_t3d` (the editor's copy text, e.g. from `blueprintreview copy`), `move_node`, `align_nodes`, `set_node_enabled`, `set_node_comment`, `refresh_nodes`, `replace_node`, `set_node_property`, `insert_into_exec_chain` |
| Pins | `link`, `break_link`, `break_all_links`, `set_pin_default`, `add_input_pins`, `remove_pin`, `split_pin`, `recombine_pin`, `promote_to_variable`, `move_links`, `insert_reroute` |
| Comments | `add_comment`, `set_comment` (text, colour, font size, bubble, move mode), `resize_comment` |
| Timelines | `spawn_node` kind `timeline`, `set_timeline`, `add_timeline_track` (float, vector, colour, event; or an external curve), `remove_timeline_track`, `rename_timeline_track`, `set_timeline_keys`, `rename_timeline` |
| Enums | `add_enumerator`, `remove_enumerator`, `rename_enumerator`, `move_enumerator`, `set_enum` (description, bitflags, tooltips) |
| Structs | `add_struct_field`, `remove_struct_field`, `rename_struct_field`, `set_struct_field` (type, default, tooltip), `move_struct_field`, `set_struct_tooltip` |
| Data tables | `add_row` (optionally a copy), `remove_row`, `rename_row`, `set_row`, `move_row`, `import_rows` (JSON merge or replace, CSV replace), `set_row_struct` |
| Whole assets (`asset_ops`) | `rename_asset` (leaves a redirector), `duplicate_asset`, `delete_asset` (refused while referenced, unless `force`) |
| Checks | `assert_linked`, `assert_pin`: read-only preconditions. Put one before a rewire, so a graph that changed since the plan was written fails loudly |

**`spawn_node` kinds:**

| `kind` | Argument object |
|---|---|
| `call_function` | `function: {name, owner?}`: `owner` is a class path, or self by default; covers library nodes (math, strings, arrays, Print String) |
| `call_parent` | `function: {name?}`: the graph's own function by default |
| `message` | `function: {owner: <interface>, name}` |
| `macro` | `macro: {name, library?}`: StandardMacros by default (`IsValid`, `ForEachLoop`, `DoOnce`, `Gate`, …); `"self"` for this Blueprint's |
| `variable_get` / `variable_set` | `variable: {name, owner?, local?}` |
| `custom_event` / `override_event` | `event: {name}`: an override is a parent or interface event, e.g. `ReceiveBeginPlay` |
| `cast` | `cast: {class, pure?}` |
| `make_struct` / `break_struct` / `switch_enum` | `struct: {path}` / `enum: {path}` |
| `dispatcher` | `dispatcher: {name, action: call \| bind \| unbind \| unbind_all, owner?}` |
| `timeline` | `timeline: {name, length?, loop?, auto_play?, float_tracks?, event_tracks?}` |
| `generic` | `generic: {class, properties?}`: **any** K2 node class (`K2Node_IfThenElse`, `K2Node_ExecutionSequence`, `K2Node_SpawnActorFromClass`, `/Script/UMGEditor.K2Node_CreateWidget`, …), its properties set before its pins are made |
| `select`, `knot` | none |

A node is placed with `pos: [x, y]` or `near: {node, dx?, dy?}`. With neither, it goes to the right of everything.
Leave the position to the plugin where you can: a `near` without `dx`/`dy`, or no hint, lines the node up with its
wires once they're made (section 6).

**An example**, [test-data/edit-plans/greeter.json](../test-data/edit-plans/greeter.json), builds a new actor:

```json
{ "op": "add_function", "id": "greet", "name": "Greet", "inputs": [{ "name": "Text", "type": { "category": "string" } }] },
{ "op": "spawn_node", "id": "branch", "graph": "greet", "kind": "generic", "generic": { "class": "K2Node_IfThenElse" }, "near": { "node": "@greet.entry" } },
{ "op": "link", "from": "@greet.entry:then", "to": "@branch:execute" },
{ "op": "spawn_node", "id": "begin", "graph": "EventGraph", "kind": "override_event", "event": { "name": "ReceiveBeginPlay" } },
{ "op": "spawn_node", "id": "call", "graph": "EventGraph", "kind": "call_function", "function": { "name": "Greet" }, "near": { "node": "@begin" } },
{ "op": "link", "from": "@begin:then", "to": "@call:execute" }
```

An edit to an existing asset names its nodes by GUID, taken from `dump`:

```json
{ "op": "set_pin_default", "pin": "3EEB4C2F4F04211C138B75930D26AEE0:5A2D…E1", "default_value": "false" },
{ "op": "insert_into_exec_chain", "node": "@log", "after": "27024A2D4FB7114544D059B86EB15A6A:then" }
```

## 6. Layout

A node the plan creates or moves never overlaps another node. The one exception is a comment over the nodes it
encloses.
- **`pos`** is kept exactly; whatever is in the way is pushed right as a block.
- **`near`, or no hint:** the node may slide down into a free slot.
- **`insert_into_exec_chain`** moves the node into its wire and pushes the downstream nodes right.
- **A node that grows** (new pins, a split struct) pushes away only what its new area covers.
- **Comments:** a comment moves with the nodes it encloses, and a new or resized comment pushes unrelated nodes out
  from under itself.

The rule is checked after the ops run, not trusted, and any overlap fails the plan. Commandlets have no Slate to
measure nodes with, so node sizes are estimated from titles and pin labels, on the generous side.

**Layout conventions.** With `"layout": "conventions"` (the default), the plugin also lays new nodes out the way the
style guides ask: "align wires, not nodes", with the white exec wires first
([Allar §3.4](https://github.com/Allar/ue5-style-guide#34-blueprint-graphs)).
- **Exec wires run straight.** A node that continues an exec wire goes level with it: its exec input at the height
  of the output it's wired from, just right of that node. After an event, that's one pin row lower than the event's
  title, as the event's delegate pin comes first.
- **Pure nodes sit by what they feed.** A getter or other pure node goes left of the input it feeds, its output
  level with that input. Where that spot is taken (another node feeding the same one), it takes the nearest free
  spot above or below.
- **The grid.** Positions the plugin works out snap to the editor's 16-unit grid (`GridSnapSize`). A straight wire
  wins over the grid where the two disagree, so a lined-up node's y may be off it.
- **Existing nodes stay put.** Only nodes this plan placed loosely move to line up, and only into free space; no
  other node is pushed for it. Where the lined-up spot is taken, the node stays where placement put it. A comment
  that enclosed a node that lines up grows to keep it.

"Loosely placed" means a `near` without `dx`/`dy`, no hint, an `insert_into_exec_chain`, a promoted variable and a
reroute without `pos`. An explicit `pos`, a `near` with an offset and `move_node` are kept as given. Nodes line up
once they're wired: before an op that reads positions (`add_comment`, `resize_comment`, `align_nodes`, `move_node`,
`collapse_nodes`, `insert_reroute`) and after the last op. `"layout": "none"` turns all of this off.

## 7. Style

Every plan is checked against the published Blueprint style guides:
[Allar's ue5-style-guide](https://github.com/Allar/ue5-style-guide) (§3, Blueprints),
[Unreal Directive's project standards](https://unrealdirective.com/resources/project-standards/naming-functions-variables)
and Epic's [Blueprint best practices](https://dev.epicgames.com/documentation/en-us/unreal-engine/blueprint-best-practices-in-unreal-engine).
What departs from them is a *style finding*: a warning that doesn't stop the plan. With `"style": "strict"` in the
plan, or `--strict-style` on `edit validate` / `edit apply`, any finding fails it and nothing is saved.

`blueprintreview edit validate` reports the rules a plan shows offline (`style: <rule>: <path>: <message>`); the
plugin checks the graph rules after the ops and lists them in the report's `style`.

| Rule | Checked | What it reports | Source |
|---|---|---|---|
| `bool-prefix` | offline | a new or renamed bool variable without the `b` prefix | [Epic coding standard](https://dev.epicgames.com/documentation/en-us/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine), [Allar](https://github.com/Allar/ue5-style-guide#bp-var-bool-names) |
| `name-case` | offline | a new or renamed variable, function, macro, event, dispatcher or component that isn't PascalCase | [Allar](https://github.com/Allar/ue5-style-guide#bp-var-naming-case) |
| `dispatcher-on` | offline | a dispatcher whose name doesn't start with `On` | [Allar](https://github.com/Allar/ue5-style-guide#bp-funcs-naming-eventhandlers), [Unreal Directive](https://unrealdirective.com/resources/project-standards/naming-functions-variables) |
| `rpc-prefix` | offline | a custom event set `NetServer` / `NetClient` / `NetMulticast` whose name doesn't start with `Server` / `Client` / `Multicast` | [Allar](https://github.com/Allar/ue5-style-guide#bp-funcs-naming-rpcs) |
| `editable-tooltip` | offline | an Instance Editable or Expose on Spawn variable without a tooltip | [Allar](https://github.com/Allar/ue5-style-guide#bp-vars-editable-tooltips) |
| `editable-category` | offline | an editable variable left in *Default* in a Blueprint with 5 or more variables | [Allar](https://github.com/Allar/ue5-style-guide#bp-vars-categories) |
| `function-description` | offline | a new public function without a description | [Allar](https://github.com/Allar/ue5-style-guide#bp-graphs-funcs-description) |
| `asset-prefix` | offline | a created, renamed or duplicated asset without its kind's prefix: `BP_`, `WBP_`/`W_`, `ABP_`, `BPI_`, `BPFL_`, `BPML_`, `E`/`E_`, `F`/`F_`/`S`/`S_`, `DT_` | [Allar §1.2](https://github.com/Allar/ue5-style-guide#anc-common), [Unreal Directive](https://unrealdirective.com/resources/project-standards/asset-naming) |
| `dangling-node` | plugin | a created node that never runs (no exec input wired) or is never read (a pure node with no output wired) | [Allar](https://github.com/Allar/ue5-style-guide#bp-graphs-dangling-nodes) |
| `cast-failed-unhandled` | plugin | a created impure cast whose *Cast Failed* is unwired | [Allar](https://github.com/Allar/ue5-style-guide#bp-graphs-cast-error-handling) |
| `backward-exec` | plugin | an exec wire at a placed node that runs right to left; use a Sequence or a reroute | [Allar](https://github.com/Allar/ue5-style-guide#bp-graphs-spaghetti) |
| `function-node-limit` | plugin | a function the plan takes past 50 nodes (comments, reroutes, casts, getters, Break Struct, the entry and Self don't count); one already past it isn't reported | [Allar](https://github.com/Allar/ue5-style-guide#bp-graphs-funcs-node-limit) |

**Naming follows the Blueprint.** Where the guides disagree, the edited Blueprint decides:
- `bool-prefix` is skipped when most of its bools have no `b` prefix, and suggests `bIs…` when most of them say it
  (Unreal Directive's `bIsAlive`) rather than Allar's `bDead`;
- `name-case` is skipped when most of its names have spaces.

With no asset to look at (no `--project`), or nothing to go by in it, the rules are Allar's.

## 8. The report

Every run writes `<Project>/Saved/BlueprintEdit/<plan>.<nonce>.report.json` (`BlueprintEditReport/2`). It holds:
- the status and the nonce;
- what each op did;
- every created node's alias, GUID and pins (`id`, `name`, `direction`), so a follow-up plan can name them;
- the compile results;
- each asset's style findings (`style`: `rule`, `graph`, `node`, `message`; section 7);
- for each Blueprint, its canonical JSON before and after the edit (`before`, `after`). `blueprintreview diff
  <before> <after>` shows the change.

## 9. The edit server

With the editor open, the plugin runs a local HTTP server, so a plan applies in seconds rather than after a
commandlet's start-up. The change goes into the editor's undo history, and you review it there.
- **Binding:** it listens on `127.0.0.1` only, on port 30910 or the next free one.
- **Token:** every request needs `Authorization: Bearer <token>`, with the token from
  `<Project>/Saved/BlueprintEdit/server.json`. Anything else gets 401.
- **Settings:** *Editor Preferences > Plugins > Blueprint Edit Automation* turns it off or changes the port.
  `BlueprintEdit.Server.Start` and `BlueprintEdit.Server.Stop` toggle it for the session.

| Route | |
|---|---|
| `POST /plan?dry_run=1&save=1&nonce=<hex>` | the body is a plan; the response is its report |
| `GET /export?asset=/Game/Dir/BP_Foo` | the asset's canonical JSON |
| `GET /ops` | the op table |
| `GET /status` | engine, project, process id, whether Play In Editor runs |

Plans are refused while Play In Editor runs. Any process that can read your project's `Saved` folder can use the
server, so turn it off on a shared machine.

## 10. Not covered

- Graphs that aren't Blueprint script graphs: AnimGraph and state machines, the Widget Designer, Behavior Trees,
  materials and Niagara. Their editors use other schemas and node models.
- Creating Widget or Anim Blueprints.
- Breakpoints and watches, which are debug state rather than asset state.
