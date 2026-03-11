# Deep Review — Claude Code Plugin Design Spec

## Overview

A Claude Code plugin that performs comprehensive PR code reviews using an Opus orchestrator and 6 parallel Sonnet specialist agents. Supports both PR review (inline GitHub comments) and local self-review (terminal output).

Replaces the default `code-review` plugin with deeper analysis: Jira alignment, architecture/pattern verification, over-engineering detection, and semantic code understanding via Serena.

## Usage

```
/deep-review                     # local review of unstaged/staged changes (terminal output)
/deep-review 123                 # review PR #123 (inline GitHub comments)
/deep-review owner/repo#123      # review PR in different repo
```

## Plugin Structure

```
deep-review/
  .claude-plugin/plugin.json
  commands/deep-review.md           # orchestrator prompt (Opus)
  agents/
    jira-alignment.md               # Jira ticket vs implementation
    bug-hunter.md                   # logic errors, edge cases, security
    architecture-patterns.md        # framework misuse, NIH, pattern drift
    over-engineering.md             # unnecessary complexity, reinventing wheels
    guidelines-compliance.md        # CLAUDE.md + code comment compliance
    historical-context.md           # git blame, previous PR comments
```

## Architecture: Orchestrator + Specialist Agents

### Why This Approach

- Opus handles triage and judgment (file classification, confidence scoring, dedup) — tasks requiring the best reasoning
- Sonnet agents run in parallel for speed, each focused on a single review dimension
- All review dimensions always run — savings come from file triage (skip/skim/deep), not from skipping agents
- On re-review, the smaller incremental diff naturally reduces agent work

---

## Orchestrator Flow (Opus)

The orchestrator is the main command (`deep-review.md`). It runs these steps sequentially:

### Step 1 — Detect Mode & Gather Inputs

- **PR mode** (argument provided): fetch diff via `gh pr diff`, PR description, existing review comments, CI status checks
- **Local mode** (no argument): use `git diff` for unstaged or `git diff --cached` for staged changes
- **Jira ticket detection**: extract ticket ID from branch name or PR title (pattern: `PROJ-123`, `PROJ_123`). If not found, ask the user via AskUserQuestion
- **Re-review detection**: search PR reviews via `gh api repos/{owner}/{repo}/pulls/{pr}/reviews` for the signature string `Reviewed with deep-review` in the review body (see Output Format > PR Mode). If found, identify commits after the last review timestamp and scope the diff to only those commits
- **Prerequisites check** (PR mode only): verify `gh auth status` succeeds before proceeding. If not authenticated, exit with a clear error message.

### Step 2 — Fetch Context

| Source | How | Purpose |
|--------|-----|---------|
| Jira ticket | Atlassian MCP (`getJiraIssue`) | Description + acceptance criteria |
| CLAUDE.md files | Glob for `**/CLAUDE.md` in modified directories + root | Guidelines |
| Existing review comments | `gh api repos/{owner}/{repo}/pulls/{pr}/comments` | Dedup — "already flagged, don't repeat" |
| PR description | `gh pr view` | Author's intent |
| CI status checks | `gh pr view --json statusCheckRollup` | Note failures, skip what CI catches |
| Previous deep-review findings | Search PR reviews for signature | Re-review: track resolved/unresolved |
| Serena project memories | `list_memories` / `read_memory` (if Serena available) | Project-specific conventions beyond CLAUDE.md |

### Step 3 — File Triage

Classify every changed file into:

| Level | Criteria | Agent Treatment |
|-------|----------|-----------------|
| **Skip** | Auto-generated, lock files, vendored deps, pure renames, binary/asset files, files with >5000 lines of individual diff | Excluded from agent context |
| **Skim** | Config changes, simple additions, test data, migrations, documentation | Agents receive diff only, no deep reading |
| **Deep review** | New files with logic, complex modifications, core business logic, files touching security/auth | Agents read surrounding code via Serena/Read |

On re-review with small incremental diffs (< 5 files, < 200 lines), skip triage and mark everything as deep review.

### Step 4 — Build Context Package

A structured payload sent to each agent:

```
context_package:
  mode: pr | local
  diff: <filtered by triage level>
  jira:
    ticket_id: PROJ-123
    summary: "..."
    acceptance_criteria: ["AC-1: ...", "AC-2: ...", ...]
  claude_md: <concatenated relevant CLAUDE.md contents>
  existing_comments: <list of already-posted review comments>
  file_triage: <map of file -> skip|skim|deep>
  ci_status: <passing|failing|pending, with failure details if any>
  previous_findings: <list of findings from last review, if re-review>
  serena_available: true|false
  serena_memories: <project conventions/patterns from Serena, if available>
  is_rereview: true|false
  pr_description: "..."
```

### Step 5 — Dispatch 6 Parallel Sonnet Agents

Use the Agent tool to launch all 6 review agents simultaneously in parallel. Each agent receives the context package embedded as a YAML code block in its prompt and returns findings in the standardized format (see Agent Output Format below).

### Step 6 — Synthesize Results

The orchestrator (Opus) processes all agent findings:

1. **Deduplicate** — same file+line range, same issue rephrased differently by multiple agents
2. **Re-score confidence** — orchestrator assigns final 0-100 score with full cross-agent context
3. **Filter** — remove anything below 80 confidence
4. **Dedup against existing comments** — compare remaining findings against existing PR review comments to avoid repeating what's already been said
5. **Re-review reconciliation** — if re-review, compare against previous findings:
   - Mark previously flagged issues as resolved/still present/partially addressed
   - Auto-resolve GitHub review threads for confirmed-resolved issues (GraphQL mutation). Only resolve threads where: (a) the first comment's `author.login` matches the current `gh api user` login, and (b) the comment body contains the deep-review category tag format `**[Category]**`

### Step 7 — Output

- **PR mode**: post inline review comments + summary comment via `gh api`
- **Local mode**: print structured report to terminal

---

## Review Agents (Sonnet)

All agents share these rules:
- Concise, professional tone. No emojis. Write like a senior engineer.
- Only flag issues in changed code — pre-existing issues are not your problem
- Don't flag what linters/CI would catch
- Use Serena tools when available, fall back to Read/Grep/Glob
- Respect file triage classifications (skip/skim/deep)
- On re-review, don't re-flag issues listed in `existing_comments` or `previous_findings`

### Agent Output Format

Every agent returns findings in this structure:

```yaml
findings:
  - file: path/to/file.py
    lines: 42-48
    severity: critical | important | suggestion
    confidence: <0-100, agent's initial confidence estimate>
    category: bug | architecture | jira | overengineering | guidelines | historical
    description: <concise description>
    evidence: <why this is real — code snippet, CLAUDE.md quote, Jira AC, pattern example>
```

The orchestrator re-scores confidence during synthesis (Step 6) with cross-agent context. Agent confidence scores are inputs to this process, not final.

### Agent Frontmatter Template

Each agent markdown file uses this frontmatter format:

```yaml
---
name: <agent-name>
description: <one-line description for orchestrator reference>
model: sonnet
---
```

### Agent 1 — Jira Alignment

**Purpose**: Verify the implementation matches the Jira ticket.

**Process**:
1. Read the Jira ticket description and each acceptance criterion
2. Map each AC to the changed files — is it implemented?
3. Flag: missing ACs, partial implementations, scope creep (significant work not described in the ticket)

**Does NOT flag**: minor refactoring alongside the feature, reasonable code cleanup in touched files.

### Agent 2 — Bug Hunter

**Purpose**: Find logic errors, security issues, and edge cases in changed code.

**Process**:
1. Read the diff carefully
2. For deep-review files, use Serena (`find_symbol`, `find_referencing_symbols`) to understand how changed functions are called and what types flow through them
3. Flag: null/undefined handling, race conditions, resource leaks, SQL injection, XSS, auth bypasses, off-by-one errors, unhandled error paths

**Does NOT flag**: theoretical issues that can't happen given the surrounding code, style issues, missing tests.

### Agent 3 — Architecture & Patterns

**Purpose**: Ensure the PR follows established codebase patterns and uses existing frameworks/utilities.

**Process**:
1. Identify what the changed code does (new endpoint? new service? new utility?)
2. Use Serena/Grep to find how similar things are done elsewhere in the codebase
3. Flag: using raw implementations when a framework is available and used elsewhere (e.g., raw SQL when ORM is standard, manual HTTP when an API client exists, hand-rolled retry logic when tenacity/backoff is used), incorrect layer placement (business logic in routers, DB queries in API layer), pattern divergence from established conventions

**Does NOT flag**: subjective structural preferences, one-off deviations with good reason.

### Agent 4 — Over-Engineering

**Purpose**: Detect unnecessary complexity and reinvented wheels.

**Process**:
1. Assess the complexity of the change relative to what it accomplishes
2. Look for: abstractions with only one consumer, configuration for things that don't need to be configurable, hand-rolled implementations of what a well-known library does, god functions/classes, premature generalization
3. Use Serena/Grep to check if the codebase already has utilities that do what the new code does

**Does NOT flag**: justified complexity (genuinely complex domain), reasonable abstractions that serve current needs.

**Key heuristic**: If the same outcome could be achieved with significantly fewer lines using existing tools/libraries/patterns in the codebase, flag it.

### Agent 5 — Guidelines Compliance

**Purpose**: Verify adherence to CLAUDE.md and code comment guidance.

**Process**:
1. Read all relevant CLAUDE.md files
2. Check each changed file against applicable rules (import patterns, naming conventions, error handling, type hints, etc.)
3. Read code comments in modified files — check if the changes violate guidance in those comments
4. For CLAUDE.md findings, quote the specific rule

**Does NOT flag**: rules that are about Claude's behavior (not code quality), issues silenced by lint-ignore comments, issues that linters/formatters enforce.

### Agent 6 — Historical Context

**Purpose**: Use git history to identify risky changes and check previous feedback.

**Process**:
1. Run `git blame` on modified files to understand the history of changed lines
2. Check if the change reverts or conflicts with recent intentional work
3. Use `gh api` to find previous PRs that touched these files — read their review comments for recurring feedback
4. Flag: changes that undo recent fixes, repeated issues that were flagged in previous PRs on the same files, modifications to code with "do not modify" or "careful" comments in git history

**Does NOT flag**: normal code evolution, intentional refactoring of old patterns.

---

## Output Format

### PR Mode — Inline Review Comments

Each finding becomes an inline comment on the specific file+line in the PR diff via GitHub's review API.

**Individual inline comment format:**

```
**[Category]** Description.

Evidence: explanation with code references or quotes.
Confidence: N
```

Example:
```
**[Bug]** Unhandled null return from `get_document()` — if the document is deleted
between the existence check (line 42) and this fetch, this raises an unhandled
AttributeError.

Evidence: `get_document` returns `Optional[Document]` per storage_service.py:89,
but no None check before accessing `.metadata` on line 51.
Confidence: 92
```

**Summary review comment (posted once):**

```markdown
### Code Review

Reviewed N files (X deep, Y skimmed, Z skipped).
Jira: PROJ-456 — M/N acceptance criteria addressed[, K missing (see inline)].

[CI: 2 checks failing — test-backend, lint. Not duplicating CI findings.]

**X critical, Y important issues found.**

Critical:
1. Brief description — file.py:84 [link]
2. Brief description — router.py:12 [link]

Important:
3. Brief description — retry.py:30 [link]

---
Reviewed with deep-review · React with :+1: if useful, :-1: if not
```

**Re-review summary (appended on subsequent reviews):**

```markdown
### Re-review (commits abc123..def456)

Previously flagged: 5 issues
- 3 resolved (threads auto-resolved)
- 1 still present (see inline comment)
- 1 partially addressed (see inline comment)

New issues found: 1
1. Brief description — file.py:92 [link]
```

### Local Mode — Terminal Output

```
## Review Summary

Reviewing 12 changed files against PROJ-456.

### Critical (2)
1. [Bug] backend/services/doc_service.py:84 — Unhandled null return from get_document()
2. [Architecture] backend/api/export/router.py:12-145 — Raw HTTP client; use existing ApiClient

### Important (1)
3. [Over-engineering] backend/utils/retry.py — Custom retry logic; tenacity already used elsewhere

### Jira Alignment
- 3/4 acceptance criteria covered
- Missing: bulk export (AC-4)

### No Issues
No guidelines or historical context issues found.
```

---

## GitHub Interaction Details

### Posting Inline Review Comments

Use `gh api` to create a pull request review with inline comments:

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/reviews \
  -X POST \
  -f commit_id=FULL_SHA \
  -f event=COMMENT \
  -f body="Summary comment" \
  -f 'comments[0][path]=file.py' \
  -f 'comments[0][line]=84' \
  -f 'comments[0][body]=Inline comment text'
```

All inline comments are submitted as a single review (not individual comments) for a clean notification experience.

### Link Format

```
https://github.com/{owner}/{repo}/blob/{FULL_SHA}/{path}#L{start}-L{end}
```

- Must use full 40-character SHA (not abbreviated)
- Must include `#L` line range notation
- Provide 1+ lines of context before and after

### Resolving Review Threads (Re-review)

Fetch unresolved threads:
```graphql
query($cursor: String) {
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: PR_NUMBER) {
      reviewThreads(first: 100, after: $cursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          isResolved
          comments(first: 1) {
            nodes { body path author { login } }
          }
        }
      }
    }
  }
}
```

Batch-resolve confirmed-fixed threads:
```graphql
mutation {
  t1: resolveReviewThread(input: {threadId: "ID_1"}) { thread { isResolved } }
  t2: resolveReviewThread(input: {threadId: "ID_2"}) { thread { isResolved } }
}
```

Only resolve threads that the re-review confirms are fixed. Never resolve threads from other reviewers.

---

## Tool Strategy

### Serena (preferred when available)

| Tool | Use Case |
|------|----------|
| `get_symbols_overview` | Understand file structure before deep review |
| `find_symbol` with `include_body: true` | Read specific functions/classes in surrounding code |
| `find_referencing_symbols` | Impact analysis — how is the modified function used? |
| `search_for_pattern` | Pattern consistency checks across codebase |
| `list_memories` / `read_memory` | Project-specific conventions, patterns, and decisions stored in Serena |

### Fallback (when Serena not installed)

| Tool | Use Case |
|------|----------|
| `Read` | Read files |
| `Grep` | Search for patterns |
| `Glob` | Find files |

### GitHub

| Tool | Use Case |
|------|----------|
| `gh pr diff` | Get PR diff |
| `gh pr view` | PR metadata, description, CI status |
| `gh api .../pulls/{pr}/reviews` | Post inline review |
| `gh api .../pulls/{pr}/comments` | Read existing comments |
| `gh api graphql` | Fetch/resolve review threads |

### Jira

| Tool | Use Case |
|------|----------|
| Atlassian MCP `getJiraIssue` | Fetch ticket description + acceptance criteria |

### Git

| Tool | Use Case |
|------|----------|
| `git diff` / `git diff --cached` | Local mode diffs |
| `git log` / `git blame` | Historical context |
| `git rev-parse HEAD` | Full SHA for links |

---

## Model Assignment

| Component | Model | Rationale |
|-----------|-------|-----------|
| Orchestrator | Opus | Triage, scoring, dedup, synthesis — needs best judgment |
| All 6 review agents | Sonnet | Focused domain review — good balance of capability and speed |

No Haiku anywhere. Minimum model is Sonnet.

---

## Allowed Tools

### Orchestrator Command Frontmatter (deep-review.md)

```yaml
---
description: "Comprehensive code review with 6 specialist agents"
argument-hint: "[PR number or owner/repo#PR]"
allowed-tools: ["Bash", "Read", "Grep", "Glob", "Agent", "AskUserQuestion"]
model: opus
---
```

The orchestrator has unscoped `Bash` access (needs `gh`, `git`, and general shell commands for triage). Atlassian MCP tools are used for Jira when available.

### Review Agents

Agents use scoped Bash access. Their allowed tools are specified in each agent's prompt instructions (agents inherit tool access from the orchestrator's Agent tool dispatch):

```
Read, Grep, Glob, Bash(git blame:*), Bash(git log:*), Bash(git diff:*),
Bash(gh api:*), Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh pr list:*)
```

Plus Serena tools when available. Agents do NOT get the Agent tool (no sub-sub-agents).

---

## Style Guide (All Components)

- Concise, professional tone
- No emojis in output
- Write like a senior engineer reviewing a colleague's PR
- Cite evidence for every finding — code snippets, CLAUDE.md quotes, Jira AC references
- Use `file:line` format for all code references
- In PR mode, all links use full SHA + line range
- Don't hedge or qualify excessively — if confidence is 80+, state the issue directly

---

## False Positive Filters

The following are NOT issues and must be ignored by all agents:

- Pre-existing issues not introduced in the PR
- Issues that linters, type checkers, or CI would catch
- Pedantic nitpicks a senior engineer wouldn't raise
- General code quality issues not explicitly required in CLAUDE.md
- Issues silenced by lint-ignore comments
- Changes in functionality that are intentional and related to the broader change
- Issues on lines the author did not modify
- Subjective style preferences not codified in CLAUDE.md

---

## Edge Cases

### No Jira ticket found
Auto-detect from branch/PR title fails → ask the user. If user says none exists, skip Jira alignment agent (pass empty Jira context, agent returns no findings).

### Local mode without Jira
Jira alignment agent runs but with empty context → returns no findings. All other agents work normally.

### Massive PR (>100 files or >5000 lines)
File triage becomes aggressive — more files classified as skip/skim. Agents focus on deep-review files only. Summary notes how many files were skipped.

### Already reviewed, no new commits
Orchestrator detects no commits after last review → outputs "No new changes since last review" and exits.

### CI checks failing
Note in summary. Don't duplicate CI findings. Agents still review — CI failures and code review findings are orthogonal.

### Atlassian MCP not available
If the Atlassian MCP plugin is not installed, ask the user to paste the Jira ticket description and acceptance criteria manually, or type "skip" to skip Jira alignment entirely.

### gh CLI not authenticated
PR mode requires `gh auth status` to succeed. If not authenticated, exit immediately with: "Error: GitHub CLI not authenticated. Run `gh auth login` first."

### Local mode re-review
Re-review tracking only applies to PR mode (persistent review comments). In local mode there is no review history — every run is a fresh review.

### Serena not available
If Serena tools are not available (not installed in the project), agents fall back to Read/Grep/Glob. The orchestrator sets `serena_available: true|false` in the context package after attempting a Serena tool call at startup. Agents check this flag to decide their tool strategy.
