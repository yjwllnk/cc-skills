---
name: postprocess-logs
description: Postprocess a coding-agent session (Claude Code or OpenAI Codex CLI) into a single session.txt inside the current WDIR. Use this at the END of any non-trivial workflow before declaring the task complete. Produces one file - <WDIR>/logs/session.txt - a hand-written summary header followed by the full, untruncated transcript. Trigger phrases - "wrap up", "we're done", "finalize", "post-process", "postprocess", "save the session", "log the session", "end of run", "before declaring done", or whenever you are about to give a final completion message for a task that ran in a WDIR under $HOME/.claude-dnjf/WDIR/, $HOME/.claude-mdil/WDIR/, or any WDIR used by a Codex run.
---

# postprocess-logs

Convert the active coding-agent session into a single `<WDIR>/logs/session.txt`: a hand-written summary header followed by the full, untruncated transcript. Works for two host agents:

- **Claude Code** — transcript jsonl under `$HOME/.claude*/projects/<slug>/*.jsonl`.
- **OpenAI Codex CLI** — rollout jsonl under `$HOME/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`.

The Python converter auto-detects which schema each line uses, so the same skill produces identical output for either host.

## When to run

Run this at the end of every workflow that uses a WDIR. Do not declare a task complete until `<WDIR>/logs/session.txt` exists and contains the summary header plus the transcript. The user has been burned by missed postprocessing before — treat it as part of the task, not optional cleanup.

Skip only if:
- the user explicitly says "no logs" / "skip postprocess",
- there is no WDIR for the run (one-off question, no files written),
- a current `logs/session.txt` already exists for this run.

## Inputs you need

1. `WDIR` — the working directory for this session. Common roots: `$HOME/.claude-dnjf/WDIR/<name>/`, `$HOME/.claude-mdil/WDIR/<name>/`, `$HOME/.llm/.openai-dnjf/WDIR/<name>/`, or any path the user has been writing into during a Codex run. If unset, ask the user or infer from where you have been writing files this session.
2. `SESSION_JSONL` — the raw transcript file for the current session. Read in place — it is never copied into `logs/`. Resolve based on host:
   - Claude Code: most recently modified `*.jsonl` under `<config-dir>/projects/<cwd-slug>/` — the converter globs every `.claude-*` config dir (`$HOME/.llm/.claude-*/projects/*/` and `$HOME/.claude-*/projects/*/`) and every project slug, so this resolves automatically.
   - Codex CLI: most recently modified `rollout-*.jsonl` under `${CODEX_HOME:-$HOME/.codex}/sessions/`. Common roots: `$HOME/.codex/sessions/`, or `$HOME/.llm/.openai-*/.codex/sessions/` when Codex is run with a per-account `CODEX_HOME`.

   If unsure which host you are running under, list candidates from both roots with `ls -t ... | head` and pick the file whose mtime matches this session's start; confirm with the user if ambiguous.

## Procedure

```bash
# 1. Resolve paths
WDIR="<resolved working dir>"          # e.g. $HOME/.claude-dnjf/WDIR/cluster-hw-survey

# Pick whichever line matches the active host. Override SESSION_JSONL manually
# if the auto-pick is wrong (e.g. multiple agents running at once).
# Honor $CODEX_HOME (defaults to ~/.codex); also glob ~/.llm/.openai-*/.codex
# for per-account Codex installs (e.g. ~/.llm/.openai-dnjf/.codex).
CODEX_SESS_DIRS=("${CODEX_HOME:-$HOME/.codex}/sessions" "$HOME"/.llm/.openai-*/.codex/sessions)

# Claude Code stores transcripts at <config-dir>/projects/<cwd-slug>/*.jsonl.
# Glob every .claude-* config dir (both ~/.llm/.claude-* and ~/.claude-*) and
# every project slug, so this resolves regardless of which host dir is active.
claude_latest="$(ls -t "$HOME"/.llm/.claude-*/projects/*/*.jsonl "$HOME"/.claude-*/projects/*/*.jsonl 2>/dev/null | head -1)"
codex_latest="$(find "${CODEX_SESS_DIRS[@]}" -type f -name 'rollout-*.jsonl' -printf '%T@ %p\n' 2>/dev/null | sort -rn | head -1 | cut -d' ' -f2-)"

# Default: pick the newest of the two (covers both Claude and Codex hosts).
SESSION_JSONL="$(ls -t "$claude_latest" "$codex_latest" 2>/dev/null | head -1)"

mkdir -p "$WDIR/logs"

# 1b. Rotate any existing session.txt so this run never overwrites a prior run.
#     Moves logs/session.txt (if present) into logs/prev-<UTC-timestamp>/.
rotate_prev_logs() {
  local logs_dir="$1"
  if [ -e "$logs_dir/session.txt" ]; then
    local ts archive
    ts="$(date -u +%Y%m%dT%H%M%SZ)"
    archive="$logs_dir/prev-$ts"
    mkdir -p "$archive"
    mv "$logs_dir/session.txt" "$archive/"
    echo "rotated prior log -> $archive/session.txt"
  fi
}
rotate_prev_logs "$WDIR/logs"
```

Then run the converter (Python is fine, no extra deps). It reads `$SESSION_JSONL` in place and writes the full transcript to `$WDIR/logs/session.txt`. Auto-detects Claude Code vs Codex CLI line schema. Nothing is truncated — `session.txt` is the only record of the run.

```bash
python3 - "$SESSION_JSONL" "$WDIR/logs/session.txt" <<'PY'
import json, sys, pathlib
src = pathlib.Path(sys.argv[1])
out_txt = open(sys.argv[2], "w")

def render_anthropic(c):
    if isinstance(c, str): return c
    if isinstance(c, list):
        parts = []
        for it in c:
            if isinstance(it, dict):
                t = it.get("type")
                if t == "text":
                    parts.append(it.get("text", ""))
                elif t == "thinking":
                    txt = it.get("thinking", "")
                    parts.append(f"[thinking]\n{txt}" if txt
                                 else "[thinking — not persisted by Claude Code; signature only]")
                elif t == "redacted_thinking":
                    parts.append("[redacted_thinking]")
                elif t == "tool_use":
                    parts.append(f"[tool_use {it.get('name')}({json.dumps(it.get('input', {}))})]")
                elif t == "tool_result":
                    c2 = it.get("content", "")
                    if isinstance(c2, list):
                        c2 = "".join(x.get("text", "") if isinstance(x, dict) else str(x) for x in c2)
                    parts.append(f"[tool_result {c2}]")
                else:
                    parts.append(json.dumps(it))
            else:
                parts.append(str(it))
        return "\n".join(parts)
    return str(c)

def render_codex_item(item):
    # OpenAI Responses-style payload from a Codex rollout `response_item` event.
    t = item.get("type")
    if t == "message":
        parts = []
        for blk in item.get("content") or []:
            if isinstance(blk, dict):
                parts.append(blk.get("text") or blk.get("input_text") or blk.get("output_text") or "")
            else:
                parts.append(str(blk))
        return "\n".join(p for p in parts if p)
    if t == "function_call":
        args = item.get("arguments") or ""
        if isinstance(args, dict): args = json.dumps(args)
        return f"[function_call {item.get('name')}({args})]"
    if t == "function_call_output":
        out = item.get("output") or ""
        if isinstance(out, (dict, list)): out = json.dumps(out)
        return f"[function_call_output {out}]"
    if t == "reasoning":
        parts = []
        for key in ("summary", "content"):
            v = item.get(key)
            if isinstance(v, list):
                parts.append(" ".join(s.get("text", "") if isinstance(s, dict) else str(s) for s in v))
            elif v:
                parts.append(str(v))
        return f"[reasoning {' '.join(p for p in parts if p)}]"
    return json.dumps(item)

for line in open(src):
    try: j = json.loads(line)
    except: continue
    t = j.get("timestamp") or j.get("time") or ""

    # --- Codex CLI rollout schema: top-level {timestamp, type, payload} ---
    if "payload" in j and isinstance(j.get("payload"), dict) and "message" not in j:
        ev = j.get("type", "?")
        payload = j.get("payload") or {}
        if ev in ("response_item", "ResponseItem"):
            item_t = payload.get("type", "?")
            role = payload.get("role", "")
            body = render_codex_item(payload)
            out_txt.write(f"--- {t} | codex.{item_t} | {role}\n{body}\n\n")
        else:
            out_txt.write(f"--- {t} | codex.{ev} |\n{json.dumps(payload)}\n\n")
        continue

    # --- Claude Code schema: top-level {type, message:{role,content}} ---
    typ = j.get("type", "?")
    msg = j.get("message", {}) or {}
    role = msg.get("role") or j.get("role") or ""
    content = msg.get("content")
    body = render_anthropic(content)
    out_txt.write(f"--- {t} | {typ} | {role}\n{body}\n\n")

out_txt.close()
print("ok")
PY
```

Finally, prepend the summary header. Write a real summary yourself (do not auto-generate it from the jsonl), then prepend it to `session.txt`:

```bash
{ cat <<'EOF'
================================================================
SESSION SUMMARY
================================================================
Date    : <YYYY-MM-DD>
WDIR    : <WDIR>
Host    : <Claude Code | Codex CLI>
Goal    : <one sentence>

What ran:
  1. <major step — tools used>
  2. ...

Outputs:
  - <path> — <one-line description>

Gotchas / unverified:
  - <anything the user should not blindly trust: drained nodes,
    spec-sheet-only numbers, missing data, etc.>
================================================================

EOF
cat "$WDIR/logs/session.txt"
} > "$WDIR/logs/session.txt.tmp" && mv "$WDIR/logs/session.txt.tmp" "$WDIR/logs/session.txt"
```

## Output checklist

After running, `<WDIR>/logs/` must contain exactly one artifact:

- `session.txt` — a plain-text `SESSION SUMMARY` header (hand-written) followed by the full, untruncated transcript: every event, every tool input/result, every user prompt inline, and Codex reasoning in full (Claude Code thinking shows a marker only — see Notes).

Verify with `ls -la "$WDIR/logs/"` and `head -40 "$WDIR/logs/session.txt"` before reporting completion.

## Notes

- The CLAUDE.md / AGENTS.md in `WDIR/` only contains a one-line pointer to this skill. Full procedure lives here so it does not load on every turn.
- `session.txt` is full fidelity — nothing is truncated, so the file can be large. That is intended: it is now the only record of the run (no raw jsonl is kept).
- Assistant thinking: Claude Code does **not** persist extended-thinking plaintext to its transcript (it keeps only a cryptographic `signature`), so for Claude Code runs `session.txt` shows `[thinking — not persisted by Claude Code; signature only]` markers — the reasoning text is unrecoverable post-hoc. Codex CLI rollouts do record `reasoning` items, so Codex thinking is captured in full.
- `SESSION_JSONL` is read in place, never copied. The host agent keeps its own copy under `$HOME/.claude*/projects/` (Claude Code) or `${CODEX_HOME:-$HOME/.codex}/sessions/` (Codex CLI — also `$HOME/.llm/.openai-*/.codex/sessions/` for per-account installs) if the raw transcript is ever needed.
- User prompts are not split into a separate file — they appear inline in the transcript, tagged `| user | user` in their event header.
- Prior-run `session.txt` is auto-rotated into `logs/prev-<UTC-timestamp>/` by `rotate_prev_logs` (step 1b) so re-running never overwrites earlier output. Clean these up manually when no longer needed.
- Codex rollout schema covered: `response_item` (with payload types `message`, `function_call`, `function_call_output`, `reasoning`) plus other event types (passed through as JSON). If a future Codex version adds new payload types, extend `render_codex_item` rather than special-casing in the main loop.
- Codex CLI now has native skills support — install this skill into `$CODEX_HOME/skills/postprocess-logs/` via the bundled `skill-installer` (`scripts/install-skill-from-github.py --repo yjwllnk/cc-skills --path skills/postprocess-logs`) and restart Codex. After install, Codex discovers the skill from frontmatter the same way Claude Code does; no `AGENTS.md` pointer is needed. The `AGENTS.md`-pointer fallback (`Before declaring done: follow the postprocess-logs procedure in <clone>/skills/postprocess-logs/SKILL.md`) still works for hosts that lack native skill loading.
