# BlueprintReview user guide

BlueprintReview opens Unreal Engine Blueprint assets (`.uasset`) without the editor. You can read the graphs, search them, copy nodes, and compare two versions side by side. This short guide covers Windows. The full [user guide](https://blueprintreview.efenggamedev.com/docs/) has a step-by-step page for every task, and the [command-line reference](https://blueprintreview.efenggamedev.com/docs/cli/) every command and option.

It reads uncooked editor assets saved by UE 4.25 to 4.27 and 5.0 to 5.8. Cooked assets from a packaged game have no graphs left in them, so they don't open.

## 1. Install

- **Installer (`.msi`):** run `blueprintreview-app-x86_64-pc-windows-msvc.msi`. It isn't code-signed yet, so Windows SmartScreen asks once: choose *More info*, then *Run anyway*. The wizard shows the license agreement. *Install* installs for you alone, needs no administrator rights and puts the files into `%LOCALAPPDATA%\Programs\BlueprintReview`. *Advanced* lets you install for every user instead (Windows asks for administrator rights), choose the folder, and pick what to install: the app (with or without the *Open with* entry) and the command-line tool (with or without adding it to `PATH`). The last page can start BlueprintReview. It also adds a Start menu entry, puts `blueprintreview` on your `PATH`, and adds BlueprintReview to Explorer's *Open with* list for `.uasset` files. It doesn't take over the default program for `.uasset`. To uninstall, use *Settings → Apps*.
  - **For every user of a machine**, silently (IT deployment): from an administrator prompt, run `msiexec /i blueprintreview-app-x86_64-pc-windows-msvc.msi ALLUSERS=1`. This installs into `C:\Program Files\BlueprintReview` and uses the system `PATH`. Add `/qn` for a silent install. Updates go the same way, with `ALLUSERS=1`. An installation is either for you alone or for every user: to switch, uninstall the one you have first, as the installer refuses a second one. While BlueprintReview is installed for every user, the wizard updates that installation.
- **Zip:** unpack `blueprintreview-app-x86_64-pc-windows-msvc.zip` anywhere. Keep `blueprintreview-app.exe` and `blueprintreview.exe` in the same folder, because the git setup looks for the app next to the CLI.

## 2. Open a Blueprint

![A Blueprint open in BlueprintReview](https://blueprintreview.efenggamedev.com/img/viewer.jpg)

- **One asset:** double-click a `.uasset` (*Open with → BlueprintReview*), drag it onto the window, or use **Open…**.
- **A whole project:** use **Open Folder…** and pick the project's `Content` folder. The *Explorer* lists the assets, and each opens in its own tab. Its top row names the project, e.g. `StackOBot/Content`.

The window is laid out like the Blueprint editor: *Components* and *My Blueprint* on the left, the graphs in the middle, *Details* on the right. Nothing can be edited.

- **Move around:** drag with the right mouse button to pan, and use the wheel to zoom (hold **Ctrl** to zoom past 1:1, as in the editor). **F** fits the graph to the window.
- **Follow a call:** double-click a node to open what it calls. **Alt+Left** goes back.
- **Copy nodes:** select them and press **Ctrl+C**, then paste into the Blueprint editor or into blueprintue.com.
- **Search:** **Ctrl+F** searches the open Blueprint; **Ctrl+Shift+F** searches every Blueprint in the folder.
- **Display scale:** **Ctrl+scroll** over the panels, **Ctrl+=** and **Ctrl+-** make everything larger or smaller, **Ctrl+0** resets it. The version label's menu at the top right has the same buttons. The scale is kept for the next start.

![Find in Blueprints](https://blueprintreview.efenggamedev.com/img/find.jpg)

## 3. Compare two versions

![The diff view](https://blueprintreview.efenggamedev.com/img/diff.jpg)

With an asset open, click **Diff against…** and pick the older version. The old version is on the left, the new one on the right. *Changes* lists every difference in the editor's wording, and clicking a row takes you to it. Added nodes are green, removed nodes red, changed nodes cyan.

**Report** shows the same diff as one page: the list of changes, then an image of each changed graph. **Save as HTML…** writes that page as a single file you can send to someone.

![The list of changes](https://blueprintreview.efenggamedev.com/img/changes.jpg)

**History** (the clock button) shows the asset's revisions from git or Perforce on a bar, and you can step through what each one changed.

![Time-lapse](https://blueprintreview.efenggamedev.com/img/timelapse.gif)

## 4. Set up git

Run this once in a terminal:

```sh
blueprintreview git-setup --global
```

Then, in your project's repository:

```sh
blueprintreview git-setup
```

The second command adds a line to `.gitattributes`. Commit that file, and teammates only need the first command.

After that:

- `git diff`, `git log -p` and `git show` print what changed in a Blueprint as text, instead of `Binary files differ`.
- `git difftool -t blueprintreview -- Content/BP_Hero.uasset` opens the diff view.
- `git difftool -d -t blueprintreview HEAD~1` lists every Blueprint a commit changed, in one window.

Assets in Git LFS work, as long as `git lfs` is installed.

## 5. Set up Perforce (P4V)

Run `blueprintreview p4-setup`; it prints these steps with your install path filled in. In P4V, open *Edit → Preferences → Diff*, and under *Specify diff application by extension* add:

| Field | Value |
|---|---|
| Extension | `.uasset` |
| Application | `%LOCALAPPDATA%\Programs\BlueprintReview\blueprintreview-app.exe` (`C:\Program Files\BlueprintReview\blueprintreview-app.exe` when installed for every user) |
| Arguments | `--diff %1 %2` |

*Diff Against Have Revision*, *Diff Against Previous* and the diffs in the *History* tab then open in BlueprintReview.

## 6. Editions, trial and license

BlueprintReview comes in two editions. **Lite** is free: it opens and searches Blueprints, copies nodes, and shows diffs as a report. **Pro** adds the interactive diff, the Time-lapse history, and the `blueprintreview diff` and `blueprintreview render` commands.

A new install runs as Pro for 14 days, with nothing to register. After that it drops back to Lite until you activate a license. During the beta, everything is unlocked.

The label at the top right shows where you stand, and the window title says it too:

| Label | Meaning |
|---|---|
| **PRO TRIAL · 12 DAYS** | the Pro trial, with the days left |
| **BETA** | the beta: everything is unlocked |
| **LITE** | the trial has ended; Pro's features are marked *PRO* |
| **PRO** | licensed |

In the trial's last three days, and once after it ends, a thin bar under the toolbar reminds you. **Not now** hides it until the next day. In Lite, *Interactive diff · PRO* and the *History* button still show; clicking them says what Pro adds.

To activate a license:

1. Click **Activate…** (or **Upgrade…** in Lite) at the top right, next to the label. Clicking the label works too.
2. Paste your key and click **Activate**. No key yet? **Buy a license…** in the same window opens the pricing page.

Activation happens once per computer. It sends the key and an anonymous machine ID: a hash, never your machine's name. After that, the license is checked offline. A Pro key works on two computers at once (say a desktop and a laptop), and a Team key on as many as it has seats. To move a license to another computer, choose **Remove license** on the old one first; that frees its place. If a computer is gone for good, write to support@efenggamedev.com and we'll free it.

## 7. Updates

Once a day, at start-up, BlueprintReview reads the newest version number from `blueprintreview.efenggamedev.com`. When a newer version is out, a *Version X available* button appears next to the version number and opens the download page. Nothing is installed automatically. To turn the check off, untick **Check for updates daily** in the version number's menu. Installed from the Microsoft Store, BlueprintReview doesn't check: the Store updates it.

Apart from this check and license activation, BlueprintReview makes no network calls. Your assets never leave your machine.

## 8. When something goes wrong

- **"a cooked asset":** the file comes from a packaged build, and cooking strips the graphs. Open the asset from the project's `Content` folder instead.
- **"a Git LFS pointer, not the asset":** the real file hasn't been downloaded yet. Run `git lfs pull`.
- **Something looks wrong or the app crashes:** click **Output Log** at the top right, or press ``Ctrl+` ``. The log file is in `%LOCALAPPDATA%\BlueprintReview\Logs`. Please attach `BlueprintReview.log` when you write to support@efenggamedev.com.
