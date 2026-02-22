---
name: ripgrep-search
description: |
  Repository-wide codebase search and change planning with ripgrep (`rg`). Use when tasks require finding all relevant code paths, impact analysis for features or bugs, gathering context for architecture decisions, or executing safe/context-aware renames/refactors (for example: `Foo` to `Bar`, `FooService` to `BarService`, and import/path updates like `foo-service` to `bar-service`).
---

# Ripgrep Search

Use this skill whenever repository-wide code discovery is required. `rg` is the default first tool for search, rename impact analysis, and architecture exploration.

## Workflow Decision

1. Identify the task type.
   - **Discovery / impact analysis**: follow "Discovery Workflow."
   - **Rename / terminology migration**: follow "Rename Workflow."
   - **Architecture exploration**: follow "Architecture Workflow."
2. Build a term matrix before searching.
3. Search in passes (literal, variant, structural regex).
4. Summarize findings as `path + symbol + why relevant`.
5. If edits are made, verify with a final stale-reference sweep.

## Build a Term Matrix

Given anchor term `Foo`, search multiple identifier and path forms:

- `Foo` (PascalCase)
- `foo` (camelCase root)
- `FOO` (constant/env style)
- `foo_bar` (snake_case)
- `foo-bar` (kebab-case / file names)
- `foo/bar` and `foo.bar` (module and path forms)
- Composed identifiers:
  - `\bFoo(?=[A-Z][A-Za-z0-9_]*)` for `FooService`
  - `\bfoo(?=[A-Z][A-Za-z0-9_]*)` for `fooService`

Use `-S/--smart-case` by default. Use `-i` when casing is unknown or inconsistent.

## Discovery Workflow

1. Start with a literal baseline:

```bash
rg -n --hidden -g '!.git' -S -F 'Foo'
```

2. Expand to a multi-variant pass:

```bash
rg -n --hidden -g '!.git' --pcre2 'Foo|foo|FOO|foo_bar|foo-bar|foo/bar|foo\.bar'
```

3. Narrow scope as needed:

```bash
rg -n 'Foo' src
rg -n 'Foo' -g '*.{ts,tsx,js,jsx}'
rg -n 'Foo' -g '!dist/**' -g '!coverage/**'
rg -n 'Foo' -t ts -t tsx
```

4. Search for structurally relevant patterns:

```bash
rg -n --pcre2 '\b(class|interface|type|enum)\s+.*Foo'
rg -n --pcre2 "from ['\"].*foo-service|import .*foo-service"
```

5. Summarize findings by layer (entrypoint, core domain, storage, API, tests, docs/configs).

## Rename Workflow (Context Aware)

Important: ripgrep does not edit files. Use `rg` to locate and verify, then edit with patching tools.

1. Define a rename map across naming conventions.
   Example `Foo` -> `Bar`:
   - `Foo` -> `Bar`
   - `foo` -> `bar`
   - `FOO` -> `BAR`
   - `foo-service` -> `bar-service`
   - `foo_service` -> `bar_service`
   - `foo/service` -> `bar/service`

2. If the term is overloaded (same token, multiple meanings), define a sense profile before any edits:
   - target context keywords (what should be renamed)
   - collision context keywords (what must not be renamed)
   - include/exclude path globs
   - symbol shapes allowed for replacement (for example: `BookReservation`, but not bare `Book`)

3. Find all candidate occurrences:

```bash
rg -n --hidden -g '!.git' --pcre2 'Foo|foo|FOO|foo-service|foo_service|foo/service'
rg -l --hidden -g '!.git' --pcre2 'Foo|foo|FOO|foo-service|foo_service|foo/service'
```

4. Build a target-only allowlist when ambiguity exists. Example for `Book` (reservation sense only):

```bash
# Target sense: reservation/hotel workflow
rg -l --hidden -g '!.git' --pcre2 "(?i)(\\bbook(?:ing|ed|s)?\\b.*\\b(hotel|reservation|room|guest|check[- ]?in)\\b|\\b(hotel|reservation|room|guest|check[- ]?in)\\b.*\\bbook(?:ing|ed|s)?\\b)" > /tmp/book-target.txt

# Collision sense: reading material workflow
rg -l --hidden -g '!.git' --pcre2 "(?i)(\\bbook(?:ing|ed|s)?\\b.*\\b(author|chapter|library|isbn|paperback)\\b|\\b(author|chapter|library|isbn|paperback)\\b.*\\bbook(?:ing|ed|s)?\\b)" > /tmp/book-collision.txt

# Keep only target files that are not collision files
sort -u /tmp/book-target.txt -o /tmp/book-target.txt
sort -u /tmp/book-collision.txt -o /tmp/book-collision.txt
comm -23 /tmp/book-target.txt /tmp/book-collision.txt > /tmp/book-allowlist.txt
```

5. Handle composed identifiers and path-like strings:

```bash
rg -n --pcre2 '\bFoo(?=[A-Z][A-Za-z0-9_]*)|\bfoo(?=[A-Z][A-Za-z0-9_]*)'
rg -n --pcre2 "['\"](?:\\./|\\.\\./)?.*foo[-_][a-z0-9_-]+"
```

For overloaded terms, run these searches only on allowlisted files:

```bash
xargs rg -n --pcre2 '\bBook(?=[A-Z][A-Za-z0-9_]*)|\bbook(?=[A-Z][A-Za-z0-9_]*)' < /tmp/book-allowlist.txt
```

6. Rename filenames when required:

```bash
rg --files | rg 'Foo|foo-service|foo_service'
```

7. Apply edits conservatively (boundary-aware patterns, path/symbol aware edits). For overloaded terms, apply edits only to files in the allowlist.

8. Run stale-reference verification:

```bash
rg -n --hidden -g '!.git' --pcre2 'Foo|foo|FOO|foo-service|foo_service|foo/service'
```

For overloaded terms, run both checks:

```bash
# Old target token should be gone in allowlisted files
xargs rg -n --pcre2 '\bBook(?:ing|ed|s)?\b' < /tmp/book-allowlist.txt

# New token should not appear in collision contexts
rg -n --hidden -g '!.git' --pcre2 "(?i)(\\bBar(?:ing|ed|s)?\\b.*\\b(author|chapter|library|isbn|paperback)\\b|\\b(author|chapter|library|isbn|paperback)\\b.*\\bBar(?:ing|ed|s)?\\b)"
```

9. Run project checks/tests after rename.

Safety rules:

- Prefer boundary-aware regex (`\b`, lookaheads/lookbehinds) over blind substring replacement.
- Never do global replacement on a polysemous root token (for example bare `Book`).
- If a token is overloaded, require: sense profile, file allowlist, and pre-edit candidate review.
- If both senses exist in the same file, restrict edits to explicit symbol/path patterns and avoid replacing the bare token.
- Explicitly decide whether to include docs, changelogs, and generated files.

## Architecture Workflow

Use iterative expansion to avoid missing related code:

1. Start with anchor term matrix.
2. Add neighboring symbols discovered in results (interfaces, events, env keys, routes, adapters).
3. Search usage and definitions separately.
4. Search code and non-code surfaces (configs, CI, Docker, IaC, docs).
5. Produce a dependency map:
   - entry points
   - core modules
   - external integrations
   - tests
   - likely blast radius

## Fast Commands

- List searchable files: `rg --files`
- Show ignored/debug context: `rg --debug 'Foo'`
- Disable filtering quickly: `rg -uuu 'Foo'`
- Literal search: `rg -F 'foo-service'`
- Whole-word search: `rg -w 'Foo'`
- Count matches: `rg -c 'Foo'`

## References

- `references/ripgrep-codebase-cheatsheet.md` for practical flags and patterns
- `references/ripgrep-guide.md` for the full upstream guide
