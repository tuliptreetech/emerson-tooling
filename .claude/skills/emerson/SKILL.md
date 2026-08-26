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
emerson peripherals pull   # optional: pull .emerson/peripherals.yaml, trim to devices you care about
emerson load ./flash.bin   # start the emerson-server container with a firmware image (mounts peripherals.yaml if present)
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
emerson peripherals pull [--force] [--catalog-only]     # pull the project's peripherals.yaml + I2C catalog from the image
emerson load <firmware-file>                            # start emerson-server container w/ firmware (mounts .emerson/peripherals.yaml if present)
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

## I2C peripherals (`.emerson/peripherals.yaml`)

`emerson peripherals pull` reads the current project's peripheral config and
I2C catalog straight out of the loaded Docker image — no running container or
session needed. It writes `.emerson/peripherals.yaml` in the current
directory (fails if one's already there; pass `--force` to overwrite). Pass
`--catalog-only` to just print the available native kinds without writing
anything.

The written file lists, per I2C controller device-tree path (e.g.
`/MEM/i2c1`), every device currently attached plus a comment block of native
kinds that controller accepts:

```yaml
/MEM/i2c1:
  - name: bq25892
    address: 0x6B
    native: bq25892
  - name: max17043
    address: 0x36
    native: max17043
  - name: eeprom
    address: 0x50
    native: at24c256
```

Each entry needs `name`, `address` (hex), and either `native: <kind>` (one of
the catalog kinds for that controller) or `path: <python-file>` for a custom
target. **To scope the emulated bus down to only the devices you care about,
edit this file directly** — delete the entries you don't need, keep/add the
ones you do. `emerson load` automatically mounts `.emerson/peripherals.yaml`
into the container when it's present in the cwd, so edits take effect on the
next `emerson load` (+ `emctl start`); no need to re-pull or rebuild anything.

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
- `emctl r <path> [reg]` — print registers, or one; `emctl r <path> <reg> <val>` to set (val can be a number or another register name). **Prefer this over `read-mem`/`write-mem` for named peripheral registers** (GPIO MODER/PUPDR/IDR/ODR, timers, etc.) — `emctl r <path>` with no reg lists every named register on that device, so there's no need to hand-compute byte offsets the way `read-mem`/`write-mem` require. Reserve `read-mem`/`write-mem` for genuinely address-based memory (SRAM/flash contents, GDDRAM-style framebuffers) that has no named-register abstraction.
- `emctl registers <path>` — all registers
- `emctl pc <path>` / `emctl ic <path>` — program/instruction counter (CPU only)
- `emctl details <path>` — kind, memory, registers
- `emctl u <path>` / `emctl ui <path>` — disassemble next 10 instrs at PC (`ui` adds p-code)
- `emctl read-mem <path> <addr> <len> [-w N] [-o FILE]` — hex dump or raw write to FILE; `-w` groups into N-byte little-endian words. **`<addr>` is relative to `<path>`'s own base, not an absolute system address** — e.g. `emctl read-mem /MEM/sram 0x484 4` (offset into that 0x2000-byte device), not `0x20000484`; the latter errors "out of range" against the device's own (small) size. The one path where relative-to-base and absolute happen to coincide is the CPU (`/Cortex-M0` or similar) — its address space starts at 0 and covers the whole system, so full linked addresses (from an ELF's symbol table, vector table, etc.) can be passed straight through: `emctl read-mem /Cortex-M0 0x20000490 4`.
- `emctl write-mem <path> <addr> (<hex>|--file FILE|--string STR)` — same relative-to-`<path>` addressing as `read-mem`.

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
- **GPIO register writes that change pin drive are rejected outright**, via either `write-mem` or `emctl r <path> <reg> <val>` — e.g. clearing `PUPDR` bits to fake a broken pull-up/corroded connector fails with `Error while writing to device gpiob: a write here changes what the pins drive onto the external circuit`. This is a deliberate guardrail (the emulator models the electrical consequence of the write), not a bug, and it isn't bypassed by using one command over the other. There's no supported way to fault-inject at the raw GPIO/bus-electrical level for this board; use a peripheral's own `emctl actions <path>` fault-injection verbs instead (e.g. bq25892's `inject_fault ntc_hot/ntc_cold/...`) to simulate a damaged/glitching sensor.

## Debugging firmware state without instrumentation

Prefer breakpoints + direct register/memory inspection over adding temporary
`printf`/UART logging to the firmware under test. It's faster to iterate
(no rebuild-reflash cycle per guess), and it doesn't risk the debug code
itself perturbing timing-sensitive behavior (watchdogs, bus timeouts) you're
trying to diagnose.

General recipe for "is this C global/struct field what I expect it to be right
now":

1. **Find the address.** Symbols aren't loaded into `emctl` — get them from the
   ELF instead: `arm-none-eabi-nm build/firmware.elf | grep -i <symbol>`. For a
   struct field (e.g. `hi2c1.ErrorCode`), `nm` only gives you the struct's base
   address; add the field's byte offset by hand from the struct's typedef
   (count each member's size, respecting natural alignment — e.g. on a Cortex-M0
   `HAL_StatusTypeDef`/enum members are 4 bytes, pointers are 4 bytes, and a
   `uint16_t` pair packs into 4 bytes without padding). Recompute this offset
   fresh after any rebuild if the struct layout could plausibly have changed —
   but note **the addresses of file-scope globals themselves can also shift
   between rebuilds** (a change elsewhere in the same translation unit,
   or even in an unrelated file, can shift `.bss`/`.data` layout), so re-run
   `nm` for the base symbol after every rebuild rather than assuming it's
   stable — don't just reuse offsets computed against a stale build.
2. **Set a breakpoint past the code you care about**, e.g. at the entry of the
   next function called after it (`emctl bp /Cortex-M0 <addr>`), then
   `emctl go` and wait for `emctl state` to report `paused`.
3. **Read the value** with `emctl read-mem /Cortex-M0 <addr> <len>` (see the
   `read-mem` addressing note above — use the CPU device path so linked/ELF
   addresses work unmodified). Sanity-check the technique on a known-good
   value first if the result looks surprising (e.g. read a handle's
   `Instance` pointer field and confirm it equals the peripheral's known base
   address, like `0x40005400` for `I2C1` on an F0) before trusting a field
   you don't have an independent way to verify.

Worked example: an I2C driver was silently failing (no errors surfaced, but
nothing appeared on a simulated OLED). Rather than adding `printf` calls,
`arm-none-eabi-nm` found `hi2c1`/`hi2c2` (`I2C_HandleTypeDef` handles), a
breakpoint was set just after the failing calls, and `read-mem` on each
handle's `ErrorCode` field (offset 76 into the struct: `Instance` (4) +
`Init` (8 × `uint32_t` = 32) + `pBuffPtr` (4) + `XferSize`+`XferCount`
(2+2) + `XferOptions` (4) + `PreviousState` (4) + `XferISR` (4) + `hdmatx`
(4) + `hdmarx` (4) + `Lock` (4) + `State` (4) + `Mode` (4) = 76) showed
`0x00000004` — `HAL_I2C_ERROR_AF` (ack failure). That pointed straight at an
electrical cause (missing GPIO pull-ups on the I2C pins) instead of a
protocol/logic bug, without touching the firmware at all.

## Scripting beyond `emctl`

For control flow (polling loops, conditionals) or capabilities `emctl` doesn't
expose (checkpoint step-back, blocking waits on debugger/serial events, etc.),
the emulator also has a Python binding installed inside the `emerson-server`
container — see the [emerson-python skill](../emerson-python/SKILL.md).
