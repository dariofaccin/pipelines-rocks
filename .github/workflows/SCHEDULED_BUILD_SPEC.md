# Spec: Scheduled "build all rocks" workflow

## Goal

Once per month, verify that **every rock in this repository still builds,
scans, and tests** successfully on `main`, independently of whether a PR
touched it. Today the equivalent logic only runs on changed rocks via
[on_pull_request.yaml](on_pull_request.yaml) and [on_push.yaml](on_push.yaml).

## Decisions

| Question            | Decision |
|---------------------|----------|
| Cadence             | Monthly (cron `0 4 1 * *` — 1st of month, 04:00 UTC) + `workflow_dispatch`. |
| Publish or dry-run  | **Dry-run only.** No extra input needed: the reusable workflow's `publish` job is gated on `github.event_name == 'push'`, so `schedule` / `workflow_dispatch` skip publish (and the dependent `integrate-on-merge`) automatically. |
| Failure reporting   | **One issue per failing rock.** On each matrix failure, open (or comment on existing) a GitHub issue titled per rock with branch + run link. |
| Branch coverage     | `main` only. |
| Reusable workflow pin | Keep `@main` (consistent with existing workflows in this repo). |

## Current CI recap

- [on_pull_request.yaml](on_pull_request.yaml) calls
  `canonical/charmed-kubeflow-workflows/.github/workflows/get-rocks-modified-and-build-scan-test-publish.yaml@main`.
- That reusable workflow runs `get-rocks-modified.yaml` (git-diff vs
  `base-ref`) and fans out a matrix calling `build-scan-test-publish-rock.yaml`
  per changed rock.
- On schedule there are no "modified" paths versus `main`, so reusing it
  as-is would be a no-op.
- The per-rock reusable workflow `build-scan-test-publish-rock.yaml`:
  - Always runs `build`, `scan`, `sanity-tests`, `integration-tests`.
  - `publish` runs only on `push` events.
  - `integrate-on-pr` runs only on PRs.
  - `integrate-on-merge` depends on `publish`, so it's skipped here too.

## Design

A new workflow `on_schedule.yaml` that:

1. Triggers on `schedule` (monthly) and `workflow_dispatch`.
2. Discovers all rock directories by finding `rockcraft.yaml` files
   (depth 2 — matches this repo's `<rock>/rockcraft.yaml` layout).
3. Fans out a matrix calling
   `canonical/charmed-kubeflow-workflows/.github/workflows/build-scan-test-publish-rock.yaml@main`
   once per rock, with `fail-fast: false`.
4. A follow-up `report-failures` job, **matrixed over the same rock list with
   `if: always()`**, inspects the conclusion of the corresponding
   `build-scan-test` matrix leg via `gh run view --json jobs` and, if it
   failed, opens (or comments on existing) an issue **dedicated to that
   rock**.

### Why per-rock issues need a matrixed reporter

A reusable workflow (`uses:`) job cannot have inline steps, and a single
post-job sees only the aggregate `needs.build-scan-test.result`. To attribute
a failure to a specific rock we must look up per-leg conclusions from the
current run via the GitHub API (`gh run view <run_id> --json jobs`) and
filter by job name (which embeds `matrix.rock-dir`).

### Why not reuse `get-rocks-modified-and-build-scan-test-publish.yaml`?

It is hard-wired to git-diff discovery. Discovering rocks via `rockcraft.yaml`
presence is explicit, requires no upstream changes, and matches the directory
convention already used in this repo.

## Workflow sketch

```yaml
name: Scheduled build all rocks

on:
  schedule:
    - cron: '0 4 1 * *'   # 1st of each month, 04:00 UTC
  workflow_dispatch:

permissions:
  contents: read
  issues: write           # to open/comment failure issues

jobs:
  list-rocks:
    name: Discover rock directories
    runs-on: ubuntu-latest
    outputs:
      paths: ${{ steps.find.outputs.paths }}
    steps:
      - uses: actions/checkout@v4
      - id: find
        run: |
          paths=$(find . -mindepth 2 -maxdepth 2 -name rockcraft.yaml \
                    -printf '%h\n' | sed 's|^\./||' | sort | jq -R . | jq -sc .)
          echo "paths=$paths" >> "$GITHUB_OUTPUT"

  build-scan-test:
    name: CI (${{ matrix.rock-dir }})
    needs: list-rocks
    if: ${{ needs.list-rocks.outputs.paths != '[]' }}
    strategy:
      fail-fast: false
      matrix:
        rock-dir: ${{ fromJson(needs.list-rocks.outputs.paths) }}
    uses: canonical/charmed-kubeflow-workflows/.github/workflows/build-scan-test-publish-rock.yaml@main
    secrets: inherit
    with:
      rock-dir: ${{ matrix.rock-dir }}
      # publish is auto-skipped: gated on github.event_name == 'push'

  report-failures:
    needs: [list-rocks, build-scan-test]
    if: ${{ always() && needs.list-rocks.outputs.paths != '[]' }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        rock-dir: ${{ fromJson(needs.list-rocks.outputs.paths) }}
    steps:
      - name: Check this rock's build result
        id: check
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ROCK_DIR: ${{ matrix.rock-dir }}
        run: |
          # Per-leg job name from `name:` above is "CI (<rock-dir>) / ..."
          # Look up the conclusion of any job whose name starts with our matrix leg.
          conclusion=$(gh run view "${{ github.run_id }}" \
            --repo "${{ github.repository }}" \
            --json jobs \
            --jq ".jobs[] | select(.name | startswith(\"CI (${ROCK_DIR})\")) | .conclusion" \
            | sort -u | paste -sd, -)
          echo "conclusion=$conclusion" >> "$GITHUB_OUTPUT"
          if echo "$conclusion" | grep -qw failure; then
            echo "failed=true" >> "$GITHUB_OUTPUT"
          else
            echo "failed=false" >> "$GITHUB_OUTPUT"
          fi

      - name: Open or update failure issue for this rock
        if: steps.check.outputs.failed == 'true'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          BRANCH: ${{ github.ref_name }}
          ROCK_DIR: ${{ matrix.rock-dir }}
          RUN_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
        run: |
          title="Scheduled build failed: ${ROCK_DIR} on ${BRANCH}"
          body="Scheduled build of \`${ROCK_DIR}\` failed on branch \`${BRANCH}\`.\n\nRun: ${RUN_URL}"
          existing=$(gh issue list --repo "${{ github.repository }}" \
            --state open --search "in:title \"${title}\"" --json number \
            --jq '.[0].number')
          if [ -n "$existing" ]; then
            gh issue comment "$existing" --repo "${{ github.repository }}" --body "$body"
          else
            gh issue create --repo "${{ github.repository }}" \
              --title "$title" --body "$body"
          fi
```

## Notes / caveats

- **Job-name matching**: `report-failures` filters jobs via
  `startswith("CI (<rock-dir>)")`, which depends on the `name:` of the
  `build-scan-test` matrix job (set to `CI (${{ matrix.rock-dir }})` above).
  If that name is changed, update the `jq` filter accordingly.
- **Token**: default `GITHUB_TOKEN` with `issues: write` is sufficient for
  `gh issue` operations.

## Out of scope

- Changing the per-rock build logic (lives in
  `canonical/charmed-kubeflow-workflows`).
- Modifying the existing PR/Push workflows.
- Running against `track/*` branches (deferred).
