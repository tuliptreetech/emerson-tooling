---
name: emerson-js
description: Build an Emerson view, a single HTML page in a project's `.emerson/views/` that Franklin lists and runs against the open session, using the Emerson JS library (`/lib/emerson.js`, `attach()`, `Session`, `Device`). Use when the user wants a live page, dashboard, control panel or "view" of an emulated device, mentions emerson.js, Franklin views, `.emerson/views`, `attach()`, `readSignal`/`invokeAction` from a browser, or wants to port a Python view server (`emerson run -p … tools/*_view.py`) to a view.
---

# Emerson JS views

A view is **one self-contained HTML file** in `<project root>/.emerson/views/`.
Franklin (`http://localhost:10314`) lists it under the open session's views and
serves it at

```
http://localhost:10314/views/{project}/{token}/workspace/{file}.html
```

The page talks to that session through the Emerson JS library. There's no
server to start, no `emerson run`, no build step and no npm package. Put the
CSS and JS inline, import only `/lib/emerson.js`, and draw charts yourself on a
`<canvas>` so the view works offline. Franklin would serve sibling files, but
keep a view to the one `.html`.

The library's own docs are authoritative; read them before writing anything
the map below doesn't cover:

```sh
curl -s http://localhost:10314/lib/emerson.md                  # the guide
curl -s http://localhost:10314/lib/types/session.d.ts          # Session, Device, readings
curl -s http://localhost:10314/lib/types/index.d.ts            # everything exported
curl -s http://localhost:10314/api/views/{project}             # what Franklin lists
```

Use the [emerson skill](../emerson/SKILL.md) for `emerson ctl` to find what to
read. The JS names follow the [Python library](../emerson-python/SKILL.md)
(`invoke_action` is `invokeAction`).

## Skeleton

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Pump Battery</title>   <!-- the name Franklin lists -->
<style>
:root { --bg: #f6f7f9; --ink: #1b1f24; --muted: #5d6673; }
@media (prefers-color-scheme: dark) { :root { --bg: #111418; --ink: #e6e9ed; --muted: #98a2ae; } }
body { margin: 0; background: var(--bg); color: var(--ink); font: 14px/1.4 system-ui, sans-serif; }
</style>
</head>
<body>
<div id="banner"></div>
<main>VBAT <b id="vbat">–</b> <span id="state"></span> <button id="unplug">Unplug USB</button></main>
<script type="module">
import { attach } from "/lib/emerson.js";

async function main() {
  let m;
  try {
    m = await attach();                         // the session in the page's address
  } catch (e) {                                 // EmersonError: no such session
    banner.textContent = `No session: ${e.message}`;
    return;
  }
  const battery = m.device("/world/battery");   // free until you call it
  m.onState(({ state }) => (document.getElementById("state").textContent = state.name ?? state));
  m.every(500, async () => {                    // never overlaps; paused while the tab is hidden
    const { value, unit, seconds } = await battery.readSignal("rail");
    vbat.textContent = `${value.toFixed(3)} ${unit} @ ${seconds?.toFixed(1)} s`;
  });
  unplug.onclick = () => m.device("/world/usb").invokeAction("outage");
}
main();   // no top-level `return` in a module: wrap the body in a function
</script>
</body>
</html>
```

## API map

- **`attach(id?, timeoutMs = 5000)`** resolves once the first heartbeat has
  arrived. It takes the session from the page's URL; `?project=…&session=…`
  overrides that.
- **`Session`**:
  - Run control: `go()`, `pause()`, `reset()`, `step(n)` and `stepBack(n)` in
    ticks (they return at once), `goToCounter(n)`.
  - Heartbeat: `state` (`"Running"`, `"Paused"`, `"Idle"`, `"BreakpointHit"`),
    `isRunning()` (true for Running and Idle), `tickCount` (a bigint),
    `guestSeconds`, `guestTime()` (asks now), and `onState(fn)`, which calls
    `fn` on every heartbeat and once at once.
  - Devices: `device(path)`, `devices()` (`{ path, kind, hasActions }`, so a
    view that works for any project can find its parts by `kind`),
    `devicePaths()`.
  - Brokers: `listBrokers()`, `subscribeBroker(name, fn)` (bytes from now on;
    returns a stop function), `sendToBroker(name, data)`.
  - Polling: `every(ms, task)`. A task that throws is only logged to the
    console, so catch errors yourself if the page should show them.
  - Escape hatches: `command()`, `commandString("signals /world/battery")`.
- **`Device`**: `invokeAction(name, args = "")` returns the text the action
  printed. Also `listActions()`, `signalDescriptors()`, `readSignal(name)`
  (gives `{ value, unit, tick, seconds }`), `getRegister`/`setRegister`
  (bigint), and `getMemory`/`setMemory` (memory devices only).

## Patterns

- **Read everything in one `Promise.all`** per poll. A part that may not be
  fitted gets its own fallback, so a missing part doesn't fail the whole poll:
  ```js
  const [charge, rail, chg] = await Promise.all([
    batt.invokeAction("get_charge"), batt.readSignal("rail"),
    charger.getRegister("REG0B_SysStatus").then(r => Number((r >> 3n) & 3n), () => null),
  ]);
  ```
- **Actions answer in text.** Parse `key=value` output with
  `Object.fromEntries([...txt.matchAll(/(\w+)=([-\d.]+)/g)].map(x => [x[1], +x[2]]))`.
  Run `emerson ctl action <path> <name>` first to see the exact format.
- **Read a UART line by line** with a streaming decoder. Subscribe once, at
  start-up; it gets only output written from then on.
  ```js
  const dec = new TextDecoder(); let buf = "";
  m.subscribeBroker("tty0", bytes => {
    buf += dec.decode(bytes, { stream: true });
    const lines = buf.split("\n"); buf = lines.pop();
    for (const line of lines) handle(line.replace(/\r$/, ""));
  });
  ```
- **Console output from before the page opened** (a boot banner, the last
  status line) isn't in the library. `commandString("broker tty0")` and the
  typed `read_broker` come back with no bytes, even though `emerson ctl broker tty0`
  shows the history. Franklin's own console reads it from an undocumented
  endpoint. Use it as a best effort, inside try/catch:
  ```js
  const r = await fetch("/query_session_history", { method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ token: m.token, device_name: "tty0", HistoryType: "Serial" }) });
  // → { events: [{ name: "tty0", data: [byte, …] }, …], oldest_index, has_more }
  ```
  Those lines have no guest time, so don't put them on a chart.
- **Stamp console lines in the poll, not as they arrive.** Queue them from
  `subscribeBroker`, and in the poll check for a reset first, then process them
  at that poll's guest time. That way a line that arrives just after a reset
  isn't credited to the boot before it.
- **Plot guest seconds**, not ticks: `reading.seconds`, `m.guestSeconds`, or a
  `guest_seconds=` field an action prints. If time hasn't moved since the last
  poll (the session is paused), don't push a duplicate sample.
- **Clear your history on a reset.** You can tell one happened when guest time
  goes backwards between polls, or `tickCount` drops in `onState`.
- **Say when the data stops.** A frozen page that looks live misleads. Track
  failures inside the `every` task, and check a hung read from a separate
  `setInterval`, since `every` won't fire again while a read is stuck:
  - red banner, page dimmed, `⚠` in `document.title`: polls failing for over
    about 1.5 s, or one read stuck for over about 4 s. Tell the user to reopen
    the view from Franklin on the new session.
  - grey banner: `m.state && !m.isRunning()`, i.e. paused and nothing new
    until Go.
  - red banner: `attach()` rejected.
- **Buttons**: map `data-cmd` to closures over `m` and the devices, `await`
  them, and show only failures (`e.message` is Franklin's reason). The page's
  next poll shows what changed.

## Gotchas

- **bigint vs number.** Counters, addresses and registers are `bigint`: write
  `3n`, `>> 3n`, and `Number(x)` to plot or index. Signal values are plain
  numbers and can be `Infinity`.
- `readSignal` works only on a `source` signal. Check the name in
  `signalDescriptors()`; the error message lists the device's sources.
- The run state is a string from the `Session` getters, but the `state` field
  in `onState` may be an object. `state.name ?? state` covers both.
- Device paths come from the project's `peripherals.yaml`. A part named
  `charger` with `native: bq25890` lives at `/MEM/i2c0/bq25890`. Confirm
  paths with `emerson ctl peripherals` or `m.devicePaths()`.
- Franklin lists the view by its `<title>`. It ignores
  `<meta name="description">`. A new file shows up the next time the views
  list loads.
- `every()` stops while the tab is hidden. Don't treat the gap as a stall
  when the tab comes back.
- Keep the page usable at phone width and in both color schemes: put colors
  in `:root` variables and redefine them under `prefers-color-scheme: dark`.

## Testing a view

1. `emerson start`, then `emerson ctl go`. Put the model in an interesting
   state first (for example `emerson ctl action /world/battery set_soc 80.0`).
2. Make sure Franklin serves it:
   `curl -s -o /dev/null -w '%{http_code}' http://localhost:10314/views/{project}/{token}/workspace/{file}.html`.
3. Syntax-check the module: pull out the `<script type="module">` body and
   run `node --check view.mjs`.
4. Load it in a real browser. Headless Chrome's `--dump-dom` and
   `--screenshot` with `--virtual-time-budget` run for only one or two polls,
   which is enough for the tiles and banners but not for charts. For charts
   and buttons, start Chrome with `--headless=new --remote-debugging-port=9333`
   and drive it over CDP from a short Node script (Node 22 has a global
   `WebSocket`): `Page.navigate`, wait in real time, `Runtime.evaluate` to
   read text and `.click()` buttons, then `Page.captureScreenshot`. Look at
   the screenshot, because a chart with axes and no lines still renders
   without errors.
5. Check the failure path: open the view with a made-up token. It should show
   the "no session" banner rather than a blank page.
