# asus-user-status-led

[Русская версия](README.ru.md)

Control the ASUS User-status LED on supported ASUS laptops from userspace.
In `auto` mode the LED reflects an active microphone capture session;
`busy` and `off` force the LED on or off manually.

Current stage: design complete, implementation not started. The utility
described here does not exist yet.

## Verified on hardware

- ASUS ExpertBook B5402CBA (ASUS WMI device `0x00040019`).
- The LED is exposed by the kernel as `/sys/class/leds/:status` with binary
  brightness (`0`/`1`); physical ON/OFF works.
- Unprivileged writes to the LED work through a dedicated group and a narrow
  udev rule, and survive reboot.
- Fn+1 reaches Wayland/Niri as `XF86Display` with clean press/release events
  and no binding conflict.
- PipeWire semantics for detecting an active microphone session, including
  the mute-while-session-active case.

## Accepted design

- Python 3 standard library only: one unprivileged user daemon plus a small
  CLI in the same executable.
- A systemd user service, one narrow udev rule, and a Niri binding example.
- Modes: `busy` (LED forced on), `off` (LED forced off), and `auto`
  (default), where the LED follows microphone capture sessions.
- In `auto`, the LED is on while a real microphone (`Audio/Source`) feeds a
  running capture client (`Stream/Input/Audio`), observed through PipeWire.
  Muting the microphone inside an application does not turn the LED off
  while the capture session stays active.
- Fn+1 (`XF86Display`, currently unbound) cycles
  `auto -> busy -> off -> auto`.

## Not implemented yet

Everything: the daemon, the CLI, the systemd service, the udev rule, and
the Niri binding.

## Prerequisites

- A kernel exposing the ASUS User-status LED as `/sys/class/leds/:status`
  (upstream-style `asus-wmi` support; the kernel patch is developed
  separately and is not part of this repository).
- PipeWire, for `auto` mode.

## Documentation

- [ARCHITECTURE.md](ARCHITECTURE.md) — detailed design (Russian version:
  [ARCHITECTURE.ru.md](ARCHITECTURE.ru.md)).
- [CHECKPOINT.md](CHECKPOINT.md) — current verified state (English-only).
- [AGENTS.md](AGENTS.md) — project contract (English-only).
