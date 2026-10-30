# What Am I Missing? (WAM)

WAM is a standalone, offline command-line tool that compares a repository's
declared development requirements with the current machine and explains the gap.

This is the distribution repository for **WAM v0.2.0**. It contains user
documentation and links to executable releases. WAM's implementation source is
private.

WAM checks a limited, documented subset of .NET, Node.js, environment templates,
and Docker Compose evidence. It reports unsupported or uncertain checks explicitly;
a successful exit does not certify that a project will build or run.

## Installation

Select the ZIP for your platform and `SHA256SUMS` from the
[WAM v0.2.0 release](https://github.com/mkk126/what-am-i-missing-public/releases/tag/v0.2.0).
Release assets become available when the release is published.

| Platform | Archive | Executable inside |
| --- | --- | --- |
| Windows x64 | `wam-v0.2.0-windows-amd64.zip` | `wam-windows-amd64.exe` |
| Linux x64 | `wam-v0.2.0-linux-amd64.zip` | `wam-linux-amd64` |
| macOS Intel | `wam-v0.2.0-darwin-amd64.zip` | `wam-darwin-amd64` |
| macOS Apple Silicon | `wam-v0.2.0-darwin-arm64.zip` | `wam-darwin-arm64` |

WAM itself requires no Go, Node.js, .NET installation, or administrator privileges.
Inspected projects may still require those tools; finding that difference is WAM's
purpose. Each ZIP includes the executable, build metadata, user documentation,
and license/notices.

After downloading the ZIP, compare its SHA-256 digest
with its matching line in `SHA256SUMS`. On Windows:

```powershell
Get-FileHash -Algorithm SHA256 .\wam-v0.2.0-windows-amd64.zip
Expand-Archive -LiteralPath .\wam-v0.2.0-windows-amd64.zip -DestinationPath .\wam-v0.2.0
.\wam-v0.2.0\wam-windows-amd64.exe version
```

On Linux use `sha256sum ARCHIVE.zip`; on macOS use `shasum -a 256 ARCHIVE.zip`.
Extract with `unzip ARCHIVE.zip -d wam-v0.2.0`, make the extracted executable
executable if needed (`chmod +x PATH_TO_BINARY`), and run its `version` command.
Rename a copy to `wam` / `wam.exe` if desired and place it in a user-owned directory
already on PATH. WAM never edits PATH automatically.

Windows and macOS binaries are unsigned and not notarized. Follow the operating
system's normal application approval process if prompted. Checksums verify bytes
against the downloaded manifest; they are not an independent publisher signature.

## Usage

Examples assume the executable is named `wam` and is on PATH; otherwise use its
full path:

```sh
wam check
wam check "path with spaces"
wam check --format json
wam check --verbose
wam check . --exclude examples --exclude testdata
wam version
wam help
```

`check` defaults to the current directory. Select an application directory or use
existing root-relative directory exclusions when examples or fixtures are outside
the intended scan. Exclusions are literal paths, not glob patterns. They do not
assert that excluded projects satisfy their requirements. See
[scan selection](docs/scan-selection.md) for path and scope rules.

The executable has no network features, so `--offline` is intentionally not
accepted. WAM does not install dependencies, execute repository scripts, load
secret `.env` files, or modify project configuration.

For a repository declaring SDK 8.0.100 with default roll-forward on a machine
reporting SDK 8.0.105, a diagnostic can look like:

```text
[PASS] .NET SDK
       An installed SDK satisfies the supported CLI selection rules.
       Required: 8.0.100
       Actual: 8.0.105

Summary: 1 passed, 0 failed, 0 warnings, 0 skipped
Environment status: Checked requirements satisfied.
```

Console reports show available source locations, group declarations by directory,
and add supporting evidence with `--verbose`. Directory headings are display
locations, not inferred build invocation scopes.

## Statuses and exit codes

| Status | Meaning |
| --- | --- |
| PASS | An established requirement was verified as satisfied. |
| FAIL | An established requirement was verified as unsatisfied. |
| WARN | A possible issue exists without enough evidence for confirmed failure. |
| SKIP | A recognized check is unavailable, unsupported, or outside inspection scope. |

| Exit code | Meaning |
| --- | --- |
| 0 | Inspection completed with no confirmed failures; WARN/SKIP may remain. |
| 1 | Inspection completed with confirmed missing/incompatible requirements. |
| 2 | Invalid usage, fatal inspection/cancellation, or output failure. |

## Supported checks

### .NET

- `global.json`: full numeric SDK versions, all nine roll-forward policies, and
  `allowPrerelease`. Comments are supported. Duplicate keys, null settings,
  trailing commas, SDKs before 3.0, custom SDK `paths`, and unknown SDK resolver
  fields produce SKIP. Explicit prerelease requests enable prereleases as in the
  supported CLI resolver behavior.
- Recognized literal Microsoft.NET.Sdk project variants in `.csproj` establish
  SDK presence. Without an in-root `global.json`, WAM verifies presence without
  inventing a required SDK version or searching outside the selected directory.
- Literal unconditional `TargetFramework` / `TargetFrameworks` declarations in
  `.csproj` and `Directory.Build.props` provide context with SKIP. They do not
  establish one required SDK or prove installed runtime compatibility. MSBuild
  imports, expressions, conditions, property inheritance, and workloads are not
  evaluated. `.sln` and `.slnx` are discovery indicators only.

SDK selection respects supported roll-forward rules; a numerically higher SDK is
not automatically compatible. Numeric floors such as `10.0.0` are supported and
do not claim that an SDK release with that exact version exists. SDK core numeric
components are limited to signed 32-bit integers, and numeric prerelease
identifiers to unsigned 32-bit integers; larger values are unsupported.

WAM runs only `dotnet --list-sdks`, once per inspection when relevant, through a
trusted CLI on PATH. Failed, truncated, or unfamiliar observations produce SKIP.
It does not build, restore, simulate Visual Studio, or probe application runtimes.

### Node.js and npm

WAM reads `package.json` engine ranges, numeric `.nvmrc` declarations, full stable
`.node-version` declarations, and exact supported npm `packageManager` pins.
Conflicting declarations remain separate. `engines.npm` is conditional unless a
supported manager pin establishes that npm is required. A bare package.json
without supported explicit requirements produces SKIP.

The supported npm constraint subset includes stable exacts/partials, common
comparators, wildcards, caret/tilde, intersections, OR groups, stable hyphen ranges,
and exact prereleases. Complex prerelease ranges, aliases, unsupported package
managers, integrity suffixes, and unsafe numeric identifiers remain unsupported.
See the [Node support matrix](docs/node-support.md) for exact syntax and recognized
installed npm layouts. WAM never invokes Corepack or installs a manager.

### Environment and Compose

`.env.example` and `.env.sample` retain variable names, source lines, and whether
examples exist. Missing/empty process variables warn; example evidence alone
never establishes a confirmed failure. Actual `.env` files and referenced
environment files are not read.

Supported default Compose filenames provide flat host-variable references,
defaults/requiredness, container assignment/passthrough context, and a possible
Docker workflow. Defaults, example values, and error messages are not printed.
Docker gets a conditional PATH presence check; no Docker command or live service
probe runs. Full Compose merges, profiles, aliases, nested substitutions, and
include/extends semantics are outside scope. See the
[environment/Compose matrix](docs/common-support.md).

## JSON and automation

`--format json` emits report schema version `"1"` with `version`, `project`,
resolved absolute `path`, `ecosystems`, `diagnostics`, and `summary`. Explicit
exclusions add `selection.excluded_directories`; default scans omit that field.
Diagnostic metadata includes status, required/actual versions, available source
locations, evidence, and suggestions. Environment details contain only presence,
emptiness, declaration flags, and confidence, never values.

JSON stdout has no progress text or ANSI decoration. Fatal errors use stderr and
do not emit a report. Clients should ignore unknown additive fields and handle
all four statuses. IDs identify check kinds and can repeat across sources. The
[JSON Schema](docs/report.schema.json) documents the format, and
[CI usage](docs/ci-usage.md) shows exit-code and report handling.

Reports can contain private project names, absolute paths, source paths, and
version declarations even though environment values are hidden. Review/redact
them before posting a public issue. WAM does not upload reports automatically.

## Scope and safety

Discovery is limited to the selected local directory. It skips symlinks and common
generated directories including `.git`, `node_modules`, `bin`, `obj`, `dist`, and
`.vs`. Default budgets are depth 16, 20,000 entries, 1 MiB per parsed file, and
16 MiB total parsed data. Inaccessible paths or limits produce SKIP rather than
silent readiness claims.

Tool observations run from isolated temporary directories with deadlines and
output caps. Trusted installed native tools remain a trust boundary; WAM is not
an operating-system sandbox for malicious executables. Live filesystem inspection
is not an atomic snapshot. See [inspection boundaries](docs/reliability.md).

WAM complements `.NET CLI` environment commands and `npm doctor` by correlating
supported project declarations across ecosystems. Development containers and
version managers provision/select environments; WAM reports differences without
provisioning or package management. It does not claim universal build validation.

## License and support

The distributed WAM executable and accompanying WAM documentation are provided
under the [MIT license](LICENSE); redistribution is permitted under those terms.
WAM's implementation source is not published in this repository. Preserve
[third-party notices](THIRD_PARTY_NOTICES.md) and the bundled dependency/Go license
texts when redistributing executable packages.

To report a diagnostic problem, include the WAM version, operating system,
command/options, expected result, and a minimal redacted manifest example. Check
project/source paths before sharing output. Never attach `.env` files,
environment dumps, secret values, or private repository archives.
