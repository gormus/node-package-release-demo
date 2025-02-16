# node-package-release-demo

A demo for creating automated releases for a node package.

The package will be created with a GitHub user or organization scope, and will be published under the same scope privately.

## Create a scoped package

First of all, make sure the package name is scoped. If you are creating a new project, use the `--scope` argument. If it is an existing project, update the name value in `package.json`, and prefix it with the scope; `"name": "@foo/whatever"`.

```sh
# accept all defaults, and create a package named "@foo/whatever",
# instead of just named "whatever"
npm init --scope=@foo --yes
```

## Personel access token

Create a [personel access token](https://github.com/settings/tokens/new?scopes=repo,workflow,write:packages,delete:packages&description=RELEASE_PLEASE_TOKEN); it will be used in the GitHub Action for creating new releases, and publising the packages.

## Configure NPM settings

Create a `.npmrc` file in the project root with following content. Change the scope name, but leave the `_authToken` value as is. The value will be read from the environment variables, the GitHub Action will set.

```ini
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
@foo:registry=https://npm.pkg.github.com
```

## Create a GitHub Action

Checkout the official documentation from [Release Please Action](https://github.com/googleapis/release-please-action), and update your version of the action file as needed.

Following is a drop-in working example.

```yaml
# .github/workflows/release-please.yml
on:
  push:
    branches:
      - main
      - staging
      - test/*

permissions:
  contents: write
  pull-requests: write

name: release-please

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - name: Create a release
        uses: googleapis/release-please-action@v4
        id: release
        with:
          # this assumes that you have created a personal access token
          # (PAT) and configured it as a GitHub action secret named
          # `RELEASE_PLEASE_TOKEN` (this secret name is not important).
          token: ${{ secrets.RELEASE_PLEASE_TOKEN }}
          # this is a built-in strategy in release-please, see "Action Inputs"
          # for more options
          release-type: node
          target-branch: ${{ github.ref_name }}

      - name: Checkout repository
        uses: actions/checkout@v4
        if: ${{ steps.release.outputs.release_created }}
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Setup Node.js
        uses: actions/setup-node@v4
        if: ${{ steps.release.outputs.release_created }}
        with:
          node-version: "lts/*"
          registry-url: https://npm.pkg.github.com/
          scope: "@${{ github.repository_owner }}"

      - name: Install dependencies
        if: ${{ steps.release.outputs.release_created }}
        run: npm clean-install

      - name: Verify the integrity of provenance attestations and registry signatures for installed dependencies
        if: ${{ steps.release.outputs.release_created }}
        run: npm audit signatures

      - name: Publish a package
        if: ${{ steps.release.outputs.release_created }}
        env:
          NODE_AUTH_TOKEN: ${{ secrets.RELEASE_PLEASE_TOKEN }}
        run: npm publish --access=restricted
```
