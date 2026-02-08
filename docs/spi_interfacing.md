# trabular SPI Interface Documentation

## Overview

trabular implements an Apple Desktop Bus (ADB) transceiver on AVR MCUs. In the standard configuration, a host microcontroller talks to trabular over a simple byte-oriented serial link, either SPI (USI-based) or UART. The SPI/serial interface is responsible for pushing device events (keyboard/mouse) and arbitrary data into trabular so it can satisfy ADB bus requests without the host having to meet ADB timing requirements itself. The SPI/serial interface is serviced in a tight polling loop and is expected to be fast and simple. The firmware supports keyboard, mouse, and an “arbitrary” device simultaneously. See `README.md`, `main.c`, and the serial/data headers for the high-level architecture and timing constraints. 【F:README.md†L1-L21】【F:main.c†L33-L58】【F:data.h†L24-L54】【F:serial.h†L21-L39】

## Transport options: SPI (USI) vs UART (USART)

The firmware chooses between SPI and UART at build time:

- **UART path (`USE_USART`)**: Initializes USART at 38,400 baud (with `util/setbaud.h`), and reads bytes from `UDR0`. If the command returns a response byte (>0), it immediately writes it back to `UDR0`. In debug mode it does not echo responses. 【F:serial.c†L21-L84】
- **SPI path (default)**: Enables the AVR USI in 3-wire mode (`USIWM0`) with external clock (`USICS1`), and then polls the USI overflow flag (`USIOIF`) to detect a full received byte. When a byte is received, it processes it and writes the response into `USIDR` to be returned on the next SPI transfer. 【F:serial.c†L54-L74】

## Polling and timing requirements

The serial/SPI interface is **poll-driven**:

- `handle_data()` must be called approximately every ~50 µs to avoid losing incoming serial data. The maximum observed `handle_serial_data()` execution time is ~10 µs in the worst case (as of 2016), leaving enough room for ADB timing. 【F:serial.h†L21-L39】【F:data.h†L33-L54】
- Certain ADB timing loops explicitly warn that SPI updates can add ~20 µs of jitter. For example, `adb_wait_for_assertion()` can be late by 5–6 timer counts due to SPI processing. 【F:adb.c†L704-L733】
- `adb_resync()` only checks the serial interface **once** at the start, and warns that long timeouts can cause SPI data loss. 【F:adb.c†L735-L755】

## SPI byte framing and command model

The SPI interface is byte-oriented. Each byte is split into:

- **Upper nibble**: command (`cmd = byte >> 4`)
- **Lower nibble**: payload (`payload = byte & 0x0F`)

This is handled by `handle_serial_data()`. 【F:serial.c†L87-L110】

### Command ranges

The command nibble determines the meaning of the payload and which device/registers are updated. The key ranges are:

- **`cmd == 0`**: special commands (status, clears, etc.). 【F:serial.c†L111-L189】
- **`cmd == 2` or `cmd == 3`** (arbitrary device register 0): lower/upper nibble writes into a small FIFO. 【F:serial.c†L190-L219】
- **`cmd == 4` and `cmd == 5`** (keyboard): lower nibble then upper nibble for an 8-bit keycode. 【F:serial.c†L130-L140】【F:serial.c†L220-L239】
- **`cmd == 6` and `cmd == 7`** (mouse buttons): lower/upper nibble writes to `mse_btn_data`. 【F:serial.c†L241-L266】
- **`cmd == 8..11`** (mouse X motion) and **`cmd == 12..15`** (mouse Y motion): signed deltas, using upper/lower nibble and sign encoded in command bits. 【F:serial.c†L268-L331】

## Special commands (`cmd == 0`)

All `cmd == 0` operations use `payload` values 0x0–0xF. Key commands include:

- **0x01: TALK STATUS** — returns a status byte starting at `0x80`, with bits set if the arbitrary device has pending data or the keyboard buffer is above half full. 【F:serial.c†L118-L146】
- **0x02: ARBITRARY REGISTER 0 READY** — sets `arb_buf0_set` if at least two bytes are in the buffer. 【F:serial.c†L147-L156】
- **0x03: ARBITRARY CLEAR REGISTER 0** — clears the register 0 buffer. 【F:serial.c†L156-L161】
- **0x04: ARBITRARY REGISTER 2 CLEAR** — clears the 2-byte register 2 values and “set” flag. 【F:serial.c†L161-L168】
- **0x05: KEYBOARD CLEAR REGISTER 0** — clears the keyboard ring buffer. 【F:serial.c†L169-L175】
- **0x06/0x07/0x08: MOUSE CLEAR BUTTONS/X/Y** — clears mouse button state or motion deltas. 【F:serial.c†L176-L188】
- **0x0C–0x0F: TALK ARBITRARY REGISTER 2** — returns each nibble of the two-byte register 2, encoded as `0x40`–`0x70` values. 【F:serial.c†L189-L216】

## Keyboard data path

Keyboard keycodes are transferred in two steps:

1. **Lower nibble**: send `cmd == 4` with lower 4 bits of the keycode. This updates a temporary nibble.
2. **Upper nibble**: send `cmd == 5` with upper 4 bits, which combines the two and enqueues the full 8-bit keycode.

This mechanism is optimized to avoid timing issues, and the `cmd == 5` path is handled earlier for performance. Keycodes are interpreted such that the top bit indicates key up/down, and certain keys (e.g., power key `0x7F`) have special handling for register bits. 【F:serial.c†L110-L142】【F:serial.c†L333-L427】

## Mouse data path

Mouse data is split across multiple commands:

- **Buttons**: `cmd == 6` sets the lower nibble; `cmd == 7` sets the upper nibble in `mse_btn_data`. 【F:serial.c†L241-L266】
- **Motion**: `cmd` in **8–11** encodes X, **12–15** encodes Y. The `cmd`’s LSB indicates sign (positive if 0), and bit 1 indicates whether the payload is the upper or lower nibble. This allows assembling signed deltas across two commands for each axis. 【F:serial.c†L268-L331】

## Arbitrary device data path

The arbitrary device exposes a small FIFO (register 0) and a 2-byte register (register 2):

- **Register 0** is written as two nibbles: `cmd == 2` sets the lower nibble temp, `cmd == 3` sets the upper nibble and commits the byte to the buffer (if space allows). The buffer is bounded by `ARB_BUF0_SIZE`. 【F:serial.c†L190-L219】【F:registers.h†L98-L115】
- **Register 2** can be cleared and read via special commands (0x04 and 0x0C–0x0F). 【F:serial.c†L161-L216】

## Response behavior

Responses are returned as a single byte from `handle_serial_data()`:

- **SPI**: the return byte is placed in `USIDR` and will be shifted out on the **next** SPI transfer (standard SPI behavior for an SPI slave). 【F:serial.c†L60-L74】
- **UART**: a non-zero return value is transmitted immediately; zero means “no response.” 【F:serial.c†L75-L84】

### SPI response timing and “dummy” bytes

Because the SPI response is loaded into `USIDR` after the current byte is received, the host must perform an **additional SPI transfer** to read the response. In practice, this means sending a “dummy” byte (often `0x00` or `0xFF`) after the command byte; the response will be clocked out during that dummy transfer. The firmware does not explicitly require `0x00`—any value will work so long as the host is prepared for the response on the subsequent transfer. 【F:serial.c†L60-L74】

There is no explicit firmware-defined delay between the command byte and the dummy transfer; the response is written to `USIDR` immediately after the received byte is processed. In other words, the host can clock the next byte as soon as the SPI hardware allows, provided the firmware has had time to poll and handle the received byte (the polling path is expected to run roughly every 50 µs). 【F:serial.c†L60-L74】【F:serial.h†L21-L39】

## Integration notes

- Ensure the host sends SPI bytes at a rate the AVR can service: missing the ~50 µs polling window can lead to lost bytes. 【F:serial.h†L21-L39】
- If you are using SPI, align the host’s command sequencing to the expected nibble order (e.g., keyboard lower nibble first, upper nibble second). Out-of-order bytes are not resynchronized by the firmware and will corrupt the current command state. 【F:serial.c†L110-L142】【F:serial.c†L220-L239】
- If you rely on timing-sensitive ADB behavior, keep in mind SPI handling can add up to ~20 µs of jitter in certain ADB wait loops. 【F:adb.c†L704-L733】

## Additional gotchas, tricks, and host-side guidance

### ADB bus behavior to expect

- **Spurious pulses vs. reset:** Short line assertions (shorter than `ADB_SIGDEL_ATTN_MIN`) are ignored; long assertions (greater than or equal to `ADB_SIGDEL_ATTN_MAX`) are treated as a bus reset and call `adb_reset()` which clears addresses/handlers and registers. This is the primary “noise” handling behavior. 【F:adb.c†L124-L142】【F:adb.c†L531-L570】
- **No explicit host-off state:** The firmware doesn’t track “host powered off” explicitly; it simply reacts to ADB line timing. Ensure your hardware pull-ups and line conditioning provide a stable idle line. 【F:adb.c†L111-L142】

### SPI timing and pacing

- **No fixed delay between command and response:** You can clock the dummy byte as soon as the SPI hardware allows; the response is loaded into `USIDR` immediately after processing the command byte. The only real constraint is that the firmware must have polled and handled the byte (the polling loop is intended to run about every ~50 µs). 【F:serial.c†L60-L74】【F:serial.h†L21-L39】
- **Plan for polling cadence:** The code relies on polling, so a host that bursts bytes too quickly may overrun the AVR’s ability to service `USIOIF`. Keep transfers paced and consider reading responses on a subsequent transfer rather than back-to-back command/response loops. 【F:serial.c†L60-L74】【F:serial.h†L21-L39】

### Command sequencing and statefulness

- **Nibble ordering matters:** Keyboard and arbitrary device data are assembled from two nibbles; if a nibble is dropped or reordered, the combined byte will be wrong. There is no resynchronization to recover state. 【F:serial.c†L110-L142】【F:serial.c†L190-L219】
- **“No response” is a valid outcome:** `handle_serial_data()` returns 0 when there is no response byte; SPI will then shift out whatever is in `USIDR` (often the last response). Always treat response bytes as meaningful only when you expect them (e.g., immediately after a status read command). 【F:serial.c†L60-L74】【F:serial.c†L296-L297】

### Raspberry Pi Pico (RP2040) host notes

- **SPI mode and clocking:** The AVR USI uses an external clock in 3‑wire mode. Use a standard SPI mode (CPOL/CPHA as required by your wiring and the AVR USI’s expectations) and verify with a logic analyzer; the firmware doesn’t self‑calibrate for mode mismatches. 【F:serial.c†L54-L74】
- **Dummy-transfer response pattern:** Plan for 2‑byte sequences for read‑type operations (command byte, then dummy byte to clock out the response). The dummy byte value can be `0x00` or `0xFF`. 【F:serial.c†L60-L74】
- **Respect pacing:** If you are generating events rapidly (e.g., mouse motion), queue them on the Pico side and pace SPI writes so the AVR’s polling loop can keep up (target ≥50 µs between bytes unless you know your firmware loop is faster in practice). 【F:serial.h†L21-L39】
