---
name: Emerson SoC Emulator
description: Use and control the Emerson hardware/SoC emulator via the `emerson` (lifecycle) and `emctl` (runtime control) commands. Use when the user mentions Emerson, the emulator, emctl, flashing/loading firmware into a simulated chip, or inspecting/stepping/debugging emulated device state (registers, memory, breakpoints, device tree, brokers).
---

# Emerson SoC Emulator

Emerson is a Dockerized SoC emulator. Two CLIs, both installed to `~/.local/bin`:

- **`emerson`** — host-side lifecycle tool. Installs/updates Emerson, loads a firmware image, starts/stops the `emerson-server` Docker container, and runs commands inside it.
- **`emctl`** — runtime control tool that talks to the running server (over HTTP, default `http://localhost:10314`). Used to start/stop emulation *sessions*, inspect and mutate device state, set breakpoints, read logs, etc. `emctl` is a thin host wrapper (`emerson/bin/emctl` script, ~38 lines) that shells out to `docker exec -it emerson-server emctl "$@"` — the real implementation lives inside the container image.

Current state (version, image, container status, loaded firmware) is set up by
`install`/`load` and stored in `~/.emerson/config` (bash-sourceable), but don't
read that file directly to check state — use `emerson info` instead:

```
$ emerson info
Emerson version: 1.0.7
Docker image: ghcr.io/tuliptreetech/emerson/stm32f030r8:1.0.7-local
Container (emerson-server): running
Firmware loaded: /Users/tuliptree/flash.bin
  Timestamp: Aug 13 16:38:51 2026
  MD5 sum:   a3ad62cf62a71c24b580cef867e706cf
```

For the current *project* (not shown by `info`), use `emctl project` rather
than grepping the config file. The config's `emerson_project` key was
`emerson_model` in older installs — same slot, and it should match the
project name test suites pass to `run_project`/`--project`, e.g. `stm32f030r8`.

## Mental model / lifecycle

```
emerson install            # one-time: PATH+symlinks, license, pull docker image
emerson load ./flash.bin   # start the emerson-server container with a firmware image
emctl start                # start an emulator *session* for the configured project (paused at reset)
emctl go / step / ...      # drive execution, inspect state
emctl stop                 # end the session (container keeps running)
emerson exec ...           # run a command inside the running container (bash, python3, etc.)
emerson server keeps running until the container is stopped/removed
```

A running Docker container (`emerson-server`) is a prerequisite for every `emctl` command except when the wrapper itself errors ("Emerson Server is not running... run 'emerson load'"). Within that server, a *project session* (`emctl start`) is a prerequisite for nearly all `emctl` subcommands except `project`, `set-project`, `shutdown`.

## `emerson` — host lifecycle commands

```
emerson install [--force-license-key] [--force-pull]   # set up PATH/symlinks, license, pull image
emerson load <firmware-file>                            # start emerson-server container w/ firmware
emerson exec [command [args]]                           # run a command in emerson-server (bash if omitted)
emerson update                                          # update the emerson/emctl scripts themselves
emerson update-image <tarball>                          # docker load a tarball, restart server from it
emerson cleanup                                         # remove old images loaded by install/update-image
emerson license set [KEY]                               # store/overwrite license key (prompts if omitted)
emerson license show                                    # show whether a key is stored (never prints it)
emerson version                                         # print installed emerson version
emerson info                                            # version, image, container status, loaded firmware
emerson help
```

`emerson exec` is the supported way to run something inside the `emerson-server`
container (`emerson exec` alone opens an interactive bash shell; add a command
and args to run it directly, e.g. `emerson exec python3 -c "..."`; stdin is
forwarded, so `emerson exec python3 -` works for piping in a script). Prefer it
over raw `docker exec ... emerson-server ...`.

Every command except `install`/`update`/`update-image`/`cleanup`/`version`/`info`/`help` checks for a newer release and nags to run `emerson update` if one exists.

## `emctl` — runtime control commands

Full built-in reference: `emctl --help` (only works while `emerson-server` is running; see Non-interactive note below). Global options: `--host HOST` (default `http://localhost:10314`), `--project NAME` (else `$EMERSON_PROJECT`, else `~/.emerson/config`).

**Config**
- `emctl project` — print current default project
- `emctl set-project [name]` — set default project (interactive picker if omitted)

**Session/server management**
- `emctl start` — start a new session for the project (only command, besides `stop`/`shutdown`, that works with no session yet)
- `emctl stop` — stop the session
- `emctl shutdown` — shut down the emulator server entirely

**Execution control**
- `emctl go [counter]` — resume; optional hex tick count to run until
- `emctl pause`
- `emctl step [n]` — step n instructions (decimal or `0x` hex), default 1.
  **Returns before the step finishes** — see the gotcha below before reading
  state afterwards.
- `emctl reset` — reset to initial state

**Status**
- `emctl state` — prints `running`, `paused`, or `halted on error`. All lowercase, and the last one is spaced, not camel-cased — match it exactly if a script compares against it.
- `emctl ticks` — current tick counter (hex)

**Device tree** (paths are absolute, e.g. `/PXA270`, `/PXA270/core0`, `/system/uart0`)
- `emctl ls [path]` — list children (default `/`)
- `emctl find [path]` — device tree as JSON
- `emctl dump` — all devices + common register values

**Snapshots**
- `emctl snap` / `emctl snap save [name]` / `emctl snap load <name>` (name = timestamp if omitted; no `.snap` extension in `load`)

**Checkpointing** (required for reverse stepping)
- `emctl checkpoint` / `emctl checkpoint enable` / `emctl checkpoint disable`

**Data brokers** (named async I/O channels, e.g. UART TTYs, external displays)
- `emctl broker` — list channel names
- `emctl broker <name>` — read available bytes (raw to stdout; `--limit N`)
- `emctl broker <name> <data>` — write a UTF-8 string

**Logs**
- `emctl logs` / `emctl logs -f` (stream) / `emctl logs --level warn` (debug<info<warn<error)

**OS awareness** (needs project's `os_handler` configured)
- `emctl os ps` / `os set <pid>` / `os unset` / `os maps [pid]` / `os modules` / `os regs [pid]` / `os scan`

**Per-device** (require `<path>`)
- `emctl r <path> [reg]` — print registers, or one; `emctl r <path> <reg> <val>` to set (val can be a number or another register name)
- `emctl registers <path>` — all registers
- `emctl pc <path>` / `emctl ic <path>` — program/instruction counter (CPU only)
- `emctl details <path>` — kind, memory, registers
- `emctl u <path>` / `emctl ui <path>` — disassemble next 10 instrs at PC (`ui` adds p-code)
- `emctl read-mem <path> <addr> <len> [-w N] [-o FILE]` — hex dump or raw write to FILE; `-w` groups into N-byte little-endian words
- `emctl write-mem <path> <addr> (<hex>|--file FILE|--string STR)`

**Breakpoints / watchpoints / stoppoints** (require `<path>`)
- `bp <path> [addr]`, `bpd <path> <id>`, `enable <path> <id>`, `disable <path> <id>`
- `wp <path> [read|write <reg|addr>]`, `wpd <path> <id>`
- `sp <path> [read|write <reg|addr>|fetch <addr>]` — halts *after* the access; `spd <path> <id>`

**Custom device actions** (board/peripheral models expose their own verbs)
- `emctl actions <path>` — list verbs + usage
- `emctl action <path> <verb> [args...]` — invoke one (args joined w/ spaces, parsed by the device)

## Non-interactive / scripted / agent use

The only `emctl` subcommand that reads stdin is a bare `set-project` with no name arg (it prompts with a numbered picker via Python's `input()`). Everything else is one-shot and non-interactive by design (per its own `--help`: "designed for scripting, automation, and LLM agent use"), so ordinary commands (`emctl state`, `emctl ls`, `emctl set-project <name>`, etc.) run fine with no TTY — just call `emctl <args...>` directly, including from agents/CI. `emctl set-project` with no name needs a real terminal, since there's no other way to answer the prompt.

## Typical session, end to end

```bash
emerson load ./flash.bin       # start container (needs valid license)
emctl start                    # begin a paused session for the configured project
emctl state                    # -> paused
emctl ls                       # top-level device tree, e.g.: M0Cpu  MEM
emctl find                     # full tree as path/kind pairs, with offsets for addressed devices
emctl pc /M0Cpu
emctl bp /M0Cpu 0x08001234
emctl go
emctl logs -f                  # Ctrl+C to stop streaming
emctl stop
```

## Gotchas

- Almost every `emctl` command needs both: (1) `emerson-server` container running (`emerson load`), and (2) a project session started (`emctl start`). The error messages name exactly which precondition is missing — read them, don't guess.
- `emerson load` requires a valid stored license (`emerson license show`/`emerson license set`).
- `action` vs top-level commands: an unrecognized top-level verb is a hard error, never silently treated as a device action — you must type `emctl action <path> <verb>` explicitly. Use `emctl actions <path>` first to see what a device supports.
- `snap load <name>` takes the snapshot name *without* the `.snap` extension.
- `sp` (stoppoint) halts the emulator *after* the access completes, unlike a breakpoint which halts before executing.
- **`emctl step` returns before the step has finished.** It dispatches the command and exits while the emulator is still executing, so anything you read immediately afterwards may be sampled mid-step. Poll `emctl state` until it reports `paused` before inspecting:

  ```bash
  emctl step 2000000
  until [ "$(emctl state)" = "paused" ]; do sleep 1; done
  emctl ticks   # only now is this a settled value
  ```

  Small steps hide this: they finish faster than the next `docker exec` round trip (~250 ms), so `emctl step 100` looks perfectly synchronous. Scale up and it stops being. Measured on `stm32f030r8` 1.0.7 — after `emctl step 2000000` the call returned in 284 ms, `emctl state` reported `running`, and three successive `emctl ticks` gave `0x407a5`, `0x62e6d`, `0x8032d`. After polling to `paused`, three reads all gave `0x3d3075`.

  This bites hardest when you read **two or more** locations per step and compare them: each read lands at a different point in emulated time, so a correlation between two counters can be destroyed (or manufactured) by the sampling alone. Tracked as [emerson-issues#11](https://github.com/tuliptreetech/emerson-issues/issues/11).
- `emerson update` only refreshes the `emerson`/`emctl` host scripts, not the running Docker image — use `emerson update-image <tarball>` for that.

## Scripting beyond `emctl`

For control flow (polling loops, conditionals) or capabilities `emctl` doesn't
expose (checkpoint step-back, blocking waits on debugger/serial events, etc.),
the emulator also has a Python binding installed inside the `emerson-server`
container — see the [emerson-python skill](../emerson-python/SKILL.md).
