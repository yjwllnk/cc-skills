# cc-skills

Personal Claude Code skill bundle. Distributed via the plugin marketplace mechanism so behavior stays consistent across HPC / dev machines.

## Skills included

| Skill | What it does |
|---|---|
| `postprocess-logs` | At end of every workflow, write `<WDIR>/logs/{session.jsonl,session.txt,summary.md,prompts.txt}`. Required step before declaring any non-trivial task complete. |

## Install on a new machine

Repo is private, so use the SSH URL form. The device must have an SSH key registered with GitHub:

```text
/plugin marketplace add git@github.com:yjwllnk/cc-skills.git
/plugin install cc-skills@cc-skills
```

Restart Claude Code after install for skill auto-discovery.

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
│   └── postprocess-logs/
│       └── SKILL.md
└── README.md
```
