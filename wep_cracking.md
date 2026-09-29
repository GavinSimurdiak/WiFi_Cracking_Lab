# WEP Cracking

With the environment set up and the target BSSID identified for the `netsec-WEP` network (see [Environment Setup](environment_setup.md)), the next objective was to recover its pre-shared key by exploiting a fundamental cryptographic weakness in WEP itself, rather than by brute-forcing it.

## Why WEP Is Broken

WEP (Wired Equivalent Privacy) encrypts each packet using the RC4 stream cipher, keyed with the network password concatenated with a 24-bit **Initialization Vector (IV)** that is sent in the clear with every single packet. Because the IV space is only 2^24 (~16.7 million) values, and because busy networks send many packets per second, IV reuse is statistically guaranteed to happen relatively quickly. Every IV reuse leaks a small amount of information about the underlying key, and once enough repeated/weak IVs have been collected, statistical attacks (FMS/PTW/KoreK, all implemented inside `aircrack-ng`) can reconstruct the full key without ever needing to guess it via brute force. This is why WEP is considered fundamentally broken, not just "weak": no amount of passphrase complexity fixes it, and it has been deprecated since 2004.

## 1. Targeted Packet Capture

I navigated to a dedicated working directory to keep the capture organized, then launched a capture scoped specifically to the WEP AP's BSSID and channel (rather than the general survey scan from setup):

```bash
cd ~/Lab1-WiFiSecurity/wep/

sudo airodump-ng -c [channel] --bssid [WEP_BSSID] -w [output_filename] [monitoring_interface]
```

Scoping the capture to a single BSSID/channel (instead of sniffing everything in range) keeps the `.cap` file focused on relevant traffic and makes IV collection much faster.

## 2. Collecting Initialization Vectors

While the capture ran, I watched the live `#Data` counter in `airodump-ng`'s output, which tracks the number of Layer 2 data packets (and therefore IVs) captured against the target BSSID. I let the capture run until it exceeded **40,000 IVs** — the generally accepted threshold for a high-probability statistical key recovery with `aircrack-ng` — before stopping the capture with `Ctrl+C`.

## 3. Cracking the Key

With enough IVs collected, recovering the key was a matter of running `aircrack-ng` against the capture file:

```bash
sudo aircrack-ng -b [WEP_BSSID] [output_filename].cap
```

`aircrack-ng` ran its statistical attack across the collected IVs and returned the cracked key, byte-by-byte with a per-byte confidence vote, in a matter of seconds — a dramatic contrast to how long a true brute-force attack against the same keyspace would take.

## Why So Many IVs Are Needed

A single IV collision or two isn't enough to recover a key — the statistical attack works by looking for *patterns across a large sample* of IVs that leak biased information about specific key bytes, then using weighted voting across all observed IVs to converge on the most likely value for each byte. With too few IVs, the noise dominates and no byte reaches high confidence; with 40,000–50,000+ IVs, enough weak/repeated IVs have typically been observed for each key byte's vote to converge correctly. It's a numbers game rooted in the IV space being too small for a busy network to avoid reuse — not a password-guessing exercise at all.

## Skills Demonstrated

- Scoping a packet capture to a specific BSSID/channel for efficient, targeted data collection.
- Understanding IV-based statistical cryptanalysis and why it defeats RC4/WEP regardless of key complexity.
- Practical use of `airodump-ng` for capture and `aircrack-ng` for key recovery.
- Recognizing why WEP is unsuitable for any modern network, informing real-world decisions to disable/replace it.

**Next:** [WPA2 Cracking](wpa2_cracking.md) →
