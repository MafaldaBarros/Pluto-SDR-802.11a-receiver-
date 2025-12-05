# Pluto SDR 802.11 Receiver

A GNU Radio Companion (GRC) project for scanning and analyzing WiFi signals in the 2.4 GHz band using the ADALM-Pluto Software Defined Radio (SDR) hardware.

## Overview

This repository contains GNU Radio blocks and flowgraphs for:
- Scanning WiFi channels in the 2.4 GHz frequency band (channels 1-13)
- Automatic channel sweeping with configurable dwell time
- Real-time spectrum visualization
- Capturing 802.11 MAC layer frames

## Images
![Constelation plot](/resources/const.jpeg)
![Decoded frames in Wireshark](/resources/wireshark.png)


## Requirements

### Software
- GNU Radio 3.8 or later (tested with 3.10)
- Python 3.7+
- gr-iio (for PlutoSDR support)
- PyQt5 (for GUI components)


## Usage

### Quick Start

1. **Connect your SDR device** (e.g., PlutoSDR via USB)

2. **Open GNU Radio Companion:**
```bash
gnuradio-companion
```

3. **Load the flowgraph:**
   - File → Open → Select `wifi_channel_scanner.grc`

4. **Configure your SDR:**
   - Update the SDR Source block with your device's connection string

5. **Run the flowgraph:**
   - Click the "Execute" button (▶️), press F6, or run the python file in the terminal


### Configuration Parameters

**PlutoSDR Settings:**
```
Center Frequency: 2437 MHz (Channel 6)
Sample Rate: 20 MHz
RF Bandwidth: 20 MHz
Gain Mode: Manual
Gain: 60-70 dB
```


## WiFi Channel Reference

### 2.4 GHz Band (802.11b/g/n)

| Channel | Frequency | Common Use |
|---------|-----------|------------|
| 1       | 2412 MHz  | Standard   |
| 2       | 2417 MHz  |            |
| 3       | 2422 MHz  |            |
| 4       | 2427 MHz  |            |
| 5       | 2432 MHz  |            |
| 6       | 2437 MHz  | Standard   |
| 7       | 2442 MHz  |            |
| 8       | 2447 MHz  |            |
| 9       | 2452 MHz  |            |
| 10      | 2457 MHz  |            |
| 11      | 2462 MHz  | Standard   |
| 12      | 2467 MHz  | EU only    |
| 13      | 2472 MHz  | EU only    |

**Formula:** Frequency (MHz) = 2407 + (Channel × 5)


## License

This project is licensed under the GNU General Public License v3.0 - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- GNU Radio project and community
- Analog Devices for PlutoSDR support
- WiFi 802.11 standards documentation

## Related Projects

- [gr-ieee802-11](https://github.com/bastibl/gr-ieee802-11) - IEEE 802.11 a/g/p transceiver
- [Scapy](https://scapy.net/) - Packet manipulation tool
- [Wireshark](https://www.wireshark.org/) - Network protocol analyzer

## Resources

- [GNU Radio Wiki](https://wiki.gnuradio.org/)
- [PlutoSDR Documentation](https://wiki.analog.com/university/tools/pluto)
- [802.11 Frame Format](https://en.wikipedia.org/wiki/802.11_Frame_Types)

---

**Note:** This tool is intended for educational and research purposes. Always ensure compliance with local regulations regarding RF monitoring and spectrum usage.