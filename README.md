# jinvk-skills

Personal Claude Code skill bundle. Ships across machines via the plugin marketplace mechanism so behavior stays consistent on every HPC / dev box.

## Skills included

| Skill | What it does |
|---|---|
| `postprocess-logs` | At end of every workflow, write `<WDIR>/logs/{session.jsonl,session.txt,summary.md,prompts.txt}`. Required step before declaring any non-trivial task complete. |

## Install on a new machine

Once this repo is on GitHub at `yjwllnk/jinvk-skills`:

```text
/plugin marketplace add yjwllnk/jinvk-skills
/plugin install jinvk-skills@jinvk-skills
```

The marketplace and plugin share the same repo (`source: "./"` in `marketplace.json`), so a single repo serves both roles — same pattern as `caveman` and `tkm`.

## Update flow

1. Edit `skills/<name>/SKILL.md` locally.
2. Bump `version` in `.claude-plugin/plugin.json` (semver).
3. `git commit && git push`.
4. On other devices: `/plugin update jinvk-skills@jinvk-skills`.

## Layout

```
jinvk-skills/
├── .claude-plugin/
│   ├── marketplace.json
│   └── plugin.json
├── skills/
│   └── postprocess-logs/
│       └── SKILL.md
└── README.md
```

## Notes

- Skills auto-discover at session start; restart Claude Code after install.
- Trigger phrases for `postprocess-logs` are listed in its SKILL.md frontmatter `description` so Claude can match user intent.
