# airtable-interface-deploy

Deploy Airtable custom interface code from GitHub Actions on merge to main.

Airtable does not support CI/CD for the Blocks CLI. Their documentation says so directly. However, `block release` is just a Node CLI and runs as well on a runner as it does on a laptop. Treat this solution as unsupported: pin the CLI version, and expect to fix it when Airtable changes something.

## What airtable-interface-deploy gives you

Code in GitHub instead of on a laptop. Merge to main, and the released version of your custom interface element updates in the base without anyone running a command.

What it does not do: create the block, place the element on an interface page, or configure its custom properties. Those are one-time manual steps in the Airtable UI.

## Prerequisites

- An Airtable custom interface element already created in a base, built with the Interface Extensions SDK. You need its `app...` and `blk...` ids.
- A working local project. `block run` should render the element at `https://localhost:9000` before you automate anything.
- `.block/remote.json` committed to your repo:

```json
{
  "blockId": "blkXXXXXXXXXXXXXX",
  "baseId": "appXXXXXXXXXXXXXX"
}
```

That file holds no secrets. It is what lets a fresh clone know which block to target.

- A personal access token with the `block:manage` scope and access to that base. Use a dedicated client service account rather than your own login, so releases do not break when one person's access changes.

## Setup

1. Copy `deploy.yml` into your project at `.github/workflows/deploy.yml`.
2. Add the token as a repository secret named `AIRTABLE_BLOCK_PAT` under Settings, Secrets and variables, Actions.
3. Confirm `.airtableblocksrc.json` is in your `.gitignore`. That is where the CLI writes tokens locally. A leaked `block:manage` token is a direct write path to your base.
4. Merge to main.

## The workflow

```yaml
name: Deploy to Airtable

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: airtable-release
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm install -g @airtable/blocks-cli@3.0.3
      - run: block set-api-key --location app "$PAT"
        env:
          PAT: ${{ secrets.AIRTABLE_BLOCK_PAT }}
      - name: Release
        env:
          MSG: ${{ github.sha }}
        run: |
          set -o pipefail
          printf '%s\n' "$MSG" | block release 2>&1 | tee release.log
          if ! grep -qF "Successfully released block" release.log; then
            echo "Release did not publish"
            exit 1
          fi
      - if: always()
        run: rm -f .airtableblocksrc.json release.log
```
## Things worth knowing

Some things that cost me a couple afternoon and are the reason this repo exists.

### `block release` silently no-ops without a comment

`block release` prompts for a comment describing the release. If it gets no input, it exits zero and publishes nothing. A workflow built on `block release < /dev/null` reports green on every merge while the base stays frozen.

Pipe a real string instead:

```
printf '%s\n' "$MSG" | block release
```

The workflow passes the commit SHA, which also makes Airtable's release history useful for telling versions apart. Keep the message to one line and plain ASCII. Commit messages can carry quotes and newlines that confuse a prompt reader.

Because an empty comment fails quietly, the workflow also greps the CLI output for its success line and fails the job if it is missing. Without that check, a no-op passes.

### Airtable enforces a server-side CLI minimum

Airtable rejects releases from a CLI below a minimum version they can raise without notice. When that happens, a workflow pinned to an exact version starts failing on a day you changed nothing.

Pin anyway, for reproducibility. Just know what the failure looks like. The error message is also misleading: it tells you to run `npm i -g @airtable/blocks`, which is the SDK, not the CLI. The package you need is `@airtable/blocks-cli`.

Notes on a few lines:

`concurrency` stops two merges racing to upload at once.

`set -o pipefail` makes the job fail if `block release` itself errors, not just if the grep misses.

The grep matches plain text rather than the emoji the CLI actually prints, since an encoding difference on the runner should not fail a good release.

The last step deletes the file `block set-api-key --location app` writes into the working directory, so the token cannot end up in an artifact or a cache.

### Merging is publishing

There is no draft or staged release state in Airtable. Once the upload lands, that version is live for anyone using the element. If you want a pause, add `environment: production` to the job and configure required reviewers on that environment.

### Nothing here verifies the element still renders

The job confirms the upload succeeded, not that the code works. A dependency bump that breaks at runtime will deploy green.

### One block per base - for now 

The block is bound to the base it was created in, so a separate environment means a separate block id and a separate remote. `block add-remote` handles that if you need it.

### Dependabot will be loud 

If Dependabot is enabled for a client GitHub, alerts in an SDK project will be raised from the webpack bundler toolchain, which is build-time only. `npm audit --omit=dev` tells you how much of it actually ships.

## License

MIT
