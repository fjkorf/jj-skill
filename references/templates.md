# Template Language Reference

Templates are expressions passed to `-T`/`--template` that control output formatting. Used with `jj log`, `jj show`, `jj op log`, and other commands.

## Core Syntax

Templates are a functional expression language. The result is rendered as text.

```bash
jj log -T 'change_id ++ " " ++ description.first_line() ++ "\n"'
jj log -T builtin_log_oneline    # use a built-in named template
```

## Operators (binding strength, highest first)

| Operator | Behavior |
|----------|---------|
| `x.f()` | Method call |
| `-x`, `!x` | Negation (integer / boolean) |
| `*`, `/`, `%` | Arithmetic (integers) |
| `+`, `-` | Addition / subtraction (integers) |
| `>=`, `>`, `<=`, `<` | Comparison (integers) |
| `==`, `!=` | Equality (Boolean, Integer, String) |
| `&&`, `\|\|` | Logical (short-circuit) |
| `x ++ y` | Concatenation (any templates) |

## Keywords (Commit context — `jj log`, `jj show`)

Zero-argument `Commit` methods work directly as keywords:

| Keyword | Type | Description |
|---------|------|-------------|
| `change_id` | ChangeId | The change ID |
| `commit_id` | CommitId | The commit hash |
| `description` | String | Full commit message |
| `author` | Signature | Author name/email/date |
| `committer` | Signature | Committer name/email/date |
| `parents` | List\<Commit\> | Parent commits |
| `bookmarks` | List\<RefName\> | Local bookmarks pointing here |
| `tags` | List\<RefName\> | Tags pointing here |
| `mine` | Boolean | True if authored by current user |
| `empty` | Boolean | True if no file changes |
| `conflict` | Boolean | True if commit has conflicts |
| `divergent` | Boolean | True if change has multiple visible commits |
| `hidden` | Boolean | True if commit is hidden |
| `root` | Boolean | True if root commit |
| `diff` | TreeDiff | File changes in this commit |
| `files` | List\<String\> | Files changed in this commit |
| `signature` | CryptoSignature | Cryptographic signature (if present) |

## Operation Keywords (`jj op log`)

| Keyword | Description |
|---------|-------------|
| `operation_id` | Operation ID |
| `description` | Operation description |
| `time` | Timestamp range |
| `user` | User who ran the operation |
| `tags` | Tags on the operation |
| `snapshot` | Whether it was an auto-snapshot |

## Global Functions

### Text Formatting

```
fill(width, content)                    Word-wrap content to width
indent(prefix, content)                 Prefix every line
pad_start(width, content, [fill])       Right-align (pad on left)
pad_end(width, content, [fill])         Left-align (pad on right)
pad_centered(width, content, [fill])    Center-align
truncate_start(width, content, [ellipsis])  Trim from start
truncate_end(width, content, [ellipsis])    Trim from end
```

### Control Flow

```
if(condition, then, [else])             Conditional expression
coalesce(content...)                    First non-empty value
concat(content...)                      Concatenate all (including empty)
join(separator, content...)             Join with separator (always)
separate(separator, content...)         Join with separator, skip empty
surround(prefix, suffix, content)       Wrap non-empty content
```

### Serialization

```
stringify(content)                      Convert to plain string (strips labels)
json(value)                             Serialize to JSON string
hash(content)                           Hex hash of content
```

### Utilities

```
label(label, content)                   Apply a color/style label
hyperlink(url, text, [fallback])        Terminal hyperlink
raw_escape_sequence(content)            Emit raw ANSI escape sequences
config(name)                            Read a config value → Option<ConfigValue>
git_web_url([remote])                   Convert commit ref to web URL
```

## Type Methods

### String

```
s.len()                      Length in bytes
s.contains(substr)           True if substr is found
s.match(pattern)             True if pattern matches (glob by default)
s.replace(from, to)          Replace all occurrences
s.lines()                    Split into list of lines
s.split(separator)           Split by separator
s.upper()                    Uppercase
s.lower()                    Lowercase
s.starts_with(prefix)        True if starts with prefix
s.ends_with(suffix)          True if ends with suffix
s.trim()                     Strip leading/trailing whitespace
s.trim_start()               Strip leading whitespace
s.trim_end()                 Strip trailing whitespace
s.substr(start, end)         Substring by byte offset
s.escape_json()              JSON-escape the string
s.first_line()               First line only (same as lines().first())
```

### Integer

```
n.abs()                      Absolute value
```

### List

```
list.len()                   Number of items
list.join(separator)         Join items as strings
list.filter(|x| expr)        Filter items matching predicate
list.map(|x| expr)           Transform each item
list.any(|x| expr)           True if any item matches predicate
list.all(|x| expr)           True if all items match predicate
list.first()                 First element (or empty)
list.last()                  Last element (or empty)
```

### Timestamp

```
ts.ago()                     Human-readable relative time ("5 minutes ago")
ts.format(fmt)               Custom format string (strftime-style)
ts.utc()                     Convert to UTC
ts.local()                   Convert to local timezone
ts.after(date)               True if after date
ts.before(date)              True if before date
```

### Signature (author / committer)

```
sig.name()                   Name string
sig.email()                  Email string
sig.timestamp()              Timestamp
```

### ChangeId / CommitId

```
id.short([len])              Short prefix (default 12 chars)
id.shortest([min_len])       Shortest unique prefix
id.normal_hex()              Full hex representation
```

### TreeDiff

```
diff.files()                 List of changed file paths
diff.color_words()           Word-level colored diff
diff.git()                   Git-style unified diff
diff.stat()                  Diff statistics (files changed, +/-)
diff.summary()               Short summary (A/M/D per file)
```

### RefName (bookmarks, tags)

```
ref.name()                   Name string
ref.remote()                 Remote name (if remote ref)
ref.present()                True if the ref exists
ref.conflict()               True if bookmark has conflicts
ref.normal_target()          The target commit (if not conflicted)
ref.removed_targets()        Commits removed in conflict
ref.added_targets()          Commits added in conflict
ref.tracking_ahead_count()   How many commits ahead of remote
ref.tracking_behind_count()  How many commits behind remote
```

## Built-in Named Templates

Use with `-T <name>`:

| Name | Description |
|------|-------------|
| `builtin_log_oneline` | Compact one-line format |
| `builtin_log_compact` | Default compact log |
| `builtin_log_detailed` | Full detail log |
| `builtin_op_log_oneline` | Operation log one-liner |
| `builtin_op_log_compact` | Default operation log |
| `builtin_op_log_detailed` | Full operation log |

## Template Aliases

Define reusable template snippets in `~/.config/jj/config.toml`:

```toml
[template-aliases]
'my_short_id' = 'change_id.short(8)'
'format_field(key, value)' = 'key ++ ": " ++ value ++ "\n"'
'format_author(sig)' = 'sig.name() ++ " <" ++ sig.email() ++ ">"'
```

Aliases can be overloaded by parameter count.

## Usage Examples

### Basic formatting

```bash
# One commit ID per line
jj log -T 'commit_id ++ "\n"'

# Change ID + short description
jj log -T 'change_id.short() ++ " " ++ description.first_line() ++ "\n"'

# Author name, date, and description
jj log -T 'author.name() ++ " " ++ author.timestamp().ago() ++ " " ++ description.first_line() ++ "\n"'
```

### Conditional output

```bash
# Show conflict marker
jj log -T 'change_id.short() ++ " " ++ if(conflict, "CONFLICT ", "") ++ description.first_line() ++ "\n"'

# Show empty marker
jj log -T 'if(empty, "(empty) ") ++ description.first_line() ++ "\n"'

# Show mine indicator
jj log -T 'if(mine, "* ", "  ") ++ change_id.short() ++ " " ++ description.first_line() ++ "\n"'
```

### Working with parents

```bash
# Show parent commit IDs
jj log -r @ -T 'parents.map(|c| c.commit_id().short()).join(", ") ++ "\n"'

# Show parent descriptions
jj log -r @ -T 'parents.map(|c| c.description().first_line()).join(" | ") ++ "\n"'
```

### Working with bookmarks

```bash
# Show bookmarks on each commit
jj log -T 'change_id.short() ++ " " ++ bookmarks.join(", ") ++ " " ++ description.first_line() ++ "\n"'

# Show only commits with bookmarks
jj log -T 'if(bookmarks, change_id.short() ++ " " ++ bookmarks.join(", ") ++ "\n")'
```

### Diff info

```bash
# Show stat for each commit
jj log -T 'description.first_line() ++ "\n" ++ diff.stat() ++ "\n"'

# List changed files per commit
jj log -T 'description.first_line() ++ "\n" ++ files.map(|f| "  " ++ f).join("\n") ++ "\n"'
```

### Machine-readable output

```bash
# Tab-separated: commit_id, change_id, author, description
jj log -T 'commit_id ++ "\t" ++ change_id ++ "\t" ++ author.email() ++ "\t" ++ description.first_line() ++ "\n"'

# JSON-like output
jj log -T '"{\"id\":\"" ++ commit_id.short() ++ "\",\"desc\":\"" ++ description.first_line().escape_json() ++ "\"}\n"'
```

### Timestamp formatting

```bash
# Show ISO date
jj log -T 'author.timestamp().format("%Y-%m-%d") ++ " " ++ description.first_line() ++ "\n"'

# Show relative time
jj log -T 'author.timestamp().ago() ++ " " ++ description.first_line() ++ "\n"'
```

### Text alignment

```bash
# Right-align change IDs in a fixed column
jj log -T 'pad_start(12, change_id.short()) ++ " " ++ description.first_line() ++ "\n"'

# Truncate long descriptions
jj log -T 'change_id.short() ++ " " ++ truncate_end(60, description.first_line()) ++ "\n"'
```

### Separating non-empty values

```bash
# Bookmarks and tags, separated by space, only when non-empty
jj log -T 'separate(" ", bookmarks, tags) ++ "\n"'

# Surround bookmarks with brackets if present
jj log -T 'surround("[", "] ", bookmarks.join(", ")) ++ description.first_line() ++ "\n"'
```
