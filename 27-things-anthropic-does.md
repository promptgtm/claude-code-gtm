# 27 Things Anthropic Does With Claude Code That You Don't

*When Claude Code's entire source code leaked via a sourcemap in npm, it revealed exactly how Anthropic engineers configure their own AI coding tool. This isn't speculation — it's what the code actually shows.*

*Here are 27 configurations, patterns, and workflows you can implement today.*

---

## SECTION 1: MEMORY THAT ACTUALLY WORKS

Most Claude Code users treat every session as a blank slate. Anthropic doesn't.

---

### 1. The 4-Level Memory Hierarchy

Claude Code loads instructions from four levels, each overriding the last:

| Priority | File | Scope |
|----------|------|-------|
| 1 (lowest) | `/etc/claude-code/CLAUDE.md` | Global (all users on machine) |
| 2 | `~/.claude/CLAUDE.md` | Personal (all your projects) |
| 3 | `CLAUDE.md` + `.claude/CLAUDE.md` + `.claude/rules/*.md` | Project |
| 4 (highest) | `.claude/CLAUDE.local.md` | Local (private, not committed) |

**What to do**: Create all four. Put your personal coding preferences in `~/.claude/CLAUDE.md`. Put project conventions in the project-level file. Put sensitive context (internal URLs, team names) in `.claude/CLAUDE.local.md`.

---

### 2. The @include Directive

You can reference external files from any CLAUDE.md:

```markdown
# Project Guidelines

@./docs/coding-standards.md
@./docs/api-conventions.md
@~/shared-rules/security-policy.md
```

This pulls the content of those files into your instructions. Supports relative paths (`@./`), home paths (`@~/`), and absolute paths (`@/`). Works with `.md`, `.txt`, `.json`, `.py`, `.ts`, `.yaml`, and dozens of other text formats.

**What to do**: Break your CLAUDE.md into focused files and compose them with @includes. One file per concern — testing policy, code style, deployment rules.

---

### 3. File-Scoped Rules with .claude/rules/

Instead of one massive CLAUDE.md, you can drop individual rule files into `.claude/rules/`:

```
.claude/rules/
  ├── testing.md        → "Always run pytest after editing test files"
  ├── migrations.md     → "Never auto-run database migrations"
  ├── api-conventions.md → "Use snake_case for all API endpoints"
  └── security.md       → "Never commit .env files"
```

All files in this directory are loaded automatically. No registration needed.

**What to do**: Start splitting your monolithic CLAUDE.md into focused rule files.

---

### 4. Auto-Memory Extraction

Claude Code has a background system that watches your conversations and automatically saves memories to persistent files. It extracts four types:

- **User memories** — your role, expertise, preferences
- **Feedback memories** — corrections you've given ("don't mock the database")
- **Project memories** — deadlines, goals, constraints
- **Reference memories** — pointers to external systems (Linear boards, Grafana dashboards)

Each memory gets its own file with frontmatter metadata:

```markdown
---
name: Testing preferences
description: User prefers integration tests over mocks
type: feedback
---

Integration tests must hit a real database, not mocks.
**Why:** Prior incident where mock/prod divergence masked a broken migration.
**How to apply:** When writing test files, always use the test database helper.
```

**What to do**: Enable auto-memory. When Claude gets something right or wrong, explicitly tell it — "remember this for next time." Check `~/.claude/projects/<your-project>/memory/` to see what it's saved.

---

### 5. The Auto-Dream Consolidation System

Every 24 hours (after at least 5 sessions), a background process called "Auto-Dream" consolidates your memories:

1. **Orient** — reads existing memory files and MEMORY.md index
2. **Gather Signal** — scans recent session transcripts for new insights
3. **Consolidate** — merges new information into topic files, resolves contradictions
4. **Prune** — keeps the index under 200 lines

This is how Anthropic keeps Claude's memory organized without manual curation.

**What to do**: Enable `autoDreamEnabled: true` in your settings. Let it run for a week, then check your memory directory — you'll find organized, deduplicated notes about your project.

---

### 6. The Session Memory Template

When Claude compacts (summarizes) a long conversation, it follows a specific template with 10 sections:

1. Session Title (5-10 words)
2. Current State (what's being worked on, next steps)
3. Task Specification (user request, design decisions)
4. Files and Functions (important files, why they matter)
5. Workflow (bash commands, how to run/interpret output)
6. Errors & Corrections (what broke, how it was fixed)
7. Codebase Documentation (components, architecture)
8. Learnings (what worked, what didn't)
9. Key Results (exact output requested)
10. Worklog (step-by-step terse summary)

Max 2,000 chars per section. 12,000 tokens total. "Current State" is the most critical section — it's what enables continuity after compaction.

**What to do**: When starting complex tasks, front-load context about your current state. Claude's compaction preserves this structure, so the clearer your initial context, the better it survives compression.

---

## SECTION 2: HOOKS THAT AUTOMATE EVERYTHING

Hooks are the most underutilized feature in Claude Code. The source reveals four hook types and 20+ event triggers.

---

### 7. The Four Hook Types

Most users only know about command hooks. The source reveals four:

**Command hooks** — run a shell command:
```json
{
  "type": "command",
  "command": "npm test -- --bail",
  "timeout": 30
}
```

**Prompt hooks** — ask an LLM to evaluate something:
```json
{
  "type": "prompt",
  "prompt": "Evaluate whether this bash command is safe: $ARGUMENTS",
  "model": "claude-haiku-4-5-20251001",
  "timeout": 60
}
```

**HTTP hooks** — call an external API:
```json
{
  "type": "http",
  "url": "https://api.example.com/webhook",
  "headers": {"Authorization": "Bearer $MY_TOKEN"},
  "timeout": 30
}
```

**Agent hooks** — spawn a verification agent:
```json
{
  "type": "agent",
  "prompt": "Verify that all tests pass and no regressions were introduced",
  "model": "claude-haiku-4-5-20251001",
  "timeout": 60
}
```

**What to do**: Start with command hooks for automated testing. Graduate to prompt hooks for intelligent code review gates. Use HTTP hooks to integrate with Slack, Linear, or your CI system.

---

### 8. Event Triggers You Didn't Know Existed

The source reveals 20+ hook events:

| Event | Fires when... |
|-------|---------------|
| `preToolUse` | Before any tool runs (gate dangerous commands) |
| `postToolUse` | After any tool completes (auto-test, auto-lint) |
| `postToolUseFailure` | When a tool fails (alert, retry logic) |
| `sessionStart` | Session begins (load context, check env) |
| `sessionEnd` | Session ends (cleanup, save state) |
| `fileChanged` | A file is modified (trigger tests, linting) |
| `userPromptSubmit` | User sends a message (input validation) |
| `preCompact` | Before context compression (save critical state) |
| `postCompact` | After compression (verify nothing was lost) |
| `taskCreated` / `taskCompleted` | Task lifecycle events |
| `cwdChanged` | Working directory changes |
| `notification` | Custom notification events |
| `permissionRequest` | Permission dialog appears |
| `permissionDenied` | User denies a permission |

**What to do**: Set up `postToolUse` hooks with a `Write` matcher to auto-run tests after file changes. Use `sessionStart` to check that your environment is configured correctly.

---

### 9. Conditional Hooks with Matchers

Hooks support pattern matching to fire only for specific tools:

```json
{
  "hooks": {
    "postToolUse": [
      {
        "matcher": "Write",
        "hooks": [{ "type": "command", "command": "npm run lint -- --fix" }]
      },
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "echo 'Bash command completed'" }]
      }
    ]
  }
}
```

You can also use `if` conditions with permission syntax:

```json
{
  "type": "command",
  "command": "npm test",
  "if": "Bash(git commit*)"
}
```

**What to do**: Create targeted hooks — lint after writes, test after git commits, notify after deployments.

---

### 10. Async Hooks That Don't Block

Add `"async": true` to run hooks in the background without blocking Claude's response:

```json
{
  "type": "command",
  "command": "npm test -- --silent",
  "async": true
}
```

Add `"asyncRewake": true` to wake Claude when the async hook completes (exit code 2 triggers a model response):

```json
{
  "type": "command",
  "command": "run-slow-tests.sh",
  "async": true,
  "asyncRewake": true
}
```

**What to do**: Use async hooks for slow operations (full test suites, deployments) so Claude can continue working while they run.

---

### 11. The One-Shot Hook Pattern

Add `"once": true` to run a hook only once per session:

```json
{
  "type": "command",
  "command": "docker compose up -d",
  "once": true
}
```

**What to do**: Use one-shot hooks for environment setup — start databases, pull latest configs, check dependencies.

---

## SECTION 3: PERMISSIONS THAT ELIMINATE FRICTION

The "approve every command" experience is the #1 complaint about Claude Code. The source reveals how to fix it.

---

### 12. Pattern-Based Permission Rules

Permissions support glob-style pattern matching:

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm test*)",
      "Bash(npm run lint*)",
      "Bash(python -m pytest*)",
      "Bash(ls *)",
      "Bash(cat *)",
      "Bash(grep *)",
      "Bash(find *)",
      "Read(**)",
      "Edit(**)",
      "Write(**/*.test.*)"
    ]
  }
}
```

`*` matches anything within the argument. `**` matches any path. You can be as broad or specific as you want.

**What to do**: Start by allowing read-only commands and git operations. Gradually expand as you build trust. The pattern is: allow safe commands, ask for destructive ones.

---

### 13. The Five Permission Modes

The source reveals five user-facing modes:

| Mode | Behavior | Best for |
|------|----------|----------|
| `default` | Ask for every sensitive operation | Beginners, shared machines |
| `plan` | Show plan first, ask for confirmation | Code review workflows |
| `acceptEdits` | Auto-approve file edits, ask for bash | Active development |
| `dontAsk` | Auto-approve everything | Trusted environments, CI |
| `bypassPermissions` | Skip all checks | Testing only |

**What to do**: Use `acceptEdits` for daily work. Switch to `plan` when doing complex refactors. Use `dontAsk` in CI pipelines with sandboxed environments.

---

### 14. Project-Level Permission Overrides

Put permission rules in `.claude/settings.json` (project root) to share them with your team:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm test*)",
      "Bash(npm run*)",
      "Bash(git status)",
      "Bash(git diff*)",
      "Bash(git log*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(*sudo*)",
      "Bash(git push --force*)"
    ]
  }
}
```

This commits to version control, so every team member gets the same safe defaults.

**What to do**: Create a shared `.claude/settings.json` with your team's safe commands pre-approved and dangerous commands explicitly denied.

---

## SECTION 4: ENVIRONMENT VARIABLES THAT UNLOCK HIDDEN FEATURES

---

### 15. Effort Levels

Control how hard Claude thinks:

```bash
export CLAUDE_CODE_EFFORT_LEVEL=high  # Options: low, medium, high
```

Or in settings: `"effortLevel": "high"`

- **low** — quick responses, minimal reasoning
- **medium** — balanced (default for Opus)
- **high** — deep thinking, thorough analysis

Higher effort = more thinking tokens = better results for complex tasks, but slower and more expensive.

**What to do**: Use `low` for quick questions and file lookups. Use `high` for complex debugging, architecture decisions, and multi-file refactors.

---

### 16. Context Window Control

Override how much context Claude uses before compacting:

```bash
export CLAUDE_CODE_MAX_CONTEXT_TOKENS=800000
export CLAUDE_CODE_MAX_OUTPUT_TOKENS=16000
```

**What to do**: If you're hitting compaction too often on large codebases, increase the context window. If responses are being cut off, increase output tokens.

---

### 17. Subagent Model Override

Force spawned agents to use a specific (cheaper) model:

```bash
export CLAUDE_CODE_SUBAGENT_MODEL=claude-haiku-4-5-20251001
```

**What to do**: Use Opus for your main conversation and Haiku for research subagents. Saves money without sacrificing quality on the main task.

---

### 18. Debug Logging

See exactly what Claude is doing under the hood:

```bash
export CLAUDE_CODE_DEBUG_LOGS_DIR=~/.claude/debug-logs
```

**What to do**: Enable debug logs when troubleshooting unexpected behavior. Check the logs to understand permission decisions, compaction triggers, and memory extraction.

---

### 19. Disable Features You Don't Need

```bash
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1      # No auto-memory
export CLAUDE_CODE_DISABLE_CLAUDE_MDS=1        # No CLAUDE.md loading
export DISABLE_AUTO_COMPACT=1                  # No auto-compaction
export CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1  # No background tasks
```

**What to do**: If you're on a slow machine or want maximum speed, disable features you don't use. Each disabled feature reduces startup time and background CPU.

---

### 20. The Bare Mode Flag

```bash
claude --bare
# or
export CLAUDE_CODE_SIMPLE=1
```

Skips hooks, LSP, plugins, and keychain checks. Pure, fast, minimal Claude Code.

**What to do**: Use bare mode for quick one-off tasks, scripting, or when hooks are causing issues.

---

## SECTION 5: AGENT ORCHESTRATION

---

### 21. Custom Agents in .claude/agents/

Create reusable agent definitions in your project:

```
.claude/agents/
  ├── security-reviewer.md
  ├── test-writer.md
  └── docs-generator.md
```

Each file defines an agent's personality, tools, and constraints. Claude can spawn them by name.

**What to do**: Create agents for your most common workflows — code review, test generation, documentation updates. Reference them with the `agent` setting or spawn them via the Agent tool.

---

### 22. Worktree Isolation for Agents

Agents can work in isolated git worktrees so they don't interfere with your working directory:

```json
{
  "worktree": {
    "symlinkDirectories": ["node_modules", ".venv"],
    "sparsePaths": ["src/", "tests/"]
  }
}
```

`symlinkDirectories` symlinks heavy directories instead of copying them. `sparsePaths` limits which files are checked out (great for monorepos).

**What to do**: Configure worktree settings for your project. Use sparse paths in monorepos to keep agent worktrees fast and focused.

---

### 23. Coordinator Mode (Experimental)

Enable multi-agent coordination where one Claude orchestrates multiple workers:

```bash
export CLAUDE_CODE_COORDINATOR_MODE=1
```

The coordinator constrains worker agents to specific tool subsets and manages a shared scratchpad directory for state passing.

**What to do**: Try this for large refactoring tasks where you want one agent planning and multiple agents executing in parallel.

---

## SECTION 6: UI AND WORKFLOW OPTIMIZATION

---

### 24. Custom Status Line

Show project-relevant info in your status bar:

```json
{
  "statusLine": {
    "type": "command",
    "command": "echo $(git branch --show-current) $(git status --porcelain | wc -l) files changed",
    "padding": 30
  }
}
```

**What to do**: Display your current branch, test status, or deployment state right in the Claude Code UI.

---

### 25. Custom Spinner Verbs

Replace the default "Thinking..." with project-relevant verbs:

```json
{
  "spinnerVerbs": {
    "mode": "append",
    "verbs": ["Deploying", "Refactoring", "Debugging", "Optimizing"]
  }
}
```

Use `"mode": "replace"` to completely override the defaults, or `"append"` to add to them.

**What to do**: A small quality-of-life improvement. Customize the verbs to match your project's domain.

---

### 26. Attribution Control

Control what appears in your commits and PRs:

```json
{
  "attribution": {
    "commit": "",
    "pr": ""
  }
}
```

Set to empty strings to remove all Claude attribution. Or customize:

```json
{
  "attribution": {
    "commit": "Assisted by AI",
    "pr": "This PR was partially generated with AI assistance."
  }
}
```

**What to do**: Decide your team's AI attribution policy and configure it once. No more manually deleting Co-Authored-By lines.

---

### 27. The Complete Power User Settings File

Here's a starting point that combines everything:

```json
{
  "model": "claude-opus-4-6",
  "effortLevel": "medium",
  "fastMode": true,
  "alwaysThinkingEnabled": true,
  "autoDreamEnabled": true,
  "showTurnDuration": true,

  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": [
      "Bash(git status*)",
      "Bash(git diff*)",
      "Bash(git log*)",
      "Bash(git branch*)",
      "Bash(git stash*)",
      "Bash(npm test*)",
      "Bash(npm run lint*)",
      "Bash(npm run build*)",
      "Bash(npx tsc*)",
      "Bash(python -m pytest*)",
      "Bash(ls *)",
      "Bash(cat *)",
      "Bash(find *)",
      "Bash(grep *)",
      "Bash(wc *)",
      "Read(**)",
      "Edit(**)",
      "Glob(**)",
      "Grep(**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(*sudo*)",
      "Bash(git push --force*)",
      "Bash(git reset --hard*)",
      "Bash(*> /dev/null*)"
    ]
  },

  "hooks": {
    "postToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint -- --fix --quiet",
            "timeout": 15,
            "async": true
          }
        ]
      }
    ],
    "sessionStart": [
      {
        "matcher": null,
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Branch:' $(git branch --show-current) '| Status:' $(git status --porcelain | wc -l) 'files modified'",
            "once": true
          }
        ]
      }
    ]
  },

  "attribution": {
    "commit": "Co-Authored-By: Claude <noreply@anthropic.com>",
    "pr": "Generated with [Claude Code](https://claude.ai/code)"
  },

  "worktree": {
    "symlinkDirectories": ["node_modules", ".venv", "vendor"],
    "sparsePaths": []
  },

  "statusLine": {
    "type": "command",
    "command": "echo $(git branch --show-current)",
    "padding": 20
  },

  "env": {
    "CLAUDE_CODE_EFFORT_LEVEL": "medium"
  }
}
```

**What to do**: Copy this to `~/.claude/settings.json`. Adjust the permission rules and hooks for your stack. Iterate.

---

## THE BOTTOM LINE

Anthropic's engineers don't use Claude Code out of the box.

They configure 4-level memory hierarchies. They set up hooks that auto-test, auto-lint, and auto-notify. They fine-tune permissions so Claude can move fast without asking for approval on every `git status`. They use environment variables to control effort, context, and model selection.

The leaked source code didn't just reveal how Claude Code works — it revealed how much of its power is hidden behind configuration that most users never touch.

Now you know. Go configure it.

---

*This guide was extracted from analysis of Claude Code's leaked source (March 31, 2026). All configurations described are available in the current public release of Claude Code.*
