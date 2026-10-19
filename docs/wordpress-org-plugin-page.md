# WordPress.org Plugin Page

This reusable workflow updates what the [WordPress.org plugin directory](https://wordpress.org/plugins/) shows on a plugin's page (the `readme.txt` and the banners, icons, screenshots, and Playground blueprints) **without releasing a new plugin version**.

Use it for changes like:

- adding or removing promotional links in `readme.txt`
- bumping `Tested up to` after verifying compatibility with a new WordPress release
- fixing typos, FAQ entries, or screenshot captions
- temporarily swapping a banner (e.g. for a campaign) and reverting it later
- refreshing screenshots or adding a Playground preview blueprint

To ship new plugin code, use [WordPress.org Release](./wordpress-org-release.md) instead. That workflow also publishes `readme.txt` as part of every release, so both workflows keep the plugin page in sync with Git.

To achieve that, this workflow:

1. Checks out the Git repository at `GIT_REF`
2. When `SYNC_README` is enabled, reads the `Stable tag` from `readme.txt`
3. When `SYNC_ASSETS` is enabled, validates every file in `ASSETS_DIR` against the WordPress.org naming rules and size limits, and fails if the directory is missing or empty
4. Checks out only the needed parts of the WordPress.org SVN repository: `trunk/readme.txt`, the list of tags, the readme of the stable tag, and `/assets`. Plugin code is never fetched
5. Verifies that the `Stable tag` in Git equals the `Stable tag` in SVN trunk and that this tag exists
6. Copies `readme.txt` into `tags/<Stable tag>/` (which is what WordPress.org renders) and into `trunk/`. The trunk copy is skipped with a warning if trunk contains unreleased changes, see [Trunk previews](#trunk-previews)
7. Mirrors `ASSETS_DIR` to SVN `/assets` via `rsync --delete`, so files removed from Git are removed from WordPress.org as well, and sets the correct `svn:mime-type` on every asset
8. Fails if anything outside `/assets`, `trunk/readme.txt`, and `tags/<Stable tag>/readme.txt` would change
9. Prints the full `svn status` and the `readme.txt` diffs
10. Commits all changes in a single SVN commit (or uploads them as an artifact in `DRY_RUN` mode). If nothing changed, the workflow ends successfully without committing

> [!IMPORTANT]
> This workflow never creates tags, never changes the `Stable tag`, and never touches plugin code. Changing the `Stable tag` means publishing a different version, which is a release.

> [!NOTE]
> WordPress.org renders the plugin page from the `readme.txt` in the tag that `Stable tag` points to, not from trunk. The WordPress.org [plugin developer FAQ](https://developer.wordpress.org/plugins/wordpress-org/plugin-developer-faq/#do-i-need-to-release-a-new-version-of-my-plugin-every-time-i-update-the-readme) explicitly allows cosmetic readme and asset changes without a new release, as long as both trunk and the tag are updated.

> [!NOTE]
> Assets are served through a CDN and cached heavily. Changes can take a few minutes, and under high load up to several hours, to show up on the plugin page.

## Simple usage example

This workflow cannot be triggered directly. Create a workflow file in your plugin's repository that calls it via `uses:`, as shown below.

```yml
name: Update WordPress.org plugin page
on:
  workflow_dispatch:
    inputs:
      GIT_REF:
        description: 'Git tag or branch to read readme.txt and assets from'
        required: true
      SYNC_README:
        description: 'Update readme.txt'
        type: boolean
        default: true
      SYNC_ASSETS:
        description: 'Update banners, icons, screenshots, and blueprints'
        type: boolean
        default: true
      DRY_RUN:
        description: 'Upload the changes as an artifact instead of committing to SVN'
        type: boolean
        default: true
jobs:
  update-plugin-page:
    uses: inpsyde/reusable-workflows/.github/workflows/wordpress-org-plugin-page.yml@main
    with:
      SVN_PLUGIN_SLUG: my-plugin
      GIT_REF: ${{ inputs.GIT_REF }}
      SYNC_README: ${{ inputs.SYNC_README }}
      SYNC_ASSETS: ${{ inputs.SYNC_ASSETS }}
      DRY_RUN: ${{ inputs.DRY_RUN }}
    secrets:
      SVN_USERNAME: ${{ secrets.SVN_USERNAME }}
      SVN_PASSWORD: ${{ secrets.SVN_PASSWORD }}
```

`GIT_REF` should point to a ref whose `readme.txt` describes the **currently published** version, such as the Git tag of the latest release plus the cosmetic change, or the branch that holds the released state. A development branch whose `readme.txt` already carries the next version's `Stable tag` fails the Stable tag check by design.

## Update assets automatically on merge

Because the assets directory is validated and mirrored, it can be synced whenever it changes on the main branch:

```yml
name: Update WordPress.org assets
on:
  push:
    branches: [main]
    paths: ['.wordpress-org/**']
jobs:
  update-assets:
    uses: inpsyde/reusable-workflows/.github/workflows/wordpress-org-plugin-page.yml@main
    with:
      SVN_PLUGIN_SLUG: my-plugin
      GIT_REF: ${{ github.sha }}
      SYNC_README: false
    secrets:
      SVN_USERNAME: ${{ secrets.SVN_USERNAME }}
      SVN_PASSWORD: ${{ secrets.SVN_PASSWORD }}
```

> [!NOTE]
> Only automate readme syncing if the branch's `readme.txt` always describes the published version. Otherwise, keep `SYNC_README: false` for automated runs and update the readme through a manually triggered run.

## Advanced usage: requiring manual approval

To require approval before the plugin page is updated, add an `environment:` key to the calling job and configure protection rules (required reviewers, wait timers, etc.) for that environment in your repository settings under **Settings → Environments**.

```yml
jobs:
  update-plugin-page:
    environment: wordpress-org  # enforces protection rules configured in repository settings
    uses: inpsyde/reusable-workflows/.github/workflows/wordpress-org-plugin-page.yml@main
    with:
      SVN_PLUGIN_SLUG: my-plugin
      GIT_REF: ${{ inputs.GIT_REF }}
    secrets:
      SVN_USERNAME: ${{ secrets.SVN_USERNAME }}
      SVN_PASSWORD: ${{ secrets.SVN_PASSWORD }}
```

> [!NOTE]
> The `environment:` key alone does nothing: protection rules must be explicitly configured in repository settings. An environment with no rules configured provides no approval gate.

## Adopting the workflow

Assets are mirrored, so the assets directory in Git must contain **every** asset that should stay on WordPress.org. Before the first run, import the current assets from SVN:

```shell
svn export https://plugins.svn.wordpress.org/my-plugin/assets .wordpress-org
```

Then:

1. Commit the `.wordpress-org` directory.
2. Add `.wordpress-org/` to `.distignore` so the assets don't end up in the plugin package or in SVN trunk during releases.
3. Run the workflow with `DRY_RUN: true`. The `svn-status.txt` in the artifact should list **no content changes** in `/assets`. Property-only changes (` M`) are expected where existing assets lack the correct `svn:mime-type`.
4. Fix any files the validation rejects. Such files are currently ignored by WordPress.org anyway, e.g. `icon_128x128.png` instead of `icon-128x128.png`.

## Trunk previews

If SVN `trunk/readme.txt` differs from `tags/<Stable tag>/readme.txt` before the sync, trunk holds unreleased changes, for example a preview published with the release workflow's `UPDATE_TRUNK_ONLY` mode. Overwriting it would silently revert those changes, so in that case the workflow only updates the tag's `readme.txt` and emits a warning. Make sure the same change is part of the upcoming release, so that it isn't lost when the next version is published.

## Supported asset files

Files must be lowercase (except the region part of a locale, e.g. `es_ES`). Hidden files like `.gitkeep` are ignored. Any other file fails the validation, because WordPress.org would ignore it.

| Type | File names | Formats | Max size |
|---|---|---|---|
| Banner | `banner-772x250`, `banner-1544x500` | png, jpg, jpeg, gif | 4 MB |
| Icon | `icon-128x128`, `icon-256x256`, `icon.svg` | png, jpg, jpeg, gif (`icon.svg` requires one of them as fallback) | 1 MB |
| Screenshot | `screenshot-1`, `screenshot-2`, … | png, jpg, jpeg, gif | 10 MB |
| Blueprint | `blueprints/blueprint.json`, `blueprints/blueprint-<name>.json` | JSON, max. 10 files | 100 KB |

Banners, icons, and screenshots can be localized with an `-rtl` and/or a locale suffix, e.g. `banner-772x250-rtl.png`, `banner-1544x500-de.png`, or `screenshot-1-es_ES.png`. Screenshot captions come from the `== Screenshots ==` section of `readme.txt`. See [Plugin Assets](https://developer.wordpress.org/plugins/wordpress-org/plugin-assets/) for details.

> [!NOTE]
> A blueprint enables the "Live Preview" button for plugin committers only. To show it to everyone, a committer must set the preview to public in the plugin's **Advanced View** on WordPress.org. See [Previews and Blueprints](https://developer.wordpress.org/plugins/wordpress-org/previews-and-blueprints/).

## Configuration parameters

### Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `SVN_PLUGIN_SLUG` | yes | — | WordPress.org plugin slug (e.g. `my-plugin`) |
| `GIT_REF` | yes | — | Git ref to read `readme.txt` and the assets directory from (tag, branch, or commit SHA) |
| `SYNC_README` | no | `true` | Update `readme.txt` in SVN trunk and in the tag referenced by `Stable tag` |
| `SYNC_ASSETS` | no | `true` | Mirror `ASSETS_DIR` to the SVN `/assets` directory |
| `ASSETS_DIR` | no | `.wordpress-org` | Directory in the Git repository holding the WordPress.org assets |
| `DRY_RUN` | no | `false` | Upload the prepared changes as an artifact instead of committing to SVN |

At least one of `SYNC_README` and `SYNC_ASSETS` must be enabled.

### Secrets

| Name | Required | Description |
|---|---|---|
| `SVN_USERNAME` | yes | WordPress.org SVN username |
| `SVN_PASSWORD` | yes | WordPress.org SVN password |

> [!NOTE]
> WordPress.org SVN does not support SSH keys or tokens. The `SVN_USERNAME` and `SVN_PASSWORD` secrets are the only supported authentication method.

## Dry run artifact

With `DRY_RUN: true`, nothing is committed. Instead, an artifact named `<slug>-plugin-page` is uploaded, containing:

- `svn-status.txt`: every file that would be added (`A`), deleted (`D`), modified (`M`), or have its properties changed (` M`)
- `changes.diff`: the full diff, including `readme.txt` changes
- `trunk/readme.txt` and `tags/<Stable tag>/readme.txt` as they would be committed
- `assets/`: the complete assets directory as it would be committed
