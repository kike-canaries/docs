---
title: CanAirIO Noise Monitor
tags:
  - device
  - firmware
  - hardware
  - noise
  - i2c
keywords: device, fixed stations, sensors, hardware, noise, ics-43434, esp32, i2c, calibration
last_updated: "October 6, 2026"
summary: "CanAirIO Noise Monitor node — environmental noise measurement according to UNE-EN ISO 1996-2:2009"
series: "hardware"
sidebar: english_sidebar
permalink: canairio_noise.html
folder: en
---

## Overview

The **CanAirIO Noise Monitor** is an I2C slave node that measures environmental noise levels according to **UNE-EN ISO 1996-2:2009** and **Decree 213/2012** (Basque Country). It connects to any CanAirIO master device via I2C and provides A-weighted noise indicators including LAeq, LAFmax, LASmax, LCpeak, L10/L90, and Lden.

The node is built around the **XIAO ESP32-S3** microcontroller and the **ICS-43434** digital MEMS microphone, which offers a flat frequency response (±1 dB), high SNR (65 dBA), and a low noise floor (~34 dB SPL) — enabling accurate measurement of quiet urban nights that analog microphones cannot capture.

Two interchangeable sensor variants share the same processing chain and I2C protocol:

| Variant | Microphone | Board | Sample Rate | Noise Floor |
|---------|-----------|-------|-------------|-------------|
| **Digital (recommended)** | ICS-43434 (MEMS I2S) | XIAO ESP32-S3 | 48 kHz | ~34 dB |
| Analog | MAX4466 (electret + ADC) | ESP32-C3 | 16 kHz | ~58-60 dB |

This guide covers the **digital variant** with the ICS-43434.

## Components

| Part Ref | Quantity | Notes |
|----------|----------|-------|
| XIAO ESP32-S3 | 1 | Microcontroller with I2S and I2C |
| ICS-43434 MEMS microphone module | 1 | Digital I2S output, 24-bit |
| CanAirIO master device | 1 | Any CanAirIO board (TTGO T-Display, M5Stack, etc.) |
| Acoustic calibrator 94 dB / 1 kHz | 1 | Class 1 or 2 (IEC 60942) — for calibration |
| Jumper wires | 4 | SDA, SCL, GND, power |

### ICS-43434 pinout diagram

![ICS-43434 module](images/ics43434_mrs179a.png)

### XIAO ESP32-S3 pinout diagram

![XIAO ESP32-S3](images/xiao_esp32s3_pinout.jpg)

## Schematics

The noise node connects to the CanAirIO master via a 4-wire I2C bus:

```
CanAirIO Master          Noise Node (XIAO ESP32-S3)
┌─────────────┐          ┌──────────────────────┐
│             │          │                      │
│  SDA ───────┼──────────┼── SDA (GPIO 5 / D4)  │
│  SCL ───────┼──────────┼── SCL (GPIO 6 / D5)  │
│  GND ───────┼──────────┼── GND                │
│  3.3V ──────┼──────────┼── 3V                 │
│             │          │                      │
└─────────────┘          │  ICS-43434:          │
                         │   BCLK → GPIO 2 (D1) │
                         │   WS   → GPIO 3 (D2) │
                         │   SD   → GPIO 4 (D3) │
                         │   SEL  → GND         │
                         │   VDD  → 3.3V        │
                         │   GND  → GND         │
                         └──────────────────────┘
```

**Important:** A common ground between both boards is mandatory. The ICS-43434 must be powered at 3.3V — never 5V.

## Basic connection

### Step 1: Wire the ICS-43434 microphone

Connect the ICS-43434 module to the XIAO ESP32-S3:

| ICS-43434 pin | XIAO ESP32-S3 pin | Function |
|---------------|-------------------|----------|
| SEL | **GND** | Channel select (L/R pin — must be GND for left channel) |
| LRCL | GPIO 3 (D2) | Word select (WS / LRCLK) |
| DOUT | GPIO 4 (D3) | Data output |
| BCLK | GPIO 2 (D1) | Bit clock |
| GND | GND | Ground |
| 3V | 3.3V | Power supply (1.5–3.6V, **never 5V**) |

**SEL must be tied to GND.** This is the `L/R` channel pin, not an I2S/PDM selector. With `I2S_CHANNEL_FMT_ONLY_LEFT` in the firmware, SEL to 3.3V leaves the node silent.

Keep I2S wires short (<10 cm): at 48 kHz, BCLK runs at 3.072 MHz. In a homemade cable, interleave the GND wire between BCLK and DOUT.

### Step 2: Connect the node to the CanAirIO master

Connect the XIAO ESP32-S3 to the CanAirIO master via I2C:

| Signal | CanAirIO master | Noise node |
|--------|-----------------|------------|
| SDA | SDA pin | GPIO 5 (D4) |
| SCL | SCL pin | GPIO 6 (D5) |
| GND | GND | GND |
| 3.3V | 3.3V output | 3V pin |

**Common ground is mandatory** even if each board has its own power supply. Most ESP32 boards already have I2C pull-ups; only add 4.7 kΩ pull-ups to 3.3V if the bus hangs.

For cables longer than 20-30 cm, lower the I2C clock to 100 kHz.

### Step 3: Power the system

Power the CanAirIO master via USB-C. The noise node can be powered from the master's 3.3V output or from a separate 3.3V supply (with common ground).

## Firmware upload

### Upload the noise node firmware

The noise node firmware is in the `noise_UNE-EN_ISO_1996-2-2009` repository. To flash it:

1. Open the repository in PlatformIO
2. Select the `seeed_xiao_esp32s3` environment
3. Build and upload

The firmware is pre-configured with:
- `SAMPLE_RATE=48000`
- `MIC_OFFSET_DB=0.0` (nominal — per-unit calibration is stored in NVS)

### Upload the CanAirIO master firmware

The CanAirIO master firmware can be installed via the [CanAirIO Web Installer](https://canair.io/installer.html) or following the [firmware upload guide](https://canair.io/docs/firmware_upload.html).

The master automatically detects I2C slave devices at address `0x08` and reads noise indicators every second.

## Calibration

The noise node requires a one-time calibration with an acoustic calibrator to establish the relationship between the microphone's digital output and the actual sound pressure level in dB SPL.

### Automatic calibration (recommended)

The repository includes an **automatic calibration** example (`examples/calibration_i2s_auto/`) that performs the full calibration process without user intervention:

1. **Resets** the NVM offset to 0
2. **Measures** 5 seconds of the calibrator tone
3. **Validates** the microphone (sensitivity, crest factor, stability)
4. **Calculates** the optimal offset
5. **Saves** the offset to NVM
6. **Verifies** the calibration

#### Requirements

- XIAO ESP32-S3 with ICS-43434 connected
- Acoustic calibrator: **94.0 dB / 1 kHz** (Class 1 or 2, IEC 60942)

#### Steps

1. Open `examples/calibration_i2s_auto/` in PlatformIO
2. Compile and flash the firmware
3. **Couple the calibrator to the microphone** (before opening the Serial Monitor)
4. Open the Serial Monitor at 115200 baud
5. The sketch performs everything automatically

#### Expected result

If the microphone is valid:

```
========================================
  CALIBRATION COMPLETED SUCCESSFULLY
========================================

  Offset saved to NVM: -13.71 dB
  Unit sensitivity: -26.15 dBFS @ 94 dB SPL

The main firmware will now use this offset
automatically on boot.
```

If the microphone is invalid or defective:

```
========================================
  RESULT: MICROPHONE NOT VALID
========================================

The microphone does not meet the validity criteria.
Possible causes:
  - Counterfeit or defective microphone
  - Calibrator not properly coupled
  - Excessive background noise
  - I2S cables improperly connected

No offset has been saved to NVM.
```

### Manual calibration (alternative)

For manual calibration, use `examples/calibration_i2s_table/` which prints CSV data and a summary every 10 seconds with the exact offset to save.

1. Flash `examples/calibration_i2s_table/`
2. Couple the 94 dB calibrator
3. Wait for an **ESTABLE** summary
4. The offset is **automatically saved** to NVS and verified

### Calibration criteria

The automatic calibration validates the microphone against these criteria:

| Criterion | Valid range | Purpose |
|-----------|-------------|---------|
| Sensitivity | −26 ± 3 dBFS @ 94 dB SPL | Detects counterfeit or wrong parts |
| Crest factor | 2.8 – 3.2 dB | Clean sine tone (no distortion) |
| Clips | 0 | No signal clipping |
| Stability | < 0.5 dB variation | Consistent readings |

### Recalibration

Recalibrate every **6 months** or whenever:
- The microphone module is replaced
- The enclosure or microphone position changes
- The MAX4466 gain trimmer is adjusted (analog variant)

## CanAirIO integration

Once calibrated, the noise node operates as a standard I2C slave. The CanAirIO master:

1. **Discovers** the node at address `0x08` on the I2C bus
2. **Reads** noise indicators every second via `CMD_GET_DATA`
3. **Publishes** the data to the CanAirIO cloud, MQTT, or InfluxDB/Grafana

### I2C protocol summary

| Command | Value | Description |
|---------|-------|-------------|
| `GET_STATUS` | `0x20` | 1 byte: 1 = valid data, 0 = invalid |
| `GET_DATA` | `0x01` | 88 bytes: `SensorData` struct with CRC-16 |
| `GET_METADATA` | `0x50` | 16 bytes: firmware version, node type, calibration offset |
| `SET_CALIB` | `0x0A` | + int16 LE: calibration offset in hundredths of dB |
| `SET_TIME` | `0x09` | + uint32 LE: local epoch (enables Ld/Le/Ln/Lden) |

### Data fields

The `SensorData` struct (88 bytes) contains:

| Field | Description |
|-------|-------------|
| `noiseAvgDb` | LAeq,1s — equivalent continuous level (dB(A)) |
| `noisePeakDb` | LAFmax — Fast (125 ms) maximum (dB(A)) |
| `noiseLASmaxDb` | LASmax — Slow (1 s) maximum (dB(A)) |
| `noiseLCpeakDb` | LCpeak — C-weighted peak (dB(C)) |
| `noiseAvgLegalDb` | L10 — level exceeded 10% of the time |
| `lowNoiseLevel` | L90 — level exceeded 90% of the time |
| `Ld` / `Le` / `Ln` | Day / evening / night energy averages |
| `noiseLden` | Day-evening-night index with penalties |

### Clock synchronization

The node has no real-time clock. The CanAirIO master must send the current time via `SET_TIME` to enable Ld/Le/Ln/Lden calculation. Send **local epoch** (not UTC) — the node applies no timezone.

Without a clock, Ld/Le/Ln/Lden remain at 0 (correct behavior, not a fault).

## Troubleshooting

| Symptom | Likely cause | Solution |
|---------|-------------|----------|
| Node not detected on I2C | Wiring, power, or common ground | Check SDA/SCL, power, and GND between boards |
| `Status 0` continuously | Microphone failure or clipping | Check ICS-43434 wiring, SEL to GND |
| Readings 13 dB too high | Counterfeit microphone | Run automatic calibration to validate |
| Ld/Le/Ln/Lden = 0 | Clock not sent | Send local epoch via `SET_TIME` |
| Erratic readings | Long I2C or I2S cables | Shorten cables, lower I2C clock to 100 kHz |
| Calibration fails | Wrong calibrator level or coupling | Use 94 dB position, check acoustic seal |

{% include links.html %}
