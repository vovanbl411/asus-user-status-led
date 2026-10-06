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

Design is accepted. Public documentation is bilingual
(`README.md`/`README.ru.md`, `ARCHITECTURE.md`/`ARCHITECTURE.ru.md`);
`AGENTS.md` and this file remain English-only. No implementation exists
yet.

Next step: implement the minimal v0 userspace component according to
`AGENTS.md`.
