---
name: code-review
description: Reviews the changes for one ticket in a fast-moving, legacy or vibe-coded codebase. Checks that the new code is clean and readable, that comments are short and free of tracker references, that every acceptance criterion of the ticket is met, that fallbacks and error handling exist where needed, that the change introduces no bugs, and that tests cover it when the project already has a test tool. Use when the user asks to review a ticket, a PR or a branch against its ticket, for example "review ticket #123", "code review this PR", "does this branch meet the AC".
---

# Code Review

Review the code that one ticket introduces. The codebase is legacy or vibe-coded, has little structure, and the team moves fast. The review must catch real problems without becoming a bottleneck.

## Principles

- **Review the diff, not the codebase.** Existing code is the baseline. Do not ask for refactors outside the ticket scope. Flag existing code only when the change makes it worse.
- **Few findings, each actionable.** Every finding names a file and line, says what is wrong, and says what to do. If a finding has no clear fix, drop it.
- **Two severities only.** A **blocker** must be fixed before merge. A **suggestion** is optional. When in doubt, it is a suggestion.
- **Match the surrounding style.** Do not impose patterns the project does not use (layers, abstractions, test frameworks, lint rules).
- **Read-only.** Do not edit code, push, or post comments on GitHub unless the user asks.

## Step 1: Get the ticket and the diff

**Ticket.** Find the ticket from what the user gave you:

- GitHub issue number or URL: `gh issue view <n> --json title,body,comments`
- PR number or URL: `gh pr view <n> --json title,body,closingIssuesReferences`, then fetch each linked issue.
- Other trackers (Linear, Jira, monday.com): use the connected MCP tool if one exists.
- Pasted text: use it as is.
- Nothing given: look for an issue number in the branch name or the PR body. If you still find nothing, ask the user once. If there is no ticket, skip the AC check and say so in the report.

**Diff.**

- PR given: `gh pr diff <n>`
- Otherwise: find the default branch (`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`, or `main`), then `git diff $(git merge-base HEAD origin/<default>)` to include committed and uncommitted changes.

Read the full changed files where the diff alone does not give enough context.

## Step 2: Extract the acceptance criteria

- If the ticket has an explicit AC (acceptance criteria) list or checklist, use it word for word.
- If it does not, derive short, testable statements from the description and comments. Mark each one **(inferred)**.
- Include AC changes agreed in the ticket comments.

## Step 3: Review the six areas

### 1. Clean, hygienic, readable code

Check the added and changed lines for:

- Names that do not say what the thing is or does (`data2`, `tmp`, `handleStuff`).
- Debug leftovers: `console.log`, `print`, `debugger`, `dd()`, `var_dump`, temporary flags.
- Dead code: unused variables, imports, functions, unreachable branches, commented-out code.
- Duplication inside the diff that a small local function would remove.
- Functions that do too much to follow in one read. Deep nesting that an early return would flatten.
- Magic numbers or strings that need a named constant.
- Files that do not belong in the change (lockfile churn with no dependency change, generated files, local config, secrets).

### 2. Comments

A comment is allowed only when it explains what the code cannot say by itself: complex logic, a non-obvious reason, a workaround. Flag every added comment that breaks one of these rules:

- **Two lines maximum.**
- **No GitHub references.** No issue or PR numbers (`#123`, `GH-123`, `PR 45`), no GitHub URLs, no commit SHAs, no resource IDs. This is a **blocker**.
- **No references to documentation or tickets** ("see docs", "per the spec", "as discussed in the ticket", links to Notion, Jira or Confluence).
- **No narration** of what the next line obviously does ("increment counter", "call the API").
- **No change history** ("added for the new flow", "fixed bug", "changed by X"). Git records this.
- **No commented-out code.**

Scan the added lines for tracker references before you read them one by one:

```bash
git diff $(git merge-base HEAD origin/<default>) | grep -nE '^\+.*((#|GH-|PR ?)[0-9]+|github\.com|/issues/|/pull/|\b[0-9a-f]{7,40}\b)'
```

Check each match by hand. A hex colour or a hash in test data is not a violation.

### 3. Acceptance criteria

For each AC, decide one status and cite the evidence (`file:line`, or the test that proves it):

- **Met**: the code does everything the criterion asks.
- **Partial**: some of it is missing. State exactly what.
- **Not met**: no code implements it.
- **Cannot verify**: it depends on something outside the diff (config, data, a third-party service). State what the user must check by hand.

Any AC that is partial or not met is a **blocker**. Also flag behaviour the diff adds that no AC asks for, as a suggestion, so the user can confirm it is intended.

### 4. Fallbacks and error handling

Look at the places where the new code can fail: network and API calls, file and database access, parsing of external input, user input, values that can be null or empty, third-party responses.

Flag:

- An operation that can fail with no handling, where a failure would crash the app or corrupt data. **Blocker** on a critical path, otherwise a suggestion.
- Swallowed errors: empty `catch`, `except: pass`, `.catch(() => {})`, errors logged and then ignored when the caller needs to know.
- Silent fallbacks that hide a failure (returning `[]` or a default value when the user should see an error).
- No user feedback when a user-facing action fails.
- Missing timeouts or retries where the project already uses them elsewhere.

Do not ask for `try/catch` around code that cannot fail, or for defensive checks on values the code already guarantees.

### 5. Bugs

Read each changed line and ask what input or state makes it behave wrongly. Trace the new code into its callers and callees, because in code with little structure a change often breaks a distant consumer. Look for:

- Logic errors: inverted or wrong conditions, off-by-one, wrong operator, wrong variable, missing `await` or `return`.
- Edge cases: empty lists, null or undefined, zero, negative numbers, duplicates, very long input, time zones and dates.
- Broken callers: a changed signature, return type, response shape, event name, route or database column that existing code still uses the old way. Search for every usage (`grep -rn <name>`).
- State and concurrency: shared mutable state, race conditions, double submits, stale caches, effects or listeners that never clean up.
- Data: queries that miss a filter (tenant, user, soft delete), migrations that break existing rows, writes that are not atomic when they must be.
- Security: unvalidated input reaching a query, a shell command or HTML; secrets in code; a missing permission check on a new endpoint or action.

A bug is a **blocker** only when you can name the input or steps that trigger it. If you suspect a bug but cannot show how to trigger it, write it as a suggestion and state what to check.

### 6. Tests

First, find out whether the project already has a test tool:

- A test dependency (`jest`, `vitest`, `mocha`, `playwright`, `cypress`, `pytest`, `rspec`, `phpunit`, `go test` and so on) in the package manifest.
- A test script (for example `"test"` in `package.json`) or a test config file.
- Existing test files (`*.test.*`, `*.spec.*`, `test_*.py`, `tests/`, `__tests__/`).

**No test tool installed:** skip this area. Do not ask for tests, because setting up a test tool is out of the ticket scope. Write one line in the report: "No test tool installed. Tests skipped."

**Test tool installed:**

- The change must have at least one test that covers its main behaviour, using the tool and patterns the project already uses. A missing test for new or changed logic is a **blocker**. A change with no logic to test (copy, styles, config) does not need one.
- Check that the tests assert the behaviour the AC describes. A test that only checks the code runs, or mocks away the logic under test, does not count.
- Run the tests that touch the change, or the full suite if it is fast. Report the command and the result. A failing test is a **blocker**.
- If the tool is installed but the area of the code has no tests yet, ask for a test only when it is practical to write one with the existing setup. Otherwise write it as a suggestion.

## Step 4: Report

The report must stay under 600 words. Write it in plain English for an international team: most readers are not native English speakers.

- Short sentences, one idea each. Common words ("use", not "utilise").
- No idioms, slang or jokes.
- Explain the problem and its effect for a human reader first. Put file paths, code names and technical detail in the notes.
- Omit any section that has nothing to say.

```
## Review Round <X>

**Verdict:** APPROVED | CHANGES REQUESTED
**Risk:** LOW | MEDIUM | HIGH

### Changes
1. **[Required]** <What is wrong and why it matters, in one or two plain sentences. What to do.>
2. **[Optional]** <...>

### Notes
- **Acceptance criteria:** <n> of <m> met. <List only the ones that are partial, not met or cannot be verified.>
- **Tests:** <command and result, or "No test tool installed. Tests skipped.">
- `path/file.ts:42`: <technical detail for change 1>
- <Anything a developer needs to act on the changes: edge cases, callers to check, things to verify by hand.>
```

**Round number.** Count the earlier "Review Round" reports for this ticket: in the PR comments (`gh pr view <n> --comments`) or in this conversation. The new report is that count plus one. With no earlier report, it is Round 1. In Round 2 and later, check first whether each earlier required change is fixed. List only what is still open or new.

**Verdict.**

- **APPROVED**: no blockers. Optional changes can still be listed.
- **CHANGES REQUESTED**: at least one blocker. Every blocker is a **[Required]** change. Every suggestion is **[Optional]**.

**Risk.** Rate the risk of merging this change as it is now, not the quality of the code.

- **LOW**: small, isolated change. Easy to undo. Does not touch data, money, authentication or shared code. Example: a text change, a style fix, a new field on one screen.
- **MEDIUM**: touches shared code with several callers, a user flow, an external API, or code with no tests around it. A bug would affect users but would be easy to fix.
- **HIGH**: touches authentication, permissions, payments, personal data, database schema or migrations, or data writes that are hard to undo. Or the change is large and has no tests. A bug could lose data, expose data or stop a core flow.

An APPROVED change can still be HIGH risk. Say what to watch after the merge in the notes.

After the report, offer to fix the required changes.
