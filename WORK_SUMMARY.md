# Session Work Summary

This documents the work done in this repo to get the RFStorm firmware building,
flashed onto a Logitech Unifying dongle (model C-U0007, USB ID `046d:c52b`), and
its Python tooling running on a modern macOS + Python 3.14 machine. The codebase
was last touched in 2016 and everything Python here was written for Python 2.

## 1. Build system

- `make` builds cleanly (`bin/dongle.bin`, `bin/dongle.formatted.bin`/`.ihx`).
- **Fixed:** `Makefile`'s SDCC version-check used `grep -P` (PCRE), which macOS's
  BSD `grep` doesn't support, so the check silently broke and threw spurious
  shell errors on every build. Replaced with a portable `sed`-based extraction.

## 2. Firmware flashed to the dongle

The C-U0007 is now running RFStorm research firmware (confirmed via
`system_profiler` / PyUSB: `1915:0102`, manufacturer "RFStorm", product
"Research Firmware" — previously `046d:c52b`, stock Logitech HID firmware).

Flashed via `prog/usb-flasher/logitech-usb-flash.py`, which required `sudo`
(raw USB kernel-driver detach). This needs to run from an interactive terminal,
not through the assistant, since `sudo` needs a real password prompt.

## 3. Python 2 → 3 port

Every Python file in this repo was Python 2 (`print` statements, `except X, e:`,
`.decode('hex')`, `ord()`/`chr()` on byte strings, `xrange`, integer `/` division,
implicit relative imports). None of it ran under the system's Python 3.14 (there
is no `python` binary on this machine at all, only `python3`). Ported to Python 3,
byte-for-byte equivalent behavior preserved:

| File | Notes |
|---|---|
| `prog/usb-flasher/unifying.py` | Also fixed 4x `is_kernel_driver_active()` calls crashing with `USBError: Entity not found` on macOS when a device exposes fewer interfaces than assumed (found via live hardware test); fixed `raise exception(...)` → `Exception` (undefined name, pre-existing bug) |
| `prog/usb-flasher/logitech-usb-flash.py` | Hex/CRC payload construction ported to `bytes`; verified offline against the actual built firmware (1665 payloads, correct sizes, CRC `0x17f9`) |
| `prog/usb-flasher/logitech-usb-restore.py` | Same port; fixed wrong arg-count check + missing `sys.exit(1)` after usage error |
| `prog/usb-flasher/usb-flash.py` | Same port; `array.tostring()` → `.tobytes()`; integer division fix |
| `prog/teensy-flasher/python/spi-flash.py`, `spi-dump.py` | Same port; installed `pyserial`; removed a dead/broken `__init__self` method (typo meant it was never called) |
| `tools/lib/nrf24.py` | `map(ord, x)` → `list(x)` throughout (bytes iterate as ints in Py3); added missing `sys` import (pre-existing bug — `sys.exit()` was called without it ever being imported) |
| `tools/lib/common.py` | `from nrf24 import *` → `from .nrf24 import *` (Python 3 dropped implicit relative imports) |
| `tools/nrf24-scanner.py`, `nrf24-sniffer.py`, `nrf24-network-mapper.py`, `nrf24-continuous-tone-test.py` | Same port; `xrange` → `range`; `.decode('hex')` → `bytes.fromhex()` |

Verified via `py_compile` on every file, and `--help` / argument-parsing tests
for all four CLI tools (matches the options documented in `readme.md`).

## 4. Python environment

- `.venv/` — local virtualenv (Python 3.14), since Homebrew's Python blocks
  global `pip install` (PEP 668). Not committed (auto-`.gitignore`d by `venv`
  itself).
- Installed: `pyusb` 1.3.1, `pyserial` 3.5.

## 5. `bastille` — unified command runner (new file)

Single executable script at the repo root wrapping every tool so nothing
requires remembering venv paths, `sudo`, or per-script CLI details:

```
./bastille build                          Build the firmware (make)
./bastille clean                          Remove build artifacts (make clean)
./bastille status                         Detect what firmware state the dongle is in
./bastille flash [firmware.bin]           Flash over USB (CrazyRadio PA / breakout / RFStorm dongle)
./bastille flash-logitech [bin] [ihx]     Flash a Logitech Unifying (C-U0007) dongle with RFStorm
./bastille restore-logitech <hex>         Restore a Logitech Unifying dongle to stock firmware
./bastille spi-flash [firmware.bin]       Flash over SPI via a Teensy (recovery path)
./bastille spi-dump                       Dump flash over SPI via a Teensy (prints Intel HEX)
./bastille docs [query]                   Browse MouseJack disclosure advisories/whitepapers
./bastille scan / sniff / map / tone      The four recon tools (-h for each tool's full real options)
```

Behavior:
- Auto re-execs itself under `.venv/bin/python3` if PyUSB isn't importable in
  whatever interpreter launched it (detected via `sys.prefix`, not a naive
  resolved-executable-path comparison — the venv's `python3` is a symlink to
  the system interpreter on macOS, so a naive check doesn't distinguish them).
- Auto re-execs under `sudo` for hardware-touching subcommands only (not for
  `build`/`clean`/`--help`, and not for `-h`/`--help` on `scan`/`sniff`/`map`/`tone`).
- `scan`/`sniff`/`map`/`tone` forward all trailing args verbatim to the real
  script (including `-h`, which shows that script's actual full option list).

## 6. Recon tool UX fixes

All four tools ran silently with zero output while working correctly (no
traffic in range ≠ hung), and dumped a raw Python traceback on Ctrl+C.

- `nrf24-scanner.py`, `nrf24-sniffer.py`: added a startup banner and a
  heartbeat log line every 3s so it's clear the scan/follow loop is alive,
  even with no packets captured yet.
- `nrf24-continuous-tone-test.py`: added a startup message; replaced the
  `while True: pass` busy-loop (100% CPU) with `time.sleep(1)`.
- `nrf24-network-mapper.py`: reports whatever addresses were found so far if
  stopped early (a full sweep can take a long time).
- All four now catch `KeyboardInterrupt` and print a clean one-line summary
  instead of a traceback.

## 7. MouseJack context (reference only, not in this repo)

Cloned `BastilleResearch/mousejack` to `~/Downloads/mousejack` (sibling
directory, submodule skipped since this repo already *is* that submodule's
content, just modified). Confirmed this repo (`nrf-research-firmware`) is
pinned in that upstream repo as a submodule at commit `02b84d1` — this repo's
current `HEAD`.

That repo contains **no additional executable code** — only disclosure
advisories (`doc/advisories/`, 11 files covering Logitech, Dell, Microsoft,
HP, Gigabyte, AmazonBasics, Lenovo) and two whitepapers/slide decks
(`doc/pdf/`). Bastille never open-sourced a keystroke-injection PoC; only the
recon tooling (this repo) and the disclosure documentation were published.
`./bastille docs [query]` browses this material.

## Known open items

- Live hardware test of `scan`/`sniff`/`map`/`tone` (and the heartbeat/Ctrl+C
  fixes) still needs to be run and confirmed by the user in their own
  terminal — `sudo` requires an interactive password prompt this session
  can't supply.
- `C-U0021` compatibility as a scan target is untested / not documented in
  any Bastille advisory (`C-U0008` is, under two different USB IDs — see
  chat history). Any ESB-based receiver should work as a passive scan
  target regardless of chip vendor; only *re-flashing* a second dongle
  requires it specifically be nRF24LU1+-based, which the current
  `unifying.py` only auto-detects via USB ID `046d:c52b`.
- No keystroke/mouse-injection ("mousejack") command exists in `bastille` —
  deliberately not built; would be new code on top of the existing
  `transmit_payload()`/`transmit_payload_generic()` primitives in
  `tools/lib/nrf24.py`, not something pulled from upstream.
