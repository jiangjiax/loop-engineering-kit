# Bash no-go zone

The `loop-foreman` skill is a **zero-interruption loop** — from user trigger to final report, the user should see no permission prompts. This is the loop's core value: the user can walk away and come back to a finished result.

Certain Bash patterns **trigger Claude Code's permission prompt system** and break this. This file is the authoritative reference for which patterns to avoid and what to use instead.

---

## Forbidden patterns

### Heredoc redirects (`<< 'EOF'`)

```bash
# ❌ Forbidden
cat >> file.md << 'EOF'
new content
EOF
```

**Why**: heredoc + redirect is the canonical "modify file" Bash pattern. Claude Code flags it for permission review.

**Use instead**: Read the file → Edit to append the new block.

```python
# ✅ Correct
Read("file.md")
Edit("file.md", old_string="...last line...", new_string="...last line...\n\nnew content")
```

### In-place stream edit (`sed -i`)

```bash
# ❌ Forbidden
sed -i 's/old/new/g' file.md
```

**Why**: `sed -i` modifies the file in place. Permission system catches it.

**Use instead**: Edit tool with `replace_all: true`.

```python
# ✅ Correct
Edit("file.md", old_string="old", new_string="new", replace_all=True)
```

### Piped sed/grep ranges

```bash
# ❌ Forbidden
sed -n '32,148p' file.md | grep -nE "(pattern1|pattern2)"
```

**Why**: pipe + range stream-edit looks like a complex file mutation pipeline. Permission system flags it.

**Use instead**: Grep tool with `path` + `multiline: true` + `output_mode: "content"` + `-n` parameter.

```python
# ✅ Correct
Grep(
  pattern="(pattern1|pattern2)",
  path="file.md",
  output_mode="content",
  -n=True,
  multiline=True
)
```

If you genuinely need to grep only a line range, Read the file with `offset` + `limit`, then Grep main-thread the returned string.

### awk

```bash
# ❌ Forbidden
awk '/start/,/end/ { print }' file.md
awk '{ sum += $1 } END { print sum }' file.md
```

**Why**: awk is a stream processor. Same category as sed.

**Use instead**: Read file → process in main thread. For aggregation, Grep with `head_limit` or count matches.

### find -exec

```bash
# ❌ Forbidden
find . -name "*.md" -exec wc -l {} \;
find . -name "*.tmp" -exec rm {} \;
```

**Why**: `-exec` runs arbitrary commands on found files. Permission system catches it as batch mutation.

**Use instead**: Glob tool to find files, then loop in main thread with individual tool calls (Read / Edit / Bash `rm` per file).

```python
# ✅ Correct
files = Glob("**/*.md")
for f in files:
    # operate on each file with native tools
    Read(f)
```

### wc with file argument

```bash
# ❌ Often forbidden (depends on settings)
wc -m file.md
wc -l file.md
```

**Why**: `wc` is a "read file metadata" command that sometimes triggers permission review.

**Use instead**: Read the file → count in main thread (single-shot count is fine), or use Grep with `head_limit: 1` for line-count-like signals.

---

## Allowed patterns (no permission prompt)

### `find -type f` for file listing

```bash
# ✅ Allowed
find .claude/skills/MY-SKILL -type f
```

Used before spawning subagents to hardcode the read list. Mechanical, no mutation.

### `cp` for file copy

```bash
# ✅ Allowed
cp source.md target.md
```

Used in Phase 6 archive step. Simple copy doesn't trigger permission review.

### `date` for timestamps

```bash
# ✅ Allowed
date -u +"%Y-%m-%dT%H:%M:%SZ"
```

Used in Phase 0 (loop start) and Phase 6 (loop end) for ISO timestamps.

### `ls` for directory listing

```bash
# ✅ Allowed
ls -la /path/to/dir
```

Used to inspect directory contents. Read-only.

---

## Heuristic: when in doubt, check three things

Before using a Bash command in the loop, ask:

1. **Does it have a pipe (`|`)?** If yes → likely triggers prompt. Find a native tool.
2. **Does it have a redirect (`>>` `>` `<<`)?** If yes → likely triggers prompt. Use Edit / Write.
3. **Does it use a stream editor (`sed -i` `awk`)?** If yes → use Edit / Read + main-thread processing.

If none of those three → probably safe. But test once before relying on it.

---

## Why this exists at all

Claude Code's permission system is calibrated to flag commands that could mutate files in ways the user didn't approve. From a security standpoint, this is good — the loop foreman could otherwise accidentally `rm -rf` your project.

But for a foreman that's running 30+ tool calls across 9 phases, **each prompt is a context-window interruption that breaks the user's "walk away" expectation**. The loop's value is being able to leave it running for 50 minutes and come back to a finished result. One prompt at minute 30 destroys that.

The fix isn't to suppress prompts (dangerous). The fix is to **use Claude Code's native tools** (Read / Edit / Write / Grep / Glob / etc.) which were designed for skill use and don't trigger prompts when invoked correctly.

This file's job is to make that translation explicit.
