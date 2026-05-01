# AGENTS.md

High-level context for agents and contributors landing in this repo cold. For build instructions, see [README.md](README.md).

## What this repo is

A community backport that brings Morse Micro HaLow firmware to the Raspberry Pi 5. The official Morse Micro OpenWrt SDK ships on OpenWrt 23.05 / kernel 5.15 and only supports Pi 4 — Pi 5 needs OpenWrt 24.10's bcm2712/RP1 hardware definitions and kernel 6.6, which Morse hasn't shipped yet. This repo vendors those hardware definitions back into the 23.05 SDK so the existing Morse package set can build for Pi 5.

The work is sometimes called a "kernel uplift backport": same Morse SDK userspace, target-wide kernel bump 5.15 → 6.6, mac80211 backports 6.1.110 → 6.12.61, plus net-new bcm2712 subtarget, RP1 patches, and Pi 5 device profile.

## Repo layout

| Path | What it is |
|---|---|
| `boards/ekh-bcm2712/target_diffconfig` | Pi 5 board config — single source of truth for what packages go in the image |
| `boards/ekh-bcm2711/` | Pi 4 reference board, unmodified from Morse's SDK |
| `target/linux/bcm27xx/bcm2712/` | New subtarget (config-6.6, default packages) |
| `target/linux/bcm27xx/patches-6.6/` | Vendored from upstream openwrt-24.10 (~1300 patches) plus our additions |
| `target/linux/bcm27xx/image/Makefile` | Contains `Device/rpi-5` definition |
| `target/linux/generic/` | Kernel 6.6 vendored from openwrt-24.10 (config, backport, hack, pending) |
| `package/kernel/mac80211/` | Wholesale-synced with openwrt-24.10 for the 6.12.61 backports |
| `package/firmware/cypress-firmware/` | Pi 5 onboard Wi-Fi firmware (CYW43455) |
| `feeds/morse/` | Morse HaLow packages (driver, mac80211 patches, LuCI apps, mode/wizard) |

## Branch and versioning

| Item | Value |
|---|---|
| Active branch | `rpi5-mm-23.05` |
| Upstream base | `mm/v23.05.5` (Morse SDK 23.05.5) |
| Kernel | 6.6.133 (vendored from openwrt-24.10) |
| mac80211 backports | 6.12.61 |
| Morse driver | 1.16.4 |

Releases on this fork are tagged `vX.Y.Z[-alpha]` and built locally on the maintainer's host (no CI yet). Image artifacts are uploaded to the GitHub release page; the repo itself does not check in build outputs.

## Hardware status

| Device | Chip | Bus | Status |
|---|---|---|---|
| Seeed Studio HaLow HAT | MM6108 | SPI | **Not binding** — overlay + RP1 SPI/DMA in place, chip doesn't probe |
| Gateworks MM8108 | MM8108 | USB | **Working** — verified 2026-04-26 |
| Pi 5 onboard Wi-Fi | BCM43455 (Cypress) | SDIO | **Not working** — see known issues |
| Panda Wireless dongle | Ralink RT5370 | USB | **Working** as 2.4 GHz AP — verified 2026-04-26 |
| HaLow mesh / batman-adv | — | — | **Working** — verified 2026-04-26 with gate + point topology |

When you read "the build is working" without qualification, default to MM8108/USB. The original goal — MM6108/SPI on Pi 5 — is still open.

### UPS / portable power

The **Waveshare UPS HAT (B) with 2×21700 cells** (designed for Pi 4) does **not** power the Pi 5 via GPIO header — the Pi 5 requires USB-C power delivery and will only show a red LED if powered through GPIO 5V pins. **Workaround:** connect the UPS HAT's USB-A output to the Pi 5's USB-C input with a USB-A to USB-C cable. This provides ~2.4A at 5V which is enough for a mesh node under normal load. The Waveshare UPS HAT (C) is the Pi 5-native version with USB-C PD output if a cleaner solution is needed.

## First-boot setup (post-flash)

After flashing the image and booting the Pi 5, SSH in via `root@192.168.1.1` (default OpenWrt address on `eth0`/`br-lan`). The image requires manual configuration — there is no first-boot wizard (see known issues).

### Morse USB radio PHY path fix

OpenWrt's `wifi detect` generates the wrong sysfs path for the Morse USB radio on Pi 5. It writes `axi/1000120000.pcie/...` but netifd needs the full path `platform/axi/1000120000.pcie/.../3-1:1.0`. Symptoms: `radio0` shows `"up": false` and logread shows `Phy not found`.

**Fix:** After boot, read the actual PHY path and update UCI:

```sh
# Find the real Morse PHY path
FULL_PATH=$(for p in /sys/class/ieee80211/phy*; do
    link=$(readlink "$p")
    echo "$link" | grep -q '3-1:1.0/ieee80211' && echo "$link" | sed 's|^../../devices/||' && break
done)
uci set wireless.radio0.path="$FULL_PATH"
uci commit wireless
wifi up
```

This must be done after every reboot because `wifi detect` regenerates the short path. A permanent fix requires either a uci-defaults script or an rc.local hook (see below), or fixing the Morse wifi detect script upstream.

**Recommended rc.local workaround** (`/etc/rc.local`):

```sh
#!/bin/sh
sleep 5
FULL_PATH=$(for p in /sys/class/ieee80211/phy*; do
    link=$(readlink "$p")
    echo "$link" | grep -q '3-1:1.0/ieee80211' && echo "$link" | sed 's|^../../devices/||' && break
done)
if [ -n "$FULL_PATH" ]; then
    CURRENT=$(uci -q get wireless.radio0.path)
    if [ "$CURRENT" != "$FULL_PATH" ]; then
        uci set wireless.radio0.path="$FULL_PATH"
        uci commit wireless
        wifi up
    fi
fi
exit 0
```

### Duplicate radio entries

`wifi detect` may also create a duplicate radio entry (e.g. `radio2`) with the correct full path, while `radio0` keeps the broken short path. Delete the duplicate and fix `radio0`:

```sh
uci delete wireless.radio2
uci delete wireless.default_radio2
uci commit wireless
```

### Mesh + BATMAN-adv point node setup

For a point node that extends internet from a gate node via HaLow mesh, the architecture is:

```
[Internet] → [Gate eth0] → [Gate bat0/HaLow mesh] ~~~~ [Point HaLow mesh/bat0] → [Point 2.4GHz AP] → [Client devices]
```

The HaLow mesh uses **802.11s with `mesh_fwding=0`** (forwarding disabled) and **BATMAN-adv** for L2 routing over `bat0`. Plain 802.11s bridging will not work with gate nodes running BATMAN-adv.

**Key UCI settings for a point node:**

```sh
# HaLow mesh → batman-adv hard interface
uci set wireless.default_radio0.network='batmesh'
uci set wireless.default_radio0.mesh_fwding='0'
uci set wireless.default_radio0.beacon_int='1000'
uci set wireless.default_radio0.mode='mesh'
uci set wireless.default_radio0.mesh_id='haven'
uci set wireless.default_radio0.encryption='sae'
uci set wireless.default_radio0.key='havenmesh'

# batmesh network — binds HaLow wlan to bat0
uci set network.batmesh=interface
uci set network.batmesh.proto='batadv_hardif'
uci set network.batmesh.master='bat0'

# bat0 — BATMAN-adv virtual interface
uci set network.bat0=interface
uci set network.bat0.proto='batadv'
uci set network.bat0.routing_algo='BATMAN_V'
uci set network.bat0.gw_mode='client'
uci set network.bat0.orig_interval='1000'

# ahwlan bridge — bat0 + client AP on 10.41.0.0/16
uci set network.ahwlan=interface
uci set network.ahwlan.proto='static'
uci set network.ahwlan.ipaddr='10.41.0.2'
uci set network.ahwlan.netmask='255.255.0.0'
uci set network.ahwlan.gateway='10.41.0.1'   # adjust to gate's actual IP
uci set network.ahwlan.dns='8.8.8.8 8.8.4.4'
uci set network.ahwlan.device='br-ahwlan'

# Bridge device with bat0
uci set network.ahwlan_dev=device
uci set network.ahwlan_dev.name='br-ahwlan'
uci set network.ahwlan_dev.type='bridge'
uci add_list network.ahwlan_dev.ports='bat0'

# 2.4GHz USB dongle as client AP on ahwlan
uci set wireless.default_radio1.network='ahwlan'
uci set wireless.default_radio1.mode='ap'
uci set wireless.default_radio1.ssid='Haven'
uci set wireless.default_radio1.encryption='psk2'
uci set wireless.default_radio1.key='havenmesh'

# Disable DHCP on point (gate serves 10.41.x.x)
uci set dhcp.ahwlan=dhcp
uci set dhcp.ahwlan.interface='ahwlan'
uci set dhcp.ahwlan.ignore='1'
```

For the full scripted setup, see [haven-manet-ip-mesh-radio/scripts/node-setup/setup-haven-point.sh](https://github.com/buildwithparallel/haven-manet-ip-mesh-radio/tree/main/scripts/node-setup/setup-haven-point.sh) (Pi 4 reference — same UCI patterns apply to Pi 5).

### LuCI Tx-Power display

LuCI shows **Tx-Power: 20 dBm** for the Morse radio. This is a cosmetic artifact — the Morse driver maps S1G channels to 5 GHz HT channels internally (e.g. S1G ch28 → HT ch114), and `iw` reads the mac80211 regulatory txpower for that mapped channel. The actual sub-GHz transmit power is controlled by the Morse firmware and board config file (BCF), typically 21–27 dBm. The `phy#1 (self-managed)` regulatory domain shows the true S1G limit: 36 dBm EIRP for US 920–928 MHz. Use `morse_cli -i wlan0 stats` to see actual RF parameters.

## Build flow

Three-line summary; see README for details:
1. `cp boards/ekh-bcm2712/target_diffconfig .config && make defconfig`
2. `./scripts/feeds update -a && ./scripts/feeds install -a` (first time or after feed bumps)
3. `make -j$(nproc) V=sc 2>&1 | tee log.txt`

The board diffconfig is what determines image contents. To add a package, append `CONFIG_PACKAGE_<name>=y` to `boards/ekh-bcm2712/target_diffconfig` and rebuild — do not write to `.config` directly, since `make defconfig` regenerates it.

## Upstream sources

| Repo | Purpose | Version pinned |
|---|---|---|
| `MorseMicro/openwrt` | Base SDK | branch `mm/v23.05.5` |
| `MorseMicro/morse-feed` | HaLow packages | branch `2.9-dev` (via `feeds.conf.default`) |
| `MorseMicro/morse_driver` | Driver source | tag 1.16.4 |
| `openwrt/openwrt` | Pi 5 hardware donor | branch `openwrt-24.10` |

`feeds.conf.default` is the active manifest. Don't fork the feeds unless you actually need to modify them.

## Known issues / gotchas

- **MM6108/SPI on Pi 5 doesn't bind.** Overlay (`mm610x-spi-pi5`), RP1 DMA, DesignWare SPI configs all in place. Chip is enumerated by SPI controller but driver doesn't attach. Open investigation.
- **No `persistent-vars-storage-bcm2712` package.** Morse provides `persistent-vars-storage-bcm2711` for Pi 4 and `persistent-vars-storage-ubootenv` for u-boot devices; Pi 5 needs its own variant. This causes `wizard-config` (which depends on it) to be silently dropped from the image, which means no first-boot UCI defaults — users must run the LuCI AP wizard manually after first boot.
- **`mac80211` Morse subsys patches dropped.** The 10 Morse 999-* patches (S1G ECSA, IBSS bridge, mesh, NDP block ack, etc.) were authored against backports 6.1.110 and didn't apply against 6.12.61. Currently dropped — must be re-authored before HaLow protocol features fully work. Alpha builds bind without them.
- **Pi 4 SPI overlay (`999-001-morse-spi-fix-spi-bcm2835-driver`) not ported.** Used a v5.3-era of_gpio API that doesn't exist in 6.6. Pi 4 HaLow over SPI will regress until re-authored against 6.6 spi-bcm2835.
- **Morse USB radio PHY path wrong after boot.** OpenWrt's `wifi detect` generates a truncated sysfs path for the MM8108 USB radio (`axi/...` instead of `platform/axi/.../3-1:1.0`). This causes `netifd` to fail with `Phy not found` on every boot. `wifi detect` also creates duplicate radio entries (e.g. `radio2`) with the correct path while leaving the broken `radio0`. Workaround: rc.local script that fixes the path (see "First-boot setup" above). Proper fix: patch the Morse `wifi detect` script or OpenWrt's `mac80211.sh` to emit the full platform path for USB devices on Pi 5.
- **Pi 5 onboard Wi-Fi (BCM43455) not working.** Two issues: (1) Missing NVRAM file — the driver looks for `brcm/brcmfmac43455-sdio.raspberrypi,5-model-b.txt` which doesn't exist in the firmware package. Workaround: symlink the Pi 4 NVRAM (`ln -sf brcmfmac43455-sdio.raspberrypi,4-model-b.txt /lib/firmware/brcm/brcmfmac43455-sdio.raspberrypi,5-model-b.txt`). Proper fix: add the symlink to the `brcmfmac-nvram-43455-sdio` package. (2) Missing `brcmfmac-wcc` vendor module — kernel 6.6+ brcmfmac was refactored to load chip-specific vendor modules (`brcmfmac-wcc.ko` for Cypress/WCC chips). The mac80211 backports 6.12.61 expect it but `broadcom.mk` only builds the monolithic `brcmfmac.ko`. Error: `brcmf_fwvid_request_module: mod=wcc: failed` → `brcmf_attach failed`. Fix requires adding `CONFIG_BRCMFMAC_WCC` to the kernel config and a `kmod-brcmfmac-wcc` package definition in `package/kernel/mac80211/broadcom.mk`. Until fixed, onboard Wi-Fi is unavailable — use the Panda USB dongle for 2.4 GHz AP instead.
- **No CI.** Builds happen on the maintainer's host. Reproduce with `boards/ekh-bcm2712/target_diffconfig` per the README.

## How to extend

- **Add a package**: edit `boards/ekh-bcm2712/target_diffconfig`, add `CONFIG_PACKAGE_<name>=y`, rebuild. If the package's deps include something Pi 5–specific (e.g. a per-platform variant), check whether a `*-bcm2712` variant exists; if not, that's a new package to write.
- **Add a Pi 5 device tree change**: drop a patch in `target/linux/bcm27xx/patches-6.6/`. Numbering 991-* is Morse-overlay, 992+ is our additions. Re-anchor against 6.6 paths (DTS files moved from `arch/arm/boot/dts/` to `arch/arm/boot/dts/broadcom/`).
- **Sync with upstream Morse changes**: `git fetch origin && git log mm/v23.05.5..origin/mm/v23.05.5` to see what's new on Morse's branch, then cherry-pick or merge selectively. Most changes since fork point are unlikely to conflict — the divergence is in `target/linux/bcm27xx/` (which Morse hasn't touched for Pi 5).
- **Refresh patches against a kernel bump**: standard OpenWrt `quilt` workflow — `make target/linux/{clean,prepare} V=sc QUILT=1`, then iterate. The vendored 24.10 patches generally still apply against 6.6.x point releases.

## Verified mesh topology (2026-04-26)

Tested end-to-end: Gate node (Pi 4, `setup-haven-gate.sh`) + Point node (Pi 5, this build) over HaLow mesh with BATMAN-adv. Internet forwarded from gate through HaLow to clients on the point's 2.4 GHz AP.

| Node | Hardware | HaLow radio | Client AP | BATMAN role | IP |
|---|---|---|---|---|---|
| Gate (green) | Pi 4 + MM8108 USB | 802.11s mesh, ch28, SAE | 5 GHz onboard | `gw_mode=server` | 10.41.0.x |
| Point (blue) | Pi 5 + MM8108 USB | 802.11s mesh, ch28, SAE | 2.4 GHz Panda USB | `gw_mode=client` | 10.41.0.2 |

- Mesh peer link: ESTAB, signal -3 to -7 dBm (devices nearby during testing)
- BATMAN originator throughput: 32.5 Mbps
- Gate ping RTT: 3–7 ms over HaLow
- Internet (8.8.8.8) RTT: 14–20 ms end-to-end through gate
- Client devices (phone on 2.4 GHz AP) get DHCP from gate via BATMAN/bat0 bridge

## TODO

- **LuCI theme: port `luci-theme-argon`** — The current UI uses `luci-theme-morse` (Morse Micro's purple-branded bootstrap fork). Replace with [luci-theme-argon](https://github.com/jerrykuku/luci-theme-argon) for a modern look (dark mode, card layout, responsive sidebar). Argon targets OpenWrt 21.02+ so it should be compatible with the 23.05 base. Package it as `luci-theme-haven`, swap in Haven branding (logo, colors), and add `CONFIG_PACKAGE_luci-theme-haven=y` to `boards/ekh-bcm2712/target_diffconfig`. As an interim step, forking `luci-theme-morse` with just logo/color swaps is also viable.

## Related but separate

- **haven-manet-ip-mesh-radio** ([buildwithparallel/haven-manet-ip-mesh-radio](https://github.com/buildwithparallel/haven-manet-ip-mesh-radio)) contains node setup scripts for gate and point nodes (`scripts/node-setup/setup-haven-gate.sh`, `setup-haven-point.sh`). Written for Pi 4 images but the UCI patterns are identical for Pi 5. Use these as the reference for configuring mesh + BATMAN-adv + client AP topology.
