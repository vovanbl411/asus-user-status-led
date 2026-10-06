# asus-user-status-led — architecture

[Русская версия](ARCHITECTURE.ru.md)

Detailed design of the accepted system. This document explains how the ASUS
User-status LED is controlled end-to-end, which parts are verified, and why
the alternatives were rejected. It is a design document, not a research
diary: only the evidence that shaped the architecture is kept.

Status: the design is accepted and v0 now implements it: CLI, daemon,
systemd user service, udev rule, and Niri binding example. End-to-end
live acceptance on hardware is still pending, and restart flicker has
not yet been evaluated. See `CHECKPOINT.md` for current verified state
and `AGENTS.md` for the stable project contract.

## 1. Purpose and scope

This repository owns only the userspace policy and integration for the ASUS
User-status LED:

- the mode model (`auto` / `busy` / `off`) and its persistence;
- the CLI that queries and changes the selected mode;
- the daemon that evaluates the mode and writes the LED state;
- the systemd user service, the udev permission rule, and the Niri binding
  example.

Kernel support is an external prerequisite; the kernel patch is developed
separately and does not belong to this repository.

Verified hardware baseline:

- ASUS ExpertBook B5402CBA;
- ASUS WMI DEVID `0x00040019`;
- LED ABI: `/sys/class/leds/:status`;
- binary brightness `0`/`1` (`max_brightness = 1`).

## 2. End-to-end architecture

The complete accepted chain:

```text
Physical Fn+1
    ↓
ASUS firmware event 0x61
    ↓
asus-wmi
    ↓
KEY_SWITCHVIDEOMODE
    ↓
XF86Display
    ↓
Niri binding
    ↓
user-status-led cycle
    ↓
persisted mode
   auto | busy | off
    ↓
systemctl --user restart user-status-led.service
    ↓
user-status-led daemon
    ↓
mode evaluation
    ↓
/sys/class/leds/:status/brightness
    ↓
asus-wmi
    ↓
ASUS WMI DEVS 0x00040019
    ↓
physical User-status LED
```

From the Niri binding down, each stage is a deliverable of this repository
(binding example, CLI, persisted state, service restart, daemon, LED
write); above it, everything is existing platform behaviour (firmware,
kernel, XKB/Wayland).

Kernel input remapping is not required. The verified input path already
delivers Fn+1 to Niri as `XF86Display` with clean press/release events, so
no kernel-side key changes are needed.

## 3. Mode model

- Modes: `auto`, `busy`, `off`.
- Default mode: `auto`.
- Cycle order: `auto -> busy -> off -> auto`.

Semantics:

- `busy` — LED forced ON;
- `off` — LED forced OFF;
- `auto` — LED follows the PipeWire-derived microphone-session state
  (section 6).

## 4. State persistence

The selected mode is persisted under XDG state storage:

```text
${XDG_STATE_HOME:-~/.local/state}/user-status-led/mode
```

Missing state means `auto`.

Accepted CLI surface (implemented in v0):

- `user-status-led status`
- `user-status-led set auto`
- `user-status-led set busy`
- `user-status-led set off`
- `user-status-led cycle`

## 5. CLI-to-daemon control path

The persisted mode file is the source of truth. Mode changes reach the
daemon through a systemd user service restart, not through a live control
channel:

```text
user-status-led set/cycle
    ↓
atomically persist selected mode
    ↓
systemctl --user restart user-status-led.service
    ↓
new daemon instance reads persisted mode
    ↓
evaluate/apply LED state immediately
    ↓
enter PipeWire monitor loop if needed
```

Accepted behaviour:

- `user-status-led set <mode>` and `user-status-led cycle`:
  1. determine the new mode;
  2. atomically persist it;
  3. run `systemctl --user restart user-status-led.service`;
  4. exit non-zero if the restart fails.
- The persisted mode is not rolled back when the restart fails: the
  requested state remains authoritative and is applied on the next
  successful service start.
- `user-status-led status` only reads and reports the persisted mode; it
  never restarts the service.
- On startup the daemon reads the persisted mode, evaluates and applies the
  required LED state immediately, and only then enters normal PipeWire
  event monitoring.

Restart is intentionally chosen over DBus, custom socket IPC, polling,
or a signal/reload mechanism: it is the simplest deterministic option
for v0 and reuses the existing systemd user service lifecycle.

A possible brief LED-off transition during restart is an acceptance item
to verify on hardware, not a problem assumed in advance. A signal/reload
mechanism is not introduced unless live acceptance later demonstrates a
concrete problem such as objectionable visible flicker.

## 6. AUTO semantics

The accepted conceptual predicate:

AUTO is ON when there is an active microphone consumer represented by an
active PipeWire connection from a real `Audio/Source` to a running
`Stream/Input/Audio` client. AUTO is OFF when no such microphone consumer
exists.

```text
Audio/Source
    ↓ active PipeWire link
Stream/Input/Audio (running)
    ↓
AUTO = ON
```

Application-level mute semantics:

- A real Vesktop voice call keeps the microphone graph present while the
  microphone is muted inside the application.
- Therefore `auto` stays ON while that microphone capture session remains
  active.
- Ending the call removes the microphone capture node and its links, making
  AUTO OFF.

The LED is a session-status indicator, not a hardware microphone-mute
indicator: muting the microphone inside an application does not extinguish
it while the capture session is still running.

## 7. Rejected AUTO predicates

The following predicates are explicitly rejected:

- Any running `Stream/Input/Audio` node. Rejected because a sink monitor can
  own a running capture stream, producing a false positive.
- Any `media.category=Capture` node. Rejected for the same reason.
- Application-name matching (Vesktop, Telegram, Firefox, Zoom, ...).
  Rejected: the AUTO predicate is defined by graph topology, not by
  application identity.
- Hardcoded microphone node names. Rejected: the predicate targets any real
  `Audio/Source`, not a specific node name.

Verified false-positive example — the Noctalia Spectrum capture path:

```text
Audio/Sink monitor
    ↓
Noctalia Stream/Input/Audio
```

This is a sink-monitor capture path and must NOT activate AUTO, which is
why the predicate requires a real `Audio/Source` as the origin of the link.

`Audio/Source/Virtual` is currently outside the accepted AUTO predicate
because no verified requirement exists for it.

## 8. PipeWire observation model

```text
pw-dump --monitor
    ↓
event / wake-up notification
    ↓
plain pw-dump
    ↓
authoritative complete graph snapshot
    ↓
evaluate AUTO
```

Verified behaviour of `pw-dump --monitor`:

- it emits the initial full state immediately as a complete JSON array;
- subsequent graph changes arrive as separate JSON arrays;
- removals appear as objects such as:

```json
{
  "id": 144,
  "info": null
}
```

Accepted design:

- do NOT maintain a custom incremental PipeWire graph cache;
- do NOT use periodic polling;
- do NOT use direct Python PipeWire bindings in v0;
- use the monitor only to wake up and re-evaluate;
- use a fresh plain `pw-dump` snapshot as the source of truth.

This intentionally trades a small amount of extra process work on
relatively rare graph changes for much simpler correctness: a full re-dump
after every wakeup cannot drift from PipeWire's actual state, while an
incremental cache can.

## 9. Privilege boundary

LED attribute:

```text
/sys/class/leds/:status/brightness
```

Accepted permission model:

```text
root:user-status-led
0660
```

Dedicated group:

```text
user-status-led
```

Verified udev topology.

Device:

```text
SUBSYSTEM=="leds"
KERNEL==":status"
```

Parent:

```text
KERNELS=="asus-nb-wmi"
SUBSYSTEMS=="platform"
DRIVERS=="asus-nb-wmi"
```

Why this model:

- the daemon remains unprivileged;
- no root daemon;
- no sudo/doas on each LED write;
- no polkit helper;
- no broader access to unrelated LEDs.

Unprivileged LED ON/OFF and reboot persistence of the permission setup have
already been verified on hardware.

## 10. Process model

Process model (implemented in v0):

- one Python 3 stdlib-only executable;
- the same executable exposes CLI and daemon behaviour;
- one systemd user service;
- no custom socket API;
- no DBus API;
- no root service;
- no internal supervisor — systemd owns the service lifecycle.

The daemon avoids redundant brightness writes: it writes only when the
desired physical LED state differs from the last applied state.

Clean shutdown: the daemon best-effort sets the LED OFF during clean
shutdown so stale state is not intentionally left behind.

## 11. Niri integration

Verified input path:

```text
Fn+1
→ firmware event 0x61
→ KEY_SWITCHVIDEOMODE
→ XF86Display
→ Niri
```

`XF86Display` is currently free in the verified active Niri configuration.

Accepted binding:

```text
XF86Display -> user-status-led cycle
```

An example snippet lives in `niri/user-status-led.kdl` for insertion
into an existing `binds { }` section. The binding itself is part of the
user's Niri configuration and is pending live verification.

## 12. Component boundaries

Kernel / firmware:

- exposes the hardware primitive (LED class device, key event);
- external to this repository.

udev:

- narrow permission setup for the single LED attribute.

Niri:

- converts the existing `XF86Display` user input into a CLI invocation.

CLI:

- updates and queries the selected mode.

Daemon:

- evaluates the mode and the PipeWire state;
- writes the desired LED state.

PipeWire:

- supplies the topology that drives `auto`.

## 13. Non-goals

- Kernel patch development.
- Firmware changes.
- Kernel key remapping.
- Application-specific integrations.
- Generic conference detection unrelated to microphone topology.
- DBus or custom IPC.
- Root daemon.
- Custom PipeWire graph synchronizer.
- Unnecessary frameworks.

## 14. Current implementation status

Architecture and design are accepted. The v0 userspace implementation
exists: the `user-status-led` executable (CLI and daemon, Python 3
standard library only), `systemd/user-status-led.service`,
`udev/70-asus-user-status-led.rules`, and `niri/user-status-led.kdl`.

The implementation has passed static validation only (compilation, CLI
surface, missing-state behaviour, offline checks of the monitor parser
and the AUTO predicate). It has NOT undergone end-to-end live acceptance
on hardware; restart flicker has not been evaluated.

The next project step is live acceptance on the ASUS ExpertBook B5402CBA:
AUTO behaviour with real audio sessions, Fn+1 cycling through the
utility, restart flicker, and the reboot/login lifecycle.
