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

## v0 implementation

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
- `niri/user-status-led.kdl` — Niri binding, installed on the target
  system.

Static validation passed before live acceptance: `py_compile`, `--help`,
missing-state and invalid-state behaviour, atomic persistence with no
rollback on restart failure, offline synthetic checks of the monitor
JSON parser, the AUTO predicate, and the relevant-event filtering, plus
`niri validate` (niri 26.04).

## Live acceptance — ASUS ExpertBook B5402CBA

```text
Manual modes ................ PASS
AUTO idle ................... PASS
pw-record capture ........... PASS
Noctalia negative-control ... PASS
Vesktop call ................ PASS
Vesktop app mute ............ PASS
Fn+1 mode cycle ............. PASS
CPU-loop fix ................ PASS

Restart flicker ............. OPEN
Reboot/login lifecycle ...... OPEN
```

- Manual modes: permissions, service, `busy` -> LED ON, `off` -> LED OFF,
  mode persistence.
- AUTO idle: brightness `0`, service active, one long-lived
  `pw-dump --monitor --no-colors`, no plain `pw-dump` loop, daemon CPU
  effectively idle (example: 393 ms CPU over 35 s, 16.7 MiB RSS).
- Controlled capture: `pw-record` active -> LED ON; stopped -> LED OFF;
  no CPU-loop regression.
- Noctalia Spectrum active: LED stays OFF (sink-monitor path rejected),
  CPU low.
- Real Vesktop call: LED OFF before, ON during the call, ON through
  application-level mute and unmute, OFF after leaving. The short PipeWire
  teardown before the LED returns to `0` is normal observed behaviour.
  Vesktop itself kept running, so AUTO follows the capture session, not
  the process lifetime.
- Fn+1 cycle: one press produces one mode transition
  (`auto -> busy -> off -> auto`); `repeat=false` accepted. Returning to
  `auto` during an active call immediately re-evaluates PipeWire and sets
  brightness `1`. The service stayed active throughout.
- Runtime health after cycling and audio activity: service active, one
  `pw-dump --monitor`, no plain `pw-dump` loop, low daemon CPU (example:
  1.979 s CPU over ~2 min 26 s wall time).

Historical note: the first AUTO live test exposed a self-induced
`pw-dump` monitor/snapshot feedback loop (roughly 67% daemon CPU).
Relevant-event filtering fixed it; hardware re-acceptance passed.

## Open items — next step

- Restart flicker: `set`/`cycle` restarts work functionally, but no
  explicit visual judgment has been made yet.
- Reboot/login lifecycle: persisted mode after reboot, user service start
  after login, udev permissions, AUTO startup, no CPU-loop regression,
  and Fn+1 cycling after reboot.
