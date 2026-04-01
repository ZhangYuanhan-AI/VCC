# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

VCC (View-oriented Conversation Compiler) compiles Claude Code JSONL conversation logs into efficient, agent-friendly views for context recovery, searching, and trace analysis. It is a real compiler with a classic pipeline: lexer → parser → IR → lowering → emitter.

Paper: https://arxiv.org/abs/2603.29678

## Running the Compiler

```bash
# Compile a conversation (produces .txt and .min.txt)
python skills/conversation-compiler/scripts/VCC.py <input.jsonl>

# With output directory
python skills/conversation-compiler/scripts/VCC.py <input.jsonl> -o <dir>

# Search mode (also produces .view.txt and prints matches to stdout)
python skills/conversation-compiler/scripts/VCC.py <input.jsonl> --grep "pattern"

# Token truncation tuning
python skills/conversation-compiler/scripts/VCC.py <input.jsonl> -t <N> -tu <N>
```

**Dependencies:** Python 3.10+, PyYAML (`pip install pyyaml`). No other external dependencies.

**No build step, test suite, linting, or CI/CD exists.** The project is a single-file compiler distributed as a Claude Code skill.

## Architecture

The entire compiler lives in `skills/conversation-compiler/scripts/VCC.py` (~1,073 lines).

### Compiler Pipeline

1. **Lexer (`lex()`)** — Parses JSONL records, filters junk record types (`queue-operation`, `file-history-snapshot`, `last-prompt`, `progress`, and several system message subtypes).

2. **Parser (`parse()`)** — Builds IR from JSONL records. Handles user/assistant/system/tool messages, extracts base64-encoded media to files, converts tool parameters to YAML, merges split assistant messages (same ID across chunks), and detects compaction boundaries.

3. **Line Assignment (`assign_lines()`)** — Assigns consistent line numbers to all IR nodes in a single pass. These numbers are shared across all three views and must never be reordered.

4. **Lowering:**
   - `lower_brief()` — Creates the brief view: hides thinking/system/internal tools, merges consecutive assistant sections, truncates text (token-based), collapses tool call+result into one-liners with line references, strips XML markup noise.
   - `lower_view()` — Creates the search view (when `--grep` is used): regex-matches per block, shows only matching blocks with preserved conversation structure, annotates with line ranges.

5. **Emitter (`emit()`)** — Generates output text from any view layer.

### Three Output Views (shared line numbers)

- **Full View (`.txt`)** — Lossless transcript with all detail
- **Brief View (`.min.txt`)** — Scannable outline with collapsed tool calls and truncated text
- **Search View (`.view.txt`)** — Only blocks matching `--grep` pattern, with line range annotations

### IR Node Structure

Each node has: `type`, `content` (list of strings), `searchable` (bool), `_sec` (section ID), `_blk` (block ID), `start_line`/`end_line`, `_tool_summary` (one-liner for tool calls), and projected fields `content_brief`/`content_view`.

### Key Internal Details

- **Token-based truncation** uses regex tokenizer `r'[a-zA-Z]+|[0-9]+|[^\sa-zA-Z0-9]|\s+'` with defaults of 128 tokens (general) and 256 tokens (user messages).
- **Sanitization** removes ANSI codes, control characters, and normalizes line endings.
- **Read tool post-processing** strips line number prefixes (`123→`) added by Claude Code.
- **Stats collection** tracks model names, API calls, tool uses, token usage (input/output/cache), subagent tokens, and duration.

### Code Layout in VCC.py

- Lines 1–330: Utilities (YAML formatting, tokenization, truncation, sanitization, lexer)
- Lines 331–510: Parser (IR construction)
- Lines 544–587: Line assignment and IR walking
- Lines 640–843: Brief mode lowering
- Lines 845–939: View mode lowering
- Lines 975–1073: Main compilation loop and CLI

## Skills

Four Claude Code skills in `skills/` provide user-facing interfaces:

| Skill | SKILL.md Location | Purpose |
|-------|-------------------|---------|
| `conversation-compiler` | `skills/conversation-compiler/SKILL.md` | Core compiler reference and workflow |
| `readchat` | `skills/readchat/SKILL.md` | Read specific JSONL conversation logs |
| `recall` | `skills/recall/SKILL.md` | Recover compacted conversation context |
| `searchchat` | `skills/searchchat/SKILL.md` | Search across conversation logs |

## Path Handling

Always use forward slashes in bash commands for cross-platform compatibility. Input files are typically from `~/.claude/projects/`.
