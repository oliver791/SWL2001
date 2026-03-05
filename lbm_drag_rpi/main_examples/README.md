# LoRaWAN SX1276 CSV Logger — Raspberry Pi

Periodical LoRaWAN uplink application for SX1276 (RFM95W/HPD13A) on Raspberry Pi, based on Semtech's SWL2001 LoRa Basics Modem (lbm_lib).

Fork of [nondetalle/SWL2001](https://github.com/nondetalle/SWL2001) with CSV logging of all LoRaWAN events (TX, RX/DOWNDATA, JOIN, JOINFAIL, TXDONE) with full radio and MAC-layer parameters, and configurable uplink parameters via command line.

## Features

- **Optional CSV logging** (`--log` flag): logging is disabled by default, enable it via command line
- Configurable uplink period via command line (default: 60 s, min: 1 s)
- Configurable packet size via command line (default: 12 bytes, max: 222 bytes)
- Fixed or variable packet size mode (`fixed` or `var`):
  - `fixed`: every uplink uses the same payload size
  - `var`: each uplink uses a random size between 1 and the configured max
- Random payload generation: payload bytes are randomized at each transmission
- CSV logging with timestamped rows and JSON-formatted EXTRA field for easy post-processing (Python, etc.)
- Captures per-event:
  - Radio: SF, BW, CR, frequency (Hz/MHz), TX output power
  - MAC: data rate, ADR data rate, TX power, nb_trans, duty cycle, RX1 delay, available TX channels
  - Downlink: RSSI (dBm), SNR (dB), payload hex, port
- Supports EU868 region (configurable)

## Project Structure
```
SWL2001/                              &lt;- Semtech root (clone from GitHub)
|-- lbm_lib/                          &lt;- Semtech LoRa Basics Modem library
+-- lbm_drag_rpi/                     &lt;- THIS REPOSITORY
    |-- main.c                        &lt;- Entry point
    |-- main.h
    |-- modem_pinout.h                &lt;- GPIO pin configuration
    |-- Makefile                      &lt;- Build system
    |-- README.md                     &lt;- This file
    |-- app_makefiles/                &lt;- Makefile includes
    |-- main_examples/
    |   |-- main_periodical_uplink.c  &lt;- Main application + CSV logger
    |   |-- example_options.h         &lt;- LoRaWAN credentials (DevEUI, AppKey)
    |   +-- main_porting_tests.c
    |-- radio_hal/                    &lt;- SX1276 HAL (SPI, GPIO)
    |-- smtc_hal_drag_rpi/            &lt;- Platform HAL for Raspberry Pi
    +-- smtc_modem_hal/               &lt;- Modem HAL implementation
```

## Prerequisites

- Raspberry Pi (tested on RPi 3B+) with Raspberry Pi OS
- SX1276-based module
- SPI enabled: run `sudo raspi-config` -> Interface Options -> SPI -> Enable
- Build tools: run `sudo apt update && sudo apt install build-essential git`

## Installation & Build

### 1. Clone Semtech's SWL2001

```bash
cd ~
git clone https://github.com/Lora-net/SWL2001.git
cd SWL2001
```

### 2. Clone this repository

```bash
cd ~/SWL2001
git clone https://github.com/oliver791/lbm_drag_rpi.git
```

### 3. Configure LoRaWAN credentials

Edit `lbm_drag_rpi/main_examples/example_options.h` with your device credentials:

```c
#define USER_LORAWAN_DEVICE_EUI  { 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 }
#define USER_LORAWAN_JOIN_EUI    { 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 }
#define USER_LORAWAN_APP_KEY     { 0x00, ..., 0x00 }
#define USER_LORAWAN_GEN_APP_KEY { 0x00, ..., 0x00 }
```

### 4. Build

```bash
# Build the application
cd ~/SWL2001/lbm_drag_rpi
make full_sx1276
```

The binary is produced in `build_sx1276_drpi/`.

### 5. Run

```bash
sudo ./build_sx1276_drpi/app_sx1276.elf [period_s] [packet_size] [fixed|var] [--log]
```

> `sudo` is required for SPI and GPIO access.

### 6. Examples

```bash
# Default: 60s period, 12 bytes, fixed size, NO logging
sudo ./build_sx1276_drpi/app_sx1276.elf

# 30s period, 50 bytes fixed, with CSV logging
sudo ./build_sx1276_drpi/app_sx1276.elf 30 50 fixed --log

# 10s period, up to 222 bytes variable (random 1-222 each TX), with CSV logging
sudo ./build_sx1276_drpi/app_sx1276.elf 10 222 var --log
```

## CSV Output

When `--log` is enabled, a file named `lorawan-YYYY-MM-DD--HH-MM-SS.csv` is created automatically at startup.

### Columns

| Column    | Description                                        |
|-----------|----------------------------------------------------|
| TIMESTAMP | Local time (YYYY-MM-DD--HH-MM-SS)                 |
| DEVEUI    | Device EUI (hex)                                   |
| EVENT     | TX, DOWNDATA, JOINED, JOINFAIL, TXDONE             |
| DATA      | Payload (hex)                                      |
| SF        | Spreading Factor (SF7-SF12)                        |
| EXTRA     | JSON object with event-specific parameters         |

### EXTRA field format

The EXTRA field is a JSON object enclosed in double-quotes with escaped internal quotes. This allows easy parsing with `json.loads()` in Python.

### Example CSV

```csv
TIMESTAMP,DEVEUI,EVENT,DATA,SF,EXTRA
&quot;2026-02-24--14-44-25&quot;,&quot;636E616D00000002&quot;,&quot;JOINFAIL&quot;,&quot;&quot;,&quot;SF12&quot;,&quot;{&quot;&quot;reason&quot;&quot; : &quot;&quot;JOINFAIL&quot;&quot;}&quot;
&quot;2026-02-24--14-44-45&quot;,&quot;636E616D00000002&quot;,&quot;JOINED&quot;,&quot;&quot;,&quot;SF8&quot;,&quot;{&quot;&quot;reason&quot;&quot; : &quot;&quot;Modem is now joined&quot;&quot;}&quot;
&quot;2026-02-24--14-44-45&quot;,&quot;636E616D00000002&quot;,&quot;TX&quot;,&quot;B49F&quot;,&quot;SF8&quot;,&quot;{&quot;&quot;port&quot;&quot; : &quot;&quot;101&quot;&quot;, &quot;&quot;counter&quot;&quot; : &quot;&quot;0&quot;&quot;, &quot;&quot;size&quot;&quot; : &quot;&quot;2&quot;&quot;, &quot;&quot;size_mode&quot;&quot; : &quot;&quot;variable&quot;&quot;, &quot;&quot;size_max&quot;&quot; : &quot;&quot;40&quot;&quot;, &quot;&quot;period&quot;&quot; : &quot;&quot;10&quot;&quot;}&quot;
&quot;2026-02-24--14-44-49&quot;,&quot;636E616D00000002&quot;,&quot;TXDONE&quot;,&quot;&quot;,&quot;SF12&quot;,&quot;{&quot;&quot;status&quot;&quot; : &quot;&quot;OK&quot;&quot;}&quot;
```

## Troubleshooting

| Problem                          | Solution                                                    |
|----------------------------------|-------------------------------------------------------------|
| SPI open failed                  | Enable SPI: `sudo raspi-config`                             |
| Permission denied                | Run with `sudo`                                             |
| No JOIN                          | Check credentials in `example_options.h`                    |
| CSV file empty                   | Check write permissions and that `--log` flag is set        |
| Compilation errors               | Verify `C_INCLUDE_PATH` export (see Build step 4)           |
| undefined reference linking errors | Ensure you are inside `lbm_drag_rpi/` when running `make` |

## How It Works
+--------------+    SPI     +----------+   LoRaWAN   +--------------+
| Raspberry Pi |<---------->|  SX1276  |<----------->| LoRa Gateway |
|              |  GPIO      | (RFM95W) |   868 MHz   |              |
+------+-------+            +----------+             +--------------+
       |
       v
  lorawan-*.csv
  (local file)

```

1. The application joins the LoRaWAN network (OTAA)
2. Every N seconds (configurable), it sends an uplink with a random payload
3. When `--log` is enabled, all events (TX, RX, JOIN, JOINFAIL) are logged to a local CSV file
4. Radio parameters (SF, BW, CR, frequency) and MAC parameters (data rate, ADR, duty cycle) are captured at each TX event
5. Downlink RSSI/SNR are extracted from metadata

## License

See the original project license: [Semtech SWL2001](https://github.com/Lora-net/SWL2001)
