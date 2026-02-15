# go-development

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) for writing production-grade Go code.

Not generic advice — concrete patterns extracted from real codebases, with rationale for every decision.

## What's Inside

- **Error handling** — Typed errors, sentinel errors, Unwrap chains, HTTP status mapping
- **Interface design** — Accept interfaces, return structs, keep interfaces small
- **Testing** — Table-driven tests, test seams via package-level vars, httptest patterns
- **Concurrency** — Bounded parallelism, context-aware cancellation, timeout patterns
- **Resilience** — Circuit breaker, exponential backoff with jitter, retry budgets
- **Tooling** — golangci-lint, gofumpt, Makefile targets, dev tool pinning

## Install

```bash
# Add to your project
npx skills add oss-skills/go-development

# Or copy manually
cp -r . ~/.claude/skills/go-development/
```

## Usage

Once installed, Claude Code automatically activates this skill when you:
- Write or review Go code
- Design interfaces, error types, or package structure
- Set up Go project tooling
- Work on concurrency or resilience patterns

## Structure

```
SKILL.md                          # Main skill instructions
references/
  errors.md                       # Error handling patterns
  interfaces.md                   # Interface design principles
  testing.md                      # Testing strategies and patterns
  concurrency.md                  # Concurrency and parallelism
  resilience.md                   # Circuit breakers, backoff, retries
```

## Credits

Patterns distilled from [gogcli](https://github.com/steipete/gogcli) by [Peter Steinberger](https://github.com/steipete) and [Effective Go](https://go.dev/doc/effective_go).

## License

MIT
