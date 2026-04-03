# /changelog — AI Decision Log Skill for Claude Code

**AI-generated code has a maintainability problem. The model that wrote it won't remember why it made a choice. This skill fixes that.**

`/changelog` is a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that generates detailed technical decision logs before every commit. It captures:

- **Claude model version** that produced the code
- **Module/dependency versions** assumed at the time
- **Reasoning chain** — why this approach, not another
- **Technical constraints** — undocumented behavior, threading quirks, SDK gotchas
- **"What Will Break"** — concrete warnings for future maintainers

## The Problem

When an AI writes code, the *what* is in the diff. But the *why* is lost the moment the conversation ends.

6 months later, you (or a new Claude session) look at the code and wonder:
- Why was this pattern chosen over the obvious alternative?
- What bug drove this seemingly over-engineered solution?
- What library version constraint made this necessary?
- What will break if I "simplify" this?

Without context, AI assistants will confidently suggest reverting to the approach that already failed.

## The Solution

Run `/changelog` before committing. It generates a structured log that lives in your git repo alongside the code.

```
docs/changelog/
  2026-04-02-rate-limiter-rewrite.md
  2026-04-02-openapi-split-dev-prod.md
  2026-04-15-auth-migration.md
```

Future Claude sessions read these (via CLAUDE.md instructions) before modifying the same files — preventing the same mistakes from being repeated.

## Installation

**1. Copy the skill file:**

```bash
mkdir -p ~/.claude/skills/changelog
cp SKILL.md ~/.claude/skills/changelog/SKILL.md
```

**2. (Optional) Add to CLAUDE.md in your repo:**

```markdown
## Change Documentation
- Before committing significant changes, run `/changelog` to generate a technical log in docs/changelog/
- Future Claude sessions: check docs/changelog/ before modifying files referenced there
```

That's it. The skill is now available in all your projects.

## Usage

```
/changelog rate limiter rewrite
```

Claude will:
1. Read your staged (or unstaged) changes
2. Read relevant dependency files (requirements.txt, package.json, Dockerfile)
3. Generate a structured changelog entry in `docs/changelog/YYYY-MM-DD-short-name.md`
4. Suggest the git commands to commit everything together

## Example Output

See [examples/sample-changelog.md](examples/sample-changelog.md) for a real-world example.

**Key sections in every entry:**

| Section | Purpose |
|---------|---------|
| **Claude Model** | Which model version produced this code |
| **Technical Environment** | Exact dependency versions at the time |
| **Why This Approach** | The reasoning chain, not just the result |
| **Key Technical Constraints** | Undocumented behavior, discovered gotchas |
| **Code Pattern** | The essential snippet future maintainers need |
| **What Will Break If You Change This** | Concrete, specific warnings |

## How It Prevents Hallucinations

Claude Code reads `CLAUDE.md` automatically at session start. When CLAUDE.md says "check docs/changelog/ before modifying these files", future sessions will:

1. Find the changelog entry for the file they're about to modify
2. Read the constraints and warnings
3. Avoid suggesting approaches that were already tried and failed

Without this, a new Claude session has zero memory of prior decisions and may confidently recommend the exact pattern that caused the bug you just fixed.

## Real-World Example

We discovered that Python's `contextvars.ContextVar` doesn't propagate writes from child threads back to the parent async context. Our LLM call counter always read 0.

**Without /changelog:** A future Claude session sees the "simple int + lock" pattern and suggests "upgrading" it to contextvars for "better async safety." The bug returns.

**With /changelog:** The entry explicitly states:

> *"NEVER use contextvars for state shared across LangGraph nodes — writes in child threads don't propagate back to the parent async context. LangGraph runs sync node functions in a thread pool via asyncio.to_thread."*

The future session reads this and avoids the trap.

## Project Structure

```
claude-changelog-skill/
  SKILL.md              # The skill — copy to ~/.claude/skills/changelog/
  README.md             # This file
  examples/
    sample-changelog.md # Real example output
```

## Works With

- Claude Code CLI
- Claude Code VS Code extension
- Claude Code JetBrains extension
- Any project (Python, Node, Go, Rust, etc.)

## License

MIT — use it, share it, modify it.

---

Built by [happydonkey.ai](https://happydonkey.ai) to solve a real problem: maintaining AI-generated code at scale.
