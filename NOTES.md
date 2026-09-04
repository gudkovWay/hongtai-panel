# Jungle Leopard / HONGTAI panels on this machine — findings

Working notes from the Linux port effort. Two HONGTAI panels are attached; they
share a USB serial number, so they are told apart by `bcdDevice`.

| symlink | USB path | bcdDevice | what it is | status |
| --- | --- | --- | --- | --- |
| `/dev/hongtai-0200` | `1-8` | `0200` | case display, `TXW813-NV3007-2.99inch-hor`, 428x142, fw 3.4, SPI/RGB565 | **works** — accepts frames, confirmed visually |
| `/dev/hongtai-0100` | `1-10` | `0100` | unidentified; claims `HONGTAI MONITOR` | does not display anything we send |

## Local changes to the upstream driver (GOG1071/hongtai-panel)

1. `hongtai_panel/protocol.py` — added PID `0x7792` (`USB_PIDS`). Upstream only
   knew `0x7791`. Vendor's own table: `7791` = LovingCool, `7792` = Jungle
   Leopard, `7793` = Jungle Leopard SPI. Same OEM, same protocol.
2. udev rule renumbered `99-` -> `70-`. **This is an upstream bug**: systemd's
   `73-seat-late.rules` runs the `uaccess` builtin, so a rule numbered above 73
   tags the device too late and the ACL is never granted. With `99-` the rule
   silently does nothing for everyone, not just here.
3. udev rule adds stable symlinks by `bcdDevice`. Needed because both panels
   report serial `A247503xX000000`, so `/dev/serial/by-id/` can only name one,
   and `ttyACM*` numbering is not stable across reboots.

Installed at `/home/q/hongtai-panel` with its own venv: `./.venv/bin/hongtai-panel`.

## Protocol facts established by probing

- Control frame: `55 AA | len_lo len_hi | key | payload | ck_lo ck_hi`,
  `len = len(payload) + 7`, `ck = sum(preceding bytes) & 0xFFFF`. Confirmed
  against the vendor's `handleData` in the decompiled Electron app.
- The `0100` panel replies to `0x03` (set brightness) with `55aa 0800 03 00 0a01`
  — key echoed, payload `00` = success. So it parses and acks commands.
- It **ignores `0x06` (get device info) entirely** — no reply at all, where the
  case panel answers with JSON. This alone hangs `hongtai-panel`, which requires
  the `0x06` reply before it will send anything.
- It ignores image frames: 7 candidate geometries (2400x1080, 1080x2400,
  480x480, 466x466, 320x240, 240x320, 128x128) x both envelopes (len-prefixed
  and bare JPEG) produced no reply and no visible change.
- Brightness acks `OK` but has no visible effect.
- Baud rate is irrelevant (17 rates tried, identical behaviour) — expected for
  CDC-ACM.
- **Do not append a padding byte to a control frame.** An extra byte makes the
  frame invalid and the panel answers `55aa 0800 ff ff 0503` (`key=0xff` =
  unknown). This looked like a handshake requirement for a while; it was not.

## Vendor software

`Jungle Leopard Display Setup 1.0.87` unpacked to the scratchpad (Electron,
17070 files). Its `connect()` is simply: open port -> `deviceClear`
(`ff d9 ff d9`, then `00 00 00 00`, raw) -> wait 200 ms -> `0x06`. No auth, no
handshake, no RSA. Download:
https://file.globalso.com/file_manage/1211/20260819/jungle-leopard-display-setup-1-0-87.zip

## Next step (pending hardware access)

While replacing the GPU holder, unplug the USB cable running from the pump head
and re-check which device disappears:

    for d in /sys/bus/usb/devices/*/; do [ "$(cat $d/idVendor 2>/dev/null)" = 33c3 ] \
      && echo "$(basename $d) $(cat $d/bcdDevice)"; done

- `1-10` disappears -> the pump panel *is* `0100`; the protocol needs more work.
- nothing disappears -> the pump panel is not wired to USB at all; it runs its
  startup animation from internal storage. That matches the observed behaviour:
  black after a day or two powered off, animation returns after power-cycling
  the PSU. Fix is the cable, not the code.

## Also unresolved

The case panel accepts frames but the pixel layout is wrong — it reports
`angle=90` with native `428x142`, the driver swaps to `142x428` and does not
rotate. Solid fills show up; text and large digits do not. Byte order / rotation
still needs to be worked out.
