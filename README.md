# cc-skills

Personal Claude Code skill bundle. Distributed via the plugin marketplace mechanism so behavior stays consistent across HPC / dev machines.

## Skills included

| Skill | What it does |
|---|---|
| `postprocess-logs` | At end of every workflow, write `<WDIR>/logs/{session.jsonl,session.txt,summary.md,prompts.txt}`. Required step before declaring any non-trivial task complete. Handles both Claude Code transcripts and OpenAI Codex CLI rollouts. |
| `pdf2willbook` | Convert PDF lecture notes into Will-format LaTeX book (B5 landscape, multicol, house style). Phase A scaffolds a new subject dir; Phase B fills chapters one at a time with exact transcription + vision figure extraction. |

## Install on a new machine

### Claude Code (native plugin)

Repo is private, so use the SSH URL form. The device must have an SSH key registered with GitHub:

```text
/plugin marketplace add git@github.com:yjwllnk/cc-skills.git
/plugin install cc-skills@cc-skills
```

Restart Claude Code after install for skill auto-discovery.

### OpenAI Codex CLI

Codex does not load Claude plugins, but it reads `AGENTS.md` files. To make `postprocess-logs` available inside a Codex run, clone the repo somewhere stable and add a pointer to your project (or global) `AGENTS.md`:

```bash
git clone git@github.com:yjwllnk/cc-skills.git ~/cc-skills
```

Then in the project's `AGENTS.md` (or `~/.codex/AGENTS.md` for global):

```markdown
## Postprocess at end of run

Before declaring any non-trivial task complete, follow the procedure in
`~/cc-skills/skills/postprocess-logs/SKILL.md`. It writes
`<WDIR>/logs/{session.jsonl,session.txt,summary.md,prompts.txt}` from the
current Codex rollout under `~/.codex/sessions/`.
```

The skill's Python converter auto-detects whether the source jsonl is a Claude Code transcript or a Codex rollout, so the same SKILL.md works for both hosts.

## Pre-install device check

Run this one-liner before `/plugin marketplace add`. Prints `On spot ...` if SSH to GitHub works, otherwise prints the underlying error:

```bash
ssh -T -o BatchMode=yes git@github.com 2>&1 | grep -q "successfully authenticated" && echo "On spot ..." || echo "SSH to GitHub failed — set up key first"
```

GitHub itself prints `Hi <user>! You've successfully authenticated...`; that string is server-controlled and cannot be customized. The wrapper above swallows it and prints `On spot ...` instead.

## Update flow

1. Edit `skills/<name>/SKILL.md` locally.
2. Bump `version` in `.claude-plugin/plugin.json` (semver).
3. `git commit && git push`.
4. On other devices: `/plugin update cc-skills@cc-skills`.

## Layout

```
cc-skills/
├── .claude-plugin/
│   ├── marketplace.json
│   └── plugin.json
├── skills/
│   ├── postprocess-logs/
│   │   └── SKILL.md
│   └── pdf2willbook/
│       └── SKILL.md
└── README.md
```
