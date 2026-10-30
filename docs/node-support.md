# Node.js support

WAM checks explicit declarations instead of assuming every package.json needs the
same runtime or package manager.

| Source | Supported meaning |
| --- | --- |
| `package.json` → `engines.node` | Declared supported Node versions compared with the observed runtime. |
| `.nvmrc` | Numeric major, major.minor, or full stable version, with optional `v` prefix and surrounding whitespace. |
| `.node-version` | Full stable version with optional `v` prefix and surrounding whitespace; build metadata is unsupported because manager lookup need not use npm equality. |
| `package.json` → `packageManager` | Exact `npm@X.Y.Z` pin, including exact prereleases; implies Node presence without inferring a Node version. |
| `package.json` → `engines.npm` | npm compatibility if npm is used. Missing/incompatible npm warns unless a supported npm manager pin establishes that npm is required. |

Each declaration is evaluated separately with source/field evidence. Conflicts
remain visible; WAM does not choose one declaration, merge projects, or resolve
version-manager inheritance outside the selected root. A bare package.json
without supported declarations produces SKIP.

Unknown managers, integrity suffixes, URLs, aliases such as `lts/*` / `node`,
version-file comments, computed settings, malformed JSON, duplicate keys, and
excessive nesting produce SKIP. Dependencies, scripts, lockfiles, and `devEngines`
are not interpreted or executed. WAM reports declared compatibility, not whether
npm's own engine-enforcement settings would reject installation.

## Supported npm range subset

- Stable exact versions and major/minor partials, with optional lowercase `v`.
- Trailing wildcards (`22.x`, `22.12.*`, `*`, `x`, `X`); no numeric component may
  follow a wildcard.
- Numeric comparators `=`, `>`, `>=`, `<`, `<=`, caret `^`, and tilde `~`.
  Wildcards may be bare or use `=`; other operators with explicit wildcard
  components are deliberately unsupported.
- Space-separated intersections, `||` unions, and standalone stable numeric
  hyphen ranges (`22.12 - 24.1`). Hyphen ranges cannot be combined with other
  comparators inside the same union branch.
- Build metadata on full versions, ignored in precedence comparisons.
- Exact prereleases, optionally prefixed with `=`. Compound prerelease expressions
  remain SKIP. Installed prereleases do not satisfy ordinary stable ranges.

Constraints are limited to 256 bytes of printable ASCII, eight OR groups, and
16 whitespace-separated tokens per group. Numeric version components are gated
at npm's safe-integer boundary. Numeric prerelease identifiers above
9,007,199,254,740,991 are unsupported in both declarations and observations;
JavaScript number conversion can collapse distinct larger integers. Alphanumeric
identifiers are not numeric components. Unsupported syntax produces SKIP rather
than a negative compatibility verdict.

The range evaluator uses a bounded, validated subset around a pinned
Masterminds/semver dependency. Its [license](licenses/Masterminds-semver.txt) is
included with executable distributions. This is not a claim of complete npm
range-language support.

## Installed tools and observation safety

`node --version` and a supported installed npm CLI version query run at most once
each per report. Node output must be one full supported semantic version.
Missing established requirements fail; optional npm compatibility warns when
absent/incompatible. Failed commands, unsafe installations, and uninterpretable
output produce SKIP.

WAM never invokes a shell, repository script, package installation, Corepack,
Yarn, or pnpm. For npm, PATH must identify either a symlink/direct entry named
`npm-cli.js` in an installed npm package, or a recognizable Windows `npm.cmd`
referencing the standard adjacent `node_modules/npm/bin/npm-cli.js`. Installed
package metadata must identify the package as npm.

The known CLI runs through trusted Node with `--version`, `--offline`,
`--ignore-scripts`, `--no-update-notifier`, and isolated user/global npm
configuration, cache, and log directories. The Windows wrapper is read only for
identification and is never executed. Prefix overrides and arbitrary
wrapper/version-manager dispatch are not simulated. Other layouts and native
npm dispatch shims produce SKIP.

Child processes remove inherited Node preloads/configuration and coverage,
compile-cache, and warning destinations; compile caching is explicitly disabled.
Native LD_/DYLD_ loader overrides are removed too. Temporary directories are
validated outside the inspected tree. Every command has a deadline and output
caps; output overflow cancels the observation and discards partial output.

No environment values or raw subprocess stderr enter reports. Tools and resolved
entrypoints inside the inspected repository are rejected, as are known script
suffixes, extensionless shebang shims, and special files. Trusted installed native
tools and native Node version-manager dispatch outside the repository remain a
trust boundary; these checks are not an operating-system sandbox.
