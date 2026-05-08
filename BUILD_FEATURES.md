# Haven Node Firmware — What's Included

A custom OpenWrt 23.05 build for Raspberry Pi 5 with the kernel uplifted to 6.6 (for full Pi 5 / RP1 support), the Morse Micro HaLow stack, and a curated set of mesh networking, VPN, and field-comms tooling.

---

## Capabilities at a Glance

### Long-Range Wi-Fi (802.11ah / HaLow)
- Morse Micro MM6108 (SPI HAT) and MM8108 (USB) chipset support
- HaLow driver, firmware, country-code regulatory database
- First-boot wizard for AP / Mesh / Station mode selection
- 802.11s mesh networking via `mesh11sd` for self-healing wireless backbone
- S1G-aware hostapd and wpa_supplicant for HaLow-native authentication

### Standard Wi-Fi
- Built-in Pi 5 dual-band 2.4 / 5 GHz Wi-Fi (Broadcom BCM43455)
- Add-on USB dongle support: Panda Wireless RT5370 (`rt2800usb`)
- WPA2/WPA3 authentication stack (`wpad-basic-mbedtls`)

### Mesh Networking
- `batman-adv` Layer-2 mesh: transparent multi-hop, no app changes required
- `batctl` CLI for diagnostics and topology inspection
- LuCI proto integration for UI-driven mesh configuration

### VPN and Encrypted Overlay
- **WireGuard**: kernel-level VPN with LuCI proto integration — minimal overhead, point-to-point or hub-and-spoke
- **Tailscale**: full client/daemon — works with Tailscale Cloud *or* a self-hosted Headscale control plane for fully off-grid private networks

### ADS-B Aircraft Tracking
- RTL-SDR userspace and library (`rtl-sdr`, `librtlsdr`)
- `dump1090` ADS-B Mode S decoder
- Python 3 runtime with `cryptography` pre-installed for `adsbcot` (Cursor-on-Target ADS-B feed)
- One-shot install on first boot: `pip3 install adsbcot`

### LoRa / Reticulum Mesh (sidecar)
- Python 3 with `pyserial` and `netifaces` baked in
- Reticulum / LXMF / NomadNet installable on first boot: `pip3 install rns lxmf nomadnet`
- USB-ACM (CDC) and USB-serial drivers for every popular LoRa devboard:
  - Heltec V1 (nRF52) / V4 (ESP32-S3)
  - RAK4631 (nRF52840)
  - Seeed Xiao ESP32-S3
  - Walter (ESP32-S3)
  - Muzi Works Base Duo
  - Null Hop Mesh Toad (CH341)

### Web Administration (LuCI)
- Morse Argon theme — modern Argon Dashboard look, applied automatically on first boot
- Morse AP / Mesh / Station setup wizard
- Per-device configuration, status, and upgrade apps
- WireGuard, batman-adv, and standard network config UIs

### Network Debugging
- `tcpdump` — packet capture
- `iperf3` — throughput testing
- `mtr` — live network path / latency analysis
- `nmap` — network discovery
- `usbutils` (`lsusb`) — USB device enumeration
- `iw` (full build) — low-level Wi-Fi CLI for scan, link, station, and mesh inspection (including HaLow / S1G channels)

### General-Purpose Utilities
- Bash shell
- `curl` with TLS and trusted CA bundle
- `nano` editor
- `htop` system monitor
- `screen` terminal multiplexer
- mDNS responder (`umdns`) for Bonjour-style device discovery

---

## Marketing-Friendly Bullet Bank

Drop these into datasheets, web copy, or one-pagers as-is. Each line is independent — pick the ones that matter for your audience.

**Networking**
- Long-range, low-power Wi-Fi (Morse Micro HaLow) for sub-GHz mesh — up to ~1 km point-to-point
- Self-healing wireless mesh built on 802.11s plus batman-adv Layer 2
- Onboard Pi 5 dual-band Wi-Fi for local management and hotspot use
- WireGuard and Tailscale built in — bring your own control plane (Headscale) for fully off-grid private networks

**Comms and field tooling**
- Out-of-the-box ADS-B aircraft tracking with feed-to-CoT (Cursor-on-Target) integration
- LoRa / Reticulum sidecar support — works with every popular ESP32 and nRF52 LoRa devboard via plug-in USB
- Modern web dashboard with wizard-driven setup — no SSH required to deploy a node
- Field-debugging toolkit pre-installed: tcpdump, iperf3, mtr, nmap, lsusb

**Platform**
- Raspberry Pi 5 (aarch64) on OpenWrt 23.05 with kernel 6.6 — current driver ecosystem on a stable LTS-style release
- Apache-licensed and MIT-licensed components only; no proprietary binary blobs except those required by the radio firmware

**Positioning lines**
- "An off-grid mesh node you can stand up in five minutes."
- "HaLow long-range, WireGuard private, ADS-B aware — out of one image."
- "From Tailscale to LoRa to long-range Wi-Fi, on a Pi 5 you can mount in a battery box."

---

## What's Not in This Image (and Why)

Transparency for technical evaluators:

- **MediaTek MT76x0 USB Wi-Fi (`kmod-mt76x0u`)** — disabled. The mt76 driver source in this OpenWrt vintage is not compatible with the 6.6 kernel + backports headers used here. Enable once mt76 is bumped.
- **Realtek RTL8821CU USB Wi-Fi (`kmod-rtl8821cu`)** — not packaged in standard OpenWrt feeds. Requires an out-of-tree driver feed if needed.
- **`adsbcot`, `python3-rns`, `python3-lxmf`, `python3-nomadnet`** — not packaged for OpenWrt; install with `pip3` on first boot. The compiled native dependencies (`cryptography`, `pyserial`, `netifaces`) are already in the image, so the pip step is fast.
- **Headscale server** — Headscale is a server-side control plane and is not appropriate for a Pi mesh node; run it on a regular Linux VM and point Tailscale at it with `tailscale up --login-server=https://your-headscale.example.com`.

---

## Target Hardware

- **Primary:** Raspberry Pi 5 (`bcm27xx/bcm2712`)
- **HaLow add-on (recommended):** Seeed Studio Mesh Hat (Morse MM6108, SPI) or Gateworks USB module (MM8108)
- **Optional sidecars:** any USB-CDC or USB-serial LoRa board; RTL-SDR USB dongle for ADS-B
