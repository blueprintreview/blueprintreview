# Get the sample project

The guides use a small sample project, a coin collector, whose git history was made for them: each commit makes
one kind of change. Clone it once, and every page's *Try it on the sample* steps work as written.

## Before you start

- [Install BlueprintReview](install.md).
- Install [Git](https://git-scm.com/) and [Git LFS](https://git-lfs.com/). The `.uasset` files are stored with
  Git LFS.

You don't need Unreal Engine: BlueprintReview reads the files on its own. With UE 5.8 you can also open the project
in the editor.

## Steps

1. Clone the project and fetch its second branch:

   ```sh
   git lfs install
   git clone https://github.com/blueprintreview/sample-project.git
   cd sample-project
   git fetch origin feature/double-points:feature/double-points
   ```

2. Set up git for Blueprints, once for your account and once in the clone:

   ```sh
   blueprintreview git-setup --global
   blueprintreview git-setup
   ```

   The first command adds BlueprintReview as a difftool (`git difftool -t blueprintreview`) and as the text
   driver `git diff` uses for Blueprints. The second makes sure the clone's `.gitattributes` sends `.uasset`
   files to that driver (the sample's already does).
   [Review changes in git](review-git.md) explains both.

3. Open the project's `Content` folder in BlueprintReview: **Open Folder…**, then pick
   `sample-project/Content`. The *Explorer* on the left lists its assets.

## What's in it

All in `Content/Sample/`:

| Asset | What it is |
|---|---|
| `BP_Coin` | A spinning coin. `GetPoints` picks its points by `CoinType`; overlapping it scores them and removes it |
| `BP_CoinSpawner` | Spawns a `BP_Coin` every `Interval` seconds, `SpawnRadius` away |
| `BP_ScoreGameMode` | Keeps `TotalScore`; `AddScore` adds to it and calls `OnScoreChanged` |
| `Data/E_CoinType` | The coin types: Bronze, Silver, Gold, Platinum |
| `Data/S_CoinReward` | A reward: points, a label and a pick-up sound |
| `Data/DT_CoinRewards` | The rewards, a row per coin type |

## The history

`git log --oneline` lists the commits, newest first. Each guide names the one it uses by its message:

| Commit | What it changes |
|---|---|
| Add the coin collector… | the starting point |
| Spawner: SpawnRadius setting… | a new variable with a default, and a new wire |
| Coin: Silver coins; GetPoints selects the points by coin type | nodes replaced; an enum entry added |
| Coin: spin with a RotatingMovement component… | a component added, others changed |
| Coin: tidy the pick-up graph | nodes moved only: no real change |
| Spawner: route the transform wire through a reroute node | a re-routed wire: no real change |
| Game mode: rename Score to TotalScore | a renamed variable and the nodes that use it |
| Rewards: rebalance the points, add the Silver row | data table rows |
| Add the Platinum coin | an enum, a data table and a Blueprint changed together |
| Rewards: a pick-up sound per reward | a struct field |
| Coin: move the scoring into a ScoreCoin function | nodes collapsed into a function |
| Spawner: comment the timer and the spawn position | comments |
| Spawner: spawn every 1.5 s, up to 800 units away | class defaults |

The `feature/double-points` branch adds two commits on top: a *double points* flag in `AddScore`, and a
`StartDoublePoints` event. [Review changes in git](review-git.md) compares it with `main`.

Every change was made by an edit plan (an experimental feature), without opening the editor; the plans are in the project's `EditPlans/`
folder. [Your first edit plan](edit-tutorial.md) walks through one.

## Next

[Open and browse a Blueprint](open-and-browse.md).
