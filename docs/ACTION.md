# BlueprintReview GitHub Action

The Action comments on a pull request with what changed in its Unreal Engine Blueprint assets: one table row per changed Blueprint, a link to its interactive side-by-side diff in a BlueprintReview web viewer, and each Blueprint's change list (variables, components, class defaults, graphs, nodes, pins and links). It updates the same comment on each push. Side-by-side SVGs of the changed graphs are attached to the workflow run as an artifact.

It runs [`blueprintreview pr-report`](CLI.md#blueprintreview-pr-report--a-pull-requests-blueprint-changes).

## Using it

Add `.github/workflows/blueprint-review.yml` to the project's repository:

```yaml
name: Blueprint review
on:
  pull_request:
    paths: ["**/*.uasset"]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true          # the assets, if they are in Git LFS
          fetch-depth: 0     # the history back to the merge base
      - uses: blueprintreview/action@v1
        with:
          viewer-url: https://blueprintreview.example.com/view/   # optional: links to the interactive diff
```

During the beta the PR viewer isn't open to other repositories, so leave `viewer-url` out: the comment then has the change lists and no links, and everything else works the same. To try the viewer on your repository, write to support@efenggamedev.com.

`@v1` exists once the first `v1.x.y` release is public (see [Releases](#releases)). During the beta there is none yet: use a pre-release's tag, `blueprintreview/action@v1.2.0-beta4`. The action is in [blueprintreview/action](https://github.com/blueprintreview/action) (MIT), and the binaries it downloads in the releases of [blueprintreview/blueprintreview](https://github.com/blueprintreview/blueprintreview/releases).

### Inputs

| Input | Default | What it does |
|---|---|---|
| `viewer-url` | empty | Address of a BlueprintReview web viewer (`…/view/`). Each changed Blueprint then links to its diff there. Without one, the comment has no links. |
| `pathspec` | `*.uasset` | The files to look at, as git pathspecs, one per line. |
| `cosmetic` | `false` | Also list cosmetic changes (moved or resized nodes, comment colours). |
| `comment` | `true` | Post or update the PR comment. The report goes to the job summary either way. |
| `upload-svgs` | `true` | Upload the changed graphs as SVGs (artifact `blueprintreview-svgs`). |
| `base`, `head` | the PR's | The commits to compare, for runs outside `pull_request` events. The old side is their merge base. |
| `version` | the action's ref | Which `blueprintreview` release to download (e.g. `v1.2.0`; `@v1` gets the newest `v1.x.y`). |
| `release-repo` | `blueprintreview/blueprintreview` | The repository whose releases hold the binaries. |
| `github-token` | `github.token` | Token for the comment and the release download. |

Outputs: `blueprints` (how many Blueprints changed) and `report` (the full report's path).

### What it does

1. Gets the `blueprintreview` binary: the release asset for the runner (Linux x64/arm64, Windows x64) matching the action's tag, checked against the release's `SHA256SUMS`. With no matching release it fails, except in the source repository, where `uses: ./` builds it from source (a few minutes the first time).
2. Runs `pr-report` on the PR's base and head commits. Git LFS objects of the base version are downloaded as needed.
3. Writes the full report to the job summary.
4. Finds its earlier comment by the report's first line and updates it, or posts a new one. It posts nothing when a PR has no Blueprint changes and there's no earlier comment; an earlier comment changes to "No Blueprint changes".
5. Uploads the SVGs.

Pull requests from forks get a read-only token, so they get no comment. The report is still in the job summary.

### Private repositories

The action and its releases are public, so private repositories use it like public ones; the default `github-token` can read public releases.

### Releases

`.github/workflows/release.yml` runs on a `vX.Y.Z` tag in the private source repository (`blueprintreview/blueprintreview-dev`). It builds the binaries and `SHA256SUMS` (`blueprintreview-<target>.tar.gz`, the CLI the action downloads, and `blueprintreview-app-<target>`, the viewer app with the CLI). Where they go depends on the repository variable `PUBLIC_RELEASES`:

- unset: a release of the private repository, and its major tag (`v1`) moved;
- `true` (on since 1.2.0-beta2): a release of the public `blueprintreview/blueprintreview` (a pre-release for tags like `v1.2.0-rc1`), and `action.yml` copied to `blueprintreview/action`, tagged there with the same tag and the major tag, so `@v1` picks up the release. Both use the `PUBLIC_REPO_TOKEN` secret, a fine-grained token with *Contents: read and write* on those two repositories (at most 366 days, which the org requires).

`RELEASE_CHECKLIST.md` lists what to check before turning `PUBLIC_RELEASES` on (`gh variable set PUBLIC_RELEASES --body true -R blueprintreview/blueprintreview-dev`).

## Hosting the viewer

During the beta, the PR viewer (`blueprintreview.efenggamedev.com/view/`) isn't open to other repositories, and hosting one needs files that aren't public yet. Leave `viewer-url` empty, or write to support@efenggamedev.com to try it. The rest of this section is how the maintainers host it.

The links point at the web viewer's `/view/` page, which needs the site's Worker (`site/worker/index.js`) to read files from GitHub as the signed-in user. Without a viewer, leave `viewer-url` empty: the comment then has the change lists and no links.

The site is a Cloudflare Worker (`site/wrangler.toml`): the trunk web build as static assets (`/view/`, plus the landing page and the demo), and a small script (`site/worker/index.js`) that runs only for `/auth/*` (GitHub sign-in) and `/api/*` (the blob proxy). `.github/workflows/web-demo.yml` builds and deploys it. The setup below is a one-time job, in this order. Steps 1–3 are the site itself, also described in the maintainers' notes (`docs/internals/hosting.md`); steps 4–6 add the sign-in.

### 1. Put the private paths behind an access gate (before the first deploy)

The internal demo (`/demo/`) ships internal fixtures, and the viewer and its sign-in read private repositories, so they must stay private (`RELEASE_CHECKLIST.md`); the landing page, `/docs/`, `/changelog/` and the public demo `/try/` are public. In Cloudflare Zero Trust, go to *Access → Applications → Add an application → Self-hosted*. Use the domain (`blueprintreview.efenggamedev.com`) with the paths `demo`, `view`, `auth` and `api` (one destination each; every path under them is covered), and add an Allow policy for your team's email addresses. The first deploy makes the custom domain live, so this must exist first.

### 2. Let GitHub Actions deploy

Create a Cloudflare API token from the *Edit Cloudflare Workers* template, covering the account and the zone (*Zone → Workers Routes → Edit*, for the custom domain). Add it and the account ID as the repo secrets `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID`.

### 3. Deploy

Push to `main`, or run *Actions → Web demo → Run workflow* on a branch. After deploying, the workflow checks that `/demo/`, `/view/`, `/auth/login` and `/api/blob` redirect to the Access login and that `/try/` answers 200, and fails if one doesn't; it warns if `/`, `/docs/` or `/changelog/` don't answer 200.

### 4. Create the GitHub App

The App is the identity "Sign in with GitHub" uses. A token from it can read only the repositories the App is installed on, and only what the signed-in person can read there.

1. *GitHub → avatar → Settings → Developer settings → GitHub Apps → New GitHub App*. If the project repositories belong to an organization, create it under the organization's *Developer settings* instead.
2. Fill in:

   | Field | Value |
   |---|---|
   | GitHub App name | Any unused name, e.g. `BlueprintReview Viewer`; it appears on GitHub's authorize page |
   | Homepage URL | `https://<host>/` |
   | Callback URL | `https://<host>/auth/callback`, exactly; OAuth calls this the redirect URI |
   | Expire user authorization tokens | on (tokens last 8 hours; the Worker refreshes them) |
   | Request user authorization (OAuth) during installation | off |
   | Enable Device Flow | off |
   | Webhook → Active | **off** |
   | Repository permissions | *Contents: Read-only*, nothing else (*Metadata: Read-only* is added automatically) |
   | Where can this GitHub App be installed? | Only on this account |

3. *Create GitHub App*. On its page, copy the **Client ID** (`Iv23li…`). This is not the numeric *App ID*.
4. *Client secrets → Generate a new client secret*, and copy it with the copy button. GitHub shows it only once. *Private keys* aren't needed.
5. *Install App → Install → Only select repositories*: pick every repository whose PRs link to the viewer. Add more later under *Settings → Applications → Installed GitHub Apps → Configure*.

GitHub App tokens work with Git LFS (checked with LFS-stored fixtures). An OAuth App works too: create it with the same callback URL, and also set the Worker secret `GITHUB_SCOPE` to `repo`. That token reaches all of the user's repositories, so prefer the App.

### 5. Give the Worker its secrets

In a terminal on your machine, from the repository's `site/` folder (its `wrangler.toml` names the Worker):

```sh
cd site
npx wrangler login                              # once: approve in the browser
npx wrangler secret put GITHUB_CLIENT_ID        # paste the Client ID at the prompt (nothing echoes)
npx wrangler secret put GITHUB_CLIENT_SECRET    # paste the client secret
# 32 random bytes that encrypt the session cookie:
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))" | npx wrangler secret put COOKIE_KEY
npx wrangler secret list                        # the three names should be listed
```

The same Worker's licensing secrets (license keys, Paddle, email) are in [LICENSING.md](LICENSING.md), "Secrets".

Secrets live on Cloudflare, not in the repository, and survive later deploys. To change one, run its `secret put` again; it takes effect at once.

### 6. Check it

Open a link to any Blueprint in a repository the App is installed on, with a full commit sha:

```
https://<host>/view/?repo=<owner>/<repo>&head=<40-character sha>&path=<path/to/BP_Foo.uasset>
```

Expect the Access login, then **Sign in with GitHub** in the viewer's toolbar, then GitHub's authorize page (the first time only), then the Blueprint. Then set the action's `viewer-url` to `https://<host>/view/`.

### Troubleshooting

| What you see | What to do |
|---|---|
| `GitHub didn't issue a token (incorrect_client_credentials …)` | The Client ID or secret is wrong. If GitHub's authorize page showed your App, the ID is right: generate a new client secret and `secret put GITHUB_CLIENT_SECRET` again. Otherwise check that `GITHUB_CLIENT_ID` is the Client ID, not the App ID. |
| `… (redirect_uri_mismatch …)`, or GitHub says the redirect_uri is not associated with the application | The App's Callback URL must be exactly `https://<host>/auth/callback`. |
| `… (bad_verification_code …)` | The sign-in took too long or was reused; open the link again. |
| `sign-in expired or was started elsewhere` | The sign-in was started in another tab or more than 10 minutes ago; open the link again. |
| `HTTP 403/404: no access to … is the GitHub App installed there?` | Install the App on that repository (step 4.5), or the signed-in person can't read it. |
| `HTTP 502: … the Git LFS object couldn't be downloaded` | The token can't read LFS objects; switch to an OAuth App with `GITHUB_SCOPE=repo`. `npx wrangler tail` (in `site/`) shows the Worker's log while you retry. |
| The comment has no links | The workflow doesn't set `viewer-url`. |

### How a link opens

The viewer fetches `/api/blob?repo=…&ref=<sha>&path=…` for each side. Without a session the Worker answers 401, and the viewer shows **Sign in with GitHub**: `/auth/login` goes to GitHub's authorize page, and `/auth/callback` exchanges the code for a token and returns to the same diff. The token is kept in an AES-GCM encrypted, HttpOnly cookie (`COOKIE_KEY`), refreshed when it expires, and is never visible to the page. The Worker reads the file through the contents API and, for a Git LFS pointer, downloads the object through the repository's LFS batch API. Files are cached in the browser only; nothing is stored on the server.
