---
name: changelog
description: Generate a detailed changelog entry capturing Claude's reasoning, model version, module versions, and technical context for every significant code change. Use before committing.
argument-hint: [short description of change]
allowed-tools: Bash, Read, Grep, Glob, Write
---

# /changelog — Technical Change Log Generator

Generate a detailed changelog entry for the current code changes. This entry will be checked into git alongside the code so future developers and Claude sessions understand exactly why changes were made.

## Steps

1. **Gather change context:**
   - Run `git diff --staged --name-only` to find modified files. If nothing staged, run `git diff --name-only` for unstaged changes.
   - Run `git diff --staged` (or `git diff`) to see the actual changes.
   - Read the key modified files to understand the full context.

2. **Gather technical metadata:**
   - Claude model: Report the model you are running as (from your system context).
   - Read requirements.txt, package.json, or equivalent to capture module versions relevant to the change.
   - Read Dockerfile if present to capture Python/Node version.
   - Note the platform (from system context).

3. **Generate the changelog entry** using the template below. Save to `docs/changelog/YYYY-MM-DD-short-name.md` where short-name is derived from the $ARGUMENTS or inferred from the changes.

4. **Create the docs/changelog/ directory** if it doesn't exist.

5. **Print a summary** of what was created, and suggest the git commands to commit both the code and the changelog entry.

## Template

```markdown
# [Title — from $ARGUMENTS or inferred]

**Date:** [YYYY-MM-DD]
**Claude Model:** [exact model ID from system context]
**Platform:** [OS from system context]
**Trigger:** [what user asked for or what problem was reported]

## What Changed
[Bullet list of files and what changed in each]

## Why This Approach
[The reasoning chain — what was the problem, what options were considered, why this solution]

## Technical Environment
| Component | Version |
|-----------|---------|
| Python / Node | [from Dockerfile or package.json engines] |
| [key framework] | [version from requirements.txt/package.json] |
| [key library] | [version] |
| [other relevant deps] | [version] |

## Key Technical Constraints
[Things that are NOT obvious from the code — threading models, async behavior, SDK quirks, undocumented behavior that was discovered]

## Code Pattern
```[language]
[The core pattern or critical code snippet — not the full file, just the essential logic that future maintainers need to understand]
```

## What Will Break If You Change This
[Specific warnings — if you do X, Y will happen. Be concrete.]

## Files Changed
[Markdown links to each file, relative to repo root]

## Related
[Links to ADRs, issues, other changelog entries, or external docs that informed this decision]
```

## Rules
- Be specific and technical — this is for future Claude sessions and developers, not a PR description
- Include module versions that are RELEVANT to the change, not every dependency
- The "What Will Break" section is the most important — be concrete and specific
- If the change was driven by a bug, describe the bug's root cause precisely
- Don't include secrets, tokens, or credentials
- If $ARGUMENTS is empty, infer the title from the changes
