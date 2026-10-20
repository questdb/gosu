# Bump gosu to fix Go vulnerability CVEs

The questdb/gosu fork rebuilds gosu with patched Go versions when upstream
(tianon/gosu) won't release for stdlib CVEs. The bump is automated; your job is to
unstick it when it stalls.

## How the automation works

1. `check-go-update.yml` (daily, or `gh workflow run check-go-update.yml --repo questdb/gosu`)
   detects a newer Go patch on the Dockerfile's minor line and opens a PR from
   `go-bump-X.Y.Z` titled `Update Go base image to version X.Y.Z`. It then tries to
   auto-merge. That merge fails with HTTP 405 (review required) because the app's
   ruleset bypass doesn't engage, so a human has to merge the PR. This is accepted.
2. Merging that PR triggers `build-release.yml`, which publishes release `1.19-goX.Y.Z`.
3. Publishing the release triggers `propagate-release.yml`, which opens
   `bump-gosu-<tag>` PRs in questdb/questdb (`core/Dockerfile`) and
   questdb/questdb-enterprise (`Dockerfile`).

Do not bump by hand (pushing to master or opening downstream PRs yourself).
Drive the automation instead.

## Workflow

### 1. Find or create the bump PR

```bash
gh pr list --repo questdb/gosu --state open --search "Update Go base image"
```

If none exists and a newer Go patch is out, run `check-go-update.yml` via
`workflow_dispatch` and wait for it. The run will fail at the merge step with a
405; that's expected once the PR exists.

### 2. Make the PR green

```bash
gh pr checks <N> --repo questdb/gosu
```

If CI fails, read the failing job logs (`gh api --allow-escape-sequences
repos/questdb/gosu/actions/jobs/<job>/logs`) and fix the cause on the `go-bump-*` branch:

- **govulncheck finds a non-stdlib vuln** (for example in `github.com/moby/sys/user`):
  a newer Go won't fix it. Bump the module to the fixed version from
  `https://vuln.go.dev/ID/<GO-ID>.json`, then rebuild and check locally:
  ```bash
  docker run --rm --mount "type=bind,src=$PWD,dst=/wd" --workdir /wd \
    --user "$(id -u):$(id -g)" --env HOME=/tmp golang:X.Y.Z-trixie bash -ec \
    'go get <module>@<fixed> && go mod tidy && CGO_ENABLED=0 go build -trimpath -o gosu-amd64'
  ./govulncheck-with-excludes.sh -mode=binary gosu-amd64 && ./test.sh gosu-amd64
  rm gosu-amd64
  ```
  The `Test (latest go)` CI job upgrades all deps. If it passes, the dep bump is
  known-safe.
- **Only exclude a vuln** in `govulncheck-with-excludes.sh` if gosu can't reach it, with a
  comment explaining why.
- **Runner/infra breakage** (for example apt package renames): fix it in both `ci.yml` and
  `build-release.yml`. The release build uses the same setup.

Push the fixes to the `go-bump-*` branch and update the PR body to mention them.

### 3. Hand off the merge

Tell the user the PR is green and ask them to merge it (or merge with
`gh pr merge <N> --repo questdb/gosu --squash --admin` if they say to).

### 4. Watch the release and propagation

```bash
gh run list --repo questdb/gosu --workflow=build-release.yml --limit 1
gh run watch <id> --repo questdb/gosu --exit-status
gh release view 1.19-goX.Y.Z --repo questdb/gosu
gh run list --repo questdb/gosu --workflow=propagate-release.yml --limit 1
gh run watch <id> --repo questdb/gosu --exit-status
```

If the release exists but propagation didn't run, re-run it with
`gh workflow run propagate-release.yml --repo questdb/gosu -f tag=1.19-goX.Y.Z`.
If build-release failed, fix the cause and re-run it with
`gh workflow run build-release.yml --repo questdb/gosu -f tag=1.19-goX.Y.Z`.

### 5. Report

- The new gosu tag and anything fixed beyond the Go bump (dep bumps, CI fixes)
- Links to the downstream PRs:
  ```bash
  for r in questdb questdb-enterprise; do
    gh pr list --repo questdb/$r --head bump-gosu-1.19-goX.Y.Z --json url --jq '.[].url'
  done
  ```

## Notes

- The gosu version (1.19) doesn't change, only the `-goX.Y.Z` suffix.
- The Go minor line (for example 1.27) only changes when someone edits the Dockerfile by hand.
  `check-go-update` only bumps patch versions.
