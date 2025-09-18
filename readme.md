# ESP32 Flow Meter with Bluetooth

This project uses an ESP32 with a YF-S201 water flow sensor (or similar) to measure water flow.
Data is sent via Bluetooth (`Abis_FlowSensor`) and saved in ESP32's Preferences (NVS).

## Features
- Pulse counting with interrupt
- Persistent storage of total liters (NVS)
- Bluetooth output
- Reset command via Bluetooth ('R' or 'r')
- Indian-style number formatting
- LED blink on pulse detection

## Hardware
- ESP32
- Flow sensor (YF-S201 or similar)
- LED indicator on GPIO 25

## License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
