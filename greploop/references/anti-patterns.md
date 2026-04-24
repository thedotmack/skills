<!--
  Mirror of check-pr/references/anti-patterns.md. The canonical copy lives in
  check-pr/; this file is duplicated (not symlinked) to match the repo's
  existing convention for per-skill references. Keep both copies in sync on
  edit.
-->

# Anti-pattern Scan

Grep patterns run between fix application and thread resolution. A match blocks the commit/push step and sends the skill back into the fix loop with the offending output.

## Staged-diff grep (pre-commit)

```bash
# Debug prints / stray TODOs / merge conflict markers
git diff --cached -U0 | \
  grep -nE 'console\.(log|debug)|print\(|dbg!|fmt\.Println|TODO|FIXME|XXX|<<<<<<<|=======|>>>>>>>' \
  && { echo "anti-pattern: debug output or conflict marker in staged diff"; exit 1; } \
  || echo "clean: debug/markers"

# Likely secrets
git diff --cached | \
  grep -nEi 'api[_-]?key\s*=|secret\s*=|password\s*=|-----BEGIN (RSA|EC|OPENSSH) PRIVATE KEY-----' \
  && { echo "anti-pattern: possible secret in staged diff"; exit 1; } \
  || echo "clean: secrets"
```

## Working-tree grep (pre-stage, Perforce)

Perforce has no staging area — run the same patterns against `p4 diff`:

```bash
p4 diff | grep -nE 'console\.(log|debug)|print\(|dbg!|fmt\.Println|TODO|FIXME|XXX|<<<<<<<' \
  || echo "clean"
```

## Language-specific sweeps (run only if the language is present)

- TypeScript/JavaScript: `grep -nE '\bany\b|@ts-ignore|@ts-expect-error' -- <changed *.ts/*.tsx>`
- Python: `grep -nE 'except\s*:|except\s+Exception\s*:\s*pass' -- <changed *.py>`
- Rust: `grep -nE '\.unwrap\(\)|\.expect\("[^"]*"\)' -- <changed *.rs>`
- Go: `grep -nE 'panic\(|_ = err' -- <changed *.go>`

## Project linter / typechecker

If the repo has a configured linter or typechecker, run it against the changed files:

- `package.json` scripts `lint`, `typecheck`, or `test` → `npm run <script>`
- `pyproject.toml` with `ruff` or `mypy` → run directly
- `Cargo.toml` → `cargo clippy -- -D warnings`

Only a *new* error on changed lines is a blocker; pre-existing baseline errors are out of scope.

## Exit contract

- All greps return `clean` (or no match) **AND** linter exit code is 0 on changed lines → proceed to resolve threads.
- Any failure → emit the offending output, return to the Fixer with the failure as additional context, do not resolve threads, do not push beyond the current branch state.
