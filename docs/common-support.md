# Environment and external tools

WAM extracts configuration evidence and compares process presence. It does not load runtime configuration, resolve secrets, launch services, or certify that a supplied value is correct.

## Recognized files

- `.env.example` and `.env.sample` only; actual `.env`, `.env.local`, `.env.production`, and arbitrary secret files are not read.
- `compose.yaml`, `compose.yml`, `docker-compose.yaml`, `docker-compose.yml`, plus each corresponding `.override.yaml` / `.override.yml` variant. Custom filenames are outside the automatic discovery subset.

Files are inspected independently. No file merge, invocation-specific precedence, profile activation, include/extends resolution, or environment-file loading occurs. Referenced `env_file`, `label_file`, config, and secret paths are never opened by the Compose extractor. Discovery may independently read a recognized example file but never follows a path discovered in Compose. All filenames are matched literally; selecting a narrower root excludes unrelated examples/fixtures.

## Templates

Single-line `NAME=value` assignments support surrounding whitespace, optional `export`, blank/comment lines, single/double-quoted values, and unquoted inline comments preceded by whitespace. An optional UTF-8 BOM at the beginning of a template is ignored. `KEY=#literal` supplies an example; `KEY= #comment` does not. Names follow `[A-Za-z_][A-Za-z0-9_]*` with a 128-byte limit. Empty values and quoted empty strings have no example value. No expansion or shell evaluation occurs. A value resembling `${OTHER}` remains example metadata, not evidence that OTHER is required.

Quoted/multiline forms that cannot be interpreted within this subset cause the file's declarations to be skipped; malformed assignment lines get a safe location-only diagnostic. No source fragments or parser messages enter reports. Example values are represented only by a boolean and are not runtime defaults.

## Compose subset

The pinned maintained parser `go.yaml.in/yaml/v3` v3.0.5 decodes nodes without expanding aliases. Supported files have a nonempty top-level services mapping with valid literal service names. YAML duplicate keys, aliases/anchors (even in unused metadata), merge keys, custom tags, non-string mapping keys, and multiple documents cause SKIP for the file. Supported scalar types are strings, null, integers, floats, and booleans; environment booleans require quoting and otherwise yield a skipped value-type check. This is an evidence parser, not a complete Compose schema validator.

String values are inspected in `services`, `name`, `version`, `networks`, `volumes`, `secrets`, and `configs`. Mapping keys are not interpolated. Service extension metadata (`x-*`), includes, and extends are not expanded. Nested extension/model semantics are outside scope; supported references remain conditional evidence rather than proof that a workflow is used.

| Form | Recorded meaning |
| --- | --- |
| `$VAR`, `${VAR}` | Direct process/configuration reference. |
| `${VAR-default}` | Default supplied if unset; empty process value is allowed. |
| `${VAR:-default}` | Default supplied if unset or empty. |
| `${VAR?message}` | Explicit Compose requirement that the variable be set; empty is allowed. |
| `${VAR:?message}` | Explicit Compose requirement for a nonempty value. |
| `$$VAR` | Escaped dollar; no host-variable reference. |

Defaults and required-error messages are discarded after recording metadata. `${VAR:+replacement}`, `${VAR+replacement}`, nested substitutions, dollar expressions inside fallback/message text, unsupported operators, and malformed braced substitutions produce SKIP for the entire scalar. Other dollar characters that do not start a valid Compose reference remain literal.

Service environment mappings and `NAME=value` list entries establish a container assignment, not a same-named host requirement. Both literals and expressions are recorded as supplied container values; expressions separately produce host-reference checks. Null mappings and bare `NAME` list entries establish passthrough concerns. Explicit empty container strings remain supplied values. No container value is evaluated or printed. Names produced by interpolation are not guessed.

## Status and confidence

The process inspector returns only `present` and `non_empty`. It never returns values to the evaluator or renderer. Each exact variable name is observed once per report, and observations are reused across declarations.

- **PASS:** Process presence satisfies the declaration's presence check. No value validation is implied. Template emptiness warns; ordinary Compose references without a nonempty rule allow empty process values.
- **WARN:** A template/passthrough/reference may need configuration, or an explicit Compose input is absent/empty. Even explicit required interpolation does not FAIL because unread `.env` files or another launch environment can supply it.
- **SKIP:** A missing/empty process input has a declared Compose default, a container assignment supplies a value/expression, a service is not probed, or the declaration exceeds the supported subset.

Confidence is `possible` for templates/plain references/passthrough, `explicit-compose` for required interpolation, `defaulted` for fallback declarations, and `container-only` for supplied container assignments. Example presence alone never upgrades confidence or suppresses an absent-variable warning.

Diagnostics retain `source` and `line` where known. For YAML block scalars, the line is the scalar's declaration/header line rather than a separately computed location for every embedded reference. The optional JSON `environment` object contains `kind`, `present`, `non_empty`, `example_supplied`, `default_supplied`, `container_value_supplied`, `explicitly_required`, and `confidence`. It contains no values. Console reports always show source locations; `--verbose` adds supporting evidence.

## Docker and services

A supported services mapping produces one Docker workflow hint per file. The Docker executable is looked up once per report through the trusted PATH policy. Presence yields PASS for this limited capability; absence yields WARN because a Compose declaration does not rule out another engine or an unused workflow. Unsafe/inaccessible resolution yields SKIP. Docker is never executed; Compose plugin availability, daemon readiness, image availability, ports, remote contexts, and live services are not inspected.

Each service declaration is explicitly SKIP for live availability. Its literal service name is displayed; an image name does not imply that a database must run on the host. Dockerfiles, arbitrary application strings, scripts mentioning `ffmpeg`/`git`, and generic executable dependencies are not inspected. The MVP adds no bespoke tool declaration format.

## Limits

Normal discovery limits still apply. Additionally: Compose input at most 128 KiB, node-tree depth 64, 20,000 nodes, 256 services, 512 combined variable/service/issue entries per file, 64 KiB per interpolation scalar, and 128 references per scalar. Templates allow 64 KiB scanner lines, 4,096 lines, and 512 variable/issue entries, plus a terminal overflow notice. Common extraction stops accepting files when their combined results would exceed 4,096 entries and adds a terminal SKIP notice. Compose overflow discards partial file requirements. All limits produce visible SKIP diagnostics. No unsupported input is treated as a verified failure.

References: [Compose interpolation](https://docs.docker.com/reference/compose-file/interpolation/), [Compose service environment](https://docs.docker.com/reference/compose-file/services/#environment), [maintained Go YAML parser](https://github.com/yaml/go-yaml).
