# Using WAM in CI

The initial public binary release is being prepared; public executable downloads
are not available yet. Once published, obtain the standalone executable for the
runner from [WAM releases](https://github.com/mkk126/what-am-i-missing-public/releases).
Pin a reviewed version and verify the downloaded ZIP against its SHA256SUMS entry
before execution. Keep installation and credential handling in trusted pipeline
configuration; never take a tool-download command from an inspected repository.

`wam check` is offline. It neither installs project dependencies nor runs
repository scripts. It observes the runner's current tools and process-variable
presence, which can differ from a developer machine or a container used later.

```sh
wam check . --format json > wam-report.json
```

| Exit code | Meaning | Suggested handling |
| --- | --- | --- |
| 0 | Completed with no confirmed failures; WARN/SKIP may remain. | Review summary and unsupported checks according to project policy. |
| 1 | Completed with confirmed missing/incompatible requirements. | Fail the environment-check step and inspect the report. |
| 2 | Invalid usage, fatal inspection/cancellation, or output failure. | Treat the report as unavailable and inspect stderr. |

Do not treat an existing redirected file as proof of a valid report: the shell
may create an empty file before a fatal error. Capture the exit code before
running another command. Preserve the ordinary failure policy for code 1/2 when
uploading report artifacts.

For a Linux/macOS shell step that processes the report later:

```sh
wam_status=0
wam check . --format json > wam-report.json || wam_status=$?
# Optional report processing goes here; preserve wam_status.
exit "$wam_status"
```

For PowerShell:

```powershell
wam check . --format json > wam-report.json
$wamStatus = $LASTEXITCODE
# Optional report processing goes here; preserve $wamStatus.
exit $wamStatus
```

Prefer selecting the intended application directory in a monorepo. Add explicit
root-relative `--exclude` directory selections when appropriate and inspect
`selection.excluded_directories`. Exclusions are scan boundaries, not assertions
that excluded projects satisfy their requirements. See
[scan selection](scan-selection.md).

Schema version 1 has `diagnostics`, `summary`, and available source/evidence
metadata. Ignore unknown additive fields and distinguish pass/fail/warn/skip.
IDs identify check kinds and can repeat across declarations; use source paths and
line numbers to distinguish them. The [JSON Schema](report.schema.json) documents
the report contract.

JSON omits environment values and decorative messages, but can contain private
project names, absolute paths, source paths, and version declarations. Review
artifact access and redact output before sharing it publicly. WAM does not upload
reports automatically.
