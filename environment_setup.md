# Environment Setup

Before any attack could be attempted, I had to stand up a proper wireless penetration testing environment: a Kali Linux VM with real, physical access to a wireless adapter capable of monitor mode and packet injection (something a VM's default virtualized NIC cannot do), and a baseline survey of the target lab network.

## 1. Hardware Pass-Through

I connected a USB Wi-Fi dongle (MediaTek `mt7921u` chipset) to the host machine and, in VMware Workstation, passed the physical USB device directly through to the Kali guest VM via **VM > Removable Devices > [Wireless Device] > Connect (Disconnect from Host)**. This is a necessary step for wireless attacks in a virtualized lab — the adapter has to be owned by the VM's OS directly, not shared/bridged, so that Kali's wireless tools can set its driver mode.

## 2. Verifying the Adapter

With the device attached, I confirmed it was visible to Kali at both the USB and network-interface level:

```bash
ifconfig      # list network interfaces
lsusb         # list all USB devices, to confirm the dongle enumerated correctly
```

`lsusb` output was checked for the wireless adapter's vendor/product string to confirm the passthrough succeeded before going any further.

## 3. Identifying the Wireless Interface

```bash
sudo airmon-ng
```

This lists the physical wireless interfaces available to `aircrack-ng`'s toolset (PHY, interface name, driver, chipset), which confirmed the adapter's interface name (e.g. `wlan0`) and that Kali had loaded the correct driver for it.

## 4. Enabling Monitor Mode

By default, a Wi-Fi adapter operates in **managed mode**, where it only processes traffic addressed to it and stays associated with a single network. To capture *all* nearby 802.11 traffic — a prerequisite for both the WEP and WPA2 attacks — the adapter needs to be switched into **monitor mode**, which lets it passively receive every frame in range regardless of destination, without being associated to any network.

```bash
sudo airmon-ng start [interface_name] [channel]
```

I ran this against the known lab channel (channel 3) to pin the adapter to the target AP's channel while enabling monitor mode. `airmon-ng` automatically flagged and offered to kill conflicting background processes (`NetworkManager`, `wpa_supplicant`) that would otherwise fight for control of the interface and keep switching it back to managed mode.

The command spawned a second, dedicated monitor-mode interface (suffixed `mon`, e.g. `wlan0mon`), which I confirmed by re-running `ifconfig` and comparing it against the original interface list.

## 5. Reconnaissance Scan

With monitor mode active, I ran a general-purpose scan to survey all nearby wireless activity and locate the two course access points:

```bash
sudo airodump-ng [monitoring_interface] -w [output_filename]
```

After roughly 10 seconds of capture (enough to pick up beacon frames from nearby APs and any active client stations), I stopped the scan with `Ctrl+C` and inspected the resulting capture file:

```bash
cat [output_filename].csv
```

This CSV contains one row per discovered access point (BSSID, channel, encryption type/cipher, ESSID) followed by a second table of observed client stations and which BSSID they're associated with. From this output I recorded the BSSID, channel, and encryption details for the two target SSIDs — one WEP-secured, one WPA2-secured — which are required inputs for the targeted captures in the next two phases.

## Skills Demonstrated

- Passing physical USB hardware through to a virtual machine for direct OS-level control.
- Distinguishing managed mode vs. monitor mode on a wireless interface, and why monitor mode is required for passive 802.11 sniffing.
- Using the Aircrack-ng suite (`airmon-ng`, `airodump-ng`) to enumerate interfaces, enable monitor mode, and survey the RF environment.
- Reading and interpreting `airodump-ng` CSV output to identify target BSSIDs, channels, and encryption schemes ahead of an attack.

**Next:** [WEP Cracking](wep_cracking.md) →
