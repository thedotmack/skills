# Subagent Contracts

Structured input/output schemas for subagents invoked by `check-pr` and `greploop`. These contracts exist so the orchestrator can enforce evidence-bearing responses rather than free-form prose.

## Fixer subagent

Used when the skill has decided to address actionable review comments.

### Input

The orchestrator passes:

- `comments`: an array of comment objects the subagent must address. Each:
  - `id`: the platform-native thread/discussion/comment identifier (string)
  - `source`: `inline` | `general` | `description`
  - `author`: login/username of the comment author
  - `file`: path relative to repo root (null for general comments)
  - `line`: 1-based line number in the new file (null for general)
  - `body`: verbatim comment text
- `vcs`: `github` | `gitlab` | `perforce`
- `working_branch` or `changelist`: the branch (git) or CL number (p4) where edits land

### Output

The subagent must return YAML matching this schema, and nothing else:

```yaml
fixes:
  - comment_id: <id from input>
    file: <path>
    line: <int>
    action: edit | noop | false_positive
    diff: |
      <unified diff snippet, ≤ 20 lines>
    rationale: <one sentence explaining the fix or why no change>
```

### Orchestrator verification (mechanical, no LLM re-judgment)

For each entry with `action: edit`:

1. `git diff -- <file>` (or `p4 diff <file>`) must be non-empty.
2. At least one changed hunk must intersect `[line - 20, line + 20]`.
3. The union of `fixes[*].file` must equal `git diff --name-only`.

If any check fails: reject the fixer's report and re-deploy with the failures listed.

### Anti-patterns the Fixer must not emit

- Edits to files outside `fixes[*].file`.
- Adding `console.log`, `print(`, `dbg!`, `TODO`, `FIXME`, `XXX`.
- Using `--no-verify`, `--force`, or bypassing pre-commit hooks.
- Renaming variables without addressing the substance of the comment.
- Adding new dependencies to satisfy a single review comment.

## Parallelism thresholds

- **≤ 3 actionable comments**: orchestrator fixes inline; do not spawn a Fixer subagent (overhead > benefit).
- **4–10 comments**: one Fixer subagent with the full set.
- **> 10 comments AND disjoint file clusters**: optional patch-producer pattern — spawn one Fixer per file cluster, each returns a unified diff only, orchestrator applies diffs serially with `git apply`. Never run concurrent fixers that write directly to the working tree.
