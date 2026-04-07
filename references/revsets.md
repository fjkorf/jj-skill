# Revset Language Reference

Revsets are expressions that select sets of commits. Most `-r` flags accept a revset.

## Operators

Listed in order of binding strength (highest first):

| Operator | Meaning |
|----------|---------|
| `x-` | Parents of `x` (can be empty) |
| `x+` | Children of `x` (can be empty) |
| `x::` | Descendants of `x`, including `x` itself |
| `x..` | Revisions that are NOT ancestors of `x` |
| `::x` | Ancestors of `x`, including `x` itself |
| `..x` | Ancestors of `x`, excluding the root commit |
| `x::y` | Descendants of `x` that are also ancestors of `y` (DAG range) |
| `x..y` | Ancestors of `y` that are NOT also ancestors of `x` |
| `::` | All visible commits |
| `..` | All visible commits excluding root |
| `~x` | Revisions NOT in `x` |
| `x & y` | Intersection — commits in both |
| `x ~ y` | Difference — commits in `x` but not `y` |
| `x \| y` | Union — commits in either set |

## Symbols

| Symbol | Meaning |
|--------|---------|
| `@` | Working copy commit |
| `@-` | Parent of working copy |
| `root()` | Virtual root commit |
| `<change-id>` | Commit by change ID prefix |
| `<bookmark-name>` | Tip of a bookmark |

## Navigation Functions

```
parents(x, [depth])          Direct parents; depth= how many levels up
children(x, [depth])         Direct children; depth= how many levels down
ancestors(x, [depth])        All ancestors of x, optionally limited by depth
descendants(x, [depth])      All descendants of x, optionally limited by depth
first_parent(x, [depth])     Parents excluding non-first parents in merges
first_ancestors(x, [depth])  Traverse only first-parent lineage
```

## Set Operation Functions

```
all()                        All visible commits and ancestors of explicitly-mentioned commits
none()                       Empty set
heads(x)                     Commits in x that are not ancestors of other commits in x
roots(x)                     Commits in x that are not descendants of other commits in x
connected(x)                 x::x — all commits reachable from x to x
reachable(srcs, domain)      All commits reachable from srcs within domain
```

## Lookup Functions

```
change_id(prefix)            Commits matching a change ID prefix
commit_id(prefix)            Commits matching a commit ID prefix
bookmarks([pattern])         Local bookmark targets (glob pattern optional)
remote_bookmarks([name], [[remote=]remote])  Remote bookmark targets
tracked_remote_bookmarks(...)   Only tracked remote bookmarks
untracked_remote_bookmarks(...) Only untracked remote bookmarks
tags([pattern])              Tag targets
remote_tags([name], [[remote=]remote])  Remote tag targets
visible_heads()              All visible heads (commits with no children)
root()                       The virtual root commit
```

## Selection Functions

```
latest(x, [count])           Most recent commits in x by timestamp; count defaults to 1
fork_point(x)                Common ancestor(s) of all commits in x
bisect(x)                    Commits where approximately half of x are descendants (git bisect)
exactly(x, count)            Returns x if it contains exactly count commits; errors otherwise
```

## Filtering Functions

```
merges()                     Merge commits (more than one parent)
empty()                      Commits modifying no files (includes merges and root)
conflicts()                  Commits with files in conflicted state
divergent()                  Commits with a divergent change (same change ID, different content)
signed()                     Cryptographically signed commits
```

## Text Matching Functions

```
description(pattern)         Match commit descriptions
subject(pattern)             Match first line of commit description only
author(pattern)              Author name or email matching
author_name(pattern)         Author name matching only
author_email(pattern)        Author email matching only
author_date(pattern)         Match by author date
mine()                       Commits where author email matches current user
committer(pattern)           Committer name or email matching
committer_name(pattern)      Committer name matching only
committer_email(pattern)     Committer email matching only
committer_date(pattern)      Match by committer date
```

## File & Content Functions

```
files(expression)            Commits modifying paths matching a fileset expression
diff_lines(text, [files])    Commits containing diffs matching the given text pattern
```

Example:
```bash
jj log -r 'files("src/auth.rs")'              # commits touching src/auth.rs
jj log -r 'files(glob:"src/**/*.ts")'         # commits touching any .ts in src/
jj log -r 'diff_lines("TODO")'               # commits adding/removing lines with "TODO"
```

## Utility Functions

```
present(x)                   Returns x, or none() if commits don't exist (suppress errors)
coalesce(revsets...)         First revset that doesn't evaluate to none()
working_copies()             Working copy commits across all workspaces
at_operation(op, x)          Evaluate x at the state of a specific operation
```

## String Pattern Syntax

Pattern strings used in text matching functions:

| Pattern | Behavior |
|---------|----------|
| `"string"` (default) | Glob matching |
| `exact:"string"` | Exact match |
| `glob:"pattern"` | Unix-style wildcards |
| `regex:"pattern"` | Regular expression |
| `substring:"string"` | Contains match |
| Add `-i` suffix | Case-insensitive (e.g., `glob-i:"fix*"`) |

Patterns can be combined: `author(exact:"alice" | exact:"bob")`

## Date Pattern Syntax

Used with `author_date()`, `committer_date()`:

```
after:"2024-02-01"            On or after this date
before:"2024-02-01"           Before this date
after:"2024-02-01T12:00:00"   With time
after:"2 days ago"            Relative dates
after:"last monday"           Natural language
```

Example:
```bash
jj log -r 'author_date(after:"2024-01-01") & mine()'
```

## Built-in Aliases

These aliases are pre-defined and can be used anywhere:

| Alias | Expands To |
|-------|-----------|
| `trunk()` | Default remote's main bookmark (usually `main` or `master`) |
| `immutable_heads()` | `trunk() \| tags() \| untracked_remote_bookmarks()` |
| `immutable()` | `::(immutable_heads() \| root())` |
| `mutable()` | `~immutable()` |
| `visible()` | `::visible_heads()` |
| `hidden()` | `~visible()` |

## Common Patterns

```bash
# My unpushed commits
jj log -r 'remote_bookmarks()..@'

# My commits on current stack above main
jj log -r '::@ ~ ::main'

# All commits above main
jj log -r 'trunk()..@'

# My recent commits
jj log -r 'mine() & latest(all(), 20)'

# Commits with conflicts
jj log -r 'conflicts()'

# Commits touching a specific file
jj log -r 'files("src/main.rs")'

# Commits by author mentioning a keyword
jj log -r 'author("alice") & description("fix")'

# Latest commit on a bookmark
jj log -r 'heads(main::)'

# Divergent commits (duplicate change IDs)
jj log -r 'divergent()'

# All visible heads
jj log -r 'visible_heads()'

# Find commits in date range
jj log -r 'author_date(after:"2024-01-01") & author_date(before:"2024-06-01")'

# Commits that changed TypeScript files
jj log -r 'files(glob:"**/*.ts")'

# Commits on this branch but not yet on remote
jj log -r 'remote_bookmarks(my-bookmark)..my-bookmark'

# Latest 5 commits from any author matching "feature"
jj log -r 'latest(description(substring:"feature"), 5)'

# All ancestors of @ that are not ancestors of trunk
jj log -r 'ancestors(@) ~ ancestors(trunk())'
```

## Revset at a Past Operation

```bash
# What did the repo look like 3 operations ago?
jj log -r 'at_operation(@-3, all())'

# Commits that existed before the last rebase
jj log -r 'at_operation(@-1, ::@)'
```
