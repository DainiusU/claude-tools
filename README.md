# Claude Tools

A collection of Claude Code plugins for development workflows.

## Plugins

### deep-review

Comprehensive PR code review using an Opus orchestrator and 6 parallel Sonnet specialist agents. Goes beyond basic linting to catch real issues:

- **Jira alignment** — verifies implementation matches ticket description and acceptance criteria
- **Bug hunting** — logic errors, security issues, edge cases with semantic code understanding
- **Architecture & patterns** — flags framework misuse, NIH syndrome, and pattern drift
- **Over-engineering detection** — unnecessary complexity, reinvented wheels, premature abstractions
- **Guidelines compliance** — CLAUDE.md and code comment adherence
- **Historical context** — git blame analysis, previous PR feedback patterns

Features:
- Inline GitHub PR review comments (not a single blob comment)
- Local self-review mode (terminal output, no GitHub posting)
- Smart file triage for large PRs (skip/skim/deep classification)
- Re-review support with incremental diff and automatic thread resolution
- Serena code intelligence integration (with Read/Grep/Glob fallback)
- Confidence-based scoring (80+ threshold) to minimize false positives
- Dedup against existing review comments

### Installation

```bash
/plugin marketplace add DainiusU/claude-tools
/plugin install deep-review@claude-tool
```

### Usage

```bash
/deep-review                     # local review of unstaged/staged changes
/deep-review 123                 # review PR #123 (inline GitHub comments)
/deep-review owner/repo#123      # review PR in different repo
```

### Requirements

- Claude Code with Max plan (uses Opus + Sonnet agents)
- `gh` CLI installed and authenticated
- Optional: Serena MCP for semantic code understanding
- Optional: Atlassian MCP for Jira ticket alignment

## Contributing

Contributions welcome. See [docs/deep-review-design.md](docs/deep-review-design.md) for the design spec.

## License

MIT
