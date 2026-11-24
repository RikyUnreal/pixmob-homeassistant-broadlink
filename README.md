# PixMob IR wristbands from Home Assistant, using a Broadlink IR blaster. Made with Claude.

Control PixMob infrared wristbands at home with a Broadlink IR remote and two Home Assistant scripts. No Arduino and no Flipper Zero needed.

Tested on one wristband model: **PixMob X2 Gen 3** (2 x CR2032). Other models may behave differently.

> **Made with Claude.** This project was created in collaboration with Claude (Anthropic's AI assistant). Claude decoded the IR protocol from the existing community documentation, wrote the scripts and the checksum fixes, and drafted this README. The human side was the idea, the hardware, and testing every code on a real wristband. It is not the work of a single person working alone.

## What you get

- **PixMob Presets** (`pixmob_presets.yaml`): pick a color and an effect from a menu. 12 colors x 7 effects.
- **PixMob Custom** (`pixmob_custom.yaml`): any RGB color from the color picker, plus fade in time, hold time, fade out time, chance to light up, and a Reset action. The IR code is computed by the script itself (Jinja template), so there are no stored codes.

## Requirements

- Home Assistant with the Broadlink integration and an IR remote entity (for example `remote.living_room`).
- A PixMob IR wristband with fresh batteries (old ones are often flat after a few days).

## Install

For each of the two files:

1. Settings, Automations & scenes, Scripts, Add Script, Create new script.
2. Three dots, Edit in YAML.
3. Paste the file content.
4. Replace `remote.YOUR_BROADLINK_REMOTE` with your own Broadlink remote entity.
5. Save.

Then run the script from the Scripts page and point the wristband at the Broadlink IR LED, a few centimeters away.

## Usage notes

- `Repeats: 1` is enough.
- In my tests, fade effects sent first after a long pause did nothing and worked when sent again a bit later. I did not investigate why.
- Presets effects (times decoded from the codes):

| Preset name | Fade in | Hold | Fade out |
| --- | --- | --- | --- |
| Flash | instant | not configurable | 32 ms |
| Fade in 2.4s | 2.4 s | 960 ms | 32 ms |
| In 480ms / out 960ms | 480 ms | 480 ms | 960 ms |
| In 2.4s / out 960ms | 2.4 s | 480 ms | 960 ms |
| Fade out 3.84s | 96 ms | 96 ms | 3.84 s |
| In 2.4s / out 480ms | 2.4 s | 32 ms | 480 ms |
| In 480ms / out 480ms | 480 ms | 32 ms | 480 ms |

- In the Custom script the hold time stops at 2.4 s and the fade out time starts at 32 ms, because the protocol notes say that hold = 3.84 s and fade out = 0 have special behaviours that are not fully understood.

## How it works

The protocol comes from the reverse engineering work of others (see Credits): a 38 kHz IR carrier, a 700 us bit time, a lookup table that encodes each 6-bit value, and a checksum.

To send it with a Broadlink:

- Each 700 us unit becomes 23 Broadlink ticks (1 tick = 8192/269 us).
- The code ends with a long gap (`00 0D 05`), the packet is padded to a multiple of 16 bytes and base64-encoded.
- It is sent with `remote.send_command` and the `b64:` prefix.

The Custom script builds the packet like this: 6-bit green, red, blue (value // 4), then attack/chance and release/sustain bytes. Each byte goes through the lookup table. The checksum is `table[(sum of the encoded bytes mod 256) >> 2]`. Bits are sent LSB first, runs of equal bits become pulses and gaps.

I checked the template against a separate Python generator on 600 random inputs (identical output), and on a real wristband.

## Findings

1. **The checksum covers the whole packet, fade tail included.** A fade code made by appending a tail to a flash code without recomputing the checksum may be ignored. This happened to me with `RED_4` from `PixMob_all_colors.ir`. The scripts here recompute it.
2. **Four codes in `PixMob_main.ir` (flipper-pixmob-ir-codes) have a checksum that does not match the documented formula:** `LT_GRN_FD_4`, `LT_GRN_FD_5`, `WHITISH_FD_4`, `WHITISH_FD_5`. I did not test the originals on a wristband. The scripts here use recomputed checksums.
3. **The `RED` code of `PixMob_main.ir` looks orange on my X2 Gen 3.** Decoded with the documented encoding, it has green = 32 and red = 132, while `RED_4` has green = 8 and red = 132 and looks red. A fade in starts red and turns orange because the green channel only shows at higher brightness (my interpretation, not verified).

## Limits

- One wristband model tested.
- The Reset action and the Custom template worked on my setup; other Home Assistant versions are untested.

## Credits

- Daniel Weidman: [pixmob-ir-reverse-engineering](https://github.com/danielweidman/pixmob-ir-reverse-engineering) and [flipper-pixmob-ir-codes](https://github.com/danielweidman/flipper-pixmob-ir-codes) (preset codes).
- jamesw343: [PixMob_IR](https://github.com/jamesw343/PixMob_IR) (protocol documentation: encoding table, checksum, fields).
- The upstream README credits @lasry1, @JulioC and Daniel Weidman for first controlling the wristbands with Broadlink devices, and @sean1983 for effect and color discovery.

## License

MIT. See the `LICENSE` file. Third party code and credits: see `THIRD_PARTY_NOTICES.md`.
