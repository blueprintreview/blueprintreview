# Your first edit plan

> **Experimental:** Blueprint editing (the *Blueprint Edit Automation* plugin, edit plans and
> `blueprintreview edit`) is experimental. The plan format and the ops may still change between releases, and
> an edit can fail or need finishing by hand in the editor. Keep the project under source control and review
> every change before you commit it.

Change a Blueprint without opening the editor: write what to change as an *edit plan* (a JSON file), check it,
try it, apply it, and review the result like any other change. People use plans for repetitive edits across many
Blueprints, for scripted refactors, and to let an AI agent make Blueprint changes that a person then reviews.

This tutorial adds an on/off switch to the sample's coin spawner: a `bEnabled` variable, and a *Branch* in
`SpawnCoin` so it spawns only while it's set. Every change in the sample project's history was made the same
way; its `EditPlans/` folder has the plans.

## Before you start

> **Not in the beta's downloads:** the Unreal plugins (*Blueprint Exporter* and *Blueprint Edit Automation*) aren't
> distributed with the beta yet. Write to [support@efenggamedev.com](mailto:support@efenggamedev.com) if you'd like
> to try them.

- **UE 5.7 or 5.8** installed. The plan runs in a headless editor (a commandlet); the editor window never opens.
- **The plugins in the project:** *Blueprint Edit Automation* and *Blueprint Exporter*, built for your engine
  version, in the project's `Plugins/` folder ([UE plugin](../PLUGIN.md)).
- **The project closed in the editor**, and the files writable (checked out, in Perforce).
- [The sample project](get-the-sample.md), if you follow along with it.

## 1. Look at the Blueprint

Plans name things the way `blueprintreview dump` prints them: graphs by name, nodes by class or GUID, pins by
name, types as `PinType` objects. Dump the Blueprint and look at the graph you'll change:

```sh
blueprintreview dump Content/Sample/BP_CoinSpawner.uasset > before.json
```

In `before.json`, the `SpawnCoin` graph starts with a `K2Node_FunctionEntry` node whose `then` pin leads to a
`K2Node_SpawnActorFromClass`. That's where the Branch goes. Opening the Blueprint in BlueprintReview shows the same;
a node's *Details* list its class and pin names.

## 2. Write the plan

Save this as `spawner-switch.json`:

```json
{
  "schema": "BlueprintEditPlan/2",
  "meta": { "description": "A switch to turn the spawner off: SpawnCoin spawns only while bEnabled is set." },
  "assets": [
    {
      "asset": "/Game/Sample/BP_CoinSpawner",
      "expect_parent": "/Script/Engine.Actor",
      "ops": [
        { "op": "add_variable", "name": "bEnabled", "type": { "category": "bool" }, "default_value": "true",
          "instance_editable": true, "category": "Spawning", "tooltip": "Spawns coins only while set." },
        { "op": "spawn_node", "id": "branch", "graph": "SpawnCoin", "kind": "generic",
          "generic": { "class": "K2Node_IfThenElse" },
          "near": { "node": { "class": "K2Node_FunctionEntry", "graph": "SpawnCoin" }, "dx": 220 } },
        { "op": "insert_into_exec_chain", "node": "@branch",
          "after": { "node": { "class": "K2Node_FunctionEntry", "graph": "SpawnCoin" }, "pin": "then" } },
        { "op": "spawn_node", "id": "enabled", "graph": "SpawnCoin", "kind": "variable_get",
          "variable": { "name": "bEnabled" }, "near": { "node": "@branch", "dx": -150, "dy": 110 } },
        { "op": "link", "from": "@enabled:bEnabled", "to": "@branch:Condition" }
      ]
    }
  ]
}
```

What each op does:

1. `add_variable` adds `bEnabled`, a bool, true by default, editable per instance, in the *Spawning* category.
2. `spawn_node` places a Branch (`K2Node_IfThenElse`) next to the function's entry, and names it `@branch` for the
   ops after it.
3. `insert_into_exec_chain` puts it on the wire from the entry, so *entry → Branch → Spawn Actor*.
4. `spawn_node` places a *Get bEnabled* node, and `link` wires it to the Branch's condition.

`expect_parent` makes the plan refuse a Blueprint that isn't what you expect. Every reference must match exactly
one node or pin; if one doesn't, the plan stops and lists what it could have meant.
`blueprintreview edit ops` lists every op and its keys; the [edit plans reference](../EDITING.md) explains them.

## 3. Check it

```sh
blueprintreview edit validate spawner-switch.json --project BlueprintReviewSample.uproject
```

This runs offline, in a second: the plan's shape, the op keys, and that the graphs, nodes and pins it names exist
in the project's assets. It prints `0 error(s), 0 warning(s)`.

## 4. Try it

```sh
blueprintreview edit apply spawner-switch.json --project BlueprintReviewSample.uproject --dry-run
```

This starts the editor headless (it takes a little while), makes every edit, compiles, and saves nothing. Each op
reports what it did:

```text
/Game/Sample/BP_CoinSpawner
  [0] ok   add_variable: added variable bEnabled : bool
  [1] ok   spawn_node: spawned K2Node_IfThenElse_0 [K2Node_IfThenElse 'Branch' in SpawnCoin, ...]
  [2] ok   insert_into_exec_chain: K2Node_FunctionEntry_0.then -> K2Node_IfThenElse_0.execute ... K2Node_IfThenElse_0.then -> K2Node_SpawnActorFromClass_0.execute
  [3] ok   spawn_node: spawned K2Node_VariableGet_2 [K2Node_VariableGet member=bEnabled 'Get bEnabled' in SpawnCoin, ...]
  [4] ok   link: K2Node_VariableGet_2.bEnabled (bool) -> K2Node_IfThenElse_0.Condition (bool)
  changes: blueprintreview diff "...BP_CoinSpawner.before.json" "...BP_CoinSpawner.after.json"
DryRunOk
```

Run the `changes:` line to see the result as a diff before anything is saved: the new variable, the Branch and the
*Get bEnabled* node, and the wires.

## 5. Apply it

```sh
blueprintreview edit apply spawner-switch.json --project BlueprintReviewSample.uproject
```

The same run, saving this time. Each file is backed up to `Saved/BlueprintEdit/backup/` first. If any op or any
compile fails, nothing is saved.

## 6. Review it

The change is an ordinary file change, so review it the usual way: in the sample's folder,

```sh
git difftool -y -t blueprintreview -- Content/Sample/BP_CoinSpawner.uasset
```

![The applied plan in the diff view: the Branch and Get bEnabled added to SpawnCoin](img/edit-result.jpg)

Commit it when it looks right, or `git checkout -- Content` to throw it away.

## With the editor open

If the project is open in the editor, `edit apply` uses the editor instead (its edit server, on by default): the
edits appear in the open Blueprint, unsaved, and <kbd>Ctrl</kbd>+<kbd>Z</kbd> undoes the whole plan. Add `--save`
to save them. See the [edit server](../EDITING.md#8-the-edit-server).

## Related

- [Edit plans reference](../EDITING.md): every op, references, types and values.
- [Compare two versions](compare.md): reviewing what a plan did.
