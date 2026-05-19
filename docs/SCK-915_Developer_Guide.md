# SpaceCommsKit SCK-915 Developer Guide

**Version 2.0 — May 2026**
**SpaceCommsKit — https://spacecommskit.com/docs**

> This document covers the full SCK-915 development stack — from the
> OpenLST protocol foundation through the production SCK-PBL-1 hardware.
> If you are new to the system, read Section 1 before anything else.

---

## Quick Reference — Search Tags

Every cross-reference in the firmware source uses a `[SCK-DEV: TAG]` comment.
Search for any tag in the codebase to jump directly to the relevant section:

| Tag | Topic | Section |
|-----|-------|---------|
| `SCK-DEV: BOARD_CONFIG` | Hardware configuration overview | 2.1 |
| `SCK-DEV: RF_FRONTEND` | CC1190 PA/LNA control | 2.2 |
| `SCK-DEV: RF_CONFIG` | 915MHz register values | 2.3 |
| `SCK-DEV: TX_POWER` | Power mode / bench vs field | 2.4 |
| `SCK-DEV: BOOTLOADER` | Bootloader timeout | 2.5 |
| `SCK-DEV: ADD_COMMAND` | Adding new commands | 3.1 |
| `SCK-DEV: CHUNKING` | Chunking large responses | 3.2 |
| `SCK-DEV: TIMING` | Prototype board timing issues | 3.3 |
| `SCK-DEV: ESP_FRAMING` | UART framing protocol | 3.4 |
| `SCK-DEV: RESPONSE_FORMAT` | Response string conventions | 3.5 |
| `SCK-DEV: BEACON` | Autonomous GPS beacon | 3.6 |
| `SCK-DEV: SD_FLIGHT_LOG` | SD card flight log format | 3.7 |

---

## 1.0 System Overview

### 1.1 Architecture

```
Ground Station (Windows C#)
    ↕ USB / Serial — COM port — 115200 baud — ESP framing
SCK-915 RF Board (CC1110 + CC1190)        ← ground unit
    ↕ 915MHz RF — 7.4 kBaud — 2-FSK + FEC
SCK-915 RF Board (CC1110 + CC1190)        ← remote / payload unit
    ↕ UART0 — 115200 baud — ESP framing
SCK-PBL-1 Payload Board (Raspberry Pi Pico)
    ├── OV2640 Camera    (SPI0 + I2C0)
    ├── MicroSD Card     (SPI0)
    ├── GPS NEO-6M       (UART1)
    └── BMP581 Altimeter (I2C1)
```

The ground unit RF board relays packets transparently between the ground
station and the remote unit. Only the remote unit requires the custom
`PICO_MSG` handler in `board.c`. This mirrors the original OpenLST
Explorer Kit architecture where Board 0001 was the transparent relay
and Board 0004 ran the custom firmware.

### 1.2 Hardware Variants

| Board | Description | Status |
|-------|-------------|--------|
| SCK-915 Prototype | Hand-wired jumper development board | Dev/test only — see Section 3.3 |
| SCK-915 RF Board | CC1110 + CC1190 RF front end PCB | Production |
| SCK-PBL-1 v1.0 Payload | Pico + camera + GPS + BMP581 + SD PCB | Production — BMP581 altimeter |
| SCK-PBL-1 v1.1 Payload | Pico + camera + GPS + MS5611 + SD PCB | Production — MS5611 altimeter |

### 1.3 Firmware Components

| File | Language | Runs On | Purpose |
|------|----------|---------|---------|
| `board.h` | C | CC1110 | Hardware configuration and defines |
| `board.c` | C | CC1110 | Board init and PICO_MSG relay |
| `main.py` | MicroPython | Pico | Payload pipeline and command handlers (MS5611 active, BMP581 preserved as comments) |
| `sdcard.py` | MicroPython | Pico | MicroPython SD card SPI driver |

### 1.4 Repository Layout

```
openlst-master/
├── Makefile                    ← top-level build
├── open-lst/                   ← DO NOT MODIFY
│   ├── common/                 ← uart0.c uart1.c radio.c ...
│   ├── radio/                  ← main.c commands.c ...
│   └── bootloader/
└── radios/openlst_437/         ← YOUR CODE LIVES HERE
    ├── Board.mk                ← build config
    ├── board.h                 ← hardware defines + custom_commands declaration
    └── board.c                 ← board_init() + custom_commands()
```

> **Never modify files inside `open-lst/`.** All customization goes in
> `radios/openlst_437/`. This keeps your changes clean and upstream
> updates safe.

---

## 2.0 Hardware Configuration Reference

### 2.1 Board Configuration — `board.h` [SCK-DEV: BOARD_CONFIG]

`board.h` configures the OpenLST firmware for SCK-915 hardware.
Key defines:

| Define | Value | Purpose |
|--------|-------|---------|
| `F_CLK` | 27000000 | Crystal frequency — must match physical crystal |
| `CUSTOM_BOARD_INIT` | 1 | Enables `board_init()` in board.c |
| `BOARD_HAS_TX_HOOK` | 1 | Enables `board_pre_tx()` macro |
| `BOARD_HAS_RX_HOOK` | 1 | Enables `board_pre_rx()` macro |
| `ADCCFG_CONFIG` | 0b00000011 | Enables AN0 and AN1 for voltage sense |
| `RADIO_RANGING_RESPONDER` | 1 | Enables ranging response |
| `CUSTOM_COMMANDS` | 1 | Enables custom command dispatch |

**Build flags** (set in `Board.mk`):

```makefile
openlst_437_CFLAGS := \
    -DCUSTOM_BOARD_INIT   \
    -DCUSTOM_COMMANDS     \
    -DUART0_ENABLED=1     \
    -I$(openlst_437_DIR)
```

> **Always use Clean + Build after any source change.** Stale `.rel`
> object files are the most common cause of mysterious firmware behavior
> that is difficult to trace. This is not optional — it is required.

### 2.2 CC1190 RF Front End [SCK-DEV: RF_FRONTEND]

The CC1190 provides PA and LNA functions between the CC1110 and antenna.

**Physical connections:**

| CC1190 Pin | Signal | CC1110 Pin | Control Method |
|-----------|--------|-----------|---------------|
| Pin 8 (PA_EN) | PA enable | P1.6 (GDO1) | Automatic via IOCFG1 |
| Pin 7 (LNA_EN) | LNA enable | P1.7 (GDO2) | Automatic via IOCFG2 |
| Pin 6 (HGM) | High gain mode | P2.0 | Software via `_rf_high_gain` |

**HGM power modes:**

| `_rf_high_gain` | HGM State | Total Output Power | Use Case |
|----------------|-----------|-------------------|---------|
| 0 | LOW | ~+10 dBm | Bench / indoor testing |
| 1 | HIGH | ~+27 dBm | Field / HAB / LEO mission |

GDO pins are configured using inverted power-down signals so the CC1190
enable lines go HIGH automatically when the corresponding stage is active.
The firmware never manually toggles PA_EN or LNA_EN — the CC1110 radio
hardware manages this through the IOCFG register settings.

### 2.3 RF Configuration — 915MHz [SCK-DEV: RF_CONFIG]

All register values were derived using SmartRF Studio 7 for the CC1110
with a 27MHz crystal.

| Parameter | Value |
|-----------|-------|
| Base Frequency | 914.999512 MHz |
| Crystal | 27.000 MHz |
| Modulation | 2-FSK |
| Data Rate | 7.415 kBaud |
| Deviation | 3.708 kHz |
| RX Filter BW | 60.268 kHz |

**Frequency control word calculation:**
```
FREQ[23:0] = (Fcarrier / Fxtal) × 65536
           = (914999512 / 27000000) × 65536
           = 0x21E38D
```
So `RF_FREQ2 = 0x21`, `RF_FREQ1 = 0xE3`, `RF_FREQ0 = 0x8D`.

**To change frequency:**
1. Open SmartRF Studio 7, select CC1110
2. Enter desired frequency and 27MHz crystal
3. Export register values
4. Update `RF_FREQ2`, `RF_FREQ1`, `RF_FREQ0` in `board.h`
5. Update `RF_FSCTRL1` if moving to a different frequency band

### 2.4 TX Power Configuration [SCK-DEV: TX_POWER]

Power mode is set at flash time — not at runtime.

| `RF_PA_CONFIG` | CC1110 Output | Total w/CC1190 | Firmware Build |
|---------------|--------------|----------------|---------------|
| `0xC0` | ~0 dBm | ~18 dBm | Bench / indoor (default) |
| `0xC2` | ~10 dBm | ~28 dBm | Field / HAB / LEO |

The ground station patches `RF_PA_CONFIG` automatically when flashing
bench or field firmware.

> **Critical:** There must be exactly **one** `#define RF_PA_CONFIG` line
> in `board.h`. The ground station patching tool searches for this exact
> string. Adding duplicates or moving this line will break the patching
> tool silently.

### 2.5 Bootloader Timeout [SCK-DEV: BOOTLOADER]

`COMMAND_WATCHDOG_DELAY = 1350000` ≈ 600ms at 27MHz.

The OpenLST default (45000 ≈ 20ms) is too short for RF OTA flashing.
The 600ms window gives the ground station time to detect the bootloader
beacon and transmit a flash command before the bootloader times out and
jumps to the application. Do not reduce below 1,000,000 for OTA-capable
deployments.

After a direct serial flash, warm up the board with 2–3 `get_telemetry`
commands before attempting OTA operations. This allows the RF synthesizer
to stabilize fully.

---

## 3.0 Firmware Extension Reference

### 3.1 Adding New Commands [SCK-DEV: ADD_COMMAND]

Adding a new command requires changes in three places. The CC1110 is a
transparent relay for all `PICO_MSG` (0x20) commands — it forwards any
payload to the Pico regardless of sub-opcode, so `board.c` typically
requires no changes.

#### Step 1 — Assign a sub-opcode in `main.py`

```python
# Add to the sub-opcode constants at the top of main.py
CMD_MY_COMMAND = 0x0A   # Next available opcode after 0x09
```

#### Step 2 — Add handler in `main.py` command dispatch loop

```python
elif sub_opcode == CMD_MY_COMMAND:
    try:
        # payload[0] = sub_opcode
        # payload[1:] = command-specific data
        param = payload[1:].decode('utf-8').strip()

        result = do_something(param)

        # Must fit in MAX_PAYLOAD (251 bytes)
        # For larger responses implement chunking — see Section 3.2
        msg = f"MYRESP:{result}"
        send_esp(msg.encode())
        blink(1)
        print(f"MY_COMMAND → {msg}")

    except Exception as e:
        send_esp(b"MYRESP:ERR:FAIL")
        print(f"MY_COMMAND error: {e}")
```

#### Step 3 — Add to C# ground station

```csharp
// In CommandOpcode enum
MyCommand = 0x0A,

// Command method
private async Task MyCommandAsync() {
    if (!CheckConnected()) return;
    ushort hwid = ActiveHwid;
    ushort seq  = IncSeqNum();
    FlushRxQueue();
    WritePacket(OpenLstProtocol.BuildPacket(
        hwid, seq, 0x20, new byte[] { 0x0A }));
    var pkt = await WaitForReply(hwid, seq, 10000);
    if (pkt?.PicoPayload != null)
        Log($"Response: {pkt.PicoPayload}", Theme.Green);
    else
        Log("No response", Theme.Red);
}
```

#### Step 4 — board.c changes (only if needed)

Only modify `board.c` if your command needs a timeout longer than the
standard 800ms (e.g. commands involving SD card writes):

```c
// Add timeout constant
#define PICO_TIMEOUT_OUTER_MY_SLOW_CMD  300  // ~3 seconds

// Add sub-opcode check in custom_commands()
outer_max = (payload_len > 0 && cmd->data[0] == PICO_SUB_MY_SLOW_CMD)
            ? PICO_TIMEOUT_OUTER_MY_SLOW_CMD
            : PICO_TIMEOUT_OUTER;
```

#### Critical rules for board.c

These rules come from hard experience. Violating any of them produces
difficult-to-trace failures:

| Rule | Consequence if violated |
|------|------------------------|
| Never set `reply->header.hwid`, `seqnum`, or `system` | Memory corruption — RF stops working |
| Only set `reply->header.command` | It is the only field `custom_commands` owns |
| Never use `WATCHDOG_CLEAR` macro | Corrupts RF receive via register conflict |
| Never declare `static __xdata` buffers | XDATA RAM overflow — board crashes |
| Use `uint8_t`/`uint16_t` only for counters | `uint32_t` causes RAM overflow crash |
| Use nested loops for timeouts >65535 | `uint16_t` wraps silently — timeout too short |
| Always send ≥1 byte payload | Zero-length ESP frames are silently rejected |

---

### 3.2 Chunking Large Responses [SCK-DEV: CHUNKING]

The maximum RF packet payload is **251 bytes** (`MAX_PAYLOAD`). Responses
larger than this must be split into chunks.

#### Pattern A — Text List Chunking (used by CMD_LIST)

```
"LIST:file1.jpg,file2.jpg,"     ← first chunk — trailing comma
"LIST+:file3.jpg,file4.jpg,"    ← continuation — trailing comma
"LIST+:file5.jpg"               ← final chunk — NO trailing comma = end signal
```

The ground station accumulates chunks until a packet with no trailing
comma arrives, then reassembles.

**Implementation template:**
```python
CHUNK = MAX_PAYLOAD - 8   # Reserve space for prefix
chunk_items = []
chunk_len   = 0
first_chunk = True

for item in items:
    entry = item + ","
    if chunk_len + len(entry) > CHUNK and chunk_items:
        prefix = "MYLIST:" if first_chunk else "MYLIST+:"
        send_esp((prefix + ",".join(chunk_items) + ",").encode())
        first_chunk = False
        chunk_items = [item]
        chunk_len   = len(item) + 1
    else:
        chunk_items.append(item)
        chunk_len += len(entry)

# Final chunk — no trailing comma signals end of list
if chunk_items:
    prefix = "MYLIST:" if first_chunk else "MYLIST+:"
    send_esp((prefix + ",".join(chunk_items)).encode())
```

#### Pattern B — Binary File Chunking (used by CMD_GET_INFO + CMD_GET_CHUNK)

Used for transferring binary files such as JPEG images and `.sckflight` logs.

**Protocol sequence:**
1. Ground station sends `CMD_GET_INFO` with filename
2. Pico responds `"INFO:<filename>:<total_bytes>:<chunk_count>"`
3. Ground station sends `CMD_GET_CHUNK` for index 0, 1 … chunk_count-1
4. Pico responds `"CHUNK:<index>:<200 bytes binary>"` for each
5. Ground station reassembles in order

**Chunk size:** 200 bytes (`CHUNK_SIZE`) — leaves 51 bytes for the
`"CHUNK:65535:"` prefix within the 251-byte limit.

```python
# GET_INFO handler
size   = uos.stat(f"/sd/{filename}")[6]
chunks = (size + CHUNK_SIZE - 1) // CHUNK_SIZE
send_esp(f"INFO:{filename}:{size}:{chunks}".encode())

# GET_CHUNK handler
chunk_index = payload[1] | (payload[2] << 8)   # little-endian uint16
filename    = payload[3:].decode('utf-8').strip()
with open(f"/sd/{filename}", 'rb') as f:
    f.seek(chunk_index * CHUNK_SIZE)
    data = f.read(CHUNK_SIZE)
send_esp(f"CHUNK:{chunk_index}:".encode() + data)
```

---

### 3.3 Timing Issues — Prototype Board [SCK-DEV: TIMING]

> **This section applies to the hand-wired SCK-915 prototype board only.**
> The SCK-PBL-1 production PCB resolves all of these issues through proper
> PCB routing and direct trace connections.

#### Issue 1 — SPI Bus Speed

Jumper wire capacitance and crosstalk limit the SPI bus to 400kHz on the
prototype. The production target on SCK-PBL-1 is 1.32MHz (the `sdcard.py`
design speed).

**Symptoms:** SD card mount fails, camera produces corrupt images,
`SD mount failed:` in serial output.

**SCK-PBL-1 fix:** Change in `main.py`:
```python
# Prototype (400kHz — safe for jumper wires)
spi = SPI(0, baudrate=400000, ...)

# SCK-PBL-1 production (1.32MHz — sdcard.py design speed)
spi = SPI(0, baudrate=1320000, ...)

# Also update slow init in mount_sd():
# Prototype: baudrate=100000
# SCK-PBL-1: baudrate=400000
```
Verify SD card reliability at 1.32MHz before deploying. Some cards support
up to 4MHz over short PCB traces for faster file transfers.

#### Issue 2 — UART Response Timing

Jumper wire noise can delay Pico UART responses enough to trigger the
CC1110 timeout, causing intermittent NACKs even though the Pico completed
the command correctly.

**Symptoms:** Commands NACK intermittently. Pico serial output shows
command completed. Same command sometimes succeeds, sometimes fails.

**Fix:** Increase `PICO_TIMEOUT_OUTER` in `board.c`:
```c
#define PICO_TIMEOUT_OUTER  40    // ~800ms standard
#define PICO_TIMEOUT_OUTER  80    // ~1600ms for prototype
```

#### Issue 3 — SPI Bus Sharing (Camera + SD)

Camera and SD card share SPI0. Both CS pins must be HIGH before switching
between devices. Missing a CS deselect causes data corruption on both devices.

```python
# Always do this before switching between camera and SD
cam_cs(1); sd_cs(1)
time.sleep_ms(10)
# Then select the device you need
```

Do not remove the explicit `cs(1)` calls — they are required on both
prototype and production boards.

#### Issue 4 — Camera Initialization Blocking UART

Camera initialization over I2C takes several hundred milliseconds.
Initializing at boot blocks UART0 and can cause the CC1110 to miss the
first command after power-on.

**Fix (already implemented):** Camera init is deferred to the first idle
timeout in the main loop, not at boot. Do not move `init_camera()` back
to the startup sequence.

#### Issue 5 — Arducam Signal Integrity

The Arducam OV2640 requires direct wire connections with minimal capacitance.
Breadboard connections add enough capacitance to cause intermittent SPI
read failures on the FIFO readout.

**Fix:** Use direct wired connections or the SCK-PBL-1 PCB traces.
Do not use breadboard for the camera SPI connections.

---

### 3.4 ESP Framing Protocol [SCK-DEV: ESP_FRAMING]

All UART0 communication between CC1110 and Pico uses ESP framing:

```
[0x22][0x69][length][payload bytes...]
  ↑      ↑      ↑         ↑
Start  Start  1 byte   1-251 bytes
byte0  byte1  payload  command data
              length
```

**OpenLST packet structure inside the payload:**

| Byte | Field | Notes |
|------|-------|-------|
| 0-1 | HWID | Little-endian destination hardware ID |
| 2-3 | Sequence | Little-endian seq number (16–64000) |
| 4 | System | 0x01 = MSG_TYPE_RADIO_IN (required) |
| 5 | Opcode | Command opcode byte |
| 6+ | Data | Command-specific payload |

> **Critical:** Length must be ≥ 1. Zero-length ESP frames are silently
> rejected by the RX interrupt service routine.

**MicroPython implementation:**

```python
ESP_START_0 = 0x22
ESP_START_1 = 0x69
MAX_PAYLOAD = 251

def send_esp(payload_bytes):
    uart.write(bytes([ESP_START_0, ESP_START_1,
                      len(payload_bytes)]) + payload_bytes)

def recv_esp(timeout_ms=5000):
    S0, S1, SL, SP = 0, 1, 2, 3
    state, payload, remaining = S0, bytearray(), 0
    deadline = time.ticks_add(time.ticks_ms(), timeout_ms)
    while time.ticks_diff(deadline, time.ticks_ms()) > 0:
        if uart.any():
            b = uart.read(1)[0]
            if state == S0:
                if b == ESP_START_0: state = S1
            elif state == S1:
                if b == ESP_START_1:   state = SL
                elif b == ESP_START_0: state = S1
                else:                  state = S0
            elif state == SL:
                if 1 <= b <= 251:
                    remaining, payload, state = b, bytearray(), SP
                else: state = S0
            elif state == SP:
                payload.append(b); remaining -= 1
                if remaining == 0: return bytes(payload)
        else:
            time.sleep_ms(1)
    return None
```

---

### 3.5 Response String Format Conventions [SCK-DEV: RESPONSE_FORMAT]

All Pico responses follow this format:
```
PREFIX:field1,field2,...
```

**Conventions:**
- Prefix uniquely identifies the response type (`PICO`, `SNAP`, `GPS`, etc.)
- Fields are comma-separated
- Error responses use `PREFIX:ERR:REASON` format
- No spaces anywhere in the string
- Ground station splits on `:` and `,`
- All responses fit within 251 bytes unless explicitly chunked

**Complete response table:**

| Command | Opcode | Success Response | Error Response |
|---------|--------|-----------------|---------------|
| PING | 0x00 | `PICO:ACK` | — |
| READ_TEMP | 0x01 | `TEMP:23.45C` | — |
| SNAP | 0x02 | `SNAP:OK:snap_001.jpg:24576` | `SNAP:ERR:NO_SD` / `SNAP:ERR:NO_CAM` |
| LIST | 0x03 | `LIST:file1.jpg,file2.jpg` | `LIST:ERR:NO_SD` / `LIST:EMPTY` |
| GET_INFO | 0x04 | `INFO:snap_001.jpg:24576:123` | `INFO:ERR:NOTFOUND` |
| GET_CHUNK | 0x05 | `CHUNK:0:<200 bytes>` | `CHUNK:ERR:FAIL` |
| DELETE | 0x06 | `DEL:OK:snap_001.jpg` | `DEL:ERR:FAIL` |
| GET_GPS | 0x07 | `GPS:35.12,-97.56,1523,8,1,1013.25,1521.3,22.5` | — |
| GET_BARO | 0x08 | `BARO:1013.25,1521.3,22.5` | `BARO:ERR:NOT_READY` |
| BEACON_CTRL | 0x09 | `BEACON:ON` / `BEACON:OFF` | — |

---

### 3.6 Autonomous GPS Beacon [SCK-DEV: BEACON]

The Pico transmits a fused GPS+barometric packet autonomously every 10
seconds when no command is being processed and the beacon is enabled.

**Beacon interval:** `GPS_BEACON_INTERVAL_MS = 10000` (10 seconds)

**Beacon format:**
```
GPS:lat,lon,gps_alt,sats,fix,hpa,baro_alt,temp_c
```

**Controlling the beacon:**
```
CMD_BEACON_CTRL (0x09) + 0x01 → enable
CMD_BEACON_CTRL (0x09) + 0x00 → disable
```

Disable during high-rate file transfers to prevent beacon packets from
interleaving with file chunk responses. Re-enable after the transfer.

Every beacon packet is also written to the SD flight log — see Section 3.7.

---

### 3.7 SD Card Flight Log Format [SCK-DEV: SD_FLIGHT_LOG]

One `.sckflight` file is created per power cycle.
Naming: `FLT-001.sckflight`, `FLT-002.sckflight`, etc.

**File format (JSON):**
```json
{"flight_id":"FLT-001","hardware":"SCK-915+SCK-PBL-1","packets":[
{"t":0.0,"lat":0.0,"lon":0.0,"gps_alt":0.0,"sats":0,"fix":0,
 "baro_hpa":1013.25,"baro_alt":215.3,"baro_temp":22.5,"event":""},
{"t":10.1,"lat":35.123456,"lon":-97.654321,"gps_alt":1523.4,
 "sats":8,"fix":1,"baro_hpa":849.12,"baro_alt":1521.3,
 "baro_temp":18.2,"event":"SNAP:snap_001.jpg"}
],
"summary":{"total_packets":42,"flight_duration_s":420.0}}
```

**Adding new fields:**

In `write_flight_packet()` in `main.py`:
```python
line = (f'{sep}{{"t":{t},'
        f'"lat":{gps_fix["lat"]:.6f},'
        # existing fields...
        f'"my_new_field":{my_value},'   # ← add here
        f'"event":"{event}"}}\n')
```
Also update the ground station `FlightReplayForm` parser.

---

## 4.0 Payload Board Hardware Reference

### 4.1 SCK-PBL-1 GPIO Assignments

| GPIO | Function | Notes |
|------|----------|-------|
| 0 | UART0 TX → CC1110 | ESP framing to RF board |
| 1 | UART0 RX ← CC1110 | ESP framing from RF board |
| 2 | I2C1 SDA (MS5611) | 400kHz — same pins as previous BMP581 |
| 3 | I2C1 SCL (MS5611) | 400kHz |
| 4 | I2C0 SDA (Camera) | 100kHz |
| 5 | I2C0 SCL (Camera) | 100kHz |
| 8 | UART1 TX → GPS | 9600 baud — UBX config only |
| 9 | UART1 RX ← GPS | 9600 baud — NMEA stream in |
| 10 | LED Camera (GREEN) | Solid = camera initialized OK |
| 11 | LED SD Card (GREEN) | Solid = SD mounted OK |
| 12 | LED GPS (BLUE) | Slow flash = searching / Solid = fix |
| 13 | LED Altimeter (YELLOW) | Solid = MS5611 initialized OK |
| 14 | LED Fault (RED) | Solid = any subsystem failed |
| 15 | Camera CS | Active LOW -- deselect before SD access |
| 16 | SPI0 MISO | SD card MISO (DAT0) |
| 17 | SD Card CS | Active LOW -- deselect before camera access |
| 18 | SPI0 SCLK | Shared camera + SD |
| 19 | SPI0 MOSI | Shared camera + SD |
| 20-24 | Free GPIO | Available for expansion payloads |
| 25 | Onboard LED | Boot confirmation blinks |
| 26-28 | Free GPIO | Available for expansion payloads |

### 4.2 SD Card Wiring Reference

Verified wiring for a standard full size SD card adapter used with a micro SD card.
View orientation: bottom of card, notch (corner cut) on the left, pins 1-9 left to right.
Pin 9 is offset and smaller -- do not connect.

```
Notch
  |
[1][2][3][4][5][6][7][8]  [9]
                           ^
                      offset -- NC
```

| SD Pin | SD Function | SPI Function | Connect To |
|--------|------------|-------------|-----------|
| 1 | DAT2 | Unused | 10k pullup to 3.3V only -- no GPIO |
| 2 | DAT3/CD | CS | GPIO17 |
| 3 | CMD | MOSI | GPIO19 |
| 4 | VDD | Power | 3.3V (Pico pin 36) |
| 5 | CLK | SCK | GPIO18 |
| 6 | VSS1 | Ground | GND |
| 7 | DAT0 | MISO | GPIO16 |
| 8 | DAT1 | Unused | 10k pullup to 3.3V only -- no GPIO |
| 9 | (offset) | NC | Do not connect |

**Critical hardware note:** Place a 0.1uF ceramic capacitor directly across
the 3.3V and GND pins at the SD card connector. Without this decoupling cap
the card will fail to initialize intermittently due to SPI bus noise on the
power supply line. Place it as close to the connector as possible.

**SPI speed:** Verified working at 100kHz with direct soldered leads.
On a production PCB with short traces raise to 1.32MHz (sdcard.py design speed).

### 4.2 Expansion Headers — 2×20 Pin Breakout

The SCK-PBL-1 payload board includes two 2×20 pin headers that break out
all Raspberry Pi Pico GPIO pins — including pins already in use by the
on-board subsystems. This allows researchers and developers to attach
additional payload hardware without requiring a new PCB revision for each
mission variant.

**J_EXP1 and J_EXP2** — full Pico pin breakout:
- All 26 GPIO pins accessible (including in-use pins)
- 3.3V, 5V (VBUS), and GND power rails available
- Do not conflict with in-use pins (GPIO 0–19, 25) unless you understand
  the bus sharing implications

**J_DEBUG** — dedicated debug header:
- UART0 TX/RX (GPIO 0/1) — monitor CC1110 ↔ Pico communication
- UART1 TX/RX (GPIO 8/9) — monitor raw GPS NMEA stream
- GND reference

**Free GPIOs for expansion:** 20–24 and 26–28 (seven free GPIO pins)

**When attaching expansion hardware:**
- SPI0 can be shared — add a new CS GPIO and manage it explicitly
- I2C buses support multiple devices — ensure unique I2C addresses
- Do not drive in-use pins as outputs without disconnecting the on-board
  subsystem first

---

## 5.0 C# Ground Station Extension Reference

### 5.1 Key Methods

| Method | Description |
|--------|-------------|
| `ActiveHwid` | Property: current HWID from header bar |
| `IncSeqNum()` | Increment and return next sequence number |
| `FlushRxQueue()` | Discard queued received packets before sending |
| `WritePacket(byte[])` | Send raw bytes to serial port |
| `WaitForReply(hwid,seq,ms)` | Async: wait for matching ack/response |
| `Log(message, color)` | Append line to main log panel |
| `LogTx(message, hwid)` | Log transmitted packet in yellow TX format |
| `CheckConnected()` | Returns false and logs error if not connected |
| `MkButton(text,x,y,w,fg,bg)` | Create styled flat button |
| `MkLabel(text,x,y,w,color,font)` | Create styled label |
| `MkSectionLabel(text,x,y)` | Create cyan section divider |
| `MkTab(text)` | Create styled tab page |

### 5.2 Adding a Custom Command (C# Template)

```csharp
private async Task MyCommandAsync() {
    if (!CheckConnected()) return;
    ushort hwid = ActiveHwid;
    ushort seq  = IncSeqNum();
    FlushRxQueue();
    WritePacket(OpenLstProtocol.BuildPacket(
        hwid, seq, 0x20, new byte[] { 0x0A }));  // sub-opcode 0x0A
    var pkt = await WaitForReply(hwid, seq, 10000);
    if (pkt?.PicoPayload != null)
        Log($"Response: {pkt.PicoPayload}", Theme.Green);
    else
        Log("No response", Theme.Red);
}
```

### 5.3 Opcode Reference

| Opcode | Name | Description |
|--------|------|-------------|
| 0x10 | `common_msg_ack` | Acknowledge response |
| 0xFF | `common_msg_nack` | Not-acknowledge response |
| 0x11 | `common_msg_ascii` | ASCII debug text |
| 0x12 | `radio_msg_reboot` | Reboot (optional 4-byte postpone_sec) |
| 0x13 | `radio_msg_get_time` | Request RTC time |
| 0x14 | `radio_msg_set_time` | Set RTC time |
| 0x17 | `radio_msg_get_telem` | Request telemetry |
| 0x18 | `radio_msg_telem` | Telemetry response (83 bytes) |
| 0x19 | `radio_msg_get_callsign` | Request callsign |
| 0x1A | `radio_msg_set_callsign` | Set callsign (8 bytes) |
| 0x20 | `PICO_MSG` | Forward payload to Pico via UART0 |

---

## 6.0 Build and Flash Reference

### 6.1 Building CC1110 Firmware

```bash
# Bench firmware (safe indoor power — default)
mingw32-make openlst_437_radio RF_PA_CONFIG=0xC0

# Field firmware (full mission power)
mingw32-make openlst_437_radio RF_PA_CONFIG=0xC2
```

Or use the ground station "Flash Firmware" button which handles build
selection and patching automatically.

> **Always Clean + Build** after any source change. Never trust an
> incremental build after modifying `board.h` or `board.c`.

### 6.2 Flashing Pico Firmware

```bash
# Using mpremote
mpremote cp main.py :main.py
mpremote cp sdcard.py :sdcard.py
```

Or hold BOOTSEL on power-up and drag-and-drop files to the USB mass
storage drive.

### 6.3 OTA Firmware Update (CC1110)

1. Power cycle the SCK-915 board
2. Ground station detects bootloader beacon within the 600ms window
3. Click "Flash Firmware" — select bench or field build
4. Board reflashes and restarts with new firmware
5. Send 2–3 `get_telemetry` commands after OTA flash before other operations

The 600ms window is controlled by `COMMAND_WATCHDOG_DELAY` in `board.h`.

---

## 7.0 Hard-Won Lessons

These lessons were accumulated during development of the SCK-915 system
starting from the OpenLST Explorer Kit prototype. Each one cost real
debugging time. Read them before starting any firmware work.

| # | Lesson | Impact |
|---|--------|--------|
| L1 | UART1 is the ground station interface — use UART0 for Pico | Critical |
| L2 | Custom commands must be in `board.c` — not a separate file | Critical |
| L3 | Never reinitialize `reply->header` in `custom_commands` | Critical |
| L4 | `WATCHDOG_CLEAR` corrupts RF receive — never use it | High |
| L5 | `uint32_t` causes XDATA RAM crash — use `uint16_t` only | High |
| L6 | `uint16_t` wraps at 65535 — use nested loops for long timeouts | High |
| L7 | Zero-length ESP payload silently rejected by RX ISR | Medium |
| L8 | SD mount corrupts SPI state for Arducam — reinit SPI after mount | High |
| L9 | Arducam needs direct wire connection — breadboard adds too much capacitance | Medium |
| L10 | Camera init blocks UART — defer to first idle timeout, never at boot | Medium |
| L11 | Ground unit must run original unmodified OpenLST firmware | High |
| L12 | After direct serial flash: send 2–3 `get_telemetry` before RF ops | Medium |
| L13 | Always Clean + Build — stale `.rel` files cause silent firmware bugs | High |
| L14 | SPI bus shared between camera and SD — always deselect both CS before switching | High |
| L15 | Prototype board SPI limited to 400kHz — jumper wire noise corrupts at higher speeds | Medium |
| L16 | PICO_TIMEOUT values tuned for production PCB — double them on prototype board | Medium |
| L17 | `RF_PA_CONFIG` must appear exactly once in `board.h` — ground station patching tool requires it | High |
| L18 | Bootloader window is 600ms — ground station must detect beacon and respond within this window | High |

---

*SpaceCommsKit Developer Guide v2.0*
*For updates and latest version see https://spacecommskit.com/docs*
