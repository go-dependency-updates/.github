While [Dependabot](https://github.com/dependabot) will do a targetted update of direct dependencies only, this app will update all
dependencies (both direct and indirect) and trim them with `go mod tidy` on a schedule. If there are changes, it will generate a pull request with the changes - see an [example](https://github.com/map-services/company-data-api/pull/99).

## Configuration

First create a new private key from https://github.com/organizations/go-dependency-updates/settings/apps/go-dependency-updates. Then, in your Golang repository, go to **Settings > Secrets and variables > Actions** and
* add `APP_ID` of **3654108** as a repository variable
* add `APP_PRIVATE_KEY` being the content of that private key 

Add a file to your repo as `.github/workflows/go_dependencies.yml`:

```yaml
name: Update Go dependencies
on:
  schedule:
    - cron: "0 0 * * 1" # Every Monday at 00:00 UTC
  workflow_dispatch:

jobs:
  update-deps:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write

    steps:
      - name: Generate token
        id: generate_token
        uses: actions/create-github-app-token@v3
        with:
          client-id: ${{ vars.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - uses: actions/checkout@v6
        with:
          token: ${{ steps.generate_token.outputs.token }}

      - uses: actions/setup-go@v6
        with:
          go-version: stable
          cache: true

      - name: Cache Go modules
        uses: actions/cache@v5
        with:
          path: ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.mod') }}
          restore-keys: |
            ${{ runner.os }}-go-

      - name: Update dependencies
        id: update
        run: |
          set -eo pipefail
          go env GOPROXY
          go get -u ./... 2>&1 | tee /tmp/go-get-output.txt
          go mod tidy

          # Build a summary of what changed in go.mod
          CHANGES=$(git diff go.mod | grep '^[+-]' | grep -v '^[+-][+-][+-]' | grep -v '^[+-]module' | grep -v '^[+-]go ' || true)
          echo "CHANGES<<EOF" >> $GITHUB_OUTPUT
          echo "$CHANGES" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v8
        with:
          token: ${{ steps.generate_token.outputs.token }}
          branch: chore/update-go-deps
          commit-message: "chore: go get -u && go mod tidy"
          title: "chore: update Go dependencies"
          body: |
            ## Dependency Updates

            Automated update via `go get -u ./... && go mod tidy`.

            ### Changes to go.mod
            ```diff
            ${{ steps.update.outputs.CHANGES }}
            ```
          labels: dependencies, go
```
