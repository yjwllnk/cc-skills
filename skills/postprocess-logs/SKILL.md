---
name: postprocess-logs
description: Postprocess a Claude Code session into a logs/ directory inside the current WDIR. Use this at the END of any non-trivial workflow before declaring the task complete. Produces session.jsonl + session.txt + summary.md + prompts.txt under <WDIR>/logs/. Trigger phrases - "wrap up", "we're done", "finalize", "post-process", "postprocess", "save the session", "log the session", "end of run", "before declaring done", or whenever you are about to give a final completion message for a task that ran in a WDIR under $HOME/.claude-dnjf/WDIR/ or $HOME/.claude-mdil/WDIR/.
---

# postprocess-logs

Convert the active Claude Code session into the four artifacts the user requires inside `<WDIR>/logs/`.

## When to run

Run this at the end of every workflow that uses a WDIR. Do not declare a task complete until `logs/` contains all four files. The user has been burned by missed postprocessing before — treat it as part of the task, not optional cleanup.

Skip only if:
- the user explicitly says "no logs" / "skip postprocess",
- there is no WDIR for the run (one-off question, no files written),
- a `logs/` directory with the four files already exists and is current.

## Inputs you need

1. `WDIR` — the working directory for this session. Usually `$HOME/.claude-dnjf/WDIR/<name>/` (legacy: `$HOME/.claude-mdil/WDIR/<name>/`). If unset, ask the user or infer from where you have been writing files this session.
2. `SESSION_JSONL` — the raw transcript file for the current session. Default location: the most recently modified `*.jsonl` under `$HOME/.claude-dnjf/projects/-home-jinvk--claude-dnjf/`. Pick the newest by `mtime`. If unsure, list candidates with `ls -t .../*.jsonl | head` and confirm.

## Procedure

```bash
# 1. Resolve paths
WDIR="<resolved working dir>"          # e.g. $HOME/.claude-dnjf/WDIR/cluster-hw-survey
PROJ_DIR="$HOME/.claude-dnjf/projects/-home-jinvk--claude-dnjf"
SESSION_JSONL="$(ls -t "$PROJ_DIR"/*.jsonl 2>/dev/null | head -1)"
mkdir -p "$WDIR/logs"

# 1b. Rotate any existing log artifacts so this run never overwrites a prior run.
#     Moves session.jsonl/session.txt/summary.md/prompts.txt (if present) into
#     logs/prev-<UTC-timestamp>/. Skips if none of the four files exist.
rotate_prev_logs() {
  local logs_dir="$1"
  local found=0
  for f in session.jsonl session.txt summary.md prompts.txt; do
    [ -e "$logs_dir/$f" ] && found=1 && break
  done
  if [ "$found" -eq 1 ]; then
    local ts
    ts="$(date -u +%Y%m%dT%H%M%SZ)"
    local archive="$logs_dir/prev-$ts"
    mkdir -p "$archive"
    for f in session.jsonl session.txt summary.md prompts.txt; do
      [ -e "$logs_dir/$f" ] && mv "$logs_dir/$f" "$archive/"
    done
    echo "rotated prior logs -> $archive"
  fi
}
rotate_prev_logs "$WDIR/logs"

# 2. Copy raw transcript
cp "$SESSION_JSONL" "$WDIR/logs/session.jsonl"
```

Then run the converter (Python is fine, no extra deps):

```bash
python3 - "$WDIR/logs" <<'PY'
import json, sys, pathlib
logs = pathlib.Path(sys.argv[1])
src = logs / "session.jsonl"
out_txt = open(logs / "session.txt", "w")
out_prompts = open(logs / "prompts.txt", "w")

def render(c):
    if isinstance(c, str): return c
    if isinstance(c, list):
        parts = []
        for it in c:
            if isinstance(it, dict):
                if it.get("type") == "text":
                    parts.append(it.get("text", ""))
                elif it.get("type") == "tool_use":
                    parts.append(f"[tool_use {it.get('name')}({json.dumps(it.get('input',{}))[:300]})]")
                elif it.get("type") == "tool_result":
                    c2 = it.get("content", "")
                    if isinstance(c2, list):
                        c2 = "".join(x.get("text", "") if isinstance(x, dict) else str(x) for x in c2)
                    parts.append(f"[tool_result {str(c2)[:500]}]")
                else:
                    parts.append(json.dumps(it)[:300])
            else:
                parts.append(str(it))
        return "\n".join(parts)
    return str(c)

for line in open(src):
    try: j = json.loads(line)
    except: continue
    t = j.get("timestamp") or j.get("time") or ""
    typ = j.get("type", "?")
    msg = j.get("message", {}) or {}
    role = msg.get("role") or j.get("role") or ""
    content = msg.get("content")
    body = render(content)
    out_txt.write(f"--- {t} | {typ} | {role}\n{body}\n\n")
    if typ == "user" and role == "user" and isinstance(content, (str, list)):
        text = body.strip()
        if (text
            and not text.startswith("[tool_result")
            and not text.startswith("<command-name>")
            and "<local-command" not in text[:50]):
            out_prompts.write(f"[{t}]\n{text}\n\n")

out_txt.close()
out_prompts.close()
print("ok")
PY
```

Finally write `summary.md` yourself (do not auto-generate from the jsonl — write a real summary). Required sections:

- Date + WDIR + host.
- Goal — one sentence.
- What ran — numbered list of major steps with tools used.
- Outputs — file paths + one-line description each.
- Gotchas / unverified — anything the user should not blindly trust (drained nodes, spec-sheet-only numbers, missing data, etc.).

## Output checklist

After running, `<WDIR>/logs/` must contain exactly:

- `session.jsonl` — raw transcript copy
- `session.txt` — human-readable conversion (every event, no truncation beyond per-tool-result 500 char cap)
- `summary.md` — your overview of the run (hand-written)
- `prompts.txt` — every user prompt with timestamp, no tool-result/command-name noise

Verify with `ls -la "$WDIR/logs/"` before reporting completion.

## Notes

- The CLAUDE.md in `WDIR/` only contains a one-line pointer to this skill. Full procedure lives here so it does not load on every turn.
- Memory file `feedback_postprocess_logs.md` reminds you this is required. Do not remove it.
- Tool-result truncation at 500 chars in `session.txt` is intentional — keeps the file readable. Raw `session.jsonl` preserves everything.
- Prior-run artifacts are auto-rotated into `logs/prev-<UTC-timestamp>/` by `rotate_prev_logs` (step 1b) so re-running never overwrites earlier output. Clean these up manually when no longer needed.
