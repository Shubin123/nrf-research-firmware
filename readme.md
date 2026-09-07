# RFStorm nRF24LU1+ Research Firmware

Firmware and research tools for Nordic Semiconductor nRF24LU1+ based USB dongles and breakout boards.

## Requirements

- SDCC (minimum version 3.1.0)
- GNU Binutils
- Python 3.9 or newer
- PyUSB
- platformio

Create an isolated Python environment and install the host-side dependencies:

```
python3 -m venv .venv
.venv/bin/pip install -U pip pyusb pyserial
```

On Ubuntu, install the build dependencies with:

```
sudo apt-get install sdcc binutils platformio
```

`bastille` (documented below) automatically uses `.venv` when it is present.

## Supported Hardware

The following hardware has been tested and is known to work.

- CrazyRadio PA USB dongle
- SparkFun nRF24LU1+ breakout board
- Logitech Unifying dongle (model C-U0007, Nordic Semiconductor based)

## Build the firmware

```
make
```

## Quick start with `bastille`

The root-level `bastille` command provides a single entry point for building,
flashing, and using the tools. Hardware-touching commands automatically request
root privileges; help and build commands do not.

```
./bastille build
./bastille status
./bastille scan --help
./bastille sniff -a 61:49:66:82:03
```

Useful commands:

```
./bastille flash [firmware.bin]
./bastille flash-logitech [formatted.bin] [formatted.ihx]
./bastille restore-logitech original-firmware.hex
./bastille spi-flash [firmware.bin]
./bastille spi-dump
./bastille map -a 61:49:66:82:03
./bastille tone -c 5
```

`scan`, `sniff`, `map`, and `tone` pass any following arguments directly to the
underlying tool. Use `./bastille <command> --help` for the current option list.

### Complete `bastille` command reference

| Command | Purpose | Arguments and defaults |
| --- | --- | --- |
| `./bastille build` | Build the RFStorm firmware. | None. Produces `bin/dongle.bin`, `bin/dongle.formatted.bin`, and `bin/dongle.formatted.ihx`. |
| `./bastille clean` | Remove firmware build artifacts. | None. |
| `./bastille status` | Report the USB identity of a connected supported dongle. | None. If it cannot see a device, retry with `sudo ./bastille status`. |
| `./bastille flash [firmware.bin]` | Flash a CrazyRadio PA, nRF24LU1+ breakout, or an already-flashed RFStorm device over USB. | Optional firmware path; defaults to `bin/dongle.bin`. Requires a device in a compatible bootloader/firmware state. |
| `./bastille flash-logitech [formatted.bin] [formatted.ihx]` | Replace the stock firmware on a compatible Logitech Unifying C-U0007 with RFStorm. | Defaults to `bin/dongle.formatted.bin` and `bin/dongle.formatted.ihx`. Build first with `./bastille build`. |
| `./bastille restore-logitech original-firmware.hex` | Restore a Logitech Unifying receiver from an original Logitech firmware image. | The `.hex` image path is required. |
| `./bastille spi-flash [firmware.bin]` | Recover or flash an nRF24LU1+ over SPI using the Teensy flasher. | Optional firmware path; defaults to `bin/dongle.bin`. Requires the separately-built/wired Teensy flasher. |
| `./bastille spi-dump` | Read nRF24LU1+ flash over SPI and print Intel HEX to standard output. | None. Requires the separately-built/wired Teensy flasher. Redirect output to save it: `./bastille spi-dump > backup.hex`. |
| `./bastille scan [options]` | Passively sweep for Enhanced ShockBurst devices. | For example: `./bastille scan -c 1 2 3 -p A9`. See `./bastille scan --help`. |
| `./bastille sniff -a ADDRESS [options]` | Follow a known nRF24 address and print decoded packets. | An address is required, for example `./bastille sniff -a 61:49:66:82:03`. See `./bastille sniff --help`. |
| `./bastille map -a ADDRESS [options]` | Probe a star network for active addresses. | A known address is required, for example `./bastille map -a 61:49:66:82:03`. This command transmits probes. See `./bastille map --help`. |
| `./bastille tone -c CHANNEL [options]` | Transmit a continuous RF test tone on a channel. | For example: `./bastille tone -c 5`. Use only where transmission is authorized; see `./bastille tone --help`. |
| `./bastille docs [query]` | List or open MouseJack disclosure advisories and whitepapers from a sibling clone. | Optional filename search, such as `./bastille docs logitech`. Set `MOUSEJACK_DIR` if the `mousejack` repository is not at `../mousejack`. |

The USB flashing and radio commands may require administrator privileges.
`bastille` requests them automatically; it leaves `status` as a non-privileged
check first so it can be used without a password prompt.

## Flash over USB

nRF24LU1+ chips come with a factory programmed bootloader occupying the topmost 2KB of flash memory. The CrazyRadio firmware and RFStorm research firmware support USB commands to enter the Nordic bootloader.

Dongles and breakout boards can be programmed over USB if they are running one of the following firmwares:

- Nordic Semiconductor Bootloader
- CrazyRadio Firmware
- RFStorm Research Firmware

To flash the firmware over USB:

```
sudo make install
```

## Flash a Logitech Unifying dongle

*The most common Unifying dongles are based on the nRF24LU1+, but some use chips from Texas Instruments.
This firmware is only supported on the nRF24LU1+ variants, which have a model number of C-U0007. The flashing
script will automatically detect which type of dongle is plugged in, and will only attempt to flash the nRF24LU1+ variants.*

To flash the firmware over USB onto a Logitech Unifying dongle:

```
sudo make logitech_install
```

## Flash a Logitech Unifying dongle back to the original firmware

Download and extract the Logitech firmware image, which will be named `RQR_012_005_00028.hex` or similar. Then, run the following command to flash the Logitech firmware onto the dongle:

```
sudo ./prog/usb-flasher/logitech-usb-restore.py [path-to-firmware.hex]
```

## Flash over SPI using a Teensy

If your dongle or breakout board is bricked, you can alternatively program it over SPI using a Teensy.

This has only been tested with a Teensy 3.1/3.2, but is likely to work with other Arduino variants as well.

### Build and Upload the Teensy Flasher

```
platformio run --project-dir teensy-flasher --target upload
```

### Connect the Teensy to the nRF24LU1+

| Teensy | CrazyRadio PA | Sparkfun nRF24LU1+ Breakout |
| ------ | ---------- | -------- |
| GND | 9 | GND |
| 8 | 3 | RESET |
| 9 | 2 | PROG |
| 10 | 10 | P0.3 |
| 11 | 6 | P0.1 |
| 12 | 8 | P0.2 |
| 13 | 4 | P0.0 |
| 3.3V | 5 | VIN |

### Flash the nRF24LU1+

```
sudo make spi_install
```

# Python Scripts

## scanner

Pseudo-promiscuous mode device discovery tool, which sweeps a list of channels and prints out decoded Enhanced Shockburst packets.

```
usage: ./nrf24-scanner.py [-h] [-c N [N ...]] [-v] [-l] [-p PREFIX] [-d DWELL]

options:
  -h, --help                          show this help message and exit
  -c N [N ...], --channels N [N ...]  RF channels
  -v, --verbose                       Enable verbose output
  -l, --lna                           Enable the LNA (for CrazyRadio PA dongles)
  -i, --index INDEX                   Dongle index
  -p PREFIX, --prefix PREFIX          Promiscuous mode address prefix
  -d DWELL, --dwell DWELL             Dwell time per channel, in milliseconds
```

Scan for devices on channels 1-5

```
./nrf24-scanner.py -c {1..5}
```

Scan for devices with an address starting in 0xA9 on all channels

```
./nrf24-scanner.py -p A9
```


## sniffer

Device following sniffer, which follows a specific nRF24 device as it hops, and prints out decoded Enhanced Shockburst packets from the device.

```
usage: ./nrf24-sniffer.py [-h] [-c N [N ...]] [-v] [-l] -a ADDRESS [-t TIMEOUT] [-k ACK_TIMEOUT] [-r RETRIES]

options:
  -h, --help                                 show this help message and exit
  -c N [N ...], --channels N [N ...]         RF channels
  -v, --verbose                              Enable verbose output
  -l, --lna                                  Enable the LNA (for CrazyRadio PA dongles)
  -i, --index INDEX                          Dongle index
  -a ADDRESS, --address ADDRESS              Address to sniff, following as it changes channels
  -t TIMEOUT, --timeout TIMEOUT              Channel timeout, in milliseconds
  -k ACK_TIMEOUT, --ack_timeout ACK_TIMEOUT  ACK timeout in microseconds, accepts [250,4000], step 250
  -r RETRIES, --retries RETRIES              Auto retry limit, accepts [0,15]
  -p PING_PAYLOAD, --ping_payload PING_PAYLOAD
                                             Ping payload, e.g. 0F:0F:0F:0F
```

Sniff packets from address 61:49:66:82:03 on all channels

```
./nrf24-sniffer.py -a 61:49:66:82:03
```

## network mapper

Star network mapper, which attempts to discover the active addresses in a star network by changing the last byte in the given address, and pinging each of 256 possible addresses on each channel in the channel list.

```
usage: ./nrf24-network-mapper.py [-h] [-c N [N ...]] [-v] [-l] [-i INDEX] -a ADDRESS [-k ACK_TIMEOUT] [-r RETRIES] [-p PING_PAYLOAD]

options:
  -h, --help                                 show this help message and exit
  -c N [N ...], --channels N [N ...]         RF channels
  -v, --verbose                              Enable verbose output
  -l, --lna                                  Enable the LNA (for CrazyRadio PA dongles)
  -i, --index INDEX                          Dongle index
  -a ADDRESS, --address ADDRESS              Known address
  -k ACK_TIMEOUT, --ack_timeout ACK_TIMEOUT  ACK timeout in microseconds, accepts [250,4000], step 250
  -r RETRIES, --retries RETRIES              Auto retry limit, accepts [0,15]
  -p PING_PAYLOAD, --ping_payload PING_PAYLOAD
                                             Ping payload, e.g. 0F:0F:0F:0F
```

Map the star network that address 61:49:66:82:03 belongs to

```
./nrf24-network-mapper.py -a 61:49:66:82:03
```

## continuous tone test

The nRF24LU1+ chips include a test mechanism to transmit a continuous tone, the frequency of which can be verified if you have access to an SDR. There is the potential for frequency offsets between devices to cause unexpected behavior. For instance, one of the SparkFun breakout boards that was tested had a frequency offset of ~300kHz, which caused it to receive packets on two adjacent channels.

This script will cause the transceiver to transmit a tone on the first channel that is passed in.

```
usage: ./nrf24-continuous-tone-test.py [-h] [-c N [N ...]] [-v] [-l]

options:
  -h, --help                          show this help message and exit
  -c N [N ...], --channels N [N ...]  RF channels
  -v, --verbose                       Enable verbose output
  -l, --lna                           Enable the LNA (for CrazyRadio PA dongles)
  -i, --index INDEX                   Dongle index

```

Transmit a continuous tone at 2405MHz

```
./nrf24-continuous-tone-test.py -c 5
```
