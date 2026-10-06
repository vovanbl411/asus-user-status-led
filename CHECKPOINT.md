# asus-user-status-led — checkpoint

Current verified state and the next step. Accepted decisions live in
`AGENTS.md`.

## Kernel / hardware

- ASUS User-status LED support is working locally through `asus-wmi`.
- LED ABI: `/sys/class/leds/:status`.
- `max_brightness = 1`.
- Physical ON/OFF through brightness `1`/`0`: PASS.
- The upstream-style kernel patch has already been developed separately;
  kernel work is outside this repository.

## Permissions — CLOSED / PASS

- Dedicated group `user-status-led` exists.
- User `vladimir` is a member.
- The udev permission model was tested.
- Unprivileged writes to `brightness` work.
- Physical ON/OFF works without privilege escalation.
- Reboot persistence was verified.

## Fn+1 — CLOSED / PASS

Verified live path:

```text
Fn+1 -> firmware event 0x61 -> KEY_SWITCHVIDEOMODE -> XF86Display -> Wayland/Niri
```

- Event delivery: PASS.
- Press/release behaviour: PASS.
- No existing active Niri conflict for `XF86Display`.
- No kernel change is required.

## PipeWire AUTO semantics — CLOSED / PASS

Rejected predicates:

- any running `Stream/Input/Audio`;
- any `media.category=Capture`.

Reason: Noctalia Spectrum creates a running capture stream from an audio
sink monitor and would create a false positive.

Verified examples:

- Noctalia Spectrum:
  `Audio/Sink monitor -> Noctalia Stream/Input/Audio` — must NOT activate
  AUTO.
- Controlled `pw-record`:
  `Digital Microphone Audio/Source -> pw-record Stream/Input/Audio` —
  represents microphone use.
- Real Vesktop voice call:
  `Digital Microphone Audio/Source -> vesktop Stream/Input/Audio`
  - active call: microphone graph present;
  - application-level mute: graph remains present;
  - call ended: microphone capture node and links disappear.

Accepted AUTO semantics:

- active microphone session -> ON;
- no microphone consumer -> OFF;
- application-level mute while the capture session remains active -> still
  ON.

## PipeWire monitor format

Verified for `pw-dump --monitor`:

- immediately emits the initial full state;
- then emits separate JSON arrays for graph changes;
- removals appear as objects such as `{ "id": N, "info": null }`.

Accepted implementation model:

- monitor = wakeup source;
- plain `pw-dump` = source of truth;
- no local incremental graph cache.

## Current status

Design is accepted, including the CLI-to-daemon v0 control path: atomic
mode persistence followed by a systemd user service restart. The
documentation architecture gate is closed.

Public documentation is bilingual (`README.md`/`README.ru.md`,
`ARCHITECTURE.md`/`ARCHITECTURE.ru.md`); `AGENTS.md` and this file remain
English-only.

The v0 userspace implementation exists:

- `user-status-led` — CLI and daemon, Python 3 standard library only;
- `systemd/user-status-led.service` — systemd user unit;
- `udev/70-asus-user-status-led.rules` — narrow LED permission rule;
- `niri/user-status-led.kdl` — Niri binding example.

Static validation passed: `py_compile`, `--help`, missing-state behaviour
(`status` prints `auto` with a clean `XDG_STATE_HOME`), invalid-state
rejection, atomic persistence with no rollback on restart failure, and
offline synthetic checks of the monitor JSON parser and the AUTO
predicate. The parser and predicate were also checked against a live
read-only `pw-dump`/`pw-dump --monitor` session (real microphone link
matched; sink-monitor path rejected). The Niri snippet syntax passed
`niri validate` (niri 26.04). The daemon's LED write-permission startup
probe fails clearly on unwritable/missing paths (offline) and its
write-only open succeeds against the real LED (no write performed;
`root:user-status-led` `0660`).

## Live acceptance — first run

Manual modes PASS on the ASUS ExpertBook B5402CBA:

```text
permissions ........ PASS
service ............ PASS
busy -> LED ON ..... PASS
off -> LED OFF ..... PASS
mode persistence ... PASS
```

AUTO baseline FAIL — real BLOCKER:

- `mode=auto`: daemon CPU ≈ 67%, 32 CPU seconds after ~23 seconds
  runtime; `pw-dump --monitor` stays running while plain `pw-dump`
  instances are spawned continuously;
- `mode=off`: daemon CPU ≈ 0%, no `pw-dump` processes;
- root cause: every complete `pw-dump --monitor` JSON value triggered a
  fresh plain `pw-dump`; the snapshot helper is itself a PipeWire client,
  so its Client add/remove events re-triggered snapshots in a
  self-sustaining feedback loop.

The machine is intentionally left in `off`.

## AUTO feedback-loop fix

Implemented: monitor events are filtered before any snapshot. Only a
typed Node/Link event, or a type-less removal (`{ "id": N, "info": null }`)
whose `N` is in the Node/Link ID set from the last snapshot, triggers a
fresh authoritative `pw-dump`. That ID set exists solely to classify
type-less removals, is replaced wholesale after every snapshot, and is
not a topology cache. No polling, debounce, or rate limiting was added.

Static validation of the fix passed: `git diff --check`, `py_compile`,
`--help`, missing-state check, and synthetic filtering cases (Client-only
event ignored; typed Node/Link events relevant; known-ID removal
relevant; unknown-ID removal ignored; snapshot ID collection includes
Nodes/Links and excludes Clients; AUTO predicate unchanged).

AUTO live acceptance remains OPEN — the fix is implemented and
static-validated only; it must be installed and tested on hardware
before AUTO may be marked PASS. Also still open: Fn+1 cycling through
the utility, restart flicker, reboot/login lifecycle.
