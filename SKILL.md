---
name: jj
description: Expert guidance for Jujutsu (jj), a Git-compatible VCS with first-class conflict handling, automatic snapshots, and a working-copy-as-commit model. Use this skill when the user needs help with jj commands, workflows, revsets, bookmarks, rebasing, splitting commits, simultaneous multi-branch editing, or migrating from Git.
allowed-tools: Bash, Read, Grep, Glob
---

# Jujutsu (jj) VCS Skill

Expert guidance for `jj`, a modern Git-compatible version control system that treats the working copy as a commit.

## Core Mental Model

**Working copy is always a commit** — marked `@` in `jj log`. Changes are automatically snapshotted after every command; there is no staging area.

**Change ID vs Commit ID** — every commit has two identifiers:
- **Change ID** (e.g. `puqltutt`): stable identity that survives rewrites/rebases. Shown in purple/bold.
- **Commit ID** (e.g. `abc123`): content hash that changes on every rewrite. Shown in blue.

**Always use change IDs, not commit hashes**, as targets for `--from`, `--into`, `-r`, etc. After any squash or rebase, all descendant commit hashes change — but change IDs stay the same. Using a stale hash silently targets an orphaned commit with no error.

**No current branch** — you work on commits directly. Bookmarks are explicit pointers you move manually.

**Automatic rebasing** — when you edit a commit, all descendants rebase automatically.

**Operation log** — every repository-modifying command is recorded; any operation can be undone.

## When to Use This Skill

Invoke when the user needs to:
- Learn jj concepts coming from Git
- Run jj commands for everyday VCS tasks
- Use revsets to select commits
- Manage bookmarks (jj's equivalent of branches)
- Split, squash, or reorganize commits
- Resolve conflicts stored in commits
- Undo operations using `jj undo` / `jj op log`
- Work with GitLab via Git interop
- Work on multiple branches simultaneously using merge commits and `jj absorb`
- Rebase all feature branches at once when trunk advances

## Prerequisites

Verify installation:
```bash
jj --version
```

If not installed, guide the user to: https://docs.jj-vcs.dev/latest/install-and-setup/

## Initializing a Repository

```bash
# New repo (pure jj, no .git)
jj git init

# New repo, colocated with Git (creates both .jj/ and .git/)
jj git init --colocate

# Clone a Git repo
jj git clone https://gitlab.com/org/repo
jj git clone --colocate https://gitlab.com/org/repo
```

> **Colocated repos**: share history between jj and git. Recommended when collaborating via GitLab. Tradeoff: some jj features interact with Git's index.

---

## Simultaneous Edits: Working on All Branches at Once

jj's merge-commit model lets you work across multiple open branches in a single working copy — editing, testing, and rebasing all of them together. This is a power workflow for managing several in-flight PRs/MRs at once.

### Step 1 — Create a merge commit over all active branches

```bash
# Given bookmarks: feat-a, feat-b, feat-c
jj new feat-a feat-b feat-c -m "merge: working copy"
```

Your working copy (`@`) now has all three as parents. Changes you make here can see all of them simultaneously.

### Step 2 — Make changes, then distribute back

**Option A — `jj absorb` (preferred when changes clearly belong to existing parents):**

```bash
jj absorb
```

jj inspects which lines were last touched by each parent and automatically routes your changes to the right one. No manual `--into` required.

**Option B — `jj squash --into` (when you need explicit control):**

```bash
jj squash --from @ --into feat-a   # move selected changes into feat-a
jj squash -i --from @ --into feat-b  # interactively pick what goes to feat-b
```

### Step 3 — Add commits to specific branches

To add a new commit on just one branch without disturbing the others:

```bash
jj new feat-b -m "#123: add extra work to feat-b"
# make changes...
jj bookmark move feat-b   # advance the bookmark to the new commit
```

Then re-create your merge commit to include it:

```bash
jj new feat-a feat-b feat-c -m "merge: working copy"
```

### Step 4 — Rebase all branches when trunk advances

When `develop`/`trunk` gets new commits, rebase every feature branch in one command:

```bash
jj rebase -s 'all:roots(trunk..@)' -o trunk
```

- `roots(trunk..@)` — finds the base commit of each feature branch
- `all:` prefix — required when the revset matches more than one root
- All branches and their descendants (including your merge commit) rebase automatically

---

## Everyday Commands

### Check status and view history

```bash
jj st                  # show working copy status
jj diff                # diff of working copy changes
jj diff --git          # diff in git format
jj log                 # commit graph (default: smart revset)
jj log -r 'all()'      # full history
jj log -r ::@          # working copy and all ancestors
jj show                # show current commit diff + message
jj show <rev>          # show specific commit
```

### Describe / commit

```bash
jj describe            # set/edit message on current commit
jj describe -m "msg"   # set message inline
jj commit              # describe + create new empty commit on top
jj commit -m "msg"     # inline message
```

### Create new commits

```bash
jj new                 # new empty commit on top of @
jj new -m "msg"        # with message
jj new <rev>           # new commit on top of specific revision
jj new <rev1> <rev2>   # new merge commit
```

### Edit a past commit

```bash
jj edit <rev>          # make @ point at that commit (no new commit created)
```

### Squash / amend

```bash
jj squash                                   # move @ changes into parent (like git commit --amend)
jj squash -i                                # interactively choose which changes to move to parent
jj squash --from @ --into <change-id>       # move changes into any commit (use change ID!)
jj squash --from @ --into <change-id> -- file1 file2  # move specific files only
```

> **Always pass `-m "message"` when source and destination have different descriptions** — without it, jj opens an interactive editor to merge commit messages, which breaks non-interactive shells and scripts.

### Absorb

```bash
jj absorb              # automatically move @ changes into the nearest mutable ancestor
                       # that already touches the same files/lines
```

`jj absorb` is especially powerful when working on a merge commit that spans multiple branches: edits you make in the working copy are automatically redistributed into whichever parent branch originally touched those lines. Use it instead of `jj squash --from … --into …` when each change obviously belongs to one existing parent.

### Split a commit

```bash
jj split               # interactively split @ into two commits
jj split -r <rev>      # split a specific commit
```

### Undo and recover

```bash
jj undo                         # undo the last operation
jj op log                       # view full operation history
jj op undo <op-id>              # undo a specific operation
jj op restore <op-id>           # restore repo state to a past operation
jj op restore --what repo <id>  # restore repo state without touching working copy
jj evolog -r <change-id>        # evolution history of a change (all past commit hashes)
```

> **`jj op restore` is your safety net.** If a rebase goes wrong (conflicts explode, divergent commits multiply), find a known-good state in `jj op log` and restore to it.

### Abandon / delete commits

```bash
jj abandon <rev>       # abandon commit (descendants rebase automatically)
jj abandon <rev1> <rev2>  # abandon multiple
```

## Bookmarks (Branches)

Bookmarks are explicit pointers. They do **not** auto-advance when you create new commits.

```bash
jj bookmark list                    # list all bookmarks
jj bookmark list --all              # include remote bookmarks
jj bookmark create <name>           # create bookmark at @
jj bookmark create <name> -r <rev>  # create at specific rev
jj bookmark move <name>             # move bookmark to @
jj bookmark move <name> --to <rev>  # move to specific rev
jj bookmark delete <name>           # delete local bookmark
jj bookmark track <remote>/<name>   # track a remote bookmark
```

## Remotes and Push/Fetch

```bash
jj git remote list
jj git remote add origin <url>

jj git fetch                        # fetch all remotes
jj git fetch --remote origin

jj git push                         # push all tracked bookmarks
jj git push -b <bookmark>           # push specific bookmark (flag is singular: --bookmark)
jj git push -r '<revset>'           # push all bookmarks pointing to commits in revset
jj git push --all                   # push all bookmarks
```

> **Push multiple bookmarks via revset** — instead of repeating `-b` per bookmark:
> ```bash
> jj git push -r 'bookmarks(glob:"5577-*") | bookmarks(glob:"558*")'
> ```

## Rebasing

```bash
jj rebase -d <dest>               # rebase @ onto dest
jj rebase -r <rev> -d <dest>      # rebase specific rev onto dest
jj rebase -s <source> -d <dest>   # rebase source and all descendants onto dest
jj rebase -b <bookmark> -d <dest> # rebase entire bookmark stack onto dest
```

Descendants always auto-rebase after any parent modification.

### Rebase all branches simultaneously

When you have multiple active feature branches and `trunk` (or `develop`) advances, rebase all of them at once:

```bash
jj rebase -s 'all:roots(trunk..@)' -o trunk
```

- `-s` — rebase a revision and all its descendants
- `all:` prefix — tells jj to expect multiple revisions; errors without it when the revset matches more than one commit
- `roots(trunk..@)` — the root commits between `trunk` and your working copy (i.e. the base of each feature branch)
- `-o trunk` — new parent for all those roots

## Conflict Handling

Conflicts are stored **inside commits** — not as working-tree markers that block you. You can commit, rebase, and continue working while conflicts exist.

```bash
jj log -r 'conflicts()'    # find all commits with conflicts
jj status                  # shows conflict markers in working copy
jj resolve                 # open interactive merge tool for working copy conflicts
jj resolve -r <rev>        # resolve conflicts in a specific commit
```

## Stack Management

Use templates to get a compact overview of a branch stack in a single command:

```bash
# Compact stack status: change ID, bookmark, conflict flag, file count, description
jj log -r '<revset>' --no-graph -T '
change_id.short(8) ++ "\t" ++
if(local_bookmarks.len() > 0, local_bookmarks.map(|b| b.name()).join(","), "-") ++ "\t" ++
if(conflict, "CONFLICT", "ok") ++ "\t" ++
pad_start(3, self.diff().files().len()) ++ " files\t" ++
description.first_line().substr(0, 70) ++ "\n"
'
```

Useful template methods for stack inspection:
```
self.diff().files().len()                        # file count per commit
self.diff().files().map(|f| f.path()).join("\n")  # all changed file paths
local_bookmarks.map(|b| b.name()).join(",")       # bookmark names at commit
```

---

## Commit Splitting (Non-Interactive)

`jj split` opens an interactive editor. For scripted/non-interactive splitting, use the **examination case** pattern:

```bash
# 1. Create empty target commits on the parent of the big commit
jj new <big-change-id>- -m "feat: logical group 1"
# → output shows: Working copy now at: aaaaaaaa ...
jj new @ -m "feat: logical group 2"
# → output shows: Working copy now at: bbbbbbbb ...

# 2. Rebase the big commit onto the last empty commit
jj rebase -s <big-change-id> -d @

# 3. Distribute files using change IDs (no jj log needed between calls!)
jj squash --from <big-change-id> --into aaaaaaaa -m "feat: logical group 1" -- file1 file2
jj squash --from <big-change-id> --into bbbbbbbb -m "feat: logical group 2" -- file3 file4
```

Change IDs are stable across rewrites, so **no `jj log` is needed between squash calls** to refresh hashes.

**Building commits from scratch** — alternative using `jj restore`:
```bash
jj new develop -m "feat: group 1"
jj restore --from <source-change-id> -- file1.php file2.php
jj new @ -m "feat: group 2"
jj restore --from <source-change-id> -- file3.php file4.php
```

`jj restore --from <rev> -- <path>` correctly handles added, deleted, and modified files.

---

## Divergence

**Divergent changes** occur when multiple live commits share the same change ID. jj marks them with `??`:

```bash
jj log -r 'divergent()'    # find all divergent commits
# Output shows:
# pmrxpmzq?? aaa111   ← local rewrite
# pmrxpmzq?? bbb222   ← CI's version (fetched)
```

Common causes:
- CI auto-commits (e.g. `chore: lint fixes`) land on remote bookmarks while you have local rewrites
- Two workspaces rewrite the same commit concurrently

**Resolution:**
```bash
# Keep one, abandon the other
jj bookmark set <bookmark-name> -r <correct-commit-id>
jj abandon <wrong-commit-id>

# Or if the CI fix is already embedded in your local version:
jj bookmark set --allow-backwards -r <local-commit>
```

**Prevention:** always `jj git fetch` before starting work on a pushed stack — CI may have landed chore commits since your last push.

---

## Internals (Brief)

jj uses Git's object store but adds a metadata layer:

| Layer | Location | Contents |
|---|---|---|
| Git objects | `.git/objects/` | Blobs, trees, commits (actual file content) |
| jj extras | `.jj/repo/store/extra/` | Change ID + predecessors per commit (protobuf sidecar) |
| jj operations | `.jj/repo/op_store/` | Operation log: views, bookmark positions, rewrite history |

- **Change IDs are 16 random bytes** — not derived from content. They're stored in the extras table, keyed by commit hash.
- **Squash/rebase creates new Git commits** (new hashes) but copies the same change ID into the new extras entry.
- **jj never stores diffs** — like Git, it stores full tree snapshots. Diffs are computed on the fly.
- **Orphaned commits** (old hashes after rewrites) still exist in `.git/objects/` until `jj util gc`. They're not in the index, but can be targeted by stale hashes — which is why change IDs are safer.

---

## Filesets

Filesets are expressions that select files. Used with `jj diff`, `jj split`, `jj squash -i`, `jj file list`, and the `files()` revset function.

### Operators
| Operator | Meaning |
|----------|---------|
| `~x` | Everything except `x` |
| `x & y` | Files matching both |
| `x ~ y` | Files matching `x` but not `y` |
| `x \| y` | Files matching either |

### Pattern types
| Pattern | Meaning |
|---------|---------|
| `"path"` or `cwd:"path"` | cwd-relative prefix match (default) |
| `file:"path"` | Exact cwd-relative path |
| `glob:"pattern"` | Unix wildcards (non-recursive) |
| `root:"path"` | Workspace-root-relative prefix |
| `root-file:"path"` | Exact root-relative path |
| `root-glob:"pattern"` | Workspace-root-relative wildcards |
| Add `-i` suffix | Case-insensitive (`glob-i:"*.TXT"`) |

### Common fileset patterns
```bash
jj diff '~Cargo.lock'                          # exclude lockfile
jj diff 'glob:"**/*.ts" ~ glob:"**/*.test.ts"' # TS, not tests
jj split 'src/auth'                            # only auth directory
jj file list 'glob:"**/*.rs"'                  # list Rust files
jj log -r 'files(glob:"**/*.go")'             # commits touching Go files
jj log -r 'files("src/auth") & mine()'        # my auth changes
```

For full fileset reference, load **references/filesets.md**.

---

## Revsets

Revsets are a query language for selecting commits. Most `-r` flags accept a revset.

### Symbols
| Symbol | Meaning |
|--------|---------|
| `@` | Working copy commit |
| `@-` | Parent of working copy |
| `root()` | Virtual root commit |
| `<change-id>` | Commit by change ID prefix |
| `<bookmark>` | Tip of a bookmark |

### Operators
| Operator | Meaning |
|----------|---------|
| `x-` | Parents of x |
| `x+` | Children of x |
| `::x` | Ancestors of x (inclusive) |
| `x::` | Descendants of x (inclusive) |
| `x::y` | DAG range from x to y |
| `x..y` | Ancestors(y) excluding ancestors(x) |
| `~x` | Complement |
| `x & y` | Intersection |
| `x ~ y` | Difference |
| `x \| y` | Union |
| `all:x` | Assert that `x` matches multiple revisions (required by flags like `-s` when revset returns >1 result) |

### Core functions
```
ancestors(x, [depth])      ancestors of x, optional depth limit
descendants(x, [depth])    descendants of x, optional depth limit
parents(x)                 direct parents of x
children(x)                direct children of x
heads(x)                   tips of x (not ancestors of other commits in x)
roots(x)                   roots of x (not descendants of others in x)
latest(x, [count])         most recent commits in x (default 1)
reachable(srcs, domain)    all commits reachable from srcs within domain
```

### Lookup functions
```
bookmarks([pattern])                      local bookmark targets
remote_bookmarks([name], [remote=remote]) remote bookmark targets
tags([pattern])                           tag targets
visible_heads()                           all visible heads
```

### Filtering functions
```
conflicts()                commits with files in conflicted state
merges()                   merge commits
empty()                    commits modifying no files
divergent()                commits with duplicate change IDs
mine()                     commits by the current user
author("name")             author name or email matching
author_date(after:"date")  commits by author date
description("text")        commits whose message matches pattern
subject("text")            commits whose first line matches
files(expression)          commits modifying paths matching a fileset
diff_lines("text")         commits containing diffs matching text
```

### String pattern syntax
```
"string"           glob (default)
exact:"string"     exact match
glob:"pattern"     unix wildcards
regex:"pattern"    regular expression
substring:"string" contains match
append -i          case-insensitive (glob-i:"fix*")
```

### Built-in aliases
| Alias | Meaning |
|-------|---------|
| `trunk()` | Default remote's main bookmark |
| `immutable()` | Commits that cannot be rewritten |
| `mutable()` | Commits that can be rewritten |

### Common revset patterns
```bash
jj log -r 'remote_bookmarks()..@'          # my unpushed commits
jj log -r '::@ ~ ::main'                   # my commits above main
jj log -r 'trunk()..@'                     # all commits above trunk
jj log -r 'conflicts()'                     # commits with conflicts
jj log -r 'description(substring:"fix")'   # commits mentioning "fix"
jj log -r 'mine() & bookmarks()'           # my bookmarks
jj log -r 'files("src/auth.rs")'           # commits touching a file
jj log -r 'files(glob:"**/*.ts")'          # commits touching TS files
jj log -r 'author_date(after:"2024-01-01") & mine()'  # my recent commits
jj log -r 'latest(description("feature"), 5)'          # 5 newest "feature" commits
```

For full revset reference (all functions, date patterns, advanced examples), load **references/revsets.md**.

---

## Templates

Templates control output formatting via `-T`/`--template`. Used with `jj log`, `jj show`, `jj op log`.

### Operators
| Operator | Meaning |
|----------|---------|
| `x.f()` | Method call |
| `x ++ y` | Concatenation |
| `if(cond, then, [else])` | Conditional |
| `separate(sep, ...)` | Join non-empty values with separator |

### Common keywords (in `jj log` context)
| Keyword | Description |
|---------|-------------|
| `change_id` | Change ID |
| `commit_id` | Commit hash |
| `description` | Full commit message |
| `author` | Author signature |
| `parents` | List of parent commits |
| `bookmarks` | Local bookmarks at this commit |
| `mine` | Boolean: authored by current user |
| `conflict` | Boolean: has conflicts |
| `empty` | Boolean: no file changes |
| `diff` | TreeDiff: file changes |

### Useful methods
```
change_id.short()                short prefix
commit_id.short()                short prefix
description.first_line()         first line only
author.name()                    author name
author.email()                   author email
author.timestamp().ago()         relative time ("5 minutes ago")
author.timestamp().format("%Y-%m-%d")  formatted date
parents.map(|c| c.commit_id().short()).join(", ")
bookmarks.join(", ")
diff.stat()                      files changed, +/-
diff.summary()                   A/M/D per file
```

### Examples
```bash
# Short ID + description
jj log -T 'change_id.short() ++ " " ++ description.first_line() ++ "\n"'

# With conflict/empty markers
jj log -T 'if(conflict, "CONFLICT ") ++ if(empty, "empty ") ++ description.first_line() ++ "\n"'

# Author and date
jj log -T 'author.name() ++ " " ++ author.timestamp().ago() ++ " " ++ description.first_line() ++ "\n"'

# Machine-readable: commit_id TAB description
jj log -T 'commit_id ++ "\t" ++ description.first_line() ++ "\n"'

# Show bookmarks when present
jj log -T 'surround("[", "] ", bookmarks.join(", ")) ++ description.first_line() ++ "\n"'

# Built-in named templates
jj log -T builtin_log_oneline
jj log -T builtin_log_detailed
```

For full template reference (all types, methods, functions, aliases), load **references/templates.md**.

## Configuration

Config file: `~/.config/jj/config.toml`

```toml
[user]
name = "Your Name"
email = "you@example.com"

[ui]
editor = "vim"
diff-editor = "vimdiff"
merge-editor = "vimdiff3"
paginate = "auto"

[aliases]
l = ["log", "-r", "remote_bookmarks()..@"]
```

```bash
jj config list
jj config get user.email
jj config set --user user.email "you@example.com"
```

## Git Interop: Command Equivalents

| Git | jj |
|-----|-----|
| `git init` | `jj git init` |
| `git clone <url>` | `jj git clone <url>` |
| `git status` | `jj st` |
| `git diff HEAD` | `jj diff` |
| `git log --graph` | `jj log` |
| `git add -p; git commit` | `jj split` or `jj squash -i` |
| `git commit -a` | `jj commit` |
| `git commit --amend` | `jj squash` |
| `git stash` | `jj new @-` |
| `git cherry-pick` | `jj duplicate -r <rev> -d @` |
| `git revert` | `jj backout -r <rev>` |
| `git reset --hard` | `jj abandon` |
| `git reset --soft HEAD~` | `jj squash --from @-` |
| `git branch` | `jj bookmark list` |
| `git branch -f <name>` | `jj bookmark move <name>` |
| `git checkout -b <name>` | `jj new -m "..." && jj bookmark create <name>` |
| `git rebase` | `jj rebase` |
| `git fetch` | `jj git fetch` |
| `git push` | `jj git push` |
| `git blame` | `jj file annotate` |
| `git reflog` | `jj op log` |

## Common Pitfalls and FAQ

**"My changes disappeared"** — They didn't. `jj op log` shows all operations. `jj undo` to recover. `jj log -r 'all()'` to find the commit.

**"How do I unstage?"** — No staging area in jj. Changes in working copy are automatically part of `@`. Use `jj split` to separate them.

**"How do I switch to another commit?"** — `jj edit <rev>` makes that commit the working copy. Or `jj new <rev>` to start fresh work on top of it.

**"Bookmark not moving after commit"** — Correct. Bookmarks are explicit. Use `jj bookmark move <name>` to advance them.

**"Push was rejected"** — jj push is a force-push by design. This is expected in the squash-before-push workflow.

**"How do I resolve conflicts?"** — `jj resolve` opens your merge editor. Or edit the file directly to remove conflict markers.

**"Change ID vs commit ID confusion"** — Always use change IDs for `--from`, `--into`, `-r`, etc. Commit hashes go stale after any rewrite and silently target orphaned commits.

**"How do I see what I'm about to push?"** — `jj log -r 'remote_bookmarks()..@'` or `jj log -r '::@ ~ ::main'`. Always do this before pushing.

**"Bookmark name with hyphens fails in revsets"** — Hyphens are parsed as subtraction. Quote the name: `jj log -r '"my-branch"..@'` or use `bookmarks(exact:"my-branch")`.

**"Squash opened an editor"** — `jj squash` opens an editor when source and destination have different descriptions. Pass `-m "message"` to avoid this in scripts and non-interactive shells.

**"Diff shows mangled lines like `oldvaluenewvalue`"** — This is a `jj diff` rendering artifact where adjacent old/new values are concatenated on one line. Read the actual file to verify: `jj file show -r <rev> path/to/file`.

**"File renames differing only in case can't be split"** — On macOS (case-insensitive filesystem), both paths resolve to the same inode. Accept the rename in whichever commit it naturally falls into.

## Best Practices

1. **Use change IDs, not commit hashes** — change IDs survive all rewrites. Hashes go stale and silently target orphaned commits.
2. **Pass `-m` with squash in scripts** — prevents the interactive editor from opening when descriptions differ.
3. **Fetch before starting** — always `jj git fetch` before beginning new work. CI may have landed commits since your last push.
4. **Never push without review** — always run `jj log` on the target revset and confirm with the user before `jj git push`.
5. **Squash before pushing** — clean up intermediate commits into logical units before any push.
6. **Use `jj undo` freely** — every operation is undoable, so experiment without fear.
7. **Describe early** — set commit messages with `jj describe` before the work grows large.
8. **Colocate when using GitLab** — `jj git init --colocate` keeps `.git` in sync.

---

## Progressive Disclosure

For deeper reference material, load these files when needed:

- **references/workflows.md** — The two primary jj workflows (squash and edit), navigation commands (`jj next`, `jj prev`, `jj edit`), inserting commits with `jj new -B`, and when to use each approach
- **references/filesets.md** — All fileset operators, pattern types, glob syntax, and examples for selecting files in diffs, splits, and squashes
- **references/revsets.md** — Complete revset function reference: navigation, lookup, filtering, text matching, date patterns, file-based selection, built-in aliases, and advanced patterns
- **references/templates.md** — Full template language reference: all keywords, type methods (String, List, Timestamp, Signature, TreeDiff, RefName), global functions (formatting, control flow, serialization), built-in named templates, and alias configuration

Load these references when:
- User asks about jj workflows, how to structure their work, or how to navigate a commit stack
- User needs a specific function signature or method they can't find in SKILL.md
- Writing complex revsets with date filters, file predicates, or advanced set operations
- Building custom `jj log` output formats or scripting jj output
- User asks about fileset syntax for `jj diff`, `jj split`, or `jj squash -i`
