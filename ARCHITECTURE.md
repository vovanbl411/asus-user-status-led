# asus-user-status-led — architecture

[Русская версия](ARCHITECTURE.ru.md)

Detailed design of the accepted system. This document explains how the ASUS
User-status LED is controlled end-to-end, which parts are verified, and why
the alternatives were rejected. It is a design document, not a research
diary: only the evidence that shaped the architecture is kept.

Status: the design is accepted and validated on hardware; the userspace
implementation does not exist yet. See `CHECKPOINT.md` for current verified
state and `AGENTS.md` for the stable project contract.

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
(binding example, CLI, daemon, persisted state, LED write); above it,
everything is existing platform behaviour (firmware, kernel, XKB/Wayland).

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
  (section 5).

## 4. State persistence

The selected mode is persisted under XDG state storage:

```text
${XDG_STATE_HOME:-~/.local/state}/user-status-led/mode
```

Missing state means `auto`.

Accepted future CLI surface — this is accepted design, not implemented yet:

- `user-status-led status`
- `user-status-led set auto`
- `user-status-led set busy`
- `user-status-led set off`
- `user-status-led cycle`

## 5. AUTO semantics

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

## 6. Rejected AUTO predicates

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

## 7. PipeWire observation model

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

## 8. Privilege boundary

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

## 9. Process model

Accepted future process model:

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

## 10. Niri integration

Verified input path:

```text
Fn+1
→ firmware event 0x61
→ KEY_SWITCHVIDEOMODE
→ XF86Display
→ Niri
```

`XF86Display` is currently free in the verified active Niri configuration.

Accepted future binding:

```text
XF86Display -> user-status-led cycle
```

Exact Niri configuration syntax is intentionally kept out of this document
until it is verified.

## 11. Component boundaries

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

## 12. Non-goals

- Kernel patch development.
- Firmware changes.
- Kernel key remapping.
- Application-specific integrations.
- Generic conference detection unrelated to microphone topology.
- DBus or custom IPC.
- Root daemon.
- Custom PipeWire graph synchronizer.
- Unnecessary frameworks.

## 13. Current implementation status

Architecture and design are accepted and validated.

The userspace implementation does NOT exist yet.

The next project step is implementing minimal v0 according to `AGENTS.md`.
