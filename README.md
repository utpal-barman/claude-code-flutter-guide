# Claude Code for Flutter Developers

A practical setup guide for Flutter developers using Claude Code — Anthropic's terminal-first AI pair programmer. Skips the AI hype and shows you exactly what to wire up, why, and what to avoid.

Sourced from the official Claude Code docs at [code.claude.com](https://code.claude.com/docs) and Flutter's official AI guidance at [docs.flutter.dev/ai](https://docs.flutter.dev/ai). Examples stick to first-party Flutter and Dart tooling — adapt freely to whichever packages your team uses.

---

## Contents

- [Why bother](#why-bother)
- [Quickstart (10 minutes)](#quickstart-10-minutes)
- [Official Flutter AI resources](#official-flutter-ai-resources)
- [The mental model](#the-mental-model)
- [Project setup](#project-setup)
- [CLAUDE.md — the project brain](#claudemd--the-project-brain)
- [Skills — reusable Flutter playbooks](#skills--reusable-flutter-playbooks)
- [Subagents — delegate big tasks](#subagents--delegate-big-tasks)
- [MCP — plug in Flutter tooling](#mcp--plug-in-flutter-tooling)
- [Hooks — guardrails that always fire](#hooks--guardrails-that-always-fire)
- [Permissions and safety](#permissions-and-safety)
- [Auto memory — let Claude learn your project](#auto-memory--let-claude-learn-your-project)
- [Path-scoped rules](#path-scoped-rules)
- [Daily workflows](#daily-workflows)
- [Debugging with Claude](#debugging-with-claude)
- [Recipes](#recipes)
- [Troubleshooting](#troubleshooting)
- [Cheat sheet](#cheat-sheet)

---

## Why bother

Flutter projects accumulate boilerplate that's tedious but not hard: widgets, models, route definitions, repository methods, widget tests. Claude Code is good at exactly this work — it reads your conventions from existing files and matches them.

Where it earns its keep:

- Generating a feature (state holder + repository + screen + test) from a one-line description
- Running `flutter analyze` and fixing what it reports
- Reading stack traces from `flutter run` and patching the offending code
- Writing widget tests for screens you already shipped

Where it struggles, so set expectations:

- It guesses package APIs when offline — give it `pubspec.yaml` access and tell it to read source
- It defaults to verbose patterns unless your CLAUDE.md tells it your preferred style
- It will happily run `dart run build_runner build` even when watch would be smarter — see [Hooks](#hooks--guardrails-that-always-fire)

---

## Quickstart (10 minutes)

Assumes you already have Flutter installed (`flutter doctor` clean) and a project to work on.

### 1. Install Claude Code

```bash
# macOS / Linux / WSL
curl -fsSL https://claude.ai/install.sh | bash

# Or Homebrew on macOS
brew install --cask claude-code

# Or Windows (PowerShell)
irm https://claude.ai/install.ps1 | iex
```

### 2. Start it in your Flutter project

```bash
cd ~/my_flutter_app
claude
```

First run prompts you to log in. After that, you're in a REPL.

### 3. Bootstrap project memory

Inside the session:

```text
/init
```

This generates a starter `CLAUDE.md` by reading your repo — `pubspec.yaml`, your widget conventions, your existing routes. Review and trim it. Aim for under 200 lines.

### 4. Add the Dart MCP server

The Dart team ships an official MCP server that gives Claude direct access to the analyzer, test runner, hot reload, and pub. Exit the session and:

```bash
claude mcp add --transport stdio dart -- dart mcp-server
```

Restart Claude and run `/mcp` — you should see `dart` connected.

### 5. Try it

```text
> add a widget test for lib/features/auth/login_screen.dart
```

Claude will read the screen, look at neighboring tests for style, write the test, and run it. If anything fails, it iterates.

That's the whole loop. Everything below is making this loop faster, safer, and team-friendly.

---

## Official Flutter AI resources

The Flutter and Dart teams publish their own AI-agent guidance at [docs.flutter.dev/ai](https://docs.flutter.dev/ai). Use it as the source of truth for what Flutter wants AI agents to know; this guide layers Claude Code's specifics on top.

### Official rule templates ([docs.flutter.dev/ai/ai-rules](https://docs.flutter.dev/ai/ai-rules))

The Flutter team maintains rule files in four sizes so you can match your tool's context budget:

| Template | Size | Source |
| --- | --- | --- |
| `rules.md` | Comprehensive | [flutter/flutter on GitHub](https://github.com/flutter/flutter/tree/main/docs/rules) |
| `rules_10k.md` | <10k chars | same repo |
| `rules_4k.md` | <4k chars | same repo |
| `rules_1k.md` | <1k chars | same repo |

For Claude Code the rules file is just `CLAUDE.md` at the repo root. Use Flutter's template as a starting point:

```bash
# Pull the comprehensive template into your project
curl -fsSL https://raw.githubusercontent.com/flutter/flutter/main/docs/rules/rules.md -o CLAUDE.md
```

Then prune what doesn't apply and add your project-specific bits. The Flutter rules cover null safety, widget patterns, async, testing, accessibility, and platform handling — all the stuff you'd otherwise re-type into every CLAUDE.md.

### Official Agent Skills ([docs.flutter.dev/ai/agent-skills](https://docs.flutter.dev/ai/agent-skills))

The Dart and Flutter teams ship curated skill bundles:

- **[`dart-lang/skills`](https://github.com/dart-lang/skills)** — generate unit tests, resolve dependencies, fix static analysis errors
- **[`flutter/skills`](https://github.com/flutter/skills)** — build responsive layouts, wire declarative routing, implement JSON serialization

Install with the official CLI (needs Node.js):

```bash
npx skills add flutter/skills --skill '*' --agent universal
npx skills add dart-lang/skills --skill '*' --agent universal
```

This drops files into `.agents/skills/` — the cross-tool convention used by Antigravity, Gemini CLI, Cursor, OpenCode, Codex CLI, and others.

> **Claude Code gotcha**: Claude Code reads `.claude/skills/` and `~/.claude/skills/` only — it does *not* automatically discover `.agents/skills/`. Two ways to bridge:
> ```bash
> # Option A: symlink so the same files serve both
> ln -s ../../.agents/skills .claude/skills
>
> # Option B: launch Claude with the directory added
> claude --add-dir .agents
> ```
> With `--add-dir .agents`, Claude Code's [skills exception](https://code.claude.com/docs/en/skills) auto-loads `.agents/skills/*/SKILL.md`. Symlinking is simpler if it's a long-term setup.

### Dart and Flutter MCP server ([docs.flutter.dev/ai/mcp-server](https://docs.flutter.dev/ai/mcp-server))

Already covered in [Quickstart](#quickstart-10-minutes), but worth noting from the official docs:

- Requires **Dart 3.9 or later**.
- Status: **Experimental** — APIs and tool names may change.
- Capabilities (per the official list): analyze and fix errors, resolve symbols and fetch docs, introspect a running app, search pub.dev, manage `pubspec.yaml`, run tests, format code.
- If your MCP client claims to support roots but doesn't, append `--force-roots-fallback` to the server command.
- File issues at [dart-lang/ai](https://github.com/dart-lang/ai/issues).

### Best practices ([docs.flutter.dev/ai/best-practices](https://docs.flutter.dev/ai/best-practices))

The Flutter team's prompting guide is worth a read if you're new to coding with AI. It covers prompt structure, expected outputs, tool calls, and interaction modes — all transferable to Claude Code.

---

## The mental model

Three things matter:

**Context is everything.** Claude only knows what's in its context window. CLAUDE.md, the current message, files it has read, and tool output are *all* it sees. Junk in → junk out. Keep CLAUDE.md tight, point Claude at specific files, and use subagents to keep exploration out of the main thread.

**Tools, not just text.** Claude doesn't just write code — it runs `flutter test`, reads `pubspec.lock`, edits files, calls MCP servers. Most of your config is about deciding what tools it can use without asking you.

**Layered config.** Settings cascade from machine-wide to user to project to local. You'll commit team settings to git and override personal preferences in a gitignored file.

```text
Highest priority
  ↓ Session flags (--model, --permission-mode)
  ↓ Environment variables
  ↓ .claude/settings.local.json   (yours, gitignored)
  ↓ .claude/settings.json         (team, committed)
  ↓ ~/.claude/settings.json       (your personal default for all projects)
  ↓ Managed policy (org-wide)
Lowest priority
```

---

## Project setup

A Flutter project ready for Claude Code looks like this:

```text
my_flutter_app/
├── .claude/
│   ├── CLAUDE.md                # or put it at repo root
│   ├── settings.json            # team settings (committed)
│   ├── settings.local.json      # your overrides (gitignored)
│   ├── skills/                  # /<skill-name> commands
│   │   ├── widget-test/
│   │   ├── new-feature/
│   │   └── fix-analyzer/
│   ├── agents/                  # subagents (committed)
│   │   ├── flutter-architect.md
│   │   └── dart-reviewer.md
│   ├── rules/                   # path-scoped instructions
│   │   ├── widgets.md
│   │   └── testing.md
│   └── hooks/                   # shell scripts hooks run
│       └── format-on-save.sh
├── .mcp.json                    # team MCP servers (committed)
├── CLAUDE.md                    # ← or here, either works
├── lib/
│   ├── features/
│   ├── core/
│   └── main.dart
├── test/
├── integration_test/
└── pubspec.yaml
```

### .gitignore additions

```gitignore
# Claude Code personal config
.claude/settings.local.json
.claude.json

# But commit team config
!.claude/settings.json
!.claude/skills/
!.claude/agents/
!.claude/rules/
!.claude/hooks/
!.mcp.json
```

---

## CLAUDE.md — the project brain

Loaded into context at the start of every session. Treat it as the answer to: *what would I tell a new teammate on day one?*

**Rules of thumb:**

- Under 200 lines. Long files hurt adherence and burn tokens.
- Facts, conventions, and standing rules — not procedures. Move procedures into [skills](#skills--reusable-flutter-playbooks).
- Things specific to a folder go in a nested `CLAUDE.md` in that folder, or use [path-scoped rules](#path-scoped-rules).
- Generate the first draft with `/init`, then prune.

### Start from Flutter's official template

The Flutter team publishes a comprehensive rules template — use it as your base instead of writing from scratch:

```bash
# Drop the master template into your project as CLAUDE.md
curl -fsSL https://raw.githubusercontent.com/flutter/flutter/main/docs/rules/rules.md -o CLAUDE.md

# Or one of the smaller variants if you're tight on context budget
# rules_10k.md, rules_4k.md, rules_1k.md (same path pattern)
```

Then trim sections that don't apply and append your project-specific bits below. The Flutter rules cover null safety, widget conventions, async, testing, accessibility, and platform handling.

### Or write a minimal template yourself

Replace anything in `<angle brackets>` with what your project actually uses. The point is to encode *your* conventions, not endorse any particular package.

```markdown
# my_flutter_app

A cross-platform Flutter app for <domain>. iOS 13+, Android API 24+, Web.

## Stack
- Flutter <version> / Dart <version>
- State management: <your library>
- Navigation: <your router>
- Models / serialization: <your codegen, if any>
- HTTP: <your client>
- Local storage: <your persistence layer>

## Layout
- `lib/features/<name>/{data,domain,presentation}/` — one folder per feature
- `lib/core/` — cross-cutting (network, storage, theme, extensions)
- `lib/app/` — `MaterialApp`, router, dependency injection
- `test/` mirrors `lib/`

## Always
- Use `const` constructors anywhere they compile.
- Prefer `StatelessWidget` over `StatefulWidget` unless you genuinely need lifecycle.
- Wrap async failures in typed exceptions, not raw strings.
- After editing a Dart file, run `flutter analyze <file>` before claiming the task is done.
- After editing files with code-generation annotations, run `dart run build_runner build --delete-conflicting-outputs`.

## Never
- Use `print` — use `dart:developer` `log` instead.
- Import `dart:io` in code that ships to web without a platform guard.
- Add a new package without checking pub.dev score and last-publish date.
- Commit generated files (`.g.dart`, `.freezed.dart`, etc.) — they're produced by CI.

## Commands
- Run app: `flutter run -d <device>`
- Test: `flutter test`
- Codegen: `dart run build_runner build --delete-conflicting-outputs`
- Lint: `flutter analyze`

## Imports
@docs/architecture.md
@docs/api-conventions.md
```

The `@path/to/file.md` syntax pulls another file into context at launch. Use it for longer reference docs you don't want bloating CLAUDE.md.

### Nested CLAUDE.md

Drop a `CLAUDE.md` inside a feature folder for feature-specific conventions. It loads only when Claude reads files in that folder — cheap context.

```text
lib/features/payments/
├── CLAUDE.md         ← payment-specific notes
├── data/
└── presentation/
```

---

## Skills — reusable Flutter playbooks

A skill is a folder with `SKILL.md` inside it. When you type `/<skill-name>`, the file's content becomes the prompt. When Claude decides a skill is relevant, it loads it automatically.

> Skills replaced the old `.claude/commands/` system. Existing command files still work — Claude treats `.claude/commands/foo.md` the same as `.claude/skills/foo/SKILL.md` — but skills support more (multi-file, dynamic injection, fork-to-subagent, path-scoping).

### Get Flutter's official skills first

Before writing your own, install the curated bundles from the Flutter and Dart teams:

```bash
npx skills add flutter/skills --skill '*' --agent universal
npx skills add dart-lang/skills --skill '*' --agent universal
```

This populates `.agents/skills/`. To make them visible to Claude Code, see the [bridge note](#official-flutter-ai-resources) above (symlink or `--add-dir`).

What you get:

- **flutter/skills** — responsive layouts, declarative routing, JSON serialization
- **dart-lang/skills** — unit test generation, dependency resolution, analyzer fixes

After installing, ask Claude: *"List my installed skills"* — it'll report what was picked up.

### Where your own skills live

| Location | Scope |
| --- | --- |
| `.claude/skills/<name>/SKILL.md` | This project (committed) |
| `~/.claude/skills/<name>/SKILL.md` | All your projects |
| `.agents/skills/<name>/SKILL.md` | Cross-tool (visible to Claude only via symlink or `--add-dir`) |

### Frontmatter cheat sheet

```yaml
---
description: One sentence. What it does + when to use it. Claude reads this to decide whether to load.
argument-hint: <file-path>             # shown in autocomplete
allowed-tools: Read Edit Bash(flutter analyze *)
disable-model-invocation: false        # true = only you can /trigger it
paths:                                 # only auto-loads on matching files
  - "lib/**/*.dart"
context: fork                          # run in subagent (isolated context)
agent: Explore                         # which subagent type for fork
model: inherit                         # or sonnet / haiku / opus
effort: medium                         # low / medium / high / max
---
```

`$ARGUMENTS`, `$0`, `$1`, `$2` inside the body get substituted with what the user typed after `/skill-name`.

`` !`<shell command>` `` in the body runs the command at load time and inlines its output. Lets Claude react to live state — your current diff, current device list, current analyzer output.

### Skill 1: `/widget-test` — generate a widget test

```yaml
---
description: Generate a widget test for the given file. Reads neighboring tests for style, drafts the test, runs it, fixes failures.
argument-hint: <path/to/widget.dart>
allowed-tools: Read Edit Write Bash(flutter test *)
---

Write a widget test for `$ARGUMENTS`.

1. Read the widget file.
2. List sibling files in the matching `test/` directory and read one or two for style.
3. Stub dependencies the same way existing tests do (check what mocking approach the project uses).
4. Cover the happy path and one error path.
5. Run `flutter test` on the corresponding test path (swap `lib/` for `test/` and append `_test`).
6. If it fails, fix the test or note a real widget bug. Don't change widget code unless I confirm.
```

Use it: `/widget-test lib/features/auth/login_screen.dart`

### Skill 2: `/new-feature` — scaffold a feature folder

```yaml
---
description: Scaffold a new feature with data/domain/presentation layers, a state holder, a basic screen, and stub test files.
argument-hint: <feature_name>
allowed-tools: Write Edit Read Bash(dart run build_runner *)
---

Scaffold feature `$ARGUMENTS`:

1. Create directories under `lib/features/$ARGUMENTS/`:
   - `data/repositories/`, `data/sources/`
   - `domain/entities/`, `domain/use_cases/`
   - `presentation/screens/`, `presentation/widgets/`, `presentation/controllers/`
2. Create matching `test/features/$ARGUMENTS/` skeleton.
3. Read `lib/app/router.dart` and add a route stub for the new screen.
4. Generate a state holder and screen following the patterns used elsewhere in `lib/features/` — read one existing feature first to match conventions.
5. Run `dart run build_runner build --delete-conflicting-outputs` if the project uses code generation.
6. Print a summary of files created and one TODO per layer.
```

### Skill 3: `/fix-analyzer` — clear analyzer errors

```yaml
---
description: Run flutter analyze, then fix every error and warning. Stops at the first ambiguous case for confirmation.
allowed-tools: Bash(flutter analyze *) Bash(dart fix --apply) Read Edit
---

## Current analyzer output
!`flutter analyze --no-fatal-warnings 2>&1 | tail -100`

## Task

Fix everything above:

1. Run `dart fix --apply` first to handle mechanical fixes.
2. For each remaining issue, open the file and fix it minimally. No drive-by refactors.
3. If a fix would change public API or runtime behavior, stop and ask.
4. Re-run analyze and confirm zero issues before finishing.
```

The `` !`flutter analyze ...` `` line runs at skill load — Claude sees real analyzer output as the prompt arrives.

### Skill 4: `/codegen` — handle build_runner

```yaml
---
description: Run build_runner. Picks build or watch based on context, deletes conflicting outputs, surfaces real errors.
argument-hint: <build|watch>
allowed-tools: Bash(dart run build_runner *) Read
---

Run build_runner with mode `$ARGUMENTS` (default `build`):

- `build`: `dart run build_runner build --delete-conflicting-outputs`
- `watch`: `dart run build_runner watch --delete-conflicting-outputs` (run in background)

If output mentions conflicts in generated files:
1. Identify the offending source files.
2. Show me the import or annotation issue.
3. Do not delete generated files manually.

If output is clean, just report "Codegen complete, N files generated."
```

### Skill 5: `/commit` — create commits the right way

```yaml
---
description: Stage and commit changes with a conventional commit message.
disable-model-invocation: true
allowed-tools: Bash(git status) Bash(git diff *) Bash(git add *) Bash(git commit *) Bash(flutter analyze) Bash(flutter test)
---

## Status
!`git status --short`

## Staged diff
!`git diff --staged`

## Unstaged diff (preview)
!`git diff | head -200`

## Task

1. If nothing is staged, ask which files to stage.
2. Before committing, run `flutter analyze` and `flutter test` on touched files.
3. Group changes into one logical commit. If the diff spans unrelated changes, propose splitting.
4. Write a conventional commit subject (`feat:`, `fix:`, `refactor:`, etc.) under 72 chars.
5. Add a body explaining *why*, not *what*.
6. Show me the message and wait for approval before committing.

Never use `--no-verify`. Never `--amend` without explicit instruction.
```

`disable-model-invocation: true` means Claude won't decide on its own to commit — you have to type `/commit`.

### Skill 6: `/upgrade-flutter` — bump SDK safely

```yaml
---
description: Upgrade the Flutter SDK, run codegen, fix breakage, run the test suite.
argument-hint: <flutter-version>
allowed-tools: Bash(flutter *) Bash(dart *) Read Edit
context: fork
agent: general-purpose
---

Upgrade Flutter to `$ARGUMENTS`.

1. Bump the SDK pin (check whether the repo uses a version manager, the system Flutter, or pubspec constraints).
2. `flutter pub upgrade --major-versions`.
3. `dart run build_runner build --delete-conflicting-outputs` if the project uses code generation.
4. `flutter analyze` — fix new analyzer errors caused by API changes only.
5. `flutter test` — fix breakage caused by SDK changes only. Skip flaky tests with a TODO.
6. Summarize what changed, what broke, and what was deferred.
```

`context: fork` runs this in a fresh subagent context so the migration noise doesn't pollute your main session.

---

## Subagents — delegate big tasks

A subagent is a specialized assistant Claude can hand work to. It runs in its own context window and returns a summary. Use them when a task would otherwise dump 10,000 tokens of grep/read into your main thread.

### File format

`.claude/agents/<name>.md`:

```markdown
---
name: dart-reviewer
description: Reviews Dart/Flutter code for quality, idioms, leaks, and state-management misuse. Invoke after edits to a feature, before opening a PR.
model: claude-sonnet-4-6
tools: Read Grep Glob Bash(flutter analyze *)
---

You are a senior Flutter engineer reviewing code.

Focus on:
- Null safety: no needless `!`, no `late` time bombs.
- State management: correct scoping, no rebuild storms, disposables disposed.
- Widgets: const constructors used; build methods under ~80 lines; rebuild scope minimized.
- Effective Dart conformance.
- Memory: every controller, stream subscription, and listener has a paired dispose.
- Platform-specific code is gated for web/desktop.
- Tests exist for non-trivial logic.

Output as inline review comments with file:line references. Suggest concrete diffs. Don't rewrite anything yourself.
```

### Three Flutter subagents worth having

**`flutter-architect.md`** — for design decisions, Opus, read-only tools:

```markdown
---
name: flutter-architect
description: Designs Flutter architecture. Use for state management decisions, layer boundaries, navigation patterns, large refactor planning. Read-only.
model: claude-opus-4-7
tools: Read Grep Glob WebFetch
---

You are an experienced Flutter architect.

Reach for the simplest pattern that fits the codebase's existing style. Read existing features before proposing new structure; match what's already there unless I ask you to break the mold.

For every recommendation:
1. Name the trade-off.
2. Show a minimal code sketch.
3. List what it implies for testing.
4. Flag platform-specific risks (web, desktop, foldables).

Never write to files. Output a written design.
```

**`flutter-ui-specialist.md`** — for animations, custom paint, layout puzzles:

```markdown
---
name: flutter-ui-specialist
description: Builds and refactors complex Flutter UI: animations, custom layouts, responsive design, accessibility, performance.
tools: Read Write Edit Bash(flutter run *)
---

You are a Flutter UI specialist.

Defaults:
- Material 3 via `ColorScheme.fromSeed` unless the project uses a custom theme.
- Implicit animations first; reach for `AnimationController` only when you need control.
- `LayoutBuilder` for responsive breakpoints.
- Wrap heavy widgets in `RepaintBoundary` when they paint independently.
- `Semantics` on interactive widgets; respect `MediaQuery.textScalerOf`.

For UI work:
1. Sketch the widget tree before coding.
2. Extract subwidgets when build exceeds ~80 lines.
3. Verify on iOS, Android, and a 600+ wide layout.
```

### Invoking subagents

Two ways:

```text
> use flutter-architect: design the offline sync layer for the messages feature
```

Or just describe a matching task and Claude routes automatically:

```text
> review the changes in lib/features/auth before I open a PR
```

If the descriptions are well-written, automatic routing usually works.

### Skills vs subagents

| | Skill | Subagent |
| --- | --- | --- |
| Triggered by | `/name` or auto-match | Auto-match or explicit handoff |
| Runs in | Your conversation | Isolated context, returns summary |
| Good for | Repeatable procedures | Heavy exploration, focused expertise |
| Persistence | Loads file each invoke | Has its own auto memory |

A common pattern: write a skill with `context: fork` and `agent: flutter-architect`. The skill defines the *task*, the subagent provides the *expertise*.

---

## MCP — plug in Flutter tooling

MCP (Model Context Protocol) lets external processes give Claude new tools. For Flutter, the must-have is the official Dart and Flutter MCP server.

### Install the Dart MCP server

```bash
claude mcp add --transport stdio dart -- dart mcp-server
```

This makes Claude able to call the analyzer, run tests, hot reload, query the running app, search pub.dev — without shelling out to `flutter` every time. It's significantly faster and gives Claude structured output instead of parsing terminal text.

> **Status: experimental** (per the [Flutter docs](https://docs.flutter.dev/ai/mcp-server)). Requires Dart 3.9+. Tool names and behavior may change. File issues at [dart-lang/ai](https://github.com/dart-lang/ai/issues).

Verify:

```text
> /mcp
```

You should see `dart` connected.

### Share with your team

Put the config in `.mcp.json` at the repo root and commit it:

```json
{
  "mcpServers": {
    "dart": {
      "command": "dart",
      "args": ["mcp-server"]
    }
  }
}
```

Anyone who clones the repo gets the same tooling. They'll see an approval prompt the first time Claude starts.

### What it gives Claude

Per the official docs, the server lets the agent:

- Analyze and fix errors in your project's code
- Resolve symbols to elements (existence check, docs, signature)
- Introspect and interact with your running application
- Search pub.dev for the right package (`pub_dev_search`)
- Manage `pubspec.yaml` dependencies
- Run tests and analyze the results
- Format code with the same formatter as `dart format`

Tool names may evolve. Run `/mcp` then select `dart` to see the current list.

### Environment variables

`.mcp.json` supports `${VAR}` expansion and `${VAR:-default}`:

```json
{
  "mcpServers": {
    "example": {
      "command": "example-mcp-server",
      "env": {
        "API_TOKEN": "${API_TOKEN}"
      }
    }
  }
}
```

Don't commit secrets. Use shell env vars or a `.env` loaded by your shell.

---

## Hooks — guardrails that always fire

Hooks are shell commands (or HTTP endpoints, or LLM prompts) that run on lifecycle events. Unlike instructions in CLAUDE.md, hooks are deterministic — they always run, regardless of what Claude decides.

### Events you'll actually use for Flutter

| Event | Use it for |
| --- | --- |
| `SessionStart` | Print device list, warn about stale `pubspec.lock` |
| `PreToolUse` (matcher: `Edit\|Write`) | Block edits to secrets, `pubspec.lock`, generated files |
| `PostToolUse` (matcher: `Edit\|Write`) | `dart format` + `flutter analyze` on touched Dart files |
| `Stop` | Run the full test suite when Claude says it's done |
| `UserPromptSubmit` | Inject current branch, current device, recent commits |

Full event list: `SessionStart`, `Setup`, `SessionEnd`, `UserPromptSubmit`, `UserPromptExpansion`, `Stop`, `StopFailure`, `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `PermissionRequest`, `PermissionDenied`, `SubagentStart`, `SubagentStop`, `TaskCreated`, `TaskCompleted`, `FileChanged`, `ConfigChange`, `InstructionsLoaded`, `CwdChanged`, `PreCompact`, `PostCompact`, `Notification`, `WorktreeCreate`, `WorktreeRemove`.

### Hook config in settings.json

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/format-and-analyze.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/protect-files.sh"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/session-start.sh"
          }
        ]
      }
    ]
  }
}
```

### Hook: format and analyze on edit

`.claude/hooks/format-and-analyze.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Read tool input from stdin (JSON)
FILE=$(jq -r '.tool_input.file_path // empty')
[ -z "$FILE" ] && exit 0
[[ "$FILE" != *.dart ]] && exit 0
[[ "$FILE" == *.g.dart ]] && exit 0
[[ "$FILE" == *.freezed.dart ]] && exit 0

# Format silently
dart format "$FILE" >/dev/null 2>&1 || true

# Analyze, feed problems back to Claude on stderr
ANALYSIS=$(cd "$CLAUDE_PROJECT_DIR" && flutter analyze "$FILE" 2>&1 | tail -20)
if echo "$ANALYSIS" | grep -qE "error •|warning •"; then
  echo "Analyzer issues in $FILE:" >&2
  echo "$ANALYSIS" >&2
fi

# Hint about codegen if the file has annotations and a build.yaml exists
if [ -f "$CLAUDE_PROJECT_DIR/build.yaml" ] && grep -qE '@[A-Z][A-Za-z]+\(' "$FILE"; then
  echo "Annotations changed in $FILE — run /codegen if you haven't already." >&2
fi
```

Make it executable: `chmod +x .claude/hooks/*.sh`.

### Hook: block edits to dangerous files

`.claude/hooks/protect-files.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

FILE=$(jq -r '.tool_input.file_path // empty')
[ -z "$FILE" ] && exit 0

# Patterns to block
case "$FILE" in
  *.env|*.env.*) BLOCKED="env file" ;;
  */secrets.dart) BLOCKED="secrets file" ;;
  */GoogleService-Info.plist) BLOCKED="Firebase iOS config" ;;
  */google-services.json) BLOCKED="Firebase Android config" ;;
  */key.properties) BLOCKED="Android signing config" ;;
  */upload-keystore.jks) BLOCKED="Android keystore" ;;
  *pubspec.lock) BLOCKED="lockfile (run flutter pub get instead)" ;;
  *.g.dart|*.freezed.dart) BLOCKED="generated file (regenerate via /codegen)" ;;
  *) exit 0 ;;
esac

echo "Refusing to edit $FILE — $BLOCKED" >&2
exit 2   # exit code 2 blocks the tool call and tells Claude why
```

### Hook: useful session start

`.claude/hooks/session-start.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "Project: $(basename "$PWD")" >&2

# Stale lockfile?
if [ pubspec.yaml -nt pubspec.lock ]; then
  echo "Warning: pubspec.yaml newer than pubspec.lock — run 'flutter pub get'" >&2
fi

# Connected devices
DEVICES=$(flutter devices --machine 2>/dev/null | jq -r '.[].name' 2>/dev/null | head -3 | tr '\n' ',' | sed 's/,$//')
[ -n "$DEVICES" ] && echo "Devices: $DEVICES" >&2
```

### Exit codes

- `0`: success, hook runs, tool call proceeds.
- `2`: block. Tool call is cancelled, stderr is shown to Claude as feedback.
- Anything else: non-blocking error, logged but doesn't stop anything.

---

## Permissions and safety

Claude's default behavior: ask before doing anything destructive. You tune this in three places.

### permissions in settings.json

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Write",
      "Edit",
      "Bash(flutter analyze*)",
      "Bash(flutter test*)",
      "Bash(flutter pub *)",
      "Bash(dart *)",
      "Bash(git status)",
      "Bash(git diff*)",
      "Bash(git log*)",
      "mcp__dart__*"
    ],
    "ask": [
      "Bash(git push*)",
      "Bash(flutter build *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(**/secrets.dart)",
      "Read(**/google-services.json)",
      "Read(**/GoogleService-Info.plist)",
      "Bash(rm -rf*)",
      "Bash(git push --force*)",
      "WebFetch"
    ],
    "defaultMode": "default"
  }
}
```

Pattern: `Tool(specifier)`. `Bash(flutter test*)` matches any command starting with `flutter test`. `Read(./.env)` matches an exact path. Wildcards (`*`, `**`) work as glob.

### defaultMode

| Mode | What happens |
| --- | --- |
| `default` | Asks for risky operations. Recommended. |
| `acceptEdits` | Auto-approves file edits, still asks on bash/network. |
| `plan` | Read-only — perfect for exploration and PR review. |
| `auto` | Claude decides per tool based on classifier. |
| `bypassPermissions` | Never asks. Dangerous. |

Toggle live with `Shift+Tab`: cycles Normal → Auto-Accept → Plan.

### Plan mode for safe exploration

```bash
claude --permission-mode plan
```

Claude can read, search, and reason, but cannot edit, run shell commands that modify state, or call write-y MCP tools. Great for:

- Reviewing a PR branch before merging
- Asking Claude to design something before committing
- Exploring an unfamiliar codebase

### Sandbox mode

The full sandbox config locks Claude into a process-level jail with filesystem and network rules. Most useful in CI or when you're trying a new untrusted plugin.

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "allowWrite": [".", "/tmp"],
      "denyWrite": ["**/secrets.dart", "**/.env*"],
      "denyRead": ["~/.aws/credentials", "~/.ssh/**"]
    },
    "network": {
      "allowedDomains": ["pub.dev", "api.flutter.dev", "github.com"]
    }
  }
}
```

---

## Auto memory — let Claude learn your project

New in Claude Code v2.1.59+. Claude writes notes to itself across sessions about your build commands, debugging patterns, and recurring preferences. You don't manage it directly — it just works.

Storage: `~/.claude/projects/<repo>/memory/`

Toggle:

```json
{
  "autoMemoryEnabled": true
}
```

Or `/memory` inside a session to view and edit.

The first 200 lines (or 25KB) of `MEMORY.md` load every session. Topic files load on demand.

What ends up in there for a Flutter project:

- Your actual run command (which device, which entry point, which flavor)
- Discovered package quirks ("plugin X needs a manual platform setup step on Android")
- Patterns Claude noticed ("tests in this repo wrap each test in a fake-async block")

If Claude is suggesting something stale, run `/memory` and edit the relevant file. It's just markdown.

---

## Path-scoped rules

For instructions that should only apply when working on specific files, use `.claude/rules/` instead of bloating CLAUDE.md.

### Example: widget conventions

`.claude/rules/widgets.md`:

```markdown
---
paths:
  - "lib/**/*.dart"
  - "!lib/**/*.g.dart"
  - "!lib/**/*.freezed.dart"
---

# Widget rules

- Every public widget needs a `const` constructor.
- Build methods over 80 lines should be split into private widgets.
- Wrap independently-animating widgets in `RepaintBoundary` inside lists.
- Use `MediaQuery.sizeOf(context)` instead of `MediaQuery.of(context).size` to avoid unnecessary rebuilds.
```

### Example: testing rules

`.claude/rules/testing.md`:

```markdown
---
paths:
  - "test/**/*.dart"
  - "integration_test/**/*.dart"
---

# Testing rules

- Use whatever mocking approach the surrounding tests use — don't introduce a new one.
- Always `await tester.pumpAndSettle()` after async actions.
- Golden tests live in `test/golden/` and only update with explicit `--update-goldens`.
- Wrap the unit under test with the minimum scope needed; don't pump full app shells unless required.
```

Rules without a `paths:` block load every session, same as CLAUDE.md. Rules with `paths:` only inject when Claude opens a matching file. Use them to keep CLAUDE.md short.

---

## Daily workflows

### New feature from a one-liner

```text
> /new-feature notifications
```

→ Scaffolds folders, state holder, screen, route. Runs codegen.

```text
> read docs/specs/notifications.md and implement step 1
```

→ Reads spec, edits the scaffolded files, runs tests.

```text
> /commit
```

→ Stages, analyzes, tests, commits.

### Fixing a stack trace

Paste the trace:

```text
> [paste error]
this happened when I tapped the profile button. fix it.
```

Claude reads the trace, finds the offending file, applies the fix, runs the relevant test.

### Pre-PR review

```text
> use dart-reviewer on the diff since main
```

The subagent reviews in its own context, returns a list of issues. Address them, then:

```text
> /commit
```

### Migrating to a new SDK version

```text
> /upgrade-flutter 3.27.1
```

Skill forks a subagent, runs the upgrade, returns a summary. Your main session stays clean.

### Debugging a running app

In one terminal: `flutter run`. In another: `claude`. Then:

```text
> use the dart MCP to get current widget tree
> the home screen has a layout issue at the top — investigate
```

Claude queries the live app via the Dart Tooling Daemon, identifies the widget, proposes a fix, you hot-reload.

---

## Debugging with Claude

Three modes of debugging, picked by how much you know about the bug.

### Mode 1: stack trace in hand — paste and patch

Fastest path when you have an error or stack trace. Just paste it:

```text
> the app crashes when I open the profile screen. trace:
>
> [paste trace]
>
> fix it.
```

Claude reads the trace, opens the offending files, applies a fix, and runs the relevant test. With the Dart MCP installed it also gets structured analyzer feedback while iterating.

Tips that significantly improve hit rate:

- Paste the **full** trace, not just the top frame. The "Caused by" chain is where the real bug usually lives.
- Mention the reproduction step (*"when I tap save"*) — narrows search dramatically.
- If the trace is from `flutter run`, paste the few log lines before it too. State at crash time helps.

### Mode 2: live app introspection — let Claude see the running app

Run `flutter run` in one terminal, `claude` in another. With the Dart MCP server connected, Claude can:

- Inspect the current widget tree
- Pull runtime errors as they happen
- Take screenshots
- Trigger hot reload after a fix

```text
> the home screen has a layout issue at the top of the list. read the widget tree and figure out why.
```

This is enormously better than describing UI bugs in words. Claude sees what the device sees.

### Mode 3: "something feels wrong" — let Claude investigate

When you don't have a stack trace but something's off (jank, occasional crash, state desync), start in plan mode so Claude reads without touching:

```bash
claude --permission-mode plan
```

Then describe the symptom and ask for hypotheses:

```text
> ultrathink: the order list re-renders every keystroke in the search field even though
> only the filtered list should change. find the cause without editing anything.
```

`ultrathink` gives Claude a ~32K thinking budget — worth it for race conditions, rebuild storms, and other "why is this happening" bugs. Claude returns a hypothesis with file:line evidence; you switch out of plan mode to apply the fix.

### A reusable `/debug-flutter` skill

`.claude/skills/debug-flutter/SKILL.md`:

```yaml
---
description: Investigate a Flutter bug. Reads the trace or symptom, narrows to suspect files, proposes a fix, verifies with a test. Use when I paste an error, describe a crash, or report unexpected behavior.
argument-hint: <description or paste>
allowed-tools: Read Grep Glob Bash(flutter test *) Bash(flutter analyze *) mcp__dart__*
---

Investigate: $ARGUMENTS

1. If a stack trace is included, identify the deepest non-framework frame and read that file first.
2. If no trace, list the most likely files based on the symptom and read the top three.
3. Form a hypothesis. Print it before changing anything.
4. If the cause is in a dependency, stop and ask — don't modify package code.
5. Apply the smallest fix. Don't refactor.
6. Run the affected test (or write one if none exists). Confirm it passes.
7. Run `flutter analyze` on touched files.
8. Summarize: root cause, fix, test result.

If hypothesis is wrong after one fix attempt, stop and report what you learned. Don't keep guessing.
```

Use it: `/debug-flutter [paste trace or symptom]`

### Common Flutter bug categories and prompts

| Bug type | Useful prompt |
| --- | --- |
| `RenderFlex overflow` | "Paste the overflow warning. Read the widget. Fix without changing logic." |
| Build errors after pub upgrade | "Run `flutter pub outdated`, identify the breaking package, propose a pin or migration." |
| Hot reload not updating state | "Why does this state survive hot reload but reset on restart? Find the initializer." |
| Test passes locally, fails in CI | "Compare local Flutter version with the CI workflow's. Find env diffs." |
| Async UI not updating | "This `FutureBuilder`/stream doesn't rebuild. Walk through the rebuild path." |
| Platform-specific crash | "Reproduces on Android only. Read platform channels and Android setup files." |

### Use bundled debugging skills

Claude Code ships two debugging skills out of the box:

- `/debug` — structured debugging walkthrough (general, not Flutter-specific but works fine)
- `/simplify` — reviews recent changes for over-engineering after a fix

Worth knowing they exist before reinventing them.

### Anti-patterns to avoid

- **Don't fix without reproducing**: ask Claude to write or run a failing test *first*, then fix until it passes. Otherwise the "fix" may be unrelated.
- **Don't let it loop**: if Claude's first two fix attempts don't work, stop and read the trace yourself. It's diverging.
- **Don't skip analyzer feedback**: a hook that pipes `flutter analyze` output back to Claude after every edit catches a class of bugs before they reach runtime.
- **Don't disable failing tests**: it's tempting, and Claude will sometimes propose it. Say no unless you can explain why the test is wrong.

---

## Recipes

### "I don't trust it to write code yet"

Start in plan mode:

```bash
claude --permission-mode plan
```

Or run `/init` and only add skills that are read-only (`/widget-test` with `allowed-tools: Read` only). Build trust incrementally.

### "It keeps suggesting patterns we don't use"

Two fixes, in order:

1. CLAUDE.md: one sentence stating your convention ("we use `<pattern>` for state; never introduce a different approach").
2. `.claude/rules/widgets.md` with the same rule and `paths: ["lib/**/*.dart"]` so it loads only when Claude edits a Dart file.

If it still happens, mention it explicitly in chat — Claude will save the correction to auto memory.

### "It can't find my custom widgets"

Claude only knows what it's read. Two options:

- Add a barrel file (e.g. `lib/core/widgets/widgets.dart`) that re-exports your widgets, then `@import` it from CLAUDE.md.
- Tell Claude: `> read lib/core/widgets/` before asking it to build UI.

### "Tests are slow and Claude runs the whole suite"

Tighten the skill:

```yaml
allowed-tools: Bash(flutter test test/features/*)
```

And in CLAUDE.md: "When you change a file in `lib/features/X/`, only run tests in `test/features/X/`."

### "Set up monorepo support"

In root CLAUDE.md, describe how packages relate and which scripts run them:

```markdown
## Monorepo
Packages live in `packages/*`. When editing inside one, prefer running scripts scoped to that package — don't run the whole-workspace test suite unless I ask.
```

Each package can have its own `CLAUDE.md` and `.claude/skills/` — they load on demand when Claude opens files in that package.

### "Get auto-coverage reports"

Hook on `Stop`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "flutter test --coverage 2>&1 | tail -5",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

Or wire it into a `/wrap-up` skill you call manually.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| "Skill not triggering" | Run `/doctor` to check skill listing budget. Sharpen the `description` with words you'd actually say. |
| "Hook runs but Claude ignores its output" | Hook must exit `2` to block, or write to `stdout` (not stderr) for `SessionStart`/`UserPromptSubmit` context injection. |
| "MCP server not connecting" | `claude --mcp-debug`. Check `dart mcp-server --help` runs in your shell. PATH issues are common — use absolute paths in `.mcp.json` if needed. |
| "Codegen output keeps confusing Claude" | Add `*.g.dart` and `*.freezed.dart` to `permissions.deny` in settings, and protect them via the `protect-files.sh` hook. |
| "Claude uses the wrong Flutter version" | Make sure your version manager (or the system `flutter`) is on PATH. Add `"env": {"PATH": "..."}` in settings if needed. |
| "Tests pass for Claude, fail in CI" | Different Flutter versions. Pin the SDK in your repo and have CI use the same pin. |
| "Context is full and Claude is slow" | `/compact` (summarize), `/clear` (start fresh), or use subagents to keep exploration out of the main thread. |

`/doctor` is the single most useful diagnostic — it lists loaded CLAUDE.md files, active skills, MCP connections, and warnings.

---

## Cheat sheet

### Built-in commands

| Command | What |
| --- | --- |
| `/init` | Generate CLAUDE.md from your repo |
| `/memory` | View and edit memory (CLAUDE.md, auto memory) |
| `/doctor` | Diagnostics: configs, skills, MCPs, warnings |
| `/compact` | Summarize conversation to free context |
| `/clear` | Start fresh, keep settings |
| `/agents` | List and manage subagents |
| `/mcp` | List MCP servers, reconnect, authenticate |
| `/hooks` | View configured hooks |
| `/permissions` | Inspect permission rules |
| `/skills` | Toggle skill visibility |
| `/cost` | Token usage and approximate cost |
| `/model` | Switch model mid-session |
| `/review` | Bundled skill: review the current PR or branch |
| `/security-review` | Bundled skill: security pass on pending changes |
| `/debug` | Bundled skill: structured debugging |
| `/simplify` | Bundled skill: review changes for quality |

### Keyboard

| Key | Action |
| --- | --- |
| `Tab` | Toggle extended thinking on/off |
| `Shift+Tab` | Cycle permission mode (Normal → Auto-Accept → Plan) |
| `Ctrl+O` | Toggle verbose mode (see reasoning) |
| `Ctrl+C` | Cancel current operation |
| `Ctrl+R` | Search history |
| `/` | Open command menu |
| `@` | Reference files / MCP resources |

### Thinking budget keywords

Drop into your prompt for more reasoning:

| Phrase | Budget |
| --- | --- |
| `think` | ~4K tokens |
| `think hard`, `megathink` | ~10K |
| `think harder`, `think longer` | ~16K |
| `ultrathink` | ~32K (max) |

Use for architecture decisions and gnarly bugs. Overkill for a one-line fix.

### `@` references

```text
> explain @lib/main.dart
> review the structure of @lib/features/
> look at @lib/main.dart:42-78
> what changed in @github:issue://123
```

### Resume

```bash
claude --continue                 # most recent
claude --resume                   # pick from history
claude -c -p "your next prompt"   # one-shot continuation
```

### Models

Current recommended IDs (as of late 2025):

- `claude-opus-4-7` — planning, architecture, hardest bugs
- `claude-sonnet-4-6` — daily driver, implementation
- `claude-haiku-4-5` — quick edits, formatting, bulk renames

Set per-session: `claude --model claude-sonnet-4-6`
Set default in `settings.json`: `"model": "claude-sonnet-4-6"`

---

## Resources

**Claude Code (official):**

- [Claude Code docs](https://code.claude.com/docs) — updated continuously
- [Skills reference](https://code.claude.com/docs/en/skills)
- [Subagents reference](https://code.claude.com/docs/en/sub-agents)
- [Hooks reference](https://code.claude.com/docs/en/hooks)
- [Settings reference](https://code.claude.com/docs/en/settings)
- [MCP reference](https://code.claude.com/docs/en/mcp)
- [Memory and CLAUDE.md](https://code.claude.com/docs/en/memory)

**Flutter AI (official):**

- [Flutter & AI overview](https://docs.flutter.dev/ai)
- [AI Rules for Flutter and Dart](https://docs.flutter.dev/ai/ai-rules) — includes `rules.md` templates
- [Agent Skills for Flutter and Dart](https://docs.flutter.dev/ai/agent-skills)
- [Dart and Flutter MCP server](https://docs.flutter.dev/ai/mcp-server)
- [AI Coding Assistants](https://docs.flutter.dev/ai/coding-assistants)
- [Best practices: prompting](https://docs.flutter.dev/ai/best-practices/prompting)
- [`flutter/skills`](https://github.com/flutter/skills) — official Flutter skill repo
- [`dart-lang/skills`](https://github.com/dart-lang/skills) — official Dart skill repo
- [`flutter/flutter/docs/rules/`](https://github.com/flutter/flutter/tree/main/docs/rules) — rule templates in four sizes

**Language and framework:**

- [Flutter docs](https://docs.flutter.dev/)
- [Dart language tour](https://dart.dev/language)
- [Effective Dart](https://dart.dev/effective-dart)
- [Agent Skills standard](https://agentskills.io) — open spec Claude Code follows

---

Issues or improvements? PRs welcome — this is a living guide.
