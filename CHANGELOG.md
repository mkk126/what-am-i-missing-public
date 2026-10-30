# Changelog

## v0.2.0

Initial binary distribution of WAM. The implementation source remains private;
binaries and accompanying WAM documentation use the MIT license.

- Compare supported .NET SDK, Node.js/npm, environment-template, and Compose
  declarations with the current local machine using offline, bounded checks.
- Respect the documented .NET SDK roll-forward subset and npm version-range
  subset, keeping uncertain or unsupported checks visible as WARN/SKIP.
- Provide console and JSON reports, declaration locations, directory grouping,
  explicit directory exclusions, and documented automation exit codes.
- Recognize UTF-8 BOM prefixes in supported .NET XML and environment templates,
  and support numeric SDK floors with applicable roll-forward behavior.
- Prevent inherited .NET tracing/hooks and npm cache/log settings from creating
  inspection side effects; stop tool observations promptly on output overflow.
- Treat numeric Node prerelease identifiers beyond the supported precision
  boundary as unsupported rather than reporting unreliable compatibility.
- Reduce console-report allocations and avoid repeated SDK-manifest decoding.
- Provide standalone packages for Windows x64, Linux x64, macOS Intel, and macOS
  Apple Silicon, with build metadata, checksums, user documentation, and notices.

Known limits include no general MSBuild evaluation, SDK custom search paths or
workloads, inferred application-runtime readiness, full Compose semantics,
arbitrary executable heuristics, dependency installation, automatic remediation,
or live service checks. Complex Node prerelease ranges and unsupported manager
layouts remain SKIP. See the [README](README.md) and support guides.
