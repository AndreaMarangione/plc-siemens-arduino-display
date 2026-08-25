<div align="center">

# plc-siemens-arduino-display

**An Arduino that listens on TCP and turns what a Siemens PLC sends into readable digits on a shop-floor display.**

Ethernet link between a SIMATIC S7-1500 and a 6-digit 7-segment module,
carrying live encoder values, fault codes and link diagnostics.

![Arduino](https://img.shields.io/badge/Arduino-00979D?style=flat-square&logo=arduino&logoColor=white)
![C++](https://img.shields.io/badge/C++-00599C?style=flat-square&logo=cplusplus&logoColor=white)
![SIMATIC S7-1500](https://img.shields.io/badge/SIMATIC%20S7--1500-009999?style=flat-square&logo=siemens&logoColor=white)

</div>

---

## What it is

A PLC has the numbers — the position of an axis, the state of a machine, the
code of an active alarm — but no cheap way of putting them in front of an
operator standing at the machine. An industrial HMI for a single value is
expensive and slow to wire in.

This is the low-cost alternative: an Arduino with an Ethernet shield
sits on the plant network as a **TCP server**, the PLC connects to it as a
client and streams short ASCII frames, and the Arduino decodes them onto a
six-digit display. The same channel carries fault codes, and the firmware
watches the link itself so that a dead connection is visible on the display
rather than frozen on the last value received.

---

## Architecture

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/architecture.png">
  <img alt="Architecture" src="assets/architecture.png">
</picture>

---

## Protocol

The PLC sends a byte at a time; the Arduino reads one byte per loop pass and
runs a small state machine over it. A frame opens with a control character,
carries up to six payload characters and closes with `f`.

| Byte | Role | Effect on the Arduino |
| --- | --- | --- |
| `e` | Value frame, axis **moving** | Clears the buffer, drops any latched fault, raises `D9`, starts capturing |
| `t` | Value frame, **target reached** | Same as `e`, and the value is shown blinking |
| *payload* | Up to six characters | Written into the display buffer in order of arrival |
| `f` | End of frame | Stops capturing, clears the alarm latch, releases `D9` |
| `a` | Alarm frame | Raises `D9` and arms the fault latch |
| *code* | Single character after `a` | Stored as the PLC fault code |

**Examples**

| Frame | Result on the display |
| --- | --- |
| `e125430f` | `125430`, steady — the axis is moving |
| `t001250f` | `001250`, blinking — the axis is at target |
| `a2f` | `plcF02` — fault 2 reported by the PLC |

A value frame clears a displayed fault, so recovery needs no dedicated
reset command.

---

## Hardware

| Part | Notes |
| --- | --- |
| **Arduino** | Any arduino board with the standard header layout |
| **W5100 Ethernet shield** | driven over SPI |
| **TM1637 6-digit display** | Common 4-wire module, driven at `BRIGHT_HIGH` |

---

## Wiring

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/wiring.png">
  <img alt="Wiring between the Arduino and the TM1637 display" src="assets/wiring.png">
</picture>

| TM1637 | Arduino |
| --- | --- |
| `CLK` | `D3` |
| `DIO` | `D2` |
| `VCC` | `5V` |
| `GND` | `GND` |

> [!IMPORTANT]
> `CLK` is on **D3** and `DIO` on **D2** — the opposite of the assignment
> used in most TM1637 tutorials. Swapping the two leaves the display blank.

---

## Notes and limitations

- **One byte per loop pass.** Throughput follows the loop rate rather than
  the network; the PLC should pace its frames accordingly.
- **Timers are loop counters, not milliseconds.** The watchdog threshold and
  the blink cycle shift if the loop time changes.
- **Frames must stay within six payload characters.** The capture index is
  not bounded, so a longer frame writes past the end of the buffer.
- **The watchdog reacts to silence, not to link loss.** It measures data
  availability, so a PLC that is connected but idle also raises `no CON`.
- **No authentication or encryption.** The device belongs on a segregated
  machine network, never on a routed or public one.

---

## License

Distributed under the **GNU General Public License v3.0**. See [`LICENSE`](LICENSE) for the full text.
