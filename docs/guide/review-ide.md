# Diff from Rider or VS Code

Open `.uasset` diffs from your IDE in BlueprintReview's side-by-side diff view. No IDE plugin is needed: Rider calls
BlueprintReview as an external diff tool, and VS Code runs it through git.

## Rider

Rider knows `.uasset` and `.umap` files (its *UAsset* and *UMap* file types), so you can give them their own diff
tool.

1. Open **Settings → Tools → Diff & Merge → External Diff Tools**.
2. Tick **Enable external tools**, then add a tool:

   | Field | Value |
   |---|---|
   | Tool group | Diff tools |
   | Program path | the full path of `blueprintreview-app.exe`, e.g. `%LOCALAPPDATA%\Programs\BlueprintReview\blueprintreview-app.exe` |
   | Tool name | `BlueprintReview` |
   | Argument pattern | `--diff %1 %2` |

3. In the list of file types on the same page, add **UAsset** with the *BlueprintReview* tool, and **UMap** if you
   want level Blueprints too.
4. Click **OK**.

Rider's diffs of those files now open in BlueprintReview: *Compare with…*, *Show Diff* in the *Commit* and *Git*
tool windows, and the *History* tab. `%1` is Rider's left side, `%2` its right side.

Your working copy is labelled `Local Revision`. A version Rider takes from git is labelled with its commit when the
app can find it (see [Review changes in git](review-git.md)).

## VS Code

VS Code has no external diff tool setting, so use git's:

1. Set up git once, as in [Review changes in git](review-git.md#set-it-up-once).
2. Run `git difftool` in VS Code's terminal (**Ctrl+`**):

   ```sh
   git difftool -y -t blueprintreview -- Content/Characters/BP_Hero.uasset
   ```

To run it on the open file without typing the path, add a task to `.vscode/tasks.json`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "BlueprintReview: diff with HEAD",
      "type": "shell",
      "command": "git difftool -y -t blueprintreview HEAD -- \"${relativeFile}\"",
      "problemMatcher": []
    }
  ]
}
```

Open the `.uasset` (VS Code says it's a binary file, which is fine), then run **Terminal → Run Task… →
BlueprintReview: diff with HEAD**.

## Related

- [Review changes in git](review-git.md): `git difftool`, `git diff` as text.
- [Review changes in Perforce](review-perforce.md): P4V's diff setting.
- [Compare two versions](compare.md): reading the diff view.
