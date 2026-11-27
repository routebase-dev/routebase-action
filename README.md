# Routebase for GitHub Actions

Run [Routebase](https://routebase.dev) API test suites and security scans in CI, push a spec generated
from code, or promote a spec version to an environment after a deploy.

Routebase derives your documentation portal, mock servers, contract tests and monitoring from one
living OpenAPI description. This action runs the parts of that which belong in a pipeline.

> Requires a Routebase account and an API key. Create one under **Settings → API Keys**.

## Run a test suite on every pull request

The suite runs against your environment, and the job fails when a test fails. `--format junit`
gives you a report GitHub renders as a real test summary.

```yaml
- uses: routebase-dev/routebase-action@v3
  with:
    command: run
    api-key: ${{ secrets.ROUTEBASE_API_KEY }}
    project: 8f14e45f-ceea-467a-9f6e-2c9a1b3d4e5f
    suite: 3c59dc04-8e88-4d5a-9e0e-1b2c3d4e5f60
    environment: staging
    format: junit
    output: reports/routebase.xml

- uses: dorny/test-reporter@v3
  if: always()
  with:
    name: API contract tests
    path: reports/routebase.xml
    reporter: java-junit
```

## Security scan into the Security tab

`--format sarif` writes a SARIF file, so findings appear as code scanning alerts on the pull
request instead of in a log nobody opens. `fail-on` gates the job.

```yaml
permissions:
  security-events: write

steps:
  - uses: routebase-dev/routebase-action@v3
    with:
      command: scan
      api-key: ${{ secrets.ROUTEBASE_API_KEY }}
      project: 8f14e45f-ceea-467a-9f6e-2c9a1b3d4e5f
      profile: 9b74c989-1f37-4a2b-8c5d-6e7f8a9b0c1d
      fail-on: high
      format: sarif
      output: routebase.sarif

  - uses: github/codeql-action/upload-sarif@v4
    if: always()
    with:
      sarif_file: routebase.sarif
```

## Push a spec generated from code

Keeps one living specification in Routebase when your OpenAPI document comes out of the code. The
push updates the spec's draft in place, so endpoint links of tests, mock rules and monitors stay.
Needs a key with the scopes `specs:write` and `specs:delete` (`specs:read` for `dry-run`), and
`@v3` of this action (CLI `3.0.0`).

A push with a breaking change against the draft, such as a removed endpoint, writes nothing. It
stops with exit code `6`, names the breaking changes and links to the page in Routebase where you
decide. There you accept the change into the draft, or you change the code so it matches the
contract again. A dry run answers the same way, so the pull request job is the check: it fails with
exit code `6` before the merge, as the push on the default branch would after it.

```yaml
# Pull request: the check. Fails with exit code 6 when the push would be refused.
- uses: routebase-dev/routebase-action@v3
  with:
    command: spec-push
    api-key: ${{ secrets.ROUTEBASE_API_KEY }}
    project: ${{ vars.ROUTEBASE_PROJECT_ID }}
    spec: Orders API
    file: openapi.json
    dry-run: true
```

On the default branch the push writes the draft. To take breaking changes on purpose without the
page in Routebase, accept them for a single run through an input of a manual run, never as a fixed
value of the job:

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      accept-breaking:
        description: Take breaking changes into the draft in this run
        type: boolean
        default: false

jobs:
  spec:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      # ... build the OpenAPI document ...
      - uses: routebase-dev/routebase-action@v3
        with:
          command: spec-push
          api-key: ${{ secrets.ROUTEBASE_API_KEY }}
          project: ${{ vars.ROUTEBASE_PROJECT_ID }}
          spec: Orders API
          file: openapi.json
          accept-breaking: ${{ inputs.accept-breaking || false }}
```

`inputs` is empty on a `push`, so the expression falls back to `false` there. A spec whose
**Breaking changes from code** setting is **Ask, decide in Routebase only** cannot be accepted from
a pipeline, and the run stays at exit code `6` until someone decides on the linked page.

When someone edited a value in Routebase that the file owns since the last push, the push also stops
with exit code `6` and lists the conflicts. Set `overwrite: true` to let the file win. The first push
into a spec that has content but was never pushed needs `overwrite: true` once.

When the spec keeps several major versions side by side, such as a 3.x line that still gets fixes
next to 4.x, each line has its own draft. Set `line: 3` in the job that builds the 3.x code, and the
push goes into the draft of line 3. When the line has no draft, the push first creates the next
patch of the line's latest release as a copy.

Each run writes a summary to the job page: the counts, the breaking changes with their reasons, the
conflicts and the link to the decision.

## Push docs and preview pull requests

Pushes a folder of Markdown pages, as written by `routebase docs pull`, into a Draft or Review
version of your documentation. With `preview: true` it also builds a portal preview of that version
and, on a pull request, posts the link as a comment. Later runs update the same comment, so the pull
request always shows one current link. Needs a key with `docs:write`, plus `docs:manage-portal` for
`preview`, and `@v2.3.0` or later of this action (CLI `2.4.0`).

```yaml
# Pull request job: push into the Draft kept for previews and comment the link
permissions:
  contents: read
  pull-requests: write
steps:
  - uses: actions/checkout@v6
  - uses: routebase-dev/routebase-action@v3
    with:
      command: docs-push
      api-key: ${{ secrets.ROUTEBASE_API_KEY }}
      project: ${{ vars.ROUTEBASE_PROJECT_ID }}
      dir: docs
      version-id: ${{ vars.ROUTEBASE_PREVIEW_VERSION_ID }}
      preview: true
```

```yaml
# Job on main: push into the version your main branch owns
- uses: routebase-dev/routebase-action@v3
  with:
    command: docs-push
    api-key: ${{ secrets.ROUTEBASE_API_KEY }}
    project: ${{ vars.ROUTEBASE_PROJECT_ID }}
    dir: docs
    version-id: ${{ vars.ROUTEBASE_DOCS_VERSION_ID }}
```

A preview shows the version the job pushed into, so a pull request job writes its pages into that
version. Keep a Draft for previews and pass its id as `version-id`, so pull requests never write into
the version your `main` branch pushes into. The link stops working after the lifetime set under
**Public Portal → Advanced → Preview links**, which is 24 hours unless you change it.

The comment needs `pull-requests: write` in the job. Without it the push and the preview still
succeed, the action warns, and the link is in the `preview-url` output. Pull requests from forks get
no repository secrets, so they get no preview either.

## Promote a version after a deploy

Records which version an environment actually serves, so drift is measured against the right
contract.

```yaml
- uses: routebase-dev/routebase-action@v3
  with:
    command: promote
    api-key: ${{ secrets.ROUTEBASE_API_KEY }}
    spec: Orders API
    version: 1.4.0
    environment: production
```

## Inputs

| Input | Required for | Description |
| --- | --- | --- |
| `command` | always | `run`, `scan`, `spec-push`, `docs-push` or `promote` |
| `api-key` | always | Pass from a repository secret, never inline |
| `region` | — | `us` if your organization is hosted in the US. See below |
| `project` | `run`, `scan`, `docs-push` | Project id. Optional for `promote` and `spec-push` |
| `suite` | `run` | Test suite id |
| `profile` | `scan` | Scan profile id |
| `fail-on` | — | Fail the scan at this severity or higher, e.g. `high` |
| `spec` | `promote`, `spec-push` | API specification, name or id |
| `file` | `spec-push` | OpenAPI document to push, YAML or JSON |
| `dry-run` | — | `spec-push` and `docs-push`: show the changes without writing |
| `overwrite` | — | `spec-push`: let the file win over edits made in Routebase |
| `folders-from-tags` | — | `spec-push`: put new endpoints into the folder of their first tag |
| `accept-breaking` | — | `spec-push`: take the breaking changes into the draft in this run. Pass it per run |
| `line` | — | `spec-push`: major version line to push into, such as `4`, when the spec keeps several lines |
| `version` | `promote` | Version to promote, e.g. `1.4.0` |
| `environment` | `promote` | Target environment. Optional for `run` |
| `format` | — | `run`: `text`, `json`, `junit`. `scan`: `text`, `json`, `sarif`. `spec-push`: `text`, `json` |
| `dir` | — | `docs-push`: folder with the Markdown pages, default `docs` |
| `doc` | — | `docs-push`: documentation id, optional when the project has exactly one |
| `version-id` | — | `docs-push`: Draft or Review version to push into, default the newest one |
| `preview` | — | `docs-push`: build a portal preview and set `preview-url` |
| `pr-comment` | — | `docs-push` with `preview` on a pull request: post and update the link as a comment, default `true` |
| `github-token` | — | Token for the comment, default the job token |
| `output` | — | Write the report to this path instead of stdout. Not used by `spec-push` and `docs-push` |
| `cli-version` | — | Version of the `Routebase.Cli` package. See below |

Inputs that do not apply to the chosen `command` are ignored.

## Outputs

| Output | Description |
| --- | --- |
| `exit-code` | The CLI's exit code, see the table below |
| `report-path` | The path given in `output`, empty when unset |
| `preview-url` | `docs-push` with `preview`: the preview link, empty otherwise |
| `preview-expires` | `docs-push` with `preview`: when the link stops working, UTC |

## Region

One host serves both the EU and the US region, and a request without a region signal is served
from the **EU**. If your organization is hosted in the US, set `region: us` — otherwise your key
is looked up in the wrong region and the run fails at authentication with exit code 4. EU
organizations need nothing.

## Exit codes

| Code | Meaning |
| --- | --- |
| `0` | Success |
| `1` | The run failed — tests failed, or a finding tripped `fail-on` |
| `2` | Configuration or input error (missing input, bad flag, a file that is not valid OpenAPI) |
| `3` | Network error, or the run did not complete |
| `4` | Authentication error — the key is wrong, in the wrong region, or lacks the permission the command needs |
| `5` | Project, suite, profile, spec or version not found |
| `6` | Refused by the current state in Routebase: a `spec-push` with breaking changes that need a decision (also in a dry run), values edited in Routebase since the last push, a docs page changed remotely, or a spec without a single draft |
| `7` | `docs-push` with `preview`: the pages were pushed, but the preview failed or was not ready within 10 minutes |

The meanings have been stable since CLI 1.0.0, with two changes. In **2.0.0** a key that
authenticates but lacks the required permission (HTTP `403`) exits `4` for every command.
`run` and `list` returned `3` for that case before, which sent you looking at the network rather
than at the key's scopes. If your pipeline branches on `3` for either command, check it before
moving `cli-version` to `2.0.0`. In **3.0.0** `spec-push` exits `6` when the file has breaking
changes against the draft, in a dry run too, unless the run sets `accept-breaking`. With CLI 2.x
such a push went through unless the spec's own setting refused it.

## Versioning

`@v3` tracks the latest v3 release and is what you want. `@v2` stays on CLI 2.4.0, where
`spec-push` writes breaking changes unless the spec's own setting refuses them. `@v1` stays on CLI
1.0.1, where a key with missing scopes exits `3` instead of `4`, see the note under Exit codes. Pin
a full version like `@v3.0.0` if you want a fixed one.

The action installs a **pinned** version of the `Routebase.Cli` NuGet package rather than the
latest. An action that silently moved to a new CLI would change what your pipeline does without
anyone deciding it. Override with `cli-version` when you want a different one.

That the action's major happens to match the CLI's is a coincidence, not a promise: the two ship
on their own cycles and will diverge at the first change that touches only one of them.

## Credentials on the runner

The CLI reads its API key from `~/.routebase/config.json`, so the action writes the key there and
**removes the file again in a step that runs even when the command fails**. On GitHub-hosted
runners the machine is discarded anyway; on self-hosted runners this is what keeps the key from
outliving the job.

## Links

- [Routebase](https://routebase.dev) · [Documentation](https://docs.routebase.dev) · [Changelog](https://routebase.dev/changelog/)
- [`Routebase.Cli` on NuGet](https://www.nuget.org/packages/Routebase.Cli)

## Licence

The action is MIT. The Routebase CLI it installs is proprietary and requires a Routebase
subscription or trial — see <https://routebase.dev/terms>.
