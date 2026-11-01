# CC1101Pro
New cc1101 library for esp32 board

High-performance driver for TI **CC1101** (sub-GHz, 300–348 / 387–464 / 779–928 MHz) for **ESP32 (all variants)** and **ESP8266**.
Written from scratch according to the SWRS061I datasheet (not a copy of external code, MIT license).

> **Important:** the code was verified only by syntactic translation against the Arduino stub API and mathematically (data rates match SmartRF values). **It has not been tested on real HW** – verify the first startup according to the "First Startup" chapter.

## 1. Why Not an Existing Library?

| Library | Pros | Cons |
|---|---|---|
| **LSatan/SmartRC-CC1101-Driver-Lib** (v3.0.2, May 2026) | Most widespread, works with rc-switch, multi-module, ESP32 core 3.x fix | Less protection against FIFO errata, no LBT, no scanner, historically chaotic API |
| **RadioLib (jgromes)** | Actively maintained, many chips, clean API | Universal = fewer CC1101 specifics (raw OOK, spectrum, PATABLE for band) |
| panStamp / SpaceTeddy / veonik | Simple | Older, unsupported (based on last update dates) |

Recommendation: for production multi-chip projects **RadioLib**, for quick toys with rc-switch **SmartRC**. CC1101Pro aims at what both do less effectively: a robust CC1101 on ESP + sub-GHz analysis.

## 2. What CC1101Pro Does Better

* **Precise calculations** of frequency, baud rate (0.6–500 kBd), deviation, and RX BW from the crystal (26 or 27 MHz) + `actual*()` getters show the actual value after rounding.
* **Auto-tuning**: deviation, RX BW (Carson + crystal budget), IF, TEST1/2 according to BW, PATABLE according to band (315/433/868/915) and OOK (index 0/1).
* **Errata-safe FIFO**: `RXBYTES/TXBYTES` are read until a stable value is reached, overflow/underflow is recovered automatically, reading only after the end of the packet.
* **ISR-driven** reception (IRAM_ATTR/ICACHE_RAM_ATTR), fallback to polling without ISR.
* **Listen-before-talk** via hardware CCA (MCSM1.CCA_MODE=3) – `transmit(..., true)` returns `CC_ERR_CHANNEL_BUSY`.
* **Spectrum scanner** (`scanSpectrum`), **carrier-sense** on GDO2, **AFC estimate** (`freqOffsetKHz`).
* **Raw OOK/FSK**: async capture to ring buffer with glitch filter + precise replay (`rawTransmit`).
* **Sleep/wake**, hardware addressing + broadcast, whitening, Manchester, CRC, promiscuous mode.
* **ESP32 core 3.x safe**: waiting for MISO via `gpio_get_level` (known issue `IO X is not set as GPIO`).
* **Read-back verification** after configuration (`CC_ERR_SPI_VERIFY` = bad wiring / too fast SPI).

## 3. Wiring

**Power the CC1101 ONLY from 3.3 V (max 3.6 V), I/Os are not 5 V tolerant.** Add 100 nF + 10 µF close to VCC.

Typical 8-pin module (green "E07/HC-style"): `GND, VCC, GDO0, CSN, SCK, MOSI, MISO(GDO1), GDO2`.

### ESP32 (classic, VSPI – default in examples)

| CC1101 | ESP32 GPIO | Note |
|---|---|---|
| VCC | 3V3 | |
| GND | GND | |
| SCK | 18 | |
| MOSI (SI) | 23 | |
| MISO (SO/GDO1) | 19 | |
| CSN | 5 | |
| GDO0 | 4 | **mandatory for RX**, interrupt |
| GDO2 | 22 | optional (carrier sense) |

Other ESP32 (S2/S3/C3…): `radio.beginSPI(sck, miso, mosi)` on any free GPIO. Avoid strapping pins (0, 2, 12, 15 on classic) and GPIO6–11 (flash), GPIO16/17 on WROVER (PSRAM).

### ESP8266 (NodeMCU / Wemos D1 mini)

| CC1101 | ESP8266 | GPIO |
|---|---|---|
| VCC / GND | 3V3 / GND | |
| SCK | D5 | 14 |
| MISO | D6 | 12 |
| MOSI | D7 | 13 |
| CSN | D8 | 15 |
| GDO0 | D1 | 5 |
| GDO2 | D2 | 4 |

GPIO15 must be LOW at boot – CC1101 does not hold it HIGH, so wiring is OK. GPIO16 has no interrupt, do not use as GDO0.

### Antenna
Quarter-wave wire (free space): **433.92 MHz ≈ 17.3 cm, 868 MHz ≈ 8.6 cm, 915 MHz ≈ 8.2 cm**, in practice ~3–5% shorter; tune according to SWR/`SpectrumScanner`. The module must be equipped with a filter/matching network for your band (433/868/915 are different variants!). Never transmit without an antenna.

## 4. Installation
Copy the `CC1101Pro` folder to `Arduino/libraries/` (or PlatformIO `lib_deps = file://...`). Requires ESP32 or ESP8266 Arduino core.

## 5. Quick Start
```cpp
#include <CC1101Pro.h>
CC1101Pro radio(5 /*CS*/, 4 /*GDO0*/, 22 /*GDO2*/);
void setup(){
  Serial.begin(115200);
  radio.beginSPI(18, 19, 23);             // ESP32; omit on ESP8266
  if (radio.begin(433.92f, 38.4f, CC1101Pro::MOD_GFSK, 10) != CC_OK) while(1);
  radio.enableInterrupt(); radio.startReceive();
}
void loop(){
  if (radio.packetReady()){ uint8_t b[61]; CC1101Pro::PacketInfo pi;
    int n = radio.readPacket(b, sizeof b, &pi); if (n>0) Serial.println(pi.rssi); }
}
```

## 6. API (Summary)

| Function | Description |
|---|---|
| `begin(Config)` / `begin(f, baud, mod, dBm)` | reset, chip detection, configuration; returns `CC_OK` or error |
| `setFrequency/Baud/Deviation/RxBandwidth/Modulation/Power/SyncWord/Address/Channel/Promiscuous` | runtime change, preserves RX state |
| `transmit(data,len,lbt)` / `transmitTo(dest,…)` | max 61 B |
| `startReceive()`, `packetReady()`, `readPacket()`, `receive(timeout)` | reception; `readPacket` returns length or `CC_ERR_*` (on `CC_ERR_CRC`, data is in buffer – useful for sniffing) |
| `getRSSI()`, `carrierSense()`, `freqOffsetKHz()`, `getLQI()` | diagnostics |
| `scanSpectrum(f0,f1,stepKHz,dwellUs,cb)` | RSSI sweep |
| `startRawCapture / rawReadFrame / rawTransmit / stopRawCapture` | async OOK/FSK pulses |
| `sleep()`, `wake()`, `idle()`, `calibrate()` | power management |
| `dumpRegisters()`, `printStatus()` | debugging |
| `strobe/readReg/writeReg/readBurst/writeBurst` | low-level access |

Return codes: `CC_OK=0`, `CC_ERR_CHIP_NOT_FOUND=-1`, `TIMEOUT=-2`, `CRC=-3`, `RX_OVERFLOW=-4`, `TX_UNDERFLOW=-5`, `TOO_LONG=-6`, `INVALID_PARAM=-7`, `NO_PACKET=-8`, `LENGTH=-9`, `CHANNEL_BUSY=-10`, `SPI_VERIFY=-11`.

### Performance Tuning Tips
* **Range vs. Speed**: 1.2–4.8 kBd + narrow RX BW (58–100 kHz) = best sensitivity; 250 kBd = worst. Narrow BW requires a precise crystal/AFC.
* `rxBwKHz = 0` selects BW automatically; increase it for cheap modules with crystal drift.
* On ESP32, leave SPI at 4 MHz (max 6.5 MHz during burst).
* `crcAutoflush = true` for production; `setPromiscuous(true)` for sniffing.
* Addressing: first payload byte = destination address (HW filter).

## 7. First Startup
1. Run `BasicTxRx`; if `init failed`: check 3.3 V, SCK/MOSI/MISO/CSN, common ground. `part` should be `0x00`, `ver` typically `0x14` (or `0x04`).
2. Two modules on the same band – after a few seconds, you will see `RX … RSSI`.
3. `SpectrumScanner` with antenna – when the second node transmits, there must be a peak at 433.92 MHz.
4. If the frequency is slightly off, verify `setCrystalMHz()` (some modules have 27 MHz).

## 8. Known Limitations (Honestly)
* Max payload **61 B** (single FIFO packet); long packets/infinite mode and FEC are not implemented.
* IRQ functions use a **single instance** (static pointer).
* `rawTransmit` is bit-banging on GDO0 with `micros()`; WiFi/BT interrupts can add dozens of µs of jitter on ESP – consider RMT (ESP32) for sensitive protocols.
* RSSI offset is a fixed 74 dB (realistically ±1–2 dB depending on speed/band).
* AGC/TEST registers are based on typical SmartRF values; verify them in SmartRF Studio for extreme performance in a specific mode.
* `wake()` rewrites the entire configuration including PATABLE just to be safe.
* Verified only by translation to stub and math, not on a real chip.

## 9. Legal and Ethical
Observe bands and power limits in your country (EU: 433.05–434.79 MHz, 868.0–868.6 MHz, 10 mW ERP, duty cycle 1%). Use raw capture/replay **only on your own devices or within an authorized penetration testing scope**. The library does not include jamming or bypassing rolling-code systems.
