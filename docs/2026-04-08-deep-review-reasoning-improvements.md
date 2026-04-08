# Deep-review plugin: reasoning depth improvements

## Problem

The deep-review plugin's 9 specialist agents catch surface-level issues but miss
findings that require multi-step reasoning. A human reviewer (donst1) found 9
issues on PR #56 of sentinel-crm that the plugin missed entirely, using the same
CLAUDE.md context. The gap is not missing context — it's shallow reasoning.

Analysis of the 9 missed findings reveals 5 abstract reasoning patterns the
agents failed to apply:

1. **Mechanism change analysis**: when code swaps one approach for another
   (ILIKE→FTS, sync→async), enumerate what the old path handled that the new
   one won't.
2. **Cancellation/cleanup tracing**: trace mutable shared state through each
   await/cancellation point — is it consistent when cleanup handlers run?
3. **Runtime-vs-intent verification**: does the code execute the way the author
   assumes? Query planner view vs table pushdown, serialization fidelity,
   transaction scope.
4. **Collection property assumptions**: does the code assume uniqueness,
   ordering, or completeness that the data source doesn't guarantee?
5. **Infrastructure coverage for new paths**: new query patterns need index
   coverage, new test files need markers, new operations need appropriate log
   levels.

Adding specific checks ("check JSON.stringify undefined", "check DROP
EXTENSION scope") is whack-a-mole. The fix must teach agents how to think, not
what to think about.

## Approach

Add abstract reasoning strategies to agent prompts + add an orchestrator
synthesis pass that checks whether agents applied those strategies to high-risk
areas. No structural changes to the 9-agent architecture.

## Changes

### 1. logic-errors.md — reasoning strategies

Add a new section after Process (line 37), before "Does NOT Flag":

```markdown
## Reasoning Strategies

Apply these when the diff contains the corresponding pattern:

**Mechanism changes (A→B swaps):** When code replaces one approach with another
(query strategy, serialization format, sync→async, ORM→raw SQL, etc.), enumerate
concrete inputs the old mechanism handled that the new one won't. Focus on
domain-relevant edge cases, not theoretical ones.

**Cancellation and cleanup tracing:** For async code with gather/wait/create_task,
walk forward through the function: at each `await`, what is the state of every
mutable shared reference (lists, dicts, counters) if CancelledError fires? Does
the cleanup/finally handler see consistent state? Pay special attention to
in-place mutations (`.clear()`, `.pop()`, `del`) that happen before an await.

**Runtime execution vs author intent:** Verify the code executes the way the
author assumes. Does the query planner see the relation the author thinks it
sees (views and CTEs don't inherit base table indexes)? Does the serializer
preserve the properties the code depends on? Does the lock/transaction scope
cover what needs to be atomic?
```

### 2. edge-cases.md — collection property assumptions

Add item 6 to the end of "Process — Systematic Branch Enumeration" (after item
5, line 56):

```markdown
6. **For collections and batches**, check:
   - Does the code assume elements are unique? If the data source provides
     at-least-once delivery, duplicates within a single batch are possible.
   - Does the code assume ordering? Is that guaranteed by the source?
   - If the collection is mutated (cleared, filtered, partitioned) and also
     referenced elsewhere, trace who else holds a reference and what they
     see after mutation.
```

### 3. architecture-patterns.md — scope and structural coverage

Add two items to the flagging list (after "Missing abstraction use", line 41):

```markdown
   - **Unverified assumptions about scope**: when code creates or drops database
     objects, verify the operation's blast radius matches the author's intent.
     Some objects (extensions, roles, event triggers) are database-global, not
     schema-scoped. Migration up/downgrade should not create or destroy objects
     whose lifecycle extends beyond the migration's own scope.
   - **Missing structural support for new code paths**: when the diff introduces
     a new query pattern or data access path, verify it has the structural
     support it needs — index coverage for new WHERE/JOIN clauses, schema
     constraints for new uniqueness assumptions. Existing queries were already
     reviewed; focus on what this diff adds.
```

### 4. guidelines-compliance.md — infrastructure conventions

Extend Process item 1 (line 30, after "architectural constraints, etc."):

```markdown
   Also identify conventions for test infrastructure (markers, fixtures, conftest
   patterns) and operational conventions (log levels, error reporting) that
   CLAUDE.md defines. These are easy to miss because they apply to new files
   rather than modified code — a new test file that lacks a required marker
   violates the convention even though no existing line was changed.
```

### 5. deep-review.md orchestrator — synthesis deep-dive

Add step 6e after 6d (Filter), before dedup-against-existing-comments:

```markdown
### 6e. Orchestrator deep-dive

Before outputting, review the combined agent findings for coverage gaps. You have
the full diff — the agents don't see each other's work, but you see all of it.

1. **Identify high-risk areas**: files with async/concurrent code, database
   migrations, mechanism changes (A→B swaps), and any file flagged by 2+ agents.

2. **Check for reasoning gaps**: for each high-risk area, ask: did any agent
   apply the three core reasoning strategies — mechanism change analysis,
   cancellation/cleanup tracing, runtime-vs-intent verification? If an area
   is high-risk and none of these were applied, do the analysis yourself.

3. **Targeted analysis only**: don't re-review the entire diff. Focus on the
   areas where agent coverage is thin relative to the risk. Add any new findings
   to the combined list with your own confidence scores.

This step exists because individual agents reason in isolation. Cross-cutting
issues — where a mutation in one function affects cleanup in another, or where
a migration's index doesn't match the query that needs it — fall between agents.
```

## What this would have caught on DART-353

| Finding | Which change catches it |
|---|---|
| #1 plainto_tsquery semantics | logic-errors: mechanism change analysis |
| #2 FTS on view won't use index | logic-errors: runtime-vs-intent verification |
| #3 clear() before gather race | logic-errors: cancellation/cleanup tracing |
| #4 In-batch duplicates | edge-cases: collection uniqueness assumption |
| #5 DROP EXTENSION global | architecture: unverified scope assumption |
| #6 Missing index for dedup query | architecture: missing structural support |
| #7 logger.exception level | guidelines: operational conventions |
| #8 JSON.stringify undefined | logic-errors: runtime-vs-intent (serializer fidelity) |
| #9 Missing pytest markers | guidelines: test infrastructure conventions |

## Out of scope

- No agent restructuring (the 9-agent split is fine)
- No model changes (Opus for bug-finding agents, Sonnet for others — already correct)
- No changes to over-engineering, historical-context, jira-alignment, security, or test-coverage agents
