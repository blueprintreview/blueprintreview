# Review changes in Perforce

Make P4V open `.uasset` diffs in BlueprintReview, and see an asset's Perforce history in the app's
[Time-lapse](timelapse.md).

## Set up P4V once

1. In a terminal, run:

   ```sh
   blueprintreview p4-setup
   ```

   It prints these steps with the path of your BlueprintReview install filled in.
2. In P4V, open **Edit → Preferences**, then **Diff**.
3. Under *Specify diff application by extension (overrides default)*, click **Add**, and fill in:

   | Field | Value |
   |---|---|
   | Extension | `.uasset` |
   | Application | the full path of `blueprintreview-app.exe`, e.g. `%LOCALAPPDATA%\Programs\BlueprintReview\blueprintreview-app.exe` |
   | Arguments | `--diff %1 %2` |

4. Click **Save**, then **OK**.

![P4V's Preferences, Diff page, with the .uasset entry](img/p4v-diff.jpg)

## Review a change

In P4V, any diff of a `.uasset` now opens in BlueprintReview:

- **Your pending change:** right-click the file → **Diff Against Have Revision**.
- **What a submitted changelist did:** in its *Files*, right-click a file → **Diff Against Previous**.
- **Any two revisions:** diff them from the file's *History* tab, as you would any file.

Each side is labelled with its revision: `Revision 3` for `#3`, `Revision shelved in CL 1234` for a shelved file,
`Local Revision` for your workspace file. The diff view itself is described in
[Compare two versions](compare.md).

## Connect the app to Perforce

For [Time-lapse](timelapse.md) the app runs `p4` itself. It usually finds your workspace on its own, from the same
`P4CONFIG` files and settings P4V uses. The **Source Control** button at the top right says what it found: `✔ Perforce`,
or `Not in source control`.

If it doesn't find it, or your server needs a login:

1. Click the source control button at the top right.
2. Set **Provider** to *Perforce* (or leave *Auto (git, then Perforce)*).
3. Fill in **Server** (`P4PORT`, e.g. `ssl:perforce.example.com:1666`), **User Name** (`P4USER`) and
   **Workspace** (`P4CLIENT`), as in the Unreal editor's *Revision Control Login*. Empty fields come from your
   Perforce environment.
4. Click **Accept Settings**. If the server asks for a login, the window asks for your password and logs you in
   (`p4 login`).

The settings are saved in `%LOCALAPPDATA%\BlueprintReview\SourceControl.json`. The app needs the `p4` command-line
program, which P4V's installer puts on your `PATH`.

## Troubleshooting

- **P4V still opens its own diff:** check that the extension is `.uasset` with the dot, and that the entry is under
  *by extension*, not the default diff application.
- **`⚠ Perforce login` at the top right:** your ticket expired. Click it and log in.
- **`Not in source control` for a file in your workspace:** open the source control settings and fill in the
  server, user and workspace by hand.

We'd like to hear how this works with your server: write to
[support@efenggamedev.com](mailto:support@efenggamedev.com).

## Related

- [Time-lapse: an asset's history](timelapse.md)
- [Compare two versions](compare.md)
