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
emerson peripherals pull [path] [--force] [--catalog-only]  # pull the project's peripherals.yaml + I2C catalog from the image
emerson peripherals clear                               # clear the stored peripherals.yaml override so 'load' falls back to the image default (doesn't delete the file)
emerson load <firmware-file>                            # start emerson-server container w/ firmware (mounts .emerson/peripherals.yaml if present)
emerson exec [command [args]]                           # run a command in emerson-server (bash if omitted)
emerson shutdown                                        # tell the running emerson-server to shut down (host-side wrapper around emctl shutdown)
emerson update                                          # update the emerson/emctl scripts themselves
emerson cleanup                                         # remove old downloaded emerson/emctl script versions from ~/.emerson/downloads (NOT docker images — see 'image cleanup')
emerson image update <tarball>                          # docker load a tarball, restart server from it (was 'emerson update-image')
emerson image cleanup                                   # remove old Docker images loaded by install/'image update' (this is what 'cleanup' used to do)
emerson license set [KEY]                               # store/overwrite license key (prompts if omitted)
emerson license show                                    # show whether a key is stored (never prints it)
emerson skills pull                                     # download the latest Emerson Claude Code skills into .claude/skills in the cwd, overwriting what's there
emerson version                                         # print installed emerson version
emerson info                                            # version, image, container status, loaded firmware
emerson help
```

**Renamed/reshuffled from older versions of this doc:** `emerson update-image` is
now `emerson image update <tarball>`. `emerson cleanup` used to remove old
Docker images — it now removes old downloaded `emerson`/`emctl` *script*
versions from `~/.emerson/downloads`; the old cleanup-old-images behavior
moved to the new `emerson image cleanup`. If you want to reclaim disk space
from stale Docker images, use `image cleanup`, not `cleanup`.

`emerson exec` is the supported way to run something inside the `emerson-server`
container (`emerson exec` alone opens an interactive bash shell; add a command
and args to run it directly, e.g. `emerson exec python3 -c "..."`; stdin is
forwarded, so `emerson exec python3 -` works for piping in a script). Prefer it
over raw `docker exec ... emerson-server ...`.

Every command except `install`/`update`/`image`/`cleanup`/`version`/`info`/`help` checks for a newer release and nags to run `emerson update` if one exists.

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

`peripherals pull` takes an optional `[path]` (a directory gets
`peripherals.yaml` appended); either way it also points Emerson's stored
peripherals reference at that file for `emerson load` to pick up. To go back
to the image's default peripherals instead of your override, run
`emerson peripherals clear` — it clears the stored reference but does not
delete the override file itself.

To inspect the catalog/current peripherals from inside a running container
without needing a started project *session*, use `emctl peripherals` — see
the runtime section below.

## `emctl` — runtime control commands

Full built-in reference: `emctl --help` (only works while `emerson-server` is running; see Non-interactive note below). Global options: `--host HOST` (default `http://localhost:10314`), `--project NAME` (else `$EMERSON_PROJECT`, else `~/.emerson/config`).

**Config**
- `emctl project` — print current default project
- `emctl set-project [name]` — set default project (interactive picker if omitted)
- `emctl peripherals` — print the I2C peripheral catalog and the project's current `peripherals.yaml`; reads static config, so no `emctl start` session is needed (just the container running)

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
- `emctl connections` — list inter-device port/pin/net wiring, e.g. `gpioc.pin0.in -> /MEM/i2c1/charger.int` or an IRQ line into the NVIC. Peripheral-pin wiring is declared per-device in `.emerson/peripherals.yaml` under a `pins:` map (e.g. `pins: { int: { device: "/MEM/gpioc", pin: 0 } }`) and shows up here once configured. **A connection listed here is not proof the device model actually drives that pin** — on this board (Emerson 1.0.11), `charger.int`/`fuel_gauge.alrt` are wired to `gpioc.pin0`/`pin2` per this command, but triggering the condition (`inject_fault`, `set_soc` past the alert threshold) never moved the target GPIO's `IDR`, confirmed by a breakpoint on the firmware's fault-handling code never firing early (tracked as [emerson-issues#17](https://github.com/tuliptreetech/emerson-issues/issues/17)). Verify pin-level effects by reading the destination GPIO's `IDR` after triggering the condition, not just by checking `connections` output.

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
- `emctl write-mem <path> <addr> (<hex>|--file FILE|--string STR)` — same relative-to-`<path>` addressing as `read-mem`. **Memory devices only** (RAM/ROM/flash) — it writes fixed-width words sized to whatever device sits at `<addr>`, which isn't a real MMIO access path; a payload that overruns the target keeps writing into whatever's mapped next. For hardware registers, use `emctl r <path> <reg> <val>` instead.
- `emctl db <path> <addr> [count]` — read-only alternate to `read-mem` via the raw command language; hex/decimal `<addr>`, defaults to 16 bytes, always prints a hex dump (no `-w`/`-o`). Prefer `read-mem` when you want word-grouping or file output.
- `emctl write <path> <addr> <hex>` — raw-command-language alternate to `write-mem`; `<addr>` may also be a register+offset (e.g. `r1+4`, `r1-4`), but only takes a positional hex string (no `--file`/`--string`). Same memory-device-only caveat as `write-mem`.

**Debugpoints — machine-wide view**
- `emctl debugpoints [path]` — walks the whole device tree and lists every breakpoint, watchpoint, and stoppoint set anywhere (kind, target, enabled/disabled, hit count, access mode), optionally filtered to devices whose path starts with `<path>`. Devices with none set, or that don't support them, are omitted. Use this to get an overview across devices instead of checking `bp`/`wp`/`sp` one path at a time.

**Breakpoints / watchpoints / stoppoints** (require `<path>`) — see the dedicated
section below for semantics, id scoping, and a real deletion bug to watch for.
- `bp <path> [addr]` — fetch breakpoint, **CPU devices only**; halts *before* the instruction runs. `bpd <path> <id>`, `enable <path> <id>`, `disable <path> <id>`.
- `wp <path> [read|write <reg|addr>]` — watchpoint; records the access (`hit=` counter) but **never halts** the machine. `wpd <path> <id>`.
- `sp <path> [read|write <reg|addr>|fetch <addr>]` — stoppoint; halts *after* the access completes. `spd <path> <id>`.

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

## Breakpoints, watchpoints, and stoppoints — which to use

All three require a device tree `<path>` (`emctl ls`/`find` to locate one). What
they have in common: creating one prints `id=N`; listing (`bp`/`wp`/`sp` with no
further args) shows `id=N [Type] {...} (access) [REG] enabled hit=N`; `hit=`
only increments on a genuine access made *by the emulated CPU/bus* — see the
gotcha below, host-side pokes don't count. IDs are scoped **per device path**,
and `wp`/`sp` share one counter on a given path (e.g. on `/MEM/crc`, a `wp`
then an `sp` then another `wp` came back `id=0`, `id=1`, `id=2`); a different
device path starts its own counter at 0. Confirmed on Emerson 1.0.10.

Pick by what you're trying to catch:

- **`bp` (breakpoint)** — you know *which instruction* you want to stop at
  (an ELF symbol, a disassembled address) and want execution to stop *before*
  it runs. CPU devices only (`/Cortex-M0`) — pointing `bp` at a peripheral
  errors `this device does not have address break points`. Best for "stop
  when this function/line is reached," regardless of what data it's about to
  touch.
- **`wp` (watchpoint)** — you want to know *whether/how often* a register or
  address is touched, without perturbing timing. It never halts the machine,
  so it's safe to leave armed across a timing-sensitive stretch (watchdogs,
  bus timeouts) and check the `hit=` counter afterwards. Good for "is this
  register even read by the firmware" before spending time on a real
  breakpoint hunt.
- **`sp` (stoppoint)** — you want a hard stop *right after* a specific
  register/address is read, written, or fetched, but don't know (or don't
  want to hunt for) which instruction does it. Halts after the access
  completes, so the access has already happened when you inspect state —
  read the *new* value, not the pre-access one. Good for catching the first
  unexpected write to a region, or the exact moment a peripheral register
  changes during a fault-injection run (see custom device actions above,
  e.g. bq25892's `inject_fault`).

Confirmed by testing on this project's session (`stm32f030r8`, Emerson
1.0.10):

- A CPU-register watchpoint (`emctl wp /Cortex-M0 read r0`) accumulated real
  hits just from normal execution (`hit=5` within a second, since r0 is
  touched on nearly every call/return) — watchpoints do track genuine guest
  activity, not just theoretically.
- **Host-initiated register writes don't trigger wp/sp.** Arming
  `sp /MEM/crc write POL` and then writing that same register from the host
  via `emctl r /MEM/crc POL 0x7` left `hit=0` and the machine `running` —
  the stoppoint only fires on an access driven by the emulated CPU/bus, not
  on a debug-interface poke from `emctl r`/`write-mem`. Don't use `emctl r`
  to "test" that a stoppoint is wired up; you have to make the firmware do
  the access.
- **`sp` on a CPU device genuinely halts the machine** — arming
  `sp /Cortex-M0 read r0` then `emctl go` came back `paused` almost
  immediately (r0 is touched on nearly every call/return), confirming a
  stoppoint really stops execution rather than just logging. But **it can
  overshoot**: the `hit=` counter read right after the halt was `3` one run
  and `5` on a repeat, not `1` — a few extra matching accesses can happen
  before the halt is actually enforced, the same class of async/batching lag
  as the documented `step`-returns-early gotcha. Don't assume you're stopped
  at the *first* matching access; check `hit=` and treat a low overshoot as
  normal. (A stoppoint armed on a peripheral register — `/MEM/i2c1` `ISR`
  read — was left running for several minutes of wall-clock time without
  ever firing in this session; it's unclear whether that's because the
  firmware simply wasn't touching that register in that stretch, or a
  peripheral-specific quirk, so treat peripheral-scoped `sp` as unverified to
  actually halt until you've seen it happen for your own case.)
- **`wpd`/`spd`/`enable`/`disable` don't work against watchpoints or
  stoppoints set on a non-CPU peripheral device.** Creating and listing a
  `wp`/`sp` on a peripheral path (e.g. `/MEM/crc`, `/MEM/i2c1`) works fine,
  but deleting or disabling that same id errors
  `this device does not have debug points` — even though the id clearly
  exists in the `wp`/`sp` listing. The identical operation against a
  `/Cortex-M0`-scoped watchpoint (by id) deleted cleanly. No workaround was
  found short of ending the session (`emctl stop` + `emctl start`); a stray
  peripheral watch/stoppoint is otherwise harmless to leave in place (`wp`
  never halts, and an un-hit `sp` never halts either), but budget for not
  being able to remove it mid-session.

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
- `emerson update` only refreshes the `emerson`/`emctl` host scripts, not the running Docker image — use `emerson image update <tarball>` for that.
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
