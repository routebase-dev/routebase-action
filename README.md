# Routebase for GitHub Actions

Run [Routebase](https://routebase.dev) API test suites and security scans in CI, or promote a spec
version to an environment after a deploy.

Routebase derives your documentation portal, mock servers, contract tests and monitoring from one
living OpenAPI description. This action runs the parts of that which belong in a pipeline.

> Requires a Routebase account and an API key. Create one under **Settings → API Keys**.

## Run a test suite on every pull request

The suite runs against your environment, and the job fails when a test fails. `--format junit`
gives you a report GitHub renders as a real test summary.

```yaml
- uses: routebase-dev/routebase-action@v1
  with:
    command: run
    api-key: ${{ secrets.ROUTEBASE_API_KEY }}
    project: 8f14e45f-ceea-467a-9f6e-2c9a1b3d4e5f
    suite: 3c59dc04-8e88-4d5a-9e0e-1b2c3d4e5f60
    environment: staging
    format: junit
    output: reports/routebase.xml

- uses: dorny/test-reporter@v1
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
  - uses: routebase-dev/routebase-action@v1
    with:
      command: scan
      api-key: ${{ secrets.ROUTEBASE_API_KEY }}
      project: 8f14e45f-ceea-467a-9f6e-2c9a1b3d4e5f
      profile: 9b74c989-1f37-4a2b-8c5d-6e7f8a9b0c1d
      fail-on: high
      format: sarif
      output: routebase.sarif

  - uses: github/codeql-action/upload-sarif@v3
    if: always()
    with:
      sarif_file: routebase.sarif
```

## Promote a version after a deploy

Records which version an environment actually serves, so drift is measured against the right
contract.

```yaml
- uses: routebase-dev/routebase-action@v1
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
| `command` | always | `run`, `scan` or `promote` |
| `api-key` | always | Pass from a repository secret, never inline |
| `region` | — | `us` if your organization is hosted in the US. See below |
| `project` | `run`, `scan` | Project id. Optional for `promote` |
| `suite` | `run` | Test suite id |
| `profile` | `scan` | Scan profile id |
| `fail-on` | — | Fail the scan at this severity or higher, e.g. `high` |
| `spec` | `promote` | API specification, name or id |
| `version` | `promote` | Version to promote, e.g. `1.4.0` |
| `environment` | `promote` | Target environment. Optional for `run` |
| `format` | — | `run`: `text`, `json`, `junit`. `scan`: `text`, `json`, `sarif` |
| `output` | — | Write the report to this path instead of stdout |
| `cli-version` | — | Version of the `Routebase.Cli` package. See below |

Inputs that do not apply to the chosen `command` are ignored.

## Outputs

| Output | Description |
| --- | --- |
| `exit-code` | The CLI's exit code, see the table below |
| `report-path` | The path given in `output`, empty when unset |

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
| `2` | Configuration error (missing input, bad flag) |
| `3` | Network error, or the run did not complete |
| `4` | Authentication error — the key is wrong, in the wrong region, or lacks the permission the command needs |
| `5` | Project, suite or profile not found |
| `6` | Conflict |

The meanings have been stable since CLI 1.0.0, with one change in **2.0.0**: a key that
authenticates but lacks the required permission (HTTP `403`) now exits `4` for every command.
`run` and `list` returned `3` for that case before, which sent you looking at the network rather
than at the key's scopes. If your pipeline branches on `3` for either command, check it before
moving `cli-version` to `2.0.0`.

## Versioning

`@v2` tracks the latest v2 release and is what you want. `@v1` stays on CLI 1.0.1, where a key
with missing scopes exits `3` instead of `4` — see the note under Exit codes. Pin a full version
like `@v2.0.0` if you want a fixed one.

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
