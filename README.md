# go-development

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) for writing Go code that holds up in production.

Covers error handling, interface design, testing, concurrency, resilience, and tooling. Each pattern includes the reasoning behind it, not just the code.

Based on patterns from [gogcli](https://github.com/steipete/gogcli) by [Peter Steinberger](https://github.com/steipete) and [Effective Go](https://go.dev/doc/effective_go).

## Install

```bash
npx skills add oss-skills/go-development
```

Or copy manually:
```bash
cp -r . ~/.claude/skills/go-development/
```

## What's in here

The skill activates when you write or review Go code, design interfaces, or set up project tooling.

```
SKILL.md                          # Main instructions
references/
  errors.md                       # Typed errors, sentinel errors, Unwrap chains
  interfaces.md                   # Accept interfaces, return structs
  testing.md                      # Table-driven tests, test seams, httptest
  concurrency.md                  # Bounded parallelism, context cancellation
  resilience.md                   # Circuit breaker, backoff with jitter
```

## License

MIT
