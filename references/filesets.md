# Fileset Language Reference

Filesets are expressions that select sets of files. They are used in commands like `jj diff`, `jj split`, `jj squash -i`, `jj file list`, and anywhere a path argument is accepted.

## Operators

Listed in order of binding strength (tightest first):

| Operator | Meaning |
|----------|---------|
| `~x` | Everything except `x` |
| `x & y` | Files matching both `x` and `y` |
| `x ~ y` | Files matching `x` but not `y` |
| `x \| y` | Files matching either `x` or `y` |

Parentheses control evaluation order.

## Built-in Functions

```
all()        Matches all files
none()       Matches no files
```

## Path Pattern Types

All patterns can be used directly or with an explicit prefix. The default (bare string) is `cwd:`.

### Working-directory-relative patterns

| Pattern | Behavior |
|---------|---------|
| `"path"` or `cwd:"path"` | Prefix match from current working directory |
| `file:"path"` or `cwd-file:"path"` | Exact file path (cwd-relative) |
| `glob:"pattern"` or `cwd-glob:"pattern"` | Unix shell wildcards, non-recursive |
| `prefix-glob:"pattern"` | Like `glob:`, but also matches path prefix components |

### Workspace-root-relative patterns

| Pattern | Behavior |
|---------|---------|
| `root:"path"` | Prefix match from workspace root |
| `root-file:"path"` | Exact file path (root-relative) |
| `root-glob:"pattern"` | Unix shell wildcards from workspace root |
| `root-prefix-glob:"pattern"` | Like `root-glob:`, but also matches path prefix components |

### Case-insensitive variants

Append `-i` to any pattern type for case-insensitive matching:

```
glob-i:"*.TXT"
root-glob-i:"src/**/*.Ts"
cwd-file-i:"Readme.md"
```

## Glob Wildcard Syntax

| Pattern | Matches |
|---------|---------|
| `*` | Any characters except `/` (non-recursive) |
| `?` | Any single character except `/` |
| `[abc]` | Any character in the set |
| `[!abc]` | Any character NOT in the set |
| `**` | Any characters including `/` (recursive, when supported) |

Note: Basic `glob:` is non-recursive — it won't descend into subdirectories unless using `**`.

## Examples

### Excluding files

```bash
# Exclude a lockfile from a diff
jj diff '~Cargo.lock'

# Exclude generated files
jj diff '~glob:"**/*.generated.ts"'

# Exclude multiple patterns
jj diff '~(Cargo.lock | glob:"*.lock")'
```

### Listing files

```bash
# List all Rust source files
jj file list 'glob:"**/*.rs"'

# List src files except tests
jj file list 'src ~ glob:"src/**/*_test.rs"'

# List all TypeScript but not in node_modules
jj file list 'glob:"**/*.ts" ~ root:"node_modules"'
```

### Splitting commits

```bash
# Split interactively, excluding a specific file
jj split '~config/generated.json'

# Split by moving only specific directory
jj split 'src/auth'

# Split: put tests into their own commit
jj squash -i 'glob:"**/*.test.ts"'
```

### Scoping diffs

```bash
# Diff only a specific directory
jj diff src/api

# Diff all Python files
jj diff 'glob:"**/*.py"'

# Diff everything except docs
jj diff '~docs'
```

### Annotate (blame)

```bash
# Annotate a specific file
jj file annotate src/main.rs

# Works with root-relative paths
jj file annotate root:"src/components/Button.tsx"
```

## Combining with Revsets

Filesets appear in the `files()` revset function to select commits by what they changed:

```bash
# Commits that modified any Rust source file
jj log -r 'files(glob:"**/*.rs")'

# Commits that touched authentication code
jj log -r 'files("src/auth")'

# Commits that changed tests but NOT production code
jj log -r 'files(glob:"**/*.test.ts") ~ files(glob:"src/**/*.ts" ~ glob:"**/*.test.ts")'

# Recent commits that touched package.json
jj log -r 'latest(files("package.json"), 10)'
```

## Common Patterns

```bash
# Show diff, skipping lockfiles and generated files
jj diff '~(Cargo.lock | glob:"**/*.generated.*" | glob:"**/package-lock.json")'

# Squash only test changes into parent
jj squash 'glob:"**/*_test.go" | glob:"**/*.test.ts"'

# List changed files in current commit, excluding vendor
jj diff --summary '~root:"vendor"'

# Find commits touching config files
jj log -r 'files(glob:"**/*.toml" | glob:"**/*.yaml" | glob:"**/*.json")'
```
