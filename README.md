# FIT Bootstrap

Opinionated FIT environment bootstrap for GitHub Actions. Sets up Bun,
installs and caches CLI dependencies (`just`, `apm`, `gh`), restores the
`node_modules` + `generated` workspace cache, optionally syncs the wiki,
and runs `./scripts/bootstrap.sh`.

Single source of truth for the FIT CI environment. The monorepo's local
`bootstrap` action and every FIT sibling action (e.g. `kata-agent`) call
this one — version-pinned at `@v1` — so they never drift.

## Usage

```yaml
- uses: actions/checkout@v4
- uses: forwardimpact/fit-bootstrap@v1
  with:
    token: ${{ steps.ci-app.outputs.token }}   # optional, enables wiki sync
    app-slug: kata-agent-team                   # optional, sets git identity
    app-id: ${{ secrets.KATA_APP_ID }}          # optional, sets git identity
```

Cold-cache runtime is ~3 minutes; warm-cache is ~15-20 seconds.

## Prerequisites

The consumer repo must follow FIT conventions:

- `scripts/install-deps.sh` — installs `just`, `apm`, `gh` into
  `$HOME/.local`. Cache key is keyed on this file's hash.
- `scripts/bootstrap.sh` — invoked after the environment is ready.
  Receives `BOOTSTRAP_WORKSPACE_CACHE_HIT={true|false}` so it can skip
  expensive setup on a warm cache.
- `bun.lock` — workspace cache key includes its hash.
- `justfile` exposing `wiki-push` recipe (only if `token:` is provided).

## Inputs

| Input         | Required | Default    | Description                                                                              |
| ------------- | -------- | ---------- | ---------------------------------------------------------------------------------------- |
| `token`       | No       | `""`       | GitHub token with write access to the wiki. When provided, the wiki is checked out into `./wiki` and pushed back during post-run cleanup. |
| `app-slug`    | No       | `""`       | GitHub App slug for git identity (e.g. `kata-agent-team`).                              |
| `app-id`      | No       | `""`       | GitHub App ID for the git identity email.                                                |
| `bun-version` | No       | `"1.3.11"` | Bun version to install.                                                                  |

## Caching

Two cache layers:

- **Deps** — `~/.local/bin` + `~/.local/lib` (the whole local install
  prefix; the consumer's `scripts/install-deps.sh` decides what goes
  there), keyed on `hashFiles('scripts/install-deps.sh')`. Hits
  whenever the consumer hasn't bumped a pinned version or added a new
  tool.
- **Workspace** — `node_modules`, `generated`, `libraries/*/src/generated`,
  keyed on `hashFiles('bun.lock', '**/*.proto', 'libraries/libcodegen/**')`.
  Hits whenever the lockfile and codegen inputs haven't changed.

Cache misses are transparent: `scripts/install-deps.sh` re-installs
deps, `scripts/bootstrap.sh` runs `bun install` end-to-end. Generated
symlinks at `libraries/*/src/generated` are not in the workspace cache
paths (symlinks don't survive `actions/cache` extraction), so
`scripts/bootstrap.sh` always runs `just codegen` on warm cache to
recreate them.

## Wiki sync

When `token` is provided:

- The wiki is checked out into `./wiki` before `scripts/bootstrap.sh`
  runs.
- A `post-run` node20 step is registered that runs `just wiki-push`
  during job cleanup, so any wiki edits made during the job are pushed
  back automatically.

When `token` is empty, both steps are skipped — the action is safe to
use in jobs that don't need the wiki (e.g. pure CI checks).
