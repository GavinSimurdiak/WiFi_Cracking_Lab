# Wi-Fi Cracking Lab

A hands-on wireless security lab demonstrating reconnaissance, traffic capture, and key-recovery attacks against WEP and WPA2-PSK secured access points, using the Aircrack-ng suite on Kali Linux. 

## Overview

This project documents a full offensive wireless workflow, from environment setup through key recovery, against two intentionally vulnerable access points set up for this course: one secured with legacy **WEP** encryption and one secured with modern **WPA2-PSK**. The goal was to understand *why* WEP is fundamentally broken and *how* WPA2 can still fall to a dictionary attack when protected by a weak passphrase, by actually carrying out both attacks end to end rather than just reading about them.

The workflow covered in this repo:

1. **[Environment Setup](environment_setup.md)** — configuring a Kali Linux VM with a monitor-mode-capable USB Wi-Fi adapter and surveying the local wireless spectrum.
2. **[WEP Cracking](wep_cracking.md)** — passively capturing initialization vectors (IVs) and using a statistical attack to recover a WEP key.
3. **[WPA2 Cracking](wpa2_cracking.md)** — capturing a WPA2 4-way handshake via a targeted deauthentication attack, then recovering the passphrase offline with a dictionary attack.

## Tools & Technologies

| Category | Tool / Technology |
|---|---|
| OS / Platform | Kali Linux (VM), VMware Workstation |
| Wireless hardware | USB Wi-Fi adapter, MediaTek `mt7921u` chipset, monitor-mode capable |
| Packet capture / injection | [Aircrack-ng suite](https://www.aircrack-ng.org/) — `airmon-ng`, `airodump-ng`, `aireplay-ng`, `aircrack-ng` |
| Networking utilities | `ifconfig`, `lsusb`, `awk`, `column` |
| Attack techniques | Monitor mode sniffing, IV collection, targeted deauthentication, WPA 4-way handshake capture, dictionary/brute-force key recovery |

## Network Architecture / Lab Topology

The lab ran entirely inside an isolated classroom Wi-Fi environment with no connectivity to the public internet or any production network:

```
                        +---------------------------+
                        |   Instructor-Managed AP    |
                        |   (channel 3, isolated)    |
                        |                             |
                        |  SSID: netsec-WEP  (WEP)   |
                        |  SSID: netsec-WPA2 (WPA2)  |
                        +---------------------------+
                           |                     |
                 (passive capture)      (assoc. client / STA)
                           |                     |
                +----------------------+   +----------------+
                |   Attacker: Kali VM   |   | TA/Instructor  |
                |  USB Wi-Fi adapter    |   |    device      |
                |  in monitor mode      |   | (deauth target)|
                +----------------------+   +----------------+
```

- A single access point broadcast two separate SSIDs on the same channel — one using WEP, one using WPA2-PSK — for direct comparison of the two protocols.
- One or more instructor/TA devices stayed associated to the WPA2 network to serve as legitimate, in-scope deauthentication targets, forcing a fresh handshake for capture.
- The AP had no upstream network connection, so successfully cracked credentials could be used to authenticate but not to reach the internet — connectivity wasn't the point, key recovery was.

## Key Technical Takeaways

- **Monitor mode vs. managed mode**: a wireless interface must be put into monitor mode to passively capture all 802.11 frames in the air (not just traffic addressed to it), which is the foundation for every attack in this lab.
- **Why WEP is broken**: WEP's RC4 stream cipher relies on a 24-bit initialization vector (IV) that is transmitted in plaintext with every packet. Because the IV space is so small, IVs inevitably repeat on a busy network, and collecting enough of them (~40,000+) lets statistical attacks (as implemented in `aircrack-ng`) recover the key in seconds — no brute force required.
- **Why WPA2 is stronger, but not unbreakable**: WPA2-PSK derives its encryption keys from the passphrase and SSID during a 4-way handshake between client and AP. Capturing that handshake doesn't reveal the password directly — it only enables an *offline* dictionary or brute-force attack, whose success depends entirely on whether the real passphrase exists in the attacker's wordlist.
- **Deauthentication as an attack primitive**: sending forged deauth frames to a connected client forces it to reassociate, which is a reliable way to capture a fresh WPA handshake on demand rather than waiting for one to occur naturally.
- **Passphrase strength matters more than protocol strength**: WPA2 itself wasn't "broken" in this lab — a weak, wordlist-guessable passphrase was. This is the practical argument for long, random, non-dictionary Wi-Fi passwords.

## Legal & Ethical Disclaimer

**All activities documented in this repository were performed exclusively against access points and client devices deployed specifically for this exercise in an authorized, isolated academic lab environment**, as part of a university network security course. The attack traffic never left a segmented virtual machine / classroom Wi-Fi setup, and no production network, public network, or third-party infrastructure was targeted at any point.
