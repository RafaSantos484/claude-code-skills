---
name: export-user-prompts
description: Export only the human user's own prompts from the current Claude Code session to a plain .txt file. Excludes all assistant text, tool calls, tool results, and system-injected content. Use when the user asks to "export my prompts", "export user prompts", "save what I asked", "export user context", or wants a clean record of just their side of the conversation (e.g. to hand off to another session for an independent review).
---

# Export User Prompts

Export a plain-text file containing **only** the text the human user actually typed in the current session — nothing written by the assistant, no tool output, no system-injected reminders or metadata.

This is meant for handoff scenarios (e.g. giving another Claude session the original requirements without any of the implementing session's own reasoning or self-assessment, to avoid biasing an independent review).

## Step 1: Locate the current session's transcript

Claude Code stores every session as JSONL at:

```
~/.claude/projects/<project-slug>/<session-id>.jsonl
```

`<project-slug>` is the current working directory's absolute path with every non-alphanumeric character replaced by `-`.

Find the transcript for the *current* session:

```bash
PROJECT_SLUG=$(pwd | sed 's/[^a-zA-Z0-9]/-/g')
PROJECT_DIR="$HOME/.claude/projects/$PROJECT_SLUG"
# The current session is normally the most recently modified file
ls -t "$PROJECT_DIR"/*.jsonl | head -n 1
```

If more than one `.jsonl` file has a very recent modification time (e.g. multiple sessions open on the same project), stop and ask the user which one to use — don't guess.

## Step 2: Extract only genuine user-authored text

Each line in the JSONL file is one JSON object. Entries with `"type": "user"` can still contain non-human content (tool results are sent back on the "user" role in the underlying message format), so filter carefully:

- **Include**: entries where `type == "user"` and the message content is a plain string, or an array containing `"type": "text"` blocks — this is what the human actually typed.
- **Exclude**: entries that are purely `tool_result` blocks, entries flagged as meta/synthetic (e.g. an `isMeta` field, or a `toolUseResult` field at the entry level), and any assistant (`type == "assistant"`) entries entirely.

Use this script (adjust field names if a newer Claude Code version changed the schema — inspect a few raw lines first with `head -n 5 <file> | python3 -m json.tool` if extraction looks wrong):

```python
import json
import sys

transcript_path = sys.argv[1]
output_path = sys.argv[2]

def extract_text(content):
    if isinstance(content, str):
        return content.strip()
    if isinstance(content, list):
        texts = [
            block.get("text", "")
            for block in content
            if isinstance(block, dict) and block.get("type") == "text"
        ]
        return "\n".join(t for t in texts if t).strip()
    return ""

prompts = []
with open(transcript_path, "r", encoding="utf-8") as f:
    for line in f:
        line = line.strip()
        if not line:
            continue
        try:
            entry = json.loads(line)
        except json.JSONDecodeError:
            continue

        if entry.get("type") != "user":
            continue
        if entry.get("isMeta") or entry.get("toolUseResult") is not None:
            continue

        message = entry.get("message", {})
        content = message.get("content")

        # Skip entries that are only tool_result blocks (no real text block)
        if isinstance(content, list):
            has_text_block = any(
                isinstance(b, dict) and b.get("type") == "text" for b in content
            )
            has_only_tool_result = all(
                isinstance(b, dict) and b.get("type") == "tool_result" for b in content
            )
            if has_only_tool_result or not has_text_block:
                continue

        text = extract_text(content)
        if text:
            prompts.append(text)

with open(output_path, "w", encoding="utf-8") as out:
    for i, prompt in enumerate(prompts, 1):
        out.write(f"--- User prompt {i} ---\n{prompt}\n\n")

print(f"Extracted {len(prompts)} user prompt(s) to {output_path}")
```

Run it:

```bash
python3 export_user_prompts.py "<transcript_path>" "<output_path>"
```

(If `python3` isn't available, use `node` with equivalent logic, or `jq` for a simpler but less robust pass.)

## Step 3: Save the output

- Default filename: `user-prompts-export.txt`
- Default location: the current project root, unless the user specifies otherwise
- If the file already exists, ask before overwriting, or append a timestamp (`user-prompts-export-<YYYYMMDD-HHMMSS>.txt`)

## Step 4: Sanity check before confirming

Skim the output for anything that looks like it slipped through filtering:

- Long JSON blobs or tool-call syntax → extraction filter missed a non-text block; fix and re-run
- Assistant-sounding text ("I'll now...", "Here's the implementation...") → wrong `type` was included; fix and re-run
- System-injected wrapper tags (e.g. `<command-name>`, `<command-message>`, `<local-command-stdout>`) → these come from slash-command invocations the user triggered; leave them as-is unless the user asks for a stripped-down version, since the user *did* type the command

Report the final prompt count and file path to the user. Do not summarize, interpret, or comment on the content of the prompts themselves — this skill's only job is a faithful, unbiased export.
