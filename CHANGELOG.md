# Changelog

What changed in each release of BlueprintReview (the app `blueprintreview-app` and the CLI `blueprintreview`). A release tag `vX.Y.Z` needs a `## [X.Y.Z] - date` section here: `.github/workflows/release.yml` takes the release notes from it (`scripts/release.py notes`), and the app's update check shows its first line.

## [Unreleased]

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
