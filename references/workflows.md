# Workflows

jj supports two primary development workflows. Both are valid — choose whichever fits your thinking style, or mix them.

## The Squash Workflow

A describe-first workflow where you name your commit up front, work in a scratch commit, and selectively move changes into the real one. Analogous to Git's staging area, but operating on commits.

### Pattern

```bash
# 1. Describe the work you intend to do
jj describe -m "feat: add user authentication"

# 2. Create a scratch commit on top (your "staging area")
jj new

# 3. Work in the scratch commit — edit files, run tests, iterate
#    Everything you do lands in @ automatically

# 4. Move finished work into the described commit
jj squash                    # move everything into parent
jj squash src/auth.rs        # move a specific file
jj squash -i                 # interactively pick hunks (opens TUI)

# 5. @ is now empty again — continue working and squashing

# 6. When done, discard the empty scratch commit
jj abandon
```

### Key points

- The described commit starts empty and accumulates changes via `jj squash`.
- `jj squash` without arguments moves all changes from `@` into its parent. With file arguments, it moves only those files. With `-i`, it opens an interactive hunk picker.
- `jj squash` is more powerful than `git add` — it can move changes between any commit and its parent, not just the working copy.
- If your scratch commit accumulates junk you don't want, `jj abandon` discards it and creates a fresh empty `@`.
- Multiple squash rounds let you build up a clean commit incrementally.

### When to use

- Exploratory work where you're not sure what the final commit should contain.
- When you want to cherry-pick specific changes out of a messy working state.
- When you want fine-grained control over what goes into each commit (like `git add -p`).

---

## The Edit Workflow

A direct-editing workflow where you work on commits in place. When you discover sub-tasks, you insert new commits before the current one and navigate between them.

### Pattern

```bash
# 1. Start a new feature
jj new -m "feat: refactor auth middleware"

# 2. Work directly in @ — all changes land in this commit

# 3. Realize you need a preparatory change? Insert one before @
jj new -B @ -m "chore: extract helper function"
# → Rebases the original commit on top automatically
# → @ is now at the new (empty) inserted commit

# 4. Do the preparatory work in @

# 5. Return to the original commit
jj next --edit           # move @ to the child commit
# or:
jj edit <change-id>      # jump directly by change ID

# 6. Continue working on the original feature
```

### Key points

- `jj new -B @` inserts an empty commit *before* the current one and rebases all descendants automatically. This always succeeds — conflicts are stored in commits, not blocking.
- `jj edit <rev>` sets `@` to an existing commit. All changes you make update that commit directly (no new commit created).
- `jj next --edit` and `jj prev --edit` move `@` to the child/parent commit for editing. Without `--edit`, they behave like `jj new` (creating a new commit on top of the target).
- Descendants auto-rebase whenever you modify a commit, keeping the stack consistent.
- Change IDs remain stable across all these rebases — only commit hashes change.

### When to use

- When you know what you're building and want to work directly on commits.
- When you discover sub-tasks mid-work and want to insert them into the stack.
- When navigating a stack of commits that need concurrent editing.

---

## Navigation Commands

These commands move `@` through a commit stack. They complement both workflows.

```bash
jj next                  # create new commit on top of @'s child (like jj new on child)
jj next --edit           # move @ to child commit (edit it in place)
jj next 2               # skip ahead 2 commits

jj prev                  # create new commit on top of @'s parent (like jj new on parent)
jj prev --edit           # move @ to parent commit (edit it in place)
jj prev 2               # go back 2 commits

jj edit <rev>            # jump @ directly to any commit by change ID
```

| Command | Creates new commit? | Edits existing commit? |
|---------|:-------------------:|:---------------------:|
| `jj next` | Yes | No |
| `jj next --edit` | No | Yes |
| `jj prev` | Yes | No |
| `jj prev --edit` | No | Yes |
| `jj edit <rev>` | No | Yes |

---

## Combining Workflows

The workflows are not mutually exclusive. Common combinations:

```bash
# Start with edit workflow, use squash for cleanup
jj new -m "feat: big feature"
# ... work directly ...
# Realize some changes belong in a separate commit:
jj new -B @ -m "refactor: extract utility"
jj squash --from <feature-change-id> --into @ -- utils.rs

# Start with squash workflow, use edit to fix up older commits
jj describe -m "feat: new feature"
jj new
# ... work and squash ...
# Spot a typo in an older commit:
jj edit <older-change-id>
# fix the typo — descendants auto-rebase
jj next --edit   # return to where you were
```

---

## Workflow Comparison

| Aspect | Squash | Edit |
|--------|--------|------|
| Mental model | Build up commits incrementally | Work on commits directly |
| Starting point | `jj describe` + `jj new` | `jj new -m` |
| Adding changes | `jj squash` from scratch commit | Edit files in `@` directly |
| Sub-tasks | Create separate described commits | `jj new -B @` to insert before |
| Navigation | Stay in scratch commit | `jj next --edit` / `jj prev --edit` |
| Git analogy | `git add -p` + `git commit --amend` | `git rebase -i` with auto-apply |
| Best for | Exploratory, selective changes | Focused, planned changes |
