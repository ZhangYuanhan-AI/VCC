# VCC Deep Dive — Discussion Summary

## What is VCC?

VCC (View-oriented Conversation Compiler) compiles Claude Code JSONL conversation logs into three efficient views. It treats conversation logs as source code and applies real compiler techniques to transform them.

Paper: https://arxiv.org/abs/2603.29678

---

## The Four Skills

| Skill | Trigger | Purpose |
|-------|---------|---------|
| `/readchat` | User mentions a .jsonl file | Read a specific conversation log |
| `/recall` | Context overflow continuation header, or user types `/recall` | Recover context from a previous session |
| `/searchchat` | User wants to search chat history | Search across many conversation logs |
| `conversation-compiler` | (reference only) | Core compiler documentation |

---

## `/recall` vs `/searchchat`

- **`/recall`** — "I know which conversation I need (from the continuation header) — reload it." Scope: one JSONL.
- **`/searchchat`** — "I don't know where we discussed X — find it." Scope: all JSONLs in `~/.claude/projects/`.

---

## `/recall` Workflow

1. Extract the JSONL path from the continuation header
2. Compile it: `python VCC.py conversation.jsonl`
3. Read `.min.txt` (last chunk = most recent, second-to-last = where previous session ended)
4. Optionally `--grep` for key topics if `.min.txt` isn't enough
5. Jump to `.txt` for full detail at referenced lines
6. Re-read all files that were touched in the previous session (they may have been externally modified)

After `/recall`, nothing is saved to disk as a "recovered state." Everything Claude read goes into the current conversation's context window. The `.txt`/`.min.txt` files are just intermediate artifacts on disk.

---

## What is `/compact`? (Claude Code Built-in, NOT VCC)

When a conversation fills the context window, Claude Code compresses it:

- **Layer 1: Micro-Compact** — automatic every turn, replaces old tool results with placeholders
- **Layer 2: Auto-Compact** — triggers at ~83.5% context capacity, summarizes everything
- **Layer 3: Manual `/compact`** — user-initiated, can guide what to retain (e.g., `/compact retain the bug fix approach`)

The summary is **lossy** — it keeps key decisions and file paths but loses exact code, intermediate debugging, and rejected approaches. VCC's `/recall` exists precisely to recover what `/compact` loses, by going back to the original JSONL.

### What the lossy summary looks like

```
This session is being continued from a previous conversation that ran out
of context. Here is a summary of what was discussed:

- We were refactoring the parser to handle document blocks
- Added base64 decoding for embedded PDFs
- Started implementing docx support but didn't finish

The user's last message was: "Now handle the docx case too"

read the full transcript at: ~/.claude/projects/vcc/abc752a87.jsonl
```

Missing: exact code written, approaches tried and rejected, error messages, specific variable names, tool call details.

---

## The Compiler Pipeline

```
conversation.jsonl
    │
    ▼ LEX — read JSONL, filter junk records
    │
    ▼ MERGE CHUNKS — combine split assistant messages (same ID)
    │
    ▼ SPLIT CHAINS — split at compact_boundary → numbered chunks
    │
    ▼ PARSE — build IR nodes from records
    │
    ▼ ASSIGN LINES — stamp every node with start_line/end_line
    │
    ├──▶ (no transform) ──▶ EMIT ──▶ .txt      (full, lossless)
    ├──▶ LOWER BRIEF    ──▶ EMIT ──▶ .min.txt  (scannable outline)
    └──▶ LOWER VIEW     ──▶ EMIT ──▶ .view.txt (search matches only, with --grep)
```

### What each stage does

**Lex** — `json.loads()` each line. Filter junk types: `queue-operation`, `file-history-snapshot`, `progress`, etc.

**Merge Chunks** — Claude Code streams assistant messages as multiple JSONL records with the same `message.id`. Merge them back into one.

**Split Chains** — When `/compact` fires, Claude Code inserts a `compact_boundary` record. Split at these boundaries → each chain becomes a separate numbered output file (`file_1.txt`, `file_2.txt`, ...).

**Parse** — Convert each JSONL record into IR nodes. One record may produce multiple nodes (e.g., an assistant message with thinking + text + tool_use → ~8 nodes). Also extracts base64 images/documents to files, converts tool parameters to YAML, strips line number prefixes from Read tool results.

**Assign Lines** — Single pass, stamps `start_line`/`end_line` on every node. These numbers are permanent and shared across all views.

**Lower Brief** — Hides thinking, system, tool results. Collapses tool calls to one-liners with line refs. Truncates text (128 tokens default, 256 for user). Strips noise XML.

**Lower View** — Regex-matches per node. Shows only matching blocks. Preserves conversation structure (headers/separators between visible sections).

**Emit** — Walks IR nodes, concatenates content lines. Same function for all three views — just reads different fields (`content`, `content_brief`, `content_view`).

---

## IR (Intermediate Representation) — The Core Data Structure

### What is an IR node?

A Python dict representing one piece of the conversation:

```python
{
    "type": "tool_call",                       # what kind of content
    "content": ["file_path: src/utils.py"],    # the actual text (full view)
    "content_brief": None,                     # brief view (None = hidden)
    "content_view": ["(demo.txt:19)..."],      # search view
    "searchable": True,                        # can --grep match this?
    "_sec": 2,                                 # section ID (one per role turn)
    "_blk": 4,                                 # block ID (groups related nodes)
    "_tool_summary": '* Read "src/utils.py"',  # one-liner for brief view
    "start_line": 19,                          # line coordinate in .txt
    "end_line": 19                             # line coordinate in .txt
}
```

### Node types

| Type | Searchable | What it represents |
|------|------------|-------------------|
| `meta` | No | Separators (`══════`), thinking markers (`>>>thinking`), tool_call markers |
| `meta_header` | No | Role headers (`[user]`, `[assistant]`, `[tool] Read:aaa111`) |
| `system` | Yes | System messages |
| `user` | Yes | User text |
| `assistant` | Yes | Assistant text |
| `thinking` | Yes | Assistant's thinking blocks |
| `tool_call` | Yes | Tool parameters (YAML formatted) |
| `tool_result` | Yes | Tool output |
| `tool_error` | Yes | Tool error output |

### How grouping works

- **`_sec` (section)** — One per role turn. All nodes in `[assistant]` section 2 share `_sec=2`. Used to decide: "hide this entire section in brief mode."
- **`_blk` (block)** — One per content block within a section. A thinking block, its `>>>thinking` marker, and its `<<<thinking` marker all share the same `_blk`. Used to: keep related nodes together, insert blank lines between blocks.

### How the three views work on the same IR

Transforms don't create new nodes — they fill in fields:

```
                    content           content_brief              content_view
thinking node:     ["Let me look"]    None (hidden)              None (no match)
tool_call node:    ["file_path:..."]  None (hidden, collapsed)   ["(demo.txt:19)..."] (match!)
tool_call meta:    [">>>tool_call"]   ['* Read "..." (19-21)']   [">>>tool_call"]
```

`_walk(ir, "content")` → full view — shows everything
`_walk(ir, "content_brief")` → brief view — skips `None` nodes
`_walk(ir, "content_view")` → search view — skips `None` nodes

### Why this design?

Line numbers are assigned once on the full IR. Since transforms only set fields to `None` (hide) or replace content (collapse), they **never reorder or renumber** nodes. This guarantees `.min.txt` line references always point to the correct `.txt` lines.

---

## The Three Output Views — Concrete Example

From a conversation where the user asks to fix a bug in `parse_config()`:

### `.txt` (Full — 115 lines)

```
[system]

You are Claude Code.

══════════════════════════════
[user]

There is a bug in utils.py - the parse_config function crashes...

══════════════════════════════
[assistant]

>>>thinking
The user reports a crash in parse_config...
<<<thinking

Let me look at the current implementation first.

>>>tool_call Read:aaa111
file_path: src/utils.py
<<<tool_call

══════════════════════════════
[tool] Read:aaa111

import json
import os

def parse_config(path):
    with open(path) as f:
        data = json.load(f)
    return data["settings"]
...
```

Everything is here. Lossless.

### `.min.txt` (Brief — 26 lines)

```
[user]

There is a bug in utils.py...

[assistant]

Let me look at the current implementation first.

* Read "src/utils.py" (demo.txt:19-21,24-39)

Found it. The issue is on line 6-7...

* Edit "src/utils.py" (demo.txt:50-68,71-73)

Now let me verify the fix works by running the tests.

* Bash "Run utils tests" (demo.txt:80-83,86-93)

Fixed! Here is what I changed...
```

What changed: system/thinking hidden, tool calls collapsed to one-liners with `.txt` line refs, tool results hidden.

The line refs mean: `(demo.txt:19-21,24-39)` → tool call at lines 19-21, result at lines 24-39 in `.txt`.

### `.view.txt` (Search for "parse_config" — 41 lines)

```
[user]

(demo.txt:8-8)
  8: There is a bug in utils.py - the parse_config function crashes...

══════════════════════════════
[tool] Read:aaa111

(demo.txt:26-39)
  29: def parse_config(path):

══════════════════════════════
[assistant]

>>>tool_call Edit:bbb222
(demo.txt:51-67)
  53:   def parse_config(path):
  58:   def parse_config(path):
<<<tool_call
```

Only blocks where "parse_config" appears, with line references and conversation structure preserved.

### Stdout (grep hits — reverse chronological)

```
(demo.txt:98-104) [assistant]
  98: Fixed! Here is what I changed in parse_config():

(demo.txt:88-93) [tool_result]
  88: test_utils.py::test_parse_config_normal PASSED

(demo.txt:51-67) [tool_call]
  53:   def parse_config(path):
```

Flat list of hits, most recent first. Quick to scan.

### `.txt` vs `.min.txt` vs `.view.txt` vs stdout

| | `.txt` | `.min.txt` | `.view.txt` | stdout |
|---|---|---|---|---|
| **When produced** | Always | Always | Only with `--grep` | Only with `--grep` |
| **Where** | File | File | File | Terminal |
| **Content** | Everything | Outline with collapsed tools | Only matching blocks | Just the match lines |
| **Size (demo)** | 115 lines | 26 lines | 41 lines | ~15 lines |
| **Use case** | Full detail at specific lines | Quick overview | Matches in context | Quick hit list |

---

## Key Lessons from VCC

1. **Conversation logs are structured data** — treat them like source code, not chat transcripts
2. **One IR, multiple views** — parse once, project many times, share coordinates across views
3. **Context is the bottleneck** — design for token efficiency (read `.min.txt` first, grep for specifics, jump to `.txt` only when needed)
4. **Compaction is lossy, keep the original** — the JSONL is ground truth, summaries are not
5. **Write tools for AI agents** — VCC's "UI" is SKILL.md files that teach Claude Code how to use the compiler
6. **Line numbers as coordinates** — assigned once, never renumbered, consistent across all views (same idea as source maps in JavaScript)
