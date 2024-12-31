# G1 AR Glasses Reverse Engineering Reference

**Last Updated:** December 30, 2024

## Table of Contents

1. [Introduction](#introduction)
2. [Firmware Information](#firmware-information)
   - [Firmware Versions](#firmware-versions)
   - [Firmware Extraction & Analysis](#firmware-extraction--analysis)
3. [Bluetooth Low Energy (BLE) Communication](#bluetooth-low-energy-ble-communication)
   - [ANCS (Apple Notification Center Service)](#ancs-apple-notification-center-service)
   - [G1 Command Structure](#g1-command-structure)
   - [Packet Structures](#packet-structures)
4. [Commands Overview](#commands-overview)
   - [General Commands](#general-commands)
   - [Dashboard Commands (0x06)](#dashboard-commands-0x06)
   - [Teleprompter (0x09) & Navigation (0x0a)](#teleprompter-0x09--navigation-0x0a)
   - [Translation Commands (0x0d-0x0e-0x0f)](#translation-commands-0x0d-0x0e-0x0f)
   - [Silent Mode (0x03)](#silent-mode-0x03)
   - [Other Notable Commands](#other-notable-commands)
5. [Notification Whitelisting](#notification-whitelisting)
6. [App Interaction & Whitelisting](#app-interaction--whitelisting)
7. [Image Handling](#image-handling)
   - [Bitmap (BMP) Handling](#bitmap-bmp-handling)
   - [Navigation Map Rendering](#navigation-map-rendering)
8. [Logging & Debugging](#logging--debugging)
   - [Enabling Debug Mode](#enabling-debug-mode)
   - [Log Prefixes and Sources](#log-prefixes-and-sources)
9. [Synchronization & Communication Issues](#synchronization--communication-issues)
   - [Master-Slave Behavior](#master-slave-behavior)
   - [Desynchronization Causes](#desynchronization-causes)
   - [Command Redundancy](#command-redundancy)
10. [Development Tools & Resources](#development-tools--resources)
    - [Firmware Analysis Tools](#firmware-analysis-tools)
    - [Libraries & Repositories](#libraries--repositories)
    - [Community Resources](#community-resources)
11. [Miscellaneous Notes](#miscellaneous-notes)
12. [Future Work & Considerations](#future-work--considerations)
13. [Conclusion](#conclusion)
14. [How to Use](#how-to-use)

---

## Introduction

This repository contains reverse engineering insights, command structures, and notes on the G1 AR glasses. The goal is to provide a consolidated reference so others can more easily explore and extend the functionality of these devices.

---

## Firmware Information

### Firmware Versions

- **1.3.7** – An earlier firmware version referenced during initial reverse engineering.  
- **1.4.1** – Current firmware version at the time of the latest updates.  
- **1.4.5** – Newer firmware version also noted and available.

### Firmware Extraction & Analysis

- **Extraction Methods**  
  - **APK Decompilation:** Some firmware builds were embedded in the official APK.  
  - **Filesystem Access:** On a rooted phone, check `/data/data/com.even.g1/files`.

- **Analysis Tools**  
  - **IDA Pro** – Disassembling and analyzing firmware binaries.  
  - **Ghidra** – Recommended decompiler with aarch64 support.  
  - **Ghidra + nRF5340 DK** – Further analysis with specific hardware headers.

- **Observations**  
  - Firmware is rich in debug strings (assisting in identifying modules).  
  - Flutter-based app complicates static analysis (layers beyond Java bytecode).  
  - RLE (Run-Length Encoding) is used for certain image data (e.g., navigation).

---

## Bluetooth Low Energy (BLE) Communication

### ANCS (Apple Notification Center Service)

- **Purpose:** Enables standard BLE devices to receive iOS notifications.  
- **Implementation:** Appears to be entirely in the G1 firmware.  
- **Reference:** [Apple ANCS Spec](https://developer.apple.com/library/archive/documentation/CoreBluetooth/Reference/AppleNotificationCenterServiceSpecification/Specification/Specification.html)

### G1 Command Structure

Each G1 command generally looks like:
[Command ID][Packet Length][Sequence ID][Subcommand][Data...]

makefile
Copy code
**Example**:
0x06 0x15 0x00 [seq_id] 0x01 [timestamp][weather data]...

shell
Copy code

### Packet Structures

#### Dashboard Update Packet (Command 0x06)


    0x06,                # Command ID: Dashboard
    0x15, 0x00,          # Packet length (0x0015 = 21 bytes)
    seq_id,              # Sequence number
    0x01,                # Subcommand: Update time/weather
    0x4F, 0x6F, 0x71, 0x67,  # 32-bit timestamp (e.g. 0x67716F4F)
    0x98, 0xC9, 0xAC, 0x31, 0x41, 0x19, 0x00, 0x00,  # 64-bit timestamp
    icon,                # Weather icon ID (0x00 - 0x10)
    temperature_c,       # Celsius
    convert_f,           # If 0x01, convert to Fahrenheit
    time_format          # If 0x01, 12h format; else 24h




**Weather Icon IDs**:
| ID  | Description    |
|-----|----------------|
| 0   | Nothing        |
| 1   | Night          |
| 2   | Clouds         |
| 3   | Drizzle        |
| 4   | Heavy Drizzle  |
| 5   | Rain           |
| 6   | Heavy Rain     |
| 7   | Thunder        |
| 8   | Thunderstorm   |
| 9   | Snow           |
| 10  | Mist           |
| 11  | Fog            |
| 12  | Sand           |
| 13  | Squalls        |
| 14  | Tornado        |
| 15  | Freezing Rain  |
| 16  | Sunny          |

---

## Commands Overview

### General Commands

- **0x02** – Possibly enables “anti-shake” functionality.  
- **0x03** – Toggles silent mode.  
- **0x05** – Subcommand 0x02 sets log level, among others.  
- **0x07** – Displays a countdown on the glasses.  
- **0x09** – Teleprompter.  
- **0x0A** – Navigation (RLE for map images).  
- **0x0D, 0x0E, 0x0F** – Translation functionalities.

### Dashboard Commands (0x06)

- Updates time, weather, and possibly additional “dashboard” info.  
- Timestamps are carried in both 32-bit and 64-bit format. Purpose unclear.

### Teleprompter (0x09) & Navigation (0x0A)

- **Teleprompter (0x09):** Displays text/cues in teleprompter style.  
- **Navigation (0x0A):**  
  - Uses RLE image compression.  
  - Possibly more advanced than just a static image (map updates).

### Translation Commands (0x0D, 0x0E, 0x0F)

- Manage translation sessions; might do on-device speech-to-text.  
- Initially assumed to offload to cloud services, but code suggests local processing.

### Silent Mode (0x03)

- Single command toggles the feature on/off. The app must track state.

### Other Notable Commands

- **0x01** – Possibly initial handshake or sync.  
- **0x26** – Sets distance and height parameters.  
- **Logging Commands**:
  - **0x236C** – Enable logs.  
  - **0x236C31** – Disable logs.  
  - Special debug triggers:
    - `#r` – Restart glasses.  
    - `#t` – Return debug info.

---

## Notification Whitelisting

- The firmware receives a list of app IDs to allow notifications from.  
- Typically limited to 100 apps.  
- Example app IDs:
  - `com.android.phone_missed`, `com.apple.mobilephone_missed`
  - `com.android.phone_incall`, `com.apple.mobilephone`
  - `com.apple.MobileSMS`, `com.android.even_sms`

---

## App Interaction & Whitelisting

- **JSON Usage:** Certain parts of the system use JSON, e.g., [setup.dart](https://github.com/meyskens/fahrplan/blob/main/lib/models/g1/setup.dart).  
- **App List Command:** Send whitelisted apps to the glasses.  
- **API:**  
  - Firmware updates from `api.evenreal.co`.  
  - Potential integration with [Flutter-nRF-Connect-Device-Manager](https://github.com/NordicSemiconductor/Flutter-nRF-Connect-Device-Manager).

---

## Image Handling

### Bitmap (BMP) Handling

- Must send a full `.BMP`, not raw bits.  
- Possibly 4-bit grayscale.  
- **Slots:** Up to 4 stored images.  
- **Artifacts:** If resolution mismatch, glasses show glitchy display.

### Navigation Map Rendering

- Utilizes RLE specifically for map images.  
- Opacity illusions possibly from layered strokes or partial intensities.

---

## Logging & Debugging

### Enabling Debug Mode

1. Set `isDebug=true` in Flutter shared preferences.  
2. Double-tap on the version label in the app.  

This enables extra debug logs and feature toggles.

### Log Prefixes and Sources

- **F4** prefix often indicates firmware logs.  
- Stored in shared preferences.  
- **Commands**:  
  - `0x236C` to enable logs.  
  - `0x236C31` to disable logs.

### Debug Strings in Firmware

- Extensive strings hint at multiple hidden features.  
- Could potentially have a JS engine.  
- Battery impact from too much logging is unknown.

---

## Synchronization & Communication Issues

### Master-Slave Behavior

- On startup, one lens is master, the other is slave.  
- Each side has its own chip.

### Desynchronization Causes

- Glasses occasionally desync despite communication.  
- Sometimes you must send commands twice (once per lens).

### Command Redundancy

- Each lens might require its own copy of a command.  
- Possibly a firmware limitation or due to the master-slave architecture.

---

## Development Tools & Resources

### Firmware Analysis Tools

- **IDA Pro** – For disassembly.  
- **Ghidra** – For decompilation, supports aarch64.  
- **nRF5340 DK + Segger J-Link** – For deeper inspection.

### Libraries & Repositories

- **[Flutter-nRF-Connect-Device-Manager](https://github.com/NordicSemiconductor/Flutter-nRF-Connect-Device-Manager)**  
- **[Even_Glasses (Python)](https://github.com/emingenc/even_glasses)**  
- **[Btlejuice](https://github.com/DigitalSecurity/btlejuice)** – BLE debugging proxy tool.  
- **[Gadgetbridge](https://gadgetbridge.org/)** – Potential for notifications without official app.

### Community Resources

- **Discord Channels:** Various dev & hacking threads.  
- **GitHub Repos:**  
  - [Fahrplan by Meyskens](https://github.com/meyskens/fahrplan) – Dart models for G1.  
- **Online Articles:**  
  - [Even Realities G1 AR Glasses Overview](https://kguttag.com/2024/08/18/even-realities-g1-minimalist-ar-glasses-with-integrated-prescription-lenses/)  
  - [JB Display Products](https://www.jb-display.com/product_des/7.html)

---

## Miscellaneous Notes

- **Temperature Handling:**  
  - API returns Kelvin, the glasses convert to Celsius or Fahrenheit.  
- **UI Customization:**  
  - Potential for custom injection or bridging around the official app.  
- **Emulation:**  
  - No known official emulator for G1.  
- **Button Taps:**  
  - 5 taps reset the glasses.  
  - 10 taps reset user settings.  
- **Firmware Features:**  
  - S/N override is possible.  
  - Multiple font slots, possibly updateable.

---

## Future Work & Considerations

- **Comprehensive Command Docs** – Gather all discovered commands in a single place.  
- **Image Rendering Enhancements** – Stereo BMPs for 3D illusions.  
- **Firmware Decompilation** – Ongoing work via Ghidra and IDA.  
- **Emulator** – Potentially replicate G1 environment for dev.  
- **API Endpoints** – Further analysis of `api.evenreal.co`.  
- **Community Collaboration** – Continue sharing findings, code, and experiments.

---

## Conclusion

This reference is an evolving document for G1 AR glasses reverse engineering. Ongoing collaboration and updates will help reveal further details about the firmware, BLE protocols, and advanced capabilities. Feel free to contribute your findings and help expand this resource.

---
