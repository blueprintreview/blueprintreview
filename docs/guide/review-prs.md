# Review pull requests

With the BlueprintReview GitHub Action on a repository, every pull request that changes Blueprints gets a comment
listing what changed, Blueprint by Blueprint, with a link to each one's side-by-side diff in your browser. This page
is for reviewers; whoever looks after the repository sets the Action up from the
[GitHub Action](../ACTION.md) page.

## Read the comment

When a pull request changes `.uasset` files, the Action posts one comment, and updates it on every push:

1. **A table** of the changed Blueprints (*Blueprint*, *Change*, *Summary*): each one's path, whether it was added, deleted, renamed or changed, and a
   summary of its changes (nodes, variables, components).
2. **Each Blueprint's changes**, in a section you can open: its variables, components and class defaults, then
   each graph with the nodes added, removed and changed, and their wires.

Moved nodes and re-routed wires are counted, but listed apart from real changes.

![The Action's comment on a pull request: the table of changed Blueprints, one section open](img/pr-comment.jpg)

The same report is in the workflow run's *Summary*, and side-by-side images of the changed graphs are attached to the
run as an artifact (`blueprintreview-svgs`).

## Open the side-by-side diff

During the beta, the PR viewer isn't open to other repositories yet, so the comment has the change lists only; to try
the viewer on your repository, write to support@efenggamedev.com. When the repository has a viewer set up, each row ends with an **Open diff** link (**Open** for an added or deleted Blueprint):

1. Click it. The diff opens in your browser, in the same diff view as the desktop app
   ([Compare two versions](compare.md)): the base version on the left, the pull request's on the right, each side
   labelled with its commit.
2. The first time, the viewer asks you to **Sign in with GitHub**, so it can read the repository's files as you.
   It reads them for this diff only; nothing is stored on a server, and the files never leave your browser.

## Review it on your machine instead

The desktop app shows any pull request's branch too:

```sh
git fetch origin pull/123/head:pr-123
git difftool -d -y -t blueprintreview main...pr-123
```

See [Review changes in git](review-git.md#compare-a-branch-with-main).

## Try it on the sample

The sample has a branch to review: `feature/double-points`. Push your clone to a repository of your own with the
Action set up, open a pull request from that branch, and its comment lists `BP_ScoreGameMode`, with *AddScore*
changed and *StartDoublePoints* added. Without a repository, review it locally:
`git difftool -d -y -t blueprintreview main...feature/double-points`.

## Related

- [GitHub Action](../ACTION.md): setting it up, and hosting the viewer.
- [Share a diff](share.md)
