# asus-user-status-led

Project contract for agents working in this repository: stable rules and
accepted decisions only. Current verified state and the next step live in
`CHECKPOINT.md`.

## Scope

This repository owns only the userspace policy and integration for the ASUS
User-status LED exposed by the upstream-style `asus-wmi` LED class interface.

Verified hardware context:

- ASUS ExpertBook B5402CBA;
- ASUS WMI device ID `0x00040019`;
- Linux LED ABI: `/sys/class/leds/:status`;
- binary brightness: `0` / `1`.

The kernel implementation is not part of this repository; kernel support is
an external prerequisite.

This repository does not own:

- kernel support;
- ASUS WMI firmware implementation;
- remapping the kernel input event;
- generic conferencing software integration unrelated to LED policy.

## Engineering principles

Prefer:

- minimal implementation;
- reproducible behaviour;
- evidence-based changes;
- standard Linux interfaces;
- small validated stages.

Do not add additional hardening, abstractions, frameworks, IPC layers,
refactoring, automation, CI, tests, packaging infrastructure, or
architectural layers unless a concrete problem requires them.

Do not reopen already validated design decisions without new material
evidence.

## Accepted architecture

The initial implementation uses:

- Python 3 standard library only;
- one unprivileged user daemon;
- a small CLI exposed by the same executable;
- a systemd user service;
- one narrow udev rule for LED write permissions;
- a Niri binding example for `XF86Display`.

Do not introduce:

- a DBus API;
- a custom Unix socket API;
- a root daemon;
- periodic polling;
- direct Python PipeWire bindings;
- application-specific logic.

## Modes

- Supported modes: `auto`, `busy`, `off`.
- Default mode: `auto`.
- Cycle order: `auto -> busy -> off -> auto`.
- `busy`: LED forced ON.
- `off`: LED forced OFF.
- `auto`: LED follows microphone-session activity detected from PipeWire
  topology.
- Application-level microphone mute does NOT disable `auto` busy state if
  the microphone capture session remains active.

## AUTO predicate

AUTO is ON when there is an active microphone consumer represented by an
active PipeWire connection from a real `Audio/Source` to a running
`Stream/Input/Audio` client. AUTO is OFF when no such microphone consumer
exists.

Do not:

- identify applications by name;
- hardcode Vesktop, Telegram, Firefox, Zoom, etc.;
- hardcode the current microphone node name;
- treat every `Stream/Input/Audio` or every `media.category=Capture` node as
  microphone activity.

A sink monitor such as the Noctalia Spectrum capture path must not activate
the LED. `Audio/Source/Virtual` is not included in the accepted predicate
because no verified requirement exists for it.

## PipeWire observation model

- `pw-dump --monitor` is used only as an event/wakeup source.
- Regular `pw-dump` is the authoritative full snapshot used to evaluate the
  current AUTO state.

Reason: `pw-dump --monitor` outputs an initial full JSON array and then
additional JSON arrays describing graph changes and removals. Maintaining a
custom incremental PipeWire object cache is unnecessary complexity for this
project.

Do not implement a custom graph synchronizer unless future evidence requires
it.

## State

Persist the selected mode under XDG state storage:

```text
${XDG_STATE_HOME:-~/.local/state}/user-status-led/mode
```

If no state exists, use `auto`.

Accepted CLI surface (documented only; not implemented yet):

- `user-status-led status`
- `user-status-led set auto`
- `user-status-led set busy`
- `user-status-led set off`
- `user-status-led cycle`

Do not implement these commands in the bootstrap commit; the interface is
documented for the future implementation.

## Fn+1 integration

Verified input path:

```text
Fn+1 -> firmware event 0x61 -> KEY_SWITCHVIDEOMODE -> XF86Display -> Niri
```

- Niri sees clean press/release events.
- `XF86Display` is currently free in the active Niri configuration.
- No kernel input remapping is required.

Future Niri integration binds `XF86Display` to `user-status-led cycle`.

## Privilege boundary

LED sysfs attribute: `/sys/class/leds/:status/brightness`.

Accepted privilege model:

- dedicated group: `user-status-led`;
- `brightness` becomes `root:user-status-led`, mode `0660`;
- permissions are applied through a narrow udev rule matching the ASUS LED;
- the daemon remains unprivileged.

Verified udev topology:

- device: `SUBSYSTEM=="leds"`, `KERNEL==":status"`;
- parent: `KERNELS=="asus-nb-wmi"`, `SUBSYSTEMS=="platform"`,
  `DRIVERS=="asus-nb-wmi"`.

Do not replace this with:

- a root daemon;
- doas/sudo on every write;
- a polkit helper;
- broader permissions on unrelated LEDs.

## LED behaviour

The daemon writes only `0` and `1` to
`/sys/class/leds/:status/brightness`.

Avoid redundant writes when the desired physical LED state has not changed.

Turn the LED OFF (best effort) during clean shutdown so stale state is not
intentionally left behind.

## Documentation and languages

- `AGENTS.md` and `CHECKPOINT.md` are English-only canonical internal
  project files; do not create Russian mirrors for them.
- Public-facing documentation is bilingual: English documents use the
  normal `.md` name, Russian mirrors use `.ru.md`.
- English is authoritative if an accidental semantic mismatch appears.
- Mirrored EN/RU documents must be updated together whenever their shared
  content changes.
- Translation must preserve meaning, scope, status, caveats, and
  architecture; it must not introduce independent facts or decisions.
- Mirrored public documents carry reciprocal language links near the top:
  `README.md` ↔ `README.ru.md`, `ARCHITECTURE.md` ↔ `ARCHITECTURE.ru.md`.
- Detailed architecture lives in `ARCHITECTURE.md` (+ mirror); current
  verified state lives in `CHECKPOINT.md`.
