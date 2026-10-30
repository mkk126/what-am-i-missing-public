# Inspection boundaries and limitations

WAM compares supported explicit requirements with current observations. A PASS
confirms only the checked requirement; it does not certify that an application
builds, starts, or behaves correctly. WARN/SKIP results can leave readiness
inconclusive even when the exit code is 0.

## Read-only and offline scope

- WAM has no network mode and never uploads repository contents or environment
  information. It does not install dependencies, modify PATH, request elevated
  privileges, or execute repository scripts, package hooks, build, or restore.
- Actual `.env` files and Compose-referenced secret/configuration files are not
  loaded. Environment checks retain only presence/emptiness and declaration
  metadata, never values. Reports can still expose project names and file paths;
  review them before sharing.
- Version commands use trusted installed tools outside the repository and run
  from isolated temporary directories. Inherited preloads, tracing/output hooks,
  and applicable configuration overrides are removed. npm configuration/cache/log
  locations and the .NET CLI home are isolated.
- No Docker command or live service probe runs. A service mentioned by Compose
  does not prove that a matching host service is required.

Trusted installed native executables remain trusted. Allowlisted arguments,
temporary directories, and environment filtering do not make WAM an
operating-system sandbox for malicious host tools. Unsafe or unrecognized tool
resolution produces SKIP.

## Discovery and execution budgets

The selected root is a supported local directory. Discovery does not follow
symlinks or search ancestor directories outside the selected root. Common
dependency/build directories are ignored. Default limits are:

| Limit | Budget |
| --- | --- |
| Directory depth | 16 |
| Filesystem entries | 20,000 |
| Parsed manifest size | 1 MiB per file |
| Combined parsed manifest data | 16 MiB |
| Inspection context | 30 seconds |
| Each external observation | 5 seconds |
| Observation stdout / stderr | 256 KiB / 16 KiB |
| Pipe-draining wait after process exit/cancellation | 250 ms |

Parsing has additional node/line/reference limits documented in the support
guides. Reaching a limit or inaccessible path produces visible SKIP evidence.
Entry-budget exhaustion discards partial discoveries rather than drawing
conclusions from an operating-system-dependent subset. Output overflow cancels
the child immediately and discards the partial observation.

Context deadlines are checked between bounded operations and constrain external
processes; ordinary filesystem/OS calls are not forcibly interruptible. Network
filesystems or concurrently changing directories are outside the intended stable
local-inspection use case.

## Live filesystem and requirement uncertainty

Inspection reads a live filesystem, not an atomic snapshot. File identity checks
reject replacements detected at open, but concurrent in-place edits can still
change observations. Keep project files stable while checking. Explicit
[scan selections](scan-selection.md) describe only the included scope.

Each discovered .NET SDK declaration is evaluated independently. WAM does not
simulate a solution invocation, Visual Studio SDK selection, general MSBuild
evaluation, workloads, custom SDK paths, or runtime readiness from framework
names. A framework alone does not establish one SDK version. Malformed or
unreadable in-root global.json declarations remain scope barriers instead of
silently producing unpinned success.

Node ranges and installed npm layouts use the narrow
[Node support matrix](node-support.md). Version-manager aliases, unsupported
managers, and complex prerelease expressions remain SKIP. No tool is downloaded
to resolve an alias or missing version.

Environment templates and Compose provide conditional evidence. An absent
variable may be supplied by another launch environment or an unread `.env` file,
so even explicit Compose interpolation concerns remain WARN rather than confirmed
failures. See the [environment/Compose matrix](common-support.md).

These boundaries preserve uncertainty rather than guessing. The
[README](../README.md) documents supported checks and exit codes, and the
[JSON Schema](report.schema.json) describes the public report format.
