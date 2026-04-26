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
- **No CI.** Builds happen on the maintainer's host. Reproduce with `boards/ekh-bcm2712/target_diffconfig` per the README.

## How to extend

- **Add a package**: edit `boards/ekh-bcm2712/target_diffconfig`, add `CONFIG_PACKAGE_<name>=y`, rebuild. If the package's deps include something Pi 5–specific (e.g. a per-platform variant), check whether a `*-bcm2712` variant exists; if not, that's a new package to write.
- **Add a Pi 5 device tree change**: drop a patch in `target/linux/bcm27xx/patches-6.6/`. Numbering 991-* is Morse-overlay, 992+ is our additions. Re-anchor against 6.6 paths (DTS files moved from `arch/arm/boot/dts/` to `arch/arm/boot/dts/broadcom/`).
- **Sync with upstream Morse changes**: `git fetch origin && git log mm/v23.05.5..origin/mm/v23.05.5` to see what's new on Morse's branch, then cherry-pick or merge selectively. Most changes since fork point are unlikely to conflict — the divergence is in `target/linux/bcm27xx/` (which Morse hasn't touched for Pi 5).
- **Refresh patches against a kernel bump**: standard OpenWrt `quilt` workflow — `make target/linux/{clean,prepare} V=sc QUILT=1`, then iterate. The vendored 24.10 patches generally still apply against 6.6.x point releases.

## Related but separate

- **OpenMANET** ([OpenMANET/firmware](https://github.com/OpenMANET/firmware)) is a parallel community fork on OpenWrt 24.10 that also includes Morse HaLow. Useful as a *layout reference* (boards/, scripts/, README structure), but it's a different SDK base — do not copy code or architecture between the two without explicit reason.
