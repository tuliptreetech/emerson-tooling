---
name: Emerson SoC Emulator
description: Use and control the Emerson hardware/SoC emulator via the single `emerson` command (`emerson load` / `emerson start` / `emerson ctl` / `emerson run`). Use when the user mentions Emerson, the emulator, emctl, flashing/loading firmware into a simulated chip, or inspecting/stepping/debugging emulated device state (registers, memory, breakpoints, device tree, brokers).
---

# Emerson SoC Emulator

Emerson is a Dockerized SoC emulator driven by one host command, `emerson`.
Runtime control is `emerson ctl <cmd>`; there is no separate `emctl`.

## Model

Four things, each outliving the one inside it:

1. **The install** — the `emerson` script and license, once per machine in
   the home directory (`emerson install`, `emerson update`).
2. **The project** — the nearest directory at or above the current one that
   holds `.emerson/`, found the way git finds `.git`. Commands work from the
   root or anywhere below it. `.emerson/` holds:
   - `config` — which image the project runs on. Committed. Written by
     `emerson init <tarball>` / `emerson image <tarball>`.
   - `files.yaml` — which file each device loads (your firmware), with paths
     relative to the project root. Committed. Written by `emerson init` /
     `emerson image` when there is none, or by `emerson files pull`.
   - `peripherals.yaml` — the off-chip hardware (optional; see below).
   - `local` — this machine's state, such as the recorded session. Not
     committed.
3. **The container** (`emerson-server`) — the project's image with the whole
   project root mounted read-only. Created only by `emerson load`, ended by
   `emerson shutdown`. **Only one project's container runs at a time.**
4. **Sessions** — running emulated machines. Each reads the files
   `files.yaml` names at the moment it is created. Several can run at once.
   `emerson start` / `emerson stop`.

Restarting a session takes about a second; restarting the container takes
several. So the edit-build-run loop is a rebuild followed by `emerson start`,
then `emerson ctl go` (a new session starts `paused` at tick 0).

## Commands

```
emerson init [tarball] [--force]     # make the current directory a project on that image (or set an existing project's image after cloning)
emerson image <tarball> [--force]    # change an existing project's image; restarts the container
                                     # init/image also write .emerson/files.yaml if there is none (an existing one is kept)
emerson files pull [--force]         # write .emerson/files.yaml from the image's template; --force for a fresh one after an image change
emerson load [--reload] [--force]    # container up on this project; --reload ends its running sessions,
                                     #   --force stops ANOTHER project's emulator and its sessions
                                     # load/start take --project N for images with several projects
emerson start [--new]                # session on the loaded project (see below); --new adds one and records it
emerson stop [token|--all]           # end a session; container stays up
emerson shutdown [--force]           # stop the container and every session in it
emerson sessions                     # running sessions, '*' = recorded
emerson info                         # version, image, container state, project root + files table, sessions
emerson version

emerson ctl <cmd> [args]             # one emulator command against the session (below)
emerson run [-p PORT] [--session T] <script.py> [args]   # Python script against the server (below)
emerson exec [cmd [args]]            # command inside emerson-server (bash if none)
emerson cli | emerson serial [chan]  # interactive REPL / serial console; need a TTY, so not for agents

emerson peripherals pull [path] [--force] [--catalog-only]
emerson peripherals clear            # single-firmware images only
emerson install                      # one-time: PATH and license only; images are per project (init)
emerson update                       # the host script only
emerson license set|show
emerson skills pull [--latest|--tag TAG]   # skills matching this install's version, into ./.claude/skills
emerson cleanup                      # unused script downloads and Docker images; asks before removing
```

### Setting up loading (`.emerson/files.yaml`)

`emerson init`, `emerson image` and `emerson files pull` write the same file:
one entry per device slot the model loads. Required
slots (the firmware) come filled in with the image's own file name, which
won't exist in your project: point them at the build output. Optional slots
are commented out, which keeps the file the image ships.

```yaml
/MEM/flash:
  image: build/firmware.bin    # relative to the project root
```

`emerson load` prints the resolved table (device, slot, path, `workspace` or
`image`, size, hash) or the reason a file can't load, and doesn't leave a
container running if it can't. `emerson load <firmware>` exists only for old
images that load a single firmware file; on a `files.yaml` image it is
refused.

### `emerson start` is the one command to re-run

- Container down → refuses and points at `emerson load`.
- Nothing changed → reattaches to the recorded session as it was.
- Any loaded file changed (a rebuild, in place or replaced), or `files.yaml`
  now points elsewhere → replaces the session. No `emerson load` needed.
- `files.yaml` or `peripherals.yaml` no longer parses, or a named file is
  missing → refuses before touching the running session.

Use `emerson load` only when the container is down, when another project's is
up, or after `emerson image`. It needs a stored license, and refuses while
sessions are running unless given `--reload`.

**One emulator at a time.** If the container belongs to another project, every
command here fails with `the emulator is running for the project at <path>`.
`emerson load --force` replaces it and ends that project's sessions, so ask
the user before using `--force`.

**Script and image versions should match.** `emerson update` changes only the
script; the image changes with `emerson image <tarball>`. `emerson info` shows
both.

## Sessions

Commands act on, in order: `--session TOKEN`, `$EMERSON_SESSION`, the session
`emerson start` recorded, or the only running session of this project (which
is then recorded). If several are running and none is recorded, they refuse
and list them. Sessions are independent of each other. `emerson start --new`
records the new session, so later commands go to it; reach the older one with
`--session`.

`emerson ctl start/stop/shutdown/sessions/set-project` are blocked. Use the
`emerson` verbs instead.

## `emerson ctl`

One-shot and non-interactive, so it is safe for agents and CI. Stdin is
forwarded, and output is byte-exact when redirected. It needs the container
up, and a session for everything except `project`, `peripherals` and
`--help`. `emerson ctl --help` is the full reference, and
`emerson ctl <cmd> --help` covers one command.

```
go [counter] | pause | step [n] | reset    # step blocks until done, then pauses; n advances `ticks` by exactly n
state                                      # lowercase: running | paused (also idle and halted-on-error states)
ticks                                      # tick counter (hex)
ls [path] | find [path] | dump             # device tree (absolute paths, e.g. /MEM/i2c0)
connections                                # pin wiring and analog signal bindings (declared, not proof the model drives it)
signal <path> [name]                       # list a device's analog signal sources/inputs, or read one now (with unit)
r <path> [reg [val]] | registers <path>    # named registers: prefer over read-mem for peripherals
pc <path> | ic <path> | u <path> | ui <path> | details <path>
read-mem <path> <addr> <len> [-w N] [-o FILE]
write-mem <path> <addr> (<hex>|--file F|--string S)   # memory devices only
db <path> <addr> [count] | write <path> <addr|reg+off> <hex>   # quick hex dump / poke; write takes e.g. r1+4
bp <path> [addr] | bpd | enable | disable  # breakpoint: CPU only, halts before the instruction
wp <path> [read|write <reg|addr>] | wpd    # watchpoint: counts hits, never halts
sp <path> [read|write <reg|addr>|fetch <addr>] | spd   # stoppoint: halts after the access
debugpoints [path]                         # every bp/wp/sp in the tree
actions <path> | action <path> <verb> [args]   # device-specific verbs (fault injection, stimuli)
broker | broker <name> [--limit N] | broker <name> <data>   # serial/data channels
broker --hex <name> <hex>                  # raw bytes; --hex goes before the name
logs [-f] [--level warn]
snap | snap save [name] | snap load <name>     # load takes the name without .snap
checkpoint [enable|disable]                     # needed for reverse stepping
os ps|set <pid>|unset|maps|modules|regs|scan   # only on projects with an os_handler; errors otherwise
files | check                              # resolved files table (as `load` prints) / processor-details check
project | peripherals [--bus B]
```

**`read-mem`/`write-mem` addresses are relative to `<path>`'s base.** Use
`/MEM/sram 0x484`, not `0x20000484`. On the CPU path the address space is the
whole system, so linked (ELF) addresses work as-is:
`read-mem <cpu-path> 0x20000490 4`.

## `emerson run`

```
emerson run tools/check.py --verbose      # options before the script, script args after
emerson run -p 8765 tools/live.py         # publish a port on host 127.0.0.1
emerson run --session <tok> tools/check.py
```

Runs `python3 <script>` in a throwaway container from the server's own image,
so the `emerson` library matches the server. Details:

- The current directory is mounted as the working directory. The script must
  be inside it, and files it writes land on the host.
- Sets `EMERSON_HOST` (`http://emerson-server:10314`; `localhost` is the
  script's own container), `EMERSON_PROJECT`, `EMERSON_BIND_HOST=0.0.0.0`,
  and `EMERSON_SESSION` when a session resolves. `emerson.attach_session()`
  reads the host, project and session. A script with its own `--host` should
  default it to `$EMERSON_HOST`.
- A served port needs `-p` and must listen on `0.0.0.0`, not `127.0.0.1`.
- It runs as the host uid, with no bytecode written.
- Needs the container up but not a session: with none running (or several
  and none recorded) `EMERSON_SESSION` is simply unset, so a script can start
  its own, and `attach_session()` raises. A token given with `--session` or
  `$EMERSON_SESSION` must be running.
- Only runs a script, so `-m pytest` won't work. Make the test file runnable
  instead:
  `if __name__ == "__main__": sys.exit(pytest.main([__file__, "-p", "no:cacheprovider", *sys.argv[1:]]))`.
- **Stopping:** Ctrl+C, or `docker kill -s INT <id>` with no terminal, raises
  `KeyboardInterrupt` and runs `finally`. `docker stop` (SIGTERM) exits
  without cleanup.

For the library itself, see the [emerson-python skill](../emerson-python/SKILL.md).

## Hardware configuration (`.emerson/peripherals.yaml`)

The chip's on-chip peripherals come with the image. Everything off-chip (the
sensors, switches, DACs, radios and cells on the board, and the pins and buses
they're wired to) is defined in `.emerson/peripherals.yaml`. Read the project's
file before changing firmware that touches hardware. Its comments are usually
the board's wiring documentation.

**Workflow:**

1. **Start from the image's file.** If the project has none, run
   `emerson peripherals pull`. It writes `.emerson/peripherals.yaml`: a
   header listing every controller and the `native:` kinds it accepts (with a
   line on what each part is), then the image's own file, whose commented
   examples show each bus's syntax. Use `--force` to overwrite and
   `--catalog-only` to just print. `emerson ctl peripherals [--bus spi]`
   prints the same catalog alongside the current file. The server reads the
   file from the project root, so it is used only at
   `.emerson/peripherals.yaml`: a copy pulled to a custom `[path]` does
   nothing until moved there. To go back to the image's own file, remove or
   rename it (`emerson peripherals clear` is for old single-firmware images).
2. **Edit entries under the controller's device-tree path.** Every entry has
   `name:` and `native: <kind>` from the catalog for *that* controller (on
   i2c, spi, adc and uart, `path: <python-file>` instead for a target written
   in Python). uart entries need nothing more.
   Then add the wiring its bus needs:

   ```yaml
   /MEM/i2c0:            # i2c: bus address
     - {name: eeprom, native: at24c256, address: 0x50}
   /MEM/spi0:            # spi: the part's own pins: cs_pin, plus busy_pin / dc_pin where it has them
     - {name: lcd0, native: ili9341, cs_pin: 17, dc_pin: 20}
   /MEM/adc:             # adc: channel
     - {name: pot, native: potentiometer, channel: 0}
   /MEM/sio:             # pin: a map of the part's pin names to GPIO numbers,
     - {name: stepper, native: drv8825, pins: {step: 2, dir: 3}}   # under the block that drives the pin (sio, or a pio)
   ```

   Parts on no bus go under the groups the image's file documents, such as
   `/board` (board electronics) and `/world` (physics models, e.g. a
   `syringe` with `driver:` and `ul_per_microstep:`). The file must stay one
   top-level mapping: delete the placeholder `{}` line when adding the first
   real entry.

   **The catalog lists kinds, not their options or `pins:` names.** Get those
   from the commented examples in the pulled file, an existing entry, or the
   error a new session reports (it names a missing required pin and the
   fields an entry accepts). Otherwise ask the user.
3. **Apply it.** An edit takes effect when the next session is created:
   `emerson stop && emerson start`, or `emerson start --new`. Plain
   `emerson start` reattaches to the old session and changes nothing, and no
   `emerson load` is needed.
4. **Verify.** Most mistakes now fail session creation with a
   `configuration error` naming the entry: an unknown key
   (``unknown key `x`: nothing reads it``), an unknown `native:` kind (it
   lists the valid ones), or a missing required pin (``needs `pins.dir` ``).
   After `emerson stop` that leaves no session running, so fix the file and
   run `emerson start` again. **An unknown name inside a `pins:` map is still
   silently ignored**, leaving that pin unwired. After every change, check:
   - `emerson ctl ls <controller>`: the device exists. i2c and adc targets
     appear under their *kind*, not their `name:`; spi and pin-wired parts
     use the name, and pin-wired parts sit under `/board`, not the
     controller.
   - `emerson ctl connections`: each pin you wired shows up.
   - The options took effect, not just that the device exists. Where
     `emerson ctl actions <device>` lists a status verb, read it; where the
     device has signals, read them with `emerson ctl signal <device>`.
     Otherwise check the effect downstream, such as the voltage the part
     drives into the next device.

Write the board facts into the file's comments next to each entry: part
number, pin functions, why an option has its value. That keeps the file the
single description of the hardware.

## Gotchas

- **A new session starts `paused`.** Run `emerson ctl go` after
  `emerson start` creates or replaces one, or nothing executes.
- **`emerson ctl step` blocks until the step is done** and leaves the machine
  `paused`, so reads after it are consistent. (The Python `Machine.step()`
  still returns early and needs `wait()`.) After `go`, wait for a breakpoint
  with `until [ "$(emerson ctl state)" = paused ]; do sleep 1; done`.
- **wp/sp hits count only accesses by the emulated CPU or bus.** A host-side
  `ctl r`/`write-mem` doesn't trigger them, so you can't test a stoppoint
  that way.
- **`sp` can overshoot**: `hit=` may already be a few past 1 when the
  machine halts.
- **wp/sp on a peripheral path can be created but not deleted or disabled**
  (`this device does not have debug points`); only a new session clears them.
  CPU-path ones delete normally.
- **Host register writes are not a reliable way to drive a peripheral.** Some
  registers refuse them (`write_stateless has not been implemented`). To
  inject a fault or stimulus, use the device's own `actions`.
- An unknown top-level `ctl` verb is an error; device verbs always need
  `ctl action <path> <verb>`.

## Debugging without instrumentation

Prefer breakpoints and memory reads over adding firmware logging:

1. Get symbol addresses from the ELF with `nm`. Re-run it after every
   rebuild, because addresses shift. Add field offsets by hand from the
   struct layout.
2. Set `bp` at the next function after the code of interest, run `go`, and
   wait for `paused`.
3. Read with `read-mem <cpu-path> <addr> <len>`. Check the technique first on
   a value you already know.

To prove an interrupt reaches firmware, check each layer: the destination
GPIO's input register flips after the device `action`; a `bp` on the ISR or
callback is hit; and its arguments name the pin you saw flip.
