# Emerson Tooling

Claude Code skills for working with [Emerson](https://github.com/tuliptreetech/emerson) — the `emerson`/`emctl` CLIs and the Python library that ship with the install.

## What's here

- **`.claude/skills/emerson`** — the `emerson` (lifecycle) and `emctl` (runtime control) CLIs: installing, loading firmware, starting sessions, inspecting/stepping device state.
- **`.claude/skills/emerson-python`** — the Python bindings inside the `emerson-server` container, for scripting beyond what `emctl` exposes.

## Using these skills

After you have emerson installed, you can run `emerson skills pull`. That will put these files in the `$cwd/.claude/skills` directory.

Or you can copy (or symlink) the `.claude/skills/emerson` and `.claude/skills/emerson-python` directories into your own project's `.claude/skills/` folder manually.

Claude Code will pick them up automatically.

Questions or issues? Join [Tulip Tree Tech Community](https://tuliptreetechcomm.slack.com) on Slack, or file a bug at [emerson-issues](https://github.com/tuliptreetech/emerson-issues).
