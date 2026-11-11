---
name: commit
description: Plan logical commit units from all working tree changes (staged, unstaged and untracked), suggest Conventional Commits titles, and create commits only after approval of the plan. Use when the user asks to commit changes or plan/split commits.
---

# Commit Planning and Splitting

When the user wants to commit changes, plan logical units from the **working tree changes** (staged, unstaged and untracked), suggest titles, and, **only after approval**, create commits in dependency order.

## Plan only until approved

- **On the commit/plan command**: If scope rules must be clarified, ask only for that clarification and stop. Otherwise, return **only the plan** (units, titles, files, footer). Do **not** run `git add` or `git commit`. Stop after showing the plan.
- **Commits require approval**: Run `git add` / `git commit` **only** when the user has sent an **approval signal** in a later message.
- **Approval signals**: Treat as approval and then create the commits when the user says things like: "ok", "please commit", "go ahead", "approved", "yes", "do it", "execute", or equivalent.
- **Flow**: First reply = scope clarification when required, otherwise plan + "Reply with 'ok' or 'please commit' to create these commits." After any required clarification, write the user's scope rules to the `## Commit Rules` block in `AGENT.md`, show the plan, and wait for a separate approval signal. After the user replies with an approval signal, create the commits and report.

## When to Apply

- User asks to "commit", "plan commits", or "split into logical commits"
- User provides a message footer (e.g. `refs: EPS-811, #1327`) to append to every commit
- User has uncommitted changes and wants to commit them (or split them into multiple commits)

## Workflow

### Phase A: Plan (run on command; then stop)

#### 1. Inspect changes and context

- Consider all working tree changes regardless of staging. If the user names specific files, limit the plan to those.
- Run `git status --short` and `git diff HEAD --stat` to see all changes; read the diffs needed to group them.
- From the repository root `AGENT.md`, read only the `## Commit Rules` block (from that heading to the next `##` heading or end of file). Follow the commit convention and `### Scopes` rules in that block; do not read unrelated sections or infer project-specific scopes from memory.
- The expected structure is:

  ```markdown
  ## Commit Rules

  ### Scopes

  - Allowed scopes: ...
  - Omit scope when: ...
  ```

- If `AGENT.md`, `## Commit Rules`, or `### Scopes` is missing, or the scope convention is incomplete, ask the user for the missing rules and stop. After the user answers, create or update that block, include `AGENT.md` in the commit plan, and then continue planning.
- Leave out files that look accidental or local-only (secrets such as `.env`, build output, editor or tool state) and list them as excluded in the plan. If there are no changes, report that and stop.
- A file holding changes for several units can be split with `git add -p` in Phase B; say so in the plan.

#### 2. Plan units and order

Group the changed files into **commit units** in **dependency order** (things that other units depend on come first).

| Priority | Unit type | Typical files |
|----------|-----------|---------------|
| 1 | Shared / foundations | Shared utilities, base classes, constants |
| 2 | Routes / config | API endpoints, routing configuration |
| 3 | Feature A | Controllers/Services, UI components, tests |
| 4 | Feature B | Business logic, UI components, tests (reuse shared from earlier unit) |
| 5 | Docs / tooling | Documentation, README, etc. |

- One logical unit = one commit
- Order so that no commit depends on a later commit
- **UI and its dependencies**: Always commit UI components along with their specific styling and translation files in the same unit.

#### 3. Suggest commit title per unit

- **Title**: Follow the [Conventional Commits 1.0.0](references/conventional-commits-1.0.0.md) specification (e.g. `feat: add validation helper`, `fix: user login error`).
- **Scope**: Use the convention and allowed scope names from `AGENT.md` → `## Commit Rules` → `### Scopes`. If they are not documented, ask the user and record the answer in that block; do not choose or infer scopes. Apply the documented rules for when a scope should be omitted.
- **Body** (optional): Bullet points for non-trivial units
- **Footer**: If the user gave one (e.g. `refs: EPS-811, #1327`), use the same footer on every commit

#### 4. Show the plan, then stop

Output:

- Numbered list of units with **suggested title** and **files**
- Excluded files, if any
- Footer line if provided
- **Stop.** Do not run Phase B. Add: "Reply with 'ok' or 'please commit' to create these commits."

**Do not run `git add` or `git commit` in Phase A.**

---

### Phase B: Execute (run only after user sends an approval signal)

When the user replies with an approval signal ("ok", "please commit", "go ahead", "approved", "yes", "do it", "execute", or equivalent), then:

#### 5. Create commits in order

First run `git reset` (keeps the working tree) so only each unit's files are staged. Then, for each unit, in order:

1. `git add <files>` for that unit only (`git add -p` for files split across units)
2. Build message:
   - First line: suggested title
   - Blank line
   - Optional body (bullets)
   - Blank line (if body present)
   - Footer line if user provided it
3. `git commit -m "<title>" [-m "<body>" -m "<footer>"]`
   Use multiple `-m` for body and footer so they become separate paragraphs.

#### 6. Report

- Print `git log --oneline -<N>` for the new commits
- Confirm working tree is clean or list what's left uncommitted

## Commit message format

Follow the [Conventional Commits 1.0.0](references/conventional-commits-1.0.0.md) specification:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

- **Type**: `feat`, `fix`, `chore`, `docs`, `style`, `refactor`, `perf`, `test`, etc.
- **Title**: One line, imperative, ~50 chars or less.
- **Body**: Optional; use for "why" or a short bullet list.
- **Footer**: If the user provides a footer (e.g., `refs: EPS-811, #1327`), append it to every commit.
- **Attribution**: Commit messages must not include AI `Co-authored-by` trailers (such as Copilot, Claude or ChatGPT). Preserve human co-author trailers.

## Guidelines

- **Plan first, commit after approval**: On the command, output only the plan and ask for approval.
- **Working tree scope**: Plan from staged, unstaged and untracked changes; staging status does not limit the scope. Honor explicit file selections.
- **One commit per unit**: Each unit is one logical change.
- **Dependency order**: Foundations and shared code before features that use them.
- **UI and its dependencies**: Always group UI components with their specific styling and translation files.
- **Same footer everywhere**: If the user specifies a footer, add it to every new commit in this run.
- **Project commit rules**: Read only the `## Commit Rules` block from the repository root `AGENT.md`. If its `### Scopes` rules are absent or incomplete, ask the user and persist their answer in that block before planning.
- **Skip unrelated paths**: Do not commit untracked tooling unless the user asks.
- **Repo root**: Run `git` from the repository root.
