# pnpm-monorepo

This is a sample monorepo with the following tools:

- [pnpm][1]: package and workspace management
- [changesets][2]: manage package versions
- [turbo][3]: run tasks quickly
- [husky][4]: run tests, linting and things on git hooks
- [vite][5]: project creation

# scripts

- **build**: use turbo to build all packages
- **lint**: use turbo to lint all packages
- **test**: use turbo to test all packages
- **changeset**: use changesets to initiate a new changeset
- **changeset:init**: initialize changesets
- **changeset:version**: use changesets to initiate version bumps
- **changeset:status**: use changesets to display the status of package versions
- **changeset:publish**: use changesets to publish packages to the registry
- **huksy**: run the husky command
- **husky:init**: initialize husky
- **husky:add**: use husky to add a git hook
- **prepare**: called after pnpm install (changeset:init and husky:init)
- **turbo**: run the turbo command
- **vite:create**: create a new vite project in the packages directory

# steps to reproduce

```sh
REPO_NAME=pnpm-monorepo2
ORG_NAME=@wankdanker
REGISTRY_URL=https://npm.pkg.github.com/
mkdir -p $REPO_NAME/packages
mkdir -p $REPO_NAME/.github/workflows
cd $REPO_NAME
git init
pnpm init
pnpm install -D @changesets/cli husky is-ci turbo
pnpm pkg set scripts.build="turbo run build"
pnpm pkg set scripts.lint="turbo run lint"
pnpm pkg set scripts.test="turbo run test"
pnpm pkg set scripts.changeset="changeset"
pnpm pkg set scripts.changeset:init="is-ci || changeset init"
pnpm pkg set scripts.changeset:version="changeset version"
pnpm pkg set scripts.changeset:status="changeset status --verbose"
pnpm pkg set scripts.changeset:publish="pnpm build && changeset publish"
pnpm pkg set scripts.huksy="husky"
pnpm pkg set scripts.husky:init="is-ci || husky install"
pnpm pkg set scripts.husky:add="husky add"
pnpm pkg set scripts.prepare="pnpm husky:init && pnpm changeset:init"
pnpm pkg set scripts.turbo="turbo"
pnpm pkg set scripts.vite:create="cd packages && pnpm create vite"

pnpm install

cat << EOF > pnpm-workspace.yaml
packages:
  - "packages/*"
EOF

cat << EOF > .gitignore
node_modules
.turbo
EOF

cat << EOF > .npmrc
$ORG_NAME:registry=$REGISTRY_URL
EOF

cat << EOF > turbo.json
{
    "$schema": "https://turborepo.org/schema.json",
    "pipeline": {
        "build" : {
            "dependsOn": ["^build"],
            "outputs": [
                "dist/**"
            ] 
        },
        "deploy": {
            "cache": false,
            "dependsOn": ["build"]
        },
        "lint": {},
        "test": {
            "dependsOn": ["build"]
        }
    }
}
EOF

cat << EOF > .github/workflows/main.yml
name: CI
on:
  push:
    branches:
      - "**"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
        with:
          version: 7
      - uses: actions/setup-node@v3
        with:
          node-version: 16.x
          cache: "pnpm"

      - run: pnpm install --frozen-lockfile
EOF

cat << EOF > .github/workflows/publish.yml
name: Publish
on:
  push:
    branches:
      - "main"

concurrency: \${{ github.workflow }}-\${{ github.ref }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
        with:
          version: 7
      - uses: actions/setup-node@v3
        with:
          node-version: 16.x
          cache: "pnpm"

      - run: pnpm install --frozen-lockfile
      - name: Creating .npmrc
        run: |
          cat << EOF > "\$HOME/.npmrc"
          //npm.pkg.github.com/:_authToken=\$GITHUB_TOKEN
          @\$GITHUB_ACTOR:registry=$REGISTRY_URL
          EOF
        env:
          GITHUB_TOKEN: \${{ secrets.GITHUB_TOKEN }}
      - name: Create Release Pull Request or Publish
        id: changesets
        uses: changesets/action@v1
        with:
          publish: pnpm changeset:publish
        env:
          GITHUB_TOKEN: \${{ secrets.GITHUB_TOKEN }}
EOF

cat << EOF >> .husky/pre-commit
#!/usr/bin/env sh
. "\$(dirname -- "$0")/_/husky.sh"

pnpm test
pnpm lint
EOF
```

- [1]: https://pnpm.io/
- [2]: https://github.com/changesets/changesets
- [3]: https://turbo.build/
- [4]: https://typicode.github.io/husky/#/
- [5]: https://vitejs.dev/
