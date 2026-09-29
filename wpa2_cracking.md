# WPA2 Cracking

The second target was `netsec-WPA2`, secured with WPA2-PSK — a far stronger protocol than WEP that isn't vulnerable to passive statistical key recovery. Instead, this phase required an **active** attack to force a handshake capture, followed by an **offline** dictionary attack against that handshake.

## Why WPA2 Requires a Different Approach

WPA2-PSK doesn't transmit the encryption key or anything derived directly from it over the air in a recoverable way. Instead, when a client connects, the client and AP perform a **4-way handshake** to independently derive matching session keys from a shared secret — the Pairwise Master Key (PMK) — which itself is derived from the network passphrase and SSID via PBKDF2. The handshake exchanges nonces and a Message Integrity Check (MIC) that can be verified against a *candidate* passphrase, but never reveals the passphrase itself. This means the only way to attack WPA2-PSK is to capture a valid handshake and then test candidate passphrases against it — there's no equivalent of WEP's IV-leakage shortcut.

## 1. Starting the Capture

I navigated to a dedicated working directory and started a capture scoped to the WPA2 AP's BSSID and channel, then left it running in one terminal tab:

```bash
cd ~/Lab1-WiFiSecurity/wpa2/

sudo airodump-ng -c [channel] --bssid [WPA2_BSSID] -w [output_filename] [monitoring_interface]
```

At this point the capture is just listening — it needs to catch a client's 4-way handshake, which only happens when a device connects or reconnects. Rather than wait indefinitely for one to occur naturally, I forced one.

## 2. Identifying an Associated Client

In a second terminal tab, I went back to the earlier reconnaissance scan CSV and filtered it for stations (client devices) already associated with the target BSSID:

```bash
cd ~/Lab1-WiFiSecurity/
cat [scan_filename].csv | column -t -s, | less -S

# Or, filtered directly to the target BSSID's associated stations:
TARGET_AP="[WPA2_BSSID]"; FILENAME="[scan_filename].csv"; awk -v bssid="$TARGET_AP" \
  '/Station MAC/ {hdr=$0; show=1; next} show && $0 ~ bssid {if (hdr) {print hdr; hdr=""} print}' \
  "$FILENAME" | column -t -s,
```

This identified one or more in-scope devices (instructor/TA machines set up specifically as attack targets for this exercise) currently connected to the WPA2 network, giving me a station MAC address to target with a deauthentication attack.

## 3. Forcing a Handshake with a Targeted Deauth

```bash
sudo aireplay-ng --deauth 5 -a [WPA2_BSSID] -c [Target_Station_MAC] [monitoring_interface]
```

This sends forged 802.11 deauthentication frames, spoofed as coming from the AP, directly to the target client. Because 802.11 management frames were unauthenticated in this deployment, the client has no way to distinguish a forged deauth from a legitimate one, and immediately disconnects — then automatically reconnects, triggering a fresh 4-way handshake. Switching back to the first terminal tab, the running `airodump-ng` capture showed a `WPA handshake:` indicator confirming it had caught the exchange, along with a spike in EAPOL frames.

## 4. Offline Dictionary Attack

With a captured handshake in hand, key recovery moves entirely offline — no further interaction with the AP or client is needed:

```bash
sudo aircrack-ng -w [wordlist_path] -b [WPA2_BSSID] [output_filename].cap
```

`aircrack-ng` iterates through the wordlist, deriving a candidate PMK/PTK for each entry and checking it against the handshake's MIC. Using the course-provided (deliberately small) wordlist, the correct passphrase was found within seconds. A production-grade attack against a strong, random passphrase would use a much larger wordlist (or brute-force character sets) and could realistically take hours to years with no guarantee of success — which is exactly the point: **WPA2 itself held up; a weak, dictionary-guessable passphrase is what failed.**

## Why Capturing a Handshake Is Enough to Crack WPA2

The 4-way handshake contains everything needed to *verify* a guess offline: the AP and client nonces (sent in the clear) and a MIC computed using session keys derived from the PMK. An attacker can take any candidate passphrase, run it through the same PBKDF2/PRF derivation using the captured SSID and nonces, and check whether the resulting MIC matches the one observed in the handshake. If the passphrase is in the attacker's wordlist, this offline comparison will find it — no rate-limiting, lockouts, or online interaction with the AP can stop this because the entire attack happens locally, at whatever speed the attacker's hardware can hash. This is precisely why passphrase strength (length, randomness, absence from common wordlists) is the *actual* security boundary for WPA2-PSK, not the protocol's handshake mechanics.

## Skills Demonstrated

- Understanding the WPA2 4-way handshake and PMK/PTK/MIC derivation at a conceptual level.
- Executing a targeted 802.11 deauthentication attack to force a handshake on demand.
- Filtering and correlating `airodump-ng` scan data (via `awk`/`column`) to identify live attack targets.
- Performing an offline dictionary attack with `aircrack-ng` and understanding why it's an offline, unrate-limited process.
- Articulating the real security takeaway: protocol strength vs. passphrase strength, and why long/random Wi-Fi passwords matter in practice.

**Back to:** [README](README.md)
