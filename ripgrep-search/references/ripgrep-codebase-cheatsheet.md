# ripgrep Codebase Cheatsheet

Derived from the upstream ripgrep guide:
https://raw.githubusercontent.com/BurntSushi/ripgrep/refs/heads/master/GUIDE.md

Use this as a fast lookup for repository-scale code search and rename planning.

## Baseline Search

```bash
# Recursive search from current directory
rg -n 'Foo'

# Search specific directory
rg -n 'Foo' src

# Literal search (disable regex parsing)
rg -n -F 'foo-service'

# Smart case (case-insensitive unless uppercase appears in pattern)
rg -n -S 'foo'
```

## Scope Control

```bash
# Respect ignore files (default), include hidden files
rg -n --hidden 'Foo'

# Disable ignore + hidden + binary filters quickly
rg -n -uuu 'Foo'

# Restrict by glob
rg -n 'Foo' -g '*.{ts,tsx,js,jsx}'
rg -n 'Foo' -g '!dist/**' -g '!coverage/**'

# Restrict by file type
rg -n 'Foo' -t ts -t tsx
rg -n 'Foo' --type-not json

# Inspect built-in types
rg --type-list
```

## Result Shaping

```bash
# File names only
rg -l 'Foo'

# Count matches per file
rg -c 'Foo'

# Whole word boundaries
rg -w 'Foo'

# Show surrounding context
rg -n -C 2 'Foo'
```

## Regex Patterns for Rename Discovery

```bash
# Match Foo and composed PascalCase identifiers like FooService
rg -n --pcre2 '\bFoo(?=[A-Z][A-Za-z0-9_]*)|\bFoo\b'

# Match foo and composed camelCase identifiers like fooService
rg -n --pcre2 '\bfoo(?=[A-Z][A-Za-z0-9_]*)|\bfoo\b'

# Match common path/name variants
rg -n --pcre2 'foo-service|foo_service|foo/service|foo\.service'
```

## Replacements

`rg --replace` (or `-r`) only changes printed output; it never edits files.

```bash
# Output preview only
rg -n 'foo' -r 'bar'
```

For actual file edits, use your patching/edit tooling after collecting targets with `rg -l` or `rg -n`.

## Debug and Reliability

```bash
# Explain why files may be ignored
rg --debug 'Foo'

# Files that rg would search
rg --files

# Disable config if needed
rg --no-config 'Foo'
```

## Suggested Search Sequence

1. Literal anchor (`-F`) for precision.
2. Variant expansion (case and naming conventions).
3. Regex sweep (`--pcre2`) for composed identifiers and path patterns.
4. Scope narrowing with `-g`/`-t` to remove noise.
5. Final stale-reference sweep after edits.
