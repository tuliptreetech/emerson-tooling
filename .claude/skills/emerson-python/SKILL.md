---
name: Emerson Python Library
description: Script the Emerson emulator directly via its Python bindings (the `emerson` module installed inside the `emerson-server` container) instead of shelling out to individual `emctl` commands. Use when the user wants a Python script or REPL against Emerson, mentions `import emerson`, `EmulatorController`, `Connection`, `Machine`, `Device`, or needs control flow (polling loops, conditionals, parsing broker/UART data) that a single `emctl` call can't express.
---

# Emerson Python Library

`emerson-server` ships a Python package, `emerson`, that's a real binding to the
same emulator engine `emctl` drives over HTTP — not just a wrapper around the
CLI. It exposes strictly more than `emctl` does (e.g. checkpoint step-back,
`run_command_string`, blocking waits on debugger/serial events), and lets you
express loops/conditionals in one process instead of many `emctl` invocations.

Requires the [emerson skill](../emerson/SKILL.md)'s prerequisites: `emerson-server`
running (`emerson load ...`) and normally a project session started (`emctl start`).

## Getting a Python shell

The library only exists inside the container — there's nothing to `pip install`
on the host. Get to it with `emerson exec` (the host lifecycle tool's
convenience wrapper around `docker exec` into `emerson-server`, same container
`emctl` itself execs into):

```sh
emerson exec                                        # interactive shell, then: python3
emerson exec python3 -c "import emerson; help(emerson)"   # one-liner
emerson exec python3 - < local_script.py                  # run a host-side script without copying it in (stdin is forwarded)
```

Equivalent raw `docker exec -it/-i emerson-server ...` forms still work, but
`emerson exec` is the supported entry point — prefer it.

## Discovering the API

`help(emerson)` (or `help()` on any class/method below) is authoritative and
always reflects the installed version — treat this doc as a map, not a
reference. Module lives at
`/usr/local/lib/python3.12/dist-packages/emerson/__init__.py` in the image.

## Key classes

- **`EmulatorController(host)`** — entry point, e.g.
  `EmulatorController("http://localhost:10314")`. `.connect()` → `Connection`
  (context manager); `.shutdown()`.
- **`Connection`** — `run_project(name)` / `stop_project(session_id)`,
  `project_list()`, `get_instance_list()` (running sessions as
  `(session_id, name)` pairs), `attach(session_id)` → `Machine` (context
  manager; auto-detaches on exit).
- **`Machine`** — the live emulator: `go()`, `step(n)` (ticks, not
  instructions), `step_back(n)` (needs checkpointing), `pause()`, `reset()`,
  `state`/`is_paused()`/`is_running()`/`is_halted_on_error()`/`ignore_error()`,
  `tick_count`/`get_tick_counter()`, `go_to_counter(count)`,
  `get_device(path)` / `get_device_tree_paths()`, brokers
  (`read_from_broker`, `send_to_broker`, `get_broker_history`,
  `list_brokers`), snapshots (`save_snapshot_to_data_store`,
  `load_snapshot_from_data_store`, `list_snapshots`,
  `get_current_snapshot`/`load_snapshot_from_bytes`), checkpointing
  (`enable_checkpointing`/`disable_checkpointing`/`checkpointing_is_enabled`),
  `await_debugger_event()` / `await_serial_event()` (blocking waits),
  `run_command_string(command, path=None)`, `log(message, level=None)`.
- **`Device`** — one node in the device tree (from `get_device`): registers
  (`get_register`, `get_common_registers`, `registers`), memory
  (`get_memory(address, size)`, `set_memory(address, data)`),
  breakpoints/watchpoints/stoppoints (`break_on_address`,
  `watch_address_{read,write,fetch}`, `watch_register_{read,write}`,
  `stop_on_address_{read,write,fetch}`, `stop_on_register_{read,write}`, plus
  matching `enable_*`/`disable_*`/`delete_*`/`get_*`(s) accessors),
  disassembly (`get_disassembly`, `get_disassembly_with_pcode`), exceptions
  (`get_exception_names`, `get_any_raised_exceptions`,
  `enable_exception_halting`/`disable_exception_halting`),
  `translate_address(address)`, and **`invoke_action(action, args='')`** /
  `list_actions()` — the same custom board/peripheral verbs `emctl action`
  exposes, callable directly.
- **`Debugpoint`** / **`DebugpointInfo`** — handles returned by the
  breakpoint/watchpoint/stoppoint creators above (`id`, `target`, `access`,
  `enabled`, `hit_count`).

## Example: attach to an already-running session and poke at state

```python
from emerson import EmulatorController

with EmulatorController("http://localhost:10314").connect() as conn:
    # find the session emctl start created for your project
    session_id = next(sid for sid, name in conn.get_instance_list())
    with conn.attach(session_id) as machine:
        print(machine.state, machine.tick_count)

        gpioa = machine.get_device("/MEM/gpioa")
        print(gpioa.get_common_registers())

        # poll a register until a bit sets, instead of many `emctl r` calls
        while machine.is_paused():
            machine.step(1000)
        print(machine.state)

        print(machine.read_from_broker if False else machine.get_broker_history("tty0"))
```

## When to reach for this vs. `emctl`

Use `emctl` for one-off inspection and simple scripts — it's less ceremony and
already non-interactive/agent-friendly (see the [emerson skill](../emerson/SKILL.md)).
Reach for the Python API when you need:

- Control flow — polling a register/broker in a loop until a condition holds,
  branching on emulator state, retry logic.
- Many reads/actions combined in one process instead of N separate `docker exec`
  round-trips.
- Capabilities `emctl` doesn't surface at all: `step_back`, `run_command_string`,
  `await_debugger_event`/`await_serial_event`, direct snapshot bytes
  (`get_current_snapshot`/`load_snapshot_from_bytes`).
- Parsing broker/UART output programmatically rather than eyeballing raw bytes
  from `emctl broker <name>`.

## Gotchas

- The package is only inside the container's image — don't try to `pip install
  emerson` or import it on the host.
- `Connection.attach(session_id)` needs a session that already exists (same
  precondition as `emctl` needing `emctl start` first); find its id via
  `conn.get_instance_list()` rather than guessing.
- `Machine.step(n)` steps **ticks**, not instructions — that's a different
  unit than `emctl step [n]` (instructions). Don't assume parity between the
  two.
- `step_back` requires checkpointing enabled (`enable_checkpointing()`) —
  same requirement as `emctl checkpoint enable`.
