---
name: wireless-hardware-attack
description: >-
  Wireless/hardware: WiFi PMKID/WPS/Evil Twin, BLE, Zigbee, NFC, SDR, UART/JTAG/SPI, side channels, fault injection. Use when attacking WiFi, BLE, RFID, SDR, or hardware interfaces.
metadata:
  tags: [penetration-testing, penetration-testing, red-team]
---

## Wireless / Hardware Attacks

```
=== Wireless/Hardware ===
WiFi: aircrack-ng/PMKID (hcxdumptool+hashcat)/WPS reaver/Evil Twin | BLE: gatttool enumerate GATT/unauthenticated read-write/Just Works
Zigbee killerbee | NFC/RFID proxmark3 clone/MIFARE mfoc | LoRa/SDR rtl-sdr+gnuradio replay
Hardware: UART baud rate scanning to get a shell | JTAG/SWD read-write firmware | SPI flash dump | side-channel DPA/timing | fault injection (voltage/clock glitching to bypass authentication) | binwalk -Me
```
