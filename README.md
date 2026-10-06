# asus-user-status-led

[Русская версия](README.ru.md)

Control the ASUS User-status LED on supported ASUS laptops from userspace.
In `auto` mode the LED reflects an active microphone capture session;
`busy` and `off` force the LED on or off manually.

Current stage: v0 is implemented and has passed live functional acceptance
for manual modes, AUTO microphone detection, false-positive rejection, and
Fn+1 integration on the ASUS ExpertBook B5402CBA. Reboot/login lifecycle
and restart-flicker acceptance remain open, so it is not yet considered
production-ready.

## Verified on hardware

- ASUS ExpertBook B5402CBA (ASUS WMI device `0x00040019`).
- The LED is exposed by the kernel as `/sys/class/leds/:status` with binary
  brightness (`0`/`1`); physical ON/OFF works.
- Manual modes: `busy` forces the LED on, `off` forces it off, and the
  selected mode persists across `set`/`cycle` restarts. Unprivileged writes
  work through a dedicated group and a narrow udev rule, and survive
  reboot.
- `auto` operation: a controlled `pw-record` capture turns the LED on and
  off with the session; the Noctalia Spectrum sink-monitor capture path is
  rejected; a real Vesktop call keeps the LED on through application-level
  muting and turns it off when the call ends.
- Fn+1 reaches Niri as `XF86Display`; the repository binding is installed
  and cycles `auto -> busy -> off -> auto`, one transition per press.

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
- Fn+1 (`XF86Display`) cycles `auto -> busy -> off -> auto`.

## Installation

The implementation targets a single user on a Linux system with systemd.

1. Group and permissions (once, as administrator). The daemon runs
   unprivileged and relies on a dedicated group; nothing in this
   repository creates the group or edits group membership:

    ```bash
    groupadd --system user-status-led    # skip if the group already exists
    usermod -aG user-status-led <user>
    ```

2. Install the udev rule and apply it to the LED:

    ```bash
    install -m 644 udev/70-asus-user-status-led.rules /etc/udev/rules.d/
    udevadm control --reload
    udevadm trigger --action=add /sys/class/leds/:status
    ```

   The rule grants the group write access to
   `/sys/class/leds/:status/brightness` only (`root:user-status-led`,
   `0660`); no other LED is affected.

3. Install the executable and the user service:

    ```bash
    install -Dm755 user-status-led ~/.local/bin/user-status-led
    install -Dm644 systemd/user-status-led.service ~/.config/systemd/user/user-status-led.service
    systemctl --user daemon-reload
    systemctl --user enable --now user-status-led.service
    ```

4. Optional: add the Fn+1 binding from `niri/user-status-led.kdl` to the
   `binds { }` section of your Niri configuration.

After a group-membership change, log out and back in (or start the
service from a session where the new group applies) before step 3.

## Usage

```text
user-status-led status                print the selected mode
user-status-led set auto|busy|off     select a mode and apply it
user-status-led cycle                 advance auto -> busy -> off -> auto
```

`set` and `cycle` persist the mode atomically under
`${XDG_STATE_HOME:-~/.local/state}/user-status-led/mode` and restart the
user service, which applies the new mode immediately. `status` only reads
the persisted mode and does not touch the LED or the service.

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
