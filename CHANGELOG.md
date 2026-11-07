# Changelog

What changed in each release of BlueprintReview (the app `blueprintreview-app` and the CLI `blueprintreview`). A release tag `vX.Y.Z` needs a `## [X.Y.Z] - date` section here: `.github/workflows/release.yml` takes the release notes from it (`scripts/release.py notes`), and the app's update check shows its first line.

## [Unreleased]

## [1.2.0-beta5] - 2026-11-07

- **One Unreal plugin, *Blueprint Review*:** the *Blueprint Exporter* and *Blueprint Edit Automation* plugins are now modules of a single plugin (`plugins/BlueprintReview`), because Fab doesn't allow a plugin to depend on another. Commandlets, console commands and menu entries are unchanged. **Upgrading:** delete `Plugins/BlueprintExporter` and `Plugins/BlueprintEditAutomation` from the project, install `BlueprintReview` and enable it (`scripts/deploy_plugin.py` does this). The Converter Path saved in the project's user settings is carried over on the first start; if it isn't, set it again under *Editor Preferences > Plugins > Blueprint Review*.
- **Diffs open in the app from the editor** (plugin, new `BlueprintReviewLink` module): Content Browser *Diff Selected*, Revision Control *Diff Against Depot*, the History window and the Blueprint editor's *Diff* menu start `blueprintreview-app --diff` with the revision labels, and fall back to the built-in window when the app is missing or a side has unsaved changes. The plugin also serves the view sync endpoint the app's *Sync* button talks to.
- **Sync with the Unreal editor** (desktop app, in the diff view): a *Sync* button ties the new side's graph to the editor's open Blueprint. Pan, zoom, selection and the shown graph follow both ways (from a background thread, at the rates the editor's preferences set: by default the editor is polled and the app sends ten times a second), the Blueprint is opened in the editor when it isn't. The button shows up when the plugin's `Saved/BlueprintReview/link.json` is found above the file; the status reads *Connected to UE 5.7*, *Editor not found* or *Connection lost*.
- **Sync starts from the diff:** turning Sync on (or reconnecting) sends the diff's graph, position, zoom and selection to the editor at once, opening the Blueprint there if it isn't open; after that either side's moves are followed.
- **Sync no longer lags with the app in front:** an editor in the background slows itself to 3 frames a second, and the sync server answers only on a frame, so every pan waited about 0.7 s for its round trip whatever the poll rate. The plugin now keeps the editor at its normal rate while the app is syncing (*Keep Editor Awake*, on by default): a round trip takes about 25 ms. Poll and send intervals go down to 10 ms.
- **Sync reloads on save:** while Sync is on, saving the Blueprint in the editor re-reads the file and refreshes the diff against the same old version in the background, keeping the open graphs, pan, zoom and selection; the status shows *Reloaded after save*.
- **Diffs list only what the editor's diff lists:** properties a native component class doesn't expose for editing (a Niagara component's parameter-store caches), an inherited component's record that then changes nothing, and a call node's `MemberParent` / `MemberGuid` (where the last compile found the function, which moves when a function is overridden or the parent changes) are no longer changes.
- The **Components** panel lists the components the native parent class adds (a Character's *Capsule Component*, *Arrow Component*, *Mesh* and, below the line, *Character Movement*), named as the editor names them, with the Blueprint's own attached under them.
- **Create Event** nodes show the delegate's signature and the selected function's dropdown below their pins, as in the editor, and an unconnected Target or Object pin shows `self`.
- **Node titles as in the editor** for nodes that showed their class name: *Create Widget* and *Construct* (with their class), async tasks such as *Async Save Game to Slot*, class and byte casts, enum and bitmask literals, local variables, *Get Data Table Row* (with its table), choosers, *Add Component*'s asset line, and every animation graph and state machine node (*Output Pose*, *Sequence Player*, states, transitions). Control Rig nodes show the editor's titles and pin labels instead of *Control Rig Graph Node*.
- Functions and events of engine plugins (Mover, CommonUI, Smart Objects, Chooser, Pose Search and every other plugin that ships with the engine) show their display names: *Event On Activated*, *Parent: Generate Simple Walk Move*, *On Clicked (Button)*. Your project's own C++ isn't covered yet.
- A function's entry node takes its graph's name, so a renamed or copied function no longer shows another function's title, and a call to a Blueprint Interface function no longer borrows the title of an engine function with the same name. Event dispatcher calls read *Call Button Pressed*.
- Nodes show the pins the editor adds when it opens an asset: parameters added to engine functions since the asset was saved (*Set Timer*'s *Max Once Per Frame*), and *Expose on Spawn* variables of the class a spawn node creates.
- **Animation Blueprints:** state machines are drawn as in the editor (state, conduit, alias and entry boxes, transitions as arrows between them), pose pins and the *Output Pose* node look as in the editor, and *My Blueprint* lists *Animation Graphs* and *Animation Layers* with their state machines, states and transition rules.
- **Transition rules:** double-click a state machine, state, conduit or transition to open its graph in the same tab, as in the editor, with the path above the graph (*AnimGraph › Locomotion › Idle to Walk (rule)*) to go back up; tabs name rules as the editor does. Hovering a transition shows its rule in words (*GroundSpeed > 0.0*), its priority and whether it's automatic or disabled.
- **Details for Animation Blueprint nodes** as in the editor: a transition's *Transition*, *Blend Settings* and *Notifications* (with the rule in words and the engine's defaults for what the asset doesn't save), a state's bound functions and notifies, an alias's states, and any anim node's settings by category (*Settings*, *Sync*…), with the values shown as pins left out.
- **Find in Blueprint** and **Find in Blueprints** search inside state machines, states, transition rules and collapsed graphs; a result there opens that graph and selects the node. Diffs name a change inside a rule by the rule (*Idle to Walk (rule): …*). More animation node titles match the editor's (*Blendspace Player*, *Save cached pose*, *Linked Anim Layer*, *Blend Poses (E_Stance)*, an alias's name), and implemented anim layers are listed under *Animation Layers*.
- **Class Defaults** look like the editor's: the native components' properties in their categories (*Transform* per component, *Animation*, *Mesh*…) with the values the Blueprint gives them, an *Actor Tick* category, *Advanced* groups, *Call In Editor* buttons, display names as labels (*Resources Held*, *UI HUD*), floats as `2.0` and empty references as *None*. Actor and component references, a parent's private variables and the categories a class hides aren't listed, as in the editor.
- Widget Blueprints list their *Is Variable* widgets under *Variables*. Levels (`.umap`) open as their Level Blueprint, and folder mode lists the levels that have one. Data assets (`DA_*`, input actions…) open as a table of their values. Redirectors open their target, and a level's external-actor files say what they are instead of *the package has no asset*.
- Diffs list pins added to or removed from a node (*Added Pin to 'Parent: Generate Move'*), variables added at their default value under *Defaults*, and the values a Blueprint changes on a native component (*Native Component CharacterMesh0*); empty inherited-component records the engine drops on save are no longer reported as removed.
- **Local variables:** a function's tab adds *Local Variables (Function)* to *My Blueprint*, listing the function's local variables with their types; selecting one (or a get/set of it on the graph) shows its name, type, description, category and default value in *Details*. *Find in Blueprint* finds them under their function, *Copy* copies one as the editor does, and diffs report them as the editor does (*Local variable Tally::Count was a Integer but is now a Float*, *… added*, *… removed*), instead of as changes to the entry node's `LocalVariables` property. `blueprintreview textconv` lists them under their function's entry node, and `blueprintreview dump` writes them as `local_variables` (schema version 5; older JSON still opens).
- **Details** shows a node as the editor does: a variable get or set its variable, a function entry or custom event its signature, an animation node its settings, other nodes nothing. The node's class, GUID, pins and saved properties and the *Info* categories moved to **Developer Settings** → *Show node internals in Details* in the version label's menu (off by default, kept for the next start), and *Graph Node Node* is now *Graph Node*. In a diff, an added or removed node still lists its pins.
- **Find References** on the graph: right-click a variable's get or set node (a member, a local, an inherited or another class's variable).
- **Find References** finds the member itself, not every name, pin or default containing its name: *Health* no longer finds *MaxHealth*, *Set Health Bar* or a pin named *HealthRegen*. A variable finds its get and set nodes, a function or macro its entry and calls, a custom event itself and its calls, a component its gets and bound events.
- *Find Results* and *Find in Blueprints* give a multi-line result (a tooltip with line breaks) the room it needs, instead of drawing it over the rows around it.
- **Blueprint editing (experimental) follows the Blueprint style guides.** New nodes line up with their wires: a node that continues an exec wire goes level with it (after an event too, whose exec pin sits below its delegate pin), and a getter or other pure node goes left of the input it feeds. Positions the plugin works out snap to the editor's 16 grid, and existing nodes are never moved to line anything up. `"layout": "none"` keeps the old placement.
- **Style findings:** `blueprintreview edit validate` and `edit apply` report where a plan departs from Allar's and Unreal Directive's style guides and Epic's best practices: bools without `b`, names that aren't PascalCase, dispatchers without `On`, RPC events without `Server`/`Client`/`Multicast`, editable variables without a tooltip or category, public functions without a description, assets without their prefix, and, from the plugin, nodes left unwired, unhandled *Cast Failed*, exec wires running backwards and functions taken past 50 nodes. Naming follows the Blueprint's own convention where it has one. They are warnings; `"style": "strict"` or `--strict-style` makes them fail the plan.
- *Guide*, *Troubleshooting*, *Buy* and the other links to the website open the browser in the desktop app; before, they did nothing and logged `Cannot open url - feature "links" not enabled`.

## [1.2.0-beta4] - 2026-11-04

- **A user guide** at [blueprintreview.efenggamedev.com/docs/](https://blueprintreview.efenggamedev.com/docs/): a step-by-step page for every task (opening and searching Blueprints, comparing, reviewing in git and Perforce, Time-lapse, pull requests, edit plans), with a sample project to follow along.
- **Try it in your browser:** the landing page's button opens the viewer at [blueprintreview.efenggamedev.com/try/](https://blueprintreview.efenggamedev.com/try/), with no sign-in. It opens a diff from the history of a demo project, a player character, and **Samples** lists its Blueprints and the guide's sample project's, with a diff for every change in their history.
- **Help** at the top right opens the guide, the shortcuts and the release notes, or starts an email to support; F1 opens the guide.
- **Advanced pins:** the arrow at the bottom of a node shows or hides its advanced pins, on both sides of a diff, and nodes saved with their advanced pins shown open that way.
- Variables, defaults and properties of a user-defined struct type show every field (with the field names the editor uses, and *N members* on array elements), instead of `()` when the fields were at their defaults.
- An enum asset lists its entries by name (*Walk*, *Run*), not as `NewEnumerator0`, `NewEnumerator1`; the internal name is in the tooltip.
- The diff view outlines changed nodes with a thin, even line that follows the node's shape at every zoom (in reports too), a comment's title bar sits inside its outline as in the editor, and unchanged graphs in *Changes* are dimmed.
- Node icons are solid: the event diamond, the interface-message envelope and the server-only and client-only corner icons.
- Data table diffs show a value the old row left at the struct's default as that default, not as added, and field names without their internal suffix when the row struct isn't found.
- Blueprint Edit Automation: spawning nodes such as *Spawn Actor from Class* no longer crashes, a second *Return Node* gets one set of pins, an override event replaces the disabled ghost a new Blueprint has, and *set_root_component* replaces the default scene root instead of listing the new root twice. `blueprintreview edit apply` says where the editor stopped when it crashes.
- **The beta has an end date:** everything stays unlocked until the end of 30 November 2026. After that, BlueprintReview without a license runs the 14-day Pro trial (counted from that date if you first started it during the beta), then Lite. Beta testers' keys keep Pro for every 1.x version.
- A file that trips over a bug in the reader shows *ran into an internal error* instead of opening, and the app goes on: no crash dialog, no folder index stuck short of the end, no *Opening…* that never finishes. `blueprintreview textconv` prints a line about it instead of failing `git diff`.
- **Report a problem…** fills in the version, the system and the log folder; *Help* also copies the support address (for webmail) and opens the log folder. After a crash, the dialog says which backup log to send.
- The empty window has **Open…**, **Open Folder…** and **User guide** buttons, and a folder dropped on the window opens like *Open Folder…*. A file that won't open links to *Troubleshooting*.
- **Find References** in *My Blueprint*'s right-click menu searches for a variable, function, event or component, as in the editor.
- *Remove license* asks first, and forgets the license only once its place is free on the server (offline, it says so and offers *Forget it here anyway*). A license saved for another installation (after reinstalling Windows) says so, with *Free its place*. An activation still finishes if the window is closed meanwhile, connection errors read plainly, and a license that can't be saved says so.
- The license file is written safely: a crash while saving can't lose it, and a damaged one is kept as `License.json.bad` instead of being replaced.
- Source control: a git or Perforce command that hangs (an unreachable server) stops after a minute, and git's own errors, such as *detected dubious ownership*, are shown instead of *Not in source control*.
- The update check offers pre-releases only to pre-release builds.
- Blueprint editing (the *Blueprint Edit Automation* plugin, edit plans and `blueprintreview edit`) is marked experimental: the plugin shows as *Experimental* in the editor's Plugins window, and the docs say so. The plan format and the ops may still change.

## [1.2.0-beta3] - 2026-11-03

- The Microsoft Store package lists *BlueprintReview CLI* in the Start menu, as the Store requires. Started from there, it shows its help and waits for Enter; from a terminal it works as before.

## [1.2.0-beta2] - 2026-11-03

- **A setup wizard for the installer:** the license agreement, then *Install* for you alone, or *Advanced* to install for every user, choose the folder and pick what to install: the app (with or without *Open with* for `.uasset` files) and the command-line tool (with or without adding it to `PATH`). The last page can start BlueprintReview.
- While BlueprintReview is installed for every user, the installer updates that installation (Windows asks for administrator rights) instead of refusing.
- An update stays in the folder you installed BlueprintReview to.
- The Linux downloads hold the command-line tool alone, and this release has no macOS build.

## [1.2.0-beta1] - 2026-11-02

- **Display scale:** Ctrl+scroll (outside graphs), Ctrl+= / Ctrl+- / Ctrl+0 or the version label's menu scale the whole window, kept between runs.
- The Explorer's top row names the project of an opened `Content` folder (`MyGame/Content`), or its plugin.
- The Windows exes start on a PC without the Visual C++ Redistributable (`VCRUNTIME140.dll` was not found).
- The installer refuses to install for every user while BlueprintReview is installed for you alone, and the other way round, and says what to do, instead of making two installations that can't be removed cleanly.
- **Source control:** git and Perforce for the open asset, and a P4V-style *Time-lapse* of its history.
- **Update check:** the app reads the newest version from the site once a day and shows a *Version X available* button; the version label's menu turns it off or checks now.
- **Diff report:** a diff as a change list and side-by-side images of each changed graph, with *Save as HTML*. Lite shows diffs this way; Pro adds the interactive diff.
- Enum, struct and data table assets open, diff and have a `textconv` outline; timelines, user-defined structs and inherited component overrides are diffed.
- Re-routing a wire through reroute nodes is a cosmetic change.
- Copy for *My Blueprint* items and *Details* values, as the editor copies them.
- *Lock/Unlock* and the split orientation in the diff view.
- The app is always dark, as the editor is; the window title and a toolbar badge name the edition.
- **Blueprint editing:** the *Blueprint Edit Automation* plugin applies JSON edit plans to Blueprints, enums,
  structs and data tables, from a commandlet, the editor console or a local server in the open editor. A plan
  names nodes and pins by the GUIDs `blueprintreview dump` prints. `blueprintreview edit validate|apply|ops|schema`
  drives it ([EDITING.md](docs/EDITING.md)).
- Generated C++ files no longer name the converter's version in their header, so regenerating with a newer `blueprintreview` only changes the files whose code changed. The `.bpmap.json` manifests still record it.

## [1.1.1] - 2026-10-25

- **Folders open without freezing the window:** every file opens in the background, diffs included, and the Explorer shows a readable progress line while it indexes.
- Fixed a crash when opening a folder through the folder dialog; the folder now opens at the click.
- Windows builds draw with Vulkan again.
- The release archives ship `blueprintreview-app` next to the CLI.

## [1.1.0] - 2026-10-25

- **Folder mode:** open a project's `Content` folder to browse it in an *Explorer*, open assets in tabs, search every Blueprint in it (*Find in Blueprints*, Ctrl+Shift+F) and follow a double-click into other Blueprints, with *Back* and *Forward*.
- **git setup in one command:** `blueprintreview git-setup` writes the difftool and textconv config and the `.gitattributes` line; `git difftool -d` lists the changed assets in a *Files* panel.
- **Perforce:** P4V's diff opens in the app, with each side labelled with its revision; `blueprintreview p4-setup` prints the steps.
- Clear messages for Git LFS pointer files, cooked assets, non-Blueprint assets and engines too old or too new.
- An *Output Log* panel (Ctrl+`) and a log file; no console window; a crash dialog that points to the log.
- An app icon, and the version in the exes, the title bar and `blueprintreview --version`.
- A preview of the Blueprint to C++ converter: `blueprintreview cpp`, and *Convert to C++…* in the editor plugin.

## [1.0.0] - 2026-10-17

- **First release:** opens uncooked Blueprint `.uasset` files from UE 4 and UE 5 without the editor, and draws their graphs the way the Blueprint editor does, with *My Blueprint*, *Components* and *Details*.
- A side-by-side diff of two versions, laid out like the editor's *Blueprint Diff* window, from *Diff against…* or as your `git difftool`, with each side labelled with its revision.
- `blueprintreview textconv`, so `git diff` shows Blueprint changes as text instead of "Binary files differ".
- `blueprintreview diff` (Markdown or JSON) and `blueprintreview render --format svg` for reviews and scripts.
- Copy a node selection (Ctrl+C) as the editor's copy text, to paste into the Blueprint editor or blueprintue.com.
- *Find in Blueprint* (Ctrl+F) across graphs, variables and functions.
- The GitHub Action: a pull request comment listing the Blueprint changes, with links to an interactive diff in the browser.
- The CLI's `info`, `props`, `dump`, `check`, `copy` and `schema` commands, and the `BlueprintExporter` editor plugin that exports the same JSON from inside UE.
