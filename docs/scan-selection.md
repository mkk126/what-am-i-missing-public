# Selecting repository scope

`wam check path/to/project` already selects a project directory. Use repeated
`--exclude` options when the intended scan is a larger repository containing
unrelated examples, fixtures, or applications:

```sh
wam check . --exclude testdata --exclude examples
wam check "repository with spaces" --exclude "sample projects" --format json
wam check . --exclude=fixtures --verbose
```

Exclusions are explicit directory boundaries, not version overrides. The default
scan continues recognizing examples and fixtures; WAM does not silently decide
that those directories are irrelevant to every project.

## Path rules

- Paths are relative to the directory passed to `check`, regardless of the shell's
  current directory. Both `/` and `\` separators are accepted on every platform.
- The directory must exist. Empty paths, the selected root, absolute/drive paths,
  parent traversal, and files such as `global.json` are rejected with exit code 2.
  Colons and NUL bytes are unsupported in this portable relative-path syntax.
- Every component must be a directory without symbolic links. No path discovered
  in a manifest becomes an exclusion or causes another file to be opened.
- Matching is literal; there is no glob, negation, ignore-file, or Git-ignore
  interpretation. Spaces and Unicode names are supported. Filesystem identity
  matching handles case aliases when the filesystem permits them.
- At most 64 arguments are accepted, each at most 4,096 bytes. A selection must be
  reachable under the depth budget: with the default depth 16, excluded paths may
  contain at most 17 directory components. That last boundary can be skipped
  before it becomes an incomplete-discovery warning.

WAM validates all selections before reading manifests or requesting runtime/tool
observations. It prunes excluded subtrees before enumerating their entries, so
their contents consume no entry/manifest budget. The excluded directory's own
name still counts as an entry in its included parent directory.

Equivalent paths are sorted and deduplicated. A selected parent makes selected
descendants redundant, including filesystem case aliases. The retained report
spelling is the cleaned, slash-separated input path; it does not claim to rename
or canonicalize the filesystem's directory names.

## Evidence and SDK scope

JSON reports with explicit selections include additive schema-version-1 metadata:

```json
"selection": {
  "excluded_directories": ["examples", "testdata"]
}
```

The field is absent for the default scan. A directory explicitly selected for
exclusion is not an inspection failure or a warning. The report describes only
the included scope; excluded applications have not been checked.

Individual configuration files cannot be excluded. In particular, an included
project cannot bypass its discovered `global.json` by hiding that one manifest.
Excluding an entire project subtree also removes its nested declarations; those
declarations never become requirements for included sibling projects. Selecting
a narrower `check` path retains the existing limitation that SDK declarations in
ancestors outside the selected root are not inspected.

Inspections are read-only. Directory identities are checked as part of matching,
but the repository is still a live filesystem rather than an atomic snapshot.
Concurrent edits retain the documented discovery/trust boundaries.
